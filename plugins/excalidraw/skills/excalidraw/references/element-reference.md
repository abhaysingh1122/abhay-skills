# Excalidraw Element Reference

Complete JSON specs for all element types used in presentations.

## Required Properties (All Elements)

## Grouping (CRITICAL)

All elements within a single slide MUST share the same `groupIds` value.

### GroupId Format
```
slide{N}-group-{randomSuffix}
```

Example: `"groupIds": ["slide1-group-x7k2m"]`

### Generate in JavaScript
```javascript
const slideGroupId = `slide1-group-${Math.random().toString(36).substr(2, 6)}`;
```

Every element MUST include:

```json
{
  "id": "unique-string",
  "seed": 1001,
  "version": 1,
  "versionNonce": 1,
  "isDeleted": false,
  "groupIds": ["slide1-group-abc123"],
  "boundElements": null,
  "link": null,
  "locked": false,
  "fillStyle": "solid",
  "strokeWidth": 2,
  "strokeStyle": "solid",
  "roughness": 1,
  "opacity": 100
}
```

## Element Types

### Rectangle

```json
{
  "type": "rectangle",
  "id": "rect-1",
  "x": 100, "y": 100, "width": 200, "height": 80,
  "angle": 0,
  "strokeColor": "#1e1e1e",
  "backgroundColor": "#a5d8ff",
  "fillStyle": "solid",
  "strokeWidth": 2,
  "roughness": 1,
  "opacity": 100,
  "roundness": { "type": 3 }
}
```

- `roundness: { type: 3 }` = rounded corners (use for cards, containers)
- `roundness: null` = sharp corners
- For container boxes that hold text, add `"boundElements": [{ "id": "text-id", "type": "text" }]`

### Ellipse

```json
{
  "type": "ellipse",
  "id": "ell-1",
  "x": 100, "y": 100, "width": 120, "height": 120,
  "strokeColor": "#1e1e1e",
  "backgroundColor": "#b2f2bb"
}
```

### Diamond

```json
{
  "type": "diamond",
  "id": "dia-1",
  "x": 100, "y": 100, "width": 100, "height": 100,
  "strokeColor": "#1e1e1e",
  "backgroundColor": "#fff3bf"
}
```

### Text (Standalone)

```json
{
  "type": "text",
  "id": "text-1",
  "x": 100, "y": 100, "width": 200, "height": 25,
  "text": "Hello World",
  "fontSize": 20,
  "fontFamily": 2,
  "textAlign": "center",
  "verticalAlign": "top",
  "strokeColor": "#1e1e1e",
  "backgroundColor": "transparent",
  "containerId": null,
  "originalText": "Hello World",
  "lineHeight": 1.25
}
```

**fontSize guide for presentations:**
- Slide title: 28-36
- Section header: 20-24
- Body text: 14-18
- Caption/annotation: 12-14
- Labels inside small boxes: 12-16

**textAlign:** `"left"`, `"center"`, `"right"`

### Text (Bound to Container)

To put text inside a rectangle/ellipse/diamond:

```json
{
  "type": "rectangle",
  "id": "box-1",
  "boundElements": [{ "id": "label-1", "type": "text" }],
  "x": 100, "y": 100, "width": 160, "height": 60
},
{
  "type": "text",
  "id": "label-1",
  "containerId": "box-1",
  "x": 110, "y": 118,
  "width": 140, "height": 24,
  "text": "Step 1",
  "fontSize": 16,
  "fontFamily": 2,
  "textAlign": "center",
  "verticalAlign": "middle",
  "originalText": "Step 1"
}
```

**Critical rules for bound text:**
- Container's `boundElements` must list the text element
- Text's `containerId` must match the container's `id`
- Text x/y should be inset ~10px from container x/y
- Text width should be container width minus ~20px
- Use `verticalAlign: "middle"` for centered text in boxes

### Arrow

```json
{
  "type": "arrow",
  "id": "arrow-1",
  "x": 300, "y": 140, "width": 80, "height": 0,
  "strokeColor": "#1e1e1e",
  "strokeWidth": 2,
  "roughness": 1,
  "points": [[0, 0], [80, 0]],
  "startArrowhead": null,
  "endArrowhead": "arrow",
  "startBinding": null,
  "endBinding": null
}
```

**Arrow directions via points:**
- Right: `[[0,0], [80,0]]`
- Down: `[[0,0], [0,80]]`
- Diagonal: `[[0,0], [80,60]]`
- Curved: `[[0,0], [40,-30], [80,0]]` (3+ points create curves)

**Arrowheads:** `null`, `"arrow"`, `"triangle"`, `"dot"`, `"bar"`

### Line

```json
{
  "type": "line",
  "id": "line-1",
  "x": 100, "y": 200, "width": 150, "height": 0,
  "strokeColor": "#868e96",
  "strokeWidth": 1,
  "points": [[0, 0], [150, 0]]
}
```

### Frame (for slide boundaries)

Use a large rectangle with very light fill as a visual slide frame:

```json
{
  "type": "rectangle",
  "id": "frame-1",
  "x": 0, "y": 0, "width": 800, "height": 500,
  "strokeColor": "#dee2e6",
  "backgroundColor": "#f8f9fa",
  "fillStyle": "solid",
  "strokeWidth": 1,
  "roughness": 0,
  "roundness": { "type": 3 }
}
```

Put the frame element FIRST in the elements array so all other elements render on top.

## Color System

### Semantic Palette

| Role | Background (light) | Stroke/Text (dark) | Use for |
|------|-------------------|--------------------|---------| 
| Primary | #a5d8ff | #1971c2 | Main concepts, containers, emphasis |
| Secondary | #d0ebff | #1971c2 | Sub-elements within primary containers |
| Success | #b2f2bb | #2f9e44 | Positive states, benefits, checkmarks |
| Warning | #fff3bf | #f08c00 | Caution, attention points |
| Danger | #ffc9c9 | #e03131 | Problems, negatives, errors |
| Neutral | #e9ecef | #495057 | Backgrounds, secondary text, dividers |
| Canvas | #f8f9fa | #dee2e6 | Slide frames, subtle backgrounds |

### Text Colors

- Primary text: `#1e1e1e`
- Secondary text: `#495057`
- Muted text: `#868e96`
- On blue background: `#1971c2`
- Positive: `#2f9e44`
- Negative: `#e03131`

## Spacing and Sizing

### Standard Dimensions

- Slide frame: 800×500
- Card/box (large): 200×80 to 280×160
- Card/box (small): 80×50 to 120×60
- Icon placeholder: 40×40 to 60×60
- Gap between parallel elements: 20-30px
- Padding inside containers: 15-20px
- Gap between slides: 100px

### Text Sizing

| Context | fontSize | height estimate |
|---------|----------|-----------------|
| Slide title | 32 | 40 |
| Section header | 22 | 28 |
| Body text | 16 | 20 per line |
| Small label | 14 | 18 |
| Annotation | 12 | 15 |

Height for multi-line text: `lines × fontSize × lineHeight`

## ID and Seed Conventions

### IDs

Use descriptive, hyphenated IDs that encode the slide and role:

```
s1-title       → Slide 1 title
s1-box-email   → Slide 1, box for "email" concept  
s1-label-email → Slide 1, label inside email box
s2-arrow-1     → Slide 2, first arrow
```

### Seeds

Use non-overlapping ranges per slide:
- Slide 1: 1000-1099
- Slide 2: 2000-2099
- Slide 3: 3000-3099
- ...and so on

Increment by 1 within each slide. No two elements anywhere should share a seed.