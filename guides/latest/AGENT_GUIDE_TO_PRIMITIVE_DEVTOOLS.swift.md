# Agent Guide to DevTools: Inspecting State and Running Tests

How AI agents use Primitive's built-in developer tools to validate live app state,
run tests, and inspect blobs during development.

## Overview

Every Primitive app ships a dev-only inspection surface for working against a
running, authenticated client. Across platforms it gives you the same three core
capabilities:

1. **Data inspection** — browse and mutate the documents, models, and records the
   client holds.
2. **Test running** — run tests in the same authenticated session as the app and
   read their pass/fail output.
3. **Blob inspection** — list, preview, upload, download, and delete blobs in a
   document.

The tools are active only in development builds and never ship to production.


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

Registered models appear automatically for each open document. Register a model with `client.registerModels([...])` or use it through its generated model API.

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

Use the timeline to see when documents load and sync. Filter by document to inspect one operation's duration and data source.

## Storage inspection

The inspector includes views for query results and locally stored data. Use them to compare stored values with the records shown under **Documents**. Ad-hoc queries are read-only; use model operations to change records.

## Events

Filter the live client event log by type or text. Pause scrolling while inspecting a sequence.

## Logs

Read inspector errors here when an action fails.

## Validation workflow

```
1. Open the banner URL in a browser
2. Act in the app → switch to Documents → pick the document → read the records
3. To run tests: Tests tab → Run all (or Run selected) → read pass/fail + output
4. To inspect blobs: Documents tab → select the document → Blobs tab → preview
5. To cross-check query behavior: Memory SQL → run the same SELECT and compare
```

## Gotchas when using the inspector

- Use a trusted development network. The inspector has no authentication, so other clients on the network can reach it.
- Release builds exclude the inspector. Distribute release builds to external users.
- Inspection actions change the running app's data; use a test account and test data when exercising writes.

## Driving the UI with idb

Use idb to tap controls and type into the simulator. Use the inspector to check the resulting data and connection state.

### The `ui_signin` smoke scenario

The template ships this pairing ready to run. `scripts/smoke-test.sh` has a
`ui_signin` scenario that boots the simulator, launches the app, signs in
end-to-end, and asserts the post-login screen renders:

```bash
bash scripts/smoke-test.sh ui_signin   # the idb-driven UI sign-in test
bash scripts/smoke-test.sh --list      # launch_survive + ui_signin
bash scripts/smoke-test.sh             # default run — launch_survive only, no idb needed
```

`ui_signin` is available but not part of the default run, so a bare
`scripts/smoke-test.sh` (and `pnpm swift:smoke` in the monorepo) stays
zero-dependency. Run `ui_signin` explicitly to opt into idb.

The test uses an app-specific simulator. Set `PRIMITIVE_SMOKE_SIM` to choose its name or `PRIMITIVE_SMOKE_SIM_BASE` to choose a device type. Interactive runs use a separate simulator selected with `PRIMITIVE_RUN_SIM` or `--sim`.

If you replace the home screen, set `PRIMITIVE_SMOKE_SUCCESS_ID` to your screen's accessibility identifier. The default is `primitive.template.home`.

### Prerequisite: a test account

Configure a derived test email using the [Authentication guide](AGENT_GUIDE_TO_PRIMITIVE_AUTHENTICATION.md). Email sign-in must be enabled, the base email must be allowed, and the app must admit the derived address.

```bash
export PRIMITIVE_SMOKE_TEST_EMAIL="you+primitivetest-smoke@example.com"
# For an invite-only app, after admitting the derived address:
export PRIMITIVE_SMOKE_TEST_EMAIL_INVITED=1
PRIMITIVE_SMOKE_PREFLIGHT_ONLY=1 scripts/smoke-test.sh ui_signin
```

The preflight checks setup without booting or building. Resolve any reported settings or CLI authentication error before running the full scenario.

### Installing idb

In an app scaffolded from the Swift template, this is one command:

```bash
bash scripts/setup-idb.sh
```

The script installs the companion and a Python 3.12 client in a shared virtual environment. It reuses a working installation. Smoke tests find the client automatically; add `~/.local/bin` to PATH for manual commands.

### Driving idb by hand

To drive the app outside the scenario, start a companion and issue `ui`
commands against it. Coordinates are in **points** — the same units
`idb ui describe-all` reports — so there is no pixel conversion:

```bash
DEVELOPER_DIR="$(bash scripts/idb-developer-dir.sh)" idb_companion --udid <UDID> --grpc-port 10882 &   # serves on a gRPC port
idb --companion localhost:10882 ui describe-all         # the accessibility tree
idb --companion localhost:10882 ui tap <X> <Y>          # tap a point
idb --companion localhost:10882 ui text "hello"         # type into the focused field
idb --companion localhost:10882 ui key 40               # 40 = Return
```

Use `scripts/idb-developer-dir.sh` as shown so the companion can locate Xcode's input frameworks. The sign-in scenario does this automatically.

To tap a control by identifier rather than raw coordinates, read
`describe-all`, find the element whose `AXUniqueId` matches (this is where a
SwiftUI `.accessibilityIdentifier` surfaces), and tap the center of its `frame`
rounded to integer points (`idb ui tap` rejects non-integer coordinates) —
which is exactly what `ui_signin` does. On a failed assertion the scenario
writes a screenshot and dumps `describe-all` to the log, so a missed identifier
is diagnosable without re-running interactively.

`ui_signin` reinstalls the app before each run (clearing any persisted session
so the login screen shows) and launches with the Debug Inspector disabled
(`PRIMITIVE_DEBUG_INSPECTOR=0`), which avoids the first-launch Local Network
permission alert that would otherwise cover the form on a clean simulator.

### Reach

Like `launch_survive`, this is macOS-only (a booted simulator plus idb) and is
**not** part of the Linux `pnpm test` suite. It's a developer- and
agent-invoked check, not a gate the Linux CI enforces.

## HTTP diagnostics

- `Server-Timing: total;dur=<milliseconds>` reports server-side request duration.
- Authenticated app responses use `Cache-Control: no-store`; avatar responses are publicly cacheable.
- The clients send every request with the HTTP cache bypassed, so concurrent identical reads run in parallel. A direct `fetch` to the app API should pass `cache: "no-store"`; in the default cache mode a browser queues concurrent requests for one URL, one round trip each.
- Blob downloads support `ETag` and conditional `If-None-Match` requests. Keep the original body yourself if reusing it after a `304`.
