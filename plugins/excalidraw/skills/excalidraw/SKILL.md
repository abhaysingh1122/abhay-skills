---
name: excalidraw-presentation
description: Create visual presentations, slide decks, and explanatory diagrams in Excalidraw. Use when user asks to create a presentation, slide deck, visual explainer, pitch deck, comparison diagram, process flow, or any multi-slide visual content. Supports two output modes — generating .excalidraw JSON files OR injecting slides directly into excalidraw.com via Chrome browser automation (clipboard JS injection + paste). Combines presentation design expertise with Excalidraw technical implementation.
---

# Excalidraw Presentation Designer

Create compelling visual presentations by combining content design thinking with Excalidraw implementation.

## Workflow

### Phase 1: Flesh Out the Idea

Before touching any JSON, think through the presentation as a storytelling problem:

1. **Core message** — What is the ONE thing the audience should remember?
2. **Narrative arc** — What structure carries that message?
   - Problem → Solution → Proof
   - Before → After → How
   - What → Why → How → What If
   - Status Quo → Tension → Resolution
3. **Slide outline** — Each slide gets ONE concept. Write the slide titles first as a sequence that tells the story even without visuals.
4. **Visual concept per slide** — For each slide, decide: comparison? process flow? hierarchy? single image with caption? grid of items?

Present the outline to the user for confirmation before building.

### Phase 2: Verify Chrome Extension
 
This skill requires the **Claude in Chrome** extension to inject slides directly into excalidraw.com.
 
**Step 1:** Check if extension is available using `tabs_context_mcp`
 
**Step 2:** If extension is NOT available or returns an error, tell the user:
 
> To use this skill, please install the Claude in Chrome extension:
> https://chromewebstore.google.com/detail/claude-in-chrome/anthropic
> 
> Once installed, restart Chrome and try again.
 
**Step 3:** If extension IS available, proceed to Phase 3.
 
Read `references/chrome-automation.md` for the full injection workflow.

### Phase 3: Build Slides

Read `references/element-reference.md` for element specs and JSON structure.
Read `references/design-principles.md` for layout patterns, color systems, and design principles.

#### CRITICAL: Generate GroupIds First

Before building any slide, generate a unique groupId. All elements within a slide MUST share the same groupId.
```javascript
const slide1GroupId = `slide1-group-${Math.random().toString(36).substr(2, 6)}`;
```

#### Slide Construction Order

For each slide:
1. Start with a frame rectangle (light background, rounded corners) at the slide's grid position
2. Place the title text (fontSize 28-36, fontFamily 2, bold color)
3. Build the visual structure (boxes, arrows, groups) — keep to 3-5 focal elements max
4. Add labels and annotations last
5. Every element needs: unique `id`, incrementing `seed`, all required properties

#### Multi-Slide Grid

Position slides so the full deck is visible when zoomed out:

```
Slide 1: x=0,    y=0      Slide 2: x=900,  y=0
Slide 3: x=0,    y=600    Slide 4: x=900,  y=600
Slide 5: x=0,    y=1200   Slide 6: x=900,  y=1200
```

Each slide occupies ~800×500px. Leave 100px gaps between slides.

### Phase 4: Verify and Iterate

After injecting/creating each slide:
- Take a screenshot to verify rendering
- Check text isn't clipped (width/height adequate for content)
- Confirm visual hierarchy reads correctly (title → main visual → supporting text)
- Adjust spacing and alignment as needed

## Quick Reference

### Chrome Clipboard Injection Pattern

```javascript
(async () => {
  const elements = [ /* slide elements here */ ];
  const clipboardData = {
    type: "excalidraw/clipboard",
    elements: elements,
    files: {}
  };
  await navigator.clipboard.writeText(JSON.stringify(clipboardData));
  return "Slide ready";
})()
```

Then paste with `cmd+v` (Mac) or `ctrl+v` (Windows/Linux).

### Color Palette

| Purpose | Light | Dark |
|---------|-------|------|
| Primary/Blue | #a5d8ff | #1971c2 |
| Success/Green | #b2f2bb | #2f9e44 |
| Warning/Yellow | #fff3bf | #f08c00 |
| Danger/Red | #ffc9c9 | #e03131 |
| Neutral/Gray | #e9ecef | #495057 |
| Text | — | #1e1e1e |
| Accent/Orange | — | #d97757 |

### Fonts

- `1` = Virgil (hand-drawn, casual)
- `2` = Helvetica (clean, professional — **default for presentations**)
- `3` = Cascadia (code/monospace)