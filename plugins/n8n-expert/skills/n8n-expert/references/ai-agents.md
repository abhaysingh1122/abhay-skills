# AI Agents — Designing Prompts that Actually Work

How to design and debug AI agent nodes (`langchain.agent` / `chainLlm`) in n8n. These patterns are the difference between an agent that follows your rules and one that doesn't.

---

## The 5-step structural fix when an agent ignores rules

Symptoms: agent defaults to "include everything", treats abstract phrases as instructions, hallucinates fields, ignores explicit rules.

**The fix is structural, not rhetorical. Don't keep rephrasing.** Apply these in order:

### 1. Set `temperature: 0`

On the chat model node (`lmChatOpenAi` / `lmChatOpenRouter` / etc.):

```js
{
  model: { __rl: true, value: 'gpt-4.1-mini', mode: 'list' },
  options: { temperature: 0 }
}
```

Eliminates "creative" interpretation. Same input → same output. Without this, you can't tell if your prompt change had any effect.

### 2. Replace prose rules with a numbered DECISION PROCEDURE

Don't write paragraphs of rules. Write a literal step-by-step procedure the agent follows:

```
STEP A — Scan the input for LITERAL X.
   - If X found: go to STEP B.
   - Otherwise: jump to STEP C.

STEP B — X found. Classify into category:
   - If matches pattern P1: emit { type: "alpha" }
   - If matches pattern P2: emit { type: "beta" }
   - Otherwise: emit { type: "other" }

STEP C — No X. Check for Y:
   ...
```

The agent will follow the procedure literally. Your job is to make the procedure cover every case.

### 3. State the DEFAULT explicitly

Don't write "include images when relevant". Write:

> *"Your DEFAULT is `image_urls: []`. You ONLY add a URL if [literal condition]. If the condition is not met, return `[]`."*

LLMs treat absence of a default as license to be helpful and add things you didn't ask for.

### 4. List ✅ what counts AND ❌ anti-examples

The anti-examples are the most important part. LLMs hallucinate that "the protagonist" implies "the same protagonist as before". Show them concretely what doesn't qualify:

```
✅ "same woman", "keep her", "preserve the character"
❌ "the protagonist" (a scene role, not a preservation instruction)
❌ "she is looking" (abstract pronoun reference)
❌ "a medium shot" (cinematography term, not subject reference)
```

For each rule the agent has been violating, add an anti-example showing the WRONG interpretation.

### 5. Add 4-5 worked examples with EXACT JSON outputs

Show input → expected output. Pattern-matching is what LLMs do best. Examples teach faster than rules.

```
Example 1:
INPUT: "<sample input>"
OUTPUT: { "type": "alpha", "image_urls": [], "summary": "..." }

Example 2:
INPUT: "<different input>"
OUTPUT: { "type": "beta", "image_urls": ["url1"], "summary": "..." }

(continue for each main mode)
```

### 6. Self-check step before output

Add this near the end of the system prompt:

> *"Before outputting, verify: if your output contains X, you must be able to quote the specific phrase from the input that justifies it. If you can't quote a justifying phrase, remove X from your output."*

Forces the agent to audit itself.

### 7. Pure pass-through downstream

The Code node consuming the agent's output must NOT add fallback inference. If the agent returns `image_urls: []`, the Code node sends `[]`. If parsing fails, default to safe empty — never re-inject context.

```js
// ❌ BAD: re-injects context the agent decided to omit
if (!parsed.image_urls.length) {
  parsed.image_urls = [previousFrameUrl];
}

// ✅ GOOD: agent has SOLE AUTHORITY
const imageUrls = Array.isArray(parsed.image_urls) ? parsed.image_urls : [];
```

The agent decided. Trust the agent.

---

## Don't put a Code node BEFORE an AI agent

Anti-pattern that creeps in when you think the agent needs help:

```
❌ Get tables → Code "Build Context" (summarizes/picks fields) → AI Agent → ...
```

Wrong because:
1. Pre-deciding context limits the AI's reasoning
2. Field-mapping bugs in the Code node break the chain
3. Defeats the point of using an AI agent
4. *"I could have just used a code node only"* — if you're going to summarize before AI, why use AI?

```
✅ Get table A → Get table B → Get table C → AI Agent (reads ALL raw via $json + cross-refs) → Code (format AI output to schema) → Save
```

