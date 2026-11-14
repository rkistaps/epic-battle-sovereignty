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
  └──── conquest (land; all people + stocks of a wiped-out state) ◄─ BATTLE ──► losses, grievances ──► RELATIONS ──► AI targeting
                                       ▲                                                   │
                                       └──────── AI decisions (work, market, army, attacks) ◄┘
                                                   fear of a giant → wars (balance of power)
  conquest ──► LOYALTY ──(too low)──► REVOLT: land, people, troops, goods → new rebel states (Rebel Alliance, at war with the parent)
                 ▲  quiet years, festivals (gold)        ▼ unrest: taxes −10%
```

**The one resource everything competes for is people.** Each peasant is either a farmer, woodcutter, miner or scientist (the work split), or is turned into a soldier. Only peasants pay tax and have children. So every system that uses people (research, army, production) takes from the others and from growth. Most balance questions come down to "what is one person worth in this job vs that one?"

---

## 2. One year, in order

The order matters: it decides which systems see this year's values and which see last year's.

**Player's turn (UI, instant).** Every human order goes through `applyCmd(g, who, {t, a})` (engine, section "commands"), which checks it and calls `act.*` or stores the plan; the UI never edits `G` for a human directly. `act.*` changes `G` at once: hire/dismiss troops, Tavern calls (`tavCall`, pays and rolls candidates), hiring one (`tavHire`) or sending them away (`tavAway`), staff posts (`post`, `govPol`, `autoPost`), the Spymaster's operation (`spy`: only stored, resolved at the end of the year, 3.11d), giving or taking back relics (`relicGive`, `relicTake`, 3.11c), dismissing generals, trade now or post/cancel orders. The work split, research allocation, attack orders and defending general are only stored. The player's market trades land in the current market year at the merchant prices set last year.

**End turn (`endTurnNow`).** Auto-sell posts surplus grain at the market price for one year. Then `ENGINE.call("endTurn", G)` runs:

1. Save migrations; logs reset; `g.avgLv` (world research average, used for catch-up pricing); `c.mod` recomputed for every state (settings changes apply).
2. **AI turns** (`aiTurn`), in state-id order, **before the harvest**: work split, market (trades happen now, so later ids see a changed book), generals (a Tavern call and hire, `aiTavern`), recruiting, research allocation, defense stance, attack orders. AI reads last year's relations, wars, market averages, and `c.wxSys` (next harvest's weather systems, rolled last year).
3. **Spies** (`spyYear`): every stored Spymaster operation (`c.spy`), in shuffled order; sabotage changes stocks, research points, pending unrest (`c.spyLoy`), the year's open gates (`g._gates`) and poisoned generals (dead, or `sick` and out of this year's battles) before any battle.
4. **Battles**: attacks filtered (alive, general exists, adjacent, not allied, not peace time); attackers taken out of the army; the rest are guards. All attack strengths (with luck and the generals' traits; the defender's general is known, for Pike Square) computed first, then battles resolved one by one in shuffled order; a Coward attacker may call his attack off. Land, peasants and grain move; relations updated per battle (`relOnAttack`).
5. `settle` (unowned land claimed), `loyaltyYear` (loyalty moves from this year's conquests and settling; states under `loyRevolt` may `revolt`, creating rebel states), `computeShore`, `refreshWonders`.
6. **Economy** per state (`economy`): harvest (× this year's weather) → eating → famine → spoilage → wood/iron → taxes − wages → empty treasury → births (none in a famine year) → research → generals age/train/recover (Drillmaster skill for field generals, Drunkard swing, Beloved recovery, Tattered Banner, Old Guard and Dragon Standard floors; retiring generals hand their relic to the Treasury, or to their successor with auto-replace, `genGone`) → collapse below `minPop`. Last year's undecided Tavern candidates are dropped the first time the Tavern is read in the new year (`tavOf`).
7. **Market year** (`mkYear`): orders age and expire, the market price is updated, merchant prices reset, merchants take orders that beat them (`mkSweep`).
8. **Weather** (`mkWeather`): systems drift/merge/clash/spawn; next year's `c.wx` and `c.wxSys` are rolled.
9. **Relations year** (`relYear`), **wars** (`warYear`: wars that ran their time end, new ones start from hostility), history, `turn++`, victory check. Computer declarations happen earlier, in step 2 (AI turns).

Consequences:
- Rebel states are created before the economy, so they farm, eat and pay tax the year they break away; famine and an empty treasury cost loyalty in the economy step, which counts toward next year's revolt check.
- Conquests, settling, shore and wonders are applied **before** the economy: new land (and the people and stocks of a wiped-out state) farm, eat and pay tax the same year; a beaten state's people squeezed past its capacity are gone before the economy runs.
- Weather is rolled a year ahead, so the AI and the player can see droughts coming (`c.wxSys`, "Weather ahead"). The random patches part is not shown ahead.
- Troops hired this turn fight this turn. Troops sent to attack do not defend this turn.

---

## 3. The systems

Each entry: **intent · what it is · key rules · reads · writes · notes**. The intent says what the system is for and which choices it should create. *(inferred)* marks intents read from the code and comments that the designer has not confirmed yet; *(your direction)* marks the designer's own decisions. When a change conflicts with an intent, ask before changing either.

### 3.1 Population
- **Intent**: People are the core currency. Growth rewards peace and room, and capacity is what makes land valuable: a state that stops growing has to expand (settle, conquer) or research density. Overcrowding is meant to hurt (births turn negative). Growth bonuses are kept small because people compound. *(inferred)*
- Population = peasants + soldiers (scientists are peasants with a job; `c.sci` is a legacy field, always 0).
- Capacity: `popCap = land × (popPerKm 3 + densPerLevel 0.16·lvl(dens)) × M(capacity) × wonder(dens)`.
- What fills capacity: `roomPop(c)` = peasants + scientists + soldiers (`c.sci` is legacy and always 0: scientists are peasants on the research job). Setting `peasantRoom` (default on since 2026-11-13) drops soldiers, so only peasants fill capacity; off restores the old rule where soldiers fill it too. `roomPop` is used by births, the battle trim, kidnap room, the AI's fullness checks (density research, land hunger, harass mode, `aiAtkRoom`/`aiAtkFull`) and the Home tab's Population card. Total population (`popOf`) still counts everyone for research size cost, festivals, rankings and collapse.
- Births: `(peasants + armyBirths 0.5 × soldiers) × growthRate × roomFactor(roomPop/cap)`, `growthRate = (birthRate 0.03 + growPerLevel·lvl(grow)) × M(births) × wonder(grow)`. `roomFactor(x) = 1 − x^birthCurve` (24): births stay near full speed until ~80% full, peak near 87%, stop at capacity; above capacity `1 − x` (a gentle decline) *(your direction: growth should not slow from half full)*. None in a famine year.
- Collapse below `minPop` 20 (land becomes unowned). In battle a state is also absorbed if it has under 20 peasants (with `keepPeople`, the capacity trim can push it there; not from soldiers with `peasantRoom` on).
- Losing land in battle (`keepPeople`): peasants above the new capacity are removed at once, so a beaten state sits at about 0.95× capacity instead of overcrowding (a land-only test without the trim left losers at 3.5–5×).
- **Reads**: land (battle, settle), dens/grow research, identity, famine, battle losses, the capacity trim of a beaten state and takeovers of wiped-out states, revolts (people leave with the land).
- **Writes**: taxes, food use, every job, research cost (size factor), AI decisions.
- Notes: peasants drive births, taxes and all jobs, so population compounds. That is why housing and growth traits apply at 40% (`traitWeight`).

### 3.2 Work split
- **Intent**: The main economic choice every year: every scientist or woodcutter is one farmer less, so no job is free. It should support distinct styles: farmer-exporter, industrial (wood/iron for an army or for sale), scholar. At least 10% always farm so a state can't starve itself by one slider. *(inferred)*
- `c.work = {wood, iron, sci}` as % of peasants; farmers are the rest. `workers()` floors each job, so rounding goes to farming.
- Player: Home page sliders; each job ≤ 80%, the three together ≤ 90% (at least 10% farm); locks.
- Output per worker per year: farmer `foodPerFarmer` (3 base), woodcutter 2.8 wood, miner 2 iron, scientist 1 research point and costs `sciWage` 3.5 gold.
- AI: food first with a margin, then scientists (by phase, what it can pay, `aiSciMax`), then wood vs iron leaning to its better yield. Surplus logic (`aiGlut*`, `sellAvg`): steady selling or piling up means it farms/digs less and moves hands to research, as far as it can pay.
- **Reads**: population, research yields, identity, market prices (AI), stores, weather ahead (AI).
- **Writes**: food, wood, iron, research points, wages.
- Notes: the empty-treasury rule rewrites `c.work.sci` directly, including the player's.

### 3.3 Food
- **Intent**: The survival floor and the first constraint. Surplus becomes gold through the market; a store buffers bad years, but spoilage (10%/yr) makes hoarding a deliberate insurance choice, not free. Famine kills soldiers first, so an army bigger than the land can feed falls apart. *(inferred)*
- `foodPerFarmer = 3 × (1 + agriPerLevel 0.05·lvl(agri)) × (1 + shoreBonus) × M(food) × wonder(agri)`; shore bonus = share of land on the shore, max +25%.
- Harvest = farmers × foodPerFarmer × `c.wx` × `g.clim` (weather).
- Use: 1 per peasant; troops eat their `upFood` (2; axe 3; cavalry 5), × (1 − the Quartermaster's cut, 3.11b).
- Shortfall: `deficit × famineDeath 0.5` die, soldiers weighted ×1.5. Then the store spoils `foodSpoil` 10%.
- **Reads**: work split, agri research, shore (land/borders), identity, weather, troop mix, staff posts (Quartermaster), spies (Burn the granary), market buys, stocks of a wiped-out state (and grain loot when `keepPeople` is off).
- **Writes**: famine deaths, births (blocked by famine), market supply/demand, AI work split, `projectedNet`, loyalty (a famine year −`loyFamine`).

### 3.4 Weather
- **Intent**: Outside shocks that give reasons to trade and to fight: drought states grow weak or desperate, fertile ones rich. Weather systems travel, end and restart elsewhere after a gap, several at once, and bad ones meeting make something worse *(your direction)*. They last decades and show a year ahead so players can plan: stockpile, buy early, farm more. Local yearly noise is texture only and must never outweigh a system *(your direction)*.
- Three layers, multiplied into the harvest:
  1. **Weather systems**: droughts and fertile spells 1–2 state-widths wide, ±12–22% at the centre, living 20–40 years, drifting, then a quiet gap and a new one elsewhere (`ws*`). Same kinds merge into Great Drought / Golden Harvest, then Age of Dust / Age of Plenty (stronger, bigger, longer). Opposite kinds clash: higher tier wins, equal tiers cancel. Can be turned off per game (`g.noWx`).
  2. **Local patches**: small random patches re-rolled every year, `wxRegional` 0.04 (kept small so systems stand out); `wxRegionalOff` 0.08 in games with weather systems off, so those games still have noticeable weather.
  3. **World years**: slow world-wide drift `g.clim`, ±4.5% at most.
- A state's weather is averaged over its land. Total capped at ±`wxMax` 25%, except a very strong system.
- **Reads**: map, state land (centres), RNG.
- **Writes**: harvest → food → famine, market prices (droughts drive grain up), AI planning (`c.wxSys` ahead), news.
- Notes: weather is the main source of supply shocks for the market. Local noise must stay well below systems, or players can't tell them apart.

### 3.5 Gold
- **Intent**: Turns people into flexibility: it pays for research (wages) and for elite military (cavalry, generals) and buys what the land lacks. Debt can't build up; overspending costs people instead, so running broke is a real setback. *(inferred)*
- Taxes: `peasants × taxPerPeasant 0.6 × (1 + finTaxPerLevel·lvl(fin)) × wonder(fin)` × (1 + a squeezing Governor's +1.5% per Mind, 3.11b).
- Wages: scientists × 3.5 + troops × `upGold` (only cavalry, 6) + Greedy generals (`trGreedyWage` 5% of their tier's price a year, `genWages`).
- Empty treasury: debt forgiven, but `ceil(−money/2)` soldiers go home, then scientists are cut.
- Sinks: troop gold cost (cavalry), Tavern calls (tier price × `genRise` 1.5 per extra call that year), market buys and buy orders (gold held). Sources: taxes, market sales (Finance adds +1.5%/level to what sellers get).
- **Reads**: population, fin research, market, loyalty (unrest: taxes −`loyUnrestTax` below `loyUnrest`), staff posts (Governor on Squeeze), spies (Rob the treasury: gold moves to the thief).
- **Writes**: research (can I pay scientists?), army (cavalry, generals, buying wood/iron), AI decisions (`projectedNet`, money dump into soldiers above `6·wages + 3000`), loyalty (festivals; an empty treasury −`loyBroke`).
- Notes: one scientist costs about six peasants' tax. That ratio is the main brake on research.

### 3.6 Wood and iron
- **Intent**: The military-industrial base. Wood is for mass troops, iron for elite ones. Leaders and land types skew yields so states can specialise and trade what they're good at. *(inferred)*
- `production`: woodcutters × 2.8 (miners × 2) × M(wood/iron) × (1 + 0.06·lvl) × wonder × (1 + forest / mountain bonus). No spoilage, no upkeep.
- Terrain: `forestBonus` = share of the state's land in forest × `forestYield` (1), max `forestMax` 25%; `mtnBonus` the same for mountains and iron. Follows the land (conquest moves it).
- Used only to train troops (wood troops: spear, archer, axe; iron troops: pike, sword, cavalry) and on the market.
- **Reads**: work split, research, identity, wonders, forest/mountain land (territory), spies (Fire the stores).
- **Writes**: troop quality (elite troops need iron), market.
- Notes: stocks pile up unless troops or the market use them. The AI now cuts these jobs when stock piles up.

### 3.7 Market (order book)
- **Intent**: A real market *(your direction)*: prices come from what states need. Shortages push prices up until demand dies out; states sell surplus for what they lack, which lets them specialise. Merchants are a last resort and a safety net, not the main trading partner. Lasting surplus should push states to produce less and research more rather than dump forever *(your direction)*. Orders can stand for long periods (1–50 years or no end) *(your direction)*.
- One order book per resource. Post an order at your price (1/5/10/50 years or no end; gold or goods held), or trade now against the best orders. Grain held for sale spoils.
- **Merchants** stand behind the book: they sell at `max(mkMerchSell 2.5 × normal, mkMerchMul 2 × market price)`, dearer the more they sell in a year (`mkSteep` 3); they buy grain at 0.9 × normal and wood/iron at 0.5 × normal, paying less the more they buy (`mkSteepB` 0.5). At year start they take any order that beats their own price (`mkSweep`), which closes buy-from-states/sell-to-merchants arbitrage.
- Market price: average trade price per year (or the book midpoint in a year with no trades), averaged over `mkYears` 3.
- AI: `aiWorth` = market price × need (store vs needs) × a little personality, kept between merchant prices; takes orders that beat it, posts the rest ±8% (`aiMargin`), takes its orders back each year.
- **Embargo**: states at war with each other can't fill each other's orders (`mkQuote` skips them); merchants still trade with both. The player sees enemy orders greyed with a ⚔️ *(your direction)*.
- **Reads**: every state's surplus and shortfall (food, wood, iron), gold, weather (through harvests and AI planning), troop demand.
- **Writes**: gold, stocks, `g.price` (used in AI troop costs and `projectedNet`), AI farming (cheap grain → farm less), news.
- Notes:
  - Merchant buy prices are the world's outside demand. They set the economy's income level: at 0.5× for grain, early gold fell ~5×. They do not follow the market price (that spiralled to zero).
  - The world over-produces by default, so prices sit near merchant buy prices unless something (drought, war, AI surplus logic) creates demand.

### 3.8 Research
- **Intent**: Long-term investment that competes with the army for people and gold. Each level is worth the same and costs `researchGrowth` 1.8× the last, so the late game converges without a hard cap and a tech lead can't become permanent; catch-up pricing keeps laggards in the game; size-scaled cost stops big empires out-researching by scale alone. *(inferred)*
- 10 areas: sci, agri, fin, dens, grow, atk, def, wood, iron, explore.
- Points: scientists × (1 + 0.06·lvl(sci)) × M(science) × wonder × `c.eff` (AI difficulty 0.8/1/1.15; humans 1). Split by `c.alloc` (whole % per area) ÷ max(100, sum): a player's shares may total under 100% and the unassigned rest is lost (Science tab warns, tab badge "!"); AI tables sum to 100.
- Level cost: `researchCost 35 × researchGrowth 1.8^L × size × edge` (cost of going from L to L+1); size = `max(1, pop/1000)^0.5`; edge = −5% per level behind the world average (max −50%), +5% per level ahead (max ×3).
- Bonus: linear, `lvl = L`, bonus = `per × L` (no cap; replaced the capped curve `L/(1+L·per/(max·base))` on 2026-11-12, tuned so each area's average bonus over years 30–150 stays within a few points of before). Per level: sci 6%, agri 5%, fin 4.7% tax + 1.5% sell price, dens +0.16/km², grow +0.11 pts birth rate, atk/def 6.5%, wood/iron 6%, explore 3.7%. Old saves (`g.resSteady` missing) convert each area to the nearest steady level with the same bonus. In battle, one side's atk/def tech counts at most `techEdgeMax` 3× the other's.
- **Reads**: work split (scientists), gold (wages), population (size cost), the world average, spies (Steal research adds points).
- **Writes**: nearly every other system's multipliers (food, tax, sell price, capacity, births, atk, def, wood, iron, land gain).
- Notes:
  - sci feeds its own points (positive loop), braked by the cost growth (×1.8 a level), size cost and lead penalty.
  - A `*PerLevel` change scales that area's bonus directly; `researchGrowth` sets how hard the late game brakes (higher = tighter soft ceiling).
  - The AI allocation is a fixed table by phase; it ignores costs.

### 3.9 Identity: civilization, leader, land type, wonders
- **Intent**: Each civilization, leader and land type pushes a play style, and no combination should dominate (bonuses add rather than multiply; values were simulated). Examples from the tables: Mountainous = defense, iron and research but poor food and expansion (turtle, tech); Plain = food, housing and expansion but weak defense (grow and settle fast); Volcanic = attack and research, poor wood; Farmer leader = food; Defender leader = keeps a big home guard. Wonders are prizes worth fighting for. *(comments + inferred)*
- Civilizations, leaders and land types are multipliers on traits: attack, defense, food, wood, iron, science, growth, housing, explore. `computeMods`: `1 + traitWeight × Σ(bonus − 1)` (bonuses add). Defense, growth and housing apply at 40% because they compound.
- Leaders also set AI personality: `mil` (chance of a militaristic persona) and `guard` (defense and research weights).
- Islands/Oceanic land types only for states with ≥3% shore.
- **Wonders** (if on): one per research area, +25% to that area's effect, held by whoever owns the cell; captured with the land.
- **Reads**: setup choices, land (wonders).
- **Writes**: multipliers everywhere; AI targeting (wonders attract attacks).
- Notes: any new trait or wonder must be checked against the compounding systems (population, defense, research). `traitWeight` is the balancing knob.

### 3.10 Troops
- **Intent**: Two families with different roles. **Wood troops** (spear, archer, axe) are cheap and fragile (toughness 0.9–1.15): for states with plenty of people and fast growth that can afford heavy losses. **Iron troops** (pike, sword, cavalry) are expensive and tough (1.6–2, about 40% fewer people lost per battle): for states short on people or with slow growth. **Swordsmen are the backbone of every army, attacking and defending** *(your direction)*: the best all-round value for most states, never the best pure attacker (axe, cavalry) or pure defender (pike, archer). Archer and pike defend, axe attacks (glass cannon), spear is the cheap all-round filler. Cavalry is the land-taker: strongest per person, holds 2 km² each, but needs iron and gold upkeep. The AI already follows this (it values a person more when births are slow and then leans to iron troops). *(inferred from the numbers and AI comments)*
- Six types; each is one person taken from the peasants (at least 50 peasants stay).

  | Type | atk / def / tough | cost food/wood/iron/gold | upkeep food/gold | holds |
  |---|---|---|---|---|
  | Spearman | 1.25 / 1.25 / 1 | 2 / 10 / 0 / 0 | 2 / 0 | 1 |
  | Archer | 0.9 / 1.85 / 1.15 | 3 / 20 / 0 / 0 | 2 / 0 | 1 |
  | Axeman | 1.85 / 0.5 / 0.9 | 5 / 35 / 0 / 0 | 3 / 0 | 1 |
  | Pikeman | 1.25 / 2.25 / 1.6 | 2 / 0 / 35 / 0 | 2 / 0 | 1 |
  | Swordsman | 1.85 / 1.85 / 1.7 | 3 / 0 / 45 / 0 | 2 / 0 | 1 |
  | Cavalry | 3 / 2 / 2 | 10 / 10 / 100 / 100 | 5 / 6 | 2 |

- Toughness divides battle losses. "Holds" = km² of conquered land a survivor can hold.
- Dismissing returns people as peasants (no refund).
- AI picks troops by strength per total cost: training + 5 years of upkeep + the person (`aiPersonValue`, higher when births are slow or peasants short) + expected losses. For the main army, strength = attack + defense with the stat its army lacks counting `aiMixTilt` 4×, so all-round troops (swordsmen) lead; defense top-ups use defense alone.
- **Reads**: population, food, wood, iron, gold, market (AI buys missing wood/iron).
- **Writes**: battle strength, food use, wages, fewer peasants (less tax, births, work).
- Notes: elite troops are the only way to more strength per person, so wood/iron supply and prices gate army quality.

### 3.11 Generals
- **Intent**: A scarce asset that grows with use, and a gamble with a floor: you choose a tier, luck chooses the man. A great roll costs the same as a poor one, so the pull is exciting; flaws are the risk. Traits give each general a character and reward building the army around him. Speed decides how much land a win can yield; lost battles hurt morale, so choosing battles matters. Needing a general per attack limits how many wars a state runs at once. No trait or post may make buying cheaper or reward swapping a general in and out within a turn: effects come from battles he commands or are yearly flows. *(your direction)*
- Needed to attack; one idle general defends. Battle factor `(0.4 + skill·0.008) × (0.5 + morale/200)`: 0.4 fresh, up to 1.2. No defending general: 0.3.
- **Tavern** (`act.tavCall`/`tavHire`/`tavAway`, `c.tav = {t: turn, n: calls per tier, pend: {tier, list}}`): a call costs the tier's price × `genRise` 1.5 per extra call that year (at most `genCalls` 10 a year per tier) and rolls `genCards` 3 candidates (`genCandidate`); hire one, the rest leave; undecided ones leave when the year turns. Tiers (`GEN_TIERS`, Settings → General tiers): Captain 600 (Mind 1–2, Speed 20–60, Age 30–50, skill 0–10, 0–1 traits, flaw 60% a slot, Legendary 0.5%), Veteran 3,000 (2–3, 40–80, 28–45, 5–15, 1, 35%, 2%), Academy graduate 20,000 (3–5, 60–100, 20–30, 0–10, 1–2, 15%, rare 8%, 6%). Each candidate rolls Legendary at its tier's `leg` chance: Mind 5–6, Speed 85–100, Age 22–35, skill 10–25, 2–3 traits, the first always rare. Retirement age 62–76 (at least 12 years away). Older saves keep their generals; their tier is read from Mind (`tierOf`), no traits. A state keeps at most `genMax` 10 generals (field, defender and staff posts together): the Tavern refuses calls and hires at 10. Generals have their own main-menu page (Generals: Generals · Staff · Tavern), after Market; War keeps Wars and Neighbors.
- **Traits** (`GEN_TRAITS`, `gen.traits`, hidden ones also in `gen.hid`): each slot rolls a flaw at the tier's chance, then a rare trait at its rare chance, otherwise a good one (`genTroopShare` 40% of good slots a troop trait); `genHide` 30% start hidden and show after his first battle (`revealTraits`). Exclusive groups: Lucky/Lucky Bastard/Jinxed, Cautious/Reckless/Coward, Beloved/Butcher. Effects (`tr*` in R): Lucky best of 2 luck rolls (×1.05 average), Lucky Bastard (rare) best of 3 (×1.075), Jinxed worst of 2 (×0.95), Reckless luck ±30%; Mountain Fox halves the defender's forest/mountain bonus when attacking and ×1.5 his own when defending; Siege Breaker (rare) halves narrow-front and sea bonuses; Iron Quartermaster (good) only works at the Quartermaster post (×2.5); troop traits (Phalanx, Master of Bows, Berserker Chief, Pike Square, Blademaster, Horse Lord) +20% (+15% swordsmen, cavalry) for that troop's share of the army's attack or defense, plus: spearmen hold 1.5 km²; defending, attackers lose 10% more; axemen lose 20% fewer; enemy cavalry −20%; swordsmen 15% tougher; Speed +20. Cautious: a third fewer losses in defeat, 20% less land won or lost. Coward: calls an attack off when the enemy looks 1.3× stronger before luck (−10 morale, no battle), −10% defending against a stronger attacker. Butcher: +10% attack; each win −1 loyalty and ×2 the beaten state's grievance. Beloved: +1 loyalty a win (at most +5 a year), morale recovers 15. Greedy: wage 5% of his tier's price a year. Drunkard: morale ±15 a year, half the training. Earned: Old Guard (20th battle, morale never below 50), Nemesis (lost twice to one state, `gen.nem`, +15% against it), Lucky Bastard (3 wins on a luck roll ≥ 1.10, replaces Lucky/Jinxed). A general also keeps `w`/`l` (won/lost), `lostTo`, `lw`, `since`, `face` (portrait seed).
- Speed caps land taken per battle (`10 + speed·2.5` km², `genSpeed` adds Horse Lord). Skill grows with battles and yearly training; morale drops after battles, recovers 10/year; retire at 62–76.
- **Reads**: gold (calls, Greedy wages), battles (record, earned traits, hidden traits showing).
- **Writes**: battle strength, losses and land taken, gold (Greedy wages via `wages`), loyalty (Beloved, Butcher), relations (Butcher's doubled grievance), AI estimates (`genEdge` = factor × average luck), staff posts (3.11b).

### 3.11b Staff posts
- **Intent**: Generals can run the state instead of fighting, which gives old, clever or flawed generals a use and makes "one more attack or one more post" a real choice. Every effect is a yearly flow worked out at the end of the year for whoever holds the post then, so swapping a general in for a moment gains nothing; no post makes buying cheaper. *(your direction)*
- `gen.post` ∈ qm, drill, spy, gov (one general each, `act.post`; the player's `post` and `govPol` commands). A posted general can't attack (orders dropped, `attack` refuses) or defend (`defGen` refuses; posting the defender hands the border to an idle field general). Effects (`postEff`, × Mind; a Drunkard at a post has a 50% bad year, `gen.hang`, at ×0.5):
  - Quartermaster: troops' grain upkeep × (1 − `postQm` 1% × Mind, Iron Quartermaster ×`postQmFit` 2.5, at most `postQmMax` 15%) in `foodUse`.
  - Drillmaster: every field general +`postDrill` 0.5 × Mind skill a year (in `economy`); his troop traits: that troop +`postDrillTroop` 5% × its share of the army's attack/defense in all the state's battles (`drillMul`, a `drillP` row in reports).
  - Spymaster (replaced the Chief of Staff on 2026-11-14; old saves' `chief` holders become `spy`, `migrateStaff`): works by skill, not Mind; one operation a year and the state's guard against foreign spies (3.11d).
  - Governor (`c.govPol`, ease by default): Ease loyalty +`postEase` 0.3 × Mind a year (Beloved ×2, Butcher ×0.5); Squeeze taxes +`postSqTax` 1.5% × Mind (Greedy ×0.5) and loyalty −`postSqLoy` 0.2 × Mind (Beloved ×0.5). A Butcher governor halves the revolt chance (`postGovRevolt`). Loyalty change goes into `loyaltyYear` (`c.loyY.gov`); taxes into `taxIncome`.
- **AI** (`aiStaff`, each turn after hiring): the three best generals by `genEdge` (Cowards count less) stay in the field; the rest take posts in order: Governor first when Rebellions are on and loyalty < 60, Quartermaster first when food production < use, Spymaster first when spies hit it in the last `aiSpyAlert` 10 years (`c.spyHitT`), then Spymaster, Drillmaster, Quartermaster, Governor, each going to the spare general who gives most there. Squeeze when loyalty > 72 and gold < 6 years of wages + 3,000, otherwise Ease. Computer states now hire up to 6 generals (was 4).
- **Reads**: generals (Mind, skill for the Spymaster, traits), the Governor's policy, loyalty (AI), food balance (AI), being spied on (AI). **Writes**: food use, taxes, loyalty and revolt chance, field generals' skill, battle strength (Drillmaster's troops), spies (3.11d), which generals can attack or defend.
- **Auto-replace** (`c.autoPost = {post id: bool}`, command `autoPost`, off by default, human only; computer states refill posts every turn in `aiStaff`): every general who leaves for good (retired in `economy`, poisoned or a Spymaster killed in `spyEffect`) goes through `genGone`; with his post's box on, `postRefill` puts the free general (no post, not the defender) with a fitting trait (`postFits`) first, then the highest `postScore`, on the post (an attack order he had is cancelled) and hands him the old holder's relic (his own goes to the Treasury); nobody free: the post stays empty and the relic goes to the Treasury. Old saves: no `autoPost`, all off.
- **Player sees**: Generals → Staff (four seats with drawn icons, portrait frame, effect now, fitting traits, an Auto-replace box; Ease/Squeeze on the Governor; the Spymaster's order with its odds and a Call off link; a picker box that lists every general with what he would give, best first, and what posting him changes; a "This year from your staff" box with the four totals, Spying showing the order); the general card's "At post" pill and Leave post button.
- **Player sees**: War → Generals (a card per general: drawn face that greys with age and gets scars from lost battles, tier, rank by battles (Major/Colonel 5/General 15/Marshal 30, for show), status, skill/morale bars, speed, Mind, won–lost, battle factor, trait chips with exact effects, hidden ones as ❔); War → Tavern (two regular boxes: Notice board, with the tiers as notice cards (drawn crests, price, calls left as tally marks, Legendary chance), and Candidates, where the candidates are dealt face down and turned one by one, a Legendary shaking and glowing gold first; compact candidate cards that stack in the side panel and go three across when wide); the attack form (trait icons, traits in the odds: average luck, Siege Breaker, Mountain Fox, Butcher, Nemesis); battle reports (faces, traits, a row per trait effect, how luck was rolled).

### 3.11c Relics
- **Intent**: Winning armies collect famous objects that make their generals stronger, and a famous relic is a prize worth a war. Four slots per general with no doubles (relic, troop or resource), a morale cost for taking one back and a rare loss in battle keep it a slow build-up, not a yearly reshuffle.
- `RELICS` (49: 16 general relics, 18 troop relics and 15 Governor relics): kind c common (Tattered Banner, Veteran's Blade, Lucky Coin, Scout's Map, Iron Shield), r rare (War Horn, Breaching Spear, Quartermaster's Ledger, Seal of Office), x cursed (Black Blade), u famous, one of each in the world (Crown of the First King, Dragon Standard, Saint's Reliquary, Hammer of the Breaker, Star Compass, The Unbroken Shield). `w` says where it works: `battle` (only while not on a post; attacks and defending), a post (`qm`, `gov`), or `any`. A relic is `{n, k, src, t, q, g}`: carried in `gen.rels` (up to `relSlots` 4; `relClash`: never two of the same relic, `tp` or `rs`, and at most one famous (`u`) relic, so the six famous ones spread over at least six generals; `relRoom`; `migrateRelics` moves extra famous relics to the Treasury) or waiting in the Treasury `c.relics`. Rule readers look a relic up by key (`relGet`, `rk` = carried and works where he is, `rv` = its rolled value of a rule), so two relics of different keys stack (attack relics multiply, land, speed, defender-loss and loyalty add; defender losses are cut at most 60%). Old saves: `migrateRelics` turns `gen.relic` into `gen.rels` (generals and pending Tavern candidates).
- **Legendary** (`g: 1`, `relTier` → `l`, orange): a rare or cursed relic forged Legendary; `relScale` × `relLegMul` 1.25 (so it sits between rare and famous: a Legendary troop relic gives ~12.5%, a famous one 15%). A max Legendary is `g` with `q` 1 (★). The Forge and the Merchant make them; spoils, conquest and auto-replace keep them.
- **Rolls**: every relic made (`newRelic`: ruins, candidates, `relSeed`) rolls `q` 0..1; `relScale` = 1 ± `relSpreadC` 0.4 (common), `relSpreadR` 0.2 (rare, cursed), `relSpreadU` 0.1333 (famous) × (2q − 1), applied to every number it gives through `rsc`/`rv` (post multipliers scale their bonus part; speed, morale and the Dragon floor round). Switches don't roll (luck die, border bonuses, never lost), nor the Black Blade's loyalty price. Older relics have no `q` and give the middle value. Spoils and conquest keep the roll.
- **Merchant** (`MERCH`, `act.merOpen`/`merTake`/`merAway`, `c.mer = {t, n per offer, pend: {tier, list}}` like the Tavern): Peddler's sack `merchSack` 600 (89.3% common / 8% rare / 2.7% Legendary), Merchant's chest `merchChest` 3000 (61 / 27 / 9 / 3% cursed), Royal coffer `merchCoffer` 15000 (8.7 / 64 / 21.3 / 6% cursed): the Legendary chance is `merchLeg` 1/3 of the rare chance, taken from common (`merOdds`); a Legendary pull is a random non-cursed rare with `g` 1 and a normal roll; each offer card lists its odds one tier a line (`merOddsList`); `merchCards` 3 relics laid out, take one; × `merchRise` 1.6 per extra one of the same offer that year, reset yearly; never famous. AI (`aiMerchant`, before `aiRelics`, only while a general carries none and its Treasury is empty (with four slots the old "fewer than 2 waiting" let computer states buy almost every year), with 6 years of wages + 5000 to spare, one a year; coffer only at 3× its price): takes the relic with the best `aiRelicValue` (kind, roll, troops it fields, Governor need, no Black Blade at low loyalty). A gold sink.
- **Forge** (`act.forge`, `forgePlan`; the player's `forge` command with three Treasury relic ids): 3 common → 1 rare for `forgeC` 2000 gold (same family, `relFam` = troop, resource or the relic itself: that family's rare; otherwise a random one of War Horn, Breaching Spear, Ledger, Seal), roll = the three rolls' average ± `forgeSpread` 0.15; 3 rare (cursed counts as rare) of one family → that family's rare made Legendary (`g` 1), roll = average ± `forgeSpread`, for `forgeR` 6000; 3 Legendary of one family → that Legendary at roll 1 (max) for `forgeL` 15000. Tiers can't be mixed; famous relics refused. AI (`aiForge`, after `aiRelics`, with 4 years of wages + 3000 to spare): three Legendaries of one family first (not max ones), then three rares, else its three weakest commons once it has 4+. A gold sink and a sink for spare relics.
- **Treasury stock and smelting** (`relStock` 50, `relFull`, `relOverflow`, `act.smelt`, command `smelt`): a full Treasury blocks `merOpen`/`merTake` and taking relics back (`relicTake`); spoils, ruins, conquest, retirement and swaps still arrive, and at the end of each state's economy step anything over the limit is smelted, lowest `relKeep` first (tier, then roll; never famous), for `smeltC/R/L` 150/600/2000 gold (cursed counts as rare), with a note for humans. The player smelts any non-famous Treasury relic for the same gold. Applies to computer states too (rarely binds: their Treasuries hold 88–117 relics in all at year 150 across the world, because free slots pull relics onto generals).
- **Auto-merge** (`forgeAuto`, `act.forgeAll`, command `forgeAll`): `forgeAuto` picks the next triple: by tier common → rare → Legendary (max ones skipped), three of one family first (lowest rolls), else any three commons. The UI's Auto-select puts it on the anvil; Merge all (after a confirm) forges `forgeAuto` picks until none is left or the gold runs out (at most 200), so results cascade upward. Only Treasury relics.
- **Where relics come from** (counters, seeds 11–13, 150 years, four slots): Merchant 684–765 a game (about 80%), hired candidates 118–147, ruins 3–12, 6 famous seeded; 39–120 melted in the Forge. With one slot it was 561–681 bought; free slots empty the Treasury, so the computer's "buy only with an empty Treasury" check passes more often, and carried relics (159–215 at year 150, was 56–94) never go on the anvil.
- **Governor relics** (`rs` resource grain/gold/wood/iron/sci, `lv` 0/1/2, `w` gov): while the carrier holds the Governor post (`govRes`), harvest (`foodProduction`), taxes (`taxIncome`, added to the Squeeze bonus), wood and iron (`production`) +`rlResC/R/U` 5/10/15%, research points (`researchPoints`) +`rlSciC/R/U` 3/6/10%. A Governor carries at most one per resource (`relClash`), so up to four resources can be raised at once, alongside the Seal and the Crown (up to 4 relics in all). The Staff page lists each on the Governor's line. Famous ones are not seeded (ruins only). Not shown in Science → Bonuses (that table covers research, identity, wonders and terrain); the Staff page shows it on the Governor's line.
- **Troop relics** (`tp` troop, `lv` 0 common / 1 rare / 2 famous; one per troop and kind): that troop +`rlTroopC/R/U` 5/10/15% for its share of the army's strength (`relTroopMul`, a `rel:k` row per troop relic in `trA`/`trD`; one per troop, so a general can boost several troops); archer relics only defending, axe relics only attacking. Extras on the rare and famous ones: Phalanx Standard / Spear of the First Legion spearmen lose 10/20% fewer (the Legion also ×2 land a spearman holds in `holdUnits`); Yew Longbow / Bow of the Hunter Queen attackers lose 5/15% more; Berserker's Totem axemen lose 10% fewer; Axe of the Mountain King removes the forest part of the terrain bonus; Hedgehog Banner / Wall of Pikes enemy cavalry −10/−25% for its share (`relCavCut`, a `relE:k` row); Duelist's Gauntlet swordsmen 10% tougher, Blade of Kings 25% fewer losses (`relLossMods`); Saddle of the Steppe / Banner of the Thousand Riders speed +10/+25. Famous troop relics are not seeded: they turn up only through ruins (the lost-famous share).
- **Effects** (`rl*` in `R`): Blade +3 skill in `genFactor`; Coin and Star Compass one more luck die (or lift a Jinx) in `luckDice`; Map +10 and Star Compass +25 speed in `genSpeed` (on top of the 100 cap); Iron Shield −10% and Unbroken Shield −20% defender losses (and no defending morale loss for the Unbroken Shield); War Horn +6%, Black Blade +12%, Dragon Standard +8% attack (a row `rel:k` in `a.trA`); Dragon Standard morale floor 60 (`oldGuardFloor`); Breaching Spear +15%, Hammer +20% land per win; Hammer removes the narrow-pass/sea bonus (replaces Siege Breaker's cut); Banner +5 morale a year; Saint's Reliquary +1 loyalty a year and never lost; Black Blade −0.5 loyalty a year; Ledger ×1.5 Quartermaster, Seal ×1.5 and Crown ×2 Governor (tax and loyalty), Crown +1 loyalty a year as Governor (`postEff` via `relPostMul`).
- **Finding** (`relSpoils`, `relRuins`, `relForCand`, `eliminate`): beating a general who carries some: `relSpoil` 5% (was 25%) to take one of them at random, never the Saint's Reliquary (either side, guarded battles with a defending general; to the winner's Treasury); a win that takes land: `relRuin` 2% to find a common one, `relRuinFamous` 20% of finds are a lost famous one; Legendary candidates always bring one (`relLegRare` 50% rare), Academy `relAcad` 10%; wiping out a state hands its Treasury and generals' relics to the winner (a collapse loses them). Famous relics start with `relFamous` 6 random computer states (`relSeed`; old saves get them on their next turn).
- **Player** (`act.relicGive(gen, n, rep)`/`relicTake(gen, n)`): giving into a free slot is free; a relic that clashes swaps out the one it clashes with; with all slots full the UI asks him to take one back first (`rep` lets a command name the one to replace); taking back or swapping costs the general `relTake` 10 morale. Dismissed, retired and poisoned generals hand theirs back (with auto-replace on his post, a retired or poisoned holder hands it to his successor, 3.11b). Famous relics changing hands are news for every human.
- **AI** (`aiRelics`, after `aiStaff`): hands Treasury relics to generals with a free slot and no clash, famous first, then Legendary, rare (by roll); with no room it swaps out a clearly weaker relic (the clashing one, or the weakest of a full general; `relTake` morale) on its two best candidates; a troop relic waits until 15% of its army is that troop; Governor relics: `aiStaff` posts a Governor first whenever one is at hand and keeps the general carrying it there, and `aiGovNeed` picks the resource (grain when short, wood/iron under 3 years of output, research for science personas, else gold); post relics to that post's holder, battle relics to the best field generals by `genEdge` (Black Blade only while loyalty > 60), `any` relics to whoever is free. War: `relCovet` 0.3 × famous relics held is added to the war motive (`warScore`) and twice that to the target score, like wonders.
- **Reads**: battles (winner, loser, land taken), generals (posts, `genEdge`), Tavern tiers, wipe-outs, loyalty (AI). **Writes**: battle strength, luck, losses and land; general speed, skill and morale; loyalty (`loyY.rel`); staff post effects; computer war motives and targets.
- **Player sees**: each general card has a 2×2 grid of relic boxes (`relSlot`; tier frame, ★ for max Legendary, greyed when it does nothing where he is, dashed + for a free slot); hovering shows the kind, effect, where it works, the one-per-troop/resource note and the roll (`gtip`); clicking opens `relPickModal` (his relics with Take back, the Treasury with Give / Swap / full), which stays open while giving and taking. Treasury: the Forge (Legendary previews), the vault, carried relics per general (n/4).
- **Player sees**: Generals → Merchant (key R): three offers as bills like the Tavern's notice board and the relics on the table turning over; a relic slot under every card's portrait (orange when it does nothing where he is); Generals → Treasury (key R): waiting relics with lore and where each came from, a give list, a picker for one general, who carries what, and the famous relics' holders; battle reports: the generals' relics, relic rows in the strength tables, a spoils or ruins box, a land chip; news notes.

### 3.11d Spies
- **Intent**: the Spymaster post gives staff an intel and sabotage side: a hidden contest of skill between neighbors' Spymasters, where an unseen hit costs no goodwill and a caught one costs as much as a small attack. Spies only spy: no gold cost, no capture, no weaker guard at home while operating; a false flag can turn a victim's grudge on a third state. *(your direction)*
- **Order** (`spy` command, `c.spy = {op, to, fr}`): one operation a year per state, against any living state that borders it (allies too); a new order replaces the old one; the map's right-click menu → Sabotage… (operations with odds, then whom to blame). `spyCan` refuses with no Spymaster, no shared border, a burned network (`c.spyBurn[j]` = year it ends), Steal research when the target leads in no area, Incite unrest without Rebellions, Open the gates in peace years.
- **Resolution** (`spyYear`, after the AI turns and before the battles; shuffled order): two rolls, skill against the target's Spymaster's skill (`spySkill`: his skill, ×`postDrunk` in a Drunkard's bad year; 0 with no Spymaster). Success = `spyOk<Op>` + `spySkill` 0.005 × (mine − theirs); found out = `spyDt<Op>` − `spySkillDt` 0.006 × (mine − theirs); both clamped to `spyMin`–`spyMax` 5–95%. Traits: Cautious found out −`spyTrait` 10%; Reckless success and found out +10%; Lucky best of 2 success rolls, Lucky Bastard best of 3, Jinxed worst of 2 (`spyDice`); Greedy keeps half the stolen gold; Butcher ×`trButcherGrudge` grievance when caught. Each operation gives the Spymaster +`spyLearn` 1 skill. The defender's Spymaster guards even while he runs his own operation.
- **Operations** (`SPY_OPS`; base success / found out at equal skill): Scout their army 85/15 (troops, stance, gold, grain, wood, iron, computer footing), Read their generals 75/20 (Mind, skill, morale, age, post, relic, every trait including hidden), Map their court 80/15 (their Spymaster and his skill); Burn the granary 65/35 (`spyGrain` 15% of grain), Fire the stores 65/35 (`spyStores` 15% of wood and iron), Rob the treasury 50/45 (`spyGold` 10% of gold to the thief), Steal research 50/30 (`spyRes` 25% of the thief's next level cost, in the area where the target leads most; the target loses nothing), Incite unrest 50/35 (loyalty −`spyLoy` 8, never below `loyRevolt`; applied in `loyaltyYear` as `c.spyLoy`, `loyY.spy`), Open the gates 40/50 (the target's defense ×(1 − `spyGate` 15%) in every battle against the thief's attacks that year, a `gates` row in `trD`; wasted, with a note, when no attack comes), Poison a general 30/55 (`gid`: a general named from the last Read their generals, else their defender, else anyone; of successes `spyPoisonDie` 10% die (`genGone`, the relic goes back to their Treasury, or to his successor with auto-replace, 3.11b; a Butcher Spymaster ×`spyButcherPoison` 1.5), `spyPoisonSick` 35% are too sick to fight that year (`gen.sick = turn`: dropped from attacks and as defender, so the border may stand unguarded; −`spyPoisonSkill` 15 skill), all survivors −`spyPoisonMorale` 30 morale; a failed poisoning that is found out kills the culprit's own Spymaster `spyPoisonBack` 8% of the time, and the computer counts that risk in `aiSpy`). Intel is stored in `c.intel[j] = {scout, gens, court}`, each with its year.
- **Outcomes**: not found out: the effect happens, no grievance and no name (loud sabotage tells the victim "unknown saboteurs"; quiet ones leave no trace; a failure tells a victim with a Spymaster only that a plot was foiled). Found out: grievance −`spyGrudge` 15 (intel −`spyGrudgeIntel` 5, poison −`spyGrudgePoison` 25, ×`spyPoisonDieX` 2 if he died) on the victim's `g` factor toward the culprit, the network in that state is burned for `spyBurn` 3 years, both sides are told; failure works the same minus the effect. **False flag** (`fr`, loud sabotage only: granary, stores, treasury, poison; a poisoning blamed on someone scales its grievance like poison's; a neighbor of the target, not the spy): when it works unseen the victim's report names the framed state and its grievance −`spyFrame` 12 goes to that state; caught framing: the culprit's grievance ×`spyFrameX` 1.5.
- **AI** (`aiSpy`, at the end of `aiTurn`, after attacks are planned): skips friends (relation ≥ 20) and allies; a foe is at war or at −30 or worse. Wants: Open the gates for its biggest attack (≥ 50 soldiers); Steal research where the target leads (0.45, science personas 0.7); Rob the treasury by how much it would bring against its wages (more when poor); foes only: Burn the granary (0.6 when the target's grain is under 1.2× its use, else 0.2), Fire the stores 0.2, Incite unrest 0.55 when the target's loyalty is under 70, Poison a general (their defender) 0.55 for its biggest attack's target, 0.3 at war, 0.15 otherwise, × (0.5 + defender's skill/100, at most 1.3). Score = want × success chance × (0.85–1.15 random) − risk × found-out chance (risk 0.1 against foes, `aiSpyRisk` 0.5 otherwise); acts on the best above `aiSpyMin` 0.12. Loud sabotage against a non-foe is blamed half the time on the target's neighbor it likes least (if it dislikes one). Being hit or catching spies sets `c.spyHitT`, which puts the Spymaster first in `aiStaff`.
- **Reads**: generals (the Spymaster's skill and traits), adjacency, wars and relations (AI), stocks, research levels, loyalty, attack orders (gates). **Writes**: grain, wood, iron, gold, research points, loyalty, battle strength (gates), generals (poison: death, sickness, skill, morale; who can attack or defend that year), relations (grievance toward the culprit or the framed state), intel, the Spymaster's skill, `aiStaff` priority.
- **Player sees**: map right-click → Sabotage… (each operation with success and found-out chances; for a poisoning, Their defender or a general named by intel; "≤/≥" best case until the court is mapped; disabled ones with the reason; then Leave no name / Blame X for loud sabotage; Call off); the Spymaster's seat and the "Spying" total on Generals → Staff; the state view's Spies tab (intel reports with their year, the network's burn); home news lines for every outcome on both sides; the 🗝️ row in battle reports; "unrest stirred by spies" in the loyalty line.

### 3.12 Battle
- **Intent**: Defending is easier: +15%, narrow passes and sea crossings, and ties go to the defender, so attacking needs a clear edge. Narrow wins take little land; crushing wins take a lot. Sending troops away leaves home open the same year, so timing and home guard are real trade-offs. A win brings land only: the beaten state keeps its people and goods up to what its land can hold, so it shrinks without being stripped, and winners must grow their own people (this slows mid-game snowballing). Wiping a state out pays: the winner takes all its people and stocks. *(your direction)*
- Attack strength = troops' attack × capped tech × M(attack) × wonder × general × luck (±15%; the general's luck traits change how it is rolled) × trait multiplier (troop traits, the defender's Pike Square, Butcher, Nemesis; `a.parts[6]`, listed in `det.trA`).
- Defense = guards' defense (troops not sent this year, split between attackers by their strength) × capped tech × M(defense) × wonder × general × 1.15 × (1 + front bonus) × luck × (1 + terrain bonus) × trait multiplier (`sD[9]`, listed in `det.trD`: the defender's troop traits, the attacker's Pike Square, the attacker's Siege Breaker and Mountain Fox and the defender's Mountain Fox on those bonuses, Nemesis, a Coward defender facing a stronger attacker). Older reports have neither trait slot. Terrain bonus (`terrainDefAD`): share of the frontier cells (defender cells touching the attacker) in forest × `forestDef` (15%) + in mountains × `mtnDef` (30%); stored last in the report's `sD` so older reports keep their order. The AI's target estimate and the player's attack odds include it. Front bonus: +40% on a border ≤ 3 km, fading to 0 at 15 km; +15% for sea-only fronts.
- Losses: attacker `0.7 × strD/(strA+strD)` (×1.2 if it lost, except raid/kidnap/massacre, which withdraw), defender `0.7 × strA/(strA+strD)`, each × the defender's stance loss multiplier and the attack mode's (`atLoss…` for the attacker, `atDef…` for the defender), capped at 100%, then ÷ toughness. No guards: attacker loses 2% (no stance). Traits: Master of Bows defending ×1.1 the attacker's losses, a Cautious attacker who loses ×(1 − 0.33); per troop type (`casualties` mods) Berserker axemen ×0.8, Blademaster swordsmen ÷1.15.
- Win if strA > strD. Decisiveness 25% at a narrow win, 100% at 2×.
- Land taken = `min((speedCap + 2√survivors) × exploreMul × decisiveness, survivors × holds)` × the defender's stance land multiplier × (1 − 0.2) for a Cautious attacker and again for a Cautious defender, at least 1; Phalanx spearmen hold 1.5. Raid, kidnap and massacre take no land.
- Attack mode (`a.mode` on the order; older orders and old `assault` orders: normal). Each mode has `atLoss` (attacker's losses), `atDef` (defender's), `atLand` and `atRel` (× every relation hit the attack causes: grievance, solidarity, wariness). Normal: 1 / 1 / 1 / 1.
- Raid, kidnap, massacre take no land and need no win; the take is `eff` × caps, where `eff` = strA/(strA+strD) (1 with no guards) and caps use the attacker's survivors. Raid (`atLossRaid 0.5`, `atDefRaid 0.5`, `atRelRaid 0.6`): `eff × min(raidMax 30%, survivors × raidCarry 30 ÷ their gold+grain+wood+iron)` of each stock. Kidnap (`0.5`, `0.75`, `1.2`): `eff × min(kidMax 15% of their peasants, survivors × kidPer 4)`, never more than the attacker's free room (`popCap − pop`). Massacre (`0.75`, `1`, `2`): kills `eff × min(masMax 20%, survivors × masPer 5)` peasants; every killed peasant costs the attacker `masFight 0.1` soldiers more. A kidnap or massacre that leaves the target under `minPop` peasants absorbs it (land and all). Battle reports keep `det.gain`. *(your direction: raids anger less than a conquest since no land is lost, kidnaps a little more, massacre in. Tuned so they pay: in tests a raid brings ~2,000 goods (about a year of an average state's taxes, 40 goods per soldier lost), a kidnap ~210–280 people for ~50–60 soldiers lost; a massacre kills ~1.7 of theirs per soldier lost and gains nothing)*
- Assault (an attacker-losses-for-land mode, ×1.15 / ×1.1) was built and removed: forced tests came out even at 1.15, but computer states used it mostly against half-beaten states standing ground, where it took less land than a normal attack. Its index (1) stays in `MODES` so old reports still read.
- Defense stance (`c.stance`, one per state for the year; older saves: hold). Fall back `stLossFall 0.6` / `stLandFall 1`: a cheaper fight for both sides, same land. Hold: 1 / 1. Stand ground `stLossStand 1.4` / `stLandStand 0.5`: bloodier for both, half the land. Only applies when there are guards. Hidden from other states; the battle report shows it (`det.st` = [stance, loss ×, land ×]). The player sets it under War → Wars (Defense). *(your direction: fall back and hold are about troops only; stand ground is the one stance that trades lives for land)*
- Spoils (`keepPeople` true, the default): the winner gets only land. The people on it move to the beaten state's remaining land; peasants and scientists above its `popCap` are removed in proportion (soldiers never). If the state is wiped out (no land, or under `minPop`), the winner takes all its peasants and scientists (as peasants) and all its gold, grain, wood and iron; its soldiers are lost. No trim when no land is left, so the winner gets everyone. A trim that leaves fewer than `minPop` peasants (soldiers fill most of the capacity) makes the state fall at once, handing over only what is left.
- With `keepPeople` false (the old rule): the share of land taken moves the same share of peasants (`capturedPeasants` 80% arrive) and `lootGrain` 50% of the same share of grain; a wiped-out state gives half its peasants and its stocks vanish.
- **Player sees**: battle reports (News, the year report, the Your battles screen that hides your results until you look) and a **replay** (▶ Watch, UI only: `rpPlan` builds it from the report's `tA`/`lA`/`tD`/`lD`, generals, terrain, mode and result; archers' share of their side's raw strength sets the share of enemy losses dealt by volleys, the rest falls in the melee and, after a rout, the pursuit; block area ∝ men). Changing what a report stores (`packDetail`) can break the replay.
- **Reads**: troops, generals (and their traits), spies (Open the gates: the defender ×(1 − `spyGate`) against that attacker, 3.11d), atk/def/explore research, identity, wonders, map fronts and their forest/mountain cells, AI/player orders (incl. attack mode), the defender's stance, the attacker's free room (kidnap), both sides' stocks (raid).
- **Writes**: land (→ capacity, production, shore, wonders), population on both sides, research cost (size), relations, general skill/morale/record and earned traits, AI memory (`lastWar`, `lostRecent`, `sentShare`, `openTurn`), loyalty (cells won: `loyGain` counts them as conquest or retake).

### 3.13 Territory and map
- **Intent**: Land is the prize. Unowned land at the start is a race; coasts are worth more (food). Collapsed states' land goes to whoever settles it first. *(inferred)*
- Cells: water, unowned or owned; adjacency = 4-neighbours plus sea routes (world map). Allies never count as targets.
- Starting shapes and sizes (`relaxStates`, every map): the land to share is every land cell a state can reach over land or sea routes; each state gets an equal share (`cap`). All states regrow from their middles at the same speed, one cell per turn, always the free cell with the lowest path cost from their land. Path cost follows a smooth random terrain field (two layers of value noise, scaled to state size) and each state's own stretch and direction (up to 1.6×), so borders wander and shapes differ. Between rounds (up to 10, the most even kept) each middle moves to the centre of its land plus the free land nearest to it, so states drift into empty regions; a state that ended short grows faster next round. Finishing: short states take nearest cells, free ones first, then from bigger neighbors (givers stay in one piece); `unthin` removes narrow arms; finishing again. Then borders are roughened: 60 passes in which border cells, in random order and with 50% chance, switch to a random 4-neighbor state that already holds 3+ of their 8 surrounding cells, if the giver stays in one piece and isn't smaller than the taker (sizes drift a cell or two); this gives ragged borders like the old generator without long strips. Measured: maps up to 300 states within ±2%; 1000-state world maps within ±5% (95% within ±2%); unowned land at start 1–2% (scraps and unreachable islands); creating a 1000-state world map takes ~14 s (150 states ~1–5 s).
- Settling: each state bordering unowned land claims `(5 + 2% of its land) × exploreMul` a year. Unowned land comes from the map start and from collapsed states.
- Shore: land cells next to water → harvest bonus. Land type is fixed at start even if the coast is lost.
- Forests and mountains (`makeFeatures`, `g.feat`: 0 open, 1 forest, 2 mountains): fixed per map, own dice from the seed (the rest of a seed's map is unchanged). Mountains: thin winding ranges where a value-noise field crosses its middle, only in some regions, share of land from the setup's Mountains slider (default 8%, `opts.mtn`; fallback `R.mtnShare`). Forests: noise blobs about a state wide, share from the Forests slider (default 16%, `opts.forest`; fallback `R.forestShare`; forests + mountains ≤ 100%). World map: real terrain instead (`WORLD_FEAT`, 0.5° grid like `WORLD`, built from NASA Blue Marble canopy colour and NOAA ETOPO1 shaded-relief roughness, both public domain; ~11% mountains, ~14% forest of land (densest canopy only)), scaled with a little noise on the edges (`worldFeatures`). Both kinds stop short of water: `shoreStrip` keeps a ragged strip of open land along coasts and rivers (~2 cells on generated maps, ~1 on the World map); generated-map shares are of all land but picked from land outside the strip, so high slider values come out a bit lower. Forest land raises wood and mountain land iron (`forestBonus`/`mtnBonus`, like shore: `forestYield`/`mtnYield` 1 = +1% per 1% of land, max `forestMax`/`mtnMax` 25%); counted per state in `computeShore` (`c.forest`, `c.mtn`) whenever borders move. Borders through forest/mountains also defend better (see 3.12). Saves made before this have no `feat` and show no terrain.
- **Reads**: battles, collapses, explore research, revolts (land handed to new rebel states). **Writes**: capacity, production (shore food, forest wood, mountain iron), wonders, fronts, adjacency, loyalty (settling lowers it a little; land share sets the ceiling).
- The list of states can grow mid-game (rebels): new ids are appended to `g.countries`, `ADJ` is dropped and `OWNED` gets a new entry; `g.cells` holds up to 32,767 ids.

### 3.14 Relations
- **Intent**: Aggression has memory and reaches beyond the victim: neighbours grow wary, allies hold solidarity, peace slowly builds trust. Revenge is understandable (a strike back counts half). The world should react to the player believably, not randomly. *(inferred from comments)*
- How each state feels about each other one, as a sum of factors (−100..100): grievance (it attacked us), solidarity (it attacked our ally), wariness (it attacked our neighbour), fear (it is much bigger), trust (years of peace), common enemy. Each has its own build-up and fade (`rg*`, `rs*`, `rw*`, `rf*`, `rt*`, `re*`). A strike back within 5 years counts half.
- **Reads**: battles (a Butcher general doubles the beaten state's grievance), spies found out (grievance −`spyGrudge`) and false flags (grievance on the framed state), land sizes, adjacency, alliances.
- **Writes**: AI threat estimate, army size, home guard, target choice, edge needed, revenge chances, protector attacks; player-facing labels, map view, warnings, and the state view's Relations tab (each state's feelings toward its neighbours and theirs toward it, worst first; moved out of the map tooltip).
- Notes: trust builds on every quiet border and makes the AI pickier; grudges can spiral (damped by revenge clearing grievance, floors and cooldowns).

### 3.14b Wars
- **Intent**: attacks happen in a context: single raids stay incidents, repeated or heavy blows, a grudge, fear of a giant or a full state with nowhere to grow lead to war, and wars end so old grudges can be forgotten *(your direction)*.
- A war between two states (`g.wars`: `{a, b, from, to, why}`) starts when **either** state's relation toward the other reaches `warHostile` −50 (all factors, fear included), or when a **computer state declares** one (`warScore`: motive × feasibility − cost above `warDeclare` 0.5; motive = grudge + ½ solidarity + fear + land hunger (≥85% full, nothing settled for 2 years) + wonders, scaled by the leader's warlikeness; feasibility = its sendable attack ÷ the edge it needs; cost = trust). Declaring costs −`warDeclareHit` 15 grievance with the victim and wariness with the declarer's other neighbors.
- A computer state declares only when it has no war and no rest years left; it can be declared on (and drawn into hostility wars) any time. While at war it attacks only its enemies.
- Each war lasts `warMin`–`warMax` (8–15) years or until a side is eliminated. At the end grievance, solidarity and wariness between the two reset (fear stays); both get `warRest` 5 years without declaring, and the pair can't go to war again in that time.
- No wars during the game's peace years. The player can't declare war; their attacks can start one through the victim's hostility, and computer states can declare on them.
- **Reads**: relations (all factors), battles (through relations), land and capacity (land hunger), settling (`c.settleT`), army strength, leaders, wonders, revolts (each rebel state starts a war with its parent, `why: "rebels"`).
- **Writes**: AI target choice (enemies only while at war), relations (declaration hit, reset at peace), news (year report to both sides, and to humans bordering or allied to either side).
- **Player sees**: War → Wars page (each war, since when, how it started; count of other known wars), ⚔️ on enemies in the neighbors list, "war" tags in the state view's Relations tab and its "At war with" line, a "Wars" map view (your enemies red, states at war orange, at peace grey).
- Notes: in 60-state games, about 100–120 wars per 150 years; most start from hostility, 3–16 are declared; states are at war ~30% of the time; ~2/3 of battles happen inside wars; 30–50% of wars start mostly from fear (the planned replacement of the coalition mechanic).

### 3.14c War footing (computer states)
- **Intent**: in peace a state fills its land (small army, density research); at war it spends people and refills them (bigger army, growth research) *(your direction)*.
- `c.footing` 0..1 moves `aiFootStep` 40% a year toward 1 (in any war) or 0.
- Army target share: `aiArmyPeace` 6% (warlike `aiArmyPeaceMil` 12%) → `aiArmyWar` 25% (35%) by footing, plus the late-game bonus (up to +10%), grudge extra; still at least the strong-neighbor minimum (half the strongest neighbor's attack, up to 20% of people). Defense top-ups arm at most `aiDefCapPeace` 30% → `aiDefCapWar` 40% of people.
- At peace (footing < 0.2) a state sends `aiPeaceCut` 10% of its soldiers above 1.25× its target home each year.
- Peasant reserve `aiKeepPeace` 80% → `aiKeepWar` 65% of people.
- Spare gold buys soldiers only in proportion to footing; in peace it stays (the gold-hoard rule moves hands to research).
- Research: density `aiDensPeace` 14 → `aiDensWar` 4, growth `aiGrowPeace` 6 → `aiGrowWar` 18; the difference comes from science.
- **Fullness and attacks** (not at war) *(your direction)*: below `aiAtkRoom` 70% full a state attacks only out of a grudge, fear or for a wonder; from 70% to `aiAtkFull` 90% the edge it needs falls from +`aiAtkRoomEdge` 50% to normal. At war, fullness doesn't matter. Effect: attacks outside wars by states under 70% full fell from 49% to 17%; conquest slower in mid-game (1000 states: 741 left at year 60 instead of 547), endgame similar (48 vs 41 at year 150). (Until 2026-11-12 the wonder test was always true, so this skip never fired; fixing it changed little: 60 states, year 150: 6–9 alive instead of 5–12, biggest state 21–32% instead of 24–40%; 1000 states: 131–179 alive instead of 129–157.)
- **Interior states** (only allied neighbors, sea routes count): a token army of `aiInteriorArmy` 2%, sending `aiInteriorCut` 20% of the surplus home a year; scientist cap +15%; research sci 35, density 30, agri 10, fin 10, growth 5, wood 5, iron 5; no spare gold into soldiers. Back to normal as soon as a non-allied neighbor appears.
- **Reads**: wars, neighbors and their armies (threat), relations, population and capacity, gold. Loyalty doesn't change footing; rebel states start at footing 0.5.
- **Writes**: army size, peasants (births, work), research allocation, scientists.
- Notes: armies react to neighbors' armies (an arms race): peaceful states still sit around 20–30% army by mid-game next to armed neighbors. Discounting peaceful neighbors' threat (`aiPeaceThreat` < 1) lowered peace armies but left states under-defended and conquest got much faster, so it stays 1. Effect in simulation: early conquest slower (60 states: ~50 left at year 30 instead of ~40; 1000 states: 854 left at year 30 instead of 686), states fuller (≈99% at year 60), endgame unchanged.

### 3.15 Alliances
- **Intent**: team scenarios with clear, fixed sides; alliances have their own slow-moving relations and wars on a separate layer from state wars *(your direction)*.
- Fixed teams chosen at setup (by count or size, regional or scattered). Allies never attack each other and win together. Solidarity links allies' feelings. No forming or breaking during play.
- **Alliance relations** (`g.arel["i>j"]`, −100..100, starting at 0): each year move `arStep` 10% toward how members feel about the other alliance's members (averaged over bordering pairs, weighted by land; no shared border: toward 0), plus direct hits: −`arWar` 1 per year of each state war between members, −`arBattle` 0.3 per battle between members (−`arBattleBig` 0.6 if over 10% of the defender's land is taken). No fear at this level.
- **Alliance wars** (`g.awars`: `{a, b, from, to}` between team indexes): start when an alliance's relation toward another reaches `awarHostile` −40 and that alliance isn't already in an alliance war (members' state wars don't hold it back). Every member is then at war with every member of the other (`atWar`, `inWar`, `foeTest` cover both kinds of war): embargo, computer targets and war footing, ⚔️ marks and the Wars map. State wars already running continue; members don't declare new state wars. Ends after `warMin`–`warMax` years or when one alliance is gone: the alliance relation resets to 0 (members' state relations stay), `warRest` years of rest for the pair.
- **Player sees**: War → Wars lists alliance wars and one line naming the alliances close to war with yours (min of both directions ≤ `awarHostile/2`, worst 3, as buttons); the Alliances tab (under the rankings) and each alliance window have the full "Alliance relations" table (both directions, bars, worst first); news to the members of both alliances when an alliance war starts or ends.
- **Reads**: members' relations, adjacency, land, state wars and battles between members, revolts. **Writes**: wars (through `atWar`), AI targets and footing, market (embargo), news.
- **Rebel Alliance**: the first revolt adds a team `{rebel:true}` named "Rebel Alliance" (tag RA, color `REBEL_COLOR`), creating `g.teams` if the game had no alliances; every rebel state joins it. States without a team stand alone (victory compares `team ?? "s"+id`), so rebels can win together.
- Notes: in 60-state, 4-alliance games, 4–7 alliance wars per game; relations bottom out around −70..−97. With hits of −3/−1/−2 alliance wars were near-constant (13–17 per game) because battles between many members add up.
- **Allied aid** *(your direction)*: allies joined over allied land, directly bordering or through a chain of bordering allies (sea links count, `allyGroup`/`aidReach`), send each other grain, wood, iron or gold (`aidSend`), arriving at once; no aid across the world (an ally with no allied neighbor gets none); the receiver's trust toward the giver rises (`aidTrust` 1 per 1,000 gold of value, at most `aidTrustMax` 5 a gift). Computer allies at peace (or interior) give up to `aiAidShare` 50% of their surplus a year (grain above 1.3 years of food incl. this harvest, wood/iron above the 3-year reserve, gold above 6 years of wages + 2,000) to reachable allies at war that are short, neediest first, before market trading. The player sends aid from their alliance's window. In 60-state, 4-alliance games: ~800–1,300 gifts per game, mostly iron.
- **Alliance window** (`allianceModal`): opened from the Allies page, War → Wars (alliance wars, alliances close to war) or the alliance line of a selected state: stats (states, land and world share, population, army, average research), alliance wars and members' state wars, relations with the other alliances, members (click to select), Send aid (your own alliance), Show on map.
- Notes: states deep inside an alliance block can have no enemy neighbours (interior states, 3.14c).

### 3.16 Balance of power (through fear wars)
- **Intent**: stop a runaway winner, the player included: the world pushes back on whoever gets too big *(your direction: fear wars replaced the old coalition)*.
- No separate mechanic any more: neighbors fear a state from 1.5× their size (relations, fear factor up to −50), and fear counts toward the −50 war trigger (3.14b), so a runaway state ends up at war with the neighbors it frightens. Fear also lowers the edge a computer state needs against a target others are already attacking.
- With Rebellions on, **loyalty** (3.16b) brakes a runaway state from the inside: conquest costs loyalty, big states recover it slowly under a lower ceiling, and a revolt splits off its newest land.
- Notes: compared with the removed coalition, runaway states grow a little faster mid-game (in 2 of 4 test games the leader held 18% vs 8% and 30% vs 24% of the land at year 70) and fight fewer wars at once; the endgame is similar. Fear's strength (`rfFrom`, `rfPer`, `rfMax`) and `warHostile` are the knobs if it needs to be stronger.

### 3.16b Loyalty and rebels
- **Intent**: a brake on states that expand too fast, the player included: conquered land doesn't stay loyal, and big empires break up into rebel states that fight them. Every state has a ceiling; rebellions don't weaken late in the game; rebels are regular states that share one alliance and can win *(your direction)*.
- New-game rule "Rebellions" (`g.rebels`, on by default; older saves: off). `c.loy` 0..100, starting at the state's ceiling.
- Yearly (`loyaltyYear`, after battles and settling): `loy += rec − loyConq × 100 × (won + loyRetake × retaken + loySettle × settled) ÷ land at the start of the year`, then clamped to `[0, ceiling]`. `rec` = `loyRec` 4 at peace / `loyRecWar` 2 at war × √(min(1, `g.land0` ÷ land)), at least `loyRecMin` 1. Cells won are counted in `transferCells` and wipe-outs (`loyGain`): a cell the state lost in the last `loyRetakeYears` 20 counts as retaken (× 0.5), except when taken back from its own rebels. Ceiling = 100 − `loyCeilMax` 70 × (share of the world's land)^`loyCeilExp` 0.5: 93 at 1%, 65 at 25%, 30 for the whole world (so conquering the world stays possible). Famine −`loyFamine` 5, empty treasury −`loyBroke` 5 (in `economy`).
- Effects: below `loyUnrest` 40 taxes −`loyUnrestTax` 10% (`taxIncome`). Below `loyRevolt` 25, a state with at least `rebParent` 1.5× `g.land0` revolts with chance (25 − loy) × `loyRevoltPer` 4% a year (no revolts in peace years).
- `revolt`: the number of rebel states is rolled from `rebOdds` (1–9: 25.4/19.5/15.1/11.6/9.0/6.9/5.3/4.1/3.1%: each about 0.77× the one before, 1–3 = 60%, 9 = the state falls apart) *(your direction)*, capped by how many pieces of `rebMinLand` 0.15 × `g.land0` fit outside the core. The core (cells with the oldest `takenAt`) never breaks away *(your direction)*. Seeds: the best-scored free cell (score = 2 × recency (won in the last `rebRecent` 30 years; settled land half) + distance from the core + noise), then each next one far from those chosen (distance × score); pieces grow at the same time, smallest first, each to a rolled `rebShareMin`–`rebShareMax` (8–12%) of the land, so they come out about equal; a piece under `rebMinLand` stays with the state. 3 pieces take about 30%, 9 about 90% (the state keeps its core and leftovers). Peasants, scientists (as peasants), gold, grain, wood, iron go by land share; troops by `rebArmy` 1 × land share (same as the land); militia: `rebMilitia` 10% of the rebels' peasants become spearmen; each rebel takes a general with `rebGen` 50% if the parent has 2+. Rebel: parent's civilization and land type, `rebResearch` 2/3 of its research levels (rounded; catch-up pricing then helps them close the gap), random leader, name from `rebelName` ("Rebels of X", "X Rebels", …), `color` `REBEL_COLOR` dark grey, `rebelOf`, `born`, the Rebel Alliance team. Grievance −`rebGrudge` 60 both ways and a war (`warStart(..., "rebels")`). The parent's loyalty +`rebAfter` 30. `g.revolts` lists the year's revolts.
- Festivals (`act.festival`): gold per point = `festCost` 1 × population ÷ 100 × √(max(1, land ÷ `g.land0`)); at most `festMax` 10 points a year; never above the ceiling.
- **AI**: festivals with gold above 3 years of wages + 500 when loyalty < `aiFestLoy` 45; not at war and loyalty < `aiLoyCalm` 30: no land grabs (raids, kidnaps, massacres still allowed); an attack whose land would leave loyalty under `aiLoyFloor` 15 (warlike 10) becomes a raid (at war or for a grudge) or is dropped.
- **Reads**: battles and wipe-outs (cells won), spies (Incite unrest, `c.spyLoy`, never below `loyRevolt`), generals (Beloved +1 a win up to +5 a year, Butcher −1 a win), the Governor (3.11b), relics (Saint's Reliquary +1, Black Blade −0.5 a year, Crown +1 as Governor; 3.11c), settling, land and world land (ceiling, recovery), wars (recovery), famine, empty treasury, gold (festivals), peace years.
- **Writes**: taxes (unrest), territory (rebel states take land), population, troops, generals, stocks (they leave with the land), relations and wars (rebels vs parent), alliances (Rebel Alliance), AI target choice and attack mode, victory (rebels win together).
- **Player sees**: ❤️ in the header; Home page Loyalty cards (in the Population box) (value / ceiling, status, last year's changes, next year's recovery, revolt chance, Festivals); War → Wars line; loyalty in state details; the Loyalty map view; the year report's 🔥 Rebellion note; war news to neighbors.
- Notes (60 states, seeds 11–16, 150 years): see section 6. Computer states mostly hold themselves back (the AI floor), so revolts are uncommon in all-computer games (about 0–7 per game) and rebels rarely last long; the main braking comes from slower expansion of the biggest states. A player who conquers fast meets revolts much sooner.

### 3.17 Computer players (AI)
- **Intent**: Play by the same rules as the player: no free resources (difficulty only scales research and how bold it is). Personalities differ so each world plays differently. The AI should use every lever the player has. *(inferred)*
- Runs every system above for computer states: work split, market, generals (Tavern: the best of Academy/Veteran it can afford at 1.25× price + wages + 500, a Captain only with no general; hires the candidate with the highest `aiGenValue`, which counts only visible traits), recruiting, defense, research allocation, attacks. Personality from the leader (`mil`, `guard`), a per-state random lean (`kk`), and difficulty (research ×0.8/1/1.15, earlier war phase, smaller attack edge needed `[1.7, 1.35, 1.2]`).
- Attack modes: harass its biggest neighbor when that one is at least `rfFrom` 1.5× its size, feared (`aiHarassFear` 0.3) and too strong to beat: kidnap if under `aiAtkFull` full, raid otherwise, massacre with a grudge over `aiMassacreGrudge` 0.7; sends at least `aiHarassMin` 40, aiming at `aiHarassStr` 60% of their defense; then waits a random `aiHarassGapMin`–`aiHarassGapMax` (3–7) years (`c.harN`). Raids a beatable neighbor instead of conquering when its gold is under `aiRaidPoor` 2 years of wages and theirs is twice its own.
- Defense stance: stand ground once land is under `aiLastStand` 50% of its biggest ever (`c.landMax`) or while holding a wonder in a war; fall back when its army's defense is under `aiFallBack` 50% of the strongest neighbor's threat; otherwise hold. In simulation about 22–34% of state-years fall back and 1–3% stand ground. Attack planning ignores the target's stance (it is hidden).
- Attack decision: estimate the target's guard (with noise; both generals at `genEdge`, so luck traits count on average; Butcher counts), apply relations, wars and exposure, send enough to win by the edge and hold the land, keep a home guard. At most 2 attacks a year; a stalemate breaker makes it bolder after 5–25 idle years.
- Spies (`aiSpy`, after the attack plans): the gates or a poisoned defender for its biggest attack, sabotage of foes, quiet theft at peace, false flags on disliked neighbors; a Spymaster first after being spied on (3.11d).
- **Every new system needs an AI side.** If the AI ignores a lever the player has, the player gets a free edge and the world drifts away from the player's experience.

### 3.18 Setup and difficulty
- **Intent**: Variety and replay value; difficulty mostly changes how bold and how fast-researching computer states are. *(inferred)*
- `newGame` options: map (random with water/lakes/rivers, or world), number of states, land per state, wonders, alliances, weather systems on/off, random starting relations, observer, hot-seat humans, civ/leader/land picks. The UI then sets peace years, fog and difficulty.
- State names: shuffled from `NAMES`; `NAME_GEN.share` of the states (default 0) and every state beyond the list get `inventName` (title pattern from `NAME_GEN.forms` with {N} = root, {A} = `nameAdj(root)`; root = start + middle (~45%) + end). Rebels use `nameRoot` in `rebelName`. All parts editable in Settings → State names (`ngen`).
- Starting values: 800 peasants, 2600 grain, 300 gold, 150 wood, 60 iron, work 15/5/10, no troops or generals.

---

## 4. Who affects whom

Rows act on columns (● strong, ○ weak).

| ↓ acts on → | Pop | Work | Food | Gold | Wood/Iron | Market | Research | Troops | Battle | Land | Relations | AI | Loyalty |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Population | | ● | ● | ● | ● | | ○ (cost) | ● | | | | ● | |
| Work split | | | ● | ● | ● | | ● | | | | | | |
| Food | ● (famine, births) | | | ○ | | ● | | ○ | | | | ● | ○ (famine) |
| Weather | | | ● | | | ● | | | | | | ○ | |
| Gold | ○ (desertion) | ● (wages) | | | | ● | ● | ● | | | | ● | ○ (festivals, empty treasury) |
| Wood/Iron | | | | | | ● | | ● | | | | ○ | |
| Market | | ○ (AI) | ● | ● | ● | | | ○ | | | | ● | |
| Research | ● | | ● | ● | ● | ○ | ● (sci) | | ● | ● (explore) | | ○ | |
| Identity | ● | | ● | | ● | | ● | | ● | ● | | ● (persona) | |
| Troops | ● | | ● | ○ | ● | ○ | | | ● | | | ● | |
| Generals | | | ○ (Quartermaster) | ○ (calls, Greedy, Governor) | | | | | ● (traits, Drillmaster) | ● | ○ (Butcher) | ○ (estimates) | ○ (Beloved, Butcher, Governor) |
| Spies | | | ○ (granary) | ○ (treasury) | ○ (stores) | | ○ (steal) | | ○ (gates, poison) | | ● (caught, false flag) | ○ (Spymaster priority) | ○ (unrest) |
| Relics | | | ○ (Ledger) | ○ (Seal, Crown) | | | | | ● (attack, losses, luck, skill) | ○ (land per win, Hammer) | | ○ (covet famous ones) | ○ (Saint, Black Blade, Crown) |
| Battle | ● (kidnap, massacre) | | ○ (wipe-out, raid) | ○ (raid) | ○ (raid) | | ○ (size) | ● | | ● | ● (× mode anger) | ● | ● (conquest) |
| Land | ● (capacity) | | ○ (shore) | | | | | | ● (fronts) | ● (settle) | ○ (fear) | ● | ● (ceiling, recovery) |
| Relations | | | | | | | | | | | | ● | |
| AI | ● | ● | | | | ● | ● | ● | ● | | | | ○ (festivals) |
| Loyalty | ● (revolt) | | ○ (stocks leave) | ○ (unrest tax) | ○ (stocks leave) | | | ● (defect, militia) | | ● (revolt) | ● (rebel wars) | ● (restraint) | |

## 5. Feedback loops

Positive (snowball) loops, and what brakes them:
- **Land → capacity → peasants → taxes, food, workers → army → more land.** Brakes: births stop at capacity (only peasants count with `peasantRoom` on, the default; off, soldiers count too and armies also brake growth), size-scaled research cost, conquest brings land but no people (only a wipe-out hands over people and stocks), fear wars against a runaway state, fear and wariness, general speed and holding limits on land per battle.
- **Science → research points.** Brakes: cost ×1.8 per level, size cost, lead penalty, scientist wages vs tax.
- **Land → settling budget.** Brake: unowned land runs out.
- **Conquest → land → more conquest** is also braked by loyalty (Rebellions on): each conquest costs loyalty, recovery slows with size and the ceiling falls with world share; a revolt hands land, people and troops to rebel states at war with the parent. Revolt → +30 loyalty → conquest resumes, so big states can cycle (revolt, retake at full cost, revolt).
- **Grudges → revenge → new grudges.** Brakes: revenge clears grievance, justified strikes count half, floors, cooldowns.
- **Winners collect relics** (spoils from beaten generals, ruins, a fallen state's Treasury). Brakes: four per general with no doubles and one famous, a 5% chance to lose one in a lost battle, the Forge's gold and relic cost for Legendaries, a famous relic draws computer wars (`relCovet`), a collapse loses them.
- **Veteran generals get better** (skill, earned Old Guard and Lucky Bastard). Brakes: retirement at 62–76, morale lost in battles, Nemesis helps only against one state; a new general (the Tavern) starts at skill 0–25.

Negative (stabilising) loops:
- Catch-up research pricing pulls laggards toward the world average.
- Market: shortages raise prices, so AI states farm more and buy less; gluts lower prices and AI states farm less and move hands to research.
- Merchant prices climb while states keep buying from them, so demand dies out in a long shortage.
- Famine kills soldiers first (×1.5), shrinking food use.
- Empty treasury sends soldiers and scientists home.
- Defense stance: outgunned states fall back, which slows how fast both armies wear down and so slows conquest; states that lost half their land stand ground, so the last cells cost more lives to take.
- Harassment: small states raid or kidnap from a feared giant next door, moving goods and people from big to small; it nibbles at leaders but bleeds the harassers' troops (raids lose ~14% of those sent) and angers the giant.
- Spies: stolen gold and research are shares of the victim's stock and of the thief's own next level, so a small state gains most from a rich, advanced neighbor, and only from one that leads it (Steal research). Brakes: one operation a year, a skilled Spymaster next door, a burned network after being caught, and the grievance that can start a war. Granary and stores fires hit a state already short of food; unrest stops at the revolt line, so spies alone never break a state.
- Spy vs spy: being hit pushes a computer state to post its own Spymaster first, which lowers the success and raises the risk of later operations against it.

## 6. Balance anchors

Numbers to compare against after any change (all-computer games, 60 states, land 100, difficulty 1, seeds 11–16 (11/12/13 for quick checks), observer mode, Rebellions on unless noted). `peasantRoom` on (the default) from 2026-11-13, measured with generals, relics and staff posts; the numbers with it off are in brackets. Refreshed with `keepPeople`, defense stances and raid/kidnap/massacre (2026-11-12); the old ranges had drifted with the map changes before them:

| Measure | Expected range |
|---|---|
| States alive at year 30 | 59–60 |
| States alive at year 90 | 44–54 (`peasantRoom` off: 38–52; Rebellions off: 35–48; old spoils rule: 3–36) |
| States alive at year 150 | 11–26 since the Spymaster (was 21–31; `peasantRoom` off: 6–35, earlier 8–18; Rebellions off: 5–20; old spoils rule: 1–9) |
| Biggest state's share of world land at year 150 | 13–19% (`peasantRoom` off: 12–28%; Rebellions off: 14–40%; old spoils rule: 37–100%) |
| Year 300 (seeds 11–16) | 2–5 states left, biggest 48–83% (seeds 11–13; `peasantRoom` off: 2–3 left, biggest 37–80%): games still end (Rebellions off, seeds 11–12: 2–4 left, biggest 59–64%) |
| Battles by mode (per game) | normal ~400, raid ~120–150, massacre ~13–15, kidnap ~3 |
| 1000 states, seeds 11–13, year 150 | 390–394 alive, biggest 1.4–1.7%, top 10 hold 12–13% (`peasantRoom` off: 233–264, 1.8–2.2%, 15–17%; earlier 129–157 alive, biggest 2–3%, top 10 17–21%), ~3,000 raids per game (old spoils rule: 38–44 alive, top 10 52–60%) |
| Games with one state left by year 300 (seeds 11–13) | 1 of 3 before raids (old spoils rule: 3 of 3); not re-run |
| Average gold at year 30 | 6.6–9.5k (`peasantRoom` off: 6.4–9.3k) |
| Average research levels (sum of areas) at year 150 | 99–101 (`peasantRoom` off: 96–106; earlier 103–109 with steady levels, ×1.8 cost; was 138–185 with capped levels) |
| Average research bonus per area, years 30–150 | within ~3 points of the capped version (e.g. year 150: sci 60%, agri 54%, atk 77%, def 75%) |
| Defense stances (share of state-years) | fall back 23–34%, stand ground 1–3% |
| Famine collapses in 150 years | 0 |
| Grain sold to merchants (share of world food use) | about 3% |
| States with an empty treasury | well under 0.1% of state-years |
| Turn time on a 1000-state map | about 200–250 ms |
| 1000 states, Rebellions on vs off (seeds 11 world, 12–13 random; 150 years) | 51–92 revolts per game (211 in all: by rebel states 1/2/3/4/5/6/7/8/9 = 63/40/28/38/13/11/10/5/3), almost all after year 60; 30–35% of the land lost per revolt; year 150: 217–312 alive (off: 138–242), biggest 1.8–2.1% (off: 2.2–2.7%), top 10 hold 15% (off: 19–20%); turns about 10% slower |
| Rebellions (60 states, seeds 11–16) | 0–8 revolts in 150 years (4/2/8/0/3/7), 0–21 rebel states; rebel states alive at the 30-year checkpoints: 0–4. Forced revolts (seeds 11–13, 96 revolts): rebel states per revolt 1–3 in ~70%, 9 in ~2%; about 30% of the land lost on average |
| `peasantRoom` on vs off (1000 states, seeds 11–13, Rebellions on, 150 years; 2026-11-13) | alive at year 90: 870–882 (off 789–800); year 150: 390–394 (off 233–264), biggest 1.4–1.7% (off 1.8–2.2%), top 10 hold 12–13% (off 15–17%); peasants at 91–98% of capacity from year 60 (off 58–78%), everyone 125–164% (off 88–98%); army share ~31% either way; gold year 150 +35–40% (114–124k vs 80–89k); research about the same. Conquest slows because armies no longer crowd out peasants and the trim stops wiping out heavily armed losers |
| Generals (Tavern and traits, seeds 11–16) | year 90: 170–195 generals in the world, about 80% Veterans, 13–18 Legendary (7–9%), 17–23 Academy graduates; year 150: mostly Academy graduates. Battles per 150-year game 540–670 (before the Tavern 470–710). With the first tier draft (Veteran skill 15–30) states alive at year 90 fell to 38–48, so starting skill was cut to the current ranges (year 90: 44–50; year 150: 12–19, one seed 33; biggest 13–30%; year 300, seeds 11–13: 2–3 left, biggest 52–77%) |
| Merchant (seeds 11–16, 150 years) | With computer buying only while a general has no relic and its Treasury holds fewer than 2: 211–307 relics in the world at year 150 (was 90–112), 53–125 carried; the rest pile up in Treasuries, mostly from conquered states. Unlimited buying (one of each offer a year) gave 600–750 and halved computer gold. States alive at year 150: 10–36 (avg 24); biggest 13–30%; battles 439–662; revolts 0–7 |
| Rolls and the Forge (seeds 11–16, 150 years) | 90–112 relics in the world at year 150, 44–69 carried (more than before: forged rares get used). States alive at year 150: 12–39 (avg 27, was 17–23 in the last runs; biggest 14–17%, was 15–24%): conquest a little slower, likely stronger relics on defenders plus seed noise; watch it. Battles per game 403–642; revolts 0–6 |
| Governor relics (seeds 11–16, 150 years) | Governors become common: 6–18 posted in the world at year 90 (was 1–7), 9–13 at year 150; 1–8 states have a Governor relic at work. States alive: year 90 38–52, year 150 10–31 (avg 17, was 20–23; within seed noise); battles per game 455–587; revolts 0–8 |
| Troop relics (seeds 11–16, 150 years) | 77–95 relics in the world at year 150, 30–59 carried; one famous troop relic turned up in ruins (seed 12). States alive at year 150: 10–42 (avg 23, was 20; within seed noise, the dice changed); battles per game 400–707; revolts 2–16 (two seeds above the earlier 1–5) |
| Legendary relics at the Merchant (seeds 11–13, 150 years; 2026-11-14) | States alive at year 150: 20, 15, 17 (was 23, 12, 19); biggest 16–33% (was 17–35%); relics in the world 279–301 (unchanged). Legendaries 47–62 at year 150 and 23–29 by year 90 (was 1–10 and 0): the Merchant becomes the main source, with no visible effect on conquest. |
| Four relics per general, Legendary (seeds 11–16, 150 years; 2026-11-14) | States alive at year 150: 14–25 (avg 18.8, the same as before in the same harness: 12–25, avg 18.8); biggest 16–27% (avg 21, was 18.5). 276–348 relics in the world (was 219–265), 159–240 carried (was 56–99), 25–37 generals with all 4 slots full; Legendaries 1–10 at year 150, none by year 90 (forged late, mostly from conquered Treasuries). Without the stricter computer Merchant rule and with Legendary ×1.5: avg 13.5 alive, 295–487 relics, so both are needed. With `relSlots` 1 the new code matches the old numbers (avg ~20.6). |
| Relics (seeds 11–16, 150 years) | 74–85 relics in the world at year 150, 25–56 carried (the rest wait in Treasuries); all 6 famous ones still held. States alive at year 150: 6–35 (avg 20, unchanged); battles per game 441–660; revolts 1–5. Year 300 (seeds 11–13): 3–4 left, biggest 30–75% |
| Staff posts (seeds 11–16, 150 years) | computer states fill 1–2 posts each by year 90 (mostly Chief of Staff and Drillmaster, Quartermaster when food is short, Governor rarely); 207–245 generals in the world at year 90. States alive: year 90 43–50 (unchanged), year 150 12–30 (avg 20, was 17.5 before posts: field skill and morale help defenders most); battles per game 476–688 (was 540–668); revolts 1–10 per game (was 4–15). Year 300 (seeds 11–13): 2–3 left, biggest 54–80%. Capping computer states at 5 generals instead of 6 changed nothing |
| Spymaster (seeds 11–16, 150 years; replaced the Chief of Staff, 2026-11-14) | 34–43 Spymasters posted at year 90, 14–25 at year 150 (Spymaster and Governor are now the usual posts, Drillmaster 6–12). 1,020–1,240 operations per game (~7 a year in the world): granary ~45%, research ~35%, gates ~10%, treasury ~5%, unrest and stores rare; ~63% succeed, ~29% found out. States alive: year 90 43–54 (avg 49, was 50.5 with the Chief of Staff), year 150 15–26 (avg 18, was 24); battles 505–692 (avg 570, was 543); revolts 0–4. With computer spying off (`aiSpyMin` 99): year 150 14–21 (avg 16.7), so the drop comes from losing the Chief of Staff's morale for field generals, not from spies |
| Poison a general (seeds 11–16, 150 years; 2026-11-14) | 55–77 poisonings per game (~6% of 980–1,160 operations), mostly against the defender of the computer's attack target; gates fell to 65–115. States alive: year 90 47–51, year 150 11–23 (avg 17, was 18); battles 511–760; revolts 0–10 |
| Starting land (any map up to 300 states) | every state within ±2% of the median |
| Creating a 1000-state world map | about 14 s |

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
7. **Performance**: avoid new full-map loops per state; up to 4000 states (New game limit; a 4000-state turn takes ~2 s).
8. **Simulate** (section 8) and compare with the anchors in section 6, before and after. Look at pacing, gold, research, famine, and the system's own numbers.
9. **Player-facing**: UI, the Rules & guide (`GUIDE` topics: its own topic, every topic that mentions it, and "How it all fits together"), a change log entry, and this document.
10. **Update this map** as described in section 10, in the same commit.

## 8. Simulating

Run the engine headless in Node to compare versions:

```js
// node sim.js path/to/index.html
const fs=require("fs"), h=fs.readFileSync(process.argv[2],"utf8"), src=h.match(/<script id="engine">([\s\S]*?)<\/script>/)[1];
const E=new Function("module",src+";return module.exports;")({exports:{}});
for(const seed of [11,12,13]){ let g=E.newGame({nStates:60,terr:100,difficulty:1,seed,observer:true}); // rebels:false to compare without Rebellions const out=[];
  for(let y=1;y<=150;y++){ g=E.endTurn(g)||g;
    if(y%30===0){ const A=g.countries.filter(c=>c.alive), avg=f=>A.reduce((s,c)=>s+f(c),0)/A.length;
      out.push(`y${y}: ${A.length} states, gold ${Math.round(avg(c=>c.money))}, research ${avg(c=>Object.values(c.lv).reduce((a,b)=>a+b,0)).toFixed(0)}, soldiers ${Math.round(avg(c=>c.soldiers))}`); } }
  console.log("seed",seed,out.join(" | ")); }
