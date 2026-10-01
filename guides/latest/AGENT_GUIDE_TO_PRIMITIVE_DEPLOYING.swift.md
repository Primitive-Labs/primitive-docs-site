# Deploying Primitive apps to production

Deploy the web template to its hosting environment, or distribute the native app through TestFlight and the App Store.


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
