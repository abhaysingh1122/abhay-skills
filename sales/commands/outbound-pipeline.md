---
description: Run the full outbound sales pipeline on a lead list
argument-hint: [lead-list-file]
allowed-tools: Read, Write, Edit, Bash, Grep, Glob, Task, WebSearch, TodoWrite, AskUserQuestion
---

Run the complete outbound sales pipeline. This is a four-phase workflow that takes a raw lead list and produces a fully enriched CSV with qualified leads, deep intelligence, LinkedIn data, and hyper-personalized email icebreakers.

**You are the orchestrator.** Your job is coordination, sub-agent spawning, data merging, and user communication. Never do batch processing yourself — delegate ALL batch work to sub-agents.

## Phase 0: Gather Context

Before doing anything, collect two critical inputs from the user using AskUserQuestion:

1. **ICP Definition**: What makes a lead qualified? Services, headcount, geography, niche, job title, revenue, etc. Push back if vague — ask "What specifically makes a company a good fit? What would make you NOT want to reach out?" Clarify AND vs OR conditions.

2. **Product/Service Context**: What are they selling and why would these leads care? Understand deeply so every phase focuses on relevant signals.

Do NOT ask about LinkedIn scraping paths. Default to the native Apify connector.
Do NOT proceed until both ICP and product context are confirmed.

If the user provided both already (inline with their request), confirm your understanding and proceed.

## Phase 1: Lead Qualification

### 1.1 Load and Inspect

```python
# Parse CSV into JSON. Report to user:
# - Total number of leads
# - Column names available
# - 2-3 sample rows
import csv, json
with open(CSV_PATH, 'r', encoding='utf-8-sig') as f:
    reader = csv.DictReader(f)
    rows = list(reader)
```

### 1.2 Prepare Batches

```python
# Create batch files of 10 leads each
batch_size = 10
num_batches = ceil(len(rows) / batch_size)
for i in range(0, len(rows), batch_size):
    batch = rows[i:i+batch_size]
    with open(f'batch_{i//batch_size}.json', 'w') as f:
        json.dump(batch, f)
```

Tell the user the plan: "I have [N] leads. I'll launch [M] sub-agents (10 leads each). Estimated time: ~2-3 minutes."

### 1.3 Record Start Time

```python
import time
start = time.time()
with open('start_time.txt', 'w') as f:
    f.write(str(start))
```

### 1.4 Spawn ALL `lead-qualifier` Sub-Agents in a Single Message

Use the **Task tool** with `subagent_type: sales:lead-qualifier` for each batch. **ALL Task calls must be in ONE message** — this is what makes them parallel.

```
<function_calls>
<!-- Repeat this block for EVERY batch (0 to num_batches-1) -->
<invoke name="Task">
  <parameter name="subagent_type">sales:lead-qualifier</parameter>
  <parameter name="description">Qualify batch {i}</parameter>
  <parameter name="prompt">
    ICP: {paste full ICP definition verbatim}
    AND/OR logic: {spell out explicitly}
    Batch file: batch_{i}.json
    Output file: results_batch_{i}.json
    Instructions: Use WebSearch 2-3 times per lead (company website + third-party sources like Clutch, G2, directories)
  </parameter>
</invoke>
</function_calls>
```

Each sub-agent receives via its prompt:
- The full ICP definition (copy it verbatim, never summarize)
- The AND/OR logic spelled out explicitly
- The JSON batch of leads (embedded in the prompt or referenced as a file path)
- The output file path: `results_batch_{N}.json`
- Instruction to use WebSearch 2-3 times per lead

**Output schema each sub-agent must produce:**
```json
[{
  "full_name": "...",
  "email": "...",
  "company_name": "...",
  "company_website": "...",
  "qualified": true,
  "reason": "1-2 sentence explanation",
  "confidence": "high|medium|low"
}]
```

### 1.5 Collect and Merge Results

After ALL sub-agents complete:

```python
import json, os, csv

all_results = []
for i in range(num_batches):
    path = f'results_batch_{i}.json'
    try:
        with open(path, 'r') as f:
            data = json.load(f)
            all_results.extend(data if isinstance(data, list) else [data])
    except json.JSONDecodeError:
        # Sub-agents sometimes produce slightly malformed JSON
        # Use regex extraction as fallback (see Data Integrity section)
        pass

# Build lookup by email (primary key)
result_by_email = {r['email'].strip().lower(): r for r in all_results if r.get('email')}

# Merge into original CSV — PRESERVE ALL ORIGINAL COLUMNS
for row in original_rows:
    match = result_by_email.get(row['email'].strip().lower())
    row['Qualified'] = 'Yes' if match and match.get('qualified') else 'No'
    row['Qualification_Reason'] = match.get('reason', '') if match else ''
    row['Confidence'] = match.get('confidence', '') if match else ''
```

**CRITICAL: Sub-agents use varying JSON key names.** Some use `full_name`, others use `name`. Some use `intelligence_report`, others use `intelligence`. Always build lookups with multiple fallback keys:

```python
email = r.get('email', '').strip().lower()
report = r.get('intelligence_report', '') or r.get('intelligence', '') or r.get('report', '')
```

### 1.6 Save and Report

Save two CSVs:
1. **Full CSV** with `Qualified`, `Qualification_Reason`, `Confidence` columns added (all leads)
2. **Qualified-only CSV** (filtered to Qualified=Yes)

Report to user:
```
Qualification complete.
- Total leads: [N]
- Qualified: [N] ([%])
- Disqualified: [N]
- Time taken: [M] minutes [S] seconds
```

### 1.7 Transition to Phase 2

Extract qualified leads into a working JSON file for the next phases:

```python
qualified_rows = [r for r in rows if r.get('Qualified') == 'Yes']
with open('qualified_leads.json', 'w') as f:
    json.dump(qualified_rows, f, indent=2)
```

---

## Phase 2: Lead Intelligence

**Layer 1 (web research) and Layer 2 (LinkedIn scraping) MUST launch in parallel.**

### 2.1 Prepare Both Layers

```python
# Layer 1: Create research batches of 5 leads each
research_batch_size = 5
num_research_batches = ceil(len(qualified) / research_batch_size)

# Layer 2: Collect all LinkedIn URLs for single Apify call
linkedin_urls = [lead['linkedin'].strip() for lead in qualified if lead.get('linkedin', '').strip()]
with open('linkedin_urls.json', 'w') as f:
    json.dump(linkedin_urls, f)
```

Tell the user: "Launching [N] lead-researcher agents + 1 linkedin-scraper agent = [N+1] total sub-agents in parallel."

### 2.2 Record Start Time

```python
import time
phase2_start = time.time()
with open('phase2_start_time.txt', 'w') as f:
    f.write(str(phase2_start))
```

### 2.3 Spawn ALL N+1 Sub-Agents in a Single Message

Use the **Task tool** for every agent. ALL calls in ONE message — researchers + scraper together.

```
<function_calls>
<!-- N lead-researcher agents (one per batch of 5) -->
<invoke name="Task">
  <parameter name="subagent_type">sales:lead-researcher</parameter>
  <parameter name="description">Research batch 0</parameter>
  <parameter name="prompt">
    Product context: {paste product context}
    Batch file: research_batch_0.json
    Output file: intel_batch_0.json
    Produce 13-section intelligence report per lead.
  </parameter>
</invoke>
<!-- ... repeat for all N batches ... -->

<!-- 1 linkedin-scraper agent (handles ALL URLs) -->
<invoke name="Task">
  <parameter name="subagent_type">sales:linkedin-scraper</parameter>
  <parameter name="description">Scrape all LinkedIn profiles</parameter>
  <parameter name="prompt">
    LinkedIn URLs file: linkedin_urls.json
    Call BOTH Apify actors:
      - Profiles: 2SyF0bVxmgGr8IVCZ
      - Posts: harvestapi/linkedin-profile-posts (NOT A3cAPGpwBEG8RJwse — that actor is deprecated and returns 0 usable posts)
    Save to: all_profiles.json, all_posts.json
    Timeout handling: capture runId + datasetId from partial response, then call get-dataset-items directly. Do NOT poll get-actor-run-list.
  </parameter>
</invoke>
</function_calls>
```

**Each `lead-researcher` receives:**
- Product context (so research focuses on relevant signals)
- Its batch of 5 leads as JSON
- Output path: `intel_batch_{N}.json`
- Instruction to produce the 13-section intelligence report per lead

