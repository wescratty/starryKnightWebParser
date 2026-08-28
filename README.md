# StarryKnight Order Parser (Web)

A single-file, browser-only port of the [StarryKnightOrderParser](https://github.com/wescratty/StarryKnightOrderParser)
Python/tkinter desktop tool. Turns a Shopify `orders_export.csv` into the
same print-ready HTML production cut sheet, but as one self-contained
`index.html` file instead of a PyInstaller build -- no install, no build
step, works by just opening the file (or hosting it) in a browser.

## Status: Phase 1 (manual CSV upload)

This covers the manual-upload path only, ported line-for-line from the
Python source and verified against it (see "Parity testing" below):
CSV parsing, category/size/color extraction, addon (wool insert/big
runner/purse/headband/gift card) classification, and the full cut-sheet
report (Shoes table, Bottoms, Leather Order, Add-ons), plus the
click-to-cycle progress marker on each row and print/PDF output.

Not yet built: the Shopify Admin API "automagic" order pull (phase 2 --
needs a small serverless proxy to hold the API token; see the project
handoff doc in `C:\dev\Job_search\starryKnightHandoff.md` for why this
can't be pure client-side JS, and the open questions on hosting).

## Usage

Open `index.html` directly in a browser (double-click it, or drag it into
a browser window) -- no server needed. Choose (or drag-and-drop) a
Shopify `orders_export.csv`, review any warnings, then Print/Save as PDF
or download the rendered report as a standalone HTML file.

Settings (recognized colors, ignore words, and the last-processed-order
timestamp used to skip already-handled orders) persist in the browser's
localStorage, scoped to wherever this file is opened from.

## Parity with the Python original

The parsing/report logic in `index.html`'s inline `<script>` was
developed and tested as a standalone module (see the `webport-tests/`
notes below) against the real Python tool, run against its own pytest
fixtures and a full real order-export CSV. Output matched byte-for-byte,
with one documented exception: the Python original dedupes multi-color
variants with `list(set(...))`, whose order is not deterministic across
Python process runs (hash randomization) -- this port uses an
insertion-order-preserving dedup instead, which is strictly more
consistent, not a silent behavior change (there was no "correct" order
in the original to match).

## Files

- `index.html` -- the entire app (HTML + CSS + JS), single file, no
  external dependencies or build step.
