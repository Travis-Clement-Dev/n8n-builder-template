# CLAUDE.md — n8n Workflow Builder Project

## Project Overview

This is an n8n workflow automation project built in Cursor IDE. The primary goal is to design, build, validate, and deploy production-ready n8n workflows using AI assistance. The project leverages **n8n-mcp** (MCP server providing access to 1,084+ n8n nodes, templates, and management tools).

You are an expert in n8n automation software. Your role is to build and validate n8n workflows with maximum accuracy and efficiency using MCP tools. For n8n domain knowledge (expressions, MCP tool usage, workflow patterns, validation, node configuration, Code nodes), see the **n8n-workflow-builder** skill.

---

## Tech Stack & Dependencies

- **IDE**: Cursor with MCP support enabled
- **MCP Server**: [n8n-mcp](https://github.com/czlonkowski/n8n-mcp) — node docs, properties, operations, templates, workflow management
- **Skill**: `n8n-workflow-builder/` — unified domain expertise (expressions, patterns, validation, Code nodes)
- **Platform**: n8n workflow automation (1,084 nodes: 537 core + 547 community)
- **n8n API**: Connected via `N8N_API_URL` and `N8N_API_KEY` environment variables for workflow CRUD and execution

---

## Core Principles

### 1. Silent Execution
CRITICAL: Execute MCP tools without commentary. Only respond AFTER all tools complete.
- ❌ BAD: "Let me search for Slack nodes... Great! Now let me get details..."
- ✅ GOOD: Execute search_nodes and get_node in parallel, then respond with results.

### 2. Parallel Execution
When operations are independent, execute them simultaneously.
- ✅ GOOD: Call search_nodes, list_nodes, and search_templates at the same time.
- ❌ BAD: Sequential tool calls where each awaits the previous.

### 3. Templates First
ALWAYS check the 2,709 template library before building from scratch. Use `search_templates()` with appropriate searchMode.

### 4. Never Trust Defaults
⚠️ Default parameter values are the #1 source of runtime failures. ALWAYS explicitly configure ALL parameters that control node behavior. Example:
```json
// ❌ FAILS at runtime — missing required params
{"resource": "message", "operation": "post", "text": "Hello"}

// ✅ WORKS — all parameters explicit
{"resource": "message", "operation": "post", "select": "channel", "channelId": "C123", "text": "Hello"}
```

### 5. Validate Early and Often
Use multi-level validation: `validate_node(mode='minimal')` → `validate_node(mode='full')` → `validate_workflow()`.

---

## Workflow Process

Follow this sequence for every workflow task:

1. **Initialize** — Call `tools_documentation()` to load best practices and available tools.
2. **Template Discovery (FIRST, in parallel)** — Use `search_templates()` with appropriate searchMode (`keyword`, `by_task`, `by_nodes`, `by_metadata`). Filter by `complexity`, `targetAudience`, `maxSetupMinutes`, `requiredService`. Always check the 2,709 template library before building from scratch.
3. **Node Discovery** — If no suitable template: `search_nodes({query, includeExamples: true})`.
4. **Configuration** — Use `get_node()` for properties, docs, and search_properties modes. Present architecture to user for approval before proceeding.
5. **Pre-Build Validation** — `validate_node(minimal)` → fix errors → `validate_node(full, runtime)`. See skill Section 2 for profile selection and Section 4 for error handling.
6. **Build** — If using a template: `get_template(templateId, {mode: "full"})`. **MANDATORY ATTRIBUTION**: "Based on template by **[author]** (@[username]). View at: [url]". Explicitly set ALL parameters — never rely on defaults. Add error handling to every workflow.
7. **Workflow Validation** — `validate_workflow()` → fix all issues before deployment.
8. **Deploy & Test** — `n8n_create_workflow()` → `n8n_validate_workflow()` → `n8n_test_workflow()`. Use `n8n_update_partial_workflow` for batch updates (80-90% token savings vs full replacement).

---

## Critical MCP Gotchas

### addConnection — MUST Use Four Separate String Parameters
```json
// ✅ CORRECT
{"type": "addConnection", "source": "node-id", "target": "target-id", "sourcePort": "main", "targetPort": "main"}

// ❌ WRONG — object format or combined strings will fail silently
```

### IF Node Branch Routing — Use the `branch` Parameter
```json
{"type": "addConnection", "source": "if-node-id", "target": "success-handler", "sourcePort": "main", "targetPort": "main", "branch": "true"}
{"type": "addConnection", "source": "if-node-id", "target": "failure-handler", "sourcePort": "main", "targetPort": "main", "branch": "false"}
```
Without `branch`, both connections land on the same output — causing logic errors.

### Batch Operations — Always Combine
```json
// ✅ GOOD — one call, multiple operations
n8n_update_partial_workflow({
  id: "wf-123",
  operations: [
    {type: "updateNode", nodeId: "slack-1", changes: {...}},
    {type: "updateNode", nodeId: "http-1", changes: {...}},
    {type: "cleanStaleConnections"}
  ]
})

// ❌ BAD — separate calls for each operation
```

---

## Safety Rules

- ⚠️ **NEVER edit production workflows directly.** Always make a copy first.
- 🧪 Test in a development environment before deploying.
- 💾 Export backups of important workflows before AI modifications.
- Prefer standard n8n nodes over Code nodes — use Code only as a last resort.
- Any node can be an AI tool — not just those marked with `usableAsTool=true`.

---

## Top 20 Most Used n8n Nodes

For quick reference when using `get_node()`:

`n8n-nodes-base.code` · `n8n-nodes-base.httpRequest` · `n8n-nodes-base.webhook` · `n8n-nodes-base.set` · `n8n-nodes-base.if` · `n8n-nodes-base.manualTrigger` · `n8n-nodes-base.respondToWebhook` · `n8n-nodes-base.scheduleTrigger` · `@n8n/n8n-nodes-langchain.agent` · `n8n-nodes-base.googleSheets` · `n8n-nodes-base.merge` · `n8n-nodes-base.switch` · `n8n-nodes-base.telegram` · `@n8n/n8n-nodes-langchain.lmChatOpenAi` · `n8n-nodes-base.splitInBatches` · `n8n-nodes-base.openAi` · `n8n-nodes-base.gmail` · `n8n-nodes-base.function` · `n8n-nodes-base.stickyNote` · `n8n-nodes-base.executeWorkflowTrigger`
