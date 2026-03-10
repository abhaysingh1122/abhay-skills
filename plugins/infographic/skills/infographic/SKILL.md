---
name: infographic
version: 1.1.0
description: |
  Generate professional LinkedIn-style infographics using Nano Banana (Gemini AI image generation).
  Creates clean, memorable visuals with ONE visual metaphor, flat colors, hand-drawn touches, and
  persistent brand guidelines. Follows the "LinkedIn Visual Insight Creator" design system.

  USE THIS SKILL WHEN:
  - User says "create infographic", "make a visual", "design an infographic", "generate infographic"
  - User wants to visualize a concept, framework, or data for social media
  - User mentions "LinkedIn post", "visual content", "carousel", "infographic series"
  - User asks to explain a concept visually or create a diagram-style image
  - User mentions "visual metaphor", "iceberg diagram", "pyramid", "funnel visualization"
  - User wants to generate images with Gemini/Nano Banana for informational content
  - User says "brand visuals", "social media graphics", "thought leadership visual"
  - User mentions "nano banana", "gemini image generation", "professional infographics"
---

# Infographic Generation Skill

Generate professional infographics and visual content using Gemini AI's native image generation. Supports visual metaphors, brand guidelines, series creation, and iterative refinement.

---

## When This Skill Loads (DO THIS IMMEDIATELY)

Follow these steps in order **before** responding to the user:

### Step 1: Read Reference Files

Read these files NOW (use Read tool):

```
1. references/visual-metaphors.md (12+ metaphor types, selection guide)
2. references/gemini-image-api.md (Nano Banana API, models, multi-turn editing)
3. references/style-guide.md (visual style rules, color systems, layouts)
4. references/prompt-engineering.md (prompt crafting for infographics)
```

### Step 2: Check Environment

Verify `GEMINI_API_KEY` is set:

```bash
echo "${GEMINI_API_KEY:+API key is set}"
```

If not set, ask the user to provide their Gemini API key and set it:
```bash
export GEMINI_API_KEY=your-key-here
```

### Step 3: Check for Brand Guidelines

Look for `.infographic-brand.json` in the working directory:

```bash
ls -la .infographic-brand.json
```

If it exists, read it to load the user's saved brand preferences. If it doesn't exist, you'll collect brand guidelines in Phase 3.

### Step 4: Create Task List (For Complex Projects)

If the user requests:
- A series of 3+ infographics
- Multiple concepts to visualize
- An iterative project with many revisions

Create a task list using TaskCreate to track progress.

### Step 5: Begin Phase 1

Start the workflow with Phase 1 below.

**Summary:** READ FILES → CHECK API KEY → LOAD BRAND → CREATE TASKS (if needed) → START PHASE 1

---

## Workflow: 7 Phases

### Phase 1: Idea Intake

**Goal:** Understand what the user wants to communicate visually.

1. **User shares a concept** — They provide a topic, data, idea, or content for an infographic

2. **Extract and confirm** the key elements:
   - **Core insight** — What's the ONE thing someone should take away?
   - **Key data points** — Numbers, comparisons, steps, categories, or main points
   - **Target audience** — Who will see this? (LinkedIn professionals, Instagram followers, Twitter audience, presentation viewers)
   - **Platform** — Where will it be posted? (LinkedIn, Instagram, Twitter, presentation, etc.)

3. **Clarify the message** with the user:
   ```
   "What's the ONE key insight or takeaway you want people to get from this infographic?"
   ```

4. **Confirm the brief** before moving forward:
   ```
   Here's what I understand:
   - Core message: [...]
   - Key points: [...]
   - Target audience: [...]
   - Platform: [...]

   Does this capture your vision?
   ```

**Output:** A clear, confirmed brief with core insight, data points, audience, and platform.

---

### Phase 2: Simplify & Structure

**Goal:** Determine if the concept fits one visual or needs a series.

