# Agent Guide to Primitive Invitations

Guidelines for AI agents implementing app membership: access modes, invitations and quotas, deferred-grant resolution, and the invite-accept wiring.

## Core operations

### App invitations (quota / create / list / cancel)

```swift
  // The caller's remaining invite quota (admins/owners are unlimited)
  let quota = try await client.invitations.quota()

  // Invite someone to the app by email
  let invitation = try await client.invitations.create(
    params: CreateInvitationParams(email: "alice@example.com", role: "member")
  )

  // Pending invitations
  let list = try await client.invitations.list()

  // Cancel one
  _ = try await client.invitations.delete(invitationId: invitationId)
```

### Accept an invite token

```swift
  let result = try await client.invitations.accept(inviteToken: inviteToken)
```

## Mental Model

Invitations grant app membership. Sharing with a new user by email also creates pending access, which takes effect when the recipient signs up. Use the document, group, or collection API to grant access; the platform manages pending state.

-----------|--------|-------|
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

## Access Modes

An app's access mode decides who can sign up:

| Mode | Who can join |
|------|--------------|
| `public` | Anyone can sign up |
| `invite-only` | Only people holding an invitation |
| `domain` | Anyone with an email on the app's `allowedDomains` (e.g. `@mycompany.com`) |

`mode` is an app setting in `app.toml` — set it there and push:

```bash
primitive config set app app.mode=public     # | invite-only | domain
primitive config push --only app
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

Use `create`, `list`, `get`, `delete`, `quota`, and `accept` to manage invitations. Admins and owners manage all invitations; members manage their own. See the examples below and the generated client reference for signatures.

### Admin / member create

Admins/owners can invite at any role; members can invite only at `role: "member"` and only when `memberInvitationsEnabled` is true (check `quota()` first). `sendEmail` defaults to `false` — when `true` the server delivers a default invitation email, otherwise you deliver the `inviteToken` yourself. `source` (≤64 chars) and `note` are optional audit/admin-UI metadata; `expiresAt` overrides the default expiry.

```swift
  // Admins/owners: any role
  _ = try await client.invitations.create(
    params: CreateInvitationParams(
      email: "alice@example.com",
      role: "member" // or "admin" / "owner" (admin/owner only)
    )
  )

  // Full options
  let expiresAt = ISO8601DateFormatter().string(from: Date().addingTimeInterval(7 * 86400))
  _ = try await client.invitations.create(
    params: CreateInvitationParams(
      email: "alice@example.com",
      role: "member",
      expiresAt: expiresAt, // optional override
      source: "team-onboarding-flow",
      note: "Backend hire — Q2 cohort",
      sendEmail: true
    )
  )

  // Members: gate on quota first
  let quota = try await client.invitations.quota()
  if !quota.unlimited && quota.remaining <= 0 {
    return // quota exhausted — hide the invite UI
  }
  _ = try await client.invitations.create(
    params: CreateInvitationParams(email: email, role: "member")
  )
```

### Member invitations + quotas

By default only admins/owners can invite. Two app fields control member invitations:

| Field | Meaning |
|-------|---------|
| `memberInvitationsEnabled` | If `true`, users with role `"member"` can create invitations |
| `memberInvitationLimit` | Max active (non-accepted, non-expired) invitations per member |

Set both in the `[invitations]` table of `app.toml` — `enabled` and `limit` (`0` = unlimited) — and apply with `primitive config push --only app`.

`quota()` returns `{ used: 0, limit: 0, remaining: 0, unlimited: false }` for a member when `memberInvitationsEnabled` is `false` — treat that as "no quota, hide the button." Admins/owners always get `unlimited: true` and are exempt from the limit. Members can only invite at `role: "member"`; passing `"admin"`/`"owner"` is rejected.

### Server error codes (in the HTTP error body)

Check `quota()` before showing the invitation form. Handle server codes using the client’s structured error fields; see [Error Handling](AGENT_GUIDE_TO_PRIMITIVE_ERROR_HANDLING.md).

### Custom invitation emails (`inviteToken`)

`AppInvitation` carries an `inviteToken`. Combine with your accept-page URL to send your own email. The token is returned **inline** when invitations are first minted:

| Where the token comes back | Field |
|---------------------------|-------|
| `client.invitations.create()` | `AppInvitationInfo.inviteToken` |
| `client.documents.updatePermissions(...)` deferred result | `DeferredPermissionGrant.inviteToken` |
| `client.groups.addMember(...)` deferred result | `DeferredGroupAdd.inviteToken` |
| `client.collections.addMember(...)` deferred result | `inviteToken` |

Capture the `inviteToken` from the mint response if you need to build accept URLs later — the deferred-result fields above carry it on the same call that creates the grant.

To resend an invitation, fetch it with `get()` and build a URL from its `inviteToken`. Only an app admin, owner, or the original inviter can read it.

```swift
  let inv = try await client.invitations.get(invitationId: invitationId)
  let inviteToken = inv.inviteToken ?? ""
  let acceptUrl = "\(baseURL)/invite/accept?inviteToken=\(inviteToken)"
  // Send `acceptUrl` to `inv.email` from your own email provider.
