# Agent Guide to Primitive Agents

An agent is a prompt of kind `agent`: it declares tools (server functions or client tools),
event kinds, turn context and who may answer a paused call. A session is one running
conversation with an agent, created by a server function; members then send messages,
answer calls, record events and cancel through the sessions API and the typed clients. The
session's conversation is an ordinary document.

## Declare an agent

```toml
# primitive/dev/prompts/advisor.toml
[prompt]
kind = "agent"
key = "advisor"
displayName = "Household advisor"

[prompt.inputSchema]
type = "object"
required = ["householdId"]

[prompt.inputSchema.properties.householdId]
type = "string"

[prompt.agent]
answeredBy = "participants"   # the owner or any participant may answer, supersede or cancel

[[prompt.agent.tools]]
name = "get_cashflow"
description = "Monthly income and spending for the household."
function = "cashflow"
invoke = "request"            # synchronous, 30 s at most (the default)

[[prompt.agent.tools]]
name = "generate_report"
description = "Build a detailed spending report. Takes a while."
function = "advisor-report"
invoke = "task"               # a durable task that reports back

[[prompt.agent.tools]]
name = "propose_budget_change"
description = "Show the member a proposed change to review."
runs = "client"
approval = true

[prompt.agent.tools.inputSchema]
type = "object"
required = ["category", "delta"]

[prompt.agent.tools.inputSchema.properties.category]
type = "string"

[prompt.agent.tools.inputSchema.properties.delta]
type = "number"

[prompt.agent.tools.outputSchema]
type = "object"
required = ["accepted"]

[prompt.agent.tools.outputSchema.properties.accepted]
type = "boolean"

[prompt.agent.turnContext]
function = "advisor-snapshot"

[prompt.agent.turnContext.schema]
type = "object"
required = ["plan"]

[prompt.agent.turnContext.schema.properties.plan]
type = "string"

[[prompt.agent.events]]
name = "applied"

[prompt.agent.events.schema]
type = "object"
required = ["callId"]

[prompt.agent.events.schema.properties.callId]
type = "string"

[[configs]]
name = "default"
active = true
provider = "openrouter"
model = "anthropic/claude-sonnet-4.5"

[configs.agent]
systemPrompt = "You advise household {{ input.householdId }}. Plan: {{ turn.plan | default: 'none' }}."
```

```bash
primitive config push
```

`[prompt.inputSchema]` validates the session's `variables`, set once at creation. There is
no `outputSchema` on an agent: the final answer is text; anything structured is a tool call.
`ctx.prompts.run` refuses an agent prompt by name — do not add tools or history to a
`chat`/`decisions` prompt to work around that.

`answeredBy` decides who may answer, supersede or cancel the active turn: the default,
`"initiator"`, admits only the member whose message started the turn (plus the owner, for
cancel); `"participants"` admits the owner or any participant.

## Tools

| Declared as | Schemas from | Runs |
| --- | --- | --- |
| `function = "<key>"`, `invoke = "request"` (default) | the function's `inputSchema`/`outputSchema` | synchronously, as that function, 30 s at most |
| `function = "<key>"`, `invoke = "task"` | the function's schemas | a durable task that reports back |
| `runs = "client"` | declared in the TOML (`inputSchema`/`outputSchema`) | a member's client answers it |
| `runs = "presentation"` | `inputSchema` in the TOML | never: data the model hands the client with its final answer |

`approval = true` on a server or client tool requires a person to approve first: a server tool runs once
approved; a client tool takes an approval, then a separate result — each stage won by the
first valid answer. `historyScope = "turn"` limits a call's full output to its own turn;
later turns see a stub. `statusText` is shown to members while the call is pending.

`endsTurn = true` is for server tools: the tool runs as usual, then the turn settles with no
further model round. A failing call does not end the turn.

