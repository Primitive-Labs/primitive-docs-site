# Prompt Feature Guide for Coding Agents

How to author, test, and execute LLM prompts on Primitive using the `primitive` CLI, TOML files, and the client SDK.

## Execute a managed prompt

```swift
  let result = try await client.prompts.execute(
    promptKey: "my-summarizer",
    options: ExecutePromptOptions(
      variables: ["text": .string(documentText), "style": "concise"],
      modelOverride: "gpt-4o" // optional; defaults to the active config's model
    )
  )
```

## Quick Mental Model

A **prompt** is a named template (e.g. `summarizer`) plus 1+ **configs**. Each config is a `(provider, model, systemPrompt, userPromptTemplate, ...)` tuple. One config is the **active** config — that's what gets used when you don't pass `--config` / `configId`. Test cases are attached to the prompt and verify outputs (regex / contains / JSON subset / LLM evaluator).

Templates use `{{ }}` interpolation. Inputs are passed as `variables: { foo }` at execution time and read as `{{ input.foo }}`.

---

## Status Lifecycle (matters less than you'd think)

| Status     | Executable from workflow | Executable from client SDK / CLI | Executable from server REST |
| ---------- | ------------------------ | -------------------------------- | --------------------------- |
| `draft`    | Yes                      | Yes                              | Yes                         |
| `active`   | Yes                      | Yes                              | Yes                         |
| `archived` | No                       | Yes (no status check)            | Yes (no status check)       |

Default for new prompts is `draft`. **Both `draft` and `active` execute from workflows** (verified in `src/workflows/steps/prompt-step.ts:66`); only the workflow path enforces the status gate. The user REST handler (`src/app-api/controllers/prompts-controller.ts`) and the admin/CLI execute endpoint do NOT filter by `status`, so `archived` prompts still execute via the SDK and CLI as long as the prompt and an active-config exist. Archive a prompt only to hide it from listings — it does not block direct execution outside workflows.

The config `status` field is separate. A config defaults to `status = "active"`. The workflow step checks this and refuses to execute archived configs ("not executable"); the SDK/REST/CLI paths do not check config status.

> If you see `HTTP 404` calling a prompt via the SDK: the prompt key is wrong (no prompt with that key in the app). Archived prompts return 200 from the user REST endpoint, not 404. If the prompt has no `activeConfigId` and you didn't pass `configId`, you'll get a 400 ("No configuration found for this prompt"), not 404.

---

## Template Syntax

Implemented in `src/workflows/runner/templates.ts`. Single source of truth — same engine for prompts and workflow steps.

### Variable access

```
{{ input.foo }}              # input.foo
{{ input.user.name }}        # nested
{{ input.items[0] }}         # array index
{{ input.items[0].name }}    # mixed
```

### Template context

```typescript
{
  input: Record<string, any>,    // your variables, e.g. variables: { x } → input.x
  selected: any,                 // alias for input
  steps: Record<string, any>,    // workflow step outputs (in workflow context only)
  outputs: Record<string, any>,  // workflow outputs
  meta: Record<string, any>,
  output?: any,                  // ONLY in evaluator prompts (see below)
}
```

### Missing variables fail silent

Missing paths render as **empty string** and emit a warning to logs. They do NOT throw.

```
template:  "Hello {{ input.name }}"
vars:      {}
output:    "Hello "
```

Use `||` chained fallbacks or the `default` filter to handle this:

```
{{ input.name || "Anonymous" }}
{{ input.name || input.username || "Anonymous" }}
{{ input.name | default: "Anonymous" }}
```

Note `||` only falls back when the resolved value is null/undefined/empty-string (`templates.ts:363`). A resolved variable equal to `0` or `false` is kept and does NOT fall through. The one exception: a literal numeric `0` written directly in the template (e.g. `{{ 0 || "x" }}`) falls through to the next variant because the literal-number branch checks `if (asNumber)` (`templates.ts:347`).

### Filters (pipe syntax)

