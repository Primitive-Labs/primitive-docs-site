# Agent Guide to Primitive Resource Metadata

Guidelines for AI agents attaching typed, access-controlled metadata to a resource — a user, a group, a collection, a database, or a workflow's subject. Metadata is grouped into named **categories**; each category has its own schema and its own CEL `readRule`/`writeRule`, so different data about the same resource can carry different rules (a self-editable `profile` category vs. a workflow-only `billing` category).

## Category configs

A category is defined once per `(resourceType, category)` — schema plus optional `readRule`/`writeRule` — and synced like any other config, one file per category:

```toml
# config/metadata-category-configs/user.profile.toml
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
primitive sync push
```

- **Field types:** `string`, `number`, `boolean`, `date`, `id`, `stringset`. `enum` (string array) is valid only on a `string` field; other supported constraints are `required`, `maxLength`, `maxCount`.
- **Category name `attrs` is reserved** — it's the read-only projected category (see **`md.self.attrs`** below), not a category you define.
- **Limits:** up to 100 keys per category, 16 KB per category item.
- **`readRule`/`writeRule` context:** `user.userId`, `user.role` (the caller), `resource.resourceType`, `resource.resourceId` (also bound as `resource.id`), `resource.category` — plus `workflow.workflowKey` when the call originates from a `metadata.write`/`metadata.read` step (so `fromWorkflow('key')` works). When the subject is a `database`, `workflow`, or `collection`, the rule can also read the resource's own columns via **`resource.attrs.<column>`** — `database`: `databaseId`, `databaseType`, `createdBy`; `workflow`: `workflowId`, `workflowKey`, `runAs`, `createdBy`; `collection`: `collectionId`, `collectionType`, `contextId`, `name`, `createdBy`. The canonical use is creator bootstrap: `writeRule = "user.userId == resource.attrs.createdBy"`. The subject row loads lazily (a rule that never references `resource.attrs` issues no extra read) and the binding fails closed: any other resource type, an unmapped column, or a missing row denies. The membership helpers `isMemberOf`/`memberGroups`/`hasRole` are also wired, so a rule can be group-scoped (`isMemberOf('class-teachers', resource.id)`) instead of only self-scoped; memberships load once, only when the rule references a membership helper (`hasRole` needs no load — it reads only `user.role`). `hasCollectionAccess` is rejected at save time in a category rule (collection-scoped only; it can never resolve here). An **app-level** owner or admin always bypasses both rules; a resource-level permission (e.g. a database's `owner`/`manager` grant) never bypasses — the rule itself is what authorizes resource-scoped callers. Omitting either rule defaults to deny.
- **Category authoring is admin-scoped** — define and update categories via TOML sync, or directly through the admin-gated `metadata-categories` REST route. `client.resourceMetadata` covers values only (`get`/`set`/`getBatch`/`list`/`delete`); the CLI's read-only `primitive metadata categories list`/`get` inspect the definitions without touching TOML.
- **A category rule can declare its own `metadataManifest`** (same `self`/`paths`/`secrets` shape as any other owning config) so it can reach the resource's *other* categories or declared secrets — see "A category rule's own manifest" below. Without one, the rule sees no `md`/`secrets` at all (unchanged from the base case).

## Values: read, write, batch read, list, delete