Push refuses: a server tool whose function is missing or lacks either schema; `endsTurn` on a
client tool, or with `approval = true`; on a presentation, `function`, `outputSchema`,
`approval`, `invoke` or `endsTurn`; `references` anywhere else, or naming a property that is
not an integer or a list of integers; a tool or event schema not translatable without loss to
every provider a non-archived config uses. While an agent uses a function (tool, `turnContext.function` or `history.function`),
archiving it, deleting it, or a schema change the agent cannot take is refused with
`FUNCTION_USED_BY_AGENT`; change the agent in the same push, or retire it.
Gemini's JSON Schema subset refuses `$ref`, `allOf`, `not`, conditionals,
`patternProperties`, `prefixItems`, `contains`, `format`, and a free-form
`{ type: "object", properties: {} }`; it silently drops `additionalProperties: false` and
`$schema`/`$id`/`$comment` (not a refusal); a string `const` becomes an `enum`; a
discriminated `oneOf` becomes `anyOf`. Name a property, don't leave a tool's schema empty.

### Presentations

A presentation (`runs = "presentation"`) is structured data the model hands the client with
its final answer — result cards to show, follow-up chips. A valid call ends the turn: it is
final when the response is stored, with the result `{ "delivered": true }`, and once every
call of that response is final the response's text is the final answer. Declare
`inputSchema` only. Beside a call a person answers, the turn pauses on that call and settles
on its answer (the model never sees it). A call whose arguments fail the schema does not end
the turn. With `historyScope = "turn"`, later turns see its arguments stubbed too.

```toml
[[prompt.agent.tools]]
name = "present"
description = "Choose which results to show as cards beside your reply, and up to three follow-up suggestions."
runs = "presentation"
references = ["show"]

[prompt.agent.tools.inputSchema]
type = "object"
required = ["show"]

[prompt.agent.tools.inputSchema.properties.show]
type = "array"

[prompt.agent.tools.inputSchema.properties.show.items]
type = "integer"

[prompt.agent.tools.inputSchema.properties.suggestions]
type = "array"

[prompt.agent.tools.inputSchema.properties.suggestions.items]
type = "string"
```

Every tool call of a turn carries `callNumber`: 1 for the turn's first, every call counted in
order, refused ones included. `references` names the top-level input properties (each an
`integer` or a list of them) that refer to earlier calls by that number. With `references`
declared, every tool result reaches the model as `{ "call": n, "result": … }` or
`{ "call": n, "error": … }`, and the tool's description gains a sentence saying so. Each
referenced number must be a call from an earlier step of the same turn that succeeded; a
refused call, a call of the same response, or an unknown number is refused
`AGENT_PRESENTATION_REFERENCE_INVALID` and the model retries next round. Never ask the model
for a call's id.

### Every tool function checks rights on its first line

A tool call runs as the system, with the turn's initiator in `ctx.user` and the session in
`ctx.tool` (set only for a tool call). The function's `access` gate still
runs first, against that same initiator. In a call about one user it may name
only the initiator (`SUBJECT_USER_FORBIDDEN` otherwise), and so may anything it starts. The model chooses when to call a
tool and with what arguments, and the SAME function can also be invoked directly over HTTP —
so make `assertTrigger(ctx, "tool")` the first line, then check the caller's current rights
before doing anything else, reading the session's `variables` for whatever the check needs.
`assertTrigger` refuses every other trigger and narrows `ctx`: after it, `ctx.tool` and
`ctx.user` need no guard:

```toml
# primitive/dev/functions/cashflow.toml
[function]
key = "cashflow"
entry = "functions/cashflow/index.ts"
access = "!isAnonymous()"   # the gate checks the turn's initiator; household membership is the body's own check below

[function.inputSchema]
type = "object"

[function.inputSchema.properties.month]
type = "string"   # optional YYYY-MM; defaults to the current month

[function.outputSchema]
type = "object"
required = ["income", "spending"]

[function.outputSchema.properties.income]
type = "number"

[function.outputSchema.properties.spending]
type = "number"
```

```ts
// functions/cashflow/index.ts
import { assertTrigger, defineFunction } from "primitive-functions";

export default defineFunction(async (_input, ctx) => {
  assertTrigger(ctx, "tool"); // refuses any other trigger
  const { householdId } = ctx.tool.variables as { householdId: string };

  await assertHouseholdMember(ctx.user.userId, householdId); // check rights FIRST

  return await loadCashflow(householdId);
});
```

