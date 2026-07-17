# Agent Guide to Primitive Invitations

Guidelines for AI agents implementing app membership: access modes, invitations and quotas, deferred-grant resolution, and the invite-accept wiring.

## Core operations

### App invitations (quota / create / list / cancel)

```typescript
  // The caller's remaining invite quota (admins/owners are unlimited)
  const quota = await client.invitations.quota();

  // Invite someone to the app by email
  const invitation = await client.invitations.create({
    email: "alice@example.com",
    role: "member",
  });

  // Pending invitations
  const { items } = await client.invitations.list();

  // Cancel one
  await client.invitations.delete(invitationId);
```

### Accept an invite token

```typescript
  const result = await client.invitations.accept(inviteToken);
```

## Mental Model

App membership in Primitive is built from a small set of primitives the client cares about, plus several internal primitives the platform manages on your behalf.

**Client-facing primitives:**

| Primitive | Grants | Scope |
|-----------|--------|-------|
| `AppInvitation` | Right to join the app | App |
| `AppMembership` | Membership in an app | App |

**Internal (platform-managed):**

| Primitive | Purpose |
|-----------|---------|
| `DeferredDocumentPermission` | Tracks an email-based document share until the recipient signs up |
| `DeferredGroupAdd` | Tracks an email-based group or collection add until the recipient signs up |

The deferred types make email-based sharing work — the platform remembers the intent until a matching user account exists. **Do not interact with them directly in product code.** The sharing APIs (documents, groups, collections) accept emails transparently. `client.invitations.listDeferredGrants(...)` exists only for admin debugging tools.

**Decision rules:**

1. If you have a `userId` → write the direct membership/grant record.
2. If you only have an email → write the same record by email; the platform handles everything else. No manual "accept" call is needed for the standard email-match path — deferred grants resolve automatically when the recipient signs in with a matching email.
3. Use `client.invitations.accept(inviteToken)` only for cross-identity acceptance (invited at one email, signed in as another) or an existing signed-in user binding a fresh grant.

---

## Critical Rules

1. **Always accept email in user-facing flows.** Users know each other's emails, not userIds. The server resolves the userId or creates a deferred grant.

2. **Never assume a deferred grant is immediate.** Email-based grants resolve at signup, not at write time. Don't show "Alice is a member" until she actually has an `AppMembership` row (i.e. her `addMember`/share response had a direct `status`, not `"pending_signup"`).

3. **Check member-invitation quota before showing invite UI.** A member with a 0 quota will hit a 403; hide the button instead.

4. **Cancel pending shares with the right call.** Email-based shares/group-adds attached to an `AppInvitation` do *not* auto-cancel when you remove a single share. Cancel one pending share by email on the per-resource endpoint (`documents.removePermission(docId, { email })`, `groups.removeMember(type, id, { email })`, `collections.removeMember(collectionId, ...)`), or cancel everything (the invite + all pending shares/group/collection adds linked to it) with `client.invitations.delete(invitationId)`. There is **no `client.invitations.revoke()`**.

---

## Access Modes

An app's access mode decides who can sign up:

| Mode | Who can join |
|------|--------------|
| `public` | Anyone can sign up |
| `invite-only` | Only people holding an invitation |
| `domain` | Anyone with an email on the app's `allowedDomains` (e.g. `@mycompany.com`) |

Set the mode from the CLI:

```bash
primitive apps update --mode public        # | invite-only | domain
```

In `domain` mode, the allowed domains are carried on the app's `allowedDomains` list. Deferred grants are re-validated against this list at resolution time (see [Domain re-validation](#domain-mode-apps)).

### The waitlist

For `invite-only` apps, `waitlistEnabled` defaults to `true`: visitors who try to sign up leave their email, and admins invite them when capacity allows — one at a time or in batches.

```bash
primitive waitlist list                                  # who's waiting
primitive waitlist invite <waitlist-id> --send-email [--signup-url <url>]  # invite one entry
primitive waitlist bulk-invite --count 10 [--send-email] # invite the next N in line
primitive waitlist remove <waitlist-id>                  # drop an entry
```

`--send-email` delivers the invitation email; `--signup-url` overrides the link the email points at. Inviting off the waitlist mints an `AppInvitation` for that email — the same primitive `invitations.create` produces.

---

## Invitations

### API surface

