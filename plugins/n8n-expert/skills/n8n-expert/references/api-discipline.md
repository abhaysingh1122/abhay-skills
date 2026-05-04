# API Discipline — Pushing Workflows Without Breaking Things

Rules for editing n8n workflows via the public REST API. Follow these every time.

---

## The standard build/patch loop

```
1. GET  /api/v1/workflows/{id}           ← fresh JSON
2. Modify nodes/connections in memory
3. Strip settings to safe fields
4. PUT  /api/v1/workflows/{id}           ← submit changes
5. GET  /api/v1/workflows/{id}           ← verify the specific change landed
6. Tell user: "Pushed and verified. Refresh n8n (Ctrl+R) before testing."
```

Never skip step 5. Never trust PUT 200 alone — n8n silently drops unknown fields.

---

## Rule 1 — Re-fetch IMMEDIATELY before every patch

Don't reuse a workflow JSON fetched 5 minutes ago. The user may have:
- Hand-edited node config in the canvas
- Toggled `disabled` on/off
- Repositioned nodes
- Added/deleted connections

If you patch from stale data, your PUT overwrites their changes silently.

```bash
curl -sS -H "X-N8N-API-KEY: $KEY" "$URL/api/v1/workflows/$WID" -o fresh.json
```

Then build the PUT body from `fresh.json`, not anything older.

---

## Rule 2 — Strip settings to safe fields before PUT

n8n's full workflow object has many fields the API rejects on PUT. Keep ONLY:

```js
{
  name: wf.name,
  nodes: wf.nodes,
  connections: wf.connections,
  settings: {
    executionOrder: wf.settings?.executionOrder || 'v1',
    ...(wf.settings?.callerPolicy ? { callerPolicy: wf.settings.callerPolicy } : {}),
    ...(wf.settings?.errorWorkflow ? { errorWorkflow: wf.settings.errorWorkflow } : {}),
  },
  staticData: wf.staticData || null,
}
```

**Fields that cause 400 if included in PUT body:**
- `id` (in URL path, not body)
- `active` (use the separate `/activate` and `/deactivate` endpoints)
- `versionId`
- `triggerCount`
- `meta`
- `pinData`
- `tags`
- `createdAt` / `updatedAt`
- Any `settings.*` field other than `executionOrder` / `callerPolicy` / `errorWorkflow`

The error message is "settings must NOT have additional properties" — confusing because the offending field is sometimes nested.

---

## Rule 3 — Verify with GET after PUT (and grep for the actual change)

PUT returning 200 + a workflow object does NOT guarantee the change persisted. Always:

```js
// After PUT
execSync(`curl ... GET /workflows/${id} > verify.json`);
const v = JSON.parse(fs.readFileSync('verify.json'));
const target = v.nodes.find(n => n.name === '<NodeName>');
console.log('updatedAt:', v.updatedAt);       // confirm timestamp moved
console.log('change present:', target.parameters.<field>.includes('<expected substring>'));
```

Print **what specifically changed** — not just node count. Look for the actual string/parameter you set.

Common false-positives:
- PUT echoes your request back as if successful, but n8n didn't save fields with unknown shapes
- The workflow object in the PUT response is sometimes the *request*, not the saved state
- `updatedAt` not advancing = silent failure

---

## Rule 4 — Tell the user to refresh after every push

n8n's editor caches node config from the moment the tab opened. API change ≠ UI update. After every PUT, instruct:

> *"Pushed and verified. **Refresh n8n (Ctrl+R / Cmd+R)** before testing — n8n editor caches node config from when you opened the tab."*

If user reports "nothing changed" or "got an error referencing old fields", first ask: "Did you refresh?"

---

## Rule 5 — Don't deactivate active toggles when patching

When patching a node the user has enabled (data flowing through), don't reset `disabled: false → true`. Re-fetching live state preserves their toggles — but be careful when constructing PUT body that you're not overwriting their state.

---

## Rule 6 — Position changes are visual only

Connections are by node NAME, not position. Safe to rearrange `position: [x, y]` for visual cleanup without breaking logic.

---

## Rule 7 — Be careful with deletes

Deleting a node from `wf.nodes` does NOT auto-delete connections referencing it. Clean up:

```js
delete wf.connections[deletedNodeName];                    // outgoing connections
for (const src in wf.connections) {                        // incoming connections
  for (const slot in wf.connections[src]) {
    wf.connections[src][slot] = wf.connections[src][slot].map(group =>
      group.filter(c => c.node !== deletedNodeName)
    );
  }
}
```

