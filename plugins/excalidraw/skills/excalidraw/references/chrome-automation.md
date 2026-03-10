# Chrome Browser Automation for Excalidraw

Step-by-step procedure for injecting Excalidraw slides directly into excalidraw.com via Claude in Chrome.

## Prerequisites

- **Claude in Chrome extension** installed and active
  - If not installed: https://chromewebstore.google.com/detail/claude-in-chrome/anthropic
  - Browser window available

### Checking Extension Availability

Before starting, call `tabs_context_mcp`. If it errors or returns no context, the extension is not available.
 
**If extension unavailable, respond to user:**
```
To create Excalidraw presentations, please install the Claude in Chrome extension:
https://chromewebstore.google.com/detail/claude-in-chrome/anthropic
 
Once installed:
1. Restart Chrome
2. Click the extension icon and connect
3. Try your request again
```
 
Do NOT fall back to creating .excalidraw files. The Chrome injection workflow is required.

## Workflow

### Step 1: Get Tab Context

```
Tool: tabs_context_mcp (createIfEmpty: true)
```

This returns available tabs. If no Excalidraw tab exists, create one:

```
Tool: tabs_create_mcp
```

### Step 2: Navigate to Excalidraw

```
Tool: navigate (tabId: <tab_id>, url: "https://excalidraw.com")
```

Wait for load, then take a screenshot to confirm the canvas is ready.

### Step 3: Focus the Canvas

**Critical:** The clipboard API requires document focus. Always click the canvas before injecting.

```
Tool: computer (action: "left_click", coordinate: [600, 400], tabId: <tab_id>)
```

Click an empty area of the canvas, not on any toolbar or UI element.

### Step 4: Inject Slide via Clipboard

### Elements Pasting on Top of Each Other / Can't Move Together

**Cause:** Missing or empty `groupIds`.
**Fix:** Ensure ALL elements share the same `groupIds` array value.

Use `javascript_tool` to write Excalidraw clipboard JSON:

```javascript
(async () => {
  const elements = [
    // ... all slide elements here
  ];
  const clipboardData = {
    type: "excalidraw/clipboard",
    elements: elements,
    files: {}
  };
  await navigator.clipboard.writeText(JSON.stringify(clipboardData));
  return "Slide N ready";
})()
```

**Key format:** The clipboard payload MUST have `type: "excalidraw/clipboard"` — Excalidraw checks this to recognize pasted content as native elements rather than plain text.

### Step 5: Paste

```
Tool: computer (action: "key", text: "cmd+v", tabId: <tab_id>)
```

Use `ctrl+v` on Windows/Linux. Elements appear selected on the canvas.

### Step 6: Verify

```
Tool: computer (action: "screenshot", tabId: <tab_id>)
```

Check the screenshot. If elements are misaligned or text is clipped, adjust and re-inject.

### Step 7: Deselect and Continue

Click an empty area to deselect the pasted elements before injecting the next slide:

```
Tool: computer (action: "left_click", coordinate: [1200, 400], tabId: <tab_id>)
```

Then repeat Steps 4-7 for each subsequent slide.

## Error Handling

### "Document is not focused" Error

This is the most common error. The clipboard API fails if the browser tab doesn't have focus.

**Fix:** Click the canvas first (Step 3), then retry the JS injection immediately.

### Extension Disconnected

If the Chrome extension loses connection mid-workflow:
- Inform the user to restart Chrome and/or the extension
- The canvas retains all previously pasted content
- Resume from the last incomplete slide after reconnection

### Elements Not Appearing After Paste

Possible causes:
- Elements positioned off-screen → check x/y coordinates are within visible viewport
- Missing required properties → every element needs `id`, `seed`, `version`, `versionNonce`, `isDeleted`, `groupIds`, `boundElements`
- Invalid `containerId` reference → bound text elements must reference an existing container element `id`

## Multi-Slide Strategy

When injecting multiple slides into one canvas:

1. **Offset each slide's elements** by the grid position (e.g., Slide 2 adds x+900 to all elements)
2. **Inject one slide at a time** — clipboard paste replaces previous clipboard content
3. **Deselect between pastes** — click empty canvas area between slides
4. **Increment seeds globally** — no two elements across any slide should share a seed value. Use ranges: Slide 1 = 1000-1099, Slide 2 = 2000-2099, etc.

## Performance Notes

- Keep each slide under ~30 elements for clean rendering
- Minimize the JS payload size — use compact property names and avoid unnecessary whitespace in the JSON
- The clipboard write + paste cycle takes ~2-3 seconds per slide
- For decks >6 slides, consider batching: inject slides 1-3, verify, then 4-6