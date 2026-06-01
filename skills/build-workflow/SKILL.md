---
name: build-workflow
description: Composes a new n8n workflow JSON from a user description, orchestrating the templates-first → search → configure → validate sequence to produce a deploy-ready output. Use this skill whenever the user says "build me a workflow that...", "create an n8n workflow for...", "automate X with n8n", "design a workflow that...", "scaffold an n8n automation", "I want a workflow that does X when Y happens", "set up an n8n flow to...", or "make me an n8n workflow". Also activate when the user describes a trigger-and-action sequence (e.g., "when a webhook arrives, send to Slack") or uploads a requirements doc, flow diagram, or sketch and asks for an n8n implementation. Do NOT use this skill for deploying ("deploy this", "push to my instance", "create this on n8n"), debugging existing workflows ("why is this failing", "fix workflow X", "the execution errored"), pure expression syntax questions, or pure node-property questions — those route to deploy-workflow, audit-workflow, n8n-expression-syntax, or n8n-node-configuration respectively. Do NOT use for purely conversational questions about n8n like "what is n8n" or "how does n8n work". The output is a validated workflow JSON; pushing it to the instance is handled by deploy-workflow afterward.
---

# Build a new n8n workflow

Use this skill whenever the user asks to design or scaffold an n8n workflow. The goal is to produce a validated workflow JSON that is ready to deploy. Do **not** try to deploy in this skill — call the `deploy-workflow` skill afterward if the user wants to push to their n8n instance.

## Skills you must compose with

This skill orchestrates; the actual n8n knowledge lives in companion skills. Invoke them at the right step:

| When you are about to… | Consult skill |
|---|---|
| Call any n8n-mcp tool | `n8n-mcp-tools-expert` (always, before the first tool call) |
| Choose the workflow architecture | `n8n-workflow-patterns` (5 proven patterns: webhook, HTTP API, database, AI, scheduled) |
| Configure node parameters | `n8n-node-configuration` (operation-aware, property dependencies) |
| Write or read `{{}}` expressions | `n8n-expression-syntax` (`$json.body` for webhooks!) |
| Add a Code node in JavaScript | `n8n-code-javascript` (`[{json: {...}}]` return format, `$input.all()`) |
| Add a Code node in Python | `n8n-code-python` (no external libs, standard library only) |
| Interpret validation errors | `n8n-validation-expert` (false positives, profiles) |

If you skip these references, you will hit known traps that cost extra tool calls to debug. Reference them inline in your reasoning so the user sees you are using them.

## The five golden rules

These rules come from the n8n-mcp project and prevent the most common failure modes. Follow them strictly.

1. **Documentation first** — Call `tools_documentation()` once at the start of any session you have not used n8n-mcp in yet. It returns the up-to-date best practices for the rest of the toolset.
2. **Parallel execution** — When operations are independent (multiple `search_nodes`, `get_node`, `search_templates`), fire them in a single message. Never serialize independent calls.
3. **Templates first** — Always call `search_templates` before composing a workflow from scratch. There are 2,500+ curated templates; reusing one is faster and more reliable than greenfield.
4. **Multi-level validation** — Validate in three escalating passes: `validate_node(mode='minimal')` → `validate_node(mode='full')` → `validate_workflow`. Never skip the final workflow-level pass.
5. **Never trust defaults** — Default parameter values are the #1 source of runtime failures. Explicitly configure every parameter that affects behavior, even when the default seems obvious.

## Token budget rules (read this before any tool call)

These rules cut typical session cost by 60-80%. Apply them by default, even when not asked.

- **`get_node` defaults to `detail: 'standard'` with `includeExamples: true`** — this is the sweet spot (~2k tokens, top 10-20 properties + real-world configs).
- **Use `detail: 'minimal'`** (~200 tokens) when you already know the node from a previous step in this session and only need a name/version refresher. Do not ask for `minimal` then immediately escalate — pick the right level upfront.
- **Use `detail: 'full'`** (~10k tokens) only as a last resort, when both `standard` and `mode: 'search_properties'` failed to surface the field you need.
- **For one specific property** (auth, pagination, body format, expression syntax), prefer:
  ```
  get_node({ nodeType, mode: 'search_properties', propertyQuery: '<keyword>' })
  ```
  ~500 tokens vs 10k for `full`.
