# Code Patterns — Writing JavaScript in n8n Code Nodes

Patterns for the JS that goes inside Code nodes. These patterns are battle-tested fixes for common silent-failure bugs.

---

## Code node modes — pick the right one

| Mode | When | Globals | Return |
|---|---|---|---|
| `runOnceForAllItems` | batch processing, formatting arrays | `$input.all()` | array of items |
| `runOnceForEachItem` | per-row transforms, binary handling | `$json`, `$binary`, `$itemIndex` | single item |

⚠️ Mode change requires Ctrl+R refresh in n8n editor or it executes with stale config.

For binary (`getBinaryDataBuffer`), per-each-item mode is more compatible with editor cache state. Switching modes without refresh causes stale executions.

---

## Cross-node references — robust pattern

Naive `$('NodeX').item.json.field` breaks when the user clicks "Execute Step" on a downstream node alone — upstream didn't run, ref returns undefined.

**Robust pattern: read from immediate `$json` first, fallback to cross-refs:**

```js
const ctx = $('Some Upstream Node').first()?.json || {};
const upstreamCtx = $('Earlier Node').first()?.json || {};

const currentRecId = $json?.id ?? ctx.frame_record_id ?? upstreamCtx.id ?? null;
const folderId     = $json?.fields?.['Folder ID']
                  ?? ctx.frameio_folder_id
                  ?? null;
```

`$json` is the immediate input from the directly-connected upstream node. It always has data when you Execute Step the current node. So putting `$json` first in the fallback chain makes test-in-isolation work.

**For HTTP node URLs/JSON bodies:**
- Prefer `$json.fields.X` (immediate input) over `$('NodeY').item.json.fields.X`
- Cross-node refs are fine in production runs (full chain) but fragile for isolated testing

**Optional chaining everywhere:**
```js
$('Node').first()?.json?.x ?? defaultValue
```

Without `?.`, accessing `.json` on undefined throws and breaks the whole expression.

---

## Static data accumulators — for collecting across loop iterations

`$items()` and `.all()` do NOT aggregate across SplitInBatches loop iterations in Code nodes. Use `$getWorkflowStaticData('global')` as a workflow-scoped accumulator.

### Pattern: 3 nodes (reset → push → read)

**1. Reset BEFORE the loop:**
```js
const sd = $getWorkflowStaticData('global');
sd.collectedIds = [];
return $input.all();
```

