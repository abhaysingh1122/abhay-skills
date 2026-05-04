# Debugging — Common Bugs & How to Find Them

When something's mysteriously broken in an n8n workflow, look here first. Each entry: **Symptom → Root cause → Fix.**

---

## 🐛 "Nothing changed after I pushed"

**Symptom:** User runs the workflow after your API push, gets the old behavior or an error referencing fields you've already changed.

**Root cause #1: User didn't refresh the editor**
n8n caches node config from when the tab was opened. API push updates the server but the editor tab still has the old config.

**Fix:** *"Refresh n8n (Ctrl+R / Cmd+R) before testing."* — say this after every push.

**Root cause #2: PUT silently dropped fields**
n8n's PUT returns 200 even when unknown fields are silently dropped. Your change "succeeded" but didn't persist.

**Fix:** Always GET after PUT and grep for the specific change. See [api-discipline.md](api-discipline.md).

**Root cause #3: PUT body had `settings` with extra properties**
PUT response was 400 but you didn't notice — strip settings to safe fields.

---

## 🐛 "Data is mysteriously missing in Airtable"

**Symptom:** Workflow runs, no errors, but the Airtable row has empty fields where you expected data.

**Root cause #1: Key format mismatch (kebab vs snake)**
AI emits `'sec1-h1-headline'`, code reads `copy.sec1_h1_headline` → undefined → defaulted to empty string.

**Fix:** Read with bracket notation: `copy['sec1-h1-headline']`. See [code-patterns.md](code-patterns.md) and [ai-agents.md](ai-agents.md).

**Root cause #2: Get vs Search operation mismatch**
Switched a node from Search → Get. Downstream `.fields.X` accesses now return undefined because Get returns flat (`$json.X`), not nested.

**Fix:** Audit downstream expressions when changing operations. See [airtable.md](airtable.md).

**Root cause #3: Static data accumulator wiped by parallel chain**
One chain saved data to `sd.foo`, another chain did `sd.foo = {}` and wiped it before the consumer read.

**Fix:** Each chain clears ONLY its own keys, never `sd.foo = {}`. See [code-patterns.md](code-patterns.md).

**Root cause #4: AI agent silently returned empty**
Look at the raw AI output in the execution log. If it's empty/garbled, the agent didn't follow instructions — restructure the prompt (temp=0, decision procedure, etc.).

---

## 🐛 "AI agent ignores my rules"

**Symptom:** Agent emits unwanted fields, ignores explicit "do not include X" instructions, hallucinates values.

**Root causes (in order to check):**

1. **Temperature ≠ 0** → set `options: { temperature: 0 }` on the chat model
2. **Rules in prose, not as a numbered decision procedure** → rewrite as STEP A / STEP B / STEP C
3. **No anti-examples (❌)** → add 4-5 ❌ examples for the failure modes you've seen
4. **No worked examples** → add 4-5 input → expected JSON output examples
5. **Code node downstream re-injects context** → make the consumer pure pass-through; agent has SOLE AUTHORITY

If you're on iteration 3+ of the same prompt and still failing, **STOP tweaking words.** Apply the full structural fix. See [ai-agents.md](ai-agents.md).

---

## 🐛 "Loop only saved the last iteration's data"

**Symptom:** SplitInBatches loop processes N items, but downstream only sees data from the last iteration.

**Root cause:** Used `$('Node').all()` or `$items()` in a Code node trying to aggregate — these don't aggregate across loop iterations.

**Fix:** Static data accumulator pattern. Reset before loop, push inside loop, read after. See [code-patterns.md](code-patterns.md).

---

## 🐛 "Cross-node ref returns null when running Execute Step"

**Symptom:** Test "Execute Step" on a downstream node, expressions like `$('UpstreamNode').first().json.x` return null/undefined.

**Root cause:** Cross-node refs only resolve if that node ran in the current execution. Execute Step in isolation doesn't run upstream.

**Fix:** Read from immediate `$json` first, fallback to cross-refs:
```js
const value = $json?.fields?.x ?? $('Upstream').first()?.json?.x ?? defaultValue;
```

Or trigger the full chain end-to-end instead of testing in isolation. See [code-patterns.md](code-patterns.md).

---

## 🐛 "Empty body sent to external API"

**Symptom:** External API (Frame.io, etc.) returns 422 "Unexpected field" with empty detail. n8n HTTP node's body looks correct in the editor.

**Root cause:** Used `contentType: 'json'` instead of `specifyBody: 'json'`. n8n silently drops the unknown key, defaults to form-encoded mode, sends empty body.

**Fix:** Always `specifyBody: 'json'`. See [http-and-external.md](http-and-external.md).

---

## 🐛 "Frame.io / S3 PUT fails with 412 PreconditionFailed or 403"