`ctx.tool` fields: `sessionId`, `turnId`, `callId` (stable across retries — use it as an
idempotency key), `toolName`, `documentId`, `ownerUserId`, `members: { viewerUserIds,
participantUserIds }`, `variables` (grant nothing by themselves), `timezone` and `locale`
(what the turn's send carried — the prompt's `turn.timezone`/`turn.locale`; `null` when
absent), `context` (the turn's validated turn context — the prompt's `turn.*`; `null` when the
turn carried none), and `attachArtifact(value)` — an app-only value on the call the model never sees; a client reads
it with `view.artifact` (see "Read a call's artifact"). Over 4 MiB it is refused
`AGENT_ARTIFACT_TOO_LARGE`.

A `task` tool is the same function, started durably instead of synchronously — same
`ctx.tool`, same first-line check:

```toml
# primitive/dev/functions/advisor-report.toml
[function]
key = "advisor-report"
entry = "functions/advisor-report/index.ts"
access = "!isAnonymous()"

[function.inputSchema]
type = "object"

[function.inputSchema.properties.month]
type = "string"   # optional YYYY-MM; defaults to the current month

[function.outputSchema]
type = "object"
required = ["summary"]

[function.outputSchema.properties.summary]
type = "string"
```

```ts
// functions/advisor-report/index.ts
import { assertTrigger, defineFunction } from "primitive-functions";

export default defineFunction(async (_input, ctx) => {
  assertTrigger(ctx, "tool"); // refuses any other trigger
  const { householdId } = ctx.tool.variables as { householdId: string };

  await assertHouseholdMember(ctx.user.userId, householdId); // check rights FIRST

  const report = await buildSpendingReport(householdId);
  await ctx.tool.attachArtifact({ reportId: report.id });
  return { summary: report.summary };
});
```

## Turn context and events

`turnContext.function` runs as the turn's initiator (`ctx.trigger.kind === "context"`) with
the session's variables, timezone and language on `input`; its return, validated against
`turnContext.schema`, renders as `turn.*` in the system prompt for every model call of that
turn, and never enters the document. With no `function`, a send's own `context` is
validated against the schema instead — declaring both on one agent is a contradiction a send
with a client `context` is refused for.

```toml
# primitive/dev/functions/advisor-snapshot.toml
[function]
key = "advisor-snapshot"
entry = "functions/advisor-snapshot/index.ts"
access = "!isAnonymous()"
```

```ts
// functions/advisor-snapshot/index.ts
export default defineFunction(async (input: { variables: { householdId: string } }, ctx) => {
  if (ctx.trigger.kind !== "context") throw new Error("advisor-snapshot only runs as turn context");
  const { householdId } = input.variables;
  return { plan: await currentPlan(householdId) };
});
```

Declared `events` are recorded by a member (`POST .../events`, `authorKind: "user"`) or
`ctx.agents.recordEvent(sessionId, event)` from a function the member invoked (`authorKind:
"app"`). An event never starts a turn; it reaches the model in a later turn's history, in
arrival order. Identify an event by its `kind`, not by `authorKind` — either path can record
the same kind. A repeated `eventId` with the same payload records nothing further; a
different payload is refused.

## Limits and history

`[prompt.agent]` lowers three limits, never raises them: `maxSteps` (model rounds per turn,
max 25), `waitLimitSeconds` (how long a paused call may wait, max 604800) and `maxRowBytes`
(JSON bytes of one conversation row, max 262144). A tool result or client answer that would
take its row past `maxRowBytes` is not written and the turn fails `AGENT_TURN_ROW_TOO_LARGE`
— keep tool output small and attach the bulk with `ctx.tool.attachArtifact`.

`[prompt.agent.history]` selects what the model sees of earlier turns; without it, the model
sees all of them. One key only — `config push` refuses a table with both:

```toml
[prompt.agent.history]
maxHistoryChars = 40000     # whole earlier turns drop, oldest first, until the rest fits
```

```toml
[prompt.agent.history]
function = "advisor-history"
```

The history function is a hook like `turnContext.function`, run as the turn's initiator
before every model call with `input.variables`, `input.timezone`, `input.locale` and the
candidate `input.messages`. Return `{ messages: [...] }`: each entry is `{ messageId }` for
a message you were given (sent in the order you list them) or `{ text }` for text of your
own, such as a summary of what you dropped. An empty list, an unknown `messageId` or any
other shape fails the turn `AGENT_TURN_HISTORY_INVALID`; a thrown error fails it
`AGENT_TURN_HISTORY_FAILED`. Whichever way it is selected, a history that still does not fit
the model's context fails the turn `AGENT_HISTORY_TOO_LARGE`.

