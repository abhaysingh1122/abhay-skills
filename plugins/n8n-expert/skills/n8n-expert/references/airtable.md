# Airtable in n8n — All the Patterns

How to integrate Airtable cleanly. The single most common source of silent failures.

---

## Get vs Search vs Update — different return shapes

The same Airtable record returns DIFFERENT JSON shapes depending on operation:

### Get (by record ID)
```json
{
  "id": "recXXX",
  "createdTime": "...",
  "Field A": "...",          ← fields at ROOT
  "Linked Records": ["recY"]
}
```
Access: `$json['Field A']`, `$json.fields_at_root`

### Search / Update / Upsert (filterByFormula or batch)
```json
{
  "id": "recXXX",
  "createdTime": "...",
  "fields": {                ← fields NESTED under .fields
    "Field A": "...",
    "Linked Records": ["recY"]
  }
}
```
Access: `$json.fields['Field A']`

⚠️ **Switching one to the other breaks every downstream expression.** Before changing operation:
1. Audit every reference to the upstream node's output
2. If they use `.fields.X`, current operation is Search/Update/Upsert → keep it in that family
3. If switching to Get, update all downstream paths to drop `.fields.`
4. Easiest fix when downstream breaks: revert to Search with a 1-result filter rather than rewrite N expressions

**Diagnostic clue:** TypeError "Cannot read properties of undefined (reading 'X')" on `.fields.X` access usually means upstream is Get returning flat.

---

## Sequential fetches via linked records — never parallel Merge

When you need data from two linked Airtable tables:

```
✅ Webhook → Get Table A by ID → use linked field $json.fields['Linked'][0] to get B's id → Get Table B → AI agent reads BOTH
❌ Webhook → Get A          ┐
            → Get B          ┤→ Merge → AI agent
            → Get C          ┘
```

Why sequential:
- Tables are connected via linked record fields — chain mirrors the structure
- Single execution path = easier debugging
- No Merge timing surprises

```js
// In AI agent prompt or downstream:
{{ $json.fields['Field From Table B'] }}
{{ $('Get Table A').item.json.fields['Field From Table A'] }}
```

---

## Trigger pattern — Airtable button URLs

Airtable interface buttons trigger via **GET request with query params**, NOT POST with body:

```
{webhook_url}?action=DoSomething&recordId={record_id}
```

In webhook node: enable `multipleMethods: true` + accept GET.
In Switch node: route by `$json.query.action`.

For each Airtable button, the URL pattern is the same — only `action` differs.

---

## Status update lives on the TRIGGERING table

When a workflow runs from Table A's button, update Table A's status fields (e.g., `Workflow Status`, `Last Synced`), NOT downstream tables. Keeps state visible to whoever triggered it.

---

## Search by stable identifier, not record ID

Search by email, slug, or domain — never by record ID. Record IDs change between environments; emails/slugs are stable references the user can paste.

```
Search filterByFormula: ={Email} = "{{ $json.body.email }}"
```

For "find then create or update" pattern (upsert), use the v2.1 `upsert` operation directly — handles search-then-create-or-update in one node.

---

## Upsert pattern (v2.1 native)

```js
{
  operation: 'upsert',
  base: { __rl: true, value: 'appXXX', mode: 'list' },
  table: { __rl: true, value: 'tblXXX', mode: 'list' },
  columns: {
    mappingMode: 'autoMapInputData',
    matchingColumns: ['ID Column']   // primary key for matching
  },
  options: { typecast: true }
}
```

Each item in the input array gets create-or-update by matching key. Time-series data (one row per entity per day) uses CREATE only — never upsert against Date alone.

---

## Linked record fields — multipleRecordLinks

When writing a linked record from a Code node:

```js
return [{
  json: {
    'Field A': 'value',
    'Linked Records': ['recXXXXX']   // array of record IDs (even for single)
  }
}];
```

Or via expression in Airtable Update node:
```
"=[\"{{ $json.id }}\"]"
```

Note the array wrapper — even single linked records use array syntax.

---

## AutoNumber is GLOBAL, not per-anything

`autoNumber` fields are sequential **across the entire table**, not per-project or per-anything. So:
- Project A's frames might have IDs `[1, 2, 3, 4, 5]`
- Project B added later: IDs `[6, 7, 8, 9, 10]`
- Patrick later adds another frame to Project A: ID `11`
- Project A's frames are now `[1, 2, 3, 4, 5, 11]` — autoNumber gap

A `multipleRecordLinks` field preserves the **add/drag order** in the UI — NOT autoNumber order. So `Projekte.Video_Frames[]` may give frames in arbitrary order.

**Compounded result:** iterating `Projekte.Video_Frames[]` directly = jumbled output.

### The sort pattern (when sequence matters)

```
Get parent record
  → Split Out (split linked-record array into N items)
  → Get child record (Airtable Get for each — returns flat shape)
  → Sort by .fields.ID asc (Sort node)
  → [downstream operates in correct sequence]
```

For "previous frame" lookup in cascades: use the sorted list and `array[currentIndex - 1]`, NOT autoNumber math (breaks across project gaps).

For lookup BY autoNumber: `filterByFormula: ={ID} = {{ value }}` works because autoNumber is unique.

---

## Filter linked records — NEVER use FIND + ARRAYJOIN

