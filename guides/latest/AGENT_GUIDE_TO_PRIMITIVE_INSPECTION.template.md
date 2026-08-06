# Agent Guide to Primitive Inspection

Guidelines for AI agents inspecting a running Primitive app from the CLI — reading what happened (workflow runs, live connections, sessions, blobs, records, metadata) without opening the Admin Console. The inspection commands share one set of conventions so they behave the same across resources, and two tailing modes — `--watch` (snapshot) and `--follow` (tail) — layer on top of any time-ordered list.

## The inspection surface

```bash
# Workflow runs (the reference tailing command)
primitive workflows runs list <workflow-id>            # recent runs
primitive workflows runs list --user-id <user-id>      # one user's runs, across every workflow
primitive workflows runs steps <workflow-id> <run-id>  # every step run of one run
primitive workflows runs status <workflow-id> <run-id> # one run's status + step results
primitive workflows runs failures <workflow-id>        # failed runs only, with cause and failed step

# The other log views
primitive integrations logs <integration-id>           # outbound calls: status, timing, actor
primitive webhooks events <webhook-id>                 # inbound deliveries and how they were handled
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
primitive documents dump <document-id>                 # every model's records as JSON
primitive documents export <document-id>               # dump a document's contents

# Metadata
primitive metadata get <type> <id> <category>          # resource metadata
```

## Uniform flags

Every inspection command honors the same read flags:

- `--app <id>` — target app; falls back to the resolved environment's app.
- `--json` — the output you parse. Most commands print the endpoint payload as-is; the log views below normalize theirs into the shared item shape described in the next section. It is always a JSON document, never a bare array. `--json` goes to stdout; status, warnings, progress, and the `CLI Version: …` banner all go to stderr, so a redirected stdout stays a single parseable document. That holds for the always-JSON commands too — `primitive documents dump <doc> | jq .` parses without a `--json` flag.
- `--limit <n>` / `--cursor <c>` — paged reads. The response envelope is always `{ items, hasMore, nextCursor? }` — read `nextCursor` and pass it back as `--cursor`. Both `records query` verbs print exactly that envelope, whatever shape the underlying endpoint returns, and neither emits the deprecated `cursor` alias; the paginated log views (workflow runs, webhook events) still add `cursor` as a deprecated alias of `nextCursor`, kept for one window. Aggregate reads walk the `nextCursor` chain to the end.

`list` always requires a **selector** — `--user-id`, `--owner`, a resource id — so it never enumerates the whole app. `--user-id` is the spelling on every list and inspection selector; `connections list`, `sessions list` and `tokens list` still accept `--user` as a deprecated alias that prints a notice on stderr. The one exception to the selector rule is a genuinely app-scoped resource such as `blob-buckets list`, which lists the app's buckets directly.

Permission sub-verbs differ by resource on purpose: documents use `permissions grant`/`revoke` (a reader/read-write/owner ladder), databases use `permissions add-manager`/`remove-manager` (a manager/owner ladder). What is uniform is `permissions list` and group nesting — not the mutation verb names.

## The log views and their shared item shape

Six views read "what happened": `workflows runs list`, `workflows runs failures`, `workflows runs steps`, `integrations logs`, `webhooks events`, `analytics events`. Under `--json` they emit the same item shape inside the view's pagination envelope — never a bare array:

```json
{
  "items": [
    {
      "source": "workflow-run",
      "timestamp": "2026-07-24T18:03:11.204Z",
      "outcome": "error",
      "nativeStatus": "failed",
      "correlation": { "runId": "01J…", "workflowId": "01J…", "userId": "01J…" },
      "detail": { "workflowKey": "summarize", "errorMessage": "…" }
    }
  ],
  "hasMore": false
}
```

| Field | Meaning |
|---|---|
| `source` | Discriminator: `workflow-run`, `workflow-step`, `integration`, `webhook`, `activity`. |
| `timestamp` | ISO-8601 event time, or `null` when the record carries none. |
| `outcome` | Normalized verdict: `ok`, `error`, `pending`, `neutral`. |
| `nativeStatus` | The source's own status, verbatim — HTTP integer (integration), `failed`/`completed`/`terminated` (run), `skipped` (step), `duplicate`/`workflow_not_active` (webhook), `null` (activity). |
| `correlation` | Pivot keys: `runId`, `stepId`, `stepRunId`, `eventId`, `traceId`, `workflowId`, `workflowKey`, `integrationKey`, `webhookId`, `workflowRunId`, `userId` — including the row's own id, so a printed row can always be looked up again. Only the keys a source records are present. |
| `detail` | Per-source allowlist of operator-facing fields — a projection, not the stored record. |

