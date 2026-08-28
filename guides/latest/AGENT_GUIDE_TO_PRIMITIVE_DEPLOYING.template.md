# Deploying Primitive apps to production

This guide is the reference for shipping a Primitive app to production through its native distribution channel.

{{#lang ts}}
## Web (Cloudflare Workers)

The web template deploys to Cloudflare Workers via `pnpm cf-deploy`. You need a Cloudflare account with Workers deploy access.

A deploy names **two independent things**, and neither is inferred from the other:

| Flag | Selects | Which means |
|---|---|---|
| `--deploy-env <name>` | the **deploy environment** | the Vite mode (`.env.<name>`) and the `[env.<name>]` block in `wrangler.toml` |
| `--primitive-env <name>` | the **Primitive environment** | the backend/app pair in `.primitive/config.json` |

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

The app ID and backend URL live in `.primitive/config.json`, as a named Primitive environment — the one place they are typed. The `primitiveEnv()` Vite plugin fills `VITE_APP_ID`, `VITE_API_URL`, `VITE_WS_URL` and `VITE_APP_NAME` into the build from it.

**A deploy errors if any of those keys appear in a `.env` file it would load, or in `process.env`.** There is no override flag: remove them. For an app scaffolded by an older CLI, that deletion is the entire migration.

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

The two axes grow separately.

**Another deploy environment:**

1. Add `[env.<name>]` (and any `[env.<name>.vars]`) to `wrangler.toml`.
2. Create `.env.<name>` with app behavior.
3. `pnpm cf-deploy --deploy-env <name> --primitive-env <backend>`.

**Another Primitive environment:** `primitive env add alpha --api-url ... --app-id ...`, then name it with `--primitive-env alpha`. Nothing in `wrangler.toml` or `.env.*` changes.

### Pinning a deploy environment to a Primitive environment

Opt-in, for apps whose `.env.<mode>` keys are only correct against one backend (per-environment resource IDs and the like). Declare the pairing in that mode's file:

```dotenv
# .env.production
VITE_EXPECTED_PRIMITIVE_ENV=prod
```

Any run whose Primitive environment resolves to something else then fails at startup — `pnpm dev`, `pnpm build`, `pnpm test` (the headless harness suite included) and `pnpm cf-deploy` alike, because all of them resolve through the `primitiveEnv()` plugin. `cf-deploy` checks it before it builds or prints a plan, so a cross-wired `--check` fails too.

Rules: absent (the default) keeps the axes fully independent; a value in the base `.env` is the default for every mode and `.env.<mode>` overrides it; an empty value cancels the check for that mode; a non-empty `VITE_EXPECTED_PRIMITIVE_ENV` in the process environment wins over the files, which is how a deliberate cross-wired run states itself (`VITE_EXPECTED_PRIMITIVE_ENV=dev PRIMITIVE_ENV=dev pnpm test --mode alpha`). The pure-env CI hatch — a build supplying both `VITE_APP_ID` and `VITE_API_URL` — resolves nothing and so skips the check; overriding only one of them does not, because the other half still comes from the resolved environment. `cf-deploy` reads `.env*` from the project root, so under a custom Vite `envDir` its `--check` will not see the declaration (a real deploy still fails inside the build).

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

   That script is the one entry point for regeneration: it emits `Models/Generated/*.swift` from `models.toml` (gitignored build products, so a fresh clone has none — and `xcodegen` can only list files that already exist), runs `xcodegen generate`, and then re-copies the app's `Package.resolved` into the project container xcodegen just rewrote. `./run-ios.sh`, `./archive.sh` and the fastlane lanes all call it, so this step is only needed when you want the regeneration on its own. It requires xcodegen (`brew install xcodegen`) and fails with that instruction if it is missing.

After that, device installs and archives both work.

### 2. Run on a physical iPhone / iPad

```bash
./run-ios.sh --device
```

Requires a paired device over USB — verify with `xcrun devicectl list devices` (shows `paired`). The script auto-picks the first paired device, builds with `-allowProvisioningUpdates` (Xcode requests provisioning profiles for you), installs via `devicectl`, and launches with `--console` so `print` / NSLog stream to the terminal. Needs `DEVELOPMENT_TEAM` set (step 1).

### 3. Set up Fastlane

The iOS template **ships Fastlane** — a root `Gemfile`, `fastlane/Appfile`, `fastlane/Fastfile`, and `fastlane/.env.example` are already in the project, giving you one-command TestFlight / App Store builds and version bumping. Install the gem:

```bash
bundle install
```

`fastlane/Appfile` is generic: it reads the app identifier and Team ID from `project.yml` at runtime, so there's nothing to edit there — set the Team ID with `primitive apple set-team-id <id>` (it writes `DEVELOPMENT_TEAM` in `project.yml`).

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

Each build lane reads the Team ID from `project.yml` (it errors with the `primitive apple set-team-id` fix if unset) and loads the API key from `fastlane/.env`. The lanes export with `signingStyle: automatic` and `-allowProvisioningUpdates`, so Xcode requests the provisioning profiles for you. Every lane also runs `scripts/sync-xcode-pins.sh` first, copying the app's `Package.resolved` over Xcode's own copy of that pin, so an archive can't be built against a package revision `swift package update` has already moved past.

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

Internal testers (added in the App Store Connect UI under TestFlight) get builds immediately — no review. External testers / groups need a one-time Beta App Review per major version. The first upload takes 10–20 min between Fastlane finishing and the build appearing in TestFlight (Apple processes the binary + export compliance); subsequent uploads ~5 min.

### 8. Submit to the App Store

```bash
bundle exec fastlane bump type:minor
bundle exec fastlane ios release
```

`upload_to_app_store` uploads + submits for review. The lane sets `skip_metadata: true` / `skip_screenshots: true` — fill in description, screenshots, keywords, age rating, and privacy answers in App Store Connect before the build can be reviewed. Once metadata is complete and the build is processed, review typically takes 24–72 hours.

### CI

Both `./run-ios.sh` and `bundle exec fastlane ios beta` run in GitHub Actions on a macOS runner. Base64-encode `api_key.p8` into a secret and decode it before the lane runs.
{{/lang}}
