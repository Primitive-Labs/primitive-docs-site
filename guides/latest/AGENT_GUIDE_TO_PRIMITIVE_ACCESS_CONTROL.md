# Agent Guide to Primitive Access Control

Guidelines for AI agents writing access rules. Server-evaluated authorization across Primitive is expressed in **CEL** (Common Expression Language) against the authenticated caller. Documents are the exception — they use direct permission grants (`reader` | `read-write` | `owner` per user/email/group), not CEL.

## Identity context (available everywhere)

| Variable / function | Meaning |
|---|---|
| `user.userId` | Caller's user ID (`''` when unauthenticated) |
| `user.role` | Caller's app role |
| `isAnonymous()` | True when the caller has no account (unauthenticated request); `!isAnonymous()` ⇒ any signed-in member. Matters where anonymous access is reachable, e.g. a `public` blob bucket |
| `hasRole(role)` | App role check: `"owner"` \| `"admin"` \| `"member"` (app-level role — distinct from the document `owner` permission) |
| `isMemberOf(groupType, groupId)` | Exact group membership (two args, strict) |
| `memberGroups(groupType)` | List of groupIds of that type the caller belongs to |

Prefer membership checks over user-ID comparisons: `isMemberOf('team', params.teamId)`, `params.teamId in memberGroups('team')`.

A rule reads caller identity and operation params only — it cannot read a database record. So a server-enforced entitlement (paid feature, role gate) can't be a flag on a row (`isSubscribed = true`): no rule can check it, so it isn't enforced. Model the entitlement as group membership, which a rule can check — `isMemberOf('subscription', 'active')` — and maintain that membership from your billing source.

## Surfaces and their extra context

| Surface | Field(s) | Extra context | Notes |
|---|---|---|---|
| Database operation | `access` per op; `defaultAccess` on `[type]`; per-param `access` inside `params` JSON | `params.*`, `database.id`, `database.celContext`, `fromWorkflow()` / `fromWorkflow(key)` | No rule and no `defaultAccess` → denied to every caller (operation `access` has no owner/manager/admin bypass). `fromWorkflow(key)` gates ops to the internal workflow runner (unspoofable). |
| Database subscription | `accessRule` (subscribe-time) + `filter` (per change) | `accessRule`: full context incl. `database.*`, membership fns, `params.*`. `filter`: **narrow** — `user.userId`, `record.*` (`record.data.*`, `record.previousData.*`, `record.modelName`/`op`/`id`), `params.*` only | `filter` cannot reference `database.*` (HTTP 400 at save) and cannot widen what `accessRule` allows. Both required; use `"true"` for filter to pass everything in scope. |
| Workflow | `accessRule` on `[workflow]` | identity context only | Evaluated on `workflows.start()` and `workflow.call`; NOT on webhook triggers. admin/owner bypass. No rule → any authenticated member can start. |
| Server-stamped fields / triggers | trigger `when` conditions; `autoPopulatedFields` values | `record.*`, `database.*`, `now()` | CEL produces values (`user.userId`, `now()`) as well as conditions. |
| Rule sets | per-op rules on a named rule set | resource objects (e.g. `group.groupType`, `group.groupId`, `group.createdBy`) | Governs **management operations** (create/edit/delete groups & collections, member management) — see below. |
| Metadata category | `readRule` (read API) / `writeRule` (write API) on the category config | `user.userId`, `user.role`, `resource.resourceType`/`resourceId`/`category`; `workflow.workflowKey` when called from a `metadata.write`/`read` step | Gates the metadata read/write API only — a *different* rule's `md.self`/`md.caller` use is authorized by declaration, not this rule. App-level owner/admin bypass both rules (a resource-level permission never bypasses). See the Resource Metadata guide. |

Any eval site with a declared metadata manifest also gains `md.self.*` (and, where declared, `md.<path>.*` / `md.caller.*`) in its CEL context — see the [Resource Metadata guide](AGENT_GUIDE_TO_PRIMITIVE_RESOURCE_METADATA.md).

## Rule sets (management operations)

Data-access rules live on operations; **rule sets** govern who may manage groups and collections:

```bash
primitive rule-sets create "team-management" \
  --resource-type group \
  --rules '{
    "group":  { "create": "true", "edit": "user.userId == group.createdBy", "delete": "user.userId == group.createdBy" },
    "member": { "create": "isMemberOf(group.groupType, group.groupId)", "edit": "user.userId == group.createdBy", "delete": "user.userId == group.createdBy" }
  }'
```

Bind via the type config: `config/group-type-configs/<type>.toml` / `config/collection-type-configs/<type>.toml` (synced), or `client.groupTypeConfigs.create({ groupType, ruleSetId })` / `client.collectionTypeConfigs.create(...)`. A **blob bucket** attaches a rule set directly via its `ruleSetId` (TOML, or `--rule-set-id` on `primitive blob-buckets create`), where it governs member-level reads/writes — see the Blob Buckets guide for precedence semantics.

