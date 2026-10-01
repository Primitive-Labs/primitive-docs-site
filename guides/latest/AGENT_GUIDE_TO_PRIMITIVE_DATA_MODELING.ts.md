# Data Modeling and Storage Architecture in Primitive

How to choose between **documents** and **databases**, and how to combine them. Read this before designing the data layer of any new feature.

## The two storage systems

| | **Documents** (js-bao) | **Databases** (js-bao-wss) |
|---|---|---|
| Backed by | On-device data, synced with the server | Isolated server-side store, read and written by server functions |
| Where data lives | On every client that has access, plus the server | Server only |
| Reads | Local, synchronous after `documents.open()` | A function call (network round trip); one call can read several models |
| Writes | Local first, async sync to server, automatic conflict-free merge | Network round-trip; last-write-wins per field |
| Concurrent edits | Concurrent edits merge automatically | No merge; concurrent writers race |
| Real-time updates | Built in for everyone with the doc open | Opt-in: the function that writes publishes to a channel; granted clients subscribe |
| Offline | Yes — reads/writes work offline, sync resumes on reconnect (`offline: true` on the client) | No — every call requires the network |
| Access control | Whole-document grant: `reader`, `read-write`, `owner` | The function's `access` gate (who may call) and its code (which rows) |
| Per-record access for end users | Not possible — anyone with the doc gets everything | Yes — the function filters what each caller sees |
| Practical size | ~10 MB per ordinary document (soft); a large document (`documentFormat: 2`) is validated at 2 GB | ~5 GB per database (one isolated instance each) |
| Server-enforced fields | Client-controlled | Yes — `timestamps`, per-model triggers, and the function's own code |
| Server logic | Server functions read and write document records (`ctx.doc(id)`; see [The typed document handle](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#the-typed-document-handle)) | Server functions, plus `timestamps` and per-model triggers |
| Aggregates / multi-step reads | Client-side over local data | `aggregate` and `count` on the typed handle; several models in one function call |

## Decision rules

- Choose documents for offline access and collaborative editing when everyone with access sees the same data.
- Choose databases for caller-specific record visibility and server-enforced fields.
- Use large documents when a dataset exceeds ordinary-document guidance but still needs one sharing boundary.

### Common false signals

Realtime updates alone do not require documents: a database-writing function can publish to a channel. Team sharing alone does not require databases: documents can be shared with groups.

### When to ask the user

Clarify the sharing boundary, expected volume, and offline requirements when the task does not establish them.

## Canonical patterns

### Document — single per-user document (personal apps)

```typescript
  // On app load, after sign-in
  const { documentId } = await client.documents.getOrCreateWithAlias({
    title: "My Data",
    alias: { scope: "user", aliasKey: "default" },
  });
  await client.documents.open(documentId);

  // Reads are local and synchronous after open()
  const tasks = await Task.query({ completed: false });
```

Use for: task managers, journals, settings, preferences. Root document also works for a single per-user doc with no sharing.

### Document — one per workspace (shared collaboratively)

```typescript
  const { metadata } = await client.documents.create({
    title: "Project Alpha",
    tags: ["workspace"],
  });
  await client.documents.open(metadata.documentId);

  await client.documents.updatePermissions(metadata.documentId, {
    email: "alice@example.com",
    permission: "read-write",
  });
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

Fields the server sets on every write, whichever function made it, come from the database type config: [`timestamps`](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#timestamps) for plain audit times, and per-model [triggers](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#triggers) when the rule depends on the record's data (`completedAt` only when `status == "done"`) or stamps who wrote the record.

### Database — realtime

The function that writes a row publishes the change to a channel in the same handler; clients the app granted that channel subscribe and receive it:

```ts
await ctx.db(input.databaseId, "support").model("ticket").patch(input.ticketId, { data: { status: "open" } });
await ctx.channels.publish(`tickets:${input.assigneeId}`, { ticketId: input.ticketId, status: "open" });
```

Load the initial state with a function call, then apply channel messages as they arrive. See [Channels](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#channels) for granting, subscribing and grant expiry.

## Worked architectures

- **Personal app:** one document per user.
- **Team workspace:** one document per independently shared project.
- **Marketplace:** database for inventory and orders; documents for private lists or drafts.

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

## Gotchas

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
