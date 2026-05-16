# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"MyStuff — Home Tracker" (Inventory 5.4): a mobile-first single-page PWA for tracking household items, where they live, who borrowed them, and what's broken/empty. The entire app is **one file**: `index.html` (~1273 lines) with embedded `<style>` and `<script>` blocks.

## Build / run / test

There is no build system, package manager, lint config, or test suite. To work on the app:

- **Run locally:** serve the directory over HTTP so the service-worker / camera APIs behave (e.g. `python3 -m http.server 8000`, then open `http://localhost:8000`). Opening `index.html` via `file://` will break the manifest, camera permission, and QR scanner.
- **Scanner / camera testing** requires HTTPS or `localhost` and a real device or browser with `BarcodeDetector` support (Chrome/Edge on Android; iOS Safari ≥ 17). On unsupported browsers the "Allow Camera" path will show but scanning silently no-ops.
- **PWA manifest references** (`/manifest.json`, `/icons/icon-192.png`, `/sw.js`) are linked in `<head>` but the files **do not exist in the repo**. Don't be surprised by 404s in the console; only add those files if the task is specifically to ship PWA install/offline support.

## Architecture

### Single-file layout

`index.html` is structured as four contiguous sections:

| Lines     | Section                                                  |
| --------- | -------------------------------------------------------- |
| 1–16      | `<head>`, Google Fonts + `qrcodejs` CDN, manifest links  |
| 17–255    | `<style>` — all CSS, dark theme tokens in `:root`        |
| 256–482   | `<body>` — five `.screen` blocks + modal `.overlay`s     |
| 484–1271  | `<script>` — all app logic, IIFE-free, runs top-to-bottom |

The script ends with `renderStuff()` — that's the initial paint. There is no router, no framework, no bundler.

### Five screens, one nav

`<nav>` buttons with `data-screen="..."` switch the active `.screen` via `showScreen(name)` (line ~823). That function also calls the per-screen render function (`renderStuff`, `initScanner`, `renderMissing`, `renderLocations`, `resetForm`) and stops the camera whenever you leave the Scan screen. Adding a screen means adding the markup, a render function, and a case in `showScreen`.

### State model

Two localStorage keys, both arrays of plain objects:

- `mystuff_v3` — items. Helpers: `load()` / `save(items)`.
- `mystuff_locations` — standalone location QR codes (room-zone-slot triples). Helpers: `loadLocs()` / `saveLocs()`.

Item shape (built in the save handler around line ~1050):
```
{ id, name, room, location, desc, serial, notes, photo,
  status: 'home' | 'out' | 'broken' | 'empty',
  borrower, purpose, whereAt, checkedOutAt, addedAt,
  log: [{ type, label, detail?, at }] }
```

`photo` is a base64 data URL from `FileReader.readAsDataURL` — this is why localStorage can fill up fast. There is no quota handling.

Every state change follows the pattern: `const items = load(); ...mutate...; save(items); rerender()`. Don't introduce a separate in-memory cache — re-reading from localStorage is the source of truth.

### Two parallel location systems (don't conflate them)

1. **Per-item location** — set on the Add/Edit form via the hierarchical picker. Driven by the `LOC_TREE` constant (line ~492) and the `locState` object (~597). Picker state machine lives in `renderLocPicker()` / `renderShelfPicker()`. The picked path is flattened to a human string like `Carriage House → Garage → Red Double — Top (12 drawers) → Drawer 7` by `locFullPath()` and stored on the item as `location` (plus `room` for filtering).

2. **Standalone location QR codes** — built on the Places screen (`renderLocBuilder`, line ~1121). Produces compact codes like `BE-A-3` (room-prefix · zone · slot) saved in `mystuff_locations`. These are independent records meant to be printed and stuck on shelves.

A scanned location QR shows "items that belong here" by matching `item.room === loc.room` (line ~1209) — i.e. by room only, not by the full per-item path. That's intentional but easy to misread.

### Room definitions (`LOC_TREE`)

Each room has a `type` that drives picker behavior:

- `shelf` — zones A–J × slots 1–20 (with custom-label escape hatches).
- `custom` — explicit `sections` with named slots and optional `subSlots` (Dressing Room, Baby Room).
- `subroom` — nested rooms (Carriage House → Garage / Room).
- `garage` — branches into Toolbox (named toolboxes, each with a `drawers` count) or Shelves.

When editing room definitions, also check `ROOM_ICONS` (~562) for the matching emoji and the Places-builder prefix (`r.substring(0,2).toUpperCase()` at ~1106) — collisions there will produce duplicate location codes.

### QR codes

- **Item QR payload:** `JSON.stringify({type:'item', id, name})` — see `qrItemVal`.
- **Location QR payload:** `JSON.stringify({type:'location', code, name})` — see `qrLocVal`.
- `handleScanned(val)` (~1193) tries `JSON.parse`, falls back to fuzzy name match on items. If you change a payload shape, update both producer and parser.
- Rendering uses the CDN `QRCode` constructor (`qrcodejs`). Download/print routines re-render the canvas into a new canvas with a label strip; they rely on a `setTimeout(...,100)` race because qrcodejs paints asynchronously — keep that delay.

### Rendering pattern + XSS

Every list/detail view rebuilds its container with `innerHTML = ...template...` and re-attaches handlers with `querySelectorAll(...).forEach(b => b.addEventListener(...))`. Two consequences:

1. **All user-supplied strings must go through `esc()`** (~592) before being interpolated into templates. Item names, borrower names, locations, notes, log details, custom zone labels — everything. Skipping `esc()` is an XSS bug because photos are data URLs and notes accept free text.
2. Don't try to "diff" or hold references to DOM nodes across rerenders — they're thrown away.

### Modals

Overlays (`.overlay` + `.sheet`) are opened by adding `.open` and closed by removing it. The detail/report/scan-result/location-QR/location-scan modals share the same dismiss-on-backdrop-click idiom (`if (e.target === overlay) close()`). Several modals re-open the detail modal on close (e.g. report → detail) via a `setTimeout(...,200)` — that delay matches the slide-down animation; don't shorten it.

### Scanner lifecycle

`initScanner()` is called on entry to the Scan screen and `stopScanner()` on exit (both from `showScreen`). The loop is a 300 ms `setInterval` calling `barcodeDetector.detect(video)`; on a hit it sets `scanActive=false`, vibrates, and opens a result modal. Every modal that follows a scan resets `scanActive=true` on close — if you add a new scan-result flow, do the same or scanning will silently stop after the first hit.

## Conventions worth following

- **No dependencies, no build step.** Don't add npm, bundlers, or frameworks unless the task explicitly calls for it. New external libs should be loaded from a CDN in `<head>` like `qrcodejs`.
- **One file.** Keep changes inside `index.html` unless the task is to break it up. The script section uses banner comments (`// ═══...`) as section dividers — match that style.
- **Vanilla JS only.** No JSX, TS, ES modules. The script tag is plain `<script>` — top-level `await`, imports, etc. won't work.
- **CSS tokens live in `:root`** (~18). Reuse `var(--green|red|amber|blue|purple|dark|card|panel|border|text|muted)` rather than hard-coding colors. The two font families are `--font-head` (Syne, display) and `--font-mono` (DM Mono, body).
- **IDs over classes for hooks.** Almost every interactive element has a unique `id` that the script reaches via `getElementById`. When adding controls, give them ids and wire them up in the same section as their parent renderer.
- **Mobile viewport assumptions.** Layout uses `100dvh`, `env(safe-area-inset-*)`, and a fixed bottom nav. Don't introduce hover-only affordances or layouts that assume desktop width.
