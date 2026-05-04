# Node Shapes — Canonical JSON for Building Nodes via API

Reference catalog for constructing n8n nodes programmatically. Use these shapes when scripting workflow PUTs.

---

## Workflow PUT body (top level)

```js
{
  name: '...',
  nodes: [...],
  connections: { ... },
  settings: {
    executionOrder: 'v1',
    callerPolicy?: 'workflowsFromSameOwner',  // pass through if exists
    errorWorkflow?: '...',                     // pass through if exists
  },
  staticData: null   // or pass through if exists
}
```

Strip ALL other top-level fields (`pinData`, `versionId`, `triggerCount`, `meta`, `tags`, `active`, `createdAt`, `updatedAt`, `id`).

---

## Node base shape

```js
{
  id: crypto.randomUUID(),       // REQUIRED unique within workflow
  name: 'Friendly Display Name', // REQUIRED unique within workflow (used by connections!)
  type: 'n8n-nodes-base.X' | '@n8n/n8n-nodes-langchain.X',
  typeVersion: <float>,          // varies by node — see catalog below
  position: [x, y],              // canvas coords (visual only)
  parameters: { ... },           // node-specific
  credentials?: { credType: { id, name } },
  disabled?: false,
  onError?: 'stopWorkflow' | 'continueRegularOutput' | 'continueErrorOutput',
  webhookId?: '<uuid>'           // required for Wait + Webhook nodes
}
```

---

## typeVersion catalog (current as of build time)

| Node type | typeVersion |
|---|---|
| `n8n-nodes-base.scheduleTrigger` | 1.2 |
| `n8n-nodes-base.webhook` | varies (1+) |
| `n8n-nodes-base.httpRequest` | 4.4 |
| `n8n-nodes-base.airtable` | 2.1 |
| `n8n-nodes-base.code` | 2 |
| `n8n-nodes-base.set` (Edit Fields) | 3.4 |
| `n8n-nodes-base.if` | 2.3 |
| `n8n-nodes-base.switch` | 3.2 |
| `n8n-nodes-base.merge` | 3.2 |
| `n8n-nodes-base.splitInBatches` | 3 |
| `n8n-nodes-base.splitOut` | 1 |
| `n8n-nodes-base.sort` | 1 |
| `n8n-nodes-base.wait` | 1.1 |
| `n8n-nodes-base.stickyNote` | 1 |
| `n8n-nodes-base.crypto` | 1 |
| `n8n-nodes-base.editImage` | 1 |
| `n8n-nodes-base.facebookGraphApi` | 1 |
| `n8n-nodes-base.slack` | 2.4 |
| `@n8n/n8n-nodes-langchain.agent` | 3 (v2 rejects newer model strings) |
| `@n8n/n8n-nodes-langchain.chainLlm` | 1.9 |
| `@n8n/n8n-nodes-langchain.lmChatOpenAi` | 1.3 |
| `@n8n/n8n-nodes-langchain.lmChatOpenRouter` | 1 |
| `@n8n/n8n-nodes-langchain.outputParserStructured` | 1.3 |

When in doubt, inspect existing working nodes of the same type and copy their `typeVersion`.

---

## Per-node parameter shapes

### HTTP Request (4.4)

```js
{
  method: 'GET' | 'POST' | 'PUT' | 'DELETE' | 'PATCH',
  url: '={{ ... }}',                       // = prefix for expressions
  authentication: 'genericCredentialType', // optional, when not using node-specific cred
  genericAuthType: 'oAuth2Api' | 'httpHeaderAuth' | 'httpBasicAuth',
  sendHeaders: true,
  headerParameters: {
    parameters: [{ name: 'X-Header', value: 'value' }]
  },
  sendBody: true,
  specifyBody: 'json' | 'binaryData' | 'form-urlencoded',  // ⚠️ NOT contentType
  jsonBody: '={ "key": "{{ value }}" }',                    // when specifyBody=json
  contentType: 'binaryData',                                // when binary PUT
  inputDataFieldName: 'data',                               // binary field name
  options: {
    response: { response: { responseFormat: 'file' | 'json' | 'text' } },  // download as binary
    timeout: 30000
  }
}
```

⚠️ **Trap:** Using `contentType: 'json'` instead of `specifyBody: 'json'` — n8n silently drops the unknown key, defaults to form-encoded body, sends empty body. Always `specifyBody`.

---

### Airtable v2.1

```js
{
  operation: 'get' | 'search' | 'create' | 'update' | 'upsert' | 'delete',
  base:  { __rl: true, value: 'appXXX', mode: 'list', cachedResultName: '...' },
  table: { __rl: true, value: 'tblXXX', mode: 'list', cachedResultName: '...' },
  
  // GET / UPDATE / DELETE — use top-level id
  id: '={{ ... }}',
  
  // SEARCH — formula-based filter
  filterByFormula: '={Status} = "Active"',
  
  // CREATE / UPDATE / UPSERT — column mapping
  columns: {
    mappingMode: 'defineBelow' | 'autoMapInputData',
    value: { 'Field Name': '={{ $json.x }}' },              // when defineBelow
    matchingColumns: ['ID Column'],                          // for upsert (match key)
    schema: [...]                                            // optional, for typecasting hints
  },
  options: { typecast: true }                                // converts strings → number/date/etc.
}
```