Airtable's `multipleRecordLinks` field, referenced as `{Field}` inside `filterByFormula`, returns each linked record's **primary field VALUE** (the display name), NOT the record ID.

```
❌ FIND("recXXX", ARRAYJOIN({Projects}))
   → ARRAYJOIN({Projects}) is "test 101", not "recXXX"
   → FIND returns 0 (failure) every time
```

**Two correct alternatives:**

### A) Skip filterByFormula — use Split Out
1. Get parent record
2. n8n Split Out node: split the linked field array
3. Per-item Airtable Get by `={{ $json.linkedFieldName }}` to fetch full child record

### B) Add a Lookup field
On the child table, add a Lookup field that pulls the parent's `Record ID` formula field. Then `FIND` against that lookup field.

---

## Airtable Meta API — schema management

For programmatic schema changes (add fields, create tables):

### Auth
```bash
Authorization: Bearer pat...
```
PAT must have `schema.bases:write` scope (for writes) or `schema.bases:read` (for GET).

### Get base schema
```bash
GET https://api.airtable.com/v0/meta/bases/{baseId}/tables
```

### Add a field
```bash
POST https://api.airtable.com/v0/meta/bases/{baseId}/tables/{tableId}/fields
{
  "name": "Field Name",
  "type": "checkbox" | "singleSelect" | "singleLineText" | ...,
  "options": { ... }   // required for some types
}
```

### Rename a field
```bash
PATCH https://api.airtable.com/v0/meta/bases/{baseId}/tables/{tableId}/fields/{fieldId}
{ "name": "New Field Name" }
```

Renames are SAFE — Airtable internally uses field IDs, automations reference IDs not names.

### Create a new table
```bash
POST https://api.airtable.com/v0/meta/bases/{baseId}/tables
{
  "name": "Table Name",
  "description": "...",
  "fields": [...]
}
```

### Common field options shapes

**Checkbox:**
```json
{ "name": "Done", "type": "checkbox", "options": { "icon": "check", "color": "greenBright" } }
```
Icons: `check`, `xCheckbox`, `star`, `heart`, `thumbsUp`, `flag`
Colors: `redBright`, `orangeBright`, `yellowBright`, `greenBright`, `blueBright`, `cyanBright`, `purpleBright`, `pinkBright`, `grayBright`

**SingleSelect:**
```json
{
  "name": "Status",
  "type": "singleSelect",
  "options": { "choices": [{ "name": "Active", "color": "greenBright" }, ...] }
}
```

**MultipleRecordLinks:**
```json
{
  "name": "Related",
  "type": "multipleRecordLinks",
  "options": {
    "linkedTableId": "tblXXX",
    "isReversed": false,
    "prefersSingleRecordLink": false
  }
}
```

**AutoNumber:**
```json
{ "name": "ID", "type": "autoNumber" }
```

**Formula:**
```json
{ "name": "Record ID", "type": "formula", "options": { "formula": "RECORD_ID()" } }
```

**Lookup:**
```json
{
  "name": "Parent Name",
  "type": "multipleLookupValues",
  "options": {
    "recordLinkFieldId": "fldXXX",
    "fieldIdInLinkedTable": "fldYYY"
  }
}
```

**DateTime (with timezone):**
```json
{
  "name": "Last Synced",
  "type": "dateTime",
  "options": {
    "dateFormat": { "name": "iso" },
    "timeFormat": { "name": "24hour" },
    "timeZone": "Europe/Berlin"
  }
}
```

**Currency:**
```json
{ "name": "Spend", "type": "currency", "options": { "precision": 2, "symbol": "€" } }
```

### Common errors
- `403 INVALID_PERMISSIONS_OR_MODEL_NOT_FOUND` → PAT missing scope OR base not in PAT's access list
- `422 INVALID_FIELD_TYPE` → type string typo (case-sensitive: `singleSelect` not `singleselect`)
- `422 INVALID_OPTIONS` → checkbox/singleSelect missing required `options`
- `422 DUPLICATE_FIELD_NAME` → field with that name already exists

---

## Airtable PAT scoping — two-dimensional permission

Airtable PATs need BOTH:
1. **Scopes** — `data.records:read/write`, `schema.bases:read/write`
2. **Base access** — explicit list of which bases the PAT can touch

A PAT with all scopes but no bases listed returns 403 on those bases. The fix is to edit the PAT and add the base, NOT regenerate.

```
airtable.com/create/tokens → Edit existing token → + Add a base
```

Token value stays the same.

If the base is in someone else's workspace, only THAT workspace owner can grant access — your PAT can't reach it no matter what.

---

## Diagnostic curl

Test what bases a PAT can see:
```bash
curl -sS -H "Authorization: Bearer $PAT" "https://api.airtable.com/v0/meta/bases"
```

Returns `{"bases": [...]}`. If a base isn't in that list, it's not in the PAT's access scope.

---

## Discipline summary

1. Always `GET /v0/meta/bases/{base}/tables` before editing schema — caches go stale
2. Field IDs (`fldXXX`) are stable — use them in workflows, not field names
3. Renaming fields is safe (automations use IDs)
4. Deleting fields is destructive — never do without explicit user confirmation
5. After schema changes, n8n Airtable nodes still work because they store `__rl.value` IDs. But cached `cachedResultName` in editor may show old name — refresh helps.
