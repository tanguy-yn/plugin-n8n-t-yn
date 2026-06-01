# n8n-mcp-builder

A self-contained Cowork plugin for designing, deploying, and auditing n8n workflows with Claude. Bundles the [n8n-mcp](https://github.com/czlonkowski/n8n-mcp) server (24 tools) and **10 complementary skills** that cover both high-level orchestration and low-level n8n knowledge.

## What's inside

### MCP server

| Component | Purpose |
|---|---|
| `n8n-mcp` server (via wrapper) | All 24 tools — 7 documentation (`search_nodes`, `get_node`, `validate_node`, `validate_workflow`, `search_templates`, `get_template`, `tools_documentation`) + 17 management (`n8n_create_workflow`, `n8n_update_partial_workflow`, `n8n_test_workflow`, `n8n_executions`, `n8n_autofix_workflow`, etc.) |

### Orchestration skills (3) — written for this plugin

| Skill | Purpose |
|---|---|
| `build-workflow` | Compose a new workflow. Templates-first → search → configure → validate, with strict token-budget rules. |
| `deploy-workflow` | Push a validated workflow to your n8n instance with progressive-zoom inspection and partial-update preference. |
| `audit-workflow` | Diagnose a failing workflow, run a pre-flight check, or migrate to current node versions. |

### Knowledge skills (7) — vendored from [czlonkowski/n8n-skills](https://github.com/czlonkowski/n8n-skills) (MIT)

| Skill | Triggers when |
|---|---|
| `n8n-mcp-tools-expert` ⭐ | Any n8n-mcp tool call — selects the right tool, validates parameter format |
| `n8n-workflow-patterns` | Designing workflow structure (5 proven patterns: webhook, HTTP API, database, AI, scheduled) |
| `n8n-node-configuration` | Configuring node properties — explains operation-aware dependencies (e.g., `sendBody` → `contentType`) |
| `n8n-expression-syntax` | Writing `{{}}` expressions, `$json`, `$node`, `$now`. **Critical:** webhook data lives at `$json.body` |
| `n8n-validation-expert` | Interpreting validation errors, distinguishing false positives, picking the right validation profile |
| `n8n-code-javascript` | Writing JS in Code nodes — `$input.all()`, `[{json: {...}}]` return format, top 5 errors covering 62% of bugs |
| `n8n-code-python` | Writing Python in Code nodes — limitations (no `pandas`/`requests`), workarounds, standard library patterns |

The orchestration and knowledge skills compose naturally: when you ask for a complex workflow, the build-workflow skill orchestrates the high-level flow while the knowledge skills load on demand to handle the specific configuration, expression, or code-node details.

## How credentials work

This plugin does **not** ship credentials inside the `.plugin` file. Instead, the bundled launcher script reads them from a local file in your home directory: `~/.n8n-mcp.env`. You create that file once; the plugin reads it every time the MCP server starts.

**Trade-off:** clean separation between plugin (shareable) and credentials (private). One-time setup of ~30 seconds.

## Setup — one-time

### 1. Create the credentials file

```bash
cat > ~/.n8n-mcp.env <<'EOF'
N8N_API_URL="https://your-n8n.example.com/api/v1"
N8N_API_KEY="n8n_api_xxxxxxxxxxxxxxxxxxxx"
EOF

chmod 600 ~/.n8n-mcp.env   # owner-only read/write
```

Replace the URL and key with your own. The URL must include `/api/v1` at the end. Get the API key from your n8n instance: **Settings → API → Create new API key**, with at least `workflow:read`, `workflow:write`, `execution:read` scopes.

### 2. Remove any duplicate `n8n-mcp` entry from `claude_desktop_config.json`

If you previously installed `n8n-mcp` directly in Claude Desktop, remove that block now to avoid running two instances:

```bash
open ~/Library/Application\ Support/Claude/claude_desktop_config.json
```

Find the `"n8n-mcp": { ... }` block inside `mcpServers` and delete it (keeping the JSON valid). If it was the only MCP server, the result will be `"mcpServers": {}`.

### 3. Quit and relaunch Cowork

Fully quit (`⌘ + Q`) so the old config is unloaded, then reopen.

### 4. Install this plugin

Drag `n8n-mcp-builder.plugin` onto the Cowork window.

### 5. Verify

Ask Claude:

> *Run an n8n health check*

You should see all 24 tools available under `mcp__plugin_n8n-mcp-builder_n8n-mcp__*` and the health check report your instance as reachable.

## Updating credentials later

Edit `~/.n8n-mcp.env`, save, and either restart the MCP from Cowork's plugin settings or quit/reopen Cowork. No re-install needed.

## Usage examples

**Build:**
> *Build me an n8n workflow that watches a Gmail label, summarizes new emails with OpenAI, and posts the summary to a Slack channel.*

The `build-workflow` skill triggers, runs `search_templates` (filtered) first, then drills into nodes with `detail: 'standard'` + examples. LLM nodes are configured with placeholder prompts only — you'll edit them in n8n later.

**Deploy:**
> *Deploy this workflow to my n8n instance and run a test.*

The `deploy-workflow` skill runs `n8n_health_check`, prefers `n8n_update_partial_workflow` over full replacement, and verifies the post-deploy execution.

**Audit:**
> *Workflow 42 is failing in production — diagnose it.*

The `audit-workflow` skill applies progressive zoom: `mode: 'minimal'` → `mode: 'structure'` → drill into the failing node only, never loads the full workflow JSON unless strictly needed.

## Token budget at a glance

The skills enforce these patterns by default:

- `get_node` defaults to `detail: 'standard'` + `includeExamples: true` (~2k tokens), never `'full'` unless `'standard'` and `mode: 'search_properties'` both fail.
- `search_templates` is always filtered (`searchMode: 'by_task' | 'by_nodes' | 'by_metadata'`), never raw query.
- `validate_node(mode='minimal')` runs first, `'full'` only if minimal passes.
- `n8n_get_workflow` defaults to `mode: 'structure'` for inspection, `'full'` only when editing parameter values.
- `n8n_update_partial_workflow` is preferred for any change touching < 30% of nodes.
- LLM node prompts are placeholders — actual prompt engineering happens in a separate session, no token waste here.

## Troubleshooting

**Only 7 tools appear (no `n8n_*` management tools):**
- `~/.n8n-mcp.env` is missing, malformed, or has wrong permissions. Check the file exists and contains both `N8N_API_URL` and `N8N_API_KEY` on separate lines, with quoted values.
- Try running the launcher manually to test: `bash ~/.n8n-mcp.env && env | grep N8N` — should print your URL and key.

**`n8n_health_check` fails:**
- API URL probably missing `/api/v1` at the end.
- API key might lack scopes — regenerate with full read/write scopes.
- Self-hosted n8n behind firewall — verify the URL is reachable from your machine.

**"command not found: npx":**
- Node.js is not in the PATH that the plugin sees. Install Node.js from [nodejs.org](https://nodejs.org), restart Cowork, or edit `scripts/launch.sh` to use the full path: `exec /usr/local/bin/npx -y n8n-mcp`.

## Credits

- **MCP server**: [czlonkowski/n8n-mcp](https://github.com/czlonkowski/n8n-mcp) by Romuald Czlonkowski (bundled via `npx`, runs locally)
- **Knowledge skills (7)**: [czlonkowski/n8n-skills](https://github.com/czlonkowski/n8n-skills) by Romuald Czlonkowski, vendored under MIT license — see `LICENSE-n8n-skills`
- **Orchestration skills (3)**: original to this plugin
- Plugin scaffolding: built with Cowork's `create-cowork-plugin` skill
