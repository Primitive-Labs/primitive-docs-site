# Agent Guide to Primitive Authentication

Use the client’s authentication APIs to sign users in, restore sessions, and sign out. Starter templates provide the UI and session handling.

## Auth Methods

| Method | When to use |
|--------|-------------|
| OAuth (Google) | Primary auth, redirect-based |
| Email sign-in | One email carrying a 6-digit code and, optionally, a sign-in link (15 min expiry) |
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
  //   googleOAuthEnabled,
  //   googleClients: { clients: ["ios": (clientId, redirectUris, usable), …] },
  //   passkeyEnabled, passkeyRpConfig, hasPasskey,
  //   appleSignInEnabled, hasApple, emailSignInEnabled

  let methods = (
    // Google registers a client per platform: this is the provider being
    // enabled AND this app's `ios` entry being usable, in one property so the
    // button and the flow cannot disagree.
    google: config.googleSignInAvailable,
    apple: config.hasApple,
    // ONE email capability: one request sends one email carrying both a code
    // and (when a link can be issued) a link, so there is no method to offer.
    email: config.emailSignInEnabled,
    passkey: config.hasPasskey
  )
```

`getAuthConfig()` returns enabled methods and configured Google clients. Each Google client entry includes `clientId`, `redirectUris`, and `usable`; secrets are excluded.

Availability is the provider being enabled **and** your platform's entry being usable, together.
`AuthConfigInfo.googleSignInAvailable` computes exactly that against the `ios` entry, and `AppConfigInfo.googleAvailable` is projected from it — the launch UI and the flow cannot disagree.

Use `emailSignInEnabled` to control email sign-in UI. It defaults to `true`.
`hasApple` reports Sign in with Apple availability (`appleSignInEnabled` plus configured Apple audiences); gate the native `signInWithApple` button on it.

## Email Sign-In (One Email, One or Two Credentials)

### Request + verify

One request sends ONE email carrying a 6-digit code and — when a link can be
issued — a sign-in link. Nothing chooses a method: the user types the code or
opens the link, and consuming either one retires both.

```swift
  // `redirectUri` is optional, and OMITTING it is how a code-only email is
  // requested: the server renders one from the same template and consults no
  // allow-list. A target that IS supplied must match the app's non-empty
  // `emailRedirectUris`, or the request is rejected 400 `Invalid redirect
  // URI` — nothing degrades to code-only on your behalf.
  _ = try await client.auth.emailSignInRequest(
    email: email,
    redirectUri: "myapp://auth/magic-link"
  )

  // Finish by typing the code from the email...
  let result = try await client.auth.otpVerify(email: email, code: code)

  // ...or by opening the link in the same email, which arrives as a
  // `.magicLink(token:purpose:)` link target. Either one retires both:
  // one email signs the user in once.
  let user = result.user
  let isNewUser = result.isNewUser ?? false
