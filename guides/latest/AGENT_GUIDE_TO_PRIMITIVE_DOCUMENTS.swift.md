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
   Add the model to `models.toml`; codegen runs on every build (regenerate by hand with `swift build` or the `run-ios.sh` codegen step if you edited the schema between builds).

5. **Prefer query filtering over in-memory filtering.** Push filter conditions into the query itself rather than fetching everything and filtering on the client.

6. **Load data at the screen level, not in leaf components.** Pass data and readiness into sub-components as props.

7. **Understand the root document's role and limitations.** The root document is a special per-user document that is automatically created and opened. It can never be shared or deleted, and there is exactly one per user. It's a natural home for user preferences and settings that should be available whenever the user signs in. The root document can hold any model, but store most application data in regular documents for greater flexibility (sharing, collaboration, multiple documents). Use the "single document" pattern with aliases for personal apps, or the "one document at a time" pattern for multi-workspace apps.

## Document Lifecycle

### 1. Open Documents Before Querying

Documents must be opened before querying or modifying data within them.

```swift
  _ = try await client.documents.open(documentId)
  let result = try Task.query([:], options: QueryOptions(documents: [documentId]))
```

Documents are ready to be queried once the `.open()` call finishes. Applications should wait for all required documents to be opened and show a loading state until then, then track document-specific readiness explicitly. `open()` is idempotent — calling it on an already-open document is a no-op. Handle open failures explicitly: surface an error or redirect; don't silently continue.

Open only the documents you need to query or want real-time updates from — every open document syncs continuously. Don't open documents you don't need, and close ones you're done with (see [Closing Documents](#closing-documents)).


