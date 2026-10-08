# Agent Guide to Primitive Multi-Client Apps

Web and native clients share one Primitive app: an app ID, environments, server configuration, and model schema. Each client builds and deploys independently.

## Scaffolding it

```bash
primitive init my-app --platform web,ios   # one app, both clients, one repository
primitive init my-app --platform web       # one client: the flat standalone layout
```

The command creates one app and repository. Selecting both platforms creates `web/` and `ios/` directories; selecting one creates the client at the repository root. Pass `--skip-install` to defer dependency installation.

## The layout

```
<repo>/
  primitive/config.json           # the environments: apiUrl + appId per environment
  primitive/<env>/                # exported server config (functions, database types, app settings)
  models/models.toml              # the model schema — one copy
  AGENTS.md                       # app-wide: this layout
  web/                            # Vue client: its own package.json, .env, AGENTS.md
  ios/                            # SwiftUI client: its own Package.swift, primitive.json, AGENTS.md
```

### Gotchas

- **One git repository, at the root.** No `.git` inside a client.
- **One `primitive/`, at the root.** The project config (`config.json`) and each environment's exported server config live there, shared by every client. `.primitive/` holds machine-local CLI state (credentials, the selected environment) and is gitignored. A client owns neither.
- **No root `package.json` and no workspace file.** The clients are independent projects sitting side by side, not a monorepo — each installs, builds, tests and deploys on its own.
- **No symlinks.** Every shared file is read in place by path.

## Adding a client to an app that already exists

Run init from inside the repository, pointing at the new client's directory:

```bash
cd my-app
primitive init ios --platform ios     # or just `primitive init --platform ios`
```

Init uses the app and selected environment from the nearest `primitive/config.json`. It shares the existing model schema and leaves new files uncommitted for review.

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
| `primitive/config.json` (environments, app ID) | Generated code (models, function invokers) |
| `primitive/<env>/` server config | Runtime connection settings (`.env`, `primitive.json`) |
| `models/models.toml` | Dependencies, build and test config |
| App secrets and config vars | Deploy configuration |
| The app-wide `AGENTS.md` | The client's own `AGENTS.md` |

Server function and database type **definitions** are per app — they live in the sync tree, and so do the database-type declarations functions compile against (`config push` writes them into the tree's `functions/` directory). The **client code** generated from the definitions is per client: run `primitive functions codegen --lang ts -o …` in the web client and `primitive functions codegen --lang swift -o …` in the native one, from each client's own directory, to write each client's typed invokers into its own source tree.

## The model schema

Keep one `models/models.toml` so clients use the same model and field names.

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

Run CLI commands from the root or either client directory. The CLI finds the nearest `primitive/config.json` and uses its selected environment.

## Environments

`primitive env use <name>` selects the environment for the whole repository — one backend/app pair that every CLI command in every client directory then targets. It is an app-wide selection, not a per-client one; there is no such thing as the web client being on `dev` while the native client is on `prod` for CLI purposes.

```bash
primitive env list          # every environment in the project config
primitive env use staging   # this machine's selection (gitignored local state)
primitive -e prod config diff
```

Both clients resolve runtime connection settings from the shared project configuration. Keep any client-specific overrides consistent with that environment.

## AGENTS.md ownership

The root `AGENTS.md` describes the app: the layout, what is shared, and where the schema is. Each client's `AGENTS.md` describes that client's stack and conventions and ships with its template — it is refreshed by template upgrades, so app-wide facts do not belong in it. When working inside a client directory, read both: the client's file for its stack, the root file for what it shares.