`get`/`set`/`delete` take `(resourceType, resourceId, category)` positionally, `list` takes `(resourceType, resourceId)`; `set` is a **full replace**, not a merge.

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
```
{{/lang}}

```bash
primitive metadata set user 01HXY... profile --data '{"tier":"pro","displayName":"Ada"}'
primitive metadata set user 01HXY... profile --data-file ./profile.json
primitive metadata get user 01HXY... profile --json
primitive metadata get-batch --resource user:01HXY...:profile,billing --resource user:01HZQ...:profile
primitive metadata get-batch --requests '[{"resourceType":"user","resourceId":"01HXY...","categories":["profile"]}]'
primitive metadata list user 01HXY...                # all stored categories (CLI reads as admin → shows everything)
primitive metadata delete user 01HXY... profile      # idempotent
primitive metadata categories list                   # read-only inspection of category definitions
primitive metadata categories get user profile      # adds the full schema JSON
```

- Batch: up to 50 resources, 200 expanded resource/category pairs per call — over either limit fails the **whole** call with `400 BATCH_TOO_LARGE` (checked before any read). Within limits the call is always `200`; per-item problems (missing category → `404 NOT_FOUND`, denied `readRule` → `403 FORBIDDEN`) surface inside `results[].categories[cat]`, never as a call-level failure.
- Errors on single read/write: `404 NOT_FOUND` (no such category on that resource type), `403 FORBIDDEN` (`readRule`/`writeRule` denied), `400` (schema validation failure, or writing the reserved `attrs` category → `RESERVED_CATEGORY`).
- All CLI text output (`keyValue`/`success`/`info`) goes to **stderr** — only `--json` output goes to stdout. A `primitive metadata get ... > out.txt` without `--json` produces an empty file.
- `set --data` and `--data-file` are mutually available; `--data-file` wins if both are passed. The payload must be a JSON object.
- **Reads are polling-only.** Metadata reads return point-in-time values; changes are not pushed to clients, and there is no subscription surface. Poll on whatever interval the use case's freshness requires. State that must react in real time on the client belongs in a [document](AGENT_GUIDE_TO_PRIMITIVE_DOCUMENTS.md) — connected clients receive document updates automatically; metadata is for configuration and state the server reads.

## Declaring metadata for CEL rules

A rule reads a resource's own metadata as `md.self.<category>.<key>` — but every category it reads must be **declared** on the owning config (a group type, collection type, database type, or workflow definition). Declaring is the authorization: a category's own `readRule` gates the read API only, not a rule author's use of it (trusted-author model).

```toml
# on the owning config (group-type-config shown; same shape for collection/database type configs and workflow definitions)
[metadata.self]
categories = ["config", "attrs"]
```

```toml novalidate
member.create = "md.self.config.tier == \"pro\""
```

### A category rule's own manifest

The declaration above is for *other* configs (group/collection/database type, workflow definition) reading the resource they're attached to. A metadata **category config** can declare the same manifest shape on itself, letting its `readRule`/`writeRule` reach the resource's *other* categories:

```toml
# config/metadata-category-configs/class-post.post.toml
[metadataCategoryConfig]
resourceType = "class-post"
category = "post"
readRule = "isMemberOf('class-teachers', md.self.classLink.classId)"
writeRule = "isMemberOf('class-teachers', md.self.classLink.classId)"

[metadataCategoryConfig.schema.fields.title]
type = "string"
required = true

[metadata.self]
categories = ["classLink"]
```

Every `md.self.<category>` the rule reads must be declared here — an undeclared reference is a save-time `400`, not a silent null. `secrets.<KEY>` follows the same declared-only rule as any other manifest-bearing config. A category config with no `metadataManifest` is unchanged — no `md`/`secrets` binds at all.

### `md.self.attrs` — the projected category

Groups and collections expose a reserved, read-only `attrs` category with no schema — declare it like any other category and it's populated from the resource's own fields, no write path:

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

`md.caller` is bindable in **database operation `access`** (including per-param access and batch), the database's own `metadataAccess` rule, DO-trigger `when`/`set`, and **workflow `accessRule`** (`start` and `workflow.call`). With no authenticated caller (anonymous request, or a `runAs:"system"` workflow), `md.caller` binds `null` — a rule that dereferences it denies rather than erroring; guard explicitly (`md.caller != null && ...`) on a strict evaluation path where errors surface instead of denying.

### Database operation substitution

Beyond CEL `access` rules, an operation's `definition` (`filter`/`data`) can substitute a declared category value directly, alongside `$params.*` / `$user.userId` / `$now` / `$steps.*`:

```toml novalidate
[operations.definition]
filter = { tier = "$md.self.profile.tier" }
```

`$md.self.<category>.<key>` resolves to `null` (not the literal string) when the category isn't declared on that database type's manifest, or the key is missing — same fail-closed convention as `$database.metadata.<key>`.

## Workflow steps: `metadata.write` / `metadata.read`

Both route through the exact same read/write path (and the same `readRule`/`writeRule` gate) as the client and CLI — no parallel authorization logic.

```toml
[[steps]]
id = "record-billing"
kind = "metadata.write"
resourceType = "user"
resourceId = "{{ input.userId }}"
category = "billing"
saveAs = "output"
[steps.data]
stripeCustomerId = "{{ input.customerId }}"
status = "active"
# Output: { ok, resourceType, resourceId, category, data, schemaVersion, size }

