# Agent Guide to Primitive Access Control

Guidelines for AI agents writing access rules. Server-evaluated authorization in Primitive is expressed in **CEL** (Common Expression Language) against the authenticated caller, and CEL has two jobs: a **server function's `access` gate** (who may invoke or start the function — the whole authorization for everything the function does) and **rule sets** over the platform's own catalogues (groups, collections, blob buckets, database types, named locks). Documents are the exception — they use direct permission grants (`reader` | `read-write` | `owner` per user/email/group), not CEL.

## Identity context (available everywhere)

| Variable / function | Meaning |
|---|---|
| `user.userId` | Caller's user ID (`''` when unauthenticated) |
| `user.role` | Caller's app role |
| `isAnonymous()` | True when the caller has no account (unauthenticated request); `!isAnonymous()` ⇒ any signed-in member. Matters where anonymous access is reachable, e.g. a `public` blob bucket |
| `hasRole(role)` | App role check: `"owner"` \| `"admin"` \| `"member"` (app-level role — distinct from the document `owner` permission) |
| `isMemberOf(groupType, groupId)` | Exact group membership (two args, strict) |
| `memberGroups(groupType)` | List of groupIds of that type the caller belongs to |

Prefer membership checks over user-ID comparisons: `isMemberOf('team', 'core')`, `size(memberGroups('team')) > 0`.

A rule reads caller identity only (plus the resource object a rule set manages) — it cannot read a database record. So a server-enforced entitlement (paid feature, role gate) can't be a flag on a row (`isSubscribed = true`): no rule can check it, so it isn't enforced. Model the entitlement as group membership, which a rule can check — `access = "isMemberOf('subscription', 'active')"` on each gated function — and maintain that membership from your billing source.

## Surfaces and their extra context

