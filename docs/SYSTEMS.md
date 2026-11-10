# Systems map

Epic Battle: Sovereignty is a game of systems that act on each other every year. This document lists every system, what it reads and writes, and how they connect, so that a new system or a balance change can be judged by its knock-on effects, not just its local effect.

Read it before adding a system or changing a rule in `R`. Update it in the same commit when a system's inputs, outputs or core formula change.

All code is in `index.html`. Function names are given rather than line numbers (they drift). Default values are from `R` at the time of writing.

---

## 1. The big picture

```
                 ┌──────────── identity: civilization · leader · land type · wonders ───────────┐
                 │                     (multipliers on almost everything)                       │
                 ▼                                                                              ▼
 LAND ──► capacity ──► POPULATION (peasants) ──► WORK SPLIT ──► farmers ──► FOOD ◄── weather
  ▲                        │   ▲                   │   │            │                │
  │                        │   └── births ◄────────┘   │            │                ▼
  │                        │                           │            │          famine deaths
  │                        ▼                           ▼            ▼
  │                      TAXES ──► GOLD ◄──────── wages of scientists ── RESEARCH ──► bonuses to every system
  │                                  │                 ▲                              (food, tax, capacity, births,
  │                                  │                 └── woodcutters/miners ──► WOOD / IRON   atk, def, wood, iron, land)
  │                                  ▼                                                  │
  │                               MARKET ◄──── surplus grain/wood/iron, shortfalls ─────┘
  │                                  │
  │                                  ▼
  │                       TROOPS (peasants → soldiers, cost food/wood/iron/gold, eat upkeep)
  │                                  │
  │              GENERALS ──────────►│
  │                                  ▼
  └──── conquest, captured peasants ◄─ BATTLE ──► losses, grievances ──► RELATIONS ──► AI targeting
                                       ▲                                                   │
                                       └──────── AI decisions (work, market, army, attacks) ◄┘
                                                   balance of power (hegemon coalition)
```

**The one resource everything competes for is people.** Each peasant is either a farmer, woodcutter, miner or scientist (the work split), or is turned into a soldier. Only peasants pay tax and have children. So every system that uses people (research, army, production) takes from the others and from growth. Most balance questions come down to "what is one person worth in this job vs that one?"

---

## 2. One year, in order

The order matters: it decides which systems see this year's values and which see last year's.

**Player's turn (UI, instant).** `act.*` changes `G` at once: hire/dismiss troops, buy/dismiss generals, trade now or post/cancel orders. The work split, research allocation, attack orders and defending general are only stored. The player's market trades land in the current market year at the merchant prices set last year.

**End turn (`endTurnNow`).** Auto-sell posts surplus grain at the market price for one year. Then `ENGINE.call("endTurn", G)` runs:

