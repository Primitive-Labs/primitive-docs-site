# App Hosting

Primitive can serve a static web front end — any build of HTML, CSS and JavaScript — directly at its own origin, `https://<slug>.primitive.app`, with no cloud account of your own.

{{#lang ts}}

## Deploying

```bash
primitive hosting deploy dist --slug my-app   # first deploy: claims the slug
primitive hosting deploy                      # later deploys: finds the app by id
```

`deploy` hashes the build, uploads only the files that changed since the last deployment, activates the new one, and waits (at most 120 s) until the origin actually serves it before returning — it reports `uploaded`/`reused` counts. `--routing spa` (the default) serves `index.html` for any path with no matching file; `--routing static` answers 404 with the deployment's own `404.html` when it has one. `--app <app-id>` targets another app than the environment's current one, on every `hosting` command.

## From the web template

```bash
pnpm build:hosted                             # codegen, type check, build into dist/
primitive hosting deploy dist --slug my-app
```

The template's hosted build bakes in no app id, API URL or WebSocket URL: it reads them from the origin at start-up and signs in through the origin's proxy, so the runtime code below is already in place. It reads `.env`, `.env.local`, `.env.hosted` and `.env.hosted.local` — not `.env.production`. `VITE_APP_NAME` sets the display name.

## Rollback and state

```bash
primitive hosting deployments           # every deployment, newest first, active one marked
primitive hosting activate <deploy-id>  # serve an earlier one again, with its own routing; nothing uploads
primitive hosting disable               # origin answers 404; slug and deployments are kept
primitive hosting enable                # serve the last active deployment again
primitive hosting rename <slug>         # move to a new slug; the old origin redirects (308) for 90 days
```

`activate`, `disable`, `enable` and `rename` prompt for confirmation unless `-y`, and each one waits for the origin before returning, the same way `deploy` does — activating while the app is disabled or suspended fails at once naming the state, and an origin that never starts serving fails with `activation_timeout` after 120 s.

## Trust: what the first claim writes

The first deploy that claims a slug, and every `rename`, adds the hosted origin to `app.toml` and pushes it (`config push --only app`):

- `[cors].allowedOrigins` gains the origin;
- `[auth].emailRedirectUris` gains the origin;
- `[auth.passkeys."<slug>.primitive.app"]` is added;
- each Google client the app already has gains the matching callback path on the new origin, under its `redirectUris`.

`deploy --no-trust` / `rename --no-trust` skip this and print the entries to add by hand instead; `primitive hosting trust` re-runs the step on demand — it is idempotent, and how you retry one that failed.

**Watch out when adding Google sign-in to a hosted origin:** the trust step only edits `app.toml` — it never registers anything with Google. Add `https://<slug>.primitive.app/oauth/callback` to the OAuth client's Authorized redirect URIs in the Google Cloud Console yourself, or the callback fails even though the server-side allow-list already names it.

## Reading the app's identity at runtime

A hosted build carries no app id, API URL or WebSocket URL at build time — fetch them from the platform at startup, and route token refresh and passkey sign-in through the proxy it runs there:

{{ example: auth/initialize-hosted-client }}

`passkey: true` sends the passkey sign-in and registration *finish* calls through the same proxy that already carries magic-link and code sign-in, so every method sets the same first-party cookie; passkey *start* calls, and listing, renaming and deleting passkeys, go straight to the API regardless.

## Gotchas

- **Size and shape limits are enforced before anything uploads:** at most 99,999 files, 25 MiB per file, 768 UTF-8 bytes per path, and an 88 MiB manifest (the full file listing) in total. A build over any of these is refused outright (`too_many_files`, `path_too_long`, `manifest_too_large`) — split or shrink it rather than retrying.
- **`__primitive/` is reserved.** A deploy containing a path under it is refused; that prefix is where `config.json` and the sign-in proxy above answer.
- **`_headers` and `_redirects` at the build root are refused.** Hosting reads neither.
- **Only an admin of the app may publish.** The CLI derives a short-lived `write`-only token from the caller's own login for every hosting command (pinned to the app for a pinned login); a CI token is sent as it is.
- **Child apps cannot claim a slug.** A child app's front end keeps the preview alias it already uses.
- **A template hosted build starts only on a hosted origin.** Served anywhere else (`pnpm preview`, a static server, a self-hosted deployment) it shows a "hosted build" message and stops before calling the API. Self-host with `pnpm build`; `pnpm cf-deploy --deploy-env hosted` is refused.
- **The template's `public/_headers` is left out of a hosted build** (the build prints `left out _headers`). Headers set there, such as the `apple-app-site-association` content type, don't apply on the hosted origin.
{{/lang}}
