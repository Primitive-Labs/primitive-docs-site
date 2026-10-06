# Agent Guide to Primitive Access Control

Use CEL rules to authorize function calls and resource operations. Rules see caller identity; each resource adds its own context. Test authorization as a regular member because app owners and admins bypass rules.

## First rule

```toml novalidate
access = "isMemberOf('team', 'engineering')"
```

This function allows members of the engineering team. Inside the function, still authorize requested resources and scope records to the caller.

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

Function access rules can check roles and group membership, but cannot query database records. Store entitlements as group memberships when they must be checked by the access rule.

## Surfaces and their extra context

| Surface | Rule controls | Default for regular members |
|---|---|---|
| Function `access` | Who can invoke or start it | Denied without a rule |
| Group or collection rule set | Resource and membership operations | Permissive when no type config exists |
| Blob bucket preset or rule set | Reads, uploads, listing, deletion, sharing | A preset or rule set is required |
| Database type rule set | Editing or deleting type configuration | Denied without a rule set |
| Lock rule set | Acquiring and renewing keys | Open without a rule set; omitted operations deny once installed |
| Metadata `readRule` / `writeRule` | Reading or writing category values | Denied without the rule |

Resource rules add context such as `group.*`, `collection.*`, or `record.key`. See the feature guide for its variables. Metadata rules govern API reads and writes, not another rule’s metadata lookups.

A function’s gate also authorizes its use of prompts and integrations; integrations require the declared capability. Admin routes and notification sending use app roles rather than CEL.

[Metadata](AGENT_GUIDE_TO_PRIMITIVE_RESOURCE_METADATA.md) supplies `md.self.*` and declared related-resource paths. [Secrets and config variables](AGENT_GUIDE_TO_PRIMITIVE_APP_SECRETS.md) must be declared before a rule can read them.

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

Bind via the type config: `primitive/<env>/group-type-configs/<type>.toml` / `primitive/<env>/collection-type-configs/<type>.toml` (synced), or `client.groupTypeConfigs.create({ groupType, ruleSetId })` / `client.collectionTypeConfigs.create(...)`. A **blob bucket** attaches a rule set directly via `ruleSetName` in its TOML, where it governs member-level reads/writes — see the Blob Buckets guide for precedence semantics.

Semantics:
- **App owners/admins bypass rule sets entirely**; rules apply to regular members.
- Group types with no config row get permissive defaults: any member can `create`; the creator can `edit`/`delete` and manage members; creator + direct members can read.
- To deny an op for everyone except admins/owners, set that op's rule to `"false"`.

**Subject-form membership** — a group rule can ask which groups the *managed group* belongs to, not just the caller: `memberGroupsOf(group.groupId, '<groupType>')` returns that group's list of groupIds of the type. Compose with `.exists()`/`.all()` to follow a relationship graph, e.g. grant a teacher of any class the group is enrolled in:

```toml novalidate
get = "memberGroupsOf(group.groupId, 'class-students').exists(c, isMemberOf('class-teachers', c))"
```

`memberGroupsOf` requires the literal `group.groupId` path and a literal group type. Use it for an existing group, not `group.create`. Metadata category rules can use a loaded `md.self.*` value as the subject.

## Testing and debugging

```bash
primitive rule-sets test <rule-set-id> --scenario '{...}'   # dry-run against a simulated scenario
primitive rule-sets debug --user <userId> --group-type <type> --category <group|member> --operation <create|edit|delete|list> [--group-id <id>]
                                                            # trace against real user/group data (console-admin only)
primitive rule-sets schema                                   # available context variables/functions
primitive functions invoke <key> --user <user-id>           # exercise a function's access gate as that app user
```

Client equivalents: `client.ruleSets.test()`, `client.ruleSets.debug()`, `client.ruleSets.schema()`. For end-to-end checks, sign in as `+primitivetest` derived users with different roles/memberships and invoke the function. Owners and admins bypass function `access` gates, rule sets, metadata rules, and bucket presets — so a gated or entitlement-gated state reads as open to them even when the gate is correct, and `primitive functions invoke` without `--user` runs as your own admin app user. Test that a gate actually denies as a plain `member`.

## Gotchas

- **Test as a member.** Owners and admins bypass access rules.
- **Scope function data in code.** The access rule cannot inspect function input. Validate requested resource IDs and check the caller’s membership or ownership before using them.
- **Resolve provider IDs from the caller.** Read a trusted user-to-provider mapping using `ctx.user`; accepting a client’s provider ID could expose another account.
- **Keep trigger-only functions closed to members.** Use `access = "false"` when only a verified webhook should invoke the function. Trigger fires do not evaluate the function gate.
- **Declare optional variables.** Missing secrets or variables can deny access; guard optional variables with `'KEY' in vars` or `vars.?KEY`.

## Recipe: subscription entitlement

Use a group membership as the entitlement, and update it from verified billing events.

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

**4. Checkout and customer-portal functions return provider URLs.** Resolve the signed-in user’s provider ID, call the integration, and return the URL. These functions do not grant membership; the verified webhook does.