```
{{ input.data | json }}                   # JSON.stringify with 2-space indent
{{ input.name | upper }}                  # uppercase (alias: uppercase)
{{ input.name | lower }}                  # lowercase (alias: lowercase)
{{ input.text | trim }}
{{ input.items | length }}                # array/string len, object key count (alias: size)
{{ input.items | first }}
{{ input.items | last }}
{{ input.obj | keys }}
{{ input.obj | values }}
{{ input.val | string }}
{{ input.val | number }}
{{ input.items | join: ", " }}            # default sep is ","
{{ input.name | default: "Anonymous" }}

# String
{{ input.text | split: "," }}
{{ input.text | replace: "old", "new" }}
{{ input.text | truncate: "100" }}        # appends "..."
{{ input.text | startsWith: "foo" }}
{{ input.text | endsWith: "bar" }}
{{ input.text | contains: "baz" }}

# Number
{{ input.n | round }} | floor | ceil | abs
{{ input.n | toFixed: "2" }}

# Date
{{ "" | now }}                            # current ISO timestamp
{{ input.ts | toISOString }}

# Array
{{ input.items | pluck: "name" }}         # [{name:"a"},{name:"b"}] → ["a","b"]
{{ input.items | where: "type", "user" }}
{{ input.items | sort: "name" }}          # or no arg for primitives
{{ input.items | reverse }}
{{ input.items | flatten }}
{{ input.items | uniq }}
{{ input.items | compact }}               # remove null/empty/false
{{ input.items | slice: "0", "5" }}
{{ input.items | concat: '["x","y"]' }}   # concat with JSON-encoded array

# Validation — THROWS on mismatch (non-retryable)
{{ input.items | expect: "array" }}       # array | object | string | number | boolean
```

Filter arguments are quoted: `| filter: "arg1", "arg2"`. Unquoted bare words also work (`| join: ,`) but quoting is safer.

Unknown filter names log a warning and pass the value through unchanged.

### Raw value vs string interpolation

If the entire template is exactly one expression, the raw value is preserved (arrays/objects not stringified). Otherwise everything becomes a string.

```
template:  "{{ input.items }}"
vars:      { items: [1,2,3] }
result:    [1,2,3]   # actual array

template:  "Items: {{ input.items }}"
vars:      { items: [1,2,3] }
result:    "Items: 1,2,3"   # string
```

Use `| json` when you need to embed objects in larger strings:

```
Data: {{ input.config | json }}
```

### Don't do this

```
# WRONG — assumes missing var throws. It doesn't.
"Hello {{ input.name }}!"   →  "Hello !"  (silent)

# WRONG — using {{}} inside JSON without escaping breaks parsing.
"Reply with {\"name\": \"{{ input.name }}\"}"
# If input.name is `Bob"; DROP TABLE users; --`, you get malformed JSON.
# Prefer outputSchema with structured output instead, or | json the whole object.

# WRONG — base64 attachment data in template context bloats prompts.
# Attachments under variables.attachments[] are auto-stripped from templates
# and sent as file parts. Don't reference them in {{ }}.
```

---

## TOML File Format

Used by `primitive sync push/pull` and `primitive prompts create --from-file`.

### Basic structure

```toml
[prompt]
key = "my-prompt"                # required, unique per app, kebab-case
displayName = "My Prompt"        # required
description = "What it does"     # optional
status = "draft"                 # optional: draft (default) | active | archived
inputSchema = '''{"type":"object","properties":{"text":{"type":"string"}},"required":["text"]}'''

[[configs]]
name = "default"                 # required, unique per prompt
description = "..."              # optional
provider = "gemini"              # required: gemini | openrouter
model = "models/gemini-3-flash-preview"   # required
userPromptTemplate = "Summarize: {{ input.text }}"   # required
systemPrompt = "You are concise."        # optional
temperature = 0.3                # optional, number or string ("0.3"); stored as string
maxTokens = 1000                 # optional integer
outputFormat = "text"            # optional: text (default) | json
```

### Field reference (verified against `src/models/app-prompt.js` and `app-prompt-config.js`)

**`[prompt]`:**

| Key            | Required | Notes                                                                                |
| -------------- | -------- | ------------------------------------------------------------------------------------ |
| `key`          | Yes      | Unique per app                                                                       |
| `displayName`  | Yes      |                                                                                      |
| `description`  | No       |                                                                                      |
| `status`       | No       | `draft` (default) \| `active` \| `archived`                                          |
| `inputSchema`  | No       | JSON Schema as a string (parsed server-side)                                         |
| `outputSchema` | No       | JSON Schema as a string. **Read by `sync push` / `prompts create --from-file`, but `sync pull` does NOT write it back** |

**`[[configs]]`:**

| Key                  | Required | Notes                                              |
| -------------------- | -------- | -------------------------------------------------- |
| `name`               | Yes      | Unique per prompt                                  |
| `description`        | No       |                                                    |
| `provider`           | Yes      | `gemini` \| `openrouter` (CLI default: `openrouter`) |
| `model`              | Yes      | Provider-specific identifier                       |
| `userPromptTemplate` | Yes      |                                                    |
| `systemPrompt`       | No       |                                                    |
| `temperature`        | No       | Stored as string; numbers in TOML are accepted     |
| `maxTokens`          | No       | Integer                                            |
| `outputFormat`       | No       | `text` (default) \| `json`                         |

**Not exposed in TOML** (model fields that exist but aren't read by sync): `topP`, `outputSchema` on the config (prompt-level outputSchema is read on push only). Set these via `primitive prompts configs update` or the API client if needed.

### Multiple configs

```toml
[prompt]
key = "summarizer"
displayName = "Document Summarizer"

