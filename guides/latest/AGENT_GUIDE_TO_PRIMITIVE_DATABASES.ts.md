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

A model needs no `id` field; every record gets one, minted on create.

2. Point the client codegen at the `db3-types/` directory (see [Codegen](#codegen)), then `primitive config push`.
3. Create (or get-or-create) a database of the type and subscribe:

```typescript
  // A new database, typed by primitive/<env>/db3-types/workspace.toml:
  const db = await client.db3.create({ title: "Q3 launch", type: "workspace" });

  // Or exactly one per user, created on first use:
  const { id } = await client.db3.getOrCreateWithAlias({
    alias: { scope: "user", aliasKey: "my-workspace" },
    title: "My workspace",
    type: "workspace",
  });

  const sub = client.db3.subscribe(id);
  await sub.available; // a readable copy is on this device
```

4. Read and write with the generated models:

```typescript
  const task = new Task({ title: "Review pull request", priority: 2 });
  await task.save({ db3: db3Id });

  task.completed = true;
  await task.save(); // writes only the fields you changed

  await task.delete();
```

## Types

A type is the schema for every database of that type, every client, and every function. `primitive config create db3-type <name>` scaffolds one; `primitive config fields db3-type` lists the top-level keys. Pushing and deleting types is for app admins and owners, and a type takes no rule set: push refuses a type file with a `ruleSetName` or `ruleSetId` line, and `PUT db3-types/{name}` refuses a `ruleSetId` with `DB3_TYPE_RULE_SET_UNSUPPORTED`. Delete the line, or run `primitive config pull`, which rewrites the file without it.

| Table / key | Meaning |
|---|---|
| `[type] name` | The type's name — what `create({ type })`, `ctx.db3(id, "<name>")` and `--type` take |
| `[type] timestamps` | `{ create = "<field>", update = "<field>", models = ["<model>"] }` — epoch-ms numbers stamped on every create/update by any writer; `models` (optional) limits them to those models; declare the fields `number` |
| `[type] autoPopulatedFields` | `{ field = "user.userId" }` fills on create; `{ field = { value = "…", on = ["update"] } }` names lifecycles. Fills only absent/`null` fields |
| `[type] models` | `["<key>", …]` — models this type holds that another file in `db3-types/` declares. Push includes those declarations. Once any file names one, a name no file declares, or one model declared differently in two files, is refused at push |
| `[models.<key>]` | A model. `<key>` is the stored model name (used by functions, the CLI, includes); `class_name` is the generated class/struct |
| `[models.<key>.fields.<field>]` | `type` = `string` \| `number` \| `boolean` \| `date` \| `id` \| `stringset` \| `vector` (see **Vector fields** below; declare `id` only to set its options; an undeclared `id` is minted on create); options `required`, `default`, `indexed`, `unique`, `enum` (string only; types the generated field, writes unchecked unless `enforce_enum = true`; see **Enforced enums** below), `max_count` / `max_length` (stringset: member count / each member's length, client-checked), `auto_assign`, `auto_stamp` (`create`\|`update`\|`both`; epoch ms, stamped by whichever side writes). A `date` field takes an ISO 8601 string (or a number) |
| `[[models.<key>.unique_constraints]]` | `name` + `fields` — a compound unique key (upsert target). A unique constraint applies to scalar fields only. A `stringset` is refused `UNIQUE_ON_STRINGSET` at push, and a client model declaring one throws `UniqueStringsetError` |
| `[models.<key>.relationships.<rel>]` | `type` = `refersTo` \| `hasMany` (`model`, `related_id_field`, optional `order_by_field`, `order_direction`) or `hasManyThrough` (`model`, `join_model`, `join_model_local_field`, `join_model_related_field`) |
| `[triggers.<key>] triggers = [...]` | `{ on = "create"\|"update"\|"save", when = "<CEL>", set = { field = "<CEL>" } }` or `refuse = { code = "…", message = "…" }` |

**Defaults.** A `default` is written on create. In a database, a record stored without the field reads as the default in both clients (generated models); the read stores nothing, and a field cleared to `null` reads empty. A record in a document with Yjs state reads only what it holds: a missing field reads empty. Queries compare stored values: `{ completed: false }` misses a record stored without `completed`; add `{ completed: { $exists: false } }` under `$or`.

Trigger and `autoPopulatedFields` expressions see `user.userId`, `user.writer`, `record.*`, `database.id`, `now()` and `lookup()`. A rule never writes `id`. `user.writer` is `"user"` (a member's device or API call, or a function acting `asUser`), `"function"` (a server function someone invoked; `user.userId` is that member) or `"system"` (cron, webhook, database change, `--as system`; `user.userId` is `sys:<appId>`). Keep members off a model only your functions write with `{ on = "save", when = "user.writer == 'user'", refuse = { code = "PIPELINE_ONLY" } }`. It stops saves and batches, not a bulk load by anyone with write access: a guard against app mistakes, not an access boundary (for that, a sync-off database only functions write). A `refuse` reaches the writer as `DB3_TRIGGER_REFUSED` with your code in `error.details.reason`; values a rule set come back as `appliedFields` (server calls) and sync to devices with the record. Reading a field the record lacks is an error, not `false`: a refusing trigger whose `when` errors refuses the write with reason `TRIGGER_EVALUATION_FAILED`, and a `set` trigger's is skipped, so guard optional fields (`has(record.status) && record.status == 'blocked'`). `has()` takes a path that starts at a name; for a looked-up record's field write `lookup('m', record.ref).?field.hasValue()`. Push refuses an expression that can never be evaluated, naming the trigger and the failing term.

**Typed databases.** Create with `type` (`client.db3.create({ type })`, `primitive db3 create --type`), or attach later with `primitive db3 type attach <db3-id> <type>` / `client.db3.setType(id, type)`; a type is never detached. On a typed database, a model the type doesn't declare is refused `DB3_MODEL_NOT_DECLARED`, the type's indexes, constraints and rules apply to every writer (a bulk load skips the rules; see Bulk loading), and a client's own model declaration is not installed. An attach over records that break the type is refused `DB3_TYPE_REFUSED` with `details.reasons`.

**Untyped databases.** Without a type, a database's indexes and unique constraints come from the declarations of the clients that write it. A client declares to an untyped database only the models it uses there: models its copy holds records of; a model it first writes there (declared just before the write); models bound to it with `BaseModel.setModelDefaultDb3(model, id)` (Swift: `client.setModelDefaultDb3(model, id)`); and, when it is the client's default (`BaseModel.setDefaultDb3(id)`; Swift: `client.setDefaultDb3(id)`), every model without a default database of its own. A model used only in other databases, runtime additions (`client.registerModel`; Swift: `client.registerModels`) included, is not declared there. To have a model's indexes and unique constraints in force before its first record, bind it with `setModelDefaultDb3` before subscribing.

**Changing a type.** Edit and `primitive config push`; the push reaches every database of the type in the background (`primitive db3-types push-status <type>`, `db3-types list`, `db3-types get <type>`). Adding models, fields and enum values, making a field optional, and `indexed`/`unique`/`default`/`description` edits are safe; a field newly `indexed`/`unique` is indexed everywhere. Once a database of the type exists, push refuses `DB3_TYPE_NOT_APPEND_ONLY` (409, `details.reasons[]` = `{ path, change, reason }`, nothing written) for a removed model or field, a changed field `type`, a removed `enum` value, a field made `required` or a new `required` field in an existing model, and a changed vector `dims`/`element`; `config push` checks every type before writing any and prints each reason. Add a new field instead (copy values with a function); with no database of the type, everything pushes. To push one anyway: set `[app] allowFieldRemoval = true` in `app.toml`, push, then `primitive config push --allow-field-removal` (HTTP: `?allowFieldRemoval=true`; refused `DB3_TYPE_FIELD_REMOVAL_NOT_ENABLED` unless the stored setting is already `true`). A database whose records break an allowed revision (e.g. a new `unique` over duplicates) refuses it and keeps its old revision; `push-status` names it, `primitive db3-types push <type> --retry <run-id>` retries after the fix.

**Reserved field names.** `type`, anything starting with `_`, and the names generated models use are refused with `RESERVED_FIELD_NAME` (use `kind`/`category`/`status`; `id` is allowed). The generated-model names: `CodingKeys`, `UniqueConstraint`, `UniqueField`, `applyAutoStampsForSave`, `applyStringSetChangeToYMap`, `clearLocalChanges`, `constructor`, `delete`, `deleteRecordLog`, `discardChanges`, `ensureLocalChanges`, `getChangedFields`, `getCurrentJSState`, `getCurrentValue`, `getDocumentId`, `getFromYjs`, `getOrCreateStringSet`, `getOriginalValue`, `getStringSetCurrentValues`, `getStringSetFromYjs`, `getValue`, `hasFieldChanged`, `hasLocalChange`, `hasUnsavedChanges`, `hashValue`, `isDirty`, `markAllChanged`, `markStringSetChange`, `normalizeNonStringSetValue`, `primitiveValues`, `related`, `resolveUnresolvedFormat2Document`, `save`, `saveRecordLog`, `schemaDefaultFor`, `setValue`, `syncState`, `toJSON`, `validateBeforeSave`, `validateFieldValue`.

**Vector fields.** `type = "vector"` with `dims` (how many numbers, at least 1) and `element` = `f32` \| `f16` \| `i8`; only `required` besides. Bytes per vector: `f32` 4·dims, `f16` 2·dims (rounded to nearest, magnitudes ≤ 65504), `i8` dims + 4 (one signed byte each plus a per-vector scale, within 1/254 of the largest magnitude); stored as RFC 4648 standard base64 with padding, 4·ceil(bytes/3) characters. No `indexed`, `unique`, `default`, `enum`, `auto_*`, `max_*`, and no unique constraint over one. A value that doesn't fit (wrong length, non-finite number, `f16` overflow, non-canonical base64) is refused `DB3_VECTOR_INVALID`; a push changing `dims`/`element` while records hold values is refused for that database (`DB3_TYPE_REFUSED`).

**Enforced enums.** `enforce_enum = true` beside a string field's `enum` (or its shared-enum `ref`) makes the database refuse any create, patch, upsert, trigger `set` or load line that leaves a non-null value outside the set, with `DB3_VALUE_NOT_IN_ENUM` (message: `tasks id=<id>: field "status" takes one of open, doing, done; the write is "archived"`); a device refuses it before sending, in the same words. `null`, an absent field and a delete pass. The value is `true` or `false` only (anything else is refused when the type is read); not on the `id` field; an enforced `default` must be in the set; a database type with databases of it is refused `ENFORCE_ENUM_DB3_ONLY`. A push or attach that turns enforcement on, or drops a value, over a database whose live records hold an outside value is refused for that database (`DB3_TYPE_REFUSED`, reason with `rows` and up to three `[id, value]` `samples`); it keeps its revision until the records are fixed and the push retried. In an `atomic: false` batch the outside operation's `results[i]` carries the code with `status: 400`; a default batch fails whole at its `index`.

Without `enforce_enum`, a refusing trigger can hold a field to a set: `{ on = "save", when = "has(record.status) && !(record.status in ['open', 'doing', 'done'])", refuse = { code = "STATUS_NOT_ALLOWED" } }`. It repeats the list, it does not run on bulk loads (`enforce_enum` holds on loads too), and a `when` that can't be evaluated refuses the write with reason `TRIGGER_EVALUATION_FAILED`.

**Shared enums.** A field can take its values from a plain string enum in `schemas/`: `ref = "Status"` (a name one `schemas/` file declares), `"common.Status"` (refused at push when `common` is also a function, prompt or db3-type key; push lists the long forms) or `"schemas.common.Status"`. Push sends the field as `type = "string"` with the def's `enum`, so the stored type, generated models and `ctx.db3` see the values; editing the def shows the type modified in `config diff` until the next push, and `config pull` keeps the `ref` while the def still gives the server's values. A shared enum def has `type = "string"`, a non-empty `enum` of distinct strings, and optionally `description`, `title`, `examples`, `deprecated` and `$comment`. Beside `ref`, write the field's other options but not `type` or `enum`.

```toml
# primitive/dev/schemas/common.toml
[defs.Status]
type = "string"
enum = ["open", "doing", "done"]
```

```toml
# primitive/dev/db3-types/workspace.toml
[type]
name = "workspace"

[models.tasks.fields.status]
ref = "Status"
indexed = true
```

**What a record holds.** Strings, numbers (integers exact to ±2^53), booleans, `null`, string sets, and vectors (as their base64 text). No nested objects or non-string arrays — flatten, or store JSON in a `string`. A field's first non-null value fixes it as a set or a scalar; the other kind is then refused `DOCUMENT_ENTRY_INVALID`. At most 1,000 field names per model, ever. Writing `null` to a string set empties it.

## Codegen

Every codegen reads the **same type files**, so clients and functions agree on model names and fields (a typed database refuses models it doesn't declare). Point client codegen at the `db3-types/` directory: it reads every `*.toml` in it and generates each model once — one set of models and one registration list for every kind of database the app opens. A model two files declare differently is refused, naming both files and the fields. Generated headers name the input files.

- Web: the template's `pnpm codegen` needs no edit. It runs js-bao-codegen-v2 once per type file in the environment's `db3-types/`, writing `src/models/<type>/`, and writes `src/models/index.ts`, whose `allModels` holds every type's classes. Pass `allModels` to the client at initialization. A model shared through `[type] models` is generated only under the type that declares it. The template's `dev`, `build` and `test` scripts run `pnpm codegen` first. Outside the template, run `js-bao-codegen-v2 -i ../primitive/dev/db3-types -o src/models` to get one run over the whole directory (`-i` also takes files, repeated). It writes `<Class>.generated.ts` per model and one `index.ts` exporting `allModels`. `--schema-import` is refused when more than one input declares models. Codegen reads `schemas/` beside `db3-types/`; for a type file that uses a shared-enum `ref` it also writes `<file>.toml.generated.ts` (the type with refs expanded), which `index.ts` registers in place of the type file, whatever `--schema-import` names.
- Web, shared enums: js-bao-codegen-v2 writes `shared-enums.generated.ts` beside the models with `type Status` and `StatusValues` (the def's values, in order, `as const`), types each field that refs the def as `Status`, and re-exports both from `index.ts`. The function invokers in `functions/generated/` import the same declaration from their own `shared-enums.generated.ts` wherever a field is exactly the enum (a direct `ref`, or a db3 row field that refs it), so function results and models share one type. A def whose name a generated file already uses (a model class, `<Class>Attrs`, `BaseModel`, `JsBaoClient`, an invoker's own type) appears as `Schemas_<PascalFile>_<Name>`. A field with `max_length` beside its `ref` stays an inline union in the invokers.
- Functions: `primitive config push` (or `primitive functions codegen`) types the `Db3` namespace of `functions/primitive.d.ts`; `ctx.db3(id, "<type>")` then accepts only existing types, `.model("<key>")` only declared models, and rows are typed (a type's `timestamps` fields required numbers on read rows of the models it stamps, absent on the others; writes leave them out).

A database that grew without a type: `primitive db3 schema <db3-id> --toml --output primitive/dev/db3-types/<name>.toml` writes its observed fields as a type file to review, push and attach.

**More than one client in one process** (a Node script, a worker, a test harness). A model class binds to the first client that uses it: a second `initializeClient` call in the same process shares that binding, and its model calls — reads, writes, subscriptions — route to the first client instead of its own; a save into a database only the second client holds is refused `DB3_MODEL_ON_OTHER_CLIENT`. Give each client its own model classes, or read and write through `client.db3.records(id)` instead of the generated models; it takes the same verbs and needs no bound class.

**Adding a model at runtime.** `client.registerModel(Model)` adds a model class to a running client, in every database it holds. It waits for databases still connecting, up to the `registerModelTimeoutMs` client option (default 30 s): at a sixth of it (10 s at most) it logs a warning naming them, at the bound it rejects `REGISTER_MODEL_TIMEOUT` with `details.documentIds` and `details.timeoutMs`. The model is then not registered; call again once they finish connecting, or raise `registerModelTimeoutMs` for a database that legitimately connects slowly. A value that is not a positive number is refused with `INVALID_ARGUMENT` when the client is built.

## Records

On a device, the generated models read and write the **local copy** of a synced database. Name the database with `{ db3: id }` on `query`/`count`/`aggregate`/`save`/`increment` (Swift: `QueryOptions(db3: [id])`, `AggregateOptions(db3: [id])`, `save(in: id)`, `Model.increment(id:fields:in: id)`), or set a default with `BaseModel.setDefaultDb3(id)` (Swift: `client.setDefaultDb3(id)`). Unless you name a database, a read applies to every database on the device.

In a server function, `ctx.db3(db3Id, "<type>").model("<key>")` reads and writes on the server with the app's authority — any database, synced or not, no grant needed. A model handle has `get`, `query`, `count`, `aggregate`, `save`, `patch`, `delete`, `increment`, `batch` and `batchRead`. Over HTTP from a client — admin tooling, or a database with sync off — `client.db3.records(id)` offers the same verbs with the same options and refusals.

### Querying

A query answers one page, never every matching row: 100 records when you omit `limit`, up to the `limit` you pass, capped at 10,000. Loop on `hasMore`, passing `nextCursor` back as the next call's cursor, to read everything a filter matches.

```typescript
  const one = await Task.find("task-id");

  // A query answers one page at a time — 100 rows when you omit `limit`,
  // never every matching row. Loop on `hasMore`/`nextCursor` to read them all.
  const urgent: Task[] = [];
  let cursor: string | undefined;
  do {
    const page = await Task.query(
      { completed: false, $or: [{ priority: { $gte: 3 } }, { tags: { $contains: "urgent" } }] },
      { db3: db3Id, sort: { priority: -1 }, uniqueStartKey: cursor },
    );
    urgent.push(...page.data);
    cursor = page.hasMore ? page.nextCursor : undefined;
  } while (cursor);

  const remaining = await Task.count({ completed: false }, { db3: db3Id });
```

Server function:

```ts
const tasks = ctx.db3(input.db3Id, "workspace").model("tasks");
const one = await tasks.get(id);                                  // row or null
const page = await tasks.query({ filter: { completed: false }, sort: { priority: -1 }, limit: 50, cursor });
// page.items, page.hasMore, page.nextCursor — loop on hasMore
const { count } = await tasks.count({ filter: { completed: false } });
```

Operators: `$eq $ne $gt $gte $lt $lte $in $nin $exists`, text `$startsWith $endsWith $containsText`, string sets `$contains $all $size`, and `$and $or`. Values compare as stored (`{ n: 5 }` never matches `"5"`). `{ field: null }` matches unset and stored-null; `$exists: false` only never-written; `$ne`/`$nin` also match unset/null; `{ field: { $eq: null } }` matches nothing. Sorting puts unset and null together — first ascending, last descending. A field can't go back to never-written once a write names it: assigning `undefined` (or `null`) stores `null`, with no operation that removes the key again; clearing a field means writing `null` to it, after which `$exists: false` stops matching but `{ field: null }` still does.

### Aggregations

`aggregate` returns one row per group: the group's values under `group`, each operation's result beside it as `count` or `<type>_<field>`. Operations: `count`, `sum`, `avg`, `min`, `max`.

```typescript
  const rows = await Task.aggregate({
    groupBy: ["category"],
    operations: [{ type: "count" }],
  });
  // Returns: [{ group: { category: "work" }, count: 8 }, { group: { category: "home" }, count: 3 }]
```

```typescript
  const rows = await Task.aggregate({
    groupBy: ["category"],
    operations: [{ type: "count" }, { type: "sum", field: "estimatedHours" }, { type: "avg", field: "priority" }],
    filter: { completed: false },
  });
  // Returns: [{ group: { category: "work" }, count: 8, sum_estimatedHours: 40, avg_priority: 2 }]
```

**StringSet facet aggregation** — grouping by a `stringset` field produces one row per member value:

```typescript
  const rows = await Task.aggregate({
    groupBy: ["tags"],
    operations: [{ type: "count" }],
  });
  // Returns: [{ group: { tags: "urgent" }, count: 5 }, { group: { tags: "review" }, count: 2 }]
```

Also group by membership (`{ field: "tags", contains: "urgent" }` → group key `has_tags_urgent`) or by a date period (`{ field: "dueDate", period: "month" | "quarter" | "year" }` → group key `dueDate_month`). A function aggregates with `ctx.db3(id, type).model(key).aggregate({ options: { groupBy, operations, filter, sort, limit } })`, which resolves to the bare row array.

### Includes

```typescript
  const posts = await Post.query({}, {
    db3: db3Id,
    include: [{ model: "authors", type: "refersTo", sourceField: "authorId", as: "author" }],
  });
  for (const post of posts.data as (Post & { _related?: { author?: Author } })[]) {
    console.log(post.title, post._related?.author?.name);
  }

  const authors = await Author.query({}, {
    db3: db3Id,
    include: [{ model: "posts", type: "hasMany", foreignKey: "authorId", as: "posts", limit: 10 }],
  });
```

`type` is `refersTo` (`sourceField` on this record), `hasMany` (`foreignKey` on the related records, with `limit`/`sort`/`filter`) or `refersToMany` (a string set of ids); related rows land under `_related[as ?? model]`. Nest up to 3 levels; includes resolve within the same database. A function passes the same `include` array to `query`.

### Writes

```typescript
  // `email` is declared unique: merges into the record with this email, if any.
  await new AppUser({ email: "alice@example.com", name: "Alice" }).save({ db3: db3Id, upsertOn: "email" });

  // A named compound constraint, `name_parentId`:
  await Category.upsertByUnique(
    "name_parentId",
    ["Work", "root"],
    { name: "Work", parentId: "root", color: "blue" },
    { db3: db3Id },
  );
```

A generated model types the upsert names: `upsertOn` takes a field one unique constraint covers alone, `upsertByUnique` a constraint name (`<model>_<field>_unique` for a `unique` field, or the declared name).A literal outside `<Model>UniqueField` / `<Model>UniqueConstraint` fails typecheck; a `string` variable compiles.
```typescript
  task.category = "claimed";
  // Refused with DB3_CONDITION_NOT_MET if someone else changed it first:
  await task.save({ condition: { category: "open" }, confirm: true });

  // Create only if no record has this id yet:
  await new Task({ id: "daily-standup", title: "Standup" }).save({ db3: db3Id, ifNotExists: true });

  await task.delete({ condition: { completed: true } });
```

```typescript
  const { views } = await Post.increment(postId, { views: 1 }, { db3: db3Id, confirm: true });
```

```typescript
  const task = await Task.find(taskId);
  if (task) {
    task.tags?.add("urgent");
    task.tags?.remove("low-priority");
    await task.save();
  }
  const urgent = await Task.query({ tags: { $contains: "urgent" } });
```

Vectors: a device writes numbers and reads them decoded; `null` clears. A stored value that doesn't fit the field reads as stored, with one warning. Server functions, the CLI and `client.db3.records(id)` carry the base64 text; a function converts it with `encodeVector(numbers, { dims, element })` and `decodeVector(text, { dims, element })` from `primitive-functions`. `$near` in a server query (record routes, functions, `client.db3.records`) is refused `QUERY_INDEX_NOT_ON_SERVER`.

A model's vector field takes `number[]` or `Float32Array` and reads as `Float32Array`. `encodeVector`, `decodeVector` and `vectorBytes` are exported by the client too.


Batches: `BaseModel.withTransaction(async () => { … })` applies the saves the callback made as one unit per database (the room applies it whole or refuses it whole); `{ atomic: false }` lets each stand alone. "Whole" describes that unit, not the callback: if the callback throws, the saves it already made are not undone — they still land together, and the error rethrows after.

```typescript
  await BaseModel.withTransaction(async () => {
    await new Task({ title: "Plan" }).save({ db3: db3Id });
    await new Task({ title: "Build" }).save({ db3: db3Id });
    await new Task({ title: "Ship" }).save({ db3: db3Id });
  });

  // `atomic: false`: each write stands on its own, so one refusal doesn't
  // undo the others.
  await BaseModel.withTransaction(async () => {
    await new Task({ title: "Optional" }).save({ db3: db3Id });
  }, { atomic: false });
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
await tasks.batch([{ action: "create", data: { title: "a" } }, { action: "increment", id, fields: { n: 1 } }]);
await ctx.db3(input.db3Id, "workspace").batch(
  [{ model: "tasks", action: "patch", id, data: { done: true } }, { model: "posts", action: "create", data: { title: "b" } }],
  { atomic: false },                                                                 // per-op results[]
);
const { reads } = await tasks.batch([{ action: "create", data: { title: "c" } }, { action: "count" }]); // reads[0].count sees the create
```

`patch` (here, over HTTP and from the CLI) changes a record that exists: a missing id is refused `DB3_RECORD_NOT_FOUND` (404) and nothing is created; create with `save`.

Batch writes: `create`, `patch`, `delete`, `upsert` (`constraint`: a `unique` field's name or a constraint name, as `upsertOn` takes; any other name is refused 400 `DOCUMENT_ENTRY_INVALID`, nothing written), `increment`, each with optional `condition` and `requestId`; up to 100 operations per call, reads included; all-or-nothing unless `atomic: false`. Pass a `requestId` (a ULID) on any server write you may retry. A batch may also carry reads (`{ model, action: "query" | "count" | "aggregate", ... }`, taking what `query`, `count` and `aggregate` take): they run in order, each seeing the writes before it, and answer in `reads` at their `index`. A `requestId` retry answers the writes as first applied and the reads again.

### Reading in one request

`batchRead` answers several reads from one state of the database, each at its `index`, with read access only:

```typescript
  const records = client.db3.records(db3Id);
  const { results } = await records.batchRead([
    { model: "projects", action: "query", sort: { updatedAt: -1 }, limit: 20 },
    { model: "tasks", action: "count", filter: { completed: false } },
    { model: "tasks", action: "aggregate", options: { groupBy: ["status"], operations: [{ type: "count" }] } },
  ]);
  // results[0] is the page, results[1] the count, results[2] the rows, each at its index

  // A batch of writes may carry reads too: each sees the writes before it.
  const { reads } = await records.batch([
    { model: "visits", action: "create", data: { page: "/home" } },
    { model: "visits", action: "count" },
  ]);
```

Server function: `await ctx.db3(id, "workspace").batchRead([{ model: "tasks", action: "count" }, ...])`, or `.model("tasks").batchRead([{ action: "count" }])`. CLI: `primitive db3 records batch-read <db3-id> --data-file reads.json` (one line per result; `--partial`, `--json`).

- Up to 100 reads; no read may use another's result.
- The whole answer is at most 8 MiB: a page that would pass it ends early with `hasMore`; a read that still can't fit (a first record, a page with `include`, aggregate rows) is refused `DB3_PAGE_TOO_LARGE` with `details.index`.
- By default the first refused read refuses the request (its code, `details.index`, `details.model`); with `atomic: false` it is answered `{ ok: false, code, status, error }` in its place.
- A batch of reads needs read access; a batch with writes needs write access even if it carries only reads (`records/batch` is refused 403 to a reader).

### Bulk loading

Use a bulk load — not saves or `batch()` — for imports, reference datasets, and any change of thousands of records or more than 1 MiB: one session uploads every row and they apply together.

```typescript
  // rows: [{ model: "securities", id: "AAPL", patch: { name: "Apple Inc.", exchange: "NASDAQ" } }, …]
  const chunks = await client.db3.ingests.buildChunks(rows);
  const session = await client.db3.ingests.create(db3Id, { mode: "merge" });
  for (const chunk of chunks) {
    await client.db3.ingests.uploadChunk(db3Id, session.sessionId, chunk);
  }
  await client.db3.ingests.commit(db3Id, session.sessionId);

  let loaded = await client.db3.ingests.get(db3Id, session.sessionId);
  while (!["complete", "failed", "aborted"].includes(loaded.state)) {
    await new Promise((resolve) => setTimeout(resolve, 1000));
    loaded = await client.db3.ingests.get(db3Id, session.sessionId);
  }
  // loaded.state is "complete", "failed" (loaded.failure says why) or "aborted";
  // loaded.refusedRows counts rows the database refused, refusedSample names a few.
```

CLI: `primitive db3 ingest <db3-id> --input <dir> -y` (one `<model>.ndjson[.gz]` per model, each line an id and a patch), `primitive db3 ingests get|abort <db3-id> <session-id>`. To move a database: `primitive db3 export <db3-id> --output <dir>`, then `primitive db3 import <dir>` (a new id), `--into <db3-id>`, or `--keep-id` (the exported id, which must be free in the target app).

- A row is `{ model, id, patch }`: values replace fields, `{ "_deleted": true }` deletes the record. `null` stores `null` on a plain field (it does not remove the key); on a string-set field it empties the set.
- `mode: "merge"` (default) touches only named records; `mode: "refresh"` also deletes every record of the rows' models no row names.
- Rows that break the type, an enforced enum, a record's value rules (value kinds, field names, vector encoding) or a unique key are skipped and counted in `refusedRows` (`refusedSample` names a few); the load continues.
- A load fills `default`s and `auto_stamp` fields but runs no `triggers`, `autoPopulatedFields` or `timestamps`, and starts no `[[function.triggers.db3]]` run.
- Records appear together at completion; subscribed devices receive them in chunks. App writes made during the load win over conflicting rows.
- `buildChunks` holds all rows in memory — use the CLI beyond a few hundred thousand rows. A load can't be undone, and stopping the CLI's wait doesn't stop it — check `primitive db3 ingests get`.
- A session's `timings.totals.counters` gives per stage `statements`, `rowsRead` and `rowsWritten`, split `byTarget` like a batch's `cost`: a load fills `manifest`, `apply` (the rows), `reconcile` (a refresh's deletions) and `register` (completion).

### From the CLI

For inspecting or fixing data by hand (admin login), not app code: `primitive db3 records query|count|aggregate|save|patch|delete|increment <db3-id> <model> …`, `primitive db3 records batch <db3-id> --data-file <file>` (at most 100 operations per call, atomic: `create` with an optional `id`, `patch`, `increment`, `delete`, `upsert`, each with an optional `condition`, and reads; a larger file is refused before sending — use `primitive db3 ingest`), `primitive db3 records batch-read <db3-id> --data-file <file>`, `primitive db3 schema <db3-id>`, `primitive db3 inspect <db3-id>`.

### Indexes

```typescript
  const index = await client.db3.indexes.register(db3Id, { model: "contacts", field: "email" });
  // index.name === "idx_records_contacts_email"

  const tags = await client.db3.indexes.register(db3Id, { model: "contacts", field: "tags", members: true });
  // tags.name === "idx_members_contacts_tags", tags.members === true

  // Only the database's own queries filter on `createdBy`: devices skip it.
  const byCreator = await client.db3.indexes.register(db3Id, { model: "contacts", field: "createdBy", on: "server" });
  // byCreator.kind === "scalar", byCreator.on === "server"

  try {
    await client.db3.indexes.register(db3Id, {
      model: "contacts",
      fields: ["accountId", "email"],
      unique: true,
      name: "account_email",
    });
  } catch (err) {
    if (isJsBaoApiError(err) && err.code === "DB3_INDEX_NOT_UNIQUE") {
      // Up to ten shared keys, each with how many records hold it.
      console.warn((err.body as { details?: { duplicates?: unknown[] } }).details?.duplicates);
    }
  }

  for (const each of await client.db3.indexes.list(db3Id)) {
    console.log(each.name, each.fields, each.unique, each.kind, each.on, each.origins);
  }
  await client.db3.indexes.drop(db3Id, index.name);
```

Every index costs on every write (server and each synced device), so declare `indexed` only on fields you filter on; adding one later is safe — existing records are indexed when the type is pushed or the index registered. Fields declared `indexed`/`unique` are indexed in every database of the type; register more per database (`primitive db3 indexes register <db3-id> --model <m> --field <f>`). Names: `idx_records_<model>_<field>`; composite (needs `unique` + `name`) `idx_named_<model>_<name>`; declared constraint `idx_uc_<model>_<constraint>`. Errors: `DB3_INDEX_NOT_UNIQUE` (existing duplicates, up to ten in `details.duplicates`), `DB3_INDEX_IN_USE` (dropping a declared index), `DB3_INDEX_SET_FIELD` (a string set field: register it with `members: true`). Registering/dropping needs the owner or an app admin. Every device with the database open takes a registration or drop within moments, without reopening it; one that doesn't have it open takes it at its next subscribe.

**Placement.** `on` places a registered index: `"both"` (default; declared indexes are always `both`), `"server"` (the database's own queries — record routes, server functions; devices skip it), `"device"` (devices' local queries only). Only the holding side pays the write cost; the other side scans. Each listed index carries `kind` (`scalar` | `members` | `text` | `vector`) and `on`. A member index defaults to `on: "server"` and may be placed `"device"` or `"both"`; a text index defaults to `"both"`; a local-only database keeps every index on the device. CLI: `--on <server|device|both>`, `--kind <kind>`. Errors: `DB3_INDEX_EXISTS` with `details.on` (same fields placed elsewhere: drop, then register), `DB3_INDEX_INVALID` (`unique` with `on: "device"`), `DB3_INDEX_PLACEMENT_UNSUPPORTED` with `details.kind`/`details.on` (a kind and placement no side builds), `DB3_LOCAL_ONLY` (`server`/`both` on a local-only database).

**Member indexes.** Unindexed, `$contains` on a string set reads every record's set: fine for thousands, seconds for millions. `{ model, field, members: true }` (CLI `--set-elements`) registers `idx_members_<model>_<field>`, listed with `members: true`: the database answers `$contains`, any-of, membership groups and unfiltered facets on the field from it, a selective `$contains` in milliseconds, answers unchanged. Cost: one stored row per member (about an eighth more storage for two or three short members per record) and a little time on each write that changes the set. Errors: `DB3_INDEX_NOT_SET_FIELD` (a single-value field, `details.field`), `DB3_INDEX_EXISTS` (the field has the other kind of index, `details.index`), `DB3_INDEX_INVALID` (`members` with `fields`, `unique` or `name`). Placed `"device"` or `"both"`, each device's copy builds it too and answers those reads from it; a `"server"` one is the database's alone, and a device's `$contains` reads the records it holds. A local-only database builds it on the device and keeps it on every write.

**Cost.** What the server answers for a database reports what the call cost its storage, in rows: `query`, `count` and `aggregate` answer `cost.rowsRead` (`cost.rowsWritten` 0); a batch answers `cost.rowsRead`, `cost.rowsWritten` and `cost.byTarget` with `records` (records and the field/unique indexes written with them), `members` (member and text indexes), `log` (write ordering, `requestId` marks), `bookkeeping` (other tables, query statistics); registering or dropping an index answers its `cost` too (a build writes a row per record covered, per member for a member index). Compare `cost.rowsRead` before and after registering an index to see what it saves, and a batch's `byTarget.records.rowsWritten` to see what it costs. A batch retried with its `requestId` answers the first answer byte for byte except `cost`, the retry's own (`rowsWritten` 0). A figure is `null` where storage keeps no counter. Measure rows, not server-side time: the server's clock doesn't move during database work. Each server read's rows are also recorded per query shape and caller: `primitive analytics db3-queries` (or `GET /app/{appId}/api/analytics/db3/queries`) ranks an app's most expensive reads; see the [Analytics guide](AGENT_GUIDE_TO_PRIMITIVE_ANALYTICS.md#finding-expensive-database-queries).

`client.db3.records(id)` resolves `cost` (a `Db3Cost`) on `query`, `count`, `aggregate`, `batch` and `batchRead`; `client.db3.indexes.register` resolves the index with `cost` beside its fields and `drop` resolves `{ name, cost }`. A local-only database answers no `cost`.

**Explaining a read.** To check which index a server read uses, explain it: `primitive db3 records query|count|aggregate <db3-id> <model> … --explain`, or from a server function `ctx.api.db3.records.explain({ db3Id, model, body: { op, filter, options, aggregate, index } })` (`op`: `query` | `count` | `aggregate`). The read runs for real and answers `indexes` (index names used), `scanned` (models whose records it walked), `statements` (each with `model`, `plan` lines, `indexes`, `scan`), `answered` and `cost.rowsRead`, not the records. `--index <name>` (body `index`) forces a field or member index of the model; `--index none` reads as if there were none (string-set reads check each record's array). Compare the two by `cost.rowsRead`: `scan` is the access path, not a row count. Needs the owner or an app admin (or a function of the app).

Watch out when explaining:

- A force that cannot serve the read is `DB3_INDEX_CANNOT_SERVE` with `details.reason`: `fields` (a field index on no field the read filters, sorts or groups on), `shape` (a member index the read has no `$contains`, any-of, membership level or unfiltered facet for, or a field index no statement of the read can take, as a facet counted from its member index alone), `plan`, `kind` (text or vector), `placement` (device-only), or `search` (a `$search` under `none`; force a field index instead). An unknown name is `DB3_INDEX_NOT_FOUND`.
- `none` can read, and bill, every record of a large model; an id filter or a page limit can stop it sooner, so `cost.rowsRead` is the measured cost. Explained reads stay out of the query-cost ranking.

Watch out when registering a member index on a large database: it is built from the existing records in that one request, which holds for about 1,000,000 records. Past that the registration can be refused and leaves no index; register the field first, then load, and each write keeps the index.

On the device that calls them, `client.db3.indexes.register` and `drop` change the copy of a database this client has subscribed before they return, so the next query uses the index. At a subscribe, `ready` returns once the copy holds the database's index list; a query made after `available` and before `ready` may still scan.

In development the client logs a `[db3]` warning for a read that scans every record of a model. `devMode: { servedIndexes: true }` also logs, at info, which index served each device read, as the read's plan names it, once per distinct read: `[db3] Note.query read Note in <db3Id> through idx_records_note_year`. `devMode: true` keeps the warnings only.

**On each device.** Each device builds the `"device"`/`"both"` indexes in its own copy, and can leave some out; nothing is sent, and the server and other devices keep them.

```typescript
  await client.db3.subscribe(db3Id).ready;

  for (const index of await client.db3.indexes.local(db3Id)) {
    // "built", "building", "unbuilt" or "declined", with why when not built.
    if (index.status !== "built") console.log(index.name, index.status, index.reason);
  }
  // The entries this device built nothing for, with why: placed on the
  // server only, or an index the device can't build.
  for (const skipped of await client.db3.indexes.skipped(db3Id)) console.log(skipped.name, skipped.reason);

  await client.db3.indexes.decline(db3Id, "idx_text_tasks");
  try {
    await Task.query({ $search: "quarterly report" }, { db3: db3Id });
  } catch (err) {
    // The index is declined here: search on the server, or accept it.
    if ((err as { code?: string }).code !== "DB3_INDEX_DECLINED_ON_DEVICE") throw err;
  }

  const index = await client.db3.indexes.accept(db3Id, "idx_text_tasks");
  // index.status === "built"
```

- `client.db3.indexes.local(db3Id)` lists each device-placed index with `status` (`built` | `building` | `unbuilt` | `declined`) and `reason`; the subscription state's `indexes` carries the same list and its state event fires on a change.
- `client.db3.indexes.skipped(db3Id)` lists the entries this copy built nothing for, each with `reason`: placed `"server"` only, an index on `id` (the primary key serves it), a name that isn't an identifier, a member or vector index over more than one field, unreadable text options. `local` lists none of them.
- A failed build (e.g. full storage) leaves the index `unbuilt`: the copy keeps syncing, its queries scan, and each sync retries.
- `decline` drops the index from this copy and keeps it declined across restarts, evicts, and a drop and re-registration, until `accept` (which rebuilds it) or `client.db3.delete`. A device `$search` needing a declined text index (or `$near` a declined vector index) is refused `DB3_INDEX_DECLINED_ON_DEVICE` (`details.model`, `details.index`); other queries scan. Unknown names: `DB3_INDEX_NOT_FOUND`; local-only databases: `DB3_LOCAL_ONLY`.
- Client option `deviceIndexes`: `"all"` (default), `"none"`, or kinds such as `["scalar", "members"]` (Swift: `.all`, `.none`, `.kinds([...])`); left-out indexes list `declined`, and indexes already built stay built.

### Text search

A text index (`kind: "text"`, one per model, `idx_text_<model>`) answers `$search` where it is placed: `on: "both"` (default) on each device holding the database and in the record routes and server functions, `"device"` or `"server"` on that side only. Matches come back ranked, each with `_score` (higher is better), best first; the server and a device rank the same records identically.

```typescript
  await client.db3.indexes.register(db3Id, {
    model: "tasks",
    fields: ["title", "category"],
    kind: "text",
    on: "both",
    text: { weights: { title: 2 } },
  });

  // On the device, over the copy it holds.
  const page = await Task.query({ $search: "quarterly report" }, { db3: db3Id });
  for (const task of page.data) console.log(task.title, task._score);

  // On the server, over every record.
  const found = await client.db3.records(db3Id).query("tasks", { filter: { $search: "quarterly report" }, limit: 20 });
  for (const row of found.items) console.log(row.title, row._score);

  // A phrase, any of the words, and the title alone.
  await Task.query({ $search: '"quarterly report"' }, { db3: db3Id });
  await Task.query({ $search: { query: "report review", mode: "any", fields: ["title"] } }, { db3: db3Id });

  // The start of a word: "repo" finds "report".
  const started = await Task.query({ $search: { query: "repo", prefix: true } }, { db3: db3Id, sort: { _score: -1 } });
```

| Option | Values |
|---|---|
| `fields` | one or more string fields; a non-string value isn't indexed |
| `text.tokenizer` | `unicode61` (default; case and diacritics fold) or `porter` (English stems) |
| `text.prefix` | word-start lengths to index (1–32, up to 8) for fast `prefix` searches |
| `text.weights` | a positive weight per field, 1 by default |
| `$search` | a string, or `{ query, mode: "all" \| "any", prefix, fields }`; a double-quoted run is a phrase |

`$search` goes at the top level of a filter (AND with the rest); without `sort`, order is `_score` then id; `sort` may name `_score`. `count` and an aggregate's `filter` take it too. Input is matched as plain words: pass search-box text as it is. CLI: `--kind text --tokenizer porter --prefix 2,3 --weight title=2`; `db3 records query <id> <model> --filter '{"$search":"…"}'` searches on the server.

Watch out when searching:

- A side holding no text index for the model refuses `$search`: `QUERY_INDEX_NOT_ON_SERVER` (record routes, server functions) or `QUERY_INDEX_NOT_ON_DEVICE` (a device), with `details.model` and, when the index lives on the other side only, `details.index`.
- The server builds its copy within the registration, for up to 2,700,000 records of the model; above that the registration is `DB3_INDEX_TOO_LARGE` (`details.model`, `details.rows`, `details.limit`). Register before loading the records, or place it `on: "device"`.
- A web copy needs `databaseConfig: { type: "opfs" }`: on another engine `$search` is refused `DB3_INDEX_PLACEMENT_UNSUPPORTED` with `details.engine` (every other query answers), and a local-only database there refuses the registration that way.
- A second text index on a model is `DB3_INDEX_EXISTS`; a text index takes no `unique` or `name`.

### Vector search

A vector index (`kind: "vector"` on one vector field, `idx_vec_<model>_<field>`, always `on: "device"`) answers `$near` on each device holding the database: the `k` nearest records, nearest first, each with `_distance`.

```typescript
  await client.db3.indexes.register(db3Id, {
    model: "notes",
    field: "embedding",
    kind: "vector",
    vector: { distance: "cos", index: { mode: "ann" } },
  });

  const { data } = await Note.query(
    { $near: { field: "embedding", vector: query, k: 10, filter: { pinned: false } } },
    { db3: db3Id }
  );
  for (const note of data) console.log(note.body, note._distance);

  // "exact" from the first record, "ann" once this device has the shared model.
  const index = (await client.db3.indexes.local(db3Id)).find((i) => i.name === "idx_vec_notes_embedding");
  console.log(index?.mode);

  // The owner has the model trained again, over the records as they are now.
  await client.db3.indexes.deleteModel(db3Id, "idx_vec_notes_embedding");
```

| Option | Values |
|---|---|
| `vector.distance` | `cos` (default; `_distance` is the cosine distance) or `l2` (`_distance` is the squared Euclidean distance) |
| `vector.index` | `"exact"` (default: every vector compared) or `{ mode: "ann", nbucket?, codesize?, quantizer? }` (approximate, reranked on the full vectors; leave the knobs at their defaults) |
| `$near.field` / `$near.vector` | the vector field; an array of its `dims` numbers |
| `$near.k` | 1 to 10,000, default 10 |
| `$near.filter` | conditions on the model's other fields (no `$near`, no `$search`) |

`$near` is the filter's only key. An `ann` index answers exactly until the device has the database's model and has filled its index, so no query waits. One device that holds at least 1,000 records trains the model in the background (off the thread serving queries) and uploads it; the others fetch it. `client.db3.indexes.list` shows it as `annModel` (`revision`, `rows`, `trainedAt`, `bytes`) beside the index's `generation` and `revision`; `client.db3.indexes.local` shows each device's `mode` (`exact` | `ann`). The owner's `deleteModel` moves the revision so a device trains it again; devices answer exactly meanwhile. `fetchModel` reads the model's bytes and `uploadModel` uploads one trained elsewhere (`DB3_INDEX_MODEL_EXISTS` once one is listed, `DB3_INDEX_GENERATION_STALE`/`DB3_INDEX_MODEL_STALE` after a re-registration or a delete). A local-only database trains and keeps its own model. CLI: `--kind vector --distance cos --index ann`; `db3 indexes model get|delete`.

Watch out when searching by vector:

- A device's store must build vector indexes: `databaseConfig: { type: "opfs" }` on the web, and the optional `primitive-vec1-node` package in Node. Elsewhere `$near` is refused `DB3_INDEX_PLACEMENT_UNSUPPORTED` with `details.engine` (everything else answers), and a local-only database refuses the registration that way.
- A field with no vector index on the device is `QUERY_INDEX_NOT_ON_DEVICE`; a registration on a field that isn't a vector field is `DB3_INDEX_INVALID`; the server's record routes and functions refuse `$near` with `QUERY_INDEX_NOT_ON_SERVER`.
- Storage on each device: a 32-bit copy of each vector (4·dims bytes, whatever the `element`), and a little more again for an `ann` index. Per 100,000 records with 768 numbers: records about 445 MiB (`f32`), 397 MiB (`f16`) or 136 MiB (`i8`), plus about 295 MiB for the vector index and 413 MiB for an `ann` index. A shared model upload is at most 64 MiB.
- Type the database before loading embeddings: an existing field can't be declared `vector` while records hold values for it (`DB3_TYPE_REFUSED`). A CLI import of an export holding a vector index needs `--type <type>` or `--into` a typed database.
- Wide vectors in the browser: an exact `$near` over 20,000 768-number vectors takes about 170 ms there; register `index: { mode: "ann" }` for them.

### Limits

| Limit | Value |
|---|---|
| Page size | default 100, max 10,000 (`DB3_PAGE_TOO_LARGE` above; never clamped) |
| Page bytes | 8 MiB of records: a plain page ends early with `hasMore: true`; a page whose `_related` would pass it is refused with `DB3_PAGE_TOO_LARGE` |
| Include depth | 3 |
| `hasMany` related rows per request | 1,000 (`DB3_PAGE_TOO_LARGE`) |
| `$in` / `$nin` values | 1,000 (`QUERY_IN_LIST_TOO_LARGE`) |
| Bound values per query | 100 (`QUERY_FILTER_TOO_MANY_BINDS`) |
| Batch operations | 100 per call |
| One write / batch | 1 MiB JSON (`DOCUMENT_ENTRY_TOO_LARGE`) |
| Field names per model | 1,000 |

## Synced databases on a device

```typescript
  const sub = client.db3.subscribe(db3Id);
  sub.on("state", (state) => console.log(state.phase, state.progress));

  // A readable copy: at once when this device already holds one, else after
  // the first load.
  await sub.available;
  const open = await Task.query({ completed: false }, { db3: db3Id });

  // `ready`: caught up with every change made elsewhere.
  await sub.ready;

  // Unsubscribed, the copy still answers queries and takes writes; they are
  // sent at the next subscribe.
  sub.dispose();
  await new Task({ title: "Write the report" }).save({ db3: db3Id });

  const state = await client.db3.state(db3Id);
  console.log(state.resident, state.pendingWrites, state.lastSyncAt);
```

| Call | Does |
|---|---|
| `client.db3.subscribe(id)` | Handle with `available` (readable copy on device), `ready` (caught up, and the user's permission known), `state`, events, `dispose()`. `ready` rejects `DB3_READY_TIMEOUT` (30 s without progress; `subscribe(id, { readyTimeoutMs })`, Swift `Db3SubscribeOptions(readyTimeout:)`), `DB3_ACCESS_REFUSED`, `DB3_SYNC_OFF`, `DB3_DELETED` (the database was deleted), `DB3_DISPOSED` (the subscription ended first), `NOT_A_DB3` |
| `client.db3.unsubscribe(id)` | Disposes every handle for the database |
| `client.db3.refresh(id)` | Subscribe → caught up → dispose |
| `client.db3.state(id)` | `{ resident, subscribed, phase, progress, rows, pendingWrites, lastSyncAt, permission, error, drift }` without subscribing |
| `client.db3.stored()` | Databases this device holds for the signed-in user |
| `client.db3.evict(id, { force })` | Delete the local copy; `DB3_PENDING_WRITES` unless `force` |
| `client.db3.setRetentionPolicy(...)` | Evict automatically (session-only, `maxDocs`, `ttlMs`); never a subscribed copy or one with unsent writes |

`phase`: `idle`, `opening`, `loading`, `catching-up`, `live`, `reconnecting`, `gone`. Events: `db3:state`, `db3:ready`, `db3:rows` (`changes`, `origin`: `local` this device's own write, applied before the database has seen it \| `sibling` another client on the same local store \| `remote` the database changing the local copy — a change from elsewhere, or the database correcting/reverting this device's own write), `db3:write-refused`, `db3:evicted`, `db3:gone` (`reason`: `permission`\|`sync-off`\|`client-outdated`\|`deleted`), `db3:error`.

A refusal is remembered on the device: the copy refuses writes (`WRITE_ACCESS_DENIED`) in every tab and after a relaunch. A grant heard while a subscription that was live still holds the copy asks the database again at once, and the subscription returns to `live`; writes are taken again from the grant on, and a `confirm: true` write made meanwhile or waiting across the change resolves once the database accepts it. Otherwise the next subscribe there asks the database again; `ready` resolves if access was restored, and rejects with the database's own reason if not. A database already gone for another reason that is then deleted emits `db3:gone` again, once, with `deleted` (at the latest on the next subscribe).

`client.db3.delete` ends the deleting client's own subscriptions as evicted (`db3:evicted`; a pending `ready` rejects `DB3_DISPOSED`); the other subscribed clients get `db3:gone` with `deleted`.

**Drift.** A device page's `drift` (and the state's) says how far behind the copy it read is, by database id: `entriesBehind` (a lower bound, as of `asOf`, when the device last heard the server's position), `live`, `pendingWrites`. `maxDrift: { entries, ageMs }` (Swift `Db3MaxDrift(entries:age:)`, seconds) on `query`, `queryOne` and `count` refuses a copy further behind with `DB3_DRIFT_EXCEEDED` (`details`: `db3Id`, `drift`, `maxDrift`); it never waits. A database the device hasn't synced with yet has no `drift` and refuses every `maxDrift` (`details.drift: null`); a local-only one has none and passes.

```typescript
  try {
    const page = await Task.query(
      { completed: false },
      { db3: db3Id, maxDrift: { entries: 100, ageMs: 5 * 60_000 } }
    );
    const drift = page.drift?.[db3Id];
    if (drift && !drift.live) {
      console.log(`${drift.entriesBehind} changes behind as of ${new Date(drift.asOf).toLocaleString()}`);
    }
    return page.data;
  } catch (error) {
    if (!(error instanceof Db3DriftExceededError)) throw error;
    // Too far behind, or never synced on this device: catch up first.
    await client.db3.refresh(db3Id);
    return (await Task.query({ completed: false }, { db3: db3Id })).data;
  }
```

A resident copy answers reads and takes writes while unsubscribed or offline; writes queue and send at the next subscribe, in order. `db3.create` for the user's own synced database (no `custodian`) resolves at once, online or offline, with `pendingCreate` true in the state, and the client saves it under the same id in the background (`db3:created`, or `db3:create-failed` with its `id`, keeping its records). A refusal only the server decides (`DB3_TYPE_NOT_FOUND`, `DB3_TYPE_REFUSED`) arrives as that event, never as a rejection of `create`. A create overlapped by a change of signed-in user (another user's sign-in, a sign-out) rejects `INVALID_ARGUMENT` with the reason `identity-changed` and keeps nothing; create again under the new user. Await `commitOfflineCreate` with the id to get the server's answer (`created`, or `refused` with the reason); `client.db3` calls that ask the server about the database wait for the commit on their own, rejecting `PENDING_CREATE` offline or after 30 s. A device may write offline for 7 days by default (`largeDocumentWindowDays` in `app.toml` `[app]`, 1–14).

`setRetentionPolicy` with `maxDocs` keeps at most that many of the copies `stored()` lists, subscribed ones included, evicting the least recently used first (`db3:evicted` with `reason: "retention"`). A copy's last use is the later of its last open and the end of its last subscription (a dispose, a sign-out, or the client shutting down). A subscribed copy, one with unsent writes and one still being created (`pendingCreate`) are never evicted and still count, so the device may keep more. `maxBytes` never removes a database copy. `maxDocs` is a positive integer; leave it unset for no cap. `0`, a negative or a non-integer is refused with `INVALID_ARGUMENT`, and the policy stays as it was.
`ttlMs` evicts copies unused for that long: a positive number of milliseconds, a fraction included; leave it unset for no TTL. `0`, a negative or a non-finite `ttlMs` is refused the same way. A copy another tab holds is kept and counts too. The policy merges into the one set before, so pass `maxDocs: undefined` or `ttlMs: undefined` to lift an earlier limit.

**Local-only databases.** `client.db3.create` with `localOnly: true` keeps a database on this device only: nothing of it is ever sent, now or in a later session. Writes are judged on the device with the database's own codes (unique keys, upserts, conditions, increments) and accepted at once, so `pendingWrites` stays 0 and `confirm: true` resolves at once. A subscription is `live` at once, `stored()` lists it with `localOnly: true`, `client.db3.indexes` works on the device, and it survives restarts for the same user.

```typescript
  const db = await client.db3.create({ title: "Scratchpad", localOnly: true });

  // Saves, queries and live updates work as on any database, at once.
  await new Task({ title: "Sketch the layout" }).save({ db3: db.id });
  const open = await Task.query({ completed: false }, { db3: db.id });

  // It is listed with the other databases this device holds.
  const local = (await client.db3.stored()).filter((stored) => stored.localOnly);

  // Removing it removes its only copy.
  await client.db3.delete(db.id);
```

| Refused | Code |
|---|---|
| `create` with `type`, `sync: false` or `custodian` | `LOCAL_ONLY_UNSUPPORTED_OPTION`, before anything is created |
| A `client.db3` call that asks the server about it (`get`, `permissions`, `setSync`, `setType`, `blobs`, `ingests`, …) and `evict` | `DB3_LOCAL_ONLY` (`details.db3Id`), before any request |

Only `client.db3.delete(id)` (nothing sent; `DB3_HELD` while another tab holds it) and `logout({ wipeLocal: true })` remove it. `evictAllLocal` and the retention policy keep it, and it counts toward no `maxDocs` cap.

A `DB3_DISPOSED` rejection carries a `reason` in its details saying why the subscription ended: `disposed` (`dispose()`), `destroyed` (the client was destroyed; `subscribe` on a destroyed client throws the same), `identity-changed` (a sign-out, or another user signed in: subscribe again under the new user) or `evicted` (the local copy was removed).

**Storage on the web.** A web client keeps its databases in the browser's origin-private file system through the `opfs` engine: a worker, a broker that shares one copy across the app's tabs, and two wasm files. The Vue template ships this in `src/config/storage.ts`; any other web app writes the same module and passes `databaseConfig: getDatabaseConfig()` to the client:

```typescript
// src/config/storage.ts in the Vue template
import type { DatabaseConfig } from "js-bao-wss-client";
import workerURL from "js-bao/format2-worker?worker&url";
import brokerURL from "js-bao/format2-broker?url";
import sqliteWasmURL from "js-bao/sqlite3.wasm?url";

function locateFile(file: string): string {
  if (file === "sqlite3.wasm") return sqliteWasmURL;
  if (file === "sql-wasm.wasm") return "/sql-wasm.wasm"; // copied into public/
  throw new Error(`locateFile: no URL for ${file}`);
}

export function getDatabaseConfig(): DatabaseConfig {
  return { type: "opfs", options: { workerURL, brokerURL, locateFile } };
}
// Pass it to the client: initializeClient({ ..., databaseConfig: getDatabaseConfig() })
```

Node has two engines. `databaseConfig: { type: "sqljs", options: { locateFile } }` keeps the database in memory with no native module to install — for tests and short-lived scripts; the Vue template's `pnpm test` client uses it, with `locateFile` pointing at the `sql-wasm.wasm` its `pnpm codegen` copies into `public/`. `databaseConfig: { type: "node-sqlite", options: { recordLogDirectory } }` needs `better-sqlite3` installed beside js-bao and keeps the copy in that directory across restarts: a restarted process resumes from it and still sends its unsent writes. Without suitable storage the client reports `FORMAT2_STORAGE_UNAVAILABLE`. The browser's answer to the persistent-storage request is `persisted`, `granted`, `declined` or `unsupported`; on the last two it may clear the copy under storage pressure.

Watch out when setting up web storage:

- `brokerURL` is started as a classic script: pass the bundle's URL (`js-bao/format2-broker?url`), never a module or a constructor import. To start a module SharedWorker yourself, pass `spawnBroker: () => new SharedWorker(url, { type: "module" }).port` instead. With neither, a second tab of the same database is refused.
- `locateFile` is asked for exactly two names, `sql-wasm.wasm` and `sqlite3.wasm` (import it from `js-bao/sqlite3.wasm`): answer each by name and throw on any other. A name passed through resolves to the app's `index.html`, which fails to load as wasm.
- Import the worker and broker as URLs (`?worker&url`, `?url`), which a Node test run imports as plain strings. Keep `?worker` and `?sharedworker` constructor imports out of any module a test imports: importing one starts a worker.
- Serve the `sql-wasm.wasm` from the `sql.js` package js-bao installed, and copy it again when that version changes: a wasm from another `sql.js` version fails startup with `sql.js wasm mismatch`. The template's `pnpm codegen` copies it into `public/`.

### Re-rendering on change

Every generated model has `subscribe(callback)` → unsubscribe function; it fires on local and remote changes. Re-run the query in the callback.

In the Vue template, `useJsBaoDataLoader` (imported from `@/composables/useJsBaoDataLoader`) wraps this: it re-runs `loadData` when the subscribed models change.

```typescript
  const { data: tasks, showSkeleton } = useJsBaoDataLoader<Task[]>({
    subscribeTo: [Task],
    queryParams: computed(() => ({ db3Id: props.db3Id })),
    documentReady: databaseReady, // true once the subscription's `available` resolves
    async loadData(params) {
      const result = await Task.query({ completed: false }, { db3: params!.db3Id, sort: { priority: -1 } });
      return result.data as Task[];
    },
  });
```

### Concurrent edits

Different fields of a record both survive; the same field converges on one write everywhere; string-set members and increments all count; a unique key goes to the first write the database accepts (an upsert merges into it); conditions are checked by the database.

### Write refusals

A device write (save, patch, delete, increment) applies to the local copy and resolves before the database has seen it. The device refuses three writes itself, before anything is sent: a write while the user holds `reader`, or before the device has learned their level (a first subscribe, until `ready`), and, once the device has opened a typed database, a model the type doesn't declare (`DB3_MODEL_NOT_DECLARED`), a value failing a field's declared kind, e.g. `""` for a `date` (`DOCUMENT_ENTRY_INVALID`, the database's own message), or a value outside an enforced enum (`DB3_VALUE_NOT_IN_ENUM`). The database remains the authority for every other refusal. If the database refuses a write at sync, the record reverts and `db3:write-refused` carries `writes` (every affected write, in order, to redo), `error.code`, and `conflictingRecord`/`current` where relevant. Pass `confirm: true` (Swift `Db3WriteOptions(confirm: true)`) on the write instead to wait for the database's answer at the call site and reject there if it's refused, rather than handling `db3:write-refused` afterward.

```typescript
  client.on("db3:write-refused", (event) => {
    const first = event.writes[0];
    if (event.error.code === "DOCUMENT_CONSTRAINT_VIOLATED" && event.conflictingRecord) {
      ui.showConflict(first.model, first.recordId, `${event.conflictingRecord.id} already holds that key`);
      return;
    }
    ui.showConflict(first.model, first.recordId, event.error.message);
  });
```

| Code | Cause |
|---|---|
| `DOCUMENT_CONSTRAINT_VIOLATED` | Unique key taken by another device first |
| `DB3_CONDITION_NOT_MET` | Condition no longer held (`current`) |
| `DB3_TRIGGER_REFUSED` | A type rule refused (`error.details.reason`: the rule's code, or `TRIGGER_EVALUATION_FAILED` when its `when` couldn't be evaluated) |
| `DB3_MODEL_NOT_DECLARED` | Model not in the database's type |
| `DB3_VALUE_NOT_IN_ENUM` | A value outside a field's enforced enum (refused on the device first; an app on an older client library gets `DOCUMENT_ENTRY_INVALID`) |
| `WRITE_ACCESS_DENIED` | User lost write access; also thrown on the device for a write to a copy the user may only read |
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

```typescript
  const permissions = client.db3.permissions(db3Id);

  // By email — works whether or not the person is a member of the app yet
  await permissions.set({ email: "colleague@example.com", permission: "read-write" });

  // With a group
  await permissions.grantGroup({ groupId: "engineering", permission: "reader" });

  // Who has access, and who hasn't accepted yet
  const members = await permissions.list();
  const pending = await permissions.listPendingInvitations();

  // Take it away
  await permissions.remove("user-abc");
```

```typescript
  const permissions = client.db3.permissions(db3Id);
  await permissions.setLinkAccess("reader");
  const { linkAccess } = await permissions.getLinkAccess(); // "reader" | "read-write" | null
  await permissions.clearLinkAccess();
```

With `sendEmail: true`, each new grantee gets an email; its link is `documentUrl` when you pass one, otherwise the app's base URL. Granting by email with `sendEmail: true` needs the app's base URL set (the invitation links to it). Grants by email work before the person joins (pending invitation): `listPendingInvitations()` lists them and `revokePendingInvitation(invitationId)` cancels one on this database only (the invitation and its other shares stay). Group grants follow group membership. Removing a grant turns the user's subscription `gone` at once: the local copy stays readable, new writes are refused `WRITE_ACCESS_DENIED`, and queued writes stay until access returns or you evict.

### Access requests

`client.db3.permissions(id).requestAccess({ permission })`lets a member ask the owners; an owner calls `listAccessRequests()`, `approveAccessRequest(requestId)` or `denyAccessRequest(requestId)`. Every owner hears`db3:access-request-created`on a new or re-submitted request; the requester and every owner hear`db3:access-request-resolved`when any owner resolves it. Events cover every database the user owns (filter by `documentId`) and also arrive under `document:` names; use the `db3:` ones. Keep a pending list without polling: list on every connect (the client reports connected once these events reach it, so nothing falls between the list and the events), apply events at once, and while a list call runs keep the events in a buffer of its own and replay them over its answer, dropping an answer a newer call superseded.
```typescript
  type AccessRequestEvent = AccessRequestCreatedEvent | AccessRequestResolvedEvent;
  const pending = new Map<string, string>(); // requestId → requesterId
  let generation = 0;
  let buffered: AccessRequestEvent[] | null = null; // events seen while the latest list call runs

  const apply = (e: AccessRequestEvent) => {
    if (e.status === "pending") pending.set(e.requestId, e.requesterId);
    else pending.delete(e.requestId);
  };

  const onEvent = (e: AccessRequestEvent) => {
    if (e.documentId !== db3Id) return; // events come for every database you own
    apply(e);
    buffered?.push(e);
    onChange(pending);
  };

  const refresh = async () => {
    const mine = ++generation;
    // A fresh buffer per call: events heard before it started are already in its answer.
    const kept: AccessRequestEvent[] = [];
    buffered = kept;
    let rows;
    try {
      rows = await client.db3.permissions(db3Id).listAccessRequests();
    } catch (err) {
      if (mine === generation) buffered = null;
      throw err;
    }
    if (mine !== generation) return; // a newer refresh started; its answer wins
    pending.clear();
    for (const row of rows) pending.set(row.requestId, row.requesterId);
    for (const e of kept) apply(e); // replay what arrived meanwhile
    buffered = null;
    onChange(pending);
  };

  client.on("db3:access-request-created", onEvent);
  client.on("db3:access-request-resolved", onEvent);
  // Nothing is replayed while the socket is closed: list again on every connect.
  client.on("status", ({ status }) => {
    if (status === "connected") refresh().catch(console.error);
  });
  refresh().catch(console.error);
```

### App access

**App access** — every current and future member: `client.db3.setAppAccess(id, "reader" | "read-write" | null)` (app admin), `primitive db3 app-access set <db3-id> reader|read-write|none`, or in a function `ctx.api.db3.setAppAccess({ db3Id, body: { level } })` with capability `db3:setAppAccess`. Content only at its level (`reader` reads, `read-write` writes; no share/delete). Refused on a sync-off database.

### Checking access

Device: `client.db3.get(id)` → `{ …, permission, accessSource }` for the signed-in user. Function: `auth.db3(id, type).permissions()` in `authorize` (caller's level or `null`); `ctx.api.db3.validateAccess({ db3Id, body: { userId } })` → `{ hasAccess, permission?, accessSource? }` for any member (always pass `userId`; omitted, it describes the function's app authority). An app role is not a content grant. Sync-off databases answer no access for members.

### Collections

A collection is a set of databases shared together: its members reach every database in it, including ones added later.

```typescript
  // Create a collection and put databases in it
  const collection = await client.collections.create({ name: "Project Phoenix" });
  await client.collections.addDocument(collection.collectionId, designDb3Id);
  await client.collections.addDocument(collection.collectionId, specDb3Id);

  // One grant covers the whole set — including databases added later
  await client.collections.addMember(collection.collectionId, {
    email: "alice@example.com",
    role: "editor",
  });
```

```typescript
  // Create
  const collection = await client.collections.create({
    name: "Q1 Reports",
    description: "All quarterly report databases",
  });

  // Add / remove a database
  await client.collections.addDocument(collection.collectionId, db3Id);
  await client.collections.removeDocument(collection.collectionId, db3Id);

  // Share with a group (reaches every database in the collection)
  await client.collections.addGroupMember(collection.collectionId, {
    groupId: "engineering",
    role: "editor",
  });

  // Share with an individual user. Also accepts { email, ... } for a
  // deferred grant that resolves on signup, exactly like databases and groups.
  await client.collections.addMember(collection.collectionId, {
    userId: targetUserId,
    role: "viewer",
  });
```

```typescript
  const { items: members } = await client.collections.listMembers(collectionId);

  const pending = await client.collections.listPendingInvitations(collectionId);
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

Every database has a `custodian`: `user:<userId>` (holder and owner; a member's own creates, a function's creates for its caller) or `app` (no owner; created from the CLI's admin token with no custodian, by an app admin naming `custodian: "app"`, or by a function naming it with capability `db3:transfer`). Transfer: owner → another member (old owner keeps `read-write`); to/from the app needs an app admin (`primitive db3 transfer <db3-id> app`) or `ctx.api.db3.transfer({ db3Id, body: { to } })` with `db3:transfer`. A transfer to the app removes the previous owner's grant (they keep only group or app access; their subscriptions turn `gone`, `permission`); a grant made right after it reopens their live subscriptions on its own. When a member leaves the app, databases they hold pass to the app if another user holds an accepted grant, any group is granted, the database belongs to a collection (above), or app access is set (link access and unaccepted invitations don't count); otherwise they are deleted. List by holder: `primitive db3 list --custodian app|user:<id>`; every database a member can open: `primitive db3 list --user-id <id> [--via held,shared,app]` (one row per `via`, sync-off databases included).

`client.db3.listMine()` lists the databases the signed-in user can open: held, shared with them, then app access, each entry naming its listing as `via`. Pass `via` to read only some; page with `cursor`. A sync-off database is not listed (the admin listing still shows it).

**Renaming.** `client.db3.update(id, { title })` sets the title `get` and `listMine` answer, and resolves with the database as `get` does. Its owner, a `read-write` grant or link, and app owners and admins rename it; anyone else is refused 403 `DOC_ACCESS_DENIED`. The title is stored trimmed (blank or over 255 characters: 400). Connected devices of direct-grant holders get the new title at once (`stored()` entries carry `title`); others see it at their next read. A local-only database is renamed on the device, nothing sent (`INVALID_ARGUMENT` for a blank or long title). CLI: `primitive db3 update <db3-id> --title <title>`; function: `ctx.api.db3.update({ db3Id, body: { title } })`, no capability. Don't rename through `documents.update`.

```typescript
  const db = await client.db3.update(db3Id, { title: "Rivera household" });
  console.log(db.title, db.modifiedAt);
```

```typescript
  const ids = new Set<string>();
  let page = await client.db3.listMine();
  for (;;) {
    for (const db of page.items) ids.add(db.id); // db.via: "held" | "shared" | "app"
    if (!page.hasMore) break;
    page = await client.db3.listMine({ cursor: page.nextCursor });
  }

  // Only the ones shared with this user:
  const shared = await client.db3.listMine({ via: ["shared"] });
```

A server function lists a named member's databases with `ctx.api.db3.listHeldByUser({ userId })` and `ctx.api.db3.listSharedWithUser({ userId })` (the `held` and `shared` entries, sync-off databases included in the held one; filter with `tag` and `type`, and keep following `nextCursor` past short pages), and resolves, binds and removes their aliases with `ctx.api.db3.aliases.resolve({ scope: "user", aliasKey, userId })`, `set({ scope: "user", aliasKey, body: { documentId, userId } })` and `delete({ scope: "user", aliasKey, userId })`. See the [Server Functions guide](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#a-members-collections-databases-and-profile).

### Aliases

An alias is a per-user key (`scope: "user"`) that names one database. `getOrCreateWithAlias` finds or creates the database behind it; `client.db3.aliases` binds a key to an existing database (`set`), reads it back (`resolve`: the database's row with `alias`, or`null`when the key names nothing) and removes it (`delete`; the database stays, and a missing key is not an error).

```typescript
  await client.db3.aliases.set({ scope: "user", aliasKey: "archive", documentId: db3Id });

  // The database's row and the alias, or null when the key names nothing.
  const archive = await client.db3.aliases.resolve({ scope: "user", aliasKey: "archive" });

  // Removes the key; the database stays.
  await client.db3.aliases.delete({ scope: "user", aliasKey: "archive" });
```

- `userId` defaults to the signed-in user; only an app owner or admin may name another member (`403` otherwise).
- Keys are case-insensitive: they are stored lowercased.
- A key names only a database: `set` naming any other id throws `409 NOT_A_DB3`, and so do `resolve` and `delete` of a key bound to one; an id that is no database of the app throws `404 DB3_NOT_FOUND`.
- `set` re-points an existing key; pass `mustNotExist: true` to get `409 alias_exists` instead.

### A per-user database the user can only read

A function creates (or finds) a user's database with that user as its only reader in one call: `permission: "reader"` on `ctx.api.db3.getOrCreateWithAlias` writes no owner row and one `reader` grant for the alias's user, who never holds write access. The function declares `db3:setOwnerRow`. Fill its records from a function (`ctx.db3(id, type)` without `asUser`) or with a console-admin credential: the database is synced and user-held, so an app owner or app admin with no grant on it is refused `403` on its records until granted `read-write`.

```toml
# primitive/dev/functions/open-inbox.toml
[function]
key = "open-inbox"
entry = "functions/open-inbox/index.ts"
capabilities = ["db3:setOwnerRow"]
```

```ts
import { defineFunction } from "primitive-functions";

export default defineFunction("open-inbox", {
  handler: async (_input, ctx) => {
    const inbox = await ctx.api.db3.getOrCreateWithAlias({
      body: {
        alias: { scope: "user", aliasKey: "inbox", userId: ctx.user!.userId },
        title: "Inbox",
        type: "inbox",
        permission: "reader",
      },
    });
    return { id: inbox.id };
  },
});
```

`permission` (`owner` by default, `read-write`, `reader`) applies only when the call creates the database. Without it, the alias's user holds the database as owner, whoever calls.

An app owner or admin can make the same call from a client; a member passing a level other than `owner` is refused `403`. A `reader` database the caller creates for themselves is read-only on their device at once: a save is refused there and nothing is sent.

```typescript
  const inbox = await client.db3.getOrCreateWithAlias({
    alias: { scope: "user", aliasKey: "inbox", userId },
    title: "Inbox",
    type: "inbox",
    permission: "reader",
  });
```

### Sync off

`sync: false` at create (`primitive db3 create … --sync off`) or `client.db3.setSync(id, false)` / `primitive db3 sync set <db3-id> off` / `ctx.api.db3.setSync({ db3Id, body: { sync: false } })` (capability `db3:setSync`); app admins and app owners only. Effects: subscribers dropped (`db3:gone`, `sync-off`), socket opens refused `DB3_SYNC_OFF`, every user content action refused (the owner keeps delete and transfer), left out of every member's listing (`listMine`, the group and collection listings), grants suspended, new grants refused `DB3_SYNC_OFF_NO_SHARING`, app access refused. Functions and admins keep full read/write. Flipping back restores grants; a flip never rebuilds data.

Serve a sync-off database through functions that check the caller and filter:

```ts
import { allow, defineFunction, deny } from "primitive-functions";

export default defineFunction("class-grades", {
  input: ["classDb"],
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
- **The first subscribe downloads the whole database** — show `state.progress`; keep databases partitioned so a device holds only what its user needs. An interrupted download (reload, closed tab or app, dropped connection) continues where it stopped at the next subscribe while the database still keeps that download, even if more data arrived since, so `progress` may start above 0; otherwise it starts over.
- **Unsent writes live only on the device** — surface `state.pendingWrites`.
- **`entriesBehind: 0` offline is not current** — it is what the device last heard; when `live` is false, judge by `asOf` or `maxDrift.ageMs`.
- **A local-only database has no other copy** — cleared app storage, a lost device or a wiping sign-out loses it for good; keep in one only what can be lost or rebuilt.
- **Codegen checks a `"<file>.<Name>"` shared-enum ref against `schemas/` and `db3-types/` keys only** — write the long form when a function or prompt shares the file's key, or push refuses what codegen accepted.
- **One declaration per model.** A model two types hold is declared in one file and named by the other with `[type] models`; a second copy drifts, and push and codegen refuse two that differ.
- **A new database's uploads wait for the server to have it** — an upload right after `create` is queued (`bytesTransferred: 0`) and sent once the create commits; await `commitOfflineCreate` first to upload at once.
- **`confirm: true` on a device write waits for the next subscribe** when the database isn't subscribed.
- **Upgrade client libraries forward only.** A local copy written by a newer client doesn't open in an older one; reload web tabs left open across a deploy.
- **Generated fields are optional in practice** — older records may lack newer fields.
- **App access and grants are refused on a sync-off database** (`DB3_SYNC_OFF_NO_SHARING`).
- **An app-held database has no owner** — nobody can open it until it is shared, given app access, or transferred.
- **Access-request events are not replayed** after a disconnect — list on every connect.
