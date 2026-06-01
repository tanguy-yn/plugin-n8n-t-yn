---
name: audit-workflow
description: Diagnoses, debugs, and repairs an existing n8n workflow that is failing, misbehaving, or needs a pre-flight check before going to production. Applies a progressive-zoom inspection pattern (minimal → structure → drill into failing node only) to keep token usage low. Use this skill whenever the user says "this workflow is broken", "why did this fail", "the execution failed", "review this workflow", "fix workflow X", "fix the errors", "audit this workflow", "diagnose workflow Y", "debug my n8n flow", "this workflow keeps erroring", "health check workflow Z", "check why this workflow doesn't work", or "migrate this workflow to current versions". Also activate when the user shares an execution error message, a stack trace from n8n, a screenshot of a failed run, or asks to upgrade a deprecated node. Do NOT use this skill for building new workflows ("build me a workflow", "design", "create from scratch") — that is build-workflow. Do NOT use for deploying validated JSON to an instance ("deploy this", "push to n8n") — that is deploy-workflow. Do NOT use for pure expression-syntax help ("how do I write {{}}") — that is n8n-expression-syntax. Do NOT use for first-time setup of a new n8n instance or initial credential configuration.
---

# Audit and repair an n8n workflow

Use this skill when a workflow is misbehaving, failing in production, or needs a pre-flight check. The objective is to produce a concrete diagnosis and a minimal repair — not to rebuild the workflow from scratch.

## Skills you must compose with

Auditing requires more low-level n8n knowledge than building, because you are diagnosing existing failures. Always reference these:

| When you are about to… | Consult skill |
|---|---|
| Inspect any workflow or node via MCP | `n8n-mcp-tools-expert` (right tool, right mode) |
| Read a validation error or warning | `n8n-validation-expert` ⭐ (this is the main reference for audit work) |
| Diagnose an expression that returns `undefined` or wrong data | `n8n-expression-syntax` (typical issues: `$json` vs `$json.body`, missing `[0]`) |
| Diagnose a Code node that errored | `n8n-code-javascript` (top 5 error patterns cover 62% of bugs) or `n8n-code-python` |
| Check whether a node config is structurally valid | `n8n-node-configuration` (operation-aware, dependency rules) |

The `n8n-validation-expert` skill is doing most of the heavy lifting in audit mode — it has the false-positive catalog that tells you which warnings to ignore. Always reference it.

## Token budget rules — progressive zoom inspection

Audit sessions on heavy workflows blow up the context window if you load the full JSON upfront. Always zoom in step by step:

1. **Identify** → `n8n_list_workflows({ limit: 20, active: true })` (~50 tokens/workflow) or `n8n_get_workflow({ id, mode: 'minimal' })` (~50 tokens). "Is this the right one? Is it active?"
2. **Locate the problem** → `n8n_get_workflow({ id, mode: 'structure' })` (~500-1500 tokens). Topology only, no parameter values. Enough to spot dangling connections, missing branches, structural issues.
3. **Inspect runtime** → `n8n_executions({ action: 'list', workflowId: id, status: 'error', limit: 5 })` (~1k tokens). Only fetch the *specific* failing execution with `action: 'get', id: <execId>` — never list+get-all.
4. **Drill into the failing node only** → `get_node({ nodeType, mode: 'docs' })` or `mode: 'search_properties'` for the property mentioned in the error. Do **not** call `get_node` on every node in the workflow.
5. **Fetch full JSON only to edit** → `n8n_get_workflow({ id, mode: 'full' })` (~5-15k tokens). Last resort, only when you have already identified the change to make.
6. **Apply with `n8n_update_partial_workflow`**, never full replacement, when fewer than ~30% of nodes are touched.

This pattern keeps a typical audit session under 5k tokens vs. 30k+ for a "load everything, then look" approach.

## Triage first

Before touching anything, decide which of the three audit modes applies:

| Symptom | Mode |
|---|---|
| Workflow exists but execution fails or errors at runtime | **Failure mode** — start with executions |
| Workflow has not been run yet, or the user wants a health check | **Pre-flight mode** — start with validation |
| Workflow is deprecated/old and needs upgrading to current node versions | **Migration mode** — start with version analysis |

