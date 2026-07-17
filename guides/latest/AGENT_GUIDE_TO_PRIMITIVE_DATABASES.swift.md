# Working with Databases in the Primitive platform

Guidelines for building apps with Primitive's server-side database storage.

## Client operations

### Run a registered operation

```swift
  let result = try await client.databases.executeOperation(
    databaseId: databaseId,
    name: "list-products",
    options: ExecuteOperationOptions(params: ["search": "widget"])
  )
  // result: { data: [...records], hasMore, nextCursor? }
```

### List / get databases

```swift
  // Databases where you're owner or manager
  let databases = try await client.databases.list()

  // Resolve a database by id — gated on owner/manager permission, an effective
  // group permission, or app-admin access; otherwise the server returns 403.
  let db = try await client.databases.get(databaseId: databaseId)
```

### Grant a group access

```swift
  _ = try await client.databases.grantGroupPermission(
    databaseId: databaseId,
    params: GrantDatabaseGroupPermissionParams(
      groupType: "team", groupId: "engineering", permission: "manager"
    )
  )
```


## Core Concept: Databases

A **database** is:

1. An isolated, server-side data store (SQLite-backed)
2. Accessed through server-configured **registered operations** with CEL-based access control
3. Schemaless — save any JSON records without upfront schema definition

**Properties:**

- Each database is an isolated instance with its own SQLite storage — strong consistency, zero-config scaling
- End-user scoped app access goes through **registered operations** with per-operation authorization; owners, managers, and app admins additionally have direct record-access APIs (schemaless save/patch/find/query/delete/count, atomic increment and StringSet ops, batch writes, aggregation, and index management) for administrative access
- Databases can be organized by **type** — a named configuration shared across many database instances
- Supports queries, mutations, counts, aggregates, multi-step pipelines, atomic operations, batch writes, apply-to-query, and real-time subscriptions

**Size Guidelines:** Individual databases can hold up to ~5 GB of data. For larger needs, split data across multiple databases (one per tenant, project, or domain) — each is a separate isolated instance that scales independently.

## When to Use Databases vs. Documents

Primitive offers two storage options: **documents** (local-first, real-time collaborative) and **databases** (server-side, fine-grained access control). Many apps use both — documents for personal/collaborative data, databases for app-wide shared data.

See the [Data Modeling guide](AGENT_GUIDE_TO_PRIMITIVE_DATA_MODELING.md) for a full decision framework, comparison table, and example app architectures.

## Quick Start

Database setup has two phases: **configuration** (via CLI and TOML config files) and **runtime usage** (via client library).

### 1. Configure database types and operations via CLI

Initialize a sync directory and create config files:

```bash
primitive sync init
```

Create a database type config with operations in `config/database-types/project.toml`:

```toml
[type]
databaseType = "project"
celContextAccess = "isMemberOf('team', database.celContext.teamId)"
timestamps = { create = "createdAt", update = "modifiedAt" }

[triggers.tasks]
triggers = [
  { on = "create", set = { createdBy = "user.userId" } },
  { on = "save", when = "record.status == 'done' && record.completedAt == null", set = { completedAt = "now()" } },
]

[[operations]]
name = "listTasks"
type = "query"
modelName = "tasks"
access = "isMemberOf('team', database.celContext.teamId)"

[operations.definition]
filter = { projectId = "$params.projectId" }
sort = { createdAt = -1 }
limit = 50

[[operations.params]]
name = "projectId"
type = "string"
required = true

[[operations]]
name = "createTask"
type = "mutation"
modelName = "tasks"
access = "isMemberOf('team', database.celContext.teamId)"

[[operations.definition.operations]]
op = "save"

[operations.definition.operations.data]
title = "$params.title"
projectId = "$params.projectId"
status = "open"
createdBy = "$user.userId"

[[operations.params]]
name = "title"
type = "string"
required = true

[[operations.params]]
name = "projectId"
type = "string"
required = true
```

