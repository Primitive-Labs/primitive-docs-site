# Agent Guide to Primitive Authentication

Implementing auth flows for Primitive apps. All methods live on `JsBaoClient` (package: `js-bao-wss-client`).

## Auth Methods

| Method | When to use |
|--------|-------------|
| OAuth (Google) | Primary auth, redirect-based |
| Magic Link | Passwordless email link |
| OTP | 6-digit email code (10 min expiry) |
| Sign in with Apple | Native one-call sign-in (`signInWithApple`); gate on `hasApple` |
| Passkey | Native one-call sign-in/registration via `AuthenticationServices`; for returning users |

Each method must be enabled in the Admin Console. Check availability with `getAuthConfig()` before showing UI.

## Client Setup (no template required)

All flows below run on a plain client — no starter template needed:

```swift
  let client = JsBaoClient(options: JsBaoClientOptions(
    apiUrl: "https://primitiveapi.com",
    wsUrl: "wss://primitiveapi.com",
    appId: "YOUR_APP_ID"
  ))

  // Wait for the bootstrap to restore a persisted session (if any).
  try await client.waitForAuthBootstrap()
  if client.isAuthenticated() {
    let userId = try await client.waitForUserId(timeout: 5)
    print("signed in as \(userId)")
  }
```

In the starter template this wiring is owned for you by `PrimitiveAppState.initialize()` + `PrimitiveAuthManager`.

## Discovering Available Methods

```swift
  let config = try await client.auth.getAuthConfig()
  // AuthConfigInfo: appId, name, mode, waitlistEnabled,
  //   googleOAuthEnabled, googleClientId, hasOAuth, redirectUris,
  //   passkeyEnabled, passkeyRpId, passkeyRpName, passkeyRpConfig, hasPasskey,
  //   appleSignInEnabled, hasApple, magicLinkEnabled, otpEnabled

  let methods = (
    google: config.hasOAuth,
    apple: config.hasApple,
    magicLink: config.magicLinkEnabled,
    otp: config.otpEnabled,
    passkey: config.hasPasskey
  )
```

`hasOAuth` is true when Google OAuth is enabled (the flag defaults to enabled when both `googleClientId` and the server-side `googleClientSecret` are configured). `magicLinkEnabled` and `otpEnabled` default to `true` unless explicitly disabled in the Admin Console.
`hasApple` reports Sign in with Apple availability (`appleSignInEnabled` plus configured Apple audiences); gate the native `signInWithApple` button on it.

## Server App Settings ↔ Client Contract

Server-side app settings must align with the origin the client app is served from. These settings live in `config/app.toml` and sync with `primitive sync push` — **edit the TOML and push as the default path** so the server stays in lockstep with what's checked in. The `primitive apps update --flag` calls below are a quick imperative escape hatch; a flag change mutates the server but leaves `app.toml` stale, so the next `primitive sync push` silently reverts it unless you mirror the change back into the TOML. Inspect the live values with `primitive apps get`; the relevant fields:

| Server field | Contract | Set via |
|---|---|---|
| `corsAllowedOrigins` | Must contain the exact serving origin (scheme+host+port). `corsMode` defaults to `custom` — an empty list blocks every cross-origin request. | `[cors]` (`mode`, `allowedOrigins`, `allowCredentials`) in `app.toml` → `sync push`; `--cors-origins "<o1>,<o2>"` for a one-off |
| `redirectUris` | OAuth callbacks are validated against this whitelist — a non-listed callback URL returns 400 `Invalid redirect URI`. | Not in `app.toml` — `primitive apps update --redirect-uris "<uri1>,<uri2>"` (non-localhost must be https) or the Admin Console |
| `baseUrl` | Used for links in auth emails / redirects. | `[app].baseUrl` in `app.toml` → `sync push`; `--base-url <url>` for a one-off |
| Provider toggles | What `getAuthConfig()` reports. | `[auth]` in `app.toml` (`googleOAuthEnabled`, `magicLinkEnabled`, `otpEnabled`, `appleSignInEnabled`, `appleAudiences`) → `sync push`. Enable Sign in with Apple by setting `appleSignInEnabled = true` and `appleAudiences = ["<bundle-id>"]` (`hasApple` then reports true). |