1. Analyze the complexity of the concept:
   - **Simple** (1 core idea, 3-6 supporting points) → Single infographic
   - **Medium** (1 core idea, 7-12 supporting points) → Single infographic with careful simplification
   - **Complex** (multiple ideas, 12+ points, multi-step process) → Propose a series of 2-4 posts

2. If splitting into a series:
   - Propose how to divide the content across posts
   - Each post should stand alone but connect to the series theme
   - Suggest a series title and individual post angles

3. If single infographic:
   - Propose how to distill the concept into one clear visual
   - Identify what to include and what to cut

4. **Present the structure to the user for approval before proceeding.**

**Output:** Approved content structure (single or series) with specific content mapping.

---

### Phase 3: Brand Guidelines

**Goal:** Establish visual identity for consistent output and save for future sessions.

#### If `.infographic-brand.json` exists (loaded in Step 3):

Present the saved brand guidelines to the user:
```
I found your saved brand guidelines:
- Primary color: [color]
- Accent color: [color]
- Background: [light/dark]
- Font style: [style]
- Tone: [tone]

Would you like to use these, or update them?
```

If user wants to update, proceed to collect new guidelines below.

#### If no saved brand guidelines:

Ask the user for brand preferences:

| Element | Question | Default if none |
|---------|----------|-----------------|
| Primary color | "What's your brand's primary color? (hex code preferred)" | `#2563EB` (professional blue) |
| Secondary/Accent color | "Secondary/accent color?" | `#F59E0B` (warm amber) |
| Background | "Light or dark background preference?" | Light (`#FAFAFA`) |
| Font style | "Font preference? (clean sans-serif / serif / handwritten / bold)" | Clean sans-serif |
| Logo | "Do you have a logo URL or description to include?" | No logo |
| Tone | "Visual tone? (professional / playful / minimal / bold / hand-drawn)" | Professional |

If the user says "no brand guide," "use defaults," or "just make it look good," use the defaults above.

#### Save Brand Guidelines

After collecting or confirming brand guidelines, save them to `.infographic-brand.json`:

```bash
cat > .infographic-brand.json <<'EOF'
{
  "primaryColor": "#2563EB",
  "accentColor": "#F59E0B",
  "background": "light",
  "backgroundColor": "#FAFAFA",
  "fontStyle": "clean sans-serif",
  "logo": null,
  "tone": "professional",
  "savedAt": "2026-02-04T10:30:00Z"
}
EOF
```

**Output:** Brand profile saved to `.infographic-brand.json` and applied to all subsequent generations in this session.

---

### Phase 4: Visualization Ideation

**Goal:** Help the user discover the perfect visual metaphor for their concept.

#### IMPORTANT: Always Ask First

Before presenting visualization options, ask the user:

```
"Would you like me to suggest 5-10 different ways to visualize this concept?
I can present various visual metaphors (like icebergs, pyramids, timelines,
comparisons, etc.) that could work for your idea."
```

**If the user says YES**, proceed with steps below. **If the user already has a specific visualization in mind**, skip to Phase 5 with their chosen approach.

#### Step-by-Step Process:

1. **Analyze the concept's "shape"** — Ask yourself: "What is the SHAPE of this idea?"
   - Hidden depth? → Iceberg
   - Hierarchy? → Pyramid
   - Transformation over time? → U-Curve
   - Complexity vs simplicity? → Tangled vs Straight Line
   - Parts of a whole? → Donut Chart, Silhouette with Branches

2. **Use the Concept Shape Mapping** (from `visual-metaphors.md`):

| If the concept is about... | Consider these metaphors |
|---------------------------|------------------------|
| Hidden depth / surface vs reality | Iceberg, Split Comparison |
| Hierarchy / levels of importance | Pyramid, Concentric Circles |
| Transformation over time | U-Curve Journey, Timeline |
| Process / steps | Funnel, Timeline, Steps |
| Parts of a whole | Donut/Pie, Silhouette with Branches |
| Comparison / contrast | Split Comparison, Venn Diagram |
| Complexity vs simplicity | Tangled vs Straight Line |
| Convergence / focus | Hourglass, Funnel, Bullseye |
| Overlap / intersection | Venn Diagram, Concentric Circles |
| Growth / accumulation | Stacked Layers, Pyramid |

