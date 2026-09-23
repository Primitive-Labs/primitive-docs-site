# Prompt Feature Guide for Coding Agents

How to author, test, and run LLM prompts on Primitive: `prompts/<key>.toml` configuration, the `primitive` CLI, and `ctx.prompts.run` from a server function.

## Run a prompt from a server function

```ts
import { defineFunction } from "primitive-functions";

export default defineFunction(async (input: { text: string }, ctx) => {
  const result = await ctx.prompts.run("summarizer", {
    variables: { text: input.text },
  });
  if (!result.success) throw new Error(result.error ?? "prompt failed");
  return { summary: result.output };
});
```

## Quick Mental Model

A **prompt** is versioned configuration: a named template (e.g. `summarizer`) plus 1+ **configs**, authored in `prompts/<key>.toml` and applied by `primitive config push`. Each config is a `(provider, model, systemPrompt, userPromptTemplate, ...)` tuple. One config is the **active** config — that's what runs when you don't pass `configId` / `--config`. Test cases are attached to the prompt and verify outputs (regex / contains / JSON subset / LLM evaluator).

A prompt runs from a **server function**, through `ctx.prompts.run(key, …)`. It has no client endpoint and no access rule of its own: the calling function's `access` gate is the whole authorization, and no capability line is needed to run a prompt. `primitive prompts execute` is the admin diagnostic for running one by hand.

Templates use `{{ }}` interpolation. Inputs are passed as `variables: { foo }` at run time and read as `{{ input.foo }}`.

---

## Availability

A prompt's availability is one server-owned `status` — `active | inactive | archived` — and it is **not** a TOML key. Every created or pushed prompt is active; `primitive prompts disable <prompt-id>` takes one out of service and `primitive prompts enable <prompt-id>` puts it back — both take the prompt ID `primitive prompts list` prints, not its key (the console has the matching action). A file that still carries a `[prompt] status` line fails `config push` with a message naming the verb.

| Status     | `ctx.prompts.run`                  | `primitive prompts execute` |
| ---------- | ---------------------------------- | --------------------------- |
| `active`   | Yes                                | Yes                         |
| `inactive` | No (`400 PROMPT_NOT_EXECUTABLE`)   | Yes — it is the diagnostic  |
| `archived` | No                                 | No — it has been deleted    |

`primitive prompts execute` is the documented way to trial an inactive prompt before enabling it.

**Archiving a prompt.** A plain admin `DELETE` (and the console's **Archive**) sets `status = "archived"` and destroys nothing: the prompt's configs and stored prompt bodies stay, so every execution and analytics row that names it keeps resolving. An archived prompt is refused everywhere — `ctx.prompts.run`, `primitive prompts execute`, test runs, and as another test case's **evaluator** — and `enable` will not bring it back. It goes on holding its `promptKey`. `DELETE ?hard=true` (the console's **Delete permanently**, and what `primitive config push --prune` sends) destroys the prompt, its configs and their stored bodies, and frees the key. To bring a key back after archiving: hard-delete the holder, then re-add the file and push — a bare re-push cannot clear a server-owned `archived`.

The per-CONFIG `status` is a different question and stays in TOML. A config defaults to `status = "active"`; `status = "archived"` takes that named version out of service, and the resolve path refuses it (`PROMPT_CONFIG_NOT_EXECUTABLE` when a run pins it). `config pull` writes the line only for a config that IS archived, so an ordinary pulled prompt file carries no `status` at all — an omitted line means active, and deleting an `archived` line puts that config back in service on the next push.

> `PROMPT_NOT_FOUND` (`404`) from `ctx.prompts.run` means no prompt with that key exists in the app. An inactive prompt is `400 PROMPT_NOT_EXECUTABLE`, not a `404`. A prompt with no active config and no `configId` in the call — or a `configId` that is not one of this prompt's configs — is `400 PROMPT_NO_CONFIG`.

---

## Template Syntax