**Symptom:** Binary upload fails after register-asset succeeded.

**Root causes:**

1. **Missing `x-amz-acl: private` header** on the PUT
2. **Wrong `Content-Type` header** (must match actual file type)
3. **`file_size` mismatch** in the prior register-asset call (must be EXACT bytes — see binary handling)
4. **Source URL expired** (signed CDN URL with TTL passed)

**Fix:** Use `this.helpers.getBinaryDataBuffer($itemIndex, 'data')` to get exact byte count for `file_size`. Set both x-amz-acl and Content-Type headers. See [code-patterns.md](code-patterns.md) for binary handling.

---

## 🐛 "Webflow asset upload fails (412)"

**Symptom:** Drive image download → Webflow asset register succeeds → S3 PUT returns 412.

**Root cause:** File is in AVIF format with `Content-Type: application/octet-stream`. Webflow rejects.

**Fix:** Use `lh3.googleusercontent.com/d/{id}=w2000` instead of `uc?export=download` — Google's thumbnail CDN auto-converts to JPEG. See [http-and-external.md](http-and-external.md).

---

## 🐛 "Logo / image bound to dynamic CMS field but doesn't render"

**Symptom:** CMS item has the image URL stored, Designer page renders blank where the image should be.

**Root cause:** SVG element instead of Image element. SVGs can't bind to CMS Image fields — need an `<img>` tag.

**Fix:** Replace SVG with Image element in Designer, then bind via Settings → Image source → purple dot.

---

## 🐛 "Theme / brand color not changing per CMS item"

**Symptom:** CMS field has the color value, page elements all use the same hardcoded color.

**Root cause:** Designer elements aren't bound to the CMS color field. The value sits in DB unused.

**Fix:** Bind via Style panel → Background Color → purple dot → Get color from CMS. Bind 5 elements (CTAs, eyebrow text, accents) for theme variation.

---

## 🐛 "Conditional visibility hides card but layout breaks"

**Symptom:** Card hides correctly when its CMS field is empty, but layout has ugly gap where the card was.

**Root cause:** Parent container is `display: grid` with fixed columns. Hiding a child leaves an empty grid cell.

**Fix:** Switch parent to `display: flex` with `wrap: wrap` and `justify: center`. Visible cards re-center automatically.

---

## 🐛 "FAL queue API returns 400 'Request still in progress'"

**Symptom:** GET response_url after Wait → 400 error.

**Root cause:** Single Wait isn't enough. Async generation can take 5s to 3min.

**Fix:** Polling loop — POST → Wait → Get Status → Switch (COMPLETED → done; else → Wait Poll → loop back). See [http-and-external.md](http-and-external.md).

---

## 🐛 "Meta Graph API returns 'Cannot retrieve advertiser insights'"

**Symptom:** GET /act_X/insights returns `(#100) Cannot retrieve advertiser insights with an app token`.

**Root cause:** Token type is App Token. App Tokens can't read ad data — only User Tokens or System User Tokens can.

**Fix:** Generate a User Access Token in Graph API Explorer with proper scopes. Extend to 60 days via Token Debugger. See [http-and-external.md](http-and-external.md).

---

## 🐛 "Meta Graph API returns 'Permission denied for ad account'"

**Symptom:** Token works for some ad accounts, fails for others with `(#100) Permission denied`.

**Root cause:** Token user not assigned to that specific ad account, even if BM admin. Ad-account-level perms are SEPARATE from BM-level perms.

**Fix:** Assign the user to each ad account in Business Settings → Ad Accounts → Add People. OR use a System User token with all ad accounts assigned via "Add Assets".

---

## 🐛 "Airtable Meta API returns 403 INVALID_PERMISSIONS_OR_MODEL_NOT_FOUND"

**Symptom:** Schema operations on a specific base return 403 even though PAT has all scopes.

**Root cause:** PAT doesn't have explicit base access. Airtable PATs need BOTH scopes AND a list of allowed bases.

**Fix:** Edit existing PAT at airtable.com/create/tokens → Access section → Add the base. No regeneration needed. See [airtable.md](airtable.md).

---

## 🐛 "AutoNumber-based 'previous frame' lookup returns wrong record"

**Symptom:** Cascade flow processing frame N tries to look up frame N-1 by autoNumber math, gets wrong record (different project).

**Root cause:** Airtable autoNumber is GLOBAL across the table. Frame N=11 might be a different project's first frame, not the previous frame in this project.

**Fix:** Sort the parent's `Video_Frames` linked-record array by autoNumber ASC, then `array[currentIndex - 1]`. Don't do autoNumber math directly. See [airtable.md](airtable.md).

---

## 🐛 "linkedRecord field filter with FIND/ARRAYJOIN returns 0"

**Symptom:** `filterByFormula: FIND("recXXX", ARRAYJOIN({Projects}))` always returns 0.

