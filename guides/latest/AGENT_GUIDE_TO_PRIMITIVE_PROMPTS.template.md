# Prompt Feature Guide for Coding Agents

How to author, test, and execute LLM prompts on Primitive using the `primitive` CLI, TOML files, and the client SDK.

## Execute a managed prompt

{{ example: prompts/prompt-execute }}

## Quick Mental Model

A **prompt** is a named template (e.g. `summarizer`) plus 1+ **configs**. Each config is a `(provider, model, systemPrompt, userPromptTemplate, ...)` tuple. One config is the **active** config — that's what gets used when you don't pass `--config` / `configId`. Test cases are attached to the prompt and verify outputs (regex / contains / JSON subset / LLM evaluator).

Templates use `{{ }}` interpolation. Inputs are passed as `variables: { foo }` at execution time and read as `{{ input.foo }}`.

---

## Availability

A prompt's availability is one server-owned `status` — `active | inactive` — and it is **not** a TOML key. Every created or pushed prompt is active; `primitive prompts disable <key>` takes one out of service and `primitive prompts enable <key>` puts it back (the console has the matching action). A file that still carries a `[prompt] status` line fails `config push` with a message naming the verb.

| Status     | Executable from a workflow | Executable from the client SDK / REST | `primitive prompts execute` |
| ---------- | -------------------------- | ------------------------------------- | --------------------------- |
| `active`   | Yes                        | Yes                                   | Yes                         |
| `inactive` | No                         | No (`400 PROMPT_NOT_EXECUTABLE`)      | Yes — it is the diagnostic  |

A prompt stored before this model may still carry `draft` or `archived`; both read `inactive`, so a legacy draft prompt no longer executes at the member endpoint. `primitive prompts execute` is the documented way to trial an inactive prompt before enabling it.

The per-CONFIG `status` is a different question and stays in TOML. A config defaults to `status = "active"`; `status = "archived"` retires that named version, and the resolve path refuses it.

> If you see `HTTP 404` calling a prompt via the SDK: the prompt key is wrong (no prompt with that key in the app). An inactive prompt is a `400`, not a `404`. If the prompt has no `activeConfigId` and you didn't pass `configId`, you'll also get a 400 ("No configuration found for this prompt").

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

### The direct LLM proxy is deprecated and off by default

`client.llm.*` / `client.gemini.*` (the `llm/chat`, `gemini/generate`, `gemini/generate-raw`, `gemini/count-tokens` routes) are the *old* way to reach a model. They spend the app's LLM credit with no rule saying who may do it, so the whole surface is **off by default**: while `[app] directLlmEnabled` is not `true` in `app.toml`, every one of those four routes answers `403 { code: "DIRECT_LLM_DISABLED" }` — to app admins and owners as well, since it is a spend gate, not a role gate. The routes are deprecated and are **removed in the next major server release**.

Use a managed prompt (`client.prompts.execute`) or a workflow LLM step instead. Neither goes through the switch: they run whatever it is set to, gated by their own `accessRule`.

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

Read and written by `primitive config pull` / `primitive config push` — the only path that creates or updates a prompt.

### Basic structure

```toml
[prompt]
key = "my-prompt"                # required, unique per app, kebab-case
displayName = "My Prompt"        # required
description = "What it does"     # optional
accessRule = "true"              # REQUIRED at create: CEL deciding who may execute.
                                 # Absent/empty = every non-admin execution denied.
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
outputFormat = "text"            # optional: text (default) | json — request/response shaping only, never a workflow's type
```

### Field reference (verified against `src/models/app-prompt.js` and `app-prompt-config.js`)

**`[prompt]`:**

| Key            | Required | Notes                                                                                |
| -------------- | -------- | ------------------------------------------------------------------------------------ |
| `key`          | Yes      | Unique per app                                                                       |
| `displayName`  | Yes      |                                                                                      |
| `description`  | No       |                                                                                      |
| `accessRule`   | Yes at create | CEL deciding who may execute. Absent/empty = every non-admin execution denied with `403 { errorCode: "PROMPT_ACCESS_DENIED" }`; `"true"` allows any app member |
| `inputSchema`  | No       | JSON Schema, as a `[prompt.inputSchema]` table or a JSON string                      |
| `outputSchema` | No       | JSON Schema, same forms. Round-trips: `config pull` writes it back                     |

`config push` applies the complete `[prompt]` table: a file with no `outputSchema` key clears any output schema stored on the server. Prompt files written by an older CLI omit that key even when the server has a schema — run `primitive config pull` once before the first push after upgrading the CLI.

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
| `outputFormat`       | No       | `text` (default) \| `json`. On openrouter, `json` requests provider JSON mode (`response_format`); on gemini it only normalizes the HTTP execute response (fence stripping) — a gemini config constrains the request with `outputSchema` instead. Either way it is **not** the type a workflow step sees: a `prompt.execute` step types its `content` with its own `expect = "json" \| "text"` declaration. |
| `outputSchema`       | No       | Config-level JSON Schema — the one a workflow `prompt.execute` step uses |
| `providerConfig`     | No       | Provider-specific options, as a `[configs.providerConfig]` table |
| `active`             | No       | Marks this entry as the live config — exactly one entry may carry `active = true` (two is an error, not first-wins). Written by `config pull`, honored on create and update. `isActive` is accepted as a legacy spelling |