```

Link issuance is **fail-closed**. The email carries a link only when the request
names a redirect target AND that target matches the app's non-empty
`emailRedirectUris`. With no target, or an empty allow-list, the same
`email-sign-in` template renders code-only; a target that misses a non-empty
allow-list is rejected 400 `Invalid redirect URI`. An app that never wants a
link deletes the `{{#if magicLink}}` block from its `email-sign-in` template —
no endpoint renders any other sign-in template, so that removal holds.

`auth.emailSignInRequest(email:redirectUri:)` takes an optional `redirectUri`; omitting it is how a code-only email is requested, and no allow-list is consulted. `auth.magicLinkVerify(token:inviteToken:)` returns a `MagicLinkVerifyResult` (`.user`, `.promptAddPasskey?`, `.isNewUser?`) and `auth.otpVerify(email:code:)` an `OtpVerifyResult`.

### Make the emailed sign-in link open your app (iOS)

The iOS template sends code-only emails by default. To add a link:

- **With a web app:** set the environment’s `webUrl`, allow-list its `/oauth/callback` URL, and set `iosAppId`. Deploy the association document before enabling the `applinks:` entitlement. The template receives universal links through `.onContinueUserActivity` and custom schemes through `.onOpenURL`.
- **Without a web app:** add `<scheme>://auth/magic-link` to `[auth].emailRedirectUris`, then set `authManager.sendsEmailSignInLink = true`. Init supplies the scheme registration and URL routing.

Merge redirect URLs into the existing allow-list; pushing only the new URL removes existing entries. Keep URLs used by installed app builds. Custom-scheme links require the app to be installed; retain code entry as an alternative.


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

## Finishing With the Code

The code half of the same email is verified with the OTP verify call, unchanged:

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

Handle `PASSKEY_USER_VERIFICATION_FAILED` by asking the user to retry and complete biometric or PIN verification. Branch on the returned code, not the app’s current verification policy.

Gate email sign-in UI on `emailSignInEnabled` from `getAuthConfig()`.

`AuthCode` also carries the SDK-generated cases `.tokenInvalid`, `.refreshFailed`, `.networkError`, and `.unauthorized`, plus `.passkeyNotEnabled`, `.passkeyRpNotConfigured`, `.passkeyUserVerificationFailed` and `.memberInvitationsDisabled` from the server. Server codes outside the enum (e.g. rate limiting) arrive with `code == nil` — fall back to `error.message`.

The same `AuthError` codes apply to `emailSignInRequest`/`magicLinkVerify`.

**Don't sign the user out on every failed call.** When a request gets a 401 the client refreshes the token and retries. If the refresh itself can't reach the server, the call does not fail as an HTTP 401 — it fails as a transport error carrying no HTTP status, because that is a transient outage rather than a rejected credential. Retry it instead of clearing the session. Only a refresh the server actually rejects surfaces as `HttpError(status: 401, message: "Invalid credentials")`, and that is the one to sign out on.

---

## Server App Settings ↔ Client Contract

Set authentication fields in the environment’s `app.toml`, then run `primitive config push --only app`. Inspect live values with `primitive apps get`.

| Server field | Contract | Set via |
|---|---|---|
| `corsAllowedOrigins` | Must contain the exact serving origin (scheme+host+port). `corsMode` defaults to `universal`; in `custom` mode an empty list blocks every cross-origin request. | `[cors]` (`mode`, `allowedOrigins`, `allowCredentials`) in `app.toml` → `config push` |
| `googleClients[<type>].redirectUris` | The Google callback is validated against the entries, and the matching one SELECTS the client used for the exchange — a URI listed by no entry returns 400 `Invalid redirect URI`, and a URI may appear in only one entry. | `[auth.google.clients.<type>].redirectUris` in `app.toml` → `config push` (non-localhost must be https) |
| `emailRedirectUris` | The sign-in-link allow-list. **Fail-closed**: with an empty or missing list the sign-in email carries the code alone; a target that misses a non-empty list is rejected 400 `Invalid redirect URI`. New apps are seeded with the localhost dev callback. `http`/`https` match by origin; a custom scheme matches on scheme + authority, so `myapp://auth` covers `myapp://auth/magic-link` but no other scheme or host. | `[auth].emailRedirectUris` in `app.toml` → `config push` |
| `baseUrl` | Used for links in auth emails / redirects. | `[app].baseUrl` in `app.toml` → `config push` |
| Provider toggles | What `getAuthConfig()` reports. | `[auth]` in `app.toml` (`googleOAuthEnabled`, `emailSignInEnabled`, `appleSignInEnabled`, `appleAudiences`) → `config push`. Enable Sign in with Apple by setting `appleSignInEnabled = true` and `appleAudiences = ["<bundle-id>"]` (`hasApple` then reports true). |


Dev → prod checklist: in `app.toml`, add the production origin to `[cors].allowedOrigins`, set `[app].baseUrl`, add the production OAuth callback to the `redirectUris` of the Google client that will redirect there (`[auth.google.clients.web]` for a browser), and add the production sign-in-link callback to `[auth].emailRedirectUris`; run `primitive config push`; then re-check `getAuthConfig()` reports the expected methods.

---

## OAuth (Google)

### Google Client Configuration

Google registers an OAuth client **per platform**, so `app.toml` states one entry per client type: `web`, `ios`, `android`, `desktop`, `chrome-extension`. Each entry carries its own `clientId` and its own `redirectUris`, and — for the types Google issues one for — its own `clientSecret`:

```toml
[auth.google.clients.web]
clientId     = "1234-web.apps.googleusercontent.com"
clientSecret = "{{secrets.GOOGLE_CLIENT_SECRET}}"
redirectUris = ["https://app.example.com/oauth/callback"]

[auth.google.clients.ios]
clientId     = "1234-ios.apps.googleusercontent.com"
redirectUris = ["com.googleusercontent.apps.1234-ios:/oauth2redirect"]
```

| client type | `clientSecret` |
|---|---|
| `web`, `desktop` | **required** — a whole `{{secrets.KEY}}` reference, never the secret itself |
| `ios`, `android`, `chrome-extension` | **rejected** — Google issues none, and the exchange proves possession with PKCE |

A literal `clientSecret` value is rejected with `GOOGLE_CLIENT_SECRET_MUST_BE_SECRET_REF`, and a reference naming a key that doesn't exist with `MISSING_GOOGLE_CLIENT_SECRET_REF`. **A redirect URI belongs to the client that redirects, and selects it at the callback** — so a URI may appear in only one entry, and an iOS custom scheme is not a valid redirect for the web client. Removing a client is how you stop using it: for `web` and `desktop` you cannot blank the secret and keep the entry, because the map would then be invalid.

Configuration reads return secret references, not secret values.

If web sign-in returns `GOOGLE_OAUTH_MISCONFIGURED`, check that the selected Google client references an existing app secret.

### Start the flow

```swift
  // True when Google OAuth is enabled AND the `ios` client entry is usable.
  let googleAvailable = await client.checkOAuthAvailable()
  if googleAvailable {
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

Both are threaded through `signInWithGoogle(...)` as well — `signInWithApple(...)` has no `startOAuthFlow` building block (native Sign in with Apple never redirects), but it takes its own `inviteToken` parameter directly; see [Native one-call sign-in](#native-one-call-sign-in-google--apple) below.

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

For Apple, this only resolves on **first** sign-in (`isNewUser == true`); a repeat sign-in from an existing Apple identity takes a different internal path and does not resolve `inviteToken` grants — call `client.invitations.accept(inviteToken:)` afterward for that case, or for any other post-hoc acceptance. The invite token is validated server-side only after the Apple identity token is cryptographically verified, and only before any user/grant mutation — so any bad, expired, or already-used token throws `HttpError` with `serverCode == "INVITE_TOKEN_INVALID"` (one code for every invalid-token reason, by design, to avoid a validity oracle). A domain-restricted app also throws `DOMAIN_NOT_ALLOWED` when the Apple-verified email itself falls outside `allowedDomains`.

Gate the buttons on the auth config: `googleSignInAvailable` for Google — the provider enabled and this app's `ios` client entry usable — and `hasApple` for Apple (`AuthConfigInfo` also carries `appleSignInEnabled`). The starter template's `PrimitiveAuthManager` wraps both helpers and renders only the providers `availableProviders` reports.

---

## One Account Across Providers

Google and Apple sign-ins with the same email use the same account. Linked provider identities remain attached to that account even if their email changes. Use `isNewUser` to decide whether onboarding is needed.

Email sign-in resolves by the submitted email. Passkeys belong to the account that registered them.

---

## Passkeys

The client wraps Apple's `AuthenticationServices` in two one-call helpers on `client.auth` (iOS/macOS), so app code never touches the WebAuthn challenge handshake directly.

### Sign in

`signInWithPasskey` runs the discoverable-credential flow — the system sheet lists the passkeys saved for the app, the user picks one, and the call applies the session token (cause `"passkeyAuth"`, emitting `.authSuccess` / `.authState`) and re-authenticates the WebSocket. It works without an existing session.

```swift
let result = try await client.auth.signInWithPasskey()   // PasskeySignInResult(user, isNewUser)
```

- `signInWithPasskey(presentationAnchor:preferImmediatelyAvailableCredentials:rpId:)` — pass `preferImmediatelyAvailableCredentials: true` to restrict the prompt to passkeys already on the device and fail fast (no QR/cross-device fallback). Throws `PasskeyError.canceled` when the user dismisses the sheet; server-side failures surface as `HttpError`.

### Register (must be authenticated)

`registerPasskey` adds a passkey to the **currently signed-in** user — sign in by another method first.

```swift
let reg = try await client.auth.registerPasskey(deviceName: "My iPhone")
```

- `registerPasskey(deviceName:inviteToken:presentationAnchor:rpId:)` — `deviceName` labels the credential (server default `"Unknown device"` when omitted). Pass `inviteToken` to fold invitation acceptance into registration. Throws `PasskeyError.canceled` on dismissal.

### Relying party

A native request sends no `Origin`, so name the relying party (RP ID) for every ceremony; with none, the server picks among the app's configured relying parties and a multi-RP app can get the wrong one. The RP ID must be a key of `passkeyRpConfig` and appear in the app's `webcredentials:` entitlement; an unconfigured one fails with `PASSKEY_RP_NOT_CONFIGURED`.

- Client-wide: `AuthConfig(passkeyRpId:)`. Per call: the `rpId:` argument on `signInWithPasskey` / `registerPasskey`, which wins over the config value.
- `PrimitiveAppState.initialize()` sets `passkeyRpId` from the selected environment's `webUrl` host. When the configured relying party is another host (e.g. `passkeyRpConfig` names `example.com`, `webUrl` is `https://app.example.com`), every ceremony fails with `PASSKEY_RP_NOT_CONFIGURED` — override `passkeyRpId(for:)` on your `PrimitiveAppState` subclass to return the configured host (or `nil` to let the server pick), or add the `webUrl` host to `passkeyRpConfig`.
- One ceremony in a `PrimitiveApp` app: `authManager.signInWithPasskey(rpId:)` / `authManager.enrollPasskey(deviceName:rpId:)`.

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

`authFailed` and `authState` are the ones most apps must handle. Each event has exactly one payload type that names it, so `client.stream(for: AuthFailedEvent.self)` (an `AsyncStream`, for a `for await` loop in a `.task`) or `client.observeOnMainActor(AuthFailedEvent.self) { ... }` (a main-actor callback, returning an `EventSubscription` to hold for as long as you want the handler live) resolves the subscription with no separate event argument:

```swift
  // The payload type names its own event, so there is no separate event
  // argument to get wrong. `observeOnMainActor` runs the handler on the main
  // actor, so it can touch UI state directly.

  // Token refresh failed or server invalidated session — prompt re-login.
  let failed = client.observeOnMainActor(AuthFailedEvent.self) { event in
    // event.reason: "refresh_failed" when a 401 on a request triggered the
    // refresh; otherwise the trigger cause itself (e.g. "networkMode:online"),
    // with "refresh_invalid" as the fallback. The launch refresh emits no event.
    redirectToLogin()
  }

  // Token applied successfully (login, refresh, or OAuth callback).
  let success = client.observeOnMainActor(AuthSuccessEvent.self) { event in
    // event.cause names the operation that produced the token.
    // Treat unknown values as a generic success — the set may grow.
  }

  // Generic state-machine event — fires on transitions.
  let state = client.observeOnMainActor(AuthStateEvent.self) { event in
    // event.authenticated, event.userId, event.mode (NetworkMode)
  }
```

A rejected token refresh reports a `reason` from a small, stable set. The launch-time refresh emits no failure event at all — a signed-out start is not an auth failure. A refresh triggered by a 401 on a request reports `reason: "refresh_failed"` (with no message). A refresh from any other trigger reports the trigger itself as the `reason` (the same cause vocabulary as the success event, e.g. `"networkMode:online"`, `"ws-challenge"`), with `"refresh_invalid"` as the fallback when no trigger is attached. Branch on `reason`, and treat unknown values as a generic failure — the set may grow.


### Minimal handler

```swift
  func promptLogin() { navigateToLogin() }
  let failed = client.observeOnMainActor(AuthFailedEvent.self) { _ in promptLogin() }
  let state = client.observeOnMainActor(AuthStateEvent.self) { event in
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
- Existing access tokens are revoked; in-flight task runs the user started are terminated.

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

  // Wait for a userId, including one from a sign-in that starts after the call.
  let userId = try await client.waitForUserId(timeout: 5)

  // Wait until signed in. Returns the userId and the network mode.
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
  let payload = client.jwtPayload
```

**Don't:**

```swift
// WRONG — opening documents before auth is ready throws or fails silently.
let doc = try await client.openDocument(id)  // before try await client.waitForAuthReady()
```

---

## JWT Persistence

Optional — opt in through the client's `auth` options so a relaunch reuses the short-lived token while it's still within the refresh window, instead of forcing a fresh sign-in. The client reports whether persistence is on.

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
  // [String: JSONValue] — ["mode": "persisted" | "memory", "prefix": <storageKeyPrefix>]
  let info = client.authPersistenceInfo
```

Call `waitForAuthBootstrap()` before checking sign-in state. It restores a usable persisted session or attempts a cookie-based refresh. `storageKeyPrefix` separates stored sessions; `authPersistenceInfo` reports the persistence mode and prefix.

### Gotchas when restoring a session

- A rejected startup refresh leaves the client signed out without emitting `authFailed`. Check the resulting authentication state.
- `logout()` clears the persisted session even without `wipeLocal`.
- After startup, a refresh rejected with 401 clears the session and emits `auth:state` with `authenticated: false`.

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

The neutral signal is the client's own auth state: `client.isAuthenticated()` is the live boolean, and `client.waitForAuthReady()` waits until a user is authenticated, returning the `userId` and auth `mode` (`online`/`offline`) — it resolves on sign-in, not on mere client readiness, and times out if nobody signs in. See [Token Inspection & Manual Token](#token-inspection--manual-token) for the compiled calls. Gate the app's main layout on that signal so child views can assume an authenticated user, and react to auth loss centrally. The patterns below are **framework wiring** around that flag — the starter template implements them; if you're not using it, replicate them.

The template ([primitive-swift-template](https://github.com/Primitive-Labs/primitive-swift-template)) provides `PrimitiveAppState` + `PrimitiveAuthManager` (`@Published isAuthenticated`/`userId`/`loginState`, and `authFailure` — the last failed sign-in with its typed `AuthCode` and message, so branch on `authFailure?.code` rather than on message text) and `AuthGateView` — SwiftUI glue that mirrors `client.isAuthenticated()` into observable state.

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
3. **No WebSocket event announces the resolution.** Refresh the inviter's UI on their next read.

See the [Invitations guide](AGENT_GUIDE_TO_PRIMITIVE_INVITATIONS.md#deferred-grants).

---


---

## Test User Sign-In (per-app whitelist)

There is **no `primitive test-users` CLI command**. The bypass is server-side: an OTP request for an email shaped like `<base-local>+primitivetest<suffix>@<base-domain>` accepts the magic code `"000000"` instead of the emailed code, but **only when the base address is on the app's `testAccountBaseEmails` whitelist**.

This is the recommended sign-in for local/dev builds as well as CI: sign in through the app's real login UI with a derived address and `"000000"`, so routine development exercises the production auth flows on short-lived, member-scoped tokens.

```swift
  // Requires the app owner to have added "alice@example.com" to the app's
  // testAccountBaseEmails whitelist. Then any `alice+primitivetest<suffix>@example.com`
  // derivative becomes a test account that accepts code "000000".
  _ = try await client.auth.emailSignInRequest(email: "alice+primitivetest@example.com")
  _ = try await client.auth.otpVerify(email: "alice+primitivetest@example.com", code: "000000")
  // client is now authenticated; the access token expires in 30 minutes

  // Role-distinguished derivatives (Gmail/Workspace deliver them to the same inbox):
  _ = try await client.auth.emailSignInRequest(email: "alice+primitivetest-teacher@example.com")
```

Guardrails:

- Per-app whitelist. The base address (`alice@example.com`) must be on the app's `testAccountBaseEmails` list — explicit owner consent.
- Only `+primitivetest<suffix>` derivatives are eligible. The bare base is never a test account.
- First verify provisions the derived user through the standard signup path and returns the real `isNewUser`, so first-run/new-user flows are testable through the bypass. Signup-mode gates apply as 403s exactly like a normal signup: `INVITATION_REQUIRED` (invite-only, no invitation), `ADDED_TO_WAITLIST` (invite-only with waitlist — the address is added), `DOMAIN_NOT_ALLOWED` (domain mode); an `inviteToken` is honored and provisions with the invitation's role (member-role invitations only — see the reserved-email boundary below).
- Issued tokens are short-lived (~30 minutes) and carry a `primitiveBypass: true` claim that gets re-checked on every request, so removing the base from the whitelist revokes sessions immediately.
- `+primitivetest*` accounts can sign in as ordinary members but are reserved at admin / owner / invitation boundaries — they cannot hold those roles. Admin-only paths (e.g. `client.notifications.send()`) need a real admin sign-in.

The whitelist is `[app].testAccountBaseEmails` in `app.toml` (max 50 bases per app), applied with `primitive config push --only app`. The web-admin settings UI edits the same list.

**Invite-only apps: pre-create the member.** A fresh derived address on an invite-only app is gated at the email step (`INVITATION_REQUIRED` / `ADDED_TO_WAITLIST`) before any code is accepted. For CI, skip the invitation flow: `primitive users create <base>+primitivetest-<x>@<domain>` adds the membership directly (no invitation email), and the OTP sign-in with `"000000"` then proceeds as an existing member (the response's `isNewUser` is `false` — pre-created members can't exercise new-user flows).

**Entitlement-gated features: grant via an admin metadata write.** When features key off resource metadata (subscription tier, feature flags), an app-level owner/admin can set the test user's state directly — `primitive metadata set user <userId> <category> --data '{...}'` — because owners/admins bypass category write rules (see the [Resource Metadata guide](AGENT_GUIDE_TO_PRIMITIVE_RESOURCE_METADATA.md)). The full CI sequence: whitelist the base once, `users create` the derived address, sign in with `"000000"`, grant metadata — a headless session gets a fresh, entitled member with zero human interaction.

**Don't use this in production user flows.**

---

## Customizing Email Templates

The `email-sign-in` email is one of the transactional types Primitive sends; override it with a custom subject and branded HTML/text body — and delete the `{{#if magicLink}}` block if the app should never send a clickable link. Sign-in renders only `email-sign-in`: an override stored under `magic-link` or `otp` is listed as retired and never rendered. Email templates are a cross-cutting configuration surface — the full type list, template variables, override/revert model, and CLI commands live in the [Configuration guide](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md). Custom types are sent from a server function with `ctx.api.email.send`.

---

## Gotchas {#implementation-checklist}

- **Check method availability before rendering sign-in controls.** Use `getAuthConfig()` so disabled providers are not offered.
- **Wait for authentication before loading app data.** Initialize auth, open documents, then run queries. React to later sign-out as well as sign-in.
- **Preserve invitation tokens through redirects.** Pass the token into sign-in verification; use explicit acceptance for an already signed-in user.
- **Keep email code entry available.** A sign-in link may be opened on a device without the app.
- **Use error codes for recovery.** Show your own message and retain a generic fallback for unknown errors.

- **Do not clear the session after a transport failure.** Retry transient failures; request sign-in again when the server rejects refresh credentials.
- **Returning Apple identities need explicit invitation acceptance.** Call `invitations.accept` after sign-in when carrying a pending token.
