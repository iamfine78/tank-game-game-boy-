# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A Game Boy–styled Battle City / Tank Battle clone in a **single self-contained file**: `tank-battle.html`. No build step, no dependencies, no package manager, no tests. Comments and all UI strings are in Chinese — match that convention when editing.

## Running

Open `tank-battle.html` directly in a browser (`start tank-battle.html`, or drag it in). There is no dev server and nothing to compile. Editing the file and refreshing the page is the entire workflow.

Debugging is done through the browser console; there is no logging infrastructure and no test harness to run.

## Architecture

Everything lives in one IIFE inside `<script>`. Layout of the file (section banners are marked with `/* ===== 注释 ===== */`):

1. Constants → themes → pixel font → map data → canvases → state → audio → sprites → terrain → text → tank/map/level logic → menus/editor → update → render → loop → input → boot.

### Coordinate system and canvases

- World is `TILE = 16`, grid is `COLS = ROWS = 13`, so the play field is `FIELD = 208`.
- Virtual screen is `VW = FIELD + HUD_W (56) = 264` by `VH = FIELD = 208`; the canvas is upscaled by CSS with `image-rendering: pixelated`. **Draw in these virtual pixels, never in CSS pixels.**
- Terrain is split across three offscreen layers to get the correct z-order:
  - `groundCv` — brick, steel, base/ruin (below tanks)
  - `grassCv` — grass (drawn *over* tanks, hiding them — this is intentional)
  - `waterFrames[2]` — two pre-rendered water tiles alternated by `(tick >> 5) & 1` for flow
- `rebuildLayers()` is the single source of truth for terrain redraw. Call it after any `map` mutation (see `destroyTile`, `fortifyBase`, `editorPaint`, `toggleTheme`).

### Two themes, one palette contract

`THEMES.gb` (monochrome green) and `THEMES.color`, selected by the module-level `themeName` and read everywhere through `T()`. Since sprites are cached per theme (`spriteCache` key is `themeName|kind|level|hurt`), **any theme change must `spriteCache.clear()` + `rebuildLayers()`** — see `toggleTheme()`.

The `gb` palette deliberately encodes depth as brightness (empty = brightest → brick → steel/tank/grass = darkest) so tanks stay legible against brick. Preserve that ordering when adding tiles; do not pick arbitrary greens.

### State machine

`state` is one of `title | menu | cheat | playing | paused | clear | over | edit`. `update()` dispatches on it; the editor has a second level (`editor.menu`) for its own dialog. `render()` branches early for `edit` and otherwise composes terrain → entities → bullets → freeze tint → grass → booms → HUD → overlay.

### Game loop

Fixed timestep, 60 Hz: `loop()` accumulates `requestAnimationFrame` deltas into `STEP = 1000/60` and runs up to 5 catch-up `update()` calls, then renders once. Wall-clock timers (`freezeTimer`, `shovelTimer`, `spawnCd`, `respawnTimer`, `clearTimer`, `killTimer`) are counted in **frames**, not milliseconds.

### Input model — `keys` vs `press`

- `keys` — held state, drives movement and continuous fire.
- `press` — "pressed this frame" edge flags, consumed by menus and single-shot actions, then wiped by `clearPress()` at the end of every `update()`.

When adding an action, choose deliberately: a menu needs `press`, movement needs `keys`. Both keyboard (`KEYMAP`, by `e.code`) and the on-screen D-pad feed the same objects, so touch and keyboard stay in sync. Holding a direction in the editor repeats with a slow→fast ramp (`editRepeat`/`editLastDir`).

### Data formats

- **Maps** are `13` strings of `13` chars each. `CHAR_TILE` / `TILE_CHAR` convert between chars (`. # @ ~ T E`) and the numeric `EMPTY/BRICK/STEEL/WATER/GRASS/BASE` tile constants. Levels live in `MAPS`; the same format is used for the editor's saved custom stage.
- **Enemy mix** per difficulty is a `{basic, fast, power, armor}` weight table in `MIXES`; `buildQueue(stage, count, mixIdx)` expands it into a per-stage `spawnQueue`, with `bonus` flags injected at 25% / 55% / 85% of the stage for powerup drops.
- **Font** is a hand-written 3×5 bitmap in `FONT` — uppercase alphanumerics plus `- . : / ! > < + =`. Missing glyphs silently skip; there are no lowercase letters.

### Editor invariants

`ensureValid()` enforces the rules on every snapshot: the four `RESERVED` cells (3 enemy spawn points + player start) are forced `EMPTY`, and exactly one `BASE` exists. Individual paints are rejected for reserved cells and for overwriting a base with a non-base brush (`USE BASE`). Any change to editor painting must go through `editorPaint()` so these checks stay in one place.

### Persistence

`localStorage` keys: `tankbattle.hi` (high score) and `tankbattle.custom` (editor stage JSON). `loadCustom()` validates shape on read and returns `null` on anything malformed — keep that defensive parsing.

### Audio

All sound is synthesized (`beep`/`hiss` over WebAudio, composed into the `sfx` object); there are no audio assets. `ac()` lazily creates and resumes the context — audio only works after a user gesture, so call `ac()` from input handlers, as the existing ones do.

### Sprites

Tanks are drawn programmatically in `makeSprite()` and pre-rotated into the 4 directions by `makeFrames()`, then cached by `sprite()`. Sprites are recolored via the theme tables (`t[kind].b` / `.a`), and `hurt` swaps those two colors for the damaged-armor state.
