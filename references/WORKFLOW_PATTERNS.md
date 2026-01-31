# Workflow Patterns — Full Implementations

Complete node configurations for each of the 5 core workflow patterns. Load this file when building a workflow matching one of these patterns.

---

## Pattern 1: Webhook Processing — Full Implementation

### Node Chain
```
Webhook → IF (validate) → Set (transform) → [Business Logic] → Respond to Webhook
                ↓ (invalid)
         Respond to Webhook (400 error)
```

### Webhook Node Config
```json
{
  "nodeType": "n8n-nodes-base.webhook",
  "parameters": {
    "httpMethod": "POST",
    "path": "my-webhook-endpoint",
    "responseMode": "responseNode",
    "options": {}
  }
}
```

### Input Validation (IF Node)
```json
{
  "nodeType": "n8n-nodes-base.if",
  "parameters": {
    "conditions": {
      "options": { "caseSensitive": true },
      "combinator": "and",
      "conditions": [
        {
          "leftValue": "={{ $json.body.email }}",
          "rightValue": "",
          "operator": { "type": "string", "operation": "isNotEmpty" }
        }
      ]
    }
  }
}
```

### Error Response (Respond to Webhook)
```json
{
  "nodeType": "n8n-nodes-base.respondToWebhook",
  "parameters": {
    "respondWith": "json",
    "responseBody": "={{ JSON.stringify({ error: 'Missing required field: email', status: 400 }) }}",
    "options": { "responseCode": 400 }
  }
}
```

### Data Transformation (Set Node)
```json
{
  "nodeType": "n8n-nodes-base.set",
  "parameters": {
    "mode": "manual",
    "duplicateItem": false,
    "assignments": {
      "assignments": [
        { "name": "email", "value": "={{ $json.body.email }}", "type": "string" },
        { "name": "name", "value": "={{ $json.body.name ?? 'Unknown' }}", "type": "string" },
        { "name": "receivedAt", "value": "={{ $now.toISO() }}", "type": "string" }
      ]
    }
  }
}
```

---

## Pattern 2: HTTP API Integration — Full Implementation

### Node Chain
```
Trigger → Set (auth headers) → HTTP Request → IF (status) → Process → Output
                                                    ↓ (error)
                                              Wait (backoff) → Retry
```

### HTTP Request with Error Handling
```json
{
  "nodeType": "n8n-nodes-base.httpRequest",
  "parameters": {
    "method": "GET",
    "url": "https://api.example.com/data",
    "authentication": "genericCredentialType",
    "genericAuthType": "httpHeaderAuth",
    "options": {
      "timeout": 30000,
      "redirect": { "redirect": { "followRedirects": true } }
    }
  },
  "continueOnFail": true
}
```

### Status Check (IF Node)
```json
{
  "conditions": {
    "combinator": "and",
    "conditions": [
      {
        "leftValue": "={{ $json.statusCode }}",
        "rightValue": 200,
        "operator": { "type": "number", "operation": "equals" }
      }
    ]
  }
}
```

### Retry with Exponential Backoff (Wait Node)
```json
{
  "nodeType": "n8n-nodes-base.wait",
  "parameters": {
    "amount": "={{ Math.pow(2, $runIndex) }}",
    "unit": "seconds"
  }
}
```

---

## Pattern 3: Database Operations — Full Implementation

### Node Chain
```
Schedule Trigger → DB Query (source) → Set (map fields) → DB Upsert (target) → DB Query (verify)
```

### Postgres Query Example
```json
{
  "nodeType": "n8n-nodes-base.postgres",
  "parameters": {
    "operation": "executeQuery",
    "query": "SELECT id, email, name, updated_at FROM users WHERE updated_at > $1",
    "options": {
      "queryReplacement": "={{ $now.minus({hours: 1}).toISO() }}"
    }
  }
}
```

### Field Mapping (Set Node)
```json
{
  "parameters": {
    "mode": "manual",
    "assignments": {
      "assignments": [
        { "name": "user_id", "value": "={{ $json.id }}", "type": "number" },
        { "name": "user_email", "value": "={{ $json.email }}", "type": "string" },
        { "name": "synced_at", "value": "={{ $now.toISO() }}", "type": "string" }
      ]
    }
  }
}
```

### Batch Processing for Large Datasets
```json
{
  "nodeType": "n8n-nodes-base.splitInBatches",
  "parameters": {
    "batchSize": 100,
    "options": {}
  }
}
```

---

## Pattern 4: AI Agent Workflow — Full Implementation

### Node Chain (with special connection types)
```
Manual Trigger ─[ai_agent]→ Agent ─[main]→ Output
                               ↑
          ChatOpenAI ──[ai_languageModel]──┘
          BufferMemory ──[ai_memory]───────┘
          HTTP Tool ──[ai_tool]────────────┘
```

### Agent Node
```json
{
  "nodeType": "@n8n/n8n-nodes-langchain.agent",
  "parameters": {
    "text": "={{ $json.chatInput }}",
    "options": {
      "systemMessage": "You are a helpful assistant. Answer questions using the provided tools."
    }
  }
}
```

### LLM Connection
```json
{
  "nodeType": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
  "parameters": {
    "model": "gpt-4o",
    "options": {
      "temperature": 0.7,
      "maxTokens": 2048
    }
  }
}
```

### Memory Connection
```json
{
  "nodeType": "@n8n/n8n-nodes-langchain.memoryBufferWindow",
  "parameters": {
    "sessionKey": "={{ $json.sessionId ?? 'default' }}",
    "contextWindowLength": 10
  }
}
```

### Connection Configuration
```json
[
  {"source": "trigger", "target": "agent", "sourcePort": "main", "targetPort": "ai_agent"},
  {"source": "openai", "target": "agent", "sourcePort": "main", "targetPort": "ai_languageModel"},
  {"source": "memory", "target": "agent", "sourcePort": "main", "targetPort": "ai_memory"},
  {"source": "httpTool", "target": "agent", "sourcePort": "main", "targetPort": "ai_tool"}
]
```

---

## Pattern 5: Scheduled Tasks — Full Implementation

### Node Chain
```
Schedule Trigger → HTTP Request (fetch) → IF (has data) → Process → Slack (notify)
                                                ↓ (no data)
                                           No Operation
```

### Schedule Trigger
```json
{
  "nodeType": "n8n-nodes-base.scheduleTrigger",
  "parameters": {
    "rule": {
      "interval": [{ "field": "cronExpression", "expression": "0 9 * * 1-5" }]
    }
  }
}
```

Common cron expressions:
- `0 9 * * 1-5` — Weekdays at 9 AM
- `0 */6 * * *` — Every 6 hours
- `0 0 1 * *` — First day of each month
- `*/15 * * * *` — Every 15 minutes

### Notification (Slack Node)
```json
{
  "nodeType": "n8n-nodes-base.slack",
  "parameters": {
    "resource": "message",
    "operation": "post",
    "select": "channel",
    "channelId": "={{ $env.SLACK_CHANNEL_ID }}",
    "text": "={{ 'Daily report: ' + $json.summary + ' | Run: ' + $execution.id }}"
  }
}
```
