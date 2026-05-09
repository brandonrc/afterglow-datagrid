# Afterglow

*phosphor faded. afterglow remains.*

A single-file, zero-dependency datagrid + dock UI that beats the [PhosphorJS DataGrid example](https://phosphorjs.github.io/examples/datagrid/) by using 2026-class browser primitives (WebGL2 instanced rendering, OffscreenCanvas workers, tile-cached text, adaptive fidelity).

## View it

Save `index.html`, double-click. Works offline. No build, no install, no internet required after download.

## Test it

Open `tests.html` in a browser. Pass/fail counts render inline.

## Constraints (verify yourself)

- `grep -E '(\bimport[ (]|<script[^>]+src|<link[^>]+href|@import)' index.html` → empty
- `index.html` makes zero network requests after the document load
- No third-party libraries; only built-in browser APIs

## Repo

https://github.com/brandonrc/afterglow-datagrid
