# Deploying Primitive apps to production

This guide is the reference for shipping a Primitive app to production through its native distribution channel.

{{#lang ts}}
## Web (Cloudflare Workers)

The web template deploys to Cloudflare Workers via `pnpm cf-deploy <environment>`. You need a Cloudflare account with Workers deploy access.

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

### 2. Configure `.env.production`

`cf-deploy` reads `.env.{environment}`. For production, `.env.production`:

```bash
# Primitive App ID — same as dev, or a separate production app.
VITE_APP_ID=your_production_app_id

# OAuth redirect URI for the production origin (must match the
# server-side OAuth config; mismatches fail the callback exchange).
VITE_OAUTH_REDIRECT_URI=https://my-app-prod.your-subdomain.workers.dev/oauth/callback
```

### 3. Deploy

```bash
pnpm cf-deploy production
```

The script reads `.env.production`, builds, and deploys to the `[env.production]` worker. Pass extra wrangler flags after `--`:

```bash
pnpm cf-deploy production -- --dry-run
```

### Adding environments

`cf-deploy <name>` generalizes to any environment:

1. Add `[env.<name>]` (and any `[env.<name>.vars]`) to `wrangler.toml`.
2. Create `.env.<name>`.
3. `pnpm cf-deploy <name>`.

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
   xcodegen generate
   ```

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

Each build lane reads the Team ID from `project.yml` (it errors with the `primitive apple set-team-id` fix if unset) and loads the API key from `fastlane/.env`. The lanes export with `signingStyle: automatic` and `-allowProvisioningUpdates`, so Xcode requests the provisioning profiles for you.

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
