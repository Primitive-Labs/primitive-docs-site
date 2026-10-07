# Deploying Primitive apps to production

Deploy the web template to its hosting environment, or distribute the native app through TestFlight and the App Store.

{{#lang ts}}
## Web (Cloudflare Workers)

The web template deploys to Cloudflare Workers via `pnpm cf-deploy`. You need a Cloudflare account with Workers deploy access.

A deploy names **two independent things**, and neither is inferred from the other:

| Flag | Selects | Which means |
|---|---|---|
| `--deploy-env <name>` | the **deploy environment** | the Vite mode (`.env.<name>`) and the `[env.<name>]` block in `wrangler.toml` |
| `--primitive-env <name>` | the **Primitive environment** | the backend/app pair in `primitive/config.json` |

Omitting either is an error. Always write "deploy environment" or "Primitive environment" — a bare "environment" is ambiguous here, because Wrangler and Vite each call their own half by a different name.

### 1. Configure `wrangler.toml`

Set the worker name and per-environment overrides:

```toml
name = "my-app"

[env.production]
name = "my-app-prod"
```

By default the app deploys to a `*.workers.dev` URL. For a custom domain, add a route under the environment:

```toml
[[env.production.routes]]
pattern = "your-domain.com"
custom_domain = true
```

### 2. Configure `.env.production` (app behavior only)

The Vite mode selects `.env.<deploy-env>`. It carries no identity:

```bash
# OAuth redirect URI for the production origin (must match the
# server-side OAuth config; mismatches fail the callback exchange).
VITE_OAUTH_REDIRECT_URI=https://my-app-prod.your-subdomain.workers.dev/oauth/callback
```

The app ID and backend URL live in `primitive/config.json`, as a named Primitive environment — the one place they are typed. The `primitiveEnv()` Vite plugin fills `VITE_APP_ID`, `VITE_API_URL`, `VITE_WS_URL` and `VITE_APP_NAME` into the build from it.

**A deploy errors if any of those keys appear in a `.env` file it would load, or in `process.env`.** There is no override flag: remove them.

### 3. Deploy

```bash
pnpm cf-deploy --deploy-env production --primitive-env prod
```

The script prints the resolved pair — deploy environment, Primitive environment, apiUrl, appId, appName, and the config path — before building. `--check` prints that plus the exact build and wrangler commands and exits without running them:

```bash
pnpm cf-deploy --deploy-env production --primitive-env prod --check
```

Pass extra wrangler flags after `--`:

```bash
pnpm cf-deploy --deploy-env production --primitive-env prod -- --dry-run
```

Passthrough arguments that would take over what the deploy already decided — `--env`/`-e`, or `--var APP_ID:`/`--var API_ORIGIN:` — are rejected; other `--var` flags pass through.

### Adding environments

Configure deployment environments and Primitive environments independently.

**Another deploy environment:**

1. Add `[env.<name>]` (and any `[env.<name>.vars]`) to `wrangler.toml`.
2. Create `.env.<name>` with app behavior.
3. `pnpm cf-deploy --deploy-env <name> --primitive-env <backend>`.

**Another Primitive environment:** `primitive env add alpha --api-url ... --app-id ...`, then name it with `--primitive-env alpha`. Nothing in `wrangler.toml` or `.env.*` changes.

### Preview a branch against a child app

A [child app](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md#child-apps) (`primitive apps children create feature-x`) is a copy of the parent app for one branch. Upload a preview version of the worker under an alias instead of deploying, to run that branch's front end against it:

```bash
pnpm cf-deploy --deploy-env production --primitive-env feature-x --preview-alias feature-x
```

This uploads the version with the child's app ID, prints the preview URL (`https://feature-x-my-app-prod.your-subdomain.workers.dev`), and adds that origin to the child's preview origins, so its CORS and email sign-in checks accept it. The live worker is untouched; running the same command again after a change keeps the same alias URL. `--check` prints the upload and the registration without running them.

- Against a committed environment, `--preview-alias` registers nothing — add the origin to `[cors].allowedOrigins` and `[auth].emailRedirectUris` in that environment's `app.toml` and push to sign in from it there.
- If registering the preview origin fails, the upload still succeeds; the command prints the registration to run by hand.
- A deploy to a child **without** `--preview-alias` prints a warning and proceeds anyway — it replaces the live worker of that deploy environment for everyone.

### Pinning a deploy environment to a Primitive environment

Opt-in, for apps whose `.env.<mode>` keys are only correct against one backend (per-environment resource IDs and the like). Declare the pairing in that mode's file:

```dotenv
# .env.production
VITE_EXPECTED_PRIMITIVE_ENV=prod
```

Any run whose Primitive environment resolves to something else then fails at startup — `pnpm dev`, `pnpm build`, `pnpm test` (the headless harness suite included) and `pnpm cf-deploy` alike, because all of them resolve through the `primitiveEnv()` plugin. `cf-deploy` checks it before it builds or prints a plan, so a cross-wired `--check` fails too. A child app's machine-local environment pairs with the mode that declares its parent: under `VITE_EXPECTED_PRIMITIVE_ENV=prod`, a child of `prod` runs and a child of `dev` is refused.

Use `VITE_EXPECTED_PRIMITIVE_ENV` when a mode contains backend-specific settings. A mode’s `.env` file overrides the base value; an empty value disables the check for that mode. A process environment value overrides the files. Builds supplied with both `VITE_APP_ID` and `VITE_API_URL` resolve no Primitive environment; `cf-deploy` rejects those overrides.

```toml
[env.test]
name = "my-app-test"

[env.test.vars]
REFRESH_PROXY_COOKIE_MAX_AGE = "604800"
REFRESH_PROXY_COOKIE_PATH = "/proxy/"
```

The deploy reads `.env.{environment}` and the matching `[env.{environment}]` block.
{{/lang}}

{{#lang swift}}
## iOS (TestFlight and the App Store)

Simulator builds run unsigned. A team ID + Apple Developer account ($99/year) is required only for physical devices, TestFlight, and the App Store.

### 1. Signing and Team ID

The Team ID is the single setting required for device, TestFlight, and App Store builds. Set it in `project.yml`, **never in the Xcode UI** — the xcodeproj is xcodegen output and UI edits are wiped on the next `xcodegen generate`.

1. Team ID: [developer.apple.com/account](https://developer.apple.com/account) → Membership Details (10 chars, e.g. `2J4V27W63D`).
2. Edit `project.yml`:

   ```yaml
   settings:
     base:
       DEVELOPMENT_TEAM: "2J4V27W63D"
       CODE_SIGN_STYLE: Automatic
   ```

3. Regenerate the xcodeproj:

   ```bash
   bash scripts/regenerate-project.sh
   ```

   The script generates model and server API types, regenerates the Xcode project, and synchronizes package revisions. Build scripts and Fastlane run it automatically. Install `xcodegen` with `brew install xcodegen`.

   Review and commit changes to generated sources, including changes from release builds.

After that, device installs and archives both work.

### 2. Run on a physical iPhone / iPad

```bash
./run-ios.sh --device
```

Connect a paired device over USB and set `DEVELOPMENT_TEAM`. Check pairing with `xcrun devicectl list devices`. The script builds, installs, and launches the app.

### 3. Set up Fastlane

The iOS template **ships Fastlane** — a root `Gemfile`, `fastlane/Appfile`, `fastlane/Fastfile`, and `fastlane/.env.example` are already in the project, giving you one-command TestFlight / App Store builds and version bumping. Install the gem:

```bash
bundle install
```

`fastlane/Appfile` is generic: it reads the app identifier and Team ID from `project.yml` at runtime, so there's nothing to edit there — set the Team ID as `DEVELOPMENT_TEAM` in `project.yml`.

### 4. App Store Connect API Key

The `ios beta` / `ios release` lanes authenticate with an App Store Connect API key.

1. [App Store Connect → Users and Access → Integrations → API Keys](https://appstoreconnect.apple.com/access/integrations/api) → create a key with role **App Manager**.
2. Download the `.p8` (one-time download) to `fastlane/api_key.p8`. **Gitignore it** — it's a private key; leaking it lets anyone upload builds as your team.
3. Note the **Key ID** and **Issuer ID**.
4. Copy the shipped template and fill in the three values:

   ```bash
   cp fastlane/.env.example fastlane/.env
   ```

   ```bash
   # fastlane/.env
   ASC_KEY_ID=ABC123XYZ
   ASC_ISSUER_ID=00000000-0000-0000-0000-000000000000
   ASC_KEY_PATH=./fastlane/api_key.p8
   ```

Gitignore `fastlane/api_key.p8` and `fastlane/.env`. If a lane runs without these, the shipped Fastfile prints the exact setup steps and stops.

### 5. The shipped lanes

You don't author the Fastfile — the template ships it, parameterized off `project.yml` so it's generic across apps. List the lanes with `bundle exec fastlane lanes`:

| Lane | What it does |
|------|--------------|
| `fastlane ios beta` | Archive, export, and upload an iOS build to TestFlight |
| `fastlane ios release` | Archive, export, and submit an iOS build to App Store review (sets `skip_metadata` / `skip_screenshots`) |
| `fastlane mac beta` | Upload a macOS build to TestFlight |
| `fastlane mac dmg` | Build a notarized DMG for direct distribution |
| `fastlane bump type:patch` | Bump the marketing + build version in `project.yml` and regenerate the xcodeproj (`major` / `minor` / `patch`) |
| `fastlane status` | Print the app version, bundle ID, Team ID, signing certificates, and whether the API key is configured |

The iOS lanes use the API key for signing and upload. The macOS beta lane requires an Xcode account for automatic signing. All build lanes use the app’s `Package.resolved` revisions.

### 6. Register the app on App Store Connect (one-time)

Done once per app, before the first upload.

1. [App Store Connect → Apps → +](https://appstoreconnect.apple.com/apps) → New App.
2. Pick **iOS**; set name, primary language, bundle ID (must match `PRODUCT_BUNDLE_IDENTIFIER` in `project.yml` — register a missing ID at [developer.apple.com/account/resources/identifiers](https://developer.apple.com/account/resources/identifiers/list)), SKU (any unique string).
3. Choose **Full Access**.

### 7. Ship a TestFlight build

```bash
bundle exec fastlane bump type:patch      # bumps version + build, regenerates xcodeproj
bundle exec fastlane ios beta             # archives, exports, uploads
```

Wait for App Store Connect to process the build, then assign testers. External testing may require Beta App Review.

### 8. Submit to the App Store

```bash
bundle exec fastlane bump type:minor
bundle exec fastlane ios release
```

The release lane uploads and submits the app for review. Complete metadata and screenshots in App Store Connect first; the lane skips their upload.

### Gotchas for CI

Both `./run-ios.sh` and `bundle exec fastlane ios beta` run in GitHub Actions on a macOS runner. Base64-encode `api_key.p8` into a secret and decode it before the lane runs.

The API key alone suffices only on a machine that keeps its keychain. On a fresh runner the lane creates a **new** Apple Distribution certificate each run, and Apple caps them per team, so repeatable CI needs the team's existing signing certificate and private key installed on the runner (export the identity to a `.p12`, keep it as a secret, import it into a temporary keychain before the lane) rather than one minted per run.
{{/lang}}
