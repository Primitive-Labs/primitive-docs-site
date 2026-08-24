# Agent Guide to Primitive Authentication

Implementing auth flows for Primitive apps. All methods live on `JsBaoClient` (package: `js-bao-wss-client`).

## Auth Methods

| Method | When to use |
|--------|-------------|
| OAuth (Google) | Primary auth, redirect-based |
| Magic Link | Passwordless email link |
| OTP | 6-digit email code (10 min expiry) |
{{#lang ts}}
| Passkey | WebAuthn for returning users (requires existing account) |
{{/lang}}
{{#lang swift}}
| Sign in with Apple | Native one-call sign-in (`signInWithApple`); gate on `hasApple` |
| Passkey | Native one-call sign-in/registration via `AuthenticationServices`; for returning users |
{{/lang}}

Each method must be enabled in the Admin Console. Check availability with `getAuthConfig()` before showing UI.

## Client Setup (no template required)

All flows below run on a plain client — no starter template needed:

{{ example: auth/initialize-client }}

{{#lang ts}}
In the starter template this wiring is owned for you by the template's `userStore`.
{{/lang}}
{{#lang swift}}
In the starter template this wiring is owned for you by `PrimitiveAppState.initialize()` + `PrimitiveAuthManager`.
{{/lang}}

## Discovering Available Methods

{{ example: auth/get-auth-config }}

Google registers a client **per platform**, so `getAuthConfig()` reports a client MAP keyed by client type (`web`, `ios`, `android`, `desktop`, `chrome-extension`) rather than a single availability flag — a single flag could only ever be right for one platform. Each entry carries `clientId`, `redirectUris` and `usable` (the server's shape verdict); no secret is ever published.

Availability is the provider being enabled **and** your platform's entry being usable, together.
{{#lang ts}}
`checkOAuthAvailable()` computes exactly that against the `web` entry; `googleWebClientAvailable(config)` is the same predicate over a config you already have, so a button and its click handler can share one definition.
{{/lang}}
{{#lang swift}}
`AuthConfigInfo.googleSignInAvailable` computes exactly that against the `ios` entry, and `AppConfigInfo.googleAvailable` is projected from it — the launch UI and the flow cannot disagree.
{{/lang}}

`magicLinkEnabled` and `otpEnabled` default to `true` unless explicitly disabled in the Admin Console.
{{#lang swift}}
`hasApple` reports Sign in with Apple availability (`appleSignInEnabled` plus configured Apple audiences); gate the native `signInWithApple` button on it.
{{/lang}}
{{#lang ts}}
`hasPasskey` requires `passkeyEnabled` plus a non-empty `passkeyRpConfig` map.

### Passkey RP config (`passkeyRpConfig`)

`passkeyRpConfig` is a map keyed by RP ID (a bare domain — no protocol, no port, no path), with each value `{ name: string }`. One app can register passkeys against several origins (e.g. `app.example.com` and `staging.example.com`):

```jsonc
"passkeyRpConfig": {
  "app.example.com":     { "name": "Example" },
  "staging.example.com": { "name": "Example (staging)" }
}
```

`getAuthConfig()` returns the effective map. Configure passkeys via `passkeyRpConfig`.
{{/lang}}

## Server App Settings ↔ Client Contract

Server-side app settings must align with the origin the client app is served from. These settings live in `config/app.toml` and are applied with `primitive config push` (or `config push --only app`) — that is the only CLI write path; no flag sets an app setting. Inspect the live values with `primitive apps get`; the relevant fields:

| Server field | Contract | Set via |
|---|---|---|
| `corsAllowedOrigins` | Must contain the exact serving origin (scheme+host+port). `corsMode` defaults to `custom` — an empty list blocks every cross-origin request. | `[cors]` (`mode`, `allowedOrigins`, `allowCredentials`) in `app.toml` → `config push` |
| `googleClients[<type>].redirectUris` | The Google callback is validated against the entries, and the matching one SELECTS the client used for the exchange — a URI listed by no entry returns 400 `Invalid redirect URI`, and a URI may appear in only one entry. | `[auth.google.clients.<type>].redirectUris` in `app.toml` → `config push` (non-localhost must be https) |
| `emailRedirectUris` | The magic-link allow-list. **Fail-closed**: an empty or missing list rejects every magic-link request. New apps are seeded with the localhost dev callback. `http`/`https` match by origin; a custom scheme must match exactly. | `[auth].emailRedirectUris` in `app.toml` → `config push` |
| `baseUrl` | Used for links in auth emails / redirects. | `[app].baseUrl` in `app.toml` → `config push` |
{{#lang ts}}
| Provider toggles | What `getAuthConfig()` reports. | `[auth]` in `app.toml` (`googleOAuthEnabled`, `magicLinkEnabled`, `passkeyEnabled` + `[auth.passkeys]`, `otpEnabled`) → `config push`. |
{{/lang}}
{{#lang swift}}
| Provider toggles | What `getAuthConfig()` reports. | `[auth]` in `app.toml` (`googleOAuthEnabled`, `magicLinkEnabled`, `otpEnabled`, `appleSignInEnabled`, `appleAudiences`) → `config push`. Enable Sign in with Apple by setting `appleSignInEnabled = true` and `appleAudiences = ["<bundle-id>"]` (`hasApple` then reports true). |
{{/lang}}

{{#lang ts}}
**CORS misconfiguration blocks bootstrap.** When the serving origin is missing from `corsAllowedOrigins`, the browser blocks the client's bootstrap refresh (`POST …/api/auth/refresh` → 403, no `access-control-allow-origin`): `initializeClient` throws `initializeClient refresh failed (network)` before `getAuthConfig()` is reached, and the template app's login surfaces the error. Fix by adding the serving origin to `[cors].allowedOrigins` and running `primitive config push --only app`; inspect with `primitive apps get`. Common triggers: serving on a non-default port, or a newly deployed domain.
{{/lang}}

Dev → prod checklist: in `app.toml`, add the production origin to `[cors].allowedOrigins`, set `[app].baseUrl`, add the production OAuth callback to the `redirectUris` of the Google client that will redirect there (`[auth.google.clients.web]` for a browser), and add the production magic-link callback to `[auth].emailRedirectUris`; run `primitive config push`; then re-check `getAuthConfig()` reports the expected methods.

---

## OAuth (Google)

### Start the flow

{{ example: auth/oauth }}

{{#lang ts}}
`startOAuthFlow` throws `Error("OAuth not configured")` if `oauthRedirectUri` was not passed to `initializeClient`. The browser navigates away — code after the call doesn't run on success.

**`autoOAuth` client option.** Pass `autoOAuth: true` (with `oauthRedirectUri`) to `initializeClient` and the client will auto-redirect to OAuth whenever it comes back online without a valid token (e.g. a refresh failed, or there was no persisted token). For apps where OAuth is the only sign-in path this avoids hand-rolling the "no token, send to login" branch. Leave it off if you have multiple sign-in methods or want to render your own login screen first.

Optional second argument supports waitlist enrollment and invite-token acceptance:

```typescript
await client.startOAuthFlow(continueUrl, {
  waitlist: { source: "landing-page", note: "interested in beta" },
  inviteToken: tokenFromEmail,
});
```
{{/lang}}
{{#lang swift}}
`startOAuthFlow(redirectUri:continueUrl:)` takes an explicit `redirectUri` and **returns** the authorization URL to open yourself (e.g. via `ASWebAuthenticationSession`). It throws `OAuth not configured` if OAuth is unavailable.

It also accepts an optional `waitlist: OAuthWaitlist` (`source` / `note`, enrolls the user via the callback) and `inviteToken: String` (accepts the named invitation when the callback resolves):

```swift
let authUrl = try await client.startOAuthFlow(
  redirectUri: redirectUri,
  waitlist: OAuthWaitlist(source: "landing-page", note: "interested in beta"),
  inviteToken: tokenFromEmail
)
```

Both are threaded through `signInWithGoogle(...)` as well — `signInWithApple(...)` has no `startOAuthFlow` building block (native Sign in with Apple never redirects), but it takes its own `inviteToken` parameter directly; see [Native one-call sign-in](#native-one-call-sign-in-google-apple) below.
{{/lang}}

### Handle the callback (instance method — preferred)

When the callback page can construct a client (you already have the JWT or are happy to re-init), extract `code`/`state` from the callback and pass them to `handleOAuthCallback`:

{{ example: auth/oauth-callback }}

### Handle the callback (static method — when no client yet)

{{ example: auth/oauth-exchange-code }}

{{#lang ts}}
If your app uses a refresh proxy, also pass `refreshProxyBaseUrl` (e.g. `` `${window.location.origin}/proxy` ``) and `refreshProxyCookieMaxAgeSeconds` to `exchangeOAuthCode`.
{{/lang}}

**Don't:**

{{#lang ts}}
```typescript
// WRONG — startOAuthFlow does not return a token. It redirects.
const token = await client.startOAuthFlow();

// WRONG — handleOAuthCallback does not return the token either; it stores it.
const { token } = await client.handleOAuthCallback(code, state);
```
{{/lang}}
{{#lang swift}}
```swift
// WRONG — handleOAuthCallback does not return the token; it stores it.
let token = try await client.handleOAuthCallback(code: code, state: state)
```
{{/lang}}

{{#lang swift}}
### Native one-call sign-in (Google / Apple)

`startOAuthFlow` + `handleOAuthCallback` are the raw building blocks. For the native experience use the one-call helpers — each presents the system auth sheet, runs the redirect + code exchange, applies the session token (cause `"oauthCallback"` / `"apple"`, emitting `.authSuccess` / `.authState`), and re-authenticates the WebSocket. Both return the signed-in `userId` and the server's `isNewUser` flag:

```swift
let google = try await client.signInWithGoogle()   // GoogleSignInResult(userId, isNewUser)
let apple = try await client.signInWithApple()      // AppleSignInResult(userId, isNewUser)
```

- `signInWithGoogle(presentationAnchor:redirectUri:continueUrl:waitlist:inviteToken:)` — when `redirectUri` is nil it derives the URI from the bundled `GoogleService-Info.plist`; it throws if neither is available. Throws `OAuthSignInError.cancelled` when the user dismisses the sheet.
- `signInWithApple(presentationAnchor:inviteToken:)` — uses the app's "Sign in with Apple" entitlement and the server's configured Apple audiences. Throws `AppleSignInError.cancelled` on dismissal, `.notConfigured` when the server has no Apple audiences.

Both take an optional `inviteToken: String?`. When present, the server accepts the named invitation as part of that sign-in and resolves its deferred grants (document/collection/group access) to the new user atomically — no follow-up `client.invitations.accept(inviteToken:)` call needed:

```swift
let apple = try await client.signInWithApple(
  presentationAnchor: anchor,
  inviteToken: tokenFromEmail
)
// apple.userId, apple.isNewUser
```

For Apple, this only resolves on **first** sign-in (`isNewUser == true`); a repeat sign-in from an existing Apple identity takes a different internal path and does not resolve `inviteToken` grants — call `client.invitations.accept(inviteToken:)` afterward for that case, or for any other post-hoc acceptance. The invite token is validated server-side only after the Apple identity token is cryptographically verified, and only before any user/grant mutation — so any bad, expired, or already-used token throws `HttpError` with `serverCode == "INVITE_TOKEN_INVALID"` (one code for every invalid-token reason, by design, to avoid a validity oracle). A domain-restricted app also still throws the pre-existing `DOMAIN_NOT_ALLOWED` when the Apple-verified email itself falls outside `allowedDomains`.

Gate the buttons on the auth config: `googleSignInAvailable` for Google — the provider enabled and this app's `ios` client entry usable — and `hasApple` for Apple (`AuthConfigInfo` also carries `appleSignInEnabled`). The starter template's `PrimitiveAuthManager` wraps both helpers and renders only the providers `availableProviders` reports.
{{/lang}}

---

## One Account Across Providers

Google and Apple identities are linked to an account, not stored as a single
current provider. Consequences to code against:

- **Same email, second provider → same account.** A user who signed in with
  Google and later signs in with Apple under the same email resolves to the
  existing account. `isNewUser` is `false` on that second sign-in; documents,
  memberships, and permissions carry over. Don't build "merge my accounts" UI.
- **A linked identity survives an email change.** The identity is matched
  before the email is, so a provider-side email change still resolves to the
  original account rather than provisioning a new one.
- **Links never move.** A provider identity already linked to one account is
  never re-pointed at another by a later sign-in — the original owner keeps it.
- **Only Google and Apple link this way.** Magic link and OTP resolve by the
  submitted email; passkeys are registered against an already-signed-in
  account.

Practical effect on sign-in handling: treat `isNewUser` as "first time in this
app", not "first time with this provider" — a first-ever Apple sign-in by an
existing Google user reports `isNewUser == false`, so onboarding gated on it is
correctly skipped.

---

## Magic Link

### Request + verify

{{ example: auth/magic-link }}

{{#lang ts}}
`magicLinkRequest` accepts an optional `redirectUri` (defaulting to the client's `oauthRedirectUri`) and throws `Error("Redirect URI not configured")` if neither is set.
{{/lang}}
{{#lang swift}}
`auth.magicLinkRequest(email:redirectUri:)` takes the `redirectUri` as a required argument. `auth.magicLinkVerify(token:inviteToken:)` returns a `MagicLinkVerifyResult` (`.user`, `.promptAddPasskey?`, `.isNewUser?`).
{{/lang}}

### Reading the token (callback page)

The callback delivers the token as a `magic_token` value — **not** `token`, `magicToken`, or `code`:

{{ example: auth/magic-link-callback }}

{{#lang ts}}
The callback URL may also carry `?purpose=login-add-passkey`. The server appends this when the link was sent for an existing user the platform thinks should add a passkey (e.g. they signed in via OTP/magic-link but have no passkey on file). The `magicLinkVerify` call itself is unchanged — apps that read `purpose` from the URL can use it as a hint to route the user to passkey registration after sign-in instead of straight to the home screen.
{{/lang}}

To accept an invitation server-side at verify time (so the deferred grant resolves to the signing-in user even when emails differ), pass `inviteToken`:

{{ example: auth/magic-link-invite }}

---

## OTP (Email Code)

{{ example: auth/otp }}

{{#lang ts}}
`otpVerify` also accepts an `{ inviteToken }` option to accept an invitation at verify time.
{{/lang}}
{{#lang swift}}
`auth.otpVerify(email:code:)` returns an `OtpVerifyResult` (`.user`, `.isNewUser?`). To accept an invitation at verify time, pass the `inviteToken` parameter: `auth.otpVerify(email:code:inviteToken:)`.
{{/lang}}

### Error handling

`AuthError` is thrown for non-2xx responses with a machine-readable code:

{{ example: auth/auth-error-handling }}

> **Caveat on OTP disabled.** When OTP is disabled the request endpoint returns a plain 400 with the message `"OTP authentication is not enabled for this app"` and **no `code` field**. Don't rely on a code to detect that case — gate the OTP UI on `getAuthConfig()`'s `otpEnabled` up front instead.

{{#lang ts}}
The exported `AUTH_CODES` constant covers: `ADDED_TO_WAITLIST`, `INVITATION_REQUIRED`, `DOMAIN_NOT_ALLOWED`, `INVALID_TOKEN`, `TOKEN_EXPIRED`, `PASSKEY_NOT_ENABLED`, `MAGIC_LINK_NOT_ENABLED`, `WAITLIST_ENTRY_UPDATED`, `INVITE_TOKEN_INVALID`, `INVITE_TOKEN_EXPIRED`, `INVITE_ALREADY_ACCEPTED`. The server may also return `RATE_LIMITED`, `OTP_MAX_ATTEMPTS`, `RESERVED_EMAIL_FOR_ADMIN`, and `GOOGLE_OAUTH_MISCONFIGURED` (the selected Google client's `clientSecret` does not resolve to a stored app secret — an operator fix, not a user one; only the web flow returns it, since a PKCE sign-in is not failed closed on an unresolvable stored value — it still needs the same fix, it just fails at Google as `INVALID_TOKEN` instead) — compare those as string literals.

The same `AuthError` codes apply to `magicLinkRequest`/`magicLinkVerify` and `passkey*` methods.
{{/lang}}
{{#lang swift}}
`AuthCode` also carries the SDK-generated cases `.tokenInvalid`, `.refreshFailed`, `.networkError`, and `.unauthorized`, plus `.passkeyNotEnabled` and `.memberInvitationsDisabled` from the server. Server codes outside the enum (e.g. rate limiting) arrive with `code == nil` — fall back to `error.message`.

The same `AuthError` codes apply to `magicLinkRequest`/`magicLinkVerify`.

**Don't sign the user out on every failed call.** When a request gets a 401 the client refreshes the token and retries. If the refresh itself can't reach the server, the call no longer fails as an HTTP 401 — it fails as a transport error carrying no HTTP status, because that is a transient outage rather than a rejected credential. Retry it instead of clearing the session. Only a refresh the server actually rejects surfaces as `HttpError(status: 401, message: "Invalid credentials")`, and that is the one to sign out on.
{{/lang}}

---

{{#lang ts}}
## Passkeys

WebAuthn passkeys for returning users, built on the browser's `@simplewebauthn/browser` helpers.

`passkeyAuthStart` works without an existing session (used to sign in). `passkeyRegisterStart` and management methods require an authenticated client.

### Sign in

```typescript
import { startAuthentication } from "@simplewebauthn/browser";

const { options, challengeToken } = await client.passkeyAuthStart();
const credential = await startAuthentication({ optionsJSON: options });
const { user, isNewUser } = await client.passkeyAuthFinish(credential, challengeToken);
```

### Register (must be authenticated)

```typescript
import { startRegistration } from "@simplewebauthn/browser";

const { options, challengeToken } = await client.passkeyRegisterStart();
const credential = await startRegistration({ optionsJSON: options });

const result = await client.passkeyRegisterFinish(
  credential,
  challengeToken,
  "MacBook Pro",
  { inviteToken: tokenFromEmail } // optional — accepts an invite during registration
);
// {
//   success: true,
//   credentialBackedUp?: boolean,           // present when the authenticator advertises BE/BS bits
//   invitation?: {                          // present only if inviteToken was supplied AND accepted
//     invitationId: string,
//     grantsResolved: { groups: number, documents: number },
//   }
// }
```

When `inviteToken` is supplied and the invitation resolves successfully, the `invitation` field reports how many deferred grants (group memberships + document permissions) were resolved to the new user — useful for a post-registration confirmation toast (e.g. "Joined 2 groups, 3 documents").

**Don't:**

```typescript
// WRONG — must call startAuthentication/startRegistration between start and finish.
const { challengeToken } = await client.passkeyAuthStart();
await client.passkeyAuthFinish(/* ??? */, challengeToken);

// WRONG — passkeyRegisterStart 401s if there is no current session.
// Sign the user in (OAuth/magic link/OTP/existing passkey) before registering.
```

### Manage

```typescript
const { passkeys } = await client.passkeyList();
// [{ passkeyId, deviceName, createdAt, lastUsedAt }]

await client.passkeyUpdate(passkeyId, { deviceName: "Work Laptop" });
await client.passkeyDelete(passkeyId);
```
{{/lang}}
{{#lang swift}}
## Passkeys

The client wraps Apple's `AuthenticationServices` in two one-call helpers on `client.auth` (iOS/macOS), so app code never touches the WebAuthn challenge handshake directly.

### Sign in

`signInWithPasskey` runs the discoverable-credential flow — the system sheet lists the passkeys saved for the app, the user picks one, and the call applies the session token (cause `"passkeyAuth"`, emitting `.authSuccess` / `.authState`) and re-authenticates the WebSocket. It works without an existing session.

```swift
let result = try await client.auth.signInWithPasskey()   // PasskeySignInResult(user, isNewUser)
```

- `signInWithPasskey(presentationAnchor:preferImmediatelyAvailableCredentials:)` — pass `preferImmediatelyAvailableCredentials: true` to restrict the prompt to passkeys already on the device and fail fast (no QR/cross-device fallback). Throws `PasskeyError.canceled` when the user dismisses the sheet; server-side failures surface as `HttpError`.

### Register (must be authenticated)

`registerPasskey` adds a passkey to the **currently signed-in** user — sign in by another method first.

```swift
let reg = try await client.auth.registerPasskey(deviceName: "My iPhone")
```

- `registerPasskey(deviceName:inviteToken:presentationAnchor:)` — `deviceName` labels the credential (server default `"Unknown device"` when omitted). Pass `inviteToken` to fold invitation acceptance into registration. Throws `PasskeyError.canceled` on dismissal.

### Manage

The management methods match the JS client:

```swift
let list = try await client.auth.passkeyList()                              // PasskeyListResult
try await client.auth.passkeyUpdate(passkeyId: id, deviceName: "Work iPad")  // rename
try await client.auth.passkeyDelete(passkeyId: id)                          // remove
```

Passkey sign-in requires `passkeyEnabled` plus a non-empty `passkeyRpConfig`, and the app's associated-domains entitlement must list the RP domain (see [Deep links and universal links](#deep-links-and-universal-links)). Gate the UI on `getAuthConfig()`'s `passkeyEnabled` rather than catching a failure.

---

## Deep links and universal links

A native app receives Primitive URLs through universal links (an `NSUserActivity`) or a custom scheme — an invitation accept link, a shared-document link, or a magic-link callback. `client.links` turns an incoming URL into a typed `LinkTarget` so the app can route it without parsing query strings by hand, and builds the canonical outbound URLs.

Point it at the app's base URL once (the same value as the server's `app.baseUrl`); the builders return `nil` until it is set, and the host is automatically trusted for resolution:

```swift
client.links.appBaseURL = URL(string: "https://app.example.com")
// Optional: extra http(s) hosts to accept when resolving incoming links
client.links.trustedLinkHosts = ["links.example.com"]
```

### Resolve an incoming link

```swift
// From a universal link (e.g. .onContinueUserActivity / scene delegate)
let target = try await client.links.resolve(userActivity: activity)

switch target {
case .document(let id):              openDocument(id)
case .documentAlias(let ref):        openDocument(/* ref resolved server-side */)
case .invitation(let token):         acceptInvite(token)
case .magicLink(let token, _):       try await client.auth.magicLinkVerify(token: token)
case .unknown(let url):              route(url)   // not a Primitive link — hand to your own router
}
```

- `resolve(url:)` / `resolve(userActivity:)` are `async throws` — they hit the network only to resolve a document **alias** to its ID (`GET /document-aliases/...`), throwing `JsBaoError(.aliasNotFound)` if it doesn't resolve, or `.invalidArgument` when an activity carries no URL.
- `parse(url:)` is the synchronous, offline counterpart — same `LinkTarget` shapes, but a `.documentAlias` comes back unresolved (no network). Use it when you only need to branch on the kind.

`LinkTarget` cases: `.document(id:)`, `.documentAlias(_:)`, `.invitation(token:)`, `.magicLink(token:purpose:)`, `.unknown(_:)`.

### Build outbound links

```swift
let share  = client.links.shareURL(forDocument: documentId)        // {appBaseURL}/document/{id}
let invite = client.links.inviteAcceptURL(inviteToken: token)      // {appBaseURL}/invite/accept?inviteToken=...
```

Both return `URL?` and are `nil` until `appBaseURL` is configured. `inviteAcceptURL` produces the same URL shape Primitive's default invitation emails use, so a share sheet and an emailed invite land on the same route.
{{/lang}}

---

## Auth Events

These are the canonical events.

{{#lang ts}}
`auth-failed` and `auth:onlineAuthRequired` are the ones most apps must handle. Register handlers with `client.on(...)`:
{{/lang}}
{{#lang swift}}
`authFailed` and `authState` are the ones most apps must handle. Each event has exactly one payload type that names it, so `client.stream(for: AuthFailedEvent.self)` (an `AsyncStream`, for a `for await` loop in a `.task`) or `client.observeOnMainActor(AuthFailedEvent.self) { ... }` (a main-actor callback, returning an `EventSubscription` to hold for as long as you want the handler live) resolves the subscription with no separate event argument:
{{/lang}}

{{ example: auth/auth-events }}

A rejected token refresh reports a `reason` from a small, stable set. The launch-time refresh emits no failure event at all — a signed-out start is not an auth failure. A refresh triggered by a 401 on a request reports `reason: "refresh_failed"` (with no message). A refresh from any other trigger reports the trigger itself as the `reason` (the same cause vocabulary as the success event, e.g. `"networkMode:online"`, `"ws-challenge"`), with `"refresh_invalid"` as the fallback when no trigger is attached. Branch on `reason`, and treat unknown values as a generic failure — the set may grow.

{{#lang ts}}
**Don't:**

```typescript
// WRONG — there is no onAuthStateChange or signInWithGoogle on JsBaoClient.
// Use the events above and the explicit start*/verify* methods.
client.onAuthStateChange((u) => {});
await client.signInWithGoogle();
```
{{/lang}}

### Minimal handler

{{ example: auth/auth-events-minimal }}

---

## Per-App User Disable

Admins can disable a user's access to a single app without deleting their global user account. The `AppUser` record carries:

| Field | Meaning |
|---|---|
| `status` | `"active"` or `"disabled"`. Missing/null is treated as `"active"`. |
| `disabledAt` | Timestamp the user was disabled. |
| `disabledBy` | `adminId` that performed the disable. |

When `status === "disabled"`:

- Every auth-completion endpoint (OAuth callback, magic-link verify, OTP verify, and any other sign-in path) rejects with `AUTH_USER_DISABLED` before issuing tokens.
- The user's open WebSocket connections are force-disconnected by the server's connection layer.
- Existing access tokens are revoked; in-flight workflow runs the user started are terminated.

Admin endpoints (admin token required):

```http
PUT /admin/api/apps/{appId}/users/{userId}/disable
PUT /admin/api/apps/{appId}/users/{userId}/enable
```

The admin console exposes the same toggles. App code does not need to special-case disabled users — the platform rejects them before they get an authenticated session. Make sure error UIs render `AUTH_USER_DISABLED` differently from generic auth errors so the user knows to contact an admin.

---

## Token Inspection & Manual Token

{{ example: auth/session-state }}

`isAuthenticated()` returns true when either an online JWT or an unlocked offline identity is present.

To read or manually set the token:

{{ example: auth/manual-token }}

**Don't:**

{{#lang ts}}
```typescript
// WRONG — there is no client.auth.setToken. It's client.setToken(...).
client.auth.setToken(jwt);

// WRONG — opening documents before auth is ready throws or fails silently.
const doc = await client.openDocument(id);     // before await client.waitForAuthReady()
```
{{/lang}}
{{#lang swift}}
```swift
// WRONG — opening documents before auth is ready throws or fails silently.
let doc = try await client.openDocument(id)  // before try await client.waitForAuthReady()
```
{{/lang}}

---

## JWT Persistence

Optional — opt in through the client's `auth` options so a relaunch reuses the short-lived token while it's still within the refresh window, instead of forcing a fresh sign-in. The client reports whether persistence is on.

{{ example: auth/jwt-persistence }}

{{#lang ts}}
The token is written to browser storage; `storageKeyPrefix` namespaces it (required for multiple clients on the same origin). `getAuthPersistenceInfo()` returns `{ mode: "memory" | "persisted", hydrated }` — `hydrated` is whether a cached token was restored this session. Persisted tokens within ~2 min of expiry are not reused; tokens are cleared on logout and on `auth-failed`.

---

## Refresh Proxy (Safari / 3rd-party cookie blockers)

```typescript
const client = await initializeClient({
  // ...
  auth: {
    refreshProxy: {
      baseUrl: `${window.location.origin}/proxy`,
      cookieMaxAgeSeconds: 7 * 24 * 60 * 60,
      enabled: true, // optional; defaults to true when refreshProxy is provided
    },
  },
});
```

`baseUrl` must be a same-origin worker that forwards `/auth/*` and `/oauth/callback` to `/app/:appId/api/auth/*`. When configured, `magicLinkVerify`, `otpVerify`, `handleOAuthCallback`, `logout`, and the OAuth-code static helper all route through the proxy.
{{/lang}}
{{#lang swift}}
The token is persisted through the client's storage provider (SQLite by default); `storageKeyPrefix` namespaces it. The Keychain holds offline-access grants, not the JWT session. `waitForAuthBootstrap()` restores any persisted session, so an authenticated user stays signed in on relaunch. `client.authPersistenceInfo` is a `[String: JSONValue]` of `["mode": "persisted" | "memory", "prefix": <storageKeyPrefix>]`. Tokens within ~2 min of expiry are not reused, and one with no readable expiry is treated as unusable. Startup does not depend on persistence being on: whenever the client launches without a usable token it tries a cookie-based refresh, so a user whose `rt-{appId}` cookie is still alive is signed back in either way. A refresh that the server rejects at startup just leaves the client signed out — it clears any persisted token and emits no `authFailed`. Any `logout()` clears the persisted JWT — `wipeLocal` is not required — and outside startup, a refresh the server rejects with 401 ends the session: the token is cleared, the persisted JWT is deleted, and `auth:state` reports `authenticated: false`.
{{/lang}}

---

## Logout

{{ example: auth/logout }}

{{#lang ts}}
`logout` accepts an options object to control teardown:

```typescript
await client.logout({
  redirectTo: "/signed-out",
  wipeLocal: true,         // delete all locally cached document data + KV cache
  revokeOffline: true,     // also delete the persisted offline grant
  clearOfflineIdentity: true, // default true — drop in-memory offline identity
  waitForDisconnect: true, // wait for WS close before resolving
});
```

Logout fires `auth:logout` immediately and `auth:logout:complete` when finished.
{{/lang}}
{{#lang swift}}
`auth.logout(options:)` takes a `LogoutOptions` — `wipeLocal` (delete locally cached document data + KV cache), `revokeOffline` (also revoke any stored offline grant), `clearOfflineIdentity` (defaults `true`), `waitForDisconnect` (await WebSocket teardown before returning; defaults `false`). Logout fires `auth:logout` immediately and `auth:logout:complete` when finished — the same event names as JS.
{{/lang}}

---

## Auth State in Apps

The neutral signal is the client's own auth state: `client.isAuthenticated()` is the live boolean, and `client.waitForAuthReady()` waits until a user is authenticated, returning the `userId` and auth `mode` (`online`/`offline`) — it resolves on sign-in, not on mere client readiness, and times out if nobody signs in. See [Token Inspection & Manual Token](#token-inspection--manual-token) for the compiled calls. Gate the app's main layout on that signal so child views can assume an authenticated user, and react to auth loss centrally. The patterns below are **framework wiring** around that flag — the starter template implements them; if you're not using it, replicate them.

{{#lang ts}}
The template ([primitive-vue-template](https://github.com/Primitive-Labs/primitive-vue-template)) provides a `userStore` (Pinia) and `AppLayout` that mirror `client.isAuthenticated()` into a reactive `userStore.isAuthenticated`.

### Two key flags (template store)

- **`isInitialized`** — one-way. Becomes `true` once the store has wired listeners and loaded auth config. Does not mean the user is signed in. Used by router guards.
- **`isAuthenticated`** — live reactive. Can flip in either direction at any time (token expiry, server invalidation, login).

### Layout gate (recommended default)

Web-template glue (Vue) gating `<router-view>` on the auth flag:

```vue
<template v-if="!userStore.isAuthenticated">
  <LoadingSpinner />
</template>
<div v-else>
  <router-view />  <!-- currentUser guaranteed non-null inside here -->
</div>
```

Components inside the gate **don't** need to null-check `currentUser` or watch `isAuthenticated`. If auth is lost, they unmount.

### Onboarding hand-off (template)

Every sign-in method in the template funnels through a shared `/onboarding` route (`PrimitiveOnboarding`): `PrimitiveLogin` (OTP verify and passkey conditional UI) and the OAuth/magic-link callback push to it with `continueURL`, `isNewUser`, and `promptAddPasskey` as query params (state survives a refresh), and the route handles profile completion and the passkey prompt before navigating to the continue URL. Configure the target with `onboardingRoute` or `onboardingUrl` on `PrimitiveLogin`/`PrimitiveOauthCallback`; omit both and sign-in navigates straight to the continue URL. Customize the step (name/avatar required or optional, passkey prompt) via the `PrimitiveOnboarding` props where the route is declared.

### Reactive watchers (downstream stores)

Web-template glue (Vue) reacting to auth-state transitions — initialize on sign-in, reset on sign-out:

```typescript
watch(
  () => userStore.isAuthenticated,
  async (isAuth, wasAuth) => {
    if (isAuth && !wasAuth) await myStore.initialize();
    else if (!isAuth && wasAuth) myStore.reset();
  },
  { immediate: true }
);
```
{{/lang}}
{{#lang swift}}
The template ([primitive-swift-template](https://github.com/Primitive-Labs/primitive-swift-template)) provides `PrimitiveAppState` + `PrimitiveAuthManager` (`@Published isAuthenticated`/`userId`/`loginState`) and `AuthGateView` — SwiftUI glue that mirrors `client.isAuthenticated()` into observable state.

### Layout gate (recommended default)

SwiftUI glue (PrimitiveApp package) — `AuthGateView(appState:appName:authManager:) { content }` is the layout gate; it walks initializing → login (`PrimitiveLoginView`) → connecting → connected and only renders `content` when connected, so views inside never null-check the user:

```swift
AuthGateView(appState: appState, appName: "MyApp", authManager: authManager) {
  RootView()  // user guaranteed non-nil inside here
}
```

### Reactive observers (downstream state)

SwiftUI/Combine glue reacting to auth-state transitions — subscribe to `authManager.$isAuthenticated` to initialize or reset downstream state:

```swift
authManager.$isAuthenticated
  .sink { isAuth in
    if isAuth { Task { await myStore.initialize() } }
    else { myStore.reset() }
  }
  .store(in: &cancellables)
```
{{/lang}}

### Initialization order

1. Auth ready (`isAuthenticated` true, or `await client.waitForAuthReady()`)
2. Open documents (`documents.open(...)`)
3. Query data

Don't open documents or hit data APIs before step 1.

---

## Deferred Grant Resolution at Signup

Sign-in resolves any pending `DeferredDocumentPermission` and `DeferredGroupAdd` records for the user's email automatically inside `UserProvisioningService`.

Implications:

1. **Don't re-grant after signup.** If a doc was shared with the email pre-signup, the new user already has access — the deferred grant resolved automatically.
2. **Domain-mode apps re-validate at resolution.** Deferred grants for emails outside allowed domains are silently dropped.
3. **`invitation`/`accepted` WS events fire after resolution** — subscribe to refresh the inviter's UI.

See the [Invitations guide](AGENT_GUIDE_TO_PRIMITIVE_INVITATIONS.md#deferred-grants).

---

{{#lang ts}}
## Invite Token Persistence Across Auth Round-Trips (Template Pattern)

The neutral contract is small: pass the `inviteToken` to the auth call that completes sign-in (`client.otpVerify(...)`, `magicLinkVerify`, or the OAuth flow) so the server resolves deferred grants atomically at verification time. The only hard part is that when the recipient isn't signed in, the token must **survive the auth redirect** — and that round-trip is web-template glue.

The template implements the round-trip in `src/lib/inviteToken.ts` (sessionStorage); the one Primitive call below is `client.otpVerify(email, code, { inviteToken })`:

```typescript
import {
  isPlausibleInviteToken,
  setPendingInviteToken,
  getPendingInviteToken,
  clearPendingInviteToken,
} from "@/lib/inviteToken";

// Stash on the accept page when user is signed out:
if (!user.isAuthenticated && isPlausibleInviteToken(rawToken)) {
  setPendingInviteToken(rawToken);  // saved to sessionStorage
}

// Consume during auth (done automatically by userStore):
const inviteToken = getPendingInviteToken() ?? undefined;
await client.otpVerify(email, code, { inviteToken });
if (inviteToken) clearPendingInviteToken();
```

The token is saved to `sessionStorage` under the key `"primitive:pendingInviteToken"`. The template's `userStore` automatically reads and forwards it through every auth path — `login()` (OAuth), `verifyOtp()`, the magic-link callback, and `registerPasskey()`. For OAuth (which round-trips through Google), the token also rides in the OAuth `state` parameter so the callback page can recover it even if the OAuth response opens in a tab without the original `sessionStorage`.

**`InviteAcceptPage` (template-provided):** The template ships a ready-made component at `src/pages/InviteAcceptPage.vue`, routed at `/invite/accept?inviteToken=...`. It handles:

- **Signed-in user** — prompts "Accept with this account or sign out and use a different one." Calls `client.invitations.accept(token)` on confirm.
- **Signed-out user** — stashes the token in `sessionStorage` and redirects to login. After sign-in the pending token is consumed inside the auth method.
- **Error states** — `INVITE_TOKEN_EXPIRED`, `INVITE_ALREADY_ACCEPTED`, `INVITE_TOKEN_INVALID`, and generic server errors.

If you're using the template, this route is already wired in `src/router/routes.ts`. Point your invitation emails at `${yourApp.baseUrl}/invite/accept?inviteToken=${token}`. You do **not** need to rebuild any of this.

**When building on the raw API** (without the template): stash and restore the token manually, pass it to the auth method that completes sign-in, and clear it immediately afterward. The `isPlausibleInviteToken` helper (16–512 chars, URL-safe charset) guards against obviously invalid input before touching storage.
{{/lang}}

---

## Test User Sign-In (per-app whitelist)

There is **no `primitive test-users` CLI command**. The bypass is server-side: an OTP request for an email shaped like `<base-local>+primitivetest<suffix>@<base-domain>` accepts the magic code `"000000"` instead of the emailed code, but **only when the base address is on the app's `testAccountBaseEmails` whitelist**.

This is the recommended sign-in for local/dev builds as well as CI: sign in through the app's real login UI with a derived address and `"000000"`, so routine development exercises the production auth flows on short-lived, member-scoped tokens.

{{ example: auth/test-user-otp }}

Guardrails:

- Per-app whitelist. The base address (`alice@example.com`) must be on the app's `testAccountBaseEmails` list — explicit owner consent.
- Only `+primitivetest<suffix>` derivatives are eligible. The bare base is never a test account.
- First verify provisions the derived user through the standard signup path and returns the real `isNewUser`, so first-run/new-user flows are testable through the bypass. Signup-mode gates apply as 403s exactly like a normal signup: `INVITATION_REQUIRED` (invite-only, no invitation), `ADDED_TO_WAITLIST` (invite-only with waitlist — the address is added), `DOMAIN_NOT_ALLOWED` (domain mode); an `inviteToken` is honored and provisions with the invitation's role (member-role invitations only — see the reserved-email boundary below).
- Issued tokens are short-lived (~30 minutes) and carry a `primitiveBypass: true` claim that gets re-checked on every request, so removing the base from the whitelist revokes sessions immediately.
- `+primitivetest*` accounts can sign in as ordinary members but are reserved at admin / owner / invitation boundaries — they cannot hold those roles. Admin-only paths (e.g. starting a `runAs = "system"` workflow from the client) need a real admin sign-in.

The whitelist is `[app].testAccountBaseEmails` in `app.toml` (max 50 bases per app), applied with `primitive config push --only app`. The web-admin settings UI edits the same list.

**Invite-only apps: pre-create the member.** A fresh derived address on an invite-only app is gated at the email step (`INVITATION_REQUIRED` / `ADDED_TO_WAITLIST`) before any code is accepted. For CI, skip the invitation flow: `primitive users create <base>+primitivetest-<x>@<domain>` adds the membership directly (no invitation email), and the OTP sign-in with `"000000"` then proceeds as an existing member (the response's `isNewUser` is `false` — pre-created members can't exercise new-user flows).

**Entitlement-gated features: grant via an admin metadata write.** When features key off resource metadata (subscription tier, feature flags), an app-level owner/admin can set the test user's state directly — `primitive metadata set user <userId> <category> --data '{...}'` — because owners/admins bypass category write rules (see the [Resource Metadata guide](AGENT_GUIDE_TO_PRIMITIVE_RESOURCE_METADATA.md)). The full CI sequence: whitelist the base once, `users create` the derived address, sign in with `"000000"`, grant metadata — a headless session gets a fresh, entitled member with zero human interaction.

**Don't use this in production user flows.**

---

## Customizing Email Templates

The `magic-link` and `otp` sign-in emails are two of the transactional types Primitive sends; override either with a custom subject and branded HTML/text body for auth emails. Email templates are a cross-cutting configuration surface — the full type list, template variables, override/revert model, and CLI commands live in the [Configuration guide](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md). Custom types are triggered from `email.send` workflow steps.

---

## Implementation Checklist

1. Call `getAuthConfig()` to discover enabled methods before rendering UI.
2. Implement at least one primary method (OAuth or Magic Link).
3. Handle the OAuth callback (`code` + `state`) and the Magic Link `magic_token`.
{{#lang ts}}
4. Listen to `auth-failed` and `auth:onlineAuthRequired` (minimum) to prompt re-login.
5. Catch `AuthError` and switch on `err.code` (use `AUTH_CODES` constants).
6. Gate your app layout on `isAuthenticated` so child components can assume `currentUser`.
7. Watch `isAuthenticated` reactively in downstream stores (it changes both directions).
8. Sequence: auth ready → open documents → query data.
9. The template's `/onboarding` route owns the post-sign-in passkey prompt and profile completion for every method; a hand-rolled auth UI should offer passkey registration when `promptAddPasskey` is true after verify.
10. Customize email templates via CLI if you need branded auth emails.
{{/lang}}
{{#lang swift}}
4. Listen to `.authFailed` and `.authOnlineRequired` (minimum) to prompt re-login.
5. Catch `AuthError` and switch on `error.code`.
6. Gate your app layout on `isAuthenticated` (via `AuthGateView`) so child views can assume an authenticated user.
7. Observe `authManager.$isAuthenticated` in downstream state (it changes both directions).
8. Sequence: auth ready → open documents → query data.
9. Customize email templates via CLI if you need branded auth emails.
{{/lang}}