- **`search_templates` must always be filtered** — never call with just `query: '...'`. Always include `complexity`, `requiredService`, `targetAudience`, or `searchMode: 'by_nodes' | 'by_task'`. An unfiltered query routinely returns 50+ results = ~15k tokens of irrelevant payload.
- **`search_nodes` with `includeExamples: true`** — almost always worth it; the examples save the next `get_node` call.
- **AI/LLM prompt content does not need to live in this conversation.** When configuring an OpenAI / Anthropic / LLM node, only the *parameter shape* (model, temperature, message structure) matters here. Tell the user to draft long prompts in a separate conversation and paste the final string in.
- **Validate before assembling, not after.** A failed `validate_workflow` at the end forces a re-fetch + re-edit cycle that doubles tokens.

## Workflow

### 1. Clarify the requirement

Before any tool call, make sure you understand:

- What event triggers the workflow? (webhook, schedule, manual, chat, form, app event)
- What is the desired outcome? (which apps/services are involved, what data flows where)
- Are there branching conditions, loops, or AI components?
- Does the user already have credentials for the apps involved?

If any of these are ambiguous, ask the user one focused clarifying question before proceeding.

### 2. Template discovery (always first)

> **Refer to `n8n-workflow-patterns`** before searching templates — it tells you which of the 5 architectural patterns matches the requirement. Picking the right pattern narrows the template search dramatically.

Run several `search_templates` calls in parallel to cast a wide net:

```
search_templates({ searchMode: 'by_task', task: '<task category>' })
search_templates({ searchMode: 'by_nodes', nodeTypes: ['n8n-nodes-base.<service>'] })
search_templates({ query: '<natural language description>' })
```

Filtering hints:

- Beginners or quick wins: `complexity: 'simple'`, `maxSetupMinutes: 30`
- By role: `targetAudience: 'marketers' | 'developers' | 'analysts'`
- By dependency: `requiredService: 'openai'`

If a template matches within 80% of the requirement, fetch it with `get_template({ templateId, mode: 'full' })` and adapt it. Adaptation is almost always cheaper than greenfield.

### 3. Node discovery (only if no template fits)

Run parallel `search_nodes` calls:

```
search_nodes({ query: '<trigger keyword>', includeExamples: true })
search_nodes({ query: '<action keyword>', includeExamples: true })
```

Use `source: 'verified'` to limit to vetted community nodes when relevant. Use `source: 'community'` only when the user accepts the maintenance risk.

### 4. Configure each node

> **Refer to `n8n-node-configuration`** before setting any parameter — it explains operation-aware required fields, displayOptions visibility rules, and property dependencies (e.g., `sendBody: true` requires `contentType`). Skipping this is the #1 cause of validation loops.
>
> **Refer to `n8n-expression-syntax`** when writing any `{{}}` expression — especially for webhook data which lives at `$json.body`, not `$json`.
>
> **Refer to `n8n-code-javascript`** (or `n8n-code-python`) when adding a Code node — the return format must be `[{json: {...}}]`, not `{json: {...}}`, and `$input.all()` / `$input.first()` patterns differ from regular nodes.

For every node in the planned workflow, call `get_node` in parallel with `detail: 'standard'` and `includeExamples: true`. The examples returned cover most common configurations and remove the need for follow-up calls:

```
get_node({ nodeType: 'nodes-base.<x>', detail: 'standard', includeExamples: true })
```

**Decision tree for node detail:**