[[configs]]
name = "default"
provider = "gemini"
model = "models/gemini-3-flash-preview"
temperature = 0.3
userPromptTemplate = "Summarize: {{ input.text }}"

[[configs]]
name = "creative"
provider = "gemini"
model = "models/gemini-3-pro-preview"
temperature = 0.8
userPromptTemplate = "Write an engaging summary of: {{ input.text }}"

[[configs]]
name = "claude"
provider = "openrouter"
model = "anthropic/claude-3-5-sonnet"
temperature = 0.5
userPromptTemplate = "Provide a concise summary: {{ input.text }}"
```

The first `[[configs]]` becomes the active config when the prompt is created. To activate a different one later: `primitive prompts configs activate <prompt-id> <config-id>`.

### Evaluator prompts

Evaluators judge another prompt's output. They get TWO context entries:

- `{{ input.* }}` — the **original input variables** that were passed to the prompt being evaluated
- `{{ output }}` — the **output text** from that prompt (top-level, NOT under `input`)

```toml
[prompt]
key = "haiku-evaluator"
displayName = "Haiku Evaluator"

[[configs]]
name = "default"
provider = "gemini"
model = "models/gemini-3-flash-preview"
temperature = 0
systemPrompt = "You judge LLM outputs. Respond ONLY with valid JSON."
userPromptTemplate = """
Original input topic: {{ input.text }}

Output to evaluate:
{{ output }}

Respond with JSON:
{
  "passed": true,
  "reasoning": "Brief overall assessment",
  "checks": [
    {"name": "Proper Haiku Form", "passed": true, "message": "Follows 5-7-5"},
    {"name": "Relevant to Subject", "passed": true, "message": "On topic"}
  ]
}
"""
```

> **Footgun:** `{{ input.output }}` does NOT give you the output to evaluate — it would only resolve if your variables happened to have an `output` key. Use bare `{{ output }}`.

The evaluator output is parsed for `{ passed, reasoning, checks: [{name, passed, message}] }`. If JSON parsing fails, only the overall `passed` count is used.

---

## CLI Reference

Set the active app once: `primitive use <app-id>`. All commands accept `--app <app-id>` or a positional `[app-id]` to override.

Use `--json` for machine-readable output.

### Prompt CRUD

```bash
primitive prompts list [app-id] [--status draft|active|archived] [--json]
primitive prompts get <prompt-id> [--json]
primitive prompts update <prompt-id> [--name X] [--description X] [--status X] [--input-schema JSON] [--output-schema JSON]
primitive prompts delete <prompt-id> [-y]          # archive (soft)
primitive prompts delete <prompt-id> --hard [-y]   # permanent
```

#### Create from CLI flags

```bash
primitive prompts create \
  --key my-prompt \
  --name "My Prompt" \
  --provider gemini \
  --model "models/gemini-3-flash-preview" \
  --user-template "Generate: {{ input.text }}" \
  [--system-prompt "..."] \
  [--temperature 0.7] \
  [--max-tokens 1000] \
  [--output-format text|json] \
  [--input-schema '{...}'] \
  [--output-schema '{...}']
