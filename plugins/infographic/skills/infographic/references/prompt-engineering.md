# Prompt Engineering for Infographic Generation

How to craft effective prompts for Gemini image generation that produce professional, readable infographics.

---

## Prompt Structure Template

Every infographic prompt should follow this structure:

```
Generate a [STYLE] infographic about [TOPIC].

Visual metaphor: [METAPHOR TYPE] layout.
[DESCRIBE THE VISUAL STRUCTURE — what goes where, how elements are arranged]

Content:
- [Element 1]: "[text label]"
- [Element 2]: "[text label]"
- [Element N]: "[text label]"

Title: "[TITLE TEXT]"
[Subtitle if applicable]: "[SUBTITLE TEXT]"

Style: [VISUAL STYLE DESCRIPTION]
Colors: [PRIMARY] for main elements, [ACCENT] for highlights, [BACKGROUND] background.
Font: [FONT STYLE].
Aspect ratio: [RATIO] for [PLATFORM].

[NEGATIVE INSTRUCTIONS — what to avoid]
```

---

## Key Principles

### 1. Describe the Scene, Not Keywords

**Bad:** "infographic, pyramid, leadership, professional, blue"

**Good:** "A pyramid infographic with 4 horizontal tiers showing leadership levels. The base tier is widest and labeled 'Individual Contributor.' Each tier above is narrower: 'Team Lead,' 'Director,' 'VP.' The pyramid uses soft blue gradients getting darker toward the top. Clean flat design on a white background."

### 2. Be Specific About Text

Gemini handles text well, but you must be explicit about:
- Exact text to display (in quotes)
- Where text appears (position relative to visual elements)
- Text style (bold, size relative to other text)

**Bad:** "add some labels"

**Good:** "Title 'The Leadership Pyramid' in bold at the top center. Each tier has a centered label in white text: bottom tier 'Individual Contributor,' second tier 'Team Lead,' third tier 'Director,' top tier 'VP.'"

### 3. Specify Layout Precisely

Tell the model where things go:
- "centered in the image"
- "top-left corner"
- "arranged vertically from top to bottom"
- "split into left and right halves"
- "surrounding the central element"

### 4. Include Style Constraints

Always specify:
- Overall style (flat illustration, hand-drawn, minimal, corporate)
- Color palette with hex codes
- Background color/type
- Font style
- Aspect ratio

---

## Brand Integration in Prompts

When the user has brand guidelines, weave them into every prompt:

```
Style: Clean, professional flat illustration.
Colors: Use #2563EB blue for primary elements, #F59E0B amber for
accent/highlight elements, #FAFAFA off-white background. All text
in #1F2937 dark gray.
Font: Modern clean sans-serif font.
Tone: Professional but approachable, suitable for LinkedIn.
```

If a logo is provided:
```
Include a small logo in the bottom-right corner: [describe the logo
or reference it].
```

---

## Multi-Image Series Consistency

When generating a series, include a "series style block" in every prompt:

```
SERIES STYLE (apply to all images in this series):
- Color palette: #2563EB blue, #F59E0B amber, #FAFAFA background
- Font: Clean sans-serif
- Layout: 4:5 portrait, title at top, visual centered
- Corner label: "Part [N] of [TOTAL]" in small text, bottom-left
- Visual language: flat illustration, soft shadows, rounded corners
```

This ensures visual coherence across all posts in the series.

---

## Prompt Patterns by Metaphor Type

### Iceberg Prompt Pattern

```
Generate a professional iceberg infographic about [TOPIC].

The image shows an iceberg in calm water. Above the waterline (small,
roughly 20% of the iceberg visible): [2-3 visible/surface items as
text labels floating near the top portion].

Below the waterline (large, roughly 80% of the iceberg submerged):
[4-6 hidden/deeper items as text labels arranged in the underwater
section].

A thin horizontal waterline divides the image. Above: light blue sky.
Below: deeper blue/teal water.

Title: "[TITLE]" at the top of the image.
Colors: [palette]. Aspect ratio: [ratio].
Style: Clean flat illustration, soft gradients for the water.
Do not add any 3D effects or photorealistic textures.
```

### Pyramid Prompt Pattern

```
Generate a pyramid infographic about [TOPIC] with [N] tiers.

The pyramid is centered in the image with the widest tier at the
bottom and narrowest at the top.

Tiers from bottom to top:
1. (widest) "[Label 1]" — [color 1]
2. "[Label 2]" — [color 2]
3. "[Label 3]" — [color 3]
4. (narrowest) "[Label 4]" — [color 4]

Each tier has its label centered in white text. Tiers are separated
by thin white lines.

Title: "[TITLE]" above the pyramid.
Style: Flat design, no 3D perspective. [palette]. [ratio].
```

### Timeline/Steps Prompt Pattern

```
Generate a timeline infographic showing [N] steps for [TOPIC].

Layout: Vertical timeline with a [color] connecting line on the left.
Each step has a numbered circle ([color]) connected to the line, with
the step title and a one-line description to the right.

Steps:
1. "[Title 1]" — "[Description 1]"
2. "[Title 2]" — "[Description 2]"
3. "[Title 3]" — "[Description 3]"
[...]

Title: "[TITLE]" at the top.
Style: Clean and minimal, [palette], [font], [ratio].
```

### Split Comparison Prompt Pattern