3. **Think beyond the catalog** — The 12 standard metaphors are starting points. If the concept naturally maps to something else, use it:
   - Bridge (connecting worlds)
   - Recipe (combining ingredients)
   - Solar system (central idea + orbits)
   - City skyline (relative importance)
   - Mountain path (journey with elevation)
   - Circuit board (interconnected systems)
   - Toolbox (collection of tools)

   The best metaphor makes the insight feel **inevitable**.

4. **Present 5-10 options** in this structured format:

```
Here are [N] ways to visualize "[concept]":

1. **[Metaphor Name]** — [Type]
   Why it works: [1-2 sentences explaining why this fits]
   Layout: [Brief structure description]
   Visual example: [What they'd see — "An iceberg with visible results
                   above water and hidden work below..."]

2. **[Metaphor Name]** — [Type]
   Why it works: [...]
   Layout: [...]
   Visual example: [...]

[Continue for 5-10 options]

Which approach(es) resonate with you? You can pick 1-3, or suggest your own idea.
```

5. **Wait for user selection** — Do NOT proceed to generation until the user picks their preferred approach(es).

**Output:** Numbered list of 5-10 visual metaphor options (only if user requests suggestions), ready for user selection.

---

### Phase 5: User Selection

**Goal:** Lock in the visual direction.

1. User picks **1-3 preferred** visualization approaches
2. For each selected approach, create a detailed content map:
   - Exact text/labels that will appear on the infographic
   - Where each data point maps to in the visual
   - Color assignments for different sections
   - Title and subtitle text
3. Present the refined approach(es) for final confirmation
4. **Do NOT generate images until the user confirms the direction.**

**Output:** Confirmed visual direction with detailed content mapping.

---

### Phase 6: Generate with Nano Banana (Gemini Image Generation)

**Goal:** Create the infographic image(s) using Gemini's image generation.

1. **Read reference files** for prompt crafting:
   - `references/prompt-engineering.md` for prompt structure
   - `references/gemini-image-api.md` for API details and model selection

2. **Select the right model:**
   - `gemini-3.0-flash-thinking-exp-01-21` (Nano Banana Pro) — Best for crisp text rendering, high-quality output ✨ **Recommended**
   - `gemini-2.0-flash-exp` (Nano Banana) — Fast iteration, good quality

   **Default:** Use **Nano Banana Pro** (`gemini-3.0-flash-thinking-exp-01-21`) as the primary model. It produces excellent text rendering which is critical for infographics. Use Nano Banana (`gemini-2.0-flash-exp`) only if speed is more important than text quality.

3. **Craft the generation prompt** incorporating:
   - Brand guidelines from Phase 3 (colors, font style, tone)
   - Chosen visual metaphor from Phase 5 (layout, structure)
   - Specific content (exact text labels, data points, title)
   - Platform-appropriate aspect ratio

4. **Determine the aspect ratio:**
   - LinkedIn feed: **4:5** (portrait) or **1:1** (square)
   - LinkedIn carousel: **4:5**
   - Instagram: **1:1** or **4:5**
   - Twitter: **16:9**
   - Stories: **9:16**
   - Presentation: **16:9**

5. **Call Gemini API** with the crafted prompt:
   ```bash
   curl -s "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-exp:generateContent?key=${GEMINI_API_KEY}" \
     -H "Content-Type: application/json" \
     -d '{...}' | jq -r '.candidates[0].content.parts[] | select(.inlineData) | .inlineData.data' | base64 -d > output.png
   ```

6. **Save the generated image** with descriptive naming:
   - Single: `infographic-[topic-slug].png`
   - Series: `infographic-[topic-slug]-01.png`, `infographic-[topic-slug]-02.png`, etc.