1. Save migrations; logs reset; `g.avgLv` (world research average, used for catch-up pricing); `c.mod` recomputed for every state (settings changes apply).
2. **AI turns** (`aiTurn`), in state-id order, **before the harvest**: work split, market (trades happen now, so later ids see a changed book), generals, recruiting, research allocation, attack orders. AI reads last year's relations, hegemon, market averages, and `c.wxSys` (next harvest's weather systems, rolled last year).
3. **Battles**: attacks filtered (alive, general exists, adjacent, not allied, not peace time); attackers taken out of the army; the rest are guards. All attack strengths (with luck) computed first, then battles resolved one by one in shuffled order. Land, peasants and grain move; relations updated per battle (`relOnAttack`).
4. `settle` (unowned land claimed), `computeShore`, `refreshWonders`.
5. **Economy** per state (`economy`): harvest (× this year's weather) → eating → famine → spoilage → wood/iron → taxes − wages → empty treasury → births (none in a famine year) → research → generals age/train/recover → collapse below `minPop`.
6. **Market year** (`mkYear`): orders age and expire, the market price is updated, merchant prices reset, merchants take orders that beat them (`mkSweep`).
7. **Weather** (`mkWeather`): systems drift/merge/clash/spawn; next year's `c.wx` and `c.wxSys` are rolled.
8. **Relations year** (`relYear`), history, **hegemon** (`computeHegemon`), `turn++`, victory check.

Consequences:
- Conquests, settling, shore and wonders are applied **before** the economy: new land and captured peasants farm, eat and pay tax the same year.
- Weather is rolled a year ahead, so the AI and the player can see droughts coming (`c.wxSys`, "Weather ahead"). The random patches part is not shown ahead.
- Troops hired this turn fight this turn. Troops sent to attack do not defend this turn.

---

## 3. The systems

Each entry: **what it is · key rules · reads · writes · notes**.

### 3.1 Population
- Population = peasants + soldiers (scientists are peasants with a job; `c.sci` is a legacy field, always 0).
- Capacity: `popCap = land × (popPerKm 3 + densPerLevel 0.15·lvl(dens)) × M(capacity) × wonder(dens)`.
- Births: `peasants × growthRate × (1 − pop/cap)`, `growthRate = (birthRate 0.03 + growPerLevel·lvl(grow)) × M(births) × wonder(grow)`. Logistic: negative above capacity. None in a famine year.
- Collapse below `minPop` 20 (land becomes unowned). In battle a state is also absorbed if it has under 20 peasants.
- **Reads**: land (battle, settle), dens/grow research, identity, famine, battle losses and captures.
- **Writes**: taxes, food use, every job, research cost (size factor), AI decisions.
- Notes: peasants drive births, taxes and all jobs, so population compounds. That is why housing and growth traits apply at 40% (`traitWeight`).

### 3.2 Work split
- `c.work = {wood, iron, sci}` as % of peasants; farmers are the rest. `workers()` floors each job, so rounding goes to farming.
- Player: Overview tab sliders; each job ≤ 80%, the three together ≤ 90% (at least 10% farm); locks.
- Output per worker per year: farmer `foodPerFarmer` (3 base), woodcutter 2.8 wood, miner 2 iron, scientist 1 research point and costs `sciWage` 3.5 gold.
- AI: food first with a margin, then scientists (by phase, what it can pay, `aiSciMax`), then wood vs iron leaning to its better yield. Surplus logic (`aiGlut*`, `sellAvg`): steady selling or piling up means it farms/digs less and moves hands to research, as far as it can pay.
- **Reads**: population, research yields, identity, market prices (AI), stores, weather ahead (AI).
- **Writes**: food, wood, iron, research points, wages.
- Notes: the empty-treasury rule rewrites `c.work.sci` directly, including the player's.

### 3.3 Food
- `foodPerFarmer = 3 × (1 + agriPerLevel·lvl(agri)) × (1 + shoreBonus) × M(food) × wonder(agri)`; shore bonus = share of land on the shore, max +25%.
- Harvest = farmers × foodPerFarmer × `c.wx` × `g.clim` (weather).
- Use: 1 per peasant; troops eat their `upFood` (2; axe 3; cavalry 5).
- Shortfall: `deficit × famineDeath 0.5` die, soldiers weighted ×1.5. Then the store spoils `foodSpoil` 10%.
- **Reads**: work split, agri research, shore (land/borders), identity, weather, troop mix, market buys, loot.
- **Writes**: famine deaths, births (blocked by famine), market supply/demand, AI work split, `projectedNet`.

### 3.4 Weather
- Three layers, multiplied into the harvest:
  1. **Weather systems**: droughts and fertile spells 1–2 state-widths wide, ±12–22% at the centre, living 20–40 years, drifting, then a quiet gap and a new one elsewhere (`ws*`). Same kinds merge into Great Drought / Golden Harvest, then Age of Dust / Age of Plenty (stronger, bigger, longer). Opposite kinds clash: higher tier wins, equal tiers cancel. Can be turned off per game (`g.noWx`).
  2. **Local patches**: small random patches re-rolled every year, `wxRegional` 0.04 (kept small so systems stand out).
  3. **World years**: slow world-wide drift `g.clim`, ±4.5% at most.
- A state's weather is averaged over its land. Total capped at ±`wxMax` 25%, except a very strong system.
- **Reads**: map, state land (centres), RNG.
- **Writes**: harvest → food → famine, market prices (droughts drive grain up), AI planning (`c.wxSys` ahead), news.
- Notes: weather is the main source of supply shocks for the market. Local noise must stay well below systems, or players can't tell them apart.

### 3.5 Gold
- Taxes: `peasants × taxPerPeasant 0.6 × (1 + finTaxPerLevel·lvl(fin)) × wonder(fin)`.
- Wages: scientists × 3.5 + troops × `upGold` (only cavalry, 6).
- Empty treasury: debt forgiven, but `ceil(−money/2)` soldiers go home, then scientists are cut.
- Sinks: troop gold cost (cavalry), generals (`genPrice`, mind is exponential), market buys and buy orders (gold held). Sources: taxes, market sales (Finance adds +1.5%/level to what sellers get).
- **Reads**: population, fin research, market.
- **Writes**: research (can I pay scientists?), army (cavalry, generals, buying wood/iron), AI decisions (`projectedNet`, money dump into soldiers above `6·wages + 3000`).
- Notes: one scientist costs about six peasants' tax. That ratio is the main brake on research.

### 3.6 Wood and iron
- `production`: woodcutters × 2.8 (miners × 2) × M(wood/iron) × (1 + 0.06·lvl) × wonder. No spoilage, no upkeep.
- Used only to train troops (wood troops: spear, archer, axe; iron troops: pike, sword, cavalry) and on the market.
- **Reads**: work split, research, identity, wonders.
- **Writes**: troop quality (elite troops need iron), market.
- Notes: stocks pile up unless troops or the market use them. The AI now cuts these jobs when stock piles up.

### 3.7 Market (order book)
- One order book per resource. Post an order at your price (1/5/10/50 years or no end; gold or goods held), or trade now against the best orders. Grain held for sale spoils.
- **Merchants** stand behind the book: they sell at `max(mkMerchSell 2.5 × normal, mkMerchMul 2 × market price)`, dearer the more they sell in a year (`mkSteep` 3); they buy grain at 0.9 × normal and wood/iron at 0.5 × normal, paying less the more they buy (`mkSteepB` 0.5). At year start they take any order that beats their own price (`mkSweep`), which closes buy-from-states/sell-to-merchants arbitrage.
- Market price: average trade price per year (or the book midpoint in a year with no trades), averaged over `mkYears` 3.
- AI: `aiWorth` = market price × need (store vs needs) × a little personality, kept between merchant prices; takes orders that beat it, posts the rest ±8% (`aiMargin`), takes its orders back each year.
- **Reads**: every state's surplus and shortfall (food, wood, iron), gold, weather (through harvests and AI planning), troop demand.
- **Writes**: gold, stocks, `g.price` (used in AI troop costs and `projectedNet`), AI farming (cheap grain → farm less), news.
- Notes:
  - Merchant buy prices are the world's outside demand. They set the economy's income level: at 0.5× for grain, early gold fell ~5×. They do not follow the market price (that spiralled to zero).
  - The world over-produces by default, so prices sit near merchant buy prices unless something (drought, war, AI surplus logic) creates demand.

### 3.8 Research
- 10 areas: sci, agri, fin, dens, grow, atk, def, wood, iron, explore.
- Points: scientists × (1 + 0.075·lvl(sci)) × M(science) × wonder × `c.eff` (AI difficulty 0.8/1/1.15; humans 1). Split by `c.alloc`.
- Level cost: `5 × (L+1)^2.5 × size × edge`; size = `max(1, pop/1000)^0.5`; edge = −5% per level behind the world average (max −50%), +5% per level ahead (max ×3).
- Bonus curve: `lvl = L / (1 + L·per/(max·base))`, approaching each area's cap (`sciMax`, `agriMax`, …) and never passing it. In battle, one side's atk/def tech counts at most `techEdgeMax` 3× the other's.
- **Reads**: work split (scientists), gold (wages), population (size cost), the world average.
- **Writes**: nearly every other system's multipliers (food, tax, sell price, capacity, births, atk, def, wood, iron, land gain).
- Notes:
  - sci feeds its own points (positive loop), braked by cost exponent, size cost, lead penalty and caps.
  - Changing a `*PerLevel` also changes how fast the cap is approached.
  - The AI allocation is a fixed table by phase; it ignores costs and caps.

### 3.9 Identity: civilization, leader, land type, wonders
- Civilizations, leaders and land types are multipliers on traits: attack, defense, food, wood, iron, science, growth, housing, explore. `computeMods`: `1 + traitWeight × Σ(bonus − 1)` (bonuses add). Defense, growth and housing apply at 40% because they compound.
- Leaders also set AI personality: `mil` (chance of a militaristic persona) and `guard` (defense and research weights).
- Islands/Oceanic land types only for states with ≥3% shore.
- **Wonders** (if on): one per research area, +25% to that area's effect, held by whoever owns the cell; captured with the land.
- **Reads**: setup choices, land (wonders).
- **Writes**: multipliers everywhere; AI targeting (wonders attract attacks).
- Notes: any new trait or wonder must be checked against the compounding systems (population, defense, research). `traitWeight` is the balancing knob.

### 3.10 Troops
- Six types; each is one person taken from the peasants (at least 50 peasants stay).

  | Type | atk / def / tough | cost food/wood/iron/gold | upkeep food/gold | holds |
  |---|---|---|---|---|
  | Spearman | 1.25 / 1.25 / 1 | 2 / 10 / 0 / 0 | 2 / 0 | 1 |
  | Archer | 0.9 / 1.85 / 1.15 | 3 / 20 / 0 / 0 | 2 / 0 | 1 |
  | Axeman | 1.85 / 0.5 / 0.9 | 5 / 35 / 0 / 0 | 3 / 0 | 1 |
  | Pikeman | 1.25 / 2.25 / 1.6 | 2 / 0 / 35 / 0 | 2 / 0 | 1 |
  | Swordsman | 1.75 / 1.75 / 1.7 | 5 / 0 / 50 / 0 | 2 / 0 | 1 |
  | Cavalry | 3 / 2 / 2 | 10 / 10 / 100 / 100 | 5 / 6 | 2 |

- Toughness divides battle losses. "Holds" = km² of conquered land a survivor can hold.
- Dismissing returns people as peasants (no refund).
- AI picks troops by strength per total cost: training + 5 years of upkeep + the person (`aiPersonValue`, higher when births are slow or peasants short) + expected losses.
- **Reads**: population, food, wood, iron, gold, market (AI buys missing wood/iron).
- **Writes**: battle strength, food use, wages, fewer peasants (less tax, births, work).
- Notes: elite troops are the only way to more strength per person, so wood/iron supply and prices gate army quality.

### 3.11 Generals
- Needed to attack; one idle general defends. Price `30 × 2.6^mind × (1 + speed/8) × (1 + (40 − age)/40)`.
- Battle factor `(0.4 + skill·0.008) × (0.5 + morale/200)`: 0.4 fresh, up to 1.2. No defending general: 0.3.
- Speed caps land taken per battle (`10 + speed·2.5` km²). Skill grows with battles and yearly training; morale drops after battles, recovers 10/year; retire at 62–76.
- **Reads**: gold. **Writes**: battle strength, land taken.

### 3.12 Battle
- Attack strength = troops' attack × capped tech × M(attack) × wonder × general × luck (±15%).
- Defense = guards' defense (troops not sent this year, split between attackers by their strength) × capped tech × M(defense) × wonder × general × 1.15 × (1 + front bonus) × luck. Front bonus: +40% on a border ≤ 3 km, fading to 0 at 15 km; +15% for sea-only fronts.
- Losses: attacker `0.7 × strD/(strA+strD)` (×1.2 if it lost), defender `0.7 × strA/(strA+strD)`, each ÷ toughness. No guards: attacker loses 2%.
- Win if strA > strD. Decisiveness 25% at a narrow win, 100% at 2×.
- Land taken = `min((speedCap + 2√survivors) × exploreMul × decisiveness, survivors × holds)`, at least 1.
- Captures: the share of land taken moves the same share of peasants (80% arrive) and half the same share of grain. Gold, wood and iron are not looted. An eliminated state's stocks vanish.
- **Reads**: troops, generals, atk/def/explore research, identity, wonders, map fronts, AI/player orders.
- **Writes**: land (→ capacity, production, shore, wonders), population on both sides, research cost (size), relations, general skill/morale, AI memory (`lastWar`, `lostRecent`, `sentShare`, `openTurn`).

### 3.13 Territory and map
- Cells: water, unowned or owned; adjacency = 4-neighbours plus sea routes (world map). Allies never count as targets.
- Settling: each state bordering unowned land claims `(5 + 2% of its land) × exploreMul` a year. Unowned land comes from the map start and from collapsed states.
- Shore: land cells next to water → harvest bonus. Land type is fixed at start even if the coast is lost.
- **Reads**: battles, collapses, explore research. **Writes**: capacity, production (shore), wonders, fronts, adjacency.

### 3.14 Relations
- How each state feels about each other one, as a sum of factors (−100..100): grievance (it attacked us), solidarity (it attacked our ally), wariness (it attacked our neighbour), fear (it is much bigger), trust (years of peace), common enemy. Each has its own build-up and fade (`rg*`, `rs*`, `rw*`, `rf*`, `rt*`, `re*`). A strike back within 5 years counts half.
- **Reads**: battles, land sizes, adjacency, alliances.
- **Writes**: AI threat estimate, army size, home guard, target choice, edge needed, revenge chances, protector attacks; player-facing labels, map view and warnings.
- Notes: trust builds on every quiet border and makes the AI pickier; grudges can spiral (damped by revenge clearing grievance, floors and cooldowns).

### 3.15 Alliances
- Fixed teams chosen at setup (by count or size, regional or scattered). Allies never attack each other and win together. Solidarity links allies' feelings. No forming or breaking during play.
- Notes: states deep inside an alliance block can have no enemy neighbours and stay idle.

### 3.16 Balance of power (hegemon)
- The largest state by land becomes the hegemon when its share passes `1.5/alive` (6–40%). Strength `k` grows with its lead.
- AI only: the hegemon's neighbours keep bigger armies, prefer it as a target, need less edge, count a joint-attack bonus, and at k ≥ 0.5 ignore other armed targets.
- Notes: measured by land only; the player as hegemon is the main brake on a runaway player.

### 3.17 Computer players (AI)
- Runs every system above for computer states: work split, market, generals, recruiting, defense, research allocation, attacks. Personality from the leader (`mil`, `guard`), a per-state random lean (`kk`), and difficulty (research ×0.8/1/1.15, earlier war phase, smaller attack edge needed `[1.7, 1.35, 1.2]`).
- Attack decision: estimate the target's guard (with noise), apply relations, coalition and exposure, send enough to win by the edge and hold the land, keep a home guard. At most 2 attacks a year; a stalemate breaker makes it bolder after 5–25 idle years.
- **Every new system needs an AI side.** If the AI ignores a lever the player has, the player gets a free edge and the world drifts away from the player's experience.

### 3.18 Setup and difficulty
- `newGame` options: map (random with water/lakes/rivers, or world), number of states, land per state, wonders, alliances, weather systems on/off, random starting relations, observer, hot-seat humans, civ/leader/land picks. The UI then sets peace years, fog and difficulty.
- Starting values: 800 peasants, 2600 grain, 300 gold, 150 wood, 60 iron, work 15/5/10, no troops or generals.

---

## 4. Who affects whom

Rows act on columns (● strong, ○ weak).

| ↓ acts on → | Pop | Work | Food | Gold | Wood/Iron | Market | Research | Troops | Battle | Land | Relations | AI |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Population | | ● | ● | ● | ● | | ○ (cost) | ● | | | | ● |
| Work split | | | ● | ● | ● | | ● | | | | | |
| Food | ● (famine, births) | | | ○ | | ● | | ○ | | | | ● |
| Weather | | | ● | | | ● | | | | | | ○ |
| Gold | ○ (desertion) | ● (wages) | | | | ● | ● | ● | | | | ● |
| Wood/Iron | | | | | | ● | | ● | | | | ○ |
| Market | | ○ (AI) | ● | ● | ● | | | ○ | | | | ● |
| Research | ● | | ● | ● | ● | ○ | ● (sci) | | ● | ● (explore) | | ○ |
| Identity | ● | | ● | | ● | | ● | | ● | ● | | ● (persona) |
| Troops | ● | | ● | ○ | ● | ○ | | | ● | | | ● |
| Generals | | | | ○ | | | | | ● | ● | | |
| Battle | ● | | ○ (loot) | | | | ○ (size) | ● | | ● | ● | ● |
| Land | ● (capacity) | | ○ (shore) | | | | | | ● (fronts) | ● (settle) | ○ (fear) | ● |
| Relations | | | | | | | | | | | | ● |
| Hegemon | | | | | | | | | | | ○ | ● |
| AI | ● | ● | | | | ● | ● | ● | ● | | | |

## 5. Feedback loops

Positive (snowball) loops, and what brakes them:
- **Land → capacity → peasants → taxes, food, workers → army → more land.** Brakes: logistic births, size-scaled research cost, captured peasants arrive at 80%, coalition against the hegemon, fear and wariness, general speed and holding limits on land per battle.
- **Science → research points.** Brakes: cost exponent 2.5, size cost, lead penalty, caps, scientist wages vs tax.
- **Land → settling budget.** Brake: unowned land runs out.
- **Grudges → revenge → new grudges.** Brakes: revenge clears grievance, justified strikes count half, floors, cooldowns.

Negative (stabilising) loops:
- Catch-up research pricing pulls laggards toward the world average.
- Market: shortages raise prices, so AI states farm more and buy less; gluts lower prices and AI states farm less and move hands to research.
- Merchant prices climb while states keep buying from them, so demand dies out in a long shortage.
- Famine kills soldiers first (×1.5), shrinking food use.
- Empty treasury sends soldiers and scientists home.

## 6. Balance anchors

Numbers to compare against after any change (all-computer games, 60 states, land 100, difficulty 1, seeds 11/12/13, observer mode):

| Measure | Expected range |
|---|---|
| States alive at year 30 | 45–51 |
| States alive at year 150 | 10–18 |
| Average gold at year 30 | 3–6k |
| Average research levels (sum of areas) at year 150 | 105–125 |
| Famine collapses in 150 years | 0 |
| Grain sold to merchants (share of world food use) | about 3% |
| States with an empty treasury | well under 0.1% of state-years |
| Turn time on a 1000-state map | about 200–250 ms |

Relationships that hold the economy together:
- A scientist costs about 6 peasants' tax (`sciWage 3.5` vs `taxPerPeasant 0.6`).
- Merchants buy grain near normal (0.9×), so surplus grain is a real but modest income; wood/iron surplus sells at half.
- Local weather noise (`wxRegional`) stays well below weather systems.
- Defense, growth and housing traits apply at 40%.

## 7. Adding or changing a system: checklist

1. **Place it on the map above.** What does it read, what does it write, and which existing loop does it strengthen or brake? Prefer hooking into an existing quantity (people, food, gold, wood/iron, land, research multipliers) over adding a new currency.
2. **Decide when in the year it runs** (section 2) and whether others should see this year's or last year's value.
3. **Spend people, gold or goods.** Anything that gives a bonus should cost something that competes with other uses of people.
4. **Give the AI a side.** Computer states must use the system, or the player gets a free edge.
5. **Put its numbers in `R`** (with a comment) and in `SETTINGS_META` so they show in Settings; read `R` live so edits apply mid-game.
6. **Old saves**: tolerate missing fields (`??`, init on first use, `// older saves` comment) or add a `migrateX(g)` called from both `loadSave` and `endTurn`. Never rename `sovereign-*` storage keys. Keep state JSON-safe.
7. **Performance**: avoid new full-map loops per state; up to 1000+ states.
8. **Simulate** (section 8) and compare with the anchors in section 6, before and after. Look at pacing, gold, research, famine, and the system's own numbers.
9. **Player-facing**: UI, Rules & tips (`helpModal`), a change log entry, and this document.

## 8. Simulating

Run the engine headless in Node to compare versions:

```js
// node sim.js path/to/index.html
const fs=require("fs"), h=fs.readFileSync(process.argv[2],"utf8"), src=h.match(/<script id="engine">([\s\S]*?)<\/script>/)[1];
const E=new Function("module",src+";return module.exports;")({exports:{}});
for(const seed of [11,12,13]){ let g=E.newGame({nStates:60,terr:100,difficulty:1,seed,observer:true}); const out=[];
  for(let y=1;y<=150;y++){ g=E.endTurn(g)||g;
    if(y%30===0){ const A=g.countries.filter(c=>c.alive), avg=f=>A.reduce((s,c)=>s+f(c),0)/A.length;
      out.push(`y${y}: ${A.length} states, gold ${Math.round(avg(c=>c.money))}, research ${avg(c=>Object.values(c.lv).reduce((a,b)=>a+b,0)).toFixed(0)}, soldiers ${Math.round(avg(c=>c.soldiers))}`); } }
  console.log("seed",seed,out.join(" | ")); }
```

Compare against the previous version with `git show HEAD:index.html > old.html`. Use several seeds: single games vary a lot.

## 9. Known fragile spots

- The empty-treasury rule rewrites `c.work.sci`, including the player's setting.
- `workers()` floors each job, so rounding always adds farmers.
- Elimination thresholds differ: battle checks peasants < 20, famine collapse checks population < 20.
- An eliminated state's gold, wood and iron vanish.
- Battle order is random; an earlier battle can make later ones on the same target do nothing.
- The hegemon is measured by land only; an alliance spread over many mid-size states never triggers it. Joint-attack size counts the leader's own allies.
- The AI research allocation is a fixed table; it ignores costs, caps and wonders.
- AI difficulty scales only research (and AI thresholds), not the economy.
- The `R` comment on `aiSellFrac` says 30% but the value is 0.5.
- The `placeWonders` comment says six wonders; ten are placed.
