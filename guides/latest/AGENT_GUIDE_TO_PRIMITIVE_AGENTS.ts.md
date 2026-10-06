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
`ctx.prompts.run` and the member execute route refuse an agent prompt by name — do not add
tools or history to a `chat`/`decisions` prompt to work around that.

## Tools

| Declared as | Schemas from | Runs |
| --- | --- | --- |
| `function = "<key>"`, `invoke = "request"` (default) | the function's `inputSchema`/`outputSchema` | synchronously, as that function, 30 s at most |
| `function = "<key>"`, `invoke = "task"` | the function's schemas | a durable task that reports back |
| `runs = "client"` | declared in the TOML (`inputSchema`/`outputSchema`) | a member's client answers it |

`approval = true` on either kind requires a person to approve first: a server tool runs once
approved; a client tool takes an approval, then a separate result — each stage won by the
first valid answer. `historyScope = "turn"` limits a call's full output to its own turn;
later turns see a stub. `statusText` is shown to members while the call is pending.

Push refuses: a server tool whose function is missing or lacks either schema; a tool or
event schema not translatable without loss to every provider a non-archived config uses.
Gemini's JSON Schema subset refuses `$ref`, `allOf`, `not`, conditionals,
`patternProperties`, `prefixItems`, `contains`, `format`, and a free-form
`{ type: "object", properties: {} }`; it silently drops `additionalProperties: false` and
`$schema`/`$id`/`$comment` (not a refusal); a string `const` becomes an `enum`; a
discriminated `oneOf` becomes `anyOf`. Name a property, don't leave a tool's schema empty.

### Every tool function checks rights on its first line

A tool call runs as the system, with the turn's initiator in `ctx.user` and the session in
`ctx.tool` (set only when `ctx.trigger.kind === "tool"`). The model chooses when to call a
tool and with what arguments, and the SAME function can also be invoked directly over HTTP —
so check the caller's current rights before doing anything else, reading the session's
`variables` for whatever the check needs:

```ts
// functions/cashflow/index.ts
import { defineFunction } from "primitive-functions";

export default defineFunction(async (_input, ctx) => {
  if (ctx.trigger.kind !== "tool") throw new Error("cashflow only runs as an agent tool");
  const { householdId } = ctx.tool.variables as { householdId: string };

  await assertHouseholdMember(ctx.user!.userId, householdId); // check rights FIRST

  return await loadCashflow(householdId);
});
```

`ctx.tool` fields: `sessionId`, `turnId`, `callId` (stable across retries — use it as an
idempotency key), `toolName`, `documentId`, `ownerUserId`, `members: { viewerUserIds,
participantUserIds }`, `variables` (grant nothing by themselves), and
`attachArtifact(value)` — an app-only value on the call's row the model never sees (over
64 KiB it becomes a blob; over 4 MiB it is refused `AGENT_ARTIFACT_TOO_LARGE`).

A `task` tool is the same function, started durably instead of synchronously — same
`ctx.tool`, same first-line check:

```ts
// functions/advisor-report/index.ts
import { defineFunction } from "primitive-functions";

export default defineFunction(async (_input, ctx) => {
  if (ctx.trigger.kind !== "tool") throw new Error("advisor-report only runs as an agent tool");
  const { householdId } = ctx.tool.variables as { householdId: string };

  await assertHouseholdMember(ctx.user!.userId, householdId); // check rights FIRST

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

```ts
// functions/advisor-snapshot/index.ts
export default defineFunction(async (input, ctx) => {
  if (ctx.trigger.kind !== "context") throw new Error("advisor-snapshot only runs as turn context");
  const { householdId } = input.variables as { householdId: string };
  return { plan: await currentPlan(householdId) };
});
```

Declared `events` are recorded by a member (`POST .../events`) or
`ctx.agents.recordEvent(sessionId, event)` from a function the member invoked. An event never
starts a turn; it reaches the model in a later turn's history, in arrival order. A repeated
`eventId` with the same payload records nothing further; a different payload is refused.

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

`createSession` works ONLY inside a run started over HTTP by a signed-in user (`invoke`, or
a `start` task) — never from a tool, a context/history function, a workflow step, a nested
start, cron or a webhook (`AGENT_SESSION_ORIGIN_REFUSED`, `details.origin`). The function's
`access` rule is the entire authorization for who may use the agent — nothing on the agent
prompt itself gates it. Variables are re-validated against `inputSchema`; a declared agent key and bad variables are
compile errors once a tree has pushed. `ctx.agents.getSession`/`listSessions` read the
INVOKING user's own sessions; `ctx.agents.recordEvent` records as that member. None of the
four ever sends, answers or manages members — that is the owner's, from a client.

## Use a session from a client

`client.agents.sessions.open(sessionId)` runs `get`, registers the platform models, opens the
session's document and returns a live view.

```typescript
  const view = await client.agents.sessions.open(sessionId);
  const unsubscribe = view.onChange(() => {
    for (const entry of view.messages) {
      if (entry.kind === "pending") continue; // not yet synced
      for (const part of entry.parts) {
        if (part.kind === "text") console.log(entry.message.authorKind, part.text);
      }
    }
  });

  // later, when the view is no longer shown:
  unsubscribe();
  await view.close();
