# Code Examples — Production-Tested Patterns

Load this file when writing JavaScript or Python in n8n Code nodes.

---

## JavaScript Patterns

### Pattern 1: Multi-Source Aggregation (All Items Mode)

Combine data from multiple upstream nodes into a single output.

```javascript
const webhookData = $('Webhook').first().json.body;
const dbRecords = $('Postgres').all();
const apiResponse = $('HTTP Request').first().json;

const combined = dbRecords.map(record => ({
  json: {
    id: record.json.id,
    name: record.json.name,
    source: webhookData.source,
    enriched: apiResponse.metadata?.[record.json.id] ?? null,
    processedAt: new Date().toISOString()
  }
}));

return combined;
```

### Pattern 2: Filtering and Validation

Filter items based on complex conditions.

```javascript
const items = $input.all();

const valid = items.filter(item => {
  const data = item.json;
  return (
    data.email &&
    data.email.includes('@') &&
    data.status === 'active' &&
    data.createdAt > '2024-01-01'
  );
});

if (valid.length === 0) {
  return [{ json: { _empty: true, message: 'No valid records found' } }];
}

return valid.map(item => ({ json: item.json }));
```

### Pattern 3: API Response Transformation

Parse and reshape complex API responses.

```javascript
const response = $input.first().json;

// Handle paginated API responses
const items = response.data || response.results || response.items || [];
const nextPage = response.next_page || response.pagination?.next || null;

const transformed = items.map(item => ({
  json: {
    id: item.id,
    title: item.title || item.name,
    description: (item.description || '').substring(0, 500),
    tags: Array.isArray(item.tags) ? item.tags.join(', ') : '',
    url: item.url || item.html_url || null,
    _meta: {
      hasNextPage: !!nextPage,
      nextPageUrl: nextPage,
      totalItems: response.total || items.length
    }
  }
}));

return transformed;
```

### Pattern 4: Deduplication

Remove duplicate items based on a key field.

```javascript
const items = $input.all();
const seen = new Set();
const unique = [];

for (const item of items) {
  const key = item.json.email?.toLowerCase() || item.json.id;
  if (!seen.has(key)) {
    seen.add(key);
    unique.push({ json: { ...item.json } });
  }
}

return unique;
```

### Pattern 5: Date Handling

Parse, compare, and format dates using built-in Luxon.

```javascript
const { DateTime } = require('luxon');
const items = $input.all();

const now = DateTime.now();
const sevenDaysAgo = now.minus({ days: 7 });

return items
  .filter(item => {
    const itemDate = DateTime.fromISO(item.json.createdAt);
    return itemDate >= sevenDaysAgo;
  })
  .map(item => ({
    json: {
      ...item.json,
      formattedDate: DateTime.fromISO(item.json.createdAt).toFormat('yyyy-MM-dd HH:mm'),
      daysAgo: Math.floor(now.diff(DateTime.fromISO(item.json.createdAt), 'days').days),
      isToday: DateTime.fromISO(item.json.createdAt).hasSame(now, 'day')
    }
  }));
```

### Pattern 6: HTTP Request with Retry Logic

Make API calls with error handling and retry.

```javascript
const maxRetries = 3;
let attempt = 0;
let response;

while (attempt < maxRetries) {
  try {
    response = await $helpers.httpRequest({
      method: 'POST',
      url: 'https://api.example.com/webhook',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${$env.API_KEY}`
      },
      body: JSON.stringify({
        data: $input.first().json,
        timestamp: new Date().toISOString()
      }),
      timeout: 10000
    });
    break;
  } catch (error) {
    attempt++;
    if (attempt >= maxRetries) {
      return [{
        json: {
          success: false,
          error: error.message,
          attempts: attempt,
          lastAttempt: new Date().toISOString()
        }
      }];
    }
    await new Promise(resolve => setTimeout(resolve, 1000 * Math.pow(2, attempt)));
  }
}

return [{ json: { success: true, data: response, attempts: attempt + 1 } }];
```

### Pattern 7: Markdown/Text Processing

Parse structured text into JSON.

```javascript
const text = $input.first().json.body || $input.first().json.text;

// Parse key-value pairs from text
const lines = text.split('\n').filter(l => l.trim());
const parsed = {};

for (const line of lines) {
  const match = line.match(/^([^:]+):\s*(.+)$/);
  if (match) {
    const key = match[1].trim().toLowerCase().replace(/\s+/g, '_');
    parsed[key] = match[2].trim();
  }
}

