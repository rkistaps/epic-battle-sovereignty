# CLAUDE.md

## Project

Epic Battle: Sovereignty is a turn-based war and economy strategy game. Each turn is one year. All states move at once. The goal is to conquer every other state on a procedurally generated map.

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

- `docs/map.png`: the README screenshot, which shows Europe in a 1000-state world-map game with the default atlas lettering. To regenerate it:
  1. Run headless Chrome (e.g. puppeteer-core) at a 1800×1100 viewport.
  2. Start a game with `ENGINE.call("newGame", {nStates:1000, terr:100, difficulty:1, map:"world", seed:424242})`.
  3. Call `render()` and hide `.mapctl`.
  4. Find the cell for lon 14 / lat 49: `x=(lon-WORLD.lon0)/360*g.W` and `y=(WORLD.latN-lat)/(WORLD.H/WORLD.res)*g.H`.
  5. Call `mainView.centerOnCell(y*g.W+x, mainView.cw/(48/360*g.W))`. This shows about 48° of longitude.
  6. Screenshot `#mapbox canvas`, cropping the bottom 24px.