```typescript
client.invitations.create({ email, role?, expiresAt?, source?, note?, sendEmail? }); // -> AppInvitationInfo
client.invitations.list({ limit?, cursor? });          // -> { items: AppInvitationInfo[], cursor? }   (admin/owner only)
client.invitations.delete(invitationId);               // CASCADES to deferred grants
client.invitations.quota();                            // -> { used, limit, remaining, unlimited }
client.invitations.get(invitationId);                  // -> AppInvitationInfo (includes inviteToken + status)
client.invitations.accept(inviteToken);                // authenticated cross-identity acceptance
client.invitations.listDeferredGrants({ email?, type?, limit? });  // admin debug only
client.invitations.revokeDeferredGrant(deferredId, "document" | "group");
```

`AppInvitationInfo` items returned by `list()` carry: `invitationId`, `email`, `role`, `invitedBy`, `invitedAt`, `expiresAt`, `accepted`, `acceptedAt`, `source`, `note`, `inviteToken`. The `status` field (`"pending" | "expired" | "accepted"`) is computed server-side and only returned by `get()`, not by `list()` — derive it on the client from `accepted` + `expiresAt` if you need it on a list row.

The cascading delete is `client.invitations.delete(id)` — there is **no `client.invitations.revoke()`**.

### Admin / member create

Admins/owners can invite at any role; members can invite only at `role: "member"` and only when `memberInvitationsEnabled` is true (check `quota()` first). `sendEmail` defaults to `false` — when `true` the server delivers a default invitation email, otherwise you deliver the `inviteToken` yourself. `source` (≤64 chars) and `note` are optional audit/admin-UI metadata; `expiresAt` overrides the default expiry.

```typescript
  // Admins/owners: any role
  await client.invitations.create({
    email: "alice@example.com",
    role: "member", // or "admin" / "owner" (admin/owner only)
  });

  // Full options
  await client.invitations.create({
    email: "alice@example.com",
    role: "member",
    expiresAt: new Date(Date.now() + 7 * 86400_000).toISOString(), // optional override
    source: "team-onboarding-flow",
    note: "Backend hire — Q2 cohort",
    sendEmail: true,
  });

  // Members: gate on quota first
  const quota = await client.invitations.quota();
  if (!quota.unlimited && quota.remaining <= 0) {
    return; // quota exhausted — hide the invite UI
  }
  await client.invitations.create({ email, role: "member" });
```

### Member invitations + quotas

By default only admins/owners can invite. Two app fields control member invitations:

| Field | Meaning |
|-------|---------|
| `memberInvitationsEnabled` | If `true`, users with role `"member"` can create invitations |
| `memberInvitationLimit` | Max active (non-accepted, non-expired) invitations per member |

Set both via `primitive apps update --member-invitations-enabled true --member-invitation-limit 5` (limit `0` = unlimited).

`quota()` returns `{ used: 0, limit: 0, remaining: 0, unlimited: false }` for a member when `memberInvitationsEnabled` is `false` — treat that as "no quota, hide the button." Admins/owners always get `unlimited: true` and are exempt from the limit. Members can only invite at `role: "member"`; passing `"admin"`/`"owner"` is rejected.

### Server error codes (in the HTTP error body)

The server returns these on `invitations.create`:

| Body shape | When |
|------------|------|
| `{ error: "INVITATION_LIMIT_REACHED", used, limit }` | Member at quota |
| `{ error: "MEMBER_INVITATIONS_DISABLED" }` | App has `memberInvitationsEnabled: false` and caller is a member |
| Plain text `"Members can only invite with role 'member'"` | Member tried `role: "admin"` |
| Plain text `"User already has a pending invitation"` | Duplicate create for the same email |
| Plain text `"User already exists in this app"` | Email is already an `AppUser` |

Gate the invite UI on `client.invitations.quota()` up front (a member with member-invites off comes back `remaining: 0`, `unlimited: false` — hide the button) so most of these never fire.

The client surfaces server errors as `Error("HTTP <status>: <jsonBody>")`. Parse the JSON to read the `error` field and branch on `"MEMBER_INVITATIONS_DISABLED"` etc.

> **Footgun:** error bodies on this endpoint use `error`. The access-request endpoints use `code`. Don't conflate the two — read the field name that matches the endpoint you called.

### Custom invitation emails (`inviteToken`)

`AppInvitation` carries an `inviteToken`. Combine with your accept-page URL to send your own email. The token is returned **inline** when invitations are first minted:

| Where the token comes back | Field |
|---------------------------|-------|
| `client.invitations.create()` | `AppInvitationInfo.inviteToken` |
| `client.documents.updatePermissions(...)` deferred result | `DeferredPermissionGrant.inviteToken` |
| `client.groups.addMember(...)` deferred result | `DeferredGroupAdd.inviteToken` |
| `client.collections.addMember(...)` deferred result | `inviteToken` |

