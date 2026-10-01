# Working with Documents in the Primitive platform.

A document stores model records locally, syncs changes, and defines a sharing boundary. Open it before querying or writing. Use separate documents when data needs separate access.

## Core Concept: Documents

Documents support offline reads and writes and automatically merge concurrent edits. Permissions apply to the whole document: `reader`, `read-write`, or `owner`.

## Documents vs. Databases

Use documents for local-first collaboration. Use databases for records served through server functions with caller-specific access. See [Data Modeling](AGENT_GUIDE_TO_PRIMITIVE_DATA_MODELING.md).

## Critical Rules

- Queries span open documents. Use the `documents` option to narrow them.
- Use model record IDs for relationships; document IDs select storage and sharing.
- Define models in `models.toml` and regenerate after changes. Keep custom code outside generated files.
- The root document is private, automatically opened, and cannot be shared or deleted. Use regular documents for shared application data.

## Document Lifecycle

### 1. Open Documents Before Querying

Documents must be opened before querying or modifying data within them.

{{ example: documents/doc-open-query }}

Await `open()` before reading or writing. Show a loading state while it runs, and handle failures before continuing. Close documents when they are no longer needed.

#### Gotchas when opening

- Queries span all open documents, so unnecessary opens also widen query results.
- An open requiring server state can fail offline or time out. Handle the error code and retry `open()` when connectivity returns.
- Open after authentication. A view or store being ready does not mean its documents are open.

