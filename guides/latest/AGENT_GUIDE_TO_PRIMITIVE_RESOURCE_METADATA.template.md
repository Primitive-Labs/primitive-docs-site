# Agent Guide to Primitive Resource Metadata

Guidelines for AI agents attaching typed, access-controlled metadata to a resource — a user, a group, a collection, or a database. Metadata is grouped into named **categories**; each category has its own schema and its own CEL `readRule`/`writeRule`, so different data about the same resource can carry different rules (a self-editable `profile` category vs. a server-function-only `billing` category).

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

- **Field types:** `string`, `number`, `boolean`, `date`, `id`, `stringset`. `enum` (string array) is valid only on a `string` field; other supported constraints are `required`, `maxLength`, `maxCount`.
- **`unique`:** set `unique = true` on one `string`/`id` field (at most one per category) to enforce that no two resources of that type share the value AND make the value reverse-resolvable (`client.resourceMetadata.resolve` / `primitive metadata resolve` / `ctx.api.resourceMetadata.resolve` from a function, all below). The indexed value is capped at 512 UTF-8 bytes. Writing a value another resource already owns is rejected `409` (atomic — the write rolls back); rewriting the same value on the same resource is idempotent; clearing the field or deleting the row frees the value in the same write. Enabling `unique` on a category that already has rows is refused (they'd be unindexed) — declare it at category-creation time, or recreate the category.
- **Category name `attrs` is reserved** — it's the read-only projected category (see **`md.self.attrs`** below), not a category you define.
- **Limits:** up to 100 keys per category, 16 KB per category item.
- **`readRule`/`writeRule` context:** `user.userId`, `user.role` (the caller), `resource.resourceType`, `resource.resourceId` (also bound as `resource.id`), `resource.category`. When the subject is a `database` or `collection`, the rule can also read the resource's own columns via **`resource.attrs.<column>`** — `database`: `databaseId`, `databaseType`, `createdBy`; `collection`: `collectionId`, `collectionType`, `contextId`, `name`, `createdBy`. The canonical use is creator bootstrap: `writeRule = "user.userId == resource.attrs.createdBy"`. The subject row loads lazily (a rule that never references `resource.attrs` issues no extra read) and the binding fails closed: any other resource type, an unmapped column, or a missing row denies. The membership helpers `isMemberOf`/`memberGroups`/`hasRole` are also wired, so a rule can be group-scoped (`isMemberOf('class-teachers', resource.id)`) instead of only self-scoped; memberships load once, only when the rule references a membership helper (`hasRole` needs no load — it reads only `user.role`). `hasCollectionAccess` is rejected at save time in a category rule (collection-scoped only; it can never resolve here). An **app-level** owner or admin always bypasses both rules; a resource-level permission (e.g. a database's `owner`/`manager` grant) never bypasses — the rule itself is what authorizes resource-scoped callers. Omitting either rule defaults to deny.
- **Category authoring is admin-scoped** — define and update categories via TOML sync, or directly through the admin-gated `metadata-categories` REST route. `client.resourceMetadata` covers values only (`get`/`set`/`getBatch`/`list`/`delete`/`resolve`); the CLI's `primitive metadata-category-configs list`/`get` inspect the definitions read-only (the `metadata` noun carries value verbs only), and a definition is removed by deleting its `metadata-category-configs/<resourceType>.<category>.toml` and running `primitive config push --prune` — there is no delete verb. Deleting a definition is a hard delete of the definition only — stored value rows are **not** deleted and, with no query path from a category to its values, become **unreachable** (reads/writes `404`, rows can't be removed by any surface). Delete the values first (`primitive metadata delete` / `resourceMetadata.delete`) if you need them gone. Re-creating the same `{resourceType, category}` resurfaces orphaned rows bound to the new schema (possibly stale/mismatched on read).
- **A category rule can declare its own `metadataManifest`** (same `self`/`paths`/`secrets` shape as any other owning config) so it can reach declared secrets or a traversal path's source category — see "A category rule's own manifest" below. Without one, the rule still gets inferred `md.self` reads but binds no `secrets`/`vars`.

## Values: read, write, batch read, list, delete, resolve

`get`/`set`/`delete` identify the value by resource type, resource id, and category; `list` takes just the resource type and id. `set` is a **full replace**, not a merge.

