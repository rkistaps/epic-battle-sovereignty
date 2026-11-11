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

## Game systems

`docs/SYSTEMS.md` maps every game system (population, work, food, weather, gold, wood/iron, market, research, identity, troops, generals, battle, territory, relations, alliances, balance of power, AI), what each reads and writes, the yearly order, feedback loops and balance anchors.

- Read it before adding a system or changing rules in `R`, and judge changes by their knock-on effects on other systems.
- Every new system needs an AI side, settings entries, old-save handling and a headless simulation compared against the anchors (checklist in the doc).
- Keep the map current in the same commit (section 10 of the doc): a new system gets its own entry and is added to every system it reads from or writes to, the interaction table, the yearly order and the loops; a changed system updates its entry and every related system's entry, and the balance anchors if balance moved.

## Rules & guide (player-facing)

The in-game guide (Menu → Rules & guide, `helpModal`) reads the `GUIDE` array in the UI script: one topic per entry, `[id, group, icon, title, () => html]`, shown with a left menu and search.

- **Every commit that adds or changes a system, rule or screen updates the guide in the same commit.** A new system gets its own topic in the right group; a changed one updates its topic and every topic that mentions it, including "How it all fits together".
- The guide must teach the whole game: a player reading it should be able to learn exactly how each system works (what it is, what drives it, its numbers, what the player and computer states do with it) and how the systems push on and balance each other (what each one feeds, what brakes it, which choices compete for the same people, gold or goods).
- Take numbers from `R` and the data tables (`TROOPS`, `CIVS`, `LEADERS`, `LANDS`, `WONDERS`, `AREAS`, `SETUP_RULES`) instead of writing them in, so the guide follows Settings.
- It is the player's version of `docs/SYSTEMS.md`: keep the two in step, but write the guide in plain words, without code names.
- Check it in a browser: every topic opens without errors and search finds the new words.

## Persistence

- Saves are stored in IndexedDB (database `sovereign`), with localStorage as a fallback.
- Storage keys keep the old `sovereign-*` names so existing saves still load. Don't rename them.
- Setup and UI preferences use `eb.*` localStorage keys.

## Conventions

- Code is dense, compact JS with short helper names (`$`, `fmt`, `esc`, `M`, `lvl`). Match the existing style.
- Player-facing text is plain and short.
- Keep everything in the single file unless asked otherwise.

## Change log

The in-game change log (header button "Change log", `changelogModal`) reads the `CHANGELOG` array in the UI script.

- **Every commit that changes the game adds an entry at the top** of `CHANGELOG`, in the same commit: `["Short title", "YYYY-MM-DD HH:MM", ["bullet", ...]]`. Use the commit's local date and time.
- Bullets say what changed, not why. Keep them plain, short and easy to digest (e.g. "Archers cheaper", "Max button next to each Train button").
- Write for players: skip internal details (refactors, storage format) unless players notice them. Commits that don't touch the game (README, CLAUDE.md, screenshots) get no entry.

## Assets

- `docs/map.png`: the README screenshot, which shows Europe in a 1000-state world-map game with the default atlas lettering. To regenerate it:
  1. Run headless Chrome (e.g. puppeteer-core) at a 1800×1100 viewport.
  2. Start a game with `ENGINE.call("newGame", {nStates:1000, terr:100, difficulty:1, map:"world", seed:424242})`.
  3. Call `render()` and hide `.mapctl`.
  4. Find the cell for lon 14 / lat 49: `x=(lon-WORLD.lon0)/360*g.W` and `y=(WORLD.latN-lat)/(WORLD.H/WORLD.res)*g.H`.
  5. Call `mainView.centerOnCell(y*g.W+x, mainView.cw/(48/360*g.W))`. This shows about 48° of longitude.
  6. Screenshot `#mapbox canvas`, cropping the bottom 24px.