## Create a session from a server function

```ts
// functions/start-advisor/index.ts
import { defineFunction } from "primitive-functions";

export default defineFunction(async (input: { householdId: string }, ctx) => {
  await assertHouseholdMember(ctx.user!.userId, input.householdId); // your own authorization

  return await ctx.agents.createSession({
    agent: "advisor",
    scope: input.householdId,
    variables: { householdId: input.householdId },
  });
  // => { sessionId, documentId }; ctx.user becomes the session's owner
});
```

The owner is `userId`, else the invoking user. `createSession` works from any kind of run:
one with no caller (cron, a webhook) must pass `userId` (`FUNCTION_SUBJECT_REQUIRED`), and
naming another member needs a run with no caller or one an app owner or admin invoked. A
tool, context or history function, and anything it starts, acts only for the turn's
initiator (`SUBJECT_USER_FORBIDDEN`). The function's `access` rule is the entire
authorization for who may use the agent — nothing on the agent prompt itself gates it.
Variables are re-validated against `inputSchema`; a declared agent key and bad variables are
compile errors once a tree has pushed.

`ctx.agents.getSession(sessionId, { userId })`, `listSessions({ userId })`,
`recordEvent(sessionId, event, { userId })` and `deleteSession(sessionId, { userId })` follow
the same rule: they act as that member, with no admin arm, and never return a session's
`object`, `turnContexts` or `lastTurn`.

`deleteSession` deletes as the owner, with the same outcome as the client's delete, and
answers `{ deleted: true, sessionId, documentId }`. It is the way to remove a member's
sessions from an account-deletion or reset function (list with `listSessions({ userId })`
and delete the items whose `role` is `"owner"`). A session whose document is already gone
can still be deleted. A participant or viewer is refused `AGENT_SESSION_OWNER_ONLY`, anyone
else `AGENT_SESSION_NOT_FOUND`. A `userId` passed as `null` or a non-string is refused
`SUBJECT_USER_INVALID`, never read as the invoking user. None of the five ever sends,
answers, cancels, marks a session read, renames or manages members — those are the
members', from a client.

## Use a session from a client

### Listing sessions and unread activity

`client.agents.sessions.list()` answers the signed-in user's sessions, newest first, without
opening any document. An item's `hasUnread` is true when the session has activity the user has
not marked read: an agent's answer, a turn paused for an answer, a turn's end, or another
member's message or event. The user's own messages, events and cancels never count, and a
session they have never marked is unread. Mark it read when the user opens it:

{{ example: agents/list-sessions }}

The mark is the user's own and is stored on the server, so the dot clears in `list()` and
`get()` on every device; `get()` answers the caller's own `hasUnread` and `lastReadAt`. The
owner, participants and viewers may mark a session read; an app owner or admin who is not in
it is refused `AGENT_SESSION_MEMBER_ONLY`.

### Keeping a session list live

`client.agents.sessions.watch(options)` answers the first page of `list()` kept current from
the server's changes, with no polling and no document opened: activity turns the dot on and
moves the session up, a read on another device clears it, and title, `status` and members
update in place. Items are ordered by last activity, newest first; `loadMore()` reads the next
page, `onChange` fires on every change, and the list reads its loaded pages again after a
reconnect. Open the session the user is viewing with `markRead: "while-open"`: it is marked
read at open and again whenever activity arrives while it is open. The view's `info` follows
the same changes.

{{ example: agents/watch-sessions }}

### Opening a session

`client.agents.sessions.open(sessionId)` runs `get`, registers the platform models, opens the
session's document and returns a live view.

{{ example: agents/open-session }}

{{ example: agents/send-message }}

A send's `timezone` and `locale` render as `turn.timezone` and `turn.locale` for its turn and
reach the context function's `input`. With no `turnContext.function`, the send also carries
`context`, validated against the schema (`AGENT_SESSION_CONTEXT_NOT_ALLOWED`,
`AGENT_SESSION_CONTEXT_INVALID`); a failed message keeps it for a retry.

A turn pauses on an approval or a client tool that takes an answer, surfacing every call of
one model response together. Find a pending call from the view's parts (`tool_call`, state
`approval-requested` or `input-available`), then:

{{ example: agents/answer-call }}

A presentation needs no answer. `view.presentations` answers an agent message's presentations
(pass the generated `presentationNames`), each with `name`, `input`, `part` and `call(n)`,
which resolves a referenced number to that call's part in the view's window:

{{ example: agents/presentation }}

### Read a call's artifact

The view's `artifact` method answers a tool call's artifact (from `ctx.tool.attachArtifact`
or a client tool handler's `artifact`), the same value whether it was stored on the row or,
being large, as a blob on the session's document. Pass a `tool_call` part; a blob is
downloaded only by this call.

{{ example: agents/read-artifact }}

{{#lang ts}}
- A call with no artifact answers `undefined`.
{{/lang}}
{{#lang swift}}
- A call with no artifact answers `nil`.
{{/lang}}
- A failed download throws the blob read's error; show a fallback rather than an empty result.
- Read the artifact through `view.artifact`, not the part's `artifact` field, which is empty
  when the artifact is stored as a blob.

### Handle client tools

`view.handleClientTools(handlers)` answers client tools for you. When the active turn is the
current user's (their message started it), each call of a named tool that waits for its
result runs the handler, and the handler's `output` (and optional `artifact`, read back with
`view.artifact`) answers the call. Calls already waiting when you register, or found again
when the document re-syncs after a reconnect, run too. It returns a function that removes the handlers; a run in flight
still answers.

{{ example: agents/handle-client-tools }}

- The handler gets the decoded `input` and the call's `sessionId`, `turnId`, `callId`,
  `messageId`, `partId` and `toolName`. A thrown error answers the call with its message
  (`output-error`) and the turn goes on.
- Type handlers with the agent's generated client-tool types, as above; a handler whose input
  or output does not match its tool is a compile error.
- A call waiting for approval is left to your UI; once approved, its handler runs.
- `runningClientToolCalls` lists the calls a handler is running now; `onChange` fires as it
  changes. The answer itself shows on the call's part.
- `onError` hears an answer that did not land. With a `code` the server refused it (for
  example `AGENT_ANSWER_INVALID` for an output that fails the tool's `outputSchema`): the call
  is not run again while it waits, so answer it with `view.answer`. Without a `code` (offline,
  network) the call runs again on the next refresh or re-sync.

{{#lang ts}}
The handler's `signal` aborts when the view closes; no answer is sent after `close()`.
{{/lang}}

{{#lang swift}}
`close()` cancels the handler's task; no answer is sent after it.
{{/lang}}

{{ example: agents/record-event }}

`view.cancel()` cancels the active turn, naming it (or `view.cancel(turnId)` for a specific
one); a stale id is refused `AGENT_TURN_STALE`.

{{#lang ts}}
`view.uiMessages()` converts the window to AI SDK v7 `UIMessage`s (no `ai` runtime
dependency; it targets the structural types). Platform-only fields ride in `metadata` and
`metadata.parts[i]`; pending sends append as `metadata.pending`; an unmappable part throws.
{{/lang}}

{{#lang swift}}
There is no `uiMessages()` in Swift: read a part's `content` (`PrimitivePartContent`: `.text`,
`.reasoning`, `.toolCall`, `.file`, `.event`, `.unknown(kind:)`) directly.
{{/lang}}

A message sent while a turn runs queues; one from the owner or whoever `answeredBy` allows,
sent while the turn is paused, supersedes it instead (unanswered calls end `superseded`, the
model sees it).

### Read a turn's steps

Each view message's `steps` are its turn's timed rows, by `seq`: `kind`, `status` (`running`,
`completed`, `failed`), `startedAt`, and once ended `endedAt`, `durationMs`, `errorCode`.

| Kind | Times |
| --- | --- |
| `model_call` | One model round; `providerDurationMs`, token usage, `cost` |
| `tool_call` | One server tool run; `functionName`, `functionVersion` |
| `client_call` | One client tool call, from the call written to its answer arriving |
| `context_call` | The turn context function |
| `history_call` | The history function, per model round |

A `client_call` step's `callId` and `toolName` match the call's part; `resolvedByUserId` is the
member whose answer or cancel made it final. An approval-gated client tool has one step over
approval and result. A call that ends unanswered is `failed` with `AGENT_CALL_CANCELLED`,
`AGENT_CALL_SUPERSEDED`, `AGENT_CALL_EXPIRED` or the failed turn's code; a denial is
`AGENT_CALL_DENIED`, an error answer `AGENT_CLIENT_TOOL_FAILED`. A presentation call has no step.

## Test an agent

A test case on an agent runs one real model round from a history you write, then checks
what the model did. Only the model call happens: no tool runs and no session is created.
For an agent that declares a `query` tool and a turn context with a `snapshot`:

```toml
# prompts/spending.tests/corrects-a-refused-input.toml
[test]
name = "corrects-a-refused-input"
inputVariables = '{"householdId": "h1"}'

[test.agent]
history = '''[
  {"role": "user", "text": "How much did we spend at Costco last month?"},
  {"role": "assistant", "calls": [{"callId": "call_1", "name": "query", "arguments": {"dimension": "vendor", "filter": "Costco"}}]},
  {"role": "tool", "callId": "call_1", "error": {"code": "INVALID_DIMENSION", "message": "dimension must be one of: merchant, category"}}
]'''
context = '{"snapshot": {"month": "2026-09"}}'
timezone = "Europe/Madrid"
locale = "es-ES"
expectedToolCall = '{"name": "query", "arguments": {"dimension": "merchant"}}'
```

```bash
primitive config push --only prompt/spending
primitive prompts tests run-all spending
```

| Key | Meaning |
| --- | --- |
| `history` | JSON text, oldest first: `user` (`text`), `assistant` (`text` and/or `calls`), `tool` (`callId` with `output` or `error`), `event` (`kind`, `refersTo`, `data`). The last `user` entry is the current turn. |
| `context` | `turn.*` for the round, as a send carries it. |
| `timezone` | `turn.timezone`; `now` renders in it. |
| `locale` | `turn.locale`. |
| `expectedToolCall` | The round calls this tool; `arguments`, when given, is matched as a subset. |
| `expectedNoToolCall` | `true`: the round calls no tool. |

`inputVariables` are the session's variables (`input.*`), checked against
`[prompt.inputSchema]`. `expectedOutputPattern` and `expectedOutputContains` check the round's
text, and `evaluatorPromptKey` judges it with a chat prompt that reads `{{ input }}`,
`{{ output }}`, `{{ calls }}` and `{{ history }}`. Each run records a `Round completed` check
with the stop reason and the calls made, and a `Calls <tool>` or `No tool call` check for the
assertions.

### Watch out when writing a test case

- Answer every call in the history exactly once, right after its assistant entry; write an
  unanswered call as an `error`.
- Put an event before the last user entry: a live event reaches the model only in a later turn.
- A declared `history.function` does not run in a test, so the history goes whole; a
  `maxHistoryChars` cap still applies.
- Assert a structured answer with `expectedToolCall`; `expectedJsonSubset` is refused on an
  agent's case.
- Removing a tool or event kind a stored case uses makes its runs fail the `Test case input`
  check with `AGENT_TEST_CASE_INVALID`; update the case.

## Admin: inspecting sessions (no member actions)

```bash
primitive agent-sessions list --user-id <user-id> [--agent <key>] [--scope <scope>]
primitive agent-sessions get <session-id>
primitive agent-sessions delete <session-id>
```

`list` marks the listed user's unread sessions with `•` in its `UNREAD` column. `get` never
copies the conversation's rows — those are read by opening `documentId` with
`primitive documents records query`. The CLI has no send/answer/cancel/events/members/rename
— those are the clients' (a recorded principle-11 deferral). A client signed in as an app
owner or admin reads the kept turn contexts from `get()`'s `turnContexts`; a member's `get()`
never carries them.

## Error codes

Every refusal and failure here carries a stable `code` under the standard error envelope
(see [Error Handling](AGENT_GUIDE_TO_PRIMITIVE_ERROR_HANDLING.md)). Codes worth branching on:

| Code | When |
| --- | --- |
| `FUNCTION_SUBJECT_REQUIRED` | A `ctx.agents` call from a run with no caller named no `userId`. |
| `SUBJECT_USER_FORBIDDEN` | The run may not name that member: an ordinary member invoked it, or an agent drives it. |
| `AGENT_SESSION_VARIABLES_INVALID` | Session variables fail `inputSchema`. `details.errors` lists each path. |
| `AGENT_TOOL_ARGUMENTS_INVALID` | A model call's arguments failed the tool's input schema — the model sees this and may retry; it is not thrown at you. |
| `AGENT_SESSION_PARTICIPANT_ONLY` | A viewer tried to send, upload a file or record an event. |
| `AGENT_SESSION_MEMBER_ONLY` | An app owner or admin who is not in the session tried to mark it read. |
| `AGENT_SESSION_RATE_LIMITED` | Past the send limit. `details.retryAfter` names the wait in seconds. |
| `AGENT_SESSION_NOT_FOUND` | The caller is not a member (and not an app owner or admin), or the id doesn't exist. |
| `AGENT_SESSION_CONTEXT_NOT_ALLOWED` | A send's `context` for an agent with no turn context, or one a `function` fills. |
| `AGENT_SESSION_CONTEXT_INVALID` | A send's `context` fails `turnContext.schema`, or a required one is missing. `details.errors` lists each path. |
| `AGENT_CALL_ANSWERED` | A second answer to a call that already has one. |
| `AGENT_TURN_STALE` | `cancel`/an answer named a `turnId` that is no longer active. |
| `AGENT_ARTIFACT_TOO_LARGE` | `ctx.tool.attachArtifact` over 4 MiB of JSON. |
| `AGENT_TURN_ROW_TOO_LARGE` | A tool result or client answer would make its row larger than `maxRowBytes`; the turn fails. |
| `AGENT_HISTORY_TOO_LARGE` | The selected history does not fit the model's context; add or tighten `[prompt.agent.history]`. |
| `AGENT_SESSION_DOCUMENT_MISSING` | The session's document was deleted directly; every operation but delete answers this. |
| `AGENT_TEST_CASE_INVALID` | A test case's `[test.agent]` block cannot be a round of this agent. `details.refusals` lists every problem. |
| `AGENT_PRESENTATION_REFERENCE_INVALID` | A presentation's `references` named a refused call, one of the same response, or a number the turn never gave; the model sees it and retries. |

## Gotchas

- A kind check (`ctx.trigger.kind === "tool"`) narrows `ctx.trigger`, not `ctx`, so
  `ctx.tool` stays possibly undefined and the push typecheck refuses reading it — call
  `assertTrigger(ctx, "tool")` instead. Always re-check `ctx.user`'s rights after it; never
  trust `ctx.tool.variables` as authorization.
- An agent has no `outputSchema`, no `userPromptTemplate`, and no single-shot run through
  `ctx.prompts.run`.
- `turnContext.function` and a send's client `context` are mutually exclusive — declaring a
  function refuses a client-supplied context by name.
- A client tool with `approval = true` has TWO answers, not one: approve, then the output.
- A presentation has none: don't answer it (an answer is refused `AGENT_CALL_ANSWERED`) and
  don't register a handler for it (the generated handler types refuse one); read it with
  `view.presentations`.
  A server tool with `approval = true` has one: approving it runs the tool.
- Every one of the initiator's open views (each tab or device) runs a waiting client tool
  with `handleClientTools`; the first answer wins and the others are dropped silently. Keep
  handlers free of side effects, or gate the work inside the handler when it must run once.
- `handleClientTools` runs only in the turn initiator's views, even under
  `answeredBy = "participants"`; another member answers through your own code and
  `view.answer`.
- Gemini's schema subset is stricter than the chat prompt's `outputSchema` clean-up: name a
  property, don't declare a free-form object (`{ type: "object", properties: {} }`).
- `createSession`, `getSession`, `listSessions` and `recordEvent` act for `userId`, else the
  invoking user; a tool can create a session only for its initiator. Sending, answering,
  cancelling, marking read and member management are client-only.
- `watch` lists only the signed-in user's sessions and refuses a `userId`; an admin screen
  for another user's sessions uses `list({ userId })` and calls it again to update.
- Close every watched list and session view when its screen goes away; until then each keeps
  listening, and a `markRead: "while-open"` view keeps marking the session read.

## Related guides

- [Prompts](AGENT_GUIDE_TO_PRIMITIVE_PROMPTS.md)
- [Server Functions](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md)
- [Documents](AGENT_GUIDE_TO_PRIMITIVE_DOCUMENTS.md)