Prompts render with the platform's strict template engine.

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
  meta: Record<string, any>,
  output?: any,                  // ONLY in evaluator prompts (see below)
}
```

### Missing variables fail the render

An unresolved reference fails the render. Guard an optional path with a `||` fallback.

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

### Direct model routes are off by default

`client.llm.*` / `client.gemini.*` (the `llm/chat`, `gemini/generate`, `gemini/generate-raw`, `gemini/count-tokens` routes) call a provider directly. They spend the app's LLM credit with no versioned template, no test cases and no function gate, so the whole surface is **off by default**: while `[app] directLlmEnabled` is not `true` in `app.toml`, every one of those four routes answers `403 { code: "DIRECT_LLM_DISABLED" }` — to app admins and owners as well, since it is a spend gate, not a role gate — and the same holds for a function calling them through `ctx.api.llm` / `ctx.api.gemini`. The `llm/models` and `gemini/models` listings stay readable.

Build model calls as prompts run with `ctx.prompts.run`, which does not go through the switch.

### Don't do this

```
# WRONG — assumes a missing var renders empty. It doesn't: the render fails.
"Hello {{ input.name }}!"   →  error (unresolved reference: input.name)

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
outputFormat = "text"            # optional: text (default) | json — request/response shaping only
reasoningEffort = "minimal"      # optional: none | minimal | low | medium | high — how much the provider may think
# reasoningBudget = 512          # ...or a token budget instead. Never both.
```

### Field reference

**`[prompt]`:**

| Key            | Required | Notes                                                                                |
| -------------- | -------- | ------------------------------------------------------------------------------------ |
| `key`          | Yes      | Unique per app                                                                       |
| `displayName`  | Yes      |                                                                                      |
| `description`  | No       |                                                                                      |
| `inputSchema`  | No       | JSON Schema, as a `[prompt.inputSchema]` table or a JSON string                      |
| `outputSchema` | No       | JSON Schema, same forms. Round-trips: `config pull` writes it back. **This is the one a server function reads** — see below                     |

`config push` applies the complete `[prompt]` table: a file with no `outputSchema` key clears any output schema stored on the server.

**`[prompt.outputSchema]` is the declaration that counts.** It is what a run sends to the provider, and it is what `ctx.prompts.run("<key>")` parses and validates the answer against, handing it back as `parsed` — typed, because `config push` renders it into `functions/primitive-prompt-types.d.ts`. A config's own `[configs.outputSchema]` is stored with that config and is **not** read by `ctx.prompts.run`.

What a function gets back — `parsed`, the `PROMPT_OUTPUT_*` codes, the generated types — is under Typed output (in Running from a server function).

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
| `outputFormat`       | No       | `text` (default) \| `json`. On openrouter, `json` requests provider JSON mode (`response_format`); on gemini it only normalizes the response (fence stripping) — a gemini config constrains the request with `outputSchema` instead. `json` alone gives `ctx.prompts.run` an untyped `parsed`. |
| `outputSchema`       | No       | Config-level JSON Schema, stored with the config. `ctx.prompts.run` validates against `[prompt.outputSchema]`, not this one |
| `providerConfig`     | No       | Stored with the config and returned by the API, but **not applied to the provider request**. Nothing reads it at execution — to bound the model's reasoning use `reasoningEffort` / `reasoningBudget` below |
| `reasoningEffort`    | No       | `none` \| `minimal` \| `low` \| `medium` \| `high`. How much the provider may spend on reasoning before answering. Mutually exclusive with `reasoningBudget` |
| `reasoningBudget`    | No       | The same control as a whole number of reasoning tokens. Mutually exclusive with `reasoningEffort` |
| `active`             | No       | Marks this entry as the live config — exactly one entry may carry `active = true` (two is an error, not first-wins). Written by `config pull`, honored on create and update |

**Not exposed in TOML**: the config's `status` (activation is its own endpoint) and the server-owned ids/timestamps. Every field in the tables above round-trips — `config pull` writes back what the server holds, and `config push` rejects a key the CLI does not recognize rather than dropping it silently.

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

Mark the live config with `active = true` on exactly one `[[configs]]` entry. `config pull` writes that marker and `config push` honors it on create AND update, so the committed file always says which config the server is actually running. Two markers is an error rather than a first-wins rule. With no marker, the first entry becomes active when the prompt is created and nothing is re-activated afterwards.

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

The app is the one the project's selected environment names in `primitive/config.json` (`primitive whoami` reports it). All commands accept `--app <app-id>` or a positional `[app-id]` to override.

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
`status = "archived"` archives one, and duplicating is copying the block under a
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
primitive prompts tests list <prompt>
primitive prompts tests get  <prompt> <test-case-id>
```

Every `<prompt>` argument of the `prompts tests` commands is the prompt's
**key** (the name `prompts list` prints beside the id, and the one
`prompts/<key>.tests/` is addressed by) or its id. An identifier that names no
prompt in the app exits non-zero with `No prompt '<arg>' in app <app-id>`, so
"No test cases found." only ever means the prompt exists and has no registered
cases.

