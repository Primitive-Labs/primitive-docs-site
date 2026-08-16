# Prompt Feature Guide for Coding Agents

How to author, test, and execute LLM prompts on Primitive using the `primitive` CLI, TOML files, and the client SDK.

## Execute a managed prompt

```typescript
  const result = await client.prompts.execute("my-summarizer", {
    variables: { text: documentText, style: "concise" },
    modelOverride: "gpt-4o", // optional; defaults to the active config's model
  });
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

Read and written by `primitive sync pull` / `primitive sync push` — the only path that creates or updates a prompt.

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
| `inputSchema`  | No       | JSON Schema, as a `[prompt.inputSchema]` table or a JSON string                      |
| `outputSchema` | No       | JSON Schema, same forms. Round-trips: `sync pull` writes it back                     |

`sync push` applies the complete `[prompt]` table: a file with no `outputSchema` key clears any output schema stored on the server. Prompt files written by an older CLI omit that key even when the server has a schema — run `primitive sync pull` once before the first push after upgrading the CLI.

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
| `topP`               | No       | Nucleus-sampling cutoff; stored as string          |
| `maxTokens`          | No       | Integer                                            |
| `outputFormat`       | No       | `text` (default) \| `json`                         |
| `outputSchema`       | No       | Config-level JSON Schema — the one a workflow `prompt.execute` step uses |
| `providerConfig`     | No       | Provider-specific options, as a `[configs.providerConfig]` table |
| `isActive`           | No       | On a NEW prompt, activates this entry after create. Not written by pull |

**Not exposed in TOML**: the config's `status` (activation is its own endpoint) and the server-owned ids/timestamps. Every field in the tables above round-trips — `sync pull` writes back what the server holds, and `sync push` rejects a key the CLI does not recognize rather than dropping it silently (#2644).

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

Mark the live config with `active = true` on exactly one `[[configs]]` entry. `sync pull` writes that marker and `sync push` honors it on create AND update, so the committed file always says which config the server is actually running. Two markers is an error rather than a first-wins rule. With no marker, the first entry becomes active when the prompt is created and nothing is re-activated afterwards.

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

### Reading prompts

```bash
primitive prompts list [app-id] [--status draft|active|archived] [--json]
primitive prompts get <prompt-id> [--json]
```

### Writing prompts (TOML only)

A prompt is `prompts/<key>.toml`: a `[prompt]` table plus one `[[configs]]` block per named model config, exactly one of which carries `active = true`. There is no create/update/delete command.

```bash
primitive config fields prompt                      # every key, type, required, default
primitive config new prompt summarizer              # scaffold prompts/summarizer.toml
primitive config set prompt/summarizer prompt.status=active
primitive sync push --only prompt/summarizer      # apply just this prompt
```

Delete a prompt by removing its file and running `primitive sync push --prune`; its `<key>.tests/` sidecar goes with it.

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
primitive prompts configs get <prompt> <config>   # by key or id, name or id
```

Configs are read here and written in `prompts/<key>.toml`: each is a
`[[configs]]` entry with `name`, `provider`, `model`, `userPromptTemplate` and
the rest of the table below. `active = true` marks the live one,
`status = "archived"` retires one, and duplicating is copying the block under a
new `name`. Apply with `primitive sync push --only prompt/<key>`.

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

Prompt configs live at `prompts/<key>.toml`, with test cases in a sibling `<key>.tests/` directory. See the [Configuration guide](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md#the-sync-loop) for the sync loop (`init`/`pull`/`diff`/`push`) and the `--dir` override.

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

`serializePrompt` takes its key set from the prompt's configuration-object
definition (`src/config-surface/prompt.ts`, #2644), so pull and push cannot
disagree about which fields exist:

- `[prompt]`: `key, displayName, description, status, inputSchema, outputSchema`
- `[[configs]]`: `active` (on the live one only), `name, description, provider, model, systemPrompt, userPromptTemplate, temperature, topP, maxTokens, outputFormat, outputSchema, providerConfig`

A field the server has not set is omitted (there is no TOML `null`), and a JSON
field TOML cannot represent faithfully — a `null` anywhere inside a schema — is
written as a JSON string instead of losing the member. A key the server returns
that this CLI version does not know is named in a warning and left out of the
file; upgrade the CLI to manage it.

### What `sync push` does

- Updates existing prompts (matched by `key`) and reconciles configs (matched by `name`).
- Creates new prompts and additional configs that don't exist on the server.
- Activates the `[[configs]]` entry marked `active = true`, on create and on update alike. With no marker, the first entry becomes active on create.
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
primitive config new prompt greeting-generator
```

That writes `prompts/greeting-generator.toml` from the type's defaults. Fill it in:

```toml
[prompt]
key = "greeting-generator"
displayName = "Greeting Generator"
status = "active"

[[configs]]
name = "default"
active = true
provider = "gemini"
model = "models/gemini-3-flash-preview"
temperature = 0.7
userPromptTemplate = "Generate a friendly greeting for {{ input.name || 'friend' }} who works as a {{ input.occupation || 'professional' }}."
```

```bash
primitive sync push --only prompt/greeting-generator
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

```toml
# prompts/<key>.toml — add the candidate alongside the live one
[[configs]]
name = "creative"
provider = "gemini"
model = "models/gemini-3-pro-preview"
temperature = 0.9
userPromptTemplate = "Generate a unique greeting for {{ input.name }} ({{ input.occupation }})."
```

```bash
primitive sync push --only prompt/<key>
primitive prompts configs list <prompt-id>          # the new config's id
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

Both are authorable: `[prompt].outputSchema` writes the prompt-level field and `[[configs]].outputSchema` writes the config-level one used by the workflow path.

---

## Client SDK

```typescript
  const result = await client.prompts.execute("my-prompt-key", {
    variables: { text: "Hello world" }, // → {{ input.text }}
    configId: "01ABC", // optional; defaults to activeConfigId
    modelOverride: "anthropic/claude-3-5-sonnet", // optional; overrides config.model
  });

  result.success; // boolean
  result.output; // string — generated text
  result.error; // string | undefined
  result.configId; // string — which config was used
  result.metrics; // { durationMs, inputTokens?, outputTokens?, totalTokens? }
  result.rawResponse; // any — raw provider response (don't depend on shape)
```

The first arg is the `promptKey`, NOT the `promptId`. The endpoint is `POST /prompts/:promptKey/execute`.

`variables` becomes the `input` namespace in templates. `variables: { x: 1 }` → `{{ input.x }}`. There is no top-level access to your variables (e.g. `{{ x }}` won't resolve).

### Don't do this

```typescript
// WRONG — passing promptId instead of promptKey
await client.prompts.execute("01HXY...PROMPT_ID", { variables: {} });
// → 404. Use the key from the TOML.

// WRONG — expecting a top-level variable
await client.prompts.execute("p", { variables: { name: "Alice" } });
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
9. `sync pull` writes every authorable field back, including `outputSchema`, `topP` and `providerConfig` — a mistyped key fails the push instead of being dropped.
10. Both `draft` and `active` prompts execute. `archived` does not. There is no separate "publish" step.
