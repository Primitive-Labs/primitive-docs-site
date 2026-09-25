# Working with Databases in the Primitive platform

Guidelines for building apps with Primitive's server-side database storage.

## Core Concept: Databases

A **database** is an isolated, server-side data store that app code reaches through a **server function**:

1. **Records are read and written from a function.** `ctx.db(databaseId, "<type>")` returns a typed handle; `.model("<Model>")` binds one model, which has `query`, `count`, `aggregate`, `save`, `patch`, `delete` and `batch`. Filters and options are plain objects.
2. **The function runs on the app's authority.** Once a caller is through the function's `access` gate, the code reads and writes every model of every database of the app. The gate decides who may call; the code decides which rows they get.
3. **Schemaless.** Save any JSON records; a model exists once something is written to it. An optional `[models.*]` declaration on the database type types the handle and drives indexes.

**Lifecycle is function code too.** Creating, resolving, sharing and inspecting databases are `ctx.api.databases.*` calls (see [Managing databases](#managing-databases)). App code has no database calls: its only database-related call is invoking a function. App admins also have the `primitive databases` CLI for administrative lifecycle and record work.

**Size:** an individual database holds up to ~5 GB. For more, split data across databases (one per tenant, project, or domain) — each is an isolated instance that scales independently.

The function surface (`ctx.db`, `ctx.api`, the generated types, the paged envelope, registered queries) is documented in full in the [Server Functions guide](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#database-records). This guide covers the database side — types, schema, the query and write bodies, lifecycle, permissions — and does not re-teach the runtime.

## When to Use Databases vs. Documents

Primitive offers two storage options: **documents** (local-first, real-time collaborative) and **databases** (server-side, reached through functions). Many apps use both — documents for personal/collaborative data, databases for app-wide shared data.

See the [Data Modeling guide](AGENT_GUIDE_TO_PRIMITIVE_DATA_MODELING.md) for the decision framework, comparison table, and example app architectures.

## Quick Start

### 1. Configure a database type

```bash
primitive config init
```

```toml
# primitive/dev/database-type-configs/project.toml
[type]
databaseType = "project"
timestamps = { create = "createdAt", update = "modifiedAt" }

[models.tasks.fields.title]
type = "string"

[models.tasks.fields.projectId]
type = "string"
indexed = true

[models.tasks.fields.status]
type = "string"
indexed = true
```

### 2. Create a database

```bash
primitive databases create "Alpha Project" --type project    # prints the databaseId
```

Databases an app creates as it runs (one per team, per project) are created from a function with `ctx.api.databases.create` — see [Managing databases](#managing-databases).

### 3. Write the function that reads and writes it

```toml
# primitive/dev/functions/create-task.toml
[function]
key = "create-task"
entry = "functions/create-task/index.ts"
access = "true"
```

```ts
// primitive/dev/functions/create-task/index.ts
import { defineFunction } from "primitive-functions";

export default defineFunction(async (input: { databaseId: string; title: string; projectId: string }, ctx) => {
  const tasks = ctx.db(input.databaseId, "project").model("tasks");
  return await tasks.save({
    data: { title: input.title, projectId: input.projectId, status: "open", createdBy: ctx.user?.userId },
  });
});
```

```bash
primitive config push
```

Push writes the declarations that type `ctx.db` from the `[models.*]` blocks above (`functions/primitive-db-types.d.ts`), so a model the type does not declare is a compile error and push refuses the function.

### 4. Call it from app code

```swift
  struct Product: Decodable, Sendable { let id: String; let name: String; let priceCents: Double }
  struct ProductPage: Decodable, Sendable { let items: [Product]; let hasMore: Bool; let nextCursor: String? }

  let result: FunctionResult<ProductPage> = try await client.functions.invoke(
    "list-products",
    input: ["databaseId": databaseId]
  )

  if result.status == "completed" {
    for product in result.output?.items ?? [] {
      print(product.name, product.priceCents)
    }
  }
```

## Reading and Writing Records

All of this runs inside a function. See [The typed handle](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#the-typed-handle) for the handle's typing, codegen and the untyped `ctx.api.databases.records.*` form.

| Call | Body | Answers |
|---|---|---|
| `query(body?)` | `{ filter?, options? }` | `{ items, hasMore, nextCursor? }` |
| `count(body?)` | `{ filter? }` | `{ count }` |
| `aggregate(body)` | `{ options: { groupBy, operations, filter?, sort?, limit? } }` | `{ result }` |
| `save(body)` | `{ id?, data, condition?, ifNotExists?, upsertOn?, stringSets? }` | `{ success, id, appliedFields? }` |
| `patch(id, body)` | `{ data, condition?, stringSets? }` | `{ success, id, appliedFields? }` |
| `delete(id, body?)` | `{ condition? }` | `{ success }` |
| `batch(body)` | `{ operations: [{ op, id?, data?, fields?, stringSets?, condition?, ifNotExists?, upsertOn? }], atomic? }` | `{ results: [{ success, id, error?, status?, code?, values? }] }` |

- The model is bound by `.model(...)`; a body may not name another. `patch` and `delete` take the record id first; a body may not carry another id.
- `ctx.db(id, type).batch({ operations })` (on the database handle, not a model) spans models — each op names its own `modelName`.
- **Every row a read hands back carries `id: string`**, whatever the schema declares. A row you build for a write needs none: `save` without an `id` gets a generated ULID, returned as `id`.
- `appliedFields` carries the server-stamped values (`timestamps`, trigger `set`), present only when non-empty.
- A refused call throws an error carrying `status` and `errorCode` (`null` when the route answered none).

### Query options

| Key | Meaning |
|---|---|
| `sort` | `{ field: 1 \| -1, … }`. Records that lack the field — or hold it as `null` — sort first ascending, last descending |
| `limit` | Page size |
| `uniqueStartKey` | The cursor: pass the previous page's `nextCursor`. Opaque — never parse or build one |
| `direction` | `1` forward (default), `-1` backward |
| `projection` | Inclusion `{ title: 1, status: 1 }` returns just those fields; exclusion `{ internalNotes: 0 }` strips them server-side. `id` is always returned |
| `include` | Related records: `refersTo` (FK to one record), `hasMany` (target FK to this record), `refersToMany` (StringSet to several). Loaded records land under `_related` on each row |

```ts
const tasks = ctx.db(input.databaseId, "project").model("tasks");
let page = await tasks.query({
  filter: { projectId: input.projectId, status: { $ne: "done" } },
  options: { sort: { createdAt: -1 }, limit: 100 },
});
const rows = [...page.items];
while (page.hasMore) {
  page = await tasks.query({
    filter: { projectId: input.projectId, status: { $ne: "done" } },
    options: { sort: { createdAt: -1 }, limit: 100, uniqueStartKey: page.nextCursor },
  });
  rows.push(...page.items);
}
```

`hasMore: true` always carries a `nextCursor`, so the loop above cannot stall with rows left behind.

#### Sorting on a field some records don't have

Records are schemaless, so the rows of one model need not all carry the field you sort on, and a row may carry it as `null`. **Absent and `null` are one value for ordering** — a record that never wrote the field sorts exactly where a record holding `null` sorts:

| Sort | Where those rows land |
|---|---|
| `{ priority: 1 }` (ascending) | **first**, before every real value |
| `{ priority: -1 }` (descending) | **last**, after every real value |

That is SQLite's own `ORDER BY` placement for nulls, and it is identical on the server and in the JS and Swift clients, so the same records page in the same order everywhere. Paging visits every matching record exactly once in either sort direction and either paging direction; ties are broken by `id`, which every record has, so boundaries are stable when many rows share a value or share having none.

Cursors are **opaque** base64 tokens — never parse or construct one. A cursor issued before this ordering was specified keeps working unchanged, so a client holding one need not restart its walk.

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

`$startsWith`, `$endsWith`, and `$containsText` are mutually exclusive on the same field — only one substring operator per field per query.

**List size.** A single `$in` or `$nin` list holds at most **1,000 values**; each list in a filter is counted on its own. Past the cap the call is refused with `400 QUERY_IN_LIST_TOO_LARGE`, naming the field, the count and the cap. How to split a longer list — chunk an `$in` and merge the pages; never merge chunked `$nin` queries, but put every chunk in one filter's `$and`, which intersects them — is in the [Server Functions guide](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#the-typed-handle).

**Absent fields (#3166).** `$ne`, `$nin` and `{ field: null }` match records that never wrote the field; equality, the range operators and `$in` match only records that carry it. To keep excluding the missing case, put `null` in the `$nin` list: `{ deleted: { $nin: [null, true] } }`. Full semantics: [Documents guide, §Absent fields](AGENT_GUIDE_TO_PRIMITIVE_DOCUMENTS.md#absent-fields).

Multiple filters on different fields are implicitly combined with AND. Use an explicit `$and` only when you need two conditions on the *same* field (e.g. a range), and combine `$or` with other top-level fields freely.

**Group membership in a filter.** Expand the group in code, then filter with `$in`:

```ts
// listMembers answers a page: { items: [{ userId, … }], hasMore, nextCursor? }
const members = await ctx.api.groups.listMembers({ groupType: "team", groupId: input.teamId });
const ids = members.items.map((m: { userId: string }) => m.userId);
const tasks = await ctx.db(input.databaseId, "project").model("tasks")
  .query({ filter: { assigneeId: { $in: ids } } });   // up to 1,000 ids per list
```

Follow `nextCursor` (`cursor` argument) for a group larger than one page.

### Writes

- **`save`** is an upsert: it creates the record, or merges `data` into the existing one. `ifNotExists: true` makes it insert-only (an existing id fails with `409`). `upsertOn: "<field>"` resolves the target by that field's value instead of by id.
- **`patch`** merges `data` into an existing record; a missing record fails with `404`.
- **`delete`** removes one record by id.
- **`stringSets`** on `save`/`patch` seeds `stringset` fields: `{ tags: ["a", "b"] }`.

### Conditional writes (compare-and-swap)

`condition` is a filter the target record must match, checked in the same transaction as the write. Keep a `version` field and write only when it still matches what you read:

```ts
await tasks.patch(task.id, {
  data: { status: "done", version: task.version + 1 },
  condition: { version: task.version },
});
```

A condition that does not hold writes nothing: a single write throws with status `409`; inside a batch the item's result carries `code: "CONDITION_NOT_MET"`. Retry by re-reading and comparing again. `condition` works on `save`, `patch`, `delete`, `increment` and the set operations.

### Batch writes

`batch` runs several writes in one request. `op` is `save`, `patch`, `delete`, `increment`, `addToSet` or `removeFromSet`.

- `atomic: true` — all or nothing: an item that fails rolls the whole batch back, and the call throws with that item's status.
- `atomic` omitted or `false` — each item succeeds or fails on its own and successful siblings commit. A per-item failure does NOT throw; check `success` on each result. `code` is one of `NOT_FOUND`, `HOOK_DENIED`, `CONDITION_NOT_MET`, `ALREADY_EXISTS`, `UNIQUE_VIOLATION`, `VALIDATION_ERROR`.

### Atomic increments and string sets

No read-modify-write round trip:

```ts
await tasks.batch({
  operations: [
    { op: "increment", id: task.id, fields: { views: 1 } },          // result carries values: { views: <new> }
    { op: "addToSet", id: task.id, stringSets: { tags: ["urgent"] } },
    { op: "removeFromSet", id: task.id, stringSets: { tags: ["triage"] } },
  ],
});
```

The standalone forms are `ctx.api.databases.records.increment({ databaseId, body: { modelName, id, fields } })` (answers `{ success, id, values }`), and `addToSet` / `removeFromSet` with `body: { modelName, id, sets }`. On a missing record each fails with `404`.

### Aggregation

```ts
const { result } = await tasks.aggregate({
  options: {
    groupBy: ["status"],
    operations: [{ type: "count" }, { type: "sum", field: "estimatedHours" }],
    filter: { projectId: input.projectId },
    sort: { field: "count", direction: -1 },
    limit: 10,
  },
});
// result: { open: { count: 12, sum_estimatedHours: 40 }, done: { … } }
```

Operation types: `count`, `sum`, `avg`, `min`, `max` (`field` required except for `count`). Result keys are fixed — `count`, and `<type>_<field>` for the rest — and `sort.field` names one of them or a `groupBy` field. Ungrouped (`groupBy: []`), the result is flat, keyed by operation.

## Access

**The function's `access` gate is the authorization.** It is a CEL expression over the caller's identity (`user.userId`, `user.role`, `isMemberOf`, `memberGroups`, `hasRole`), evaluated on every call; app owners and admins bypass it. It does not see the function's input. See [The access gate](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#the-access-gate).

**The code scopes the rows.** The platform does not narrow a function's read to its caller:

- Filter on the caller: `query({ filter: { ownerId: ctx.user!.userId } })` (`ctx.user` is non-null on an HTTP invoke, `null` on a webhook or cron fire).
- Or bind the caller with a registered query's `$caller` parameter (below).
- A check that depends on the input — "is this caller in the team that owns `input.databaseId`?" — is code: read `ctx.api.groups.listUserMemberships({ userId: ctx.user!.userId, type: "team" })` (a bare array of `{ groupType, groupId, name, … }`) and refuse before touching the rows.

**Nothing is declared per database or per model.** A function reaches every database of the app; the capability lines a function declares are for integrations, secrets and the high-blast operations only (creating, deleting or re-owning a database among them — see [Capabilities](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#capabilities)).

## Registered queries

`defineQuery(name, definition)` and `defineMutation(name, definition)` from `primitive-functions` are a **registration helper over the same handle**, not a separate query language. `run(db, params)` is ordinary code over a `ctx.db(...)` handle. Registration adds:

- **A name**, unique per bundle (registering one twice throws).
- **Declared parameters**, validated, coerced and defaulted; an undeclared parameter is refused. `run`'s `params` is typed from the declaration.
- **The `$caller` binding** — `{ caller: true }` (or `type: "$caller"`): the platform injects the invoking user's id and refuses a call that supplies it. It throws in an invocation with no caller (webhook, cron).
- **A verified-run cache**, for queries only: `cache: { ttlMs }` is honored only when every model the run touched is listed in `models` and nothing was written; TTL is capped at one minute (`MAX_QUERY_CACHE_TTL_MS`). A mutation is never cached.

They are called from function code, with a handle — not by app code. Register at module scope (a registration inside the handler is invisible to push's manifest). The registration example, parameter grammar, cache keying and invalidation: [Registered queries](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#registered-queries).

## Realtime

Realtime for database data is **channels**: the function that writes a row publishes to a channel in the same handler, and clients that were granted and subscribed to that channel receive it. See [Channels](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#channels) for authorizing, subscribing, publishing and grant expiry.

## Configuring with the CLI

Database types are TOML files managed with `primitive config` — see the [Configuration guide](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md#the-sync-loop) for the sync loop (`init`/`pull`/`diff`/`push`).

```
database-type-configs/*.toml    # Database type configs
rule-sets/*.toml                # Rule sets (who may administer a type)
group-type-configs/*.toml       # Group type configs
```

### Database type config files

```toml
# primitive/dev/database-type-configs/project.toml
[type]
databaseType = "project"
ruleSetName = "project-admin-rules"                          # optional — who may edit/delete this type's config
timestamps = { create = "createdAt", update = "modifiedAt" } # optional — stamp every save/patch

[models.tasks.fields.title]
type = "string"

[models.tasks.fields.status]
type = "string"
indexed = true

[triggers.tasks]
triggers = [
  { on = "create", set = { createdBy = "user.userId" } },
  { on = "save", when = "record.status == 'done' && record.completedAt == null", set = { completedAt = "now()" } },
]
```

**TOML quoting tip:** a single-quoted TOML string cannot contain an apostrophe and has no escapes, so write CEL that uses `'…'` literals in double quotes: `when = "record.status == 'done'"`.

Deleting a database type is deleting `database-type-configs/<type>.toml` and running `primitive config push --prune`. The delete refuses with a `409` while live databases of that type exist — delete them first (`primitive databases delete <id>`); prune reports the type as blocked and keeps it, and the rest of the prune proceeds.

### Rule set config files

A rule set bound through `ruleSetName` governs who may administer the type's config:

```toml
# primitive/dev/rule-sets/project-admin-rules.toml
[ruleSet]
name = "project-admin-rules"
resourceType = "database_type"
description = "Controls who can manage the project database type"

[rules.config]
edit = "hasRole('admin')"
delete = "hasRole('admin')"
```

See the [Access Control guide](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md) for rule sets in general.

### Group type config files

See the [Users and Groups guide](AGENT_GUIDE_TO_PRIMITIVE_USERS_AND_GROUPS.md) for group types.

```toml
# primitive/dev/group-type-configs/team.toml
[groupTypeConfig]
groupType = "team"
ruleSetName = "team-rules"     # optional — rule set for group management
autoAddCreator = true          # auto-add creator as member (default: true)
```

### CLI commands for direct database management

```bash
primitive database-type-configs list | get <type>
primitive databases list [--owner <user-id>] | get <id> | create "Title" --type <type> [--initial-metadata '{...}'] | delete <id>

# Admin record introspection (app admins). `get` on a missing id prints null at
# exit 0 — a miss is not an error. Every --filter also accepts --filter-file
# <path> (JSON or TOML). `query --json` prints the list envelope
# `{ items, hasMore, nextCursor? }`. Without --group-by, aggregate covers the
# whole model; --json returns the { result } envelope keyed by the operation
# ({"result":{"count":2}}, {"result":{"sum_age":60}}).
primitive databases records models <id>
primitive databases records describe <id> <model-name>
primitive databases records query <id> <model-name> --filter '{"status":"open"}' [--limit <n>] [--cursor <c>]
primitive databases records get <id> <model-name> <record-id>
primitive databases records count <id> <model-name> [--filter '{...}']
primitive databases records aggregate <id> <model-name> --op avg --field price [--group-by status]

# Admin record writes — both MERGE the given fields (omitted fields keep their
# stored value). On a missing record: `save` creates it (id generated when
# omitted), `patch` fails 404. On an explicit null: `save` removes the key,
# `patch` stores the value null.
primitive databases records save <id> <model-name> [record-id] --data '{"status":"open"}'
primitive databases records patch <id> <model-name> <record-id> --data '{"status":"closed"}'
primitive databases records delete <id> <model-name> <record-id> [-y]
primitive databases records delete-all <id> <model-name> [--filter '{...}'] [-y]

# Atomic multi-model operations blob — the same file `documents records bulk`
# takes: { "operations": [{ "model", "action": "create"|"patch"|"delete",
# "id", "data", "precondition"? }, ...] }. All-or-nothing; reports
# { applied, added, updated, deleted }. `create` is a strict create (an
# existing id fails the batch), and a `precondition` that does not hold rolls
# the whole batch back. A `null` in a precondition means "present and holding
# null" — an absent field does not satisfy it. A precondition value must be a
# string or a number other than 0/1: a database compares JSON booleans as 1/0,
# so the CLI rejects those values. `deleted` counts delete operations applied,
# not records that existed.
primitive databases records bulk <id> --data-file ops.json -y

# Indexes
primitive databases indexes list <id> [--model <model-name>]
primitive databases indexes create <id> --model <model-name> --field <field> [--type string|number|boolean] [--unique]
primitive databases indexes drop <id> --model <model-name> --field <field> [-y]
primitive databases reindex <id> --from-schema     # re-provision the schema's unique indexes on one database

# Schema scaffold from live records
primitive databases schema generate <database-type>

# Data copy (records + indexes + constraints; type config excluded — run config push on the target first)
primitive databases export <id> --output ./out
primitive databases import ./out --overwrite [--dry-run] [--batch-size 5000] [--stop-on-error]
```

`databases import` writes records in chunked batch requests: `--batch-size` sets records per request (default 5000, ceiling 25000; an invalid value fails before any write). A failing chunk is reported and the run continues by default — the command still exits non-zero at the end; `--stop-on-error` aborts the whole run at the first failing chunk. Record upserts are keyed by `_id`, so re-running an import after a partial failure is safe.

## Database Types

A **database type** is a named configuration shared by every database of that type:

- **`[models.*]` schema** — optional model declarations: types `ctx.db` and drives indexes (see [Schema](#schema))
- **Triggers** — computed fields evaluated server-side before each save
- **`timestamps`** — created/modified stamps on every save and patch
- **`[metadata]` manifest** — resource metadata categories the type's trigger CEL may read
- **Rule set attachment** — who may edit or delete the type's config

### Schema

`[models.<Name>.fields.<field>]` blocks in the type's TOML declare its models. The database stays schemaless — a write is not refused for an undeclared field — but the declaration:

- **Types the function handle.** `config push` renders it into `functions/primitive-db-types.d.ts`; `ctx.db(id, "<type>").model("<Name>")` is then checked against it, rows carry the declared field types, and `config diff` reports when a schema change has left the declarations stale.
- **Drives indexes.** A field marked `indexed = true` or `unique = true` is provisioned on every database of the type when it is created, and a push that newly marks one back-provisions it across every existing database automatically.

```toml
[models.product.fields.name]
type = "string"

[models.product.fields.priceCents]
type = "number"
indexed = true

[models.product.fields.sku]
type = "string"
unique = true
```

**Field types — no object/JSON type.** `[models.*.fields.*]` accepts `string`, `number`, `boolean`, `date`, `id`, `stringset` only. `type = "object"` fails with `TOML_PARSE_ERROR: Invalid field type`. To store a structured payload, declare the field as `string`, write JSON-encoded values, and parse where consumed.

**Scaffolding a schema** for a type with live data: `primitive databases schema generate <database-type>` introspects the live records, infers field types where it can, and splices a `[models.*]` block into the local `primitive/<env>/database-type-configs/<type>.toml`. Review the result — it guesses from observed values — then `primitive config push` (or `--dry-run` first).

A schema over the per-type cap (100 KB) is refused with `413 SCHEMA_TOO_LARGE`.

### Triggers

Triggers are computed fields that run server-side before a record is saved, on every write path — a function's handle and the admin CLI alike. Configured per model in `[triggers.<modelName>]`.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `on` | string | Yes | When to fire: `"create"`, `"update"`, or `"save"` (both) |
| `when` | string | No | CEL boolean expression — trigger only fires if true. Errors → trigger does not fire (silent deny) |
| `set` | object | Yes | Map of field name to CEL expression. Each expression is evaluated and assigned. If the expression errors, the field is NOT set |

**Trigger CEL context:**

| Variable | Description |
|----------|-------------|
| `user.userId` | The user the write is attributed to — a function's caller |
| `user.role` | The role the write carried |
| `record.*` | The record being saved (current field values) |
| `database.id` | The database ID |
| `md.self.<category>.<key>` | A [resource metadata](AGENT_GUIDE_TO_PRIMITIVE_RESOURCE_METADATA.md) value on this database — loaded automatically when a trigger names it |
| `secrets.*` | App secrets — binds only the keys declared in the config's `secrets` manifest |
| `now()` | Current ISO 8601 timestamp |
| `lookup(modelName, id)` | Load another record by ID; returns `null` if missing |
| `isMemberOf(groupType, groupId)`, `memberGroups(groupType)`, `hasRole(role)` | Membership/role checks against the attributed user |

**Trigger CEL is fail-closed.** An error in `when` or `set` silently skips that trigger — a malformed expression doesn't crash the write, but it doesn't run either. Test triggers explicitly.

### Timestamps

`timestamps` on the `[type]` config stamps created and/or modified fields on `save` and `patch` writes. Field names are yours. Stamp values are epoch milliseconds, written only when the field is absent or `null` in the submission.

```toml
[type]
databaseType = "project"
timestamps = { create = "createdAt", update = "modifiedAt" }
```

| Key | Type | Description |
|-----|------|-------------|
| `create` | string | Field stamped with the current time on record creation. Optional. |
| `update` | string | Field stamped on every save and patch (including the first save). Optional. |
| `models` | string[] | Restrict stamping to these models. Optional — omit to stamp every model of the type. |

- **Every write path, every verb that carries data.** The directive belongs to the type, so it stamps a function's `save` and `patch`, each `save` and `patch` inside a batch — `upsertOn` included, on the create and on the update — and the admin CLI's writes. `delete`, `increment`, `addToSet` and `removeFromSet` carry no record body, so they never stamp.
- A batch `save` that inserts is stamped as a create even when an earlier operation in the same batch made it one (a failing save of the same id, or a `delete` of it).
- If a model has both a trigger and `timestamps`, the trigger fires after the stamp, so a trigger that sets the same field wins.

Use `timestamps` for plain audit times; use triggers when the rule depends on the record's data (`completedAt` only when `status == "done"`).

## Managing databases

Lifecycle calls go through `ctx.api.databases` in a function. Each takes one options object with its path parameters and a `body`; each answers the route's JSON. The high-blast calls need a capability line on the function (`capabilities = [...]`) — an undeclared one is refused with `FUNCTION_HIGH_BLAST_GRANT_MISSING`, naming the string to add.

| Call | Args | Capability |
|---|---|---|
| `create` | `{ body: { title, databaseType, initialMetadata? } }` → the database (`databaseId`, `title`, `databaseType`, `createdBy`, …) | `databases:create` |
| `get` | `{ databaseId }` | — |
| `update` | `{ databaseId, body: { title } }` | — |
| `delete` | `{ databaseId }` — removes every record and permission | `databases:delete` |
| `listPermissions` | `{ databaseId }` → `[{ userId, permission, grantedAt, grantedBy, … }]` | — |
| `addManager` | `{ databaseId, body: { userId, permission: "manager" } }` | `databases:addManager` |
| `revokePermission` | `{ databaseId, userId }` | `databases:revokePermission` |
| `transferOwnership` | `{ databaseId, body: { newOwnerId } }` | `databases:transferOwnership` |
| `listGroupPermissions` | `{ databaseId }` | — |
| `grantGroupPermission` | `{ databaseId, body: { groupType, groupId, permission: "manager" } }` | `databases:grantGroupPermission` |
| `revokeGroupPermission` | `{ databaseId, groupType, groupId }` | `databases:revokeGroupPermission` |
| `records.models` | `{ databaseId }` → `{ models: [...] }` | — |
| `records.describe` | `{ databaseId, modelName }` → `{ fields: [{ model_name, field_name, inferred_type, first_seen_at }] }` | — |

- **Creator and owner.** A database a function creates records the function's caller as `createdBy` and gives them the `owner` permission (a trigger fire, having no caller, records the app's system principal). A database type named in `create` that has no config yet is created with an empty one.
- **Another app's database id** answers as one that does not exist.
- **`records.describe`** reports each model's observed fields with inferred types `string`, `number`, `boolean`, `array`, `object`.

### One database per team

Reuse the database id as the group id, so a user's memberships name their databases:

```ts
// capabilities = ["databases:create"]
export default defineFunction(async (input: { title: string }, ctx) => {
  const db = await ctx.api.databases.create({ body: { title: input.title, databaseType: "workspace" } });
  await ctx.api.groups.create({ body: { groupType: "workspace", groupId: db.databaseId, name: input.title } });
  await ctx.api.groups.addMember({ groupType: "workspace", groupId: db.databaseId, body: { userId: ctx.user!.userId } });
  return { databaseId: db.databaseId };
});
```

Discovery is the same shape in reverse: `ctx.api.groups.listUserMemberships({ userId: ctx.user!.userId, type: "workspace" })` (a bare array of `{ groupType, groupId, name, … }`), then `ctx.api.databases.get({ databaseId: m.groupId })` for each, in a `Promise.all`. `ctx.api.groups.listDatabases({ groupType, groupId })` answers the databases a group holds a **permission grant** on (a bare array of `{ databaseId, databaseType, permission, … }`) — that is the administrative group grant below, not membership-by-id.

## Permissions

### How end users reach data

End users reach records **through functions**: the function's `access` gate decides who may call it, and its code decides which rows they get. Database permissions do not enter into it — a function reaches every database of the app whoever called.

### Owner and manager (administrative)

Owner and manager are **permission records on the database itself**, not a way into its data. The database routes refuse a non-admin caller, so holding `owner` or `manager` lets a user do nothing directly: app admins manage the records (`primitive databases permissions list | add-manager | remove-manager`, or a function via `ctx.api.databases.addManager` / `revokePermission` / `grantGroupPermission`, which runs on the app's authority), and a function reads them with `ctx.api.databases.listPermissions` / `listGroupPermissions` to make its own decisions — for example, letting only a database's managers rename it through your function.

| Level | Meaning |
|-------|---------|
| `owner` | The creator, set automatically at create |
| `manager` | An administrator the app has named; the only level a group grant can hold |

A group grant (`grantGroupPermission`) gives every member of the group `manager` at once and follows membership changes automatically. A permission record grants no bypass on a [resource metadata](AGENT_GUIDE_TO_PRIMITIVE_RESOURCE_METADATA.md) category's `readRule`/`writeRule` — only the app role does.

**Don't add end users as managers** to share data with them. Share by writing the function that serves them the rows.

## Best Practices

### Database design

- **Create multiple databases for isolation.** Each database is a separate isolated instance. Use separate databases for separate tenants, projects, or data domains.
- **Use database types** to share models, indexes and triggers across databases of the same kind.
- **Use triggers and `timestamps`** for server-side invariants (created times, audit fields) — don't trust client-provided values that pass through a function's input.
- **Per-database configuration** goes in a [resource metadata](AGENT_GUIDE_TO_PRIMITIVE_RESOURCE_METADATA.md) category (separate `readRule`/`writeRule`) or in a settings record inside the database.

### Function design

- **Gate tightly, scope in code.** `access` says who may call; the code says which rows. An unfiltered `query()` reads every row of the model, whoever called.
- **Never trust an id from input for authorization.** Take the caller from `ctx.user` or a `$caller` parameter, not from `input.userId`.
- **One function per screen's worth of reads.** Read several models with `Promise.all` over the handle and return them together — one client round trip (see the [Performance guide](AGENT_GUIDE_TO_PRIMITIVE_PERFORMANCE.md)).
- **Bulk, not loops.** One `query` with `$in` over a list of keys (up to 1,000) instead of a query per key; `pMap` from `primitive-functions` when per-item calls are unavoidable.
- **Prefer server-assigned ids.** Omit `id` on `save` unless you genuinely need a deterministic one (idempotency, a precomputed key).

### Performance

- **Index** every field used in `filter` or `sort` — declare `indexed = true` in the schema. Without an index, the database scans every record of that model.
- **Cap with `limit`** and page with `uniqueStartKey`.
- **`count` is much cheaper than `query`** for "how many".
- **`batch`** for many writes in one request.

## Common Patterns

### Settings record

Store mutable settings (feature flags, visibility toggles) as a regular record and read it in the same function as the data it governs:

```ts
import { defineFunction } from "primitive-functions";

export default defineFunction(async (input: { databaseId: string }, ctx) => {
  const db = ctx.db(input.databaseId, "classroom");
  const [settings] = (await db.model("settings").query({ filter: { key: "class-settings" }, options: { limit: 1 } })).items;

  const mine = { authorId: ctx.user!.userId };
  const filter = settings?.peerVisible ? { $or: [mine, { status: "approved" }] } : mine;
  return await db.model("posts").query({ filter, options: { sort: { createdAt: -1 }, limit: 50 } });
});
```

### Several models in one call

```ts
export default defineFunction(async (input: { databaseId: string; projectId: string }, ctx) => {
  const db = ctx.db(input.databaseId, "project");
  const [recent, byStatus, openBugs] = await Promise.all([
    db.model("tasks").query({ filter: { projectId: input.projectId }, options: { sort: { createdAt: -1 }, limit: 5 } }),
    db.model("tasks").aggregate({ options: { groupBy: ["status"], operations: [{ type: "count" }], filter: { projectId: input.projectId } } }),
    db.model("tasks").count({ filter: { projectId: input.projectId, label: "bug", status: "open" } }),
  ]);
  return { recent: recent.items, byStatus: byStatus.result, openBugs: openBugs.count };
});
```

### User-scoped data

A `$caller` registered query (see [Registered queries](#registered-queries)) or a filter on `ctx.user!.userId`. For writes, stamp the owner from `ctx.user` in code or with a trigger (`{ on = "create", set = { ownerId = "user.userId" } }`), never from input.

## Reserved Field Names

Two names are reserved, and a record write carrying either is refused with **400 `RESERVED_FIELD_NAME`** — by the database engine for a database record, and by the document records API for a document record:

- **`type`** — the internal `_type` column, which holds the **model name**. A filter on a field of that name matches the model name rather than a stored value.
- **Any name starting with `_`** — the storage engine's own columns (`_id`, `_type`, `_data`).

`id` is not reserved: it is the record's primary key.

A `[models.*]` schema that DECLARES either name is refused at push with the same 400, naming the block, and stores nothing — a refused update leaves the previously stored schema exactly as it was:

```
[models.product.fields.type]: Field 'type' is reserved (maps to internal _type column in queries)
```

Rename the declared field (`kind`, `category`, `status`) and push again.

## Common Errors

| Symptom | Cause | Fix |
|---------|-------|-----|
| `403 FUNCTION_ACCESS_DENIED` calling the function | The function's `access` gate denied the caller | Check the caller's role/groups against the gate |
| A user sees other users' rows | The function's query has no caller filter | Filter on `ctx.user!.userId` or use a `$caller` registered query |
| Compile error / refused push naming a model | The model is not declared in the type's `[models.*]` schema, or the declarations are stale | Declare the model, or run `config push` / `primitive functions codegen` to refresh |
| `409` on `patch` / `save` with `condition` | The record no longer matches the condition | Re-read and retry |
| `409` on `save` with `ifNotExists` | The record already exists | Use a plain `save` to upsert |
| `404` on `patch`, `increment`, set ops | The record doesn't exist | `save` creates; `patch` does not |
| `400 QUERY_IN_LIST_TOO_LARGE` | A `$in`/`$nin` list over 1,000 values | Chunk `$in` and merge; put `$nin` chunks in one `$and` |
| Per-item failures in a non-atomic `batch` | Each item's outcome is independent | Check `success` / `code` on each result, or pass `atomic: true` |
| Slow query | Missing index on a filtered or sorted field | Declare `indexed = true` on the field |
| Records not found after save | Querying the wrong model name | Model names are case-sensitive |
| Using `addManager` for end-user access | Manager is administrative control, not scoped data access | Put the access decision in a function |
| 400 `RESERVED_FIELD_NAME` on push or write | A field named `type` or starting with `_` | Rename the field. See [Reserved Field Names](#reserved-field-names) |
| 413 `SCHEMA_TOO_LARGE` on push | The inline schema exceeds the per-type cap | Trim the schema or split the models across types |
