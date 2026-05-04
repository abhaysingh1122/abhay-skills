# HTTP & External Integrations

Patterns for HTTP Request nodes and common external APIs that come up in n8n workflows.

---

## HTTP Request node — the body-format trap

**The most common silent failure:** using `contentType: 'json'` instead of `specifyBody: 'json'`.

n8n silently drops unknown keys, defaults to form-encoded body mode, and sends an empty/wrong body. Receivers return cryptic errors ("422 Unexpected field" with empty detail).

### Correct shape for JSON body

```js
{
  method: 'POST',
  url: '...',
  sendBody: true,
  specifyBody: 'json',                  // ← THIS exact key
  jsonBody: '={ "key": "{{ value }}" }',
  options: {}
}
```

### Correct shape for binary PUT (S3 signed URLs etc.)

```js
{
  method: 'PUT',
  url: '={{ $json.upload_url }}',
  sendBody: true,
  specifyBody: 'binaryData',
  inputDataFieldName: 'data',
  sendHeaders: true,
  headerParameters: {
    parameters: [
      { name: 'x-amz-acl', value: 'private' },
      { name: 'Content-Type', value: 'image/jpeg' }
    ]
  }
}
```

### Correct shape for form-urlencoded

```js
{
  method: 'POST',
  sendBody: true,
  specifyBody: 'form-urlencoded',
  bodyParameters: {
    parameters: [{ name: 'key', value: 'value' }]
  }
}
```

### Correct shape for downloading a file as binary

```js
{
  method: 'GET',
  url: '...',
  options: {
    response: { response: { responseFormat: 'file' } }
  }
}
```

The result has `binary.data` with the file contents (filesystem mode in n8n cloud — see [code-patterns.md](code-patterns.md) for handling).

---

## Always verify the body-format key

Before scripting JSON manipulation that includes body params, inspect an existing working HTTP node in the same workflow:

```js
const httpNodes = wf.nodes.filter(n => n.type === 'n8n-nodes-base.httpRequest' && n.parameters.specifyBody);
console.log(httpNodes.map(n => ({ name: n.name, body: n.parameters.specifyBody })));
```

Match their pattern. If you're getting 422 errors, first re-verify the body key — `contentType` is the most common silent killer.

---

## FAL queue API — polling, not single wait

FAL's `queue.fal.run/...` endpoints are async. POST submits a job and returns:

```json
{
  "request_id": "019deee7-...",
  "status_url": "https://queue.fal.run/.../{id}/status",
  "response_url": "https://queue.fal.run/.../{id}",
  "cancel_url": "..."
}
```

Status flow: `IN_QUEUE` → `IN_PROGRESS` → `COMPLETED`.

### ❌ WRONG (will hit 400 "Request is still in progress")

```
POST → Wait 25s → GET response_url
```

A single wait isn't enough. Generation takes anywhere from 5s (simple image) to 3min (video). 400 errors guaranteed on slow runs.

### ✅ RIGHT — polling loop

```
POST → Wait initial (5-30s) → Get Status → Switch on $json.status:
                                              ├─ COMPLETED → Get Result (response_url)
                                              └─ IN_QUEUE/IN_PROGRESS → Wait Poll (5-15s) → loop back to Get Status
```

n8n implementation: `Switch v3.2` with rule `{{ $json.status }} == "COMPLETED"` (out 0) + fallback "extra" output (out 1) for the polling branch. Wait Poll loops back to Get Status.

### Initial wait sizing
- Image generation (simple): 5s initial, 5s poll
- Image generation (complex / edit): 10-15s initial
- Video generation: 30s initial, 15s poll

### Auth
All FAL endpoints use the same `Authorization: key {key_id}:{secret}` header — same key works across all FAL models.

---

## Image hosting CDN gotchas

### Google Drive image downloads — use lh3, not uc?export=download

When downloading user-supplied Google Drive images:

❌ `https://drive.google.com/uc?export=download&id={id}` — returns raw bytes (often AVIF) with `Content-Type: application/octet-stream`. Many APIs reject.

