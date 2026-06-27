# Canvas Board — Acceptance Criteria

A single-page, infinite-canvas board application. No server required. All state lives in the browser.

---

## 1. Canvas

- The canvas is infinite in all directions. There is no fixed boundary.
- The viewport fills the entire browser window (no scrollbars, no chrome).
- The background is solid black (`#000000`) with a subtle white dot grid (`rgba(255,255,255,0.07)`, 28 px spacing) rendered behind all content.
- The entire canvas can be panned and zoomed.

### 1.1 Pan

- **Drag on empty background** (left mouse button, select tool) → pans the canvas.
- **Middle-mouse button drag** (any tool) → pans the canvas.
- Pan is stored as a pixel offset `{x, y}` applied to the world container.

### 1.2 Zoom

- **Scroll wheel** (when not editing text and no rect is selected) → zooms in/out.
- Zoom is centred on the cursor position — the world point under the cursor stays fixed.
- Zoom range: `0.05×` to `20×`.
- Each scroll step multiplies/divides by `1.12`.

---

## 2. State & Persistence

- The entire canvas state is auto-saved to **IndexedDB** (debounced, ~300 ms) after every mutating action.
- Two IndexedDB object stores:
  - `state` — canvas state (object list without image `src`, pan, zoom, next ID counter).
  - `images` — image Blobs, keyed by object ID.
- Images are kept in memory as both a data URL (`obj.src`, used by `<img>`) and in an `imageBlobs: Map<id, Blob>` cache used exclusively for persistence.
- On page load, blobs are fetched and converted back to data URLs to populate `obj.src`.
- **One-time migration**: if a `canvas-board-v1` key exists in `localStorage` (old format), it is migrated to IndexedDB on first load and then removed.
- Persistence failures are silently ignored.

### 2.1 Object Model

Every object on the canvas has:

| Field | Type | Description |
|---|---|---|
| `id` | integer | Unique, monotonically increasing |
| `type` | `"image"` \| `"rect"` \| `"text"` | Object kind |
| `x` | number | World-space left edge |
| `y` | number | World-space top edge |
| `w` | number | World-space width |
| `h` | number | World-space height |

Additional fields by type:

**image**: `src` — base64 data URL of the image.

**rect**: `colorIndex` — integer index (0–3) into the color palette (default `0` = red).

**text**: `text` (string), `color` (`"#ffffff"` or `"#000000"`), `fontSize` (number, world-space pixels). `w` and `h` are captured from the rendered input size at commit time.

---

## 3. Tools

A floating toolbar at the top-centre of the screen provides three tools. The active tool is visually highlighted. Switching tools (via button or keyboard shortcut) clears any browser focus ring from the toolbar. Switching tools also commits any in-progress text and clears the selection.

A separate **file menu button** (frosted glass, 36 × 36 px, top-left at `top: 20px; left: 20px`) opens a panel below it with Import, Export, and Wipe actions (see §3.4).

### 3.1 Select (keyboard: `S`)

- Default tool on load.
- **Click an object** → selects it, clears any previous selection.
- **Click empty space** → deselects all, pans the canvas.
- **Click or drag anywhere within the selection bounding box** → moves all selected objects together.
- **Drag any selected object** → moves all selected objects together.

#### Multi-select

- **Shift+click an object** → adds that object to the selection without clearing others.
- **Shift+drag** (anywhere; rubber-band activates after 6 px of movement):
  - **Right drag** (contained mode) — solid blue border preview; an object is selected only if its entire bounding box lies **fully inside** the rubber-band rectangle.
  - **Left drag** (intersecting mode) — dashed blue border preview; any object with **any overlap** with the rectangle is selected.
  - Objects already in the selection remain selected.
  - A shift+mousedown that never reaches 6 px of movement is treated as a plain shift-click on whatever was under the cursor.
- **Delete** or **Backspace** → removes **all** selected objects.
- **Arrow keys** → nudges **all** selected objects by 1 px in the pressed direction.

#### Selection visual

