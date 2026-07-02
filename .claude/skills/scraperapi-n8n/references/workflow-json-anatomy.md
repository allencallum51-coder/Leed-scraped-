# n8n Workflow JSON — Anatomy for Importing

A valid n8n workflow JSON has four top-level keys: `name`, `nodes`,
`connections`, and `settings`. n8n will ignore unknown top-level keys
silently, but missing required keys break import.

## Top-level skeleton

```json
{
  "name": "My ScraperAPI workflow",
  "nodes": [],
  "connections": {},
  "settings": { "executionOrder": "v1" }
}
```

Optional but useful: `pinData` (test fixtures), `staticData`, `meta`,
`tags`, `versionId`. None are required for import to succeed.

## Per-node anatomy

```json
{
  "parameters": { "...": "..." },
  "type": "n8n-nodes-base.set",
  "typeVersion": 3.4,
  "position": [480, 300],
  "id": "any-unique-string",
  "name": "Display name shown in editor",
  "credentials": { "credentialKey": { "id": "...", "name": "..." } },
  "disabled": false,
  "notes": "Optional commentary"
}
```

- **`id`** must be unique within the workflow. UUIDs work, but any
  unique string is fine on import — n8n re-assigns IDs internally.
- **`name`** is the editor label and is the key used by the
  `connections` map. Rename consistently.
- **`type`** is the registered node identifier. For built-in nodes, it
  starts with `n8n-nodes-base.`. For the ScraperAPI community node, it
  is `n8n-nodes-scraperapi-official.scraperApi`.
- **`typeVersion`** matters. Mismatched versions sometimes silently drop
  parameter fields the importer doesn't recognize for that version. For
  ScraperAPI use `1`; for built-ins use the latest version that field
  shapes correspond to (Set is currently at `3.4`).
- **`position`** is `[x, y]` in pixels. Anything works; n8n's auto-layout
  is one click away.

## Connections map

The connections object maps **source node name** → connection type →
**array of arrays of targets**:

```json
"connections": {
  "Webhook": {
    "main": [
      [
        { "node": "ScraperAPI", "type": "main", "index": 0 }
      ]
    ]
  },
  "ScraperAPI": {
    "main": [
      [
        { "node": "Parse JSON", "type": "main", "index": 0 }
      ]
    ]
  }
}
```

The outer array is **per output port** (most nodes have one); the inner
array is **fan-out** (one source can feed many targets in parallel).
The `IF` node has two output ports — true and false — so its
connections object has two outer entries.

## Common companion nodes

These are the ones a ScraperAPI workflow almost always pairs with.

### Manual Trigger

```json
{
  "parameters": {},
  "type": "n8n-nodes-base.manualTrigger",
  "typeVersion": 1,
  "position": [240, 300],
  "id": "trigger-1",
  "name": "When clicking 'Test workflow'"
}
```

### Schedule Trigger (daily at 09:00)

```json
{
  "parameters": {
    "rule": {
      "interval": [
        { "field": "hours", "hoursInterval": 24, "triggerAtHour": 9 }
      ]
    }
  },
  "type": "n8n-nodes-base.scheduleTrigger",
  "typeVersion": 1.2,
  "position": [240, 300],
  "id": "schedule-1",
  "name": "Daily 9am"
}
```

### Webhook

```json
{
  "parameters": {
    "httpMethod": "POST",
    "path": "scrape",
    "responseMode": "responseNode"
  },
  "type": "n8n-nodes-base.webhook",
  "typeVersion": 2,
  "position": [240, 300],
  "id": "webhook-1",
  "name": "Webhook"
}
```

Pair with `respondToWebhook` to return data to the caller.

### Code (parse the SDE response)

```json
{
  "parameters": {
    "language": "javaScript",
    "jsCode": "return items.map(item => ({ json: JSON.parse(item.json.response.body) }));"
  },
  "type": "n8n-nodes-base.code",
  "typeVersion": 2,
  "position": [720, 300],
  "id": "code-1",
  "name": "Parse JSON"
}
```

### Set (shape the final output)

```json
{
  "parameters": {
    "mode": "manual",
    "fields": {
      "values": [
        { "name": "title", "type": "stringValue", "stringValue": "={{ $json.name }}" },
        { "name": "price", "type": "stringValue", "stringValue": "={{ $json.pricing }}" }
      ]
    }
  },
  "type": "n8n-nodes-base.set",
  "typeVersion": 3.4,
  "position": [960, 300],
  "id": "set-1",
  "name": "Shape"
}
```

### Loop Over Items (rate-limit batching)

```json
{
  "parameters": { "batchSize": 5, "options": {} },
  "type": "n8n-nodes-base.splitInBatches",
  "typeVersion": 3,
  "position": [480, 300],
  "id": "loop-1",
  "name": "Loop Over Items"
}
```

Wire its **first output (item)** into the ScraperAPI branch and its
**second output (done)** into whatever comes after the loop.

### IF (gate on change)

```json
{
  "parameters": {
    "conditions": {
      "options": { "caseSensitive": true, "leftValue": "", "typeValidation": "strict" },
      "conditions": [
        {
          "leftValue": "={{ $json.current_price }}",
          "rightValue": "={{ $json.previous_price }}",
          "operator": { "type": "number", "operation": "lt" }
        }
      ],
      "combinator": "and"
    }
  },
  "type": "n8n-nodes-base.if",
  "typeVersion": 2,
  "position": [960, 300],
  "id": "if-1",
  "name": "Price dropped?"
}
```

## Expression syntax

n8n expressions live inside `={{ ... }}` strings. Inside expressions:

- `$json` — the current item's JSON
- `$json.field.subfield` — dot-access into nested fields
- `$('Node Name').item.json.field` — reach into a specific upstream node
- `$node['Node Name'].json.field` — older syntax, still works
- `$items('Node Name')` — array of all items from a node

Examples:

- `={{ $json.response.body }}` — raw body of the ScraperAPI response
- `={{ JSON.parse($json.response.body).pricing }}` — inline parse + field
- `={{ $('ScraperAPI').item.json.response.body }}` — reference by node
  name (useful when the immediate previous node is a Merge or IF)

## Importing the JSON

The user has two ways to import:

1. **Workflows → Import from File** — file picker; selects the .json.
2. **Workflows → Import from URL** — paste a URL the n8n instance can
   reach.

After import, each ScraperAPI node will show a "credential not selected"
warning. The user clicks each node and picks their existing **ScraperAPI
API** credential from the dropdown. That's the only required manual
step on import (assuming the community node is already installed).

## Common import errors

- **"Unrecognized node type: n8n-nodes-scraperapi-official.scraperApi"**
  — the community node is not installed. Install via Settings →
  Community Nodes → `n8n-nodes-scraperapi-official`.
- **"Credentials of type X are not configured"** — expected. Click the
  node and select the credential.
- **A node's parameters look mostly empty after import** — usually a
  `typeVersion` mismatch. Verify the node's `typeVersion` matches what
  the installed node version supports.
