# Agent Guide to DevTools: Inspecting State and Running Tests

How AI agents use Primitive's built-in developer tools to validate live app state,
run tests, and inspect blobs during development.

## Overview

Every Primitive app ships a dev-only inspection surface for working against a
running, authenticated client. Across platforms it gives you the same three core
capabilities:

1. **Data inspection** — browse and mutate the documents, models, and records the
   client holds, plus server-side databases.
2. **Test running** — run tests in the same authenticated session as the app and
   read their pass/fail output.
3. **Blob inspection** — list, preview, upload, download, and delete blobs in a
   document.

The tools are active only in development builds and never ship to production.

## Server Timing

Every REST response from the platform (`/app/{appId}/api/*` and `/admin/api/*`)
carries a `Server-Timing: total;dur=<int-ms>` header attributing the request's
server-side handler time. The header is listed in `Access-Control-Expose-Headers`,
so any HTTP tooling — or the response object of a raw fetch — can read it when
attributing a slow request to server work vs. transport.


The tools are the **Debug Inspector**: a dev-only panel served by the running app
and opened in a web browser. The inspector compiles to zero code in release builds
(the whole module is behind `#if DEBUG`), so it never ships to production.

## Setup

The inspector is built into `PrimitiveApp` and starts automatically in DEBUG
builds — no configuration. When `PrimitiveAppState.initialize()` runs it prints a
banner with the URLs to open:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[PrimitiveInspector] 01KN7M… listening on:
  http://localhost:9999
  http://192.168.1.42:9999
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Open one of those URLs in any browser on the same network. The simulator shares
the Mac's loopback, so `localhost:9999` reaches it directly; a physical device is
reached over Wi-Fi (the first run prompts for Local Network permission — accept it,
or the device silently drops the connection).

- **Opt out** for a single run: set `PRIMITIVE_DEBUG_INSPECTOR=0` in the environment.
- **Pin a port** if 9999 is taken: `PRIMITIVE_DEBUG_INSPECTOR_PORT=9998`.

The UI polls bulk state every ~2s and receives a live event stream, so values
update on their own. Every mutation you trigger calls the corresponding
`JsBaoClient` method on the main actor.

## The tabs

The inspector opens on an **Overview** dashboard (read-only): connection state,
current user, a document summary, a storage summary, and a tail of the last ~20
events. The other tabs are below.

## Documents

Three-column CRUD for documents plus a per-model records browser.

- **Left** — filterable document list, color-coded for open / pending / local-only.
  Clicking a document auto-opens it.
- **Middle** — the **Records** table: pick one of the registered models for the
  selected document, read its rows, add a record via a schema-typed form (inputs
  dispatched by field `kind`: string/number/boolean/date/id/stringset/json), and
  delete rows.
- **Right** — detail for the selected document: rename / close / delete / share,
  plus raw metadata.

### Common tasks

```
Verify a record exists / its field values:
  1. Left list → click the document (auto-opens)
  2. Middle "Records" → pick the model
  3. Read the rows directly

Verify a record was deleted:
  1. Pick the model in "Records"
  2. Confirm the row is gone / the count dropped

Create a record by hand:
  1. Pick the model → "new record" form
  2. Fill the schema-typed inputs → submit
```

### Surfacing your models

Models appear in the Records table automatically — the base `PrimitiveAppState`
conforms to `InspectableModelHost`, and its default `inspectableModels` reads the
client's shared cross-document store (one entry per model per open document). A
model is listed as soon as it's registered (`client.registerModels([...])`) or
read/written once through its codegen'd facade. The standard path needs no
inspector glue.

For a hand-built runtime-schema `DynamicModel` that doesn't go through the
facade, `override` the `open var inspectableModels` and append:

```swift
class MyAppState: PrimitiveAppState {
  override var inspectableModels: [InspectableModel] {
    guard let docId = modelsDocId else { return super.inspectableModels }
    var out = super.inspectableModels        // keep the auto-surfaced models
    if let m = runtimeTaskModel { out.append(.from(m, documentId: docId)) }
    return out
  }
}
```

`InspectableModel.from(...)` takes the `DynamicModel` and wires `loadAll` /
`deleteById` / `createWith` so the table, delete buttons, and create form all
work with no extra code. Field descriptors come from the model's schema.

## Tests

Runs the tests your app state registers, sequentially, in the same authenticated
session as the app. Each test's output streams into the right pane; per-test
pass/fail + duration shows in the tree on the left.

- Run selected / Run all / Run one / Clear results.
- Copy all output, or copy only failed output, to the clipboard.

Tests are plain Swift closures — they can call real client methods, assert through
`ctx.check(...)`, and log via `ctx.log(...)`. Conform your `PrimitiveAppState`
subclass to `InspectorTestHost`:

```swift
extension MyAppState: InspectorTestHost {
  var inspectorTests: [InspectorTest] {
    [
      InspectorTest(group: "Client", name: "is connected") { [weak self] ctx in
        guard let self, let client = self.client else {
          throw TestFailure(message: "no client")
        }
        ctx.log("connection id: \(client.connectionId)")
        try ctx.check(client.isConnected, "client is not connected")
      },
      // … more tests …
    ]
  }
}
```

