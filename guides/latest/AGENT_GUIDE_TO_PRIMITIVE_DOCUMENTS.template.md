# Working with Documents in the Primitive platform.

Guidelines for building apps with Primitive's document-based architecture.

## Core Concept: Documents

A **document** is:

1. A container for js-bao model objects
2. A sharing boundary—each document can be shared with different users at different permission levels
3. Can be used entirely interally, or exposed to app users as a concept that make sense for the application. For example a document might map to a Company in a business application, a Portfolio in a financial app, or a Channel in a communication app.

**Properties:**

- Documents are read/written locally. js-bao handles sync with the server.
- When other clients edit a document, local data updates in real-time.
- Access is all-or-nothing: users either have access to the entire document or none of it.

**Decision rule**: If data needs to be shared independently, it belongs in separate documents.

**Permission Levels:**

- **Reader** - View-only access
- **Read-write** - View and edit capabilities
- **Owner** - Full control including sharing and deletion

**Size Guidelines:** Documents work best around ~10 MB each (soft limit). For most apps (thousands of records, years of data), this is sufficient.

## Documents vs. Databases

Primitive also provides **Databases** — isolated, server-side storage. Documents are best for personal data, real-time collaboration, and offline access. Databases are best for app-wide shared data, large datasets, and fine-grained access control. Many apps use both.

See the [Data Modeling guide](AGENT_GUIDE_TO_PRIMITIVE_DATA_MODELING.md) for a full decision framework, comparison table, and example app architectures. See the [Databases guide](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md) for database API documentation.

## Critical Rules

1. **Queries operate over ALL open documents.** NEVER iterate over documents to query. Filter results by documentId or other fields in the query itself.

2. **Use model IDs, not document IDs.** Model data references entirely in objects using model IDs. Use documentIds ONLY when required for APIs (sharing, save location). In routes and queries, prefer model IDs.

3. **NEVER remove fields from models.** Add a deprecation comment instead.