**The `linkedin-scraper` receives:**
- Path to `linkedin_urls.json`
- Instructions to call BOTH Apify actors (profiles: `2SyF0bVxmgGr8IVCZ`, posts: `harvestapi/linkedin-profile-posts`)
- ALL URLs in a single API call per actor (never split)
- Save profiles to `all_profiles.json`, posts slim array to `all_posts.json`
- Timeout handling: capture `runId` + `datasetId` from the partial response header, then fetch dataset directly — no polling

**Intel output schema:**
```json
[{
  "full_name": "...",
  "email": "...",
  "company_name": "...",
  "intelligence_report": "SUMMARY | ... | ACHIEVEMENTS"
}]
```

### 2.4 Merge via Python Script (MANDATORY)

After ALL sub-agents complete, run a Python merge script. **Never merge inline in conversation** — large datasets overflow context and cause data loss.

The merge script must handle:

**Sub-agent JSON key inconsistencies:**
```python
# Sub-agents use different key names. ALWAYS try multiple fallbacks:
email = r.get('email', '').strip().lower()
report = (r.get('intelligence_report', '') or
          r.get('intelligence', '') or
          r.get('report', ''))
```

**Broken JSON recovery:**
```python
# Some sub-agents produce malformed JSON. Use regex extraction as fallback:
def extract_from_broken_json(path):
    with open(path, 'r') as f:
        raw = f.read()
    try:
        return json.load(raw)  # Try normal parse first
    except:
        # Regex fallback: extract email + report pairs
        pattern = r'"email"\s*:\s*"([^"]+)".*?"intelligence(?:_report)?"\s*:\s*"'
        # ... extract records individually
```

**Multi-key matching for CSV rows:**
```python
# Match by email first, then name, then company name as fallbacks
intel = (intel_by_email.get(email, '') or
         intel_by_name.get(name, '') or
         intel_by_company.get(company, ''))
```

**LinkedIn posts matching:**
```python
# Posts use 'query.profilePublicIdentifier' (NOT 'query.targetUrl')
# when fetched with flatten parameter
def normalize_url(url):
    url = url.lower().strip().rstrip('/')
    url = re.sub(r'^https?://', '', url)
    url = re.sub(r'^(www\.|[a-z]{2}\.)', '', url)
    return url

for post in posts:
    target = post.get('query.profilePublicIdentifier', '')
    norm = normalize_url(target)
    posts_by_url.setdefault(norm, []).append(post)
```

**LinkedIn profile block format:**
```
=== LINKEDIN PROFILE ===
Name: [fullName]
Headline: [headline]
Current Role: [jobTitle] at [companyName]
Location: [location]
Connections: [connections] | Followers: [followers]
Email (from LI): [email]
About: [about, max 500 chars]

=== RECENT POSTS ([count] found) ===
Post 1 ([postedAt.date]): [engagement.likes] likes, [engagement.comments] comments, [engagement.shares] shares
  [content, max 300 chars, newlines replaced with spaces]
```

### 2.5 Save Enriched CSV

Add two new columns:
- `General Lead Intelligence` — pipe-separated 13-section report
- `LinkedIn Lead Research` — profile + posts block

Also save as `enriched_leads.json` for Phase 3.

Calculate phase duration:
```python
with open('phase2_start_time.txt', 'r') as f:
    phase2_start = float(f.read().strip())
phase2_duration = time.time() - phase2_start
phase2_mins = int(phase2_duration // 60)
phase2_secs = int(phase2_duration % 60)
```

Report to user:
```
Lead Intelligence complete.
- Leads enriched with web intelligence: [N]/[qualified]
- LinkedIn profiles matched: [N]/[qualified]
- LinkedIn posts matched: [N]/[qualified]
- Time taken: [M]m [S]s
```

---

## Phase 3: Email Personalization

### 3.1 Write 2 Test Icebreakers

Pick 2 leads with rich data (LinkedIn posts, strong intel). Write 2-3 icebreaker VARIATIONS for each. Present all options to the user.

These test icebreakers must already follow all writing rules from the `email-personalization` skill. The goal is to calibrate tone, style, and angle before scaling.

### 3.2 Get Approval