Semantics:
- **App owners/admins bypass rule sets entirely**; rules apply to regular members.
- Group types with no config row get permissive defaults: any member can `create`; the creator can `edit`/`delete` and manage members; creator + direct members can read.
- To deny an op for everyone except admins/owners, set that op's rule to `"false"`.

## Testing and debugging

```bash
primitive rule-sets test <rule-set-id> --scenario '{...}'   # dry-run against a simulated scenario
primitive rule-sets debug --user <userId> --group-type <type> --category <group|member> --operation <create|edit|delete|list> [--group-id <id>]
                                                            # trace against real user/group data (console-admin only)
primitive rule-sets schema                                   # available context variables/functions
```

Client equivalents: `client.ruleSets.test()`, `client.ruleSets.debug()`, `client.ruleSets.schema()`. For end-to-end checks of operation rules, sign in as `+primitivetest` derived users with different roles/memberships and execute the operation. Owners and admins bypass workflow access rules, rule sets, metadata rules, and bucket presets — but NOT database operation `access`, which is evaluated for every caller — so on a bypassing surface a gated or entitlement-gated state reads as open to them even when the gate is correct. Test that a gate actually denies as a plain `member`.

## Patterns

```toml novalidate
access = "user.userId != ''"                                   # any authenticated user
access = "hasRole('admin') || hasRole('owner')"                # app admins
access = "isMemberOf('team', database.celContext.teamId)"      # bound team per database instance
access = "params.createdBy == user.userId"                     # record owner
access = "params.teamId in memberGroups('team')"               # any of the caller's teams
access = "fromWorkflow('refresh-prices')"                      # only the named workflow
```

- Default-deny, widen deliberately. Missing rules deny (database ops) or allow any member (workflows) — know which surface you're on.
- One group type per concept (`team`, `org`, `team-admin`); group-level "admin" is modeled as its own group type, not a built-in.
- Parameterize the resource (`params.teamId`) so one operation serves every team.
- External identifiers: never trust a client-supplied provider id (a payment `customer_id`) for a server-side action — a caller could substitute another user's id. Keep the user→external-id mapping in a system-write store (write op gated `access = "fromWorkflow('key')"`) with reads scoped to the caller (`definition` filter on `$user.userId`), and resolve the id server-side from the authenticated user.

## Recipe: subscription entitlement

A paid-SaaS feature gate (Stripe-style billing) ties together four pieces — the rule, the membership that backs it, the provider-id store, and the user-facing checkout. The whole shape:

**1. The entitlement is a group; the rule checks membership.** A rule can't read a row (above), so the paid-feature gate is membership in an entitlement group, never an `isSubscribed` column. Put the check on every gated operation:

```toml novalidate
# on each operation behind the paywall
access = "isMemberOf('subscription', 'active')"
```

**2. A verified webhook maintains that membership.** The billing provider posts subscription-lifecycle events to an inbound webhook bound to a `runAs = "system"` workflow that holds the `membership` capability; the workflow flips the entitlement with `group.addMember` / `group.removeMember`:

```toml novalidate
# config/workflows/process-stripe.toml — see the Workflows guide for the full step shape
[workflow]
key = "process-stripe"
runAs = "system"                # webhook-triggered workflows are always system
capabilities = ["membership"]   # required for group.addMember/removeMember in a system run
```

The webhook declaration (`verificationScheme = "stripe"`, `workflowKey = "process-stripe"`) and the group step kinds live in the [Workflows guide](AGENT_GUIDE_TO_PRIMITIVE_WORKFLOWS.md). What stops a forged client call is the signature check plus the system-invocation gate (members get a 403) — not `accessRule`, which a system workflow doesn't evaluate on a webhook trigger.

**3. Provider ids live in a system-write store, read back per user.** Map the provider's `customer_id` to the user in a database whose **write** op is workflow-only and whose **read** op is scoped to the caller, so only the webhook writes it and a caller sees only their own row:

```toml novalidate
[[operations]]                  # webhook records the mapping (a save mutation)
name = "recordBillingCustomer"
type = "mutation"
access = "fromWorkflow('process-stripe')"

[[operations]]                  # caller reads only their own — filter, not trust
name = "myBillingCustomer"
type = "query"
access = "true"
[operations.definition]
filter = { userId = "$user.userId" }
limit = 1
```

Resolve the provider id server-side from the authenticated user — never accept a client-supplied `customer_id` (see External identifiers, above).

**4. Checkout and customer-portal are caller-run workflows.** "Start a subscription" and "manage billing" are ordinary `runAs = "caller"` workflows that call the provider's API (the outbound integration — see the [Integrations guide](AGENT_GUIDE_TO_PRIMITIVE_INTEGRATIONS.md)) to mint a Checkout or Billing-Portal session URL for the signed-in user and return it. They grant no access themselves; membership changes only when the provider's webhook (step 2) reports the subscription started or ended.

Net: the rule (1) trusts only group membership; membership (2) is written only by the verified webhook; the provider id (3) is never client-trusted; the checkout/portal calls (4) never grant access directly.
