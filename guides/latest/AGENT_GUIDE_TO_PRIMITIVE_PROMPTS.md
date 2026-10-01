# Prompt Guide for Coding Agents

A prompt stores model settings and a reusable template in `prompts/<key>.toml`.
A server function runs it by key. Change and test configurations separately
from the function that calls them.

## Define a prompt

Define a prompt in TOML and push it with `config push`. The `[prompt]` table
carries its identity; the template text lives on a **configuration** alongside
the provider and model:

```toml
# primitive/dev/prompts/summarizer.toml
[prompt]
kind = "chat"
key = "summarizer"
displayName = "Document Summarizer"

[[configs]]
name = "default"
active = true    # exactly one entry carries this — it is the config the server runs
provider = "gemini"
model = "models/gemini-3-flash-preview"

[configs.chat]
temperature = 0.5
systemPrompt = "You are a skilled summarizer. Create clear, accurate summaries."
userPromptTemplate = """
Summarize the following text in a {{ input.style || 'brief' }} style:

{{ input.text }}"""
```

```bash
primitive config push
```

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

Pass values under `variables`; templates read them under `input`.
`configId` selects one of the prompt's configurations (any other id answers
`PROMPT_NO_CONFIG`) and `modelOverride` changes the model for one call. No prompt capability is required; the function’s `access` rule
controls callers.

## Template syntax

```
{{ input.text }}
{{ input.style || 'brief' }}
{{ input.items[0].name }}
```

Missing variables fail rendering. Use a fallback for optional values. Use the
`json` filter when interpolating structured values; do not assemble JSON by
quoting unescaped input.

## Test before activation

Define test cases to validate prompt behavior before you activate a change. A
case is a TOML file beside the prompt, applied by `config push`:

```toml
# prompts/summarizer.tests/basic-test.toml
[test]
name = "basic-test"
inputVariables = '{"text": "Long article text...", "style": "bullet points"}'
expectedOutputContains = '["•"]'
```

```bash
primitive config push --only prompt/summarizer
primitive prompts tests list summarizer
primitive prompts tests run-all summarizer
```

Test commands accept the prompt key or ID. Push new or edited test files
before running them: tests execute the registered server-side cases.

Use `primitive config fields prompt` for test-case fields. Remove a case file
and push with `--prune` to delete it. See
[Test Case Identity](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md).

Verification types include substring `contains`, regex pattern, JSON subset, and LLM-as-judge.

Use `primitive prompts preview` to inspect rendered text without a model call.
Use `primitive prompts execute` for an admin test. `execute` can run a disabled
prompt; a server function cannot.

## Configurations

Each `[[configs]]` entry has a unique `name`, a `provider`, and a `model`.
Set exactly one `active = true`; runs use it unless they specify `configId`.
Use `status = "archived"` to retire a named configuration.

Shared keys are `name`, `status`, `description`, `provider`, `model`,
`providerConfig`, and `active`. `providerConfig` is stored metadata and does
not modify the provider request.

| Chat key | Purpose |
| --- | --- |
| `systemPrompt` | Instructions to the model. |
| `userPromptTemplate` | Template rendered from input variables. |
| `temperature`, `topP` | Sampling settings. |
| `maxTokens` | Output token limit. |
| `outputFormat` | Text or JSON output. |
| `outputSchema` | Configuration-level schema; function result validation uses the prompt-level schema. |
| `reasoningEffort`, `reasoningBudget` | Alternative reasoning controls; choose one. |
| `strictOutput` | OpenRouter strict schema output; requires a compatible model and prompt output schema. |

Chat keys go under `[configs.chat]`. A decisions configuration uses
`[configs.decisions]` with `questions`. The prompt’s `kind` selects the block
and is fixed at creation. Use `primitive config fields prompt` for field types.

## Typed output

When a prompt answers JSON, declare its shape once on the prompt and stop
parsing it by hand. `[prompt.outputSchema]` is sent to the provider, the answer
is validated against it, and `config push` renders it into
`functions/primitive-prompt-types.d.ts` so the function's call is typed from it:

```toml
# primitive/dev/prompts/categorize.toml
[prompt]
key = "categorize"
displayName = "Categorize"

[prompt.outputSchema]
type = "object"
required = ["category"]

[prompt.outputSchema.properties.category]
type = "string"
```

```ts
const answer = await ctx.prompts.run("categorize", {
  variables: { text: input.text },
});
if (!answer.success) throw new Error(answer.error ?? "categorize failed");
// `parsed` is the validated JSON value, typed from the schema above.
return { category: answer.parsed.category };
```

Check `success` before reading `parsed`. Its fields are typed from the prompt’s
output schema.



Push writes `functions/primitive-prompt-types.d.ts` from prompt schemas.
`parsed` is typed from `[prompt.outputSchema]`; without a schema it is unknown.

## Decisions prompts