The document-open lifecycle is owned by `PrimitiveAppState` (see [The `PrimitiveAppState` document lifecycle](#the-primitiveappstate-document-lifecycle) below). Subclass it and drive doc setup from `connectClient()`; reads and writes go through the codegen'd model facade (`TaskRecord.query(...)`, `record.save(in:)`), so there is no per-document binding step. For session-scoped documents (small bounded set), open once at launch; for transient per-item docs, open on view appear and close on disappear. Stray opens widen query scope — facade queries span every open document by default.

### 2. Finding documents a user can access

There is **no single "my documents" list**. A user reaches documents through **four distinct paths** — query each separately and combine in the UI. Do NOT try to unify them into one server-side list.

**a. Documents they own** (`ownedDocuments` — created, or ownership transferred):

```swift
  // A typed `[DocumentInfo]` — each row carries title, permission, tags, …
  let owned = try await client.me.ownedDocuments(tag: "channel")

  for doc in owned {
    print(doc.title, doc.permission)
  }
```

**b. Documents shared directly with them** (`sharedDocuments` — non-owner `DocumentPermission` rows + pending `DocumentInvitation`s; group/collection shares do NOT appear here):

```swift
  let page = try await client.me.sharedDocuments(limit: 50, tag: "channel")

  for share in page.items {
    // Each row nests the base document fields (title, createdAt, …) under
    // `.document`, alongside the share extras (grantedBy, source, invitationId).
    print(share.document.title, share.document.permission, share.grantedBy)
  }

  // `cursor` is a raw-JSON pagination cursor — pass it back for the next page.
  if let cursor = page.cursor {
    _ = try await client.me.sharedDocuments(cursor: cursor)
  }
```

**c. Documents shared via a group** (`groups.listDocuments`):

```swift
  let documents = try await client.groups.listDocuments(groupType: "team", groupId: "engineering")
```

**d. Documents shared via a collection** (`collections.listDocuments`):

```swift
  let page = try await client.collections.listDocuments(
    collectionId: collectionId,
    options: PaginationOptions(limit: 50)
  )
  let items = page.items
```

Swift can't inherit a struct, so each row from `sharedDocuments` is a `SharedDocument` that holds the base fields under `.document` (a `DocumentInfo`, including `permission` — never `.owner`) alongside the share-only extras `grantedBy`, `source` (`"permission"` | `"invitation"`), and `invitationId` (invitation rows only). Group and collection rows have their own shapes — `groups.listDocuments` returns `GroupDocumentInfo`, and `collections.listDocuments` returns flat `CollectionDocumentInfo` (`documentId`, `title`, `permission`, `addedBy`, `addedAt`, …).


## Core data operations

Every example below is compiled against the real client as part of the docs build. The [Querying Data](#querying-data) and [Saving Data](#saving-data) sections below go deeper on projections, includes, and save options.

The generated model statics route through the process-wide default client — call `JsBaoClient.configureDefault(client)` once at startup, before the first read or write. `query`, `queryOne`, `count`, `aggregate`, `find`, and `findAll` are synchronous `throws` against the local document store. All reads span every open document by default (scope with `QueryOptions(documents: [docId])`); writes target one document — `save(in:)` inserts or updates in place and throws (it validates the write and requires the document to be open), `delete(in:)` throws only if the document isn't open.

### Create

```swift
  let task = try Task(
    id: UUID().uuidString,
    title: "Review pull request",
    priority: 2,
    dueDate: ISO8601DateFormatter().string(from: Date())
  ).save(in: documentId)
  _ = task
```

### Read (find / query / first / count)

```swift
  // Find one by id
  let task = try Task.find("task-id")

  // Query with filters
  let urgent = try Task.query(["priority": ["$gte": 2], "completed": false])

  // First match (with a sort)
  let topTask = try Task.query(
    ["completed": false],
    options: QueryOptions(sort: ["priority": -1])
  ).first

  // Count
  let remaining = try Task.count(["completed": false])
```

### Update

```swift
  if var task = try Task.find(taskId) {
    task.completed = true
    try task.save(in: documentId)
  }
```

### Delete

```swift
  if let task = try Task.find(taskId) {
    try task.delete(in: documentId)
  }
```

### Upsert by natural key

```swift
  let user = AppUser(
    id: UUID().uuidString,
    email: "alice@example.com",
    name: "Alice"
  )
  // "email" must have a single-field unique constraint in models.toml.
  // On merge, the returned record carries the existing record's id.
  let resolved = try user.save(in: documentId, upsertOn: "email")
```

### Upsert by named unique constraint

```swift
  let category = Category(
    id: UUID().uuidString,
    name: "Work",
    parentId: "root",
    color: "blue"
  )
  // "name_parentId" is the named constraint declared in models.toml. `mode`
  // defaults to .either (create-or-update); pass .mustExist / .mustNotExist to
  // require a single path.
  let resolved = try category.upsertByUnique("name_parentId", in: documentId)
```

### Logical query operators

```swift
  let result = try Task.query([
    "$or": [
      ["priority": 3],
      ["dueDate": ["$lt": "2026-06-02T00:00:00Z"]],
    ],
  ])
```

### Sort + cursor pagination

```swift
  let page1 = try Task.queryPaged(
    ["completed": false],
    options: QueryOptions(sortOrder: [("priority", -1)], limit: 20)
  )

  if let cursor = page1.nextCursor {
    let page2 = try Task.queryPaged(
      ["completed": false],
      options: QueryOptions(sortOrder: [("priority", -1)], limit: 20, cursor: cursor)
    )
    _ = page2
  }
```

### Aggregation

```swift
  let stats = try Task.aggregate(AggregateOptions(
    groupBy: ["category"],
    operations: [
      AggregateOperation(type: .count),
      AggregateOperation(type: .avg, field: "priority"),
      AggregateOperation(type: .sum, field: "estimatedHours"),
    ],
    filter: ["completed": false],
    sort: AggregateSort(field: "count", direction: -1),
    limit: 10
  ))

  // Grouping by a stringset field counts per member value (facet):
  let tagCounts = try Task.aggregate(AggregateOptions(
    groupBy: ["tags"],
    operations: [AggregateOperation(type: .count)]
  ))

  // Group by whether the set contains a value (membership) — rows carry
  // a "has_tags_urgent" key of "true" / "false":
  let urgentSplit = try Task.aggregate(AggregateOptions(
    groupBy: [.stringSetMembership(field: "tags", contains: "urgent")],
    operations: [AggregateOperation(type: .count)]
  ))
```

Grouping by a `stringset` field counts per member value (facet); a membership `groupBy` entry groups by whether the set contains one specific value. Only one stringset facet field is allowed per aggregation, and a facet can't be mixed with other `groupBy` entries — unsupported mixes degrade (the facet is dropped or the result is empty) rather than throw.

### Subscribe to changes

```swift
  let unsubscribe = Task.subscribe {
    // re-query and update your UI
  }

  // later, when you no longer need updates:
  unsubscribe()
```

### Resolve-or-create a singleton document

```swift
  let result = try await client.documents.getOrCreateWithAlias(
    options: GetOrCreateWithAliasOptions(
      alias: AliasRef(scope: .user, aliasKey: "default-doc"),
      title: "My Data"
    )
  )
  _ = try await client.documents.open(result.documentId)
```

### Share a document (user / email / group)

```swift
  // By user ID
  _ = try await client.documents.updatePermissions(
    documentId: documentId,
    params: .user("user-abc", permission: "read-write")
  )

  // By email — works whether or not the recipient is a member yet
  _ = try await client.documents.updatePermissions(
    documentId: documentId,
    params: .email("colleague@example.com", permission: "read-write")
  )

  // With a group
  _ = try await client.documents.grantGroupPermission(
    documentId: documentId,
    params: GrantGroupPermissionParams(groupType: "team", groupId: "engineering", permission: "read-write")
  )
```

### Link access ("anyone with the link")

```swift
  // Any signed-in app user who has the document ID can now read it, with no
  // explicit grant. The level is a floor: a user who already has a higher
  // grant keeps it.
  _ = try await client.documents.setLinkAccess(documentId: documentId, level: .reader)

  // Read the current state — the level, plus who last changed it and when.
  let state = try await client.documents.getLinkAccess(documentId: documentId)
  print(state.linkAccess as Any) // .reader | .readWrite | nil

  // Turn it off. Anyone relying on the link loses access immediately;
  // explicit grants are untouched.
  _ = try await client.documents.clearLinkAccess(documentId: documentId)
```

`setLinkAccess(documentId, level)` sets a permission **floor**: any signed-in app user who has the document ID resolves to at least `level` (`"reader"` | `"read-write"`) with no explicit grant. Effective access is `max(direct grant, group grants, link floor)` — the floor only ever raises a caller's level, never lowers an explicit grant. It counts for reading/writing document data but NOT for sharing or management: a link-only caller can't reshare, change permissions, or delete. Requires document owner, app owner, or read-write editor; root documents can't be link-shared (throws).

- A document reached only through the floor reports `accessSource: "link"` on `documents.get(documentId)` and does NOT appear in `me.sharedDocuments()` — track it client-side if you want a "recently opened" affordance.
- `getLinkAccess(documentId)` returns `{ documentId, linkAccess, linkAccessUpdatedBy?, linkAccessUpdatedAt? }` (`linkAccess` is `null`/`nil` when off) and is authorized for any caller who can read OR manage the document, so an app owner can inspect it without a content grant.
- `clearLinkAccess(documentId)` — or lowering the level — evicts connected link-only viewers immediately; explicit grants are untouched.

### Update thumbnail / metadata

```swift
  _ = try await client.documents.update(
    documentId: documentId,
    data: UpdateDocumentData(
      title: "Q2 Planning",
      thumbnailBlobId: .value(blobId),                              // a blob you uploaded
      metadata: ["color": "blue", "tags": ["plan", "q2"]]          // ≤4KB JSON, replace semantics
    )
  )
```

## Common Document Usage Patterns


### Pattern 1: Single Document (Personal Apps)

**Best for:** Personal tools, single-user apps, no sharing needed

Each user gets exactly one document that holds all their data. The document is opened on app load / user sign-in. No document management UI is needed.

**Examples:** Personal task manager, habit tracker, journal app, budgeting tool

**User experience:** Users sign in and immediately see their data. No concept of "documents" is exposed in the UI.

**Implementation** — resolve-or-create the per-user document on app init, then open it (see [Resolve-or-create a singleton document](#resolve-or-create-a-singleton-document) above for the compiled call). `result.created === true` if a new document was just created.

### Pattern 2: One Document at a Time (Workspaces)

**Best for:** Apps where users create discrete projects/workspaces they might share independently — accounting (per company), project management (per project), shared shopping lists (per household).

Users have multiple documents but work in one at a time, switching between them. Track the current document and call `open()` on the chosen one.


List with `me.ownedDocuments()` and `open()` the selected document; create a new workspace document with `create()` and open it:

```swift
  let result = try await client.documents.create(
    options: CreateDocumentOptions(title: "New Project", tags: ["workspace"])
  )
  // The generated id lives inside the returned metadata.
  let documentId = result.metadata?["documentId"]?.stringValue
  // The new document is already open — query or save into it right away.
```

### Pattern 3: Multiple Documents

**Best for:** Apps that query across many documents, each with its own sharing context — chat (per channel), multi-tenant dashboards, collaborative workspaces with distinct collections.

All documents that need live updates or cross-document queries must be open. Tag documents so you can fetch a set with a tag-filtered `me.ownedDocuments` (and `me.sharedDocuments` if the user can also be a non-owner), open each, and track per-document readiness yourself. For collective sharing of multiple documents as a unit, prefer the server-side Collections API (`client.collections.*` — see [Collections](#collections) below) over local tracking.

```swift
  // Open every document with a given tag
  let channels = try await client.me.ownedDocuments(tag: "channel")
  for channel in channels {
    _ = try await client.documents.open(channel.documentId)
  }

  // Query runs across all open documents by default
  let messages = try Message.query([:])
```


The most common multi-doc shape is one ambient library/index document plus N per-item documents (each shareable independently); see [Index doc + per-item docs](#index-doc--per-item-docs-keying-tombstones-reconcile) below for keying, tombstones, and the reconcile pass. Open auxiliary docs with `appState.openAuxiliaryDoc(_:)` and close them with `appState.closeAuxiliaryDoc(_:)`.

### The `PrimitiveAppState` document lifecycle

The neutral lifecycle is: resolve-or-create the doc (`client.documents.getOrCreateWithAlias` — see [Resolve-or-create a singleton document](#resolve-or-create-a-singleton-document)), open it, then read through the codegen'd facade scoped to it (see [Read](#read-find--query--first--count)). Models need no per-document binding — the facade statics read from every open document, backed by the process-wide default client (`JsBaoClient.configureDefault`).

`PrimitiveAppState` is the **SwiftUI app-state glue** (PrimitiveApp package) that owns that lifecycle: subclass it for app-specific state and override `connectClient()` to drive doc setup after connect (`configureDefault` is wired for you during `initialize()`). The subclass below is framework glue; the Primitive call in it is the `getOrCreateWithAlias` resolve-or-create:

```swift
@MainActor
final class MyAppState: PrimitiveAppState {
    // connectClient is `open` — call super first (it connects and fetches
    // /me + the document list), then run app-specific setup. Don't reach
    // for a Combine sink on `$isConnected`; this override is the path.
    override func connectClient() async {
        await super.connectClient()
        // Optional but recommended: pre-register models so every open
        // document is mirrored into the client's shared store immediately
        // (the facade also lazily registers on first read).
        client?.registerModels([TodoItem.self])
        await openLibraryDoc()
    }

    private func openLibraryDoc() async {
        guard let client else { return }
        do {
            // Atomic resolve-or-create. Don't split into aliases.resolve +
            // createWithAlias — that has a TOCTOU window where two clients
            // onboarding at once both create and one doc is lost.
            let result = try await client.documents.getOrCreateWithAlias(
                alias: DocumentAlias(scope: .user, aliasKey: "library"),
                title: "Library"
            )
            // selectDocumentAwaiting opens the doc and routes the base
            // class's sync hooks at it; facade reads see it immediately.
            await selectDocumentAwaiting(result.documentId)
        } catch {
            errorMessage = "Failed to open library: \(error.localizedDescription)"
        }
    }
}
```

For per-document setup beyond models, override the `onDocumentOpened(doc:documentId:)` hook — the base class opens the doc once and hands you the live `YDocument`, so don't call `openDocument(...)` again just to get one.

For a **fresh doc you'll write immediately**, use `client.createDocument(options:)` — it returns a `CreateDocumentResult` with `metadata: JSONValue?`; extract `metadata?["documentId"]?.stringValue` to get the new document's id. The new document is already open and writable, so write through it directly rather than reopening. If you do reopen with `waitForLoad: .network`, Swift retries sync availability until the background create commit lands; healthy cases resolve after the handshake, with the fallback bounded by `availabilityWaitMs` (default 30s).

**Multi-doc apps (one ambient library doc + N per-item docs).** `selectDocumentAwaiting(_:)` is the *single*-selected-doc lifecycle — it closes the previously selected doc first, so using it for a per-item detail view closes your library/index doc. For one ambient doc plus transient detail docs, use `appState.openAuxiliaryDoc(_:)` from the detail view's `.task` and `appState.closeAuxiliaryDoc(_:)` from `.onDisappear`. These register the doc for sync, but they don't touch `selectedDocId` or fire `onDocumentOpened`. Once open, read the doc's records through the facade scoped to that document — the same scoped query as [Open Documents Before Querying](#1-open-documents-before-querying). The view is framework glue around `openAuxiliaryDoc` + that scoped query:

```swift
struct ItemDetailView: View {
    let documentId: String
    @EnvironmentObject var appState: MyAppState
    @State private var todos: [TodoItem]?

    var body: some View {
        Group {
            if let todos { /* render */ } else { ProgressView() }
        }
        .task {
            _ = try? await appState.openAuxiliaryDoc(documentId)
            todos = try? TodoItem.query([:], options: QueryOptions(documents: [documentId]))
        }
        .onDisappear { Task { await appState.closeAuxiliaryDoc(documentId) } }
    }
}
```

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

```swift
  // Filter the user's owned documents by tag
  let todoLists = try await client.me.ownedDocuments(tag: "todolist")

  // Add a tag to an existing document
  _ = try await client.documents.addTag(documentId: documentId, tag: "archived")

  // Remove a tag from a document
  _ = try await client.documents.removeTag(documentId: documentId, tag: "archived")
```

You can also create a tagged document and filter locally:

```swift
  let result = try await client.documents.create(
    options: CreateDocumentOptions(title: "My List", tags: ["todolist"])
  )

  // Filter locally
  let owned = try await client.me.ownedDocuments()
  let todoLists = owned.filter { $0.tags?.contains("todolist") == true }
```

## Defining Models

Models are declared in a TOML schema and the client model types are generated from it by codegen. The field types and options are the same across platforms; the schema file, codegen command, and generated artifacts differ.


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

CRUD through the facade is local-first (applied to the document store immediately, synced in the background). Writes throw — `try record.save(in: documentId)` inserts or updates in place; `save(in:upsertOn:)` matches on a single-field unique constraint instead of `id` and returns the resolved record; `try record.delete(in: documentId)`; all three throw if the target document isn't open. Reads: `query(...)`, `queryOne`, `count`, `aggregate`, `find(_ id:)`, and `findAll()` are **synchronous** local reads against the document store and span every open document by default (scope with `QueryOptions(documents: [...])`). The generated facade methods are not annotated `@MainActor` — they're commonly called from MainActor-bound SwiftUI glue (`BaoDataLoader` / your `PrimitiveAppState` subclass), but the methods themselves carry no actor isolation. Predicate operators: equality, `$gt`/`$gte`/`$lt`/`$lte`, `$containsText`, `$or`/`$and`/`$not`.

> **SourceKit footgun, first time only.** Editing the companion before codegen has ever run shows a red `No such module 'PrimitiveApp'` underline. The real cause is that the generated type doesn't exist yet — run `swift build` once (or the `run-ios.sh` codegen step) and it clears. Only the very first scaffold hits this.

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

```swift
// Author.generated.swift — hasMany resolves a plain array, ordered per the TOML
public func posts() throws -> [Post]

// Post.generated.swift — refersTo resolves the parent record (or nil)
public func author() throws -> Author?
```

Use these at runtime:

```swift
  guard let author = try Author.find(authorId) else { return }

  // hasMany: author.posts() returns a plain array, ordered per the relationship
  let posts = try author.posts()
  guard let firstPost = posts.first else { return }

  // refersTo: post.author() returns the parent record (or nil)
  let backRef = try firstPost.author()
```


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

Traversal accepts an optional page size and cursor, paging the join leg the same way a query pages rows:

```swift
  guard let post = try Post.find(postId) else { return }

  // Every tag linked to this post, ordered by the join model.
  let allTags = try post.tags()

  // Or page the join leg with a cursor.
  let page1 = try post.tags(limit: 20)
  let firstTag = page1.data.first
  if let cursor = page1.nextCursor {
    let page2 = try post.tags(limit: 20, afterCursor: cursor)
    _ = page2
  }
```

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

```swift
  if var task = try Task.find(taskId) {
    // Add/remove tags
    task.tags?.insert("urgent")
    task.tags?.remove("low-priority")

    // Check membership
    if task.tags?.contains("urgent") == true {
      // ...
    }

    try task.save(in: documentId)
  }
```

### Working with Dates

Dates are stored as ISO-8601 strings. Convert for comparisons:

```swift
  let now = ISO8601DateFormatter().string(from: Date())

  // Store
  if var task = try Task.find(taskId) {
    task.dueDate = now
    try task.save(in: documentId)

    // Compare
    if let dueDate = task.dueDate, dueDate < now {
      // overdue
    }
  }

  // Query with date comparison
  let overdue = try Task.query(["dueDate": ["$lt": now]])
```

## Querying Data


### Loading Related Data (Includes)

Pass `include` in a query to batch-load related records alongside the rows, instead of following each relationship one row at a time. A `refersTo` relationship attaches its single parent; a `hasMany` attaches the matching children.

```swift
  // refersTo — each Post's Author (the `authorId` FK lives on Post):
  let posts = try Post.query(include: [Post.includeAuthor()])
  for post in posts {
    let author = post.relatedAuthor          // Author?
    print(post.title, author?.name ?? "—")
  }

  // hasMany — every Post that points back at each Author, newest first:
  let authors = try Author.query(include: [Author.includePosts(limit: 10)])
  for author in authors {
    let authored = author.relatedPosts       // [Post]
    print(author.name, authored.count)
  }
```


### View-data binding with `BaoDataLoader`

The data a view renders is a plain facade query — `TodoItem.findAll()` / `TodoItem.query(...)` (see [Read](#read-find--query--first--count)) — re-run whenever the records change, which you observe with `TodoItem.subscribe` (see [Subscribe to changes](#subscribe-to-changes)). `BaoDataLoader<[T]>` is the **SwiftUI glue** (PrimitiveApp package) that wires that re-run into a view: bind it rather than subscribing to `client.events.on(...)` directly or rolling a `@Published var items` + manual `refresh()`. The loader owns its subscription lifecycle (cancelled on deinit), debounces bursts (~50ms), runs the first load immediately, and re-runs a synchronous `load` closure on every trigger.

The view below is framework glue; the only Primitive calls in it are the `findAll()` query and the `TodoItem.subscribe` trigger:

```swift
struct TodoListView: View {
    @EnvironmentObject var appState: MyAppState
    @StateObject private var loader = BaoDataLoader<[TodoItem]>()

    var body: some View {
        Group {
            // Render through `loader.phase`, not `loader.data ?? []`.
            // `?? []` collapses "not yet loaded" with "loaded, empty",
            // flashing the empty state for ~50ms on every appearance.
            switch loader.phase {
            case .loading:           ProgressView()
            case .empty:             Text("No todos yet")
            case .loaded(let todos): List(todos) { /* row */ }
            }
        }
        // Bind once, from a plain `.task`. Don't conditionally bind on
        // doc readiness — set `loader.documentReady` instead: the loader
        // fires its initial load when it flips to true, and resets
        // `initialDataLoaded` if the doc closes.
        .task {
            loader.documentReady = appState.selectedDocId != nil
            loader.bind(
                client: appState.client,
                subscribeTo: [.onModel(subscribe: TodoItem.subscribe)]
            ) { _ in
                TodoItem.findAll().sorted { $0.sortOrder < $1.sortOrder }
            }
        }
        .onChange(of: appState.selectedDocId) { _, id in
            loader.documentReady = id != nil
        }
    }
}
```

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

## Saving Data

Writes go through record instances and are local-first — applied to the document store immediately and synced in the background (see [`models.toml` + codegen + the model facade](#modelstoml--codegen--the-model-facade) for the save/delete contract). `try record.save(in: documentId)` uses merge semantics: `nil` optional fields are not written, so fields you didn't set are preserved on update.


## Design Patterns

### Singleton Model per Document (Avoiding ID Confusion)

Create a singleton model per document for metadata. Child models reference by model ID, not document ID. Declare both models in the schema:


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



### Handling Invitations

**Auto-accept vs Manual:**
- `autoAcceptInvites: true` - Invitations are automatically accepted when the document tag matches a registered collection. Documents appear immediately in the user's list.
- `autoAcceptInvites: false` - Users must manually accept invitations on a management page. Provides more control but requires UI for viewing/accepting invitations.

When a user accepts an invitation, you typically want to navigate them to the content. Since routes use model IDs (not document IDs), query for the model in the newly accessible document first, then route to it.


### Closing Documents

When your app opens many documents over a session (e.g., viewing individual items that each live in their own document), close documents you're no longer using to avoid accumulating sync connections:

```swift
  // Close and stop syncing
  await client.documents.close(documentId)

  // Close and remove the local cached copy. `close` returns a
  // CloseDocumentResult — inspect `.evicted` when eviction matters.
  let result = await client.documents.close(
    documentId,
    options: CloseDocumentOptions(evictLocal: true)
  )
  if !result.evicted {
    // Server was not fully in sync — the local copy was retained.
  }
```

When `evictLocal: true` is passed, the client performs a state vector check against the server before removing local data. If the server hasn't received all local writes (e.g. due to a brief network interruption), eviction is skipped and `evicted: false` is returned. This prevents data loss during WebSocket instability.

### Sync Verification

Use these methods to confirm the server has received your writes before taking irreversible actions (e.g., logging out, clearing local storage):

```swift
  // Point-in-time checks
  let hasAllWrites = await client.documents.includesWrites(documentId: documentId)
  let fullyInSync = await client.documents.inSync(documentId: documentId)

  // Polling helpers: wait until confirmed
  _ = await client.documents.waitForWriteConfirmation(documentId: documentId)
  try await client.documents.waitForInSync(documentId: documentId)
```

The point-in-time checks return `false` if the client is disconnected or the check times out. The `waitFor*` polling helpers exist on both `documents.*` and the client root: `waitForWriteConfirmation` returns true on success / false on timeout, and `waitForInSync` throws on timeout.

The point-in-time checks accept a `timeoutMs` (default 5s). For a cheap synchronous local read — no round-trip — use `documents.isSynced(documentId:)`.

### Updating Document Metadata

Update a document's title, thumbnail, and presentation metadata — see [Update thumbnail / metadata](#update-thumbnail--metadata) above for the compiled call. Each field is optional; omit one to leave it unchanged.

`thumbnailBlobId` and `metadata` are clearable: pass `thumbnailBlobId: .clear` (an `Updatable<String>`) and `metadata: .null` (a `JSONValue`) to null them out; passing `nil` or omitting the argument leaves the field unchanged.

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

```swift
  // Must be closed first
  try await client.documents.delete(documentId: documentId)

  // Force-close before deleting
  try await client.documents.delete(
    documentId: documentId,
    options: DeleteDocumentOptions(forceCloseIfOpen: true)
  )
```

### Programmatic Sharing — Full Reference

Beyond the Quick Reference and the `PrimitiveShareDocumentDialog` UI, the full programmatic surface. Inspecting a document's current members and pending invites:

```swift
  // Current members (accepted permission grants)
  let members = try await client.documents.getPermissions(documentId: documentId)

  // Pending email invites on this document
  let pending = try await client.documents.listPendingInvitations(documentId: documentId)
```

#### Direct shares: `updatePermissions`

The method is **`updatePermissions`** (no `setPermissions`). Two forms — single user (by id or email) and batch (any mix of userId and email):


Optional fields on either form: `sendEmail`, `documentUrl`, `note`.

When `sendEmail: true`, the server delivers per-recipient emails:

- **Existing app members** receive the `document-share` template, populated with the caller-supplied `documentUrl`.
- **Non-members (deferred grants)** receive the `document-share-deferred` template, populated with an accept URL composed from `app.baseUrl` + the new `inviteToken` (shape `${app.baseUrl}/invite/accept?inviteToken=...`).

Both branches share two preconditions when `sendEmail: true`: `documentUrl` must be supplied in the request, and the app must have `baseUrl` configured (so the deferred branch can compose its accept URL). Either missing returns HTTP 400 (`"documentUrl is required when sendEmail is true"` or `"Cannot send share email: app baseUrl is not configured"`). Customize either email type with `primitive email-templates set document-share ...` or `primitive email-templates set document-share-deferred ...`.

Repeated email-based calls are idempotent: a second `updatePermissions` call for the same email updates the existing pending `DeferredDocumentPermission` in place rather than creating a duplicate row, so the latest `permission` value wins at signup-time resolution and `client.documents.listPendingInvitations(documentId)` shows one entry per pending recipient.

**There is no `permission: null` to remove.** Removal is a separate call (`removePermission` by userId or by email; `transferOwnership` to hand a document to a new owner):

```swift
  // Remove a current member by userId (a bare string literal also works):
  try await client.documents.removePermission(documentId: documentId, .userId("user-abc"))

  // Cancel a pending email-based invite, OR remove a current member matched by email:
  try await client.documents.removePermission(documentId: documentId, .email("alice@example.com"))

  try await client.documents.transferOwnership(documentId: documentId, newOwnerId: newOwnerId)
```

There is no `setPermissions`, and `updatePermissions` with a lower permission does **not** lower a higher group-derived permission — the user keeps access via the group.

Lowering a user's direct permission while they still have a higher one via a group is a no-op — the group wins (effective = MAX). To actually lower it, also lower or revoke the group permission.

#### Response shape


`results` is only present when at least one entry was deferred (i.e. the batch contained an email that didn't yet map to an app user). For all-direct grants — single-user or batch — the response is just `{ success: true, message }` with no `results` array. Branch on `status` per row when `results` is present.

#### Group sharing

The method is **`grantGroupPermission`** (no `setGroupPermission`). Member changes inside the group propagate automatically — no per-membership permission calls.

```swift
  _ = try await client.documents.grantGroupPermission(
    documentId: documentId,
    params: GrantGroupPermissionParams(groupType: "team", groupId: "engineering", permission: "read-write")
  )

  // Listing / revoking
  _ = try await client.documents.listGroupPermissions(documentId: documentId)
  _ = try await client.documents.revokeGroupPermission(documentId: documentId, groupType: "team", groupId: "engineering")
```

#### Looking up users

`client.users.lookup(email)` reports whether an email maps to an existing app user — use it to decide whether to share by `userId` (definitive) or `email` (will defer if no app user yet). The argument is a plain string, not `{ email }`.


#### Redeeming a deferred grant

The recipient redeems a deferred grant (after signup or in a different session) with `client.invitations.accept(inviteToken)`, using the `inviteToken` returned from the share call. Only needed for the cross-identity path — email-matched signup resolves automatically. See the [Invitations guide](AGENT_GUIDE_TO_PRIMITIVE_INVITATIONS.md#deferred-grants) for when the explicit accept call is required versus automatic resolution.

### Document Access Requests

The owner UI when a non-member hits a 403 they're allowed to escalate. The end-to-end flow (a user with a link requests access; an owner lists and approves) is compiled here:

```swift
  // A user with the link requests access
  _ = try await client.documents.requestAccess(
    documentId: documentId,
    options: RequestAccessOptions(permission: .readWrite, message: "Please add me to this doc")
  )

  // An owner lists pending requests and approves one
  let requests = try await client.documents.listAccessRequests(documentId: documentId)
  _ = try await client.documents.approveAccessRequest(documentId: documentId, requestId: requestId)
```

#### Detect on `documents.get` failure

The 403 body is JSON shaped `{ error, status, timestamp, code: "DOC_ACCESS_DENIED", details: { code: "DOC_ACCESS_DENIED", canRequestAccess: boolean } }`. The client throws a typed error carrying that body already parsed — branch on `canRequestAccess` directly, no message-parsing needed:


`canRequestAccess` returns `true` only when:
- Caller is an `AppUser` (regular member, not anonymous).
- Caller is **not** an admin/owner (they already have access).
- Document exists, is not the root document.
- Caller has no existing direct or group permission.

#### `requestAccess` requires `permission`


Calling with no `permission` will 400. Re-requesting from the same user updates the existing pending request silently — there is no `ACCESS_REQUEST_ALREADY_PENDING` error. Real codes:

| Body code | When |
|-----------|------|
| `ALREADY_HAS_ACCESS` | Caller already has any permission |
| `RATE_LIMITED` | Too many requests; body has `retryAfter` |
| `ACCESS_REQUEST_ALREADY_RESOLVED` | Approve/deny called on a non-pending request |

#### Owner / admin flow

The owner lists pending requests with `listAccessRequests`, then resolves each with `approveAccessRequest` (optional `permission` override) or `denyAccessRequest`.


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

```swift
  // Create
  let collection = try await client.collections.create(
    params: CreateCollectionParams(name: "Q1 Reports", description: "All quarterly report documents")
  )
  let collectionId = collection.collectionId

  // Add / remove documents
  _ = try await client.collections.addDocument(collectionId: collectionId, documentId: documentId)
  _ = try await client.collections.removeDocument(collectionId: collectionId, documentId: documentId)

  // Share with a group (fans out to every document in the collection)
  _ = try await client.collections.grantGroupPermission(
    collectionId: collectionId,
    params: GrantCollectionGroupPermissionParams(groupType: "team", groupId: "engineering", permission: "read-write")
  )

  // Share with an individual user (O(1)). userId only — no email form.
  _ = try await client.collections.addMember(
    collectionId: collectionId,
    params: .user(targetUserId, permission: .reader)
  )
```

A collection's members + pending invites in one call:

```swift
  let access = try await client.collections.getAccess(collectionId: collectionId)

  // Or fetch just the pending (not-yet-signed-up) invitations:
  let pending = try await client.collections.listPendingInvitations(collectionId: collectionId)
```

Additional collection calls:


Collections accept email-based members exactly like documents and groups — a deferred grant that resolves on signup:


The deferred-grant flow when adding a collection member by email (`collections.addMember`) mirrors the document share path: `app.baseUrl` must be configured, `sendEmail: true` requires `collectionUrl`, and `client.invitations.delete(invitationId)` cancels every pending collection add (plus any pending document shares and group adds) attached to the invitation. See the [Invitations guide](AGENT_GUIDE_TO_PRIMITIVE_INVITATIONS.md#deferred-grants) for the resolution lifecycle.

For per-context CEL rules using `collectionType` + `contextId` (and the `hasCollectionAccess` helper), see [Rule Sets for Collection Management](AGENT_GUIDE_TO_PRIMITIVE_USERS_AND_GROUPS.md#rule-sets-for-collection-management) in the Users and Groups guide.

`collections.addDocument`/`removeDocument` are idempotent, and the same max-wins cascade applies. The template's local app-state pattern assumes one collection per doc (`ListRef.collectionId`); for multi-membership, model a `[String]` field or query `collections.listCollectionsForDocument(documentId:)` on demand. Read collection lists with `collections.list(options:)` and members with `collections.getAccess(collectionId:)` (see [Permission / collection reads](#permission--collection-reads) below).

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

### Sharing Discovery Cheat Sheet

Pick the call that answers the question you're actually asking:

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

**Anti-patterns:**

- Calling a method that doesn't exist: `setPermissions`, `setGroupPermission`. The correct names are `updatePermissions(documentId:params:)` and `grantGroupPermission(documentId:params:)`; resolve a user by email with `client.users.lookup(email:)`.
- Passing `permission: null` to remove a grant — there is no null form. Use `removePermission`.
- Lowering a user's direct permission while they still have a higher one via group — the group wins (effective = MAX).
- Assuming `me.sharedDocuments()` includes group- or collection-shared docs — it only carries direct `DocumentPermission` rows and pending `DocumentInvitation`s. Combine with `collections.list()` / `groups.listUserMemberships(...)` for a complete picture.
- Showing a "request access" button without checking the caught error's `canRequestAccess` detail.
- Calling `client.invitations.delete()` to cancel a single pending document share — it cascades to every share and group add linked to that invitation.
- Polling `client.invitations.list` / `listDeferredGrants` to populate "Members + Pending" rows — those are app-level / admin surfaces; per-resource `listPendingInvitations` is the product UI source.

**Sharing error codes** (server-emitted body codes):

The client throws a typed `JsBaoError` with `.code` and `.details` — read the code and details straight off the caught error, no manual message-parsing needed.

| Code | Endpoint | Meaning |
|------|----------|---------|
| `DOC_ACCESS_DENIED` (with `details.canRequestAccess`) | `documents.get` | 403; check the hint to decide between request-access UI and hard deny |
| `ALREADY_HAS_ACCESS` | `requestAccess` | Caller already has a direct or group permission |
| `RATE_LIMITED` | `requestAccess` | Body has `retryAfter` (seconds) |
| `ACCESS_REQUEST_ALREADY_RESOLVED` | `approveAccessRequest`, `denyAccessRequest` | Request is no longer pending |

App-membership and invitation error codes live in the [Invitations guide](AGENT_GUIDE_TO_PRIMITIVE_INVITATIONS.md#error-codes-quick-reference).

### Read-Only Permission Handling

When a user has "reader" permission, disable all edit functionality: hide create/add buttons, delete buttons and drag handles, and share buttons (only owners can share); disable inputs and checkboxes; and prevent inline editing. Derive a read-only flag from the document's `permission` field (`"reader"`) and thread it down to child views.


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

