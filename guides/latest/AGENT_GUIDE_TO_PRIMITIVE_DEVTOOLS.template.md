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

{{#lang ts}}
The tools are a single browser overlay (the **DevTools** overlay) provided by the
`primitiveDevTools` Vite plugin, opened from a floating button in the running app.
It is active only when:

- `vite` is running in dev mode (`command === "serve"`), AND
- `enabled !== false`, AND
- the user is authenticated (the button only shows after auth).

It is excluded from production builds.

## Setup

Add the plugin in `vite.config.ts`:

```typescript
import { primitiveDevTools } from "primitive-app/vite";

export default defineConfig({
  plugins: [
    vue(),
    primitiveDevTools({
      appName: "My App",                       // Shown in header
      testsDir: "src/tests",                   // default
      testPattern: "**/*.primitive-test.ts",   // default
      enabled: true,                           // default true in dev
      keyboardShortcut: "cmd+shift+b",         // optional toggle
    }),
  ],
});
```

The plugin is loaded at Vite config time (Node), so `primitiveDevTools` MUST
be imported from `"primitive-app/vite"`, NOT from `"primitive-app"`.

```typescript
// WRONG — this entry point is browser-only and will fail at config time
import { primitiveDevTools } from "primitive-app";

// CORRECT
import { primitiveDevTools } from "primitive-app/vite";
```

## Opening DevTools

1. Wait for authentication (the floating Primitive-logo button only appears
   when `client.isAuthenticated()` is true).
2. The button is by default at the bottom-right corner; users can drag it to
   any edge. Position is saved in localStorage. Short click opens; drag moves.
3. Opening pushes `#devtools` to the URL hash. Refreshing with `#devtools`
   re-opens automatically. Browser back/forward open/close.
4. Close: click the X in the header, or press Escape.
5. Optional `keyboardShortcut` toggles the overlay.

The overlay is a DOM element teleported to `<body>`, NOT a route. It covers
the current page; the underlying app keeps running.

The active tab (`documents` / `tests` / `blobs`) is persisted to localStorage
under `primitive-devtools-active-tab`.

### Browser automation flow

```
1. Navigate to any page in the dev-mode app
2. Wait for auth → floating button appears
3. Click the floating button (avoid drag — keep movement under 3px)
4. Click a sidebar icon to switch tabs:
   - Document icon → Document Explorer
   - Clipboard/checklist icon → Test Harness
   - Cloud-upload icon → Blob Explorer
```

---

## Document Explorer

Three-panel unified view (no "index → model card" drilldown):

- **Left panel** — Document sidebar: search, "Add Document" button,
  document list, pending invitations, "Mass Delete Owned" action.
  Each row shows the document title plus a badge: `root` (amber) or
  permission label `Owner` / `Editor` / `Viewer`.
- **Middle panel** — Records table. Header has a "{ModelName} model"
  dropdown (switch model), an info button (toggles a Schema /
  Relationships / Methods / Indexes / Unique Constraints panel), and a
  "New {ModelName}" button (write users only). Below the header is a
  Filter Bar (filter chips + "Add filter" button), the records table,
  and pagination.
- **Right panel** — Document properties (only ≥ lg breakpoint by default,
  collapsible). Shows title, document ID (click to copy), created date,
  permission, tags, aliases, sharing, and a **Models** summary listing each
  model with `>0` records as a clickable name plus record count. Clicking
  a model name navigates the middle panel to that model.

### Common tasks

#### Verify a record exists

```
1. Sidebar → click target document
2. Middle panel header → click the "{ModelName} model" dropdown → choose model
   (or click the model name in the right-panel Models summary)
3. Use the Filter Bar to narrow down (see "Filtering")
4. Read the table rows
```

#### Verify record count

```
- Right panel → "Models" section shows "{count} records" per model
- Or middle panel → pagination footer shows total
- Or model selector dropdown → each entry shows its count
```

#### Verify field values

```
1. Navigate to model view (above)
2. Read cell values directly, OR
3. Click the row Edit (pencil) icon for full values in the edit dialog
```

#### Filtering

The Filter Bar sits below the model header. There is no per-column filter
icon.

```
1. Click "Add filter"
2. Field dropdown: pick the field
3. Operator dropdown: equals / not equals / is null / is not null /
   contains / not contains / starts with / ends with / in list / not in list /
   > / >= / < / <=
4. Value input (hidden for is_null/is_not_null; boolean shows a select;
   number uses a number input; "in list"/"not in list" take comma-separated)
5. Click "Add" (or "Save" when editing an existing chip)
6. Click an existing filter chip to edit it; click its X to remove;
   "Clear all" clears the bar
```

Operators offered are filtered by field type:

- `string` → all
- `id` → equals/not_equals/is_null/is_not_null/in/not_in
- `number` → equality, list, gt/gte/lt/lte
- `boolean` → equality + null only
- `date` → equality + null + comparison (no text ops)
- `stringset` → equality + null + contains/not_contains + in/not_in

Filters are persisted per model in localStorage.

#### Sorting

Single-field only, via column header click. Sorting is **only allowed on
indexed fields** — clicking a non-indexed column header does nothing.

```
- 1st click on column header → ascending
- 2nd click → descending
- 3rd click → cleared (default id sort)
```

There is no separate "Sort tab" or multi-field sort UI.

#### Inspect schema / methods / indexes

```
1. Open a model in the middle panel
2. Click the info icon next to the model dropdown (title: "Show model properties")
3. Expandable panel reveals: Schema fields + types, Relationships,
   Instance/Static methods + getters, Indexes, Unique Constraints
```

#### Verify a record was deleted

```
1. Navigate to the model
2. Filter by id (operator "equals", value = the deleted ID), OR scroll
3. Confirm zero results / count decreased
```

### Element-identification cheat sheet

| Element | How to find it |
|---|---|
| DevTools floating button | `<button class="pdt-btn">` containing the Primitive logo image, fixed-position to one screen edge |
| Tab icons | Left icon bar in overlay; `title="Document Explorer" | "Test Harness" | "Blob Explorer"` |
| Document row | Left panel; text = title; trailing badge = `root` or `Owner`/`Editor`/`Viewer` |
| Add Document | Button in left panel header (write-permitted users only) |
| Model selector | Middle-panel header button: text "{ModelName} model ▾" |
| Schema/info toggle | Icon button next to model selector; `title="Show model properties"` |
| New Record | Primary button in middle-panel header right side; text "New {ModelName}" (write users) |
| Filter Bar | Strip below model header; "Add filter" button + chip list |
| Row edit / delete | Per-row icons in the data table (pencil / trash) |
| Bulk delete records | Selection action bar replaces model header when rows are checked; shows "{N} selected", Clear, Delete |
| Pagination | Footer of middle panel |
| Right-panel collapse | Arrow on the panel edge (≥ lg breakpoint only) |
| Dropdowns/modals | Tagged with `data-devtools-dropdown` / `data-devtools-modal` so Escape lets them handle the key first |

---

## Test Harness

Registered `.primitive-test.ts` test groups run in two contexts: interactively
in this panel, in the same authenticated session as the host app, and headlessly
in Node under vitest (see "Headless runs (vitest / CI)" below). Key
invariants (both contexts unless noted):

1. **Explicit document lifecycle.** Tests that need database/model operations
   call `createTestDocument()` / `destroyTestDocument()` themselves.
   `createTestDocument()` creates a local-only document titled
   `===TEST=== {timestamp}-{random}`, opens it with
   `enableNetworkSync: false`, and calls `client.setDefaultDocumentId(docId)`.
   Model operations like `.save()`, `.find()`, `.query()` use it implicitly.
   Pass `createTestDocument({ networkSync: true })` for a **server-resident**
   document — required for server-side operations (blob upload, collection
   membership) that return `404 Document not found` against a local-only doc.
2. **Pure-logic tests skip documents.** Tests that only use in-memory model
   instances, utility functions, or validation logic don't need a test
   document — they just use the simple `(log) => ...` signature.
3. **Document cleanup via try/finally.** `destroyTestDocument()` cleans up the
   document: a local-only doc is closed and `evict()`ed (no server round-trip);
   a `networkSync: true` doc is deleted server-side
   (`documents.delete(id, { forceCloseIfOpen: true })`) so test docs don't
   accumulate. Always call it in a `finally` block.
4. **Scope queries to the test document.** `createTestDocument()` sets its
   document as the default, but a model query spans every open document — an
   unscoped `Task.query({ ... })` can pick up records from other documents
   open in the session. Pass `{ documents: doc.docId }` (a single id or an
   array) so the assertion sees only its own data:
   ```ts
   const highPriority = await Task.query({ priority: 2 }, { documents: doc.docId });
   ```
5. **Host-app docs are quiesced.** `beginTestRun()` closes every currently
   open host document; `endTestRun()` re-opens them after the run.
6. **Leftover cleanup.** `deleteAllTestDocuments()` runs on app start, when
   the overlay opens, and at the start of each run — anything titled with
   `===TEST===` is evicted.
7. **Timeouts differ by context.** The browser panel never kills a test — it
   can hang indefinitely. The headless adapter times each test out at 60s
   (`testTimeoutMs`).

### UI

**Left panel — selection:**
- Master "Available Tests" checkbox + "{N} selected" / Clear / Run bulk
  action bar (overlays header when any test is selected).
- All tests are auto-selected whenever the test list changes.
- Group rows: checkbox (full/partial/empty), group name, "{selected}/{total}".
- Test rows: checkbox, status icon (spinning clock = running, green check =
  passed, red X = failed, blue % circle = scored), test name, result badge
  ("Passed" / "Failed" / "{passed}/{total} ({pct}%)").
- Click a test name to focus the output log on that test (other lines dim
  to 20% opacity); click again to unfocus.
- Footer: "{total} tests in {N} groups" + green "{passed} passed" / red
  "{failed} failed" once results exist.

**Right panel — output:**
- Buttons: **Copy** (all output), **Copy Failing Tests** (lines whose
  `testId` matches a failed test), **Clear** (output + results + focus).
- Monospace lines: `HH:MM:SS AM/PM - <message>`. Per-test `log()`
  output is prefixed with `[<test-id>]`.
- During a run: progress bar at the bottom: "Running: {group} :: {name}"
  and "{completed}/{total}".

### Running tests

```
1. Open Test Harness tab
2. (All tests are pre-selected.) Adjust with checkboxes if needed.
3. Click "Run" in the bulk action bar (only visible when ≥1 test selected)
4. Wait for status icons to stop spinning and progress bar to reach total
5. Read result badges + footer summary
6. For failures: click the test name to focus its log lines, or use
   "Copy Failing Tests" to copy them for analysis
```

### Writing tests

Files: `src/tests/**/*.primitive-test.ts` (matches the configured
`testsDir` + `testPattern`). Default and named array exports both work;
the plugin's virtual module spreads arrays and wraps single groups.

The `run` signature is `(log: TestLogFn) => Promise<string>`. `log` is
always provided — no optional chaining needed.

**Pure-logic test** (no document needed):

```typescript
import type { TestGroup } from "primitive-app";

const myTests: TestGroup = {
  name: "My Feature",                    // shown as group header
  tests: [
    {
      id: "validate-email",             // MUST be globally unique
      name: "Email validation",         // shown in test row
      run: async (log) => {
        log("testing email format...");
        if (!isValidEmail("a@b.com")) throw new Error("rejected valid email");
        if (isValidEmail("not-an-email")) throw new Error("accepted invalid");
        return "ok";                     // PASS
      },
    },
  ],
};

export default myTests;
// or: export const myTests = [...]   // arrays are spread
```

**CRUD test** (needs a document for `.save()`, `.find()`, `.query()`, `.delete()`):

```typescript
import { createTestDocument, destroyTestDocument } from "primitive-app";
import type { TestGroup } from "primitive-app";
import { Task } from "@/models/Task";

const myTests: TestGroup = {
  name: "Task CRUD",
  tests: [
    {
      id: "task-create-find",
      name: "Task: create and find",
      run: async (log) => {
        const doc = await createTestDocument();
        try {
          const task = new Task({ title: "Hello" });
          await task.save();               // implicitly uses the test document

          const found = await Task.find(task.id as string);
          if (!found) throw new Error("not found");
          if (found.title !== "Hello")
            throw new Error(`title=${found.title}`);

          log("task created and found");
          return "ok";                     // PASS
        } finally {
          await destroyTestDocument(doc);
        }
      },
    },
  ],
};

export default myTests;
```

| Outcome | Produce by | UI shows |
|---|---|---|
| Pass | Return any string | Green "Passed" + the string |
| Fail | Throw an `Error` | Red "Failed" + `error.message` |
| Scored | Return a string starting with `N/M (P%)` (e.g. `"3/5 (60%)"`) | Blue score badge with the parsed numbers |

The score regex is `/^(\d+)\/(\d+)\s*\((\d+(?:\.\d+)?)%\)/` — it MUST
appear at the start of the returned string.

### Common test mistakes

```typescript
// WRONG — test signature mismatched: missing async / wrong return type
const bad: TestGroup = {
  name: "Bad",
  tests: [
    {
      id: "x",
      name: "x",
      run: (log) => "synchronous return"  // run MUST return Promise<string>
    } as any
  ]
};

// WRONG — duplicate id across files: only one will appear in the runner
{ id: "test-1", ... }   // in tests/a.primitive-test.ts
{ id: "test-1", ... }   // in tests/b.primitive-test.ts  — CONFLICT

// WRONG — no error means PASS even if the assertion is missing
run: async (log) => {
  const doc = await createTestDocument();
  try {
    const found = await Task.find("nope");
    // forgot to throw if !found
    return "ok";
  } finally {
    await destroyTestDocument(doc);
  }
}

// WRONG — missing finally block: document leaks if test throws
run: async (log) => {
  const doc = await createTestDocument();
  await new Task({ title: "x" }).save();
  await destroyTestDocument(doc);   // never reached if save() throws
  return "ok";
}

// WRONG — the file extension matters. Vite plugin discovers
// `**/*.primitive-test.ts` only. `.test.ts` will be ignored.
//   src/tests/foo.test.ts          ← NOT discovered
//   src/tests/foo.primitive-test.ts ← discovered

// WRONG — using createTestDocument when no database ops are needed
// (wastes time creating/destroying a document for nothing)
run: async (log) => {
  const doc = await createTestDocument();  // unnecessary
  try {
    if (2 + 2 !== 4) throw new Error("math broken");
    return "ok";
  } finally {
    await destroyTestDocument(doc);
  }
}
// RIGHT — pure logic, no document needed
run: async (log) => {
  if (2 + 2 !== 4) throw new Error("math broken");
  return "ok";
}
```

### Scored test example

```typescript
{
  id: "compat",
  name: "compatibility checks",
  run: async (log) => {
    const checks = [check1(), check2(), check3(), check4(), check5()];
    const passed = checks.filter(Boolean).length;
    const total = checks.length;
    const pct = ((passed / total) * 100).toFixed(1);
    log(`${passed}/${total} passed`);
    return `${passed}/${total} (${pct}%)`;   // blue score badge
  },
}
```

### Environment-scoped tests

A test or a whole group can restrict where it runs:

```typescript
export interface TestGroup {
  name: string;
  environment?: "browser" | "node";    // restrict every test in the group (default: both)
  tests: Array<{
    id: string;
    name: string;
    environment?: "browser" | "node";  // per-test; overrides the group setting
    run: (log: TestLogFn) => Promise<string>;
  }>;
}
```

Resolution is `test.environment ?? group.environment`; unset runs in both
contexts. `"browser"` tests are `it.skip`ped by the vitest adapter (reported
skipped, never fail CI) — use for DOM/canvas/`MediaRecorder` dependencies.
`"node"` tests show as skipped in the browser panel (grey skip icon and a
skipped count in the summary).

### Headless runs (vitest / CI)

The same registered groups run in Node under `vitest run` — no browser or
headless-browser stack, so `pnpm test` can gate merges. The template ships the
wiring; the pieces, all load-bearing:

- `src/tests/primitive-tests.spec.ts` — the single vitest spec that adapts
  every registered group:

  ```typescript
  import { registerPrimitiveTests } from "primitive-app/testing";
  import { allModels } from "@/models";

  await registerPrimitiveTests({
    models: allModels,
    testModules: import.meta.glob("./**/*.primitive-test.ts"),
    appId: import.meta.env.VITE_APP_ID,
    apiUrl: import.meta.env.VITE_API_URL,
    wsUrl: import.meta.env.VITE_WS_URL,
  });
  ```

- `vitest.config.ts` — merges the app's Vite config (so `@` aliases and
  codegen resolve exactly as in the browser) and sets
  `resolve.dedupe: ["js-bao", "js-bao-wss-client", "yjs"]` (prevents model
  base-class split-brain across module copies),
  `test.server.deps.inline: ["primitive-app", "js-bao-wss-client"]`
  (extension-less ESM dist), `testTimeout: 60_000`, `hookTimeout: 120_000`.
- `vite.config.ts` applies the browser `ws` stub alias only when
  `!process.env.VITEST`.
- Dev dependencies `vitest` and `ws`. In Node the client dynamically imports
  `ws` when no global `WebSocket` exists; without it the connection fails with
  `Cannot find package 'ws'`.
- Script: `"test": "pnpm codegen && vitest run"`.

`registerPrimitiveTests(options)` — from `primitive-app/testing`:

| Option | Default | Notes |
|---|---|---|
| `models` | required | The app's model classes, typically `allModels` from `@/models` |
| `testModules` | required | `import.meta.glob("./**/*.primitive-test.ts")`; each module's default export must be a `TestGroup` or `TestGroup[]` |
| `appId` / `apiUrl` / `wsUrl` | `process.env.VITE_APP_ID` / `VITE_API_URL` / `VITE_WS_URL` | |
| `email` | `process.env.PRIMITIVE_TEST_EMAIL` | Must derive from a whitelisted test-account base (`primitive apps update <app-id> --test-account-bases "..."`). Use a stable suffix per CI project so the find-or-create provisioner reuses one test user across runs |
| `otpCode` | `"000000"` (`PRIMITIVE_TEST_OTP_CODE`) | The test-account OTP bypass code |
| `testTimeoutMs` | `60_000` | Per-test timeout |
| `clientOptions` | storage `{ type: "auto" }` | Partial client options; `auto` storage uses better-sqlite3 if installed, memory otherwise — no native deps required |
| `cleanupTestDocuments` | `true` | Deletes leftover `===TEST===` documents after the run |
| `failBelowFullScore` | `true` | A scored result below full marks (`"7/10 (70%)"`) FAILS the run; the panel shows the same result as scored without failing |

Auth and failure semantics:

- Sign-in is the normal test-account flow: a non-whitelisted email rejects
  with a clear auth error (never hangs); invite-only / domain-restricted /
  waitlist apps reject provisioning the same way.
- The bypass token lives ~30 minutes — split or shard suites that run longer.
- A test file that fails to load surfaces as a failing test, never a silent
  skip.
- JUnit output for CI ingestion:
  `pnpm vitest run --reporter=junit --outputFile=test-results.xml`.

### Element-identification cheat sheet

| Element | How to find it |
|---|---|
| Test Harness tab | Sidebar icon, `title="Test Harness"` |
| Master checkbox | First checkbox in left panel header (adjacent to "Available Tests") |
| Bulk action bar | Overlays the left-panel header when ≥1 test selected; contains "{N} selected", Clear, Run |
| Run button | Inside the bulk action bar; play icon + text "Run" |
| Group row | Has the group name, a checkbox, and "{selected}/{total}" |
| Test row | Has a checkbox, a status icon, the test name, and (after run) a result badge |
| Output log line | Selector `[data-test-id="<test-id>"]` per per-test line |
| Copy / Copy Failing Tests / Clear | Top-right of the right panel |
| Progress bar | Bottom of right panel during a run |

---

## Blob Explorer

Three-panel layout sharing the document sidebar with Document Explorer.

- **Left** — Document sidebar (selection state shared with Document Explorer).
- **Middle** — Toolbar (document title, blob ID search, Refresh, Upload),
  paginated table (Filename / Content Type / Size / Uploaded / Blob ID),
  per-row hover Download/Delete, checkbox column for bulk select (write
  users only), and a bulk action bar showing selected count + Delete.
- **Right** — Collapsible blob detail (≥ lg breakpoint by default):
  Preview (images / PDFs / short text/CSV), Actions (Download, Open in
  new tab, Delete), Info (Blob ID, Filename, Content Type, Size,
  Uploaded date, SHA-256). IDs are click-to-copy.

Search mode: typing in the blob-ID search switches to exact server-side
lookup and hides pagination. Pagination size (10/25/50/100) is persisted
in localStorage. Upload requires write access. Bulk-delete requires write
access.

### Element-identification cheat sheet

| Element | How to find it |
|---|---|
| Blob Explorer tab | Sidebar icon, `title="Blob Explorer"` |
| Blob ID search | Text input in middle-panel toolbar |
| Upload | "Upload" button in toolbar (write users) |
| Refresh | Circular-arrow icon in toolbar |
| Row download / delete | Hover actions on each row |
| Bulk action bar | Replaces toolbar when rows are checked |
| Detail panel | Right edge; collapse arrow on its left edge |

---

## Browser-console debugging

When the host runs on a debug origin (`localhost`, `127.0.0.1`, `[::1]`,
`*.localhost`, `*.local`), the client and registered models are exposed
on `window`:

```javascript
// Models registered via the JsBaoClientService config (PascalCase keys)
Object.keys(window.__primitiveAppModels)
// → ["Task", "Product", ...]

