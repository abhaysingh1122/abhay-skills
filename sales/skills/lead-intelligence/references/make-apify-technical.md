# Make.com + Apify Technical Reference

> **FALLBACK PATH ONLY.** This reference is for the Make.com fallback path. The PRIMARY path for LinkedIn scraping is the native Apify MCP connector (using `call-actor`, `get-dataset-items`, etc. directly). Only use Make.com if the native Apify connector is not working.

Detailed setup, configuration, and troubleshooting guide for running LinkedIn scraping via Make.com scenarios with Apify actors.

## Check for Existing Scenarios and Tools First

Before creating anything new, always check what already exists to avoid duplication.

1. Run `scenarios_list` for the user's team and scan names/descriptions for anything related to Apify, LinkedIn scraping, or lead intelligence (e.g., a scenario named "Run Apify Scrapers")
2. If a matching scenario exists, use `scenarios_get` to inspect its blueprint. Check if it has the right actors configured (A3cAPGpwBEG8RJwse for posts, 2SyF0bVxmgGr8IVCZ for profiles). If so, just update the input URLs in the mapper and reuse it.
3. Before creating fetch tools, check existing tools by looking at recent scenario runs. If a dataset-fetch tool already exists, reuse it with the new dataset ID.

Only create new scenarios or tools when nothing suitable exists. If you find a partial match (e.g., a scenario with only the posts actor), update the existing one.

## Setting Up the Scenario (only if none exists)

1. Create a new scenario on the user's team using `scenarios_create`
2. Build a blueprint with two parallel routes off a Router, both triggered by a BasicTrigger:
   - **Route 1**: Apify Run Actor (posts actor A3cAPGpwBEG8RJwse)
   - **Route 2**: Apify Run Actor (profile actor 2SyF0bVxmgGr8IVCZ)

## Blueprint Configuration Details

The Apify "Run Actor" module needs these mapper fields:
- `actorId`: the actor ID string
- `inputBody{actorId}`: a JSON string of the input (the field name includes the actor ID with no spaces)
  - Example: `inputBodyA3cAPGpwBEG8RJwse` for the posts actor
  - Example: `inputBody2SyF0bVxmgGr8IVCZ` for the profile actor
- `actor_search_type`: "store"

The input body must be a JSON STRING (not an object), containing all LinkedIn URLs from the qualified leads list.

### Common Mistakes to Avoid

- Do NOT use `inputBody` as the field name. It MUST be `inputBody` + the exact actor ID concatenated
- Do NOT pass company page URLs to the personal profile scraper (2SyF0bVxmgGr8IVCZ)
- Do NOT forget that the BasicTrigger auto-executes when you activate the scenario, so deactivate immediately after the run completes
- Do NOT try to fetch results before the scenario has finished executing (check execution status first)

## Running the Scenario

1. Activate the scenario using `scenarios_activate` (the BasicTrigger fires immediately)
2. Wait for execution to complete by checking `executions_list` for the scenario
3. Deactivate the scenario immediately with `scenarios_deactivate`
4. Find the Apify run IDs by checking the scenario's data stores or execution details

## Fetching Results from Apify Datasets

After the scenario runs, get data from Apify datasets:

1. Create a Make.com Tool (using `tools_create`) that calls: `GET /v2/datasets/{datasetId}/items`
2. Run the tool with `scenarios_run` using the dataset ID

### Fetching Large Datasets

Large datasets (40+ profiles) will return 500 errors if you fetch everything at once.

**Solution**: Fetch in batches of 5 using `limit` and `offset` query parameters.

If even batches of 5 are too large, use the `fields` parameter to request only essential fields:
`fields=linkedinUrl,firstName,lastName,fullName,headline,connections,followers,email,jobTitle,companyName,location,about,experience`

Create separate Make.com tools for each batch with hardcoded offset values. Do NOT try to use dynamic parameters like `{{2.offset}}` in tool URLs because Make.com tool parameter referencing does not work that way and will cause validation errors.

Run batch tools in parallel for speed.

## Processing LinkedIn Posts Data

If using Make.com, the posts data comes back in a wrapper format:
```json
[{"type": "text", "text": "{\"executionId\":\"...\",\"outputs\":{\"tool_output\":{\"body\":[...actual posts...]}}}"}]
```

Parse through this wrapper to get the actual post array:
```python
for item in raw_data:
    if item.get('type') == 'text':
        data = json.loads(item['text'])
        posts = data['outputs']['tool_output']['body']
```

Group posts by their `query.targetUrl` field (maps back to the LinkedIn URL queried).

## Processing Profile Data

Profile data from batched fetches may come in tool-result files or inline responses. Parse accordingly.

Key fields to extract: `linkedinUrl`, `firstName`, `lastName`, `fullName`, `headline`, `connections`, `followers`, `email`, `jobTitle`, `companyName`, `location`, `about`, `experience`, `companyIndustry`, `companySize`, `companyWebsite`.