return [{ json: parsed }];
```

### Pattern 8: Slack Block Kit Message

Build rich Slack messages with blocks.

```javascript
const data = $input.first().json;

const blocks = [
  {
    type: 'header',
    text: { type: 'plain_text', text: `📊 ${data.title || 'Report'}` }
  },
  {
    type: 'section',
    text: {
      type: 'mrkdwn',
      text: `*Status:* ${data.status}\n*Updated:* ${new Date().toLocaleDateString()}`
    }
  },
  { type: 'divider' }
];

// Add dynamic sections
if (data.items && Array.isArray(data.items)) {
  for (const item of data.items.slice(0, 10)) {
    blocks.push({
      type: 'section',
      text: { type: 'mrkdwn', text: `• *${item.name}*: ${item.value}` }
    });
  }
}

return [{
  json: {
    blocks: JSON.stringify(blocks),
    text: `${data.title}: ${data.status}`
  }
}];
```

---

## Python Patterns

### Pattern 1: Data Transformation

```python
from datetime import datetime
import json

items = _input.all()

processed = []
for item in items:
    data = item["json"]
    processed.append({
        "json": {
            "id": data.get("id"),
            "name": data.get("name", "").strip().title(),
            "email": data.get("email", "").lower(),
            "processed_at": datetime.now().isoformat(),
            "valid": bool(data.get("email") and "@" in data.get("email", ""))
        }
    })

return processed
```

### Pattern 2: Text Parsing with Regex

```python
import re

text = _input.first()["json"].get("body", "")

# Extract emails
emails = re.findall(r'[\w.+-]+@[\w-]+\.[\w.-]+', text)

# Extract URLs
urls = re.findall(r'https?://[^\s<>"{}|\\^`\[\]]+', text)

# Extract phone numbers
phones = re.findall(r'\+?[\d\s-]{10,}', text)

return [{
    "json": {
        "emails": list(set(emails)),
        "urls": list(set(urls)),
        "phones": [p.strip() for p in phones],
        "email_count": len(set(emails)),
        "url_count": len(set(urls))
    }
}]
```

### Pattern 3: Statistical Calculations

```python
import math
from collections import Counter

items = _input.all()
values = [item["json"].get("value", 0) for item in items if item["json"].get("value") is not None]

if not values:
    return [{"json": {"error": "No values to process"}}]

n = len(values)
mean = sum(values) / n
sorted_vals = sorted(values)
median = sorted_vals[n // 2] if n % 2 else (sorted_vals[n // 2 - 1] + sorted_vals[n // 2]) / 2
variance = sum((x - mean) ** 2 for x in values) / n
std_dev = math.sqrt(variance)

return [{
    "json": {
        "count": n,
        "sum": sum(values),
        "mean": round(mean, 2),
        "median": median,
        "min": min(values),
        "max": max(values),
        "std_dev": round(std_dev, 2),
        "mode": Counter(values).most_common(1)[0][0]
    }
}]
```

### Pattern 4: JSON Flattening

```python
def flatten(obj, prefix=""):
    result = {}
    if isinstance(obj, dict):
        for key, value in obj.items():
            new_key = f"{prefix}.{key}" if prefix else key
            result.update(flatten(value, new_key))
    elif isinstance(obj, list):
        for i, value in enumerate(obj):
            result.update(flatten(value, f"{prefix}[{i}]"))
    else:
        result[prefix] = obj
    return result

items = _input.all()
return [{"json": flatten(item["json"])} for item in items]
```

---

## Anti-Patterns to Avoid

### In JavaScript
```javascript
// ❌ Never use require() for external packages
const axios = require('axios');  // Will fail — not available

// ❌ Never use console.log for output
console.log(result);  // Output is lost

// ❌ Never modify $input items directly
$input.first().json.name = "New";  // Mutates shared state

// ✅ Always clone
const data = { ...$input.first().json, name: "New" };
```

### In Python
```python
# ❌ Never import external libraries
import pandas as pd  # Will fail — not available
import requests       # Will fail — not available

# ❌ Never use print for output
print(result)  # Output is lost

# ✅ Use urllib for HTTP (standard library)
from urllib.request import urlopen
import json
response = json.loads(urlopen("https://api.example.com/data").read())
```
