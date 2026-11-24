# Agent Guide to Primitive Resource Metadata

> **Status: experimental, deprecated.** Experimental. Production does not serve this; every other environment does. Deprecated. Use user properties, or a database keyed by the group or collection id, instead.

Resource metadata stores typed values about users, groups, and collections. Define categories with schemas and read/write rules, then read or replace each category’s values.

Do not build new features on it. Store per-user data as user properties (`user-properties.toml`, `ctx.users`), and group or collection data in a db3 database keyed by the group or collection id. Existing calls keep working and are marked deprecated in the typings.

## Category configs

> **Status: beta, deprecated.** Beta. Served in production; the API can still change. Deprecated. Use user properties, or a database keyed by the group or collection id, instead.

A category is defined once per `(resourceType, category)` — schema plus optional `readRule`/`writeRule` — and synced like any other config, one file per category:

```toml
# primitive/dev/metadata-category-configs/user.profile.toml
[metadataCategoryConfig]
resourceType = "user"
category = "profile"
readRule = "user.userId == resource.resourceId"
writeRule = "user.userId == resource.resourceId"
description = "Per-user profile metadata"

[metadataCategoryConfig.schema.fields.tier]
type = "string"
required = true
enum = ["free", "pro", "enterprise"]

[metadataCategoryConfig.schema.fields.displayName]
type = "string"
maxLength = 120
```

```bash
primitive config push
```

| Setting | Meaning |
|---|---|
| Field type | `string`, `number`, `boolean`, `date`, `id`, or `stringset` |
| `readRule` / `writeRule` | CEL conditions; omitted rules deny regular members |
| `unique` | One `string` or `id` field per category; declare before storing values |
| Limits | 100 keys and 16 KB per category item; unique values up to 512 UTF-8 bytes |

App owners and admins bypass category rules. Resource owners and managers do not. Use `resource.attrs` only for supported group and collection attributes.

## Values: read, write, batch read, list, delete, resolve

`get`/`set`/`delete` identify the value by resource type, resource id, and category; `list` takes just the resource type and id. `set` is a **full replace**, not a merge.

```swift
_ = try await client.resourceMetadata.set(
    resourceType: "user", resourceId: userId, category: "profile",
    data: ["tier": "pro", "displayName": "Ada"]
)
// -> ResourceMetadataWriteResult { resourceType, resourceId, category, data, schemaVersion, size }

let profile = try await client.resourceMetadata.get(
    resourceType: "user", resourceId: userId, category: "profile"
)
// -> ResourceMetadataReadResult { resourceType, resourceId, category, data, schemaVersion, exists }
// `data` is [String: JSONValue]; check `exists` before reading it. Nothing stored is
// exists == false + empty data + schemaVersion == nil, NOT an error.

let batch = try await client.resourceMetadata.getBatch(requests: [
    .init(resourceType: "user", resourceId: userA, categories: ["profile", "billing"]),
    .init(resourceType: "user", resourceId: userB, categories: ["profile"]),
])
// batch.results[i] = { resourceType, resourceId, ok,
//   categories: [<category>: { ok, data, schemaVersion, exists, status, code, message }] }
// The unions decode as flag-carrying structs: branch on `ok`, or read the `error`
// accessor (nil when the entry succeeded) for the status/code/message triple. A
// whole-resource failure has ok == false, an `error`, and no `categories`.

let listed = try await client.resourceMetadata.list(resourceType: "user", resourceId: userId)
// -> { resourceType, resourceId, categories: [{ category, data, schemaVersion }] }
// Only categories whose readRule permits the caller are returned (owner/admin sees all);
// a resource with no stored metadata returns an empty array, not an error.

let removed = try await client.resourceMetadata.delete(
    resourceType: "user", resourceId: userId, category: "profile"
)
// -> { resourceType, resourceId, category, deleted } — idempotent: absent item → deleted == false, not a 404.

let hit = try await client.resourceMetadata.resolve(
    resourceType: "user", category: "billing", key: "stripeCustomerId", value: "cus_ABC"
)
// -> ResourceMetadataResolveResult { resourceId, resourceType } — both nil on a miss.
// Read `hit.resolved` for the hit arm as one value (nil on a miss). A miss is a 200,
// never an error. The category readRule is evaluated against the RESOLVED resource and a
// denied read returns the same nil resourceId — the response BODY carries no
// distinguisher. Not constant-time, though: a denied resolve does the rule evaluation and
// its lookups a miss skips, so latency can still separate the two. A key that is not the
// category's active unique field throws HttpError with status == 400 and
// serverCode == "NOT_UNIQUE_FIELD" (a config error, distinct from a miss).
```