- Each selected object displays its own Apple-blue outline (`rgba(10,132,255,0.85)`, zoom-invariant 1.5 px) via CSS `box-shadow: 0 0 0 calc(1.5px / var(--zoom)) ...`.
- 8 resize handles appear only when exactly **one non-text object** is selected (see §5).
- Cursor: default arrow; changes to move cursor when hovering over any selected object.

### 3.2 Text (keyboard: `T`)

- **Double-click empty space or any non-text object** (from any tool) → enters text mode, opens an inline text input at that world position.
- **Double-click a text object** (from any tool) → opens that text object for inline editing.
- **Single click anywhere while in text mode** → commits any in-progress text and switches to Select tool.
- While editing:
  - The text input is positioned in world coordinates and rendered with the same font/weight/size as committed text objects.
  - The caret colour is Apple blue (`#0A84FF`).
  - **Scroll wheel** resizes the font (range: 6 px – 400 px, step: 2 px per tick). The canvas does **not** zoom while text is being edited.
  - **Escape** → discards the text without saving, restores the original object if editing an existing one.
  - Clicking away (blur) → commits the text.
- On commit:
  - Empty or whitespace-only text is discarded.
  - The text color (`#ffffff` or `#000000`) is chosen automatically for maximum WCAG contrast against the pixels underneath (see §6).
  - When editing an existing text object: the original is hidden during editing and updated in-place on commit (not duplicated).
- Font size persists across text placements within a session.

### 3.3 Highlight (keyboard: `H`)

- **Click and drag** → draws a dashed red rectangle preview; releases commit it as a highlight rect.
- Minimum committed size: 6 × 6 px. Smaller drags are discarded.
- **Click without dragging** → switches to Select tool and selects the object under the cursor (if any).
- Preview style: dashed red border (`rgba(255,59,48,0.65)`), very light red fill.
- Committed rect style: solid 2 px border, 7% opacity fill, 4 px border radius.
- Cursor: crosshair.

#### Highlight color cycling

- Default color is red. When a single rect is selected, **scroll wheel** cycles through four colors:
  - Scroll up = forward; scroll down = backward.
  - Order: red (`#FF3B30`) → yellow (`#FFD60A`) → light blue (`#64D2FF`) → green (`#30D158`) → red …
  - Each color change is undoable and persisted.

### 3.4 File Menu

A frosted glass button (same visual language as toolbar) fixed at `top: 20px; left: 20px`. Clicking it toggles a panel that expands downward with three actions. Clicking outside the panel closes it.

#### Import
- Opens a native file picker filtered to `.json`.
- Validates the selected file: must be valid JSON with `_type === "canvas-board-export"`.
- On validation failure: shows an `alert()` and aborts.
- On success: shows a `confirm()` warning that the current canvas will be replaced, then replaces state and saves.

#### Export
- Serializes the current canvas (objects with full image data URLs, pan, zoom, nextId) to a JSON file with the markers `_type: "canvas-board-export"` and `_version: 1`.
- Triggers a browser download named `canvas-board-export.json`.

#### Wipe
- Requires confirmation via `confirm()` dialog.
- On confirm: removes all objects, resets the ID counter to 1, clears selection, clears the `images` IndexedDB store, saves.

---

## 4. Paste Images

- Pasting (`Ctrl+V` / `Cmd+V`) reads image items from the clipboard.
- Each pasted image is centred on the current viewport centre (in world space).
- Stored as a data URL on the object (`obj.src`) and as a Blob in the `imageBlobs` cache.
- Natural image dimensions are used for initial width/height.
- Pushes an undo checkpoint before adding.

---

## 5. Resize

- 8 resize handles (circular, 9 px, white disc with Apple-blue ring) appear in **screen space** at corners and midpoints of the selected object's bounding box. Handle size is always 9 px regardless of zoom.
- Handles are shown only when exactly **one non-text** object is selected. Multi-select and text objects show no handles.
- Handles scale to `1.35×` on hover.
- **Drag a handle** → resizes the object from that edge/corner.
- **Hold Shift + drag a corner handle** → locks the aspect ratio; the axis that moved proportionally more drives the scale.
- **Hold Shift + drag an edge handle** → locks the aspect ratio, adjusting the other dimension symmetrically around the object's centre.
- Minimum object size after resize: 10 × 10 px.

