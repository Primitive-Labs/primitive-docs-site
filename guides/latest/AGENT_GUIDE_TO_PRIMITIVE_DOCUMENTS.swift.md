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

```swift
  _ = try await client.documents.open(documentId)
  let result = try Task.query([:], options: QueryOptions(documents: [documentId]))
```

Await `open()` before reading or writing. Show a loading state while it runs, and handle failures before continuing. Close documents when they are no longer needed.

#### Gotchas when opening

- Queries span all open documents, so unnecessary opens also widen query results.
- An open requiring server state can fail offline or time out. Handle the error code and retry `open()` when connectivity returns.
- Open after authentication. A view or store being ready does not mean its documents are open.

Open documents from `PrimitiveAppState.connectClient()`. See [The document lifecycle](#the-primitiveappstate-document-lifecycle).

### 2. Finding documents a user can access

List owned, directly shared, group-shared, and collection-shared documents separately, then combine them for navigation.

**a. Documents they own** (`ownedDocuments` — created, or ownership transferred):

```swift
  // A typed `[DocumentInfo]` — each row carries title, permission, tags, …
  let owned = try await client.me.ownedDocuments(tag: "channel")

  for doc in owned {
    print(doc.title, doc.permission)
  }
```

**b. Documents shared directly with them** (`sharedDocuments` — direct non-owner grants; group and collection shares use their own lists):

```swift
  let page = try await client.me.sharedDocuments(limit: 50, tag: "channel")

  for share in page.items {
    // Each row nests the base document fields (title, createdAt, …) under
    // `.document`, alongside the share extras (grantedBy, source).
    print(share.document.title, share.document.permission, share.grantedBy)
  }

  // `nextCursor` is an opaque pagination token — pass it back as `cursor` for the next page.
  if let cursor = page.nextCursor {
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

`sharedDocuments` returns `SharedDocument` values: document fields are under `.document`, with `grantedBy` and `source` alongside them. Group and collection listings have their own result types.


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
    // Only `completed` is written — fields you didn't assign are left alone,
    // so another device's concurrent edit to them survives. Assign the result
    // back: it's the saved record, with no pending changes left to re-write.
    task = try task.save(in: documentId)
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

### Absent fields

`$ne`, `$nin` and a `null` equality **match records where the field is absent** — a record that never wrote the field is not equal to any value, as in MongoDB — so a `$ne: true` filter on `deleted` is the "false or not set" filter a soft-delete list needs, with no backfill onto existing records. Equality with a value, the comparison operators (`$gt` / `$gte` / `$lt` / `$lte`) and `$in` **match only records that carry the field**. A schema `default` is applied on read and does not materialise the field in storage, so it does not change what a filter matches.

Test presence with `$exists`. Its treatment of an *explicitly stored* null is path-dependent: the server counts a **stored JSON null** as present (`$exists: true` matches it), while the browser and Swift replicas keep each field in a typed column where a stored null is indistinguishable from an absent field (`$exists: false` matches it). Absent fields behave the same on every path; only explicit nulls differ. `$in` with a `null` entry matches nothing for that entry — use a `null` equality or `$exists: false` instead.

**Negative operators include absent fields.** A filter that uses `$ne` or `$nin` to *exclude* records by a possibly-absent field matches the records lacking it — including access filters injected by `beforeQuery` hooks and write conditions. To exclude the missing case, add a `null` entry to `$nin` — it matches only records carrying a non-null value other than the excluded one, identically on every path:

```swift
["deleted": ["$nin": [nil, true]]]
```

`$exists: true` alongside the negative operator is **not** equivalent: on the server an explicitly stored JSON null counts as present, so a `$ne: true, $exists: true` filter still matches a record whose `deleted` is null — a record the old filter excluded.

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

#### Sorting on a field some records don't have

Records are schemaless, so the records of one model need not all carry the field you sort on, and a record may carry it as `null`. **Absent and `null` are one value for ordering** — a record that never wrote the field sorts exactly where a record holding `null` sorts:

| Sort | Where those records land |
|---|---|
| `{ priority: 1 }` (ascending) | **first**, before every real value |
| `{ priority: -1 }` (descending) | **last**, after every real value |

Pagination uses the same ordering on clients and the server. Ties use `id`; a page with more results includes a cursor. Pass the cursor back unchanged.

Cursors are **opaque** base64 tokens — never parse or construct one. A cursor issued before this ordering was specified keeps working unchanged, so a client holding one need not restart its walk.

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

### Local-only documents

`documents.create({ localOnly: true })` creates a document that never reaches the server — not in the creating session, not in any later one, and not after the local copy is evicted and its metadata reloaded from the stored row. Every edit to it is dropped before it would be queued outbound, so there is nothing to mark unsynced and nothing to commit later.

```swift
  let result = try await client.documents.create(
    options: CreateDocumentOptions(title: "Draft", localOnly: true)
  )
  let documentId = result.metadata?["documentId"]?.stringValue
  if let documentId {
    _ = try await client.documents.open(
      documentId,
      options: OpenDocumentOptions(waitForLoad: .local, enableNetworkSync: false)
    )
  }
```

Open a local-only document with `waitForLoad: .local` and `enableNetworkSync: false` — the content never reaches the wire regardless of the open options, but these are the ones that skip an unreachable network wait.

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
  if let documentId {
    // Open before querying or writing — a write to an unopened document throws.
    _ = try await client.documents.open(documentId)
  }
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

To create and immediately edit a document, call `client.createDocument(options:)`, read `metadata?["documentId"]?.stringValue`, then open that document before writing. Creation returns metadata; it does not open the document.

The default open can use the local copy. A network-required open waits up to `availabilityWait` (30 seconds by default), then throws `.networkTimeout` if the document is still unavailable.

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
    task = try task.save(in: documentId)

    // Compare
    if let dueDate = task.dueDate, dueDate < now {
      // overdue
    }
  }

  // Query with date comparison
  let overdue = try Task.query(["dueDate": ["$lt": .string(now)]])
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

The data a view renders is a plain facade query — `TodoItem.findAll()` / `TodoItem.query(...)` (see [Read](#read-find--query--first--count)) — re-run whenever the records change, which you observe with `TodoItem.subscribe` (see [Subscribe to changes](#subscribe-to-changes)). `BaoDataLoader<[T]>` is the **SwiftUI glue** (PrimitiveApp package) that wires that re-run into a view: bind it rather than subscribing to client events directly or rolling a `@Published var items` + manual `refresh()`. The loader owns its subscription lifecycle (cancelled on deinit), debounces bursts (~50ms), runs the first load immediately, and re-runs a synchronous `load` closure on every trigger.

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

**Saving an existing record writes only the fields you assigned** since you read it. Fetch–mutate–save is therefore a field-level write, and two devices editing different fields of the same record merge instead of overwriting each other:

```swift
var task = try TaskRecord.find(id)!   // read: nothing pending
task.title = "New title"              // only `title` is marked changed
task = try task.save(in: documentId)  // only `title` is written
```

Inserting a record the document doesn't have yet writes every field, so copying a record into another document still carries all of it. If the other document ALREADY holds that record, the save is an update — and an unmodified record has nothing to write, so it does nothing. Call `record.markAllChanged()` first when you mean to copy the whole record over it.

`save(in:)` returns the record **as saved** — re-read from the document with no pending changes left, so a field another device changed while you held your copy, and any schema defaults filled in on insert, are present. Assign it back (as above) when you keep using the value, or call `record.discardChanges()` to drop pending edits without writing them. Constructing a record (`TaskRecord(id:title:)`) or decoding one from JSON marks every field it carries as changed; only a record read back through the facade starts clean.


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
- `documents.createWithAlias(options)` - Create document with alias atomically (fails if alias already exists). Takes `title`, `tags`, `metadata` and `documentFormat`, each applied at creation exactly as on `documents.create`.
- `documents.getOrCreateWithAlias(options)` - Get existing document by alias, or create a new one if not found. Returns `{ documentId, created: boolean, documentFormat?, tags?, metadata?, ... }`. Use this for idempotent initialization. Takes the same options, applied only when the call creates the document; a stated `documentFormat` that the existing document does not have is refused with `DOCUMENT_FORMAT_MISMATCH` (409) rather than handing back a document of the other kind.
- All three create routes (`documents.create`, `createWithAlias`, `getOrCreateWithAlias`) **refuse** a top-level body key they do not read, with 400 `VALIDATION_FAILED` and `details` naming it — including `name`, `parentId` and a flat `scope`/`aliasKey` pair. The typed client methods send only keys the routes read.
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

**Refresh document navigation on launch, foreground, or pull-to-refresh.** Read each access path, combine permissions using the highest level, and update references keyed by document ID. Remove references absent from the refreshed set, while protecting pending local creates as described below.

> **Guard freshly-created docs from the prune.** `createDocument(...)` is local-first — it returns a `CreateDocumentResult` (metadata only; read the id from `metadata?["documentId"]?.stringValue`) immediately but commits to the server in the background, so a doc you just created is **not yet** in `me.ownedDocuments`. A reconcile firing in that window deletes the brand-new ref and the item vanishes until a later reconcile re-adds it. Track locally-created ids in a `pendingCreateIds: Set<String>` (add on create, remove once the id appears in the server set) and exclude them from the prune: `where !serverIds.contains(ref.id) && !pendingCreateIds.contains(ref.id)`.

For the reconcile reads, the "shared with me" set is the union of `me.ownedDocuments(tag:limit:)` and `me.sharedDocuments(tag:limit:)` — merge them yourself (the same entity can surface in both), then dedupe by document id.

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

The point-in-time checks accept a `timeout` — a `TimeInterval` in seconds, default `5`. For a cheap synchronous local read — no round-trip — use `documents.isSynced(documentId:)`.

**A document that cannot sync says so.** The `documentSyncStateChanged` event reports `state: "error"` for an open document whose sync handshake goes unanswered for the whole handshake budget (10 s by default) — what the app is rendering has stopped converging — and repeats it on each timeout while the document stays behind. The client keeps retrying underneath, at a backoff that caps at 15 s, and after three consecutive timeouts on a connection that still reads as open it rebuilds the connection itself (once per stalled document, once per connection). Once the document does sync, the event fires `"synced"` once more so the app can clear what it surfaced:

```swift
  // Drop this in a SwiftUI `.task`: the loop runs for as long as the view is on
  // screen and unsubscribes when it goes away.
  for await event in client.stream(for: DocumentSyncStateChangedEvent.self) {
    switch event.state {
    case "error":
      // The document's sync handshake went unanswered for its whole budget
      // (10 s by default). Repeats on each timeout while the document stays
      // behind; the client keeps retrying on its own.
      ui.markStale(event.documentId)
    case "synced":
      // Fires as remote updates are applied — and once more when a document
      // that reported an error catches up, so the warning can be cleared.
      ui.markSynced(event.documentId)
    default:
      break
    }
  }
```

The same event reports `"synced"` as each remote update is applied to an open document, so a loader that reloads on every remote write subscribes here too. It says nothing about a document's overall caught-up/behind state — that is what the point-in-time checks above answer.

The handshake budget is `SyncConfig.handshakeTimeout`, in seconds.

### Connectivity vs network mode

Network **mode** is user intent — `auto` by default, or pinned by `goOffline()` / `goOnline()`. **Reachability** is whether the device currently has a network path. They are separate, and the client never turns one into the other: losing connectivity in `auto` pauses the socket and suppresses reconnect, but the reported mode stays `"auto"`. When the network comes back the client reconnects on its own, still in `auto`.

So test connectivity with `isOnline`, not with the mode:

`client.networkStatus` carries the same `mode` / `isOnline` / `reason` triple, `NetworkModeEvent` fires on both kinds of transition, and `client.isOnline()` is the check to branch on. Drive offline UI from `isOnline`, not from `networkMode == .offline`.

While the device is unreachable the client behaves as it does when offline is pinned: reads come from the local cache and queued blob uploads wait. Nothing is lost — it resumes on reconnect.

HTTP calls made while offline fail fast, without attempting the request, with a `JsBaoError` whose code is `.offline`. A request that is attempted and fails in transit (DNS failure, connection refused) throws `JsBaoNetworkError` instead; its `urlErrorCode` carries the underlying `URLError` code. Treat both as retryable.

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

The one exception: if the caller is neither the owner nor an app owner, the platform falls back to checking every collection the document belongs to and allows the delete if **any** one collection's `document.delete` CEL rule passes. That rule defaults to `"false"` (deny) — see [Collection Rule Sets](#collection-rule-sets) — so deletion never widens past owner/app-owner unless an app explicitly configures it. Because that rule is evaluated per collection, adding a document to a collection can extend who is able to delete it; configure `document.add` and `document.delete` together with that reach in mind.

```swift
  // Must be closed first
  try await client.documents.delete(documentId: documentId)

  // Force-close before deleting
  try await client.documents.delete(
    documentId: documentId,
    options: DeleteDocumentOptions(forceCloseIfOpen: true)
  )
```

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
| `RATE_LIMITED` | Too many requests; `details.retryAfter` gives the wait in seconds |
| `ACCESS_REQUEST_ALREADY_RESOLVED` | Approve/deny called on a non-pending request |

#### Owner / admin flow

The owner lists pending requests with `listAccessRequests`, then resolves each with `approveAccessRequest` (optional `permission` override) or `denyAccessRequest`.


> **Note:** `denyAccessRequest` does not accept a `reason` field — only `documentUrl`. Don't try to pass one.

#### Constraints

- 30-day TTL on unresolved requests.
- One pending request per `(document, requester)` — re-requesting updates it in place.
- Resolved requests are immutable.

#### Refreshing access requests

Refresh `listAccessRequests()` when the owner opens the requests view. The requester receives an email with the outcome.

### Collections

A collection shares a set of documents as one unit. Access from collections and direct grants combines; the highest permission wins. Deleting a collection preserves its documents.

```swift
  // Create a collection and put documents in it
  let collection = try await client.collections.create(
    params: CreateCollectionParams(name: "Project Phoenix")
  )
  _ = try await client.collections.addDocument(collectionId: collection.collectionId, documentId: designDocId)
  _ = try await client.collections.addDocument(collectionId: collection.collectionId, documentId: specDocId)

  // One grant covers the whole set — including documents added later
  _ = try await client.collections.addMember(
    collectionId: collection.collectionId,
    params: .email("alice@example.com", permission: .readWrite)
  )
```

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

  // Share with an individual user (O(1)). `.email(...)` also works for a
  // deferred grant that resolves on signup, exactly like documents and groups.
  _ = try await client.collections.addMember(
    collectionId: collectionId,
    params: .user(targetUserId, permission: .reader)
  )
```

```swift
  let access = try await client.collections.getAccess(collectionId: collectionId)

  // Or fetch just the pending (not-yet-signed-up) invitations:
  let pending = try await client.collections.listPendingInvitations(collectionId: collectionId)
```

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

```swift
  // Current members (accepted permission grants)
  let members = try await client.documents.getPermissions(documentId: documentId)

  // Pending email invites on this document
  let pending = try await client.documents.listPendingInvitations(documentId: documentId)
```

```swift
  let access = try await client.collections.getAccess(collectionId: collectionId)

  // Or fetch just the pending (not-yet-signed-up) invitations:
  let pending = try await client.collections.listPendingInvitations(collectionId: collectionId)
```

### Sharing Discovery Cheat Sheet

Pick the call that answers the question you're actually asking:

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

**Anti-patterns:**

- Calling a method that doesn't exist: `setPermissions`, `setGroupPermission`. The correct names are `updatePermissions(documentId:params:)` and `grantGroupPermission(documentId:params:)`; resolve a user by email with `client.users.lookup(email:)`.
- Passing `permission: null` to remove a grant — there is no null form. Use `removePermission`.
- Lowering a user's direct permission while they still have a higher one via group — the group wins (effective = MAX).
- Assuming `me.sharedDocuments()` includes group- or collection-shared docs — it only carries direct grants. Combine with `collections.list()` / `groups.listUserMemberships(...)` for a complete picture.
- Showing a "request access" button without checking the caught error's `canRequestAccess` detail.
- Calling `client.invitations.delete()` to cancel a single pending document share — it cascades to every share and group add linked to that invitation.
- Polling `client.invitations.list` / `listDeferredGrants` to populate "Members + Pending" rows — those are app-level / admin surfaces; per-resource `listPendingInvitations` is the product UI source.

**Sharing error codes** (server-emitted body codes):

The client throws a typed `JsBaoError` with `.code` and `.details` — read the code and details straight off the caught error, no manual message-parsing needed.

| Code | Endpoint | Meaning |
|------|----------|---------|
| `DOC_ACCESS_DENIED` (with `details.canRequestAccess`) | `documents.get` | 403; check the hint to decide between request-access UI and hard deny |
| `ALREADY_HAS_ACCESS` | `requestAccess` | Caller already has a direct or group permission |
| `RATE_LIMITED` | `requestAccess` | `details.retryAfter` gives the wait in seconds |
| `ACCESS_REQUEST_ALREADY_RESOLVED` | `approveAccessRequest`, `denyAccessRequest` | Request is no longer pending |

App-membership and invitation error codes live in the [Invitations guide](AGENT_GUIDE_TO_PRIMITIVE_INVITATIONS.md#error-codes-quick-reference).

### Read-Only Permission Handling

When a user has "reader" permission, disable all edit functionality: hide create/add buttons, delete buttons and drag handles, and share buttons (only owners can share); disable inputs and checkboxes; and prevent inline editing. Derive a read-only flag from the document's `permission` field (`"reader"`) and thread it down to child views.


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

```swift
  let result = try await client.documents.create(
    options: CreateDocumentOptions(title: "Ledger", documentFormat: 2)
  )
  let documentId = result.metadata?["documentId"]?.stringValue
  // The format the create recorded locally, before the server commit lands.
  let format = result.metadata?["documentFormat"]?.numberValue
  _ = format

  if let documentId {
    // Open before querying or writing, as with any document.
    _ = try await client.documents.open(documentId)
  }
```

The same format option applies to `getOrCreateWithAlias`; use an alias to create one document safely across concurrent attempts.

### Gotchas for large documents

- An explicit format on `getOrCreateWithAlias` must match an existing document or the call fails with `DOCUMENT_FORMAT_MISMATCH`.
- Offline writes expire after the configured window (7 days by default, configurable from 1 to 14 days). Sync before resuming writes. Handle `DOCUMENT_OFFLINE_WINDOW_EXPIRED`.
- Queries must select ordinary documents or large documents, not both. Use `documents` to avoid `FORMAT2_QUERY_SCOPE`.
- Nested collaborative values are rejected; store plain JSON field values.
- Sync pending changes before eviction. Without `force`, eviction refuses to discard unsynced writes.
- On `FORMAT2_FOLD_BROKEN`, reconnect to restore the local data before reading or writing.


Use an on-disk store. An in-memory store cannot open a large document and reports `FORMAT2_STORAGE_UNAVAILABLE`.

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

