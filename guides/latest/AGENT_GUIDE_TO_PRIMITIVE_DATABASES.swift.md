# Agent Guide to Primitive Databases

All app data on Primitive lives in **databases**. In code and on the CLI a database is a **`db3`**: `client.db3`, `primitive db3`, `ctx.db3(...)` in server functions, `db3-types/` in the config tree. This guide is the one data-storage guide: it covers types and codegen, records, synced databases, access, and partitioning.

## The decision that shapes everything

Every database is **synced** (the default) or has **sync off**. Decide per database with two questions: **does each reader need all of this data to use the app**, and **may everyone who can read any of it read all of it?** Both yes → sync it. Either no → sync off, served by functions.

| | Synced | Sync off |
|---|---|---|
| Where data lives | A full copy on every subscribing device; the first open downloads it all (one-time), then only changes arrive | Server only; nothing downloaded |
| Reads and writes | Local: instant, offline-capable, live updates, concurrent edits merge | A request per call, through your server functions (and app admins) |
| Access | Whole-database grants: users, groups, link, whole app; the data model is multi-user from the start | No member access; each function checks the caller and returns only their slice |
| Fits | Data a person or group uses together: a user's lists or budget, a team's project, a chat channel, a small catalog | Large datasets each user needs a slice of (securities master, order history); per-person rules (grades, buyer/seller views) |

Most apps use both. Access is all-or-nothing per database, and a synced database's size is what every subscriber downloads — keep synced databases small and partitioned (one per user, team, project, channel, class or tenant; about 5 GB at most per database). Never put all app data in one database.

## Quick start

1. Declare a type in `primitive/<env>/db3-types/<name>.toml`:

```toml
# primitive/dev/db3-types/workspace.toml
[type]
name = "workspace"
timestamps = { create = "createdAt", update = "updatedAt" }

[models.tasks]
class_name = "Task"

[models.tasks.fields.title]
type = "string"
required = true

[models.tasks.fields.completed]
type = "boolean"
default = false

[models.tasks.fields.priority]
type = "number"
indexed = true

[models.tasks.fields.tags]
type = "stringset"

[models.tasks.fields.createdAt]
type = "number"

[models.tasks.fields.updatedAt]
type = "number"
```

