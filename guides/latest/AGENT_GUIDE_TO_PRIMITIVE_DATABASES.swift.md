# Agent Guide to Primitive Databases

> **Status: beta.** Beta. Served in production; the API can still change.

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

A type is the schema for every database of that type, every client, and every function. `primitive config create db3-type <name>` scaffolds one; `primitive config fields db3-type` lists the top-level keys. Pushing and deleting types is for app admins and owners, and a type takes no rule set: push refuses a type file with a `ruleSetName` or `ruleSetId` line, and `PUT db3-types/{name}` refuses a `ruleSetId` with `DB3_TYPE_RULE_SET_UNSUPPORTED`. Delete the line, or run `primitive config pull`, which rewrites the file without it.

| Table / key | Meaning |
|---|---|
| `[type] name` | The type's name — what `create({ type })`, `ctx.db3(id, "<name>")` and `--type` take |
| `[type] timestamps` | `{ create = "<field>", update = "<field>", models = ["<model>"] }` — epoch-ms numbers stamped on every create/update by any writer; `models` (optional) limits them to those models; declare the fields `number` |
| `[type] autoPopulatedFields` | `{ field = "user.userId" }` fills on create; `{ field = { value = "…", on = ["update"] } }` names lifecycles. Fills only absent/`null` fields |
| `[type] models` | `["<key>", …]` — models this type holds that another file in `db3-types/` declares. Push includes those declarations. Once any file names one, a name no file declares, or one model declared differently in two files, is refused at push |
| `[models.<key>]` | A model. `<key>` is the stored model name (used by functions, the CLI, includes); `class_name` is the generated class/struct (Swift codegen refuses `TypedQueryModel`, `PrimitiveModel` and `PrimitiveRowDecodable`: pick another `class_name`) |
| `[models.<key>.fields.<field>]` | `type` = `string` \| `number` \| `boolean` \| `date` \| `id` \| `stringset` \| `vector` (see **Vector fields** below; declare `id` only to set its options; an undeclared `id` is minted on create); options `required`, `default`, `indexed`, `unique`, `enum` (string only; types the generated field, writes unchecked unless `enforce_enum = true`; see **Enforced enums** below), `max_count` / `max_length` (stringset: member count / each member's length, client-checked), `auto_assign`, `auto_stamp` (`create`\|`update`\|`both`; epoch ms, stamped by whichever side writes). A `date` field takes an ISO 8601 string (or a number) |
| `[[models.<key>.unique_constraints]]` | `name` + `fields` — a compound unique key (upsert target). A unique constraint applies to scalar fields only. A `stringset` is refused `UNIQUE_ON_STRINGSET` at push, and a client model declaring one throws `UniqueStringsetError` |
| `[models.<key>.relationships.<rel>]` | `type` = `refersTo` \| `hasMany` (`model`, `related_id_field`, optional `order_by_field`, `order_direction`) or `hasManyThrough` (`model`, `join_model`, `join_model_local_field`, `join_model_related_field`) |
| `[triggers.<key>] triggers = [...]` | `{ on = "create"\|"update"\|"save", when = "<CEL>", set = { field = "<CEL>" } }` or `refuse = { code = "…", message = "…" }` |

**Defaults.** A `default` is written on create. A scalar field with a literal default (not a string set, not a generator such as `"$now"`, not a string starting with `$`) reads the default whenever the stored value is missing, `null` or of another kind (judged as stored: `"5"` in a number field, `1` in a boolean field) on every read: the generated web and iOS models, `ctx.db3` and the record routes, a database filling from the type revision it holds. It is typed non-optional there (web model, iOS model, the function `Db3` row and the row projection an `outputSchema` ref inlines); a create may omit it, and a typed write of `null` to it does not compile. In a document with Yjs state a stored `0` or `1` in a boolean field still reads `false` or `true`. A database record without a string set reads the set's default. The read stores nothing: a saved `null` stays `null`, a change trigger's payload and socket sync carry the stored row, and a read that selects fields (an include's `projection`) fills only those. Filters, sorts, counts, aggregates and cursors compare stored values: `{ completed: false }` misses a record stored without `completed`; add `{ completed: { $exists: false } }` under `$or`. An `include` relates records by stored keys, never by a filled default. A page of a typed database is measured with its defaults filled, so it can end earlier.

Trigger and `autoPopulatedFields` expressions see `user.userId`, `user.writer`, `record.*`, `database.id`, `now()` and `lookup()`. A rule never writes `id`. `user.writer` is `"user"` (a member's device or API call, or a function acting `asUser`), `"function"` (a server function someone invoked; `user.userId` is that member) or `"system"` (cron, webhook, database change, `--as system`; `user.userId` is `sys:<appId>`). Keep members off a model only your functions write with `{ on = "save", when = "user.writer == 'user'", refuse = { code = "PIPELINE_ONLY" } }`. It stops saves and batches, not a bulk load by anyone with write access: a guard against app mistakes, not an access boundary (for that, a sync-off database only functions write). A `refuse` reaches the writer as `DB3_TRIGGER_REFUSED` with your code in `error.details.reason`; values a rule set come back as `appliedFields` (server calls) and sync to devices with the record. Reading a field the record lacks is an error, not `false`: a refusing trigger whose `when` errors refuses the write with reason `TRIGGER_EVALUATION_FAILED`, and a `set` trigger's is skipped, so guard optional fields (`has(record.status) && record.status == 'blocked'`). `has()` takes a path that starts at a name; for a looked-up record's field write `lookup('m', record.ref).?field.hasValue()`. Push refuses an expression that can never be evaluated, naming the trigger and the failing term.

**Typed databases.** Create with `type` (`client.db3.create({ type })`, `primitive db3 create --type`), or attach later with `primitive db3 type attach <db3-id> <type>` / `client.db3.setType(id, type)`; a type is never detached. On a typed database, a model the type doesn't declare is refused `DB3_MODEL_NOT_DECLARED`, the type's indexes, constraints and rules apply to every writer (a bulk load skips the rules; see Bulk loading), and a client's own model declaration is not installed. An attach over records that break the type is refused `DB3_TYPE_REFUSED` with `details.reasons`.

**Untyped databases.** Without a type, a database's indexes and unique constraints come from the declarations of the clients that write it. A client declares to an untyped database only the models it uses there: models its copy holds records of; a model it first writes there (declared just before the write); models bound to it with `BaseModel.setModelDefaultDb3(model, id)` (Swift: `client.setModelDefaultDb3(model, id)`); and, when it is the client's default (`BaseModel.setDefaultDb3(id)`; Swift: `client.setDefaultDb3(id)`), every model without a default database of its own. A model used only in other databases, runtime additions (`client.registerModel`; Swift: `client.registerModels`) included, is not declared there. To have a model's indexes and unique constraints in force before its first record, bind it with `setModelDefaultDb3` before subscribing.

**Changing a type.** Edit and `primitive config push`; the push reaches every database of the type in the background (`primitive db3-types push-status <type>`, `db3-types list`, `db3-types get <type>`). What to write: to add, write the new model, field or enum value (also safe: making a field optional, `indexed`/`unique`/`default`/`description` edits; a field newly `indexed`/`unique` is indexed everywhere); to stop using a field, retire it (below); to rename a field, `storedAs` (below); to change a field's type, add a new field and copy the values (below); to remove outright, the override (below). Once a database of the type exists, push refuses `DB3_TYPE_NOT_APPEND_ONLY` (409, `details.reasons[]` = `{ path, change, reason }`, nothing written), and each refusal has its alternative: a removed field (retire it), a removed model (leave it declared, or the override), a changed field `type` (a new field plus a copy), a removed `enum` value (keep the value; stop writing it), a field made `required` or a new `required` field in an existing model (add it optional, with a `default`), a changed vector `dims`/`element` (a new vector field). `config push` checks every type before writing any and prints each reason; with no database of the type, everything pushes. To push one anyway: set `[app] allowFieldRemoval = true` in `app.toml`, push, then `primitive config push --allow-field-removal` (HTTP: `?allowFieldRemoval=true`; refused `DB3_TYPE_FIELD_REMOVAL_NOT_ENABLED` unless the route's own app has the stored setting `true`, so another app's setting never applies; a type with `allowLocalOnly = true` refuses with the flag too). The override removes only the declaration: stored values stay, server reads still return them and builds that declare the field still read them; retire a field to hide its values. A database whose records break an allowed revision (e.g. a new `unique` over duplicates) refuses it and keeps its old revision; `push-status` names it, `primitive db3-types push <type> --retry <run-id>` retries after the fix.

**Retiring a field.** Three pushes, each a field key: `deprecated = true` (still read and written; generated `@deprecated` / `@available(*, deprecated)`; type writes answer `deprecatedFields: ["<model>.<field>"]`), then `cleared = true` beside it (needs `deprecated`; refused with `required = true`), then the name moves into the model's `removedFields = ["<field>"]` (a tombstone: never declared again in that model by any type of the app, never removed; deleting the last type that declares the model, with no database of it, releases the name). A cleared or removed field is left out of every read and generated model, reads as unset in filters, sorts and aggregates, is stripped from writes (an old build's write is accepted, not refused), and a record's stored value is dropped on its next write or read. Refusals are `DB3_TYPE_NOT_APPEND_ONLY` with `change` `cleared-field-in-use` (a compound unique constraint, trigger, `autoPopulatedFields` or `timestamps` names it), `cleared-field-required` (clearing a field stored `required` on a type with databases; push it optional and ship that build first), `cleared-field-defaulted` (clearing a field stored with a `default` on a type with databases, even when the same push drops the default; push it without the default and ship that build first), `field-tombstoned-without-clear`, `tombstoned-name-declared` or `tombstone-removed`; all but `cleared-field-required` and `cleared-field-defaulted` carry `always: true` and hold with no database and under `--allow-field-removal`. A database holding a registered index over several fields, or a vector index, that names the field refuses the clearing revision (`DB3_TYPE_REFUSED`): `DELETE /indexes/<name>`, then `db3-types push <type> --retry <run-id>`. Removing `cleared = true` un-clears: the write answers `unclearedFields`, server reads return values not yet dropped, dropped registered indexes are not restored, and a device keeps its local copy until it next downloads the database whole. `db3-types get <type>` prints `Deprecated`, `Cleared` and `Removed` rows, each `<model>.<field>`. Clearing is not erasure.

**Renaming a field.** Change the key and add `storedAs = "<old name>"` (e.g. `[models.tasks.fields.dueAt]` with `storedAs = "due"`); records, indexes and older builds keep the stored name, nothing is rewritten. Later renames change the key again and keep `storedAs`; a field may take back a former name. Type writes answer `renamedFields: [{ model, storedAs, from, to }]`; `GET db3-types/:name` answers `formerNames` (`{ <model>: { <stored>: [names] } }`); `config pull` never writes former names. The server takes the current, former or stored name in record data, filters, sorts, aggregates, write options, index registrations and hooks; HTTP record routes answer stored names unless `?names=current` (`ctx.db3` and the CLI pass it). Refusals: `field-name-collision` (a name another field has or had, or a removed field's) and `hook-name-not-compilable` (an expression using `record` whole or a computed key on a renamed model, or a type-wide name stored two ways), both `always: true`; changing or dropping an existing `storedAs` is `field-removed`, and also `field-name-collision` under the same key (it stays reserved to the old stored name). `storedAs` is refused while legacy databases of the type exist. `db3 export` writes a renamed field's records and declaration under its current name (the declaration keeps `storedAs`); `db3 import` loads them under the stored name, and an export from before or after a rename imports with this CLI (compared by stored name). Regenerate models from the renamed file: generated models expose only the current name and write the stored one (`CodingKeys` raw value in Swift), and a single-field unique constraint keeps its stored-derived name (`tasks_due_unique`). `client.db3.records(id)` answers the build's current names and sends stored names, but only on a database whose type the model was generated from (a `schemaToml` with `[type] name` counts; `defineModelSchema` doesn't); elsewhere it keeps stored names. It never sends `names=current`, so a later push doesn't change a shipped build's answers. JS records calls go to the server, which also takes former names; a device (models, Swift `client.db3.records`) takes only the current and the stored name. Upsert targets are sent as written: a constraint of that name wins over a field. A rename is live in a database once the push has committed there.

**Changing a field's type.** A field's `type` never changes on a type with databases; replace the field in four pushes. Add the new field beside the old (`sizeN` of type `number` beside `size` of type `string`) and ship code that writes both; copy each record's value with a server function called once per database (`ctx.db3(db3Id, "<type>").model("tasks")`: page with `query({ limit, cursor })` and `nextCursor`, `patch(id, { data: { sizeN: Number(task.size) } })` wherever `sizeN` differs, since the old field is the source until cleared); deprecate the old field and ship code that reads the new one and still writes both; clear the old field. Reads then return only the new field. Builds from before the first push write only the old field, and each copy run catches `sizeN` up with their changes: run it again just before the clear, since a cleared field can't be read. An old field written after that last run is lost with the clear, so clear once those builds have stopped writing, or accept the loss.

**Archiving a type.** `[type] archived = true` (or `primitive db3-types archive <type>`, which edits the file and pushes it; `unarchive` removes the line and pushes) stops new databases of the type: `client.db3.create` with it, `getOrCreateWithAlias` whose alias names nothing, and `setType` on an untyped database are refused 409 `DB3_TYPE_ARCHIVED` (`details.type`) for every caller, functions and admins included. Existing databases of it, an alias already naming one, re-attaching it to a database already of it, and pushing new revisions work as before; `db3-types list`/`get` show the status, and `PUT db3-types/:name` takes `archived` (`true`, `false`/`null`; omitted keeps it). A create made on the device resolves first, then `db3:create-failed` fires with `code: "DB3_TYPE_ARCHIVED"`; its records stay on the device. Archiving, like any type push, reinstalls the type in each of its databases.

```swift
  // Listen before creating. Hold the subscription for as long as the handler
  // should run; dropping it stops the handler.
  let failures = client.observeOnMainActor(Db3CreateFailedEvent.self) { [weak ui] failed in
    if failed.code == JsBaoErrorCode.db3TypeArchived.rawValue, let type = failed.type {
      ui?.showTypeRetired(type)
    }
  }

  // Resolves at once; a refusal arrives later through `failures`.
  let db = try await client.db3.create(title: "Q3 plan", type: "project")
```

**Who creates a type.** `[type] create = "admins"` keeps creating databases of the type to app admins and owners; `[app] untypedDb3Create = "admins"` in `app.toml` does the same for databases with no type. A member is then refused 403 `DB3_CREATE_ROLE_REQUIRED` (`details.type`, `null` for no type), nothing created, on `client.db3.create`, `getOrCreateWithAlias` whose alias names nothing, and `setType` naming a restricted type. Exempt: app admins and owners, server functions (`ctx.api.db3.create` creates for the calling member), and `localOnly: true` creates. `"members"` or no key is open to every member; `PUT db3-types/:name` takes `create` (`"admins"`, `"members"`/`null`; omitted keeps it) and `db3-types list`/`get` show it. A create made on the device resolves first, then `db3:create-failed` fires with the `id`, `code: "DB3_CREATE_ROLE_REQUIRED"` and the `type`; its records stay on the device, and a create that already committed still answers it. Offer the create only to users who can make it. `primitive init` sets `untypedDb3Create = "admins"` on an app it creates for the web template.

```swift
  // Listen before creating. Hold the subscription for as long as the handler
  // should run; dropping it stops the handler.
  let failures = client.observeOnMainActor(Db3CreateFailedEvent.self) { [weak ui] failed in
    if failed.code == JsBaoErrorCode.db3CreateRoleRequired.rawValue {
      ui?.showAdminsOnly(failed.type)
    }
  }

  // Returns at once; a refusal arrives later through `failures`.
  let db = try await client.db3.create(title: "Design team", type: "team")
```

**Types for local-only databases.** `[type] allowLocalOnly = true` lets the type describe local-only databases: the generated code carries its declarations, and a device judges a local-only database of the type by them. `PUT db3-types/:name` takes `allowLocalOnly` (omitted keeps it); `db3-types list`/`get` show it; `config pull` writes it only when true. From the push after the one that sets it, every change must only add, with or without a database of the type and with `--allow-field-removal` too; otherwise `DB3_TYPE_NOT_APPEND_ONLY` with `details.allowLocalOnly: true`:

| Pushes | Refused (`change`) |
| --- | --- |
| New models, fields (a `default` allowed), enum values, `indexed`, `unique`, constraints, an `enum`, `enforce_enum = true`; clearing, `removedFields`, `storedAs` renames; `description`, `class_name`, `deprecated` | Every append-only refusal above; `field-uncleared`; `index-dropped` (`indexed`/`unique` removed or a constraint removed or changed, unless the field is cleared); `default-changed` (an existing field's `default` added, changed or removed); `enum-removed`; `enum-enforcement-removed`; `field-option-changed` (any other option); `local-only-hooks` (a trigger, `autoPopulatedFields` or `timestamps`, which also refuse setting the key; use a second type for hooks); `allow-local-only-removed` (`false`, `null` or the line deleted; also deleting the type, by `config push --prune` or `DELETE db3-types/:name`, `force` included, with or without databases: keep the file); `tombstone-not-stored-name` (a renamed field moved to `removedFields` under its current name: name its stored name) |

**Reserved field names.** `type`, anything starting with `_`, and the names generated models use are refused with `RESERVED_FIELD_NAME` (use `kind`/`category`/`status`; `id` is allowed). The generated-model names: `CodingKeys`, `UniqueConstraint`, `UniqueField`, `applyAutoStampsForSave`, `applyStringSetChangeToYMap`, `clearLocalChanges`, `constructor`, `delete`, `deleteRecordLog`, `discardChanges`, `ensureLocalChanges`, `getChangedFields`, `getCurrentJSState`, `getCurrentValue`, `getDocumentId`, `getFromYjs`, `getOrCreateStringSet`, `getOriginalValue`, `getStringSetCurrentValues`, `getStringSetFromYjs`, `getValue`, `hasFieldChanged`, `hasLocalChange`, `hasUnsavedChanges`, `hashValue`, `isDirty`, `markAllChanged`, `markStringSetChange`, `normalizeNonStringSetValue`, `primitiveValues`, `related`, `resolveUnresolvedFormat2Document`, `save`, `saveRecordLog`, `schemaDefaultFor`, `setValue`, `syncState`, `toJSON`, `validateBeforeSave`, `validateFieldValue`.

**Vector fields.** `type = "vector"` with `dims` (how many numbers, at least 1) and `element` = `f32` \| `f16` \| `i8`; only `required` besides. Bytes per vector: `f32` 4·dims, `f16` 2·dims (rounded to nearest, magnitudes ≤ 65504), `i8` dims + 4 (one signed byte each plus a per-vector scale, within 1/254 of the largest magnitude); stored as RFC 4648 standard base64 with padding, 4·ceil(bytes/3) characters. No `indexed`, `unique`, `default`, `enum`, `auto_*`, `max_*`, and no unique constraint over one. A value that doesn't fit (wrong length, non-finite number, `f16` overflow, non-canonical base64) is refused `DB3_VECTOR_INVALID`; a push changing `dims`/`element` while records hold values is refused for that database (`DB3_TYPE_REFUSED`).

**JSON fields.** In a `db3-types/` file with a `[type]` table, a field may `ref` an object def (`type = "object"`), a list def (`type = "array"`, e.g. `items = { type = "string" }`) or a map def (`type = "object"` with a schema-valued `additionalProperties`, e.g. `additionalProperties = { type = "number" }`) in `schemas/` (same three ref forms as a shared enum); only `required` sits beside it, and no unique constraint names it. The record stores canonical JSON text (RFC 8785) in a plain `string` field: the object, or the array itself for a list def (no wrapper); push sends `type = "string"`. In functions the member is `Db3Json<Schemas.<Name>>` (a list def's `Schemas.<Name>` is `T[]`, a map def's `Record<string, T>`; a plain string doesn't compile): write `encodeDb3Json(value)`, read `decodeDb3Json(row.field)`, both from `primitive-functions` and taking an object or an array; neither checks the def, and the server stores any string. Filters take only `$exists` on it; no sort, index or path filter. A JSON field's def may use only `type`, `enum`, `properties`/`fields`, `required`, `additionalProperties` (boolean, or a schema every key `properties` doesn't declare must match, a `required` key among them), one-schema `items`, `oneOf`, `minimum`, `maximum`, `minLength`/`maxLength` (Unicode code points), `pattern` (JavaScript and `NSRegularExpression` syntax, no backreference such as `\1` or `\k<name>`), `minItems`, `maxItems` and the annotations `title`, `description`, `default`, `format`, `examples`, `$comment`, `deprecated`, `readOnly`, `writeOnly`; push and `functions codegen` refuse any other. `encodeDb3Json` refuses a root that is neither an object nor an array, a non-finite number or one above 2^53 in magnitude (`DB3_JSON_INVALID`); store such a number as a string. An app build older than this release reads a list field as empty (`undefined` / `nil`). A function or prompt schema that refs a map def is refused at push and by `functions codegen` (`additionalProperties must be true or false`).
On iOS, `primitive functions codegen --lang swift` declares in `Primitive.generated.swift` each such def's type, the alias `Db3JSON_<file>_<Name>` (a `-` in the file stem written `_`) and a `Db3JSONDef_<file>_<Name>` binding the models use; the app target must compile it. The model property is `Db3JSON_<file>_<Name>?` (Optional even when `required`, which the save checks); an in-place change (`order.shipping_address?.city = "x"`) marks it changed. Every save encodes an assigned value and validates it against the def, throwing `Db3JSONValueError` (`path`, `reason`, `kind`) and writing nothing when it doesn't match or holds an `Int` above 2^53 in magnitude. Only the generated web and iOS models validate: functions, bulk loads, raw `client.db3` writes and older clients don't. A list def's type is a Swift array (`[String]`, or `[Address]` for `items = { ref = "Address" }`), a map def's a dictionary (`[String: Double]`); beside declared `properties`, the catch-all is typed by the value schema. Stored text that doesn't decode into the field's type reads `nil` (logged once per field) and stays in `<field>Raw`; a save that didn't assign the field leaves it. A struct drops keys it doesn't declare when saved, so add def fields as optional. The model's hash combines only whether a JSON field holds a value.

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

**What a record holds.** Strings, numbers (integers exact to ±2^53), booleans, `null`, string sets, and vectors (as their base64 text). No nested objects or non-string arrays — flatten, or use a **JSON field** (below). A field's first non-null value fixes it as a set or a scalar; the other kind is then refused `DOCUMENT_ENTRY_INVALID`. At most 1,000 field names per model, ever. Writing `null` to a string set empties it.

## Codegen

Every codegen reads the **same type files**, so clients and functions agree on model names and fields (a typed database refuses models it doesn't declare). Point client codegen at the `db3-types/` directory: it reads every `*.toml` in it and generates each model once — one set of models and one registration list for every kind of database the app opens. A model two files declare differently is refused, naming both files and the fields. Generated headers name the input files.

- iOS: the template's `scripts/codegen.sh` regenerates the structs on every scripted build path (`./build.sh`, `./run.sh`, `./test.sh`) and every Xcode build; a bare `swift build` compiles the committed output. Put `bao-codegen.json` (`{ "inputs": ["../../primitive/dev/db3-types"] }`: type files or directories, generated with one `GeneratedModels` barrel; or `{ "input": "<type file>" }` for one) at the root of the app target's sources, and call `GeneratedModels.register(on: client)` once at startup (the template's `AppState` does). The SwiftPM codegen plugin reads the same file. Elsewhere run `swift run swift-bao-codegen --input primitive/dev/db3-types --output Sources/<App>/Models/Generated` (`--input` repeatable; a file or a directory). Codegen reads `schemas/` beside `db3-types/` to expand a shared-enum `ref`. The SwiftPM plugin reruns when a type file changes, so after editing only a `schemas/` def, clean the build folder. Load a type that uses `ref` through its generated models: `TomlSchemaLoader` reads fields as written.
- iOS, shared enums: the type is `enum Status`, declared in `Primitive.generated.swift` by `primitive functions codegen --lang swift` (the template's `scripts/codegen.sh` runs it when `schemas/` is synced); the app target must compile that file. A struct field that refs the def is `status: Status?` (assign a case; a `String` does not compile) over a read-only `statusRaw: String?` holding the stored text, so a value outside the enum reads `nil` and is kept; a required one takes `Status` in the initializers. The function helpers and `Primitive.generated.swift`'s row types use the same `Status`. `statusEnum` and `<Model>.StatusValue` are deprecated; `<Model>.StatusValue` is not generated when a shared enum the model refs is named `StatusValue`.
- Known types: `--known-types-only` (both generators; the web template's `scripts/codegen.mjs` sets it through its `knownTypesOnly` constant) declares every input type in the barrel, archived ones marked; on iOS `GeneratedModels.register(on:)` registers them. With it, `client.db3.create` of an undeclared or build-archived type is refused on the device `INVALID_ARGUMENT` (`details.reason: "unknown-type"`, `type`, `knownTypes`, `archived`), before anything is created; `getOrCreateWithAlias` refuses an undeclared type and sends an archived one. Subscribing to (and `client.db3.records(id)` on) a database of an undeclared type is refused `DB3_TYPE_UNKNOWN_TO_CLIENT` (`details.db3Id`, `type`): nothing applied, not retried, the state carries the error; `stored()` and `listMine()` mark it `knownType: false` and `evict` removes it. `allowUnknownType: true` on create, alias, subscribe/refresh and `records(id, …)` lets that database through on the client for its lifetime; listings still mark it. A type attached while a device has the database open is refused at that device's next open (reconnect, subscribe, restart). Functions, CLI and console are not guarded.

```swift
  let subscription = try client.db3.subscribe(db3Id)
  do {
    try await subscription.ready()
    return subscription
  } catch let error as JsBaoError where error.code == .db3TypeUnknownToClient {
    var options = Db3SubscribeOptions()
    options.allowUnknownType = true
    let escaped = try client.db3.subscribe(db3Id, options: options)
    try await escaped.ready()
    return escaped
  }
```

- Functions: `primitive config push` (or `primitive functions codegen`) types the `Db3` namespace of `functions/primitive.d.ts`; `ctx.db3(id, "<type>")` then accepts only existing types, `.model("<key>")` only declared models, and rows are typed (a type's `timestamps` fields required numbers on read rows of the models it stamps, absent on the others; writes leave them out).

A database that grew without a type: `primitive db3 schema <db3-id> --toml --output primitive/dev/db3-types/<name>.toml` writes its observed fields as a type file to review, push and attach.


## Records

On a device, the generated models read and write the **local copy** of a synced database. Name the database with `{ db3: id }` on `query`/`count`/`aggregate`/`save`/`increment` (Swift: `QueryOptions(db3: [id])`, `AggregateOptions(db3: [id])`, `save(in: id)`, `Model.increment(id:fields:in: id)`), or set a default with `BaseModel.setDefaultDb3(id)` (Swift: `client.setDefaultDb3(id)`). Unless you name a database, a read applies to every database on the device.

In a server function, `ctx.db3(db3Id, "<type>").model("<key>")` reads and writes on the server with the app's authority — any database, synced or not, no grant needed. A model handle has `get`, `query`, `count`, `aggregate`, `save`, `patch`, `delete`, `increment`, `batch`, `batchRead`, `deleteWhere` and `patchWhere`. Over HTTP from a client — admin tooling, or a database with sync off — `client.db3.records(id)` offers the same verbs with the same options and refusals.

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

Operators: `$eq $ne $gt $gte $lt $lte $in $nin $exists`, text `$startsWith $endsWith $containsText`, string sets `$contains $all $size`, and `$and $or`. Values compare as stored (`{ n: 5 }` never matches `"5"`). `{ field: null }` matches unset and stored-null; `$exists: false` only never-written; `$ne`/`$nin` also match unset/null; `{ field: { $eq: null } }` matches nothing. Sorting puts unset and null together — first ascending, last descending. A field can't go back to never-written once a write names it: assigning `undefined` (or `null`) stores `null`, with no operation that removes the key again; clearing a field means writing `null` to it, after which `$exists: false` stops matching but `{ field: null }` still does.

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

### Checked queries

A generated model can check a filter, sort and aggregate against its fields at compile time:

```swift
  let f = Task.fields
  let open = ModelFilter<Task>.and([f.completed.eq(false), f.dueDate.lt("2026-07-01")])

  let tasks = try Task.query(where: open, sort: [.asc(f.dueDate), .desc(f.priority)], options: QueryOptions(db3: [db3Id]))
  let remaining = try Task.count(where: open, options: QueryOptions(db3: [db3Id]))

  let category = f.category.group, hours = f.estimatedHours.sum()
  let rows = try Task.aggregate(groupBy: [category], operations: [hours], where: open, db3: [db3Id])
  let totals = rows.map { row in (row[category] ?? "none", row[hours] ?? 0) } // (String, Double)
```


Every generated model has `fields`, one token per field (`id` included; none for a `vector` field), and conforms to `TypedQueryModel`, which adds `query(where:sort:options:)`, `queryPaged(where:sort:options:)`, `queryOne(where:sort:options:)` (each also with `include:`), `count(where:options:)` and `aggregate(groupBy:operations:where:sort:limit:documents:db3:)` beside the `DocumentFilter` forms, which are unchanged. A token's type fixes its operators and operand: `eq`/`ne`/`in`/`notIn`/`exists`/`isNull` on every kind but string sets and JSON; `gt`/`gte`/`lt`/`lte` on `string`, `id`, `date` (an ISO `String`, not a `Date`) and `number` (`Double`); `startsWith`/`endsWith`/`containsText` on `string` and `stringset`; `contains`/`all`/`size` on `stringset` (`size(2)` or comparisons, `size(.gte(1), .lt(3))`); a JSON field only `exists`; a shared-enum field takes its enum or a `String`. A field the model lacks, a wrong-typed operand or a missing operator fails to compile. Combine with `ModelFilter.and([...])`/`.or([...])`; `ModelFilter(unchecked: DocumentFilter)` for what tokens can't say (`$search`). `sort: [.asc(f.x), .desc(f.y)]` is sent as `sortOrder` in that order; with an `options` that sets `sort` or `sortOrder` too it throws `.invalidArgument`. Aggregate levels: `f.x.group` (a `stringset` facet answers the member `String`), `f.tags.has("m")` (`Bool`), `f.date.period(.month)` (`String`); operations: `Model.aggregateCount()` (`Int`), `f.n.sum()`/`avg()` (`Double`), `min()`/`max()` (the field's type), each with `as:` for a custom result key. `row[key]` answers `Value?` (`nil` for a key not requested); `row.raw` is the `aggregate(_:)` row.

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
    .save(in: db3Id, upsertOn: .email)

  // A named compound constraint, `name_parentId`:
  _ = try Category(id: UUID().uuidString, name: "Work", parentId: "root", color: "blue")
    .upsertByUnique(.name_parentId, in: db3Id)
```

A generated model types the upsert names: `upsertOn` takes a field one unique constraint covers alone, `upsertByUnique` a constraint name (`<model>_<field>_unique` for a `unique` field, or the declared name).Pass the nested `UniqueField` / `UniqueConstraint` cases (`upsertOn: .email`); the `String` forms remain.
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

Vectors: a device writes numbers and reads them decoded; `null` clears. A stored value that doesn't fit the field reads as stored, with one warning. Server functions, the CLI and `client.db3.records(id)` carry the base64 text; a function converts it with `encodeVector(numbers, { dims, element })` and `decodeVector(text, { dims, element })` from `primitive-functions`. `$near` in a server query (record routes, functions, `client.db3.records`) is refused `QUERY_INDEX_NOT_ON_SERVER`.


A model's vector field is `[Float]` (`PrimitiveValue.vector`; a raw row's `row["embedding"]?.floatArrayValue`). `VectorCodec.encode(_:encoding:)`, `VectorCodec.decode(_:encoding:)` and `VectorCodec.bytes(_:)` convert the stored text.


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

`patch` (here, over HTTP and from the CLI) changes a record that exists: a missing id is refused `DB3_RECORD_NOT_FOUND` (404) and nothing is created; create with `save`. A `condition` is judged first: a `patch` or `delete` with a `condition` on a missing id is refused 409 `DB3_CONDITION_NOT_MET`, not 404, with `details.current.row` `null` (the machine-readable "no such record"). A `delete` of a missing id without a `condition` is a no-op, not an error. An `increment` of a missing id is always 404 `DB3_RECORD_NOT_FOUND`, with or without a `condition`. A trigger's `refuse` is also 409 (`DB3_TRIGGER_REFUSED`, your code in `details.reason`), so tell the 409s apart by `code`, never by status.

Batch writes: `create`, `patch`, `delete`, `upsert` (`constraint`: a `unique` field's name or a constraint name, as `upsertOn` takes; any other name is refused 400 `DOCUMENT_ENTRY_INVALID`, nothing written), `increment`, each with optional `condition` and `requestId`; up to 1,000 operations per call, reads included; all-or-nothing unless `atomic: false`. The writes are also limited to 1 MiB of JSON together (each run between two reads, or each operation with `atomic: false`), which a batch of larger records reaches first: 413 `DOCUMENT_ENTRY_TOO_LARGE` with `details.bytes` and `details.limitBytes`, nothing written; split it into smaller batches. Pass a `requestId` (a ULID) on any server write you may retry. A batch may also carry reads (`{ model, action: "query" | "count" | "aggregate", ... }`, taking what `query`, `count` and `aggregate` take): they run in order, each seeing the writes before it, and answer in `reads` at their `index`. A `requestId` retry answers the writes as first applied and the reads again.

### Reading in one request

`batchRead` answers several reads from one state of the database, each at its `index`, with read access only:

```swift
  let answer = try await client.db3.batchRead(db3Id, [
    .query(model: "projects", sort: ["updatedAt": -1], limit: 20),
    .count(model: "tasks", filter: ["completed": false]),
    .aggregate(model: "tasks", options: ["groupBy": ["status"], "operations": [["type": "count"]]]),
  ])
  // answer.results[0].answer is .query(items:hasMore:nextCursor:), then .count(n), then .aggregate(rows)

  // A batch of writes may carry reads too: each sees the writes before it.
  let written = try await client.db3.batch(db3Id, [
    .create(model: "visits", data: ["page": "/home"]),
    .count(model: "visits"),
  ])
```

Server function: `await ctx.db3(id, "workspace").batchRead([{ model: "tasks", action: "count" }, ...])`, or `.model("tasks").batchRead([{ action: "count" }])`. Each result is typed by the read at its position: after `if (r.ok)`, a `query` read's `items` are that model's rows (as `model(m).query()` types them), a `count` read has `count`, an `aggregate` read `result`; a mixed `batch`'s `reads` likewise, in op order, writes skipped. No cast is needed, and a field the model doesn't declare, or a cast to an unrelated row type, fails the push typecheck. CLI: `primitive db3 records batch-read <db3-id> --data-file reads.json` (one line per result; `--partial`, `--json`).

- Up to 100 reads; no read may use another's result.
- The whole answer is at most 8 MiB: a page that would pass it ends early with `hasMore`; a read that still can't fit (a first record, a page with `include`, aggregate rows) is refused `DB3_PAGE_TOO_LARGE` with `details.index`.
- By default the first refused read refuses the request (its code, `details.index`, `details.model`); with `atomic: false` it is answered `{ ok: false, code, status, error }` in its place.
- A batch of reads needs read access; a batch with writes needs write access even if it carries only reads (`records/batch` is refused 403 to a reader).

### Several databases in one call

`client.db3.parallel` sends 1 to 10 separate `batch` / `batch-read` requests, each on a different database, in one call. Not one batch: nothing is atomic or ordered between them; each is authorized and answered as its own call would be.

```swift
  let response = try await client.db3.parallel([
    .batchRead(projectsDb3, [.query(model: "projects", limit: 20)]),
    .batchRead(teamDb3, [.count(model: "tasks", filter: ["completed": false])]),
    .batch(myDb3, [.create(model: "visits", data: ["page": "/home"])]),
  ])
  // One result per request, in order; a refused request is answered in its place.
  for result in response.results {
    if case .refused(let refusal) = result.answer { print(result.status, refusal.code ?? refusal.error) }
  }
```

Server function: `await ctx.api.db3.parallel({ body: { requests: [{ db3Id, route: "batch-read", body: { operations } }, ...] } })`. CLI: `primitive db3 records parallel --file requests.json` (exits 1 when any request answered 400 or more; `--json`).

- Each request is `{ db3Id, route: "batch" | "batch-read", body }`, `body` exactly that call's body. The answer is `results: [{ status, body }]` in request order; a refused request is answered in its place, never thrown.
- The call itself is refused 400 only for its shape: `DB3_PARALLEL_INVALID` (no requests, more than 10, a bad item, `details.index`) and `DB3_PARALLEL_DUPLICATE_DB3` (a database named twice, `details.indexes`); nothing runs.
- Access is per request: a reader's `batch-read` is answered, their `batch` refused 403 in its place. `names` and `userId` apply to every request.
- With N requests each database answers within `floor(8 MiB / N)` bytes, writes included: a page ends early with `hasMore`; an answer that can't fit is refused `DB3_PAGE_TOO_LARGE` in its place.
- Retry by resending the whole call with a `requestId` on each `batch`: each replays its first answer. A replay larger than its share is refused `DB3_PAGE_TOO_LARGE` though it applied; send it alone to read its answer.

### Changing every record that matches a filter

`deleteWhere` and `patchWhere` delete, or merge one `data` object into, the records of one model matching a filter, a page per call, each page one write.

```swift
  var cursor: String?
  repeat {
    let page = try await client.db3.deleteWhere(db3Id, model: "transactions", filter: ["accountId": .string(accountId)], cursor: cursor)
    cursor = page.nextCursor
  } while cursor != nil

  // Patch takes the fields to merge into every matching record:
  let patched = try await client.db3.patchWhere(db3Id, model: "transactions", filter: ["reviewed": false], data: ["reviewed": true])
```

Server function:

```ts
const txns = ctx.db3(input.db3Id, "finance").model("transactions");
let cursor: string | undefined;
do {
  const page = await txns.deleteWhere({ filter: { accountId: input.accountId }, cursor });
  cursor = page.nextCursor;
} while (cursor);
await txns.patchWhere({ filter: { accountId: input.accountId, reviewed: false }, data: { reviewed: true } });
```

CLI: `primitive db3 records delete-where <db3-id> <model> --filter '<json>' [--limit n] [--cursor c] [--all] [-y] [--json]` (asks first unless `-y`) and `primitive db3 records patch-where <db3-id> <model> --filter '<json>' --data '<json>' [--limit n] [--cursor c] [--all] [--json]`; `--all` follows `nextCursor` until `hasMore` is false and prints the total.

| Parameter | |
|---|---|
| `filter` | Required; what `query` takes. `{}` is every record; an absent `filter` is refused 400. |
| `data` | `patchWhere` only: the fields merged into each record, as `patch`'s `data`. |
| `limit` | Records per call: 1,000 by default and at most for `deleteWhere` and `patchWhere`; larger is refused 400, never clamped. |
| `cursor` | The previous answer's `nextCursor`. |
| `requestId` | A ULID; a retry answers the first answer, a refusal included. |

- Answers `{ changed, hasMore, nextCursor?, cutBy?, lsn }`. `hasMore: false` means nothing matching is left: stop without another call. `lsn` is `null` when nothing matched.
- Records are taken in id order. `$search` filters but does not rank: with `limit` below the matches you get the first by id.
- Each record meets an id-based write's rules (write access, the type's `set`/`refuse` rules and timestamps on a patch, unique keys); a change trigger gets one run per call. One refused record refuses the call with its code, `details.model` and `details.recordId` (no `index`), and nothing changes.
- A page is cut to what fits, `hasMore: true`; continue with `nextCursor`. `cutBy: "entryBytes"`: its changes would pass 1 MiB. `cutBy: "rowBytes"`: the records it reads would pass 16 MiB stored. A first record over either alone is `DOCUMENT_ENTRY_TOO_LARGE` (413); change it by id.
- The body takes no `condition` (put the test in `filter`), `atomic`, `sort` or `include`; each is refused 400.

Gotchas:

- **A patch that leaves records matching loops forever without the cursor.** `patchWhere({ filter: { accountId }, data: { reviewed: true } })` matches the same records afterwards; always pass `nextCursor` back, or filter on what the patch changes (`reviewed: false`).
- **Signed-out runs are refused.** A function run for a signed-out request gets `FUNCTION_SIGNED_OUT_REFUSED` (403) from both.

### Bulk loading

Use a bulk load — not saves or `batch()` — for imports, reference datasets, and any change of thousands of records or more than 1 MiB: one session uploads every row and they apply together.

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

From a server function, `ctx.db3(id, type).load(rows)` runs the whole load in one call ([Server functions guide](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#bulk-loads-load)).

CLI: `primitive db3 ingest <db3-id> --input <dir> -y` loads one `<model>.ndjson[.gz]` per model, each line one record with its `id` and fields side by side; `primitive db3 ingests get|abort <db3-id> <session-id>`. A line of `Security.ndjson`:

```json
{"id":"sec-vti","symbol":"VTI","name":"Vanguard Total Stock Market ETF"}
```

To move a database: `primitive db3 export <db3-id> --output <dir>`, then `primitive db3 import <dir>` (a new id), `--into <db3-id>`, or `--keep-id` (the exported id, which must be free in the target app).

- In code, a row is `{ model, id, patch }`: values replace fields, `{ "_deleted": true }` deletes the record. `null` stores `null` on a plain field (it does not remove the key); on a string-set field it empties the set.
- `mode: "merge"` (default) touches only named records; `mode: "refresh"` also deletes every record of the rows' models no row names.
- Rows that break the type, an enforced enum, a record's value rules (value kinds, field names, vector encoding) or a unique key are skipped and counted in `refusedRows` (`refusedSample` names a few); the load continues. `primitive db3 ingest` exits 1 when every row is refused and warns with the count when some are; `primitive db3 inspect <db3-id> --refused <session-id>` lists them.
- A load fills `default`s and `auto_stamp` fields but runs no `triggers`, `autoPopulatedFields` or `timestamps`, and starts no `[[function.triggers.db3]]` run.
- Records appear together at completion; subscribed devices receive them in chunks. App writes made during the load win over conflicting rows.
- `buildChunks` holds all rows in memory — use the CLI beyond a few hundred thousand rows. A load can't be undone, and stopping the CLI's wait doesn't stop it — check `primitive db3 ingests get`.
- A session's `timings.totals.counters` gives per stage `statements`, `rowsRead` and `rowsWritten`, split `byTarget` like a batch's `cost`: a load fills `manifest`, `apply` (the rows), `reconcile` (a refresh's deletions) and `register` (completion).

### From the CLI

For inspecting or fixing data by hand (admin login), not app code: `primitive db3 records query|count|aggregate|save|patch|delete|increment <db3-id> <model> …`, `primitive db3 records batch <db3-id> --data-file <file>` (at most 1,000 operations per call, atomic: `create` with an optional `id`, `patch`, `increment`, `delete`, `upsert`, each with an optional `condition`, and reads; a larger file is refused before sending — use `primitive db3 ingest`), `primitive db3 records batch-read <db3-id> --data-file <file>`, `primitive db3 schema <db3-id>`, `primitive db3 inspect <db3-id>`.

### Indexes

```swift
  let index = try await client.db3.indexes.register(
    db3Id, Db3IndexRegistration(model: "contacts", field: "email")
  )
  // index.name == "idx_records_contacts_email"

  let tags = try await client.db3.indexes.register(
    db3Id, Db3IndexRegistration(model: "contacts", field: "tags", members: true)
  )
  // tags.name == "idx_members_contacts_tags", tags.members == true

  // Only the database's own queries filter on `createdBy`: devices skip it.
  let byCreator = try await client.db3.indexes.register(
    db3Id, Db3IndexRegistration(model: "contacts", field: "createdBy", on: .server)
  )
  // byCreator.kind == .scalar, byCreator.on == .server

  _ = try await client.db3.indexes.register(
    db3Id,
    Db3IndexRegistration(model: "contacts", fields: ["accountId", "email"], unique: true, name: "account_email")
  )

  for each in try await client.db3.indexes.list(db3Id) {
    print(each.name, each.fields, each.unique, each.kind, each.on, each.origins)
  }
  try await client.db3.indexes.drop(db3Id, name: index.name)
```

Every index costs on every write (server and each synced device), so declare `indexed` only on fields you filter on; adding one later is safe — existing records are indexed when the type is pushed or the index registered. Fields declared `indexed`/`unique` are indexed in every database of the type; register more per database (`primitive db3 indexes register <db3-id> --model <m> --field <f>`). Names: `idx_records_<model>_<field>`; composite (needs `unique` + `name`) `idx_named_<model>_<name>`; declared constraint `idx_uc_<model>_<constraint>`. Errors: `DB3_INDEX_NOT_UNIQUE` (existing duplicates, up to ten in `details.duplicates`), `DB3_INDEX_IN_USE` (dropping a declared index), `DB3_INDEX_SET_FIELD` (a string set field: register it with `members: true`). Registering/dropping needs the owner or an app admin. Every device with the database open takes a registration or drop within moments, without reopening it; one that doesn't have it open takes it at its next subscribe.

**Placement.** `on` places a registered index: `"both"` (default; declared indexes are always `both`), `"server"` (the database's own queries — record routes, server functions; devices skip it), `"device"` (devices' local queries only). Only the holding side pays the write cost; the other side scans. Each listed index carries `kind` (`scalar` | `members` | `text` | `vector`) and `on`. A member index defaults to `on: "server"` and may be placed `"device"` or `"both"`; a text index defaults to `"both"`; a local-only database keeps every index on the device. CLI: `--on <server|device|both>`, `--kind <kind>`. Errors: `DB3_INDEX_EXISTS` with `details.on` (same fields placed elsewhere: drop, then register), `DB3_INDEX_INVALID` (`unique` with `on: "device"`), `DB3_INDEX_PLACEMENT_UNSUPPORTED` with `details.kind`/`details.on` (a kind and placement no side builds), `DB3_LOCAL_ONLY` (`server`/`both` on a local-only database).

**Member indexes.** Unindexed, `$contains` on a string set reads every record's set: fine for thousands, seconds for millions. `{ model, field, members: true }` (CLI `--set-elements`) registers `idx_members_<model>_<field>`, listed with `members: true`: the database answers `$contains`, any-of, membership groups and unfiltered facets on the field from it, a selective `$contains` in milliseconds, answers unchanged. Cost: one stored row per member (about an eighth more storage for two or three short members per record) and a little time on each write that changes the set. Errors: `DB3_INDEX_NOT_SET_FIELD` (a single-value field, `details.field`), `DB3_INDEX_EXISTS` (the field has the other kind of index, `details.index`), `DB3_INDEX_INVALID` (`members` with `fields`, `unique` or `name`). Placed `"device"` or `"both"`, each device's copy builds it too and answers those reads from it; a `"server"` one is the database's alone, and a device's `$contains` reads the records it holds. A local-only database builds it on the device and keeps it on every write.

**Cost.** What the server answers for a database reports what the call cost its storage, in rows: `query`, `count` and `aggregate` answer `cost.rowsRead` (`cost.rowsWritten` 0); a batch answers `cost.rowsRead`, `cost.rowsWritten` and `cost.byTarget` with `records` (records and the field/unique indexes written with them), `members` (member and text indexes), `log` (write ordering, `requestId` marks), `bookkeeping` (other tables, query statistics); registering or dropping an index answers its `cost` too (a build writes a row per record covered, per member for a member index). Compare `cost.rowsRead` before and after registering an index to see what it saves, and a batch's `byTarget.records.rowsWritten` to see what it costs. A batch retried with its `requestId` answers the first answer byte for byte except `cost`, the retry's own (`rowsWritten` 0). A figure is `null` where storage keeps no counter. Measure rows, not server-side time: the server's clock doesn't move during database work. Each server read's rows are also recorded per query shape and caller: `primitive analytics db3-queries` (or `GET /app/{appId}/api/analytics/db3/queries`) ranks an app's most expensive reads; see the [Analytics guide](AGENT_GUIDE_TO_PRIMITIVE_ANALYTICS.md#finding-expensive-database-queries).

**Explaining a read.** To check which index a server read uses, explain it: `primitive db3 records query|count|aggregate <db3-id> <model> … --explain`, or from a server function `ctx.api.db3.records.explain({ db3Id, model, body: { op, filter, options, aggregate, index } })` (`op`: `query` | `count` | `aggregate`). The read runs for real and answers `indexes` (index names used), `scanned` (models whose records it walked), `statements` (each with `model`, `plan` lines, `indexes`, `scan`), `answered` and `cost.rowsRead`, not the records. `--index <name>` (body `index`) forces a field or member index of the model; `--index none` reads as if there were none (string-set reads check each record's array). Compare the two by `cost.rowsRead`: `scan` is the access path, not a row count. Needs the owner or an app admin (or a function of the app).

Watch out when explaining:

- A force that cannot serve the read is `DB3_INDEX_CANNOT_SERVE` with `details.reason`: `fields` (a field index on no field the read filters, sorts or groups on), `shape` (a member index the read has no `$contains`, any-of, membership level or unfiltered facet for, or a field index no statement of the read can take, as a facet counted from its member index alone), `plan`, `kind` (text or vector), `placement` (device-only), or `search` (a `$search` under `none`; force a field index instead). An unknown name is `DB3_INDEX_NOT_FOUND`.
- `none` can read, and bill, every record of a large model; an id filter or a page limit can stop it sooner, so `cost.rowsRead` is the measured cost. Explained reads stay out of the query-cost ranking.

Watch out when registering a member index on a large database: it is built from the existing records in that one request, which holds for about 1,000,000 records. Past that the registration can be refused and leaves no index; register the field first, then load, and each write keeps the index.

On the device that calls them, `client.db3.indexes.register` and `drop` change the copy of a database this client has subscribed before they return, so the next query uses the index. At a subscribe, `ready` returns once the copy holds the database's index list; a query made after `available` and before `ready` may still scan.


**On each device.** Each device builds the `"device"`/`"both"` indexes in its own copy, and can leave some out; nothing is sent, and the server and other devices keep them.

```swift
  try await client.db3.subscribe(db3Id).ready()

  for index in try await client.db3.indexes.local(db3Id) where index.status != .built {
    // .building, .unbuilt or .declined, with why.
    print(index.name, index.status, index.reason ?? "")
  }
  // The entries this device built nothing for, with why: placed on the
  // server only, or an index the device can't build.
  for skipped in try await client.db3.indexes.skipped(db3Id) { print(skipped.name, skipped.reason) }

  _ = try await client.db3.indexes.decline(db3Id, name: "idx_text_tasks")
  do {
    _ = try Task.query(["$search": "quarterly report"], options: QueryOptions(db3: [db3Id]))
  } catch let error as JsBaoError where error.code == .db3IndexDeclinedOnDevice {
    // The index is declined here: search on the server, or accept it.
  }

  let index = try await client.db3.indexes.accept(db3Id, name: "idx_text_tasks")
  // index.status == .built
```

- `client.db3.indexes.local(db3Id)` lists each device-placed index with `status` (`built` | `building` | `unbuilt` | `declined`) and `reason`; the subscription state's `indexes` carries the same list and its state event fires on a change.
- `client.db3.indexes.skipped(db3Id)` lists the entries this copy built nothing for, each with `reason`: placed `"server"` only, an index on `id` (the primary key serves it), a name that isn't an identifier, a member or vector index over more than one field, unreadable text options. `local` lists none of them.
- A failed build (e.g. full storage) leaves the index `unbuilt`: the copy keeps syncing, its queries scan, and each sync retries.
- `decline` drops the index from this copy and keeps it declined across restarts, evicts, and a drop and re-registration, until `accept` (which rebuilds it) or `client.db3.delete`. A device `$search` needing a declined text index (or `$near` a declined vector index) is refused `DB3_INDEX_DECLINED_ON_DEVICE` (`details.model`, `details.index`); other queries scan. Unknown names: `DB3_INDEX_NOT_FOUND`; local-only databases: `DB3_LOCAL_ONLY`.
- Client option `deviceIndexes`: `"all"` (default), `"none"`, or kinds such as `["scalar", "members"]` (Swift: `.all`, `.none`, `.kinds([...])`); left-out indexes list `declined`, and indexes already built stay built.

### Text search

A text index (`kind: "text"`, one per model, `idx_text_<model>`) answers `$search` where it is placed: `on: "both"` (default) on each device holding the database and in the record routes and server functions, `"device"` or `"server"` on that side only. Matches come back ranked, each with `_score` (higher is better), best first; the server and a device rank the same records identically.

```swift
  _ = try await client.db3.indexes.register(
    db3Id,
    Db3IndexRegistration(
      model: "tasks", fields: ["title", "category"], kind: .text, on: .device,
      text: Db3TextIndexOptions(weights: ["title": 2])
    )
  )

  let page = try client.codegen.queryPaged(
    Task.primitiveSchema, filter: ["$search": "quarterly report"], options: QueryOptions(db3: [db3Id])
  )
  for row in page.data { print(row["title"] ?? .null, row["_score"] ?? .null) }

  // A phrase, any of the words, and the title alone.
  _ = try Task.query(["$search": "\"quarterly report\""], options: QueryOptions(db3: [db3Id]))
  _ = try Task.query(
    ["$search": ["query": "report review", "mode": "any", "fields": ["title"]]], options: QueryOptions(db3: [db3Id])
  )

  // The start of a word: "repo" finds "report".
  let started = try Task.query(
    ["$search": ["query": "repo", "prefix": true]], options: QueryOptions(sort: ["_score": -1], db3: [db3Id])
  )
```

| Option | Values |
|---|---|
| `fields` | one or more string fields; a non-string value isn't indexed |
| `text.tokenizer` | `unicode61` (default; case and diacritics fold) or `porter` (English stems) |
| `text.prefix` | word-start lengths to index (1–32, up to 8) for fast `prefix` searches |
| `text.weights` | a positive weight per field, 1 by default |
| `$search` | a string, or `{ query, mode: "all" \| "any", prefix, fields }`; a double-quoted run is a phrase |

`$search` goes at the top level of a filter (AND with the rest); without `sort`, order is `_score` then id; `sort` may name `_score`. `count` and an aggregate's `filter` take it too. Input is matched as plain words: pass search-box text as it is. CLI: `--kind text --tokenizer porter --prefix 2,3 --weight title=2`; `db3 records query <id> <model> --filter '{"$search":"…"}'` searches on the server.

A generated model's `query` answers its records in rank order; `_score` rides each raw row, so read it from `client.codegen.query` / `queryPaged` (or `PrimitiveRow`), as above.

Watch out when searching:

- A side holding no text index for the model refuses `$search`: `QUERY_INDEX_NOT_ON_SERVER` (record routes, server functions) or `QUERY_INDEX_NOT_ON_DEVICE` (a device), with `details.model` and, when the index lives on the other side only, `details.index`.
- The server builds its copy within the registration, for up to 2,700,000 records of the model; above that the registration is `DB3_INDEX_TOO_LARGE` (`details.model`, `details.rows`, `details.limit`). Register before loading the records, or place it `on: "device"`.
- A second text index on a model is `DB3_INDEX_EXISTS`; a text index takes no `unique` or `name`.

### Vector search

A vector index (`kind: "vector"` on one vector field, `idx_vec_<model>_<field>`, always `on: "device"`) answers `$near` on each device holding the database: the `k` nearest records, nearest first, each with `_distance`.

```swift
  _ = try await client.db3.indexes.register(
    db3Id,
    Db3IndexRegistration(
      model: "notes", field: "embedding", kind: .vector,
      vector: Db3VectorIndexOptions(distance: .cos, index: .ann())
    )
  )

  let near: JSONValue = [
    "field": "embedding", "vector": .array(query.map { .number(Double($0)) }), "k": 10, "filter": ["pinned": false],
  ]
  let page = try client.codegen.queryPaged(Note.primitiveSchema, filter: ["$near": near], options: QueryOptions(db3: [db3Id]))
  for row in page.data { print(row["body"] ?? .null, row["_distance"] ?? .null) }

  // .exact from the first record, .ann once this device has the shared model.
  let index = try await client.db3.indexes.local(db3Id).first { $0.name == "idx_vec_notes_embedding" }
  print(index?.mode ?? .exact)

  // The owner has the model trained again, over the records as they are now.
  try await client.db3.indexes.deleteModel(db3Id, name: "idx_vec_notes_embedding")
```

| Option | Values |
|---|---|
| `vector.distance` | `cos` (default; `_distance` is the cosine distance) or `l2` (`_distance` is the squared Euclidean distance) |
| `vector.index` | `"exact"` (default: every vector compared) or `{ mode: "ann", nbucket?, codesize?, quantizer? }` (approximate, reranked on the full vectors; leave the knobs at their defaults) |
| `$near.field` / `$near.vector` | the vector field; an array of its `dims` numbers |
| `$near.k` | 1 to 10,000, default 10 |
| `$near.filter` | conditions on the model's other fields (no `$near`, no `$search`) |

`$near` is the filter's only key. An `ann` index answers exactly until the device has the database's model and has filled its index, so no query waits. One device that holds at least 1,000 records trains the model in the background (off the thread serving queries) and uploads it; the others fetch it. `client.db3.indexes.list` shows it as `annModel` (`revision`, `rows`, `trainedAt`, `bytes`) beside the index's `generation` and `revision`; `client.db3.indexes.local` shows each device's `mode` (`exact` | `ann`). The owner's `deleteModel` moves the revision so a device trains it again; devices answer exactly meanwhile. `fetchModel` reads the model's bytes and `uploadModel` uploads one trained elsewhere (`DB3_INDEX_MODEL_EXISTS` once one is listed, `DB3_INDEX_GENERATION_STALE`/`DB3_INDEX_MODEL_STALE` after a re-registration or a delete). A local-only database trains and keeps its own model. CLI: `--kind vector --distance cos --index ann`; `db3 indexes model get|delete`.

`_distance` rides each raw row: read it from `client.codegen.query` / `queryPaged` (or `PrimitiveRow`), as above.

Watch out when searching by vector:

- A field with no vector index on the device is `QUERY_INDEX_NOT_ON_DEVICE`; a registration on a field that isn't a vector field is `DB3_INDEX_INVALID`; the server's record routes and functions refuse `$near` with `QUERY_INDEX_NOT_ON_SERVER`.
- Storage on each device: a 32-bit copy of each vector (4·dims bytes, whatever the `element`), and a little more again for an `ann` index. Per 100,000 records with 768 numbers: records about 886 MiB (`f32`), 790 MiB (`f16`) or 268 MiB (`i8`), plus about 295 MiB for the vector index and 413 MiB for an `ann` index. A shared model upload is at most 64 MiB.
- Type the database before loading embeddings: an existing field can't be declared `vector` while records hold values for it (`DB3_TYPE_REFUSED`). A CLI import of an export holding a vector index needs `--type <type>` or `--into` a typed database.

### Limits

| Limit | Value |
|---|---|
| Page size | default 100, max 10,000 (`DB3_PAGE_TOO_LARGE` above; never clamped) |
| Page bytes | 8 MiB of records: a plain page ends early with `hasMore: true`; a page whose `_related` would pass it is refused with `DB3_PAGE_TOO_LARGE` |
| Include depth | 3 |
| `hasMany` related rows per request | 1,000 (`DB3_PAGE_TOO_LARGE`) |
| `$in` / `$nin` values | 1,000 (`QUERY_IN_LIST_TOO_LARGE`) |
| Bound values per query | 100 (`QUERY_FILTER_TOO_MANY_BINDS`) |
| Batch operations | 1,000 operations per call (`BATCH_TOO_LARGE`); a ceiling, the 1 MiB below is usually reached first |
| Records per `deleteWhere` / `patchWhere` call | 1,000 by default and at most |
| Stored records a `deleteWhere` / `patchWhere` call reads | 16 MiB (`cutBy: "rowBytes"`) |
| One write / batch | 1 MiB JSON for a batch's writes together, or each run between reads (`DOCUMENT_ENTRY_TOO_LARGE`, `details.bytes`) |
| Batch answer | 8 MiB |
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
| `client.db3.subscribe(id)` | Handle with `available` (readable copy on device), `ready` (caught up), `state`, events, `dispose()`. `ready` rejects `DB3_READY_TIMEOUT` (30 s without progress; `subscribe(id, { readyTimeoutMs })`, Swift `Db3SubscribeOptions(readyTimeout:)`), `DB3_ACCESS_REFUSED`, `DB3_SYNC_OFF`, `DB3_DELETED` (the database was deleted), `DB3_DISPOSED` (the subscription ended first), `NOT_A_DB3` |
| `client.db3.unsubscribe(id)` | Disposes every handle for the database |
| `client.db3.refresh(id)` | Subscribe → caught up → dispose |
| `client.db3.state(id)` | `{ resident, subscribed, phase, progress, rows, pendingWrites, lastSyncAt, permission, error, drift }` without subscribing |
| `client.db3.stored()` | Databases this device holds for the signed-in user |
| `client.db3.evict(id, { force })` | Delete the local copy; `DB3_PENDING_WRITES` unless `force` |
| `client.db3.setRetentionPolicy(...)` | Evict automatically (session-only, `maxDocs`, `ttlMs`); never a subscribed copy or one with unsent writes |

`phase`: `idle`, `opening`, `loading`, `catching-up`, `live`, `reconnecting`, `gone`. Events: `db3:state`, `db3:ready`, `db3:rows` (`changes`, `origin`: `local` this device's own write, applied before the database has seen it \| `sibling` another client on the same local store \| `remote` the database changing the local copy — a change from elsewhere, or the database correcting/reverting this device's own write), `db3:write-refused`, `db3:evicted`, `db3:gone` (`reason`: `permission`\|`sync-off`\|`client-outdated`\|`deleted`), `db3:error`.

A refusal is remembered on the device: the copy refuses writes (`WRITE_ACCESS_DENIED`) in every tab and after a relaunch. A grant heard while a subscription that was live still holds the copy asks the database again at once, and the subscription returns to `live`. Otherwise the next subscribe there asks the database again; `ready` resolves if access was restored, and rejects with the database's own reason if not. A database already gone for another reason that is then deleted emits `db3:gone` again, once, with `deleted` (at the latest on the next subscribe).

**Drift.** A device page's `drift` (and the state's) says how far behind the copy it read is, by database id: `entriesBehind` (a lower bound, as of `asOf`, when the device last heard the server's position), `live`, `pendingWrites`. `maxDrift: { entries, ageMs }` (Swift `Db3MaxDrift(entries:age:)`, seconds) on `query`, `queryOne` and `count` refuses a copy further behind with `DB3_DRIFT_EXCEEDED` (`details`: `db3Id`, `drift`, `maxDrift`); it never waits. A database the device hasn't synced with yet has no `drift` and refuses every `maxDrift` (`details.drift: null`); a local-only one has none and passes.

```swift
  do {
    let page = try Task.queryPaged(
      ["completed": false],
      options: QueryOptions(db3: [db3Id], maxDrift: Db3MaxDrift(entries: 100, age: 5 * 60))
    )
    if let drift = page.drift?[db3Id], !drift.live {
      print("\(drift.entriesBehind) changes behind as of \(drift.asOf)")
    }
    return page.data
  } catch let error as JsBaoError where error.code == .db3DriftExceeded {
    // Too far behind, or never synced on this device: catch up first.
    _ = try await client.db3.refresh(db3Id)
    return try Task.query(["completed": false], options: QueryOptions(db3: [db3Id]))
  }
```

A resident copy answers reads and takes writes while unsubscribed or offline; writes queue and send at the next subscribe, in order. `db3.create` for the user's own synced database (no `custodian`) resolves at once, online or offline, with `pendingCreate` true in the state, and the client saves it under the same id in the background (`db3:created`, or `db3:create-failed` with its `id`, keeping its records). A refusal only the server decides (`DB3_TYPE_NOT_FOUND`, `DB3_TYPE_REFUSED`) arrives as that event, never as a rejection of `create`. A create overlapped by a change of signed-in user (another user's sign-in, a sign-out) rejects `INVALID_ARGUMENT` with the reason `identity-changed` and keeps nothing; create again under the new user. Await `client.db3.commitCreate` with the id to get the server's answer (`created`, or `refused` with the reason; a refused create is tried again), and `client.db3.cancelPendingCreate` drops a create the server doesn't have yet, sending nothing; `client.db3` calls that ask the server about the database (`delete` included) wait for the commit on their own, rejecting `PENDING_CREATE` offline or after 30 s; drop an unwanted pending create with `cancelPendingCreate`, not `delete`. A device may write offline for 7 days by default (`largeDocumentWindowDays` in `app.toml` `[app]`, 1–14).

`setRetentionPolicy` with `maxDocs` keeps at most that many of the copies `stored()` lists, subscribed ones included, evicting the least recently used first (`db3:evicted` with `reason: "retention"`). A copy's last use is the later of its last open and the end of its last subscription (a dispose, a sign-out, or the client shutting down). A subscribed copy, one with unsent writes and one still being created (`pendingCreate`) are never evicted and still count, so the device may keep more. `maxBytes` never removes a database copy. `maxDocs` is a positive integer; leave it unset for no cap. `0`, a negative or a non-integer is refused with `INVALID_ARGUMENT`, and the policy stays as it was.
`ttl` evicts copies unused for that long: a positive number of seconds, a fraction included; `nil` is no TTL. `0`, a negative or a non-finite `ttl` is refused the same way. `client.db3.setRetentionPolicy` throws the refusal; assigning a refused policy to `client.retentionPolicy` leaves it unchanged and logs an error.

**Local-only databases.** `client.db3.create` with `localOnly: true` keeps a database on this device only: nothing of it is ever sent, now or in a later session. Writes are judged on the device with the database's own codes (unique keys, upserts, conditions, increments) and accepted at once, so `pendingWrites` stays 0 and `confirm: true` resolves at once. A subscription is `live` at once, `stored()` lists it with `localOnly: true`, `client.db3.indexes` works on the device, and it survives restarts for the same user.

```swift
  let db = try await client.db3.create(title: "Scratchpad", localOnly: true)

  // Saves, queries and live updates work as on any database, at once.
  _ = try Task(title: "Sketch the layout").save(in: db.id)
  let open = try Task.query(["completed": false], options: QueryOptions(db3: [db.id]))

  // It is listed with the other databases this device holds.
  let local = try await client.db3.stored().filter(\.localOnly)

  // Removing it removes its only copy.
  _ = try await client.db3.delete(db.id)
```

| Refused | Code |
|---|---|
| `create` with `sync: false`, `custodian`, or a `type` the generated code carries no local-only declarations for | `LOCAL_ONLY_UNSUPPORTED_OPTION`, before anything is created |
| A `client.db3` call that asks the server about it (`get`, `permissions`, `setSync`, `setType`, `blobs`, `ingests`, …) and `evict` | `DB3_LOCAL_ONLY` (`details.db3Id`), before any request |

**Typed local-only databases.** `create` with `type` and `localOnly: true` takes a type whose file says `allowLocalOnly = true`: codegen embeds its declarations, and the device judges every write by them at the commit (`DB3_MODEL_NOT_DECLARED`, `DOCUMENT_ENTRY_INVALID`, `DB3_VALUE_NOT_IN_ENUM`, `DB3_VECTOR_INVALID`, unique keys; `details.scope: "local"`). Its constraints and declared indexes come from the type with the server's names, and `stored()` lists its `type`. With `--known-types-only`, a type the build doesn't know is refused `INVALID_ARGUMENT` (`unknown-type`) first.

```swift
  let db = try await client.db3.create(title: "Journal", type: "journal", localOnly: true)

  // Checked on the device against the type's fields, enums and unique keys.
  _ = try Task(title: "Write the intro").save(in: db.id)

  let state = try await client.db3.state(db.id)
  if let error = state.error, error.code == JsBaoErrorCode.db3TypeRefused.rawValue {
    ui.warn(error.message)
  }
```

On open, the device compares this build's declarations with the ones the copy holds:

| Build's declarations | Result |
|---|---|
| Same hash | Nothing read or written |
| Only additions | Installed in one transaction once the records pass them; otherwise nothing installed, the copy works under its declarations, and `state(id).error` is `DB3_TYPE_REFUSED` with `reasons` (`path`, `reason`, `rows`/`duplicates`, `samples`), retried at each open |
| An older build's | The installed declarations stay and judge this build's writes; nothing reported |
| Neither contains the other | Reads work; every write is refused `DB3_TYPE_REFUSED` (`details.diverged`), and the state says so |
| None for the type | Opens under the installed declarations |

Copy whose declarations can't be read: writes are refused `DB3_TYPE_REFUSED` (`details.unreadable`) until a build that declares the type opens it, checks the records and installs its declarations.

Only `client.db3.delete(id)` (nothing sent) and `logout(wipeLocal: true)` remove it. `evictAllLocal` and the retention policy keep it, and it counts toward no `maxDocs` cap. Another user signing in on the device does not see it; its own user sees it again.

A `DB3_DISPOSED` rejection carries a `reason` in its details saying why the subscription ended: `disposed` (`dispose()`), `destroyed` (the client was destroyed; `subscribe` on a destroyed client throws the same), `identity-changed` (a sign-out, or another user signed in: subscribe again under the new user) or `evicted` (the local copy was removed).


### Re-rendering on change

Every generated model has `subscribe(callback)` → unsubscribe function; it fires on local and remote changes. Re-run the query in the callback.

In the iOS template, `BaoDataLoader` wraps this: `bind(client:subscribeTo:load:)` re-runs `load` when the subscribed models or the database change, and `phase` is `.loading`, `.empty` or `.loaded(data)`. A change made while `isPaused` is true reloads once when `isPaused` goes back to false.

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

A device write (save, patch, delete, increment) applies to the local copy and resolves before the database has seen it. The device refuses three writes itself, before anything is sent: a write while the user holds `reader`, and, once the device has opened a typed database, a model the type doesn't declare (`DB3_MODEL_NOT_DECLARED`), a value failing a field's declared kind, e.g. `""` for a `date` (`DOCUMENT_ENTRY_INVALID`, the database's own message), or a value outside an enforced enum (`DB3_VALUE_NOT_IN_ENUM`). The database remains the authority for every other refusal. If the database refuses a write at sync, the record reverts and `db3:write-refused` carries `writes` (every affected write, in order, to redo), `error.code`, and `conflictingRecord`/`current` where relevant. Pass `confirm: true` (Swift `Db3WriteOptions(confirm: true)`) on the write instead to wait for the database's answer at the call site and reject there if it's refused, rather than handling `db3:write-refused` afterward.

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
| `DB3_CONDITION_NOT_MET` | Condition no longer held (`current`); also a conditional patch or delete of a missing record, with `current.row` `null` |
| `DB3_RECORD_NOT_FOUND` | Increment of a missing record, with or without a condition |
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

With `sendEmail: true`, each new grantee gets an email; its link is `documentUrl` when you pass one, otherwise the app's base URL. Granting by email with `sendEmail: true` needs the app's base URL set (the invitation links to it). Grants by email work before the person joins (pending invitation): `listPendingInvitations()` lists them and `revokePendingInvitation(invitationId)` cancels one on this database only (the invitation and its other shares stay). Group grants follow group membership. Removing a grant turns the user's subscription `gone` at once: the local copy stays readable, new writes are refused `WRITE_ACCESS_DENIED`, and queued writes stay until access returns or you evict.

CLI (app admin):

```bash
primitive db3 permissions grant <db3-id> --user-id <id> --permission reader   # or --email; add --send-email [--url <url>] [--note <text>]
primitive db3 permissions list <db3-id>
primitive db3 permissions revoke <db3-id> --email <email>                     # or --user-id <id>
primitive db3 pending-invitations list <db3-id>
primitive db3 pending-invitations revoke <db3-id> <invitation-id>
primitive db3 group-permissions grant <db3-id> --group-id <id> --permission reader
primitive db3 group-permissions list <db3-id> --include-system
primitive db3 group-permissions revoke <db3-id> <group-id>
primitive db3 link-access set <db3-id> --level reader|read-write              # also get, clear
```

`--url`/`--note` need `--send-email`. Revokes confirm; pass `-y` in scripts.

### Access requests

`client.db3.permissions(id).requestAccess(RequestAccessOptions(permission: .reader))`lets a member ask the owners; an owner calls `listAccessRequests()`, `approveAccessRequest(requestId)` or `denyAccessRequest(requestId)`. Every owner hears`Db3AccessRequestCreatedEvent`on a new or re-submitted request; the requester and every owner hear`Db3AccessRequestResolvedEvent`when any owner resolves it. Events cover every database the user owns (filter by `documentId`) and also arrive under `document:` names; use the `db3:` ones. Keep a pending list without polling: list on every connect (the client reports connected once these events reach it, so nothing falls between the list and the events), apply events at once, and while a list call runs keep the events in a buffer of its own and replay them over its answer, dropping an answer a newer call superseded.Observe both events with `observeOnMainActor` (emit order across types), not two `stream(for:)` loops, so a resolved event never lands before its created one.
```swift
  private(set) var pending: [String: String] = [:] // requestId → requesterId
  private var generation = 0
  private var buffered: [(requestId: String, requesterId: String?)]? // events seen while the latest list call runs

  private func apply(_ requestId: String, requesterId: String?) {
    pending[requestId] = requesterId // nil (resolved) removes it
  }

  private func receive(_ requestId: String, requesterId: String?) {
    apply(requestId, requesterId: requesterId)
    buffered?.append((requestId, requesterId))
  }

  func refresh(client: JsBaoClient, db3Id: String) async throws {
    generation += 1
    let mine = generation
    // A fresh buffer per call: events heard before it started are already in its answer.
    buffered = []
    let rows: [DocumentAccessRequest]
    do {
      rows = try await client.db3.permissions(db3Id).listAccessRequests()
    } catch {
      if mine == generation { buffered = nil }
      throw error
    }
    guard mine == generation else { return } // a newer refresh started; its answer wins
    pending = Dictionary(rows.map { ($0.requestId, $0.requesterId) }, uniquingKeysWith: { _, last in last })
    for event in buffered ?? [] { apply(event.requestId, requesterId: event.requesterId) } // replay
    buffered = nil
  }

  // Run in a SwiftUI `.task`: it stops when the view goes away.
  func watch(client: JsBaoClient, db3Id: String) async {
    // Main-actor observers run in emit order across event types, so a resolved
    // event is never applied before the created one it follows (two
    // `stream(for:)` loops give no such order). Events come for every database you own.
    let subscriptions = [
      client.observeOnMainActor(Db3AccessRequestCreatedEvent.self) { [weak self] e in
        if e.documentId == db3Id { self?.receive(e.requestId, requesterId: e.requesterId) }
      },
      client.observeOnMainActor(Db3AccessRequestResolvedEvent.self) { [weak self] e in
        if e.documentId == db3Id { self?.receive(e.requestId, requesterId: nil) }
      },
    ]
    defer { subscriptions.forEach { $0.cancel() } }
    // Nothing is replayed while the socket is closed: list again on every connect.
    let connects = client.stream(for: StatusChangedEvent.self)
    try? await refresh(client: client, db3Id: db3Id)
    for await e in connects where e.status == .connected {
      try? await refresh(client: client, db3Id: db3Id)
    }
  }
```

CLI: `primitive db3 access-requests list|approve|deny <db3-id> [<request-id>]` (`approve --permission <level>` overrides the requested level). `access-requests create <db3-id> --permission reader` files the signed-in user's own request; an admin login is refused `ALREADY_HAS_ACCESS`.

### App access

**App access** — every current and future member: `client.db3.setAppAccess(id, "reader" | "read-write" | null)` (app admin), `primitive db3 app-access set <db3-id> reader|read-write|none`, or in a function `ctx.api.db3.setAppAccess({ db3Id, body: { level } })` with capability `db3:setAppAccess`. Content only at its level (`reader` reads, `read-write` writes; no share/delete). Refused on a sync-off database.

### Public access

**Public access** — lets server functions serve a database to callers who are not signed in (only while the app's `signedOutAccess` is on):`client.db3.permissions(id).setPublicAccess(.reader)` (or `nil`)(app owner or admin; a non-admin owner is refused 403), `primitive db3 public-access set <db3-id> reader|none`, or in a function `ctx.api.db3.setPublicAccess({ db3Id, body: { level: "reader" } })` with capability `db3:setPublicAccess`. Answers `{ id, level, changed, signedOutAccess }`. Not a sharing level: members with no grant still can't open, sync, list or read it. `db3.get` shows `publicAccess`, `publicAccessUpdatedBy`, `publicAccessUpdatedAt` to app admins, the owner, `read-write` grantees and functions while it is set. A function serves it with `requirePublicDb3` or `requireDb3Level(…, { orPublic: true })` in `authorize` (see [Server Functions](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#serving-a-database-named-by-the-caller)). Populate it server-side — an admin tool's catalog load, or a scheduled function copying a figure computed from private data — and keep a public function that writes behind its own `authorize`, rate limit and usually Turnstile.

```swift
  let permissions = client.db3.permissions(db3Id)
  let change = try await permissions.setPublicAccess(.reader)
  if !change.signedOutAccess { print("Stored; it takes effect once the app's signedOutAccess is on") }
  try await permissions.setPublicAccess(nil) // clear it
```

### Checking access

Device: `client.db3.get(id)` → `{ …, permission, accessSource }` for the signed-in user. Function: `auth.db3(id, type).permissions()` in `authorize` (caller's level or `null`); `ctx.api.db3.validateAccess({ db3Id, body: { userId } })` → `{ hasAccess, permission?, accessSource? }` for any member (always pass `userId`; omitted, it describes the function's app authority). CLI: `primitive db3 access get <db3-id> --user-id <id>`. An app role is not a content grant. Sync-off databases answer no access for members.

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

Every database has a `custodian`: `user:<userId>` (holder and owner; a member's own creates, a function's creates for its caller) or `app` (no owner; created from the CLI's admin token with no custodian, by an app admin naming `custodian: "app"`, or by a function naming it with capability `db3:transfer`). Transfer: owner → another member (old owner keeps `read-write`); to/from the app needs an app admin (`primitive db3 transfer <db3-id> app`) or `ctx.api.db3.transfer({ db3Id, body: { to } })` with `db3:transfer`. A transfer to the app removes the previous owner's grant (they keep only group or app access; their subscriptions turn `gone`, `permission`); a grant made right after it reopens their live subscriptions on its own. When a member leaves the app, databases they hold pass to the app if another user holds an accepted grant, any group is granted, the database belongs to a collection (above), or app access is set (link access and unaccepted invitations don't count); otherwise they are deleted. List by holder: `primitive db3 list --custodian app|user:<id>`; every database a member can open: `primitive db3 list --user-id <id> [--via held,shared,app]` (one row per `via`, sync-off databases included).

`client.db3.listMine()` lists the databases the signed-in user can open: held, shared with them, then app access, each entry naming its listing as `via`. Pass `via` to read only some; page with `cursor`. A sync-off database is not listed (the admin listing still shows it).

**Renaming.** `client.db3.update(id, { title })` sets the title `get` and `listMine` answer, and resolves with the database as `get` does. Its owner, a `read-write` grant or link, and app owners and admins rename it; anyone else is refused 403 `DOC_ACCESS_DENIED`. The title is stored trimmed (blank or over 255 characters: 400). Connected devices of direct-grant holders get the new title at once (`stored()` entries carry `title`); others see it at their next read. A local-only database is renamed on the device, nothing sent (`INVALID_ARGUMENT` for a blank or long title). CLI: `primitive db3 update <db3-id> --title <title>`; function: `ctx.api.db3.update({ db3Id, body: { title } })`, no capability. Don't rename through `documents.update`.

```swift
  let db = try await client.db3.update(db3Id, title: "Rivera household")
  print(db.title, db.modifiedAt)
```

```swift
  var ids = Set<String>()
  var page = try await client.db3.listMine()
  while true {
    for db in page.items { ids.insert(db.id) } // db.via: .held, .shared or .app
    if !page.hasMore { break }
    page = try await client.db3.listMine(Db3ListMineOptions(cursor: page.nextCursor))
  }

  // Only the ones shared with this user:
  let shared = try await client.db3.listMine(Db3ListMineOptions(via: [.shared]))
```

A server function lists a named member's databases with `ctx.api.db3.listHeldByUser({ userId })` and `ctx.api.db3.listSharedWithUser({ userId })` (the `held` and `shared` entries, sync-off databases included in the held one; filter with `tag` and `type`, and keep following `nextCursor` past short pages), and resolves, binds and removes their aliases with `ctx.api.db3.aliases.resolve({ scope: "user", aliasKey, userId })`, `set({ scope: "user", aliasKey, body: { documentId, userId } })` and `delete({ scope: "user", aliasKey, userId })`. See the [Server Functions guide](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#a-members-collections-databases-and-profile).

### Aliases

An alias is a per-user key (`scope: "user"`) that names one database. `getOrCreateWithAlias` finds or creates the database behind it; `client.db3.aliases` binds a key to an existing database (`set`), reads it back (`resolve`: the database's row with `alias`, or`nil`when the key names nothing) and removes it (`delete`; the database stays, and a missing key is not an error).

```swift
  _ = try await client.db3.aliases.set(SetAliasParams(scope: .user, aliasKey: "archive", documentId: db3Id))

  // The database's row and the alias, or nil when the key names nothing.
  let archive = try await client.db3.aliases.resolve(AliasRef(scope: .user, aliasKey: "archive"))

  // Removes the key; the database stays.
  try await client.db3.aliases.delete(AliasRef(scope: .user, aliasKey: "archive"))
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

```swift
  let inbox = try await client.db3.getOrCreateWithAlias(
    Db3GetOrCreateWithAliasOptions(
      alias: AliasRef(scope: .user, aliasKey: "inbox", userId: userId),
      title: "Inbox",
      type: "inbox",
      permission: .reader
    )
  )
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
- **A new database's uploads wait for the server to have it** — an upload right after `create` is queued (`bytesTransferred: 0`) and sent once the create commits; await `client.db3.commitCreate` first to upload at once.
- **`confirm: true` on a device write waits for the next subscribe** when the database isn't subscribed.
- **Upgrade client libraries forward only.** A local copy written by a newer client doesn't open in an older one; reload web tabs left open across a deploy.
- **Generated fields are optional in practice** — older records may lack newer fields.
- **App access and grants are refused on a sync-off database** (`DB3_SYNC_OFF_NO_SHARING`).
- **An app-held database has no owner** — nobody can open it until it is shared, given app access, or transferred.
- **Access-request events are not replayed** after a disconnect — list on every connect.
