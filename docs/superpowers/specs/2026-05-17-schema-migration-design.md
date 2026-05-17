# Schema Migration Feature Design

## Overview

Add schema migration capabilities to pgconsole by integrating with [pgschema](https://github.com/pgplex/pgschema) CLI and a git-based schema source. Users can visually compare their database's current schema against a desired state stored in a git repository, review the diff, and apply migrations — all within the pgconsole UI.

## Configuration

Each connection can optionally bind a git-based schema source in `pgconsole.toml`:

```toml
[[connections]]
id = "staging"
name = "Staging"
host = "staging.example.com"
port = 5432
database = "myapp"
username = "app_user"
password = "staging_password"

[connections.schema_source]
repo = "https://github.com/myorg/db-schema.git"
branch = "main"                    # optional, default: repo default branch
path = "schema/main.sql"           # path to schema entry file (single file or multi-file main.sql)
schema = "public"                  # optional, default: "public"
```

### Config Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `repo` | string | Yes | Git repository URL |
| `branch` | string | No | Branch to track (default: repo default branch) |
| `path` | string | Yes | Path to schema entry file within the repo |
| `schema` | string | No | PostgreSQL schema to compare against (default: `public`) |

### Git Authentication

The server relies on the host environment's git credentials (SSH agent, git credential helper, etc.). No git auth is stored in `pgconsole.toml`. This means the server process must have access to credentials that can clone the configured repos.

### Config Parsing

Add `SchemaSourceConfig` interface and parse `schema_source` sub-table within each `[[connections]]` entry in `server/lib/config.ts`. Extend `ConnectionConfig` with an optional `schema_source` field.

## Backend: MigrationService

New ConnectRPC service in `proto/migration.proto` and `server/services/migration-service.ts`.

### Proto Definition

```protobuf
syntax = "proto3";

package migration.v1;

service MigrationService {
  rpc PlanMigration(PlanMigrationRequest) returns (PlanMigrationResponse);
  rpc ApplyMigration(ApplyMigrationRequest) returns (stream ApplyMigrationResponse);
  rpc GetSchemaSourceStatus(GetSchemaSourceStatusRequest) returns (GetSchemaSourceStatusResponse);
}

message PlanMigrationRequest {
  string connection_id = 1;
}

message PlanMigrationResponse {
  string plan_id = 1;              // unique ID for this plan (used in ApplyMigration)
  string branch = 2;               // git branch used
  string commit_hash = 3;          // git commit hash
  string source_fingerprint = 4;   // pgschema schema fingerprint (detects concurrent changes)
  repeated SchemaDiff diffs = 5;
  bool can_run_in_transaction = 6;  // whether all diffs can run in a single transaction
  string summary = 7;              // human-readable summary (e.g., "2 to add, 1 to modify")
}

message SchemaDiff {
  string sql = 1;                  // DDL statement to execute
  string type = 2;                 // table, index, view, function, trigger, etc.
  string operation = 3;            // create, alter, drop
  string path = 4;                 // fully qualified name (e.g., "public.users")
  bool can_run_in_transaction = 5;
}

message ApplyMigrationRequest {
  string connection_id = 1;
  string plan_id = 2;             // reference to a previously generated plan
}

message ApplyMigrationResponse {
  int32 step = 1;                  // current step number
  int32 total_steps = 2;
  string sql = 3;                  // DDL being executed
  string status = 4;              // "running", "completed", "failed"
  string error = 5;               // error message if failed
}

message GetSchemaSourceStatusRequest {
  string connection_id = 1;
}

message GetSchemaSourceStatusResponse {
  bool configured = 1;
  string repo = 2;
  string branch = 3;
  string path = 4;
  string schema = 5;
}
```

### Core Flow

#### PlanMigration

1. Look up connection config and validate `schema_source` is set
2. Permission check: user needs `read` permission (plan is read-only, just shows diff)
3. Clone or pull the git repo to a server-local cache directory:
   - Path: OS temp dir + `/pgconsole-schema/{connectionId}/`
   - First call: `git clone --branch {branch} --depth 1 {repo}`
   - Subsequent calls: `git fetch origin {branch} && git reset --hard origin/{branch}`
4. Execute pgschema CLI:
   ```bash
   pgschema plan \
     --host {conn.host} --port {conn.port} --db {conn.database} \
     --user {conn.username} --password {conn.password} \
     --sslmode {conn.ssl_mode} --schema {schema_source.schema} \
     --file {repo_dir}/{schema_source.path} \
     --output-json {temp_dir}/plan.json
   ```
5. Parse the JSON plan output
6. Store plan in memory (keyed by generated plan_id) for later apply
7. Return structured diff response

#### ApplyMigration

1. Permission check: user needs `ddl` permission
2. Look up the stored plan by plan_id
3. Execute pgschema CLI:
   ```bash
   pgschema apply \
     --host {conn.host} --port {conn.port} --db {conn.database} \
     --user {conn.username} --password {conn.password} \
     --sslmode {conn.ssl_mode} \
     --plan {plan_file_path} \
     --auto-approve \
     --lock-timeout {conn.lock_timeout}
   ```
4. Stream progress by monitoring pgschema stdout
5. Return step-by-step status via streaming response

#### GetSchemaSourceStatus

Simple config lookup — returns whether a connection has `schema_source` configured and its details.

### Plan Storage

Generated plans are stored in memory (a `Map<string, StoredPlan>`) with:
- `plan_id`: UUID generated on creation
- `plan_json_path`: path to the plan JSON file on disk
- `connection_id`: which connection this plan is for
- `created_at`: timestamp
- Plans expire after 30 minutes — apply requests for expired plans return an error asking the user to re-plan
- Plans are cleared on server restart

### Git Repo Caching

- Repos are cloned to `{os.tmpdir()}/pgconsole-schema/{connectionId}/`
- Shallow clone (`--depth 1`) to minimize disk usage
- Subsequent plan requests do `git fetch + reset` instead of full clone
- Cache is ephemeral — cleared on server restart (temp dir)

### Error Handling

- `schema_source` not configured → return clear error message
- Git clone/pull fails → return git error (network, auth, repo not found)
- pgschema binary not found → return error suggesting installation
- pgschema plan fails → return pgschema error output
- Plan expired (schema changed since plan) → pgschema fingerprint check catches this on apply

## Frontend: Schema Diff View

### Entry Point

In the existing Schema tab components (`TableSchemaContent.tsx`, `ViewSchemaContent.tsx`, etc.), add a **"Compare with Git"** button. This button only appears when the connection has `schema_source` configured (checked via `GetSchemaSourceStatus`).

Additionally, a connection-level "Schema Migration" entry could appear in the schema browser tree when `schema_source` is configured.

### Components

#### MigrationPanel

Main container component. Replaces or overlays the current schema detail view when migration mode is active.

**States:**
- **Idle** — "Compare with Git" button visible
- **Loading** — Spinner while `PlanMigration` executes (clone + pgschema analysis)
- **No Changes** — "Schema is up to date with git" message
- **Diff View** — Shows structured diff with apply option
- **Applying** — Progress indicator during `ApplyMigration`
- **Error** — Error message with retry option

#### DiffSummary

Displays change summary grouped by object type:

```
3 changes: 1 to add, 1 to modify, 1 to drop

Tables:
  + users_audit          CREATE
  ~ products             ALTER
    + discount_rate column
    - old_price column
  - legacy_data          DROP

Indexes:
  + idx_users_email      CREATE
```

Color coding: green for create, yellow/blue for alter, red for drop.

#### SQLPreview

Shows all DDL statements with syntax highlighting (reuse existing CodeMirror SQL highlighting). Read-only view.

#### ApplyConfirmation

Dialog before applying:
- Shows summary of changes
- Warns about non-transactional operations if `can_run_in_transaction` is false
- Requires explicit confirmation
- Only visible to users with `ddl` permission

### Permission Gating

| Action | Permission Required |
|--------|-------------------|
| View "Compare with Git" button | `read` (and `schema_source` configured) |
| Run PlanMigration (view diff) | `read` |
| Apply migration | `ddl` |

Uses existing `useConnectionPermissions` hook:
```tsx
const { hasRead, hasDdl } = useConnectionPermissions(connectionId)
```

### Data Flow

```
User clicks "Compare with Git"
  → Frontend calls PlanMigration RPC
  → Backend: git clone/pull → pgschema plan --output-json → parse JSON
  → Frontend receives PlanMigrationResponse
  → Renders DiffSummary + SQLPreview

User clicks "Apply Plan"
  → Frontend shows ApplyConfirmation dialog
  → User confirms
  → Frontend calls ApplyMigration RPC (streaming)
  → Backend: pgschema apply --plan → stream progress
  → Frontend shows step-by-step progress
  → On completion: refresh schema cache + show success
```

## Testing Strategy

### Backend Tests

- Config parsing: validate `schema_source` sub-table parsing, missing fields, invalid values
- MigrationService: mock pgschema CLI output, test plan JSON parsing
- Permission checks: verify `ddl` required for apply, `read` sufficient for plan

### Frontend Tests

- Component rendering: diff summary with various change types
- Permission gating: apply button hidden without `ddl`
- State transitions: loading → diff view → applying → success

## Dependencies

- **pgschema CLI**: Must be installed on the server and available in PATH
- **git**: Must be installed on the server
- Server environment must have git credentials configured for accessing private repos

## Out of Scope (for initial implementation)

- Selective apply (applying only some diffs) — pgschema applies the full plan
- Migration history/audit log
- Git branch selection in UI (uses configured branch)
- Schema editing in UI (edit happens in git)
- Rollback functionality
- Multi-schema support per connection