await window.__primitiveAppModels.Task.query({})
await window.__primitiveAppModels.Task.query({ completed: false })
await window.__primitiveAppModels.Task.query({ priority: { $gte: 3 } })
await window.__primitiveAppModels.Task.find("some-id")
await window.__primitiveAppModels.Task.count({ completed: false })

// New record needs an explicit targetDocument when no default is set
const t = new window.__primitiveAppModels.Task({ title: "x" });
await t.save({ targetDocument: "some-doc-id" });

// Underlying client (auth, documents, etc.)
await window.__primitiveAppClient.me.get();
await window.__primitiveAppClient.me.ownedDocuments();
```

Not exposed in production builds or non-debug hostnames.

---

## Browser automation tips

- Wait for the floating button before doing anything else — it's gated
  on auth and on host-app mount (a `MutationObserver` watches `#app`).
- The first overlay open runs `deleteAllTestDocuments()` (a syncMetadata
  + list + evict-each pass). On a slow connection this can take a few
  seconds — wait for the spinner labelled "Initializing dev tools..."
  to clear before interacting.
- If you see "Initialization Failed", click "Retry". This usually means
  leftover-test-doc cleanup hit an error.
- Validation workflow: act in the host app → open Document Explorer →
  pick the document → switch model via dropdown → filter → read.
- Test workflow: Test Harness tab → adjust selection → Run → wait for
  spinners to stop and progress to reach total → read badges → use
  "Copy Failing Tests" to capture failures.
- Don't try to drag the floating button — keep pointer movement under 3px
  during click, otherwise it repositions instead of opening.
- Escape behavior is layered: nested `[role=dialog][data-state=open]`,
  `[data-devtools-modal]`, `[data-devtools-dropdown]`, and focused
  inputs/textareas/selects all consume Escape before the overlay does.
{{/lang}}

{{#lang swift}}
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
{{/lang}}