```
Generate a split comparison infographic about [TOPIC].

The image is divided vertically into two halves:

Left half — "[Label A]" (use [color A] tones):
- [Point 1]
- [Point 2]
- [Point 3]

Right half — "[Label B]" (use [color B] tones):
- [Point 1]
- [Point 2]
- [Point 3]

A clean dividing line or subtle gradient separates the two halves.
Title: "[TITLE]" centered at the top spanning both halves.
Style: [palette], [font], [ratio].
```

### Funnel Prompt Pattern

```
Generate a funnel infographic about [TOPIC] with [N] stages.

The funnel is centered and shows progressive narrowing from top to bottom:

1. (widest) "[Stage 1]" — [number/percentage]
2. "[Stage 2]" — [number/percentage]
3. "[Stage 3]" — [number/percentage]
4. (narrowest) "[Stage 4]" — [number/percentage]

Each stage is a different shade, transitioning from [color1] at top
to [color2] at bottom. Labels are centered in each stage with the
number/percentage on the right.

Title: "[TITLE]" above the funnel.
Style: [palette], [font], [ratio].
```

### Concentric Circles Prompt Pattern

```
Generate a concentric circles (bullseye) infographic about [TOPIC].

The image shows [N] nested circles, centered:
- Innermost circle (smallest, darkest): "[Core concept]"
- Ring 2: "[Layer 2]"
- Ring 3: "[Layer 3]"
- Outermost ring (largest, lightest): "[Layer 4]"

Colors gradient from dark [color] at center to light [color] at edges.
Labels are placed within each ring or connected by thin lines.

Title: "[TITLE]" at the top.
Style: [palette], [font], [ratio].
```

### Venn Diagram Prompt Pattern

```
Generate a Venn diagram infographic about [TOPIC].

Show [2-3] overlapping circles:
- Circle A ("[Label A]", [color A]): contains [items unique to A]
- Circle B ("[Label B]", [color B]): contains [items unique to B]
- Overlap (A∩B): contains [shared items]

Circles are semi-transparent so the overlap area is clearly visible.
Each section has its items listed as short text labels.

Title: "[TITLE]" at the top.
Style: [palette], [font], [ratio].
```

---

## Iteration Prompts

When editing a previously generated image, structure the edit prompt clearly:

### Minor Text Changes
```
Edit this infographic: change the title from "[old title]" to "[new title]".
Keep everything else exactly the same.
```

### Color Adjustments
```
Edit this infographic: change the color scheme from blue tones to warm
coral/orange tones. Use #F97316 as the primary color and #14B8A6 as
the accent. Keep the layout, text, and structure identical.
```

### Layout Tweaks
```
Edit this infographic: add more white space around the edges. Move the
title slightly higher. Make the main visual element 10% smaller to
create more breathing room. Keep all text and colors the same.
```

### Adding Elements
```
Edit this infographic: add a small "1/4" label in the bottom-left
corner in light gray text. Also add a thin decorative line under
the title. Keep everything else the same.
```

### Style Shifts
```
Edit this infographic: make the style feel more hand-drawn and
approachable. Add slight curves to the lines, soften the corners,
and give elements a subtle sketch-like quality. Keep the same
content and colors.
```

---

## Negative Prompting

Tell Gemini what to AVOID for cleaner results:

```
Do not include:
- Photorealistic textures or 3D effects
- Stock photo elements or people photos
- Gradients that are too dramatic or neon
- Decorative borders or frames
- Clip art or cartoon characters
- Watermarks or attribution text
- More than [N] colors
```

Common exclusions for professional infographics:
```
Avoid busy backgrounds, drop shadows, glossy effects, or overly
decorative elements. Keep the design clean and flat.
```

---

## Tips for Better Results

1. **Length matters** — Longer, more detailed prompts produce better infographics. Don't be terse.
2. **Quote all text** — Put every text label in quotes so Gemini renders it exactly
3. **Describe spatial relationships** — "above," "below," "to the left of," "centered"
4. **Specify what you DON'T want** — Negative instructions prevent common issues
5. **Reference real design styles** — "flat illustration like you'd see on Dribbble" or "clean infographic style like those on Visual Capitalist"
6. **Iterate in small steps** — Change one thing at a time in edit prompts
7. **Re-state style on edits** — Remind Gemini of the style rules when editing

---

## Creative Freedom Beyond Standard Metaphors

The visual metaphors in the reference guide are starting points, not limits. Effective infographics can use **any visual concept** that makes the data intuitive:

### Think in Terms of Concept Shapes

Ask: "What shape does this idea naturally take?"

- **A bridge** — connecting two worlds or ideas
- **A map/territory** — geographic or conceptual landscape
- **A recipe/ingredients** — components that combine into something
- **A dashboard/cockpit** — multiple metrics at a glance
- **A bookshelf** — categorized knowledge
- **A periodic table** — systematic classification
- **A city skyline** — relative scale/importance shown as building heights
- **A garden** — growth, nurturing, seasons
- **A circuit board** — interconnected systems
- **A mountain path** — journey with elevation changes and rest stops
- **A solar system** — central idea with orbiting concepts
- **A sandwich/layer cake** — stacked components
- **A compass/radar** — directional positioning
- **A thermometer/gauge** — measuring intensity or progress
- **A puzzle** — pieces fitting together to form a whole
- **A toolbox** — collection of tools/resources for a purpose
- **An equation** — inputs combining to produce an output

The key question is always: **"Does this visual make the insight click faster?"**

If you invent a new visual metaphor that perfectly fits the concept, use it. The reference guide provides proven patterns, but the best infographic is the one that makes the viewer understand instantly — regardless of whether it follows a template.