A decisions prompt sends a state value and named questions to a decisions
model. Its response contains an answer for each question. Use
`kind = "decisions"` with an OpenRouter configuration:

```toml
# primitive/dev/prompts/transaction-categorizer.toml
[prompt]
kind = "decisions"
key = "transaction-categorizer"
displayName = "Transaction categorizer"

[[configs]]
name = "jev"
active = true
provider = "openrouter"            # the only provider a decisions config runs on
model = "typesafe/jev-1.13"

[configs.decisions.questions.category]
type = "choice"
instructions = "Which household category does this bank transaction belong to?"
criteriaSource = "static"          # the options are the table below

[configs.decisions.questions.category.criteria]
gas-and-fuel = "Gas & Fuel (Auto & Transport)"
groceries = "Groceries (Food & Restaurants)"
```

A decisions config has no `userPromptTemplate` and no `systemPrompt`: the
questions ARE the prompt.

### `state` is the input

A decisions run takes one variable, `state` — the value the questions are asked
about. It is passed to the provider verbatim, whatever its JSON shape, and
nothing is templated:

```ts
const answer = await ctx.prompts.run("transaction-categorizer", {
  variables: {
    state: {
      transaction: {
        bank_description: "CHEVRON 0093847",
        amount: -61.2,
        date: "2026-08-11",
      },
    },
  },
});
```

Always supply `variables.state`. It can be any JSON value, including `null`,
`0`, or an empty string. Decisions prompts accept state and question options;
use chat prompts for attachments.

### The answers

`output` is the answers object serialized, and a server function receives it
as `parsed` with no declaration needed — a decisions run always answers JSON.
`metrics` carries `inputTokens`, `outputTokens`, `totalTokens` and
**`metrics.cost`**, the price of the call in USD as the provider reported it
(absent when the provider reports none).

There are three question types. To get `parsed` **typed**, declare the matching
`[prompt.outputSchema]` — typed access comes only from that declaration, never
from the questions, because any active config of the prompt can be pinned by
`configId` and only the prompt-level schema is enforced at run time.

