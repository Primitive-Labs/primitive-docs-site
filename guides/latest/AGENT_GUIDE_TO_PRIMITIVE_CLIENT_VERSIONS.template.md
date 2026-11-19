# Agent Guide to Primitive Client Versions

A `[clients.<platform>]` policy in `app.toml` declares, per platform, an app's `latest`, `minRecommended` and `minRequired` versions and an `updateUrl`. A built client reports its own version and reads the policy through `client.appVersion`, which computes a status; the Vue and Swift templates use the status to show a dismissible banner or a blocking gate, and the server can opt in to refusing outdated builds on its own.

## Declaring a policy

```toml
[clients.ios]
latest = "2.4.0"
minRecommended = "2.2"
minRequired = "2.0"
updateUrl = "itms-apps://apps.apple.com/app/id1234567890"

[clients.web]
latest = "10.1"
minRequired = "9.0"
```

Push it with `primitive config push --only app`. A platform name is a lowercase letter followed by up to 31 lowercase letters, digits or hyphens. A version is one to three dot-separated numbers of up to six digits, compared numerically (`1.10` > `1.9`; `1.0` == `1`). `minRequired` may not be above `minRecommended`, neither may be above `latest`, and `updateUrl` must use `https:`, `itms-apps:`, `itms-beta:`, `macappstore:` or `market:`. Every key is optional, but a table needs at least one; quote every version or TOML reads `1.10` as the number `1.1`.

Anyone, signed in or not, can read the policy from `GET /app/{appId}/api/settings/client-versions` {{ status: op:GET /app/{appId}/api/settings/client-versions }}, which answers `{ clients: { <platform>: { latest?, minRecommended?, minRequired?, updateUrl?, enforce? } } }`.

{{#lang ts}}
```ts
const { clients } = await client.getClientVersions();
```
{{/lang}}

**Gotchas when pushing:**
- A push that lowers or removes a stored `minRequired`/`minRecommended` (including by removing its table) asks for confirmation; pass `--yes` with no terminal, or it fails and applies nothing.
- If the stored policy changed since the push compared against it, the push fails with `CLIENT_VERSIONS_CHANGED` and writes nothing — push again.
- `enforce` (`true`/`false`, unquoted) is refused in a table with no `minRequired`.

## Reporting your version

Pass `appVersion` and, where it isn't implied, `platform` when constructing the client:

{{ example: client-versions/report-status }}

{{#lang ts}}
`appVersion` must be given explicitly; `platform` defaults to `"web"` in a browser, otherwise unset. An illegal value for either throws `INVALID_ARGUMENT` at construction, naming the value.
{{/lang}}
{{#lang swift}}
With no options, `appVersion` reads the bundle's `CFBundleShortVersionString` and `platform` reads the OS (`ios` or `macos`); the options replace either. An illegal value — from an option or the bundle — is a precondition failure at construction (the initializer does not throw), naming the value.
{{/lang}}

With both known, the client sends `X-Primitive-App-Version` and `X-Primitive-App-Platform` on every app-API call and socket connect, and stamps the version on analytics events as `app_version`.

{{#lang ts}}
`setAppVersionOverride` still wins over the stamped version — see the analytics guide's Plan and App Version Overrides.
{{/lang}}
{{#lang swift}}
`setAppVersionOverrideAsync` still wins over the stamped version — see the analytics guide's Plan and App Version Overrides.
{{/lang}}

## Reading the status

`client.appVersion` exposes `version`, `platform`, `getStatus()`, `refresh()` and `onStatusChange(...)`{{#lang swift}}, plus `getPolicy()` for this platform's raw entry{{/lang}}. The status is `updateRequired` below `minRequired`, else `updateRecommended` below `minRecommended`, else `updateAvailable` below `latest`, else `current`; `unknown` with no version or platform, no policy ever fetched, or no entry for the platform.

{{#lang ts}}
`onStatusChange` delivers the full `AppVersionState` (`status`, `version`, `platform`, `policy`, `refused`) on every change; `getStatus()`/`refresh()` resolve to the same shape. Neither rejects.
{{/lang}}
{{#lang swift}}
`onStatusChange` delivers only the `AppVersionStatus`; read `getPolicy()` separately for the platform's entry (its `updateUrl` included). Neither `getStatus()` nor `refresh()` throws.
{{/lang}}

Nothing is fetched until the app asks (any of the three methods above) or the server refuses the build. From then on: a random wait of up to 30 s after each socket reconnect, on each return to the foreground, and at most hourly. The last fetched policy is kept for offline use and the next launch; a failed fetch never changes the status.

A server refusal (426 over HTTP, 4426 on the socket, both carrying `CLIENT_UPDATE_REQUIRED`) sets `updateRequired` at once, asked or not, and the client holds off reconnecting on its own until a later fetch moves the status off `updateRequired`. A status computed from the policy alone never closes or holds a connection — only a refusal does.

## The template banner and gate

Both templates ship a dismissible banner (`updateAvailable`, `updateRecommended`) and a blocking gate (`updateRequired`), each independent and removable on its own.

{{#lang ts}}
**Vue:** `AppUpdateBanner.vue` and `AppUpdateGate.vue`, mounted once each in `src/App.vue`. The build stamps `package.json`'s `version` into the client as `appVersion` and writes `version.json` at the build output's root; the banner also shows when the page's build id no longer matches the one its origin now serves. A dismissal is remembered (per version/build id) in local storage. To remove either, delete its file and its one import line and one tag in `src/App.vue`.
{{/lang}}
{{#lang swift}}
**SwiftUI:** `AppUpdateModel` is the only thing that talks to `client.appVersion`; `AppState` attaches it at launch (counts as the app's first ask), and `AppUpdateBanner`/`AppUpdateGate` only read its published state. To remove either, delete its file and its one modifier line in `ContentView.swift` (`.safeAreaInset` for the banner, `.appUpdateGate` for the gate). In a debug build, set `PRIMITIVE_APP_VERSION` in the environment to preview any status without shipping an old build.
{{/lang}}

## Enforcing a minimum version

Add `enforce = true` beside `minRequired` to have the server refuse that platform's builds below it:

```toml
[clients.ios]
minRequired = "2.0"
enforce = true
```

A call or connect reporting `ios` and a version below `2.0` is refused — 426 `CLIENT_UPDATE_REQUIRED` over HTTP, close code 4426 on the socket. The policy route, `settings/sign-in` (and its `oauth-config` alias), `auth/refresh` and `auth/logout` still answer. A caller reporting no version, another platform, or a version it cannot parse is admitted. Refusals are logged and counted per app per day in the `client.update_refused` usage meter (`resourceId` is the platform).

**This is not a security boundary.** It only catches a caller that truthfully reports its own outdated platform and version; nothing stops a caller from sending no version or a version that passes. Don't rely on `enforce` to keep a capability away from an old build — gate the capability itself.

## Gotchas

- Turning `enforce` on refuses every older build at once — ship the build that meets `minRequired` first.
- A lowered minimum can take up to a minute to reach every call; a build at the new minimum may still be refused until then.
{{#lang ts}}
- The construction check (illegal `appVersion`/`platform`) happens once, at build time — there's nothing to catch later; fix the value and rebuild.
{{/lang}}
{{#lang swift}}
- The construction check (illegal `appVersion`/`platform`, including an illegal bundle version) happens once, at build time — there's nothing to catch later; fix the value, or the bundle's `CFBundleShortVersionString`, and rebuild.
{{/lang}}