- Default → `detail: 'standard'` + `includeExamples: true` (~2k tokens)
- Already configured this exact node earlier in the session → `detail: 'minimal'` (~200 tokens)
- Need ONE specific property (auth, body format, headers, etc.) → `mode: 'search_properties'` (~500 tokens):
  ```
  get_node({ nodeType: 'nodes-base.<x>', mode: 'search_properties', propertyQuery: '<keyword>' })
  ```
- Last resort, schema/full reference required → `detail: 'full'` (~10k tokens). Use only after standard + search_properties failed.

**For complex JSON bodies / expressions** (HTTP Request body, Code node, Set node with deep structure): the example configs returned by `includeExamples: true` are usually enough. If they are not, drill in with `mode: 'search_properties', propertyQuery: 'json'` rather than escalating to `full`.

**For LLM nodes (OpenAI, Anthropic, Gemini, AI Agent, etc.) — placeholder-only rule:**

When configuring any LLM node, do **not** write the real prompt. Use a minimal placeholder and tell the user they will replace it later. The user prefers to do prompt engineering in a separate conversation to avoid bloating the n8n session context.

Concrete pattern:

- **System message** → `"You are a helpful assistant. Replace this with the real system prompt later."`
- **User message / main prompt** → `"{{ $json.prompt }}"` (n8n expression so the prompt comes from the previous node's data)
- **Model** → suggest a sensible default (`gpt-4o-mini`, `claude-haiku-4-5-20251001`, etc.) and flag it as "change if needed"
- **Temperature, max tokens, response format** → use n8n's defaults unless the user explicitly mentions a constraint
- **Tools / function calling** → leave empty unless explicitly requested

Always end the LLM-node configuration with a one-liner like:
> *Prompt placeholder set. Edit the System / User message in n8n once you've drafted the real prompt — no need to redeploy the workflow.*

Rationale: writing real prompts here costs 5-10k tokens of context that the user does not want to spend during workflow design. The prompt engineering happens elsewhere.

### 5. Validate before assembling

> **Refer to `n8n-validation-expert`** when interpreting any error or warning returned. It distinguishes false positives (skip them) from real errors (fix them), and explains which validation profile to use at which stage.

For each configured node:

```
validate_node({ nodeType, config, mode: 'minimal' })   // <100ms required-field check
validate_node({ nodeType, config, mode: 'full', profile: 'runtime' })  // catches expression and dependency errors
```

Profiles: `minimal`, `runtime` (default for production), `ai-friendly` (relaxed for prototyping), `strict` (CI / governance).

### 6. Assemble and validate the workflow

Compose the full workflow JSON with nodes and connections, then:

```
validate_workflow({ workflow: <fullJson> })
```

This catches whole-graph issues that node-level validation cannot: AI agent missing language model, dangling connections, streaming-mode constraints, missing memory or output parsers.

If the user is building an AI Agent workflow, this step is non-negotiable — the AI-specific validation lives only at the workflow level.

### 7. Hand off

Present the validated workflow JSON to the user with:

- A one-paragraph description of what it does
- The list of credentials they will need to provision in n8n
- A note on whether to run the `deploy-workflow` skill next, or if they want to import the JSON manually

## Output format

Return the final workflow as a fenced JSON block, preceded by a short summary. Do not return the workflow until `validate_workflow` has passed cleanly.

## When things go wrong

- **`search_templates` returns nothing useful**: drop to step 3 (node discovery). Do not invent a template.
- **`validate_node` keeps failing on the same field**: use `get_node` with `mode: 'search_properties'` to find the canonical property name — n8n property names are not always intuitive.
- **`validate_workflow` reports a connection error**: re-read the connections object. n8n connections are keyed by source node *name*, not type, and are case-sensitive.
- **The user asks for something that requires a community node**: check `source: 'verified'` first. If only `source: 'community'` matches, surface the maintenance risk before adopting.

## What this skill does not do

- It does not deploy workflows to an n8n instance — that is `deploy-workflow`.
- It does not debug or repair existing workflows — that is `audit-workflow`.
- It does not manage credentials, only describes which ones the user will need.