```

Treat `inviteToken` as a bearer credential: anyone holding it can redeem the invitation.

### Token-based acceptance (authenticated caller)

For the cases where the platform can't infer intent from the recipient's email:

- **Cross-identity acceptance** — invited at `work@example.com`, signing in as `home@gmail.com`.
- **Existing user binding a fresh deferred grant** — the invitee is already signed in to a working account and wants to redeem an invite without going through signup again.

Email-matched signup resolves deferred grants automatically — your app does NOT call accept for that path. Use `invitations.accept` only when the invitee is already authenticated as someone other than (or in addition to) the email the invitation was sent to.

The accept call (shown in [Accept an invite token](#accept-an-invite-token) above) returns `{ status: "accepted", invitationId, grantsResolved: { groups, documents } }`, and throws `401 INVITE_TOKEN_INVALID` for any bad token (invalid, expired, or already redeemed — the server returns one code to avoid leaking invitation existence).

Resolve the URL with `client.links`. Accept immediately when signed in; otherwise retain `inviteToken` and pass it through sign-in. For a returning Apple identity, call `invitations.accept` after sign-in. Handle `INVITE_TOKEN_INVALID` by offering to request another invitation.

---

## Deferred Grants

Email-based shares remain pending until the recipient joins. The document, group, and collection APIs manage these grants.

### Lifecycle

1. Agent writes a grant by email for `alice@outside.com`.
2. The server creates or reuses an invitation and records the pending access.
3. Send the invitation email with `sendEmail: true`, or deliver a custom link using `inviteToken`.
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

```swift
  let result = try await client.invitations.listDeferredGrants(
    email: "alice@example.com"
  )
  let items = result.items
  let nextCursor = result.nextCursor
```

Reserve this for admin/debug UIs. For end-user "people with access + pending invitations" UI, use the per-resource `listPendingInvitations` endpoints on documents/groups/collections (see the [Documents guide](AGENT_GUIDE_TO_PRIMITIVE_DOCUMENTS.md#building-a-members--pending-ui)).

---

## Common Patterns

### Onboard a teammate end-to-end

Check quota, mint the app invitation, share the project document by email, and add the email to the team group. After signup all three resolve in one server-side transaction.

```swift
  // 1. Invite a teammate
  _ = try await client.invitations.create(
    params: CreateInvitationParams(email: "newhire@example.com", role: "member")
  )

  // 2. Share a project document with them (pending until signup)
  _ = try await client.documents.updatePermissions(
    documentId: projectDocId,
    params: .email("newhire@example.com", permission: "read-write")
  )

  // 3. Add them to the engineering group (pending until signup)
  _ = try await client.groups.addMember(
    groupType: "team",
    groupId: "engineering",
    params: .email("newhire@example.com")
  )

  // When they sign up, all three apply in one transaction. They land in the
  // app with team-group access and the project already shared with them.
```

---

<a id="anti-patterns"></a>

## Gotchas {#critical-rules}

1. **Always accept email in user-facing flows.** Users know each other's emails, not userIds. The server resolves the userId or creates a deferred grant.

2. **Never assume a deferred grant is immediate.** Email-based grants resolve at signup, not at write time. Don't show "Alice is a member" until she actually has an `AppMembership` row (i.e. her `addMember`/share response had a direct `status`, not `"pending_signup"`).

3. **Check member-invitation quota before showing invite UI.** Hide the button when `remaining` is zero and `unlimited` is false.

4. **Choose the cancellation scope.** Use the resource’s `removePermission` or `removeMember` operation to cancel one pending share. `invitations.delete` cancels the invitation and every pending grant attached to it.

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