**Not exposed in TOML**: the config's `status` (activation is its own endpoint) and the server-owned ids/timestamps. Every field in the tables above round-trips — `config pull` writes back what the server holds, and `config push` rejects a key the CLI does not recognize rather than dropping it silently.

### Multiple configs

```toml
[prompt]
key = "summarizer"
displayName = "Document Summarizer"
accessRule = "true"

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

Mark the live config with `active = true` on exactly one `[[configs]]` entry. `config pull` writes that marker and `config push` honors it on create AND update, so the committed file always says which config the server is actually running. Two markers is an error rather than a first-wins rule. With no marker, the first entry becomes active when the prompt is created and nothing is re-activated afterwards.

### Evaluator prompts

Evaluators judge another prompt's output. They get TWO context entries:

- `{{ input.* }}` — the **original input variables** that were passed to the prompt being evaluated
- `{{ output }}` — the **output text** from that prompt (top-level, NOT under `input`)

```toml
[prompt]
key = "haiku-evaluator"
displayName = "Haiku Evaluator"
accessRule = "true"

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
primitive prompts list [app-id] [--status active|inactive] [--json]
primitive prompts get <prompt-id> [--json]
```

### Writing prompts (TOML only)

A prompt is `prompts/<key>.toml`: a `[prompt]` table plus one `[[configs]]` block per named model config, exactly one of which carries `active = true`. There is no create/update/delete command.

```bash
primitive config fields prompt                  # every key, type, required, default
primitive config create prompt summarizer       # scaffold prompts/summarizer.toml
primitive config push --only prompt/summarizer  # apply just this prompt
```

Delete a prompt by removing its file and running `primitive config push --prune`; its `<key>.tests/` sidecar goes with it.

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
new `name`. Apply with `primitive config push --only prompt/<key>`.

### Test cases

Test cases are authored in TOML, one file per case, in a sidecar directory
beside the prompt, and applied by `config push`:

```toml
# prompts/greeting.tests/basic-test.toml
[test]
name = "Basic test"
description = "Greets by name"
inputVariables = '{"text":"hello"}'
# Empty means "not set": the case runs against the active config, unpinned and
# unscored. Blanking a field and pushing CLEARS it server-side.
configName = ""
evaluatorPromptKey = ""
evaluatorConfigName = ""
expectedOutputPattern = ""
expectedOutputContains = '["hi","hello"]'
expectedJsonSubset = '{}'
```

`primitive config fields prompt` lists every key. `inputVariables`,
`expectedOutputContains` and `expectedJsonSubset` carry JSON **text** — invalid
JSON fails the push preflight, before anything is applied. Deleting a case is
removing its file and running `primitive config push --prune`.

Read them back from the CLI:

```bash
primitive prompts tests list <prompt-id>
primitive prompts tests get  <prompt-id> <test-case-id>
```

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

Attachments are authored as files in `prompts/<key>.tests/<case>/` and uploaded
by `primitive config push`; removing a file and running `config push --prune`
deletes it server-side. The CLI reads server state:

```bash
primitive prompts tests attachments list     <prompt-id> <test-case-id>
primitive prompts tests attachments download <prompt-id> <test-case-id> doc.pdf [output-path]
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

### What `config pull` actually writes

`serializePrompt` takes its key set from the prompt's shared configuration-object
definition, so pull and push cannot disagree about which fields exist:

- `[prompt]`: `key, displayName, description, status, accessRule, inputSchema, outputSchema`
- `[[configs]]`: `active` (on the live one only), `name, description, provider, model, systemPrompt, userPromptTemplate, temperature, topP, maxTokens, outputFormat, outputSchema, providerConfig`

A field the server has not set is omitted (there is no TOML `null`), and a JSON
field TOML cannot represent faithfully — a `null` anywhere inside a schema — is
written as a JSON string instead of losing the member. A key the server returns
that this CLI version does not know is named in a warning and left out of the
file; upgrade the CLI to manage it.

### What `config push` does

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
primitive config create prompt greeting-generator
```

That writes `prompts/greeting-generator.toml` from the type's defaults. Fill it in:

```toml
[prompt]
key = "greeting-generator"
displayName = "Greeting Generator"
status = "active"
accessRule = "true"

[[configs]]
name = "default"
active = true
provider = "gemini"
model = "models/gemini-3-flash-preview"
temperature = 0.7
userPromptTemplate = "Generate a friendly greeting for {{ input.name || 'friend' }} who works as a {{ input.occupation || 'professional' }}."
```

```bash
primitive config push --only prompt/greeting-generator
```

### Test it

```bash
primitive prompts preview <prompt-id> --vars '{"name":"Alice","occupation":"engineer"}'
primitive prompts execute <prompt-id> --vars '{"name":"Alice","occupation":"engineer"}'
```

### Add a regression test

```toml
# prompts/greeting-generator.tests/mentions-name-and-occupation.toml
[test]
name = "Mentions name and occupation"
inputVariables = '{"name":"Bob","occupation":"teacher"}'
expectedOutputContains = '["Bob","teacher"]'
```

```bash
primitive config push --only prompt/greeting-generator
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
primitive config push --only prompt/<key>
primitive prompts configs list <prompt-id>       # the new config's id
primitive prompts tests run-all <prompt-id> --config <new-config-id>
primitive prompts tests runs <prompt-id> --json  # compare runs across configs
```

### Version control

```bash
primitive config pull
git add .primitive/sync && git commit -m "Snapshot prompts"

