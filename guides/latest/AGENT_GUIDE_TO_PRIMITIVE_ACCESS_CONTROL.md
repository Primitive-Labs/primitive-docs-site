# Agent Guide to Primitive Access Control

A server function's own call gate is a method you write: every version pushed today defines an `authorize` method that decides who may call it in code (below). Rule sets and metadata rules instead use CEL — test a CEL rule as a regular member, because app owners and admins bypass rules; an `authorize` method gives them no such bypass.

## First rule

```ts
import { defineFunction, requireMember } from "primitive-functions";

export default defineFunction("team-report", {
  authorize: requireMember((input: { teamId: string }) => input.teamId),
  handler: async (input, ctx) => ({ team: ctx.auth.groupId }),
});
```

This function admits a member of the group the input names. Inside the function, still authorize requested resources and scope records to the caller.

## Identity context

Available to a rule set or metadata rule, but not to a webhook handshake
`when`, which is CEL over the request shape (`body`, `query`, `headers`,
`rawBody`), not an access rule; see the server-functions guide's webhook
verification section.

| Variable / function | Meaning |
|---|---|
| `user.userId` | Caller's user ID (`''` when unauthenticated) |
| `user.role` | Caller's app role |
| `isAnonymous()` | True when the caller has no account (unauthenticated request); `!isAnonymous()` ⇒ any signed-in member. Matters where anonymous access is reachable, e.g. a `public` blob bucket |
| `hasRole(role)` | App role check: `"owner"` \| `"admin"` \| `"member"` (app-level role — distinct from a database's `owner` permission) |
| `isMemberOf(groupId)` | Caller holds any role on the group |
| `hasGroupRole(groupId, role)` | Caller's group role is `role` or higher (`owner` > `manager` > `member`); an unknown role literal is refused at save (`INVALID_GROUP_ROLE`) |
| `memberGroups()` | List of groupIds the caller holds a role on |
| `isMemberOf(groupType, groupId)` | Deprecated (use `isMemberOf(groupId)`): exact group membership |
| `memberGroups(groupType)` | Deprecated (use `memberGroups()`): groupIds of that type the caller belongs to |

Prefer membership checks over user-ID comparisons: `isMemberOf('01JD7E8V3K9Q2N6R4T5W0X1Y2Z')` (the `groupId` the group's create returned), `size(memberGroups()) > 0`.

A rule set or metadata rule can check roles and group membership, but cannot read database records. Store entitlements as group memberships when they must be checked by a rule, or move the check into a function's `authorize` method (below), which can read records directly.

## The `authorize` method

Give a function an `authorize` method (object form of `defineFunction`) when the decision needs more than identity — a resource the input names, a group or collection role, a record that names the caller. It runs after `inputSchema` validation and before the handler:

```ts
import { allow, defineFunction, deny } from "primitive-functions";

export default defineFunction("get-order", {
  authorize: async (input: { shopDb: string; orderId: string }, auth) => {
    if (!auth.user) return deny("FORBIDDEN");
    const order = await auth.db3(input.shopDb, "orders").model("orders").get(input.orderId);
    if (!order) return deny("NOT_FOUND");
    if (order.userId !== auth.user.userId && !(await auth.isMemberOf("support-team"))) {
      return deny("FORBIDDEN");
    }
    return allow({ order });
  },
  handler: async (input, ctx) => ctx.auth.order, // loaded and checked by authorize
});
```

`allow(resolved)` becomes `ctx.auth` in the handler (`allow()` resolves `null`). `deny(code, message?, { expose? })` takes `FORBIDDEN` | `NOT_FOUND` | `INVALID`. A denied caller gets `403 FUNCTION_ACCESS_DENIED` whatever the code, unless the deny used `{ expose: true }` on `NOT_FOUND` (then `404` with that code and message; `expose` does nothing on the other codes). No implicit allow: a throw, a timeout, a return value other than `allow()`/`deny()`, or a non-serializable `resolved` all deny the caller the same uniform way.

`authorize` runs on every door: invoke and task start, `--as system`, `ctx.functions.start`, agent tool/context/history calls, webhook deliveries, cron fires and database change triggers. A webhook, cron or db3 deny records the fire's run as `denied` before the handler runs, with the code leading its error message (`FORBIDDEN: …`); the provider still gets its normal answer and a cron fire still counts.

### Declaring input keys

`input` lists the top-level input keys the handler reads. After an allow on a caller door, the handler gets only those keys; `authorize` still reads the whole input. An `authorize` function with no `input` hands its handler `{}` there — the migration is "add `authorize`, declare `input`":

```ts
import { defineFunction, requireMember } from "primitive-functions";

export default defineFunction({
  // The one input key the handler reads. `teamId` is for authorize alone.
  input: ["note"],
  authorize: requireMember((input: { teamId: string; note: string }) => input.teamId, "manager"),
  handler: async (input, ctx) => {
    // input is { note }; the team is the one authorize checked.
    return { teamId: ctx.auth.groupId, note: input.note.trim() };
  },
});
```

Every listed key must be a property of the TOML `inputSchema`:

```toml
[function.inputSchema]
type = "object"
required = ["teamId", "note"]

[function.inputSchema.properties.teamId]
type = "string"
maxLength = 64

[function.inputSchema.properties.note]
type = "string"
maxLength = 500
```

The handler's input is typed to the listed keys, so reading another key fails the push typecheck. A listed key the caller didn't send is absent, not `undefined`. Push refuses a list it can't record:

| Code | When |
|---|---|
| `FUNCTION_INPUT_LIST_INVALID` | Not an array of strings, a repeated key, over 64 keys, or a key that is empty, over 64 characters, or outside `[A-Za-z0-9_$.-]` |
| `FUNCTION_INPUT_KEY_UNDECLARED` | A listed key isn't an `inputSchema` property; names the key and the declared properties |
| `FUNCTION_INPUT_SCHEMA_REQUIRED` | `input` declared, but `inputSchema` declares no object properties |

Trigger and agent hook doors (`webhook`, `cron`, `db3`, `users-batch`, `context`, `history`) hand the handler the whole input whatever `input` lists.

**Big inputs:** the list names top-level keys, so fold a wide form into one nested object and list it once. Keep ids at the top level, where `authorize` checks them:

```ts
import { defineFunction, requireMember } from "primitive-functions";

interface SaveProfileInput {
  // Top level: the id authorize checks.
  householdId: string;
  // Nested: the form, named once in the list.
  profile: { name: string; birthYear: number; riskTolerance: "low" | "medium" | "high" };
}

export default defineFunction({
  input: ["profile"],
  authorize: requireMember((input: SaveProfileInput) => input.householdId, "member"),
  handler: async (input, ctx) => {
    // input is { profile }; the household is the one authorize checked.
    return { householdId: ctx.auth.groupId, name: input.profile.name.trim() };
  },
});
```

Declare `profile` as an object property of `inputSchema` with its own `properties`. A new form field is a schema edit inside `profile`; a field naming another resource goes top level, where `authorize` can check it. There is no wildcard: an explicit list makes a handler that reads a new key fail the push typecheck until the list names it. Inside the nested object, `inputSchema` is the only guard, so bound its fields there.

### Doors at a glance

| Door | `auth.user` | Handler's input | A deny |
|---|---|---|---|
| `http` (invoke or start) | The caller | The listed keys (`{}` with no list) | `403`/`404` before a run id exists; the call's invocation log records it `denied` |
| `function` (a nested start) | The chain's root caller, the same at every level however deep it nests; `null` when a trigger rooted it. `auth.trigger.functionKey` names only the parent directly above | The listed keys | Same as `http`. Every level's own `authorize` runs — a parent's `ctx.functions.start` is the capability to try, not a bypass |
| `manual` (`--as system`) | The admin who ran it, with role `"admin"` | The listed keys | Same as `http` |
| `tool` | The turn's initiator | The listed keys | `AGENT_TOOL_ACCESS_DENIED`, the tool error the model reads; logged the same way |
| `context`, `history` | The turn's initiator | The whole input | Fails the turn; logged the same way |
| `webhook` | `null` | The whole input | The run settles `denied`; the provider still gets its normal answer |
| `cron` | `null` | The whole input | The run settles `denied`; the fire still counts |
| `db3` (a `[[function.triggers.db3]]` change) | `null` | The whole input | The run settles `denied` |
| `users-batch` (a [users walk](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#walking-every-user)'s batch) | `null` | The whole input | The run settles `denied`, which fails the whole walk at once with `USER_WALK_DENIED` |

### auth context (read-only)

| Member | Reads |
|---|---|
| `auth.user` | `{ userId, email, role }` or `null`. Agent tool/context/history: the turn's user. Nested start: the chain's first caller. `--as system`: the admin, role `"admin"` |
| `auth.hasRole(role)` | Exact app role match (`"owner"` \| `"admin"` \| `"member"`) |
| `auth.memberships()`, `auth.isMemberOf(groupId)`, `auth.memberGroups()`, `auth.hasGroupRole(groupId, role)` | The caller's membership list, with the same one-argument forms as CEL |
| `auth.collections.resolveRole(collectionId)` | Caller's collection role, or `null` |
| `auth.db3(db3Id, type).permissions()` / `.model(m).get(id)` / `.query(...)` | The caller's level on the database (`reader` \| `read-write` \| `owner` \| `null`), and its records on the app's authority, typed like `ctx.db3` |
| `auth.userProperties()` | The caller's own [properties](AGENT_GUIDE_TO_PRIMITIVE_USER_PROPERTIES.md), defaults applied, or `null` with no caller. One read, made only when called |
| `auth.users.get(userId)` / `.getBy(name, value)` | Another member with their properties, or the holder of a unique value; `null` if none. One read each, made only when called |
| `auth.configVar(name)` | A declared app config variable |
| `auth.trigger` | The door that started the call |

No write, secret, integration, or start call reaches the platform from `auth` — only reads. Budget: 16 reads (`[function.limits] authorizeReads`, up to 64), enforced — exceeding it denies. CPU is measured against a 50 ms target (`authorizeCpuMs`) and logged, not separately enforced; the function's own `cpuMs` ceiling stops a real spin. `authorize` runs under the call's own wall-clock timeout — past it, deny — and the handler keeps its own full budget after an allow.

### Helpers

| Helper | Resolves | Example |
|---|---|---|
| `allowAnyone()` | `null` | `authorize: allowAnyone()` |
| `requireSignedIn()` | `{ userId }` | `authorize: requireSignedIn()` |
| `requireRole(role)` | `{ userId, role }` | `authorize: requireRole("admin")` — ranked owner > admin > member |
| `requireMember(groupIdOf, role?)` | `{ userId, groupId, role }` | `authorize: requireMember((input) => input.teamId, "manager")` |
| `requireCollectionRole(collectionIdOf, minRole)` | `{ userId, collectionId, role }` | `authorize: requireCollectionRole((input) => input.collectionId, "editor")` |
| `allowCallers(rules)` | varies | admits by door, see below |

A selector denies `INVALID` when it can't read an id off the input. Helpers are ordinary `async (input, auth) => decision` functions — call one from inside a custom `authorize` and `await` its decision to compose checks.

`allowCallers({ webhook?, cron?, manual?, db3?, functions?: string[], agents?: boolean | AuthorizeFn, users?: AuthorizeFn })` admits by the caller's door: `users` decides `http` calls only; `agents: true` applies the `users` rule to the agent tool/context/history doors too, or pass a function to decide them separately; `webhook`/`cron`/`manual`/`db3` admit on `true`; `functions` lists the parent function keys admitted for a nested start. A door with no rule, including one added later, is denied:

```ts
import { allowCallers, defineFunction, requireRole } from "primitive-functions";

export default defineFunction("daily-sweep", {
  authorize: allowCallers({ cron: true, manual: true, users: requireRole("admin") }),
  handler: async () => ({ swept: true }),
});
```

`agents: true` lets one check cover both a member's own call and the same function run as an agent tool for that member's turn:

```ts
import { allowCallers, defineFunction, requireRole } from "primitive-functions";

export default defineFunction("send-reminder", {
  input: ["householdId"],
  authorize: allowCallers({ agents: true, users: requireRole("admin") }),
  handler: async (input: { householdId: string }) => {
    await sendReminder(input.householdId);
    return { sent: true };
  },
});
```

An admin's own call and a tool call driven by an admin's turn both pass `users`; a tool call driven by a member's turn is denied on either door. Give `agents` a function of its own to admit a caller the `users` rule would refuse.

### `ctx.reauthorize()`

A task run's `ctx.auth` is the value resolved at its start, on every step and every resume after a `step.sleep`, whatever changed since. `ctx.reauthorize()` runs `authorize` again for the same user, with the same input, against current data: it resolves the fresh value (`ctx.auth` is unchanged) or throws the deny as `fail` would, with its code. It runs under the same read budget, and works in a request handler too:

```ts
import { defineFunction, requireMember } from "primitive-functions";

export default defineFunction("team-digest", {
  authorize: requireMember((input: { teamId: string }) => input.teamId),
  handler: async (input, ctx, step) => {
    await step.sleep("collect", "1 hour");
    await ctx.reauthorize(); // throws FORBIDDEN once the caller has left the team
    return step.do("send", async () => ({ team: ctx.auth.groupId }));
  },
});
```

### `fail`

Call `fail(code, message?)` from a handler to end it with your own code; the caller gets `{ errorCode: code, error: message ?? code }`. Any non-empty string is a valid code. Called from inside `authorize` instead, it throws and denies the caller the same uniform way.

### Testing `authorize` without running the handler

`primitive functions authorize <key>` runs only `authorize` — the handler never runs — through an admin-only endpoint, and prints the decision:

```bash
primitive functions authorize <key> --user <id> --role <role> --trigger <door> --runtime request --input '{"classId":"c-1"}'
primitive functions authorize <key> --cases          # run functions/<key>.auth.json
primitive functions authorize --cases --all          # run every case file in the tree
```

| Flag | |
|---|---|
| `--user <id>` | `auth.user`: an app user id or an `@fixture` alias. Default: no user |
| `--role <role>` | Overrides `auth.user.role` for the check; memberships are unchanged |
| `--trigger <door>` | `http` (default) \| `webhook` \| `cron` \| `manual` \| `context` \| `history` \| `db3` \| `function:<key>` \| `tool`\|`tool:<name>` |
| `--runtime <runtime>` | `request` \| `task`; defaults to the door's own |
| `--input <json>` | Default `{}`; `@fixture` aliases resolve inside it |
| `--json` | Machine-readable output |

`--trigger manual` needs `--user` (the admin who'd start the run) and defaults `--role` to `admin`; `--trigger function:<key>` needs the parent function's key; a bare `--trigger tool` defaults the tool name to the function's own key. Exit codes: `0` allow, `1` deny (`authorize-error` — a throw, timeout, or bad return value — included), `2` no check ran (a refusal, or a version with no `authorize`).

**Case files**: `functions/<key>.auth.json`, a JSON array of `{ name, user?, role?, trigger?, runtime?, input?, expect: "allow"|"deny", code?: "FORBIDDEN"|"NOT_FOUND"|"INVALID" }` (`name` required and unique; `code` only with `"expect": "deny"`). `--cases`/`--cases --all` print a table (`CASE EXPECT GOT CODE MS READS RESULT`, `FUNCTION` added for `--all`) and a `N passed, M failed` line, and exit non-zero on a failure. Not a push gate — wire it into your own CI step.

**Fixture aliases**: a bare `"@name"` in `--user`/`user` or anywhere inside `--input`/`input` resolves from a fixture map; `"@@"` escapes a literal `@`. Never uploaded — the bundler refuses a build that reads one. A committed environment's map is `primitive/<env>/auth-fixtures.json`, committed beside `app.toml`. A [child app](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md)'s map is its own, machine-local file beside its sync baseline (`<slug>.auth-fixtures.json`) — never the parent's, not even as a fallback, because a child shares the tree but not the data; `apps children delete` removes it with the child's other local state.

**Testing a new `authorize` in a child app**: create one (`primitive apps children create <slug>`), push the branch into it, run `--cases`/`--cases --all` against it, iterate, then select the parent and push the merged tree — the same loop as any other config change.

**Push lints** (`config push`, on a build with `authorize`): warns when `authorize` reads neither `auth.user` nor `auth.trigger` (use `allowAnyone()` if that's deliberate); warns on an `inputSchema` string with no `maxLength` or array with no `maxItems`, by path; warns when a key in `input` is also a key of an object literal passed to `allow(…)` (`'<key>' is both declared in input and resolved by authorize`): read `ctx.auth.<key>` and drop the key from `input`, unless the handler needs the caller's own value — declare what the caller sent and the handler merely needs, resolve what `authorize` checked or loaded; refuses outright when `authorize` admits anyone (`allowAnyone()`, or an admitting `allowCallers` arm) and the version is write-capable — a write/send family the manifest records, or a declared `integration:`/high-blast capability — unless `[function] public = true` is set. Without it the push is refused `FUNCTION_PUBLIC_WRITE_UNDECLARED`. `public = true` also opens the function to callers who are not signed in when the app sets `[app] signedOutAccess = true`; `auth.user` is then `{ userId: "", email: null, role: "anonymous" }`. A version that declares it without `authorize` is refused `FUNCTION_PUBLIC_REQUIRES_AUTHORIZE`.

## Surfaces and their extra context

| Surface | Rule controls | Default for regular members |
|---|---|---|
| Function `authorize` | Who can invoke it, decided in code | Required — push refuses a new or changed version with none |
| Blob bucket preset or rule set | Reads, uploads, listing, deletion, sharing | A preset or rule set is required |
| Metadata `readRule` / `writeRule` | Reading or writing category values | Denied without the rule |

Resource rules add context such as `record.blobCreatedBy`. See the feature guide for its variables. Metadata rules govern API reads and writes, not another rule's metadata lookups.

Groups and collections have no rule set: every group and collection has the same fixed, platform-enforced roles — see [Users and Groups](AGENT_GUIDE_TO_PRIMITIVE_USERS_AND_GROUPS.md#group-roles); collections: owner, manager, editor, contributor, viewer. A rule the roles don't express goes in a server function's `authorize` (`requireMember`, `requireCollectionRole`, above).

A function’s gate also authorizes its use of prompts, integrations and named locks, which carry no rule of their own; integrations require the declared capability. Admin routes and notification sending use app roles rather than CEL.

[Metadata](AGENT_GUIDE_TO_PRIMITIVE_RESOURCE_METADATA.md) supplies `md.self.*` and declared related-resource paths. [Secrets and config variables](AGENT_GUIDE_TO_PRIMITIVE_APP_SECRETS.md) must be declared before a rule can read them.

## Rule sets (management operations)

A function's `authorize` method decides who may run its code; **rule sets** govern who may manage platform resources directly — blob buckets. Groups and collections don't use one: every group and collection has the same fixed, platform-enforced roles (groups: owner, manager, member; collections: owner, manager, editor, contributor, viewer — see [Users and Groups](AGENT_GUIDE_TO_PRIMITIVE_USERS_AND_GROUPS.md#group-roles)), and `config push` refuses a `group-type-configs/` or `collection-type-configs/` file. Put a rule those roles don't express in a server function whose `authorize` uses `requireMember` or `requireCollectionRole` (above).

```toml
# primitive/dev/rule-sets/bucket-access.toml
[ruleSet]
name = "bucket-access"
resourceType = "blob_bucket"

[rules.blob]
read = "!isAnonymous()"
write = "!isAnonymous()"
delete = "record.blobCreatedBy == user.userId"
```

```bash
primitive config push --only rule-set/bucket-access
```

A **blob bucket** attaches a rule set via `ruleSetName` in its TOML — see the Blob Buckets guide for precedence semantics.

Semantics:
- **App owners/admins bypass rule sets entirely**; rules apply to regular members.
- A blob bucket with no rule set bound falls back to its preset.
- To deny an op for everyone except admins/owners, set that op's rule to `"false"`.

**Subject-form membership in a metadata rule** — a category's `readRule`/`writeRule` can ask which groups a *loaded subject* belongs to, not just the caller: `memberGroupsOf(<subjectId>, '<groupType>')` returns that subject's list of groupIds of the type (a deprecated alias kept for metadata rules; new code checks membership with `isMemberOf`/`memberGroups`, one argument). Compose with `.exists()`/`.all()` to follow a relationship graph, e.g. grant a teacher of any class a student's metadata links to:

```toml novalidate
readRule = "memberGroupsOf(md.self.classLink.classId, 'class-students').exists(c, isMemberOf(c))"
```

`memberGroupsOf`'s subject must be a literal path such as `md.self.<category>.<key>`, and its type a literal string.

## Testing and debugging

```bash
primitive rule-sets test <rule-set-id> --scenario '{...}'   # dry-run against a simulated scenario
primitive rule-sets schema                                   # available context variables/functions
primitive functions invoke <key> --user <user-id>           # exercise a function's authorize as that app user
```

For end-to-end checks, sign in as `+primitivetest` derived users with different roles/memberships and invoke the function. Owners and admins bypass rule sets, metadata rules, and bucket presets — so a gated or entitlement-gated state reads as open to them even when the gate is correct, and `primitive functions invoke` without `--user` runs as your own admin app user. Test that a gate actually denies as a plain `member`. An `authorize` method has no such bypass — invoking it as an admin exercises it exactly as any other caller would.

## Gotchas

- **Resolve provider IDs from the caller.** Read a trusted user-to-provider mapping using `ctx.user`; accepting a client’s provider ID could expose another account.
- **Declare optional variables.** An undeclared or dynamic reference (`vars.?KEY`) is refused at save; guard an optional variable with `'KEY' in vars`.
- **`authorize` gives owners and admins no bypass.** Admit them with an explicit check (`requireRole("admin")` or `if (auth.hasRole("admin")) return allow();`).
- **A module-scope `assertRuntime` hides behind a uniform deny.** `authorize` shares the handler's module scope: a task-only function's module-scope `assertRuntime("task")` throwing while `authorize` loads settles as an `authorize-error` (uniform `403`). Call it the runtime it expects.
- **Agent calls answer their own way.** An agent-call deny is `AGENT_TOOL_ACCESS_DENIED`; on context/history it fails the turn.
- **Deep in a chain, `ctx.user` is still the root.** Inside a nested start, `ctx.user` names the member who started the chain's first function, never whoever called the function directly above you. Scope by what this function's own `authorize` resolved into `ctx.auth`, not by re-trusting `ctx.user` at each level.
- **Write `authorize` above `handler`.** `ctx.auth` is typed from `authorize`'s `allow` with no annotations, but only when `authorize` comes first: after the handler, an unannotated `authorize` arrow leaves `ctx.auth` typed `WriteAuthorizeBeforeHandler | null`, and every field read fails to compile.
- **A task run's `ctx.auth` is capped.** Over 64 KB, the task start fails `500 FUNCTION_AUTH_TOO_LARGE` (a cron fire records a failed run); over 16 KB, each step of the run logs a warning. Resolve ids and the rows the handler needs.
- **A flagged `inputSchema` defaults closed.** A build with `authorize` defaults `additionalProperties: false` on an `inputSchema` that doesn't set it; a schema whose `anyOf`/`oneOf` members declare their own `properties` is left open instead, and push warns.
- **A trigger-door handler is still typed to the list.** The compiler can't tell the door, so list the keys a webhook, cron, db3, walk or hook handler reads (it gets them anyway), or cast `input`.
- **A platform-built body must pass the closed schema.** On a webhook, db3, walk or agent hook door, set `additionalProperties = true` under `[function.inputSchema]` or declare every key the platform sends; otherwise the call is refused as invalid before `authorize` runs.

## Recipe: subscription entitlement

Use a group membership as the entitlement, and update it from verified billing events.

**1. The entitlement is a group; `authorize` checks membership.** A CEL rule can't read a row (above), so the paid-feature gate is membership in an entitlement group, never an `isSubscribed` column. Create the group once and admit it with `requireMember` on every gated function:

```toml novalidate
# functions/export-report.toml — each function behind the paywall
[function]
key = "export-report"
entry = "functions/export-report/index.ts"
```

```ts
import { defineFunction, requireMember } from "primitive-functions";

export default defineFunction("export-report", {
  authorize: requireMember(() => "01JD7EB2C8H4K6M9P3R5S7T1V0"),
  handler: async (input, ctx) => { /* ... */ },
});
```

**2. A verified webhook maintains that membership.** The billing provider posts subscription-lifecycle events to a function's webhook trigger; the function flips the entitlement with `ctx.api.groups.addMember({ groupId, body: { userId } })` / `ctx.api.groups.removeMember({ groupId, userId })`, `groupId` being the entitlement group's id:

```toml novalidate
# functions/stripe-events.toml
[function]
key = "stripe-events"
entry = "functions/stripe-events/index.ts"

[function.triggers.webhook]
verificationScheme = "stripe"
signingSecret = "{{secrets.STRIPE_WEBHOOK_SECRET}}"
```

```ts
authorize: allowCallers({ webhook: true }),   // no member may invoke it; only the webhook door is admitted
```

What stops a forged call is the webhook's signature verification — a delivery that fails it never runs the function — and an `authorize` that admits only the `webhook` door, so nobody can invoke the handler over HTTP. See the [Server Functions guide](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#triggers).

**3. Provider ids live in a store only functions write.** Map the provider's `customer_id` to the user in a database the webhook function writes, and read it back inside a function by `ctx.user`, so a caller only ever reaches their own row. Never accept a client-supplied `customer_id`, and the same for any internal id a function would otherwise take from its input (see External identifiers, above).

**4. Checkout and customer-portal functions return provider URLs.** Resolve the signed-in user’s provider ID, call the integration, and return the URL. These functions do not grant membership; the verified webhook does.