Dev → prod checklist: add the production origin to `[cors].allowedOrigins` and set `[app].baseUrl` in `app.toml`, `primitive sync push`, add the production OAuth callback to `redirectUris` (`primitive apps update --redirect-uris` or Admin Console — not in `app.toml`), and re-check `getAuthConfig()` reports the expected methods.

---

## OAuth (Google)

### Start the flow

```swift
  let hasOAuth = await client.checkOAuthAvailable()
  if hasOAuth {
    // Open this URL in a browser / ASWebAuthenticationSession.
    let authUrl = try await client.startOAuthFlow(
      redirectUri: redirectUri,
      continueUrl: continueUrl
    )
    _ = authUrl
  }

  // On the callback (?code=&state=): token is stored, WS reconnects.
  try await client.handleOAuthCallback(code: code, state: state)
```

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

### Handle the callback (instance method — preferred)

When the callback page can construct a client (you already have the JWT or are happy to re-init), extract `code`/`state` from the callback and pass them to `handleOAuthCallback`:

```swift
  if !code.isEmpty && !state.isEmpty {
    try await client.handleOAuthCallback(code: code, state: state)
    // Token now stored, WebSocket reconnected. Navigate.
  }
```

### Handle the callback (static method — when no client yet)

```swift
  let token = try await JsBaoClient.exchangeOAuthCode(
    apiUrl: apiUrl,
    appId: appId,
    code: code,
    state: state
  )
  // Persist however your app does (e.g. Keychain), or pass to the client.
```


**Don't:**

```swift
// WRONG — handleOAuthCallback does not return the token; it stores it.
let token = try await client.handleOAuthCallback(code: code, state: state)
```

### Native one-call sign-in (Google / Apple)

`startOAuthFlow` + `handleOAuthCallback` are the raw building blocks. For the native experience use the one-call helpers — each presents the system auth sheet, runs the redirect + code exchange, applies the session token (cause `"google"` / `"apple"`, emitting `.authSuccess` / `.authState`), and re-authenticates the WebSocket. Both return the signed-in `userId` and the server's `isNewUser` flag:

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

Gate the buttons on the auth config: `hasOAuth` for Google, `hasApple` for Apple (`AuthConfigInfo` also carries `appleSignInEnabled`). The starter template's `PrimitiveAuthManager` wraps both helpers and renders only the providers `availableProviders` reports.

---

## Magic Link

### Request + verify

```swift
  // Request the email. Pass the magic-link callback redirect URI.
  _ = try await client.auth.magicLinkRequest(
    email: email,
    redirectUri: "https://app.example.com/auth/magic-callback"
  )

  // On the callback, read the `magic_token` value and verify it.
  let result = try await client.auth.magicLinkVerify(token: magicToken)
  // Token is now stored on the client and the WS auto-connects.
  let user = result.user
  let isNewUser = result.isNewUser ?? false
```

`auth.magicLinkRequest(email:redirectUri:)` takes the `redirectUri` as a required argument. `auth.magicLinkVerify(token:inviteToken:)` returns a `MagicLinkVerifyResult` (`.user`, `.promptAddPasskey?`, `.isNewUser?`).

### Reading the token (callback page)

The callback delivers the token as a `magic_token` value — **not** `token`, `magicToken`, or `code`:

```swift
  let magicToken = URLComponents(url: callbackURL, resolvingAgainstBaseURL: false)?
    .queryItems?.first(where: { $0.name == "magic_token" })?.value

  if let magicToken {
    let result = try await client.auth.magicLinkVerify(token: magicToken)
    // Token is now stored on the client and WS auto-connects.
    if result.isNewUser == true { showOnboarding() }
    if result.promptAddPasskey == true { offerPasskeyRegistration() }
  }
```


To accept an invitation server-side at verify time (so the deferred grant resolves to the signing-in user even when emails differ), pass `inviteToken`:

```swift
  let result = try await client.auth.magicLinkVerify(
    token: magicToken,
    inviteToken: inviteTokenFromUrl
  )
  let user = result.user
  let isNewUser = result.isNewUser ?? false
```