{{#lang ts}}
```typescript
await client.resourceMetadata.set("user", userId, "profile", { tier: "pro", displayName: "Ada" });
// -> { resourceType, resourceId, category, data, schemaVersion, size }

const profile = await client.resourceMetadata.get("user", userId, "profile");
// -> { resourceType, resourceId, category, data, schemaVersion, exists }

const { results } = await client.resourceMetadata.getBatch({
  requests: [
    { resourceType: "user", resourceId: userA, categories: ["profile", "billing"] },
    { resourceType: "user", resourceId: userB, categories: ["profile"] },
  ],
});
// results[i] = { resourceType, resourceId, ok: true,
//   categories: { <category>: { ok: true, exists, data, schemaVersion }
//               | { ok: false, status, code, message } } }
// A whole-resource failure (e.g. a malformed id) instead returns
// { resourceType, resourceId, ok: false, status, code, message } with no `categories`.

const { categories } = await client.resourceMetadata.list("user", userId);
// -> { resourceType, resourceId, categories: [{ category, data, schemaVersion }, ...] }
// Only categories whose readRule permits the caller are returned (owner/admin sees all);
// a resource with no stored metadata returns an empty array, not an error.

const { deleted } = await client.resourceMetadata.delete("user", userId, "profile");
// -> { resourceType, resourceId, category, deleted } — idempotent: absent item → deleted: false, not a 404.

const hit = await client.resourceMetadata.resolve({
  resourceType: "user", category: "billing", key: "stripeCustomerId", value: "cus_ABC",
});
// -> { resourceId, resourceType } on a hit; { resourceId: null } on a miss (a miss is a 200, never an error).
// The category readRule is evaluated against the RESOLVED resource and a denied read returns the
// same { resourceId: null } — the response BODY carries no distinguisher. Not constant-time,
// though: a denied resolve does the rule evaluation and its lookups a miss skips, so latency can
// still separate the two. A key that is not the category's active unique field is 400
// NOT_UNIQUE_FIELD (a config error, distinct from a miss).
```
{{/lang}}
{{#lang swift}}
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
{{/lang}}

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
| `collection` | `collectionId`, `collectionType`, `contextId`, `name`, `createdBy` |

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

A collection or database create can stamp metadata in the same operation — category name → values, applied once the resource exists (so `md.self` resolves) instead of via a follow-up write. From the CLI:

```bash
primitive databases create "Class Roster" --type roster --initial-metadata '{"settings":{"visibility":"class-only"}}'
primitive collections create "Class 42" --initial-metadata '{"settings":{"visibility":"class-only"}}'
```

- Each category is schema-validated **before** the resource is created — an invalid entry fails the whole create (all-or-nothing), not a partial create with dropped metadata.
- The category's `writeRule` is **waived** for this stamp — creation authority already covers it. The waiver is unreachable from the regular REST write route: it never accepts a caller-supplied `resourceId`, so it can't be used to bypass `writeRule` on an existing resource.
- Capped at 10 categories per create.

In the client, `collections.create()` takes an optional `initialMetadata` — category name → that category's values:

{{ example: documents/collection-initial-metadata }}

On a collection specifically, staging `initialMetadata` at create time can also gate the `collection.create` rule itself — see [Gating Collection Creation on Staged Metadata](AGENT_GUIDE_TO_PRIMITIVE_DOCUMENTS.md#gating-collection-creation-on-staged-metadata) in the Documents guide. A database create has no caller create rule to gate; `group.create` takes no `initialMetadata`.

## Metadata lifecycle

A write never checks that the target resource exists — a write for a not-yet-created or already-deleted resource succeeds silently, and deleting a resource does not delete its metadata (no cascade). This is deliberate: it keeps a write to a single cheap put, and doesn't race provisioning flows that write metadata immediately after — or interleaved with — creating the resource itself.

**Consequences an implementation should account for:**
- Gate writes with a category `writeRule` (owner-only, or `"false"` for a category only server functions write) to limit who can create dangling metadata in the first place.
- Whatever flow deletes a resource must also delete that resource's metadata — the platform doesn't do it for you. Delete each category with `delete` (client/CLI) or `ctx.api.resourceMetadata` from a function; a teardown function that deletes the resource deletes its categories in the same invocation.
- **Order deletes correctly.** Delete a category's *values* before deleting the category *config* — once the config is gone, the values are orphaned and can no longer be deleted through any surface. And when a `writeRule` reads `resource.attrs.<column>`, delete the metadata before the owning resource — the rule loads the resource's own columns to authorize the delete, so it fails closed once the resource row is gone.

## Anti-patterns

- Declaring `[metadata.self] categories` for a plain `md.self.<category>` read — it's redundant, since self-reads are inferred; declaration matters only to load a category the rule never names, or to serve as the source category for a declared traversal path's `via` key. A **traversal path** (`md.<pathName>`) is the opposite: it's never inferred, so referencing one without a `[metadata.paths.*]` declaration is an undeclared reference. `secrets.<KEY>` likewise must be in the config's `secrets` allowlist. How an undeclared path or out-of-allowlist secret fails depends on the rule site: a **metadata category `readRule`/`writeRule`** is linted at save time (a save-time 400), but a **group/collection rule set** is not — there the undeclared reference binds `null` and denies at runtime instead.
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