4. **ALWAYS declare new models in the schema and regenerate.**
{{#lang ts}}
   Add the model to `models.toml`, then run `npx js-bao-codegen-v2`.
{{/lang}}
{{#lang swift}}
   Add the model to `models.toml`; codegen runs on every build (regenerate by hand with `swift build` or the `run-ios.sh` codegen step if you edited the schema between builds).
{{/lang}}

5. **Prefer query filtering over in-memory filtering.** Push filter conditions into the query itself rather than fetching everything and filtering on the client.

6. **Load data at the screen level, not in leaf components.** Pass data and readiness into sub-components as props.

7. **Understand the root document's role and limitations.** The root document is a special per-user document that is automatically created and opened. It can never be shared or deleted, and there is exactly one per user. It's a natural home for user preferences and settings that should be available whenever the user signs in. The root document can hold any model, but store most application data in regular documents for greater flexibility (sharing, collaboration, multiple documents). Use the "single document" pattern with aliases for personal apps, or the "one document at a time" pattern for multi-workspace apps.
{{#lang ts}}
   The template stores user preferences in the root document via `userStore`.
{{/lang}}

## Document Lifecycle

### 1. Open Documents Before Querying

Documents must be opened before querying or modifying data within them.

{{ example: documents/doc-open-query }}

Documents are ready to be queried once the `.open()` call finishes. Applications should wait for all required documents to be opened and show a loading state until then, then track document-specific readiness explicitly. `open()` is idempotent — calling it on an already-open document is a no-op. Handle open failures explicitly: surface an error or redirect; don't silently continue.

Open only the documents you need to query or want real-time updates from — every open document syncs continuously. Don't open documents you don't need, and close ones you're done with (see [Closing Documents](#closing-documents)).

{{#lang ts}}
Track readiness with an `isReady` ref.

**Wrong** — querying or saving before the document is open throws (or returns nothing for queries on no-document models):

```typescript
// DON'T: kick off open() and immediately query
jsBaoClient.documents.open(documentId);          // missing await
const result = await TodoItem.query({});         // throws DocumentClosedError on save,
                                                 // returns empty data for query
```

**Note on `jsBaoDocumentsStore.isReady`:** The template provides `jsBaoDocumentsStore` with an `isReady` property. This indicates that the **store itself** has finished initializing (the document list and invitation list have loaded) — it does NOT indicate that any particular document has been opened. You still need to track document-specific readiness separately (e.g., after calling `documents.open()`) before querying data in those documents.

**Where in the Vue tree to open documents:**

- **Open in pages/layouts/stores, not sub-components.** Sub-components receive readiness and data as props.
- **Open after authentication, not before.** Gate on `userStore.isAuthenticated` (not `isInitialized`). The template's `AppLayout` already gates rendering on `isAuthenticated`, so components mounted inside it can call `open()` safely.
- **Session-scoped documents** (small bounded set, < ~20): open once at app/layout level for the session.
- **Route-scoped documents** (per-page, unbounded count, or transient): open on route entry, close on route leave. Render a loading state until `documents.open()` resolves and (with `useJsBaoDataLoader`) `initialDataLoaded` is true.
- **Stray opens widen query scope** — JavaScript queries span every open document by default, so an unneeded open document doesn't just cost sync traffic, it changes query results.
{{/lang}}

{{#lang swift}}
The document-open lifecycle is owned by `PrimitiveAppState` (see [The `PrimitiveAppState` document lifecycle](#the-primitiveappstate-document-lifecycle) below). Subclass it and drive doc setup from `connectClient()`; reads and writes go through the codegen'd model facade (`TaskRecord.query(...)`, `record.save(in:)`), so there is no per-document binding step. For session-scoped documents (small bounded set), open once at launch; for transient per-item docs, open on view appear and close on disappear. Stray opens widen query scope — facade queries span every open document by default.
{{/lang}}

### 2. Finding documents a user can access

There is **no single "my documents" list**. A user reaches documents through **four distinct paths** — query each separately and combine in the UI. Do NOT try to unify them into one server-side list.

**a. Documents they own** (`ownedDocuments` — created, or ownership transferred):

{{ example: documents/list-owned }}

**b. Documents shared directly with them** (`sharedDocuments` — non-owner `DocumentPermission` rows + pending `DocumentInvitation`s; group/collection shares do NOT appear here):

{{ example: documents/list-shared }}

**c. Documents shared via a group** (`groups.listDocuments`):

{{ example: documents/list-group-documents }}

**d. Documents shared via a collection** (`collections.listDocuments`):

{{ example: documents/list-collection-documents }}

{{#lang ts}}
Each row from `sharedDocuments` extends the base `DocumentInfo` (`title`, `createdBy`, `createdAt`, `lastModified`, plus `tags`/`metadata`/`thumbnailBlobId` when set) with the share-only extras `permission` (never `"owner"`), `source` (`"permission"` | `"invitation"`), `grantedBy`, `invitationId` (invitation rows only).
{{/lang}}
{{#lang swift}}
Swift can't inherit a struct, so each row from `sharedDocuments` is a `SharedDocument` that holds the base fields under `.document` (a `DocumentInfo`, including `permission` — never `.owner`) alongside the share-only extras `grantedBy`, `source` (`"permission"` | `"invitation"`), and `invitationId` (invitation rows only). Group and collection rows have their own shapes — `groups.listDocuments` returns `GroupDocumentInfo`, and `collections.listDocuments` returns flat `CollectionDocumentInfo` (`documentId`, `title`, `permission`, `addedBy`, `addedAt`, …).
{{/lang}}

{{#lang ts}}
`ownedDocuments` and `sharedDocuments` return the unified `{ items, cursor }` envelope (raw-JSON `cursor`, NOT base64url). `ownedDocuments()` returns a flat `DocumentInfo[]` by default, or the envelope with `returnPage: true`.

For an "everything I can access" surface, combine these two calls with group and collection memberships:

```typescript
const owned  = await jsBaoClient.me.ownedDocuments();
const shared = (await jsBaoClient.me.sharedDocuments()).items;
const collections = await jsBaoClient.collections.list();
// then iterate collections / groups.listUserMemberships and call
// collections.listDocuments / groups.listDocuments.
```

`jsBaoClient.documents.hasLocalCopy(documentId)` is the synchronous local-cache check, useful when deciding whether to render skeletons before `open()` resolves.

#### Do not use

- **`client.documents.list()`** — returns the union of owner + reader + read-write rows and logs a console warning on every call. Use `me.ownedDocuments` and `me.sharedDocuments`; they have the same option set (`tag`, `limit`, `cursor`, `returnPage`).
- **`client.documents.createInvitation(...)`, `documents.acceptInvitation(...)`, `documents.declineInvitation(...)`, `documents.listPendingInvitationsForUser(...)`** — the per-document `DocumentInvitation` flow. Use `documents.updatePermissions(documentId, { email, ... })` for the share path; the platform creates an `AppInvitation` + `DeferredDocumentPermission` and the recipient redeems it via `client.invitations.accept(inviteToken)`. `client.me.pendingDocumentInvitations()` is the current "invitations I can accept" lookup.
- **`client.me.bookmarks.*`** — render "my documents" from `me.ownedDocuments()` + `me.sharedDocuments()` (and `collections.list()` / `groups.listUserMemberships(...)` if you also want group/collection access).
{{/lang}}

## Core data operations

Every example below is compiled against the real client as part of the docs build. The [Querying Data](#querying-data) and [Saving Data](#saving-data) sections below go deeper on projections, includes, and save options.

{{#lang swift}}
The generated model statics route through the process-wide default client — call `JsBaoClient.configureDefault(client)` once at startup, before the first read or write. `query`, `queryOne`, `count`, and `aggregate` are synchronous against the local document store; `find` and `findAll` are `async throws` (use `try await`). All reads span every open document by default (scope with `QueryOptions(documents: [docId])`); writes target one document — `save(in:)` inserts or updates in place and throws (it validates the write and requires the document to be open), `delete(in:)` throws only if the document isn't open.
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

### Sort + cursor pagination

{{ example: documents/query-paginate }}

### Aggregation

{{ example: documents/aggregate }}

Grouping by a `stringset` field counts per member value (facet); a membership `groupBy` entry groups by whether the set contains one specific value. Only one stringset facet field is allowed per aggregation, and a facet can't be mixed with other `groupBy` entries — unsupported mixes degrade (the facet is dropped or the result is empty) rather than throw.

### Subscribe to changes

{{ example: documents/subscribe }}

### Resolve-or-create a singleton document

{{ example: documents/get-or-create-doc }}

### Share a document (user / email / group)

{{ example: documents/share-document }}

### Update thumbnail / metadata

{{ example: documents/update-metadata }}

## Common Document Usage Patterns

{{#lang ts}}
**Helper Stores:** The template includes `jsBaoDocumentsStore` (the underlying tracked-documents store) and `singleDocumentStore` (a higher-level wrapper for Pattern 1 / Pattern 2) in `/src/stores/`. These stores handle document opening, closing, readiness tracking, and state management. They can be used as-is, customized to fit your needs, or ignored in favor of application-specific approaches.
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
**UI components** in `src/components/documents/`: `PrimitiveDocumentSwitcher` (sidebar dropdown) and `PrimitiveDocumentList` (full management page with rename/share/delete).
{{/lang}}

List with `me.ownedDocuments()` and `open()` the selected document; create a new workspace document with `create()` and open it:

{{ example: documents/create-document }}

### Pattern 3: Multiple Documents

**Best for:** Apps that query across many documents, each with its own sharing context — chat (per channel), multi-tenant dashboards, collaborative workspaces with distinct collections.

All documents that need live updates or cross-document queries must be open. Tag documents so you can fetch a set with a tag-filtered `me.ownedDocuments` (and `me.sharedDocuments` if the user can also be a non-owner), open each, and track per-document readiness yourself. For collective sharing of multiple documents as a unit, prefer the server-side Collections API (`client.collections.*` — see [Collections](#collections) below) over local tracking.

{{ example: documents/open-tagged-docs }}

{{#lang ts}}
The template does not ship a built-in "multi-document" Pinia store. A minimal store for "all documents tagged `channel`":

```typescript
// stores/channelDocsStore.ts
import { defineStore } from "pinia";
import { ref, computed } from "vue";
import { useJsBaoClient } from "primitive-app";

export const useChannelDocs = defineStore("channelDocs", () => {
  const client = useJsBaoClient();
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
{{/lang}}

{{#lang swift}}
The most common multi-doc shape is one ambient library/index document plus N per-item documents (each shareable independently); see [Index doc + per-item docs](#index-doc--per-item-docs-keying-tombstones-reconcile) below for keying, tombstones, and the reconcile pass. Open auxiliary docs with `appState.openAuxiliaryDoc(_:)` and close them with `appState.closeAuxiliaryDoc(_:)`.

### The `PrimitiveAppState` document lifecycle

The neutral lifecycle is: resolve-or-create the doc (`client.documents.getOrCreateWithAlias` — see [Resolve-or-create a singleton document](#resolve-or-create-a-singleton-document)), open it, then read through the codegen'd facade scoped to it (see [Read](#read-find--query--first--count)). Models need no per-document binding — the facade statics read from every open document, backed by the process-wide default client (`JsBaoClient.configureDefault`).

`PrimitiveAppState` is the **SwiftUI app-state glue** (PrimitiveApp package) that owns that lifecycle: subclass it for app-specific state and override `connectClient()` to drive doc setup after connect (`configureDefault` is wired for you during `initialize()`). The subclass below is framework glue; the Primitive call in it is the `getOrCreateWithAlias` resolve-or-create:

{{ example: documents/app-state-lifecycle }}

For per-document setup beyond models, override the `onDocumentOpened(doc:documentId:)` hook — the base class opens the doc once and hands you the live `YDocument`, so don't call `openDocument(...)` again just to get one.

For a **fresh doc you'll write immediately**, use `client.createDocument(options:)` — it returns a `CreateDocumentResult` with `metadata: JSONValue?`; extract `metadata?["documentId"]?.stringValue` to get the new document's id. The new document is already open and writable, so write through it directly rather than reopening. If you do reopen with `waitForLoad: .network`, Swift retries sync availability until the background create commit lands; healthy cases resolve after the handshake, with the fallback bounded by `availabilityWaitMs` (default 30s).

**Multi-doc apps (one ambient library doc + N per-item docs).** `selectDocumentAwaiting(_:)` is the *single*-selected-doc lifecycle — it closes the previously selected doc first, so using it for a per-item detail view closes your library/index doc. For one ambient doc plus transient detail docs, use `appState.openAuxiliaryDoc(_:)` from the detail view's `.task` and `appState.closeAuxiliaryDoc(_:)` from `.onDisappear`. These register the doc for sync, but they don't touch `selectedDocId` or fire `onDocumentOpened`. Once open, read the doc's records through the facade scoped to that document — the same scoped query as [Open Documents Before Querying](#1-open-documents-before-querying). The view is framework glue around `openAuxiliaryDoc` + that scoped query:

{{ example: documents/app-state-auxiliary-doc }}

**Watch for access loss in detail views.** When a peer revokes your access or hard-deletes a doc you have open, the server collapses both to the same wire shape: a `.documentMetadataChanged` event with `action == "deleted"` (and `metadata == nil`). Subscribe to it, filter on `action == "deleted"`, and dismiss the view. Retain the returned `EventSubscription` on a property and `[weak self]` the closure, or the handler is dropped:

```swift
private var deletedSub: EventSubscription?

.task {
    deletedSub = client.events.on(.documentMetadataChanged) { [weak self] (ev: DocumentMetadataChangedEvent) in
        Task { @MainActor in
            guard let self, ev.action == "deleted", ev.documentId == self.openDocId else { return }
            self.dismiss()
        }
    }
}
```
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

**CRITICAL: NEVER edit `*.generated.ts` files or `src/models/index.ts`.** Both are overwritten on every codegen run.
{{/lang}}

{{#lang swift}}
### `models.toml` + codegen + the model facade

Define models in `Sources/MyApp/Models/models.toml`; `swift-bao-codegen` emits one `PrimitiveModel` struct per `[models.X]` block into a gitignored `Models/Generated/`. The generated type IS the API: static reads across all open documents (`TodoItem.query(...)`, `find`, `findAll`, `count`, `aggregate`, `subscribe`), instance writes targeting one document (`try record.save(in: documentId)`, `try record.delete(in: documentId)`), backed by the process-wide default client.

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

`required = true` makes the emitted `init?(record:)` reject construction without that field; `indexed = true` registers a SQLite index for query-path filtering.

**Codegen is wired on both build paths by the scaffold.** `swift build` / `swift test` run `JsBaoCodegenPlugin` automatically. The Xcode app path (`./run-ios.sh`, archives) compiles its source list from `.pbxproj`, so the SPM plugin never fires there — `run-ios.sh` runs the codegen tool before `xcodegen generate`, writing into `Models/Generated/`. The SPM target carries `exclude: ["Models/Generated"]` so the two producers don't collide. The footgun: editing `models.toml` and then hitting **Run in Xcode directly** compiles stale `Generated/` files with no error pointing at the cause — build through `./run-ios.sh` (which regenerates first) or run the codegen step by hand.

`Models/Generated/` is **gitignored** (regenerated every build; only its `README.md` is tracked). The codegen sweep only deletes files carrying the `// Generated by swift-bao-codegen` banner, so a hand-written companion (`TodoItem+Extensions.swift`) for `Identifiable` conformance, computed helpers, camelCase aliases, or convenience inits survives every regen.

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

CRUD through the facade is local-first (applied to the document store immediately, synced in the background). Writes throw — `try record.save(in: documentId)` inserts or updates in place; `save(in:upsertOn:)` matches on a single-field unique constraint instead of `id` and returns the resolved record; `try record.delete(in: documentId)`; all three throw if the target document isn't open. Reads: `query(...)`, `queryOne`, `count`, and `aggregate` are **synchronous** local reads against the document store and span every open document by default (scope with `QueryOptions(documents: [...])`). The generated facade methods are not annotated `@MainActor` — they're commonly called from MainActor-bound SwiftUI glue (`BaoDataLoader` / your `PrimitiveAppState` subclass), but the methods themselves carry no actor isolation. `find(_ id:)` and `findAll()` are **`async throws`** — use `try await TodoItem.find(id)`. Predicate operators: equality, `$gt`/`$gte`/`$lt`/`$lte`, `$containsText`, `$or`/`$and`/`$not`.

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
| `stringset` | Collection of strings (tags) | `maxCount: 20`                 |

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
public func posts() async throws -> [Post]

// Post.generated.swift — refersTo resolves the parent record (or nil)
public func author() async throws -> Author?
```
{{/lang}}

Use these at runtime:

{{ example: documents/relationships }}

{{#lang ts}}
Relationship traversal uses the same engine as `Model.query(...)` with `include` specs — see [Loading Related Data](#loading-related-data-includes) for the lower-level query-level include syntax.
{{/lang}}

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
```

### Working with StringSets

{{ example: documents/stringset-ops }}

### Working with Dates

Dates are stored as ISO-8601 strings. Convert for comparisons:

{{ example: documents/date-handling }}

## Querying Data

{{#lang ts}}
`Model.query()` returns a `PaginatedResult`: `{ data: T[], nextCursor?, prevCursor?, hasMore }`. ALWAYS access rows through `.data`.

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

Group and calculate statistics — see [Aggregation](#aggregation) above for the compiled call. The result is a **nested object keyed by group values** (not an array):

```typescript
// Returns:
// {
//   work:     { count: 8, avg_priority: 2.5, sum_estimatedHours: 40 },
//   personal: { count: 3, avg_priority: 1.0, sum_estimatedHours:  6 },
// }
```

Multi-field `groupBy` produces deeper nesting (`result[group1][group2] = { ...ops }`). Operation result keys are `count`, `sum_<field>`, `avg_<field>`, `min_<field>`, `max_<field>`.

**StringSet facet aggregation** — grouping by a `stringset` field counts per value. When the only operation is `count`, the value collapses to a number:

```typescript
const tagCounts = await Task.aggregate({
  groupBy: ["tags"],  // "tags" is a stringset field
  operations: [{ type: "count" }],
});
// Returns: { "work": 15, "urgent": 8, "personal": 5, ... }
```

Only one stringset facet field is allowed per aggregation. To check membership of a specific value across records, use a `StringSetMembership` groupBy entry: `{ field: "tags", contains: "urgent" }`.

### useJsBaoDataLoader Pattern

The data a component renders is a plain `Model.query` (see [Read](#read-find--query--first--count) for the compiled call) that you re-run whenever the underlying records change. `useJsBaoDataLoader` is the **web template's** Vue composable that wires that re-run for you — it is framework glue around the same query, not a different API.

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
- NEVER add a watch on `loadData` results. Do processing inside `loadData`.
- NEVER rely on component remounting for route param changes. The loader only sees changes via `queryParams`.
- Gate skeleton/loading UI (e.g. `PrimitiveLoadingGate`) on `showSkeleton`, not on `documentReady` or raw load state — it handles the initial-load wait and suppresses the warm-reload flash for you.
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

The data a view renders is a plain facade query — `TodoItem.findAll()` / `TodoItem.query(...)` (see [Read](#read-find--query--first--count)) — re-run whenever the records change, which you observe with `TodoItem.subscribe` (see [Subscribe to changes](#subscribe-to-changes)). `BaoDataLoader<[T]>` is the **SwiftUI glue** (PrimitiveApp package) that wires that re-run into a view: bind it rather than subscribing to `client.events.on(...)` directly or rolling a `@Published var items` + manual `refresh()`. The loader owns its subscription lifecycle (cancelled on deinit), debounces bursts (~50ms), runs the first load immediately, and re-runs a synchronous `load` closure on every trigger.

The view below is framework glue; the only Primitive calls in it are the `findAll()` query and the `TodoItem.subscribe` trigger:

{{ example: documents/dataloader-glue }}

`.onModel(subscribe: TodoItem.subscribe)` fires on **any** add/update/delete recorded in that model's shared store — local writes and remote writes both — so `reloadNow()` after a write is unneeded; call it only when the `load` closure reads something the loader can't subscribe to (a REST resource). Other triggers (`LoaderTrigger`): `.onSync`, `.onDocumentSyncStateChanged`, `.onDocumentEvents`, `.onConnect`, `.onModelChange(_:)` for a hand-built runtime-schema `DynamicModel`, and `.custom((client, reload) -> EventSubscription?)`.

`loader.phase` is a trinary: `.loading` (first load not complete), `.empty` (first load complete, data conforms to `LoaderEmptiness` and is empty), `.loaded(Data)`. `[T]`, `String`, and `Optional` get `LoaderEmptiness` out of the box.

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

A model instance is **not** a POJO, and you cannot spread or clone it. Each declared field is a getter/setter defined on the class **prototype** (backed internally by copy-on-write change tracking over the document's Yjs state) — *not* an own enumerable property on the instance. JavaScript's spread and rest operators only copy *own enumerable properties*, so they never invoke those getters and the field data is silently dropped.

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
- `documents.createWithAlias(options)` - Create document with alias atomically (fails if alias already exists)
- `documents.getOrCreateWithAlias(options)` - Get existing document by alias, or create a new one if not found. Returns `{ documentId, created: boolean, ... }`. Use this for idempotent initialization.
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

**Reconcile server access → local refs in one pass.** "Shared-with-me" lists are *not* reactive — `me.ownedDocuments`, `me.sharedDocuments`, `collections.list`, `documents.getPermissions`, and the invitation reads are plain server queries with no change feed, so nothing fires on the user's open docs when someone shares with them. Run an explicit reconcile on the triggers you control (app launch / `connectClient`, foreground via `scenePhase` → `.active`, pull-to-refresh) that: enumerates every access channel, computes the **max** permission across channels, upserts a ref per server entity keyed by entity id, and tombstones/deletes refs whose `id` is no longer in the server set. Because refs are keyed by entity id the prune is `where !serverIds.contains(ref.id)` and a double-run can't duplicate.

> **Guard freshly-created docs from the prune.** `createDocument(...)` is local-first — it returns a `CreateDocumentResult` (metadata only; read the id from `metadata?["documentId"]?.stringValue`) immediately but commits to the server in the background, so a doc you just created is **not yet** in `me.ownedDocuments`. A reconcile firing in that window deletes the brand-new ref and the item vanishes until a later reconcile re-adds it. Track locally-created ids in a `pendingCreateIds: Set<String>` (add on create, remove once the id appears in the server set) and exclude them from the prune: `where !serverIds.contains(ref.id) && !pendingCreateIds.contains(ref.id)`.

For the reconcile reads, the "shared with me" set is the union of `me.ownedDocuments(tag:limit:)` and `me.sharedDocuments(tag:limit:)` — merge them yourself (the same entity can surface in both), then dedupe by document id.
{{/lang}}

## Sharing Documents

Documents can be shared with individual users (by userId or email), with groups, with collections, or exposed to a request-access flow for users with a link. Sharing **by email** to someone who isn't a user yet creates a *deferred grant* that resolves automatically at signup — that resolution lifecycle, app membership, invitation quotas, and the invite-accept wiring live in the [Invitations guide](AGENT_GUIDE_TO_PRIMITIVE_INVITATIONS.md).

### Sharing mental model

**Document permission levels:** `"owner"` > `"read-write"` > `"reader"`. (`"admin"` is an app-role projection, not a grantable document permission.) Effective permission = MAX(direct, group-derived, collection-derived). Granting a *lower* group permission to someone who already has a higher direct grant is a no-op for them.

| Primitive | Grants | Scope |
|-----------|--------|-------|
| `DocumentPermission` | Direct access to one document | Document |
| `DocumentGroupPermission` | Access via group membership | Document |
| `Collection` membership | Access to every document in a collection | Collection |
| `CollectionGroupPermission` | Collection access via group membership | Collection |
| `DocumentAccessRequest` | Request the owner notice you | Document |

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

Render the user's documents from two calls: `client.me.ownedDocuments()` for documents they own, and `client.me.sharedDocuments()` for documents shared directly with them (non-owner `DocumentPermission` plus pending `DocumentInvitation`s). Group- and collection-shared documents are listed through `groups.listDocuments` / `collections.listDocuments`.
{{/lang}}

{{#lang ts}}
### Using PrimitiveShareDocumentDialog

When allowing users to share documents, use `PrimitiveShareDocumentDialog`:

```vue
<PrimitiveShareDocumentDialog
  :is-open="showShareDialog"
  :document-id="currentDocumentId"
  :document-label="currentList?.title ?? 'Document'"
  :invite-url-template="`${window.location.origin}/lists`"
  @close="showShareDialog = false"
/>
```

**Critical:** The `invite-url-template` prop is REQUIRED when users send email notifications. Without it, the API returns HTTP 400. The URL should point to a page where invited users can see and accept their invitations.
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
The point-in-time checks accept a `timeoutMs` (default 5s). For a cheap synchronous local read — no round-trip — use `documents.isSynced(documentId:)`.
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

The one exception: if the caller is neither the owner nor an app owner, the platform falls back to checking every collection the document belongs to and allows the delete if **any** one collection's `document.delete` CEL rule passes. That rule defaults to `"false"` (deny) — see [Rule Sets for Collection Management](AGENT_GUIDE_TO_PRIMITIVE_USERS_AND_GROUPS.md#rule-sets-for-collection-management) — so deletion never widens past owner/app-owner unless an app explicitly configures it. Because that rule is evaluated per collection, adding a document to a collection can extend who is able to delete it; configure `document.add` and `document.delete` together with that reach in mind.

{{ example: documents/delete-document }}

### Programmatic Sharing — Full Reference

Beyond the Quick Reference and the `PrimitiveShareDocumentDialog` UI, the full programmatic surface. Inspecting a document's current members and pending invites:

{{ example: sharing/document-members }}

#### Direct shares: `updatePermissions`

The method is **`updatePermissions`** (no `setPermissions`). Two forms — single user (by id or email) and batch (any mix of userId and email):

{{#lang ts}}
```typescript
// Single user (by id or email)
await client.documents.updatePermissions(documentId, {
  email: "alice@example.com",
  permission: "read-write",
});

// Batch (any mix of userId and email)
await client.documents.updatePermissions(documentId, {
  permissions: [
    { userId: "user-abc", permission: "read-write" },
    { email: "alice@example.com", permission: "reader" },
    { email: "bob@example.com",   permission: "read-write" },
  ],
});
```
{{/lang}}

Optional fields on either form: `sendEmail`, `documentUrl`, `note`.

When `sendEmail: true`, the server delivers per-recipient emails:

- **Existing app members** receive the `document-share` template, populated with the caller-supplied `documentUrl`.
- **Non-members (deferred grants)** receive the `document-share-deferred` template, populated with an accept URL composed from `app.baseUrl` + the new `inviteToken` (shape `${app.baseUrl}/invite/accept?inviteToken=...`).

Both branches share two preconditions when `sendEmail: true`: `documentUrl` must be supplied in the request, and the app must have `baseUrl` configured (so the deferred branch can compose its accept URL). Either missing returns HTTP 400 (`"documentUrl is required when sendEmail is true"` or `"Cannot send share email: app baseUrl is not configured"`). Customize either email type with `primitive email-templates set document-share ...` or `primitive email-templates set document-share-deferred ...`.

Repeated email-based calls are idempotent: a second `updatePermissions` call for the same email updates the existing pending `DeferredDocumentPermission` in place rather than creating a duplicate row, so the latest `permission` value wins at signup-time resolution and `client.documents.listPendingInvitations(documentId)` shows one entry per pending recipient.

**There is no `permission: null` to remove.** Removal is a separate call (`removePermission` by userId or by email; `transferOwnership` to hand a document to a new owner):

{{ example: documents/manage-permissions }}

{{#lang ts}}
**Don't do this** (silent no-op for someone with a higher group permission, and there is no `null` form):

```typescript
// WRONG — there is no setPermissions
await client.documents.setPermissions(documentId, [
  { userId, permission: null },
]);

// WRONG — downgrades direct grant to "reader" but does NOT lower a higher
// group-derived permission. The user keeps read-write via the group.
await client.documents.updatePermissions(documentId, {
  userId, permission: "reader",
});
// Correct: also lower or revoke the group permission.
```
{{/lang}}
{{#lang swift}}
There is no `setPermissions`, and `updatePermissions` with a lower permission does **not** lower a higher group-derived permission — the user keeps access via the group.
{{/lang}}

Lowering a user's direct permission while they still have a higher one via a group is a no-op — the group wins (effective = MAX). To actually lower it, also lower or revoke the group permission.

#### Response shape

{{#lang ts}}
```typescript
// {
//   success: true,
//   message: "...",
//   results?: [
//     // Direct (existing user):
//     { status: "granted" | "updated", userId: "user-abc", permission: "read-write" },
//
//     // Deferred (email not yet an app user):
//     { status: "pending_signup",
//       email: "bob@example.com",
//       permission: "read-write",
//       appInvitationCreated: true,
//       invitationId: "inv-123",
//       inviteToken: "..." | null }   // combine with your accept URL
//   ]
// }
```
{{/lang}}

`results` is only present when at least one entry was deferred (i.e. the batch contained an email that didn't yet map to an app user). For all-direct grants — single-user or batch — the response is just `{ success: true, message }` with no `results` array. Branch on `status` per row when `results` is present.

#### Group sharing

The method is **`grantGroupPermission`** (no `setGroupPermission`). Member changes inside the group propagate automatically — no per-membership permission calls.

{{ example: documents/group-permission }}

#### Looking up users

`client.users.lookup(email)` reports whether an email maps to an existing app user — use it to decide whether to share by `userId` (definitive) or `email` (will defer if no app user yet). The argument is a plain string, not `{ email }`.

{{#lang ts}}
```typescript
const result = await client.users.lookup("alice@example.com");
// { exists: true, user: { userId, name, email } }
// or { exists: false }
```
{{/lang}}

#### Redeeming a deferred grant

The recipient redeems a deferred grant (after signup or in a different session) with `client.invitations.accept(inviteToken)`, using the `inviteToken` returned from the share call. Only needed for the cross-identity path — email-matched signup resolves automatically. See the [Invitations guide](AGENT_GUIDE_TO_PRIMITIVE_INVITATIONS.md#deferred-grants) for when the explicit accept call is required versus automatic resolution.

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
| `RATE_LIMITED` | Too many requests; body has `retryAfter` |
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

#### Real-time delivery

The server pushes WS frames `document:access-request-created` (to owners/admins) and `document:access-request-resolved` (to the requester). **These are not surfaced as typed `client.on(...)` events.** Poll `listAccessRequests()` on the owner side, and check `documents.get` again on the requester side after a sensible interval, or wire your own WS frame handler if you need lower latency.

### Collections

Group documents into a **collection** to share them as a unit. Permissions granted on a collection materialize onto all current and future documents in it.

**Key properties:**
- Permissions are **additive, max-wins** — a collection can only add access, never restrict it.
- A document can be in multiple collections; access from all sources combines.
- Deleting a collection revokes its permissions but never deletes the documents or any direct grants.
- Member access is O(1) regardless of collection size (uses system-managed groups internally).
- **Cascade rule:** sharing a collection automatically propagates access to every document inside it (current and future). Don't add per-document grants for documents already shared at the collection level — the collection grant is canonical. Prefer collection-level sharing over per-document sharing whenever the same set of users should see a related set of docs.

Create a collection, add/remove documents, and share it with a group or an individual user:

{{ example: documents/collection-manage }}

A collection's members + pending invites in one call:

{{ example: sharing/collection-access }}

Additional collection calls:

{{#lang ts}}
```typescript
// Optional, immutable-after-create — bind create() to a CollectionTypeConfig
// rule set and an external entity (exposed to CEL as collection.contextId).
// `initialMetadata` stamps resource-metadata categories at create time (schema-
// validated, writeRule waived, capped at 10 categories) — see the Resource
// Metadata guide's "Create-time initial metadata" section.
const collection = await client.collections.create({
  name: "Q1 Reports",
  collectionType: "class-reports",
  contextId: "math-101",
  initialMetadata: { settings: { visibility: "class-only" } },
});

// List
await client.collections.listDocuments(collection.collectionId);
// → { items: CollectionDocumentInfo[], cursor?: string }
await client.collections.listCollectionsForDocument(documentId);

await client.collections.revokeGroupPermission(collection.collectionId, "team", "engineering");
await client.collections.removeMember(collection.collectionId, targetUserId);
// Change a user's permission: call addMember again with the new level.
```
{{/lang}}

Collections accept email-based members exactly like documents and groups — a deferred grant that resolves on signup:

{{#lang ts}}
```typescript
// Add an individual member by userId
await client.collections.addMember(collectionId, {
  userId: "user-abc",
  permission: "read-write",              // "reader" or "read-write"
});

// Or by email — deferred grant resolves on signup, mirroring documents/groups
const result = await client.collections.addMember(collectionId, {
  email: "newhire@example.com",
  permission: "reader",
  sendEmail: true,                       // optional: platform sends an invite email
  collectionUrl: "https://...",          // required when sendEmail is true
  note: "Sharing the onboarding docs",
});
// result.status: "added" | "already_member" | "pending_signup"
// Pending case carries { invitationId, inviteToken, expiresAt }.

// Share with a group (fans out to every document in the collection)
await client.collections.grantGroupPermission(collectionId, {
  groupType: "team",
  groupId: "engineering",
  permission: "reader",
});

// Inspect / undo
const access = await client.collections.getAccess(collectionId);
// → { members: [...], groups: [...] }
const pending = await client.collections.listPendingInvitations(collectionId);
// → [{ email, permission, invitationId, expiresAt, ... }]
await client.collections.removeMember(collectionId, "user-abc");
await client.collections.revokeGroupPermission(collectionId, "team", "engineering");
// Change a user's permission: call addMember again with the new level.
```
{{/lang}}

The deferred-grant flow when adding a collection member by email (`collections.addMember`) mirrors the document share path: `app.baseUrl` must be configured, `sendEmail: true` requires `collectionUrl`, and `client.invitations.delete(invitationId)` cancels every pending collection add (plus any pending document shares and group adds) attached to the invitation. See the [Invitations guide](AGENT_GUIDE_TO_PRIMITIVE_INVITATIONS.md#deferred-grants) for the resolution lifecycle.

For per-context CEL rules using `collectionType` + `contextId` (and the `hasCollectionAccess` helper), see [Rule Sets for Collection Management](AGENT_GUIDE_TO_PRIMITIVE_USERS_AND_GROUPS.md#rule-sets-for-collection-management) in the Users and Groups guide.

{{#lang swift}}
`collections.addDocument`/`removeDocument` are idempotent, and the same max-wins cascade applies. The template's local app-state pattern assumes one collection per doc (`ListRef.collectionId`); for multi-membership, model a `[String]` field or query `collections.listCollectionsForDocument(documentId:)` on demand. Read collection lists with `collections.list(options:)` and members with `collections.getAccess(collectionId:)` (see [Permission / collection reads](#permission--collection-reads) below).
{{/lang}}

**CLI:**

```bash
primitive collections create "Q1 Reports" --description "Quarterly reports"
primitive collections create "Q1 Reports" --initial-metadata '{"settings":{"visibility":"class-only"}}'
primitive collections list
primitive collections docs {add|remove|list} <collection-id> [<document-id>]
primitive collections share <collection-id> --group team/engineering --permission read-write
primitive collections unshare <collection-id> --group team/engineering
primitive collections members add <collection-id> <user-id> --permission reader
primitive collections members remove <collection-id> <user-id>
primitive collections access <collection-id>
```

### Building a "Members + Pending" UI

The canonical sharing panel: people with access + people invited but not yet signed up. Use the per-resource `listPendingInvitations` endpoints — each shareable resource exposes a denormalized list so callers don't have to filter the app-level invitation list. Don't reach into `client.invitations.listDeferredGrants(...)` (admin-debug) or `client.invitations.list(...)` (app-level, admin/owner only) for product UI — the per-resource endpoints are the source. Removing yourself from a document also evicts the local copy.

{{#lang ts}}
```typescript
// Current members
const members = await client.documents.getPermissions(documentId);
// [{ userId, email, name, permission, grantedAt }, ...]   (getPermissions, not listPermissions)

// Pending email shares for one document
const docPending = await client.documents.listPendingInvitations(documentId);
// [{ email, permission, invitationId, createdAt, expiresAt, grantedBy? }]

// Pending group adds for one group
const groupPending = await client.groups.listPendingInvitations(groupType, groupId);
// [{ email, role, invitationId, deferredId, createdAt, expiresAt, addedBy? }]
// deferredId → invitations.revokeDeferredGrant(deferredId, "group") cancels it

// Pending collection adds for one collection
const colPending = await client.collections.listPendingInvitations(collectionId);
```

**Canonical "share + render" flow:**

```typescript
async function shareAndReload(documentId: string, email: string) {
  const result = await client.documents.updatePermissions(documentId, {
    email,
    permission: "read-write",
  });
  // result.results is present only when at least one row was deferred
  // (email didn't yet map to an app user). All-direct grants return just
  // { success, message } — that's success; refetch the panel.
  const [members, pending] = await Promise.all([
    client.documents.getPermissions(documentId),
    client.documents.listPendingInvitations(documentId),
  ]);
  return { members, pending };
}
```

**Cancelling:**

```typescript
// Cancel one pending document share / group add (by email):
await client.documents.removePermission(documentId, { email });
await client.groups.removeMember(groupType, groupId, { email });

// Remove someone who already has access (by userId):
await client.documents.removePermission(documentId, userId);
await client.groups.removeMember(groupType, groupId, userId);

// Nuclear: cancel the whole AppInvitation, cascading every pending document
// share and group add linked to it. Use only for "uninvite from the app
// entirely" — NOT to cancel a single share.
await client.invitations.delete(invitationId);
```
{{/lang}}

{{#lang swift}}
#### Permission / collection reads {#permission--collection-reads}

The permission and collection reads return the raw server rows — they do **no** client-side assembly, so merging and field resolution are yours to do in the sharing screens that need them:

| Read | Returns |
|---|---|
| `me.ownedDocuments(tag:limit:cursor:)` | `[DocumentInfo]` — documents the user owns |
| `me.sharedDocuments(tag:limit:cursor:)` | `SharedDocumentListResult` (`.items`, `.cursor`) — documents shared with the user |
| `collections.list(options:)` | `PaginatedResult<CollectionInfo>` |
| `collections.listDocuments(collectionId:options:)` | `PaginatedResult<CollectionDocumentInfo>` |
| `documents.getPermissions(documentId:)` | `[DocumentPermissionEntry]` — each row carries `userId`; resolve `email`/`name` yourself if you need them |
| `collections.getAccess(collectionId:)` | `CollectionAccessInfo` — collection members and their permission levels |

The "accessible documents" set is the **union** of `me.ownedDocuments` and `me.sharedDocuments`: call both and dedupe by document id (the same doc can surface in both).

For **writes**, use the typed params factories — `documents.updatePermissions(documentId:params: .email("…", permission: "read-write", sendEmail: false, documentUrl: …))` (or `.user(…)` / `.batch([…])`) and `collections.addMember(collectionId:params: .email("…", permission: .readWrite))` (or `.user(…)`). To cancel a pending email invite on a document, call `documents.removePermission(documentId:, .email("…"))`; alternatively, read the row's `deferredId` via `client.invitations.listDeferredGrants(...)` and call `client.invitations.revokeDeferredGrant(deferredId:type:)` (`.document` for a per-doc invite, `.group` for a collection one). A group's pending entries need no such detour: each `PendingGroupInvitationEntry` from `groups.listPendingInvitations(groupType:groupId:)` carries its `deferredId` directly — revoke with `type: .group`.
{{/lang}}

### Sharing Discovery Cheat Sheet

Pick the call that answers the question you're actually asking:

{{#lang ts}}
| Question | Call |
|----------|------|
| Documents the user owns | `client.me.ownedDocuments({ tag?, limit?, cursor?, returnPage? })` |
| Documents directly shared with the user (`DocumentPermission` + pending `DocumentInvitation`) | `client.me.sharedDocuments({ tag?, cursor?, limit? })` → `{ items, cursor }` |
| Pending document invitations the user can accept | `client.me.pendingDocumentInvitations()` |
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
| Documents the user owns | `client.me.ownedDocuments(cursor:limit:tag:)` → `[DocumentInfo]` (or `ownedDocumentsPage(cursor:limit:tag:)` → `DocumentListPage` for the `{ items, cursor }` envelope) |
| Documents directly shared with the user (`DocumentPermission` + pending `DocumentInvitation`) | `client.me.sharedDocuments(cursor:limit:tag:)` → `SharedDocumentListResult` (`{ items, cursor }`) |
| Pending document invitations the user can accept | `client.me.pendingDocumentInvitations()` → `[PendingDocumentInvitation]` |
| Documents inside a collection | `client.collections.listDocuments(collectionId:options:)` → `PaginatedResult<CollectionDocumentInfo>` |
| Documents shared with a group | `client.groups.listDocuments(groupType:groupId:)` → `[GroupDocumentInfo]` |
| Collections the user is a direct member of | `client.collections.list(options:)` → `PaginatedResult<CollectionInfo>` |
| Members + groups on a collection | `client.collections.getAccess(collectionId:)` |
| Group permissions on a document | `client.documents.listGroupPermissions(documentId:)` |
| Pending email invites on a document / group / collection | `client.documents.listPendingInvitations(documentId:)` / `groups.listPendingInvitations(groupType:groupId:)` / `collections.listPendingInvitations(collectionId:)` |

`options:` takes a `PaginationOptions(limit:cursor:)`; the paginated reads return `PaginatedResult` (`.data`, `.nextCursor`, `.hasMore`).

The permission / collection reads — `documents.getPermissions(_:)`, `collections.getAccess(_:)`, `collections.listPendingInvitations(_:)` — return the raw server rows ([Permission / collection reads](#permission--collection-reads) above). Permission rows carry `userId`, not `email`/`name`, so a UI that shows who has access resolves those fields itself.
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
- Assuming `me.sharedDocuments()` includes group- or collection-shared docs — it only carries direct `DocumentPermission` rows and pending `DocumentInvitation`s. Combine with `collections.list()` / `groups.listUserMemberships(...)` for a complete picture.
- Showing a "request access" button without checking the caught error's `canRequestAccess` detail.
- Calling `client.invitations.delete()` to cancel a single pending document share — it cascades to every share and group add linked to that invitation.
- Polling `client.invitations.list` / `listDeferredGrants` to populate "Members + Pending" rows — those are app-level / admin surfaces; per-resource `listPendingInvitations` is the product UI source.

**Sharing error codes** (server-emitted body codes):

{{#lang ts}}
The client throws a typed `JsBaoApiError` with `.status`, `.code`, and `.body` (the parsed error body) — read the code and details straight off the caught error, no manual message-parsing needed.
{{/lang}}
{{#lang swift}}
The client throws a typed `JsBaoError` with `.code` and `.details` — read the code and details straight off the caught error, no manual message-parsing needed.
{{/lang}}

| Code | Endpoint | Meaning |
|------|----------|---------|
| `DOC_ACCESS_DENIED` (with `details.canRequestAccess`) | `documents.get` | 403; check the hint to decide between request-access UI and hard deny |
| `ALREADY_HAS_ACCESS` | `requestAccess` | Caller already has a direct or group permission |
| `RATE_LIMITED` | `requestAccess` | Body has `retryAfter` (seconds) |
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

The `primitive` CLI provides export and import commands for migrating or backing up document data. These are admin operations, not used in application code.

```bash
# Export a single document (Yjs state, blobs, permissions, aliases)
primitive documents export <document-id> --output ./primitive-export

# Export all documents for a user
primitive documents export-all --user-id <user-id> --output ./primitive-export
primitive documents export-all --user-id <user-id> --owned-only  # Only owned documents

# Import from an export directory
primitive documents import <path>                         # Default: skip existing aliases
primitive documents import <path> --aliases overwrite     # Overwrite existing user-scoped aliases
primitive documents import <path> --aliases skip          # Keep existing aliases (default)
primitive documents import <path> --overwrite --dry-run   # Preview without changes
```

Export creates a directory per document containing `metadata.json`, `document.yjs` (Yjs state), `permissions.json` (for reference), and `blobs/` (attachments). Permissions are **not** restored on import — the importing admin becomes the new owner and manages sharing in the target app. Document IDs are preserved across import. User-scoped aliases can be restored with `--aliases overwrite` or kept as-is with `--aliases skip`.

{{#lang ts}}
## Common Errors

| Symptom                                         | Cause                                                                    | Fix                                                                                                  |
| ----------------------------------------------- | ------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------- |
| Need document from item                         | N/A                                                                      | Use `item.getDocumentId()`                                                                           |
| Data doesn't update when route param changes    | Vue reuses components; `useJsBaoDataLoader` doesn't see the change       | Add the route param to `queryParams` in the data loader, OR use `:key="routeParam"` on the component |
| Store loads once but never reacts to changes    | `useJsBaoDataLoader` called outside a component — its `onMounted` subscriptions never register | In a Pinia store / non-component context call `Model.subscribe(reload)` directly in `setup()`. See [Subscribing Outside a Component](#subscribing-outside-a-component) |
| Spread/clone of a model is empty or missing fields | Fields are prototype getters, not own properties, so `{ ...model }` / `{ id, ...rest }` copy nothing | Read fields explicitly: `{ id: model.id, title: model.title }`. See [Model Instances Are Not Plain Objects](#model-instances-are-not-plain-objects) |
| Query `field: false` misses items               | Items with `field: undefined` don't match `field: false`                 | Use a default value in schema, OR filter in JavaScript with `item.field ?? false`                    |
| Document created but not in sidebar/list        | Created via `documents.create()` directly without updating tracked state | Use the demo `jsBaoDocumentsStore.createDocument()` (or your own tracker) so reactive lists update   |
| HTTP 400 when sharing with email                | Missing `documentUrl` in invitation                                       | Pass `invite-url-template` prop to `PrimitiveShareDocumentDialog`                                    |
| New document not queryable immediately          | Document not opened after creation                                        | After `documents.create()`, call `documents.open(metadata.documentId)` before querying              |
| `setPermissions is not a function`              | Method doesn't exist                                                      | Use `updatePermissions(documentId, { userId, permission })`                                          |
| `setGroupPermission is not a function`          | Method doesn't exist                                                      | Use `grantGroupPermission(documentId, { groupType, groupId, permission })`                           |
| "Model not properly initialized" on save/query  | Schema not registered — `*.generated.ts` or `index.ts` is out of sync with `models.toml` | Re-run `npx js-bao-codegen-v2`; never manually attach a schema or edit generated files               |
| `upsertByUnique`: "constraint not found"        | Passed field array instead of constraint name                             | Pass the named constraint string (e.g. `"users_email_unique"`)                                       |
| `upsertByUnique`: "targetDocument is required"  | Creating a new record without specifying its document                     | Pass `{ targetDocument: docId }` as the 4th argument                                                 |
| `query()` result missing `.map`/`.filter`       | Forgot result is a `PaginatedResult`                                      | Use `result.data` (also has `.nextCursor`, `.hasMore`)                                               |
{{/lang}}
