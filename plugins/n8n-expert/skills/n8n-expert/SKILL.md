---
name: n8n-expert
description: Use when building, debugging, or extending n8n workflows. Triggers on intents like "build workflow", "add a Switch route", "push to n8n", "new n8n flow", "fix node", "workflow JSON", "debug execution", "AI agent in n8n", "Airtable in n8n", "Frame.io integration", "Meta Ads in n8n", "FAL polling", "webhook trigger", "n8n credential", "splitInBatches", or any work touching n8n's `/api/v1/workflows/` REST API. Encodes a battle-tested style of designing n8n systems — single-webhook + Switch routing, AI agent placement, Airtable patterns, API push discipline, common bug fixes.
---

# n8n Expert — Build n8n Systems Right

A skill that encodes a senior automation engineer's style for designing, building, debugging, and shipping n8n workflows on n8n cloud or self-hosted instances. When this skill is invoked, follow the principles in this file and read sub-files when the task touches their topic.

> *Trademark: Crafted by Abhay Singh Nagarkoti.*

---

## 🧭 The 7 Commandments — read first, always apply

These override any contradictory generic advice. Internalize before touching n8n.

### 1. ONE webhook, Switch routes for new functions

When extending an existing system, **add a new Switch rule + branch — never a new webhook**. One webhook URL per system. Routes by `$json.query.action` (GET) or `$json.body.action` / `body.type` (POST). Adding a new function = adding a new entry to the Switch's `rules.values` and wiring the new branch to `connections[SwitchName].main[N]`.

> Why: One webhook URL to share with external triggers (Airtable buttons, automations, third-party services). All routing visible in one node. New features additive, don't touch existing branches.

### 2. AI agents read RAW data — Code nodes go AFTER, never before

```
✅ Fetch table A → Fetch table B → Fetch table C → AI Agent (reads raw via $json + cross-refs) → Code (format AI output to destination schema) → Save
❌ Fetch tables → Code "Build Context" (summarizes/picks fields) → AI Agent → ...
```

Pre-processing data with a Code node before the AI agent destroys the agent's reasoning, introduces field-mapping bugs, and defeats the point of using AI. Feed all relevant tables raw. Let the AI choose what's relevant. Code AFTER the AI is fine — it's just shaping JSON for the destination (CMS POST body, Airtable Create columns, etc.).

### 3. Re-fetch live workflow before EVERY patch — GET after PUT

```
GET /workflows/{id} → fresh JSON
... apply changes to fresh JSON ...
PUT /workflows/{id}
GET /workflows/{id} → verify the specific change landed
```

Never trust a workflow JSON fetched minutes ago — the user may have hand-edited the canvas. Never trust a PUT 200 alone — n8n silently drops unknown fields. **Always GET after PUT and grep for the actual change.**

See [references/api-discipline.md](references/api-discipline.md) for full PUT body shape, settings stripping, credential API, editor cache.

### 4. LLM agents misbehave structurally — fix structurally

If an AI agent in n8n ignores a rule, the fix isn't to rephrase the prompt. It's:

1. Set `temperature: 0` on the chat model
2. Replace prose rules with a numbered decision procedure
3. State the DEFAULT explicitly (e.g., "your default is `image_urls: []`. You ONLY add a URL if [literal condition]")
4. List `❌ anti-examples` of what doesn't count
5. 4–5 worked input → output examples
6. Self-check step before output
7. **Pure pass-through downstream** — Code node consuming agent output must NOT add fallback inference

See [references/ai-agents.md](references/ai-agents.md) for the full pattern + the cinematic image-prompt baseline.

### 5. Static data accumulators — clear ONLY your own keys

Parallel chains writing to `$getWorkflowStaticData('global').foo` will stomp each other if any chain does `sd.foo = {}`. Each chain knows its keys; clear surgically:

```js
const sd = $getWorkflowStaticData('global');
if (!sd.uploadedImageUrls) sd.uploadedImageUrls = {};
['key1','key2'].forEach(k => delete sd.uploadedImageUrls[k]); // ✅ surgical
// ❌ NEVER: sd.uploadedImageUrls = {}
```

For loop aggregation across SplitInBatches iterations, use this accumulator pattern — `$items()` and `.all()` do NOT aggregate across iterations in Code nodes.

See [references/code-patterns.md](references/code-patterns.md) for full accumulator + binary handling + cross-node ref patterns.