```

Required: `--key`, `--name`, `--model`, `--user-template`. Default `--provider` is `openrouter`.

#### Create from TOML

```bash
primitive prompts create --from-file ./summarizer.toml
```

Reads `[prompt]` + the FIRST `[[configs]]` entry. Additional configs are NOT created — for multi-config use `sync push` instead.

### Execute & preview

```bash
primitive prompts execute <prompt-id> --vars '{"text":"Hello"}' [--config <config-id>] [--json]
primitive prompts preview <prompt-id> --vars '{"text":"Hello"}' [--config <config-id>] [--json]
primitive prompts schema  <prompt-id> [--json]
```

`preview` renders the template without calling the LLM — fast for verifying interpolation.
`schema` returns `{ promptId, promptKey, displayName, inputSchema, outputSchema, inputVariables, activeConfigId, activeConfigName }`. `inputVariables` is an array of `{ name, type, description, required }` derived from `inputSchema.properties` (NOT from `{{ }}` references in the template). When no `inputSchema` is set, `inputVariables` is an empty array.

### Configs

```bash
primitive prompts configs list <prompt-id>
primitive prompts configs create <prompt-id> --name X --provider X --model X --user-template X [...]
primitive prompts configs update <prompt-id> <config-id> [--name X] [--model X] [--temperature X] [--system-prompt X] [--user-template X] [--status active|archived]
primitive prompts configs activate <prompt-id> <config-id>
primitive prompts configs duplicate <prompt-id> <config-id> [--name X]
```

`configs create` requires `--name`, `--model`, `--user-template`. Default provider is `openrouter`.

### Test cases

```bash
primitive prompts tests list <prompt-id>
primitive prompts tests get  <prompt-id> <test-case-id>
primitive prompts tests delete <prompt-id> <test-case-id> [-y]

primitive prompts tests create <prompt-id> \
  --name "Basic test" \
  --vars '{"text":"hello"}' \
  [--pattern "regex"] \
  [--contains '["substr1","substr2"]'] \
  [--json-subset '{"key":"value"}'] \
  [--config <config-id>] \
  [--evaluator-prompt <prompt-id>] \
  [--evaluator-config <config-id>]

primitive prompts tests update <prompt-id> <test-case-id> \
  [--name X] [--vars X] [--pattern X] [--contains X] [--json-subset X] [--config X] \
  [--clear-pattern] [--clear-contains] [--clear-json-subset]
```

Required for `create`: `--name`, `--vars`. `--vars` MUST be valid JSON (the CLI parses it).

### Running tests

```bash
primitive prompts tests run <prompt-id> <test-case-id> [--config <config-id>] [--json]
primitive prompts tests run-all <prompt-id> [--config <config-id>] [--test-cases "id1,id2,id3"] [--json]
primitive prompts tests runs <prompt-id> [--limit 20] [--group <comparison-group>] [--json]
```

`run-all` exits with code `1` if any test fails. Useful for CI.

### Batch (parallel) test execution

Runs tests in parallel via the workflow engine — much faster for large suites.

```bash
primitive prompts tests batch start  <prompt-id> [--config <config-id>] [--test-cases "id1,id2"] [--json]
primitive prompts tests batch status <prompt-id> <batch-id> [--wait] [--json]
primitive prompts tests batch cancel <prompt-id> <batch-id> [-y]
```

`status --wait` polls every 2s until completion. Exits `1` if any test failed.

### Test case attachments (PDFs, images, etc.)

```bash
primitive prompts tests attachments list     <prompt-id> <test-case-id>
primitive prompts tests attachments upload   <prompt-id> <test-case-id> ./doc.pdf [--name custom.pdf]
primitive prompts tests attachments download <prompt-id> <test-case-id> doc.pdf [output-path]
primitive prompts tests attachments delete   <prompt-id> <test-case-id> doc.pdf [-y]
```

Upload size limit: **10 MB**. Attachments are sent to the model as file parts (`gemini`) or vision parts (`openrouter`) and are NOT visible in template context.

> Pass attachments at runtime by including `attachments: [{name, type, data}]` in `variables`. The base64 `data` is stripped from the template context automatically and forwarded as a file part.

---

## Sync (TOML version control)

```bash
primitive sync pull [app-id]
primitive sync push [app-id] [--dry-run]
```

The sync directory auto-resolves to `.primitive/sync/<env>/<appId>/`; pass `--dir <path>` to override it.

### Directory layout (verified in `cli/src/commands/sync.ts` — see the layout block in the `sync` command's help text)

```
config/
  prompts/
    summarizer.toml
    summarizer.tests/                # NOTE: dir name is `<key>.tests`
      basic.toml
      edge-case.toml
      basic/                         # attachments dir (one per test case slug)
        document.pdf
    evaluator.toml
  workflows/
    ...
  .primitive-sync.json               # auto-generated state — commit this