Wait for user feedback. Common adjustments:
- "Too formal / too casual"
- "I prefer option A's style"
- "Don't reference X, reference Y instead"
- New rules to add

**Do NOT proceed until the user approves.** This is a mandatory human checkpoint.

### 3.3 Prepare Icebreaker Batches

```python
# Slim down leads to only fields needed for icebreakers (save tokens)
batch_size = 5
for lead in leads:
    slim = {
        'full_name': lead['full_name'],
        'email': lead['email'],
        'company_name': lead['company_name'],
        'job_title': lead['job_title'],
        'city': lead.get('city', ''),
        'state': lead.get('state', ''),
        'country': lead.get('country', ''),
        'keywords': lead.get('keywords', '')[:300],
        'General Lead Intelligence': lead.get('General Lead Intelligence', ''),
        'LinkedIn Lead Research': lead.get('LinkedIn Lead Research', ''),
    }
```

### 3.4 Record Start Time

```python
import time
phase3_start = time.time()
with open('phase3_start_time.txt', 'w') as f:
    f.write(str(phase3_start))
```

### 3.5 Spawn ALL `icebreaker-writer` Sub-Agents in a Single Message

Use the **Task tool** with `subagent_type: sales:icebreaker-writer` for each batch. ALL Task calls in ONE message.

```
<function_calls>
<!-- Repeat for EVERY batch (0 to num_batches-1) -->
<invoke name="Task">
  <parameter name="subagent_type">sales:icebreaker-writer</parameter>
  <parameter name="description">Icebreakers batch {i}</parameter>
  <parameter name="prompt">
    Product context: {paste product context}
    Approved examples: {paste the 2 exact icebreakers user approved}
    Writing rules: {paste all rules from skill + user feedback}
    Batch file: icebreaker_batch_{i}.json
    Output file: icebreaker_results_{i}.json
  </parameter>
</invoke>
</function_calls>
```

**Every sub-agent receives in its prompt:**
1. Product/service context
2. The 2 approved example icebreakers (copy the exact text the user approved)
3. ALL writing rules from the skill + any user-added rules from feedback
4. Its batch of 5 leads with all enrichment data
5. Output path: `icebreaker_results_{N}.json`

**Icebreaker output schema:**
```json
[{
  "full_name": "...",
  "email": "...",
  "company_name": "...",
  "icebreaker": "the full icebreaker text",
  "skipped": false,
  "skip_reason": ""
}]
```

### 3.6 Quality Check (Programmatic)

After collecting all icebreakers, run automated violation scanning:

```python
bad_starts = ["saw your post", "saw your recent", "noticed your", "noticed that",
              "impressive to see", "loved seeing", "loved that", "your post on",
              "your recent post", "i noticed", "i came across"]
bad_words = ["spot on", "data-driven", "data driven", "values-driven", "ai-first",
             "compelling", "resonated", "innovative", "leverage", "synergize"]

for r in icebreakers:
    ib = r['icebreaker']
    # Check bad starts
    for bs in bad_starts:
        if ib.lower().startswith(bs):
            # Fix: "Saw your post on X" -> "Was on your LinkedIn, your post on X"
    # Check bad words
    for bw in bad_words:
        if bw in ib.lower():
            # Fix: regex replace with approved alternative
    # Check m-dashes
    if '—' in ib or '---' in ib:
        # Fix: replace with comma or semicolon
```

### 3.7 Merge and Save

Add `Email Personalization` column to CSV. Match by email (primary) then full_name (fallback).

Calculate phase duration:
```python
with open('phase3_start_time.txt', 'r') as f:
    phase3_start = float(f.read().strip())
phase3_duration = time.time() - phase3_start
phase3_mins = int(phase3_duration // 60)
phase3_secs = int(phase3_duration % 60)
```

Report:
```
Personalization complete.
- Icebreakers written: [N]
- Leads skipped: [N] (person-company mismatches or thin data)
- Quality violations found and fixed: [N]
- Time taken: [M]m [S]s
```

---

## Phase 4: Final Report

```
Pipeline Complete.
- Total leads processed: [N]
- Qualified: [N] ([%])
- Enriched with web intelligence: [N]/[qualified]
- Enriched with LinkedIn data: [N]/[qualified] profiles, [N] with posts
- Icebreakers written: [N]
- Leads skipped during personalization: [N]
- Total time: [duration]
```