[[steps]]
id = "load-billing"
kind = "metadata.read"
resourceType = "user"
resourceId = "{{ input.userId }}"
category = "billing"
saveAs = "output"
# Output: { ok, resourceType, resourceId, category, data, schemaVersion, exists }
```

- `resourceType`/`resourceId`/`category`/`data` are all templated like any other step's params.
- `metadata.write` is a **full replace** of the category, matching the REST `PUT` semantics.
- Gate a category to exactly one workflow with `fromWorkflow('workflowKey')` in its `writeRule`/`readRule` — a REST call or a different workflow gets `403`. The workflow identity here is a privileged, call-local value the step runner passes in-process; it is never derived from a request header (an `X-Workflow-Context` header on a REST call has no effect).
- Error behavior mirrors `database.*` steps: a 4xx (schema validation, rule denial, reserved category, bad segment) is **non-retryable**; a 429 or 5xx is retryable.
- A `runAs:"system"` run's metadata calls carry no `user.*` context (empty `{}`) and get no app-level owner/admin bypass — only `fromWorkflow('key')` can authorize a system-run write/read. A `runAs:"caller"` run's calls still get the app-level owner/admin bypass.

## Create-time initial metadata

A collection or database create can stamp metadata in the same operation — category name → values, applied once the resource exists (so `md.self` resolves) instead of via a follow-up write. From the CLI:

```bash
primitive databases create "Class Roster" --type roster --initial-metadata '{"settings":{"visibility":"class-only"}}'
primitive collections create "Class 42" --initial-metadata '{"settings":{"visibility":"class-only"}}'
```

- Each category is schema-validated **before** the resource is created — an invalid entry fails the whole create (all-or-nothing), not a partial create with dropped metadata.
- The category's `writeRule` is **waived** for this stamp — creation authority already covers it. The waiver is unreachable from the regular REST write route: it never accepts a caller-supplied `resourceId`, so it can't be used to bypass `writeRule` on an existing resource.
- Capped at 10 categories per create.

{{#lang ts}}
In the client, `collections.create()` / `databases.create()` take an optional `initialMetadata: Record<string, Record<string, unknown>>`:

```typescript
const database = await client.databases.create({
  title: "Class Roster",
  databaseType: "roster",
  initialMetadata: {
    settings: { visibility: "class-only" },
  },
});
```
{{/lang}}

## Metadata lifecycle

A write never checks that the target resource exists — a write for a not-yet-created or already-deleted resource succeeds silently, and deleting a resource does not delete its metadata (no cascade). This is deliberate: it keeps a write to a single cheap put, and doesn't race provisioning flows that write metadata immediately after — or interleaved with — creating the resource itself.

**Consequences an implementation should account for:**
- Gate writes with a category `writeRule` (owner-only, or `fromWorkflow('key')`) to limit who can create dangling metadata in the first place.
- Whatever flow deletes a resource must also delete that resource's metadata categories — the platform doesn't do it for you.

## Anti-patterns

- Reading `md.self.<category>` in a rule without declaring `categories` for it in the config's `[metadata.self]` — the reference fails to save (400), not silently returns null.
- Expecting a category's own `readRule` to restrict what a *different* rule can read via `md.self`/`md.caller` — it only gates the read API; declaration is the authorization for CEL use (trusted-author model).
- Assuming `md.caller` works inside a group or collection rule set — it isn't bound there; it's for database operations and workflow `accessRule`.
- Relying on resource deletion to clean up its metadata — there's no cascade; delete metadata explicitly in the same flow.
- Trying to create or list category configs from the client SDK — that surface is TOML-sync/admin-REST only.
- Treating `set()` as a merge — it's a full replace of the category's data.
- Expecting a metadata change to appear on other clients without a new read — updates are never pushed; poll for freshness, or model live client state in a document instead.
- Using `hasCollectionAccess` in a category rule expecting it to resolve — it's rejected at save time; it only works in collection rules.
- Expecting the create-time initial-metadata `writeRule` waiver to also apply to a later write on the same resource — it's create-only and unreachable once the resource exists.

## Related guides

- **access-control** — the shared CEL identity context every rule builds on, including the membership helpers
- **users-and-groups** — group/collection `metadataManifest` and the projected `attrs` category
- **databases** — `md.self`/`md.caller` in operation `access` and `$md.self.*` substitution
- **workflows** — the `metadata.write`/`metadata.read` step shapes and `fromWorkflow()`
- **app-secrets** — the declared-only `secrets.*` binding a category manifest can also reach
