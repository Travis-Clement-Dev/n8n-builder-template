---
name: n8n-workflow-builder
description: Expert n8n workflow automation skill covering expression syntax, MCP tool usage, workflow patterns, validation, node configuration, and Code node development. Use when building n8n workflows, writing n8n expressions with {{}} or $json/$node/$now/$env variables, searching for n8n nodes, validating n8n configurations, using n8n-mcp MCP tools, creating webhook/HTTP/database/AI agent/scheduled workflows, debugging n8n validation errors or false positives, configuring n8n node properties and dependencies, writing JavaScript or Python in n8n Code nodes, using $input/$helpers in Code nodes, or troubleshooting n8n expression errors.
---

# n8n Workflow Builder

Unified expert guidance for building production-ready n8n workflows. Consolidates expression syntax, MCP tool usage, 5 workflow patterns, validation, node configuration, and Code node development.

---

## 1. Expression Syntax

### Core Syntax

All n8n expressions use double curly braces: `{{ expression }}`.

| Variable | Purpose | Example |
|---|---|---|
| `$json` | Current item data | `{{ $json.email }}` |
| `$json.body` | **Webhook payload** (NOT `$json` directly) | `{{ $json.body.name }}` |
| `$node["Name"].json` | Data from specific node | `{{ $node["HTTP Request"].json.id }}` |
| `$now` | Current datetime (Luxon) | `{{ $now.toISO() }}` |
| `$today` | Start of today | `{{ $today.toISO() }}` |
| `$env` | Environment variables | `{{ $env.API_KEY }}` |
| `$execution.id` | Current execution ID | `{{ $execution.id }}` |
| `$workflow.id` | Workflow ID | `{{ $workflow.id }}` |
| `$('Node')` | Function form of $node | `{{ $('Webhook').item.json.body }}` |
| `$input.all()` | All input items | Used in Code nodes only |
| `$runIndex` | Current run index | `{{ $runIndex }}` |

### Critical Expression Rules

1. **Webhook data is ALWAYS nested under `$json.body`** — this is the #1 expression mistake:
   ```
   ❌ {{ $json.email }}           → undefined for webhook data
   ✅ {{ $json.body.email }}      → correct webhook access
   ```

2. **Dot notation for simple paths, bracket for special characters**:
   ```
   ✅ {{ $json.user.name }}
   ✅ {{ $json["first-name"] }}   → hyphens require brackets
   ✅ {{ $json["my field"] }}     → spaces require brackets
   ```

3. **Ternary for conditional values**:
   ```
   {{ $json.status === "active" ? "Yes" : "No" }}
   ```

4. **Optional chaining to prevent errors**:
   ```
   {{ $json.user?.address?.city ?? "Unknown" }}
   ```

5. **Date manipulation with Luxon** (built-in):
   ```
   {{ $now.minus({days: 7}).toISO() }}
   {{ DateTime.fromISO($json.date).toFormat("yyyy-MM-dd") }}
   ```

### Top 5 Expression Mistakes

| # | Mistake | Fix |
|---|---|---|
| 1 | Accessing webhook data as `$json.field` | Use `$json.body.field` |
| 2 | Using `$json` in Code nodes | Use `$input.first().json` or `$input.all()` |
| 3 | Missing `{{ }}` wrapper | Wrap ALL expressions: `{{ $json.name }}` |
| 4 | String concatenation without template | Use `{{ "Hello " + $json.name }}` |
| 5 | Date comparison as strings | Parse with `DateTime.fromISO()` first |

---

## 2. MCP Tool Selection Guide

### 7 Core MCP Tools — Quick Reference

| Tool | When to Use | Key Parameters |
|---|---|---|
| `tools_documentation` | **START HERE** — load best practices | None |
| `search_nodes` | Find nodes by keyword | `query`, `includeExamples: true`, `source` |
| `get_node` | Get node details | `nodeType`, `detail`, `mode`, `includeExamples` |
| `validate_node` | Validate node config | `nodeType`, `config`, `mode`, `profile` |
| `validate_workflow` | Validate complete workflow | `workflow` JSON |
| `search_templates` | Find workflow templates | `searchMode`, `query`, filters |
| `get_template` | Get full template JSON | `templateId`, `mode` |