---

## 6. Automatic Text Color (WCAG Contrast)

Text color is determined at placement time and recalculated whenever a text object is moved. The algorithm selects either white (`#ffffff`) or black (`#000000`):

1. Render all image objects that overlap the text bounding box into an offscreen `w × h` canvas (world pixels). Background defaults to black. Rect objects are not sampled.
2. Compute per-pixel WCAG relative luminance:
   - `c_linear = c/255 ≤ 0.04045 ? c/12.92 : ((c+0.055)/1.055)^2.4`
   - `L = 0.2126·R + 0.7152·G + 0.0722·B`
3. Average luminance `L` over all pixels.
4. Compare contrast ratios: white contrast = `1.05 / (L+0.05)`, black contrast = `(L+0.05) / 0.05`.
5. Choose white if white contrast ≥ black contrast, else black.

---

## 7. Undo

- `Ctrl+Z` / `Cmd+Z` undoes the last mutating action.
- Undo stack depth: 40 levels.
- Undoable actions: paste image, create highlight rect, change highlight color, create text, edit text, delete object(s), move object(s), resize object, nudge object(s) with arrow keys.
- Undo restores the object list only (pan and zoom are not undoable).
- After undo, the selection is cleared.

---

## 8. Keyboard Shortcuts

| Key | Action | Condition |
|---|---|---|
| `S` | Switch to Select tool | Not editing text |
| `T` | Switch to Text tool | Not editing text |
| `H` | Switch to Highlight tool | Not editing text |
| `Delete` / `Backspace` | Delete all selected objects | Not editing text |
| `Arrow keys` | Nudge all selected objects 1 px | Not editing text |
| `Ctrl/Cmd + Z` | Undo | Not editing text |
| `Shift` (held) | Add to selection on click; lock aspect ratio on resize handle drag | — |
| `Escape` | Discard text input | While editing text |

Switching tools via keyboard shortcut removes the browser focus ring from any previously clicked toolbar button.

---

## 9. Visual Design (Apple HIG)

| Element | Spec |
|---|---|
| Background | `#000000` |
| Grid | White dots, 7% opacity, 28 px spacing |
| Toolbar background | `rgba(30,30,32,0.6)`, `backdrop-filter: blur(40px) saturate(200%)` |
| Toolbar border | `rgba(255,255,255,0.11)`, 1 px |
| Toolbar border radius | 22 px |
| Toolbar shadow | `0 12px 40px rgba(0,0,0,0.55)`, `0 2px 8px rgba(0,0,0,0.3)` |
| Tool button (inactive) | `rgba(255,255,255,0.45)` text, transparent bg |
| Tool button (hover) | `rgba(255,255,255,0.07)` bg, `rgba(255,255,255,0.8)` text |
| Tool button (active) | `rgba(255,255,255,0.14)` bg, `#ffffff` text |
| File menu button | Frosted glass, 36 × 36 px, border-radius 14 px, same glass treatment as toolbar |
| File menu panel items (hover) | `rgba(255,255,255,0.07)` bg |
| File menu Wipe item (hover) | `rgba(255,59,48,0.15)` bg, `#FF3B30` text |
| Highlight default color | `#FF3B30` (Apple system red) |
| Selection color | `#0A84FF` (Apple system blue) |
| Selection outline | Zoom-invariant 1.5 px `box-shadow` per selected object |
| Resize handle | 9 px white disc, `rgba(10,132,255,0.9)` 1.5 px ring, `0 2px 8px rgba(0,0,0,0.55)` shadow |
| Text font | `-apple-system`, `SF Pro Display`, `system-ui`, sans-serif |
| Text weight | 600 |
| Text line height | 1.3 |
| Caret color | `#0A84FF` |
| Toolbar shortcut underline | The character matching the shortcut key is underlined in each tool label |

---

## 10. Hint Bar

- A centred hint appears at the bottom of the screen when the canvas is empty.
- It fades out once at least one object exists.
- Content: `"Paste an image to add it · Scroll to zoom · Drag background to pan"` and `"Delete removes selected · Shift + drag corner/edge = lock aspect ratio"`.