{{#lang ts}}
Open session documents in the app layout or store; open route-specific documents on entry and close them on exit.
{{/lang}}
{{#lang swift}}
Open documents from `PrimitiveAppState.connectClient()`. See [The document lifecycle](#the-primitiveappstate-document-lifecycle).
{{/lang}}

### 2. Finding documents a user can access

List owned, directly shared, group-shared, and collection-shared documents separately, then combine them for navigation.

**a. Documents they own** (`ownedDocuments` — created, or ownership transferred):

{{ example: documents/list-owned }}

**b. Documents shared directly with them** (`sharedDocuments` — direct non-owner grants; group and collection shares use their own lists):

{{ example: documents/list-shared }}

**c. Documents shared via a group** (`groups.listDocuments`):

{{ example: documents/list-group-documents }}

**d. Documents shared via a collection** (`collections.listDocuments`):

{{ example: documents/list-collection-documents }}

{{#lang ts}}
Each row from `sharedDocuments` extends the base `DocumentInfo` (`title`, `createdBy`, `createdAt`, `lastModified`, plus `tags`/`metadata`/`thumbnailBlobId` when set) with the share-only extras `permission` (never `"owner"`), `grantedBy` (who granted the access) and `source` (always `"permission"` — a direct, non-owner grant).
{{/lang}}
{{#lang swift}}
`sharedDocuments` returns `SharedDocument` values: document fields are under `.document`, with `grantedBy` and `source` alongside them. Group and collection listings have their own result types.
{{/lang}}

{{#lang ts}}
`ownedDocuments` and `sharedDocuments` return the unified `{ items, nextCursor, hasMore }` envelope (raw-JSON `nextCursor`, not base64url — pass it back as the `cursor` option for the next page). `ownedDocuments()` returns a flat `DocumentInfo[]` by default, or the envelope with `returnPage: true`.

For an "everything I can access" surface, combine these two calls with group and collection memberships:

```typescript
const owned  = await jsBaoClient.me.ownedDocuments();
const shared = (await jsBaoClient.me.sharedDocuments()).items;
const collections = await jsBaoClient.collections.list();
// then iterate collections / groups.listUserMemberships and call
// collections.listDocuments / groups.listDocuments.
```

`jsBaoClient.documents.hasLocalCopy(documentId)` is the synchronous local-cache check, useful when deciding whether to render skeletons before `open()` resolves.



#### `syncMetadata()` reference

`client.syncMetadata(options?: SyncMetadataOptions)` refreshes the local metadata index from the server. Its options are conditional on `scope` in ways the type signature doesn't show:

- `scope: "all"` (default) syncs every document the user can reach, paginating the listing to completion; `scope: "single"` + `documentId` syncs one row.
- `authoritative` (default `true`) treats the response as a complete snapshot of the scope: local rows the response doesn't mention are dropped from the index and, if they had cached metadata, `documentMetadataChanged` fires with `action: "deleted"`. Pass `authoritative: false` to merge only, with no eviction. Both scopes default to authoritative — `scope: "single"` is **not** forced non-authoritative, and it's exactly this scope that produces the `action: "deleted"` event for a single document a peer revoked or hard-deleted.
- `payloadType` doesn't disable eviction, it **defers** it: an `"ids"` sync re-fetches the document and only evicts, with a synthetic authoritative upsert, once a 404/403 confirms it's actually gone — it never deletes on the strength of an id list alone. It defaults to `"full"` under `scope: "all"`; under `scope: "single"` it defaults to `"full"` when the fetched or supplied document exists and `"ids"` when it doesn't.
- `retainIds` protects documents from eviction during `scope: "all"` only. Use the `shouldRetain(documentId)` predicate when a single-document sync must also protect a document, such as one still opening.
- `background: true` swallows sync errors instead of throwing — use it for a periodic or on-navigation refresh you don't want to fail the caller over.

```typescript
await client.syncMetadata({ scope: "all", background: true });
```

`documentMetadataChanged` fires whenever a document's locally cached metadata changes — a title/tag/thumbnail edit, a permission-cache update, an eviction, or a delete — with:

- `action`: `"created"` | `"updated"` | `"evicted"` | `"deleted"`.
- `changedFields`: which fields changed (`"title"`, `"tags"`, `"lastKnownPermission"`, `"thumbnailBlobId"`, `"docMetadata"`, …).
- `source`: `"local"` (this device wrote it), `"server"` (a WS push), or `"idb"` (replayed from IndexedDB).
- `metadata`: the new cache entry, or `null` for `"evicted"` / `"deleted"`.

`"deleted"` is the "the document is gone" signal — a peer revoking your access and a peer hard-deleting the document collapse to the same shape (`metadata: null`), so branch on `action`, not on whether `metadata` is present. `"evicted"` is a separate, local-only concern (e.g. an explicit `client.evictLocalDocument()` call freeing storage) — the document may still be reachable, but there's no cached copy to show until the next sync.

`permission` fires whenever the client learns the signed-in user's access level for a document, or that level changes — independent of the other metadata fields, so subscribe to it separately rather than inferring permission changes from `documentMetadataChanged`:

```typescript
client.on("permission", ({ documentId, permission }) => {
  switcherCache.setPermission(documentId, permission);
});
```

**No event covers a collection-membership change.** Adding a document to (or removing it from) a collection the user belongs to fires neither `documentMetadataChanged` nor `permission` for that grant — `collections.listDocuments()` is a plain query with no push counterpart. Refresh it on the same timer or navigation trigger you use for `syncMetadata()`.

#### Cache maintenance: keeping a combined listing fresh

Re-querying all four paths on every navigation is simpler and equally correct for a view that isn't long-lived (opened once per session, say) — reach for a cache only once a view stays mounted across many navigations, such as a document switcher.

To keep a cached combination current instead of re-walking every listing: key the cache by `documentId`, and store only what you render (`title`, `tags`, `permission`, `source`, which of the four paths surfaced the row). Then, on the events above:

```typescript
client.on("documentMetadataChanged", ({ documentId, action, metadata }) => {
  if (action === "deleted" || action === "evicted") {
    switcherCache.delete(documentId);
  } else {
    switcherCache.patch(documentId, metadata);
  }
});

client.on("permission", ({ documentId, permission }) => {
  switcherCache.setPermission(documentId, permission);
});
```

Drop the row on either `"deleted"` or `"evicted"` — in both cases there's no cached copy to show until the next sync. Call `syncMetadata({ scope: "all", background: true })` on a timer or on becoming visible to pick up changes that happened while the events weren't flowing (app closed, tab backgrounded), and re-fetch `collections.listDocuments()` on that same trigger, since no event covers a collection-membership change. That's the full maintenance loop, with no re-walk of the other three listings needed.

#### An app-owned document index (optional)

Having four access paths means "everything I can access" is a query result, not necessarily what the app should show — a user's server-side access and the set an app presents are different concepts, and for some apps they diverge. Two patterns follow from that, both building on the root document's role as "a natural home for user preferences and settings" (see Critical Rule 7 above): it's also a natural home for an app-controlled index of documents, since it's per-user, always available once signed in, and not itself shareable.

- **A curated "documents to show" index.** Store document references (at minimum the `documentId`; optionally a cached `title`/`permission` for a switcher) in a field on the root document, and drive navigation from that list instead of the raw union of the four paths. This gives the app a definite, app-controlled set — e.g. only documents the user has explicitly added, in an order the user picked — decoupled from whatever the server currently reports the user can reach.
- **An app-side acceptance flow.** The platform has no accept/reject gate for sharing — granting a permission (or resolving an invitation) gives access immediately, with no pending state the recipient controls. An app that wants an explicit "accept before it appears" step layers this on top: track the accepted set (e.g. an array of `documentId`s) in the root document, show a document from `sharedDocuments`/group/collection listings as "pending" until its id is in that set, and add to the set when the user accepts.

Neither is universal. Many apps should simply render what `sharedDocuments`, `groups.listDocuments`, and `collections.listDocuments` return with no app-side index at all — that's the right default when shared documents should just appear, or when access is meant to flow entirely through group or collection membership.
{{/lang}}

## Core data operations

Every example below is compiled against the real client as part of the docs build. The [Querying Data](#querying-data) and [Saving Data](#saving-data) sections below go deeper on projections, includes, and save options.

{{#lang swift}}
The generated model statics route through the process-wide default client — call `JsBaoClient.configureDefault(client)` once at startup, before the first read or write. `query`, `queryOne`, `count`, `aggregate`, `find`, and `findAll` are synchronous `throws` against the local document store. All reads span every open document by default (scope with `QueryOptions(documents: [docId])`); writes target one document — `save(in:)` inserts or updates in place and throws (it validates the write and requires the document to be open), `delete(in:)` throws only if the document isn't open.
{{/lang}}

### Create

{{ example: documents/model-create }}

### Read (find / query / first / count)

{{ example: documents/model-read }}

### Update

{{ example: documents/model-update }}

### Delete

{{ example: documents/model-delete }}

### Upsert by natural key

{{ example: documents/model-upsert }}

### Upsert by named unique constraint

{{ example: documents/upsert-by-unique }}

### Logical query operators

{{ example: documents/query-logical }}

### Absent fields

`$ne`, `$nin` and a `null` equality **match records where the field is absent** — a record that never wrote the field is not equal to any value, as in MongoDB — so a `$ne: true` filter on `deleted` is the "false or not set" filter a soft-delete list needs, with no backfill onto existing records. Equality with a value, the comparison operators (`$gt` / `$gte` / `$lt` / `$lte`) and `$in` **match only records that carry the field**. A schema `default` is applied on read and does not materialise the field in storage, so it does not change what a filter matches.

Test presence with `$exists`. Its treatment of an *explicitly stored* null is path-dependent: the server counts a **stored JSON null** as present (`$exists: true` matches it), while the browser and Swift replicas keep each field in a typed column where a stored null is indistinguishable from an absent field (`$exists: false` matches it). Absent fields behave the same on every path; only explicit nulls differ. `$in` with a `null` entry matches nothing for that entry — use a `null` equality or `$exists: false` instead.

**Negative operators include absent fields.** A filter that uses `$ne` or `$nin` to *exclude* records by a possibly-absent field matches the records lacking it — including access filters injected by `beforeQuery` hooks and write conditions. To exclude the missing case, add a `null` entry to `$nin` — it matches only records carrying a non-null value other than the excluded one, identically on every path:

{{#lang ts}}
```typescript
{ deleted: { $nin: [null, true] } }
```
{{/lang}}
{{#lang swift}}
```swift
["deleted": ["$nin": [nil, true]]]
```
{{/lang}}

`$exists: true` alongside the negative operator is **not** equivalent: on the server an explicitly stored JSON null counts as present, so a `$ne: true, $exists: true` filter still matches a record whose `deleted` is null — a record the old filter excluded.

### Sort + cursor pagination

{{ example: documents/query-paginate }}

#### Sorting on a field some records don't have

Records are schemaless, so the records of one model need not all carry the field you sort on, and a record may carry it as `null`. **Absent and `null` are one value for ordering** — a record that never wrote the field sorts exactly where a record holding `null` sorts:

| Sort | Where those records land |
|---|---|
| `{ priority: 1 }` (ascending) | **first**, before every real value |
| `{ priority: -1 }` (descending) | **last**, after every real value |

Pagination uses the same ordering on clients and the server. Ties use `id`; a page with more results includes a cursor. Pass the cursor back unchanged.

Cursors are **opaque** base64 tokens — never parse or construct one. A cursor issued before this ordering was specified keeps working unchanged, so a client holding one need not restart its walk.

### Aggregation

{{ example: documents/aggregate }}

Grouping by a `stringset` field counts per member value (facet); a membership `groupBy` entry groups by whether the set contains one specific value. Only one stringset facet field is allowed per aggregation, and a facet can't be mixed with other `groupBy` entries — unsupported mixes degrade (the facet is dropped or the result is empty) rather than throw.

### Subscribe to changes

{{ example: documents/subscribe }}

### Resolve-or-create a singleton document

{{ example: documents/get-or-create-doc }}

### Local-only documents

`documents.create({ localOnly: true })` creates a document that never reaches the server — not in the creating session, not in any later one, and not after the local copy is evicted and its metadata reloaded from the stored row. Every edit to it is dropped before it would be queued outbound, so there is nothing to mark unsynced and nothing to commit later.

{{ example: documents/local-only-document }}

{{#lang ts}}
Open a local-only document with `waitForLoad: "local"` and `enableNetworkSync: false`. `enableNetworkSync` defaults to `true`, so omitting it — or passing `waitForLoad: "network"` — throws `JsBaoError` with code `LOCAL_ONLY_UNSUPPORTED_OPTION`; pass both options explicitly on every open.
{{/lang}}
{{#lang swift}}
Open a local-only document with `waitForLoad: .local` and `enableNetworkSync: false` — the content never reaches the wire regardless of the open options, but these are the ones that skip an unreachable network wait.
{{/lang}}

### Share a document (user / email / group)

{{ example: documents/share-document }}

### Link access ("anyone with the link")

{{ example: documents/link-access }}

`setLinkAccess(documentId, level)` sets a permission **floor**: any signed-in app user who has the document ID resolves to at least `level` (`"reader"` | `"read-write"`) with no explicit grant. Effective access is `max(direct grant, group grants, link floor)` — the floor only ever raises a caller's level, never lowers an explicit grant. It counts for reading/writing document data but NOT for sharing or management: a link-only caller can't reshare, change permissions, or delete. Requires document owner, app owner, or read-write editor; root documents can't be link-shared (throws).

- A document reached only through the floor reports `accessSource: "link"` on `documents.get(documentId)` and does NOT appear in `me.sharedDocuments()` — track it client-side if you want a "recently opened" affordance.
- `getLinkAccess(documentId)` returns `{ documentId, linkAccess, linkAccessUpdatedBy?, linkAccessUpdatedAt? }` (`linkAccess` is `null`/`nil` when off) and is authorized for any caller who can read OR manage the document, so an app owner can inspect it without a content grant.
- `clearLinkAccess(documentId)` — or lowering the level — evicts connected link-only viewers immediately; explicit grants are untouched.

### Update thumbnail / metadata

{{ example: documents/update-metadata }}

## Common Document Usage Patterns

{{#lang ts}}
**Document state is application state.** Hold document opening, closing, readiness tracking, and "which document am I in" in a small Pinia store of your own, alongside the template's `userStore`. Each pattern below says what that store comes to; [Pattern 3](#pattern-3-multiple-documents) has a minimal one you can strip down for Patterns 1 and 2.
{{/lang}}

### Pattern 1: Single Document (Personal Apps)

**Best for:** Personal tools, single-user apps, no sharing needed

Each user gets exactly one document that holds all their data. The document is opened on app load / user sign-in. No document management UI is needed.

**Examples:** Personal task manager, habit tracker, journal app, budgeting tool

**User experience:** Users sign in and immediately see their data. No concept of "documents" is exposed in the UI.

**Implementation** — resolve-or-create the per-user document on app init, then open it (see [Resolve-or-create a singleton document](#resolve-or-create-a-singleton-document) above for the compiled call). `result.created === true` if a new document was just created.

### Pattern 2: One Document at a Time (Workspaces)

**Best for:** Apps where users create discrete projects/workspaces they might share independently — accounting (per company), project management (per project), shared shopping lists (per household).

Users have multiple documents but work in one at a time, switching between them. Track the current document and call `open()` on the chosen one.

{{#lang ts}}
Build the workspace switcher and the management page on the client surface: `me.ownedDocuments()` / `me.sharedDocuments()` to list, `documents.update(documentId, { title })` to rename, `documents.delete()` to delete, and [Building a share UI](#building-a-share-ui) for the share row.
{{/lang}}

List with `me.ownedDocuments()` and `open()` the selected document; create a new workspace document with `create()` and open it:

{{ example: documents/create-document }}

### Pattern 3: Multiple Documents

**Best for:** Apps that query across many documents, each with its own sharing context — chat (per channel), multi-tenant dashboards, collaborative workspaces with distinct collections.

All documents that need live updates or cross-document queries must be open. Tag documents so you can fetch a set with a tag-filtered `me.ownedDocuments` (and `me.sharedDocuments` if the user can also be a non-owner), open each, and track per-document readiness yourself. For collective sharing of multiple documents as a unit, prefer the server-side Collections API (`client.collections.*` — see [Collections](#collections) below) over local tracking.

{{ example: documents/open-tagged-docs }}

{{#lang ts}}
A minimal Pinia store for "all documents tagged `channel`":

```typescript
// stores/channelDocsStore.ts
import { defineStore } from "pinia";
import { ref, computed } from "vue";
import { jsBaoClientService } from "primitive-app";

export const useChannelDocs = defineStore("channelDocs", () => {
  const documentIds = ref<string[]>([]);
  const openedIds = ref<Set<string>>(new Set());
  const loadError = ref<Error | null>(null);

  // True only after every channel doc this user can see is open.
  const allReady = computed(
    () =>
      documentIds.value.length > 0 &&
      documentIds.value.every((id) => openedIds.value.has(id))
  );

  async function load() {
    loadError.value = null;
    try {
      const client = await jsBaoClientService.getClientAsync();
      const owned = await client.me.ownedDocuments({ tag: "channel" });
      const shared = (await client.me.sharedDocuments({ tag: "channel" })).items;
      documentIds.value = [...owned, ...shared].map((d) => d.documentId);
      await Promise.all(
        documentIds.value.map(async (id) => {
          await client.documents.open(id);
          openedIds.value.add(id);
        })
      );
    } catch (err) {
      loadError.value = err as Error;
    }
  }

  return { documentIds, openedIds, allReady, loadError, load };
});
```

Then feed `allReady` into `useJsBaoDataLoader` so queries don't run until every channel doc is open:

```typescript
const channelDocs = useChannelDocs();
onMounted(() => channelDocs.load());

const { data: messages } = useJsBaoDataLoader({
  documentReady: () => channelDocs.allReady,
  load: () => Message.query({}, { limit: 100 }),
});
```

The same pattern adapts to "the K most recent" or "channels the user belongs to" — change the `me.ownedDocuments` / `me.sharedDocuments` calls and the readiness condition; everything else stays.

`me.ownedDocuments` includes newly created owned documents from the local cache while the server listing catches up. You can refresh the list after creation or add the result immediately for faster UI feedback.
{{/lang}}

{{#lang swift}}
The most common multi-doc shape is one ambient library/index document plus N per-item documents (each shareable independently); see [Index doc + per-item docs](#index-doc--per-item-docs-keying-tombstones-reconcile) below for keying, tombstones, and the reconcile pass. Open auxiliary docs with `appState.openAuxiliaryDoc(_:)` and close them with `appState.closeAuxiliaryDoc(_:)`.

### The `PrimitiveAppState` document lifecycle

The neutral lifecycle is: resolve-or-create the doc (`client.documents.getOrCreateWithAlias` — see [Resolve-or-create a singleton document](#resolve-or-create-a-singleton-document)), open it, then read through the codegen'd facade scoped to it (see [Read](#read-find--query--first--count)). Models need no per-document binding — the facade statics read from every open document, backed by the process-wide default client (`JsBaoClient.configureDefault`).

`PrimitiveAppState` is the **SwiftUI app-state glue** (PrimitiveApp package) that owns that lifecycle: subclass it for app-specific state and override `connectClient()` to drive doc setup after connect (`configureDefault` is wired for you during `initialize()`). The subclass below is framework glue; the Primitive call in it is the `getOrCreateWithAlias` resolve-or-create:

{{ example: documents/app-state-lifecycle }}

For per-document setup beyond models, override the `onDocumentOpened(doc:documentId:)` hook — the base class opens the doc once and hands you the live `YDocument`, so don't call `openDocument(...)` again just to get one.

To create and immediately edit a document, call `client.createDocument(options:)`, read `metadata?["documentId"]?.stringValue`, then open that document before writing. Creation returns metadata; it does not open the document.

The default open can use the local copy. A network-required open waits up to `availabilityWait` (30 seconds by default), then throws `.networkTimeout` if the document is still unavailable.

**Multi-doc apps (one ambient library doc + N per-item docs).** `selectDocumentAwaiting(_:)` is the *single*-selected-doc lifecycle — it closes the previously selected doc first, so using it for a per-item detail view closes your library/index doc. For one ambient doc plus transient detail docs, use `appState.openAuxiliaryDoc(_:)` from the detail view's `.task` and `appState.closeAuxiliaryDoc(_:)` from `.onDisappear`. These register the doc for sync, but they don't touch `selectedDocId` or fire `onDocumentOpened`. Once open, read the doc's records through the facade scoped to that document — the same scoped query as [Open Documents Before Querying](#1-open-documents-before-querying). The view is framework glue around `openAuxiliaryDoc` + that scoped query:

{{ example: documents/app-state-auxiliary-doc }}

**Watch for access loss in detail views.** When a peer revokes your access or hard-deletes a doc you have open, the server collapses both to the same wire shape: a `.documentMetadataChanged` event with `action == "deleted"` (and `metadata == nil`). Subscribe to it, filter on `action == "deleted"`, and dismiss the view. Retain the returned `EventSubscription` on a property, or the handler is dropped the moment the call returns. `observeOnMainActor`'s handler already runs on the main actor, so there is no `Task { @MainActor in … }` wrapper — and a SwiftUI `View` is a struct, so capture the values you need rather than `[weak self]`:

```swift
@Environment(\.dismiss) private var dismiss
@State private var deletedSub: EventSubscription?

.task {
    let openDocId = self.openDocId
    deletedSub = client.observeOnMainActor(DocumentMetadataChangedEvent.self) { ev in
        guard ev.action == "deleted", ev.documentId == openDocId else { return }
        dismiss()
    }
}
.onDisappear { deletedSub?.cancel() }
```

When the handler needs the event's own timing — a debug timeline, a latency report — use the `withDelivery:` form: `client.observeOnMainActor(SomeEvent.self, withDelivery: { event, delivery in … })`. `delivery.emittedAt` is the instant the event was emitted (a `Date()` taken inside the plain handler also includes the hop onto the main actor), and `delivery.sequence` is a monotonic per-client counter — sort timelines on it, not on timestamps.
{{/lang}}

## Data Modeling Decisions

### Separate Documents When:

- Items need independent sharing (e.g., each todo list shared with different people)
- Items are logically distinct workspaces/projects
- You want to limit sync scope

### Single Document When:

- All data should always be shared together
- Data is tightly coupled
- Simplicity is more important than granular sharing

### Tagging Documents

Use tags to categorize documents by type. Pass `tags` to `create()`, filter the user's owned documents by tag server-side, or add/remove tags on an existing document:

{{ example: documents/tag-documents }}

You can also create a tagged document and filter locally:

{{ example: documents/tag-filter-local }}

## Defining Models

Models are declared in a TOML schema and the client model types are generated from it by codegen. The field types and options are the same across platforms; the schema file, codegen command, and generated artifacts differ.

{{#lang ts}}
### Creating New Model Files

Models are defined in `src/models/models.toml` and TypeScript classes are generated from that file. Follow this workflow:

**Step 1: Add the model to `src/models/models.toml`** using TOML syntax. Use snake_case for option names (`auto_assign`, `max_length`, etc.) — the loader maps them to camelCase at runtime.

```toml
[models.todos.fields.id]
type = "id"
auto_assign = true
indexed = true

[models.todos.fields.title]
type = "string"
indexed = true

[models.todos.fields.completed]
type = "boolean"
default = false
```

**Step 2: Run `npx js-bao-codegen-v2`** to generate `Todo.generated.ts` and regenerate the barrel `src/models/index.ts`. Codegen emits a typed `TodoAttrs` interface, a merged `Todo` interface extending `BaseModel`, and a `Todo` class extending `BaseModelImpl`. The barrel auto-registers every model at app startup.

**Step 3: Import models from the barrel** (`src/models/index.ts`), never directly from `*.generated.ts` files. The barrel ensures every model is registered exactly once.

```typescript
import { Todo } from "@/models";
```

**Step 4: Make additional edits** to the schema in `models.toml` and run `npx js-bao-codegen-v2` again.

**CRITICAL: Never edit `*.generated.ts` files or `src/models/index.ts`.** Both are overwritten on every codegen run.

### Registering a Model at Runtime

The `models` list a client is built with is fixed for that client. When the shape is only known later — a plugin type, a tenant-supplied schema, an import that defines its own columns — register it on the running client:

```typescript
import { defineModelSchema, createModelClass } from "js-bao";

const Tag = createModelClass({
  schema: defineModelSchema({
    name: "tag",
    fields: {
      id: { type: "id", autoAssign: true, indexed: true },
      name: { type: "string", indexed: true },
    },
  }),
});

await client.registerModel(Tag);
const { data } = await Tag.query({});
```

- Documents the client already has open are initialized for the model, so saves and queries against them work as soon as the call resolves.
- Scoped to the client it is called on; each client registers its own class.
- Registering the same class twice is not an error and does no further work.
- On a document-level failure the error names the document, the model stays registered, and a repeat call finishes only the documents left incomplete.
- Refused, naming what to change, on a destroyed client, for a class with no name, for a different class under a name the client already holds, and for a class belonging to another live client.
{{/lang}}

{{#lang swift}}
### `models.toml` + codegen + the model facade

Define models in `Sources/MyApp/Models/models.toml`; `swift-bao-codegen` emits one `PrimitiveModel` struct per `[models.X]` block into `Models/Generated/`. The generated type IS the API: static reads across all open documents (`TodoItem.query(...)`, `find`, `findAll`, `count`, `aggregate`, `subscribe`), instance writes targeting one document (`try record.save(in: documentId)`, `try record.delete(in: documentId)`), backed by the process-wide default client.

```toml
[models.todos]
class_name = "TodoItem"

[models.todos.fields.id]
type = "id"

[models.todos.fields.text]
type = "string"
required = true

[models.todos.fields.completed]
type = "boolean"
required = true

[models.todos.fields.createdAt]
type = "number"
required = true

[models.todos.fields.sortOrder]
type = "number"
required = true
indexed = true
```

| TOML `type` | Swift storage | Notes |
|---|---|---|
| `id` | `String` (non-optional) | One `id` field per model; runtime guarantees the value. |
| `string` | `String` / `String?` | |
| `number` | `Double` / `Double?` | Round-trips as `Double` — cast to `Int` on read, wrap in `Double(...)` on write. |
| `boolean` | `Bool` / `Bool?` | |
| `date` | `String` / `String?` (ISO-8601) | No parsing at the storage boundary; for timestamps you sort/compare on, prefer `number` (epoch seconds). |
| `stringset` | `Set<String>` / `Set<String>?` | |

`required = true` makes the emitted `init?(record:)` reject construction without that field; `indexed = true` indexes the field for query-path filtering.

**Codegen is wired on both build paths by the scaffold.** `swift build` / `swift test` run `JsBaoCodegenPlugin` automatically. The Xcode app path compiles its source list from `.pbxproj`, so the SPM plugin never fires there — the app target instead carries a **pre-build phase** ("Generate models from models.toml") running `scripts/generate-models.sh`, which writes into `Models/Generated/`. Xcode's Run button, a bare `xcodebuild` and CI therefore all compile the current schema. The phase declares `models.toml` as its input file and a stamp as its output, so an unchanged schema skips it. The SPM target carries `exclude: ["Models/Generated"]` so the two producers don't collide.

The one case that still needs a hand is **adding a new model**: it emits a new file, and the Xcode project lists its sources explicitly, so `xcodegen generate` has to run before Xcode can compile it. `./run-ios.sh` does that (codegen, then regenerate); a build started from Xcode fails naming the missing file and pointing at `scripts/regenerate-project.sh`, rather than compiling an app silently missing the type. Changing an existing model needs nothing.

The two build paths also carry separate package pins: `swift package update` moves the app's `Package.resolved`, while xcodebuild resolves from Xcode's own copy inside the `.xcodeproj`. Left apart, codegen runs against one revision of the client and the app compiles against another. `run-ios.sh`, `archive.sh`, and the fastlane lanes all run `scripts/sync-xcode-pins.sh` first, which copies the app's pin over Xcode's; run it by hand after `swift package update` if you build from the Xcode UI.

`Models/Generated/` is **committed** — generated code is code, reviewed and tested like any other source — and **regenerated by every build path**, so a schema change shows up as a working-tree diff you commit with the change. Never edit those files by hand. The codegen sweep only deletes files carrying the `// Generated by swift-bao-codegen` banner, so a hand-written companion (`TodoItem+Extensions.swift`) for `Identifiable` conformance, computed helpers, camelCase aliases, or convenience inits survives every regen.

```swift
// Sources/MyApp/Models/TodoItem+Extensions.swift
extension TodoItem: Identifiable {}

public extension TodoItem {
    init(text: String) {
        self.init(
            id: UUID().uuidString,
            text: text,
            completed: false,
            createdAt: Date().timeIntervalSince1970,
            sortOrder: Date().timeIntervalSince1970
        )
    }
}
```

**Value-type / merge semantics (load-bearing):**

- **No `nil` in document-backed fields** — the document store doesn't model `nil`. Use `""` for absent strings, `0` for absent numbers, sentinel timestamps for "never", and check those values explicitly.
- **Wire field names are forever.** TOML keys are the wire field names; renaming a key after data is on disk orphans every existing record (Swift *and* JS clients reading the same doc). **snake_case is the cross-client convention** when web/Node and Swift read the same doc; **camelCase is fine for Swift-only docs**. The tool preserves whatever you write — add camelCase aliases in the companion if snake_case wire keys read awkwardly.
- **IDs are `String`** — supply `UUID().uuidString` (or a ULID) when not provided.
- Register models with `client.registerModels([TodoItem.self])` at connect time (or rely on the facade's lazy registration on first read) — registered models are mirrored into the client's shared store and listed in the in-app debug inspector automatically.

Model writes apply locally and sync in the background. `save(in:)` inserts or updates; `save(in:upsertOn:)` matches a unique field; `delete(in:)` removes a record. Open the target document first.

Reads are synchronous and span open documents. Narrow them with `QueryOptions(documents: [...])`. The model methods have no actor isolation; call them from the appropriate context in your app.

> **SourceKit footgun, first time only.** Editing the companion before codegen has ever run shows a red `No such module 'PrimitiveApp'` underline. The real cause is that the generated type doesn't exist yet — run `swift build` once (or the `run-ios.sh` codegen step) and it clears. Only the very first scaffold hits this.
{{/lang}}

### Field Types

| Type        | Description                  | Common Options                 |
| ----------- | ---------------------------- | ------------------------------ |
| `id`        | Unique identifier            | `autoAssign: true`             |
| `string`    | Text values                  | `indexed: true`, `default: ""` |
| `number`    | Numeric values               | `indexed: true`, `default: 0`  |
| `boolean`   | True/false                   | `default: false`               |
| `date`      | ISO-8601 strings             | `indexed: true`                |
| `stringset` | Collection of strings (tags); never `unique` | `maxCount: 20` |

### Field Options

```toml
[models.tasks.fields.id]
type = "id"
auto_assign = true
indexed = true

[models.tasks.fields.title]
type = "string"
indexed = true

[models.tasks.fields.priority]
type = "number"
default = 0

[models.tasks.fields.dueDate]
type = "date"

[models.tasks.fields.tags]
type = "stringset"
max_count = 10

[models.tasks.fields.archived]
type = "boolean"
default = false
```

### Reserved Field Names

A model may not declare a field named `type`, nor any field whose name starts with `_`. Both collide with the storage engine's own columns. `type` is the internal `_type` column, which holds the **model name** — so a filter on a declared `type` field matches the model name instead of the value the record stores, while the record still projects the value you saved. Wrong rows, and nothing in the response to say so. `id` is not reserved: it is the record's primary key.

Codegen refuses such a schema rather than generating for it, naming the block and leaving no file behind:

```
[models.tasks.fields.type]: Field 'type' is reserved (maps to internal _type column in queries)
```

A function version's `documentSchema` — the copy of `models/models.toml` a push carries inside the version — is refused at the push with a 400 carrying the same sentence.

The record routes refuse the key itself, whether or not a schema declares it: `POST .../documents/:documentId/records/:model`, `PATCH .../records/:model/:recordId` and `POST .../records/bulk` answer **400 `RESERVED_FIELD_NAME`** with the same sentence and write nothing. A bulk blob is all-or-nothing, so one bad record refuses the whole body. `primitive documents records save <document-id> <model> --data '{"type":"expense"}'` is therefore an error rather than a record no filter can find.

**If your schema already declares `type`:** a schema the server has already stored still parses, and stored records keep their values, so nothing already stored is lost; the refusal appears the next time you run codegen or push — and straight away on any record write that carries the key. To clear it, rename the field (`kind`, `category`, `status`) and regenerate. Renaming the declaration moves no data — records written under the old name still carry it (visible in `primitive documents records query <document-id> <model> --json`), so copy each record's old value to the new field in a one-off pass before dropping it from your code.

### Defining Relationships in models.toml

Declare relationships in `models.toml` using `[models.X.relationships.Y]` sections. Codegen emits typed traversal methods on the generated model types.

```toml
# Author hasMany Posts
[models.authors.relationships.posts]
type = "hasMany"
model = "posts"
related_id_field = "authorId"
order_by_field = "createdAt"
order_direction = "DESC"

# Post refersTo Author
[models.posts.relationships.author]
type = "refersTo"
model = "authors"
related_id_field = "authorId"
```

After running codegen, the generated model types include typed traversal methods:

{{#lang ts}}
```typescript
// Author.generated.ts
export interface Author extends AuthorAttrs, BaseModel {
  posts(options?: PaginationOptions): Promise<PaginatedResult<Post>>;
}

// Post.generated.ts
export interface Post extends PostAttrs, BaseModel {
  author(): Promise<Author | null>;
}
```
{{/lang}}
{{#lang swift}}
```swift
// Author.generated.swift — hasMany resolves a plain array, ordered per the TOML
public func posts() throws -> [Post]

// Post.generated.swift — refersTo resolves the parent record (or nil)
public func author() throws -> Author?
```
{{/lang}}

Use these at runtime:

{{ example: documents/relationships }}

{{#lang ts}}
Relationship traversal uses the same engine as `Model.query(...)` with `include` specs — see [Loading Related Data](#loading-related-data-includes) for the lower-level query-level include syntax.
{{/lang}}

For many-to-many links, declare a `hasManyThrough` relationship over a **join model** — a record that carries one field pointing at each side. `join_model_local_field` points back at the source; `join_model_related_field` points at the target. Optional `join_model_order_by_field` / `join_model_order_direction` order the join leg (default `id` ascending):

```toml
# Post hasManyThrough Tags (via the postTags join model)
[models.posts.relationships.tags]
type = "hasManyThrough"
model = "tags"
join_model = "postTags"
join_model_local_field = "postId"
join_model_related_field = "tagId"
```

{{#lang ts}}
```typescript
// Post.generated.ts — hasManyThrough resolves a PaginatedResult, paging the join leg
export interface Post extends PostAttrs, BaseModel {
  tags(options?: PaginationOptions): Promise<PaginatedResult<Tag>>;
}
```
{{/lang}}
{{#lang swift}}
```swift
// Post.generated.swift — hasManyThrough resolves the linked rows; the paginated
// overload returns a PagedQueryResult, paging the join leg by its declared order
public func tags() throws -> [Tag]
public func tags(
  limit: Int,
  afterCursor: String? = nil,
  beforeCursor: String? = nil,
  direction: CursorDirection = .forward
) throws -> PagedQueryResult<Tag>
```
{{/lang}}

Traversal accepts an optional page size and cursor, paging the join leg the same way a query pages rows:

{{ example: documents/relationships-through }}

### Unique Constraints

Two ways to enforce uniqueness — both declared in `models.toml`:

```toml
# 1. Single-field uniqueness via the field's `unique` option.
#    Use this whenever possible — enables `upsertOn` on save.
[models.users.fields.email]
type = "string"
unique = true
indexed = true

# 2. Multi-field (composite) uniqueness via [[models.X.unique_constraints]].
#    Each entry is a NAMED constraint — the name is what you pass to
#    upsertByUnique / findByUnique at runtime.
[[models.categories.unique_constraints]]
name = "name_parent_unique"
fields = ["name", "parentId"]
```

After codegen, single-field constraints get an auto-generated runtime name of `<modelName>_<fieldName>_unique` (e.g. `users_email_unique`); composite constraints use the `name` you declared (e.g. `name_parent_unique`).

**Wrong** — these TOML shapes are silently rejected or fail at codegen:

```toml novalidate
# DON'T: nesting under `options`. The array lives directly on the model.
[[models.categories.options.unique_constraints]]
name = "name_parent_unique"
fields = ["name", "parentId"]

# DON'T: bare array of fields. Each constraint must be a table with
# both `name` and `fields`.
unique_constraints = [["name", "parentId"]]

# DON'T: `unique` on a stringset, or a stringset in a composite constraint.
[models.posts.fields.tags]
type = "stringset"
unique = true
```

`unique` applies to scalar fields only. No writer can build a consistent key from a set of strings, so a unique stringset — on the field, or named in a composite constraint — is refused wherever a schema is declared: `defineModelSchema`, `loadSchemaFromTomlString`, the generated model barrel and `JsBaoClient`'s `schemaToml` throw `UniqueStringsetError`, and codegen writes nothing:

```
Model "posts": field "tags" is a stringset and cannot be unique. A unique constraint applies to scalar fields only.
```

`primitive config push` refuses the tree in its preflight, before any change, when `models/models.toml` declares one; a function push and the database type routes answer **400 `UNIQUE_ON_STRINGSET`**. Swift's `TomlSchemaLoader` throws `.uniqueOnStringset`, and a `PrimitiveSchema` built in code registers but its first write throws `JsBaoError` `.invalidArgument` with the same sentence.

**If your schema already declares one:** remove `unique = true` from the stringset field, or the stringset field from the constraint, and regenerate — an app whose generated models declare it throws at startup after upgrading js-bao. A constraint a document already recorded is ignored by the server (records sharing a member all save; every other field is unchanged), so nothing stored is lost.

### Working with StringSets

{{ example: documents/stringset-ops }}

### Working with Dates

Dates are stored as ISO-8601 strings. Convert for comparisons:

{{ example: documents/date-handling }}

## Querying Data

{{#lang ts}}
`Model.query()` returns a `PaginatedResult`: `{ data: T[], nextCursor?, prevCursor?, hasMore }`. Always access rows through `.data`.

```typescript
// Query a specific document
const result = await TodoItem.query(
  { completed: false },
  { documents: documentId, sort: { order: 1 } }
);
const items = result.data;             // T[]
const more = result.hasMore;           // boolean

// Query across all open documents (default)
const all = await TodoItem.query({ completed: false });

// Single result helper — returns T | null
const item = await TodoItem.queryOne({ id: someId });
```

**Wrong** — `.query()` does NOT return an array directly:

```typescript
// DON'T:
const items = await TodoItem.query({ completed: false }); // items is { data, nextCursor, ... }
items.map(...);  // TypeError: items.map is not a function
```

### Query Operators

| Operator        | Description                    | Example                                              |
| --------------- | ------------------------------ | ---------------------------------------------------- |
| `$eq`           | Equals (default)               | `{ status: "active" }`                               |
| `$ne`           | Not equals                     | `{ status: { $ne: "deleted" } }`                     |
| `$gt`, `$lt`    | Greater/less than              | `{ priority: { $gt: 5 } }`                           |
| `$gte`, `$lte`  | Greater/less or equal          | `{ dueDate: { $lte: today } }`                       |
| `$in`           | Matches any in array           | `{ status: { $in: ["active", "pending"] } }`         |
| `$nin`          | Not in array                   | `{ status: { $nin: ["deleted", "archived"] } }`      |
| `$startsWith`   | String prefix match            | `{ title: { $startsWith: "Bug:" } }`                 |
| `$endsWith`     | String suffix match            | `{ filename: { $endsWith: ".md" } }`                 |
| `$containsText` | Case-insensitive contains      | `{ title: { $containsText: "urgent" } }`             |
| `$exists`       | Field exists/not null          | `{ dueDate: { $exists: true } }`                     |
| `$contains`     | StringSet contains value       | `{ tags: { $contains: "tutorial" } }`                |
| `$all`          | StringSet contains all values  | `{ tags: { $all: ["work", "urgent"] } }`             |
| `$size`         | StringSet size comparison      | `{ tags: { $size: { $gte: 2 } } }`                   |

Which of these match a record that never wrote the field — and which do not — is in [Absent fields](#absent-fields) above — `$ne` and `$nin` are the ones that changed.

**Logical operators** — see [Logical query operators](#logical-query-operators) above for the compiled `$or` example. Plain field maps AND together:

```typescript
const result = await Task.query({
  completed: false,
  priority: { $gte: 3 },
  category: { $in: ["work", "urgent"] },
});
```

### Pagination

Use cursor-based pagination for large result sets — see [Sort + cursor pagination](#sort--cursor-pagination) above for the compiled `nextCursor` / `uniqueStartKey` example.

### Counting Records

```typescript
const activeCount = await Task.count({ completed: false });
const totalCount = await Task.count({});
```

{{/lang}}

### Loading Related Data (Includes)

Pass `include` in a query to batch-load related records alongside the rows, instead of following each relationship one row at a time. A `refersTo` relationship attaches its single parent; a `hasMany` attaches the matching children.

{{ example: documents/includes }}

{{#lang ts}}
In JavaScript the include is a plain spec object: related records are attached under `._related` on each result row (rows live on `.data`), keyed by the include's `as` (defaults to the model name). The full set of include shapes — including `refersToMany` (StringSet-backed) and nesting — is:

**Include types:**

| Type | Relationship | FK location | Required spec field |
|------|-------------|-------------|--------------------|
| `refersTo` | One related record | FK field on source model | `sourceField` |
| `hasMany` | Multiple related records | FK field on target model pointing back | `foreignKey` |
| `refersToMany` | Multiple related records | StringSet field on source model holding target IDs | `sourceField` |

```typescript
// refersTo: Post has an authorId pointing to a User
const result = await Post.query({}, {
  include: [{
    model: "users",
    type: "refersTo",
    sourceField: "authorId",  // FK field on Post
    as: "author",             // key in _related (defaults to model name)
    projection: { name: 1 },  // optional field subset
  }],
});
// result.data[0]._related.author = { id, name }

// hasMany: Comment has a postId field pointing back to Post
const result = await Post.query({}, {
  include: [{
    model: "comments",
    type: "hasMany",
    foreignKey: "postId",    // FK on Comment pointing to Post
    localField: "id",        // field on Post to match against (defaults to "id")
    as: "comments",
    sort: { createdAt: -1 },
    limit: 10,               // per-parent cap
    filter: { status: "approved" },
  }],
});
// result.data[0]._related.comments = [{ id, text, ... }]

// refersToMany: Post has a tagIds StringSet field containing Tag IDs
const result = await Post.query({}, {
  include: [{
    model: "tags",
    type: "refersToMany",
    sourceField: "tagIds",   // StringSet field on Post
    as: "tags",
  }],
});
// result.data[0]._related.tags = [{ id, name }, ...]
```

Includes can be nested (up to 3 levels deep) by adding an `include` array to an include spec:

```typescript
const result = await Article.query({}, {
  include: [{
    model: "comments",
    type: "hasMany",
    foreignKey: "articleId",
    as: "comments",
    include: [{
      model: "users",
      type: "refersTo",
      sourceField: "authorId",
      as: "author",
      projection: { name: 1 },
    }],
  }],
});
// result.data[0]._related.comments[0]._related.author = { id, name }
```

### Aggregations

Group and calculate statistics — see [Aggregation](#aggregation) above for the compiled call. `groupBy` must name at least one field: an empty `groupBy` throws `Invalid aggregation configuration`, so reach for [`Model.count(filter)`](#counting-records) when you want a plain total. The result is therefore always a **nested object keyed by group values** (not an array), one level per `groupBy` entry, and the leaf under the last group value depends on which operations you asked for.

**A lone operation** — the leaf collapses to that operation's bare value, for `count`, `sum`, `avg`, `min` and `max` alike, so there is no `{ count: n }` or `{ sum_estimatedHours: n }` wrapper to read through. The collapse itself holds for **every** `groupBy` type, a plain indexed field included (not just the stringset facets below) — but a stringset facet computes `count` and nothing else, so only `count` has a value to collapse there:

```typescript
const listId = "01M12A5DJ7DW4Z1BVDVVTZ1CXN";
const counts = (await TodoItem.aggregate({
  groupBy: ["listId"],              // a plain indexed string field
  operations: [{ type: "count" }],
  filter: { completed: false },
})) as Record<string, number>;      // narrow before indexing — see below
// Returns:
// { "01M12A5DJ7DW4Z1BVDVVTZ1CXN": 4, "01M12A84FEBE3MXTWVC4DHZRXW": 3 }

const open = counts[listId] ?? 0;   // ✅ the count itself
// counts[listId]?.count            // ❌ always undefined — renders 0
```

`aggregate` is typed `Record<string, any> | Record<string, any>[]`, so the scaffolded (strict) TypeScript app rejects indexing the result directly: `counts[listId]` on that union is `TS7053: Element implicitly has an 'any' type`. Assert the leaf shape on the call before you read a group out of it — `as Record<string, number>` for any single operation, `as Record<string, { count: number; sum_estimatedHours: number }>` for the operation-keyed leaves below.

**Two or more operations** — the leaf is an object keyed by operation. Operation result keys are `count`, `sum_<field>`, `avg_<field>`, `min_<field>`, `max_<field>`:

```typescript
const stats = await Task.aggregate({
  groupBy: ["category"],
  operations: [
    { type: "count" },
    { type: "avg", field: "priority" },
    { type: "sum", field: "estimatedHours" },
  ],
});
// Returns:
// {
//   work:     { count: 8, avg_priority: 2.5, sum_estimatedHours: 40 },
//   personal: { count: 3, avg_priority: 1.0, sum_estimatedHours: 6 },
// }
```

A lone `sum`, `avg`, `min`, or `max` collapses exactly as a lone `count` does — read it as `result[group]`, not as `result[group].sum_estimatedHours`:

```typescript
const hours = await Task.aggregate({
  groupBy: ["category"],
  operations: [{ type: "sum", field: "estimatedHours" }],
});
// Returns:
// {
//   work:     40,
//   personal: 6,
// }
```

That rule holds on **every surface**, so an aggregation you declare once reads the same wherever you run it: these model statics, the document HTTP aggregate (`POST /app/{appId}/api/documents/{documentId}/records/{model}/aggregate`) and the `primitive documents records aggregate` CLI, database aggregates and registered `aggregate` operations, workflow pipeline steps, and `connectDoDb` model bindings. HTTP and the CLI's `--json` deliver it inside the usual `{ result }` envelope:

```json
{ "result": { "work": 40, "personal": 6 } }
```

Multi-field `groupBy` produces deeper nesting (one level per field) and applies the same leaf rule. Group values become object keys, so a numeric field's values are stringified:

```typescript
const byCategoryAndPriority = await Task.aggregate({
  groupBy: ["category", "priority"],
  operations: [{ type: "count" }],
});
// Returns:
// {
//   work:     { "1": 2, "3": 5 },
//   personal: { "2": 1 },
// }
```

**StringSet facet aggregation** — grouping by a `stringset` field counts per member value, with the same leaf rule:

```typescript
const tagCounts = await Task.aggregate({
  groupBy: ["tags"],                // "tags" is a stringset field
  operations: [{ type: "count" }],
});
// Returns: { "work": 15, "urgent": 8, "personal": 5 }
```

Only one stringset facet field is allowed per aggregation. A facet aggregation computes `count` only — asking it for a `sum`/`avg`/`min`/`max` returns `undefined` for that operation rather than a number, so group by a plain indexed field when you need one. To check membership of a specific value across records, use a `StringSetMembership` groupBy entry: `{ field: "tags", contains: "urgent" }`.

### useJsBaoDataLoader Pattern

The data a component renders is a plain `Model.query` (see [Read](#read-find--query--first--count) for the compiled call) that you re-run whenever the underlying records change. `useJsBaoDataLoader` is the **web template's** Vue composable that wires that re-run for you — it is framework glue around the same query, not a different API.

The scaffolded template ships it at `src/composables/useJsBaoDataLoader.ts` — part of your app's source, yours to read and change:

```typescript
import { useJsBaoDataLoader } from "@/composables/useJsBaoDataLoader";
```

**It is component-only**: it registers its model subscriptions and document-event listeners inside `onMounted`, which fires only for mounted Vue components. Calling it from a Pinia store's `setup()`, a router guard, or any other non-component context will load data once but **never react to subsequent changes** — the `onMounted` callback never runs there, so no subscriptions are registered. For those contexts, subscribe directly (see [Subscribing Outside a Component](#subscribing-outside-a-component) below).

It handles four key concerns:

1. **Waiting for documents to be ready** - The `documentReady` ref/computed tells the loader when all required documents have been opened. Queries won't run until `documentReady` is true, preventing errors from querying before documents are available.

2. **Knowing when UI is ready to render** - The `initialDataLoaded` ref becomes true after the first successful data load, and the `showSkeleton` ref gates loading UI without flashing: it is true immediately while the document is still opening, but on reloads after that it only turns true once the load has run longer than `skeletonDelayMs` (option, default 100 ms) — a fast warm reload never flashes a skeleton.

3. **Subscribing to model changes** - When any model in `subscribeTo` changes (local edits or sync from other clients), the loader automatically re-runs `loadData` to keep the UI current.

4. **Reactive query parameters** - When `queryParams` change (route params, filters, pagination, etc.), the loader re-runs `loadData` with the new parameters. Route params are not automatically included in `queryParams` so you should include them if changes to route params should trigger reloading data.

**Data flow pattern:** Update `queryParams` based on page state, UI filters, or pagination → triggers `loadData` → returns data → reactive UI update. This keeps data loading centralized and predictable.

**Best practices:**

- Centralize all data loading for a component in a single `useJsBaoDataLoader` call
- Push filtering logic into js-bao `.query()` calls rather than fetching everything and filtering in JavaScript
- Always pass `documentReady` - typically a ref that becomes true after your document opening logic completes

Under the hood it wraps the same compiled client calls documented above — `Model.subscribe` ([Subscribe to changes](#subscribe-to-changes)) to react to changes and `Model.query` to load — so this section is just the Vue-composable binding around them:

{{ example: documents/dataloader-glue }}

**Rules:**

- Use `useJsBaoDataLoader` no more than once per component
- **Return a single structured object** from `loadData`
- Never add a watch on `loadData` results. Do processing inside `loadData`.
- Never rely on component remounting for route param changes. The loader only sees changes via `queryParams`.
- Gate skeleton/loading UI on `showSkeleton`, not on `documentReady` or raw load state — it handles the initial-load wait and suppresses the warm-reload flash for you. The gate the template ships is `import PrimitiveLoadingGate from "@/components/shared/PrimitiveLoadingGate.vue"`.
- `initialDataLoaded` becomes true after the first successful `loadData`. Make rendering/redirect decisions ONLY after `initialDataLoaded` is true.
- For side effects after load (like redirects), watch `initialDataLoaded` and act when it becomes true.
- For sequences of mutations (save/delete/reorder), set `pauseUpdates` while mutating, then call `reload()` afterward to avoid flicker.

### Subscribing Outside a Component

`useJsBaoDataLoader` is the right tool inside a component. Outside one — a **Pinia store**, a singleton service, a router guard — do not reach for it: its `onMounted`-based subscriptions never register there, so reactive updates silently never fire.

`Model.subscribe(callback)` is a static method that works **anywhere**, independent of the Vue component lifecycle (see [Subscribe to changes](#subscribe-to-changes) above for the compiled call). It returns an unsubscribe function and fires the callback whenever any record of that model changes (local edits or sync from other clients). The neutral pattern is just `subscribe` + re-`query`; everything below is **web-template glue** (Pinia) showing where to hang that wiring:

{{ example: documents/subscribe-store }}

Call `start()` once when the store first comes into use (e.g. after login / first document open) and `stop()` when tearing down (e.g. on logout) to release the listener.

Rules:

- Make `subscribe` setup **idempotent** — guard with the stored unsubscribe handle so re-entry doesn't stack duplicate listeners that each trigger a reload.
- Always keep and eventually call the unsubscribe function; an orphaned listener leaks and keeps reloading after the store is no longer needed.
- The callback receives no arguments — re-run your query inside it; don't expect a changed-record payload.
- Subscribe only after the relevant documents are open, or your initial `reload()` may query before data is available.

This is the same `Model.subscribe()` that `useJsBaoDataLoader` calls internally — you are just registering it in a place that doesn't depend on `onMounted`.
{{/lang}}

{{#lang swift}}
### View-data binding with `BaoDataLoader`

The data a view renders is a plain facade query — `TodoItem.findAll()` / `TodoItem.query(...)` (see [Read](#read-find--query--first--count)) — re-run whenever the records change, which you observe with `TodoItem.subscribe` (see [Subscribe to changes](#subscribe-to-changes)). `BaoDataLoader<[T]>` is the **SwiftUI glue** (PrimitiveApp package) that wires that re-run into a view: bind it rather than subscribing to client events directly or rolling a `@Published var items` + manual `refresh()`. The loader owns its subscription lifecycle (cancelled on deinit), debounces bursts (~50ms), runs the first load immediately, and re-runs a synchronous `load` closure on every trigger.

The view below is framework glue; the only Primitive calls in it are the `findAll()` query and the `TodoItem.subscribe` trigger:

{{ example: documents/dataloader-glue }}

`.onModel(subscribe: TodoItem.subscribe)` fires on **any** add/update/delete recorded in that model's shared store — local writes and remote writes both — so `reloadNow()` after a write is unneeded; call it only when the `load` closure reads something the loader can't subscribe to (a REST resource). Other triggers (`LoaderTrigger`): `.onSync`, `.onDocumentSyncStateChanged`, `.onDocumentEvents`, `.onConnect`, `.onModelChange(_:)` for a hand-built runtime-schema `DynamicModel`, and `.custom((client, reload) -> EventSubscription?)`.

`loader.phase` is a trinary: `.loading` (first load not complete), `.empty` (first load complete, data conforms to `LoaderEmptiness` and is empty), `.loaded(Data)`. `[T]`, `String`, and `Optional` get `LoaderEmptiness` out of the box.

`loader.showSkeleton` is a separate anti-flash flag for gating skeleton/placeholder UI: it is `true` immediately while the document is opening (`documentReady == false`), then during an in-flight first load only once the load has run longer than `skeletonDelay` (default 100 ms) — so a warm reload that resolves from local CRDT state in a few milliseconds never flashes a skeleton. It returns to `false` once the first load completes. Gate loading UI on `showSkeleton` rather than `phase == .loading` when you want to suppress that flash.

> **Empty-vs-pending when a local index is hydrated by an async server fetch.** If the loader reads a local document mirror that a `reconcile()` fills from the server *after* the view binds, the first load completes against an empty store → `.empty` → the placeholder flashes before the server data arrives. `loader.phase` can't tell "genuinely empty" from "fetch still pending" — gate the empty state on a "first reconcile attempted" flag (set on attempt, success or failure, so an offline brand-new user still reaches the empty state instead of spinning forever):
>
> ```swift
> case .empty:
>     if appState.hasReconciled { EmptyState() }
>     else { ProgressView() }
> ```

**Building the views themselves** — layout, navigation, platform gating, list affordances — is governed by the `ios-design` skill bundled in the Swift starter template (at `.claude/skills/ios-design/` in the scaffolded project). It triggers on SwiftUI edits inside the project, so consult it explicitly before writing views; UI decisions made around scaffolding time can otherwise miss it.
{{/lang}}

## Saving Data

{{#lang swift}}
Writes go through record instances and are local-first — applied to the document store immediately and synced in the background (see [`models.toml` + codegen + the model facade](#modelstoml--codegen--the-model-facade) for the save/delete contract). `try record.save(in: documentId)` uses merge semantics: `nil` optional fields are not written, so fields you didn't set are preserved on update.

**Saving an existing record writes only the fields you assigned** since you read it. Fetch–mutate–save is therefore a field-level write, and two devices editing different fields of the same record merge instead of overwriting each other:

```swift
var task = try TaskRecord.find(id)!   // read: nothing pending
task.title = "New title"              // only `title` is marked changed
task = try task.save(in: documentId)  // only `title` is written
```

Inserting a record the document doesn't have yet writes every field, so copying a record into another document still carries all of it. If the other document ALREADY holds that record, the save is an update — and an unmodified record has nothing to write, so it does nothing. Call `record.markAllChanged()` first when you mean to copy the whole record over it.

`save(in:)` returns the record **as saved** — re-read from the document with no pending changes left, so a field another device changed while you held your copy, and any schema defaults filled in on insert, are present. Assign it back (as above) when you keep using the value, or call `record.discardChanges()` to drop pending edits without writing them. Constructing a record (`TaskRecord(id:title:)`) or decoding one from JSON marks every field it carries as changed; only a record read back through the facade starts clean.
{{/lang}}

{{#lang ts}}
### Save to a Specific Document (when creating new objects)

```typescript
const newItem = new TodoItem();
newItem.title = "Buy groceries";
await newItem.save({ targetDocument: documentId });
```

### Update Existing Item

```typescript
// Items remember their document
todo.completed = true;
await todo.save();
```

**Wrong** — common save footguns:

```typescript
// DON'T: forget to await — the next read may not see the change yet,
// and unhandled rejections (e.g. document closed) get swallowed.
todo.completed = true;
todo.save();              // missing await
router.push("/done");

// DON'T: try to spread/clone a model object — instances are not POJOs.
const copy = { ...todo }; // drops every field; see "Model Instances Are Not Plain Objects"

// DO: read fields directly into a plain object when you need a snapshot.
const snapshot = { id: todo.id, title: todo.title, completed: todo.completed };
```

### Model Instances Are Not Plain Objects

A model instance is **not** a POJO, and you cannot spread or clone it. Each declared field is a getter/setter defined on the class **prototype** (backed internally by copy-on-write change tracking over the document's document data) — *not* an own enumerable property on the instance. JavaScript's spread and rest operators only copy *own enumerable properties*, so they never invoke those getters and the field data is silently dropped.

```typescript
// ❌ All of these lose the model's field data — they do NOT throw, they just produce empty/partial objects:
const copy            = { ...task };              // field data gone
const { id, ...rest } = task;                     // `rest` is empty-ish
const updated         = { ...task, done: true };  // task's other fields are gone
await Message.create({ ...task });                // fields don't come along
```

There is no public `.attrs` bag and no public `.toJSON()` to spread either. To move a model's data into a plain object, read the fields you need explicitly:

```typescript
// ✅ Snapshot for serialization, an integration call, or structuredClone:
const snapshot = { id: task.id, title: task.title, done: task.done, priority: task.priority };

// ✅ Duplicate into a new record — list the fields; don't spread the source instance.
const dup = new Task({ title: task.title, priority: task.priority });
await dup.save({ targetDocument: docId });

// ✅ Mutate in place, then save — no copy needed.
task.done = true;
await task.save();
```

Note the direction: constructors (`new Task({...})`), `save()`, and `query`/`find` filters all accept plain objects, so spreading a *plain object* into them is fine. Spread only fails when the **source** of the spread is a model instance.

### Choosing How to Target Documents for Saves

These mechanisms govern **where saves route** — they have no effect on what queries see. A query always spans every open document (see [Querying Data](#querying-data)); setting a default document or a model-document mapping narrows writes, never reads. To scope a read, pass `{ documents }` on the query itself.

When saving new objects, you need to specify which document they go into. There are three ways to do this:

1. **Pass `targetDocument` explicitly on each save** (preferred for most cases):
   ```typescript
   await item.save({ targetDocument: documentId });
   ```

2. **`jsBaoClient.setDefaultDocumentId(docId)`** — sets a default document for all subsequent saves that don't specify a `targetDocument`. Good when many consecutive saves go to the same document (e.g., during app initialization or a bulk import).

3. **`jsBaoClient.addDocumentModelMapping(modelName, docId)`** — routes all saves of a specific model to a specific document. Good when a model *always* goes to the same document for the lifetime of the app session.

**Anti-pattern: Frequently switching defaults.** If your app writes to different documents based on context (e.g., the user switches between workspaces, or items are routed to different documents based on their properties), do NOT repeatedly call `setDefaultDocumentId()` or update model-document mappings to redirect writes. This is fragile and error-prone — it creates implicit state that's easy to get out of sync. Instead, pass `targetDocument` explicitly on each `.save()` call. Reserve `setDefaultDocumentId` and `addDocumentModelMapping` for cases where the target doesn't change, or changes only rarely.

### Deleting Records

Delete a record after finding it — see [Delete](#delete) above for the compiled call.

### Upsert by Unique Constraint

`upsertByUnique(constraintName, lookupValue(s), data, options?)` finds an existing record by a named constraint and updates it, or creates one if none exists — see [Upsert by named unique constraint](#upsert-by-named-unique-constraint) above for the compiled call. The `data` object MUST include the same constraint field values as `lookupValue` (mismatch throws), and `targetDocument` is REQUIRED whenever a new record may be created. A single-field constraint takes a scalar lookup value instead of an array.

Single-field constraints declared via `unique = true` in TOML get an auto-generated name of `<modelName>_<fieldName>_unique` (where `modelName` is the `[models.<name>]` block key, not the class name). Use `[[models.X.unique_constraints]]` (model level — not under `options`) to control the name explicitly.

For single-field upserts where the value already lives on the instance, `save({ upsertOn })` is simpler than `upsertByUnique` — see [Upsert by natural key](#upsert-by-natural-key) above for the compiled call.

**Wrong** — common mistakes that throw at runtime:

```typescript
// DON'T: pass field names instead of the constraint name
await Category.upsertByUnique(["name", "parentId"], ...); // throws: constraint not found

// DON'T: omit targetDocument when creating
await Category.upsertByUnique("name_parentId", ["Work", "root"], { name: "Work", parentId: "root" });
// throws: targetDocument is required when creating new records

// DON'T: data values that don't match lookupValue
await Category.upsertByUnique("name_parentId", ["Work", "root"], { name: "Home", parentId: "root" }, { targetDocument });
// throws: Mismatch between dataToUpsert.'name' and uniqueLookupValue
```

### Upsert by Natural Key (`upsertOn`)

Use the `upsertOn` option in `save()` to upsert by a natural unique field (e.g., `email`, `slug`) without knowing the existing record's ID. The field must have a single-field `uniqueConstraints` entry on the model. See [Upsert by natural key](#upsert-by-natural-key) above for the compiled call.

Behavior:
- **No existing record**: creates a new record with an auto-generated ID (or the caller-provided ID)
- **Existing record found**: merges the provided fields into the existing record; unprovided fields are preserved; returns the existing record's ID
- **Caller provides an ID that mismatches the found record**: throws an error (conflict)

`upsertOn` validates that the field has a registered unique index. It throws if the field is missing from the data or has a null/empty value.

### Save Merge Semantics

`save()` uses **merge semantics**: only the fields you set on the instance are written; existing fields not included in the change set are preserved. Setting a field to `null` removes it from the stored record.

This applies both to new saves and updates, and is consistent across single saves and batch operations.
{{/lang}}

## Design Patterns

### Singleton Model per Document (Avoiding ID Confusion)

Create a singleton model per document for metadata. Child models reference by model ID, not document ID. Declare both models in the schema:

{{#lang ts}}
```toml
# TodoList — one per document
[models.todo_lists.fields.id]
type = "id"
auto_assign = true
indexed = true

[models.todo_lists.fields.title]
type = "string"

[models.todo_lists.fields.createdAt]
type = "number"

[models.todo_lists.fields.createdBy]
type = "string"

# TodoItem references TodoList by MODEL ID (not document ID)
[models.todo_items.fields.id]
type = "id"
auto_assign = true
indexed = true

[models.todo_items.fields.listId]
type = "string"
indexed = true

[models.todo_items.fields.title]
type = "string"

[models.todo_items.fields.completed]
type = "boolean"
```

Run `npx js-bao-codegen-v2` and import as `import { TodoList, TodoItem } from "@/models";`.
{{/lang}}

**Use this pattern when:**

- Documents represent a meaningful entity (project, list, workspace)
- You need document-level metadata
- Child models need to reference their parent container

### Singleton Documents with Aliases

For documents that should exist exactly once (default document, settings), use `getOrCreateWithAlias`. This single call atomically resolves an existing alias or creates a new document with that alias, eliminating race conditions when multiple clients initialize simultaneously. See [Resolve-or-create a singleton document](#resolve-or-create-a-singleton-document) above for the compiled call.

**Alias scope:** aliases are unique per user — each user can have their own document with a given alias.

**Alias API methods:**

- `documents.openAlias(params)` - Open document by alias (throws if not found)
- `documents.createWithAlias(options)` - Create document with alias atomically (fails if alias already exists). Takes `title`, `tags`, `metadata` and `documentFormat`, each applied at creation exactly as on `documents.create`.
- `documents.getOrCreateWithAlias(options)` - Get existing document by alias, or create a new one if not found. Returns `{ documentId, created: boolean, documentFormat?, tags?, metadata?, ... }`. Use this for idempotent initialization. Takes the same options, applied only when the call creates the document; a stated `documentFormat` that the existing document does not have is refused with `DOCUMENT_FORMAT_MISMATCH` (409) rather than handing back a document of the other kind.
- All three create routes (`documents.create`, `createWithAlias`, `getOrCreateWithAlias`) **refuse** a top-level body key they do not read, with 400 `VALIDATION_FAILED` and `details` naming it — including `name`, `parentId` and a flat `scope`/`aliasKey` pair. The typed client methods send only keys the routes read.
- `documents.aliases.resolve(params)` - Get alias info (returns null if not found)
- `documents.aliases.set(params)` - Set an alias for an existing document
- `documents.aliases.delete(params)` - Remove an alias
- `documents.aliases.listForDocument(documentId)` - List all aliases for a document

{{#lang swift}}
### Index doc + per-item docs (keying, tombstones, reconcile)

The most common multi-doc shape is one library document per user (alias-resolved on launch) holding an index of refs to per-item documents (each shareable independently). Three patterns make it converge correctly:

**Key refs by the entity id, never a random `UUID()`.** A ref's row id in the document store is the map key, and `save(in:)` is an upsert on that key. Set `id` to the document/collection id the ref points at, so two writes for the same entity (your create path and a reconcile pass, possibly on another device) converge on one row instead of producing a permanent visible duplicate:

```swift
public extension LibraryItemRef {
    init(itemDocumentId: String, title: String, sortOrder: Double) {
        self.init(
            id: itemDocumentId,        // row id == entity id ⇒ create is an idempotent upsert
            itemDocumentId: itemDocumentId,
            cachedTitle: title,
            sortOrder: sortOrder
        )
    }
}
```

Reserve random `UUID()` ids for records *born* in the document store with no external identity (todos, notes, messages).

**Tombstone instead of hard-delete.** Soft-delete with a `deletedAt` field and filter at query time (`LibraryItemRef.query(["deletedAt": ""])`). Setting `deletedAt` rather than calling `.delete(in:)` avoids the race where a reconcile pass recreates a just-deleted ref because the server still reports the entity as accessible.

**Refresh document navigation on launch, foreground, or pull-to-refresh.** Read each access path, combine permissions using the highest level, and update references keyed by document ID. Remove references absent from the refreshed set, while protecting pending local creates as described below.

> **Guard freshly-created docs from the prune.** `createDocument(...)` is local-first — it returns a `CreateDocumentResult` (metadata only; read the id from `metadata?["documentId"]?.stringValue`) immediately but commits to the server in the background, so a doc you just created is **not yet** in `me.ownedDocuments`. A reconcile firing in that window deletes the brand-new ref and the item vanishes until a later reconcile re-adds it. Track locally-created ids in a `pendingCreateIds: Set<String>` (add on create, remove once the id appears in the server set) and exclude them from the prune: `where !serverIds.contains(ref.id) && !pendingCreateIds.contains(ref.id)`.

For the reconcile reads, the "shared with me" set is the union of `me.ownedDocuments(tag:limit:)` and `me.sharedDocuments(tag:limit:)` — merge them yourself (the same entity can surface in both), then dedupe by document id.
{{/lang}}

## Sharing Documents

Documents can be shared with individual users (by userId or email), with groups, with collections, or exposed to a request-access flow for users with a link. Sharing **by email** to someone who isn't a user yet creates a *deferred grant* that resolves automatically at signup — that resolution lifecycle, app membership, invitation quotas, and the invite-accept wiring live in the [Invitations guide](AGENT_GUIDE_TO_PRIMITIVE_INVITATIONS.md).

### Sharing mental model

**Document permission levels:** `"owner"` > `"read-write"` > `"reader"`. (`"admin"` is an app-role projection, not a grantable document permission.) Effective permission = MAX(direct, group-derived, collection-derived). Granting a *lower* group permission to someone who already has a higher direct grant is a no-op for them.

Access can come from a direct grant, a group, or a collection. An access request asks the owner for a grant; it does not grant access itself.

**Decision rules:**

1. If you have a `userId` → write the direct permission record. If you only have an email → write the same record by email; the platform resolves it or defers it. No manual "accept" call is needed for the standard email-match path.
2. **Share at the highest level that fits — `collection` > `group` > `doc`.** A collection grant propagates to every document inside it (current and future); a group add propagates to every doc that group can see. Don't add per-document grants for something already shared at the collection or group level.
3. Use groups for team-scoped access; use direct permissions for individual access.
4. If a user hits a 403 → read the `canRequestAccess` hint off the thrown typed error and either show "Request access" or a hard-deny screen.
5. Read "documents I own" from `client.me.ownedDocuments()` and "documents shared directly with me" from `client.me.sharedDocuments()`. Group- and collection-scoped access is read through `groups.listDocuments` / `collections.listDocuments`.

### Quick Reference

The core share calls — by userId, by email, and with a group — are in [Share a document (user / email / group)](#share-a-document-user--email--group) above. The full surface adds batch grants, deferred email shares, and access requests:

{{#lang ts}}
```typescript
// Batch — multiple users at once
await client.documents.updatePermissions(documentId, {
  permissions: [
    { userId: "user-abc", permission: "read-write" },
    { userId: "user-xyz", permission: "reader" },
  ],
});

// By email with notification — resolves if the user exists, otherwise creates a
// deferred grant that auto-applies when the recipient signs up. With
// `sendEmail: true`, `documentUrl` is required AND the app must have `baseUrl`
// configured (used to compose the accept URL for the deferred-share email).
// Both preconditions return HTTP 400 if missing.
await client.documents.updatePermissions(documentId, {
  email: "alice@example.com",
  permission: "read-write",
  sendEmail: true,
  documentUrl: `${window.location.origin}/lists`,
});
// Returns either a DirectPermissionGrant (existing user) or a
// DeferredPermissionGrant ({ invitationId, inviteToken, ... }) that the
// recipient redeems via client.invitations.accept(inviteToken) after signup.

// Respond to a 403 with canRequestAccess hint. `permission` is REQUIRED.
try {
  await client.documents.open(documentId);
} catch (err) {
  if (err instanceof JsBaoApiError && err.body?.details?.canRequestAccess) {
    await client.documents.requestAccess(documentId, {
      permission: "read-write",
      message: "Please grant me access",
    });
  }
}
```

**Wrong** — these names look reasonable but do not exist on the API:

```typescript
// DON'T:
await client.documents.setPermissions(...);      // use updatePermissions
await client.documents.setGroupPermission(...);  // use grantGroupPermission
await client.documents.requestAccess(id, { message: "..." }); // missing required `permission`
```

Render the user's documents from two calls: `client.me.ownedDocuments()` for documents they own, and `client.me.sharedDocuments()` for documents shared directly with them (direct non-owner grants). Group- and collection-shared documents are listed through `groups.listDocuments` / `collections.listDocuments`.
{{/lang}}

{{#lang ts}}
### Building a share UI

Build your share dialog on `client.documents.*`. Three reads fill it and three writes drive it:

| Row / control | Call |
| --- | --- |
| People with access | `documents.getPermissions(documentId)` → `DocumentPermissionEntry[]` (`userId`, `email`, optional `name`, `permission`) |
| Invited, not signed up yet | `documents.listPendingInvitations(documentId)` → `PendingInvitationEntry[]` (`email`, `permission`, `invitationId`, `expiresAt`) |
| Groups with access | `documents.listGroupPermissions(documentId)` |
| What one user may do | `documents.validateAccess(documentId, { userId })` → `{ hasAccess, permission?, accessSource?, appRole? }` — with no options it answers the current user; decide content writes on `permission`, never `appRole` |
| Invite / change a level | `documents.updatePermissions(documentId, { userId \| email, permission })` |
| Remove a person, cancel an invite | `documents.removePermission(documentId, { userId })` / `{ email }` |
| Share with a group | `documents.grantGroupPermission(documentId, { groupType, groupId, permission })` |

```vue
<script setup lang="ts">
import { ref, watch } from "vue";
import { jsBaoClientService } from "primitive-app";
import type { DocumentPermissionEntry, PendingInvitationEntry } from "js-bao-wss-client";

// `documentUrl` is the caller's, not this component's to invent: an existing
// member's share email links to it verbatim, so it has to be an absolute URL
// your app really routes — pass the page you show this document on.
const props = defineProps<{ documentId: string; documentUrl: string }>();

const members = ref<DocumentPermissionEntry[]>([]);
const pending = ref<PendingInvitationEntry[]>([]);
const inviteEmail = ref("");

async function refresh(): Promise<void> {
  const client = await jsBaoClientService.getClientAsync();
  members.value = await client.documents.getPermissions(props.documentId);
  pending.value = await client.documents.listPendingInvitations(props.documentId);
}
watch(() => props.documentId, refresh, { immediate: true });

async function invite(): Promise<void> {
  const client = await jsBaoClientService.getClientAsync();
  await client.documents.updatePermissions(props.documentId, {
    email: inviteEmail.value,
    permission: "read-write",
    sendEmail: true,
    // REQUIRED with sendEmail: true — where the recipient lands.
    documentUrl: props.documentUrl,
  });
  inviteEmail.value = "";
  await refresh(); // an unknown email lands in `pending`, a known one in `members`
}

// One call covers both "remove a member" (userId) and "cancel an invite" (email).
async function revoke(target: { userId: string } | { email: string }): Promise<void> {
  const client = await jsBaoClientService.getClientAsync();
  await client.documents.removePermission(props.documentId, target);
  await refresh();
}
</script>

<template>
  <ul>
    <li v-for="member in members" :key="member.userId">
      {{ member.name ?? member.email }} — {{ member.permission }}
      <button v-if="member.permission !== 'owner'" @click="revoke({ userId: member.userId })">
        Remove
      </button>
    </li>
    <li v-for="invite in pending" :key="invite.invitationId">
      {{ invite.email }} — invited as {{ invite.permission }}
      <button @click="revoke({ email: invite.email })">Cancel</button>
    </li>
  </ul>
  <form @submit.prevent="invite"><input v-model="inviteEmail" type="email" /></form>
</template>
```

**Critical:** whenever you pass `sendEmail: true`, `documentUrl` is REQUIRED — without it the API returns HTTP 400 — and it should point at a page where the recipient can see and accept the invitation. The app must also have `baseUrl` configured, or the same call returns 400.

`documentUrl` has to be a URL your app actually routes to. A recipient who is already a member gets the `document-share` email, whose link is the `documentUrl` you sent, verbatim — a path you invented lands them on your Not Found page. The scaffolded template routes `/`, `/login`, `/logout`, the OAuth callback and `/invite/accept`, and everything else falls to the catch-all, so add the document's own route (`src/router/routes.ts`) and build the URL from it, or send a page you already have. (A recipient who is not a member yet is a different email: the deferred `document-share-deferred` template carries a tokenized accept URL composed from the app's `baseUrl`, which the template does route.)

Both lists are point-in-time reads, not live queries: re-run `refresh()` after every write, and after the user accepts an invitation elsewhere. The owner row cannot be removed — hand the document over with `documents.transferOwnership(documentId, newOwnerId)` instead. The full surface, including batch grants and access requests, is in [Quick Reference](#quick-reference) above.
{{/lang}}

### Handling Invitations

**Auto-accept vs Manual:**
- `autoAcceptInvites: true` - Invitations are automatically accepted when the document tag matches a registered collection. Documents appear immediately in the user's list.
- `autoAcceptInvites: false` - Users must manually accept invitations on a management page. Provides more control but requires UI for viewing/accepting invitations.

When a user accepts an invitation, you typically want to navigate them to the content. Since routes use model IDs (not document IDs), query for the model in the newly accessible document first, then route to it.

{{#lang ts}}
```typescript
async function handleInvitationAccepted(documentId: string): Promise<void> {
  // Brief delay for document to sync after acceptance
  await new Promise((resolve) => setTimeout(resolve, 500));

  // Query for the model in the newly accessible document
  const result = await TodoList.query({}, { documents: documentId });
  const list = result.data[0];
  if (list) {
    router.push({ name: "todo-list", params: { listId: list.id } });
  }
}
```
{{/lang}}

### Closing Documents

When your app opens many documents over a session (e.g., viewing individual items that each live in their own document), close documents you're no longer using to avoid accumulating sync connections:

{{ example: documents/close-document }}

When `evictLocal: true` is passed, the client performs a state vector check against the server before removing local data. If the server hasn't received all local writes (e.g. due to a brief network interruption), eviction is skipped and `evicted: false` is returned. This prevents data loss during WebSocket instability.

### Sync Verification

Use these methods to confirm the server has received your writes before taking irreversible actions (e.g., logging out, clearing local storage):

{{ example: documents/sync-verification }}

The point-in-time checks return `false` if the client is disconnected or the check times out. The `waitFor*` polling helpers exist on both `documents.*` and the client root: `waitForWriteConfirmation` returns true on success / false on timeout, and `waitForInSync` throws on timeout.

{{#lang ts}}
The point-in-time checks accept an optional `timeoutMs`.
{{/lang}}
{{#lang swift}}
The point-in-time checks accept a `timeout` — a `TimeInterval` in seconds, default `5`. For a cheap synchronous local read — no round-trip — use `documents.isSynced(documentId:)`.
{{/lang}}

**A document that cannot sync says so.** The `documentSyncStateChanged` event reports `state: "error"` for an open document whose sync handshake goes unanswered for the whole handshake budget (10 s by default) — what the app is rendering has stopped converging — and repeats it on each timeout while the document stays behind. The client keeps retrying underneath, at a backoff that caps at 15 s, and after three consecutive timeouts on a connection that still reads as open it rebuilds the connection itself (once per stalled document, once per connection). Once the document does sync, the event fires `"synced"` once more so the app can clear what it surfaced:

{{ example: documents/sync-state-events }}

The same event reports `"synced"` as each remote update is applied to an open document, so a loader that reloads on every remote write subscribes here too. It says nothing about a document's overall caught-up/behind state — that is what the point-in-time checks above answer.

{{#lang ts}}
The handshake budget is `sync.handshakeTimeoutMs` on the client options.
{{/lang}}
{{#lang swift}}
The handshake budget is `SyncConfig.handshakeTimeout`, in seconds.
{{/lang}}

### Connectivity vs network mode

Network **mode** is user intent — `auto` by default, or pinned by `goOffline()` / `goOnline()`. **Reachability** is whether the device currently has a network path. They are separate, and the client never turns one into the other: losing connectivity in `auto` pauses the socket and suppresses reconnect, but the reported mode stays `"auto"`. When the network comes back the client reconnects on its own, still in `auto`.

So test connectivity with `isOnline`, not with the mode:

{{#lang ts}}
```typescript
const { mode, isOnline, reason } = client.getNetworkStatus();
// mode:     "auto" | "online" | "offline"   — what the app asked for
// isOnline: boolean                          — whether it can reach the server now
// reason:   "user_set" | "connectivityLost" | "connectivityRestored"

client.on("networkMode", ({ mode, isOnline, reason }) => {
  // Fires on a mode change AND on a connectivity change. On a connectivity
  // change `mode` is unchanged and only `isOnline` moves.
  showOfflineBanner(!isOnline);
});
```

Reading `mode === "offline"` to drive offline UI is wrong: it is true only when the app pinned offline itself.
{{/lang}}
{{#lang swift}}
`client.networkStatus` carries the same `mode` / `isOnline` / `reason` triple, `NetworkModeEvent` fires on both kinds of transition, and `client.isOnline()` is the check to branch on. Drive offline UI from `isOnline`, not from `networkMode == .offline`.
{{/lang}}

While the device is unreachable the client behaves as it does when offline is pinned: reads come from the local cache and queued blob uploads wait. Nothing is lost — it resumes on reconnect.

{{#lang ts}}
HTTP calls made while offline fail fast, without attempting the request, with a `JsBaoError` whose code is `OFFLINE`. A request that is attempted and fails in transit (DNS failure, connection refused) throws `JsBaoNetworkError` instead. Treat both as retryable.
{{/lang}}
{{#lang swift}}
HTTP calls made while offline fail fast, without attempting the request, with a `JsBaoError` whose code is `.offline`. A request that is attempted and fails in transit (DNS failure, connection refused) throws `JsBaoNetworkError` instead; its `urlErrorCode` carries the underlying `URLError` code. Treat both as retryable.
{{/lang}}

### Updating Document Metadata

Update a document's title, thumbnail, and presentation metadata — see [Update thumbnail / metadata](#update-thumbnail--metadata) above for the compiled call. Each field is optional; omit one to leave it unchanged.

{{#lang ts}}
Pass `null` to clear `thumbnailBlobId` or `metadata`.
{{/lang}}
{{#lang swift}}
`thumbnailBlobId` and `metadata` are clearable: pass `thumbnailBlobId: .clear` (an `Updatable<String>`) and `metadata: .null` (a `JSONValue`) to null them out; passing `nil` or omitting the argument leaves the field unchanged.
{{/lang}}

`documents.isOpen(documentId)` reports whether a document is currently open.

`thumbnailBlobId` points at a blob you've already uploaded; the platform marks the referenced blob readable to anyone with access to the document. `metadata` is a JSON-serializable object with a 4KB cap on its serialized UTF-8 form — keep it to the kind of presentation hints that need to travel with the document (cover image references, badge colors, list layout). Failures return:

| Error code | Meaning |
|---|---|
| `METADATA_TOO_LARGE` | Serialized `metadata` exceeds 4KB |
| `BLOB_NOT_FOUND` | `thumbnailBlobId` references a blob the platform can't resolve |
| `BLOB_DOC_MISMATCH` | The blob exists but belongs to a different document |

### Deleting Documents

Delete a document (it must be closed first, or pass `forceCloseIfOpen: true`) — see the compiled call below. Root documents cannot be deleted. Deletion requires **direct `owner` permission** on the document or the app `owner` role — group-derived permission never qualifies, and `read-write` editors can delete records and content but not the document itself.

The one exception: if the caller is neither the owner nor an app owner, the platform falls back to checking every collection the document belongs to and allows the delete if **any** one collection's `document.delete` CEL rule passes. That rule defaults to `"false"` (deny) — see [Collection Rule Sets](#collection-rule-sets) — so deletion never widens past owner/app-owner unless an app explicitly configures it. Because that rule is evaluated per collection, adding a document to a collection can extend who is able to delete it; configure `document.add` and `document.delete` together with that reach in mind.

{{ example: documents/delete-document }}

### Document Access Requests

The owner UI when a non-member hits a 403 they're allowed to escalate. The end-to-end flow (a user with a link requests access; an owner lists and approves) is compiled here:

{{ example: sharing/request-access }}

#### Detect on `documents.get` failure

The 403 body is JSON shaped `{ error, status, timestamp, code: "DOC_ACCESS_DENIED", details: { code: "DOC_ACCESS_DENIED", canRequestAccess: boolean } }`. The client throws a typed error carrying that body already parsed — branch on `canRequestAccess` directly, no message-parsing needed:

{{#lang ts}}
```typescript
import { JsBaoApiError } from "js-bao-wss-client";

try {
  const doc = await client.documents.get(documentId);
} catch (err) {
  const details = err instanceof JsBaoApiError ? (err.body as any)?.details : undefined;
  if (details?.canRequestAccess) {
    await client.documents.requestAccess(documentId, {
      permission: "read-write",                  // REQUIRED
      message: "Working on the Q2 planning deck",
    });
    showPendingUI();
  } else {
    showHardDeniedUI();
  }
}
```
{{/lang}}

`canRequestAccess` returns `true` only when:
- Caller is an `AppUser` (regular member, not anonymous).
- Caller is **not** an admin/owner (they already have access).
- Document exists, is not the root document.
- Caller has no existing direct or group permission.

#### `requestAccess` requires `permission`

{{#lang ts}}
```typescript
await client.documents.requestAccess(documentId, {
  permission: "read-write",          // REQUIRED — "read-write" or "reader"
  message: "...",                    // optional, max 500 chars
  documentUrl: "https://...",        // optional, embedded in owner email
  reviewUrl: "https://...",          // optional, owner's review CTA
  sendEmail: false,                  // optional, default true
});
```
{{/lang}}

Calling with no `permission` will 400. Re-requesting from the same user updates the existing pending request silently — there is no `ACCESS_REQUEST_ALREADY_PENDING` error. Real codes:

| Body code | When |
|-----------|------|
| `ALREADY_HAS_ACCESS` | Caller already has any permission |
| `RATE_LIMITED` | Too many requests; `details.retryAfter` gives the wait in seconds |
| `ACCESS_REQUEST_ALREADY_RESOLVED` | Approve/deny called on a non-pending request |

#### Owner / admin flow

The owner lists pending requests with `listAccessRequests`, then resolves each with `approveAccessRequest` (optional `permission` override) or `denyAccessRequest`.

{{#lang ts}}
```typescript
const requests = await client.documents.listAccessRequests(documentId);
// DocumentAccessRequest[]: { requestId, requesterId, status, requestedPermission, message?, ... }

await client.documents.approveAccessRequest(documentId, requestId, {
  permission: "read-write",          // optional override
  documentUrl: "https://...",        // optional, embedded in approval email
});

await client.documents.denyAccessRequest(documentId, requestId, {
  documentUrl: "https://...",        // optional
});
```
{{/lang}}

> **Note:** `denyAccessRequest` does not accept a `reason` field — only `documentUrl`. Don't try to pass one.

#### Constraints

- 30-day TTL on unresolved requests.
- One pending request per `(document, requester)` — re-requesting updates it in place.
- Resolved requests are immutable.

#### Refreshing access requests

Refresh `listAccessRequests()` when the owner opens the requests view. The requester receives an email with the outcome.

### Collections

A collection shares a set of documents as one unit. Access from collections and direct grants combines; the highest permission wins. Deleting a collection preserves its documents.

{{ example: sharing/collection-create }}

{{ example: documents/collection-manage }}

{{ example: sharing/collection-access }}

#### Gotchas for collections

- Use collection-level sharing when the same people need the same set of documents. Avoid duplicating those grants on each document.
- `collectionType` cannot change after creation.
- `initialMetadata` validates and writes up to 10 categories during creation; category write rules are waived for that initial write.
- Collection-wide sharing changes and deletion fail above 200 documents. Deletion also checks a documents × groups limit of 400. Remove documents or grants before retrying `COLLECTION_FANOUT_LIMIT`.
- Emails for people who have not signed up create deferred grants.

### Collection Rule Sets

Collections use the same rule-set pipeline as groups — a `CollectionTypeConfig` row binds a `collectionType` to a rule set. The CEL namespace is `collection.*` (separate from `group.*`), and an extra helper `hasCollectionAccess(collectionId)` is available only inside collection rule sets. The SDK equivalents are `client.collectionTypeConfigs.{ list, get, create, update, delete }` (parallel to `client.groupTypeConfigs.*`). The rule-set mechanism itself — defining and binding a rule set, owner/admin bypass, `test()`/`debug()` — is documented once in the [Access Control guide's rule sets section](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md#rule-sets-management-operations); read that first. See [Rule Sets for Groups](AGENT_GUIDE_TO_PRIMITIVE_USERS_AND_GROUPS.md#rule-sets-for-groups) in the Users and Groups guide for the parallel group treatment.

**Resource type:** `collection`. **Categories and operations:**

- `category: "collection"` — `create`, `edit`, `delete`, `get` (the read op, parallel to `group.get`; use `get` in TOML configs).
- `category: "document"` — `add`, `remove`, `delete`, `list` (controls which documents the collection can hold, plus authorization for deleting a member document outright — see [Deleting Documents](#deleting-documents)).
- `category: "member"` — `add`, `remove`, `list`.

**Default rule set** — applies to any collection type with no `CollectionTypeConfig` row:

| Op | Default | Meaning |
|----|---------|---------|
| `collection.create` | `"true"` | Any signed-in member |
| `collection.edit` / `delete` | `user.userId == collection.createdBy` | Creator only |
| `collection.get` | `user.userId == collection.createdBy \|\| hasCollectionAccess(collection.collectionId)` | Creator or collection member (direct or via `CollectionGroupPermission`) |
| `document.add` / `remove` | `user.userId == collection.createdBy` | Creator only |
| `document.delete` | `"false"` | Denied. Unlike every other write op above, NOT creator-only — an app must explicitly configure this op to let a non-owner/non-app-owner delete a member document at all |
| `document.list` | `user.userId == collection.createdBy \|\| hasCollectionAccess(collection.collectionId)` | Creator or collection member |
| `member.add` / `remove` | `user.userId == collection.createdBy` | Creator only |
| `member.list` | `user.userId == collection.createdBy \|\| hasCollectionAccess(collection.collectionId)` | Creator or collection member |

A non-creator reader/writer removing their own membership via `member.remove` is denied (403) unless the rule set grants it.

`document.delete` is distinct from `document.remove`: `remove` only detaches a document from this collection, while `delete` authorizes destroying the whole document (`client.documents.delete`) when the caller isn't the document's owner or the app owner — the delete endpoint checks every collection containing the document and allows the delete if any one collection's `document.delete` rule passes. Because that check runs per collection, granting `document.add` on a collection can extend who is able to delete documents placed in it — configure `document.add` and `document.delete` together with that reach in mind.

**CEL context**, beyond the identity context:

| Variable | Always present? | Description |
|----------|-----------------|-------------|
| `collection.collectionType` | yes | Collection's type (matches the `CollectionTypeConfig` this rule set is bound to) |
| `collection.collectionId` | yes (after create) | Collection's ID |
| `collection.name` | yes | Display name |
| `collection.createdBy` | yes (after create) | userId of the collection's creator |
| `target.userId` | only `category: "member"`, ops `add` / `remove` | The user being added or removed. Absent for `member.list`. |

Plus the collection-only helper:

- `hasCollectionAccess(collectionId)` — true when the caller has direct collection membership (the platform-managed `_col-reader` / `_col-writer` system groups) OR membership in a non-system user-group that holds a `CollectionGroupPermission` of `reader` or `read-write` on the collection. Resolves to `false` outside collection rule sets, and to `false` on `collection.create` (no `collectionId` in scope yet).

### Keying a Collection Rule Set on an External Id

Store a collection's external-entity id (the class, team or project it represents) in a [resource metadata](AGENT_GUIDE_TO_PRIMITIVE_RESOURCE_METADATA.md) category and read it in the rule set as `md.self.<category>.<key>`. The category a rule references is inferred and loaded automatically — no declaration needed — and reads `null` until a value is stored, so stamp the value when the collection is created.

**1. Define a category** for the link, with separate read/write rules:

```toml
# primitive/dev/metadata-category-configs/collection.classLink.toml
[metadataCategoryConfig]
resourceType = "collection"
category = "classLink"
readRule = "true"
writeRule = "user.userId == resource.attrs.createdBy"

[metadataCategoryConfig.schema.fields.classId]
type = "string"
required = true
```

**2. Read `md.self.classLink.classId`** in the rule set, and stamp `classId` when the collection is created (via `initialMetadata` on `collections.create()`) so the value exists when these ops evaluate:

```toml
[rules.collection]
get    = "isMemberOf('class', md.self.classLink.classId) || hasCollectionAccess(collection.collectionId)"

[rules.document]
add    = "isMemberOf('class', md.self.classLink.classId)"
list   = "isMemberOf('class', md.self.classLink.classId) || hasCollectionAccess(collection.collectionId)"
```

The `collection.create` rule reads the value staged in the same `collections.create()` call — see [Gating Collection Creation on Staged Metadata](#gating-collection-creation-on-staged-metadata).

### Gating Collection Creation on Staged Metadata

A collection type's `collection.create` rule is evaluated against the `initialMetadata` staged in the same `collections.create()` call — **before** the collection is persisted. The staged values bind to `md.self.<category>.<key>`, so a create rule can gate creation on the exact linkage the create is about to stamp:

```toml
# the collection type's create rule
create = "isMemberOf('class-teachers', md.self.classLink.classId)"
```

- The `md.self.attrs.*` projected columns (`collectionType`, `name`, `createdBy`) are also bound in the create rule; `collectionId` is `null` (unassigned).
- **Fail-closed:** once a create rule reads `md.self.<category>`, a create omitting that category is denied (the value binds `null`), so a create that stamps the linkage in a later write is denied for that type — the linkage must be staged in the create call (atomic create-with-linkage).
- **No traversal from the staged subject.** A create rule may read the staged value directly (`md.self.<category>.<key>`) but may not follow a declared path off it (`md.<pathName>.*`) — such a rule is rejected when the rule set is saved, since the subject does not exist yet to traverse from. (Traversal from a *persisted* subject in a non-create rule is unaffected.)

### Migrating off `contextId`

Collections used to carry a `contextId` field, read in rules as `collection.contextId`. The field is removed from the API, the rule context, the clients and the CLI; a metadata category ([Keying a Collection Rule Set on an External Id](#keying-a-collection-rule-set-on-an-external-id)) binds a collection to an outside entity now.

- **Rules.** A collection rule set that reads `collection.contextId` (any selector form, `.?contextId` and `["contextId"]` included) or `md.self.attrs.contextId` is refused at save, and so by `primitive config push`, with an error naming the replacement. A rule set saved before the removal that still reads it is **denied** for every operation, whatever its shape — `!has(collection.contextId) || …` denies rather than allowing. Rewrite it to read `md.self.<category>.<key>`. `group.contextId` in group rule sets is unaffected.
- **Creation code.** `collections.create()` / `collections.update()` with `contextId` is a 400, whatever the value, before anything is created. Pass `initialMetadata: { <category>: { <key>: <value> } }` on the create.
- **Workflow steps.** A `collection.create` step carrying `contextId` fails the run naming `initialMetadata`; `primitive config push` refuses the key. Set `initialMetadata` on the step; templates resolve inside it.
- **CLI import.** `collections export` writes no `contextId`; an older `collections.json` still imports, without the field, with one note counting the collections that carried it.
- **Existing collections.** The platform does not copy stored values across, and once the removal is deployed no API returns them. Before that release is deployed, define the category and copy each value — for every collection `primitive collections list --all --json` lists with a `contextId`:

```bash
primitive metadata set collection <collectionId> classLink --data '{"classId":"<contextId>"}'
```

### Building a "Members + Pending" UI

Show current access separately from invitations waiting for signup. Refresh after a membership change:

{{ example: sharing/document-members }}

{{ example: sharing/collection-access }}

### Sharing Discovery Cheat Sheet

Pick the call that answers the question you're actually asking:

{{#lang ts}}
| Question | Call |
|----------|------|
| Documents the user owns | `client.me.ownedDocuments({ tag?, limit?, cursor?, returnPage? })` |
| Documents directly shared with the user (direct non-owner grants) | `client.me.sharedDocuments({ tag?, cursor?, limit? })` → `{ items, nextCursor, hasMore }` |
| A document's outstanding deferred grants | `client.documents.listPendingInvitations(documentId)` |
| Documents inside a collection | `client.collections.listDocuments(collectionId, { limit?, cursor? })` |
| Documents shared with a group | `client.groups.listDocuments(groupType, groupId)` |
| Collections the user is a direct member of | `client.collections.list({ limit?, cursor? })` |
| Members + groups on a collection | `client.collections.getAccess(collectionId)` |
| Group permissions on a document | `client.documents.listGroupPermissions(documentId)` |
| Pending email invites on a document / group / collection | `client.{documents\|groups\|collections}.listPendingInvitations(...)` |
{{/lang}}
{{#lang swift}}
| Question | Call |
|----------|------|
| Documents the user owns | `client.me.ownedDocuments(cursor:limit:tag:)` → `[DocumentInfo]` (or `ownedDocumentsPage(cursor:limit:tag:)` → `DocumentListPage` for the `{ items, nextCursor, hasMore }` envelope) |
| Documents directly shared with the user (direct non-owner grants) | `client.me.sharedDocuments(cursor:limit:tag:)` → `SharedDocumentListResult` (`{ items, nextCursor, hasMore }`) |
| A document's outstanding deferred grants | `client.documents.listPendingInvitations(documentId:)` |
| Documents inside a collection | `client.collections.listDocuments(collectionId:options:)` → `PaginatedResult<CollectionDocumentInfo>` |
| Documents shared with a group | `client.groups.listDocuments(groupType:groupId:)` → `[GroupDocumentInfo]` |
| Collections the user is a direct member of | `client.collections.list(options:)` → `PaginatedResult<CollectionInfo>` |
| Members + groups on a collection | `client.collections.getAccess(collectionId:)` |
| Group permissions on a document | `client.documents.listGroupPermissions(documentId:)` |
| Pending email invites on a document / group / collection | `client.documents.listPendingInvitations(documentId:)` / `groups.listPendingInvitations(groupType:groupId:)` / `collections.listPendingInvitations(collectionId:)` |

`options:` takes a `PaginationOptions(limit:cursor:)`; the paginated reads return `PaginatedResult` (`.items`, `.nextCursor`, `.hasMore`).

The permission / collection reads — `documents.getPermissions(_:)`, `collections.getAccess(_:)`, `collections.listPendingInvitations(_:)` — return the raw server rows. A document permission row carries `email` (and `name` when the user has one) alongside `userId`; a collection member row carries only `userId`, so a UI that shows who has access to a collection resolves the email and name itself.
{{/lang}}

**Anti-patterns:**

{{#lang ts}}
- Calling a method that doesn't exist: `setPermissions`, `setGroupPermission`, `client.users.lookup({ email })`. The correct names are `updatePermissions`, `grantGroupPermission`, `client.users.lookup(email)`.
{{/lang}}
{{#lang swift}}
- Calling a method that doesn't exist: `setPermissions`, `setGroupPermission`. The correct names are `updatePermissions(documentId:params:)` and `grantGroupPermission(documentId:params:)`; resolve a user by email with `client.users.lookup(email:)`.
{{/lang}}
- Passing `permission: null` to remove a grant — there is no null form. Use `removePermission`.
- Lowering a user's direct permission while they still have a higher one via group — the group wins (effective = MAX).
- Assuming `me.sharedDocuments()` includes group- or collection-shared docs — it only carries direct grants. Combine with `collections.list()` / `groups.listUserMemberships(...)` for a complete picture.
- Showing a "request access" button without checking the caught error's `canRequestAccess` detail.
- Calling `client.invitations.delete()` to cancel a single pending document share — it cascades to every share and group add linked to that invitation.
- Polling `client.invitations.list` / `listDeferredGrants` to populate "Members + Pending" rows — those are app-level / admin surfaces; per-resource `listPendingInvitations` is the product UI source.

**Sharing error codes** (server-emitted body codes):

{{#lang ts}}
The client throws a typed `JsBaoApiError` with `.status`, `.code`, and `.body` (the parsed error body) — read the code and details straight off the caught error, no manual message-parsing needed. `JsBaoApiError` means the server responded (non-2xx). A request that never reaches the server throws one of two retryable errors instead: a `JsBaoError` with code `OFFLINE` when the client is offline (pinned, or no network detected) and never attempts the request, or `JsBaoNetworkError` when the attempt fails in transit (DNS failure, connection refused or aborted) — no `.status`, just `.cause` carrying the underlying fetch error. Branch with `isJsBaoError(err) && err.code === "OFFLINE"` and `isJsBaoNetworkError(err)`, never by parsing the message.
{{/lang}}
{{#lang swift}}
The client throws a typed `JsBaoError` with `.code` and `.details` — read the code and details straight off the caught error, no manual message-parsing needed.
{{/lang}}

| Code | Endpoint | Meaning |
|------|----------|---------|
| `DOC_ACCESS_DENIED` (with `details.canRequestAccess`) | `documents.get` | 403; check the hint to decide between request-access UI and hard deny |
| `ALREADY_HAS_ACCESS` | `requestAccess` | Caller already has a direct or group permission |
| `RATE_LIMITED` | `requestAccess` | `details.retryAfter` gives the wait in seconds |
| `ACCESS_REQUEST_ALREADY_RESOLVED` | `approveAccessRequest`, `denyAccessRequest` | Request is no longer pending |

App-membership and invitation error codes live in the [Invitations guide](AGENT_GUIDE_TO_PRIMITIVE_INVITATIONS.md#error-codes-quick-reference).

### Read-Only Permission Handling

When a user has "reader" permission, disable all edit functionality: hide create/add buttons, delete buttons and drag handles, and share buttons (only owners can share); disable inputs and checkboxes; and prevent inline editing. Derive a read-only flag from the document's `permission` field (`"reader"`) and thread it down to child views.

{{#lang ts}}
```typescript
const isReadOnly = computed(() => {
  if (!currentDocumentId.value) return true;
  const doc = todoStore.todoListDocuments.find(
    (d) => d.documentId === currentDocumentId.value
  );
  return doc?.permission === "reader";
});
```

Pass `isReadOnly` to child components and use it to gate UI: `v-if="!isReadOnly"` on create/delete controls, `:disabled="isReadOnly"` on inputs.
{{/lang}}

## Admin CLI: Export / Import

Export each user's documents separately, then import with the intended owner:

```bash
primitive documents export-all --user-id <user-id> --output ./export
primitive documents import ./export --owner user@example.com
```

### Gotchas when importing

- `--overwrite` merges ordinary document data; it does not guarantee imported values win. Large-document imports require an empty target.
- IDs are preserved except for root documents, which import into the target user's root document.
- Restore document sharing in the target app. Import documents before collections.
- `collections import --dry-run` previews changes. Existing names are skipped without `--overwrite`; overwrite requires matching owner and type. A file that still carries `contextId` imports without it, with one note.
- `collections import` matches by name. A name that more than one target collection has is ambiguous: that collection is refused as a `collection` problem naming every matching ID, with or without `--overwrite`, and nothing is written for it. A name repeated inside the file is refused for the second and later occurrences. A missing owner or an ambiguous target name refuses that collection outright.

## Admin CLI: Inspecting documents

```bash
primitive documents get <document-id>
primitive documents records models <document-id>
primitive documents records query <document-id> <model>
primitive documents permissions list <document-id>
```

## Large Documents

Choose `documentFormat: 2` at creation for datasets validated at 2 GB. The format cannot change later. Configure persistent local storage, then use the ordinary document API.

{{ example: documents/create-large-document }}

The same format option applies to `getOrCreateWithAlias`; use an alias to create one document safely across concurrent attempts.

### Gotchas for large documents

- An explicit format on `getOrCreateWithAlias` must match an existing document or the call fails with `DOCUMENT_FORMAT_MISMATCH`.
- Offline writes expire after the configured window (7 days by default, configurable from 1 to 14 days). Sync before resuming writes. Handle `DOCUMENT_OFFLINE_WINDOW_EXPIRED`.
- Queries must select ordinary documents or large documents, not both. Use `documents` to avoid `FORMAT2_QUERY_SCOPE`.
- Nested collaborative values are rejected; store plain JSON field values.
- Sync pending changes before eviction. Without `force`, eviction refuses to discard unsynced writes.
- On `FORMAT2_FOLD_BROKEN`, reconnect to restore the local data before reading or writing.

{{#lang ts}}
For persistent browser storage, configure `databaseConfig: { type: "opfs", options: { workerURL, brokerURL } }`. `brokerURL` enables sharing across tabs. An unavailable store reports `FORMAT2_STORAGE_UNAVAILABLE`.
{{/lang}}

{{#lang swift}}
Use an on-disk store. An in-memory store cannot open a large document and reports `FORMAT2_STORAGE_UNAVAILABLE`.
{{/lang}}

### Snapshotting a large document on demand

```bash
primitive documents snapshots build <document-id> --wait
primitive documents snapshots list <document-id>
primitive documents snapshots get <document-id> <build-id>
```

Only verified snapshots are served to clients. Creating one requires document write access or app-admin authority. Stopping the CLI wait leaves the build running.

### Bulk-loading a large document

```bash
primitive documents ingest <document-id> --input ./records -y
primitive documents ingests get <document-id> <session-id>
```

Supply one `<model>.ndjson` or `<model>.ndjson.gz` file per model, or an export of the same document. Each line identifies a record and supplies a merge patch. `null` removes a field; `{"_deleted": true}` deletes the record.

**Gotchas:** loading changes live records and cannot be undone. All records become visible together. Ending the CLI wait does not stop the load. Handle `documentOfflineWritesResolved` when a load overlaps offline writes: writes to deleted records can be dropped, and other writes may need review.

{{#lang ts}}
## Gotchas

| Symptom                                         | Cause                                                                    | Fix                                                                                                  |
| ----------------------------------------------- | ------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------- |
| Need document from item                         | N/A                                                                      | Use `item.getDocumentId()`                                                                           |
| Data doesn't update when route param changes    | Vue reuses components; `useJsBaoDataLoader` doesn't see the change       | Add the route param to `queryParams` in the data loader, OR use `:key="routeParam"` on the component |
| Store loads once but never reacts to changes    | `useJsBaoDataLoader` called outside a component — its `onMounted` subscriptions never register | In a Pinia store / non-component context call `Model.subscribe(reload)` directly in `setup()`. See [Subscribing Outside a Component](#subscribing-outside-a-component) |
| Spread/clone of a model is empty or missing fields | Fields are prototype getters, not own properties, so `{ ...model }` / `{ id, ...rest }` copy nothing | Read fields explicitly: `{ id: model.id, title: model.title }`. See [Model Instances Are Not Plain Objects](#model-instances-are-not-plain-objects) |
| Query `field: false` misses items               | Items with `field: undefined` don't match `field: false`                 | Use a default value in schema, OR filter in JavaScript with `item.field ?? false`                    |
| Document created but not in sidebar/list        | Created via `documents.create()` directly without updating tracked state | Push the new entry into your own document store (or re-run `me.ownedDocuments()`) so reactive lists update |
| HTTP 400 when sharing with email                | `sendEmail: true` with no `documentUrl` in the request                    | Pass `documentUrl` to `documents.updatePermissions()` — see [Building a share UI](#building-a-share-ui)     |
| New document not queryable immediately          | Document not opened after creation                                        | After `documents.create()`, call `documents.open(metadata.documentId)` before querying              |
| `setPermissions is not a function`              | Method doesn't exist                                                      | Use `updatePermissions(documentId, { userId, permission })`                                          |
| `setGroupPermission is not a function`          | Method doesn't exist                                                      | Use `grantGroupPermission(documentId, { groupType, groupId, permission })`                           |
| "Model not properly initialized" on save/query  | Schema not registered — `*.generated.ts` or `index.ts` is out of sync with `models.toml` | Re-run `npx js-bao-codegen-v2`; never manually attach a schema or edit generated files               |
| `upsertByUnique`: "constraint not found"        | Passed field array instead of constraint name                             | Pass the named constraint string (e.g. `"users_email_unique"`)                                       |
| `upsertByUnique`: "targetDocument is required"  | Creating a new record without specifying its document                     | Pass `{ targetDocument: docId }` as the 4th argument                                                 |
| `query()` result missing `.map`/`.filter`       | Forgot result is a `PaginatedResult`                                      | Use `result.data` (also has `.nextCursor`, `.hasMore`)                                               |
{{/lang}}