Ask the user one clarifying question if the symptom is ambiguous (e.g., "Is this currently failing in production, or is it a pre-flight check?").

## Failure mode

### 1. Pull the failing execution

```
n8n_executions({ action: 'list', workflowId: <id>, status: 'error', limit: 5 })
n8n_executions({ action: 'get', id: <executionId> })
```

Identify the failing node and the exact error message. Do not skip this — most n8n failures are property-level, not structural, and you cannot diagnose them without the runtime error.

### 2. Validate the workflow definition

Start with the topology view — that's enough to spot 60% of structural issues without loading parameter values:

```
n8n_get_workflow({ id, mode: 'structure' })
```

Only escalate to `mode: 'full'` if the runtime error points to a parameter value (expression, body, header) that you actually need to inspect. Then validate:

```
validate_workflow({ workflow: <json> })
```

Cross-reference the validator output with the runtime error. If they agree on the failing node, you have your culprit. If they disagree, the issue is environmental (credentials, external service, expression evaluation against actual data).

### 3. Drill into the failing node

```
get_node({ nodeType: <type>, mode: 'docs' })
get_node({ nodeType: <type>, mode: 'search_properties', propertyQuery: '<error keyword>' })
validate_node({ nodeType, config: <currentConfig>, mode: 'full', profile: 'runtime' })
```

### 4. Propose a fix

Show the user the diff (before → after) for the offending node, with a one-line rationale per change. Do not apply the fix yet.

### 5. Apply (with consent)

If the user approves:

```
n8n_update_partial_workflow({ id, operations: [...] })
```

Then re-test:

```
n8n_test_workflow({ id })
n8n_executions({ action: 'list', workflowId: id, limit: 1 })
```

## Pre-flight mode

### 1. Fetch and validate

```
n8n_get_workflow({ id, mode: 'full' })
n8n_validate_workflow({ id })   // server-side, checks credentials and node availability
validate_workflow({ workflow: <json> })   // local, checks structure and AI agent rules
```

### 2. Run autofix in dry-run

```
n8n_autofix_workflow({ id, dryRun: true })
```

If `dryRun` is not supported in the current MCP version, capture the workflow JSON before calling autofix without `dryRun`, so you can diff the changes after.

### 3. Report

Return a short report:

- **Validation issues**: bullet list, severity, remediation
- **Autofix proposals**: each change with a "safe / behavior-changing" tag
- **Recommendation**: ship-as-is, ship-after-fix, or do-not-ship

## Migration mode

Used when a workflow is on old node versions and needs upgrading.

### 1. Identify version drift

For each node in the workflow:

```
get_node({ nodeType, mode: 'versions' })
get_node({ nodeType, mode: 'breaking' })
get_node({ nodeType, mode: 'migrations' })
```

Use `mode: 'compare'` to see exactly what changed between the current and target version.

### 2. Plan the upgrade

Group breaking changes by node. Surface the list to the user before touching the workflow — some breaking changes (auth method swaps, removed properties) need user input.

### 3. Apply with partial updates

Prefer `n8n_update_partial_workflow` per node, never a full replacement. This keeps the diff readable and lets you bisect if something regresses.

### 4. Re-validate and test

Same as failure mode steps 5 onward.

## Universal rules for repair

- **Never edit blind**: always run `validate_node` and `validate_workflow` before pushing a change. The cost is milliseconds; the cost of breaking a production automation at 3 AM is hours.
- **Prefer partial over full updates**: it produces a reviewable diff and limits blast radius.
- **Document autofix changes**: when you apply autofix, list every change explicitly. The user owns the workflow and needs to know what was modified.
- **Stop and ask if a fix changes business logic**: changing a default value, switching auth methods, or altering a connection topology requires user consent — not just validator approval.

## Output format

End every audit session with:

1. **Diagnosis** — one paragraph, plain language
2. **Changes applied** — bullet list, one per change
3. **Verification** — execution result of the post-fix test run
4. **Open issues** — anything you could not auto-resolve, with a recommendation

## What this skill does not do

- Build new workflows — that is `build-workflow`.
- Promote workflows from staging to production — that is `deploy-workflow`.
- Manage n8n credentials, environment variables, or API access — those live outside the MCP server's scope.
