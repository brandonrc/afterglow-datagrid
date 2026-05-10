# Afterglow

A single-file, zero-dependency datagrid + dock UI in the same spirit as the [PhosphorJS DataGrid example](https://phosphorjs.github.io/examples/datagrid/) — built to explore what the same kind of demo looks like with 2026 browser primitives instead of 2014 ones. Vibe-coded by an AI in one session. No third-party libraries. No imports. No build step. Save the HTML, double-click, ship.

## Live demo

**[brandonrc.github.io/afterglow-datagrid](https://brandonrc.github.io/afterglow-datagrid/)** (deploys via GitHub Actions on push to `main`)

## How it differs from Phosphor

Phosphor is genuinely sophisticated. It runs entirely on Canvas2D, tracks dirty regions, and bit-blits valid pixels on scroll — that's *why* it's fast on a single thread without a GPU. The architecture is great engineering for its era; calling it "old" understates the work.

Afterglow is the same demo with a different toolbox — the things browsers gained between 2014 and 2026:

1. **Three layered canvases per grid** — fill / text / overlay. A selection update writes to one canvas. A cell tick writes to one tile of one canvas. Phosphor handles the equivalent on a single Canvas2D with dirty-region tracking; we get it from layer separation. Different paths to the same goal.
2. **Fill layer is WebGL2 in an OffscreenCanvas Worker.** One drawcall draws all visible cells via instanced quads. The viridis colormap is a 256-texel GPU texture, sampled in the fragment shader. The worker thread holds the GL context, so input handlers and grid math on the main thread can't drop a paint frame. *Tradeoff:* GPU + 9 worker threads = more memory and more wattage than Phosphor's pure-CPU approach. Win on input responsiveness under heavy load. Loss on resource footprint.
3. **Text layer is tile-cached Canvas2D on main.** 256×256 px tiles, dirty-tile invalidation, LRU cap. Same idea Phosphor uses (paint only what changed) just split across orthogonal layers.
4. **Adaptive fidelity** — fast scroll suppresses the text layer entirely; text restores via `requestIdleCallback` when scroll velocity drops below threshold. Like iOS list views.
5. **Transferable `ArrayBuffer`** for value uploads to the worker (zero-copy ownership transfer; no `SharedArrayBuffer` so it works from `file://` without COOP/COEP headers).
6. **Priority-queued single rAF heartbeat** — input → focused grid → background grids → HUD. Dropped frames preferentially land on background tabs, never the focused one.

Every technique above is built-in browser API. No three.js, no twgl, no Comlink, no glmatrix.

This is *not* a "we beat Phosphor" claim. It's "here's what the same UI looks like if you redo it on 2026 primitives." Phosphor's CPU-bound design is the right call for resource-constrained environments. Afterglow's GPU/worker design wins on input isolation and stays smooth under heavy concurrent ticking.

## Tabs

| Tab | What it shows |
|---|---|
| **Two Trillion Rows/Cols** | 2×10¹² × 2×10¹² virtual grid (blue stripes) |
| **Streaming Rows** | 50 columns; rows insert/remove every 250ms (brown stripes) |
| **Random Ticks 1** | 15×10, one cell mutates every 60ms (red/green/black text) |
| **Random Ticks 2** | 80×80 viridis heatmap, one cell every 60ms |
| **JSON Data** | Cars dataset — 406 rows × 9 cols (UCI Auto MPG) |
| **9 Quadrillion** | `Number.MAX_SAFE_INTEGER` × `Number.MAX_SAFE_INTEGER` virtual |
| **200×200 Heatmap** | All 40,000 cells tick every 16ms |
| **10K Stream @ 1ms** | 20 cols × 10,000 rows, full row update every 1ms |
| **Frozen + Sortable** | Cars data with frozen 2 rows + 2 cols, click headers to sort |

The first five mirror Phosphor's tabs. The other four use the same engine to demo update rates and grid sizes that lean on the off-main-thread isolation.

## Interactions

- **Wheel** — scrolls. Shift+wheel = horizontal.
- **Click** in a cell — selects. Drag to extend the range. Selection mode (cell / row / column) is set per grid.
- **Double-click a column header** — auto-sizes that column to fit the widest visible cell.
- **Double-click the top-left corner** — auto-sizes every visible column.
- **Drag a column-header right edge** (cursor flips to col-resize) — manually resize that column.
- **Drag a row-header bottom edge** — manually resize row height (uniform).
- **Click a sort-enabled column header** (Frozen + Sortable tab) — toggles asc / desc / off.
- **Drag a tab** by its label — a translucent landing-shape preview fills the future home of the tab; release on the center of any pane to merge into its strip, or release on an edge to split. <kbd>esc</kbd> cancels.
- **Drag a splitter** between panes — resize.

## Run locally

```sh
git clone git@github.com:brandonrc/afterglow-datagrid.git
cd afterglow-datagrid
open index.html         # macOS — or just double-click
```

Or serve over HTTP:

```sh
python3 -m http.server 8000 --bind 127.0.0.1
open http://127.0.0.1:8000/index.html
```

## Test

Open `tests.html` in a browser. 57 tests run in an iframe against `index.html`'s exposed `afterglow.X` namespace; pass/fail counts render inline. Zero deps. No Node, no Vitest, no Jest.

## Bench

In the page console:

```js
afterglow.bench(5)
```

Auto-scrolls every grid for 5 seconds and prints per-grid `avg / p50 / p99` frame times.

## HUD

Top-right corner shows live FPS, frame ms (with p99), idle %, cells/sec, GPU draws/sec, and worker buffer transfers/sec. Toggle with `?`.

## Constraints (verify yourself)

```sh
grep -E '(\bimport[ (]|<script[^>]+src|<link[^>]+href|@import)' index.html
# Expected: empty (exit 1)
```

DevTools → Network panel → reload → only the one HTML document loads. Zero network calls afterwards.

```sh
wc -c index.html
# ~120KB unminified, single file
```

## Layout

The dock starts in this configuration:

```
┌─────────────────────┬──────────────────────────────┐
│ Two Trillion        │ Streaming · 10K Stream       │
│ · 9 Quadrillion     ├──────────────────────────────┤
├─────────────────────┤ Random Ticks 2 · 200×200     │
│ Random Ticks 1      ├──────────────────────────────┤
│                     │ JSON Data · Frozen+Sortable  │
└─────────────────────┴──────────────────────────────┘
```

## Build / deploy

`.github/workflows/pages.yml` deploys to GitHub Pages on every push to `main`. The workflow re-runs the import grep against the staged `index.html` and fails the build if anything snuck in.

## Credits & honest acknowledgments

- **Phosphor / Lumino** — the inspiration. Sphinx Andrew, Steven Silvester, Brian Granger, and the rest of the Phosphor contributors built a serious widget kit. The dirty-region painting + bit-blit-on-scroll architecture is solid engineering and is *why* their datagrid stays fast on Canvas2D without a GPU. This project's framing doesn't say "we beat them" — it says "here's the same demo retold on a newer toolbox."
- **UCI Auto MPG dataset** — Quinlan, J. Ross. (1993). The cars dataset, public domain.
- **Viridis colormap** — Stefan van der Walt, Nathaniel Smith, et al. Matplotlib's viridis LUT, public domain.

## License

MIT.