Provide links to final output files:
1. Full enriched CSV (all qualified leads with all columns)
2. Icebreakers-only CSV (only leads with written icebreakers)

---

## Data Integrity Rules (Learned from Production Runs)

### Sub-Agent JSON Key Inconsistency
Sub-agents frequently use different key names for the same field. The merge script MUST handle:
- `intelligence_report` vs `intelligence` vs `report`
- `full_name` vs `name` vs `first_name`+`last_name`
- `qualified` (boolean) vs `qualified` (string "Yes"/"No")
- `company_name` vs `company`

Always use fallback chains: `r.get('key1', '') or r.get('key2', '') or r.get('key3', '')`

### Broken JSON Recovery
Sub-agents sometimes produce malformed JSON (unescaped quotes in company descriptions, newlines in report text). The merge script must:
1. Try `json.load()` first
2. Try `json.loads(raw, strict=False)` second
3. Fall back to regex extraction of individual records

### Multi-Key Matching
When merging sub-agent results back to the CSV, match using multiple keys in priority order:
1. Email (most reliable, normalize to lowercase)
2. Full name (normalize to lowercase)
3. Company name (normalize to lowercase, handles edge cases where email format changed)

### LinkedIn Posts Field Name
When posts are fetched with the `flatten` parameter, the profile URL field is `query.profilePublicIdentifier` (NOT `query.targetUrl`). Always check the actual field names in fetched data before building the merge logic.

### Persist Everything to Disk
Never hold large datasets in conversation context. Save all intermediate results as JSON files immediately. Context compaction WILL lose data that only exists in the conversation.

---

## Parallelism Rules

**The single most important rule: ALL sub-agents in a phase MUST be spawned in ONE message using the `Task` tool.**

Sub-agents are spawned using the **`Task` tool**. Each call requires these parameters:
- `subagent_type`: The agent identifier (see table below)
- `prompt`: The full instructions, ICP, batch data, output path, etc.
- `description`: A short label (e.g., "Qualify batch 3")

**Agent identifiers (subagent_type values):**
| Agent | subagent_type |
|---|---|
| Lead Qualifier | `sales:lead-qualifier` |
| Lead Researcher | `sales:lead-researcher` |
| LinkedIn Scraper | `sales:linkedin-scraper` |
| Icebreaker Writer | `sales:icebreaker-writer` |

**To achieve parallel execution, ALL Task tool calls MUST be in a single message.** This is the only way to trigger parallelism. If you emit one Task call, wait for the result, then emit another — that is sequential. You must emit all of them at once.

**Concrete example — spawning 3 parallel agents:**
```
<function_calls>
<invoke name="Task">
  <parameter name="subagent_type">sales:lead-qualifier</parameter>
  <parameter name="description">Qualify batch 0</parameter>
  <parameter name="prompt">... full prompt with ICP + batch data ...</parameter>
</invoke>
<invoke name="Task">
  <parameter name="subagent_type">sales:lead-qualifier</parameter>
  <parameter name="description">Qualify batch 1</parameter>
  <parameter name="prompt">... full prompt with ICP + batch data ...</parameter>
</invoke>
<invoke name="Task">
  <parameter name="subagent_type">sales:lead-qualifier</parameter>
  <parameter name="description">Qualify batch 2</parameter>
  <parameter name="prompt">... full prompt with ICP + batch data ...</parameter>
</invoke>
</function_calls>
```

**Phase-specific agent counts (for 137 leads):**
- Phase 1: 14 `sales:lead-qualifier` agents (ceil(137/10))
- Phase 2: 28 `sales:lead-researcher` agents + 1 `sales:linkedin-scraper` agent = 29 total
- Phase 3: 28 `sales:icebreaker-writer` agents (ceil(137/5))

**Never batch sequentially.** Never spawn 5, wait, spawn 5 more. One shot, all agents, one message.

---

## CSV Column Progression

The CSV grows through the pipeline. Never drop columns — only add.

```
Original CSV columns (from lead source)
  + Phase 1: Qualified, Qualification_Reason, Confidence
  + Phase 2: General Lead Intelligence, LinkedIn Lead Research
  + Phase 3: Email Personalization
```

The CSV is the single source of truth throughout the pipeline.
