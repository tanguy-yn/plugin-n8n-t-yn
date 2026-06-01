---
name: deploy-workflow
description: Pushes a validated n8n workflow JSON to the user's live n8n instance via API and verifies it with a post-deploy test execution. Use this skill whenever the user says "deploy this workflow", "push to n8n", "push to my instance", "create this on n8n", "update workflow X", "send this to my n8n", "install this workflow on my instance", "let's put this live", "make this active", "run this workflow on n8n", or "ship this to production". Also activate when a workflow JSON has just been validated by build-workflow and the user asks to put it in production, or when the user provides a workflow ID and asks to update/replace it. Do NOT use this skill for composing a new workflow from scratch ("build me a workflow", "design", "create from a description") — that is build-workflow. Do NOT use for diagnosing failures or repairing broken workflows ("why is this failing", "debug", "fix the errors") — that is audit-workflow. Do NOT use for inspecting workflows without modifying them — call n8n_get_workflow or n8n_list_workflows directly. Always verify N8N_API_URL and N8N_API_KEY are configured (via n8n_health_check) before any deploy step.
---

# Deploy a workflow to n8n

Use this skill when a validated workflow JSON needs to land in the user's n8n instance. This skill assumes the workflow has already been validated (either by the `build-workflow` skill or by an explicit `validate_workflow` call). Do not deploy unvalidated JSON.

## Skills you must compose with

| When you are about to… | Consult skill |
|---|---|
| Call any n8n_* management tool | `n8n-mcp-tools-expert` (parameter formats, common gotchas) |
| Interpret a validation result from `n8n_validate_workflow` | `n8n-validation-expert` (false positives vs real errors) |
| Decide between full and partial update operations | `n8n-mcp-tools-expert` (operation format for `n8n_update_partial_workflow`) |

## Token budget rules

Read this before any tool call.

- **Updates: partial > full, always.** `n8n_update_partial_workflow` sends a 200-token diff; `n8n_update_full_workflow` re-uploads 5-15k tokens of JSON. Use `partial` for any change touching less than ~30% of nodes.
- **Fetching for inspection: pick the right mode.** `n8n_get_workflow` defaults to `mode: 'full'` (~5-15k tokens). Use `mode: 'minimal'` (~50 tokens) for "is it active?", `mode: 'structure'` (~500-1500 tokens) for "what's the topology?", `mode: 'details'` (~2k tokens) for "+ execution stats". Reserve `full` for when you actually need the parameter values to edit them.
- **Listing workflows: always paginate and filter.** `n8n_list_workflows({ limit: 20, active: true })` instead of unfiltered listing.
- **Test execution inspection: minimal first.** `n8n_executions({ action: 'list', limit: 1 })` returns a summary; only call `action: 'get'` on the specific execution ID you actually need to inspect.

## Prerequisites check

Before any deploy step, verify that the n8n management tools are usable:

```
n8n_health_check()
```

If this fails, stop and surface the error to the user. The most common causes:

- `N8N_API_URL` or `N8N_API_KEY` env vars are not set in the plugin's MCP config
- The n8n instance is not reachable from the user's machine
- The API key lacks the required scopes (workflow:read, workflow:write, execution:read)

Do not attempt any other deploy step until `n8n_health_check` returns green.

## Decide: create or update?

Ask the user (if not already clear) whether this is a new workflow or an update to an existing one. If updating, get the workflow ID. Use `n8n_list_workflows` to help them locate it if needed:

```
n8n_list_workflows({ limit: 50, active: true })
```

## Path A — create a new workflow

1. Final validation pass:

   ```
   validate_workflow({ workflow: <json> })
   ```

   Even if `build-workflow` already validated, re-run here. It is cheap and catches drift.

2. Create:

   ```
   n8n_create_workflow({ name, nodes, connections, settings })
   ```

3. Validate the created workflow on the n8n side (this is different from local validation — it checks credentials and node availability on the actual instance):

   ```
   n8n_validate_workflow({ id: <newId> })
   ```

4. If `n8n_validate_workflow` reports issues, run autofix:

   ```
   n8n_autofix_workflow({ id: <newId> })
   ```

   Review what autofix changed before accepting. Some fixes are safe (typos, deprecated property renames). Others change behavior (default credential swaps, version bumps) and should be confirmed with the user.

## Path B — update an existing workflow

Prefer **partial updates** when possible. They are safer and produce a clean diff.

1. Fetch the current workflow at the right zoom level. Default to `mode: 'structure'` (topology only, ~500-1500 tokens) — that is enough to plan most edits. Only escalate to `mode: 'full'` if you need to read existing parameter values:

   ```
   n8n_get_workflow({ id, mode: 'structure' })   // default for planning edits
   n8n_get_workflow({ id, mode: 'full' })        // only when you need parameter values
   ```

2. If the change is small (one node config, one connection), use:

   ```
   n8n_update_partial_workflow({ id, operations: [...] })
   ```

3. If the change is large or structural, use:

   ```
   n8n_update_full_workflow({ id, workflow: <json> })
   ```

   Warn the user that this is a complete replacement — anything not in the new JSON disappears.

4. Re-validate on n8n side:

   ```
   n8n_validate_workflow({ id })
   ```

## Test the deployed workflow

Before declaring done, run a real test. The MCP server auto-detects the trigger type:

```
n8n_test_workflow({ id, data: <optional sample payload> })
```

For webhook triggers, you can pass `headers` and a custom HTTP method. For chat triggers, pass `message` and `sessionId`. For form/manual triggers, pass `data`.

Then inspect the execution:

```
n8n_executions({ action: 'list', workflowId: id, limit: 1 })
n8n_executions({ action: 'get', id: <executionId> })
```

If the execution failed, surface the failing node and the error message to the user. Do not try to "fix in place" — hand off to the `audit-workflow` skill if repair is needed.

## Versioning

After a successful deploy, mention `n8n_workflow_versions` to the user. They can use it to roll back if a future change breaks something:

```
n8n_workflow_versions({ id, action: 'list' })
n8n_workflow_versions({ id, action: 'rollback', versionId })
```

## Templates from n8n.io

If the user wants to deploy a public template directly (rather than a custom JSON), use:

```
n8n_deploy_template({ templateId, autoFix: true })
```

This pulls from n8n.io and runs autofix on the way in.

## Output format

After a successful deploy, return:

- The new workflow ID and the n8n URL to view it (construct as `<N8N_API_URL minus /api/v1>/workflow/<id>`)
- Whether it was activated or left inactive (defaults are inactive — confirm with the user before activating production workflows)
- The result of the test execution
- Any autofix changes that were applied, listed explicitly

## What this skill does not do

- Compose new workflows from a description — that is `build-workflow`.
- Diagnose failing executions in depth — that is `audit-workflow`.
- Manage n8n credentials. The MCP server does not expose credential management; the user must provision credentials manually in the n8n UI before workflows can run.