Capture the `inviteToken` from the mint response if you need to build accept URLs later — the deferred-result fields above carry it on the same call that creates the grant.

For resend / lookup after the initial response is gone, fetch the invitation by id and rebuild the accept URL from its `inviteToken`. The returned record carries `invitationId`, `email`, `role`, `invitedBy`, `invitedAt`, `expiresAt`, `accepted`, `acceptedAt`, `source`, `note`, `inviteToken`, and a computed `status` (`"pending" | "expired" | "accepted"`).

```typescript
  const inv = await client.invitations.get(invitationId);
  const acceptUrl = `${baseUrl}/invite/accept?inviteToken=${inv.inviteToken}`;
  // Send `acceptUrl` to `inv.email` from your own email provider.
```

Permissions for `invitations.get`: app admin/owner, OR the invitation's original inviter. Members who did not create the invitation receive 403 — `inviteToken` is a bearer credential, so read access is intentionally narrow.

### Token-based acceptance (authenticated caller)

For the cases where the platform can't infer intent from the recipient's email:

- **Cross-identity acceptance** — invited at `work@example.com`, signing in as `home@gmail.com`.
- **Existing user binding a fresh deferred grant** — the invitee is already signed in to a working account and wants to redeem an invite without going through signup again.

Email-matched signup resolves deferred grants automatically — your app does NOT call accept for that path. Use `invitations.accept` only when the invitee is already authenticated as someone other than (or in addition to) the email the invitation was sent to.

