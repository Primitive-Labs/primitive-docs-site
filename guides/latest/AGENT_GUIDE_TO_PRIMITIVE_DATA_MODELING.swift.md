# Data Modeling and Storage Architecture in Primitive

How to choose between **documents** and **databases**, and how to combine them. Read this before designing the data layer of any new feature.

## The two storage systems

| | **Documents** (js-bao) | **Databases** (js-bao-wss) |
|---|---|---|
| Backed by | On-device document store (Yjs), synced over WebSocket | Isolated server-side store (SQLite), called over HTTPS |
| Where data lives | On every client that has access, plus the server | Server only |
| Reads | Local, synchronous after `documents.open()` | Network round-trip per call |
| Writes | Local first, async sync to server, automatic conflict-free merge | Network round-trip; last-write-wins per field |
| Concurrent edits | Merge cleanly (Yjs) — true collaborative editing | No merge; concurrent writers race |
| Real-time updates | Built in for everyone with the doc open | Opt-in via registered subscriptions (CEL filter per change) |
| Offline | Yes — reads/writes work offline, sync resumes on reconnect (`offline: true` on the client) | No — every call requires the network |
| Access control | Whole-document grant: `reader`, `read-write`, `owner` | Per-operation CEL on registered operations |
| Per-record access for end users | Not possible — anyone with the doc gets everything | Yes — operation CEL + filters scope what each caller sees |
| Practical size | ~10 MB per document (soft) | ~5 GB per database (one isolated instance each) |
| Server-enforced fields | No (client writes Yjs updates directly) | Yes — `autoPopulatedFields` and per-model triggers |
| Aggregates / multi-step reads | Client-side over local data | `aggregate`, `pipeline`, `count` operations |

The corollaries that follow are what to use when picking sides.

## Decision rules

Apply these in order. Stop at the first one that fits.

1. **Different users need to see different records inside the same dataset?** → **Database**. Documents grant access to the whole document; you cannot project rows out per user.
2. **Multiple users editing the same data live (Google-Docs style)?** → **Document**. Yjs is the only system here that merges concurrent edits without conflict.
3. **Must work offline?** → **Document** (open the client with `offline: true`). Databases need the network for every call.
4. **Dataset will exceed ~10 MB for a single sharing unit, or users only need a slice?** → **Database**. Documents replicate fully to every client.
5. **Server must own a field (timestamps, audit fields, computed status, role assignments)?** → **Database**. Use triggers; documents have no equivalent.
6. **Need aggregates, group-by, or one round-trip that touches several models?** → **Database** (`aggregate`, `pipeline`).
7. **None of the above and the data is per-user or per-shared-workspace?** → **Document**. Cheaper, lower latency, simpler.

If after running through these the answer is still ambiguous, ask the user before designing the data layer. Migrating between the two systems later is expensive.

### Common false signals

- "Real-time" alone does **not** mean documents. Databases push live changes via registered subscriptions; the difference is that documents also merge concurrent **edits**.
- "Shared with a team" alone does **not** mean databases. Documents share cleanly with groups when every member should see everything in the document.
- "Has a server" does not mean databases. Documents are also synced through a server — but the server treats the doc as opaque Yjs state and cannot enforce per-record rules.

### When to ask the user

If you cannot answer one of these from context, ask before building:

- **Sharing model**: private per user, shared identically with a group, or per-record visibility?
- **Volume**: order of magnitude of records and total bytes per sharing unit?
- **Roles**: do different roles see different subsets of the same data?
- **Offline / collaborative editing**: required, nice-to-have, or irrelevant?

## Canonical patterns

### Document — single per-user document (personal apps)

```swift
  // On app load, after sign-in
  let result = try await client.documents.getOrCreateWithAlias(
    options: GetOrCreateWithAliasOptions(
      alias: AliasRef(scope: .user, aliasKey: "default"),
      title: "My Data"
    )
  )
  _ = try await client.documents.open(result.documentId)

  // Reads run against local state after open()
  let openTasks = try Task.query(["completed": false])
```

Use for: task managers, journals, settings, preferences. Root document also works for a single per-user doc with no sharing.

### Document — one per workspace (shared collaboratively)

```swift
  let created = try await client.documents.create(
    options: CreateDocumentOptions(title: "Project Alpha", tags: ["workspace"])
  )
  let documentId = created.metadata?["documentId"]?.stringValue ?? ""
  _ = try await client.documents.open(documentId)

  _ = try await client.documents.updatePermissions(
    documentId: documentId,
    params: .email("alice@example.com", permission: "read-write")
  )
```