Feed the agent EVERYTHING raw. Let the agent choose what's relevant. Code node POST-AI is fine — it's just shaping JSON.

**Specific rules:**
- Multiple Airtable tables = more context = better output. Chain them sequentially before the AI.
- AI agent prompt should reference EVERY relevant field via `{{ $json['Field Name'] }}` and `{{ $('NodeName').item.json['Field Name'] }}`
- System prompt = strong directive role + clear output JSON shape

---

## System prompt skeleton

Standard structure for any directive AI agent:

```markdown
# Role
[1 sentence: who the agent is, what they produce. e.g., "Senior B2B copywriter producing landing-page copy. Always in client's native language. Style: agency-grade, never SaaS template."]

# Input
[What data the agent receives. List the field names so the agent knows what to look for.]

# Step 1 — Extract context (MANDATORY)
[Force the agent to scan the input data BEFORE writing output. List what to extract.]

# Step 2 — Apply the procedure
[Numbered decision steps]

# Strict Rules
## CRITICAL anti-patterns (NEVER do these)
❌ ...
❌ ...

## Positive patterns (ALWAYS prefer)
✅ ...
✅ ...

# Output Format — STRICT JSON only
[Show the exact JSON shape with placeholder values + types]

# Pre-Output Checklist
[Force the agent to self-verify before output]

[Closing instruction: "Output JSON only. No commentary. No markdown fences."]
```

Save this skeleton and adapt per use case.

---

## Image-prompt agents — the cinematic baseline

For ANY agent that outputs prompts for image generation models (Nano Banana, Gemini, Midjourney, DALL-E), use this richness level. Minimal prompts produce minimal results.

### The skeleton

```markdown
# Role
Award-winning Creative Director / Film Director / Prompt Engineer for AI-generated [context]. Working in [aspect ratio]. Style: Netflix Documentary + Apple Commercial.

# Input
[List exactly what data the agent receives.]

# Step 1 — Extract Customer Context (MANDATORY)
[Force the agent to scan input data BEFORE writing prompts.]

# Step 2 — Two-Stage Thinking per Image
For each image:
1. VISUAL — what is the subject / environment / lighting / camera / mood?
   - SUBJECT: who/what is doing what concrete action?
   - ENVIRONMENT: foreground/midground/background with specific objects
   - LIGHTING: source direction, color temperature, hard or soft
   - CAMERA: angle, focal length (35/50/85mm), distance
   - MOOD: emotional tone fitting THIS section's purpose
2. PROMPT — write as one flowing photorealistic paragraph (150-300 words)

# Strict Rules

## Visual Variety — CRITICAL
Each image must feel visually DISTINCT from every other. Vary:
- Shot scale (wide / medium / close-up / extreme close-up)
- Camera angle (eye-level / low / high / POV / over-shoulder)
- Focal length (35mm / 50mm / 85mm)
- Lighting direction (front / side / back / top / bottom)
- Color temperature

No two slots should share the same shot scale + angle + lighting combo.

## Style — Netflix Documentary + Apple Commercial
Every prompt MUST describe:
- Photorealistic, cinematic
- Lens specification (35mm wide / 50mm portrait / 85mm tight)
- Shallow depth of field (f/1.8 – f/2.8)
- Natural directional lighting with clear source
- High dynamic range
- Realistic textures (skin pores, fabric weave, dust, sweat)
- Color grading: muted, cinematic, never saturated
- Subtle film grain, volumetric light where appropriate

## Aspect Ratio
[Specify. Never deviate.]

## Negative prompt (apply to all)
"split screen, collage, grid layout, storyboard sheet, multi-panel image, readable text in image, watermarks, embedded text, captions, charts, infographics, dashboard screenshots, UI overlay, generic stock-photo aesthetic, generic team in office cliche, generic handshake cliche, generic lightbulb idea cliche."

## Safety
No injury, violence, dangerous activity, faces of real public figures, copyrighted characters.

# Slot Definitions
[List each image slot with its purpose, mood, and any constraints]

# CRITICAL OUTPUT RULES
- Slot count: EXACTLY [N]
- Slot names: EXACT identifiers
- Each prompt 150-300 words, one flowing paragraph
- Each prompt explicitly says "no embedded text, no charts, no UI overlay"

# Output Format — STRICT JSON
{
  "prompts": [
    { "slot": "slot-name", "prompt": "<150-300 word cinematic paragraph>" },
    ...
  ]
}

# Pre-Generation Checklist
- Exactly [N] prompts
- Visual variety rule holds
- Each prompt has lens, f-stop, lighting direction
- Each prompt fits Netflix Doc + Apple Commercial vocabulary
- 16:9 (or specified ratio) mentioned or implied
- No stock-photo cliches

Output JSON only.
```

