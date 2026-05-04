# Error Handling — Loud Mutations, Tolerant Deletes

How to set `onError` policies on n8n nodes. Wrong defaults cause silent state divergence; right defaults surface bugs immediately.

---

## The rule by node intent

n8n HTTP/Airtable/Database nodes have an `onError` setting:
- `stopWorkflow` — fail loudly, halt execution
- `continueRegularOutput` — emit input as output, swallow error (silent)
- `continueErrorOutput` — emit error info to a separate "error" output (rare, for explicit error-handling chains)

### Use `stopWorkflow` (LOUD) for these node types:

| Node intent | Why loud |
|---|---|
| Create resources (Airtable Create, POST to external APIs) | Silent fail = missing data, no audit trail |
| Update resources (Airtable Update, PUT, PATCH) | Silent fail = stale data |
| Save asset/ID back to a DB | Silent fail = orphaned resources elsewhere |
| Binary uploads (S3 PUT, Frame.io upload) | Silent fail = empty assets |

If these fail silently, downstream nodes get empty data and you accumulate divergent state across systems.

### Use `continueRegularOutput` (TOLERANT) for these:

| Node intent | Why tolerant |
|---|---|
| DELETE operations | 404 "already gone" = success for idempotent flows |
| Optional/best-effort writes | Status updates, logging — should never block main flow |
| Notification nodes (Slack, email) | Comms failure shouldn't break business logic |

### Default for new mutation nodes: `stopWorkflow`

When building a new workflow, set every Create/Update/PUT/POST node to `stopWorkflow`. Only relax to `continueRegularOutput` when you've decided "this specific node is best-effort."

---

## DELETE nodes specifically

For idempotent delete flows (delete-then-recreate patterns), 404 means "already gone" which IS the desired state. Tolerate it:

```js
{
  type: 'n8n-nodes-base.httpRequest',
  // ...
  onError: 'continueRegularOutput'   // ← 404 OK for DELETE
}
```

Without this, a re-run of a partial workflow fails on assets already deleted from a previous run.

For Frame.io specifically: 404 on DELETE is common after retry. Set `continueRegularOutput`.

---

## Why this rule matters — silent failures = divergent state

**The classic bug:** workflow has `continueRegularOutput` defaults across all mutation nodes. A regen flow partially succeeds — some Airtable rows updated, some Frame.io assets created, but the linking step silently failed. Now Airtable rows point at deleted IDs and Frame.io has orphaned assets.

Took an audit to find. Switching mutation nodes to `stopWorkflow` would have surfaced the issue on the first failed run.

---

## Diagnostic clue

If a chain produces inconsistent results across runs (some rows correct, some stale, some duplicates) → check for silent error swallowing on mutation nodes. Likely they all default to `continueRegularOutput`.

---

## Setting `onError` in workflow JSON

```js
{
  type: 'n8n-nodes-base.airtable',
  name: 'Save Asset ID',
  parameters: { /* ... */ },
  onError: 'stopWorkflow',   // mutation = loud
  // ...
}

{
  type: 'n8n-nodes-base.httpRequest',
  name: 'Delete Old Frame.io File',
  parameters: { method: 'DELETE', /* ... */ },
  onError: 'continueRegularOutput',   // delete = tolerant
  // ...
}
```

---

## When you NEED to handle errors gracefully (rare)

For workflows where one client's failure shouldn't kill the loop for other clients:

```
SplitInBatches Loop → [chain] → next iteration
```

Set `onError: 'continueRegularOutput'` on the chain's mutation nodes IF you've decided per-iteration failure is recoverable. But:
- Log the error somewhere visible (Slack alert, error table)
- Tag the iteration as failed in a status field
- Don't silently move on

Better pattern: keep `stopWorkflow` and use a `continueErrorOutput` on a wrapping IF, with the error branch logging + skipping. More work but visible.

---

## Wait + polling chains — different concern

For polling loops (FAL queue API, etc.), use `Switch` not `IF` for status branches. Don't rely on `onError` for "still processing" signals — the API returns 200 with `status: IN_PROGRESS`, that's NOT an error. Route via Switch on `$json.status`.

`onError` is for genuine errors (HTTP 4xx/5xx, network failures, validation failures). Status checking is happy-path branching.

---

## Summary

| Node | `onError` |
|---|---|
| Airtable Create / Update / Upsert | `stopWorkflow` |
| HTTP POST / PUT / PATCH (mutations) | `stopWorkflow` |
| HTTP DELETE | `continueRegularOutput` |
| S3 / Frame.io binary upload | `stopWorkflow` |
| Slack / Email notification | `continueRegularOutput` |
| Logging / Status update side-effects | `continueRegularOutput` |
| Airtable Search / Get (read) | leave default |
| AI agent / LLM node | leave default — failures should halt to surface prompt issues |