### nodeType Format — CRITICAL

```
Core nodes:      n8n-nodes-base.slack
                 n8n-nodes-base.httpRequest
                 n8n-nodes-base.webhook

LangChain/AI:    @n8n/n8n-nodes-langchain.agent
                 @n8n/n8n-nodes-langchain.lmChatOpenAi
                 @n8n/n8n-nodes-langchain.memoryBufferWindow

Community:       n8n-nodes-community.customNode
```

⚠️ Search tools use short format (`nodes-base.slack`), workflow tools use full format (`n8n-nodes-base.slack`). The MCP server auto-sanitizes, but always use full format in workflow JSON.

### search_nodes Best Practices

```
search_nodes({query: "slack", includeExamples: true})
search_nodes({query: "AI agent", source: "verified"})
search_nodes({query: "database postgres"})
```

- Always set `includeExamples: true` for configuration examples
- Use `source: "verified"` to filter to trusted community nodes
- Keep queries to 1-3 keywords for best results

### get_node Modes and Detail Levels

| Mode | Detail | Use Case |
|---|---|---|
| `info` | `minimal` | Quick check — name, type, description |
| `info` | `standard` | **Default** — essential properties + operations |
| `info` | `full` | Everything — all properties, all versions |
| `docs` | — | Markdown documentation |
| `search_properties` | — | Find specific property by name |
| `versions` | — | Version history and changes |

### validate_node Profiles

| Profile | Speed | Use Case |
|---|---|---|
| `minimal` | <100ms | Quick required-field check before building |
| `runtime` | ~500ms | Full validation matching n8n runtime behavior |
| `ai-friendly` | ~500ms | AI workflow specific (LangChain connections) |
| `strict` | ~1s | Maximum validation — all rules enforced |

Validation sequence: `minimal` first → fix errors → `runtime` or `ai-friendly` → fix → `validate_workflow`.

### search_templates searchModes

| Mode | When to Use | Example |
|---|---|---|
| `keyword` | General text search | `query: "slack notification"` |
| `by_task` | Task-based discovery | `task: "webhook_processing"` |
| `by_nodes` | Find templates using specific nodes | `nodeTypes: ["n8n-nodes-base.slack"]` |
| `by_metadata` | Filter by complexity/audience | `complexity: "simple"` |

Filters: `complexity`, `targetAudience`, `maxSetupMinutes`, `requiredService`.

---

## 3. Workflow Patterns

Five proven architectural patterns for n8n workflows. Select based on trigger type and data flow.

### Pattern Selection

| Trigger | Data Flow | Pattern |
|---|---|---|
| External HTTP request | Receive → Process → Respond | **Webhook Processing** |
| Timer or cron | Fetch → Transform → Deliver | **Scheduled Tasks** |
| Internal trigger | Call API → Handle errors → Process | **HTTP API Integration** |
| Data sync needed | Query → Transform → Write → Verify | **Database Operations** |
| AI/LLM needed | Agent → Tools → Memory → Output | **AI Agent Workflow** |

### Pattern 1: Webhook Processing

```
Webhook → Validate Input → Transform Data → Business Logic → Respond to Webhook
                ↓ (on error)
          Error Response (4xx)
```

Key rules:
- Always validate incoming data before processing
- Always respond with `Respond to Webhook` node — timeouts cause client errors
- Access payload via `$json.body`, headers via `$json.headers`
- Add authentication (header check or HMAC) for production webhooks

### Pattern 2: HTTP API Integration

```
Trigger → Set Auth Headers → HTTP Request → IF (status check) → Process Response
                                                    ↓ (on error)
                                              Retry / Error Handler
```

Key rules:
- Set `continueOnFail: true` on HTTP Request to handle errors gracefully
- Check response status in IF node before processing
- Implement retry logic for transient failures (429, 503)
- Store credentials in n8n Credentials, reference via credential ID

### Pattern 3: Database Operations

```
Trigger → Query Source → Transform/Map Fields → Upsert Target → Verify Write
```

Key rules:
- Use `upsert` over insert/update to handle both cases
- Map fields explicitly in a Set node — never pass raw data
- Add a verification query after writes for critical operations
- Use `splitInBatches` for large datasets (>1000 records)

### Pattern 4: AI Agent Workflow

