# Agent Guide to Primitive Inspection

Use the CLI to read function logs, inspect runs, and query app state. Start with the failing invocation or run, then use its IDs to find related records.

## Triaging a failing function

1. `primitive functions logs <function-id>` — find the failed invocation; add `--json` for the full error and captured output.
2. `primitive functions logs <function-id> --invocation <id>` — inspect one invocation.
3. `primitive functions logs <function-id> --run <run-id>` — inspect a task's step trace and logs.
4. `primitive functions runs <function-id>` — check task, cron, and webhook run status.
5. `primitive functions get <function-id>` — check the function's trigger configuration.

## The inspection surface

```bash
# Server functions
primitive functions list                                  # functions + status + each one's webhook and crons (app-scoped)
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
primitive integrations logs <integration-id> --run <run-id>   # just the calls one run made
primitive integrations logs <integration-id> --run <run-id> --step <name>   # one step of that run
primitive analytics events                             # app activity events

# Blob storage
primitive blob-buckets list                            # buckets in the app (app-scoped)
primitive blob-buckets head <bucket> <blob-id>         # object metadata without downloading

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
primitive documents stats <document-id>                # counts and size; `sizeBasis` says what the size measures
primitive documents export <document-id>               # dump a document's contents

# Metadata
primitive metadata get <type> <id> <category>          # resource metadata
```

## Uniform flags

| Option | Meaning |
|---|---|
| `--app <id>` | Target app; defaults to the environment's app |
| `--json` | JSON on stdout; status, warnings, and progress on stderr |
| `--limit <n>` / `--cursor <c>` | Pagination where the command supports it |

Every `list --json` result contains `items`, `hasMore`, and an optional `nextCursor`. Some listings also include resource-specific metadata. Follow the cursor while `hasMore` is true; a single call does not fetch every page. A non-paginated list returns `hasMore: false`.

Use `primitive help --json` to discover flags. Most listings need a user, owner, or resource selector; app-wide functions and buckets can be listed directly.

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
| `correlation` | IDs for looking up related records; only recorded keys are present. |
| `detail` | Fields specific to the source. |

Outcome mapping, by source:

| Source | `ok` | `error` | `pending` | `neutral` |
|---|---|---|---|---|
| Integration | HTTP < 400 | HTTP ≥ 400 | — | — |
| Function log | `completed` | `failed`, `timeout`, a platform refusal code (`FUNCTION_BUNDLE_MISSING`, …) | `running` (a task slice that has printed and settled nothing) | anything else, and an absent status |
| Activity | `function.invoke` with `status: "completed"` | `function.invoke` with `status: "failed"` or `"timeout"` | `function.invoke` with `status: "started"` | every other action, always |

### Gotchas when interpreting logs

- Logs are retained for seven days. Output printed before a task pauses is not recoverable; use step results to track progress.
- `--invocation` cannot be combined with `--run`, `--follow`, `--cursor`, or `--limit`.
- A filtered run query may return a continuation cursor with no matching rows. Follow it before concluding that no records exist.

- Activity events normally have outcome `neutral`. `function.invoke` uses its recorded status; a task start is `pending`, so check the run for completion.
- Missing or unparseable event context stays `neutral`.
- Authorization, schema, and rate-limit refusals happen before execution and write no invocation log. Inspect the HTTP response.

The `function-log` source is a server function's invocation records. Its `detail` carries `functionKey`, `configId`, `contentHash`, `triggerKind` (`http`, `webhook`, `cron`, `function`, `manual`), `runtime` (`request` or `task`), `errorCode`, `errorMessage`, `errorStack`, `stdout`, `stderr`, `truncated`, `logsUnavailable` and `contentSuppressed`; `stdout` and `stderr` are arrays of `{ t, s, line }` entries, where `t` is milliseconds after the capture began and `s` is `out` or `err`. Its `correlation` carries the record's own `eventId` (the invocation id), the `runId` of a trigger fire or task run, and the attributed `userId`. Records are kept seven days and outlive an archived function.

The `integration` source is one outbound call. Its `detail` carries `method`, `path`, `callSource` (`user`, `admin`, `test`, `workflow` or `function`), `requestBytes`, `responseBytes`, `errorCode`, `actorType`, `actorEmail` and — for a call a server function made — `functionKey`, plus `stepAttempt` for a call made inside a function's `step.do` (which run of the body, from 1; it restarts after the task pauses, exactly as the function log's attempt does). Its `correlation` carries the `traceId`, the `integrationKey`, the attributed `userId`, and the keys that say where the call came from: `stepId` for a workflow step or a function step, `functionId` for a server function, and `runId` for either one's run. A function step also carries `correlation.stepOccurrence`, which use of the step name it was (from 0), so `runId`, `stepId` and `stepOccurrence` together identify one step of the run. A call made from a request invocation has no `runId`, because a request invocation writes no run row.

Pagination per view: `functions logs` returns `{ items, hasMore, nextCursor? }` and takes `--limit` (default 25, max 100) / `--cursor`; `integrations logs` returns `{ items }` and takes `--limit` plus `--status`/`--from`/`--to`/`--run`/`--step` (it filters inside a bounded scan rather than paging); `analytics events` returns `{ items, page, pageSize, totalRows }` and takes `--page`/`--window-days`/`--user-id`.

Not in the shared shape: `functions runs --json` prints the run rows as `{ items, nextCursor }` (each row adds `runtime`), and `functions runs steps --json` prints one run's full trace under `items`, never paged.

The normalization is `--json`-only. The human tables stay per-view, because each carries columns the shared shape has no room for.

Invalid filter values are rejected, not ignored: an unparseable `--from`/`--to`, a non-positive `--limit`, or a malformed `--cursor` fails with the server's validation message.

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

Slow invocations are included when their records become available. The command also reads additional pages when a poll finds more than one page of new records.