| Surface | Field(s) | Extra context | Notes |
|---|---|---|---|
| Server function | `access` on `[function]` | identity context only — no input, no params | Evaluated on every HTTP invoke and start. admin/owner bypass. **No rule → denied**; push refuses code for a function with no gate. A rule that throws also denies. Denial: `403 { errorCode: "FUNCTION_ACCESS_DENIED" }`, the same body for every cause, and it is checked before any answer about the function's state. The gate IS the authorization: inside, the code acts with the app's authority, so scope reads and writes in code on `ctx.user`. Not evaluated for a trigger fire (webhook, cron) or for a callee started with `ctx.functions.start` — the root's gate was the authorization. |
| Prompt, integration | — | — | No rule of their own and no client endpoint: only a function reaches them (`ctx.prompts.run`, `ctx.integrations.call`), so the function's `access` gate is their authorization, plus the function's `integration:<key>` capability for an integration. |
| Rule sets | per-op rules on a named rule set | resource objects (e.g. `group.groupType`, `group.groupId`, `group.createdBy`; `collection.*`) | Governs **management operations** (create/edit/delete groups & collections, member management) — see below. |
| Blob bucket | `preset`, or `ruleSetId` naming a rule set | per-operation blob context | Member-level reads/writes on the bucket's blobs. See the Blob Buckets guide. |
| Database type | `ruleSetName` on the database type config | identity context | Who may edit or delete the type's configuration. No rule set → denied for non-admins. |
| Named locks (`acquire`, `renew`) | per-op rules on a single `lock` rule set | `record.key` — the exact requested key, scopeable by prefix or `isMemberOf(...)`; `user.userId`, `user.role` | No `lock` rule set installed → open (any member, any key) — the default. Once installed, every operation it does not define is denied for members; at most one `lock` rule set per app. `release` is never rule-gated. Denial: `403 { errorCode: "LOCK_ACCESS_DENIED" }`. See the [Locks guide](AGENT_GUIDE_TO_PRIMITIVE_LOCKS.md#access-control). |
| Metadata category | `readRule` (read API) / `writeRule` (write API) on the category config | `user.userId`, `user.role`, `resource.resourceType`/`resourceId`/`category` | Gates the metadata read/write API only — it doesn't govern a *different* rule's `md.self`/`md.caller` use (`md.self` is inferred; `md.caller`/paths are declared). App-level owner/admin bypass both rules (a resource-level permission never bypasses). See the Resource Metadata guide. |
| Server-stamped fields / triggers | trigger `when` conditions; `autoPopulatedFields` values | `record.*`, `database.*`, `now()` | CEL produces values (`user.userId`, `now()`) as well as conditions. |
| Direct LLM/Gemini routes | `directLlmEnabled` on `[app]` (`app.toml`) | — | No per-resource rule; one app-wide switch, **off by default**. While it is off the four spend routes (`llm/chat`, `gemini/generate`, `gemini/generate-raw`, `gemini/count-tokens`) answer `403 { code: "DIRECT_LLM_DISABLED" }` to every caller, app admins and owners included; `directLlmEnabled = true` opts in. A prompt run with `ctx.prompts.run` does not go through the switch. |
| Notification send, admin routes | route role | — | Not CEL: `notifications.send` and the administrative routes require the app admin role. A function run is readable by the member who started its tree. |

Any manifest-supporting eval site gains `md.self.*` in its CEL context — self-category reads are inferred, no declaration needed (and, where declared, `md.<path>.*` / `md.caller.*`) — see the [Resource Metadata guide](AGENT_GUIDE_TO_PRIMITIVE_RESOURCE_METADATA.md).

## Rule sets (management operations)

A function's `access` gate decides who may run its code; **rule sets** govern who may manage platform resources directly — groups, collections, blob buckets, database types, locks:

```toml
# primitive/dev/rule-sets/team-management.toml
[ruleSet]
name = "team-management"
resourceType = "group"

[rules.group]
create = "true"
edit = "user.userId == group.createdBy"
delete = "user.userId == group.createdBy"

[rules.member]
create = "isMemberOf(group.groupType, group.groupId)"
edit = "user.userId == group.createdBy"
delete = "user.userId == group.createdBy"
```

```bash
primitive config push --only rule-set/team-management
```

Bind via the type config: `primitive/<env>/group-type-configs/<type>.toml` / `primitive/<env>/collection-type-configs/<type>.toml` (synced), or `client.groupTypeConfigs.create({ groupType, ruleSetId })` / `client.collectionTypeConfigs.create(...)`. A **blob bucket** attaches a rule set directly via its `ruleSetId` in TOML, where it governs member-level reads/writes — see the Blob Buckets guide for precedence semantics.

Semantics:
- **App owners/admins bypass rule sets entirely**; rules apply to regular members.
- Group types with no config row get permissive defaults: any member can `create`; the creator can `edit`/`delete` and manage members; creator + direct members can read.
- To deny an op for everyone except admins/owners, set that op's rule to `"false"`.

**Subject-form membership** — a group rule can ask which groups the *managed group* belongs to, not just the caller: `memberGroupsOf(group.groupId, '<groupType>')` returns that group's list of groupIds of the type. Compose with `.exists()`/`.all()` to follow a relationship graph, e.g. grant a teacher of any class the group is enrolled in:

```toml novalidate
get = "memberGroupsOf(group.groupId, 'class-students').exists(c, isMemberOf('class-teachers', c))"
```

Both arguments are validated at save time: argument 1 must be the literal path `group.groupId` (a non-literal/computed arg is rejected, same class as a dynamic `md[expr]` access), argument 2 a string literal. It is allowed only on existing-group operations (`get`/`edit`/`delete`, `member.*`) — **not** `group.create`, where `group.groupId` is caller-supplied — and `target.userId` is never a valid subject. Metadata category rules get the same function rooted at a loaded `md.self.*` value.

## Testing and debugging

```bash
primitive rule-sets test <rule-set-id> --scenario '{...}'   # dry-run against a simulated scenario
primitive rule-sets debug --user <userId> --group-type <type> --category <group|member> --operation <create|edit|delete|list> [--group-id <id>]
                                                            # trace against real user/group data (console-admin only)
primitive rule-sets schema                                   # available context variables/functions
primitive functions invoke <key> --user <user-id>           # exercise a function's access gate as that app user
```

Client equivalents: `client.ruleSets.test()`, `client.ruleSets.debug()`, `client.ruleSets.schema()`. For end-to-end checks, sign in as `+primitivetest` derived users with different roles/memberships and invoke the function. Owners and admins bypass function `access` gates, rule sets, metadata rules, and bucket presets — so a gated or entitlement-gated state reads as open to them even when the gate is correct, and `primitive functions invoke` without `--user` runs as your own admin app user. Test that a gate actually denies as a plain `member`.

## Patterns

```toml novalidate
access = "user.userId != ''"                                   # any authenticated user
access = "hasRole('admin') || hasRole('owner')"                # app admins
access = "isMemberOf('subscription', 'active')"                # an entitlement group
access = "size(memberGroups('team')) > 0"                      # any of the caller's teams
edit = "user.userId == group.createdBy"                        # rule set: the group's creator
```

- Default-deny, widen deliberately. A function with no `access` gate denies every caller; an installed `lock` rule set denies every operation it does not define.
- One group type per concept (`team`, `org`, `team-admin`); group-level "admin" is modeled as its own group type, not a built-in.
- Gate the function, scope in the code. The gate cannot see input, so which team, document or row a caller may touch is decided inside the function — e.g. check `ctx.user` against the team's membership (`ctx.api.groups.listUserMemberships`) or filter on `ctx.user!.userId` — never by trusting an id from the input.
- External identifiers: never trust a client-supplied provider id (a payment `customer_id`) for a server-side action — a caller could substitute another user's id. Keep the user→external-id mapping in a database only your functions write, and resolve the id inside the function from `ctx.user` (`query({ filter: { userId: ctx.user!.userId } })`). The same rule holds for ids the app minted itself (a `householdId`, an `itemId`) once they arrive as function input rather than as something derived server-side — "external" describes where the id came from into this request, not who defined it.

## Recipe: subscription entitlement

A paid-SaaS feature gate (Stripe-style billing) ties together four pieces — the rule, the membership that backs it, the provider-id store, and the user-facing checkout. The whole shape:

**1. The entitlement is a group; the gate checks membership.** A rule can't read a row (above), so the paid-feature gate is membership in an entitlement group, never an `isSubscribed` column. Put the check on every gated function:

```toml novalidate
# functions/export-report.toml — each function behind the paywall
[function]
key = "export-report"
entry = "functions/export-report/index.ts"
access = "isMemberOf('subscription', 'active')"
```

**2. A verified webhook maintains that membership.** The billing provider posts subscription-lifecycle events to a function's webhook trigger; the function flips the entitlement with `ctx.api.groups.addMember({ groupType: "subscription", groupId: "active", body: { userId } })` / `ctx.api.groups.removeMember({ groupType: "subscription", groupId: "active", userId })`:

```toml novalidate
# functions/stripe-events.toml
[function]
key = "stripe-events"
entry = "functions/stripe-events/index.ts"
access = "false"                     # no member may invoke it; the webhook fire is not gated

[function.triggers.webhook]
verificationScheme = "stripe"
signingSecret = "{{secrets.STRIPE_WEBHOOK_SECRET}}"
```

What stops a forged call is the webhook's signature verification — a delivery that fails it never runs the function — and an `access` gate no member passes, so nobody can invoke the handler over HTTP. See the [Server Functions guide](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#triggers).

**3. Provider ids live in a store only functions write.** Map the provider's `customer_id` to the user in a database the webhook function writes, and read it back inside a function by `ctx.user`, so a caller only ever reaches their own row. Never accept a client-supplied `customer_id`, and the same for any internal id a function would otherwise take from its input (see External identifiers, above).

**4. Checkout and customer-portal are ordinary functions.** "Start a subscription" and "manage billing" are functions with `access = "user.userId != ''"` that resolve the caller's provider id (3) and call the provider's API through an integration (`ctx.integrations.call`, under an `integration:<key>` capability — see the [Integrations guide](AGENT_GUIDE_TO_PRIMITIVE_INTEGRATIONS.md)) to mint a Checkout or Billing-Portal session URL for the signed-in user and return it. They grant no access themselves; membership changes only when the provider's webhook (step 2) reports the subscription started or ended.

Net: the gate (1) trusts only group membership; membership (2) is written only by the verified webhook; the provider id (3) is never client-trusted; the checkout/portal functions (4) never grant access directly.
