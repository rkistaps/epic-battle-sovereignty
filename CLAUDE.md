# CLAUDE.md

## Project

The Sovereign (in-game title: **Epic Battle**) is a turn-based war and economy strategy game. Each turn is one year. All states move at once. The goal is to conquer every other state on a procedurally generated map.

## Structure

The whole game is one file, `index.html`. There is no build step, no dependencies and no package manager. To run it, open the file in a browser.

- `<style>`: all CSS. Colors are CSS variables on `:root`.
- `<script id="engine">`: the rules engine. **It must stay DOM-free.** At startup its text is turned into a Blob and run as a Web Worker (see "engine" in the UI script), so map generation and turn resolution happen off the main thread. It also runs in-page as a fallback.
  - Sections, marked with `// ---------- name ----------`: civilizations & leaders, troop types, pre-made maps, alliances, balance of power, wonders, terrain, player/AI actions, AI, relations, turn resolution, settings.
  - Entry points: `newGame(opts)` and `endTurn(data)`.
  - Tunable rules live in `R` and can be edited in the in-game Settings window.
- `<script>` (UI): rendering, tabs, modals, map view (canvas zoom/pan), fog of war, saves, news archive, auto-run.
  - The UI talks to the engine via `ENGINE.call("newGame"|"endTurn", data)`.
  - Global game state: `G`. UI state: `UI`.

## Persistence

- Saves are stored in IndexedDB (database `sovereign`), with localStorage as a fallback.
- Storage keys keep the old `sovereign-*` names so existing saves still load. Don't rename them.
- Setup and UI preferences use `eb.*` localStorage keys.

## Conventions

- Code is dense, compact JS with short helper names (`$`, `fmt`, `esc`, `M`, `lvl`). Match the existing style.
- Player-facing text is plain and short. The Rules & tips modal (`helpModal`) documents the mechanics. Update it when rules change.
- Keep everything in the single file unless asked otherwise.

## Assets

- `docs/map.png`: map screenshot used in the README. To regenerate it, run headless Chrome (e.g. puppeteer-core) and call `ENGINE.call("newGame", {nStates:16, terr:100, difficulty:1, water:15, waterStyle:"lakes", seed:424242})`. Then `render()`, click `[data-map=fit]`, hide `.mapctl`, and screenshot `#mapbox canvas`.