The accept call (shown in [Accept an invite token](#accept-an-invite-token) above) returns `{ status: "accepted", invitationId, grantsResolved: { groups, documents } }`, and throws `401 INVITE_TOKEN_INVALID` for any bad token (invalid, expired, or already redeemed — the server returns one code to avoid leaking invitation existence).

**Template-provided acceptance page:** The template ships `InviteAcceptPage.vue` (`src/pages/InviteAcceptPage.vue`) at the route `/invite/accept?inviteToken=...`. It handles the signed-in confirmation path (calls `client.invitations.accept(token)`) and the signed-out path (stashes the token in `sessionStorage` via `src/lib/inviteToken.ts`, redirects to login, and the auth methods in `userStore` consume it automatically at sign-in). All error states are also handled. If you're using the template, point your invitation emails at `${yourApp.baseUrl}/invite/accept?inviteToken=${token}` — the page is already wired up in the router. See the [Authentication guide](AGENT_GUIDE_TO_PRIMITIVE_AUTHENTICATION.md#invite-token-persistence-across-auth-round-trips-template-pattern) for the persistence details.

---

## Deferred Grants

Deferred grants bridge "I have an email" and "they're in the app." When you share a document, add someone to a group, or add them to a collection **by email** and that email isn't yet a user, the platform mints (or reuses) an `AppInvitation` and records the pending grant. The grant-side mechanics live with each resource (documents in the [Documents guide](AGENT_GUIDE_TO_PRIMITIVE_DOCUMENTS.md#sharing-documents), groups in the [Users and Groups guide](AGENT_GUIDE_TO_PRIMITIVE_USERS_AND_GROUPS.md#managing-members)); the resolution lifecycle is below.

### Lifecycle

1. Agent writes a grant by email for `alice@outside.com`.
2. Server creates (or reuses) an `AppInvitation` plus a `DeferredDocumentPermission` / `DeferredGroupAdd`.
3. Alice receives an invitation email (default flow) or a custom one you sent using `inviteToken`.
4. Alice becomes a user — via one of two paths the **recipient** chooses, not the app.

### Path A — automatic email-match (the common case)

Alice signs in with `alice@outside.com` (any auth method). The signup flow resolves *every* pending deferred grant for that email in one transaction — app membership, document shares, group adds, collection memberships — **with no manual `accept` call**. Don't re-grant after signup; the access is already there.

### Path B — explicit `accept(inviteToken)` (cross-identity)

When the recipient is signed in (or wants to sign in) under a **different** email than the invite was sent to — invited at `work@example.com`, redeeming as `home@gmail.com` — or is an existing user binding a fresh grant to their current account, the platform can't infer intent from the email. The app calls `client.invitations.accept(inviteToken)` from that session. The invitation is marked accepted (write-once) and every grant linked to it binds to the **currently signed-in user**, regardless of the invited email.

### Domain-mode apps

Deferred grants are re-validated at resolution time. In a `domain`-restricted app, a grant for an email outside `allowedDomains` is silently dropped at resolution and the invitation is rejected. Don't use deferred grants as a side channel around domain policy.

### Cascade on revoke

`client.invitations.delete(invitationId)` removes every pending grant attached to the invitation in the same operation — there's no risk of an orphan share activating after you change your mind. Conversely, deleting an invitation to cancel a *single* share is too broad: use the per-resource `removePermission`/`removeMember` by email for that.

### Inspecting pending state (debug only)

```typescript
  const { grants, nextCursor } = await client.invitations.listDeferredGrants({
    email: "alice@example.com",
  });
```

Reserve this for admin/debug UIs. For end-user "people with access + pending invitations" UI, use the per-resource `listPendingInvitations` endpoints on documents/groups/collections (see the [Documents guide](AGENT_GUIDE_TO_PRIMITIVE_DOCUMENTS.md#building-a-members--pending-ui)).

---

## Common Patterns

### Onboard a teammate end-to-end

Check quota, mint the app invitation, share the project document by email, and add the email to the team group. After signup all three resolve in one server-side transaction.

```typescript
  // 1. Invite a teammate
  await client.invitations.create({ email: "newhire@example.com", role: "member" });

  // 2. Share a project document with them (pending until signup)
  await client.documents.updatePermissions(projectDocId, {
    email: "newhire@example.com",
    permission: "read-write",
  });

  // 3. Add them to the engineering group (pending until signup)
  await client.groups.addMember("team", "engineering", { email: "newhire@example.com" });

  // When they sign up, all three apply in one transaction. They land in the
  // app with team-group access and the project already shared with them.
```

---

## WebSocket Events

The `invitation` event is the only typed app-membership event. The lifecycle action is on `event.action` (not `event.type`, which is always `"invitation"`). Branch on `action`: `created`/`updated`/`cancelled` go to the invitee only; `declined` to both; `accepted` to the inviter only (with `event.acceptedBy` carrying the userId). Treat unknown actions as a no-op.

```typescript
  client.on("invitation", (event) => {
    switch (event.action) {
      case "created":   /* invitee only */ break;
      case "updated":   /* invitee only */ break;
      case "cancelled": /* invitee only */ break;
      case "declined":  /* both invitee and inviter */ break;
      case "accepted":  /* inviter only — event.acceptedBy carries the userId */ break;
      default: /* future-proof: no-op, don't throw */ break;
    }
  });
```

**Targeting is asymmetric** — most actions go to one side only. `accepted` goes to the *inviter*; `created` goes to the *invitee*. Don't expect both sides to see the same events. Polling `invitations.list()` for acceptance is an anti-pattern — subscribe and switch on the `accepted` action instead.

---

## Anti-Patterns

- Calling a method that doesn't exist: `client.invitations.revoke`, `client.deferredGrants.list`. The correct names are `client.invitations.delete`, `client.invitations.listDeferredGrants`.
- Calling `client.invitations.delete()` to cancel a single pending document share — it cascades to every share, group add, and collection add linked to that invitation. Use the per-resource `removePermission`/`removeMember` by email.
- Showing a member invite button without checking `client.invitations.quota()` first — a member with a 0 quota hits a 403.
- Re-granting access after signup — email-matched deferred grants already resolved (Path A); the user has access.
- Polling `invitations.list()` for acceptance. Subscribe to the `invitation` event and switch on the `accepted` action (delivered to the inviter).

---

## Error Codes Quick Reference

Server-emitted body codes, surfaced on the thrown HTTP error.

| Code / `error` field | Endpoint | Meaning |
|----------------------|----------|---------|
| `INVITATION_LIMIT_REACHED` | `invitations.create` | Member at quota; body has `used`, `limit` |
| `MEMBER_INVITATIONS_DISABLED` | `invitations.create` | App has member invitations off and caller is a member |
| `INVITE_TOKEN_INVALID` | `invitations.accept` | 401 for any bad token: doesn't decode, past `expiresAt`, or already redeemed |

---

## Related Guides

- [Documents](AGENT_GUIDE_TO_PRIMITIVE_DOCUMENTS.md) — Document sharing, access requests, and collections (the grant side of deferred email shares)
- [Users and Groups](AGENT_GUIDE_TO_PRIMITIVE_USERS_AND_GROUPS.md) — Group membership by email and CEL
- [Authentication](AGENT_GUIDE_TO_PRIMITIVE_AUTHENTICATION.md) — How signup triggers deferred-grant resolution