---

## OTP (Email Code)

```swift
  _ = try await client.auth.otpRequest(email: email)

  // User enters the 6-digit code from the email.
  let result = try await client.auth.otpVerify(email: email, code: code)
  // Token is now stored on the client and the WS auto-connects.
  let user = result.user
  let isNewUser = result.isNewUser ?? false
```

`auth.otpVerify(email:code:)` returns an `OtpVerifyResult` (`.user`, `.isNewUser?`). To accept an invitation at verify time, pass the `inviteToken` parameter: `auth.otpVerify(email:code:inviteToken:)`.

### Error handling

`AuthError` is thrown for non-2xx responses with a machine-readable code:

```swift
  do {
    _ = try await client.auth.otpVerify(email: email, code: code)
  } catch let error as AuthError {
    switch error.code {
    case .invalidToken,          // bad/expired code
         .tokenExpired,          // token expired
         .invitationRequired,    // invite-only app, no invitation
         .domainNotAllowed,      // domain-mode app, email not in allowed domains
         .addedToWaitlist,       // waitlist enabled, user added
         .waitlistEntryUpdated,  // existing waitlist entry updated
         .magicLinkNotEnabled,   // magic link off in admin console
         .inviteTokenInvalid,    // bad invite token
         .inviteTokenExpired,    // invite token expired
         .inviteAlreadyAccepted: // invite already used
      showUserMessage(error.message)
    default:
      throw error
    }
  }
```

> **Caveat on OTP disabled.** When OTP is disabled the request endpoint returns a plain 400 with the message `"OTP authentication is not enabled for this app"` and **no `code` field**. Don't rely on a code to detect that case — gate the OTP UI on `getAuthConfig()`'s `otpEnabled` up front instead.

`AuthCode` also carries the SDK-generated cases `.tokenInvalid`, `.refreshFailed`, `.networkError`, and `.unauthorized`, plus `.passkeyNotEnabled` and `.memberInvitationsDisabled` from the server. Server codes outside the enum (e.g. rate limiting) arrive with `code == nil` — fall back to `error.message`.

The same `AuthError` codes apply to `magicLinkRequest`/`magicLinkVerify`.

---

## Passkeys

The client wraps Apple's `AuthenticationServices` in two one-call helpers on `client.auth` (iOS/macOS), so app code never touches the WebAuthn challenge handshake directly.

### Sign in

`signInWithPasskey` runs the discoverable-credential flow — the system sheet lists the passkeys saved for the app, the user picks one, and the call applies the session token (cause `"passkey"`, emitting `.authSuccess` / `.authState`) and re-authenticates the WebSocket. It works without an existing session.

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

---

## Auth Events

These are the canonical events.

`authFailed` and `authState` are the ones most apps must handle. Register handlers with `client.events.on(...)`, which delivers typed event structs and returns an `EventSubscription` — hold it for as long as you want the handler live:

```swift
  // Token refresh failed or server invalidated session — prompt re-login.
  let failed = client.events.on(.authFailed) { (event: AuthFailedEvent) in
    redirectToLogin()
  }

  // Token applied successfully (login, refresh, or OAuth callback).
  let success = client.events.on(.authSuccess) { (event: AuthSuccessEvent) in
    // event.cause names the operation that produced the token.
    // Treat unknown values as a generic success — the set may grow.
  }

  // Generic state-machine event — fires on transitions.
  let state = client.events.on(.authState) { (event: AuthStateEvent) in
    // event.authenticated, event.userId, event.mode (NetworkMode)
  }
```


### Minimal handler

```swift
  func promptLogin() { navigateToLogin() }
  let failed = client.events.on(.authFailed) { (_: AuthFailedEvent) in promptLogin() }
  let state = client.events.on(.authState) { (event: AuthStateEvent) in
    if !event.authenticated { promptLogin() }
  }
```

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

```swift
  let signedIn = client.isAuthenticated() // Bool

  // Wait until a userId is available.
  let userId = try await client.waitForUserId(timeout: 5)

  // Wait until authenticated AND offline DBs are ready. Returns the mode.
  let ready = try await client.waitForAuthReady(timeout: 6)
```