**Root cause:** Airtable linked-record fields, when referenced as `{Field}` in formulas, return primary field VALUES not record IDs. So ARRAYJOIN returns the display name, not "recXXX".

**Fix:** Use Split Out + per-item Get instead of filterByFormula. OR add a Lookup field that pulls Record ID. See [airtable.md](airtable.md).

---

## 🐛 "Mutation node failed silently, downstream got empty data"

**Symptom:** Inconsistent results across runs — some rows correct, some duplicates, some stale.

**Root cause:** Mutation nodes have `onError: 'continueRegularOutput'` — silent failures pass empty input to downstream.

**Fix:** Set Create/Update/PUT/POST nodes to `onError: 'stopWorkflow'`. Only DELETE and notification nodes should be `continueRegularOutput`. See [error-handling.md](error-handling.md).

---

## 🐛 "Wait node doesn't fire after sleep"

**Symptom:** Workflow halts at Wait, never resumes.

**Root cause:** Missing `webhookId` on the Wait node — n8n needs it to track the wait state.

**Fix:** Set `webhookId: '<uuid>'` on every Wait node. See [node-shapes.md](node-shapes.md).

---

## 🐛 "$binary.data.data is undefined"

**Symptom:** Code node trying to access `$binary.data.data` returns undefined on n8n cloud.

**Root cause:** n8n cloud uses filesystem mode for binary. Bytes aren't in the JSON object — they're on disk.

**Fix:** Use the helper:
```js
const buffer = await this.helpers.getBinaryDataBuffer($itemIndex, 'data');
```

See [code-patterns.md](code-patterns.md).

---

## 🐛 "Agent node says 'Model not supported'"

**Symptom:** Red banner on `langchain.agent` node — "model not supported in version N of the Agent node."

**Root cause:** Agent typeVersion is too old for the model. Common with `gpt-4.1-mini` on agent v2.

**Fix:** Bump `typeVersion: 3` on the agent node. Don't change the model — the model is fine. See [node-shapes.md](node-shapes.md).

---

## 🐛 "IF node downstream gets null when one branch ran"

**Symptom:** IF's TRUE/FALSE branches both feed into a Set node. When the FALSE branch runs, the Set node tries to access data from the TRUE branch's upstream → null.

**Root cause:** When IF's TRUE/FALSE branches both feed into the same downstream node, that node's `$json` is the input from whichever branch fired — NOT a merge of both. Cross-refs to the unfired branch's upstream throw.

**Fixes (in order of preference):**
1. Restructure to single linear path — eliminate the IF if business logic allows
2. Optional chaining + nullish: `$('NodeOnTrue').first()?.json?.x ?? $('NodeOnFalse').first()?.json?.y`
3. Use a Merge node — explicit n8n Merge combines branches into one item

---

## 🛠️ Universal diagnostic tools

### Dump raw AI output
Add a temporary Code node:
```js
const raw = $input.first().json.output ?? $input.first().json.text ?? '';
console.log('RAW AI OUTPUT:', JSON.stringify(raw).slice(0, 1000));
return [$input.first()];
```
Run once, check execution log, remove after fixing.

### Inspect static data state
```js
console.log('STATIC DATA:', JSON.stringify($getWorkflowStaticData('global'), null, 2));
return $input.all();
```

### Test what bases an Airtable PAT can see
```bash
curl -sS -H "Authorization: Bearer $PAT" "https://api.airtable.com/v0/meta/bases"
```

### Check Meta token validity + scopes
```bash
curl -sS "https://graph.facebook.com/v22.0/debug_token?input_token=$TOKEN&access_token=$TOKEN"
```

### Re-fetch n8n workflow + diff against expected
```bash
curl -sS -H "X-N8N-API-KEY: $KEY" "$URL/api/v1/workflows/$WID" -o current.json
node -e "const j=JSON.parse(require('fs').readFileSync('current.json')); const t=j.nodes.find(n=>n.name==='X'); console.log(JSON.stringify(t.parameters,null,2));"
```

### Fetch latest execution to see what actually happened
```bash
curl -sS -H "X-N8N-API-KEY: $KEY" "$URL/api/v1/executions?workflowId=$WID&limit=1&includeData=true" -o exec.json
```

The execution data shows each node's actual input + output → invaluable for diagnosing flow issues.

---

## When in doubt

1. Re-fetch the workflow JSON, look at the actual saved node config
2. Run an execution, fetch its data, look at each node's input/output
3. Add `_diag_*` fields to Code node outputs for visibility
4. Tell the user to refresh their n8n tab and try again
5. If still stuck, dump raw AI output and check the actual key format

Most "mysterious" bugs are one of: editor cache, key format mismatch, static data race, or silent error swallowing. Check those four first.