Files written before migration may encode `definition`/`params` as single-line JSON strings — the server treats the encodings identically; see [Operation types](#operation-types) and `primitive sync migrate-toml` for converting a file.

Push configuration to the server:

```bash
primitive sync push
```

### 2. Use databases in app code

Construct the client, then create a database and call its registered operations:


```swift
  // Create a database instance of the configured type
  let db = try await client.databases.create(params: CreateDatabaseParams(
    title: "Alpha Project", databaseType: "project"
  ))

  // Execute registered operations
  let result = try await client.databases.executeOperation(
    databaseId: db.databaseId, name: "listTasks",
    options: ExecuteOperationOptions(params: ["projectId": "proj-1"])
  )

  let createResult = try await client.databases.executeOperation(
    databaseId: db.databaseId, name: "createTask",
    options: ExecuteOperationOptions(params: ["title": "Ship v1", "projectId": "proj-1"])
  )
  // executeOperation returns a JSONValue; mutation result shape is { results: [{ id }] }.
  let taskId = createResult["results"]?.arrayValue?.first?["id"]?.stringValue // server-assigned ULID
```

## Configuring with the CLI

All database configuration — types, operations, triggers, rule sets, group types — is managed through TOML config files and the `primitive sync` command. This keeps configuration version-controlled alongside your code.

```bash
primitive sync init            # Initialize config dir (auto-resolves .primitive/sync/<env>/<appId>/)
primitive sync pull            # Pull current config from server
primitive sync diff            # Preview changes
primitive sync push            # Push local config to server
primitive sync push --dry-run  # See what would change without applying
primitive sync migrate-toml    # Rewrite database-type and workflow files to native TOML tables
primitive sync migrate-toml --dry-run  # Preview the rewrite without writing files
# Override with a fixed path:
primitive sync init --dir ./config
primitive sync push --dir ./config
```

`migrate-toml` is a purely local rewrite — it converts JSON-string fields to native TOML tables in place, semantically identical on the server: `definition`/`params` in database-type files (see [Operation types](#operation-types) for the two forms and the fallback rules) and `inputSchema`/`outputSchema` in workflow files.

The config directory structure:

```
config/
  database-types/*.toml           # Database type configs + operations
  rule-sets/*.toml                # Access rule sets (CEL rules)
  group-type-configs/*.toml       # Group type configs
```

### Database type config files

Each file in `database-types/` defines a database type with its triggers and operations.

**TOML quoting tip:** CEL expressions containing apostrophes (e.g., `isMemberOf('class-teachers', database.id)`) must use triple-quoted strings in TOML, because single-quoted TOML strings don't support escaping. Use `'''...'''` for literal strings or `"""..."""` for basic strings:

```toml novalidate
# WRONG — TOML parse error due to apostrophes in single-quoted string:
access = 'isMemberOf('team', database.metadata.teamId)'

# CORRECT — triple-quoted literal string:
access = '''isMemberOf('team', database.metadata.teamId)'''

# ALSO CORRECT — double-quoted string with escaped quotes or no apostrophes:
access = "isMemberOf('team', database.metadata.teamId)"
```

Note: Double-quoted strings (`"..."`) also work for CEL with single quotes inside, since TOML only requires escaping the quote character that delimits the string.

**File:** `config/database-types/project.toml`

```toml
[type]
databaseType = "project"
ruleSetName = "project-admin-rules"    # optional — rule set for managing this type's config
celContextAccess = "isMemberOf('team', database.celContext.teamId)"  # optional — CEL for CEL context updates
timestamps = { create = "createdAt", update = "modifiedAt" }         # optional — auto-stamp timestamps on every write

[triggers.tasks]
triggers = [
  { on = "create", set = { createdBy = "user.userId" } },
  { on = "save", when = "record.status == 'done' && record.completedAt == null", set = { completedAt = "now()" } },
]

[[operations]]
name = "listTasks"
type = "query"
modelName = "tasks"
access = "isMemberOf('team', database.metadata.teamId)"
[operations.definition]
filter = { projectId = "$params.projectId" }
sort = { createdAt = -1 }
limit = 50
projection = { title = 1, status = 1 }

[[operations.params]]
name = "projectId"
type = "string"
required = true

[[operations]]
name = "createTask"
type = "mutation"
modelName = "tasks"
access = "isMemberOf('team', database.metadata.teamId)"
[[operations.definition.operations]]
op = "save"

[operations.definition.operations.data]
title = "$params.title"
projectId = "$params.projectId"
status = "open"
createdBy = "$user.userId"
createdAt = "$now"

[[operations.params]]
name = "title"
type = "string"
required = true

[[operations.params]]
name = "projectId"
type = "string"
required = true

[[operations]]
name = "countOpenTasks"
type = "count"
modelName = "tasks"
access = "isMemberOf('team', database.metadata.teamId)"
[operations.definition]
filter = { projectId = "$params.projectId", status = "open" }

[[operations.params]]
name = "projectId"
type = "string"
required = true

[[operations]]
name = "tasksByStatus"
type = "aggregate"
modelName = "tasks"
access = "isMemberOf('team', database.metadata.teamId)"
[operations.definition]
groupBy = ["status"]
filter = { projectId = "$params.projectId" }
sort = { field = "total", direction = -1 }
limit = 10
operations = [
  { type = "count", outputField = "total" },
  { type = "sum", field = "estimatedHours", outputField = "totalHours" },
]

[[operations.params]]
name = "projectId"
type = "string"
required = true

[[operations]]
name = "projectDashboard"
type = "pipeline"
modelName = "_pipeline"
access = "isMemberOf('team', database.metadata.teamId)"
[operations.definition]
return = "all"

[[operations.definition.steps]]
name = "recentTasks"
type = "query"
modelName = "tasks"
filter = { projectId = "$params.projectId" }
sort = { createdAt = -1 }
limit = 5

[[operations.definition.steps]]
name = "statusBreakdown"
type = "aggregate"
modelName = "tasks"
filter = { projectId = "$params.projectId" }
groupBy = ["status"]
operations = [{ type = "count", outputField = "total" }]

[[operations.definition.steps]]
name = "openBugs"
type = "count"
modelName = "tasks"
filter = { projectId = "$params.projectId", label = "bug", status = "open" }

[[operations.params]]
name = "projectId"
type = "string"
required = true
```

### Rule set config files

Rule sets define CEL-based access rules for managing database type configuration and group operations.

**File:** `config/rule-sets/project-admin-rules.toml`

```toml
[ruleSet]
name = "project-admin-rules"
resourceType = "database_type"
description = "Controls who can manage project database type config and operations"

[rules.config]
edit = "hasRole('admin')"

[rules.operations]
create = "hasRole('admin')"
edit = "hasRole('admin')"
delete = "hasRole('admin')"
```

### Group type config files

Group types configure how groups behave. See the [Users and Groups guide](AGENT_GUIDE_TO_PRIMITIVE_USERS_AND_GROUPS.md) for full group documentation.

**File:** `config/group-type-configs/team.toml`

```toml
[groupTypeConfig]
groupType = "team"
ruleSetName = "team-rules"     # optional — rule set for group management
autoAddCreator = true          # auto-add creator as member (default: true)
```

### CLI commands for direct database management

Beyond sync, the CLI exposes commands for one-off ops (use `--help` for full flags):

```bash
primitive database-types list | get <type> | delete <type> | operations list <type>
primitive databases list | get <id> | create "Title" --type <type> [--cel-context '{...}'] [--initial-metadata '{...}'] | delete <id>
primitive databases cel-context update <id> --data '{"teamId":"team-1"}'

# Admin record introspection
primitive databases records models <id>
primitive databases records describe <id> <model>
primitive databases records query <id> <model> --filter '{"status":"open"}'

# Data migration (records + indexes + constraints; type config excluded — run sync push on target first)
primitive databases export <id> --output ./out
primitive databases import ./out --overwrite [--dry-run] [--batch-size 5000] [--stop-on-error]

# CSV bulk import — load a CSV file into a database via a batch save operation
primitive databases import-csv <database-id> <file.csv> --model <name> \
  [--operation seed_save] [--column-map '{"CSV Header":"field"}'] \
  [--types '{"price":"number"}'] [--id-column <col>] [--batch-size 5000] \
  [--delimiter ,] [--dry-run] [--stop-on-error] [--json]
```

`databases import` writes records in chunked batch requests: `--batch-size` sets records per request (default 5000, ceiling 25000; an invalid value fails before any write). A failing chunk is reported and the run continues by default — the command still exits non-zero at the end; `--stop-on-error` aborts the whole run (including a multi-database export dir) at the first failing chunk. Record upserts are keyed by `_id`, so re-running an import after a partial failure is safe.

`database-types delete <type>` refuses with a 409 when live database instances of that type still exist — delete the instances first (`primitive databases delete <id>`) or pass `--force` to delete the type anyway (this orphans the instances). `-y`/`--yes` only skips the confirmation prompt; it does not bypass the guard. Once the 409 guard passes, the delete cascades: the type's operations and subscriptions are removed first, then the type config row is removed as the commit point — a failure removing a child leaves the whole type intact and the delete is retryable. The response reports the cascade counts: `{ success: true, deletedOperations: number, deletedSubscriptions: number }`.


## Database Types

A **database type** is a named configuration shared across many databases. It provides:

- **Registered operations** (`type` is one of `query`, `mutation`, `count`, `aggregate`, `pipeline`, `applyToQuery`) with per-operation CEL `access`
- **Triggers** — computed fields evaluated server-side before each save
- **`celContextAccess`** — CEL expression that lets non-owner/manager users read and update the database's CEL context (defaults to deny when unset; owner/manager always have access)
- **`autoPopulatedFields`** — declarative server-side field stamping on writes (see below)
- **`defaultAccess`** — fallback CEL access rule applied to operations that omit their own `access`
- **`[models.*]` schema** — optional server-enforced model declaration. When present, every op edit (and the schema edit itself) is checked against it; see [Schema gate](#schema-gate)
- **Subscriptions** — type-scoped real-time subscription definitions (managed via `[[subscriptions]]` blocks in the TOML)
- **Rule set attachment** — controls who can edit the type config and its operations

Swift `executeOperation` can call **any** registered operation regardless of its `type` (it returns `JSONValue`). Operation *management* is narrower: Swift's `DatabaseOperationType` enum — used by `createOperation` / `listOperations` — models only `query`, `mutation`, `count`, and `aggregate`. Define and manage `pipeline` and `applyToQuery` operations through TOML and the `primitive` CLI rather than the typed Swift management API.

Real-time subscriptions are also part of the type config — see [Real-Time Subscriptions](#real-time-subscriptions). One subscription definition serves every database of that type. Define them as `[[subscriptions]]` blocks in the same TOML file; `primitive sync push` manages them alongside operations.

### Triggers

Triggers are computed fields that run server-side before a record is saved. Configured per model in the `[triggers.<modelName>]` block of the database type TOML.

**Trigger fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `on` | string | Yes | When to fire: `"create"`, `"update"`, or `"save"` (both) |
| `when` | string | No | CEL boolean expression — trigger only fires if true. Errors → trigger does not fire (silent deny) |
| `set` | object | Yes | Map of field name to CEL expression. Each expression is evaluated and assigned. If the expression errors, the field is NOT set (returns `undefined`) |

**Trigger CEL context** (note: bare CEL expressions, NOT `$` substitution syntax):

| Variable | Description |
|----------|-------------|
| `user.userId` | The user performing the save |
| `user.role` | The user's app role |
| `record.*` | The record being saved (current field values) |
| `database.id` | The database ID |
| `database.celContext.*` | The database's CEL context object (also accessible as `database.metadata.*`) |
| `secrets.*` | App secrets — binds only the keys declared in the config's `secrets` manifest (loaded when the expression references `secrets.`) |
| `now()` | Current ISO 8601 timestamp |
| `lookup(modelName, id)` | Load another record by ID; returns `null` if missing |
| `isMemberOf(groupType, groupId)`, `memberGroups(groupType)`, `hasRole(role)` | Membership/role checks |
| `fromWorkflow()`, `fromWorkflow(key)` | True if the write was issued by a workflow step (or a specific workflow) |

**Don't confuse trigger CEL with operation substitutions.** Triggers use bare CEL — `user.userId`, `now()`. Operation `definition` and `params` use `$user.userId`, `$now`, `$params.x`, `$database.celContext.x` substitutions, which are deep string-replacement on the JSON template, NOT CEL. (`$database.metadata.x` is accepted as an alternate alias for the same value.)

A declared [resource metadata](AGENT_GUIDE_TO_PRIMITIVE_RESOURCE_METADATA.md) category value is also substitutable this way: `$md.self.<category>.<key>` in `filter`/`data`, resolving to `null` (not the literal string) when the category isn't declared on the database type's manifest or the key is missing. Access rules (`access`, `defaultAccess`) can likewise reference `md.self.<category>.<key>` and, where declared, `md.caller.<category>.<key>` (the caller's own metadata, rooted at `user.userId`) — see the Resource Metadata guide for declaring the manifest.

### Timestamps

The `timestamps` knob on the `[type]` config stamps `createdAt` and/or `modifiedAt` fields automatically on `save` and `patch` writes, removing the need for per-model trigger boilerplate. Field names are caller-configurable. Stamp values are epoch milliseconds (numbers), written only when the field is absent or `null` in the submission. `increment`, `addToSet`, and `removeFromSet` do not stamp — those ops neither set the create field nor bump the update field.

```toml
[type]
databaseType = "project"
timestamps = { create = "createdAt", update = "modifiedAt" }
```

| Key | Type | Description |
|-----|------|-------------|
| `create` | string | Field name stamped with the current time (epoch milliseconds) on record creation. Optional. |
| `update` | string | Field name stamped on every save and patch (including the first save). Optional. |
| `models` | string[] | Restrict stamping to these models only. Optional — omit to stamp every model under the type. |

Either lifecycle key can be omitted independently: `timestamps = { create = "createdAt" }` stamps only on create. Field names are yours to choose (`createdAt`/`modifiedAt`, `created_at`/`updated_at`, etc.).

```toml
# Only stamp accounts and holdings
timestamps = { create = "createdAt", update = "modifiedAt", models = ["accounts", "holdings"] }
```

If a model has both an explicit `[[triggers]]` block AND `timestamps`, the trigger fires after the auto-stamp (last writer wins). Since the auto-stamp checks "is the field absent" first, an explicit trigger that sets the field always wins on its lifecycle.

Use `timestamps` for the common "every save/patch should have a created/modified stamp" pattern. Use per-model triggers when the rule depends on the record's data (e.g. `completedAt` only when `status == "done"`). Use `autoPopulatedFields` when you need CEL-resolved values beyond `now()` (e.g. `updatedBy = "user.userId"`).

### Auto-populated fields

`autoPopulatedFields` is a declarative alternative to writing the same `createdAt`/`createdBy`/`updatedAt` triggers on every model. Set it on the database type config; the engine stamps the listed fields server-side, applied per op-kind (`create` ⇔ `save` ⇔ `applyToQuery` insert path; `update` ⇔ `patch` ⇔ `applyToQuery` update path).

```toml
[type]
databaseType = "project"

[type.autoPopulatedFields]
ownerId  = "user.userId"                                       # shorthand → on = ["create"]
createdAt = { value = "now()", on = "create" }
updatedAt = { value = "now()", on = ["create", "update"] }
```

Each entry is either:
- A **shorthand** string — the CEL expression to evaluate. Defaults to `on = ["create"]`.
- A **verbose** map `{ value, on }`. `value` is the CEL expression. `on` is `"create"`, `"update"`, or an array of those (the string form is normalized to a single-element array). Other values are rejected.

The `value` CEL shares the same context as the operation's access expression: `user.*`, `database.*`, `params.*`, plus `now()` and the membership functions. Stamps reference `secrets.*` only when needed (the engine refuses to evaluate stamps that read secrets if the operation didn't pre-declare it).

Use auto-populated fields for cross-model invariants (every write should stamp a timestamp); use per-model triggers for invariants that depend on the record's data (e.g. setting `completedAt` only when `status == "done"`).

### `defaultAccess`

Specify a CEL access rule that applies to operations that don't declare their own `access`:

```toml
[type]
databaseType = "project"
defaultAccess = "isMemberOf('team', database.celContext.teamId)"
```

An operation can override the default by setting its own `access`. Without `defaultAccess` (and no per-operation rule), the operation is denied for every caller — operation `access` rules have no owner/manager/admin bypass.

### Schema gate

A database type can carry an optional schema — one or more `[models.<Name>.fields.<field>]` blocks in the same TOML file as the type config. When a schema is present, the server enforces consistency between ops and the schema in both directions:

- **Op-edit gate.** Creating or editing an operation runs every static `modelName` and field reference in `filter` / `projection` / `sort` / `data` / `access` / mutation `condition` against the schema. References that don't resolve fail with HTTP 422 `OPERATION_REFERENCES_UNDEFINED` (the response payload lists each unresolved ref).
- **Schema-edit gate.** Editing or deleting the schema runs every existing op against the proposed new schema. If any op would break, the edit fails with HTTP 422 `SCHEMA_BREAKS_OPERATIONS`. If any op contains references the gate can't statically resolve (see "Dynamic references" below), the edit fails with HTTP 422 `SCHEMA_HAS_UNCHECKABLE_OPS` until you re-run with `primitive sync push --accept-warnings`. Deleting the schema entirely (removing all `[models.*]` blocks from the local file) is rejected with HTTP 409 `OPS_EXIST` while any operations are still registered.

```toml
# config/database-types/inventory.toml
[type]
databaseType = "inventory"

[models.product.fields.id]
type = "id"
auto_assign = true
indexed = true

[models.product.fields.name]
type = "string"

[models.product.fields.priceCents]
type = "number"
indexed = true

[[operations]]
name = "list-products"
type = "query"
modelName = "product"
access = "true"
[operations.definition]
filter = { priceCents = { "$gt" = 0 } }
```

Adding the `[models.product.fields.*]` blocks above means any future `[[operations]]` that references `product.nameTypo` (instead of `name`) is rejected at push time, before it can return broken data. Note the 422 *message* doesn't name the offending field — the unresolved refs are in the response payload's `refs` array.

**Field types — no object/JSON type.** `[models.*.fields.*]` accepts `string`, `number`, `boolean`, `date`, `id`, `stringset` only. `type = "object"` fails with `TOML_PARSE_ERROR: Invalid field type` — even though operation **params** do accept `"object"`. To store a structured payload (nested objects/arrays), declare the field as `string`, write JSON-encoded values, and parse on the consumer side — e.g. `parse_json(input.payload)` in a workflow `script` step.

**Dynamic references.** Some refs can't be statically checked — for example, `modelName = "$params.kind"` (model determined at call time), a raw `filter` or `projection` lifted from `$params`, `record[expr]` field access, or lambda bodies inside pipelines. These are surfaced as *warnings*, not errors. The op is accepted, but the schema-edit gate later flags them so the operator can review whether the dynamic reference is still consistent with the new schema.

**Bootstrapping an existing type.** If a type already has registered operations and you want to add a schema retroactively, use the scaffolding command:

```bash
primitive databases schema generate <database-type>
```

It calls a server-side endpoint that inspects existing ops + introspects the live database, infers field types where it can, and splices a `[models.*]` block into the local `config/database-types/<type>.toml` file (just before the first `[[operations]]` block). The scaffold is enriched so the output is directly pushable: in addition to sampling live records, it infers field types from how operation params are used, marks fields that operation params require as `required = true`, and emits string enum constraints for fields whose params restrict them to a fixed value set. Review the result — the generator still guesses from observed values, and you may need to fix what it got wrong — then run `primitive sync push` (or `primitive sync push --dry-run` first) to attach it.

**Schemaless types.** A type without any `[models.*]` block is unchanged from the pre-gate behavior — ops are accepted without static consistency checks. Once you add a schema, the consistency invariant holds: the schema-edit gate prevents removing it while ops remain, so future op edits stay aligned with the schema.

## Registered Operations

Registered operations are named, parameterized database operations defined at the database-type level. They are the primary data access layer — all user interaction with database data goes through operations.

### Access control model

- **All end-user data access** goes through registered operations, controlled by each operation's `access` CEL expression
- **Owner/manager** are administrative roles for managing the database itself — not for granting end-user access (see [Permissions](#permissions))
- If no operations are registered, non-owner/manager users are denied access entirely

### CEL access expressions

Each operation has an `access` field — a CEL expression evaluated at call time:

```
"true"                                             // all authenticated users
"hasRole('admin')"                                 // app admins only
"isMemberOf('team', database.celContext.teamId)"   // team members
"user.userId == params.userId"                     // only your own data
```

The shared identity context (`user.*`, `hasRole`, `isMemberOf`, `memberGroups`, `fromWorkflow`) is documented in the [Access Control guide](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md). **Database-specific CEL context:** `database.id`, `database.celContext`, `database.metadata`, `params.*`.

Only `database.id` and the CEL context object are available in CEL — other database fields like `createdBy` are not exposed. `database.celContext` and `database.metadata` are aliases for the same object; prefer `database.celContext` in new code. To check ownership, store the creator's ID in metadata at creation time or use group membership.

`fromWorkflow()` is true when the call originated from a workflow step (any workflow); `fromWorkflow("key")` restricts to a specific workflow. Use it to lock down ops that should only be invoked by an internal workflow:

```toml
[[operations]]
name = "bulkUpdatePricesFromWorkflow"
access = "fromWorkflow('refresh-security-prices')"
# …
```

This is the cleanest expression of "only the named cron-fired workflow can call this — no user, not even an admin, can call it directly with a hand-crafted request." The workflow identity is only injected when the caller is the internal workflow step runner; external HTTP clients cannot spoof it.

An operation's `access` rule is evaluated on every call, including from a `runAs = "system"` workflow run — system runs are app-privileged on the raw document/record baseline but do **not** bypass a registered operation's own `access` CEL. So a workflow-only op must be `access = "fromWorkflow('key')"`; setting `access = "false"` denies the system run too (`403`).

For group-based access patterns, per-parameter access, and detailed CEL examples, see the [Users and Groups guide](AGENT_GUIDE_TO_PRIMITIVE_USERS_AND_GROUPS.md).

### Parameters

Declare parameters callers must provide — one `[[operations.params]]` entry per parameter:

```toml
[[operations]]
name = "listProjectTasks"
type = "query"
modelName = "tasks"
access = "true"

[operations.definition]
filter = { projectId = "$params.projectId", status = "$params.status" }

[[operations.params]]
name = "projectId"
type = "string"
required = true

[[operations.params]]
name = "status"
type = "string"
```

| Field | Required | Description |
|-------|----------|-------------|
| `type` | yes | One of `"string"`, `"number"`, `"boolean"`, `"integer"`, `"object"`, `"array"`, or the name of a model declared in the type's `schema` |
| `required` | no | Default `false`. If `true`, request fails 400 when missing |
| `access` | no | CEL expression evaluated against the caller's value (bound as `value`); false → 403 |
| `coerce` | no | Default `true`. If `false`, type-mismatched inputs are rejected instead of being coerced. Only valid on a scalar param — setting it on a model-typed param is rejected at registration |
| `enum` | no | Allowed values. Valid on a `"string"` param, or on an `"array"` param with `items = "string"` (checked per element) |
| `items` | no | Element type for `type = "array"`: `"string"`, `"number"`, `"boolean"`, or `"integer"`. Required when `type = "array"`, rejected on any other type |

By default, scalar mismatches are coerced where safe (`"42"` → `42` for type `"number"`, `"true"` → `true` for type `"boolean"`, etc.). Undeclared params (not in the schema) are rejected with 400.

**Array params.** A `type = "array"` param (typically feeding `$in`/`$nin` in a filter — `filter = { symbol = { "$in" = "$params.symbols" } }`) rejects non-array values with 400 (`Parameter "<name>" must be an array`) and validates each element against `items`, honoring `coerce` and reporting mismatches with the element index (`element 2 must be of type string`); `enum` membership is checked per element.

**Model-typed params.** A param's `type` can name a model declared in the type's `schema`, and the param arrives as an object validated field-by-field against that model — an unknown field is rejected (`UNKNOWN_FIELD`), a wrong field type is rejected (`FIELD_TYPE_MISMATCH`), and on a `save` op every required field must be present (`MISSING_REQUIRED_FIELD`; a `patch` validates only the fields present). The model must exist in the type's `schema` or registration is rejected (`PARAM_TYPE_SCHEMA_REQUIRED` / `PARAM_TYPE_UNKNOWN_MODEL`). A save/patch op writes the whole object with `"data": "$params.row"`, or merges its fields with the `$spread` directive:

```toml
[[operations]]
name = "createTask"
type = "mutation"
modelName = "tasks"
[[operations.definition.operations]]
op = "save"

[operations.definition.operations.data]
"$spread" = "$params.row"
userId = "$user.userId"

[[operations.params]]
name = "row"
type = "Task"
required = true
```

`$spread` expands the object's fields into the write; at most one `$spread` per data object, and it's not available inside an `applyToQuery` action's `data`. Explicit sibling keys and server-managed stamps (auto-populated fields, timestamps) always win over spread values, order-independent — so a server-stamped `userId` can't be spoofed from the caller's object. A `$spread` of a non-object value fails at execute time (`SPREAD_NON_OBJECT`).

**Per-parameter access control** restricts what value a caller may pass:

```toml novalidate
[[operations.params]]
name = "studentId"
type = "string"
required = true
access = "value in memberGroups('parent-of')"
```

### Substitution variables

Used **as string values inside operation definitions** (filter values, data fields, IDs, modelName). Substitution is a literal deep string-replacement on the JSON template — it is NOT CEL.

| Variable | Resolves to |
|----------|-------------|
| `$user.userId` | Current user's ID |
| `$now` | Current ISO 8601 timestamp |
| `$database.id` | The database instance ID |
| `$database.celContext.key` | Value from database CEL context (`null` if key missing) — `$database.metadata.key` resolves to the same value |
| `$params.fieldName` | Caller-provided parameter (`undefined` if not passed) |
| `$steps.stepName.<accessor>` | Pipeline cross-step reference (see Pipelines) |

**Missing-value rule (critical):** if a substitution resolves to `undefined` (e.g. an optional param the caller didn't pass), `JSON.stringify` drops the surrounding key entirely. The resulting filter / data object simply doesn't have that key — it is NOT set to `null` or empty string. Any value the caller actually passes (including `""`, `0`, `false`, explicit `null`) flows through verbatim.

This is how optional filters work: declare `{required: false}` and the filter narrows when the param is provided, matches all when it isn't.

```toml
[[operations]]
name = "listPosts"
type = "query"
modelName = "posts"
access = "true"
[operations.definition]
filter = { status = "approved", authorId = "$params.authorId" }
sort = { createdAt = -1 }
limit = 50

[[operations.params]]
name = "authorId"
type = "string"
required = false
```

| Caller input | Resolved filter |
|---|---|
| `{}` | `{"status":"approved"}` (all approved) |
| `{"authorId":"u-1"}` | `{"status":"approved","authorId":"u-1"}` |
| `{"authorId":""}` | `{"status":"approved","authorId":""}` (matches empty string — only `undefined` drops) |

> **Gotcha:** If a filter or data field references `$params.X` but `X` isn't declared in `params`, the substitution always resolves to `undefined` and the key silently drops — your operation becomes a match-all for that field, or your save omits the field entirely. The validator catches obvious cases at registration time, but always double-check the params schema covers every `$params.*` reference.

### Boolean gate conditions

Substitution variables (`$database.metadata.*`, `$params.*`, `$steps.*`) can appear **directly as elements** in `$and`/`$or` arrays (not as key-value pairs). The resolved boolean value gates that branch:

| Value | In `$and` | In `$or` |
|-------|-----------|----------|
| `true` | No-op — remaining conditions apply | Short-circuits to match-all |
| `false` / `null` / missing | Short-circuits to no-match (empty result, no DB hit) | Removed — other branches still apply |

**Common use case — per-database feature flag:**

```json
{
  "filter": {
    "$or": [
      { "authorId": "$user.userId" },
      { "$and": ["$database.metadata.peerVisibility", { "status": "approved" }] }
    ]
  }
}
```

When `$database.metadata.peerVisibility` is `true`, the `$and` branch includes approved posts. When it's `false` or missing, the branch short-circuits to no-match — users only see their own records. A missing key evaluates to `null`, so the gate is safely closed before the flag is set.

This also works with `$steps.*` references in pipelines (see [Settings record pattern](#settings-record-pattern)).

### Operation types

Operations are defined in the `[[operations]]` array in the database type TOML file. `definition` is written as an `[operations.definition]` table (nested tables like `[operations.definition.filter]`, or compact inline tables like `definition = { filter = { "$in" = [...] } }`; mutation steps as `[[operations.definition.operations]]` array-of-tables), and `params` as one `[[operations.params]]` entry per parameter (`name`, `type`, `required`, and any other param keys):

```toml
[[operations]]
name = "approvePost"
type = "mutation"
modelName = "posts"
access = "isMemberOf('class-teachers', database.id)"

[[operations.definition.operations]]
op = "patch"
id = "$params.id"

[operations.definition.operations.data]
status = "approved"

[[operations.params]]
name = "id"
type = "string"
required = true
```

`sync pull` writes new files in this form. Files written before migration may instead carry a single-line **JSON-string encoding** (`definition = '{"operations":[...]}'`, with `params` as an object keyed by param name) — the server treats the two identically, and `sync pull` preserves whichever form each operation already uses (per op, so mixed files stay mixed). Convert a file with `primitive sync migrate-toml` (add `--dry-run` to preview); it's a purely local rewrite. A value TOML cannot represent — a `null` (or `undefined`) anywhere in the value, including inside an array — stays a JSON string for that field on emit, with a log line naming the operation; the rest of the file stays native. Mixed-type arrays (a `$and` gate mixing a `$steps.*` reference with a filter object, say) carry natively — see [Settings record pattern](#settings-record-pattern).

`sync push` accepts both forms and warns (does not block) on an unrecognized filter operator — the supported set is `$eq`, `$ne`, `$gt`, `$gte`, `$lt`, `$lte`, `$in`, `$nin`, `$exists`, `$contains`, `$startsWith`, `$endsWith`, `$containsText`, plus logical `$and`/`$or`.

#### Query — read records

```toml
[[operations]]
name = "listTasks"
type = "query"
modelName = "tasks"
access = "true"
[operations.definition]
filter = { projectId = "$params.projectId" }
sort = { createdAt = -1 }
limit = 50
projection = { title = 1, status = 1 }

[[operations.params]]
name = "projectId"
type = "string"
required = true
```

Definition fields: `filter` (required), `sort`, `limit` (1–1000), `projection`, `include`.

`projection` values are exactly `1` (inclusion — returns the projected fields plus `id`) or `0` (exclusion — the named fields are stripped server-side before the response leaves the store, so exclusion withholds a column, it doesn't just shrink the payload). Validation, enforced both when the operation is saved and on the runtime request (400): inclusion and exclusion can't mix; an exclusion can't name `id` or `type` (system identity fields, always returned); and a query can't sort on a field its projection excludes (the sort value would leak through the pagination cursor).

Callers can override `limit`, `cursor`, and `direction` at call time (effective limit is the lesser of definition and caller limits).

**Response:** `{ data: [...], hasMore: boolean, nextCursor?: string, prevCursor?: string }` (`prevCursor` appears from the second page on)

**Cacheable queries.** Add an `[operations.definition.cache]` block to opt a query into a server-side result cache:

```toml
[[operations]]
name = "myOrders"
type = "query"
modelName = "orders"
access = "true"
[operations.definition]
filter = { ownerId = "$user.userId" }
sort = { createdAt = -1 }

[operations.definition.cache]
enabled = true
ttlMs = 30000
scope = "user"   # or "shared"
```

- `scope = "user"` folds the caller's identity into the cache key — each user gets their own cached entry.
- `scope = "shared"` serves one cache entry to every caller. The filter must contain no `$user.*` token — this is checked at config-save time; pushing a shared-scope operation with a `$user.*`-referencing filter is rejected with `400`, naming the offending filter path (e.g. `filter.owner`) in the error.
- `ttlMs` is clamped to a server-side ceiling (default 300000ms / 5 minutes).
- Every response from the operation carries an `X-Cache` header: `HIT` (served from cache), `MISS` (executed and stored), or `OFF` (no `cache` block configured — the default, fully back-compatible).
- A cache hit still runs the op's access check per caller (a denied caller never sees a cached result) and still reports `_timing` when `timing: true` is passed, with `cacheHit: 1` marking the hit.

#### Mutation — write records

Supports: `save`, `patch`, `delete`, `increment`, `addToSet`, `removeFromSet`.

```toml
[[operations]]
name = "createTask"
type = "mutation"
modelName = "tasks"
access = "isMemberOf('team', database.metadata.teamId)"
[[operations.definition.operations]]
op = "save"

[operations.definition.operations.data]
title = "$params.title"
status = "open"
createdBy = "$user.userId"
createdAt = "$now"

[[operations.params]]
name = "title"
type = "string"
required = true
```

**Mutation operation types:**

| Op | Description | Key fields |
|----|-------------|------------|
| `save` | Create or replace a record | `data`, optional `ifNotExists`, `condition`, `stringSets`, `upsertOn` |
| `patch` | Partial update | `id`, `data` |
| `delete` | Remove a record | `id` |
| `increment` | Atomic numeric increment/decrement | `id`, `fields` |
| `addToSet` | Add values to StringSet fields | `id`, `stringSets` |
| `removeFromSet` | Remove values from StringSet fields | `id`, `stringSets` |

**Response:** `{ results: [{ op, success, id, values? }] }` — one entry per definition step, in definition order, so a multi-step op (save + increment, two increments, …) reports every step. `op` names the step's kind (`"save"` | `"patch"` | `"delete"` | `"increment"` | `"addToSet"` | `"removeFromSet"`); `values` carries an increment step's post-increment counters.

**`upsertOn`** — pass the **field name** (`"upsertOn": "email"`, NOT a `$params.*` substitution) in a `save` op to create-or-update by a unique field instead of requiring an explicit `id`. The match **value** comes from `data` — the server looks up a record where that field equals `data.<field>`; if found, it patches it; if not, it inserts a new record. Useful for "ensure this user exists with these attributes" patterns:

```toml
[[operations]]
name = "ensureContact"
type = "mutation"
modelName = "contacts"
access = "hasRole('admin')"
[[operations.definition.operations]]
op = "save"
upsertOn = "email"

[operations.definition.operations.data]
email = "$params.email"
name = "$params.name"
updatedAt = "$now"

[[operations.params]]
name = "email"
type = "string"
required = true

[[operations.params]]
name = "name"
type = "string"
required = true
```

Two failure modes to know:

- `"upsertOn": "$params.email"` gets **value-substituted** like any other definition string, so the server receives the email *value* as the field name and rejects with `upsertOn field 'alice@example.com' must be present in data and not null/empty`.
- `upsertOn` requires a **unique index** on the field — declare `unique = true` on the field in the type schema. Databases created from the type provision the index automatically; a static `upsertOn` naming a field that isn't declared `unique = true` is rejected at push time. When you push a schema that newly declares a field `unique = true` (or `indexed = true`), the server back-provisions that index across every existing database of the type — the `sync push` / type-PATCH response carries a `reindexFanout` handle (`{ runId, instanceCount, statusUrl }`) you can poll at `.../databases/types/<type>/reindex-status?runId=...` for `{ status, total, completed, failed }`. Opt out with `?reindexInstances=false`. To back-provision a single database by hand (or after opting out), run `primitive databases reindex <database-id> --from-schema` (idempotent). Without the index, saves fail with `upsertOn field '<field>' does not have a registered unique index`.

#### Count — count matching records

```toml
[[operations]]
name = "countOpenTasks"
type = "count"
modelName = "tasks"
access = "true"
[operations.definition]
filter = { projectId = "$params.projectId", status = "open" }

[[operations.params]]
name = "projectId"
type = "string"
required = true
```

**Response:** `{ count: 42 }`

#### Aggregate — group and compute

```toml
[[operations]]
name = "tasksByStatus"
type = "aggregate"
modelName = "tasks"
access = "true"
[operations.definition]
groupBy = ["status"]
filter = { projectId = "$params.projectId" }
sort = { field = "total", direction = -1 }
limit = 10
operations = [
  { type = "count", outputField = "total" },
  { type = "sum", field = "estimatedHours", outputField = "totalHours" },
  { type = "avg", field = "estimatedHours", outputField = "avgHours" },
]

[[operations.params]]
name = "projectId"
type = "string"
required = true
```

Aggregate types: `count`, `sum`, `avg`, `min`, `max`.

**Response:** `{ result: { "open": { total: 15, totalHours: 120 }, ... } }`

#### Pipeline — multi-step read operations

Pipelines are **read-only** — they support `query`, `count`, and `aggregate` steps only. Mutations cannot be included in a pipeline. For server-side "query-then-mutate" in a single request, use `applyToQuery` instead (see below).

Pipelines execute multiple read-only steps with cross-step references:

```toml
[[operations]]
name = "projectDashboard"
type = "pipeline"
modelName = "_pipeline"
access = "isMemberOf('team', database.metadata.teamId)"
[operations.definition]
return = "all"

[[operations.definition.steps]]
name = "recentTasks"
type = "query"
modelName = "tasks"
filter = { projectId = "$params.projectId" }
sort = { createdAt = -1 }
limit = 5

[[operations.definition.steps]]
name = "statusBreakdown"
type = "aggregate"
modelName = "tasks"
filter = { projectId = "$params.projectId" }
groupBy = ["status"]
operations = [{ type = "count", outputField = "total" }]

[[operations.definition.steps]]
name = "openBugs"
type = "count"
modelName = "tasks"
filter = { projectId = "$params.projectId", label = "bug", status = "open" }

[[operations.params]]
name = "projectId"
type = "string"
required = true
```

**Pipeline step references** — only these accessors exist (no `$steps.X.result`):

| Variable | Valid for step type | Resolves to |
|----------|---------------------|-------------|
| `$steps.X.first` | query | First record (object) or `null` |
| `$steps.X.first.fieldName` | query | A field from the first record, or `null` |
| `$steps.X.all.fieldName` | query | Array of that field across all records (filters out undefined) |
| `$steps.X.count` | query / count / aggregate | Record count, count value, or # of aggregate group keys |
| `$steps.X.results` | query / count / aggregate | Full data array, count number, or aggregate object |

**Response shape** depends on `return`:

- `"return": "all"` (default) → `{ steps: { stepName: { type, data | count | result } } }`
- `"return": "<stepName>"` → just that step's payload at top level: `{ count }` for count, `{ result }` for aggregate, and for a query step the full paginated envelope `{ data, hasMore, nextCursor?, prevCursor? }` — identical to a bare query op, with caller `cursor`/`limit`/`direction` applied to the returned step (effective limit = min of step and caller limits). `return = "all"` and non-query returns take no pagination: caller cursor/limit are ignored there.

#### applyToQuery — server-side query+mutate

`applyToQuery` finds records matching a filter and applies a mutation action to every matched record in a single server-side request, eliminating the query-then-mutate round trip.

```toml
[[operations]]
name = "archiveCompleted"
type = "applyToQuery"
modelName = "tasks"
access = "hasRole('admin')"
[operations.definition.source]
filter = { status = "completed", updatedAt = { "$lt" = "$params.olderThan" } }
limit = 500

[operations.definition.action]
op = "patch"
data = { archived = true }

[[operations.params]]
name = "olderThan"
type = "string"
required = true
```

Supported action ops: `delete`, `patch` (requires `data`), `increment` (requires `fields`), `addToSet`/`removeFromSet` (require `stringSets`).

**Limits and partial-mutation safety:**

- When `source.limit` is **not set**, a default cap of 1000 records applies. If the matching set exceeds this cap, the request is **rejected with HTTP 400** to prevent silent partial mutation — narrow the filter or set `source.limit` explicitly.
- When `source.limit` **is set**, the action applies to the first `limit` records. If truncated, the response includes `truncated: true, appliedLimit: <limit>`.

**Safe to loop on `truncated: true`:** Only when the mutation causes matched records to no longer satisfy the filter (e.g., `delete`, or `patch` that changes a filtered field). Looping on `increment` or unrelated `patch` re-processes the same records indefinitely.

**Response:** `{ matched: 500, affected: 498, failed: 2 }`

Pass `?dryRun=true` to preview matched records without applying the action.

## Using Databases in App Code

The client library is used at runtime to create database instances, execute operations, and manage data.

### Creating database instances

```swift
  let db = try await client.databases.create(params: CreateDatabaseParams(
    title: "Alpha Project",
    databaseType: "project" // must match a configured database type
  ))
  // db: DatabaseInfo { databaseId, title, databaseType, celContext, permission, createdBy, ... }
```

`db.databaseType` is the configured type; `db.celContext` holds the per-database CEL context dict (`db.metadata` resolves to the same field).

### Listing and fetching databases

`list()` returns DBs where the user has a **direct** permission (owner or manager). Databases reachable only via CEL-gated operations or `DatabaseGroupPermission` are NOT returned by `list()` (use `groups.listDatabases` for group-shared ones). App admins see all DBs in the app. Passing `{ databaseType }` to `list()` is a post-join filter that narrows the set, never widens it.

```swift
  // Databases where the caller is owner or manager (admins see all).
  // Databases reachable only via CEL-gated operations or group grants are
  // NOT returned here — use groups.listDatabases for group-shared ones.
  let databases = try await client.databases.list()

  // Narrow to one databaseType (post-join filter — narrows, never widens).
  let projects = try await client.databases.list(databaseType: "project")

  let db = try await client.databases.get(databaseId: databaseId)

  _ = try await client.databases.update(
    databaseId: databaseId,
    params: UpdateDatabaseParams(title: "New Title")
  )

  // Owner only — permanently removes all records and permissions.
  _ = try await client.databases.delete(databaseId: databaseId)
```

### Executing operations

Callers can override `limit`, `cursor`, and `direction` at call time:

```swift
  let result = try await client.databases.executeOperation(
    databaseId: databaseId,
    name: "listTasks",
    options: ExecuteOperationOptions(
      params: ["projectId": "proj-1"],
      limit: 10,
      cursor: previousCursor,
      direction: .ascending // forward; use .descending to page backward
    )
  )
  // result: { data: [...records], hasMore, nextCursor? }
```

**Response shapes by operation type:**

| Operation Type | Response Shape |
|----------------|---------------|
| `query` | `{ data: [...records], hasMore: boolean, nextCursor?: string, prevCursor?: string }` |
| `mutation` | `{ results: [{ op, success, id, values? }] }` — one entry per definition step, in definition order; `values` on increments (entire request returns 409 if any sub-op fails) |
| `count` | `{ count: number }` |
| `aggregate` | `{ result: { [groupValue]: { ...computedFields } } }` |
| `pipeline` | When `return = "all"`: `{ steps: { [stepName]: { type, data \| count \| result } } }`. When `return = "<step>"`: that step's payload at top level — a returned query step paginates like a bare query (`hasMore`/`nextCursor`/`prevCursor`, caller cursor/limit honored) |
| `applyToQuery` | `{ matched, affected, failed, truncated?, appliedLimit?, warning? }` (or `{matched, affected: 0, failed: 0, dryRun: true, sample: [...]}` when called with `?dryRun=true`) |

**Operation timing:** Pass `timing: true` to get per-phase millisecond timings in `result._timing`. Works on all operation types.

```swift
  let result = try await client.databases.executeOperation(
    databaseId: databaseId,
    name: "listTasks",
    options: ExecuteOperationOptions(params: ["projectId": "proj-1"], timing: true)
  )
  // result._timing: { totalMs, databaseLookup, operationLookup, celEvaluation, ... }
```

### Bulk operation calls (`executeBatch`)

`executeBatch` invokes a single registered **mutation** operation many times in one HTTP request (up to 100,000 items). Each item is a `{ params }` object — the operation runs once per item, with its `op.access` and per-parameter `access` rules **re-evaluated against each item's params**.

```swift
  let result = try await client.databases.executeBatch(
    databaseId: databaseId,
    operationName: "import-contacts",
    batch: [
      DatabaseBatchOperation(params: ["name": "Alice", "email": "alice@example.com"]),
      DatabaseBatchOperation(params: ["name": "Bob", "email": "bob@example.com"]),
    ]
  )
  // result: { imported, failed }
```

**Constraints:**

- Only `type = "mutation"` operations are accepted (400 otherwise).
- All items in the batch must satisfy the operation's `access` and per-param `access` rules. If **any** item fails authorization, the **whole batch is rejected with 403** before any writes happen.
- Param schema validation is also per-item; the first invalid item rejects the batch with 400.
- Save/patch/delete ops from successfully-resolved items are funneled through one `/batch` call (transactional at the storage layer); increment / addToSet / removeFromSet ops are dispatched in parallel afterwards.
- Response is `{ imported, failed }` — `failed` counts items whose individual write succeeded at the auth/validation stage but was rejected by the storage layer (e.g., `ifNotExists` conflict).

**Don't:** there is no per-item `itemAccess` field on registered operations. To restrict what params a caller can pass per item, put the rule on the operation's `access` (referencing `params.*`) or on individual `params.<name>.access` (referencing `value`) — both are re-evaluated for every batch item.

### Managing database CEL context

The CEL context stores per-database values that operations and triggers can reference via `$database.celContext.*` (`$database.metadata.*` resolves to the same values):

```swift
  _ = try await client.databases.updateCelContext(
    databaseId: databaseId,
    celContext: ["teamId": "team-alpha", "projectId": "proj-1"]
  )
```


Via CLI:

```bash
primitive databases cel-context update <database-id> --data '{"teamId":"team-alpha"}'
primitive databases cel-context get <database-id>
```

## Direct Record Operations

Direct record operations require owner or manager permission. For most apps, use registered operations instead.

`connect()` returns a `DoDb` handle. Save (upsert), patch (partial update), find a single record, delete, and count all run against it. A `save` is an upsert; pass `ifNotExists` for insert-only, `condition` for a conditional write, and `stringSets` to seed StringSet fields:

```swift
  let db = client.databases.connect(databaseId: databaseId)

  // Save (upsert) — data must carry an "id" (or use upsertOn).
  _ = try await db.save(modelName: "tasks", data: ["id": "task-1", "title": "Ship v1"])

  // Insert only — fails if the record already exists.
  _ = try await db.save(
    modelName: "tasks", data: ["id": "task-3", "title": "Draft"],
    options: DoDbSaveOptions(ifNotExists: true)
  )

  // Conditional write — only proceeds if the existing record matches.
  _ = try await db.save(
    modelName: "tasks", data: ["id": "task-3", "title": "Draft"],
    options: DoDbSaveOptions(condition: ["completed": false])
  )

  // Seed StringSet fields on the write.
  _ = try await db.save(
    modelName: "tasks", data: ["id": "task-4", "title": "Tagged"],
    options: DoDbSaveOptions(stringSets: ["tags": ["featured", "sale"]])
  )

  // Patch (partial update) — only the provided fields change.
  _ = try await db.patch(modelName: "tasks", id: "task-1", data: ["priority": 5])
  _ = try await db.patch(
    modelName: "tasks", id: "task-1", data: ["priority": 7],
    options: DoDbPatchOptions(condition: ["completed": false])
  )

  // Find a single record by id (or nil).
  let task = try await db.find(modelName: "tasks", id: "task-1")

  // Delete — true if a record existed.
  let deleted = try await db.delete(modelName: "tasks", id: "task-2")

  // Count, optionally filtered.
  let total = try await db.count(modelName: "tasks")
  let open = try await db.count(modelName: "tasks", filter: ["completed": false])
```

### Query

The handle queries with the full filter operator grammar, sort + cursor pagination, projection, and related-data `include`:

```swift
  // Filter operators: exact match, comparisons, set membership, text, $or/$and.
  let result = try await db.query(modelName: "tasks", filter: [
    "completed": false,
    "priority": ["$gte": 5],
    "category": ["$in": ["work", "urgent"]],
    "title": ["$startsWith": "Ship"],
    "tags": ["$contains": "featured"],
    "$or": [["category": "work"], ["priority": ["$gte": 8]]],
  ])

  // Multiple fields on the same object are ANDed; use explicit $and to put
  // two conditions on one field.
  let priced = try await db.query(modelName: "tasks", filter: [
    "$and": [["priority": ["$gte": 1]], ["priority": ["$lte": 5]]],
  ])

  // Sort, limit, and cursor pagination.
  let page1 = try await db.query(
    modelName: "tasks", filter: [:],
    options: DoDbQueryOptions(sort: DoDbSort("priority", .descending), limit: 20)
  )
  var page2: DoDbQueryResult?
  if page1.hasMore {
    page2 = try await db.query(
      modelName: "tasks", filter: [:],
      options: DoDbQueryOptions(sort: DoDbSort("priority", .descending), limit: 20, uniqueStartKey: page1.nextCursor)
    )
  }

  // Projection — return only the named fields.
  let titles = try await db.query(
    modelName: "tasks", filter: [:],
    options: DoDbQueryOptions(projection: DoDbProjection([("title", .include), ("completed", .include)]))
  )

  // Include related records — they land under _related on each parent.
  let withCategory = try await db.query(
    modelName: "tasks", filter: [:],
    options: DoDbQueryOptions(include: [
      DoDbInclude(model: "categories", type: .refersTo, sourceField: "category", alias: "categoryInfo")
    ])
  )
```

### Filter operators

| Operator | Description | Example |
|----------|-------------|---------|
| *(exact match)* | Equals a value (shorthand for `$eq`) | `{ status: "active" }` |
| `$eq` | Explicit equality | `{ status: { $eq: "active" } }` |
| `$ne` | Not equal | `{ status: { $ne: "archived" } }` |
| `$gt` | Greater than | `{ price: { $gt: 10 } }` |
| `$gte` | Greater than or equal | `{ price: { $gte: 10 } }` |
| `$lt` | Less than | `{ price: { $lt: 50 } }` |
| `$lte` | Less than or equal | `{ price: { $lte: 50 } }` |
| `$in` | Matches any value in array | `{ category: { $in: ["books", "music"] } }` |
| `$nin` | Matches none of the values in array | `{ status: { $nin: ["archived", "deleted"] } }` |
| `$startsWith` | String prefix match | `{ name: { $startsWith: "Pro" } }` |
| `$endsWith` | String suffix match | `{ filename: { $endsWith: ".pdf" } }` |
| `$containsText` | Full-text search | `{ name: { $containsText: "deluxe" } }` |
| `$exists` | Field exists (true) or is null/missing (false) | `{ avatar: { $exists: true } }` |
| `$contains` | StringSet contains a value | `{ tags: { $contains: "featured" } }` |
| `$all` | StringSet contains all values | `{ tags: { $all: ["featured", "sale"] } }` |
| `$size` | StringSet element count (number or comparison) | `{ tags: { $size: 3 } }` or `{ tags: { $size: { $gte: 1 } } }` |
| `$or` | Matches any of the conditions | `{ $or: [{ status: "active" }, { priority: { $gte: 5 } }] }` |
| `$and` | Matches all conditions (useful when multiple conditions target the same field) | `{ $and: [{ price: { $gte: 10 } }, { price: { $lte: 50 } }] }` |
| `$inGroup` | Matches any current member of a group — expanded server-side into `$in` | `{ assigneeId: { $inGroup: { type: "team", id: "$params.teamId" } } }` |

`$startsWith`, `$endsWith`, and `$containsText` are mutually exclusive on the same field — only one substring operator per field per query.

Multiple filters on different fields are implicitly combined with AND. Use an explicit `$and` only when you need two conditions on the *same* field (e.g. a range), and combine `$or` with other top-level fields freely.

#### `$inGroup` group-membership filter

`$inGroup` is available on any field in a `query`, `count`, or `aggregate` filter (including inside `$and`/`$or` and pipeline read steps) — it composes into the existing filter structure rather than introducing a new grouping mechanism:

```
{ <field>: { $inGroup: { type: "<groupType>", id: "<group-id-or-substitution>" } } }
```

- The server expands `$inGroup` into `{ $in: [...memberIds] }` once per query, before it reaches the database.
- Expansion is gated by the target group's `member.list` CEL rule — a caller who isn't allowed to list the group's members gets `403 GROUP_EXPANSION_FORBIDDEN`.
- A missing or empty group expands to `$in: []` (matches nothing), never match-all.
- If the same field also carries a caller-supplied `$in`, the result is the **intersection** — group membership AND the caller's list — not a replacement.
- An invalid spec (missing or empty `type`/`id`) is rejected with `400 INVALID_IN_GROUP`.
- Expansion is capped at 5,000 members (server-configurable); exceeding it fails with `400 GROUP_FILTER_TOO_LARGE`.

Include types: `refersTo` (FK to one record), `hasMany` (target FK to this record), `refersToMany` (StringSet to multiple records). The loaded records land under `_related` on each parent (e.g. `result.data[0]._related.customer`).

## Atomic Operations

Increment/decrement numeric fields (returns the post-increment values) and add/remove StringSet members atomically — no read-modify-write round trip:

```swift
  // Increment/decrement numeric fields; returns the post-increment values.
  let newValues = try await db.increment(
    modelName: "tasks", id: "task-1",
    fields: ["priority": 1, "estimatedHours": -2]
  )

  // Atomically add/remove StringSet members.
  try await db.addToSet(modelName: "tasks", id: "task-1", sets: ["tags": ["featured"]])
  try await db.removeFromSet(modelName: "tasks", id: "task-1", sets: ["tags": ["sale"]])
```

## Batch Writes (direct)

`db.batch` executes multiple writes atomically at the storage layer. It returns one `BatchOperationResult` per input op (`{success, id, error?, values?}` — `values` for increment ops); a per-item failure does NOT throw, so check `.success` on each result.

```swift
  let results = try await db.batch([
    DoDbBatchOperation(op: .save, modelName: "tasks", id: "t-1", data: ["title": "A"]),
    DoDbBatchOperation(op: .patch, modelName: "tasks", id: "t-2", data: ["completed": true]),
    DoDbBatchOperation(op: .delete, modelName: "tasks", id: "t-3"),
    DoDbBatchOperation(op: .increment, modelName: "tasks", id: "t-4", fields: ["priority": 1]),
    DoDbBatchOperation(op: .addToSet, modelName: "tasks", id: "t-5", stringSets: ["tags": ["urgent"]]),
  ])

  for r in results where !r.success {
    print("op failed:", r.id, r.error ?? "")
  }
```

### Aggregation (direct)

Group by one or more fields and compute `count`/`sum`/`avg`/`min`/`max`, with an optional filter, sort, and limit:

```swift
  let result = try await db.aggregate(modelName: "tasks", options: DoDbAggregationOptions(
    groupBy: [.field("category")],
    operations: [
      DoDbAggregationOperation(type: .count),
      DoDbAggregationOperation(type: .sum, field: "estimatedHours"),
      DoDbAggregationOperation(type: .avg, field: "estimatedHours"),
      DoDbAggregationOperation(type: .min, field: "priority"),
      DoDbAggregationOperation(type: .max, field: "priority"),
    ],
    filter: ["completed": false],
    limit: 10,
    sort: DoDbAggregationSort(field: "count", direction: .descending)
  ))
```


Database field types: `id` (primary key, use `auto_assign = true` for ULIDs), `string`, `number`, `boolean`, `date`, `stringset` (set-of-strings field — use with `addToSet`/`removeFromSet`). There is no object/JSON field type — store structured payloads as JSON strings and parse on read.

### Indexes

Add an index to every field you filter or sort on; without one the engine scans all records of that model. Register them manually, declare composite unique constraints, list and drop them, or sync a model's declared index state at once:

```swift
  // Sync the desired index state for one or more models (additive — the
  // server registers only what's missing).
  _ = try await db.syncIndexes(models: [
    DoDbModelSyncState(
      modelName: "tasks",
      indexes: [
        DoDbModelSyncState.Index(fieldName: "category"),
        DoDbModelSyncState.Index(fieldName: "priority", fieldType: "number"),
      ]
    )
  ])

  // Or register indexes manually.
  try await db.registerIndex(modelName: "tasks", fieldName: "category", fieldType: "string")
  try await db.registerIndex(modelName: "tasks", fieldName: "priority", fieldType: "number")
  try await db.registerIndex(modelName: "appUsers", fieldName: "email", fieldType: "string", unique: true)

  // Composite unique constraint across multiple fields.
  try await db.registerUniqueConstraint(modelName: "categories", constraintName: "name_parentId", fields: ["name", "parentId"])

  // List and drop.
  let indexes = try await db.listIndexes(modelName: "tasks")
  try await db.dropIndex(modelName: "tasks", fieldName: "category")

  let constraints = try await db.listUniqueConstraints(modelName: "categories")
  try await db.dropUniqueConstraint(modelName: "categories", constraintName: "name_parentId")
```

## Permissions

### How end users access databases

**Use registered operations with CEL access expressions** to control what end users can do. Do not use `addManager` to give application users access to a database — that grants administrative control over the database itself, not scoped data access.

The correct pattern:
1. Define operations with appropriate `access` CEL expressions (e.g., `isMemberOf('team', database.celContext.teamId)`)
2. Any authenticated user who satisfies the CEL expression can call the operation — no permission grant needed
3. If no operations are registered, non-owner/manager users are denied access entirely

### Owner and manager roles (administrative)

Owner and manager permissions control **who can manage the database itself** — not who can use the app's data. These are administrative roles, not end-user roles.

| Level | Grant/revoke perms | Update title | Update metadata | Delete DB | Bypass CEL gate |
|-------|-------------------|-------------|-----------------|-----------|-----------------|
| `owner` | Yes | Yes | Yes | Yes | Yes |
| `manager` | No (list only) | Yes | Yes | No | Yes |

The database creator is automatically the `owner`. Console admins bypass all checks.

These calls are for administrative access — not for end-user data access:

```swift
  _ = try await client.databases.addManager(
    databaseId: databaseId,
    params: AddManagerParams(userId: coAdminUserId)
  )

  let permissions = try await client.databases.listPermissions(databaseId: databaseId)
  // [DatabasePermissionEntry { databaseId, userId, permission, grantedAt, grantedBy, userName?, userEmail? }]

  _ = try await client.databases.removeManager(databaseId: databaseId, userId: coAdminUserId)

  _ = try await client.databases.transferOwnership(databaseId: databaseId, newOwnerId: newOwnerId)
```

**Don't add end users as managers.** `manager` bypasses every CEL operation gate. The right pattern is: a small fixed set of admin accounts hold owner/manager; everyone else goes through registered operations with `access` rules.

### Group-Based Database Access

`DatabaseGroupPermission` grants the same admin-level access (`manager`) to every member of a group at once. Membership changes propagate automatically. Only `"manager"` is supported.

```swift
  _ = try await client.databases.grantGroupPermission(
    databaseId: databaseId,
    params: GrantDatabaseGroupPermissionParams(
      groupType: "team", groupId: "engineering", permission: "manager"
    )
  )

  let groupPerms = try await client.databases.listGroupPermissions(databaseId: databaseId)
  _ = try await client.databases.revokeGroupPermission(
    databaseId: databaseId, groupType: "team", groupId: "engineering"
  )

  // Group members discover their group-accessible databases via this
  // (databases.list() does NOT return group-shared databases).
  let dbs = try await client.groups.listDatabases(groupType: "team", groupId: "engineering")
```

| Call | Resolves group access? |
|------|------------------------|
| `databases.get(id)` | Yes |
| `databases.list()` | **No** — direct grants only |
| `groups.listDatabases(type, id)` | Yes — canonical for group members |

For a unified "all DBs I can access" view, combine `databases.list()` with `groups.listDatabases(...)` for each membership.

**Group permissions are still admin-level access — they don't replace operation-level CEL.** Only the database owner (or an app admin) can grant/revoke group permissions.

## Real-Time Subscriptions

Databases push changes to connected clients over WebSocket. A subscription answers "how does the client find out something changed?" — the client registers interest in a named subscription, and the server pushes batched record changes as they happen. Subscriptions are **type-scoped** — one definition per `(databaseType, subscriptionKey)` applies to every database of that type. The model key is `(appId, databaseType, subscriptionKey)`. Max 20 subscriptions per database type.

### Mental model

- Subscriptions deliver **deltas only**. Always pair a subscription with an initial `executeOperation` load (see [Canonical Pattern: Load + Subscribe](#canonical-pattern-load--subscribe)).
- The **writer's connection** is excluded from fanout, not the writer's **user**. Another connection of the same user (e.g. another tab) still receives the change. Use `isOrigin` / `isOriginUser` to suppress optimistic echoes.
- Subscriptions require both `access` and `filter` CEL. `access` is checked once at subscribe time with full user/membership context; `filter` runs per change with a narrow context (no memberships, no `database.*`).
- There is **no replay on reconnect** — the client auto re-issues the subscription, but changes missed while disconnected are gone. Re-load if you need consistency.
- `subscribe()` returns an `unsub()` function — there is **no** event-emitter API (`.on()` / `.unsubscribe()`). Call `unsub()` on teardown or you leak `ConnectionMapping` rows and dead callbacks.

### Registering a subscription

Subscriptions can be managed via TOML config files with `primitive sync push` (recommended) or via the admin HTTP API at `/databases/types/<databaseType>/subscriptions`.

**Via TOML (recommended)** — add `[[subscriptions]]` blocks to your database type config file:

```toml
# config/database-types/support-desk.toml
[type]
databaseType = "support-desk"

[[subscriptions]]
subscriptionKey = "my-open-tickets"
displayName = "My open tickets"
modelName = "ticket"
accessRule = "user.userId != ''"
filter = "record.data.assigneeId == user.userId && record.data.status == 'open'"
select = ["id", "title", "priority", "updatedAt"]
emit = ["enter", "update", "leave"]

[[subscriptions]]
subscriptionKey = "tickets-by-team"
displayName = "Tickets by team"
modelName = "ticket"
accessRule = "isMemberOf('team', params.teamId)"
filter = "record.data.teamId == params.teamId"
[subscriptions.params]
teamId = { type = "string", required = true }
```

`primitive sync push` creates new subscriptions, updates changed ones, and deletes keys present on the server but missing from the TOML. `primitive sync pull` round-trips subscriptions back into `[[subscriptions]]` blocks.

**Via admin HTTP API** — POST/PUT/DELETE directly against `/databases/types/<databaseType>/subscriptions` from a server-side client that holds admin permission:

```swift
  _ = try await adminClient.makeRequest(
    "POST",
    "/databases/types/support-desk/subscriptions",
    [
      "subscriptionKey": "my-open-tickets",
      "displayName": "My open tickets",
      "modelName": "ticket",
      "access": "user.userId != ''",
      "filter": "record.data.assigneeId == user.userId && record.data.status == 'open'",
      "select": ["id", "title", "priority", "updatedAt"],
      "emit": ["enter", "update", "leave"],
    ]
  )
```

Endpoints:

- `GET /databases/types/<databaseType>/subscriptions` — list active subscriptions for the type.
- `POST /databases/types/<databaseType>/subscriptions` — create. Returns 409 if the `subscriptionKey` collides with an existing (or archived) subscription — use a different key or hard-delete first.
- `GET /databases/types/<databaseType>/subscriptions/<subscriptionKey>` — read one.
- `PUT /databases/types/<databaseType>/subscriptions/<subscriptionKey>` — update (`filter`, `access`, `select`, `emit`, `params` are all patchable).
- `DELETE /databases/types/<databaseType>/subscriptions/<subscriptionKey>` — hard-delete.

Field reference:

| Field | TOML key | HTTP key | Required | Notes |
|-------|----------|----------|----------|-------|
| Subscription key | `subscriptionKey` | `subscriptionKey` | Yes | Per-`(app, databaseType)` unique. Clients reference it on `subscribe()`. Must NOT contain `#`. |
| Display name | `displayName` | `displayName` | Yes | Human label. |
| Model name | `modelName` | `modelName` | Yes | Scopes the subscription to one model. |
| Access rule | `accessRule` | `access` | Yes | CEL — evaluated once at subscribe time. Non-empty string required. |
| Filter | `filter` | `filter` | Yes | CEL — evaluated per change. Non-empty string required (use `"true"` for "match every change `access` allowed"). **Cannot reference `database.*`** — rejected with HTTP 400 at save time. |
| Select | `select` | `select` | No | Array of field names to project `data` / `previousData` to before broadcast, **server-side**. Fields not listed never leave the server — use it to keep sensitive fields off the wire. |
| Emit | `emit` | `emit` | No | Array restricting delivered `changeType` values: `"enter"`, `"update"`, `"leave"`. Omit for all. |
| Params | `[subscriptions.params]` | `params` | No | `{ <name>: { type: "string" \| "number" \| "boolean", required?: boolean } }`. Strict type checks (no coercion). Max 5. Bound at subscribe time, exposed as `params.*` in `access` and `filter`. |

Note the wire-format difference: the TOML field is `accessRule`; the HTTP API field is `access`. POSTing `accessRule` to the HTTP API is rejected with `access (CEL expression) is required`.

### CEL context (access vs. filter)

The two phases see different contexts:

`access` (subscribe time — full context, evaluated once):

| Variable | Notes |
|----------|-------|
| `user.userId`, `user.role` | `role` is the caller's app role. |
| `database.id` | The database instance ID. |
| `database.celContext.*` | The database's CEL context (also `database.metadata.*`). |
| `isMemberOf(type, id)`, `memberGroups(type)`, `hasRole(role)` | Membership/role helpers. |
| `params.*` | Subscriber-supplied params. |

`filter` (per-change broadcast — narrow context, NO memberships, NO `database.*`):

| Variable | Notes |
|----------|-------|
| `user.userId` | The subscriber's user id. |
| `user.role` | **Always `""` empty string in filter context.** Don't rely on it. |
| `record.modelName`, `record.op`, `record.id` | Top-level change metadata. |
| `record.data.<fieldName>` | Post-write record fields. `null` on `delete`. |
| `record.previousData.<fieldName>` | Pre-write row, when available (patch/delete). `null` on a fresh insert. |
| `params.*` | Bound params from `subscribe()`. |

`filter` cannot grant access that `access` denies — they're ANDed. Put group-based and database-context-based authorization in `access`, not `filter`. In `filter`, record payload fields live under `record.data.*` — `record.<fieldName>` does NOT work (only `record.modelName`, `record.op`, `record.id` are top-level).

### Subscribing from the client

`subscribe()` returns an `unsub()` function (synchronously). There is no event-emitter API.

```swift
  let unsub = try client.databases.subscribe(
    databaseId: databaseId,
    subscriptionKey: "my-open-tickets",
    options: DatabaseSubscribeOptions(onChange: { event in
      // event.originConnectionId, event.originUserId, event.isOrigin, event.isOriginUser
      if event.isOrigin {
        // This same tab wrote it — the UI is already updated optimistically.
        return
      }
      for change in event.changes {
        // change.changeType: "enter" | "update" | "leave"
        // change.op decides the shape of change.data (all subject to `select`):
        //   "save"  → the full row                     "delete" → nil
        //   "patch" → the patched fields' new values
        //   "increment" / "addToSet" / "removeFromSet" → the op input
        //     (amounts / values added or removed), NOT the resulting values —
        //     derive those from previousData (the pre-write row).
        if change.op == "delete" {
          removeTicket(change.id)
        } else if change.op == "save" {
          replaceTicket(change.id, change.data)
        } else if change.op == "patch" {
          // Merge — assigning change.data would blank every untouched field.
          mergeTicket(change.id, change.data as? [String: Any] ?? [:])
        } else {
          // increment / addToSet / removeFromSet: compute each field's new value.
          let prev = change.previousData as? [String: Any] ?? [:]
          var resolved: [String: Any] = [:]
          for (field, input) in change.data as? [String: Any] ?? [:] {
            if change.op == "increment" {
              resolved[field] = (prev[field] as? Double ?? 0) + (input as? Double ?? 0)
            } else {
              let current = prev[field] as? [String] ?? []
              let values = input as? [String] ?? []
              resolved[field] = change.op == "addToSet"
                ? current + values.filter { !current.contains($0) }
                : current.filter { !values.contains($0) }
            }
          }
          mergeTicket(change.id, resolved)
        }
      }
    })
  )

  // Return the teardown handle — call it on unmount to stop receiving deltas.
  return unsub
```

Parameterized:

```swift
  let unsub = try client.databases.subscribe(
    databaseId: databaseId,
    subscriptionKey: "tickets-by-team",
    options: DatabaseSubscribeOptions(
      params: ["teamId": "eng"],
      onChange: { event in
        for change in event.changes { handleChange(change) }
      }
    )
  )
```

The client auto-reissues `db.subscribe` on WebSocket reconnect — no app code needed.

#### Wrong

```swift
// WRONG — there is no event-emitter. `subscribe` takes an `onChange` closure
// and returns an unsub function (`() -> Void`) synchronously; the returned
// handle has no `.on(...)` / `.unsubscribe()`.
let unsub = try client.databases.subscribe(databaseId: databaseId, subscriptionKey: "my-open-tickets", options: opts)
unsub.on("change", handler)   // ✗ unsub is a function, not an emitter

// WRONG — onChange receives a DatabaseChangePayload envelope whose `changes`
// is an array, not a single record. Iterate `event.changes`.
DatabaseSubscribeOptions(onChange: { event in render(event) })

// WRONG — in the subscription's filter CEL, record payload fields are nested
// under `record.data`, not spread on `record`.
filter: "record.assigneeId == user.userId"
// CORRECT:
filter: "record.data.assigneeId == user.userId"
```

### Change envelope shape

The `onChange` closure receives a `DatabaseChangePayload`; each element of its `changes` array is a `DatabaseChangeEvent`. `op` and `changeType` are plain `String`s; `data` / `previousData` are opaque record blobs typed `Any?`.

```swift
public struct DatabaseChangePayload {
    public let databaseId: String
    public let subscriptionKey: String
    public let changes: [DatabaseChangeEvent]   // 1+ changes; batched per write op
    public let timestamp: String                // ISO
    /// Writer's connection id, or nil for server-side writes (cron, workflow, admin).
    public let originConnectionId: String?
    /// Writer's user id, or nil for server-side writes.
    public let originUserId: String?
    /// True iff this exact connection produced the write. Synthesized per recipient.
    public let isOrigin: Bool
    /// True iff any session signed in as the current user produced the write.
    public let isOriginUser: Bool
}

public struct DatabaseChangeEvent {
    /// "save" | "patch" | "delete" | "increment" | "addToSet" | "removeFromSet".
    public let op: String
    /// Filter-set transition: "enter" (newly matching), "update" (in-set change),
    /// "leave" (no longer matches). nil on older server frames.
    public let changeType: String?
    public let modelName: String
    public let id: String
    public let data: Any?          // save → the full row as persisted (server-applied fields win); patch → the patched fields' new values
                                   // (both plus any `timestamps`-stamped fields at their persisted values);
                                   // increment/addToSet/removeFromSet → the op INPUT (amounts / values
                                   // added-or-removed), not the resulting values; nil on delete.
                                   // Subject to `select`.
    public let previousData: Any?  // the pre-write row (nil for a fresh insert); the deleted row
                                   // on delete. Subject to `select`.
}
```

**The shape of `data` follows `op`, not `changeType`.** A `save` delivers the full row; a `patch` delivers **the patched fields only, at their new values**; `increment` delivers the per-field increment **amounts** and `addToSet`/`removeFromSet` deliver the values **added or removed** per field — the op *input*, not the resulting values; `delete` delivers no `data`. Server-applied fields — `timestamps` stamps, `autoPopulatedFields`, and trigger `set` values — ride along in `data` on `save` and `patch` at their persisted server values (subject to `select`), and a server-applied value wins over the caller's submission for the same field, so a trigger that overrides a submitted value delivers the trigger's value. They are visible to the subscription `filter` too, so a filter can match on a trigger-computed field. A patch's `data` is the patched fields plus those server-applied fields; `increment` and the set ops don't stamp, so their frames carry none. The pre-write row rides in `previousData` on all of them. An `applyToQuery` operation arrives as one `patch`/`increment`/set-op/`delete` change per matched record — there is no `applyToQuery` op on the wire. Consequences:

- An `enter` transition does NOT imply a full row: a patch that brings a row into the filter set is `changeType: "enter"` with only the changed fields in `data`.
- **Merge, don't assign.** Replacing a cached row with `change.data` on a non-`save` op silently blanks every field the write didn't touch. Key the cache on `change.id`; replace on `save`, merge the fields present in `data` on `patch`, remove on `delete`.
- **For `increment` and the set ops, derive — don't merge.** Merging `change.data` writes the delta over the value (an `increment` of `{views: 1}` would set `views` to `1`; a `removeFromSet` would set the field to the values just removed). Compute the new value from the base row plus the input: add the amount, union the added values, subtract the removed ones (the pattern the subscribe example above and the load-then-subscribe example below follow).
- A field the write didn't include (a grouping key, a display label) is readable from `previousData` — both fields pass through the `select` projection, so anything `select` excludes is in neither.

### Change-frame origin attribution

Every `db.change` frame carries who produced the write so subscribers can suppress their own optimistic echoes:

| Field | Meaning |
|---|---|
| `originConnectionId` | The writer's WebSocket connection id, or `null` for server-side writes (cron, workflow steps, admin imports, HTTP writes without `X-JB-Connection-Id`) |
| `originUserId` | The writer's user id, or `null` for server-side writes |
| `isOrigin` | Synthesized per-recipient: `true` iff `originConnectionId` matches this client's current WS connection id |
| `isOriginUser` | Synthesized per-recipient: `true` iff any tab/process signed in as the receiver wrote it. Differs from `isOrigin` across tabs of the same user — useful for cache invalidation that only cares about user identity. |

On WS reconnect the local connection id rotates, so a frame for the writer's own pre-reconnect write may arrive with `isOrigin: false`. That's expected and harmless because the writer-exclusion server-side fanout still drops the frame from the writing connection itself. `isOriginUser` is the stable cross-tab signal — use it for cache invalidation that doesn't care which tab made the write.

### Reconnect & cleanup behavior

- The WS client persists the registry of `(databaseId, subscriptionKey, params, onChange)` tuples and re-issues `db.subscribe` automatically when the socket reopens. No app code needed.
- **No replay** of changes that occurred while disconnected. If you need consistency on reconnect, re-run your initial-load query.
- The writer's connection is excluded from broadcast via `excludeConnectionId`. The writer's user is NOT excluded — other tabs/devices of the same user receive the change.
- Auth refresh that does NOT require a hard reconnect leaves subscriptions intact. A hard reconnect re-runs the registry pass (so `access` is re-evaluated against the current user/memberships).
- **Workflow mutations fan out too.** A `database.mutate` step in a workflow wakes up every matching subscription. Workflow (and cron-spawned) writes arrive with `originConnectionId: null` / `originUserId: null` and both `isOrigin` flags `false` — the subscription doesn't know or care that the write came from a server-side automation.

### Canonical Pattern: Load + Subscribe

```swift
  // Loaded rows arrive as JSONValue, subscription deltas as [String: Any] —
  // normalize to one representation so deltas can merge into loaded rows.
  let rowDictionary: (JSONValue) -> [String: Any] = { value in
    guard let data = try? JSONEncoder().encode(value) else { return [:] }
    return (try? JSONSerialization.jsonObject(with: data)) as? [String: Any] ?? [:]
  }

  // 1. Initial load — full current state.
  let loaded = try await client.databases.executeOperation(
    databaseId: databaseId, name: "list-my-open-tickets"
  )
  var byId: [String: [String: Any]] = [:]
  for ticket in loaded["data"]?.arrayValue ?? [] {
    if let id = ticket["id"]?.stringValue { byId[id] = rowDictionary(ticket) }
  }
  render(Array(byId.values))

  // 2. Subscribe for delta updates.
  let unsub = try client.databases.subscribe(
    databaseId: databaseId,
    subscriptionKey: "my-open-tickets",
    options: DatabaseSubscribeOptions(onChange: { event in
      for change in event.changes {
        if change.op == "delete" {
          byId.removeValue(forKey: change.id)
          continue
        }
        if change.op == "save" {
          // save delivers the full row.
          byId[change.id] = change.data as? [String: Any] ?? [:]
          continue
        }
        // On a cache miss (the write brought the row into the filter set),
        // the pre-write row in previousData supplies the base.
        var row = byId[change.id] ?? change.previousData as? [String: Any] ?? [:]
        for (field, input) in change.data as? [String: Any] ?? [:] {
          if change.op == "patch" {
            // patch delivers the patched fields' new values — merge them;
            // assigning change.data would blank the rest.
            row[field] = input
          } else if change.op == "increment" {
            // increment delivers the amounts, not the results — add them.
            row[field] = (row[field] as? Double ?? 0) + (input as? Double ?? 0)
          } else {
            // addToSet / removeFromSet deliver the values added or removed —
            // union or subtract against the current set.
            let current = row[field] as? [String] ?? []
            let values = input as? [String] ?? []
            row[field] = change.op == "addToSet"
              ? current + values.filter { !current.contains($0) }
              : current.filter { !values.contains($0) }
          }
        }
        byId[change.id] = row
      }
      render(Array(byId.values))
    })
  )

  // 3. Return teardown — call this on unmount.
  return unsub
```

Make the initial-load operation's filter and the subscription's `filter` semantically equivalent. If they diverge, the UI will flicker (records the operation returned but the subscription never updates, or vice versa).

There is no built-in "reconnected" callback. To re-run the initial load after a disconnect, observe client status — `client.events.on(.status) { (event: StatusChangedEvent) in if event.status == .connected { ... } }` — and re-fire your loader when the socket reconnects. `BaoDataLoader` does exactly this for you: it reloads when the connection flips back to `.connected`.

### Critical Rules

1. **Always pair initial load with subscription.** Subscriptions deliver deltas only — fetch initial state with a regular `executeOperation` call and keep the query's filter semantically equivalent to the subscription's `filter`.
2. **Subscriptions require both `access` and `filter`** — non-empty CEL strings. `access` runs once at subscribe time with full context; `filter` runs per change with no memberships and no `database.*`. Use `"true"` for `filter` if `access` is the only narrowing you need.
3. **In `filter`, record fields live under `record.data.*`** — not `record.<fieldName>`. Only `record.modelName`, `record.op`, `record.id` are top-level.
4. **Writer's connection is excluded server-side, not the writer's user.** Another connection of the same user (e.g. another tab) still receives the change. Use `isOrigin` / `isOriginUser` to suppress optimistic echoes.
5. **No replay on reconnect.** Re-query on reconnect; the server does not buffer missed changes.
6. **`unsub()` leaks if not called.** Each `subscribe()` returns an `unsub` function. Call it on view teardown or you accumulate `ConnectionMapping` rows and dead callbacks.
7. **Workflow mutations fan out too.** A `database.mutate` step wakes up every matching subscription. Workflow/cron writes arrive with `originConnectionId: null` / `originUserId: null` and both `isOrigin` flags `false`.
8. **`select` is privacy-affecting.** Fields not listed never leave the server. Use it instead of trying to scrub fields client-side.

### Anti-patterns

- Polling a database on an interval to find changes. Use a subscription.
- Putting `isMemberOf()` or per-user membership logic in subscription `filter` — `filter` has no membership data. Put it in `access`.
- Writing `record.fieldName` in `filter` — record payload fields live under `record.data.fieldName`.
- Sending an empty / missing `filter` on POST or PUT — both `access` and `filter` are required non-empty CEL strings. Use `"true"` if you don't want filter narrowing.
- Sending the body field `accessRule` to the HTTP API — the wire-format field name is `access`.
- Assuming the writer's own mutation comes back through THEIR subscription on the SAME connection — it doesn't. (Other connections of the same user DO receive it.)
- Relying on replay after disconnect. There is none. Re-load if you need consistency.
- Forgetting to call the returned `unsub()`. Leaks `ConnectionMapping` rows and registry entries.
- Subscribing on every render. Subscribe once per view, unsub on unmount.
- Creating >20 subscriptions per database type. Consolidate.

### Debugging subscriptions

- Subscribe call returns an error message (sent over WS as `type: "error"`, `context: "db.subscribe"`):
  - `"Database not found"` / `"Subscription not found"` — wrong id/key or archived.
  - `"Access denied to subscription"` — `access` returned false. Test the rule against the caller's user/memberships.
  - `"Missing required parameter: ..."` / `"Undeclared parameter: ..."` / `"Parameter ... must be of type ..."` — params don't match schema.
- Changes aren't arriving — verify (a) the write completed, (b) the subscription's `filter` matches the actual record (remember `record.data.field`, not `record.field`), (c) the subscriber's connection is still open.
- "Seeing my own writes" — only happens if a different connection performed the write (e.g. another tab) or you have a client-side optimistic update on the same path.

For driving subscriptions from a scheduled write (a cron-triggered workflow mutating a database), see the [Workflows agent guide](AGENT_GUIDE_TO_PRIMITIVE_WORKFLOWS.md#cron-triggers). The cron-spawned mutation is just a normal write; the subscription doesn't know cron exists.

## Schema Introspection

Databases are schemaless — the system tracks fields and inferred types as records are written.

List the models (collections) in a database via the raw records endpoint `GET /app/<appId>/api/databases/<databaseId>/records/models` (returns `{ models: ["contacts", "orders", "products"] }`):

```swift
  let result = try await client.makeRequest(
    "GET",
    "/databases/\(databaseId)/records/models"
  )
  let models = (result as? [String: Any])?["models"] as? [String]
  // models: ["contacts", "orders", "products"]
```

Describe a model's observed fields:

```swift
  let fields = try await client.databases.describe(databaseId: databaseId, modelName: "products")
  // [["model_name": "products", "field_name": "name", "inferred_type": "string", ...]]
```

Inferred types: `string`, `number`, `boolean`, `array`, `object`.

## CSV Import

For one-off, non-programmatic bulk loads, use the CLI path:

```bash
primitive databases import-csv <database-id> <file.csv> --model products \
  --column-map '{"Product Name":"name","Unit Price":"price"}' \
  --types '{"price":"number"}' --id-column SKU
```

The CLI imports through a registered batch (bulk) save operation — `--operation` defaults to `seed_save`, so the database type needs a save-like op (`{ modelName, id, data }`) by that name (or pass your own). `--batch-size` defaults to 5000, `--delimiter` to `,`; use `--dry-run` to report row/batch counts without writing and `--stop-on-error` to abort on the first failing chunk (default is best-effort continue).

For programmatic imports with per-row `transform` or progress callbacks, use `importCsv` from app code:

```swift
  let result = try await client.databases.importCsv(
    databaseId: databaseId,
    options: CsvImportOptions(
      csv: csvString, // provide csv or data (pre-parsed rows), not both
      modelName: "products",
      columnMap: ["Product Name": "name", "Unit Price": "price"],
      transform: { row, _ in
        guard let name = row["name"]?.stringValue, !name.isEmpty else {
          return nil // return nil to skip a row
        }
        var out = row
        out["importedAt"] = .string(ISO8601DateFormatter().string(from: Date()))
        return out
      },
      types: ["price": .number],
      idColumn: "sku",
      onProgress: { progress in
        print("\(progress.processed)/\(progress.total) processed")
      }
    )
  )
  // CsvImportResult: imported, failed, errors, indexesCreated, durationMs
```

Other options: `data` (pre-parsed rows, instead of `csv`), `delimiter`, `batchSize` (default 5000), `idGenerator` (per-row ID factory; the fallback chain is `idColumn` → `idGenerator` → generated ULID), `onBatchError` (return `false` to abort remaining batches), and `operationName` (the registered save op, default `"save"`, called as `{ modelName, id, data }`).

A generated model class can stand in for `modelName` — the import then filters columns to the model's schema and syncs its indexes afterwards (`syncIndexes`, reported as `indexesCreated`):

```swift
let result = try await client.databases.importCsv(
  databaseId: databaseId,
  options: CsvImportOptions(
    csv: csvString,
    model: Product.self,
    columnMap: ["Product Name": "name", "Unit Price": "price"]
  )
)
```

### Database design

- **Create multiple databases for isolation.** Each database is a separate isolated instance. Use separate databases for separate tenants, projects, or data domains to leverage per-database scaling.
- **Use database types** to share operation definitions and triggers across databases of the same kind.
- **Keep CEL context minimal — use groups for access control.** The CEL context is meant for a few identifying fields (e.g. a `teamId`) used as lookup keys in CEL — `isMemberOf('team', database.celContext.teamId)`. Group membership controls access; the CEL context just provides the key. Don't replicate data into metadata for field-by-field CEL checks — model that with groups. For runtime-toggleable settings, store them as records and read them in a pipeline (see [Settings record pattern](#settings-record-pattern)).
- **Use triggers** to enforce server-side invariants (created timestamps, audit fields) — don't trust client-provided values.

### Operations design

- **Keep operations focused.** Each operation should serve one purpose with a clear access rule.
- **Use parameterized filters**, not hardcoded values. Let callers pass IDs and filter criteria via `$params.*`.
- **Set restrictive access by default.** Start with specific CEL expressions and widen as needed.
- **Use projections** to limit response payloads — only return fields the client needs.
- **Use pipelines** for dashboard-style views that need data from multiple models in one round-trip. Pipelines are read-only — for "read-then-mutate" flows, use a pipeline to read, then call a separate mutation operation.
- **Default to server-assigned IDs in `save`.** When neither `opDef.id` nor `data.id` is provided, the server generates a ULID and returns it in `results[].id`. You may pass an explicit id via `opDef.id` or `data.id` (they must match if both are set, otherwise the request is rejected) — but only do that when you genuinely need a deterministic id (idempotency, foreign-key linkage to a precomputed key). Never pipe `$params.id` into save data unless that param is itself derived from a trusted source.

### Security

- **`$params.*` is caller-controlled. Never use it for authorization-relevant fields.** Substitution is literal — whatever the caller passes ends up in the resolved JSON.

  ```toml
  # WRONG — caller can claim any role:
  [[operations.definition.operations]]
  op = "save"

  [operations.definition.operations.data]
  authorRole = "$params.authorRole"
  title = "$params.title"

  # RIGHT — split into per-role operations, hardcode the role server-side, gate each with CEL:
  [[operations]]
  name = "createTeacherPost"
  type = "mutation"
  modelName = "posts"
  access = "isMemberOf('class-teachers', database.metadata.classId)"

  [[operations.definition.operations]]
  op = "save"

  [operations.definition.operations.data]
  authorRole = "teacher"
  title = "$params.title"
  authorId = "$user.userId"

  [[operations.params]]
  name = "title"
  type = "string"
  required = true
  ```

- **Don't use `addManager` to grant "access to data."** `manager` bypasses every operation-level CEL gate and grants administrative control. End-user data access goes through registered operations with CEL `access`; `manager` is reserved for the small set of accounts that should be able to update title, metadata, and (for `owner`) delete the database.
- **Per-parameter `access` is enforced on both `executeOperation` and `executeBatch`.** A param rule like `"access": "value == user.userId"` is re-evaluated for every batch item — that's the right place to put per-item authorization, NOT a (non-existent) `itemAccess` field.
- **Trigger CEL is fail-closed.** Any error during a `when` or `set` expression silently skips the trigger — so a malformed expression doesn't crash the write but also doesn't run. Test triggers explicitly.

For broader access-control patterns (group-based access, per-parameter access, rule sets), see the [Users and Groups guide](AGENT_GUIDE_TO_PRIMITIVE_USERS_AND_GROUPS.md).

### Performance

- **Register indexes** on every field used in `filter` or `sort`. Without an index, the database falls back to scanning all records of that model.
- **Cap with `limit`.** Definition `limit` clamps caller-supplied `limit` to its own value (effective limit = `min(definition.limit, callerLimit)`).
- **`count` ops are much cheaper than `query` for "how many".**
- **Use `executeBatch`** for client-side bulk writes (up to 100,000 items per call); use `applyToQuery` to mutate every record matching a server-side filter without shipping the IDs across the wire.

## Common Patterns

For team-based and group-based access patterns, see the [Users and Groups guide](AGENT_GUIDE_TO_PRIMITIVE_USERS_AND_GROUPS.md). For architecture-level patterns (when to use databases vs. documents), see the [Data Modeling guide](AGENT_GUIDE_TO_PRIMITIVE_DATA_MODELING.md).

### Settings record pattern

For mutable application settings (feature flags, visibility toggles, etc.), store them as a regular database record rather than in `database.metadata`. Use a pipeline to read the settings record and incorporate its values into subsequent query filters via `$steps.*`:

**In `config/database-types/classroom.toml`:**

```toml
[[operations]]
name = "listVisiblePosts"
type = "pipeline"
modelName = "_pipeline"
access = "isMemberOf('class-students', database.id)"

[operations.definition]
return = "posts"

[[operations.definition.steps]]
name = "settings"
type = "query"
modelName = "settings"
limit = 1

[operations.definition.steps.filter]
key = "class-settings"

[[operations.definition.steps]]
name = "posts"
type = "query"
modelName = "posts"
limit = 50

[[operations.definition.steps.filter."$or"]]
authorId = "$user.userId"

[[operations.definition.steps.filter."$or"]]
"$and" = [ "$steps.settings.first.peerVisible", { status = "approved" } ]

[operations.definition.steps.sort]
createdAt = -1
```

This pipeline first reads the settings record, then uses `$steps.settings.first.peerVisible` as a **boolean gate** in the posts query. When `peerVisible` is `true`, the `$and` branch opens and includes approved posts from all students. When the settings record doesn't have `peerVisible` set (or it's `false`), the gate short-circuits to no-match, so students only see their own posts. See [Boolean gate conditions](#boolean-gate-conditions) for details on this pattern. Note the mixed-type `$and` array carries natively in TOML, and an operation that takes no params simply omits `params`.

### User-scoped data via operations

Operations that scope data to the calling user using `$user.userId`:

**In `config/database-types/app_data.toml`:**

```toml
[[operations]]
name = "myItems"
type = "query"
modelName = "items"
access = "true"
[operations.definition]
filter = { ownerId = "$user.userId" }
sort = { createdAt = -1 }
limit = 50

[[operations]]
name = "createItem"
type = "mutation"
modelName = "items"
access = "true"

[[operations.definition.operations]]
op = "save"

[operations.definition.operations.data]
title = "$params.title"
ownerId = "$user.userId"
createdAt = "$now"

[[operations.params]]
name = "title"
type = "string"
required = true
```

**In app code:**

```swift
  // Each user only sees their own items (the operation filters on $user.userId).
  let result = try await client.databases.executeOperation(
    databaseId: dbId, name: "myItems"
  )

  // Creates an item owned by the calling user; server assigns the id.
  let createResult = try await client.databases.executeOperation(
    databaseId: dbId, name: "createItem",
    options: ExecuteOperationOptions(params: ["title": "My Item"])
  )
  // executeOperation returns a JSONValue; the mutation result shape is
  // { results: [{ success, id }] } — read the server-assigned id from it.
  let itemId = createResult["results"]?.arrayValue?.first?["id"]?.stringValue // server-assigned ULID
```

## Reserved Field Names

Any field starting with `_` (underscore) is reserved for internal use. The internal columns `_id`, `_type`, and `_data` are used by the storage engine. The `id` field is the record's primary key.

## Common Errors

| Symptom | Cause | Fix |
|---------|-------|-----|
| 403 on operation execute | CEL `access` expression returned false | Check user's groups/roles match the access expression |
| 403 on direct record access | User is not owner/manager | Use registered operations for non-privileged users |
| Using `addManager` for end-user access | Granting manager/owner gives administrative control, not scoped data access | Define registered operations with CEL access expressions instead |
| 400 on operation registration | Invalid CEL expression or missing required fields | Validate CEL syntax; ensure `name`, `type`, `modelName`, `access`, `definition` are provided |
| Empty results from query | Missing index on filtered field | Register indexes on fields used in filters |
| `$params.x` not substituted (filter key dropped, save field missing) | Parameter not declared in `params` schema | Add `x` to the operation's `params` schema; otherwise `$params.x` resolves to `undefined` and the surrounding key is silently dropped |
| 400 "ID conflict in save operation" | Both `opDef.id` and `data.id` were provided and differ | Set the id in one place only |
| 400 "applyToQuery matched more than 1000 records" | Default cap hit because `source.limit` was not set | Narrow `source.filter`, or set `source.limit` explicitly to opt into truncation |
| 409 from a mutation operation | One or more sub-ops in the batch failed (e.g., `ifNotExists` conflict) | Inspect the error message; mutation operations bundle all sub-ops into one `/batch` call and surface the first failure |
| `executeBatch` rejected with 400 "executeBatch only supports mutation operations" | Operation type is not `mutation` | Only `type = "mutation"` operations work with `executeBatch` |
| Records not found after save | Querying wrong `modelName` | Model names are case-sensitive collection identifiers |
| 422 `OPERATION_REFERENCES_UNDEFINED` on op create/edit | The op references a model or field not present in the type's `[models.*]` schema | Check the response's `refs` list; either fix the typo, or add the field to the schema. See [Schema gate](#schema-gate) |
| 422 `SCHEMA_BREAKS_OPERATIONS` on schema edit | A schema change would invalidate at least one existing op | Response lists the breaking ops + unresolved refs. Either reshape the schema or update the ops first |
| 422 `SCHEMA_HAS_UNCHECKABLE_OPS` on schema edit | At least one op has dynamic refs (e.g. `modelName = "$params.kind"`) the gate can't statically verify against the new schema | Confirm the dynamic refs are still consistent and re-run with `primitive sync push --accept-warnings` to commit |
| 409 `OPS_EXIST` on schema deletion | Removing the schema (`schema: null` via the API, or stripping all `[models.*]` blocks from the local file) is blocked while operations remain | Delete the registered operations first, then remove the schema |
| 413 `SCHEMA_TOO_LARGE` on schema edit | The inline schema exceeds the per-type cap | Trim the schema or split the model surface across multiple types |
