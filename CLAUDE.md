# Canvas Board — Agent Notes

This is a single self-contained HTML file (`index.html`). No build step, no dependencies, no server.

The product spec and acceptance criteria live in `SPEC.md`. Read that first to understand what the app should do before changing behaviour.

**Whenever a new requirement is implemented or existing behaviour changes, update `SPEC.md` to reflect it before considering the task done.**

---

## Architecture

The canvas is DOM-based, not `<canvas>`-based.

- `#viewport` — full-screen container, `overflow: hidden`, `position: relative`
- `#world` — `position: absolute; width: 0; height: 0; transform-origin: 0 0` — receives a CSS `translate + scale` transform for pan/zoom. All canvas objects are absolutely-positioned children.
- Objects (`.obj`) are `<div>` elements appended to `#world`. Their `left`/`top` are world-space coordinates. The CSS transform on `#world` handles all pan/zoom visually.
- `#handle-overlay` — `position: fixed` (screen space). The resize handle box is repositioned via `getBoundingClientRect()` on the selected object, keeping handle size constant at 9 px regardless of zoom.
- `#text-input` — `contenteditable="plaintext-only"` div, child of `#world`. Re-appended to `#world` last on every activation so it sits above all `.obj` elements in DOM stacking order.

## Key Invariants

- `toWorld(clientX, clientY)` converts screen pixels to world coordinates: `{x: (cx - pan.x) / zoom, y: (cy - pan.y) / zoom}`.
- `sampleLuminance(wx, wy, ww, wh)` draws overlapping image objects into an offscreen canvas and returns `"#ffffff"` or `"#000000"` — never throws; uses `try/catch` around `drawImage`.
- `commitText()` is idempotent — safe to call when `textActive` is false.
- `setTool()` always calls `commitText()` first.
- `suppressBlur` flag prevents double-commit when `openTextAt`/`openEditAt` transitions temporarily move focus.
- `user-select: none` is on `body`; `#text-input` overrides with `user-select: text` — required for contenteditable to render typed characters in Chrome.

## State Shape

```js
state = {
  objects: [{ id, type, x, y, w?, h?, src?, text?, color?, fontSize? }],
  pan:     { x, y },   // screen-space pixel offset
  zoom:    1,           // scalar multiplier
  nextId:  1,           // monotonically increasing
}
```

Persisted to `localStorage` under key `canvas-board-v1`. Only `objects`, `pan`, `zoom`, `nextId` are saved (not tool or selection).

## Undo

`undoStack` holds JSON snapshots of `state.objects` (not full state). Max 40 entries. Pan/zoom are not undoable.

## Common Pitfalls

- `getImageData` is a method on the 2D context (`ctx`), not on the canvas element (`oc`).
- Text objects cannot be resized via handles (`obj.type === 'text'` is excluded in the resize mousedown handler).
- The draw-preview `<div>` must be re-appended to `#world` after `renderAll()` and after each new object is created, because `renderAll` removes all `.obj` children and re-adds them.
- When editing an existing text object, hide it with `opacity: 0` (not `display: none`) to avoid layout shift; restore on commit or discard.
- Text color must be recalculated on commit (new or edit) AND after a drag-move completes.
- Double-click fires after two mousedown/mouseup cycles. Set `drag = null` in the dblclick handler to cancel any drag state set up by the preceding clicks.
