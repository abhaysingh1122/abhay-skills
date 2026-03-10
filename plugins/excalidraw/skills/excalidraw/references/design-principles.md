# Presentation Design Principles

Visual design thinking for content creators building Excalidraw presentations.

## Core Rule: One Concept Per Slide

A slide that tries to say two things says nothing. Every slide should pass this test: "Can someone glance at this for 3 seconds and understand the point?"

If you need a paragraph of text to explain a slide, the slide design has failed. Use visuals to carry the meaning, text to label.

## Narrative Structures

### Problem → Solution → Proof (Best for pitches)

```
Slide 1: The Problem (pain point, visual of what's broken)
Slide 2: Why It Exists (root cause, systemic diagram)
Slide 3: The Solution (your approach, clean visual)
Slide 4: How It Works (process flow, 3-5 steps)
Slide 5: Proof (results, metrics, before/after)
Slide 6: Next Steps (CTA, single action)
```

### Before → After → How (Best for tutorials)

```
Slide 1: Before State (messy, complex, red tones)
Slide 2: After State (clean, simple, green tones)
Slide 3-N: Step by Step (process flow showing the transformation)
```

### What → Why → How (Best for explainers)

```
Slide 1: What is [concept] (visual definition)
Slide 2: Why it matters (impact/consequences diagram)
Slide 3-4: How it works (mechanism/process)
Slide 5: How to apply it (practical steps)
```

### Comparison/Framework (Best for educational)

```
Slide 1: Overview/title
Slide 2: Side-by-side comparison (left vs right)
Slide 3-5: Deep dive into each dimension
Slide 6: Summary matrix or decision framework
```

## Slide Layout Patterns

### Title Slide
- Large centered title text (fontSize 36)
- Subtitle below (fontSize 18, muted color)
- Optional icon or accent shape
- Clean, minimal — set the tone

### Comparison (Left vs Right)
- Vertical divider (line or implicit space) at center
- Left side: "Before" / "Bad" / "Old" — use red/gray tones
- Right side: "After" / "Good" / "New" — use green/blue tones
- Arrow or "→" between them
- Mirrored layout (same box sizes both sides)

### Process Flow (Horizontal)
- 3-5 boxes in a row, connected by arrows
- Each box: icon placeholder + label
- Consistent sizing across all steps
- Number labels if sequence matters

### Process Flow (Vertical)
- Steps stacked top to bottom
- Arrows pointing down between each
- Good for more than 5 steps or when steps need descriptions

### Hierarchy / Nested
- Large outer container (light fill)
- Smaller boxes inside (darker fill)
- Shows composition, containment, or grouping
- Label the outer container with the category name

### Grid Layout
- 2×2 or 2×3 grid of equal-sized cards
- Each card: icon + title + brief text
- Good for features, benefits, team members
- Consistent card dimensions and spacing

### Single Concept + Annotation
- One large visual element centered
- 2-3 callout arrows pointing to parts
- Labels explaining each part
- Good for architecture diagrams, product shots

### Metrics / Results
- Large number (fontSize 48-60) as the focal point
- Label underneath explaining the metric
- Before/after numbers side by side
- Use green for positive changes, red for negative

## Visual Hierarchy Rules

### Size Communicates Importance
- Most important element is the largest
- Supporting elements are smaller
- Annotations are smallest

### Color Communicates Meaning
- Blue: primary concepts, the "thing you're explaining"
- Green: positive outcomes, correct approaches, benefits
- Red: problems, pain points, what's wrong
- Yellow/Orange: warnings, attention points, transitions
- Gray: background elements, secondary information

### Position Communicates Sequence
- Left to right = time flow or progression
- Top to bottom = hierarchy or priority
- Center = focal point
- Periphery = supporting context

### Contrast Creates Focus
- High contrast (dark stroke on light fill) = important
- Low contrast (light stroke on light fill) = background
- One element should "pop" per slide — the thing the eye goes to first

## Typography Rules for Excalidraw

### Always Use fontFamily: 2 (Helvetica) for Presentations

Hand-drawn font (Virgil) is fine for informal diagrams but makes presentations hard to read. Use clean Helvetica for all presentation text.

### Text Sizing Hierarchy

Each slide should use at most 3 font sizes:
1. Title: 28-36
2. Labels/Headers: 16-20
3. Supporting text: 12-14

Never use more than 3 sizes on one slide.

### Text Alignment

- Titles: `textAlign: "center"`
- Labels in boxes: `textAlign: "center"`, `verticalAlign: "middle"`
- Body text / lists: `textAlign: "left"`
- Metrics / numbers: `textAlign: "center"`

## Common Mistakes to Avoid

1. **Too many elements** — If a slide has more than 15-20 elements, split it into two slides
2. **Text walls** — If you're writing sentences, you're doing it wrong. Use keywords and visuals.
3. **Inconsistent sizing** — Parallel elements (boxes in a row, cards in a grid) must be the same size
4. **Rainbow colors** — Use max 2-3 colors per slide from the semantic palette
5. **No visual anchor** — Every slide needs one element that's clearly the "main thing"
6. **Orphan elements** — Every element should be visually connected to something (via proximity, arrows, containers, or alignment)
7. **Roughness mismatch** — Use `roughness: 1` (subtle hand-drawn) consistently, or `roughness: 0` (clean) consistently. Don't mix.