```
Trigger → Agent → [Tools: HTTP, Code, etc.] → Memory → Output Parser → Response
              ↑        ↓
         LM Chat    ai_tool connections (NOT main)
```

Key rules:
- LangChain nodes use `@n8n/n8n-nodes-langchain.*` nodeType format
- Agent connections use `ai_agent`, `ai_tool`, `ai_memory`, `ai_outputParser` — NOT `main`
- Use `validate_node(profile: 'ai-friendly')` for AI nodes
- Any node can be an AI tool — not just those marked `usableAsTool: true`
- Connect LLM to agent via `ai_languageModel` connection type

### Pattern 5: Scheduled Tasks

```
Schedule Trigger → Fetch Data → Process/Transform → Notify/Store → Log Result
```

Key rules:
- Use `scheduleTrigger` node with cron expressions
- Add error notification (email/Slack) for failed runs
- Include execution metadata in logs (`$execution.id`, `$now`)
- Use `IF` node to skip processing when no new data exists

**For full pattern implementations with node configurations, see [references/WORKFLOW_PATTERNS.md](references/WORKFLOW_PATTERNS.md).**

---

## 4. Validation & Error Handling

### Validation Loop

Expect 2-3 validation iterations per workflow. This is normal.

```
Build Node Config → validate_node(minimal) → Fix Errors
                                                  ↓
                    validate_node(full, runtime) ← ┘
                              ↓
                    Build Workflow JSON
                              ↓
                    validate_workflow → Fix Errors → Re-validate
                              ↓
                    Deploy (n8n_create_workflow)
```

### Top 9 Error Types — Quick Fix Guide

| Error | Cause | Fix |
|---|---|---|
| `MISSING_REQUIRED` | Required property not set | Set the property explicitly — never rely on defaults |
| `INVALID_TYPE` | Wrong value type | Check expected type with `get_node(mode: 'search_properties')` |
| `UNKNOWN_PROPERTY` | Property doesn't exist on node | Verify property name in node docs |
| `INVALID_EXPRESSION` | Malformed `{{ }}` expression | Check syntax — common: missing brackets, typos in `$json` |
| `MISSING_CONNECTION` | Node has no input connection | Add connection from previous node |
| `CIRCULAR_DEPENDENCY` | Workflow contains a loop | Restructure to break circular reference |
| `INVALID_CREDENTIAL` | Credential not found or wrong type | Verify credential ID and type match node requirements |
| `DUPLICATE_NODE_NAME` | Two nodes share a name | Rename to unique names |
| `INVALID_AI_CONNECTION` | Wrong connection type for AI node | Use `ai_tool`/`ai_memory`/`ai_languageModel`, not `main` |

### False Positive Patterns — When to Ignore

These validation warnings can be safely ignored:

1. **Optional property warnings** — Properties with defaults don't need explicit values unless overriding
2. **Credential warnings in development** — Credentials are environment-specific; validate only on deployment
3. **Expression validation on dynamic values** — Expressions referencing runtime data can't be statically validated
4. **Community node warnings** — Community nodes may have incomplete validation schemas
5. **Deprecated property warnings** — Still functional; update when convenient
6. **AI tool flexibility warnings** — Nodes marked as non-tool can still be used as tools

**For complete error catalog with fix procedures, see [references/ERROR_CATALOG.md](references/ERROR_CATALOG.md).**

---

## 5. Node Configuration

### Operation-Aware Configuration

n8n nodes reveal different required fields based on the selected `resource` and `operation`. Always configure in this order:

```
1. Set resource    → reveals operation options
2. Set operation   → reveals required properties
3. Set required    → reveals optional properties
4. Set optional    → complete configuration
```

Example — Slack node:
```json
// Step 1-2: Resource + Operation determine required fields
{"resource": "message", "operation": "post"}

// Step 3: Required fields for message.post
{"select": "channel", "channelId": "C123", "text": "Hello"}

// Step 4: Optional enhancements
{"jsonParameters": true, "attachments": [...]}
```

### Property Dependency Chains

Some properties only appear when parent properties have specific values:

```
resource: "message"
  └→ operation: "post"
       └→ select: "channel"     ← only shown when operation is "post"
            └→ channelId        ← only shown when select is "channel"
```