```

Compare against the previous version with `git show HEAD:index.html > old.html`. Use several seeds: single games vary a lot.

## 9. Known fragile spots

- The empty-treasury rule rewrites `c.work.sci`, including the player's setting.
- Tavern Legendary chance is per candidate, so a call's chance is about 3× it (Veteran ~6%, Academy ~17%); computer states end up with 7–9% Legendary generals. Lower `leg` or roll once per call if they should feel rarer.
- A Coward attacker who calls off leaves no battle report, only a note to its owner; the guards it would have drawn stayed home all along.
- Trait effects that depend on troop mix (troop traits, Pike Square) are not in the player's attack odds or the AI's estimate; luck, Siege Breaker, Mountain Fox, Butcher and Nemesis are.
- Troop relic bonuses, like troop traits, are not in attack odds or AI estimates (they depend on the army mix).
- Relics are rolled with the game dice too (candidates, spoils, ruins, `relSeed` at the end of `newGame`), so seeds differ from before relics. Relic effects are in the AI's estimates only where they go through `genFactor`, `genSpeed` or `luckDice` (`genEdge`); attack, loss and land multipliers are not.
- `rollGenTraits` uses the game dice (`R01`), so adding traits or tiers changes what every later roll of a seed gives.
- Rebels add states mid-game: anything that sizes per-state arrays once (instead of from `g.countries.length` each time) breaks. Caches keyed by adjacency rebuild after `ADJ.delete`.
- Computer states rarely revolt because they stop conquering near `aiLoyFloor`; lowering it gives more revolts (12: 2–8 per 300 years), raising it fewer.
- `workers()` floors each job, so rounding always adds farmers.
- Elimination thresholds differ: battle checks peasants < 20, famine collapse checks population < 20.
- An eliminated state's gold, wood and iron vanish only with `keepPeople` off; otherwise the winner takes them.
- `keepPeople` (with `peasantRoom` off): a beaten state whose soldiers nearly fill its remaining capacity loses its peasants to the trim and falls at once, even with land left; the winner then gets few or no people (only its stocks). Handing over the untrimmed people instead was tested: endgame much faster (states alive at year 150: 4/1/3), so it was dropped.
- Battle order is random; an earlier battle can make later ones on the same target do nothing.
- The AI research allocation is a fixed table; it ignores costs and wonders.
- AI difficulty scales only research (and AI thresholds), not the economy.
- The `R` comment on `aiSellFrac` says 30% but the value is 0.5.
- The `placeWonders` comment says six wonders; ten are placed.
- **Arms race in peace** (left as is, by choice): a computer state's threat is its strongest neighbor's *whole* army (relations shift it at most ±40%), and both the strong-neighbor minimum (up to 20% of people) and the defense top-up (up to 30% in peace) follow it, so armed neighborhoods keep each other at ~25–33% army even in peace. Halving peaceful neighbors' threat (`aiPeaceThreat` 0.5) left states under-defended and sped conquest up a lot. Candidate fixes: count only the share of a neighbor's army that could reach us (split across its fronts or by shared border), and count our defender bonus (+15%) and narrow-front/sea bonuses in the defense target.

## 10. Keeping this map current

This map is only useful if it matches the code. Update it in the same commit as the change.

**When a new system is added:**
- Add its own entry to section 3 (intent, what it is, key rules and `R` values, reads, writes, notes).
- Add it to every existing entry it reads from or writes to: their **Reads**/**Writes** lines must name it too, so each relationship is recorded on both sides.
- Add a row and a column to the table in section 4.
- Add it to the diagram in section 1 if it moves people, food, gold, goods or land.
- Place it in the yearly order in section 2.
- Add any loop it creates or brakes to section 5.
- Add an anchor to section 6 if it has a number worth watching, and re-run the simulation to refresh the existing ranges if they moved.

**When an existing system changes** (a formula, an input, an output, when it runs, or what the AI does with it):
- Update its entry in section 3, including its intent if the change alters what the system is for (confirm with the designer first).
- Check every system that lists it under **Reads** or **Writes**, and update those entries if the relationship changed (new link, removed link, or a link that got much stronger or weaker).
- Update the strength marks in section 4, the yearly order in section 2 and the loops in section 5 where they are affected.
- If balance moved, re-run the simulation and update section 6. Say in the commit why the new ranges are acceptable.

**Always** update the player's Rules & guide (`GUIDE` in the UI script) in the same commit: it must explain each system fully and how the systems balance each other, in plain words.

**When a fragile spot is fixed**, remove it from section 9. When a new one is found, add it.

## 11. Open design questions

Intents the code can't answer. When one is decided, write the answer into the system's intent (marked *(your direction)*) and remove it here.

1. **Troop roles**: confirm wood troops = cheap and fragile for populous, fast-growing states, and iron troops = elite and people-efficient for states short on people.
2. **Victory**: should a peaceful research or economy strategy be able to win (or at least survive to the end), or is conquest the only path?
3. **Weather**: should it reward states that keep big grain stores? Should a drought make computer states more likely to attack a fertile neighbour?
4. **Alliances**: should they stay fixed, or form and break during play (relations could drive it)?
5. **Balance of power**: fear (and so fear wars) is measured by land only; should it use overall power (population, army, research)? Should alliances count as one bloc?
6. **Difficulty**: should harder levels give computer states economic bonuses, or only smarter and bolder play?
7. **Market**: should relations block trade later (hostile states can't buy from each other)? Is merchants paying 0.9× normal for grain the intended income level for farming states?
8. **Research**: should computer states adapt their research to their situation (costs, wonders, threats) instead of a fixed table?

## 12a. Planned: more for generals (designed, not built yet)

Designed with the player in mock-ups; the Tavern, cards and traits (3.11), staff posts (3.11b) and relics (3.11c, Treasury as a Generals sub-page) are built; the command limit became a flat `genMax` 10 and wages were dropped.
- Retire with honours after 10 years of service (a Hall of past generals; needs a new cost now that wages are gone).
- Later: death and capture in battle (Lucky Bastard never falls), rivals (Rival trait), epithets, banners on the map.

## 12. Planned: war and peace (designed, not built yet)

Agreed with the designer *(your direction)*. When a part is built, move it into section 3 (and the table, yearly order and loops) and delete it here.

**Why:** today every attack is a one-off, the computer's army size follows the calendar and personality, and research ignores war. Wars give attacks a context, and war footing makes states fill their land in peace and spend people in war.

### 12.1 State wars — **built** (see 3.14b); kept here for the record
- **Start** (either is enough):
  1. **Hostility:** either state's relation toward the other reaches **−50**. All factors count, fear included (a war of fear against a runaway state).
  2. **Computer declaration:** each year, while planning attacks, the computer scores every neighbor: *motive × feasibility − cost*, and declares on the best one above a threshold.
     - Motive: grudge (grievance, solidarity); fear of a runaway; land hunger (≥ ~85% full and no unowned land to settle); a wonder they hold; warlike leaders count motive higher.
     - Feasibility: its attack against their estimated defense (the edge it needs for attacks today); an army free to use (not tied down, home not exposed).
     - Cost: trust with them; their allies (once alliance wars exist).
- **Limits on starting wars:** a state never declares while it is in a state war, while its alliance is in an alliance war, during the game's peace years, or within **5 rest years** after a war ended. It can still be declared on, so it can be in several wars, never by its own choice. States already at war **can** be declared on.
- **During a war:** a state at war attacks only its enemies (no side raids).
- **End:** each war lasts a random **8–15 years**, or until one side is eliminated. At the end: grievance, solidarity and wariness between the two reset to 0 (fear stays, it reflects size); 5 rest years; news "Peace between X and Y".
- **The player:** no declaring war or offering peace. The player takes part through normal play: their attacks can start a war through the victim's hostility, and computer states can declare on them.

### 12.2 War footing (computer states) — **built** (see 3.14c); kept here for the record
- Level **0 = peace, 1 = in any war** (state or alliance); moves 40% a year toward that (2–3 years to mobilise or stand down). Values below blend with the level.

| | Peace (0) | War (1) |
|---|---|---|
| Army, warlike persona | 12% of population | 35% |
| Army, other persona | 6% | 25% |
| Late-game bonus (up to +10% over time) | as today | as today |
| Strong-neighbor defense minimum | as today | as today |
| Peasant reserve (`aiKeepPop`) | 80% (fill the land) | 65% (mobilise) |
| Spare gold | science and reserve | soldiers (as today) |
| Density research share | 14% | 4% |
| Growth research share | 6% | 18% |

- Attack and defense research keep today's table; the density/growth shift is taken from or given to science.
- Starting values: simulate against section 6 and tune.

### 12.2b Interior states (no non-allied neighbors)
- A computer state whose only neighbors are allies (sea routes count as borders) has no one to fight and no land to conquer *(your direction)*:
  - **Army:** shrinks to a token guard, about **2% of population**, dismissing about 20% of the surplus a year (soldiers return to the fields).
  - **Work and research:** pushes research: the scientist cap rises (as with surplus, up to +15%), and research leans hard to **density** (its only way to grow), then science.
  - **Spare gold:** science, not soldiers.
- **Back to normal** as soon as a non-allied neighbor appears (borders changed, an ally fell, or a new sea route): it uses the normal peace or war values at once and rebuilds its army at the war-footing pace.
- Unowned land next to it doesn't change this (settling is automatic); it is still an interior state.

### 12.3 Balance of power — **built** (see 3.16)
- Fear wars against a runaway state **replace the coalition mechanic** (section 3.16); remove `coalition*` rules and hegemon targeting once state wars exist.

### 12.4 Trade embargo — **built** (see 3.7)
- States at war with each other can't fill each other's market orders (they don't see each other's orders). Merchants still trade with both.

### 12.5 Alliance relations and alliance wars — **built** (see 3.15; hit sizes tuned down to −1 / −0.3 / −0.6)
- **Alliance relation:** −100..+100 for each pair of alliances, each direction, **starting at 0**.
  - Yearly target: how members feel about the other alliance's members, averaged over neighboring pairs and weighted by land; alliances with no shared border drift toward 0. No fear at this level.
  - Moves **10% a year** toward the target (slow).
  - Direct hits on top: **−3** per year a state war runs between members; **−1** per battle between members (**−2** if more than 10% of the defender's land is taken). So state wars between members push their alliances toward war.
- **Alliance war:** starts when an alliance relation reaches **−40**. An alliance never declares while already in an alliance war; members' state wars don't hold it back.
  - During it, every member is at war with every member of the enemy alliance (targeting and war footing). State wars already running continue; members can't start new state wars.
  - Ends like state wars: random 8–15 years (or one alliance eliminated); the alliance relation resets to 0; members' own state relations stay; 5 rest years.

### 12.6 Showing wars — **built** (see 3.14b)
- War tab: "At war with X" (since year, years left hidden or shown — decide when building).
- Map tooltip: a war mark on neighbors at war.
- A "Wars" map view.
- News when wars start and end.

### 12.7 Single-alliance view — **built** (see 3.15); not yet opened from map labels (canvas text isn't clickable)
- Opened from an alliance's tag on map labels, its row in the Alliances tab, or the map tooltip.
- Header (colour, name, tag, members left/fallen); stats (land, population, army attack/defense, average research, share of the world); member list (sortable, click selects on the map); relations with other alliances (bars, both directions); active wars and recent battles; "Show on map".

### 12.9 Allied aid — **built** (see 3.15)
- Allies can send each other **grain, wood, iron and gold** *(your direction)*. Troops are left for later (they need army-movement rules).
- **Who gives (computer):** interior states and allies at peace, from surplus only: grain above their reserve, wood/iron above the 3-year reserve, gold above a cushion. At most **50% of the surplus a year**, so a donor never weakens itself. Aid comes **before** selling to the market, so surplus goes to the front instead of to merchants.
- **Who receives:** allies at war or bordering an enemy that are short of grain, of wood/iron for troops, or of gold. Neediest first, front-line states ahead of others.
- **When:** once a year, in the computer's turn, before its market trading; aid arrives at once.
- **The player:**
  - Sends aid from the single-alliance view: a "Send aid" form (ally, resource, amount).
  - Receives aid from allied computer states when at war or on the front line, like any ally.
  - News both ways ("Realm of X sent you 2,000 grain").
  - Hot seat: humans on one team can send each other aid.
- **Effects:** aid raises the receiver's trust toward the giver (later: can feed alliance strength).

### 12.8 Build order
1. ~~State wars~~ — built.
2. ~~War footing and interior states~~ — built.
3. ~~Showing wars~~ — built.
4. ~~Remove the coalition mechanic~~ — done.
5. ~~Trade embargo~~ — built.
6. ~~Alliance relations and alliance wars~~ — built.
7. ~~Single-alliance view and allied aid~~ — built. **The war and peace plan is complete.**
