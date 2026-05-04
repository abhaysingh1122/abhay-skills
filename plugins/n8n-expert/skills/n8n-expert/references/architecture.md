# Architecture — How to Shape an n8n Workflow

How to design the bones of an n8n workflow before writing any node config.

---

## The default architecture: Webhook → Switch → Branches

Every system starts here unless there's a strong reason otherwise.

```
[Webhook]              ← single entry point, multipleMethods: true
    ↓
[Switch]               ← routes by $json.query.action OR $json.body.action / body.type
    ├─→ Branch 1: action="Function A"
    ├─→ Branch 2: action="Function B"
    ├─→ Branch 3: action="Function C"
    └─→ Branch N: action="<New Function>"   ← extensions go HERE, not as new workflows
```

### Why one webhook, not many

- One URL to share with external triggers (Airtable buttons, automations, Slack, third-party services)
- All routing logic visible in a single Switch
- New features additive — don't modify existing branches
- Easy to see the whole system in one view

### When you SHOULD branch into a separate workflow

Only when:
- The trigger is a fundamentally different external system (e.g., a webhook from a third-party SaaS that posts a totally different payload)
- The new function would need totally different credentials and there's no benefit to colocation
- The Switch has 20+ rules and is becoming unscannable (rare; use sticky notes instead)

Even then — prefer a `body.type` field in the same Switch over a new webhook.

---

## Adding a new branch to an existing Switch (the canonical pattern)

```
1. Inspect Switch.parameters.rules.values — note the routing field ($json.query.action, etc.)
2. Append a new rule with unique outputKey + matching condition
3. Build the new chain (FB API call → Code → Airtable → ...)
4. Wire connections[SwitchName].main[N] (where N = new rule's index) to first node of new chain
5. Add a sticky note explaining the trigger pattern (action="X" → ...)
6. Disable chain nodes if external credentials aren't ready — never push something that breaks production
```

---

## Trigger types and when to use each

| Trigger | Use case |
|---|---|
| **Webhook** (multipleMethods, GET+POST) | Airtable button URLs, external webhooks, manual HTTP calls |
| **Schedule Trigger** | Daily/hourly recurring sync jobs |
| **Form Trigger** | New client onboarding (n8n hosts the form) |
| **Manual Trigger** | Dev/test only — never production |

Webhook + Schedule are 90% of cases. Form Trigger is sometimes nice for client-facing onboarding.

---

## Data flow shape (the canonical chain)

```
Webhook/Trigger
    ↓
Fetch context (1 or more sequential Airtable Gets, NOT parallel Merge)
    ↓
[Optional] HTTP fetches (Apify, Firecrawl, etc.)
    ↓
AI Agent (reads ALL raw data via $json + cross-refs)
    ↓
Code node (formats AI JSON to destination schema)
    ↓
Airtable / Webflow / external write
    ↓
Status update / Slack alert / loop back / done
```

### Don't put a Code node BETWEEN fetches and the AI

A "Build Context" Code node before the AI agent is an anti-pattern. It pre-decides what context the AI gets, introduces field-mapping bugs, and defeats the AI's reasoning. Feed everything raw. Let the AI extract what it needs.

### DO put Code nodes AFTER the AI

Format the AI's JSON output to match the destination schema (CMS field slugs, Airtable column names). This is fine and often necessary.

---

## Sequential Airtable fetches via linked records (no parallel Merge)

When you need data from multiple linked Airtable tables:

```
✅ Webhook → Get Table A by ID → Use linked-record field to get Table B's ID → Get Table B → AI agent reads $('Get Table A').item.json + $json
❌ Webhook → Get Table A     ┐
            → Get Table B    ┤→ Merge → AI agent (relies on Merge to combine)
            → Get Table C    ┘
```

Why sequential beats parallel-Merge:
- Tables are connected via linked record fields — the chain mirrors the data structure
- Single execution path = easier debugging
- No Merge timing surprises

**Trigger pattern for Airtable interface buttons:** GET request with `query.action` + `query.recordId`. NOT POST with body.

**Status updates** go on the TRIGGERING table (the one whose button was pressed), not downstream tables.

---

## Loops with SplitInBatches

```
[upstream] → SplitInBatches (batchSize=1) → [body chain] → [last body node connects BACK to SplitInBatches]
                ↓ done
            [downstream after loop completes]
```

Output indexing:
- `main[0]` = "done" — fires after all items processed, drives downstream
- `main[1]` = "loop body" — fires per item, drives the body chain

The body chain's last node MUST connect back to SplitInBatches at index 0 to continue iterating.

For collecting outputs across iterations, use static-data accumulator (see [code-patterns.md](code-patterns.md)).

---

## Common chain shapes

### Per-record AI generation
```
Trigger → Get record by ID → Get linked records → AI Agent → Code (parse) → Update record → Status update
```

### Time-based scrape + analyze
```
Schedule (daily) → Search records (Status=Active) → SplitInBatches loop:
    → Fetch external data (HTTP/Graph API)
    → Code (combine/format)
    → AI Analyzer (per row)
    → Airtable Create
    → If high severity → Slack
    → loop back
```

### Multi-table upsert + insights
```
Schedule → Search clients → Loop:
    → Fetch entities (campaigns, ad sets, ads)
    → Code (format) → Airtable Upsert (matchingColumns)
    → Fetch metrics at multiple levels
    → Code (combine) → AI Analyzer
    → Airtable Create (time-series)
    → loop back
```

---

## Sticky notes — use them

A workflow with 20+ nodes becomes unreadable without section labels. Drop sticky notes:

- Above each Switch route to label what action it handles
- At the start of each major chain
- On non-obvious patterns (e.g., "static data accumulator — clears nav-company-logo at start of run")
- On disabled chains explaining why they're disabled

Sticky note color conventions: gray=info, yellow=warning, green=working, red=broken/needs attention.

---

## Workflow naming

- Workflow name: descriptive, includes client/project context. e.g., "[ClientName] — Meta Ads Dashboard Sync"
- Avoid version numbers in names ("v2", "v3 final final"). Use git/version-control on the JSON if you need versioning.
- One system = one workflow per logical concern. Don't split arbitrarily.

---

> *Reference: see [api-discipline.md](api-discipline.md) for how to build all this via the API.*