```

### Test case TOML schema

Test case TOMLs use a `[test]` table with **JSON-encoded strings** for structured fields (verified in `cli/tests/unit/sync-helpers.test.ts`):

```toml
[test]
name = "Basic greeting"
description = "Optional"
inputVariables = '{"name":"Bob","occupation":"teacher"}'   # JSON string
configName = "default"                                     # key-based ref to a config
evaluatorPromptKey = "output-evaluator"                    # key-based ref to evaluator prompt
evaluatorConfigName = "default"
expectedOutputPattern = "^Hello.*"                          # regex
expectedOutputContains = '["Bob","teacher"]'                # JSON array string
expectedJsonSubset = '{"status":"ok"}'                      # JSON string
```

Key-based refs (`configName`, `evaluatorPromptKey`, `evaluatorConfigName`) are portable across apps. ID-based refs (`configId`, `evaluatorPromptId`, `evaluatorConfigId`) are also accepted but tied to a specific app — prefer the key-based forms.

### What `sync pull` actually writes

`serializePrompt` (`cli/src/commands/sync.ts:215`) writes:

- `[prompt]`: `key, displayName, description, status, inputSchema`
- `[[configs]]`: `name, description, provider, model, temperature, maxTokens, outputFormat, systemPrompt, userPromptTemplate`

It does NOT write: `outputSchema` (prompt or config), `topP`. If you set these and run `pull`, they will be missing from the TOML — round-trip is lossy for those fields.

### What `sync push` does

- Updates existing prompts (matched by `key`) and reconciles configs (matched by `name`).
- Creates new prompts and additional configs that don't exist on the server.
- For new prompts, the FIRST `[[configs]]` entry becomes the active config.
- Test case TOMLs in `<key>.tests/` are pushed and matched by filename slug.
- Skips files unchanged since the last sync (use `--force` to bypass).
- Conflict detection: if a prompt was modified on the server since the last pull, push fails — pull and re-merge.

---

## Verification Types

### Pattern (regex)

```bash
--pattern "^Hello.*world$"
```

### Contains (substring AND)

```bash
--contains '["expected", "phrase", "another"]'
```

All strings must appear in output. JSON array of strings.

### JSON subset

```bash
--json-subset '{"status":"success","data":{"valid":true}}'
```

Output must be valid JSON containing all key/value pairs (deep). Extra fields in output are fine.

### LLM evaluator

```bash
--evaluator-prompt <evaluator-prompt-id> [--evaluator-config <config-id>]
```

The evaluator prompt receives `{{ input.* }}` (original input vars) and `{{ output }}` (the generated output). See [Evaluator prompts](#evaluator-prompts) above.

Multiple verification types can stack on a single test case. All must pass for the test to pass.

---

## Common Workflows

### Create from scratch

```bash
cat > my-prompt.toml << 'EOF'
[prompt]
key = "greeting-generator"
displayName = "Greeting Generator"

[[configs]]
name = "default"
provider = "gemini"
model = "models/gemini-3-flash-preview"
temperature = 0.7
userPromptTemplate = "Generate a friendly greeting for {{ input.name || 'friend' }} who works as a {{ input.occupation || 'professional' }}."
EOF

primitive prompts create --from-file my-prompt.toml
```

### Test it

```bash
primitive prompts preview <prompt-id> --vars '{"name":"Alice","occupation":"engineer"}'
primitive prompts execute <prompt-id> --vars '{"name":"Alice","occupation":"engineer"}'
```

### Add a regression test

```bash
primitive prompts tests create <prompt-id> \
  --name "Mentions name and occupation" \
  --vars '{"name":"Bob","occupation":"teacher"}' \
  --contains '["Bob","teacher"]'

primitive prompts tests run-all <prompt-id>
```

### Compare configs

```bash
primitive prompts configs create <prompt-id> \
  --name "creative" \
  --provider gemini \
  --model "models/gemini-3-pro-preview" \
  --temperature 0.9 \
  --user-template "Generate a unique greeting for {{ input.name }} ({{ input.occupation }})."

primitive prompts tests run-all <prompt-id> --config <new-config-id>
primitive prompts tests runs <prompt-id> --json     # compare runs across configs
```

### Version control

```bash
primitive sync pull
git add .primitive/sync && git commit -m "Snapshot prompts"

