# Afterglow

*phosphor faded. afterglow remains.*

A single-file, zero-dependency datagrid + dock UI built to **decisively beat** the [PhosphorJS DataGrid example](https://phosphorjs.github.io/examples/datagrid/). Vibe-coded by an AI in one session. No third-party libraries. No imports. No build step. Save the HTML, double-click, ship.

## Live demo

**[brandonrc.github.io/afterglow-datagrid](https://brandonrc.github.io/afterglow-datagrid/)** (deploys via GitHub Actions on push to `main`)

## How it goes faster than Phosphor

Phosphor (2014, IE11-era architecture) does single-thread Canvas2D, full grid repaint per change, single rAF event arrival.

Afterglow (2026 architecture) layers three rendering pipelines:

1. **Fill layer — WebGL2 in OffscreenCanvas Worker.** Per-grid worker thread holds the GPU context. Cell backgrounds and viridis heatmaps render via instanced quads — *one drawcall* for the entire visible grid, not 40,000 `fillRect`s. The viridis colormap is a 256-texel texture sampled in the fragment shader.
2. **Text layer — Tile-cached Canvas2D on main.** Body text is broken into 256×256 px tiles. A single ticking cell only invalidates the one tile it touches. Phosphor would re-render the whole visible window.
3. **Overlay layer — Canvas2D on main.** Gridlines, cursor, selection. Selection changes never repaint the text layer.

Plus:

- **Adaptive fidelity** — fast scroll suppresses the text layer entirely (fills + lines only); text restores via `requestIdleCallback` when scroll velocity drops. Like iOS list views.
- **Priority-queued single rAF heartbeat** — input → focused grid → background grids → HUD. Dropped frames preferentially happen on background tabs, never the focused one.
- **Transferable `ArrayBuffer`** for value uploads to the worker (zero-copy ownership transfer, no `SharedArrayBuffer` so it works from `file://`).

Every technique above is built-in browser API. No three.js, no twgl, no Comlink, no glmatrix.

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

The first 5 are direct counterparts to Phosphor's tabs (with "Trillion" → "Two Trillion" because virtual rows are free). The other 4 demonstrate things Phosphor's example structurally cannot do as smoothly.

## Run locally

```sh
git clone git@github.com:brandonrc/afterglow-datagrid.git
cd afterglow-datagrid
open index.html         # macOS — or just double-click
```

Or serve over HTTP if you want DevTools Network panel to behave normally:

```sh
python3 -m http.server 8000 --bind 127.0.0.1
open http://127.0.0.1:8000/index.html
```

## Test

Open `tests.html` in a browser. 57 tests run in an iframe against `index.html`'s exposed `afterglow.X` namespace; pass/fail counts render inline. **Zero deps.** No Node, no Vitest, no Jest.

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
# File size
wc -c index.html
# ~190KB unminified, single file
```

## Layout

The dock starts in this configuration. Drag any tab anywhere — top/right/bottom/left of any panel splits, center merges. <kbd>esc</kbd> cancels a drag.

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

`.github/workflows/pages.yml` deploys to GitHub Pages on every push to `main`. The workflow includes a sanity check that re-runs the import grep against the deployed `index.html` and fails the build if anything snuck in.

## License

MIT.