✅ `https://lh3.googleusercontent.com/d/{id}=w2000` — Google's thumbnail CDN. Returns clean JPEG with proper Content-Type. Bypasses virus-scan interstitials. Resolves AVIF → JPEG transparently.

```js
// In a Code node:
const driveMatch = url.match(/drive\.google\.com\/file\/d\/([a-zA-Z0-9_-]+)/);
if (driveMatch) {
  url = `https://lh3.googleusercontent.com/d/${driveMatch[1]}=w2000`;
}
```

Caveats:
- File must still be set to "Anyone with the link" sharing
- Returns JPEG even if original was PNG (transparency loss)
- `=w2000` caps dimensions

For PNG with transparency, use Drive API with OAuth, or have the client upload to a CDN they control.

### Signed CDN URLs expire

When downloading from signed CDNs (e.g., Kling, AWS S3 signed URLs), check the URL's TTL parameter (`ksTime`, `Expires`, `X-Amz-Expires`). If old, signature has expired → 403.

Strategy: for automated pipelines, download to permanent storage (Drive/S3/Airtable attachment) IMMEDIATELY after generation, save permanent URL — don't keep the upstream signed URL.

For Frame.io `remote_upload` and similar "fetch by URL" endpoints: source URL must be publicly accessible without auth at the time the destination service fetches.

---

## Frame.io V4 API — endpoint structure

All V4 endpoints nest under `/v4/accounts/{account_id}/...`. No flat `/v4/folders/...` paths exist.

### Common endpoints

| Action | Method | Path |
|---|---|---|
| List workspaces | GET | `/v4/accounts/{acct}/workspaces` |
| Create project | POST | `/v4/accounts/{acct}/workspaces/{ws}/projects` |
| Create folder under folder | POST | `/v4/accounts/{acct}/folders/{folder_id}/folders` |
| Create file (binary upload) | POST | `/v4/accounts/{acct}/folders/{folder_id}/files/local_upload` |
| Create file (URL fetch) | POST | `/v4/accounts/{acct}/folders/{folder_id}/files/remote_upload` |
| Get file | GET | `/v4/accounts/{acct}/files/{file_id}` |
| Delete file | DELETE | `/v4/accounts/{acct}/files/{file_id}` |
| Get comment | GET | `/v4/accounts/{acct}/comments/{comment_id}` |
| Register webhook | POST | `/v4/accounts/{acct}/workspaces/{ws}/webhooks` |

### Body shapes

**Create project:**
```json
{ "data": { "name": "..." } }
```
Only `name` required. NO `private` / `restricted` fields (rejected as "Unexpected field").

**Create file (local_upload):**
```json
{ "data": { "name": "...", "file_size": 1234567 } }
```
`file_size` MUST be exact integer bytes. Returns `{ data: { id, upload_urls: [{url, size}, ...] } }`. PUT binary to `upload_urls[0].url` for files <5MB. Larger files chunk across multiple URLs.

**Create file (remote_upload):**
```json
{ "data": { "name": "...", "source_url": "https://..." } }
```
Frame.io's servers fetch the URL and store binary. Source URL must be publicly accessible.

**Register webhook:**
```json
{ "data": { "name": "...", "url": "https://your-server.com/webhook?action=...", "events": ["comment.created"] } }
```
Returns `{ data: { id, secret, ... } }` — `secret` is HMAC signing key for verifying incoming webhooks.

### S3 upload (after local_upload)

PUT to `upload_urls[0].url` with headers:
- `x-amz-acl: private`
- `Content-Type: image/jpeg` (or video/mp4 — must match)

n8n: `specifyBody: 'binaryData'`, `inputDataFieldName: 'data'`.

### DELETE semantics

404 on DELETE = "already gone" = success for idempotent flows. Set `onError: 'continueRegularOutput'` on Frame.io DELETE nodes.

### Auth

OAuth2 via Adobe IMS — Frame.io is now Adobe Frame.io. n8n credential type: `oAuth2Api`.

### Common errors
- `404 no route found for X` — usually URL has empty interpolation OR wrong path
- `422 Unexpected field` — body has unsupported field; remove
- `404 Entity not found` — asset already deleted (handle on DELETE nodes)

---

## Meta Ads / Facebook Graph API

### Token types — only USER tokens (or System User) read ad data

| Token | Lifespan | Reads ad metrics? |
|---|---|---|
| **System User Token** | Never expires | ✅ Best for production cron |
| **Long-lived User Token** | 60 days | ✅ MVP / dev |
| **Short-lived User Token** | 1-2 hours | ⚠️ Initial test only |
| **App Token** | N/A | ❌ Cannot read insights |
| **Page Token** | varies | ❌ Page management only |

App Tokens return `(#100) Cannot retrieve advertiser insights with an app token`.

### Permissions (minimum for ad scraping)
- `ads_read` — read insights, campaigns, ad sets, ads
- `business_management` — list ad accounts under business portfolios
- `ads_management` — write access (optional, for budget edits)
- `read_insights` — page-level insights (optional)

### Extending short-lived → 60 days
Token Debugger has "Extend Access Token" button at the bottom: `developers.facebook.com/tools/debug/accesstoken/`

### System User token (permanent)
1. business.facebook.com → BM Settings → Users → System Users → Add
2. Name as a service identity, Role: Admin
3. Click new System User → Add Assets → multi-select ad accounts → "View performance"
4. Generate New Token → app: any owned app → permissions → Expiration: Never
5. Copy immediately

### The 4-level data hierarchy

```
Ad Account (act_XXXXX)
  └── Campaign (campaign_id)
       └── Ad Set (adset_id)
            └── Ad (ad_id)
```

API call: `GET /v22.0/act_X/insights?level={LEVEL}&fields=...`

| level= | Returns | Use when |
|---|---|---|
| `account` | 1 row aggregate per account | Top-line dashboard |
| `campaign` | N rows (one per campaign) | Per-campaign performance |
| `adset` | M rows (one per ad set) | Audience analysis |
| `ad` | P rows (one per ad) | Creative-level analysis |

At each deeper level, request the parent IDs/names too:
- `level=adset` → fields can include `campaign_id`, `campaign_name`, `objective`, `adset_id`, `adset_name`
- `level=ad` → can include all of those + `ad_id`, `ad_name`

### Auto-discovery of accessible ad accounts

```bash
# All accounts the token user has access to
GET /me/adaccounts?fields=name,id,account_id,account_status,currency,business

# Or scoped to one business portfolio:
GET /me/businesses?fields=name,id                                    # find biz id
GET /{biz_id}/owned_ad_accounts?fields=name,id,account_id           # accounts BM owns
GET /{biz_id}/client_ad_accounts?fields=name,id,account_id          # accounts BM manages for clients
```

### Account status codes
- `1` = ACTIVE
- `2` = DISABLED
- (other codes for closed/pending/grace)

### Budgets are in CENTS

Meta API returns `daily_budget` and `lifetime_budget` as integer strings in cents:
- `"50000"` = €500.00
- Always `parseFloat(daily_budget) / 100` when storing as currency

### Common errors
- `(#100) Cannot retrieve...` → wrong token type OR missing scope
- `(#100) Permission denied for ad account` → token user not assigned to that specific ad account, even if BM admin (ad-account-level perms are separate)
- `190` → token expired

---

## n8n credential creation via Public API

Workflows reference credentials by `id`. To bootstrap a credential alongside a workflow PUT:

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

Response includes `id`. Use it in node config:
```js
credentials: { facebookGraphApi: { id: '<id>', name: 'Friendly Display Name' } }
```

### Discover the data shape for any credential type
```bash
GET /api/v1/credentials/schema/{credential_type}
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
- Update existing credential VALUES (no PATCH on cred values)
- Read existing credential values (security — only via UI)
- List credentials on cloud (404)

For updates, edit via UI. The API is create-only on cloud instances.

### Reusing existing credentials in pushed workflows

```js
const findCred = (nodeName, credType) => {
  const n = wf.nodes.find(x => x.name === nodeName);
  return n?.credentials?.[credType];   // returns { id, name }
};

const AT_CRED = findCred('Some Existing Airtable Node', 'airtableTokenApi');
// then use it on new nodes:
newNode.credentials = { airtableTokenApi: AT_CRED };
```

Avoids creating duplicates when a working credential already exists.