# edit the synced prompts/*.toml files

primitive sync push --dry-run        # preview
primitive sync push                  # apply
```

---

## Provider & Model Cheat Sheet

### Gemini

```toml novalidate
provider = "gemini"
model = "models/gemini-3.5-flash"         # fast/cheap (GA)
model = "models/gemini-3-flash-preview"   # fast/cheap
model = "models/gemini-3-pro-preview"     # higher quality
```

The `gemini` provider enforces a server-side allowlist of model names; a model not on the allowlist is rejected at execution time. `models/gemini-3.5-flash` is on the allowlist.

### OpenRouter (everything else)

```toml novalidate
provider = "openrouter"
model = "anthropic/claude-3-5-sonnet"
model = "openai/gpt-4o"
model = "google/gemini-2.0-flash-001"
```

Pick `gemini` provider for native Gemini features (file parts, structured output via `outputSchema`). Use `openrouter` for non-Google models or OpenRouter-specific routing.

`outputSchema` (structured JSON output) is **only honored by the gemini provider** in `src/services/block-executor.ts:563`. With openrouter, set `outputFormat = "json"` and instruct the model in the system prompt instead.

Both `AppPrompt` (prompt-level) and `AppPromptConfig` (config-level) have an `outputSchema` field, and which one is honored depends on the execution path:

- SDK / user REST `POST /prompts/:promptKey/execute` and admin/CLI execute endpoint → use **`prompt.outputSchema`**.
- Workflow `prompt.execute` step → uses **`config.outputSchema`**.

The TOML `[prompt].outputSchema` field is read on push and applies to the prompt-level field. There is no TOML field that writes `config.outputSchema`; set it via `primitive prompts configs update` if you need it for the workflow path.

---

## Client SDK

```swift
  let result = try await client.prompts.execute(
    promptKey: "my-prompt-key",
    options: ExecutePromptOptions(
      variables: ["text": "Hello world"], // → {{ input.text }}
      modelOverride: "anthropic/claude-3-5-sonnet", // optional; overrides config.model
      configId: "01ABC" // optional; defaults to activeConfigId
    )
  )

  let success = result.success         // Bool — generated successfully
  let output = result.output           // String — generated text
  let error = result.error             // String? — error message, if any
  let configId = result.configId       // String — which config was used
  let metrics = result.metrics         // { durationMs, inputTokens?, outputTokens?, totalTokens? }
  let rawResponse = result.rawResponse // raw provider response (don't depend on shape)
```

The first arg is the `promptKey`, NOT the `promptId`. The endpoint is `POST /prompts/:promptKey/execute`.

`variables` becomes the `input` namespace in templates. `variables: { x: 1 }` → `{{ input.x }}`. There is no top-level access to your variables (e.g. `{{ x }}` won't resolve).

### Don't do this

```swift
// WRONG — passing the prompt id instead of the prompt key
try await client.prompts.execute(promptKey: "01HXY...PROMPT_ID", options: ExecutePromptOptions(variables: [:]))
// → 404. Use the key from the TOML.

// WRONG — expecting a top-level variable
try await client.prompts.execute(promptKey: "p", options: ExecutePromptOptions(variables: ["name": "Alice"]))
// template: "Hi {{ name }}"   ← won't resolve, renders as "Hi "
// Fix: template: "Hi {{ input.name }}"
```

---

## Tips for Coding Agents

1. Use `--json` whenever piping output to other tools.
2. `primitive use <app-id>` once per session beats `--app` everywhere.
3. Prefer TOML + `sync push` over CLI flags for anything with multiple configs or test cases.
4. Always `preview` before `execute` when debugging templates — much faster.
5. Missing variables silently render as empty. Use `||` fallbacks or `| expect: "..."` to fail loudly. (Note: `inputSchema` is metadata only — it is NOT validated against `variables` at execute time.)
6. Evaluator prompts use `{{ output }}` (top-level), NOT `{{ input.output }}`.
7. `outputSchema` only works with `provider = "gemini"`. With openrouter, use `outputFormat = "json"` + prompt the model.
8. Test case `--vars` MUST be valid JSON — single-quote in shell, double-quote inside.
9. `sync pull` is lossy for `outputSchema` and `topP` — set those via the API client/CLI.
10. Both `draft` and `active` prompts execute. `archived` does not. There is no separate "publish" step.