Outcome mapping, by source:

| Source | `ok` | `error` | `pending` | `neutral` |
|---|---|---|---|---|
| Integration | HTTP < 400 | HTTP ≥ 400 | — | — |
| Workflow run | `completed` | `failed`, `terminated` | `queued`, `running`, `apply_pending`, `apply_claimed` | — |
| Workflow step | `completed` | `failed` | — | `skipped` |
| Webhook | `accepted`, `duplicate`, `handshake`, `workflow_not_active` | `rejected`, `error` | — | — |
| Activity | — | — | — | always |

`workflow_not_active` is `ok` on purpose: the delivery was accepted and deliberately not dispatched. Activity events are always `neutral` — they carry no success signal, so no `ok` is invented for them.

Pagination per view: `workflows runs list`, `workflows runs failures` and `webhooks events` return `{ items, hasMore, nextCursor? }` and take `--limit`/`--cursor`; the two run views add `scanned` (runs examined) whenever a filter is in play, since a filtered read searches the index rather than reading one page; `workflows runs steps` returns `{ items }` (a run's steps are not paged); `integrations logs` returns `{ items }` and takes `--limit` plus `--status`/`--from`/`--to`/`--source` (it filters inside a bounded scan rather than paging); `analytics events` returns `{ items, page, pageSize, totalRows }` and takes `--page`/`--window-days`.

The normalization is `--json`-only. The human tables stay per-view, because each carries columns the shared shape has no room for — a run's queue delay, a step's inter-step gap and token counts, a webhook event's id. `--watch --json` reprints the same envelope each tick; `--follow --json` emits one item per line (newline-delimited JSON), since a tail has no closing bracket to wait for.

Invalid filter values are rejected, not ignored: an unparseable `--from`/`--to`, a non-positive `--limit`, an unknown `--source`, or a malformed `--cursor` fails with the server's validation message. A filter a view cannot serve (`webhooks events --from`, for instance) fails with a message listing the flags that view supports.

## Triaging failures

`workflows runs failures <workflow-id>` is the grouping view: failed runs only, each row naming the cause and the step that produced it, so you can tell one repeated bug from several distinct ones without opening each run.

```bash
primitive workflows runs failures <workflow-id>
primitive workflows runs failures <workflow-id> --json | jq -r '.items[].detail.errorTitle' | sort | uniq -c
```

The table is `RUN ID | STEP | ERROR | STARTED | ENDED`. `ERROR` is the run's failure message, truncated to fit; `--json` carries it whole.

Under `--json` each item is a `workflow-run` item whose `detail` carries the failure fields:

| Field | Meaning |
|---|---|
| `errorMessage` | The failure message, verbatim. |
| `errorTitle` | `errorMessage` with ids, numbers, URLs and quoted free-text values replaced by placeholder tokens; JSON keys, quoted `SCREAMING_SNAKE` error enums and a status under a `code` / `status` key are kept, so two upstream errors of the same shape stay distinct. A `Caused by:` chain folds to its **head** message, so `errorTitle` shows the failure, not the chain — the full chain stays in `errorMessage`. Runs that failed for the same reason share one title, so this is the field to group and count on — and it is the same string `primitive analytics errors-groups` titles its groups with, so a spiking group can be grepped straight back to the runs that caused it. Both are derived from the message's first 2,000 characters, so an extremely long message groups by that prefix. |
| `failedStepId` | The step that failed the run — the lowest-index step whose status is `failed`. Also in `correlation.stepId`, so it pivots to `runs steps` / `runs error`. |
| `failedStepKind` | That step's kind (`database.query`, `llm`, …). |
| `failedStepErrorTitle` | Normalized title of the step's own error, for grouping by step-level cause. |

A `-` in the STEP column is a normal outcome, not a bug: a run can fail before any step runs, be reclaimed after its executor died, or fail output-schema validation after every step completed. The failure message is still there. Runs that finished before this attribution shipped have no step either. Note the two layers spell that absence differently: the HTTP response always carries all five failure keys, explicitly `null` when there is nothing to name, while the CLI's `--json` projection drops empty keys, so `detail.failedStepId` is absent rather than `null` (`jq` treats the two the same; a JS test for `=== null` does not).

`runs error <workflow-id> <run-id>` remains the drill-down — it adds the caret-annotated expression, the step's input and config, and the full error detail, which the list deliberately never carries.

The `--status` filter on `runs list` (and `runs failures`, which is `--status failed`) searches the index rather than filtering one page, so a failure older than the most recent successes is still found.

`--limit` on `runs failures` is a number of **failures**, not a number of runs to look at: the command keeps searching back through history until it has that many failures, history runs out, or it has examined `--max-scan` runs (default 5,000; `--max-scan 0` removes the row cap). `--max-scan` is honoured to the nearest request — the count is checked after each request and one request examines up to 1,000 runs, so a small `--max-scan` still examines up to 1,000; the number in the closing line is always the number actually examined. One invocation issues at most 200 requests regardless of `--max-scan`, so a very deep search finishes across several `--cursor` continuations rather than in one command. It never reports a bare "no failures" over a search that stopped early — the closing line is one of:

```
No failures. Searched all 342 runs.
No failures in the 5000 most recent runs. More history remains — re-run with --cursor eyJ… (or raise --max-scan).
3 of 10 requested failures found in the 5000 most recent runs. More history remains — re-run with --cursor eyJ… (or raise --max-scan).
```

A sweep that stopped on the 200-request ceiling offers only `--cursor` (raising `--max-scan` cannot move that bound). A sweep started from `--cursor` counts rows from that position, so its wording is "… in N runs from this position", never "the N most recent runs" — do not read a resumed result as a statement about recent history.

Under `--json` the same fact is a number: `scanned` is the total runs examined across every page the command fetched, and `nextCursor` is present only when history remains. `jq '{found: (.items|length), scanned, more: .hasMore}'` is the check for "how much of the history does this answer cover".

```bash
primitive workflows runs failures <workflow-id> --limit 25 --max-scan 20000
primitive workflows runs failures <workflow-id> --json | jq '{found: (.items|length), scanned, more: .hasMore}'
```

`runs list --status <s>` stays a pager — its `--limit` is a page size and it does not resume itself — but its empty state reports the same way: how many runs it searched, and the `--cursor` to continue.

## Reading one user's activity

```bash
primitive workflows runs list --user-id <user-id>            # every run that user started, app-wide
primitive workflows runs list <workflow-id> --user-id <id>   # narrowed to one workflow
primitive analytics events --user-id <user-id>               # that user's activity events
```

`<workflow-id>` is optional when `--user-id` is given. The user-keyed run view reads a by-user index, so it does not offer `--follow`; use `--watch`.

`integrations logs` and `webhooks events` take no `--user-id`. An integration invocation records the acting user in `detail`/`correlation` but is indexed by integration, and a webhook event carries no user identity at all — it comes from an outside system. The recipe for those: read the user's runs first, then match `correlation.runId` or `correlation.traceId` in `integrations logs`, and `correlation.workflowRunId` in `webhooks events`.

## `--watch` vs `--follow`

Both poll on an interval — there is no server push — but they answer different questions:

- **`--watch`** re-fetches the current snapshot each interval and re-renders the whole view. It is a periodic re-`list`/re-`get` and works on any list command with no server change. Use it to keep an eye on current state (statuses updating in place).
- **`--follow`** tails: it appends new or changed rows since a server-owned checkpoint, like `tail -f`. Its first poll establishes a baseline and prints nothing for rows that predate the invocation ("show me what happens from now"). Use it to watch activity as it arrives.

```bash
primitive workflows runs list <workflow-id> --watch     # re-render the list every 2s
primitive workflows runs list <workflow-id> --follow    # append runs as they start or change
primitive workflows runs list <workflow-id> --follow --interval 5   # poll every 5s
```

Rules:

- `--interval <seconds>` sets the poll interval — minimum 1s, default 2s.
- `--watch` and `--follow` are mutually exclusive.
- `--follow` is offered **only** where the endpoint supports the resume contract (today: `workflows runs list`). Other tailing candidates offer `--watch` until their endpoint adds it; passing `--follow` where it isn't supported fails with a clear message.
- `--json --follow` emits **NDJSON** — one JSON object per new row per line. A tail is an unbounded stream, so it can't be one array; pipe it to `jq -c` and read line by line. `--json --watch` emits one array per redraw.
- Ctrl-C stops a tail cleanly and exits 0.

## What `--follow` guarantees

`--follow` shows the **latest observed version** of a row, not every state change. It re-emits a run when a newer version is observed between polls, so a run you already saw reappears at its new position after its status changes — that is expected, not a duplicate. Two transitions that happen between the same pair of polls collapse to the latest stored version, so `--follow` is not an exactly-once event log.

This is **near-lossless observed-version tailing**: it orders runs by when they last changed (`modifiedAt`), which has no guaranteed order among rows sharing a timestamp, and the underlying index is eventually consistent. So under a burst, rows sharing a timestamp or a delayed index update can occasionally be skipped or re-shown. Use `--follow` to watch activity, and `workflows runs status <workflow-id> <run-id>` for the authoritative state of one run.
