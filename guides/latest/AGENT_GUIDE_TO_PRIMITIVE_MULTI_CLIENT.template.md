# Agent Guide to Primitive Multi-Client Apps

Guidelines for AI agents working in a Primitive app that has more than one client — a web client and a native client against the same backend. Such a product is ONE Primitive app: one app ID, one set of environments, one server-config export, one model schema. `primitive init` produces that repository, and this guide describes what it produces and which parts are shared.

## The layout

```
<repo>/
  .primitive/config.json          # the environments: apiUrl + appId per environment
  .primitive/sync/<env>/<appId>/  # exported server config (workflows, database types, app settings)
  models/models.toml              # the model schema — one copy
  AGENTS.md                       # app-wide: this layout
  web/                            # Vue client: its own package.json, .env, AGENTS.md
  ios/                            # SwiftUI client: its own Package.swift, primitive.json, AGENTS.md
```

Properties that hold, and that a change must not break:

- **One git repository, at the root.** No `.git` inside a client.
- **One `.primitive/`, at the root.** A client never has its own project config, credentials, or sync tree.
- **No root `package.json` and no workspace file.** The clients are independent projects sitting side by side, not a monorepo — each installs, builds, tests and deploys on its own.
- **No symlinks.** Every shared file is read in place by path.

## Scaffolding it

```bash
primitive init my-app --platform web,ios   # one app, both clients, one repository
primitive init my-app --platform web       # one client: the flat standalone layout
```

`--platform` takes one platform or a comma-separated list; the interactive prompt is a multi-select. A multi-platform run creates the app once, downloads and validates every template before creating anything, puts each client in a platform-named directory, and makes one initial commit at the root.

## Adding a client to an app that already exists

Run init from inside the repository, pointing at the new client's directory:

```bash
cd my-app
primitive init ios --platform ios     # or just `primitive init --platform ios`
```

Init finds the nearest ancestor `.primitive/config.json`, and adds the client to THAT app: the app ID and backend URL come from the selected environment, the client is wired to the repo's `models/models.toml`, and the run writes no nested `.primitive/`, no nested `.git/` and no commit — the new files are left for you to review with `git status` and commit in the outer repository. It never creates a second app.

Adding a client to a repository whose single client sits at the root (the flat layout above) moves the schema to `<repo>/models/models.toml` and rewires the existing client to it, with your confirmation. Nothing else about the existing client moves: its `package.json`, its sources and its build config stay exactly where they are.

Non-interactive (CI, scripted setup) — `.primitive-init.toml`:

```toml
action = "add-client"     # the app comes from the project config; name none here
platform = "ios"
promote_schema = true     # consent to move a flat repo's schema to the root
```

## Per app vs per client

| Per app — at the repo root | Per client — in its directory |
|---|---|
| `.primitive/config.json` (environments, app ID) | Generated code (models, workflow invokers, database types) |
| `.primitive/sync/<env>/<appId>/` server config | Runtime connection settings (`.env`, `primitive.json`) |
| `models/models.toml` | Dependencies, build and test config |
| App secrets and config vars | Deploy configuration |
| The app-wide `AGENTS.md` | The client's own `AGENTS.md` |

Workflow and database **definitions** are per app — they live in the sync tree. Their **generated code** is per client: run `primitive workflows codegen --lang ts -o …` in the web client and `--lang swift -o …` in the native one, from each client's own directory.

## The model schema

`models/models.toml` is the app's one schema and is never copied. Its TOML keys are the wire field names, so a second copy that drifts orphans the other one's records — a client reading a stale copy writes fields nothing else can read.

Each client points at it by path:

- **Web** — the codegen input (`js-bao-codegen-v2 -i ../models/models.toml -o src/models`). The generated barrel imports the same file, so codegen and the runtime schema load can never disagree.
- **Swift** — `bao-codegen.json` at the root of the target's source directory:

  ```json
  { "input": "../../../models/models.toml" }
  ```

  The path resolves against the target directory. When the file is present it replaces the plugin's scan of the target's own sources, and the Xcode pre-build script reads the same key.

Adding a client adds the models its template's own code reads, if the schema does not declare them yet, and reports them; models already in the schema keep their definitions. A client that cannot be pointed at the shared schema stops the run instead of keeping a copy of its own.

To add or change a model: edit `models/models.toml`, then run each client's codegen (`pnpm codegen` in the web client, `bash scripts/codegen.sh` in the Swift one). Never edit generated model files.

## Running CLI commands

Every `primitive` command walks up from the working directory to the nearest `.primitive/config.json`, the way git finds `.git`. So a command run in `web/`, in `ios/`, or at the root resolves the same project, the same environment and the same sync directory — no `--dir` and no path flag. A path flag is how the tree and the environment drift apart; the walk-up is why one is never needed.

## Environments

`primitive env use <name>` selects the environment for the whole repository — one backend/app pair that every CLI command in every client directory then targets. It is an app-wide selection, not a per-client one; there is no such thing as the web client being on `dev` while the native client is on `prod` for CLI purposes.

```bash
primitive env list          # every environment in the project config
primitive env use staging   # this machine's selection (gitignored local state)
primitive -e prod config diff
```

Each client also carries its own runtime connection settings — the web client's Vite plugin and the Swift client's `primitive.json` both resolve from that same project config, and both find it by the same walk-up. Keep any client-specific overrides naming the same backend/app pairs as the environments; two answers to "which backend" is the problem this layout removes.

## AGENTS.md ownership

The root `AGENTS.md` describes the app: the layout, what is shared, and where the schema is. Each client's `AGENTS.md` describes that client's stack and conventions and ships with its template — it is refreshed by template upgrades, so app-wide facts do not belong in it. When working inside a client directory, read both: the client's file for its stack, the root file for what it shares.