⚠️ **Get returns FLAT (`$json.fieldName`); Search/Update/Upsert return NESTED (`$json.fields.fieldName`).** See [airtable.md](airtable.md).

---

### Set / Edit Fields (3.4)

```js
{
  assignments: {
    assignments: [{
      id: '<uuid>',
      name: 'fieldName',
      value: '={{ expression }}',
      type: 'string' | 'number' | 'boolean' | 'array' | 'object'
    }]
  },
  options: {}
}
```

---

### IF (2.3)

```js
{
  conditions: {
    options: { caseSensitive: true, leftValue: '', typeValidation: 'strict', version: 3 },
    conditions: [{
      id: '<uuid>',
      leftValue: '={{ $json.x }}',
      rightValue: 'value',
      operator: { type: 'string'|'number'|'boolean', operation: 'equals'|'notEquals'|'notEmpty'|... }
    }],
    combinator: 'and' | 'or'
  },
  options: {}
}
// Outputs: main[0]=TRUE, main[1]=FALSE
```

---

### Switch (3.2)

```js
{
  rules: {
    values: [{
      conditions: {
        options: { caseSensitive: true, leftValue: '', typeValidation: 'strict', version: 3 },
        conditions: [{
          id: '<uuid>',
          leftValue: '={{ $json.action }}',
          rightValue: 'X',
          operator: { type: 'string', operation: 'equals' }
        }],
        combinator: 'and'
      },
      renameOutput: true,
      outputKey: 'Display Name'
    }]
  },
  options: { fallbackOutput: 'extra', renameFallbackOutput: 'Default' }  // optional fallback
}
// Outputs: main[N] for rule[N], plus optional fallback at last index
```

---

### SplitInBatches (3)

```js
{ batchSize: 1, options: {} }
// Outputs: main[0]=done (after all items), main[1]=loop body (per item)
// Body's last node MUST connect back to this node (main[0] of last → SplitInBatches main[0])
```

---

### SplitOut (1)

```js
{ fieldToSplitOut: 'Video_Frames' | 'fields.Video_Frames', options: {} }
// Each output item carries the array element on the same field name
// Supports nested dot-paths
```

---

### Sort (1)

```js
{
  type: 'simple',
  sortFieldsUi: { sortField: [{ fieldName: 'fields.ID', order: 'ascending' | 'descending' }] },
  options: {}
}
```

---

### Code (2)

```js
{
  mode: 'runOnceForAllItems' | 'runOnceForEachItem',
  jsCode: '...'   // available globals: $input, $json, $binary, $itemIndex, this.helpers, $('NodeName'), $getWorkflowStaticData
}
```

**Modes:**
- `runOnceForAllItems`: get array via `$input.all()`, return array
- `runOnceForEachItem`: `$json` is current item, `$itemIndex` is position, return single item

⚠️ Mode change requires Ctrl+R refresh in n8n editor or it executes with stale config.

---

### Wait (1.1)

```js
{
  amount: 5,
  unit: 'seconds' | 'minutes' | 'hours',
  webhookId: '<uuid>'   // REQUIRED — n8n stores wait state via this
}
```

---

### Sticky Note (1)

```js
{
  width: 400,
  height: 200,
  color: 1-7,        // 1=gray, 2=red, 3=orange, 4=yellow, 5=green, 6=blue, 7=purple
  content: '## Heading\n\nMarkdown text.'
}
```

---

### Schedule Trigger (1.2)

```js
{
  rule: {
    interval: [
      { triggerAtHour: 8 },                       // daily at 8am
      // OR: { hoursInterval: 6 }                  // every 6 hours
      // OR: { triggerAtDay: [1, 3, 5], triggerAtHour: 9 }  // Mon/Wed/Fri at 9am
    ]
  }
}
```

---

### Langchain Agent (3)

```js
{
  promptType: 'define',
  text: '=User prompt with {{ expressions }} from upstream',
  hasOutputParser: true,
  messages: { messageValues: [{ message: 'You are... [system prompt body]' }] },
  options: { systemMessage: 'optional alt system prompt placement' }
}
// Connect a chat model via type='ai_languageModel'
// Connect a structured output parser via type='ai_outputParser' if hasOutputParser
```

---

### Chain LLM (1.9)

```js
{
  promptType: 'define',
  text: '=User prompt: {{ $json.x }}',
  hasOutputParser: true,
  messages: { messageValues: [{ message: 'System role + rules' }] },
  batching: {}
}
// Same model + parser connection pattern as Agent
```

---

### LM Chat OpenAI (1.3) / OpenRouter (1)

```js
// OpenAI:
{
  model: { __rl: true, value: 'gpt-4.1-mini', mode: 'list' },
  options: { temperature: 0 }
}

// OpenRouter:
{
  model: 'openai/gpt-4o-mini',
  options: { temperature: 0 }
}
```

