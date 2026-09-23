# Data Modeling and Storage Architecture in Primitive

How to choose between **documents** and **databases**, and how to combine them. Read this before designing the data layer of any new feature.

## The two storage systems

| | **Documents** (js-bao) | **Databases** (js-bao-wss) |
|---|---|---|
| Backed by | On-device document store (Yjs), synced over WebSocket | Isolated server-side store, read and written by server functions |
| Where data lives | On every client that has access, plus the server | Server only |
| Reads | Local, synchronous after `documents.open()` | A function call (network round trip); one call can read several models |
| Writes | Local first, async sync to server, automatic conflict-free merge | Network round-trip; last-write-wins per field |
| Concurrent edits | Merge cleanly (Yjs) — true collaborative editing | No merge; concurrent writers race |
| Real-time updates | Built in for everyone with the doc open | Opt-in: the function that writes publishes to a channel; granted clients subscribe |
| Offline | Yes — reads/writes work offline, sync resumes on reconnect (`offline: true` on the client) | No — every call requires the network |
| Access control | Whole-document grant: `reader`, `read-write`, `owner` | The function's `access` gate (who may call) and its code (which rows) |
| Per-record access for end users | Not possible — anyone with the doc gets everything | Yes — the function filters what each caller sees |
| Practical size | ~10 MB per ordinary document (soft); a large document (`documentFormat: 2`) is validated at 2 GB | ~5 GB per database (one isolated instance each) |
| Server-enforced fields | No (client writes Yjs updates directly) | Yes — `timestamps`, per-model triggers, and the function's own code |
| Server logic | Server functions read and write document records (`ctx.doc(id)`; see [The typed document handle](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#the-typed-document-handle)) | Server functions, plus `timestamps` and per-model triggers |
| Aggregates / multi-step reads | Client-side over local data | `aggregate` and `count` on the typed handle; several models in one function call |

The corollaries that follow are what to use when picking sides.

## Decision rules

Apply these in order. Stop at the first one that fits.

1. **Different users need to see different records inside the same dataset?** → **Database**. Documents grant access to the whole document; you cannot project rows out per user.
2. **Multiple users editing the same data live (Google-Docs style)?** → **Document**. Yjs is the only system here that merges concurrent edits without conflict.
3. **Must work offline?** → **Document** (open the client with `offline: true`). Databases need the network for every call.
4. **Dataset will exceed ~10 MB for a single sharing unit, or users only need a slice?** → **Database**. Documents replicate fully to every client. Size alone is the exception: when every member of the sharing unit needs all of the data, create it as a **large document** (`documentFormat: 2`) instead — records live in a persisted local store instead of in memory, validated at 2 GB. A Node client opens one with no extra configuration; a browser client needs the durable engine configured (`databaseConfig: { type: "opfs", options: { workerURL } }`) or the open is refused. See the large-document pattern below.
5. **Server must own a field (timestamps, audit fields, computed status, role assignments)?** → **Database**. Use `timestamps`, triggers, or the writing function's code; documents have no equivalent.
6. **Need aggregates, group-by, or one round-trip that touches several models?** → **Database** (`aggregate` on the handle; one function reading several models with `Promise.all`).
7. **None of the above and the data is per-user or per-shared-workspace?** → **Document**. Cheaper, lower latency, simpler.

If after running through these the answer is still ambiguous, ask the user before designing the data layer. Migrating between the two systems later is expensive.

### Common false signals

- "Real-time" alone does **not** mean documents. A function that writes a database row can publish the change to a channel; the difference is that documents also merge concurrent **edits**.
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

### Document — past the size threshold (large document)

Use when a workspace document's dataset will exceed ~10 MB and every member still needs all of it — multi-year records, imports. Create it with `documentFormat: 2` (`primitive documents create <title> --large` on the CLI) instead of splitting the data across documents or moving it to a database. Past creation it is the same document API: open it, then read and write through the same model classes — offline writes, live collaboration and permissions behave exactly as they do on an ordinary document. Creating one, which clients can open it, its limits and its read path: [Large Documents](AGENT_GUIDE_TO_PRIMITIVE_DOCUMENTS.md#large-documents).

### Database — a function scoped to its caller

```toml
# primitive/dev/functions/list-my-tasks.toml
[function]
key = "list-my-tasks"
entry = "functions/list-my-tasks/index.ts"
access = "true"
```

```ts
// primitive/dev/functions/list-my-tasks/index.ts
import { defineFunction, defineQuery } from "primitive-functions";

const myTasks = defineQuery("myTasks", {
  models: ["tasks"],
  params: { assignee: { caller: true } },   // $caller: injected by the platform, never supplied
  run: (db, params) =>
    db.model("tasks").query({
      filter: { assigneeId: params.assignee },
      options: { sort: { createdAt: -1 }, limit: 50 },
    }),
});

export default defineFunction(async (input: { databaseId: string }, ctx) => {
  return await myTasks(ctx.db(input.databaseId, "project"));
});
```

Use for: any data where what a caller sees depends on who they are. The `access` gate decides who may call; the code — here a `$caller` registered query — decides which rows they get. The client calls it with `client.functions.invoke("list-my-tasks", { input: { databaseId } })`. See the [Databases guide](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#access).

### Database — server-enforced fields

For plain audit times on every model, use `timestamps` on the type config:

```toml
[type]
databaseType = "project"
timestamps = { create = "createdAt", update = "modifiedAt" }
```

For invariants that depend on the record's data (e.g. set `completedAt` only when `status == "done"`), or that stamp who wrote a record, use a per-model trigger:

```toml
[triggers.tasks]
triggers = [
  { on = "create", set = { createdBy = "user.userId" } },
  { on = "save", when = "record.status == 'done' && record.completedAt == null", set = { completedAt = "now()" } },
]
```

Both apply server-side on every save and patch, whichever function made it.

### Database — realtime

The function that writes a row publishes the change to a channel in the same handler; clients the app granted that channel subscribe and receive it:

```ts
await ctx.db(input.databaseId, "support").model("ticket").patch(input.ticketId, { data: { status: "open" } });
await ctx.channels.publish(`tickets:${input.assigneeId}`, { ticketId: input.ticketId, status: "open" });
```

Load the initial state with a function call, then apply channel messages as they arrive. See [Channels](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#channels) for granting, subscribing and grant expiry.

## Worked architectures

### Personal productivity app

All data is per user. Documents only. One per-user document via `getOrCreateWithAlias`. Settings can sit in the root document via `userStore`.

### Collaborative workspace (shopping lists, project boards)

Documents only. One document per workspace. Owner shares with teammates via group or email. Real-time edits are free.

### Classroom / LMS (mixed)

- **Documents** for student work being actively drafted with teacher feedback (collaborative editing, offline drafting).
- **Database** (`type: "classroom"`, one per class) for assignments, grades, roster. Server functions enforce per-role visibility: a submit function records the work under the caller's own id; a grading function checks the caller's `teacher` membership for the class before it writes. Triggers stamp `submittedAt`, `gradedBy`.

### Multi-tenant SaaS / project management

- **Document** per user for personal preferences, dashboard layout.
- **Database** per organization (`type: "org"`) and **per project** (`type: "project"`). Each database is an isolated instance, so per-project databases scale and fail independently. The functions that read and write them check group membership (`team`, `admin`) in their `access` gate or code.

### E-commerce

- **Document** per user for cart and wishlist (offline, instant).
- **Database** (`type: "catalog"`, app-wide) for products, search, reviews. Per-seller `type: "seller_store"` databases for orders/inventory.

### Chat / messaging

- **Documents** per channel for messages — real-time merge-based sync handles concurrent posting and edits, full history available offline.
- **Database** (app-wide) for the channel directory and user profiles, behind `search-users` / `create-channel` functions.

## Design principles

### One database per logical boundary

Each database is one isolated instance. Split by tenant, project, or domain. Don't put everything in one giant database — multiple smaller databases scale better and isolate failures.

### Functions are your API

Server functions are the only way end users reach database data:

- Name them like API endpoints (`list-tasks`, `create-task`, `task-stats`).
- Start `access` restrictive (`hasRole(...)`, `isMemberOf(...)`), widen as needed; scope rows in the code.
- Take the caller from `ctx.user` or a `$caller` parameter — never from an id in the input.
- One function per screen's worth of reads: return several models from one call, and let optional input narrow the filter instead of writing `listX` and `listXByY` separately.

### Use server-side stamps, not client-supplied values, for invariants

`createdAt`, `createdBy`, `updatedAt`, computed status — all of these belong in `timestamps`, `[triggers.<model>]`, or the writing function's code, never in a value the client passes in. Even if your client always sets them correctly, a future client (or a malicious one) won't.

### Keep settings and state in records

Application settings, feature flags, display metadata and other mutable state belong in a record inside the database, read by the function that needs them. Records have no size cap beyond the per-database limit and change with an ordinary write. Per-database values that need their own read and write rules go in a [resource metadata](AGENT_GUIDE_TO_PRIMITIVE_RESOURCE_METADATA.md) category.

### Use groups for access control, not user-ID checks

Membership checks — `isMemberOf(groupType, groupId)` in a function's `access` gate, or the caller's memberships read in its code — scale better than per-user conditions and let membership change without rewriting functions. Reach for a group whenever the same access pattern applies to more than one user.

### When in doubt

- Concurrent editing of the same content? → **Documents**.
- Server must enforce who sees what? → **Databases**.
- Both? → Documents for the editing surface, databases for the structured/queryable layer (see Classroom and E-commerce above).

## Anti-patterns

| Don't | Do |
|---|---|
| Iterate documents to query (`for (doc of docs) Model.query(...)`) | Open all needed documents and let `.query()` run across them; filter by `documentId` if needed |
| Grant `manager` to end users so they can read records | Serve the records from a function; `manager`/`owner` are administrative roles only |
| Switch `setDefaultDocumentId()` repeatedly when the user changes context | Pass `targetDocument` explicitly on each `.save()` |
| Trust an id passed in the function's input to say who the caller is | Use `ctx.user` or a `$caller` registered query |
| Put one giant database for the whole app | One database per tenant/project/domain |
| Read several models with several function calls from one screen | One function that reads them with `Promise.all` and returns them together |
| Trust client-provided `createdAt`, `createdBy`, role fields | Set them with `timestamps`, `[triggers.<model>]`, or in the function |
| One function per filter combination (`list-posts`, `list-posts-by-author`, `list-posts-by-status`...) | One function whose optional input narrows the filter |
| `access = "true"` on a function that reads every row | Gate it, or filter the rows to the caller in the code |
| Use a database for real-time collaborative editing | Use a document — Yjs handles merge; databases will lose concurrent edits |
| Use a document for a 100k-row dataset where each user only needs 50 rows | Use a database behind a function that filters to the caller |

## Record identity and external IDs

Let Primitive assign the primary record ID as a ULID. Keep an external system's identifier in a separate field, such as `plaidTransactionId`, `stripeCustomerId`, or `externalId`. Relationships between app records should reference their Primitive IDs.

A provider identifier or a key assembled from business fields is a lookup value, not the primary record ID. Declare the appropriate unique field or composite constraint for deduplication, including the provider or account scope when the external ID is only unique within that scope. On a retry, look up that identity and reuse the existing record; generating another ULID alone does not make an import idempotent.

For model creates, omit the ID and use the model's automatic assignment. When an API requires an explicit ID, such as a document bulk create, use Primitive's ULID generator. In a function started as a task (`start`), generate those IDs inside `step.do` so a replay reuses them.

## Further reading

- [Documents guide](AGENT_GUIDE_TO_PRIMITIVE_DOCUMENTS.md) — full document API, sharing, sync, patterns
- [Databases guide](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md) — database types, the query and write bodies, triggers, lifecycle
- [Server Functions guide](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md) — the typed handle, registered queries, channels
- [Users and Groups guide](AGENT_GUIDE_TO_PRIMITIVE_USERS_AND_GROUPS.md) — group membership, CEL functions, role-based access
- **`ios-design` skill** — the iOS/SwiftUI UI-convention skill bundled in the Swift starter template (at `.claude/skills/ios-design/` in the scaffolded project). It governs the view layer the data-side rules here don't cover — platform gating, navigation, list affordances. It only activates once you're editing SwiftUI files inside the scaffolded project, so consult it explicitly before building any UI; architecture decisions made around scaffolding time can otherwise miss it.