Use when each workspace is a sharing unit and every member of that workspace needs the full contents. Stays under ~10 MB per workspace.

### Database — registered operation with CEL access

```toml
# config/database-types/project.toml
[[operations]]
name = "listMyTasks"
type = "query"
modelName = "tasks"
access = "isMemberOf('team', database.celContext.teamId)"
[operations.definition]
filter = { assigneeId = "$user.userId" }
sort = { createdAt = -1 }
limit = 50
```

```swift
  let db = try await client.databases.create(params: CreateDatabaseParams(
    title: "Alpha",
    databaseType: "project"
  ))
  _ = try await client.databases.updateCelContext(
    databaseId: db.databaseId,
    celContext: ["teamId": "team-1"]
  )

  let result = try await client.databases.executeOperation(
    databaseId: db.databaseId,
    name: "listMyTasks"
  )
```

Use for: any data where access scopes change per record or per caller. The operation's CEL gate plus `$user.userId` / `$params.*` substitution does the per-row scoping for you.

### Database — server-enforced fields

For cross-model invariants (`createdAt`/`createdBy`/`updatedAt`), use `autoPopulatedFields` on the type config — one declaration covers every model:

```toml
[type.autoPopulatedFields]
ownerId   = "user.userId"
createdAt = { value = "now()", on = "create" }
updatedAt = { value = "now()", on = ["create", "update"] }
```

For invariants that depend on the record's data (e.g. set `completedAt` only when `status == "done"`), use a per-model trigger:

```toml
[triggers.tasks]
triggers = [
  { on = "save", when = "record.status == 'done' && record.completedAt == null", set = { completedAt = "now()" } },
]
```

Both fire server-side before the write reaches storage. Clients cannot bypass them.

### Database — real-time subscription

Subscriptions are declared on the type config and apply to every database of that type. Push them with `primitive sync push`.

```toml
[[subscriptions]]
subscriptionKey = "my-open-tickets"
displayName = "My open tickets"
modelName = "ticket"
accessRule = "user.userId != ''"
filter = "record.assigneeId == user.userId && record.status == 'open'"
select = ["id", "title", "priority", "updatedAt"]  # optional projection
```

Subscribe from the client and pair it with an operation call for the initial state:

```swift
  let initial = try await client.databases.executeOperation(databaseId: dbId, name: "listMyTickets")
  let unsub = try client.databases.subscribe(
    databaseId: dbId,
    subscriptionKey: "my-open-tickets",
    options: DatabaseSubscribeOptions(onChange: { event in
      if event.isOrigin { return } // this tab wrote it
      applyChanges(event.changes)
    })
  )
```

Subscriptions deliver deltas only — always pair with an operation call for the initial state, and use semantically equivalent filters. See the Databases guide for the full frame shape (`originConnectionId`, `originUserId`, `isOrigin`, `isOriginUser`, `changeType`).

## Worked architectures

### Personal productivity app

All data is per user. Documents only. One per-user document via `getOrCreateWithAlias`. Settings can sit in the root document via `userStore`.

### Collaborative workspace (shopping lists, project boards)

Documents only. One document per workspace. Owner shares with teammates via group or email. Real-time edits are free.

### Classroom / LMS (mixed)

- **Documents** for student work being actively drafted with teacher feedback (collaborative editing, offline drafting).
- **Database** (`type: "classroom"`, one per class) for assignments, grades, roster. Operations like `submitWork` (`access: "params.studentId == user.userId"`) and `gradeSubmission` (`access: "isMemberOf('teacher', database.celContext.classId)"`) enforce per-role visibility. Triggers stamp `submittedAt`, `gradedBy`.

### Multi-tenant SaaS / project management

- **Document** per user for personal preferences, dashboard layout.
- **Database** per organization (`type: "org"`) and **per project** (`type: "project"`). Each database is an isolated instance, so per-project databases scale and fail independently. Group membership (`team`, `admin`) gates operations.

### E-commerce

- **Document** per user for cart and wishlist (offline, instant).
- **Database** (`type: "catalog"`, app-wide) for products, search, reviews. Per-seller `type: "seller_store"` databases for orders/inventory.

### Chat / messaging

- **Documents** per channel for messages — real-time merge-based sync handles concurrent posting and edits, full history available offline.
- **Database** (app-wide) for the channel directory and user profiles, with `searchUsers`/`createChannel` operations.

## Design principles

### One database per logical boundary

Each database is one isolated instance. Split by tenant, project, or domain. Don't put everything in one giant database — multiple smaller databases scale better and isolate failures.

### Operations are your API

Registered operations are the only sane way to expose database data to end users:

