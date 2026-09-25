# Agent Guide to Primitive Inspection

Guidelines for AI agents inspecting a running Primitive app from the CLI — reading what happened (server function runs and invocation logs, live connections, sessions, blobs, records, metadata) without opening the Admin Console. The inspection commands share one set of conventions so they behave the same across resources.

## The inspection surface

```bash
# Server functions
primitive functions list                                  # functions + status (app-scoped)
primitive functions get <function-id>                     # active version, capabilities, manifest, triggers
primitive functions configs <function-id>                 # every pushed version, newest first
primitive functions runs <function-id>                    # run rows: task starts, cron fires, webhook deliveries
primitive functions runs steps <function-id> <run-id>     # one task run's step trace
primitive functions runs wait <function-id> <run-id>      # poll a run until it settles
primitive functions logs <function-id>                    # invocation records: output, error, stack
primitive functions logs <function-id> --run <run-id>     # one run: step trace, then its records oldest first
primitive functions logs <function-id> --invocation <id>  # one record, by the id an invoke answered with
primitive functions logs <function-id> --follow           # tail new invocations

# `runs steps` works on an in-flight run: finished steps so far plus a
# `running` row for the step executing now (a sleeping run's sleep step
# stays `running` until it wakes).

# The other log views
primitive integrations logs <integration-id>           # outbound calls: status, timing, actor
primitive analytics events                             # app activity events

# Blob storage
primitive blob-buckets list                            # buckets in the app (app-scoped)
primitive blob-buckets head <bucket> <key>             # object metadata without downloading

# Live connections and sessions
primitive connections list --user-id <id>              # active WebSocket connections
primitive sessions list --user-id <id>                 # auth sessions

# Records and documents
primitive databases list --owner <user-id>             # databases one user created
primitive databases records query <database> ...       # read database records
primitive databases records get <database> <model-name> <record-id>
primitive databases records count <database> <model-name> [--filter '{...}']
primitive databases records aggregate <database> <model-name> --op <count|sum|avg|min|max>
primitive documents records query <document> <model-name> [--filter '{...}']
primitive documents records get <document> <model-name> <record-id>
primitive documents records count <document> <model-name> [--filter '{...}']
primitive documents records aggregate <document> <model-name> --op <count|sum|avg|min|max>
primitive documents dump <document-id>                 # every model's records as JSON
primitive documents export <document-id>               # dump a document's contents

# Metadata
primitive metadata get <type> <id> <category>          # resource metadata
```

## Uniform flags

Every inspection command honors the same read flags:

- `--app <id>` — target app; falls back to the resolved environment's app.
- `--json` — the output you parse. Most commands print the endpoint payload as-is; the log views below normalize theirs into the shared item shape described in the next section. It is always one JSON document, and never a bare array. `--json` goes to stdout; status, warnings, progress, and the `CLI Version: …` banner all go to stderr, so a redirected stdout stays a single parseable document. That holds for the always-JSON commands too — `primitive documents dump <doc> | jq .` parses without a `--json` flag.
- **The list envelope.** Every `list` verb prints the same envelope under `--json` — every noun, with no exceptions, the type-config readers included: `{ items, hasMore, nextCursor? }`, where `items` holds the rows, `hasMore` says whether the listing continues past them, and `nextCursor` is present only when it does. So `jq '.items[]'` reads any noun's listing, and an empty result is `{ "items": [], "hasMore": false }` rather than nothing or `[]`. A `list` whose subject carries something that is not a row — `env list`'s current selection, `guides list`'s resolved docs version, `metadata list`'s resource — prints that as extra keys BESIDE those three, which always mean the same thing.
- `--limit <n>` / `--cursor <c>` — paged reads. Pagination follows the ROUTE, not the verb: a `list` whose route pages declares both flags, prints exactly one page, and reports that page's `hasMore`/`nextCursor` — pass the cursor back as `--cursor` to get the next one. It never walks the chain on your behalf, so a script that means "all of them" follows `nextCursor` itself. A `list` whose route does not page declares neither flag and prints the whole set with `hasMore: false`. `primitive help --json` says which flags a given verb carries. `functions logs` and both `records query` verbs print the same envelope whatever shape the underlying endpoint returns; aggregate reads, which are not `list` verbs, walk the `nextCursor` chain to the end.

