# Deploying Primitive apps to production

Deploy the web template to its hosting environment, or distribute the native app through TestFlight and the App Store.

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