`isAuthenticated()` returns true when either an online JWT or an unlocked offline identity is present.

To read or manually set the token:

```swift
  // Manually set a token (e.g. obtained out-of-band). Triggers authSuccess
  // and pushes through the normal apply-token pipeline.
  client.updateToken(jwt, cause: "external")

  // Read the current token via the decoded JWT payload, or track it from the
  // authSuccess event.
  let payload = client.getJwtPayload()
```

**Don't:**

```swift
// WRONG — opening documents before auth is ready throws or fails silently.
let doc = try await client.openDocument(id)  // before try await client.waitForAuthReady()
```

---

## JWT Persistence

Optional — opt in through the client's `auth` options so a relaunch reuses the short-lived token while it's still within the refresh window, instead of forcing a fresh sign-in. `getAuthPersistenceInfo()` reports whether persistence is on.

```swift
  let client = JsBaoClient(options: JsBaoClientOptions(
    apiUrl: "https://primitiveapi.com",
    wsUrl: "wss://primitiveapi.com",
    appId: "YOUR_APP_ID",
    auth: AuthConfig(
      persistJwtInStorage: true,
      storageKeyPrefix: "my-app"
    )
  ))

  try await client.waitForAuthBootstrap()
  // ["mode": "persisted" | "memory", "prefix": <storageKeyPrefix>]
  let info = client.getAuthPersistenceInfo()
```

The token is persisted through the client's storage provider (SQLite by default); `storageKeyPrefix` namespaces it. The Keychain holds offline-access grants, not the JWT session. `waitForAuthBootstrap()` restores any persisted session, so an authenticated user stays signed in on relaunch. `getAuthPersistenceInfo()` returns `["mode": "persisted" | "memory", "prefix": <storageKeyPrefix>]`. Tokens within ~2 min of expiry are not reused; an expired persisted token triggers a cookie-based refresh on startup and is cleared if that refresh fails. `logout(wipeLocal: true)` clears the persisted JWT.

---

## Logout

```swift
  try await client.auth.logout(options: LogoutOptions(
    wipeLocal: true, // delete locally cached document data + KV cache
    waitForDisconnect: true // wait for the WS to close before resolving
  ))
```

`auth.logout(options:)` takes a `LogoutOptions` — `wipeLocal` (delete locally cached document data + KV cache), `revokeOffline` (also revoke any stored offline grant), `clearOfflineIdentity` (defaults `true`), `waitForDisconnect` (await WebSocket teardown before returning; defaults `false`). Logout fires `auth:logout` immediately and `auth:logout:complete` when finished — the same event names as JS.

---

## Auth State in Apps

The neutral signal is the client's own auth state: `client.isAuthenticated()` is the live boolean, and `client.waitForAuthReady()` gates work until auth (and offline DBs) are ready — see [Token Inspection & Manual Token](#token-inspection--manual-token) for the compiled calls. Gate the app's main layout on that signal so child views can assume an authenticated user, and react to auth loss centrally. The patterns below are **framework wiring** around that flag — the starter template implements them; if you're not using it, replicate them.