A `readRule`/`writeRule` denial on the single read/write calls (`get`/`set`/`list`/`delete`) throws `HttpError` with `status == 403`; inside `getBatch` the same denial is a per-entry `ok: false` and the call still succeeds. `resolve` is the exception: a denial there is reported as a miss (see above), never a 403.

```bash
primitive metadata set user 01HXY... profile --data '{"tier":"pro","displayName":"Ada"}'
primitive metadata set user 01HXY... profile --data-file ./profile.json
primitive metadata get user 01HXY... profile --json
primitive metadata get-batch --resource user:01HXY...:profile,billing --resource user:01HZQ...:profile
primitive metadata get-batch --requests '[{"resourceType":"user","resourceId":"01HXY...","categories":["profile"]}]'
primitive metadata list user 01HXY...                # all stored categories (CLI reads as admin → shows everything)
primitive metadata delete user 01HXY... profile      # idempotent
primitive metadata resolve user billing stripeCustomerId cus_ABC   # reverse lookup; miss prints "Not found" and exits 0
primitive metadata-category-configs list             # read-only inspection of category definitions
primitive metadata-category-configs get user profile # adds the full schema JSON
```

Deleting a definition is deleting its file plus a pruning push, scoped to that one definition with `--only` so the rest of the tree is left alone (`--dry-run` first to see the plan):

```bash
rm primitive/dev/metadata-category-configs/user.profile.toml
primitive config push --only 'metadata-category-config/user#profile' --prune --dry-run
primitive config push --only 'metadata-category-config/user#profile' --prune --yes
```

The selector's key is the `resourceType#category` pair the file declared, NOT its dotted file name. `--prune` is what makes naming a deleted definition legal: the sync state still tracks it, and that is where the pruning push gets its candidates. Without `--prune` the same selector aborts with `No local file declares…`, since a scoped push with nothing to apply would report success having done nothing. Drop `--only` to prune every removed definition in one run.

- Batch: up to 50 resources, 200 expanded resource/category pairs per call — over either limit fails the **whole** call with `400 BATCH_TOO_LARGE` (checked before any read). Within limits the call is always `200`; per-item problems (missing category → `404 NOT_FOUND`, denied `readRule` → `403 FORBIDDEN`) surface inside `results[].categories[cat]`, never as a call-level failure.
- Errors on single read/write: `404 NOT_FOUND` (no such category on that resource type), `403 FORBIDDEN` (`readRule`/`writeRule` denied), `400` (schema validation failure, or writing the reserved `attrs` category → `RESERVED_CATEGORY`).
- All CLI text output (`keyValue`/`success`/`info`) goes to **stderr** — only `--json` output goes to stdout. A `primitive metadata get ... > out.txt` without `--json` produces an empty file.
- `set --data` and `--data-file` are mutually available; `--data-file` wins if both are passed. The payload must be a JSON object.
- **Reads are polling-only.** Metadata reads return point-in-time values; changes are not pushed to clients, and there is no subscription surface. Poll on whatever interval the use case's freshness requires. State that must react in real time on the client belongs in a synced [database](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md) — subscribed clients receive its changes automatically; metadata is for configuration and state the server reads.

## Declaring metadata for CEL rules