Always use `get_node(detail: 'standard')` to discover the full dependency tree before configuring.

### AI Workflow Connection Rules

LangChain nodes use special connection types, NOT the standard `main` type:

| Connection Type | Connects | Example |
|---|---|---|
| `ai_agent` | Trigger → Agent | ManualTrigger → Agent |
| `ai_languageModel` | LLM → Agent | ChatOpenAI → Agent |
| `ai_tool` | Tool → Agent | HTTPRequest → Agent (as tool) |
| `ai_memory` | Memory → Agent | BufferMemory → Agent |
| `ai_outputParser` | Parser → Agent | JSONParser → Agent |

### Authentication Patterns

- **OAuth2**: Configure via n8n Credentials UI, reference by credential ID in node config
- **API Key**: Store in n8n Credentials or `$env` variables — never hardcode
- **Header Auth**: Use Set node before HTTP Request to add `Authorization` header
- **No Auth**: Set `authentication: "none"` explicitly — don't rely on defaults

---

## 6. Code Node Patterns

### When to Use Code Nodes

Use standard n8n nodes for 95% of operations. Use Code nodes only when:
- Complex data transformation that no standard node handles
- Custom API response parsing
- Multi-step calculations or conditional logic beyond IF/Switch

### JavaScript (Primary — 95% of Code node use)

#### Mode Selection

| Mode | Access Pattern | Use When |
|---|---|---|
| **Run Once for All Items** | `$input.all()` | Processing arrays, aggregating, batch operations |
| **Run Once for Each Item** | `$input.item` | Per-item transformations, simple mapping |

#### Data Access Patterns

```javascript
// All Items mode (most common — 26% of usage)
const items = $input.all();
return items.map(item => ({
  json: { ...item.json, processed: true }
}));

// First Item mode (25% of usage)
const data = $input.first().json;
return [{ json: { result: data.value * 2 } }];

// Each Item mode (19% of usage)
const item = $input.item;
return [{ json: { ...item.json, transformed: true } }];

// Reference other nodes
const webhookData = $('Webhook').first().json.body;
const httpResult = $('HTTP Request').first().json;
```

#### CRITICAL: Return Format

```javascript
// ✅ MUST return array of objects with "json" key
return [{ json: { name: "Alice", status: "active" } }];

// ✅ Multiple items
return items.map(i => ({ json: { id: i.json.id } }));

// ❌ FAILS — missing json wrapper
return [{ name: "Alice" }];

// ❌ FAILS — not an array
return { json: { name: "Alice" } };
```

#### Built-in Helpers

```javascript
// HTTP requests (no external packages needed)
const response = await $helpers.httpRequest({
  method: 'GET',
  url: 'https://api.example.com/data',
  headers: { 'Authorization': `Bearer ${$env.API_KEY}` }
});
return [{ json: response }];
```

#### Top 5 JavaScript Errors

| # | Error | Fix |
|---|---|---|
| 1 | Not returning `[{json: {...}}]` | Always wrap in array + json key |
| 2 | Using `$json` instead of `$input` | In Code nodes, use `$input.first().json` |
| 3 | Webhook data at wrong depth | Access `$input.first().json.body` not `.json` |
| 4 | Async without await | Add `await` before `$helpers.httpRequest()` |
| 5 | Modifying input directly | Clone with spread: `{...item.json}` |

### Python (Secondary — 5% of use cases)

Use Python only when specific standard library functions are needed or Python syntax is strongly preferred.

#### Key Differences from JavaScript

```python
# Data access uses underscore prefix
items = _input.all()          # not $input.all()
first = _input.first()        # not $input.first()
item = _input.item            # not $input.item

# Return format — same structure, Python syntax
return [{"json": {"name": "Alice", "processed": True}}]

# Reference other nodes
webhook = _node["Webhook"].json["body"]
```

#### Python Limitations

- **No external libraries** — no pandas, numpy, requests, beautifulsoup
- **Standard library only**: json, datetime, re, math, collections, itertools, hashlib, base64, urllib
- **No pip install** — sandbox environment has no package manager
- **No file system access** — cannot read/write files

**For production-tested code patterns, see [references/CODE_EXAMPLES.md](references/CODE_EXAMPLES.md).**