2. Point the client codegen at that file (see [Codegen](#codegen)), then `primitive config push`.
3. Create (or get-or-create) a database of the type and subscribe:

```swift
  // A new database, typed by primitive/<env>/db3-types/workspace.toml:
  let db = try await client.db3.create(title: "Q3 launch", type: "workspace")

  // Or exactly one per user, created on first use:
  let mine = try await client.db3.getOrCreateWithAlias(
    Db3GetOrCreateWithAliasOptions(
      alias: AliasRef(scope: .user, aliasKey: "my-workspace"),
      title: "My workspace",
      type: "workspace"
    )
  )

  let sub = try client.db3.subscribe(mine.id)
  try await sub.available() // a readable copy is on this device
```

4. Read and write with the generated models:

```swift
  var task = try Task(title: "Review pull request", priority: 2).save(in: db3Id)

  task.completed = true
  task = try task.save(in: db3Id) // writes only the fields you changed

  try task.delete(in: db3Id)
```

## Types

A type is the schema for every database of that type, every client, and every function. `primitive config create db3-type <name>` scaffolds one; `primitive config fields db3-type` lists the top-level keys.

| Table / key | Meaning |
|---|---|
| `[type] name` | The type's name — what `create({ type })`, `ctx.db3(id, "<name>")` and `--type` take |
| `[type] timestamps` | `{ create = "<field>", update = "<field>" }` — epoch-ms numbers stamped on every create/update by any writer; declare the fields `number` |
| `[type] autoPopulatedFields` | `{ field = "user.userId" }` fills on create; `{ field = { value = "…", on = ["update"] } }` names lifecycles. Fills only absent/`null` fields |
| `[models.<key>]` | A model. `<key>` is the stored model name (used by functions, the CLI, includes); `class_name` is the generated class/struct |
| `[models.<key>.fields.<field>]` | `type` = `string` \| `number` \| `boolean` \| `date` \| `id` \| `stringset`; options `required`, `default`, `indexed`, `unique`, `enum` (string only; types the generated field, writes unchecked), `max_count` / `max_length` (stringset: member count / each member's length, client-checked), `auto_assign`, `auto_stamp` (`create`\|`update`\|`both`; epoch ms, stamped by whichever side writes). A `date` field takes an ISO 8601 string (or a number) |
| `[[models.<key>.unique_constraints]]` | `name` + `fields` — a compound unique key (upsert target). A unique constraint applies to scalar fields only. A `stringset` is refused `UNIQUE_ON_STRINGSET` at push, and a client model declaring one throws `UniqueStringsetError` |
| `[models.<key>.relationships.<rel>]` | `type` = `refersTo` \| `hasMany` (`model`, `related_id_field`, optional `order_by_field`, `order_direction`) or `hasManyThrough` (`model`, `join_model`, `join_model_local_field`, `join_model_related_field`) |
| `[triggers.<key>] triggers = [...]` | `{ on = "create"\|"update"\|"save", when = "<CEL>", set = { field = "<CEL>" } }` or `refuse = { code = "…", message = "…" }` |

Trigger and `autoPopulatedFields` expressions see `user.userId`, `record.*`, `database.id`, `now()` and `lookup()`. A rule never writes `id`. A `refuse` reaches the writer as `DB3_TRIGGER_REFUSED` with your code in `error.details.reason`; values a rule set come back as `appliedFields` (server calls) and sync to devices with the record.

**Typed databases.** Create with `type` (`client.db3.create({ type })`, `primitive db3 create --type`), or attach later with `primitive db3 type attach <db3-id> <type>` / `client.db3.setType(id, type)`; a type is never detached. On a typed database, a model the type doesn't declare is refused `DB3_MODEL_NOT_DECLARED`, the type's indexes, constraints and rules apply to every writer, and a client's own model declaration is not installed. An attach over records that break the type is refused `DB3_TYPE_REFUSED` with `details.reasons`.

**Changing a type.** Edit and `primitive config push`; the push reaches every database of the type in the background (`primitive db3-types push-status <type>`, `db3-types list`, `db3-types get <type>`). Adding models and fields is safe; a field newly `indexed`/`unique` is indexed everywhere. A database whose records break the new revision (e.g. a new `unique` over duplicates) refuses it and keeps its old revision; `push-status` names it, `primitive db3-types push <type> --retry <run-id>` retries after the fix. Removing a field leaves stored values; renaming is a new field plus a copy.

**Reserved field names.** `type`, anything starting with `_`, and the names generated models use are refused with `RESERVED_FIELD_NAME` (use `kind`/`category`/`status`; `id` is allowed). The generated-model names: `CodingKeys`, `applyAutoStampsForSave`, `applyStringSetChangeToYMap`, `clearLocalChanges`, `constructor`, `delete`, `deleteRecordLog`, `discardChanges`, `ensureLocalChanges`, `getChangedFields`, `getCurrentJSState`, `getCurrentValue`, `getDocumentId`, `getFromYjs`, `getOrCreateStringSet`, `getOriginalValue`, `getStringSetCurrentValues`, `getStringSetFromYjs`, `getValue`, `hasFieldChanged`, `hasLocalChange`, `hasUnsavedChanges`, `hashValue`, `isDirty`, `markAllChanged`, `markStringSetChange`, `normalizeNonStringSetValue`, `primitiveValues`, `related`, `resolveUnresolvedFormat2Document`, `save`, `saveRecordLog`, `schemaDefaultFor`, `setValue`, `syncState`, `toJSON`, `validateBeforeSave`, `validateFieldValue`.

**What a record holds.** Strings, numbers (integers exact to ±2^53), booleans, `null`, and string sets. No nested objects or non-string arrays — flatten, or store JSON in a `string`. A field's first non-null value fixes it as a set or a scalar; the other kind is then refused `DOCUMENT_ENTRY_INVALID`. At most 1,000 field names per model, ever. Writing `null` to a string set empties it.

## Codegen

Every codegen reads the **same type file**, so clients and functions agree on model names and fields (a typed database refuses models it doesn't declare).

- iOS: the template's `scripts/codegen.sh` regenerates the structs on every build path (Xcode and `swift build`). Put `bao-codegen.json` (`{ "input": "../../primitive/dev/db3-types/workspace.toml" }`) at the root of the app target's sources, and call `GeneratedModels.register(on: client)` once at startup.
- Functions: `primitive config push` (or `primitive functions codegen`) writes `functions/primitive-db3-types.d.ts`; `ctx.db3(id, "<type>")` then accepts only existing types, `.model("<key>")` only declared models, and rows are typed.

A database that grew without a type: `primitive db3 schema <db3-id> --toml --output primitive/dev/db3-types/<name>.toml` writes its observed fields as a type file to review, push and attach.


## Records

On a device, the generated models read and write the **local copy** of a synced database. Name the database with `{ db3: id }` on `query`/`count`/`aggregate`/`save`/`increment` (Swift: `QueryOptions(db3: [id])`, `AggregateOptions(db3: [id])`, `save(in: id)`, `Model.increment(id:fields:in: id)`), or set a default with `BaseModel.setDefaultDb3(id)` (Swift: `client.setDefaultDb3(id)`). Unless you name a database, a read applies to every database on the device.

In a server function, `ctx.db3(db3Id, "<type>").model("<key>")` reads and writes on the server with the app's authority — any database, synced or not, no grant needed. A model handle has `get`, `query`, `count`, `aggregate`, `save`, `patch`, `delete`, `increment` and `bulk`. Over HTTP from a client — admin tooling, or a database with sync off — `client.db3.records(id)` offers the same verbs with the same options and refusals.

### Querying

A query answers one page, never every matching row: 100 records when you omit `limit`, up to the `limit` you pass, capped at 10,000. Loop on `hasMore`, passing `nextCursor` back as the next call's cursor, to read everything a filter matches.

```swift
  let one = try Task.find("task-id")

  // A query answers one page at a time — 100 rows when you omit `limit`,
  // never every matching row. Loop on `hasMore`/`nextCursor` to read them all.
  var urgent: [Task] = []
  var cursor: String? = nil
  repeat {
    let page = try Task.queryPaged(
      ["completed": false, "$or": [["priority": ["$gte": 3]], ["tags": ["$contains": "urgent"]]]],
      options: QueryOptions(sortOrder: [("priority", -1)], cursor: cursor, db3: [db3Id])
    )
    urgent.append(contentsOf: page.data)
    cursor = page.hasMore ? page.nextCursor : nil
  } while cursor != nil

  let remaining = try Task.query(["completed": false], options: QueryOptions(db3: [db3Id])).count
```

Server function:

```ts
const tasks = ctx.db3(input.db3Id, "workspace").model("tasks");
const one = await tasks.get(id);                                  // row or null
const page = await tasks.query({ filter: { completed: false }, sort: { priority: -1 }, limit: 50, cursor });
// page.items, page.hasMore, page.nextCursor — loop on hasMore
const { count } = await tasks.count({ filter: { completed: false } });
```

Operators: `$eq $ne $gt $gte $lt $lte $in $nin $exists`, text `$startsWith $endsWith $containsText`, string sets `$contains $all $size`, and `$and $or`. Values compare as stored (`{ n: 5 }` never matches `"5"`). `{ field: null }` matches unset and stored-null; `$exists: false` only never-written; `$ne`/`$nin` also match unset/null; `{ field: { $eq: null } }` matches nothing. Sorting puts unset and null together — first ascending, last descending.

### Aggregations

`aggregate` returns one row per group: the group's values under `group`, each operation's result beside it as `count` or `<type>_<field>`. Operations: `count`, `sum`, `avg`, `min`, `max`.

```swift
  let rows = try Task.aggregate(AggregateOptions(
    groupBy: ["category"],
    operations: [AggregateOperation(type: .count)]
  ))
  // Returns: [["group": ["category": "work"], "count": 8], ["group": ["category": "home"], "count": 3]]
```

```swift
  let rows = try Task.aggregate(AggregateOptions(
    groupBy: ["category"],
    operations: [
      AggregateOperation(type: .count),
      AggregateOperation(type: .sum, field: "estimatedHours"),
      AggregateOperation(type: .avg, field: "priority"),
    ],
    filter: ["completed": false]
  ))
  // Returns: [["group": ["category": "work"], "count": 8, "sum_estimatedHours": 40, "avg_priority": 2]]
```

**StringSet facet aggregation** — grouping by a `stringset` field produces one row per member value:

```swift
  let rows = try Task.aggregate(AggregateOptions(
    groupBy: ["tags"],
    operations: [AggregateOperation(type: .count)]
  ))
  // Returns: [["group": ["tags": "urgent"], "count": 5], ["group": ["tags": "review"], "count": 2]]
```

Also group by membership (`{ field: "tags", contains: "urgent" }` → group key `has_tags_urgent`) or by a date period (`{ field: "dueDate", period: "month" | "quarter" | "year" }` → group key `dueDate_month`). A function aggregates with `ctx.db3(id, type).model(key).aggregate({ options: { groupBy, operations, filter, sort, limit } })`, which resolves to the bare row array.

### Includes

```swift
  let posts = try Post.query(nil, options: QueryOptions(db3: [db3Id]), include: [Post.includeAuthor()])
  for post in posts {
    print(post.title, post.relatedAuthor?.name ?? "—")
  }

  let authors = try Author.query(nil, options: QueryOptions(db3: [db3Id]), include: [Author.includePosts(limit: 10)])
```

`type` is `refersTo` (`sourceField` on this record), `hasMany` (`foreignKey` on the related records, with `limit`/`sort`/`filter`) or `refersToMany` (a string set of ids); related rows land under `_related[as ?? model]`. Nest up to 3 levels; includes resolve within the same database. A function passes the same `include` array to `query`.

### Writes

```swift
  // `email` is declared unique: merges into the record with this email, if any.
  _ = try AppUser(id: UUID().uuidString, email: "alice@example.com", name: "Alice")
    .save(in: db3Id, upsertOn: "email")

  // A named compound constraint, `name_parentId`:
  _ = try Category(id: UUID().uuidString, name: "Work", parentId: "root", color: "blue")
    .upsertByUnique("name_parentId", in: db3Id)
```

```swift
  var task = task
  task.category = "claimed"
  // Refused with DB3_CONDITION_NOT_MET if someone else changed it first:
  task = try await task.save(in: db3Id, options: Db3WriteOptions(condition: ["category": "open"], confirm: true))

  // Create only if no record has this id yet:
  try await Task(id: "daily-standup", title: "Standup").save(in: db3Id, options: Db3WriteOptions(ifNotExists: true))

  try await task.delete(in: db3Id, options: Db3WriteOptions(condition: ["completed": true]))
```

```swift
  let values = try await Post.increment(id: postId, fields: ["views": 1], in: db3Id, options: Db3WriteOptions(confirm: true))
```

```swift
  if var task = try Task.find(taskId) {
    task.tags?.insert("urgent")
    task.tags?.remove("low-priority")
    try task.save(in: db3Id)
  }
  let urgent = try Task.query(["tags": ["$contains": "urgent"]], options: QueryOptions(db3: [db3Id]))
```


Server function writes:

```ts
const tasks = ctx.db3(input.db3Id, "workspace").model("tasks");
const { record, appliedFields } = await tasks.save({ data: { title: "x" } });       // id minted when omitted
await tasks.save({ data: { email, name }, upsertOn: "email" });                     // unique field or constraint name
await tasks.save({ id: "daily", data: { title: "Standup" }, ifNotExists: true });
await tasks.patch(id, { data: { status: "done" }, condition: { status: "open" } });  // 409 DB3_CONDITION_NOT_MET
await tasks.patch(id, { data: { tags: { $add: ["urgent"], $remove: ["low"] } } });  // also { $clear: true }
const { values } = await tasks.increment(id, { views: 1 });                          // 404 DB3_RECORD_NOT_FOUND
await tasks.delete(id, { condition: { completed: true } });
await tasks.bulk([{ action: "create", data: { title: "a" } }, { action: "increment", id, fields: { n: 1 } }]);
await ctx.db3(input.db3Id, "workspace").bulk(
  [{ model: "tasks", action: "patch", id, data: { done: true } }, { model: "posts", action: "create", data: { title: "b" } }],
  { atomic: false },                                                                 // per-op results[]
);
```

Bulk actions: `create`, `patch`, `delete`, `upsert` (`constraint`: a constraint name — a `unique` field's is `<model>_<field>_unique`), `increment`, each with optional `condition` and `requestId`; up to 500 per call; all-or-nothing unless `atomic: false`. Pass a `requestId` (a ULID) on any server write you may retry.

### Bulk loading

Use a bulk load — not saves or `bulk()` — for imports, reference datasets, and any change of thousands of records or more than 1 MiB: one session uploads every row and they apply together.

```swift
  // rows: [DocumentIngestRow(model: "securities", id: "AAPL", patch: ["name": "Apple Inc."]), …]
  let chunks = try client.db3.ingests.buildChunks(rows: rows)
  let session = try await client.db3.ingests.create(db3Id: db3Id, options: DocumentIngestCreateOptions(mode: .merge))
  for chunk in chunks {
    _ = try await client.db3.ingests.uploadChunk(db3Id: db3Id, sessionId: session.sessionId, chunk: chunk)
  }
  _ = try await client.db3.ingests.commit(db3Id: db3Id, sessionId: session.sessionId)

  var loaded = try await client.db3.ingests.get(db3Id: db3Id, sessionId: session.sessionId)
  while !["complete", "failed", "aborted"].contains(loaded.state) {
    try await _Concurrency.Task.sleep(nanoseconds: 1_000_000_000)
    loaded = try await client.db3.ingests.get(db3Id: db3Id, sessionId: session.sessionId)
  }
  // loaded.state is "complete", "failed" (loaded.failure says why) or "aborted";
  // loaded.refusedRows counts rows the database refused, refusedSample names a few.
```

CLI: `primitive db3 ingest <db3-id> --input <dir> -y` (one `<model>.ndjson[.gz]` per model, each line an id and a patch), `primitive db3 ingests get|abort <db3-id> <session-id>`. To move a database: `primitive db3 export <db3-id> --output <dir>`, then `primitive db3 import <dir>` (a new id), `--into <db3-id>`, or `--keep-id` (the exported id, which must be free in the target app).

- A row is `{ model, id, patch }`: values replace fields, `null` removes one, `{ "_deleted": true }` deletes the record.
- `mode: "merge"` (default) touches only named records; `mode: "refresh"` also deletes every record of the rows' models no row names.
- Rows that break the type, a record's value rules, or a unique key are skipped and counted in `refusedRows` (`refusedSample` names a few); the load continues.
- Records appear together at completion; subscribed devices receive them in chunks. App writes made during the load win over conflicting rows.
- `buildChunks` holds all rows in memory — use the CLI beyond a few hundred thousand rows. A load can't be undone, and stopping the CLI's wait doesn't stop it — check `primitive db3 ingests get`.

### From the CLI

For inspecting or fixing data by hand (admin login), not app code: `primitive db3 records query|count|aggregate|save|patch|delete|increment <db3-id> <model> …`, `primitive db3 records bulk <db3-id> --data-file <file>`, `primitive db3 schema <db3-id>`, `primitive db3 inspect <db3-id>`.

### Indexes

```swift
  let index = try await client.db3.indexes.register(
    db3Id, Db3IndexRegistration(model: "contacts", field: "email")
  )
  // index.name == "idx_records_contact_email"

  _ = try await client.db3.indexes.register(
    db3Id,
    Db3IndexRegistration(model: "contacts", fields: ["accountId", "email"], unique: true, name: "account_email")
  )

  for each in try await client.db3.indexes.list(db3Id) {
    print(each.name, each.fields, each.unique, each.origins)
  }
  try await client.db3.indexes.drop(db3Id, name: index.name)
```

Every index costs on every write (server and each synced device), so declare `indexed` only on fields you filter on; adding one later is safe — existing records are indexed when the type is pushed or the index registered. Fields declared `indexed`/`unique` are indexed in every database of the type; register more per database (`primitive db3 indexes register <db3-id> --model <m> --field <f>`). Names: `idx_records_<model>_<field>`; composite (needs `unique` + `name`) `idx_named_<model>_<name>`; declared constraint `idx_uc_<model>_<constraint>`. Errors: `DB3_INDEX_NOT_UNIQUE` (existing duplicates, up to ten in `details.duplicates`), `DB3_INDEX_IN_USE` (dropping a declared index), `DB3_INDEX_SET_FIELD` (string sets need none). Registering/dropping needs the owner or an app admin; devices pick up new indexes at their next subscribe.

### Limits

| Limit | Value |
|---|---|
| Page size | default 100, max 10,000 (`DB3_PAGE_TOO_LARGE` above; never clamped) |
| Page bytes | 8 MiB of records: a plain page ends early with `hasMore: true`; a page whose `_related` would pass it is refused with `DB3_PAGE_TOO_LARGE` |
| Include depth | 3 |
| `hasMany` related rows per request | 1,000 (`DB3_PAGE_TOO_LARGE`) |
| `$in` / `$nin` values | 1,000 (`QUERY_IN_LIST_TOO_LARGE`) |
| Bound values per query | 100 (`QUERY_FILTER_TOO_MANY_BINDS`) |
| Bulk ops | 500 per call |
| One write / batch | 1 MiB JSON (`DOCUMENT_ENTRY_TOO_LARGE`) |
| Field names per model | 1,000 |

## Synced databases on a device

```swift
  let sub = try client.db3.subscribe(db3Id)
  let watching = sub.observe { event in
    if case .state(let state) = event { print(state.phase, state.progress as Any) }
  }

  // A readable copy: at once when this device already holds one, else after
  // the first load.
  try await sub.available()
  let open = try Task.query(["completed": false], options: QueryOptions(db3: [db3Id]))

  // `ready()`: caught up with every change made elsewhere.
  try await sub.ready()

  // Unsubscribed, the copy still answers queries and takes writes; they are
  // sent at the next subscribe.
  sub.dispose()
  try Task(title: "Write the report").save(in: db3Id)

  let state = try await client.db3.state(db3Id)
  print(state.resident, state.pendingWrites, state.lastSyncAt as Any)
```

| Call | Does |
|---|---|
| `client.db3.subscribe(id)` | Handle with `available` (readable copy on device), `ready` (caught up), `state`, events, `dispose()`. `ready` rejects `DB3_READY_TIMEOUT` (30 s without progress; `subscribe(id, { readyTimeoutMs })`, Swift `Db3SubscribeOptions(readyTimeout:)`), `DB3_ACCESS_REFUSED`, `DB3_SYNC_OFF`, `NOT_A_DB3` |
| `client.db3.unsubscribe(id)` | Disposes every handle for the database |
| `client.db3.refresh(id)` | Subscribe → caught up → dispose |
| `client.db3.state(id)` | `{ resident, subscribed, phase, progress, rows, pendingWrites, lastSyncAt, permission, error }` without subscribing |
| `client.db3.stored()` | Databases this device holds for the signed-in user |
| `client.db3.evict(id, { force })` | Delete the local copy; `DB3_PENDING_WRITES` unless `force` |
| `client.db3.setRetentionPolicy(...)` | Evict automatically (session-only, max count/bytes, TTL) |

`phase`: `idle`, `opening`, `loading`, `catching-up`, `live`, `reconnecting`, `gone`. Events: `db3:state`, `db3:ready`, `db3:rows` (`changes`, `origin`: `local`\|`remote`\|`sibling`), `db3:write-refused`, `db3:evicted`, `db3:gone` (`reason`: `permission`\|`sync-off`\|`client-outdated`), `db3:error`.

A resident copy answers reads and takes writes while unsubscribed or offline; writes queue and send at the next subscribe, in order. `db3.create` for the user's own synced database (no `custodian`) resolves at once, online or offline, with `pendingCreate` true in the state, and the client saves it under the same id in the background (`db3:created`, or `db3:create-failed` with its `id`, keeping its records). A refusal only the server decides (`DB3_TYPE_NOT_FOUND`, `DB3_TYPE_REFUSED`) arrives as that event, never as a rejection of `create`. Await `commitOfflineCreate` with the id to get the server's answer (`created`, or `refused` with the reason); `client.db3` calls that ask the server about the database wait for the commit on their own, rejecting `PENDING_CREATE` offline or after 30 s. A device may write offline for 7 days by default (`largeDocumentWindowDays` in `app.toml` `[app]`, 1–14).


### Re-rendering on change

Every generated model has `subscribe(callback)` → unsubscribe function; it fires on local and remote changes. Re-run the query in the callback.


```swift
    Group {
      switch loader.phase {
      case .loading: ProgressView()
      case .empty: Text("Nothing to do")
      case .loaded(let tasks): List(tasks) { Text($0.title) }
      }
    }
    .task {
      let sub = try? client.db3.subscribe(db3Id)
      try? await sub?.available()
      loader.documentReady = true
      loader.bind(client: client, subscribeTo: [.onModel(subscribe: Task.subscribe)]) { _ in
        (try? Task.query(["completed": false], options: QueryOptions(sortOrder: [("priority", -1)], db3: [db3Id]))) ?? []
      }
    }
```

### Concurrent edits

Different fields of a record both survive; the same field converges on one write everywhere; string-set members and increments all count; a unique key goes to the first write the database accepts (an upsert merges into it); conditions are checked by the database.

### Write refusals

A device write (save, patch, delete, increment) applies to the local copy and resolves before the database has seen it. If the database refuses it at sync, the record reverts and `db3:write-refused` carries `writes` (every affected write, in order, to redo), `error.code`, and `conflictingRecord`/`current` where relevant. Pass `confirm: true` (Swift `Db3WriteOptions(confirm: true)`) on the write instead to wait for the database's answer at the call site and reject there if it's refused, rather than handling `db3:write-refused` afterward.

```swift
  for await event in client.stream(for: Db3WriteRefusedEvent.self) {
    guard let first = event.writes?.first else { continue }
    if event.error.code == .documentConstraintViolated, let conflicting = event.conflictingRecord {
      ui.showConflict(model: first.model, recordId: first.recordId, message: "\(conflicting) already holds that key")
    } else {
      ui.showConflict(model: first.model, recordId: first.recordId, message: event.error.message)
    }
  }
```

| Code | Cause |
|---|---|
| `DOCUMENT_CONSTRAINT_VIOLATED` | Unique key taken by another device first |
| `DB3_CONDITION_NOT_MET` | Condition no longer held (`current`) |
| `DB3_TRIGGER_REFUSED` | A type rule refused (`error.details.reason`) |
| `DB3_MODEL_NOT_DECLARED` | Model not in the database's type |
| `WRITE_ACCESS_DENIED` | User lost write access |
| `DOCUMENT_ENTRY_INVALID` / `DOCUMENT_ENTRY_TOO_LARGE` | Value/name not storable / over 1 MiB |
| `DOCUMENT_OFFLINE_WINDOW_EXPIRED` / `DOCUMENT_EPOCH_FROZEN` / `DOCUMENT_EPOCH_BUDGET` | Offline too long / too many offline writes — reconnect |
| `DOCUMENT_STORE_PURGED` | Local copy cleared before sending |

## Access

| Level | Can |
|---|---|
| `reader` | Read all records |
| `read-write` | Read and write all records |
| `owner` | Also share, transfer, delete |

App owners/admins read, write and manage every database.

### Sharing a synced database

```swift
  let permissions = client.db3.permissions(db3Id)

  // By email — works whether or not the person is a member of the app yet
  _ = try await permissions.set(.email("colleague@example.com", permission: "read-write"))

  // With a group
  _ = try await permissions.grantGroup(GrantGroupPermissionParams(groupId: "engineering", permission: "reader"))

  // Who has access, and who hasn't accepted yet
  let members = try await permissions.list()
  let pending = try await permissions.listPendingInvitations()

  // Take it away
  try await permissions.remove("user-abc")
```

```swift
  let permissions = client.db3.permissions(db3Id)
  try await permissions.setLinkAccess(.reader)
  let state = try await permissions.getLinkAccess() // state.linkAccess: .reader | .readWrite | nil
  try await permissions.clearLinkAccess()
```

Grants by email work before the person joins (pending invitation). Group grants follow group membership.`client.db3.permissions(id).requestAccess(RequestAccessOptions(permission: .reader))`lets a member ask the owner. Removing a grant turns the user's subscription `gone` at once: the local copy stays readable, new writes are refused `WRITE_ACCESS_DENIED`, and queued writes stay until access returns or you evict.

### App access

**App access** — every current and future member: `client.db3.setAppAccess(id, "reader" | "read-write" | null)` (app admin), `primitive db3 app-access set <db3-id> reader|read-write|none`, or in a function `ctx.api.db3.setAppAccess({ db3Id, body: { level } })` with capability `db3:setAppAccess`. Content only at its level (`reader` reads, `read-write` writes; no share/delete). Refused on a sync-off database.

### Checking access

Device: `client.db3.get(id)` → `{ …, permission, accessSource }` for the signed-in user. Function: `auth.db3(id, type).permissions()` in `authorize` (caller's level or `null`); `ctx.api.db3.validateAccess({ db3Id, body: { userId } })` → `{ hasAccess, permission?, accessSource? }` for any member (always pass `userId`; omitted, it describes the function's app authority). An app role is not a content grant. Sync-off databases answer no access for members.

### Collections

A collection is a set of databases shared together: its members reach every database in it, including ones added later.

```swift
  // Create a collection and put databases in it
  let collection = try await client.collections.create(
    params: CreateCollectionParams(name: "Project Phoenix")
  )
  _ = try await client.collections.addDocument(collectionId: collection.collectionId, documentId: designDb3Id)
  _ = try await client.collections.addDocument(collectionId: collection.collectionId, documentId: specDb3Id)

  // One grant covers the whole set — including databases added later
  _ = try await client.collections.addMember(
    collectionId: collection.collectionId,
    params: .email("alice@example.com", role: .editor)
  )
```

```swift
  // Create
  let collection = try await client.collections.create(
    params: CreateCollectionParams(name: "Q1 Reports", description: "All quarterly report databases")
  )
  let collectionId = collection.collectionId

  // Add / remove a database
  _ = try await client.collections.addDocument(collectionId: collectionId, documentId: db3Id)
  _ = try await client.collections.removeDocument(collectionId: collectionId, documentId: db3Id)

  // Share with a group (reaches every database in the collection)
  _ = try await client.collections.addGroupMember(
    collectionId: collectionId, groupId: "engineering", role: .editor
  )

  // Share with an individual user. `.email(...)` also works for a
  // deferred grant that resolves on signup, exactly like databases and groups.
  _ = try await client.collections.addMember(
    collectionId: collectionId,
    params: .user(targetUserId, role: .viewer)
  )
```

```swift
  let members = try await client.collections.listMembers(collectionId: collectionId)

  let pending = try await client.collections.listPendingInvitations(collectionId: collectionId)
```

| Role | Can |
|---|---|
| `owner` | Everything a manager can, and delete the collection; one per collection (the custodian) |
| `manager` | Edit the collection and its settings; add, remove and re-role members |
| `editor` | Add databases; remove any database |
| `contributor` | Add databases; remove the ones they added |
| `viewer` | Read the collection and its databases |

- Members are user ids or `group:<groupId>` (a group can hold any role but `owner`). Collection grants add to a database's own; the highest wins. Deleting a collection keeps its databases.
- A sync-off database can't be added, and a group can't be granted a role on a collection holding one (`DB3_SYNC_OFF_NO_SHARING`).
- Group sharing changes and deletion are refused above 200 databases, and deletion also above databases × groups 400 (`COLLECTION_FANOUT_LIMIT`).
- Custodian: who holds the collection, `user:<userId>` (that member holds `owner`) or `app` (no owner). A client's create is held by its creator; a client can't name a custodian (`403 CUSTODIAN_NOT_ALLOWED`). A function's `ctx.api.collections.create({ body: { name } })` is held by the invoking member, who can use the `owner` role directly; `body.custodian` names `"app"` or any member instead (`404 SUBJECT_USER_NOT_FOUND` for a non-member). A run with no caller (cron, webhook) must name one (`CUSTODIAN_REQUIRED`), and a run an agent drives may name `"app"` or its initiator but no other member (`SUBJECT_USER_FORBIDDEN`). The admin API (`primitive collections create --owner`) also names one.
- Functions: `ctx.api.collections` creates, deletes and shares collections and adds databases; `ctx.api.collections.listDb3({ collectionId })` lists them. Rules the roles don't express go in a function whose `authorize` uses `requireCollectionRole`.
- CLI: `primitive collections create|get|delete|transfer`, `collections documents add|remove <collection-id> <db3-id>`, `collections members add|set-role|list|remove`.

### Custody

Every database has a `custodian`: `user:<userId>` (holder and owner; a member's own creates, a function's creates for its caller) or `app` (no owner; created from the CLI's admin token with no custodian, by an app admin naming `custodian: "app"`, or by a function naming it with capability `db3:transfer`). Transfer: owner → another member (old owner keeps `read-write`); to/from the app needs an app admin (`primitive db3 transfer <db3-id> app`) or `ctx.api.db3.transfer({ db3Id, body: { to } })` with `db3:transfer`. When a member leaves the app, databases they hold pass to the app if another user holds an accepted grant, any group is granted, or app access is set (link access and unaccepted invitations don't count); otherwise they are deleted. List by holder: `primitive db3 list --custodian app|user:<id>`; `client.db3.listMine()` lists the databases the signed-in user can open.

### Sync off

`sync: false` at create (`primitive db3 create … --sync off`) or `client.db3.setSync(id, false)` / `primitive db3 sync set <db3-id> off` / `ctx.api.db3.setSync({ db3Id, body: { sync: false } })` (capability `db3:setSync`); app admins and app owners only. Effects: subscribers dropped (`db3:gone`, `sync-off`), socket opens refused `DB3_SYNC_OFF`, every user content action refused (the owner keeps delete and transfer), grants suspended, new grants refused `DB3_SYNC_OFF_NO_SHARING`, app access refused. Functions and admins keep full read/write. Flipping back restores grants; a flip never rebuilds data.

Serve a sync-off database through functions that check the caller and filter:

```ts
import { allow, defineFunction, deny } from "primitive-functions";

export default defineFunction("class-grades", {
  authorize: async (input: { classDb: string }, auth) => {
    if (!auth.user) return deny("FORBIDDEN");
    const roster = auth.db3(input.classDb, "classroom").model("roster");
    const [me] = (await roster.query({ filter: { userId: auth.user.userId }, limit: 1 })).items;
    return me ? allow({ role: me.role }) : deny("NOT_FOUND");
  },
  handler: async (input, ctx) => {
    const filter = ctx.auth.role === "teacher" ? {} : { studentId: ctx.user!.userId };
    return (await ctx.db3(input.classDb, "classroom").model("grades").query({ filter })).items;
  },
});
```

`auth.db3` is read-only (`get`, `query`). A function creates databases with `ctx.api.db3.create({ body: { title, type, sync, custodian } })`; `custodian: "app"` needs capability `db3:transfer` and `sync: false` needs `db3:setSync`. To react to changes in databases of a type, declare `[[function.triggers.db3]]` with `type = "<type>"` in the function's TOML; the run receives the database id and the changed record keys.

A sync-off database grants nothing to its members (roster rows like the one above are app data, not a grant), so `listMine()` never lists it and the client has no way to discover `classDb` on its own. Keep a directory it can read: the same function that adds or removes someone from the roster also writes the database's id into a record of theirs — a field on their profile, or a row in a database they do hold — and a lookup function (or the client reading that record directly) answers which ids to pass in.

## Partitioning patterns

| Data | Partition | Access | Sync |
|---|---|---|---|
| A user's own data (todos, finances, journal) | One per user, `getOrCreateWithAlias` with a user-scoped alias | User holds it; may share whole | On |
| A team's / project's / channel's shared work | One per team/project/channel | Grant the group; group several with a collection | On |
| Small reference data all users read (catalog, plans) | One app-held | App access `reader`; written by admin/function | On |
| Large reference data (securities master, price history) | App-held, split by dataset | Functions answer the queries screens need | Off |
| Per-person visibility (grades, orders, HR) | One per class/store/tenant, app-held | Functions filter by caller | Off |

Store another system's id in a `unique` field and import with `upsertOn` it; let the platform mint record ids; reference records by platform id.

## Gotchas

- **Hiding records in the UI is not access control.** A synced database is on every reader's device in full. Split it, or turn sync off and serve it through functions.
- **Wait for `available` before reading** a database the device may not hold yet; otherwise `DB3_NOT_RESIDENT`.
- **Index what you filter on.** An unindexed equality filter reads every record of the model; development builds log a warning.
- **The first subscribe downloads the whole database** — show `state.progress`; keep databases partitioned so a device holds only what its user needs.
- **Unsent writes live only on the device** — surface `state.pendingWrites`.
- **One type file for every codegen.** Two schema files drift, and a typed database refuses models its type doesn't declare.
- **A new database's uploads wait for the server to have it** — an upload right after `create` is queued (`bytesTransferred: 0`) and sent once the create commits; await `commitOfflineCreate` first to upload at once.
- **`confirm: true` on a device write waits for the next subscribe** when the database isn't subscribed.
- **Upgrade client libraries forward only.** A local copy written by a newer client doesn't open in an older one; reload web tabs left open across a deploy.
- **Generated fields are optional in practice** — older records may lack newer fields.
- **App access and grants are refused on a sync-off database** (`DB3_SYNC_OFF_NO_SHARING`).
- **An app-held database has no owner** — nobody can open it until it is shared, given app access, or transferred.
