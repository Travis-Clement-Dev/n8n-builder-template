# Error Catalog — Complete Fix Procedures

Load this file when debugging n8n validation errors or workflow execution failures.

---

## Error Type 1: MISSING_REQUIRED

**Cause**: A property that the node requires for the selected operation is not set.

**Diagnosis**:
```
get_node({nodeType: "n8n-nodes-base.slack", mode: "info", detail: "standard"})
```
Check which properties are `required: true` for the current resource + operation combination.

**Fix**:
```json
// Error: "Missing required property: channelId"
// Context: resource=message, operation=post, select=channel

// Before (broken)
{"resource": "message", "operation": "post", "text": "Hello"}

// After (fixed) — add ALL required properties
{"resource": "message", "operation": "post", "select": "channel", "channelId": "C123", "text": "Hello"}
```

**Prevention**: Always use `validate_node(mode: 'minimal')` before building.

---

## Error Type 2: INVALID_TYPE

**Cause**: Property value doesn't match the expected type (string vs number vs boolean).

**Fix**:
```json
// Error: "Property 'batchSize' expected number, got string"

// Before
{"batchSize": "100"}

// After
{"batchSize": 100}
```

Use `get_node(mode: 'search_properties', propertyQuery: 'batchSize')` to check the expected type.

---

## Error Type 3: UNKNOWN_PROPERTY

**Cause**: The property name doesn't exist on this node or is misspelled.

**Common causes**:
- Typo in property name (`chanelId` instead of `channelId`)
- Property from wrong API version
- Property that only exists under certain operation selections

**Fix**: Use `get_node(detail: 'full')` to list all valid properties.

---

## Error Type 4: INVALID_EXPRESSION

**Cause**: Malformed expression syntax inside `{{ }}` blocks.

**Common sub-errors**:

```
// Missing closing braces
❌ {{ $json.name }
✅ {{ $json.name }}

// Unescaped special characters in string
❌ {{ "it's a test" }}
✅ {{ "it\\'s a test" }}

// Referencing undefined variable
❌ {{ $json.nonExistent.deep.path }}
✅ {{ $json.nonExistent?.deep?.path ?? "default" }}

// Using JS reserved words
❌ {{ $json.class }}
✅ {{ $json["class"] }}
```

---

## Error Type 5: MISSING_CONNECTION

**Cause**: A node has no input connection, so it would never execute.

**Fix**: Add a connection from the appropriate upstream node.

**Common scenarios**:
- Forgot to connect a newly added node
- Deleted a node that was the source of a connection
- Branching logic left a path disconnected

Use `validate_workflow()` to detect all orphaned nodes.

---

## Error Type 6: CIRCULAR_DEPENDENCY

**Cause**: Node A connects to B, B connects to C, C connects back to A.

**Fix**: Break the cycle by:
1. Identifying the loop with `validate_workflow()`
2. Removing the backwards connection
3. If iteration is needed, use `splitInBatches` node which has built-in loop support

---

## Error Type 7: INVALID_CREDENTIAL

**Cause**: The credential ID referenced doesn't exist or doesn't match the node's expected credential type.

**Fix**:
- Verify the credential exists in the n8n instance
- Check the credential type matches (e.g., `slackOAuth2Api` not `slackApi`)
- For development, use `authentication: "none"` and add auth later

**Note**: This is often a false positive in development — credentials are environment-specific.

---

## Error Type 8: DUPLICATE_NODE_NAME

**Cause**: Two or more nodes in the workflow share the same `name` field.

**Fix**: Ensure every node has a unique name. Convention:
```
"Slack"       → "Slack - Send Alert"
"HTTP Request"→ "HTTP Request - Fetch Users"
"IF"          → "IF - Has Email"
```

Descriptive names also improve `$node["Name"].json` references.

---

## Error Type 9: INVALID_AI_CONNECTION

**Cause**: An AI/LangChain node is connected using the `main` connection type instead of the required AI-specific type.

**Fix**:
```json
// ❌ Wrong — using main connection
{"source": "openai", "target": "agent", "sourcePort": "main", "targetPort": "main"}

// ✅ Correct — using ai_languageModel connection
{"source": "openai", "target": "agent", "sourcePort": "main", "targetPort": "ai_languageModel"}
```

**AI connection types**: `ai_agent`, `ai_languageModel`, `ai_tool`, `ai_memory`, `ai_outputParser`.

Always use `validate_node(profile: 'ai-friendly')` for LangChain workflows.

---

## Validation Iteration Strategy

### Expected Iteration Count by Complexity

| Workflow Complexity | Expected Iterations | Notes |
|---|---|---|
| Simple (3-5 nodes) | 1-2 | Usually passes on second try |
| Medium (6-12 nodes) | 2-3 | Property dependencies often missed first pass |
| Complex (13+ nodes) | 3-5 | AI workflows need extra ai-friendly validation |

### When to Stop Iterating

Stop validation iteration when:
- All `MISSING_REQUIRED` and `INVALID_TYPE` errors are resolved
- Remaining warnings are documented false positives
- `validate_workflow()` returns only warnings (not errors)
- All connections are verified and non-circular

### When to Override Validation

Override validation (proceed despite warnings) when:
- Warning is a known false positive (see SKILL.md Section 4)
- Community node has incomplete validation schema
- Expression references runtime-only data that can't be statically checked
- Credential validation fails in development but works in production environment