⚠️ Set `temperature: 0` for any agent that needs deterministic output.

---

### Output Parser Structured (1.3)

```js
{
  jsonSchemaExample: JSON.stringify({
    insight_type: 'normal',
    severity: 'normal',
    summary: 'text here'
  }, null, 2)
}
```

---

### Crypto (1) — for hashing

```js
{
  action: 'hash',
  type: 'MD5' | 'SHA256',
  binaryPropertyName: 'data',         // when hashing binary
  dataPropertyName: '={{ $json.x }}',  // when hashing string
  encoding: 'hex'
}
```

---

### Edit Image (1) — for resize/format

```js
{
  operation: 'resize',
  options: { ... },
  width: 800,
  height: 600
}
```

---

### Facebook Graph API (1)

```js
{
  graphApiVersion: '=v22.0/act_{{ $json.account_id }}/insights',
  node: 'me',                                  // legacy field, keep as 'me'
  options: {
    fields: { field: [{ name: 'spend' }, { name: 'impressions' }] },
    queryParameters: {
      parameter: [
        { name: 'date_preset', value: 'yesterday' },
        { name: 'level', value: 'campaign' }
      ]
    }
  }
}
// Credential: credentials: { facebookGraphApi: { id, name } }
```

---

### Slack (2.4)

```js
{
  select: 'channel',
  channelId: { __rl: true, value: 'C0XXXX', mode: 'id' },
  text: '=Message body with {{ expressions }}',
  otherOptions: { unfurl_links: false }
}
```

---

## Connections — wiring nodes

```js
wf.connections = {
  'Source Node Name': {
    main: [
      [{ node: 'Target1', type: 'main', index: 0 }],   // src out 0 → Target1
      [{ node: 'Target2', type: 'main', index: 0 }],   // src out 1 → Target2 (e.g., FALSE branch of IF)
    ],
    ai_languageModel: [                                  // for Agent → Chat Model
      [{ node: 'Agent Node', type: 'ai_languageModel', index: 0 }]
    ],
    ai_outputParser: [
      [{ node: 'Agent Node', type: 'ai_outputParser', index: 0 }]
    ]
  }
}
```

**Output index meanings:**

| Node | Out 0 | Out 1+ |
|---|---|---|
| IF v2 | TRUE | FALSE |
| Switch v3 (N rules) | rule[0] | rule[1]..rule[N-1], optional fallback last |
| SplitInBatches v3 | done | loop body |
| Most others | result | — |

**Helper for adding connections:**
```js
function addConn(srcName, srcOutIdx, dstName, dstInIdx = 0, type = 'main') {
  if (!wf.connections[srcName]) wf.connections[srcName] = {};
  if (!wf.connections[srcName][type]) wf.connections[srcName][type] = [];
  while (wf.connections[srcName][type].length <= srcOutIdx) wf.connections[srcName][type].push([]);
  wf.connections[srcName][type][srcOutIdx].push({ node: dstName, type, index: dstInIdx });
}
```

---

## Position conventions

- Coordinates can be very negative (large workflows often use X = -38000 to -22000)
- Step between sequential nodes: ~240px horizontal
- Loop body: shift down ~200px from main row
- Sticky notes: place above their related nodes
- Don't worry about pixel-perfect placement — n8n auto-routes connections

---

## Common credential refs

```js
credentials: { airtableTokenApi:    { id: '<id>', name: '<name>' } }   // Airtable PAT
credentials: { oAuth2Api:           { id: '<id>', name: '<name>' } }   // Frame.io / Adobe IMS
credentials: { openAiApi:           { id: '<id>', name: '<name>' } }
credentials: { openRouterApi:       { id: '<id>', name: '<name>' } }
credentials: { facebookGraphApi:    { id: '<id>', name: '<name>' } }
credentials: { slackApi:            { id: '<id>', name: '<name>' } }
credentials: { httpHeaderAuth:      { id: '<id>', name: '<name>' } }
```

Cred IDs are per-instance. Inspect existing nodes of the same type or use `findCred()` helper:

```js
const findCred = (nodeName, credType) => {
  const n = wf.nodes.find(x => x.name === nodeName);
  return n?.credentials?.[credType];
};
```

---

## Common construction mistakes (silent failures)

1. `contentType: 'json'` instead of `specifyBody: 'json'` → empty body sent
2. Airtable Update with top-level `id` param + `mappingMode: defineBelow` — `id` must go inside `columns.value` for some operations
3. Forgetting `webhookId` on Wait nodes — node won't trigger after sleep
4. `runOnceForAllItems` mode but writing per-item code (`$json` instead of `$input.all()`)
5. `$('NodeX').first()` outside execution context — test isolation breaks it; use `$json` fallback
6. Forgetting to clean up `wf.connections[deletedNodeName]` when removing nodes
7. Setting agent `typeVersion: 2` with newer model strings — error says "model not supported", real fix is bump to `typeVersion: 3`
8. Not setting `temperature: 0` on chat models for agents that need deterministic output