7. **Verify the file** was created:
   ```bash
   ls -la infographic-*.png
   ```

8. **Show to user** — Use Read tool to display the image for review

**Output:** Generated image(s) saved to disk and displayed for user feedback.

---

### Phase 7: Iterate & Series

**Goal:** Refine based on feedback and complete any series.

1. **Feedback loop:**
   - User provides feedback ("make it warmer," "change the title," "more whitespace")
   - Use Gemini multi-turn chat to edit while preserving context and style
   - Re-generate and show updated version
   - Repeat until user is satisfied

2. **Series generation** (if applicable):
   - After the first post is approved, generate remaining posts
   - Maintain consistent style across all posts (same colors, fonts, layout family)
   - Each post gets its own content but shares the visual language
   - Save with sequential numbering

3. **Final export:**
   - Confirm all images are saved with proper naming
   - Provide a summary of all generated files
   - Suggest optimal posting times/order for series (if applicable)

**Output:** Final approved image(s) with file paths listed.

---

## Quick Reference

### API Key Setup
```bash
export GEMINI_API_KEY=your-key-here
```

### Generate with Nano Banana (Quick)
```bash
# Using Nano Banana (gemini-2.0-flash-exp) - Fast iteration
curl -s "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-exp:generateContent?key=${GEMINI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "contents": [{"parts": [{"text": "Generate a professional infographic..."}]}],
    "generationConfig": {"responseModalities": ["TEXT", "IMAGE"]}
  }' | jq -r '.candidates[0].content.parts[] | select(.inlineData) | .inlineData.data' | base64 -d > output.png

# Using Nano Banana Pro (gemini-3.0-flash-thinking-exp-01-21) - Best text rendering
curl -s "https://generativelanguage.googleapis.com/v1beta/models/gemini-3.0-flash-thinking-exp-01-21:generateContent?key=${GEMINI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "contents": [{"parts": [{"text": "Generate a professional infographic..."}]}],
    "generationConfig": {"responseModalities": ["TEXT", "IMAGE"]}
  }' | jq -r '.candidates[0].content.parts[] | select(.inlineData) | .inlineData.data' | base64 -d > output.png
```

### Supported Aspect Ratios
| Platform | Ratio | Use |
|----------|-------|-----|
| LinkedIn feed | 4:5 or 1:1 | Standard posts (4:5 recommended) |
| LinkedIn carousel | 4:5 | Swipeable series |
| Instagram | 1:1 or 4:5 | Feed posts |
| Twitter | 16:9 | Timeline images |
| Stories | 9:16 | Vertical stories |
| Presentation | 16:9 | Slides |

### Brand Guidelines File
```json
{
  "primaryColor": "#2563EB",
  "accentColor": "#F59E0B",
  "background": "light",
  "backgroundColor": "#FAFAFA",
  "fontStyle": "clean sans-serif",
  "logo": null,
  "tone": "professional",
  "savedAt": "2026-02-04T10:30:00Z"
}
```

---

## Important Rules

1. **Never generate without user approval** — Always confirm the visual direction in Phase 5
2. **One metaphor per infographic** — Don't mix metaphors in a single image. If you need multiple metaphors, create a series
3. **Iterate, don't restart** — Use multi-turn Gemini chat for edits. Send the previous image back with modification instructions
4. **Save every generation** — Always save to disk with descriptive filenames before showing to user
5. **Brand consistency** — Apply the same brand profile (from `.infographic-brand.json`) across all images in a session
6. **Simplify ruthlessly** — If text won't be readable on mobile, cut it. Less is more. Maximum 6-8 text labels per infographic
7. **Ask before assuming** — When in doubt about style or content, ask the user
8. **Use Nano Banana models only** — Always use `gemini-2.0-flash-exp` or `gemini-3.0-flash-thinking-exp-01-21`. Do NOT use Imagen models.
9. **Series numbering** — For multi-post series, always include "Part X of Y" on each image
10. **White space is essential** — Target 40%+ empty canvas. Breathing room makes it professional
