# Afterglow — Design Spec

*phosphor faded. afterglow remains.*

A single-file, zero-dependency clone of the [PhosphorJS DataGrid example](https://phosphorjs.github.io/examples/datagrid/) that **decisively beats it on perf** through 2026-class browser primitives Phosphor's 2014 architecture structurally cannot adopt without a rewrite. Ships in one HTML file. Written from scratch — no Phosphor/Lumino source code is read or copied.

## Goals

1. **Feature parity** with all five Phosphor demo tabs (Two Trillion Rows/Cols, Streaming Rows, Random Ticks 1, Random Ticks 2, JSON Data).
2. **Decisively beat Phosphor on perf** — measurably smoother scroll and tick at 60fps (and 120fps where the display supports it), proven by an in-page FPS HUD and a `bench()` helper.
3. **Add four showcase tabs** that demonstrate capabilities Phosphor's example doesn't:
   - 9 Quadrillion Rows/Cols (virtual ceiling at `Number.MAX_SAFE_INTEGER`)
   - 200×200 Heatmap with full-grid ticks at 60Hz
   - 10K-row firehose stream with 1ms updates
   - Frozen rows/cols + sortable headers
4. **Zero external dependencies** — single `index.html`, no imports, no CDN, no fonts, no fetch, no build step for the consumer.
5. **Modern evergreen browsers only** (Chrome/Firefox/Safari, latest 2 versions).

## Non-Goals

- Mobile/touch is supported via Pointer Events but not specifically polished for phones.
- No accessibility tree work beyond keyboard navigation.
- No internationalization, RTL, or font fallback configuration.
- No persistence of dock layout across reloads.
- No theming beyond the five style presets needed for parity.

## Hard Constraints

- **No imports of any kind.** No `import ... from ...`, no `import(...)`, no `<script src=...>`, no `<link rel="stylesheet" href=...>`, no `@import` in CSS, no external `new Worker(url)`. Verifiable by `grep -E '(\\bimport[ (]|<script[^>]+src|<link[^>]+href|@import)' index.html` returning empty.
- **No external network requests at runtime.** DevTools Network tab shows only the single HTML document load.
- **No copy-paste from Phosphor / Lumino source.** Behavior is observed in a browser and reimplemented from primitives. The Cars dataset (public-domain reference data, not Phosphor's code) is the one allowed inline asset.
- **No third-party libraries of any kind.** WebGL is used directly via `getContext('webgl2')` with hand-written GLSL strings — no three.js, twgl, regl, glmatrix, etc. Workers are constructed inline via Blob-URLed strings — no Comlink, no worker libraries. Tile cache, scheduler, vector math: all hand-written.
- **One file** — `index.html` containing one `<style>` block and one inline `<script type="module">`. Worker code lives as a string literal inside the same file and is materialized via `URL.createObjectURL(new Blob([workerSource], {type:'text/javascript'}))`.

## Architecture

```
┌──────────────────────────────────── index.html ──────────────────────────────┐
│  <style>  (~250 lines: reset, dock chrome, splitters, scrollbars, HUD,       │
│            brand surface, drop-zone overlay)                                 │
│  <script type="module">                                                      │
│     §1  Constants, viridis LUT, brand tokens, cars dataset                   │
│     §2  EventChannel + tiny priority queue (RenderScheduler)                 │
│     §3  DataModel base + 6 concrete models                                   │
│     §4  Style presets, cell renderers (CPU side)                             │
│     §5  GLSL shader strings (vertex + fragment)                              │
│     §6  WORKER_SOURCE — string literal containing the renderer worker        │
│     §7  Grid: layered renderer (3 canvases, worker-backed WebGL2 +           │
│              tile-cached Canvas2D text + Canvas2D overlay)                   │
│     §8  DockNode tree (SplitNode, TabNode, drop-zone overlay)                │
│     §9  PerfHUD                                                              │
│     §10 main()                                                               │
│  </script>                                                                   │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Boot sequence

`main()` runs once on load and:

1. Materializes the worker once (`URL.createObjectURL(new Blob([WORKER_SOURCE]))`).
2. Constructs nine `DataModel` instances.
3. Constructs nine `Grid` instances bound to those models. Each `Grid`:
   - Allocates one `OffscreenCanvas` for the WebGL fill layer, transfers it to a per-grid `Worker` instance (each grid owns its worker thread).
   - Creates two main-thread `<canvas>` elements: one for tile-cached text, one for overlay (gridlines, cursor, frozen-pane shadows, selection stroke).
   - Allocates a Transferable `ArrayBuffer` for cell-value uploads to the worker (zero-copy).
4. Wraps each grid in a `Tab` and inserts them into the recursive `DockNode` tree.
5. Mounts the root `DockNode` to `#app`.
6. Starts the master `RenderScheduler` (single rAF loop with priority queue).

### The performance thesis

This is the section that explains why Afterglow wins. Five compounding techniques:

#### T1 — Layered rendering: separate concerns, separate canvases

Per grid, **three stacked canvases** with distinct redraw triggers:

| Layer | Tech | Owns | Redraws when |
|---|---|---|---|
| **Fill** (back) | WebGL2 in worker, via `OffscreenCanvas` | Cell backgrounds, stripes, heatmap fills, selection rectangles | Cell value or selection changes |
| **Text** (mid) | Canvas2D, main thread, tile-cached | All text glyphs in body + headers | Cell text changes (rare for heatmaps; common for streaming) |
| **Overlay** (front) | Canvas2D, main thread | Gridlines, row/col cursor, frozen-pane shadows, sort glyphs | Cursor or scroll changes |

Phosphor (and most grid libraries) re-render every pixel through one Canvas2D context whenever anything changes. Afterglow's selection box update is a 5-byte uniform write to a shader; Phosphor's selection box update is a full grid repaint.

#### T2 — WebGL2 instanced quads for fills

The fill layer renders all visible cells as one instanced draw call: a single unit quad, instanced N times where N = visible cell count. Per-instance attributes are `(col, row, valueIndex)`. The fragment shader samples the **viridis LUT as a 256×1 texture** to produce the heatmap color — zero CPU per cell, GPU-parallel for all cells.

For a 200×200 fully-ticking heatmap (~40,000 cells), this is **one** drawcall instead of 40,000 `fillRect` calls. Theoretical perf delta: ~50–100×.

#### T3 — Tile-cached text rendering

The text layer is broken into 256×256 px tiles, each owning its own `OffscreenCanvas`. When a cell's text changes, only the tile(s) it intersects are dirtied. Clean tiles are blitted to the visible canvas via `drawImage`. Scroll-and-no-data-change → zero text re-rendering, just blit translation.

For Random Ticks 1 (one cell ticks every 60ms), exactly 1 tile out of ~6 visible tiles re-renders — Phosphor re-renders all 6.

#### T4 — Off-main-thread WebGL via OffscreenCanvas + Worker

Each grid's WebGL canvas is owned by a `Worker`. The main thread is free for input, dock UX, scroll handling, and tile-cache compositing — it cannot be blocked by GPU command buffer flushes or shader compilations. Cell data flows main→worker via **Transferable `ArrayBuffer`** (zero-copy ownership transfer; no `SharedArrayBuffer`, so it works from `file://` without COOP/COEP headers).

Phosphor's example runs entirely on the main thread; a slow input handler can drop frames. Afterglow cannot drop a render frame because of an input handler.

#### T5 — Adaptive fidelity during scroll

The scroll handler tracks pointer/wheel velocity. When velocity exceeds a threshold, the text layer is **suppressed**: only fills + gridlines render. Once velocity drops below the threshold for one frame, text re-render is scheduled via `requestIdleCallback` (with a `setTimeout(50)` fallback in browsers without it). User sees buttery-smooth scroll; text fills in within 50ms of stopping. This is how iOS list views work; Phosphor does not.

#### T6 — Priority-queued frame scheduler (`RenderScheduler`)

A 40-line scheduler holds an array of `{priority, fn}`. The single rAF callback drains highest-priority items first within an 8ms budget; remaining work defers to the next frame. Priorities: input response (10) > focused-tab fill (8) > focused-tab text (6) > background-tab updates (2) > HUD (1). Dropped frames preferentially happen on background tabs and the HUD, never on the focused grid.

This replaces our earlier "single rAF heartbeat" idea with something quantitatively smarter, while staying entirely in user code (no `Scheduler.postTask` dependency, since that's Chromium-only).

## Components

### 2. EventChannel + RenderScheduler

`EventChannel` is a 30-line pub/sub: `on(type, fn)`, `off(type, fn)`, `emit(type, payload)`.

`RenderScheduler` is a 40-line priority queue + single global rAF loop:
```js
class RenderScheduler {
  schedule(priority, fn) { ... }   // dedup by fn identity
  cancel(fn) { ... }
  // internal: rAF callback drains queue in priority order with 8ms budget
}
```

### 3. DataModel

Same shape as the prior spec:
```js
class DataModel {
  rowCount(region) { ... }
  columnCount(region) { ... }
  data(region, row, col) { ... }
  emit(event) { ... }
}
```

Concrete models (unchanged from prior spec):

| Model | Constructor | Backs |
|---|---|---|
| `LargeModel(rows, cols)` | virtual; `data()` returns `(r, c)` string | Two Trillion + 9 Quadrillion |
| `StreamingModel(cols, maxRows=500, intervalMs=250)` | rows splice in/out at random index | Streaming Rows |
| `RandomTicksModel(rows, cols, intervalMs)` | one or all random cells mutate per tick | Random Ticks 1, 2, 200×200 |
| `FirehoseModel(cols, rows, intervalMs)` | row of values updated per tick at 1ms | 10K Stream |
| `JSONModel(rows[], schema)` | static, indexed | JSON Data |
| `SortableModel(jsonModel, frozenRows, frozenCols)` | wraps JSONModel with index permutation + freeze | Frozen+Sortable |

**Numeric models also expose a typed-array view** (`getValueBuffer(): Float32Array`) so the grid can transfer values to the worker without per-cell function calls. Models that only have text (`LargeModel`, `JSONModel`'s string columns) skip this — text goes through the tile-cached Canvas2D path on the main thread.

### 4. Styles + cell renderers

Five style presets unchanged. Cell renderers split into two families now:

**CPU renderers** — used by the text layer (Canvas2D):
- `textRenderer` — default sans-serif, left-aligned.
- `redGreenBlackRenderer` — value buckets to black/red/green text. Right-aligned, 2-digit fixed.
- `viridisInverseTextRenderer` — text color is inverse-viridis(value), used over heatmap fills so text stays legible.

**GPU renderers** — used by the fill layer (WebGL2 fragment shader). Selected per-grid via a uniform:
- `flatFillShader` — uniform color (used for default-style backgrounds).
- `stripeFillShader` — alternating row/col tints, two uniform colors.
- `viridisHeatmapShader` — samples viridis LUT texture by per-instance `value` attribute.

The viridis LUT is a 256-element RGB array uploaded once as a 256×1 RGBA8 texture. ~3KB of data baked into a JS array literal in the file.

### 5. GLSL shader strings (~150 lines)

Two shader programs, both inlined as JS template strings:

**Program A: stripe/flat shader** (vertex + fragment, ~50 lines)
- Vertex: instanced quad, position from `(col, row)` instance attribute.
- Fragment: `gl_FragColor = mix(stripeColorA, stripeColorB, mod(row+col, 2.0))` plus selection overlay.

**Program B: heatmap shader** (vertex + fragment, ~80 lines)
- Vertex: same as A.
- Fragment: `gl_FragColor = texture(viridisLUT, vec2(value, 0.5))` plus selection overlay.

Both share a small uniform block: `view`, `cellSize`, `gridDims`, `selectionBounds`. Compiled once on worker boot.

### 6. WORKER_SOURCE (~300 lines, inlined)

A self-contained string containing the renderer worker. Receives messages:
- `init({offscreenCanvas, shaderId, vridisLUT})` — sets up GL context.
- `resize({width, height, dpr})`
- `setStyle({...})` — uniform updates.
- `uploadValues({buffer, rowOffset, rowCount, colCount})` — `buffer` is a `Float32Array` transferred into the worker. Worker uploads as a buffer attribute; previous buffer is `transfer`-returned to main if main wants it back.
- `setView({scrollX, scrollY, viewWidth, viewHeight})`
- `paint()` — issues one drawcall, calls `requestAnimationFrame` inside the worker (workers have rAF on `OffscreenCanvas`).

The worker never touches text, never touches DOM, never touches user models. It only renders fills based on what it's told.

### 7. Grid (layered renderer)

A `Grid` orchestrates three layers, the worker, the tile cache, the model, and input.

**Geometry** (uniform sizing — non-uniform out of scope):
- `rowHeight`, `columnWidth`, `rowHeaderWidth`, `columnHeaderHeight` (defaults 20/64/64/20).
- `scrollX`, `scrollY` in CSS pixels.
- `firstVisibleRow`, `lastVisibleRow`, `firstVisibleCol`, `lastVisibleCol` are O(1).

**Three layers** (z-stacked DOM canvases at `position:absolute` inside the grid container):
- `fillCanvas` — owned by worker via `OffscreenCanvas`.
- `textCanvas` — main-thread Canvas2D. Backed by tile cache.
- `overlayCanvas` — main-thread Canvas2D. Gridlines + cursor + scrollbars.

**Tile cache** for text:
- 256×256 px tile size. Each tile is a lazily-allocated `OffscreenCanvas`.
- `tileMap: Map<"col,row", {canvas, dirty}>`.
- On cell change: mark intersecting tiles dirty.
- On paint: render dirty tiles, then composite all visible tiles to `textCanvas`.
- On scroll without data change: skip dirty-tile pass, just re-composite (translate blits).
- LRU eviction when tile count > 64 (cap memory at ~16MB).

**Paint flow per dirty frame:**
1. RenderScheduler triggers the grid at appropriate priority.
2. **If only scroll changed:** translate-blit all three layers (overlayCanvas full-redraw; tile cache composites at new offset; fillCanvas sent `setView` message — worker re-paints, which is essentially free).
3. **If selection changed:** `setStyle` on worker (uniform update; one paint), redraw overlayCanvas. Text layer untouched.
4. **If cells changed:** dirty-tile pass on text, transfer-upload value buffer to worker, worker paints fill. Overlay redraws cursor.
5. **If scroll velocity > threshold:** skip step 4's text dirty-tile pass. Schedule a `requestIdleCallback(textRefreshFn)` for when scroll stops.

**Scrollbars:** Two thin DOM scrollbar tracks. Above ~33M-px content height, fractional virtual scrolling kicks in. Wheel events bypass DOM, drive `scrollX/scrollY` directly through input priority.

**Mouse / keyboard:** Same as prior spec — click selects, drag range-selects, header-edge resizes, arrow keys move cursor, etc. Input handlers run at priority 10 in the scheduler so they always preempt rendering.

### 8. Dock panel

Three node kinds: `SplitNode`, `TabNode`, `Tab`. Identical to the prior spec — drag-to-dock with 5 drop zones, splitter dragging, single `ResizeObserver` triggers tree relayout.

**Resize integration with workers:** When a grid resizes, the main thread sends `resize({width, height, dpr})` to the worker. The worker's `OffscreenCanvas` is resized synchronously inside the worker and the next paint draws at the new size. No flicker because the main-thread layers (text, overlay) resize in the same frame.

### 8b. Brand & polish surface

(Unchanged from prior spec — wordmark, tab strip, drop-zone overlay, color tokens.)

### 9. PerfHUD

Fixed top-right overlay. Toggle with `?` key. **Three** columns now (added Workers):

**Global** (rolling 60-frame avg): `FPS`, `Frame ms (p99)`, `Idle %`, `Heap` (Chromium only).

**Active grid**: `Cells/frame`, `Cells/sec`, `Dirty tiles`, `Updates/sec`.

**Workers**: `Worker rAF Hz`, `GPU draw count`, `Buffer transfers/sec`.

Updates once per second. Helper `window.afterglow.bench(seconds=5)` auto-scrolls every grid for N seconds and prints per-grid p50/p99/avg frame times to console.

## Data flow

```
Model.emit('cells-changed', range)
        │
        ├──→ Grid.markCellsDirty(range)
        │       ├─ tileCache.markDirty(range)
        │       └─ valueBuffer.copyFromModel(range)
        │
        ▼
RenderScheduler.schedule(priority=8, grid.repaint)
        │
        ▼
[next rAF]
        │
        ├─ scrollVelocityHigh? → skipTextLayer = true
        │       └─ scheduler.scheduleIdle(grid.refreshText)
        │
        ├─ tileCache.composite(textCanvas, scrollOffset)        (if !skipText)
        ├─ worker.postMessage({op:'uploadValues', buffer}, [buffer])  (zero-copy)
        ├─ worker.postMessage({op:'paint'})
        │       worker → GPU: 1 drawcall, 40k instanced quads
        ├─ overlayCanvas.redrawCursorAndGridlines()
        ▼
3 layers composited by browser, vsynced
```

## Testing

- **Manual interactive smoke test** — open in Chrome/Firefox/Safari, click every tab, scroll each grid, drag tabs between zones, resize splitters, resize browser window. HUD must hold ≥58 FPS.
- **`window.afterglow.bench(5)`** must report p99 frame time < 16.7ms across all nine grids. Stretch goal on a 120Hz display: p99 < 8.3ms.
- **Side-by-side with Phosphor** — Phosphor demo and Afterglow in two windows. Scroll Two Trillion in both simultaneously. Compare HUD readings.
- **Worker isolation check** — DevTools → Performance → Main thread is mostly idle while heatmap ticks; Worker thread shows the GPU work. (This is the visible proof that T4 actually does anything.)
- **Constraint check** — `grep -E '(\\bimport[ (]|<script[^>]+src|<link[^>]+href|@import)' index.html` returns empty.

## Implementation notes / risks

- **OffscreenCanvas transfer is one-shot.** Once `transferControlToOffscreen()` is called, the main thread cannot draw to that canvas. Confirmed; we only need worker to draw it.
- **Worker count = 9 (one per grid).** Modern OSes are fine with 9 worker threads. The alternative — single shared worker rendering all grids — has worse priority isolation. We pick per-grid workers; if memory becomes a problem we can pool workers later (out of scope).
- **WebGL context loss handling.** Worker re-creates the GL context on `webglcontextlost` and re-uploads textures + shaders. ~30 lines.
- **`requestIdleCallback` Safari support.** Safari 18+ has it; older Safari (< 16.4) doesn't ship it. We `setTimeout(50)` fallback. (Modern evergreen target makes this a non-issue in practice.)
- **`OffscreenCanvas` from worker requires Safari 16.4+.** All evergreen browsers as of 2026 support it.
- **Trillion-row scrollbar mapping.** Cap inner spacer at `MAX_ELEMENT_HEIGHT` (~33M px Chrome), map fractionally to model row range.
- **Worker construction must be synchronous-enough.** We materialize the worker once at boot, before any user input. Worker startup ~10ms, hidden behind initial paint.
- **DPR-aware canvas across worker boundary.** Main thread reads `window.devicePixelRatio` and includes it in every `resize` message. Worker scales accordingly. Re-sent on `matchMedia('(resolution)').change`.

## Acceptance criteria

A reviewer can clone, open `index.html` directly from the filesystem (no server), and:

- [ ] All nine tabs render correctly and match the visual style described.
- [ ] Drag any tab to any drop zone — layout restructures, no glitches.
- [ ] Resize splitters and the browser window — every grid resizes smoothly, no stale paint.
- [ ] Scroll Two Trillion grid to row 1,500,000,000,000 — values render correctly.
- [ ] HUD shows ≥58 FPS during simultaneous scroll of all visible grids on a 60Hz display.
- [ ] HUD shows ≥110 FPS on a 120Hz display when only one heavy grid (200×200 heatmap) is focused.
- [ ] `window.afterglow.bench(5)` prints p99 < 16.7ms for every grid.
- [ ] DevTools Performance shows worker threads doing GPU work while main thread is mostly idle during heatmap tick.
- [ ] Network tab shows one HTML request, nothing else.
- [ ] `grep -E '(\\bimport[ (]|<script[^>]+src|<link[^>]+href|@import)' index.html` returns empty.
- [ ] Side-by-side with Phosphor's Trillion Rows tab, Afterglow's Two Trillion tab scrolls visibly smoother on the same machine.