- Name them like API endpoints (`listTasks`, `createTask`, `tasksByStatus`).
- Start `access` restrictive (`isMemberOf('team', database.celContext.teamId)`), widen as needed.
- Use `$user.userId`, `$params.*`, `$database.celContext.*` for substitution; never hardcode user IDs.
- Use `params: { ..., required: false }` to make a single operation accept optional filters instead of declaring `listX` and `listXByY` separately.

### Use server-side stamps, not client-supplied values, for invariants

`createdAt`, `createdBy`, `updatedAt`, computed status — all of these belong in `autoPopulatedFields` (cross-model) or `[triggers.<model>]` (model-specific). Even if your client always sets them correctly, a future client (or a malicious one) won't.

### Never store application state in `database.celContext`

`celContext` is **only** for values that CEL rules evaluate — typically a stable `teamId` / `projectId` / `tenantId` referenced from an `access` expression like `isMemberOf('team', database.celContext.teamId)`. If a value isn't read by a CEL rule, it does not belong here.

Do **not** put any of the following in `celContext`:

- Application settings or user preferences
- Feature flags
- Display names, descriptions, or other UI metadata
- Mutable runtime state of any kind
- Anything you might want to update from the client more than once

Why: `celContext` is hard-capped (~4 KB total, 1 KB per value, 20 keys), every change invalidates cached CEL evaluation context, and reads/writes go through an admin-gated path. It is rule-evaluation infrastructure, not a key-value store.

Where the data should live instead: a record inside the database itself. Read it from operations via a pipeline step or `$steps.*` reference, or just query it directly. Records have no size cap (within the per-database limit), can be updated by any caller authorized by the operation's CEL, and don't blow out the rule cache.

### Use groups for access control, not user-ID checks

`isMemberOf(groupType, groupId)` and `memberGroups(groupType)` scale better than per-user CEL conditions and let membership change without rewriting operations. Reach for a group whenever the same access pattern applies to more than one user.

### When in doubt

- Concurrent editing of the same content? → **Documents**.
- Server must enforce who sees what? → **Databases**.
- Both? → Documents for the editing surface, databases for the structured/queryable layer (see Classroom and E-commerce above).

## Anti-patterns

| Don't | Do |
|---|---|
| Iterate documents to query (`for (doc of docs) Model.query(...)`) | Open all needed documents and let `.query()` run across them; filter by `documentId` if needed |
| Grant `manager` to end users so they can read records | Define registered operations with CEL `access`; `manager`/`owner` are administrative roles only |
| Use direct record access (`db.connect(...).query(...)`) from end-user clients | Use `client.databases.executeOperation(...)`; direct access requires owner/manager |
| Switch `setDefaultDocumentId()` repeatedly when the user changes context | Pass `targetDocument` explicitly on each `.save()` |
| Put any application state in `database.celContext` (settings, flags, UI fields, anything not read by a CEL rule) | Store as a record; read via a pipeline `$steps.*` reference. `celContext` is rule-evaluation infrastructure, not a KV store |
| Put one giant database for the whole app | One database per tenant/project/domain |
| Hardcode IDs in operation `definition` | Use `$user.userId`, `$params.*`, `$database.celContext.*` |
| Trust client-provided `createdAt`, `createdBy`, role fields | Set them with `autoPopulatedFields` or `[triggers.<model>]` |
| One operation per filter combination (`listPosts`, `listPostsByAuthor`, `listPostsByStatus`...) | One operation with `params` declared `required: false` |
| Make every operation `access: "true"` | Start restrictive (`isMemberOf(...)`, `hasRole(...)`) and widen explicitly |
| Use a database for real-time collaborative editing | Use a document — Yjs handles merge; databases will lose concurrent edits |
| Use a document for a 100k-row dataset where each user only needs 50 rows | Use a database with a filtered registered operation |

## Further reading

- [Documents guide](AGENT_GUIDE_TO_PRIMITIVE_DOCUMENTS.md) — full document API, sharing, sync, patterns
- [Databases guide](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md) — full database API, operations, triggers, subscriptions, pipelines
- [Users and Groups guide](AGENT_GUIDE_TO_PRIMITIVE_USERS_AND_GROUPS.md) — group membership, CEL functions, role-based access
- **`ios-design` skill** — the iOS/SwiftUI UI-convention skill bundled in the Swift starter template (at `.claude/skills/ios-design/` in the scaffolded project). It governs the view layer the data-side rules here don't cover — platform gating, navigation, list affordances. It only activates once you're editing SwiftUI files inside the scaffolded project, so consult it explicitly before building any UI; architecture decisions made around scaffolding time can otherwise miss it.