### 6. Airtable Get returns FLAT, Search returns NESTED — they're not interchangeable

```js
// Get operation:    $json.fieldName               (fields at root)
// Search/Update:    $json.fields.fieldName        (fields nested under .fields)
```

Switching one for the other breaks every downstream expression. Before changing the operation, audit every reference. See [references/airtable.md](references/airtable.md).

### 7. Loud for mutations, tolerant for deletes

n8n nodes' `onError` setting matters:

| Node intent | `onError` |
|---|---|
| Create / Update / PUT / POST / Save | `stopWorkflow` (loud — fail = halt) |
| DELETE (Frame.io, Airtable, etc.) | `continueRegularOutput` (404 "already gone" = success) |
| Optional/best-effort writes | `continueRegularOutput` |

Default new mutation nodes to `stopWorkflow`. Silent mutation failures cause divergent state across systems and are nightmare to audit.

---

## 🗂️ Decision tree — which sub-file to read

When working on n8n, scan this list first:

| You're doing this... | Read |
|---|---|
| Designing a new workflow's overall shape | [references/architecture.md](references/architecture.md) |
| Building/modifying nodes via PUT to /api/v1/workflows | [references/api-discipline.md](references/api-discipline.md) + [references/node-shapes.md](references/node-shapes.md) |
| Writing JS in a Code node | [references/code-patterns.md](references/code-patterns.md) |
| Designing or fixing an LLM agent's prompt | [references/ai-agents.md](references/ai-agents.md) |
| Anything touching Airtable | [references/airtable.md](references/airtable.md) |
| HTTP requests, Frame.io, FAL, Meta Ads, Drive images | [references/http-and-external.md](references/http-and-external.md) |
| Error handling on a chain | [references/error-handling.md](references/error-handling.md) |
| Debugging "data is mysteriously missing" | [references/debugging.md](references/debugging.md) |

---

## 🧱 Quick reference — always-true facts

- **PUT body shape:** `{ name, nodes, connections, settings, staticData }`. Strip everything else. `settings` only allows `executionOrder`, `callerPolicy`, `errorWorkflow`.
- **n8n editor caches** node config from when the tab was opened. After every API push, **tell the user to Ctrl+R / Cmd+R before testing**.
- **Default deploy posture:** OK to push additive changes. Ask permission for destructive overwrites or first-time activation of production-touching workflows.
- **Credentials are NOT readable via API** (security). Can create new ones via `POST /credentials`, can't fetch existing values. To reuse an existing credential in a new node, copy its `{ id, name }` ref from another node of the same type.
- **typeVersion drift breaks things** — older `langchain.agent` versions reject newer model strings. Bump typeVersion before swapping models.

---

## 🛠️ Standard build/patch loop

When implementing any change:

```
1. GET fresh workflow JSON
2. Identify exact node(s) to add/modify
3. Build PUT body (strip settings to safe fields)
4. PUT
5. GET again — grep for the specific change
6. Tell the user: "Pushed and verified. Refresh your n8n tab (Ctrl+R) before testing."
7. Update vault memory with what changed (if it's a new pattern worth remembering)
```

Never skip step 5. Never claim success based on PUT 200 alone.

---

## 🎨 Naming conventions

- **Node names:** Title Case with spaces. Descriptive. ("Get Onboarding Data", "Build CMS Body", "AI Copywriter"). Never camelCase, never abbreviated.
- **Code node mode:** default to `runOnceForAllItems` for batch operations, `runOnceForEachItem` for per-row transforms. Mode change requires Ctrl+R or n8n executes with stale config.
- **Code node placement:** between "data source" and "consumer" — never before AI agents.
- **Sticky notes:** use to label major sections of the workflow (Switch routes, multi-step chains). Keeps a 100+ node workflow scannable.

---

## 🚦 When in doubt

- Inspect existing working nodes of the same type in the workflow — copy their parameter shape, credential ref, typeVersion.
- Check the [references/node-shapes.md](references/node-shapes.md) catalog for the canonical JSON shape.
- Add diagnostic fields to Code node output during development (`_diag_*` keys) so executions surface what was attempted.
- For mysterious bugs, dump the AI output, the static data, or the failing node's input as a temporary Code node — then remove after fixing.

---

> *Trademark: Crafted by Abhay Singh Nagarkoti.*
> *Pattern library distilled from production n8n systems across multiple agencies and instances. Use these as defaults; deviate only with reason.*
