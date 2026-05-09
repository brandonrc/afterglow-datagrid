# Afterglow — Design Spec

*phosphor faded. afterglow remains.*

A single-file, zero-dependency clone of the [PhosphorJS DataGrid example](https://phosphorjs.github.io/examples/datagrid/) that beats it on perf, ships in one HTML file, and adds four showcase tabs Phosphor can't match. Written from scratch — no Phosphor/Lumino source code is read or copied.

## Goals

1. **Feature parity** with all five Phosphor demo tabs (Two Trillion Rows/Cols, Streaming Rows, Random Ticks 1, Random Ticks 2, JSON Data).
2. **Beat Phosphor on perf** — measurably smoother scroll and tick at 60fps, proven by an in-page FPS HUD.
3. **Add four showcase tabs** that demonstrate capabilities Phosphor's example doesn't:
   - 9 Quadrillion Rows/Cols (virtual ceiling at `Number.MAX_SAFE_INTEGER`)
   - 200×200 Heatmap with full-grid ticks at 60Hz
   - 10K-row firehose stream with 1ms updates
   - Frozen rows/cols + sortable headers
4. **Zero external dependencies** — single `index.html`, no imports, no CDN, no fonts, no fetch, no build step.
5. **Modern evergreen browsers only** (Chrome/Firefox/Safari, latest 2 versions).

## Non-Goals

- Mobile/touch is supported via Pointer Events, but not specifically polished for phones.
- No accessibility tree work beyond keyboard navigation. (The reference has none either.)
- No internationalization, RTL, or font fallback configuration.
- No persistence of dock layout across reloads.
- No theming beyond the five style presets needed for parity.

## Hard Constraints

- **No imports of any kind.** No `import ... from ...` statements, no `import(...)` dynamic imports, no `<script src=...>`, no `<link rel="stylesheet" href=...>`, no `@import` in CSS, no `new Worker(url)` to external code. Verifiable by `grep -E '(\\bimport[ (]|<script[^>]+src|<link[^>]+href|@import)' index.html` returning empty.
- **No external network requests at runtime.** DevTools Network tab shows only the single HTML document load.
- **No copy-paste from Phosphor / Lumino source.** Behavior is observed in a browser and reimplemented from primitives. The Cars dataset (which is public-domain reference data, not Phosphor's code) is the one allowed inline asset.
- **One file** — `index.html` containing one `<style>` block and one inline `<script type="module">`.

## Architecture

```
┌──────────────────────────────────── index.html ──────────────────────────────┐
│  <style>  (~150 lines: reset, dock chrome, splitters, scrollbars, HUD)       │
│  <script type="module">                                                      │
│     §1 Constants, viridis LUT, cars dataset                                  │
│     §2 EventChannel (pub/sub)                                                │
│     §3 DataModel + 6 concrete models                                         │
│     §4 Style presets, cell renderers                                         │
│     §5 Grid (canvas, scroll, paint, hit-test, kbd/mouse)                     │
│     §6 DockNode tree (SplitNode, TabNode, drag/drop overlay)                 │
│     §7 PerfHUD                                                               │
│     §8 main()                                                                │
│  </script>                                                                   │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Boot sequence

`main()` runs once on load and:

1. Constructs nine `DataModel` instances.
2. Constructs nine `Grid` instances bound to those models.
3. Wraps each grid in a `Tab` and inserts them into the recursive `DockNode` tree.
4. Mounts the root `DockNode` to `#app`.
5. Starts one master `requestAnimationFrame` loop that paints any dirty grid in the active tab of each dock zone.

### Single rAF heartbeat

All redraws go through one `requestAnimationFrame` callback. Models that emit changes (streaming inserts, ticks, firehose updates) only mark their grid `dirty`; they never call paint directly. Scroll, drag, and resize also only mark dirty. The rAF callback paints each visible-and-dirty grid exactly once per frame.

This is the central perf decision and the principal way Afterglow beats Phosphor at high update rates: a 1ms firehose emits ~16 events per frame; Phosphor's example reacts per event, Afterglow paints once per frame regardless.

## Components

### 2. EventChannel

A 30-line pub/sub: `on(type, fn)`, `off(type, fn)`, `emit(type, payload)`. Used by `DataModel` to notify grids of `cells-changed`, `rows-inserted`, `rows-removed`, `reset`. No external observable library.

### 3. DataModel

Base class:
```js
class DataModel {
  rowCount(region) { ... }            // 'body' | 'column-header'
  columnCount(region) { ... }         // 'body' | 'row-header'
  data(region, row, col) { ... }      // 'body' | 'row-header' | 'column-header' | 'corner-header'
  emit(event) { ... }                 // forwards to grid via EventChannel
}
```

Concrete models:

| Model | Constructor | Behavior |
|---|---|---|
| `LargeModel(rows, cols)` | virtual; `data()` returns `(r, c)` string | Backs Two Trillion + 9 Quadrillion tabs. |
| `StreamingModel(cols, maxRows=500, intervalMs=250)` | rows splice in/out at random index | Backs Streaming Rows. |
| `RandomTicksModel(rows, cols, intervalMs)` | one random cell mutates per tick | Backs Random Ticks 1, 2, and 200×200 showcase (where `intervalMs=16` and tick mutates *all* cells). |
| `FirehoseModel(cols, rows, intervalMs)` | one full row's worth of values updated per tick at 1ms | Backs 10K Stream showcase. |
| `JSONModel(rows[], schema)` | static, indexed | Backs JSON Data tab. |
| `SortableModel(jsonModel, frozenRows, frozenCols)` | wraps JSONModel with index permutation + freeze metadata | Backs Frozen+Sortable showcase. |

The 200×200 model variant is parameterized: when `intervalMs <= 16`, the tick batches all cells into a single `cells-changed` event covering the whole body region rather than per-cell events.

### 4. Style presets and cell renderers

Five style presets, faithful to Phosphor's hex values:
- `defaultStyle` — white bg, light gray gridlines, blue selection.
- `blueStripeStyle` — alternating row tint `rgba(138,172,200,0.3)`, column tint `rgba(100,100,100,0.1)`. Used by Two Trillion + 9 Quadrillion.
- `brownStripeStyle` — alternating column tint `rgba(165,143,53,0.2)`. Used by Streaming Rows.
- `greenStripeStyle` — alternating row tint `rgba(64,115,53,0.2)`. Used by JSON Data + Frozen showcase.
- `lightSelectionStyle` — translucent white selection over heatmap bg. Used by Random Ticks 2 and 200×200 showcase.

Cell renderers (function from `{ value, row, col }` → `{ font, color, bg, format, align }`):
- `textRenderer` — default sans-serif, left-aligned.
- `redGreenBlackRenderer` — `value ≤ ⅓` → black, `≤ ⅔` → red, else green. Right-aligned, 2-digit fixed format. Used by Random Ticks 1.
- `viridisHeatmapRenderer` — fill = viridis(value), text = inverse-viridis(value). Right-aligned, 2-digit fixed format. Used by Random Ticks 2 and 200×200 showcase. Backed by an inline 256-entry RGB lookup table (~3KB).

### 5. Grid

One `<canvas>` per grid, sized to container × `devicePixelRatio`.

**Geometry** (uniform sizing — non-uniform is a non-goal):
- `rowHeight`, `columnWidth`, `rowHeaderWidth`, `columnHeaderHeight` (defaults 20/64/64/20).
- `scrollX`, `scrollY` in CSS pixels.
- `firstVisibleRow(scrollY, height)` and `lastVisibleRow(...)` are O(1) — this is what makes virtual trillion-row counts feasible.

**Paint pipeline** (per dirty frame):
1. Compute visible cell range from scroll + size.
2. Apply `dirtyRect` clip if not full repaint.
3. **Pass 1 — backgrounds:** group cells by `fillStyle`, batch `fillRect` per color group.
4. **Pass 2 — text:** group cells by `(font, color)`, set context state once per group, batch `fillText`.
5. **Pass 3 — gridlines:** single `Path2D`, one `stroke()`.
6. **Pass 4 — headers:** repeat passes 1–3 for row/col/corner headers, with translation reset so headers stay frozen at 0,0.
7. **Pass 5 — selection + cursor:** translucent fill rect over selection, 2px stroke around cursor cell.

**Dirty regions:** Each grid tracks one `dirtyRect | null`. Single-cell ticks dirty just that cell's bounds; scroll dirties everything. Multiple cell-changed events in the same frame union into one rect. The 200×200 heatmap collapses to "full" each frame — no penalty vs naive — but Random Ticks 1 (one cell at a time) gets a massive savings.

**Scrollbars:** Two thin DOM scrollbar tracks (right + bottom edges). For row counts above the browser's max element height (~33M px on Chrome), the inner spacer is capped at the max and we map fractional scrollbar position → model row range. Wheel events bypass DOM and drive `scrollX/scrollY` directly.

**Mouse / keyboard:**
- Click selects a cell. Shift-click extends selection. Ctrl-click toggles cell in selection.
- Drag in body: range select. Drag on header edge: resize that row/col (single uniform size, applied to all). Drag on header body: select entire row/col.
- Wheel: scroll. Shift+wheel: horizontal.
- Arrows: move cursor. PgUp/PgDn: page. Home/End: line. Ctrl+Home/End: corner.
- Selection mode (`'cell' | 'row' | 'column'`) per grid, set at construction.
- Click on sort-enabled column header (Frozen+Sortable tab only): toggle asc → desc → off.

### 6. Dock panel

Three node kinds in a recursive tree:

- **`SplitNode`** — `orientation: 'h' | 'v'`, `children: DockNode[]`, `sizes: number[]` (sum to 1.0). Renders as flex row/column with 4px draggable splitter handles between children.
- **`TabNode`** — tab strip + content area. Holds `tabs: Tab[]` and `activeIndex`.
- **`Tab`** — leaf with `{ title: string, content: Grid }`.

**Drag-to-dock:** When a tab is dragged out of its strip, a translucent overlay appears over every `TabNode` showing 5 drop zones (top/right/bottom/left → split that node in that direction with the dragged tab; center → merge into the strip). Releasing on a zone restructures the tree. Empty `SplitNode`s with one remaining child collapse upward.

**Pointer events** (not mouse events): drag uses `setPointerCapture` so it survives the cursor leaving the window.

**Resize:** A single `ResizeObserver` on `#app` triggers a tree relayout. Each grid recomputes its canvas size and marks itself dirty if changed.

**Initial layout** (matches Phosphor's split structure; the 4 new tabs are added as additional tabs in existing slots so the initial dock geometry is unchanged):
```
SplitNode(h)
├── SplitNode(v)                          (~50%)
│   ├── TabNode { Two Trillion, 9 Quadrillion }   (top of left col)
│   └── TabNode { Random Ticks 1 }                (bottom of left col)
└── SplitNode(v)                          (~50%)
    ├── TabNode { Streaming Rows, 10K Stream @ 1ms }    (top of right col)
    ├── TabNode { Random Ticks 2, 200×200 Heatmap }     (mid of right col)
    └── TabNode { JSON Data, Frozen + Sortable }        (bottom of right col)
```
Every leaf is a `TabNode`; single-tab nodes still show their tab strip (so the tab is grabbable). This matches Phosphor's behavior where every panel is technically a tabbed container.

### 7. PerfHUD

Fixed top-right overlay. Toggle with `?` key or a small button. Two columns:

**Global** (rolling 60-frame avg):
- `FPS`, `Frame ms (p99)`, `Idle %`, `Heap` (Chromium only).

**Active grid** (the grid under the pointer):
- `Cells/frame`, `Cells/sec`, `Dirty rect`, `Updates/sec`.

Updates once per second to avoid skewing its own measurements. Total overhead: ~0.02ms/frame.

Hidden helper: `window.afterglow.bench(seconds = 5)` auto-scrolls every grid for N seconds and prints per-grid p50/p99/avg frame times to console.

## Data flow

```
Model.emit('cells-changed', {region, row, col, ...})
        │
        ▼
Grid.onModelChange(event)
        │  unions event bounds into this.dirtyRect
        ▼
Grid.markDirty()
        │
        ▼
[next requestAnimationFrame tick]
        │
        ▼
For each visible-tab grid in dock:
    if (grid.dirty) grid.paint()
```

User input (pointer/keyboard/wheel) writes to `scrollX/scrollY` or `selection`, then calls `grid.markDirty()` with the same flow.

## Testing

This is a single-file vibe-coded demo, not a production library, so the testing strategy is pragmatic:

- **Manual interactive smoke test** — open in Chrome/Firefox/Safari, click every tab, scroll each grid, drag tabs between zones, resize splitters, resize browser window. The HUD must hold ≥58 FPS during all of these.
- **`window.afterglow.bench(5)`** must report p99 frame time < 16.7ms across all nine grids.
- **Side-by-side with Phosphor** — open Phosphor's demo and Afterglow in two browser windows, scroll the Trillion / Two Trillion tabs simultaneously, compare smoothness on the same machine. Subjective, but the goal.
- **Constraint check** — `grep -E '(import |<script src|<link.+href)' index.html` returns empty.

## Implementation notes / risks

- **Trillion-row scrollbar mapping.** Above ~33M rows × 20px row height, the browser refuses to render a spacer that tall. We cap the spacer at `MAX_ELEMENT_HEIGHT` (~33,554,400 px on Chrome) and map fractional scrollbar position → row range. Phosphor handles this; we will too.
- **DPR-aware canvas.** Canvas backing-store size is `cssSize × devicePixelRatio`; CSS size is `cssSize`. Context is scaled by DPR once after each resize.
- **Font selection.** System font stack only. Tests on macOS / Windows / Linux will get different fonts; that's fine.
- **OffscreenCanvas / Workers.** Explicitly *not* used. Single thread, single canvas per grid. Was considered (Approach B) and rejected because Approach A meets the budget and OffscreenCanvas adds boilerplate that breaks the single-file constraint.
- **WebGL.** Explicitly *not* used. Text rendering quality and atlas management would consume the file budget and degrade visual fidelity vs Canvas2D.

## Acceptance criteria

A reviewer can clone, open `index.html` directly from the filesystem (no server), and:

- [ ] All nine tabs render correctly and match the visual style described.
- [ ] Drag any tab to any drop zone — layout restructures, no glitches.
- [ ] Resize splitters and the browser window — every grid resizes smoothly, no stale paint.
- [ ] Scroll Two Trillion grid to row 1,500,000,000,000 — values render correctly.
- [ ] HUD shows ≥58 FPS during simultaneous scroll of all visible grids.
- [ ] `window.afterglow.bench(5)` prints p99 < 16.7ms for every grid.
- [ ] Network tab shows one HTML request, nothing else.
- [ ] `grep -E '(\\bimport[ (]|<script[^>]+src|<link[^>]+href|@import)' index.html` returns empty.