`list` always requires a **selector** — `--user-id`, `--owner`, a resource id — so it never enumerates the whole app. `--user-id` is the spelling on every list and inspection selector. The exception to the selector rule is a genuinely app-scoped resource such as `blob-buckets list` or `functions list`, which list the app's buckets and functions directly.

Permission sub-verbs differ by resource on purpose: documents use `permissions grant`/`revoke` (a reader/read-write/owner ladder), databases use `permissions add-manager`/`remove-manager` (a manager/owner ladder). What is uniform is `permissions list` and group nesting — not the mutation verb names.

## The log views and their shared item shape

Three views read "what happened" in the shared shape: `functions logs`, `integrations logs`, `analytics events`. Under `--json` they emit the same item shape inside the view's envelope — never a bare array:

```json
{
  "items": [
    {
      "source": "function-log",
      "timestamp": "2026-07-24T18:03:11.204Z",
      "outcome": "error",
      "nativeStatus": "failed",
      "durationMs": 412,
      "correlation": { "eventId": "01J…", "runId": "01J…", "userId": "01J…" },
      "detail": { "functionKey": "summarize", "triggerKind": "cron", "runtime": "task", "errorCode": "…", "errorMessage": "…" }
    }
  ],
  "hasMore": false
}
```

| Field | Meaning |
|---|---|
| `source` | Discriminator: `function-log`, `integration`, `activity`. |
| `timestamp` | ISO-8601 event time, or `null` when the record carries none. |
| `outcome` | Normalized verdict: `ok`, `error`, `pending`, `neutral`. |
| `nativeStatus` | The source's own status, verbatim — HTTP integer (integration), `completed`/`failed`/`timeout`/`running` or a platform refusal code (function log), `null` for an activity row other than `function.invoke`. |
| `correlation` | Pivot keys — including the row's own id, so a printed row can always be looked up again. Only the keys a source records are present. |
| `detail` | Per-source allowlist of operator-facing fields — a projection, not the stored record. |

Outcome mapping, by source:

| Source | `ok` | `error` | `pending` | `neutral` |
|---|---|---|---|---|
| Integration | HTTP < 400 | HTTP ≥ 400 | — | — |
| Function log | `completed` | `failed`, `timeout`, a platform refusal code (`FUNCTION_BUNDLE_MISSING`, …) | `running` (a task slice that has printed and settled nothing) | anything else, and an absent status |
| Activity | `function.invoke` with `status: "completed"` | `function.invoke` with `status: "failed"` or `"timeout"` | `function.invoke` with `status: "started"` | every other action, always |

Activity events are `neutral` with ONE exception. They are counts, and inventing a verdict for a count would be fabricating a signal — but `function.invoke` carries the settled status in its own event context, so a failed invocation is reported as `error`, which is what makes failures countable through this contract. `started` is `pending`: a task start is in flight when its row is written, and its terminal outcome lives on the run row (`functions runs`). The context may arrive as an object or as a JSON string; an unparseable one stays `neutral` rather than being guessed at.

The `function-log` source is a server function's invocation records. Its `detail` carries `functionKey`, `configId`, `contentHash`, `triggerKind` (`http`, `webhook`, `cron`, `function`, `manual`), `runtime` (`request` or `task`), `errorCode`, `errorMessage`, `errorStack`, `stdout`, `stderr`, `truncated`, `logsUnavailable` and `contentSuppressed`; `stdout` and `stderr` are arrays of `{ t, s, line }` entries, where `t` is milliseconds after the capture began and `s` is `out` or `err`. Its `correlation` carries the record's own `eventId` (the invocation id), the `runId` of a trigger fire or task run, and the attributed `userId`. Records are kept seven days and outlive an archived function.