**2. Push INSIDE the loop body** (between body's last node and the loopback):
```js
const sd = $getWorkflowStaticData('global');
if (!Array.isArray(sd.collectedIds)) sd.collectedIds = [];
sd.collectedIds.push($input.first().json.id);
return $input.all();   // pass through unchanged
```

**3. Read + clear AFTER the loop:**
```js
const sd = $getWorkflowStaticData('global');
const ids = Array.isArray(sd.collectedIds) ? sd.collectedIds : [];
sd.collectedIds = [];   // clear for next run
return [{ json: { ids, count: ids.length } }];
```

### CRITICAL: parallel chains must clear ONLY their own keys

If two parallel chains both write to `sd.something`, one chain doing `sd.something = {}` wipes the other.

**❌ WRONG (race condition):**
```js
const sd = $getWorkflowStaticData('global');
sd.uploadedUrls = {};   // wipes entire accumulator including other chain's data
sd.uploadedUrls['key1'] = ...;
```

**✅ RIGHT (surgical clear):**
```js
const sd = $getWorkflowStaticData('global');
if (!sd.uploadedUrls) sd.uploadedUrls = {};
['key1','key2','key3'].forEach(k => delete sd.uploadedUrls[k]);
sd.uploadedUrls['key1'] = ...;
```

Each chain knows its keys. Each chain clears only those. Other chains' data preserved.

### Reset-at-start pattern (alternative)

For workflows that want a clean slate each run without affecting parallel chains:

1. Add a "Reset State" node at the very top, right after trigger, before any fan-out
2. That node does the FULL reset
3. Chains fan out — none of them does another full reset, only key-level deletes

### Race condition warning

Static data is workflow-scoped, NOT execution-scoped. Two simultaneous executions share state. For high-concurrency workflows, prefix keys with execution ID. For typical webhook-triggered sequential runs, this isn't an issue.

---

## Binary data on n8n cloud (filesystem mode)

n8n cloud uses `binaryDataMode: filesystem` — binary stored on disk, NOT base64-embedded in JSON. So:

- `$binary.data.data` is `undefined` in cloud
- `binary.data.fileSize` is a formatted string `"1.22 MB"`, NOT byte count

**To get actual bytes, use the helper:**

```js
// runOnceForEachItem mode
const buffer = await this.helpers.getBinaryDataBuffer($itemIndex, 'data');
const bytes  = buffer.length;
return {
  json:   { ...$json, file_size_bytes: bytes },
  binary: $binary
};
```

**For runOnceForAllItems mode**, loop with index:
```js
const items = $input.all();
const out = [];
for (let i = 0; i < items.length; i++) {
  const buffer = await this.helpers.getBinaryDataBuffer(i, 'data');
  out.push({
    json: { ...items[i].json, file_size_bytes: buffer.length },
    binary: items[i].binary
  });
}
return out;
```

⚠️ **Don't even try** parsing the human-readable `fileSize` string ("1.22 MB" → bytes math) — APIs validate exact byte count.

---

## Safe parsing of AI agent output

AI agents return strings that are usually JSON but can be wrapped in markdown fences or have other artifacts.

```js
const safeParse = (raw) => {
  if (raw && typeof raw === 'object') return raw;       // already parsed
  if (typeof raw !== 'string') return {};
  try {
    // Find first { ... } block (handles markdown fence wrapping)
    const m = raw.match(/\{[\s\S]*\}/);
    return m ? JSON.parse(m[0]) : {};
  } catch (e) {
    return {};
  }
};

const item = $input.first();
const raw  = item.json.output ?? item.json.text ?? '';
const parsed = safeParse(raw);
```

Both `output` and `text` are common output fields depending on the agent type — try both.

---

## Reading AI output with the RIGHT key format

When AI emits kebab-case JSON (matching CMS slugs / Airtable column names), code reading with snake_case silently returns undefined → blank fields downstream.

Always match the format. For kebab-case keys, use bracket notation:

```js
// ❌ WRONG — JS interprets hyphen as subtraction
const value = parsed.sec1-h1-headline;

// ✅ RIGHT — bracket notation
const value = parsed['sec1-h1-headline'];
```

**Helper for clean repeated reads:**
```js
const C = (k) => (parsed[k] ?? '').toString();

const fieldData = {
  'sec1-h1-headline': C('sec1-h1-headline'),
  'sec1-h2-subheadline': C('sec1-h2-subheadline'),
  // ...
};
```

⚠️ During development, throw on missing keys to surface bugs:
```js
const required = ['key1', 'key2', 'key3'];
for (const k of required) {
  if (parsed[k] === undefined) throw new Error(`Missing key: ${k}. Got: ${Object.keys(parsed).join(', ')}`);
}
```

After deployment is stable, soften to defaults — but the throw saves hours of "why is everything blank" debugging.

---

## Adding diagnostic fields to Code node output

When debugging, emit `_diag_*` fields so executions surface what was attempted:

```js
return [{
  json: {
    logoUrl: transformedUrl,
    slug,
    companyName,
    hasUrl: !!transformedUrl,
    _diag_originalUrl: originalUrl,
    _diag_transformPattern: matchPattern,
    _diag_transformedTo: 'lh3-thumbnail'
  }
}];
```

Diagnostics flow downstream and appear in execution logs. Easy to remove later, invaluable while building.

---

## Stripping markdown fences from AI output

If the AI insists on wrapping JSON in ```fences```:

```js
const stripFences = (s) => {
  if (typeof s !== 'string') return s;
  return s.replace(/^```(?:json)?\s*\n?/, '').replace(/\n?```\s*$/, '').trim();
};
const raw = stripFences(item.json.output);
const parsed = JSON.parse(raw);
```

Combined with `safeParse` above gives belt + suspenders.

---

## Defensive type coercion for metrics

API responses are inconsistent — Meta/etc. sometimes return numbers as strings, sometimes as numbers, sometimes null.

```js
const safeNum = (v) => v != null ? parseFloat(v) || 0 : 0;
const safeInt = (v) => v != null ? parseInt(v) || 0 : 0;
const safeStr = (v) => (v ?? '').toString();
```

Use everywhere you read external API data.

---

## Per-iteration output formatting (multi-level data)

When merging multiple data sources at different "levels" (e.g., account-level + campaign-level Meta insights):

```js
const client = $('Loop Over Clients').item.json;
const accData = $('FB Insights Account').first()?.json?.data || [];
const campData = $('FB Insights Campaign').first()?.json?.data || [];

const baseRow = (d) => ({
  client_name: client['Client Name'] || '',
  account_id: client['account id'] || '',
  date: d.date_stop || (new Date(Date.now() - 86400000).toISOString().slice(0, 10)),
  spend: safeNum(d.spend),
  impressions: safeInt(d.impressions),
  // ...
});

const rows = [];
for (const d of accData)  rows.push({ json: { ...baseRow(d), level: 'account', campaign_name: '', campaign_id: '' } });
for (const d of campData) rows.push({ json: { ...baseRow(d), level: 'campaign', campaign_name: d.campaign_name || '', campaign_id: d.campaign_id || '' } });

// Always emit at least one row to keep downstream chain alive
if (rows.length === 0) {
  rows.push({ json: { ...baseRow({}), level: 'account', campaign_name: '', campaign_id: '' } });
}
return rows;
```

---

## Don't put a Code node BEFORE an AI agent

This is an architecture principle but worth repeating in the code section:

```
✅ Get tables → AI Agent (reads $json + cross-refs) → Code (format AI JSON to schema) → Save
❌ Get tables → Code "Build Context" → AI Agent → ...
```

Pre-processing context with code BEFORE the AI defeats the AI's reasoning, introduces field-mapping bugs, and is rarely needed. AI agents are smart enough to handle raw multi-table input.

---

## Always-emit-something rule

If your Code node could legitimately receive empty input (zero items from upstream), explicitly emit a placeholder to keep the chain alive:

```js
if (rows.length === 0) {
  rows.push({ json: { _empty: true, _reason: 'no data from upstream' } });
}
return rows;
```

Otherwise downstream nodes get no input and the chain quietly stops.