The template ([swift-primitive-app-dev](https://github.com/Primitive-Labs/swift-primitive-app-dev)) provides `PrimitiveAppState` + `PrimitiveAuthManager` (`@Published isAuthenticated`/`userId`/`loginState`) and `AuthGateView` — SwiftUI glue that mirrors `client.isAuthenticated()` into observable state.

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


---

## Test User Sign-In (per-app whitelist)

There is **no `primitive test-users` CLI command**. The bypass is server-side: an OTP request for an email shaped like `<base-local>+primitivetest<suffix>@<base-domain>` accepts the magic code `"000000"` instead of the emailed code, but **only when the base address is on the app's `testAccountBaseEmails` whitelist**.

```swift
  // Requires the app owner to have added "alice@example.com" to the app's
  // testAccountBaseEmails whitelist. Then any `alice+primitivetest<suffix>@example.com`
  // derivative becomes a test account that accepts code "000000".
  _ = try await client.auth.otpRequest(email: "alice+primitivetest@example.com")
  _ = try await client.auth.otpVerify(email: "alice+primitivetest@example.com", code: "000000")
  // client is now authenticated; the access token expires in 30 minutes

  // Role-distinguished derivatives (Gmail/Workspace deliver them to the same inbox):
  _ = try await client.auth.otpRequest(email: "alice+primitivetest-teacher@example.com")
```

Guardrails:

- Per-app whitelist. The base address (`alice@example.com`) must be on the app's `testAccountBaseEmails` list — explicit owner consent.
- Only `+primitivetest<suffix>` derivatives are eligible. The bare base is never a test account.
- First verify provisions the derived user through the standard signup path and returns the real `isNewUser`, so first-run/new-user flows are testable through the bypass. Signup-mode gates apply as 403s exactly like a normal signup: `INVITATION_REQUIRED` (invite-only, no invitation), `ADDED_TO_WAITLIST` (invite-only with waitlist — the address is added), `DOMAIN_NOT_ALLOWED` (domain mode); an `inviteToken` is honored and provisions with the invitation's role (member-role invitations only — see the reserved-email boundary below).
- Issued tokens are short-lived (~30 minutes) and carry a `primitiveBypass: true` claim that gets re-checked on every request, so removing the base from the whitelist revokes sessions immediately.
- `+primitivetest*` accounts can sign in as ordinary members but are reserved at admin / owner / invitation boundaries — they cannot hold those roles.

Manage the whitelist with `primitive apps update --test-account-bases …` (max 50 bases per app), or in the web-admin settings UI — both edit the same `testAccountBaseEmails` list.

**Invite-only apps: pre-create the member.** A fresh derived address on an invite-only app is gated at the email step (`INVITATION_REQUIRED` / `ADDED_TO_WAITLIST`) before any code is accepted. For CI, skip the invitation flow: `primitive users create <base>+primitivetest-<x>@<domain>` adds the membership directly (no invitation email), and the OTP sign-in with `"000000"` then proceeds as an existing member (the response's `isNewUser` is `false` — pre-created members can't exercise new-user flows).

**Entitlement-gated features: grant via an admin metadata write.** When features key off resource metadata (subscription tier, feature flags), an app-level owner/admin can set the test user's state directly — `primitive metadata set user <userId> <category> --data '{...}'` — because owners/admins bypass category write rules (see the [Resource Metadata guide](AGENT_GUIDE_TO_PRIMITIVE_RESOURCE_METADATA.md)). The full CI sequence: whitelist the base once, `users create` the derived address, sign in with `"000000"`, grant metadata — a headless session gets a fresh, entitled member with zero human interaction.

**Don't use this in production user flows.**

---

## Customizing Email Templates

The Magic Link, OTP, and other emails Primitive sends can be customized via the CLI:

```bash
primitive email-templates list                       # all types + override status
primitive email-templates get magic-link             # subject + body + variables
primitive email-templates variables magic-link       # available {{vars}}
primitive email-templates set magic-link \
  --subject "Sign in to MyApp" \
  --html-file ./emails/magic-link.html \
  --text-file ./emails/magic-link.txt
primitive email-templates test magic-link            # send test email
primitive email-templates delete magic-link          # revert to default
```

Overrides are tracked by `primitive sync` (TOML in `email-templates/`). Custom templates can be triggered from `email.send` workflow steps — see the [Workflows guide](AGENT_GUIDE_TO_PRIMITIVE_WORKFLOWS.md).

---

## Implementation Checklist

1. Call `getAuthConfig()` to discover enabled methods before rendering UI.
2. Implement at least one primary method (OAuth or Magic Link).
3. Handle the OAuth callback (`code` + `state`) and the Magic Link `magic_token`.
4. Listen to `.authFailed` and `.authOnlineRequired` (minimum) to prompt re-login.
5. Catch `AuthError` and switch on `error.code`.
6. Gate your app layout on `isAuthenticated` (via `AuthGateView`) so child views can assume an authenticated user.
7. Observe `authManager.$isAuthenticated` in downstream state (it changes both directions).
8. Sequence: auth ready → open documents → query data.
9. Customize email templates via CLI if you need branded auth emails.