A rule reads a resource's own metadata as `md.self.<category>.<key>`. These self-reads are **inferred** — referencing a category loads it automatically, so you don't declare it on the owning config (a rule set, or a metadata category config). An explicit `[metadata.self]` declaration is also supported and unions with the inferred set (declare a category the rule doesn't name directly, and it loads too). A category's own `readRule` gates the read API only, not a rule author's use of it (trusted-author model).

```toml
# on the owning config (a metadata category config shown)
[metadata.self]
categories = ["config"]
```

```toml novalidate
writeRule = "md.self.config.tier == \"pro\""
```

### A category rule's own manifest

The declaration above can sit on a rule set or on a metadata category config; on a category config it lets its `readRule`/`writeRule` reach the resource's *other* categories:

```toml
# primitive/dev/metadata-category-configs/class-post.post.toml
[metadataCategoryConfig]
resourceType = "class-post"
category = "post"
readRule = "isMemberOf(md.self.classLink.classId + '-teachers')"
writeRule = "isMemberOf(md.self.classLink.classId + '-teachers')"

[metadataCategoryConfig.schema.fields.title]
type = "string"
required = true
```

`md.self.<category>` reads are inferred — you don't declare them here. A referenced category is loaded automatically and binds `null` when the subject lacks it. An explicit `[metadata.self] categories = [...]` block is also supported and unions with the inferred set (load a category no rule names directly). Traversal **paths** (`md.<pathName>`) are the exception — they must be declared under `[metadata.paths.*]`, since the reference alone doesn't carry the `via` key, target `type`, or categories. `secrets.<KEY>` is declared-only and bounded by the config's `secrets` allowlist: an out-of-allowlist reference is a save-time `400`, and secrets are never inferred past the allowlist. A category config with no `metadataManifest` binds no `secrets`/`vars` at all.

### `md.self.attrs` — the projected category

Groups and collections expose a reserved, read-only `attrs` category with no schema — reference it like any other category and it's populated from the resource's own fields, no write path:

| Resource type | `md.self.attrs.*` |
|---|---|
| `group` | `groupType`, `groupId`, `name`, `createdBy` |
| `collection` | `collectionId`, `collectionType`, `name`, `createdBy` |

```toml novalidate
writeRule = "md.self.attrs.createdBy == user.userId"
```

**Subject-form membership in a metadata rule** — a category's `readRule`/`writeRule` can ask which groups a *loaded subject* belongs to, not just the caller: `memberGroupsOf(<subjectId>, '<groupType>')` returns that subject's list of groupIds of the type (a deprecated alias kept for metadata rules; new code checks membership with `isMemberOf`/`memberGroups`, one argument). Compose with `.exists()`/`.all()` to follow a relationship graph, e.g. grant a teacher of any class a student's metadata links to:

```toml novalidate
readRule = "memberGroupsOf(md.self.classLink.classId, 'class-students').exists(c, isMemberOf(c))"
```

`memberGroupsOf`'s subject must be a literal path such as `md.self.<category>.<key>`, and its type a literal string.

### Declared paths — a related resource's metadata

A manifest can chain to another resource (up to 3 hops) via a metadata key holding its id:

```toml
[metadata.self]
categories = ["system"]

[metadata.paths.school]
from = "self"          # "self" or another already-declared path name
via = "system.schoolId" # "<category>.<key>" on the source; that category must be declared there
type = "school"         # the target's resourceType
categories = ["system"]
```

```toml novalidate
md.school.system.tier == "pro"
```

### `md.caller` — the calling user's own metadata

One path root is server-authenticated rather than caller-supplied: `rootFrom = "user.userId"` (requires `type = "user"`; any other `user.*` root is rejected at save time). Every other root (`input.*`, `record.*`, `params.*`) is a static, caller-declared context path, not an identity binding.

```toml
[metadata.paths.caller]
rootFrom = "user.userId"
type = "user"
categories = ["billing"]
```

```toml novalidate
list = "md.caller.billing.status in ['trialing', 'active', 'past_due']"
```

`md.caller` is bindable in a metadata category's `readRule`/`writeRule` whose traversal path declares `rootFrom = "user.userId"` (the rule entry example below). With no authenticated caller, `md.caller` binds `null` — a rule that dereferences it denies rather than erroring; guard explicitly (`md.caller != null && ...`) on a strict evaluation path where errors surface instead of denying.

### Declaring a path on the rule entry (rule sets)

A config-level `[metadata.paths.*]` manifest applies to every rule in the
config. On a **rule set**, a single operation can carry its own declaration
instead: the operation value may be a `{ expr, loads }` table rather than a
bare CEL string, where `loads.paths` takes the same shape as
`[metadata.paths.*]`.

```toml novalidate
[rules.blob]
write = "true"

[rules.blob.list]
expr = "md.caller.billing.status == 'active'"

[rules.blob.list.loads.paths.caller]
rootFrom = "user.userId"
type = "user"
categories = ["billing"]
```

Both forms resolve through the same loader, so `md.<name>` binds identically;
the table form only keeps a one-rule declaration next to its rule. Bare-string
and table entries coexist in one category block and `primitive config`
round-trips both byte-identically.

Constraints: only `loads.paths` is accepted on a rule entry — `loads.secrets`
and `loads.vars` are config-level and rejected here. An empty `loads` or
`loads.paths` collapses back to a bare expression, so the two spellings carry
identical intent.

## From a server function

A server function reads, writes, deletes, and reverse-resolves metadata through `ctx.api.resourceMetadata` — the same operations as `client.resourceMetadata`, on the app's own authority. Function code acts as the system, which is owner-equivalent, so category `readRule`/`writeRule` are bypassed exactly as they are for an app owner or admin; the function's own `authorize` method is the authorization, and nothing is declared in `capabilities`.

That is how to make a server-owned category: give it a `writeRule` no client satisfies (`writeRule = "false"`), keep `readRule` as open as clients need, and write it only from the function that owns it (e.g. the webhook-triggered function that records a payment provider's customer id). There is no rule-side way to name the one function allowed to write; keep writes to such a category in one function by convention.

## Create-time initial metadata

A collection or group create can stamp metadata in the same operation — category name → values, applied once the resource exists (so `md.self` resolves) instead of via a follow-up write. From the CLI:

```bash
primitive collections create "Class 42" --initial-metadata '{"settings":{"visibility":"class-only"}}'
primitive groups create --name "Reading group" --initial-metadata '{"classLink":{"classId":"class-A"}}'
```

- Each category is schema-validated **before** the resource is created — an invalid entry fails the whole create (all-or-nothing), not a partial create with dropped metadata.
- The category's `writeRule` is **waived** for this stamp — creation authority already covers it. The waiver is unreachable from the regular REST write route and applies only to the resource the create makes, so it can't be used to bypass `writeRule` on an existing resource.
- Capped at 10 categories per create.

`collections.create()` takes an optional `initialMetadata` — category name → that category's values — from the client:

```swift
  let collection = try await client.collections.create(
    params: CreateCollectionParams(
      name: "Class 42",
      initialMetadata: ["settings": ["visibility": .string("class-only")]]
    )
  )
```

`groups.create()` takes the same `initialMetadata`, but only a server function or the admin API calls it — see [Users and Groups](AGENT_GUIDE_TO_PRIMITIVE_USERS_AND_GROUPS.md#creating-groups) — so from the CLI it's the same `--initial-metadata` flag shown above.

A group's stamp only writes fresh rows: a category already stored under the new group's id fails the create with `409 METADATA_EXISTS` (delete the stored category, or leave it out of `initialMetadata`), and a staged unique value another resource already holds fails it with `409 UNIQUE_VALUE_CONFLICT` (change the value). Either way neither the group nor any metadata is created, and a stored row is untouched.

## Copy a metadata category into properties

Declare and push the target [user properties](AGENT_GUIDE_TO_PRIMITIVE_USER_PROPERTIES.md) fields, add a batch function (`[function]` with `key = "copy-user-metadata"`, `entry = "functions/copy-user-metadata.ts"`), and run it as a [walk](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#walking-every-user). `primitive metadata report user` lists the categories holding rows (see the user-properties guide). The core of the function, per user in the batch:

```ts
// row: the user's category data, from ctx.api.resourceMetadata.getBatch (50 users per call)
const patch: Record<string, unknown> = {};
for (const [key, property] of Object.entries(input.fields)) {
  if (Object.hasOwn(row, key)) patch[property] = valueFor(key, row[key]); // valueFor returns raw unchanged
}
if (Object.keys(patch).length === 0) skipped++;
else {
  try {
    await ctx.users.setProperties(userId, patch);
    copied++;
  } catch (error) {
    const code = (error as PrimitivePlatformError).errorCode;
    if (code === "USER_PROPERTY_UNIQUE_CONFLICT" || code === "USER_PROPERTY_INVALID") refused.push({ userId, code });
    else throw code ? new Error(`${code}: ${(error as Error).message}`) : error; // fails the batch
  }
}
// the batch returns { copied, skipped, refused }
```

```bash
primitive users walk start --function copy-user-metadata --key copy-billing-1 --input '{"category":"billing","fields":{"customerId":"stripeCustomerId","plan":"plan"}}' --wait
```

- `authorize: allowCallers({ usersBatch: true })` — only a walk runs it, never a client with its own list of users.
- The walk's `input` is `{ category, fields: { <metadata key>: <property name> } }`. Throw at once when it's missing or malformed, so the walk fails with nothing written. From a function, start it with `ctx.users.forEachBatch({ function: "copy-user-metadata", key, input })`.
- A batch of up to 100 users costs one `getBatch` per 50 users plus one write per user with a row. A per-category answer with `ok: false` (for example, a deleted config) must throw `<code>: <message>`. Treating it as "no row" copies nothing silently.
- A unique conflict or a value that doesn't fit is one user's data problem: record it under `refused` and carry on. Any other error (for example `USER_PROPERTY_UNKNOWN`, an undeclared target) fails the batch, so the walk retries it and then fails `USER_WALK_BATCHES_FAILED`. Put the code in the thrown message, because the walk records only the message.
- A completed walk copied everyone not under `refused`. Sum the `refused` lists from the batch runs (`primitive functions runs <function-id> --json`) before treating the copy as done.
- Every write is a merge-patch of the row's current values, so a rerun under a new `key` is safe and picks up changed rows.

### Copy one property into a new one

To change a field's type or make it unique, declare the new field and run the same function with two changes. Read the source from the user instead of the category: `const row = (await ctx.users.get(userId))?.properties ?? {}`. And convert in `valueFor`, because a field keeps its type and a copy without conversion refuses every user. For example, use `Number(raw)` for text into a number when `Number.isFinite` holds for the result (a non-finite number would be written as `null` and clear the field), or `[raw]` for a scalar into a stringset. Return what you can't convert unchanged, so the write refuses it into `refused`. A unique target lists each conflicting user under `refused`; the walk still completes.

## Metadata lifecycle

A write never checks that the target resource exists — a write for a not-yet-created or already-deleted resource succeeds silently, and deleting a resource does not delete its metadata (no cascade). This is deliberate: it keeps a write to a single cheap put, and doesn't race provisioning flows that write metadata immediately after — or interleaved with — creating the resource itself.

**Consequences an implementation should account for:**
- Gate writes with a category `writeRule` (owner-only, or `"false"` for a category only server functions write) to limit who can create dangling metadata in the first place.
- Whatever flow deletes a resource must also delete that resource's metadata — the platform doesn't do it for you. Delete each category with `delete` (client/CLI) or `ctx.api.resourceMetadata` from a function; a teardown function that deletes the resource deletes its categories in the same invocation.
- **Order deletes correctly.** Delete a category's *values* before deleting the category *config* — once the config is gone, the values are orphaned and can no longer be deleted through any surface. And when a `writeRule` reads `resource.attrs.<column>`, delete the metadata before the owning resource — the rule loads the resource's own columns to authorize the delete, so it fails closed once the resource row is gone.

## Gotchas

- Declare traversal paths and secret keys before using them in rules. Plain `md.self.<category>` reads are inferred. Undeclared references can reject a category config or deny a group/collection operation.
- Expecting a category's own `readRule` to restrict what a *different* rule can read via `md.self`/`md.caller` — it only gates the read API, not CEL use of the value (trusted-author model).
- Relying on resource deletion to clean up its metadata — there's no cascade; delete metadata explicitly in the same flow.
- Trying to create or list category configs from the client SDK — that surface is TOML-sync/admin-REST only.
- Treating `set()` as a merge — it's a full replace of the category's data.
- Expecting a metadata change to appear on other clients without a new read — updates are never pushed; poll for freshness, or model live client state in a synced database instead.
- Declaring `[metadata.self]`/`[metadata.paths.*]` expecting it to gate a group or collection operation — groups and collections have no rule set to bind it to; see [Group Roles](AGENT_GUIDE_TO_PRIMITIVE_USERS_AND_GROUPS.md#group-roles) and collection roles for what governs them instead.
- Expecting the create-time initial-metadata `writeRule` waiver to also apply to a later write on the same resource — it's create-only and unreachable once the resource exists.

## Related guides

- **access-control** — the shared CEL identity context every rule builds on, including the membership helpers
- **users-and-groups** — group/collection `metadataManifest` and the projected `attrs` category
- **server-functions** — `ctx.api.resourceMetadata` and the system authority it runs with
- **app-secrets** — the declared-only `secrets.*` binding a category manifest can also reach
- **user-properties** — typed fields on the platform user, where the copy recipe above lands a category's values
