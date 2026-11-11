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
                                                   fear of a giant → wars (balance of power)
```

**The one resource everything competes for is people.** Each peasant is either a farmer, woodcutter, miner or scientist (the work split), or is turned into a soldier. Only peasants pay tax and have children. So every system that uses people (research, army, production) takes from the others and from growth. Most balance questions come down to "what is one person worth in this job vs that one?"

---

## 2. One year, in order

The order matters: it decides which systems see this year's values and which see last year's.

**Player's turn (UI, instant).** `act.*` changes `G` at once: hire/dismiss troops, buy/dismiss generals, trade now or post/cancel orders. The work split, research allocation, attack orders and defending general are only stored. The player's market trades land in the current market year at the merchant prices set last year.

**End turn (`endTurnNow`).** Auto-sell posts surplus grain at the market price for one year. Then `ENGINE.call("endTurn", G)` runs:

1. Save migrations; logs reset; `g.avgLv` (world research average, used for catch-up pricing); `c.mod` recomputed for every state (settings changes apply).
2. **AI turns** (`aiTurn`), in state-id order, **before the harvest**: work split, market (trades happen now, so later ids see a changed book), generals, recruiting, research allocation, attack orders. AI reads last year's relations, wars, market averages, and `c.wxSys` (next harvest's weather systems, rolled last year).
3. **Battles**: attacks filtered (alive, general exists, adjacent, not allied, not peace time); attackers taken out of the army; the rest are guards. All attack strengths (with luck) computed first, then battles resolved one by one in shuffled order. Land, peasants and grain move; relations updated per battle (`relOnAttack`).
4. `settle` (unowned land claimed), `computeShore`, `refreshWonders`.
5. **Economy** per state (`economy`): harvest (× this year's weather) → eating → famine → spoilage → wood/iron → taxes − wages → empty treasury → births (none in a famine year) → research → generals age/train/recover → collapse below `minPop`.
6. **Market year** (`mkYear`): orders age and expire, the market price is updated, merchant prices reset, merchants take orders that beat them (`mkSweep`).
7. **Weather** (`mkWeather`): systems drift/merge/clash/spawn; next year's `c.wx` and `c.wxSys` are rolled.
8. **Relations year** (`relYear`), **wars** (`warYear`: wars that ran their time end, new ones start from hostility), history, `turn++`, victory check. Computer declarations happen earlier, in step 2 (AI turns).

Consequences:
- Conquests, settling, shore and wonders are applied **before** the economy: new land and captured peasants farm, eat and pay tax the same year.
- Weather is rolled a year ahead, so the AI and the player can see droughts coming (`c.wxSys`, "Weather ahead"). The random patches part is not shown ahead.
- Troops hired this turn fight this turn. Troops sent to attack do not defend this turn.

---

## 3. The systems

Each entry: **intent · what it is · key rules · reads · writes · notes**. The intent says what the system is for and which choices it should create. *(inferred)* marks intents read from the code and comments that the designer has not confirmed yet; *(your direction)* marks the designer's own decisions. When a change conflicts with an intent, ask before changing either.

### 3.1 Population
- **Intent**: People are the core currency. Growth rewards peace and room, and capacity is what makes land valuable: a state that stops growing has to expand (settle, conquer) or research density. Overcrowding is meant to hurt (births turn negative). Growth bonuses are kept small because people compound. *(inferred)*
- Population = peasants + soldiers (scientists are peasants with a job; `c.sci` is a legacy field, always 0).
- Capacity: `popCap = land × (popPerKm 3 + densPerLevel 0.15·lvl(dens)) × M(capacity) × wonder(dens)`.
- Births: `(peasants + armyBirths 0.5 × soldiers) × growthRate × roomFactor(pop/cap)`, `growthRate = (birthRate 0.03 + growPerLevel·lvl(grow)) × M(births) × wonder(grow)`. `roomFactor(x) = 1 − x^birthCurve` (24): births stay near full speed until ~80% full, peak near 87%, stop at capacity; above capacity `1 − x` (a gentle decline) *(your direction: growth should not slow from half full)*. None in a famine year.
- Collapse below `minPop` 20 (land becomes unowned). In battle a state is also absorbed if it has under 20 peasants.
- **Reads**: land (battle, settle), dens/grow research, identity, famine, battle losses and captures.
- **Writes**: taxes, food use, every job, research cost (size factor), AI decisions.
- Notes: peasants drive births, taxes and all jobs, so population compounds. That is why housing and growth traits apply at 40% (`traitWeight`).

### 3.2 Work split
- **Intent**: The main economic choice every year: every scientist or woodcutter is one farmer less, so no job is free. It should support distinct styles: farmer-exporter, industrial (wood/iron for an army or for sale), scholar. At least 10% always farm so a state can't starve itself by one slider. *(inferred)*
- `c.work = {wood, iron, sci}` as % of peasants; farmers are the rest. `workers()` floors each job, so rounding goes to farming.
- Player: Overview tab sliders; each job ≤ 80%, the three together ≤ 90% (at least 10% farm); locks.
- Output per worker per year: farmer `foodPerFarmer` (3 base), woodcutter 2.8 wood, miner 2 iron, scientist 1 research point and costs `sciWage` 3.5 gold.
- AI: food first with a margin, then scientists (by phase, what it can pay, `aiSciMax`), then wood vs iron leaning to its better yield. Surplus logic (`aiGlut*`, `sellAvg`): steady selling or piling up means it farms/digs less and moves hands to research, as far as it can pay.
- **Reads**: population, research yields, identity, market prices (AI), stores, weather ahead (AI).
- **Writes**: food, wood, iron, research points, wages.
- Notes: the empty-treasury rule rewrites `c.work.sci` directly, including the player's.

### 3.3 Food
- **Intent**: The survival floor and the first constraint. Surplus becomes gold through the market; a store buffers bad years, but spoilage (10%/yr) makes hoarding a deliberate insurance choice, not free. Famine kills soldiers first, so an army bigger than the land can feed falls apart. *(inferred)*
- `foodPerFarmer = 3 × (1 + agriPerLevel·lvl(agri)) × (1 + shoreBonus) × M(food) × wonder(agri)`; shore bonus = share of land on the shore, max +25%.
- Harvest = farmers × foodPerFarmer × `c.wx` × `g.clim` (weather).
- Use: 1 per peasant; troops eat their `upFood` (2; axe 3; cavalry 5).
- Shortfall: `deficit × famineDeath 0.5` die, soldiers weighted ×1.5. Then the store spoils `foodSpoil` 10%.
- **Reads**: work split, agri research, shore (land/borders), identity, weather, troop mix, market buys, loot.
- **Writes**: famine deaths, births (blocked by famine), market supply/demand, AI work split, `projectedNet`.

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
- Taxes: `peasants × taxPerPeasant 0.6 × (1 + finTaxPerLevel·lvl(fin)) × wonder(fin)`.
- Wages: scientists × 3.5 + troops × `upGold` (only cavalry, 6).
- Empty treasury: debt forgiven, but `ceil(−money/2)` soldiers go home, then scientists are cut.
- Sinks: troop gold cost (cavalry), generals (`genPrice`, mind is exponential), market buys and buy orders (gold held). Sources: taxes, market sales (Finance adds +1.5%/level to what sellers get).
- **Reads**: population, fin research, market.
- **Writes**: research (can I pay scientists?), army (cavalry, generals, buying wood/iron), AI decisions (`projectedNet`, money dump into soldiers above `6·wages + 3000`).
- Notes: one scientist costs about six peasants' tax. That ratio is the main brake on research.

### 3.6 Wood and iron
- **Intent**: The military-industrial base. Wood is for mass troops, iron for elite ones. Leaders and land types skew yields so states can specialise and trade what they're good at. *(inferred)*
- `production`: woodcutters × 2.8 (miners × 2) × M(wood/iron) × (1 + 0.06·lvl) × wonder. No spoilage, no upkeep.
- Used only to train troops (wood troops: spear, archer, axe; iron troops: pike, sword, cavalry) and on the market.
- **Reads**: work split, research, identity, wonders.
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
- **Intent**: Long-term investment that competes with the army for people and gold. Caps make the late game converge so a tech lead can't become permanent; catch-up pricing keeps laggards in the game; size-scaled cost stops big empires out-researching by scale alone. *(inferred)*
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
- **Intent**: A scarce asset that grows with use. Mind (learning) is expensive but pays off over a long career; speed decides how much land a win can yield; lost battles hurt morale, so choosing battles matters. Needing a general per attack limits how many wars a state runs at once. *(inferred)*
- Needed to attack; one idle general defends. Price `30 × 2.6^mind × (1 + speed/8) × (1 + (40 − age)/40)`.
- Battle factor `(0.4 + skill·0.008) × (0.5 + morale/200)`: 0.4 fresh, up to 1.2. No defending general: 0.3.
- Speed caps land taken per battle (`10 + speed·2.5` km²). Skill grows with battles and yearly training; morale drops after battles, recovers 10/year; retire at 62–76.
- **Reads**: gold. **Writes**: battle strength, land taken.

### 3.12 Battle
- **Intent**: Defending is easier: +15%, narrow passes and sea crossings, and ties go to the defender, so attacking needs a clear edge. Narrow wins take little land; crushing wins take a lot. Sending troops away leaves home open the same year, so timing and home guard are real trade-offs. Conquest captures people and grain but not the treasury. *(inferred)*
- Attack strength = troops' attack × capped tech × M(attack) × wonder × general × luck (±15%).
- Defense = guards' defense (troops not sent this year, split between attackers by their strength) × capped tech × M(defense) × wonder × general × 1.15 × (1 + front bonus) × luck. Front bonus: +40% on a border ≤ 3 km, fading to 0 at 15 km; +15% for sea-only fronts.
- Losses: attacker `0.7 × strD/(strA+strD)` (×1.2 if it lost), defender `0.7 × strA/(strA+strD)`, each ÷ toughness. No guards: attacker loses 2%.
- Win if strA > strD. Decisiveness 25% at a narrow win, 100% at 2×.
- Land taken = `min((speedCap + 2√survivors) × exploreMul × decisiveness, survivors × holds)`, at least 1.
- Captures: the share of land taken moves the same share of peasants (80% arrive) and half the same share of grain. Gold, wood and iron are not looted. An eliminated state's stocks vanish.
- **Reads**: troops, generals, atk/def/explore research, identity, wonders, map fronts, AI/player orders.
- **Writes**: land (→ capacity, production, shore, wonders), population on both sides, research cost (size), relations, general skill/morale, AI memory (`lastWar`, `lostRecent`, `sentShare`, `openTurn`).

### 3.13 Territory and map
- **Intent**: Land is the prize. Unowned land at the start is a race; coasts are worth more (food). Collapsed states' land goes to whoever settles it first. *(inferred)*
- Cells: water, unowned or owned; adjacency = 4-neighbours plus sea routes (world map). Allies never count as targets.
- Starting sizes: on the world map, starting states are evened out to within `startGap` 50 km² of each other (oversized states hand outer land to smaller neighbours or leave it unowned; undersized ones claim unowned or spare neighbour land; squeezed ones restart in the biggest unowned pocket).
- Settling: each state bordering unowned land claims `(5 + 2% of its land) × exploreMul` a year. Unowned land comes from the map start and from collapsed states.
- Shore: land cells next to water → harvest bonus. Land type is fixed at start even if the coast is lost.
- **Reads**: battles, collapses, explore research. **Writes**: capacity, production (shore), wonders, fronts, adjacency.

### 3.14 Relations
- **Intent**: Aggression has memory and reaches beyond the victim: neighbours grow wary, allies hold solidarity, peace slowly builds trust. Revenge is understandable (a strike back counts half). The world should react to the player believably, not randomly. *(inferred from comments)*
- How each state feels about each other one, as a sum of factors (−100..100): grievance (it attacked us), solidarity (it attacked our ally), wariness (it attacked our neighbour), fear (it is much bigger), trust (years of peace), common enemy. Each has its own build-up and fade (`rg*`, `rs*`, `rw*`, `rf*`, `rt*`, `re*`). A strike back within 5 years counts half.
- **Reads**: battles, land sizes, adjacency, alliances.
- **Writes**: AI threat estimate, army size, home guard, target choice, edge needed, revenge chances, protector attacks; player-facing labels, map view, warnings, and the map tooltip (each state's feelings toward its neighbours and theirs toward it, worst first).
- Notes: trust builds on every quiet border and makes the AI pickier; grudges can spiral (damped by revenge clearing grievance, floors and cooldowns).

### 3.14b Wars
- **Intent**: attacks happen in a context: single raids stay incidents, repeated or heavy blows, a grudge, fear of a giant or a full state with nowhere to grow lead to war, and wars end so old grudges can be forgotten *(your direction)*.
- A war between two states (`g.wars`: `{a, b, from, to, why}`) starts when **either** state's relation toward the other reaches `warHostile` −50 (all factors, fear included), or when a **computer state declares** one (`warScore`: motive × feasibility − cost above `warDeclare` 0.5; motive = grudge + ½ solidarity + fear + land hunger (≥85% full, nothing settled for 2 years) + wonders, scaled by the leader's warlikeness; feasibility = its sendable attack ÷ the edge it needs; cost = trust). Declaring costs −`warDeclareHit` 15 grievance with the victim and wariness with the declarer's other neighbors.
- A computer state declares only when it has no war and no rest years left; it can be declared on (and drawn into hostility wars) any time. While at war it attacks only its enemies.
- Each war lasts `warMin`–`warMax` (8–15) years or until a side is eliminated. At the end grievance, solidarity and wariness between the two reset (fear stays); both get `warRest` 5 years without declaring, and the pair can't go to war again in that time.
- No wars during the game's peace years. The player can't declare war; their attacks can start one through the victim's hostility, and computer states can declare on them.
- **Reads**: relations (all factors), battles (through relations), land and capacity (land hunger), settling (`c.settleT`), army strength, leaders, wonders.
- **Writes**: AI target choice (enemies only while at war), relations (declaration hit, reset at peace), news (year report to both sides, and to humans bordering or allied to either side).
- **Player sees**: War tab "Wars" section (each war, since when, how it started; count of other known wars), ⚔️ on enemies in the neighbors list and in the map tooltip, a "Wars" map view (your enemies red, states at war orange, at peace grey).
- Notes: in 60-state games, about 100–120 wars per 150 years; most start from hostility, 3–16 are declared; states are at war ~30% of the time; ~2/3 of battles happen inside wars; 30–50% of wars start mostly from fear (the planned replacement of the coalition mechanic).

### 3.14c War footing (computer states)
- **Intent**: in peace a state fills its land (small army, density research); at war it spends people and refills them (bigger army, growth research) *(your direction)*.
- `c.footing` 0..1 moves `aiFootStep` 40% a year toward 1 (in any war) or 0.
- Army target share: `aiArmyPeace` 6% (warlike `aiArmyPeaceMil` 12%) → `aiArmyWar` 25% (35%) by footing, plus the late-game bonus (up to +10%), grudge extra; still at least the strong-neighbor minimum (half the strongest neighbor's attack, up to 20% of people). Defense top-ups arm at most `aiDefCapPeace` 30% → `aiDefCapWar` 40% of people.
- At peace (footing < 0.2) a state sends `aiPeaceCut` 10% of its soldiers above 1.25× its target home each year.
- Peasant reserve `aiKeepPeace` 80% → `aiKeepWar` 65% of people.
- Spare gold buys soldiers only in proportion to footing; in peace it stays (the gold-hoard rule moves hands to research).
- Research: density `aiDensPeace` 14 → `aiDensWar` 4, growth `aiGrowPeace` 6 → `aiGrowWar` 18; the difference comes from science.
- **Fullness and attacks** (not at war) *(your direction)*: below `aiAtkRoom` 70% full a state attacks only out of a grudge, fear or for a wonder; from 70% to `aiAtkFull` 90% the edge it needs falls from +`aiAtkRoomEdge` 50% to normal. At war, fullness doesn't matter. Effect: attacks outside wars by states under 70% full fell from 49% to 17%; conquest slower in mid-game (1000 states: 741 left at year 60 instead of 547), endgame similar (48 vs 41 at year 150).
- **Interior states** (only allied neighbors, sea routes count): a token army of `aiInteriorArmy` 2%, sending `aiInteriorCut` 20% of the surplus home a year; scientist cap +15%; research sci 35, density 30, agri 10, fin 10, growth 5, wood 5, iron 5; no spare gold into soldiers. Back to normal as soon as a non-allied neighbor appears.
- **Reads**: wars, neighbors and their armies (threat), relations, population and capacity, gold.
- **Writes**: army size, peasants (births, work), research allocation, scientists.
- Notes: armies react to neighbors' armies (an arms race): peaceful states still sit around 20–30% army by mid-game next to armed neighbors. Discounting peaceful neighbors' threat (`aiPeaceThreat` < 1) lowered peace armies but left states under-defended and conquest got much faster, so it stays 1. Effect in simulation: early conquest slower (60 states: ~50 left at year 30 instead of ~40; 1000 states: 854 left at year 30 instead of 686), states fuller (≈99% at year 60), endgame unchanged.

### 3.15 Alliances
- **Intent**: team scenarios with clear, fixed sides; alliances have their own slow-moving relations and wars on a separate layer from state wars *(your direction)*.
- Fixed teams chosen at setup (by count or size, regional or scattered). Allies never attack each other and win together. Solidarity links allies' feelings. No forming or breaking during play.
- **Alliance relations** (`g.arel["i>j"]`, −100..100, starting at 0): each year move `arStep` 10% toward how members feel about the other alliance's members (averaged over bordering pairs, weighted by land; no shared border: toward 0), plus direct hits: −`arWar` 1 per year of each state war between members, −`arBattle` 0.3 per battle between members (−`arBattleBig` 0.6 if over 10% of the defender's land is taken). No fear at this level.
- **Alliance wars** (`g.awars`: `{a, b, from, to}` between team indexes): start when an alliance's relation toward another reaches `awarHostile` −40 and that alliance isn't already in an alliance war (members' state wars don't hold it back). Every member is then at war with every member of the other (`atWar`, `inWar`, `foeTest` cover both kinds of war): embargo, computer targets and war footing, ⚔️ marks and the Wars map. State wars already running continue; members don't declare new state wars. Ends after `warMin`–`warMax` years or when one alliance is gone: the alliance relation resets to 0 (members' state relations stay), `warRest` years of rest for the pair.
- **Player sees**: War tab lists alliance wars and an "Alliance relations" table (both directions, bars); news to the members of both alliances when an alliance war starts or ends.
- **Reads**: members' relations, adjacency, land, state wars and battles between members. **Writes**: wars (through `atWar`), AI targets and footing, market (embargo), news.
- Notes: in 60-state, 4-alliance games, 4–7 alliance wars per game; relations bottom out around −70..−97. With hits of −3/−1/−2 alliance wars were near-constant (13–17 per game) because battles between many members add up.
- **Allied aid** *(your direction)*: allies joined over allied land, directly bordering or through a chain of bordering allies (sea links count, `allyGroup`/`aidReach`), send each other grain, wood, iron or gold (`aidSend`), arriving at once; no aid across the world (an ally with no allied neighbor gets none); the receiver's trust toward the giver rises (`aidTrust` 1 per 1,000 gold of value, at most `aidTrustMax` 5 a gift). Computer allies at peace (or interior) give up to `aiAidShare` 50% of their surplus a year (grain above 1.3 years of food incl. this harvest, wood/iron above the 3-year reserve, gold above 6 years of wages + 2,000) to reachable allies at war that are short, neediest first, before market trading. The player sends aid from their alliance's window. In 60-state, 4-alliance games: ~800–1,300 gifts per game, mostly iron.
- **Alliance window** (`allianceModal`): opened from the Alliances tab, the War tab (alliance wars, alliance relations) or the alliance line of a selected state: stats (states, land and world share, population, army, average research), alliance wars and members' state wars, relations with the other alliances, members (click to select), Send aid (your own alliance), Show on map.
- Notes: states deep inside an alliance block can have no enemy neighbours (interior states, 3.14c).

### 3.16 Balance of power (through fear wars)
- **Intent**: stop a runaway winner, the player included: the world pushes back on whoever gets too big *(your direction: fear wars replaced the old coalition)*.
- No separate mechanic any more: neighbors fear a state from 1.5× their size (relations, fear factor up to −50), and fear counts toward the −50 war trigger (3.14b), so a runaway state ends up at war with the neighbors it frightens. Fear also lowers the edge a computer state needs against a target others are already attacking.
- Notes: compared with the removed coalition, runaway states grow a little faster mid-game (in 2 of 4 test games the leader held 18% vs 8% and 30% vs 24% of the land at year 70) and fight fewer wars at once; the endgame is similar. Fear's strength (`rfFrom`, `rfPer`, `rfMax`) and `warHostile` are the knobs if it needs to be stronger.

### 3.17 Computer players (AI)
- **Intent**: Play by the same rules as the player: no free resources (difficulty only scales research and how bold it is). Personalities differ so each world plays differently. The AI should use every lever the player has. *(inferred)*
- Runs every system above for computer states: work split, market, generals, recruiting, defense, research allocation, attacks. Personality from the leader (`mil`, `guard`), a per-state random lean (`kk`), and difficulty (research ×0.8/1/1.15, earlier war phase, smaller attack edge needed `[1.7, 1.35, 1.2]`).
- Attack decision: estimate the target's guard (with noise), apply relations, wars and exposure, send enough to win by the edge and hold the land, keep a home guard. At most 2 attacks a year; a stalemate breaker makes it bolder after 5–25 idle years.
- **Every new system needs an AI side.** If the AI ignores a lever the player has, the player gets a free edge and the world drifts away from the player's experience.

### 3.18 Setup and difficulty
- **Intent**: Variety and replay value; difficulty mostly changes how bold and how fast-researching computer states are. *(inferred)*
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
| AI | ● | ● | | | | ● | ● | ● | ● | | | |

## 5. Feedback loops

Positive (snowball) loops, and what brakes them:
- **Land → capacity → peasants → taxes, food, workers → army → more land.** Brakes: births stop at capacity, size-scaled research cost, captured peasants arrive at 80%, fear wars against a runaway state, fear and wariness, general speed and holding limits on land per battle.
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
10. **Update this map** as described in section 10, in the same commit.

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
- The AI research allocation is a fixed table; it ignores costs, caps and wonders.
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

**When a fragile spot is fixed**, remove it from section 9. When a new one is found, add it.

## 11. Open design questions

Intents the code can't answer. When one is decided, write the answer into the system's intent (marked *(your direction)*) and remove it here.

1. **Troop roles**: confirm wood troops = cheap and fragile for populous, fast-growing states, and iron troops = elite and people-efficient for states short on people.
2. **Victory**: should a peaceful research or economy strategy be able to win (or at least survive to the end), or is conquest the only path?
3. **Weather**: should it reward states that keep big grain stores? Should a drought make computer states more likely to attack a fertile neighbour?
4. **Alliances**: should they stay fixed, or form and break during play (relations could drive it)?
5. **Conquest loot**: should a defeated or eliminated state's gold, wood and iron go to the winner instead of vanishing?
6. **Balance of power**: fear (and so fear wars) is measured by land only; should it use overall power (population, army, research)? Should alliances count as one bloc?
7. **Difficulty**: should harder levels give computer states economic bonuses, or only smarter and bolder play?
8. **Market**: should relations block trade later (hostile states can't buy from each other)? Is merchants paying 0.9× normal for grain the intended income level for farming states?
9. **Research**: should computer states adapt their research to their situation (costs, caps, wonders, threats) instead of a fixed table?

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