Easier: don't delete via API. Disconnect by removing/redirecting connections, leave the orphan node for the user to delete visually.

---

## Rule 8 — Track credentials per node

When adding new nodes, attach the right credential ref:

```js
credentials: { airtableTokenApi: { id: '<CRED_ID>', name: '<Display Name>' } }
credentials: { oAuth2Api:        { id: '<CRED_ID>', name: 'Frame.io OAuth2' } }
credentials: { facebookGraphApi: { id: '<CRED_ID>', name: 'Meta Ads' } }
credentials: { openAiApi:        { id: '<CRED_ID>', name: 'OpenAI' } }
credentials: { openRouterApi:    { id: '<CRED_ID>', name: 'OpenRouter' } }
credentials: { slackApi:         { id: '<CRED_ID>', name: 'Slack' } }
```

Cred IDs are per-instance. Inspect existing nodes of the same type to copy the right ref:

```js
const findCred = (nodeName, credType) => {
  const n = wf.nodes.find(x => x.name === nodeName);
  return n?.credentials?.[credType]; // returns { id, name }
};
const AT_CRED = findCred('Some Existing Airtable Node', 'airtableTokenApi');
```

---

## Creating credentials programmatically

If the credential doesn't exist yet, create it via API:

```bash
curl -sS -X POST "https://INSTANCE/api/v1/credentials" \
  -H "X-N8N-API-KEY: $KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Friendly Display Name",
    "type": "facebookGraphApi",
    "data": { "accessToken": "EAA..." }
  }'
```

Response includes the new credential's `id`. Use it in your workflow nodes' credential refs.

### Discover the data shape for a credential type
```bash
curl -sS -H "X-N8N-API-KEY: $KEY" \
  "https://INSTANCE/api/v1/credentials/schema/{credential_type}"
```

Common shapes:
| Type | data keys |
|---|---|
| `facebookGraphApi` | `accessToken` |
| `airtableTokenApi` | `accessToken` (the PAT) |
| `openRouterApi` | `apiKey` |
| `openAiApi` | `apiKey`, `organizationId` (optional) |
| `slackApi` | `accessToken` |
| `httpHeaderAuth` | `name`, `value` |

### What the API CAN'T do
- Update existing credentials (no PATCH)
- Read existing credential VALUES (security restriction)
- List all credentials (404 on cloud)

For updates, edit via UI. The API is create-only.

---

## Endpoints quick reference

```
GET    /api/v1/workflows                  ← list (?active=true&limit=N&cursor=...)
GET    /api/v1/workflows/{id}             ← read one
PUT    /api/v1/workflows/{id}             ← full replace (strip settings!)
POST   /api/v1/workflows/{id}/activate    ← activate (no body)
POST   /api/v1/workflows/{id}/deactivate  ← deactivate
DELETE /api/v1/workflows/{id}             ← delete
GET    /api/v1/executions                 ← list executions
GET    /api/v1/executions/{id}            ← full execution data (include &includeData=true)
DELETE /api/v1/executions/{id}            ← delete execution
GET    /api/v1/credentials/schema/{type}  ← credential data shape
POST   /api/v1/credentials                ← create credential
DELETE /api/v1/credentials/{id}           ← delete credential
```

---

## Auth header (always)
```
X-N8N-API-KEY: <jwt>
```

JWT is generated from n8n Settings → API page in the UI. NOT the user's password. Keys for instances should be saved in a secure vault, never in code/git.

---

## Common HTTP errors

| Code | Likely cause |
|---|---|
| 401 | Bad/expired API key |
| 400 | Body has unsupported field (strip settings); malformed JSON; settings has additional properties |
| 404 | Wrong workflow/execution ID |
| 500 | Server error — usually retry succeeds |

---

## Editor cache — always tell user to refresh

After every PUT, in your reply to the user:

> *"Refresh n8n (Ctrl+R / Cmd+R) before testing."*

If they report errors that don't match your saved config, the first question is: "Did you refresh?"

---

## Defaults for new workflows

```js
settings: {
  executionOrder: 'v1',
  callerPolicy: 'workflowsFromSameOwner',
}
```

`executionOrder: v1` is required for modern execution semantics. `workflowsFromSameOwner` is the safe default for sub-workflow access.
