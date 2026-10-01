# Agent Guide to Primitive Resource Metadata

Resource metadata stores typed values about users, groups, collections, and databases. Define categories with schemas and read/write rules, then read or replace each category’s values.

## Category configs

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

App owners and admins bypass category rules. Resource owners and managers do not. Use `resource.attrs` only for supported database and collection attributes.

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
- **Reads are polling-only.** Metadata reads return point-in-time values; changes are not pushed to clients, and there is no subscription surface. Poll on whatever interval the use case's freshness requires. State that must react in real time on the client belongs in a [document](AGENT_GUIDE_TO_PRIMITIVE_DOCUMENTS.md) — connected clients receive document updates automatically; metadata is for configuration and state the server reads.

## Declaring metadata for CEL rules

A rule reads a resource's own metadata as `md.self.<category>.<key>`. These self-reads are **inferred** — referencing a category loads it automatically, so you don't declare it on the owning config (a group type, collection type, or database type). An explicit `[metadata.self]` declaration is also supported and unions with the inferred set (declare a category the rule doesn't name directly, and it loads too). A category's own `readRule` gates the read API only, not a rule author's use of it (trusted-author model).

```toml
# on the owning config (group-type-config shown; same shape for collection/database type configs)
[metadata.self]
categories = ["config"]
```

```toml novalidate
member.create = "md.self.config.tier == \"pro\""
```

### A category rule's own manifest

The declaration above is for *other* configs (group/collection/database type) reading the resource they're attached to. A metadata **category config** can declare the same manifest shape on itself, letting its `readRule`/`writeRule` reach the resource's *other* categories:

```toml
# primitive/dev/metadata-category-configs/class-post.post.toml
[metadataCategoryConfig]
resourceType = "class-post"
category = "post"
readRule = "isMemberOf('class-teachers', md.self.classLink.classId)"
writeRule = "isMemberOf('class-teachers', md.self.classLink.classId)"

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
group.get = "md.self.attrs.createdBy == user.userId"
```

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
access = "md.caller.billing.status in ['trialing', 'active', 'past_due']"
```

`md.caller` is bindable in database trigger `when`/`set`, and in a **group or collection rule set** whose traversal path declares `rootFrom = "user.userId"` (the rule-set example below). With no authenticated caller, `md.caller` binds `null` — a rule that dereferences it denies rather than erroring; guard explicitly (`md.caller != null && ...`) on a strict evaluation path where errors surface instead of denying.

### Declaring a path on the rule entry (rule sets)

A config-level `[metadata.paths.*]` manifest applies to every rule in the
config. On a **rule set**, a single operation can carry its own declaration
instead: the operation value may be a `{ expr, loads }` table rather than a
bare CEL string, where `loads.paths` takes the same shape as
`[metadata.paths.*]`.

```toml novalidate
[rules.member]
create = "true"

[rules.member.list]
expr = "md.caller.billing.status == 'active'"

[rules.member.list.loads.paths.caller]
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

A server function reads, writes, deletes, and reverse-resolves metadata through `ctx.api.resourceMetadata` — the same operations as `client.resourceMetadata`, on the app's own authority. Function code acts as the system, which is owner-equivalent, so category `readRule`/`writeRule` are bypassed exactly as they are for an app owner or admin; the function's own `access` gate is the authorization, and nothing is declared in `capabilities`.

That is how to make a server-owned category: give it a `writeRule` no client satisfies (`writeRule = "false"`), keep `readRule` as open as clients need, and write it only from the function that owns it (e.g. the webhook-triggered function that records a payment provider's customer id). There is no rule-side way to name the one function allowed to write; keep writes to such a category in one function by convention.

## Create-time initial metadata

A collection, database, or group create can stamp metadata in the same operation — category name → values, applied once the resource exists (so `md.self` resolves) instead of via a follow-up write. From the CLI:

```bash
primitive databases create "Class Roster" --type roster --initial-metadata '{"settings":{"visibility":"class-only"}}'
primitive collections create "Class 42" --initial-metadata '{"settings":{"visibility":"class-only"}}'
primitive groups create --type class-reading-group --name "Reading group" --initial-metadata '{"classLink":{"classId":"class-A"}}'
```

- Each category is schema-validated **before** the resource is created — an invalid entry fails the whole create (all-or-nothing), not a partial create with dropped metadata.
- The category's `writeRule` is **waived** for this stamp — creation authority already covers it. The waiver is unreachable from the regular REST write route and applies only to the resource the create makes, so it can't be used to bypass `writeRule` on an existing resource.
- Capped at 10 categories per create.

In the client, `collections.create()` and `groups.create()` take an optional `initialMetadata` — category name → that category's values:

```swift
  let collection = try await client.collections.create(
    params: CreateCollectionParams(
      name: "Class 42",
      collectionType: "class",
      initialMetadata: ["settings": ["visibility": .string("class-only")]]
    )
  )
```

```swift
  let group = try await client.groups.create(
    params: CreateGroupParams(
      groupType: "class-reading-group",
      name: "Reading group",
      initialMetadata: ["classLink": ["classId": .string("class-A")]]
    )
  )
```

Staging `initialMetadata` at create time can also gate the create rule itself — see [Gating Collection Creation on Staged Metadata](AGENT_GUIDE_TO_PRIMITIVE_DOCUMENTS.md#gating-collection-creation-on-staged-metadata) in the Documents guide and [Gating group creation on staged metadata](AGENT_GUIDE_TO_PRIMITIVE_USERS_AND_GROUPS.md#gating-group-creation-on-staged-metadata) in the Users and Groups guide. A database create has no caller create rule to gate.

A group's stamp only writes fresh rows. Group metadata is keyed by `groupId` alone (no type), so a category already stored under that id — another type's group, or a deleted group's retained metadata — fails the create with `409 METADATA_EXISTS`; neither the group nor any metadata is created, and the stored row is untouched. Pick another `groupId`, or delete the stored category first.

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
- Expecting a metadata change to appear on other clients without a new read — updates are never pushed; poll for freshness, or model live client state in a document instead.
- Using `hasCollectionAccess` in a category rule expecting it to resolve — it's rejected at save time; it only works in collection rules.
- Expecting the create-time initial-metadata `writeRule` waiver to also apply to a later write on the same resource — it's create-only and unreachable once the resource exists.

## Related guides

- **access-control** — the shared CEL identity context every rule builds on, including the membership helpers
- **users-and-groups** — group/collection `metadataManifest` and the projected `attrs` category
- **server-functions** — `ctx.api.resourceMetadata` and the system authority it runs with
- **app-secrets** — the declared-only `secrets.*` binding a category manifest can also reach