Tests run on the main actor, so they can touch `@MainActor`-isolated state
directly. `ctx.log` accumulates output; `ctx.check(Bool, String)` throws on false;
any thrown `Error` marks the test failed with its description. Writing a test that
does the minimum setup to reproduce a bug is usually the fastest way to build a
repro and iterate.

## Databases

Two CRUD paths over a server-side database, in one view:

1. **Models (primary)** — generic CRUD on rows, bypassing operation rules. Gated
   server-side to app admins and database owners; a non-admin query returns
   401/403 and the UI auto-expands the operations path with an explanation. This
   is the admin debugging door.
2. **Registered operations (secondary)** — the CEL-gated, app-defined named
   operations registered via `DatabasesAPI.createOperation(databaseId:params:)`.
   Anyone with access can run an operation within its rule; the result renders as
   a table (array of dicts) or raw JSON.

Both coexist so any user can interact with any database: admins browse raw via
Models; everyone else uses the operations path.

## Collections

Two-column CRUD for collections of documents. Left = filter + list + create.
Right = the selected collection's detail: rename / delete, the documents in it
(per-row remove, add by documentId), and its access (raw `getAccess()` dump plus
grant-group and add-member prompts).

## Blobs

Two-column file browser, scoped to whichever document is selected on the Documents
tab.

- **Left** — blob list + a file-picker "upload" button.
- **Right** — selected blob detail with an in-browser preview: `image/*` inline,
  text / JSON / XML / YAML / TOML decoded into a `<pre>`, everything else as an
  open / download link pair.

## Performance

A live chronological timeline of every document-level phase event the inspector
has seen — `loadedFromSqlite`, `loadedFromServer`, `synced`, `remoteUpdate`,
`closed` — with plain-English labels. Summary cards totalize on-disk vs server
volume and timing. The whole view derives from the continuous event stream, so new
events appear the instant they arrive; a filter-by-doc dropdown isolates one
document's trace when several are syncing at once. Read the timeline to answer
"what data loaded, from where, in how long?" — each `synced` event is annotated
with the models that became available on that document.

## Memory SQL

A live browser and ad-hoc query runner over the in-memory SQLite that backs
`model.query()` / `count` / `aggregate` / `queryPaged` / `findByUnique`. The engine
is per-(model, document) and lives entirely in RAM, kept in sync with the
document's data via an update observer.

- **Model picker + context strip** — the model name and the `documentId` the
  projection is bound to.
- **Schema strip** — `PRAGMA table_info` for the active table (name, type, PK, NOT
  NULL), so you can see what an indexed query actually hits.
- **Table view** — a true SQL table; each row has a `del` button that routes through
  the model's delete (document-store → engine projection), not a raw SQL DELETE.
- **Create row** — a `+ new row` modal, inputs dispatched by field `kind`.
- **Custom query box** — a SQL textarea (Cmd/Ctrl+Enter to run). Accepts only
  `SELECT`, `PRAGMA`, and `WITH` (CTE) statements; direct DML is rejected because
  it would desync the projection from its document-store source of truth.
- **Polling** — while the tab is active, the catalog + active table refresh every
  ~2s; a button pauses/resumes (useful for a stable result set).

Useful for: which columns the engine projects for a model, whether stringset
junction tables populate on writes, whether SQL rows match the records on the
Documents tab, and how a filter/index translates to SQL.

## SQLite (disk)

Read-only browser over the client's on-disk durability store — the file at
`<Documents>/JsBaoClient/<appId>:<userId>/jsbao_storage.sqlite`. This is the
durability layer (encoded document blobs, `meta`, `kv`, `auth`), not the query layer;
for per-model SQL tables use Memory SQL. Table tabs along the top show each store
with its row count; clicking loads rows (key, value, metadata, updatedAt),
most-recently-updated first.

The **diag** toggle opens a resolution trace: the `appId` being scanned, the
filesystem roots checked, every `appId:*` subdir found (with mtime), the resolved
storage path, and a file probe (open status, size, journal mode, listed tables,
row counts pre/post checkpoint). If the tab looks blank, diag says why — typically
"no logged-in namespace dir yet".

## Events

The full live event stream from `client.events` as a filterable log: pause,
auto-scroll, filter-by-type, text filter. The stream is captured continuously (not
tab-gated), so switching back to this tab shows you didn't miss anything.

## Logs

Inspector-internal log tail — failed actions, internal errors and warnings. It
self-refreshes while active so failures surface immediately, and an action failure
re-loads logs so the same error has richer context next time you look.

## Validation workflow

```
1. Open the banner URL in a browser
2. Act in the app → switch to Documents → pick the document → read the records
3. To run tests: Tests tab → Run all (or Run selected) → read pass/fail + output
4. To inspect blobs: Documents tab → select the document → Blobs tab → preview
5. To cross-check query behavior: Memory SQL → run the same SELECT and compare
```

## Security + limits

- The whole inspector is behind `#if DEBUG`; release builds open no port, capture
  no events, and read no UI resources.
- No authentication — any client on the same network can hit the endpoints. Fine
  on a personal dev network; don't run DEBUG builds on public Wi-Fi, and don't
  ship a DEBUG build to external testers.
- Every action is a round trip over the network (~5–20ms on a good LAN), so it's
  visibly not real-time; the event stream is push-based, so observation lag stays
  low even when action lag doesn't.