# edit the synced prompts/*.toml files

primitive config push --dry-run  # preview
primitive config push            # apply
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

`outputFormat` shapes the **provider request** on openrouter (`json` → `response_format: { type: "json_object" }`) and the **HTTP execute response's normalization** on both providers (fence stripping). On gemini it does NOT constrain the request — that is `outputSchema`'s job, per the paragraph above. It never decides what a workflow step receives either: a `prompt.execute` step declares `expect = "json"` or `"text"` (the default) and that declaration alone types its `content` output — see the workflows guide. Switching the active config between `text` and `json` therefore no longer retypes what downstream steps and scripts get. Recommended pairing for a `expect = "json"` step: `outputFormat = "json"` on an openrouter config, `outputSchema` on a gemini one.

Both `AppPrompt` (prompt-level) and `AppPromptConfig` (config-level) have an `outputSchema` field, and which one is honored depends on the execution path:

- SDK / user REST `POST /prompts/:promptKey/execute` and admin/CLI execute endpoint → use **`prompt.outputSchema`**.
- Workflow `prompt.execute` step → uses **`config.outputSchema`**.

Both are authorable: `[prompt].outputSchema` writes the prompt-level field and `[[configs]].outputSchema` writes the config-level one used by the workflow path.

---

## Client SDK

{{ example: prompts/prompt-execute-config }}

The first arg is the `promptKey`, NOT the `promptId`. The endpoint is `POST /prompts/:promptKey/execute`.

`variables` becomes the `input` namespace in templates. `variables: { x: 1 }` → `{{ input.x }}`. There is no top-level access to your variables (e.g. `{{ x }}` won't resolve).

An execution the prompt's `accessRule` refuses answers `403 { errorCode: "PROMPT_ACCESS_DENIED" }` — the same answer whether the rule evaluated false or the prompt has no rule (admins and owners always pass).
{{#lang swift}}
The refusal reaches Swift as a thrown `HttpError` with `serverCode == "PROMPT_ACCESS_DENIED"` — switch on that, not on the message text.
{{/lang}}

### Don't do this

{{#lang ts}}
```typescript
// WRONG — passing promptId instead of promptKey
await client.prompts.execute("01HXY...PROMPT_ID", { variables: {} });
// → 404. Use the key from the TOML.

// WRONG — expecting a top-level variable
await client.prompts.execute("p", { variables: { name: "Alice" } });
// template: "Hi {{ name }}"   ← won't resolve, renders as "Hi "
// Fix: template: "Hi {{ input.name }}"
```
{{/lang}}
{{#lang swift}}
```swift
// WRONG — passing the prompt id instead of the prompt key
try await client.prompts.execute(promptKey: "01HXY...PROMPT_ID", options: ExecutePromptOptions(variables: [:]))
// → 404. Use the key from the TOML.

// WRONG — expecting a top-level variable
try await client.prompts.execute(promptKey: "p", options: ExecutePromptOptions(variables: ["name": "Alice"]))
// template: "Hi {{ name }}"   ← won't resolve, renders as "Hi "
// Fix: template: "Hi {{ input.name }}"
```
{{/lang}}

---

## Tips for Coding Agents

1. Use `--json` whenever piping output to other tools.
2. `primitive use <app-id>` once per session beats `--app` everywhere.
3. Prefer TOML + `config push` over CLI flags for anything with multiple configs or test cases.
4. Always `preview` before `execute` when debugging templates — much faster.
5. Missing variables silently render as empty. Use `||` fallbacks or `| expect: "..."` to fail loudly. (Note: `inputSchema` is metadata only — it is NOT validated against `variables` at execute time.)
6. Evaluator prompts use `{{ output }}` (top-level), NOT `{{ input.output }}`.
7. `outputSchema` only works with `provider = "gemini"`. With openrouter, use `outputFormat = "json"` + prompt the model.
   `outputFormat` does **not** decide what a workflow gets: a `prompt.execute` step declares `expect = "json"` (or `"text"`, the default) to fix the type of its `content`. Pair the step's declaration with whatever constrains YOUR provider — `outputFormat = "json"` on openrouter, `outputSchema` on gemini (where `outputFormat` only normalizes the response).
8. A test case's `test.inputVariables` is JSON **text** in the sidecar TOML (`prompts/<key>.tests/<case>.toml`) — a single-quoted TOML string holding a valid JSON object.
9. `config pull` writes every authorable field back, including `outputSchema`, `topP` and `providerConfig` — a mistyped key fails the push instead of being dropped.
10. Availability is `primitive prompts disable`/`enable`, never a TOML key — there is no separate "publish" step, and everything you push is live.