**`choice`** — pick one of the `criteria` keys. `criteria` is a table of at
least two `option key = description` pairs, each description a non-empty string
or a non-empty JSON object, and every `choice` question declares where it comes
from with `criteriaSource`: `"static"` for a table in the config, as above, or
`"dynamic"` for options each run supplies (see
[Options supplied per run](#options-supplied-per-run)).

```json
{ "type": "choice", "choice": "gas-and-fuel",
  "probabilities": { "gas-and-fuel": 1, "groceries": 0 }, "confidence": 1 }
```

```toml
[prompt.outputSchema]
type = "object"

[prompt.outputSchema.properties.category]
type = "object"

[prompt.outputSchema.properties.category.properties.type]
const = "choice"

[prompt.outputSchema.properties.category.properties.choice]
enum = ["gas-and-fuel", "groceries"]

[prompt.outputSchema.properties.category.properties.probabilities]
type = "object"

[prompt.outputSchema.properties.category.properties.confidence]
type = "number"
```

**`score`** — a number between the two ends of a scale. Here the `criteria` is an **array**
of exactly the two endpoint descriptions, low first; the endpoint answers
`400` without it. The answer carries the `legend` it scored against beside the
score:

```json
{ "type": "score", "score": 0.01,
  "legend": { "0": "not unusual at all", "1": "extremely unusual" },
  "probabilities": { "0": 0.99, "1": 0.01 }, "confidence": 0.98 }
```

```toml
[configs.decisions.questions.unusual]
type = "score"
instructions = "How unusual is this transaction for a typical household?"
criteria = ["not unusual at all", "extremely unusual"]
```

**`noul`** — a bare number, with no confidence:

```json
{ "type": "noul", "noul": 0.81 }
```

Any other key on a question is passed through to the provider, which validates
it — except `criteriaSource`, which is the platform's own and is never sent.

### Options supplied per run

Some questions only have options at run time: "which of this household's
previous transactions is the same merchant as this one" has a different set of
candidates for every transaction. Declare such a question
`criteriaSource = "dynamic"`, with no `criteria` table, and pass the options
with each run as `variables.criteria.<question>`, beside `state`:

```toml
# primitive/dev/prompts/precedent-matcher.toml
[prompt]
kind = "decisions"
key = "precedent-matcher"
displayName = "Precedent matcher"

[[configs]]
name = "jev"
active = true
provider = "openrouter"
model = "typesafe/jev-1.13"

[configs.decisions.questions.precedent]
type = "choice"
instructions = "Which previous transaction is the same merchant as this one, if any?"
criteriaSource = "dynamic"         # each run must pass variables.criteria.precedent

[configs.decisions.questions.category]
type = "choice"
instructions = "Which household category does this bank transaction belong to?"
criteriaSource = "static"          # the table below is required; a run may not supply one

[configs.decisions.questions.category.criteria]
gas-and-fuel = "Gas & Fuel (Auto & Transport)"
groceries = "Groceries (Food & Restaurants)"
other = "None of the above"
```

```ts
const answer = await ctx.prompts.run("precedent-matcher", {
  variables: {
    state: { transaction },
    criteria: {
      precedent: {
        p1: { bank_description: "CHEVRON 00938", category: "gas-and-fuel", amount: -42.1, date: "2026-07-30" },
        p2: { bank_description: "SHELL 4411", category: "gas-and-fuel", amount: -50.0, date: "2026-07-12" },
        none: "No previous transaction is the same merchant",
      },
    },
  },
});
```

The config still owns each question's name, type and instructions, so a run
cannot change what is asked — only the options it chooses between. The provider
is asked exactly the config's questions, with the run's table as each
`"dynamic"` question's `criteria`, and the answer comes back typed as for any
`choice` question. The provider's answer is passed through as it came, so a
function that acts on `choice` checks it against the keys it supplied.

A run's options follow the same rule as a config's table — at least two;
option keys non-empty with no leading or trailing whitespace; each description
a non-empty string or a non-empty JSON object — and are checked **before** the
provider call, so a malformed run is never billed. These all answer
`success: false` with `errorCode: "PROMPT_CRITERIA_INVALID"`, no tokens or cost
in `metrics`, and an `error` naming the question and the rule:

- a `"dynamic"` question with no `variables.criteria.<question>`, or a table the rule refuses;
- options for a `"static"` question, for a `score` or `noul` question, or for a name the config does not declare;
- a `variables.criteria` that is not an object;
- a stored `choice` question with no `criteriaSource` — declare it and push again.

`criteriaSource` is required on every `choice` question: push, and the admin
create and update routes, refuse one without it, a `"static"` question without
a table, a `"dynamic"` question with one, and `criteriaSource` on a `score` or
`noul` question, naming the question. On a chat prompt `variables.criteria` is
an ordinary variable (`{{ input.criteria }}`).

### Testing a decisions prompt

A test case carries `state` in its `inputVariables` and asserts on the answers
with `expectedJsonSubset` — the chosen option is what you pin:

```toml
# prompts/transaction-categorizer.tests/chevron.toml
[test]
name = "chevron"
inputVariables = '{"state": {"transaction": {"bank_description": "CHEVRON 0093847", "amount": -61.2, "date": "2026-08-11"}}}'
expectedJsonSubset = '{"category": {"choice": "gas-and-fuel"}}'
```

A case for a `"dynamic"` question supplies the run's options the same way,
beside `state` — a case's `inputVariables` is the run's `variables`:

```toml
# prompts/precedent-matcher.tests/chevron-repeat.toml
[test]
name = "chevron-repeat"
inputVariables = '{"state": {"transaction": {"bank_description": "CHEVRON 0093847", "amount": -61.2, "date": "2026-08-11"}}, "criteria": {"precedent": {"p1": {"bank_description": "CHEVRON 00938", "category": "gas-and-fuel", "amount": -42.1, "date": "2026-07-30"}, "none": "No previous transaction is the same merchant"}}}'
expectedJsonSubset = '{"precedent": {"choice": "p1"}}'
```

A case whose options are refused is recorded as a failed run whose one failed
check names the question and the rule — on `primitive prompts tests run`,
`tests run-all` and `tests batch start` alike — and no provider call is made,
so the run history says why the case failed.

A decisions prompt may not be a test case's **evaluator**: an evaluator prompt
is handed the evaluated input and output as template variables, which a
decisions run cannot read, so an evaluator must be a chat prompt. Naming one is
a `400` at test-case create and update.

## Reasoning budget

Choose `reasoningEffort` or `reasoningBudget` on a named configuration, then
compare its test results and `metrics.reasoningTokens`. Support depends on the
model. A numeric budget may become an effort level on models without native
token budgets; measure actual usage instead of assuming an exact cap.

## Gotchas

- Use the prompt key, not its database ID, in `ctx.prompts.run`.
- Check `success` before reading `parsed`. Shape failures return
  `PROMPT_OUTPUT_NOT_JSON` or `PROMPT_OUTPUT_SCHEMA_VIOLATION`; the model call
  still occurred and may be billed.
- Provider failures include `upstreamStatus` when available. A timeout uses
  `PROMPT_UPSTREAM_TIMEOUT`. Retry transient failures only within the remaining
  function budget.
- Declare `[prompt.outputSchema]` for typed function output. A configuration’s
  schema does not replace that validation contract.
- Removing optional configuration fields clears their values. Omitting a
  `[[configs]]` block does not delete it; archive the configuration explicitly.
- Push test files before running them. Test `inputVariables` and expected-output
  fields are JSON text inside TOML, not native tables.
- Evaluator prompts must be chat prompts. They read the evaluated result as
  `{{ output }}`, not `{{ input.output }}`.

## Related guides

- [Server Functions](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md)
- [Configuration](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md)