### Running tests

```bash
primitive prompts tests run <prompt> <test-case-id> [--config <config-id>] [--json]
primitive prompts tests run-all <prompt> [--config <config-id>] [--test-cases "id1,id2,id3"] [--json]
primitive prompts tests runs <prompt> [--limit 20] [--group <comparison-group>] [--json]
```

`run-all` exits with code `1` if any test fails. Useful for CI. It executes the **registered** cases (the ones a push has sent), not whatever is on disk — see [the case lifecycle](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md#a-case-file-is-local-until-a-push-registers-it) for the local/registered distinction and `config diff`'s counters.

### Batch (parallel) test execution

Runs tests in parallel — much faster for large suites.

```bash
primitive prompts tests batch start  <prompt> [--config <config-id>] [--test-cases "id1,id2"] [--json]
primitive prompts tests batch status <prompt> <batch-id> [--wait] [--json]
primitive prompts tests batch cancel <prompt> <batch-id> [-y]
```

`status --wait` polls every 2s until completion. Exits `1` if any test failed.

### Test case attachments (PDFs, images, etc.)

Attachments are authored as files in `prompts/<key>.tests/<case>/` and uploaded
by `primitive config push`; removing a file and running `config push --prune`
deletes it server-side. The CLI reads server state:

```bash
primitive prompts tests attachments list     <prompt> <test-case-id>
primitive prompts tests attachments download <prompt> <test-case-id> doc.pdf [output-path]
```

Upload size limit: **10 MB**. Attachments are sent to the model as file parts (`gemini`) or vision parts (`openrouter`) and are NOT visible in template context.

> Pass attachments at runtime by including `attachments: [{name, type, data}]` in `variables`. The base64 `data` is stripped from the template context automatically and forwarded as a file part.

---

## Sync (TOML version control)

Prompt configs live at `prompts/<key>.toml`, with test cases in a sibling `<key>.tests/` directory. See the [Configuration guide](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md#the-sync-loop) for the sync loop (`init`/`pull`/`diff`/`push`) and how the directory is resolved.

### Directory layout

```
primitive/<env>/
  prompts/
    summarizer.toml
    summarizer.tests/                # NOTE: dir name is `<key>.tests`
      basic.toml
      edge-case.toml
      basic/                         # attachments dir (one per test case slug)
        document.pdf
    evaluator.toml
  functions/
    ...
  .sync-state.json                   # auto-generated state — commit this
```

### Test case TOML schema

Test case TOMLs use a `[test]` table with **JSON-encoded strings** for structured fields:

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

Pull and push share one key set, so they cannot disagree about which fields exist:

- `[prompt]`: `key, displayName, description, inputSchema, outputSchema`
- `[[configs]]`: `active` (on the live one only), `name, description, provider, model, systemPrompt, userPromptTemplate, temperature, topP, maxTokens, outputFormat, outputSchema, providerConfig, reasoningEffort, reasoningBudget`

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

## Common Tasks

### Create from scratch

```bash
primitive config create prompt greeting-generator
```

That writes `prompts/greeting-generator.toml` from the type's defaults. Fill it in:

```toml
[prompt]
key = "greeting-generator"
displayName = "Greeting Generator"

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
git add primitive/ && git commit -m "Snapshot prompts"

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

`outputSchema` **constrains the provider request only on the gemini provider** (native structured output). With openrouter, set `outputFormat = "json"` and instruct the model in the system prompt instead. Either way, `ctx.prompts.run` validates the answer against `[prompt.outputSchema]` on the platform, so a declared shape is enforced on both providers.

`outputFormat` shapes the **provider request** on openrouter (`json` → `response_format: { type: "json_object" }`) and the **response normalization** on both providers (fence stripping). On gemini it does NOT constrain the request — that is `outputSchema`'s job, per the paragraph above. Recommended pairing for a prompt that answers JSON: `[prompt.outputSchema]` always, plus `outputFormat = "json"` on an openrouter config.

`ctx.prompts.run`, the admin execute endpoint and `primitive prompts execute` all use **`[prompt.outputSchema]`**. `[[configs]].outputSchema` is authorable and round-trips, but none of them reads it.

---

## Reasoning budget

On a reasoning-by-default model, thinking is most of what a call costs — in
latency as well as money. Measured on a merchant-categorisation prompt
(`google/gemini-3.6-flash` through openrouter), reasoning was 49–82% of the
output tokens on every call, and the duration tracked the output count almost
linearly. A classification task does not want that, and changing the model is
not the answer: it re-opens accuracy on a prompt whose test cases are tuned to
the current one.

A config states the budget in one of two ways, never both:

```toml novalidate
[[configs]]
reasoningEffort = "minimal"   # none | minimal | low | medium | high
# reasoningBudget = 512       # ...or a whole number of reasoning tokens
```

It travels with the NAMED config, so a reasoning and a non-reasoning variant of
the same prompt are two `[[configs]]` entries and can be A/B'd through the
ordinary test cases.

**Nothing is silently dropped.** Each value either maps onto the provider's own
spelling or is refused at push, naming the remedy:

| Provider / family | `reasoningEffort` | `reasoningBudget` |
| ----------------- | ----------------- | ----------------- |
| `openrouter` | Sent as OpenRouter's `reasoning.effort`; `none` becomes `reasoning: { enabled: false }` | Sent as `reasoning.max_tokens` |
| `gemini`, 3.x | `thinkingConfig.thinkingLevel`. Gemini 3 cannot turn thinking off, so `none` is refused; Pro takes `low`/`high` only | Refused — Gemini 3 translates a budget to a level rather than honoring it as a bound. Use `reasoningEffort` |
| `gemini`, 2.5 | Only `none`, which means a budget of 0. Anything else is refused | `thinkingConfig.thinkingBudget`. Gemini 2.5 Pro cannot stop thinking: its floor is 128 tokens |
| `gemini`, 2.0 and earlier | Refused — the model has no reasoning control | Refused |

Two caveats worth knowing before you rely on a number:

- On `openrouter`, `reasoningBudget` is an exact bound only on budget-native
  models. Effort-only models translate it to the nearest effort level. The
  budget is a request, and `metrics.reasoningTokens` is the record of what was
  actually spent.
- Every openrouter request that carries a reasoning setting also carries
  `provider: { require_parameters: true }`, which keeps it away from endpoints
  that would accept the request and ignore the setting. A model that cannot
  honor what you asked for therefore FAILS the execution with the provider's
  own reason — for example, "Reasoning is mandatory for this endpoint and
  cannot be disabled" — rather than quietly running without it.

### Measuring it

`metrics.reasoningTokens` is reported separately by every path that returns
metrics: `ctx.prompts.run` in a server function, the admin execute endpoint,
and `primitive prompts execute`. It is the only honest number: OpenRouter counts
reasoning INSIDE `completion_tokens`, Gemini counts it OUTSIDE
`candidatesTokenCount`, so neither headline figure says how much of the decode
was deliberation. It is absent when the provider reports none, and `0` when a
setting successfully declined it.

---

## Running from a server function

`ctx.prompts.run(promptKey, { variables, modelOverride, configId })` — see [Server Functions](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#running-a-prompt) for the full envelope.

- The first argument is the `promptKey`, NOT the `promptId`. Once the tree has pushed, a key it does not declare is a compile error.
- `variables` becomes the `input` namespace in templates. `variables: { x: 1 }` → `{{ input.x }}`. There is no top-level access to your variables (`{{ x }}` won't resolve).
- `configId` pins a config of THIS prompt; `modelOverride` swaps the model for this call only.
- No capability line: a prompt is reachable from any function, and the function's `access` gate is the authorization.
- The model call is bounded by the invocation's remaining time; past it the call fails with `PROMPT_UPSTREAM_TIMEOUT`.
- A run with a caller emits a `prompt.executed` analytics event attributed to that caller; a trigger-fired run (no caller) emits none.

| Answer | Meaning |
|---|---|
| `success: true` | `output` is the text; `parsed` is the validated JSON value when `[prompt.outputSchema]` is declared (typed), or when the config that ran declares `outputFormat = "json"` (untyped). |
| `success: false`, `errorCode: "PROMPT_OUTPUT_NOT_JSON"` | Declared JSON; the answer did not parse. `output` has the text. |
| `success: false`, `errorCode: "PROMPT_OUTPUT_SCHEMA_VIOLATION"` | Parsed, and `[prompt.outputSchema]` refuses it; `error` names the failing paths. |
| `success: false`, no `errorCode` | The provider call failed; `error` says why. |
| `404 PROMPT_NOT_FOUND` | No prompt with that key. |
| `400 PROMPT_NOT_EXECUTABLE` | The prompt is inactive or archived. |
| `400 PROMPT_NO_CONFIG` / `PROMPT_CONFIG_NOT_EXECUTABLE` | No usable config: none active, a `configId` that is not this prompt's, or a pinned config that is archived. |

### Typed output

Declare `[prompt.outputSchema]` and stop parsing JSON by hand: the platform sends the schema to the provider, parses the answer, validates it against the schema, and hands it back as `parsed`.

```toml
# prompts/categorize.toml
[prompt]
key = "categorize"
displayName = "Categorize"

[prompt.outputSchema]
type = "object"
required = ["suggested_transactions"]

[prompt.outputSchema.properties.suggested_transactions]
type = "array"
```

```ts
const answer = await ctx.prompts.run("categorize", { variables: { payload } });
if (!answer.success) throw new Error(answer.error ?? "categorize failed");
// `parsed` is typed from the schema above; no JSON.parse, no failure branch.
return { suggested: answer.parsed.suggested_transactions };
```

- `parsed` is on the **success arm only**, which is what the `success` check unlocks. Reading it unguarded is a compile error, and so is a field the schema does not declare.
- A bad answer is the envelope's failure arm, never an HTTP error. Two shape failures, both `success: false` with an `errorCode`, both keeping `output`, `metrics` and `configId` because the run happened and was billed: `PROMPT_OUTPUT_NOT_JSON` (declared JSON; the model answered text that does not parse, or that parses to a number JSON cannot represent) and `PROMPT_OUTPUT_SCHEMA_VIOLATION` (it parsed and the schema refuses it; `error` names the failing paths).
- A **provider** failure has `error` set and **no** `errorCode`. That is how to tell a wrong shape from a failed model.
- A validation `error` names paths and the schema's constraints and never quotes the model's answer — `output` is where the answer is, so a diagnostic in a log carries no generated content.
- `ctx.prompts.run` validates against `[prompt.outputSchema]`; a config's own `[configs.outputSchema]` is not read here. `outputFormat = "json"` on the config that ran gives `parsed` when the text parses, but untyped (`unknown`); declare the schema to get a type.
- **Generated types.** `config push` renders `functions/primitive-prompt-types.d.ts`: `<Key>PromptOutput` for every `prompts/*.toml` declaring a `[prompt.outputSchema]`, and the `PromptSchemas` augmentation keyed by prompt key that types `ctx.prompts.run("<key>")`. A prompt declaring no schema gets an empty entry, so its `parsed` stays optional and `unknown` — and removing a schema and pushing makes a handler reading `parsed.field` a compile error rather than a runtime `undefined`.
- The prompt key is the one push deploys: `[prompt] key` when declared, the file name otherwise.

### Don't do this

```ts
// WRONG — passing the prompt id instead of the prompt key
await ctx.prompts.run("01HXY...PROMPT_ID", { variables: {} });
// → PROMPT_NOT_FOUND. Use the key from the TOML.

// WRONG — expecting a top-level variable
await ctx.prompts.run("p", { variables: { name: "Alice" } });
// template: "Hi {{ name }}"   ← unresolved reference, the render fails
// Fix: template: "Hi {{ input.name }}"
```

---

## Tips for Coding Agents

1. Use `--json` whenever piping output to other tools.
2. Run inside the project so the environment names the app, rather than passing `--app` everywhere.
3. Prefer TOML + `config push` over CLI flags for anything with multiple configs or test cases.
4. Always `preview` before `execute` when debugging templates — much faster.
5. A missing variable fails the render. Use `||` fallbacks for paths that may be absent. (Note: `inputSchema` is metadata only — it is NOT validated against `variables` at execute time.)
6. Evaluator prompts use `{{ output }}` (top-level), NOT `{{ input.output }}`.
7. `outputSchema` constrains the provider request only with `provider = "gemini"`. With openrouter, use `outputFormat = "json"` + prompt the model. Declare `[prompt.outputSchema]` either way: `ctx.prompts.run` validates against it on both providers and types `parsed` from it.
8. A test case's `test.inputVariables` is JSON **text** in the sidecar TOML (`prompts/<key>.tests/<case>.toml`) — a single-quoted TOML string holding a valid JSON object.
9. `config pull` writes every authorable field back, including `outputSchema`, `topP` and `providerConfig` — a mistyped key fails the push instead of being dropped.
10. Availability is `primitive prompts disable`/`enable`, never a TOML key — there is no separate "publish" step, and everything you push is live. Archiving a prompt outright is `primitive prompts archive <id>` (the delete flow's soft path: the row and its configs stay, it keeps its `promptKey`, `enable` refuses it, and there is no un-archive).