Pagination per view: `functions logs` returns `{ items, hasMore, nextCursor? }` and takes `--limit` (default 25, max 100) / `--cursor`; `integrations logs` returns `{ items }` and takes `--limit` plus `--status`/`--from`/`--to` (it filters inside a bounded scan rather than paging); `analytics events` returns `{ items, page, pageSize, totalRows }` and takes `--page`/`--window-days`/`--user-id`.

Not in the shared shape: `functions runs --json` prints the run rows as `{ items, nextCursor }` (each row adds `runtime`), and `functions runs steps --json` prints one run's full trace under `items`, never paged.

The normalization is `--json`-only. The human tables stay per-view, because each carries columns the shared shape has no room for.

Invalid filter values are rejected, not ignored: an unparseable `--from`/`--to`, a non-positive `--limit`, or a malformed `--cursor` fails with the server's validation message.

## Triaging a failing function

1. `functions logs <function-id>` — newest first; the table is `TIME | STATUS | TRIGGER | RUNTIME | VERSION | RUN/INVOCATION ID | CODE | ERROR`, where ERROR is the first line of the error (or of stderr). `--json` carries the whole message, the stack, and every printed line.
2. `functions logs <function-id> --invocation <id>` — one record in full. A record that never existed, belongs to another function, or aged past seven days all answer the same 404.
3. For a task run: `functions logs <function-id> --run <run-id>` prints the step trace, then each record the run wrote, oldest first, with its lines under it. A run that slept writes one record per slice that settled; output printed before a hibernation is not recoverable.
4. `functions runs <function-id>` — the run level: `RUN ID | STATUS | FIRED BY | RUNTIME | VERSION | PARENT | REFRESHES | RESETS | STARTED | ENDED | CODE`. Every cron fire and every webhook delivery writes a run row, so this is where a schedule's or a provider's effect shows. A run that RESET (a platform deploy tore a slice down and the engine replayed the step) and then completed carries no error — the RESETS column is the only place it shows.
5. `functions get <function-id>` — the triggers as the platform holds them: the webhook's id, URL, scheme, status and last delivery; each cron entry's name, schedule, timezone, status, next fire, fire count and last run.

`--invocation` cannot be combined with `--run`, `--follow`, `--cursor` or `--limit`. A filtered `--run` page that holds only other runs' records is followed a bounded number of times; if it is still empty the command prints the `--cursor` to continue rather than a bare "none".

A gate refusal (403 access, 404 unknown key, 400 input schema, 429 rate) writes no record — it never reached the code; debug it from the HTTP response.

## Reading one user's activity

```bash
primitive analytics events --user-id <user-id>   # that user's activity events, function.invoke included
```

An invocation's record carries the attributed user as `correlation.userId`, so a `function.invoke` row in the user's events pivots to `functions logs` by function and time. Work with no human behind it is recorded against the app's own principal `sys:<appId>`, which the analytics surfaces display as `System`; `analytics events --user-id sys:<appId>` reads exactly that activity.

## `--follow`

`functions logs <function-id> --follow` tails: it appends invocations as their records are written, like `tail -f`. Its first poll establishes a baseline and prints nothing for rows that predate the invocation ("show me what happens from now").

```bash
primitive functions logs <function-id> --follow
primitive functions logs <function-id> --follow --interval 5   # poll every 5s
```

Rules:

- `--interval <seconds>` — a positive number, default 2. There is no server push; the CLI polls.
- `--follow` cannot be combined with `--cursor` (a tail starts from now), `--run` (a run's records are a bounded set; a tail walks the whole index), or `--invocation`.
- `--json --follow` emits **NDJSON** — one shared-shape item per line. Pipe it to `jq -c` and read line by line.
- Ctrl-C stops a tail cleanly.

What it guarantees: an invocation id is minted at START and its record written at SETTLE, so a slow invocation lands below rows already printed. The tail looks back a minute past its high-water mark (the 30 s request ceiling plus the token's grace) and remembers which ids in that window it has shown, so each record prints once and the slow, failed and timed-out ones are not skipped. A burst larger than one page between polls is walked page by page back to the mark rather than skipped.