```

```typescript
  try {
    await view.send(question);
  } catch (error) {
    if (error instanceof JsBaoApiError) {
      console.error(error.code, error.message); // e.g. AGENT_SESSION_RATE_LIMITED
    }
  }
```

A turn pauses on an approval or a client tool, surfacing every call of one model response
together. Find a pending call from the view's parts (`tool_call`, state
`approval-requested` or `input-available`), then:

```typescript
  await view.answer(callId, { approval: { approved } });
  if (approved) {
    await view.answer(callId, { output: { accepted: true } });
  }
```

```typescript
  await view.recordEvent({
    kind: "applied",
    refersTo: callId,
    data: { appliedAt: new Date().toISOString() },
  });
```

`view.cancel()` cancels the active turn, naming it (or `view.cancel(turnId)` for a specific
one); a stale id is refused `AGENT_TURN_STALE`.

`view.uiMessages()` converts the window to AI SDK v7 `UIMessage`s (no `ai` runtime
dependency; it targets the structural types). Platform-only fields ride in `metadata` and
`metadata.parts[i]`; pending sends append as `metadata.pending`; an unmappable part throws.


A message sent while a turn runs queues; one from the owner or whoever `answeredBy` allows,
sent while the turn is paused, supersedes it instead (unanswered calls end `superseded`, the
model sees it).

## Admin: inspecting sessions (no member actions)

```bash
primitive agent-sessions list --user-id <user-id> [--agent <key>] [--scope <scope>]
primitive agent-sessions get <session-id>
primitive agent-sessions delete <session-id>
```

`get` never copies the conversation's rows — those are read by opening `documentId` with
`primitive documents records query`. The CLI has no send/answer/cancel/events/members/rename
— those are the clients' (a recorded principle-11 deferral).

## Error codes

Every refusal and failure here carries a stable `code` under the standard error envelope
(see [Error Handling](AGENT_GUIDE_TO_PRIMITIVE_ERROR_HANDLING.md)). Codes worth branching on:

| Code | When |
| --- | --- |
| `AGENT_SESSION_ORIGIN_REFUSED` | `ctx.agents.*` called from a disallowed trigger kind. `details.origin` names it. |
| `AGENT_SESSION_VARIABLES_INVALID` | Session variables fail `inputSchema`. `details.errors` lists each path. |
| `AGENT_TOOL_ARGUMENTS_INVALID` | A model call's arguments failed the tool's input schema — the model sees this and may retry; it is not thrown at you. |
| `AGENT_SESSION_PARTICIPANT_ONLY` | A viewer tried to send, upload a file or record an event. |
| `AGENT_SESSION_RATE_LIMITED` | Past the send limit. `details.retryAfter` names the wait in seconds. |
| `AGENT_CALL_ANSWERED` | A second answer to a call that already has one. |
| `AGENT_TURN_STALE` | `cancel`/an answer named a `turnId` that is no longer active. |
| `AGENT_ARTIFACT_TOO_LARGE` | `ctx.tool.attachArtifact` over 4 MiB of JSON. |
| `AGENT_SESSION_DOCUMENT_MISSING` | The session's document was deleted directly; every operation but delete answers this. |

## Gotchas

- `ctx.tool` is only set when `ctx.trigger.kind === "tool"` — the same function can also run
  as a plain invocation; guard it, and always re-check `ctx.user`'s rights, never trust
  `ctx.tool.variables` as authorization.
- An agent has no `outputSchema`, no `userPromptTemplate`, and no single-shot run through
  `ctx.prompts.run`.
- `turnContext.function` and a send's client `context` are mutually exclusive — declaring a
  function refuses a client-supplied context by name.
- A client tool with `approval = true` has TWO answers, not one: approve, then the output.
  A server tool with `approval = true` has one: approving it runs the tool.
- Gemini's schema subset is stricter than the chat prompt's `outputSchema` clean-up: name a
  property, don't declare a free-form object (`{ type: "object", properties: {} }`).
- `createSession`, `getSession`, `listSessions` and `recordEvent` only ever act for the
  invoking user. Sending, answering, cancelling and member management are client-only.

## Related guides

- [Prompts](AGENT_GUIDE_TO_PRIMITIVE_PROMPTS.md)
- [Server Functions](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md)
- [Documents](AGENT_GUIDE_TO_PRIMITIVE_DOCUMENTS.md)