### Why this pattern works

1. **Role identity** primes the LLM for expert-level output
2. **Two-stage thinking** prevents shallow prompts
3. **Visual variety rule** ensures images don't all look the same
4. **Technical specs** anchor in cinematography vocabulary
5. **Netflix Doc + Apple Commercial** is a CONCRETE style reference
6. **Pre-Generation Checklist** forces self-verification

### What NOT to do

- ❌ Short system prompts ("write 6 image prompts for these slots")
- ❌ "photorealistic" alone with no style anchor
- ❌ No technical vocabulary (lens, depth of field, lighting direction)
- ❌ No variety enforcement → all images feel the same
- ❌ Generic stock-photo aesthetic

---

## Copywriter agents — what to encode

Same structural rigor as image prompts, applied to text generation:

```markdown
# CRITICAL ANTI-PATTERNS (NEVER write these)

## H1 anti-patterns (B2B agency context)
❌ "Save 5 Hours Weekly" (quantitative, brittle, doesn't scale)
❌ "Get 10x Results"
❌ "Reduce X by Y%"
❌ "Cut [task] Time in Half"

## Vocabulary anti-patterns
❌ "tool" / "platform" / "software" / "app" (when client is an agency)
❌ "Free Trial" / "Free Demo" / "Sign Up"
❌ "$X/month" or pricing in copy (B2B custom-scoped deals)
❌ Generic SaaS template language

# POSITIVE PATTERNS (ALWAYS prefer)

## H1 patterns — transformation/identity/triad
✅ "[Domain], die mit Ihnen wachsen." (transformation)
✅ "Where Strategy Meets Execution." (identity)
✅ "Klarheit. Effizienz. Wachstum." (three-word power triad)

## Subheadline patterns
Describe what the agency BUILDS / DELIVERS for the client + the outcome.
✅ "Wir bauen die Systeme, die Ihr Unternehmen vom täglichen Chaos befreien — damit Sie sich auf Wachstum konzentrieren können."
```

The principle: **anti-examples + positive examples + worked examples >> abstract rules.**

---

## Output JSON key format — match exactly

Critical for Code nodes downstream. If AI emits kebab-case (`'sec1-h1-headline'`), code reading snake_case (`copy.sec1_h1_headline`) silently returns undefined → blank fields.

**Lock down the format in the system prompt:**

```markdown
# Output Format — STRICT JSON only

{
  "sec1-h1-headline": "<text>",
  "sec1-h2-subheadline": "<text>",
  "brand-color": "<hex>",
  ...
}
```

Show the EXACT shape with literal keys. Don't say "output the fields with the slugs" — show the JSON.

Then in the Code node consuming output, use bracket notation:
```js
const C = (k) => (parsed[k] ?? '').toString();
fieldData['sec1-h1-headline'] = C('sec1-h1-headline');
```

---

## Real failure mode: 3 iterations of prompt failure

Common debugging arc:
- v1: discursive prose with "rules" — agent ignores them
- v2: keywords but no decision procedure — agent confused intent
- v3: directive (temp=0 + numbered steps + ❌ anti-examples + 5 worked examples + self-check) — finally works

If you're on v2 of a prompt and it still doesn't work, **stop tweaking and apply the full structural fix.** Adding more rules to a paragraph won't help.

---

## Verifying the agent followed instructions

In execution data, inspect:
1. Raw agent output (the `output` or `text` field)
2. Confirm it parses as JSON
3. Confirm the keys match what your Code node reads
4. Confirm the values reflect your rules

If raw output looks right but downstream fields are blank → key format mismatch (see [code-patterns.md](code-patterns.md)).
If raw output ignores a rule → restructure the prompt (the 7 steps above).
