# IRONHORIZON
## Master Development Prompt — A 3D Open-World Low-Poly Train Simulation & Delivery Game

**Document version:** 1.0
**Codename / repo slug:** `ironhorizon`
**Status:** Implementation-ready. Self-contained. Hand directly to a coding AI or a development team.
**Companion files expected to be created in-repo:** `docs/PROGRESS.md`, `docs/OPEN_QUESTIONS.md`, `docs/DECISIONS.md`, `content/*.json`

---

# 0. READ FIRST — Operating Instructions for the Coding AI

You are the lead engineer and technical designer for `IRONHORIZON`. You will build this game incrementally, over many sessions, in the phase order defined in §10. There is no prior context you are missing — this document is complete. Read it fully before writing a single line of code, then begin at Phase 1.

### 0.1 The ten rules of engagement

1. **One phase at a time.** Never begin a phase until the previous phase's Definition of Done (§10) is demonstrably met, and that evidence is written into `docs/PROGRESS.md` (what was built, how it was verified, what was cut).
2. **The simulation core is engine-agnostic and deterministic.** All code that mutates simulation state lives in `packages/sim/` and **must not import** rendering, input, audio, or UI libraries. Input arrives as an immutable `InputFrame` object. Time advances in fixed steps of `dt = 1/60 s`. Same seed + same input sequence = identical world state, always. This is not optional: it powers saves, automated tests, balance simulation, and any future co-op.
3. **Data over code.** Anything a designer would tune — payouts, drag coefficients, weather weights, palettes, dialogue, station names, speed limits — lives in `content/*.json`. Adding content must never require touching a `switch` statement.
4. **Every phase ends with a runnable build, a phase note, and tests.** Each new simulation rule gets at least one automated test. Prefer "golden run" regression tests (replay a fixed input sequence, assert on final state hashes and key numbers) over brittle unit tests of internals.
5. **Never silently invent an answer.** If this document is ambiguous, implement the simplest option consistent with the Design Pillars (§0.2), leave a `// DECISION-NEEDED: Q<n>` comment referencing the question ID in §11, and log it in `docs/OPEN_QUESTIONS.md`. Do not block.
6. **Performance budgets are contracts.** The budgets in §9.7 are hard limits. A phase that breaks the frame budget is not done.
7. **No premature polish.** Use flat debug colors and untextured primitives until Phase 25 unless a phase explicitly requires art. Placeholder art that looks "finished" prevents honest evaluation later.
8. **No hidden state.** Anything the game tracks must be visible somewhere (HUD, debug overlay, or a data dump). If you can't inspect it, you can't balance it.
9. **Telegraph before you punish.** Every systemic hazard must emit a readable signal 5–15 seconds before it can hurt the player.
10. **Ship the small loop first.** A player who can complete one satisfying 10-minute haul in a 2 km world has more valuable feedback than a player who can open a map of a world with no loop in it.

### 0.2 The ten non-negotiable design pillars (these arbitrate every conflict)

| # | Pillar | What it means in practice |
|---|--------|---------------------------|
| 1 | **The journey is the point** | No mechanic may reduce a haul to a loading screen unless the player explicitly opts in (§3.5). Time-skips are earned, priced, and imperfect; they still run the simulation and can still be interrupted. |
| 2 | **Freedom is never revoked** | Progression adds opportunity. It never removes a route, a train, or a choice the player already had. The full continent is physically drivable from hour one. |
| 3 | **Every train plays differently** | A new locomotive is a new *control problem*, not a bigger number. Each class has one signature management system that no other class has. |
| 4 | **Failure is expensive, not fatal** | The player loses time, money, and reputation — rarely the run itself. There is no game-over screen in the default mode. |
| 5 | **Nothing is unavoidable** | Every hazard telegraphs and offers at least two viable responses. Randomness creates *problems*, never *outcomes*. |
| 6 | **The world is legible at 300 m** | Silhouette, color, and motion must identify things at distance. Identification must never depend on texture detail. |
| 7 | **Earned calm** | Long stretches of smooth running are a reward, not dead air. They are interrupted *deliberately*, on a pacing curve, never at random. |
| 8 | **Story rides with you** | Narrative arrives in the cab, on the platform, and over the radio — not in a modal menu. The player should be able to name five NPCs after 5 hours. |
| 9 | **Deterministic under the hood** | Reproducibility is a feature, not a constraint. It enables regression tests, save integrity, telemetry-driven balancing, and future lockstep co-op. |
| 10 | **Data-driven content** | A new station, train, contract, hazard, or quest is a JSON file. Modding is a free byproduct of good architecture. |

### 0.3 Glossary (use these terms exactly in code and docs)

| Term | Meaning |
|------|---------|
| **Edge** | A single continuous piece of track between two Nodes. Arc-length parameterized. Has speed limit, gradient profile, curvature profile, traction class. |
| **Node** | A point on the track graph: platform, junction, switch, signal, buffer stop, depot, siding end. |
| **Ribbon** | The precomputed arc-length lookup for an Edge: at any distance `s`, it returns position, tangent, up-vector, curvature, and grade. |
| **Block** | The atomic signalling unit of track between two signals. Only one train may occupy a block (with defined exceptions: permissive signals, station limits). |
| **Consist** | The full train: one or more locomotives plus zero or more vehicles, in order, with couplers between them. |
| **Vehicle** | One physical unit on rails: locomotive, coach, wagon, tender. Has mass, length, bogie spacing, brake mass, damage state. |
| **μ (mu) / Adhesion** | The friction multiplier between wheel and rail. `1.0` dry, down to `0.42` on ice. Drives wheelslip, braking distance, and acceleration. |
| **TSR** | Temporary Speed Restriction. A signed, telegraphed, time-limited lower speed limit on an Edge (usually after a works train or weather event). |
| **Game clock** | The world's clock. Base scale is **4× real time** (1 game day = 6 real hours). Distinct from the fixed 60 Hz simulation step. |
| **Time scale** | The player-selectable multiplier on the game clock: `1×` (default — already 4× real time), `2×`, `4×` (cruise), `8×` (requires Cruise Assist), `60×` (parked/sleeping only). Gating rules in §3.5. |
| **Dwell** | The scheduled or voluntary stop time at a station. Doors/loading only operate during Dwell. |
| **Comfort index** | Per-run passenger satisfaction metric, `0.00–1.00`. Highest-value soft currency in the passenger business. |
| **Scorecard** | The end-of-run settlement panel. Contains timeliness, comfort/condition, fuel, violations, and the final payout breakdown. |

---

# 1. Executive Concept Summary

### 1.1 One-line hook
**Trains don't steer — you do the thinking.** In `IRONHORIZON` the skill isn't reflex, it's anticipation: reading a gradient, spending energy early, and delivering 400 tonnes of something fragile exactly on time.

### 1.2 The pitch

`IRONHORIZON` is a 3D open-world train simulation and delivery game set on **Aurelia**, a fictional continent of four contiguous, hand-authored regions rendered in a confident, art-directed low-poly style — clean geometry, bold color, stylized lighting, zero asset-flip noise. You begin as a licensed but broke independent operator: one light railcar, a bunk above a depot office, and a debt to the freight company you used to work for. From hour one, the entire **4,096 km² continent and its ~1,250 km of track is yours to drive**. There are no region gates, no locked map, no invisible walls, and no road you are forbidden from taking. You accept contracts to carry passengers, parcels, grain, cattle, mail, or regulated chemicals — or you refuse them all and run a scenic loop because you want to watch the sun come up over the highlands. There are always at least two ways to get from anywhere to anywhere. Choosing between them, and living with the consequences, is the game.

What separates the moment-to-moment from every other driving game is that **a train cannot swerve**. Removing steering removes the reflex layer entirely and replaces it with a planning layer: gradient, curvature, adhesion, brake-pipe charge time, consist mass, block occupancy, and a timetable that is always quietly ticking. A 600-tonne freight consist takes 1.1 km to stop; knowing that — *feeling* that — is the whole craft. And because no two classes handle the same way, the player's mastery is never finished: the regional passenger unit rewards smooth, punctual handling and punishes jerk; the heavy freight diesel turns every hill into an energy-budgeting problem with a real risk of wheelslip; the livestock train adds a welfare meter that degrades if you brake too hard around 400 head of cattle; the tanker train imposes speed caps and manifests that can be audited. Progression is 28+ trains, 48 stations, 5 licence endorsements, 6 reputation tracks, and 8 recurring character arcs — none of which ever take a route away from you. They only add reasons to take it.

The audience is the 25–55 player who has already enjoyed *Euro Truck Simulator 2*, *Train Sim World*, *Derail Valley*, or *Microsoft Flight Simulator*, and the adjacent "structured comfort" audience that plays *PowerWash Simulator*, *A Short Hike*, or *Unpacking* for the feeling of competent labor done well. Sessions run 20–45 minutes, with contracts sized so that "one more haul" is a real option and quitting mid-run is safe and always resumable. Single-player at launch, premium-priced, no microtransactions, cosmetic and region DLC post-launch. PEGI 7 / E10+ target rating. The tone is warm, dry-humored, slightly melancholy — a working railway in a sparsely populated country, in an era of analogue gauges and hand-written manifests.

### 1.3 Unique selling points

1. **Authored freedom.** Other train games give you either a sandbox with no stakes (*Train Sim World* free roam) or authored scenarios with no agency (its career). `IRONHORIZON` gives you a fully free route-planning sandbox with a consequence layer — timetables, fuel, wear, reputation, and story — attached to every choice.
2. **Trains as character classes.** Seven classes, each with one signature management system that exists nowhere else in the game (comfort index, wheelslip and brake-pipe management, chain-of-custody time windows, animal welfare, hazmat compliance, catenary and regenerative power, starter versatility). Not stat reskins.
3. **A living, operated railway.** AI freight and passenger services share the same track graph as the player. Signals hold you for them. Rival operators compete for the same contract pool, occupy the same platforms, and are visibly doing better than you until they aren't.
4. **Story that rides with you.** Eight recurring passengers and colleagues whose arcs advance based on *who and what you carried, and where*. A nurse who asks you to make an emergency halt in Chapter 2 remembers whether you said yes in Chapter 7.
5. **Failure that costs instead of kills.** Derailments are expensive, insured, narratively fertile, and survivable. Recovery is a real gameplay sequence — inspect the consist on foot, call for relief, work out the claim — not a reload prompt.
6. **Built to be built.** An engine-agnostic deterministic simulation core, fully data-driven content, and 28 phases with hard definitions of done. The project is designed to survive incremental, AI-assisted, multi-session development without collapsing into spaghetti.

### 1.4 Positioning matrix

| Dimension | Euro Truck Sim 2 | Train Sim World | Derail Valley | Death Stranding | **IRONHORIZON** |
|---|---|---|---|---|---|
| Route freedom | Full road network | Linear scenarios | One open valley | Corridor routes | **Full 4-region rail graph, always open** |
| Driving skill | Steering + traffic | Scripted procedures | Deep train physics | Terrain routing | **Anticipation, energy, braking, timeliness** |
| Machine differentiation | Stats + cosmetics | Per-loco fidelity | Loco class depth | Uniform | **7 classes × 7 distinct management systems** |
| Narrative | Radio flavour | None | None | Heavy authored | **Authored arcs delivered in-cab, always optional** |
| Failure model | Damage + fines | Scenario fail | Hardcore derailment | Corpse loss | **Cost, insurance, recovery sequence** |
| Delivery loop | Cargo contracts | None | Job board | Cargo hauling | **Contract economy + factions + scorecard** |
| Art direction | Photoreal-ish | Photoreal | Stylized realistic | Photoreal | **Art-directed low-poly, painterly** |

---

# 2. Core Gameplay Loop

### 2.1 The control model (there is no steering)

The player controls power and braking, never direction. The full control set, its range, and its default input mapping:

| Control | Range / behaviour | Keyboard | Gamepad | Notes |
|---|---|---|---|---|
| **Reverser** | F / N / R | `Q` cycles | LB+DPad | Cannot move under power unless out of N. Must be at 0 speed to change. |
| **Regulator (traction)** | Notches 0–8 | `W` up, `S` down | Right trigger (analog) | Notch 0 = idle; each notch = discrete traction-step. Moving from 8 to 0 is one action: `X`. |
| **Train brake (automatic)** | Release → Notch 1–5 | `[` / `]` | LT (analog) | Applies to the whole consist. Takes **2.4 s per 100 m of consist** to propagate. Recharging from a deep application takes 40–90 s. |
| **Direct brake (locomotive)** | Release → Notch 1–3 | `,` / `.` | DPad ud | Fast-response, locomotive only. Used for precision stops and holding. |
| **Dynamic / retarder brake** | 0–8 | `Z` / `C` | RB held + analog | Converts motion to heat. Overheats (a real failure mode). Free energy, expensive heat. |
| **Sanding** | On/Off (auto in Assisted) | `B` | `Y` | Raises effective μ by 0.08–0.15 for 20 s, limited reservoir. |
| **Emergency brake** | Latching | `Space` | `B` (hold) | Full application, 1.2× service deceleration, must be reset at 0 speed. Costs a scorecard penalty unless justified by an event. |
| **Horn** | Short / long | `H` / `Shift+H` | `X` / hold | Required at level crossings and in tunnels by regulation; violations are fined. |
| **Handbrake** | On/Off | `P` | DPad lr | Required before leaving the cab. |
| **Headlights / cab lights / wipers / heater / PA** | Toggles | `L`, `K`, `V`, `T`, `Y` | Menu | Headlight mode (off/dim/full) affects visibility and NPC reactions. |
| **Look** | Free-look | Mouse | Right stick | In-cab, over-shoulder, side-window and "lean out" views. |
| **Camera** | Cab / Chase / Cinematic / Free / Ride-along | `1`–`5` | DPad ud | Cinematic auto-cuts on long straights. |
| **Route map / Timetable** | Overlay | `Tab` | `Back` | Shows the track graph, block occupancy, gradient profile, your schedule. |
| **DRA acknowledge** | Momentary | `E` | `A` | Required for signal aspects; ignored alarms accumulate until they trigger a brake application. |
| **Radio** | Channel select, PTT | `R` | DPad | Dispatcher, region, contract, story channels. |

**Two control models, chosen per contract at acceptance:**

| | **Assisted** | **Engineer** |
|---|---|---|
| Notch changes | Smoothed, safety-limited, wheelslip auto-corrected | Raw, instant, fully the player's problem |
| Brake pipe | Auto-charges; auto-sanding | Manual charge management; sand decisions manual |
| HUD | Explicit guidance: brake-point markers, speed advisor, gradient bar | Gauges only. No speed advisor. |
| Payout modifier | **−20%** | **+0% (full)** |
| Intended for | New players, casual sessions, comfortable commutes | Mastery, high-value contracts, leaderboards |

This is the *only* difficulty selector. It is not a hidden handicap — it is a chosen professional standard, framed diegetically as "running with an assistance computer."

### 2.2 Second-to-second: what the player is actually doing

Every moment is one of five activities, and the player is always in exactly one:

1. **Reading the line** (constant background): scanning the gradient bar, the curve-ahead indicator, the posted speed limits, the next signal aspect, and the block occupancy overlay on the map. The next 2 km of track is always summarised in a single HUD strip — this is the game's "rear-view mirror."
2. **Spending energy** (most of the run): choosing notches for the grade ahead, deciding whether to hold speed up a hill and lose it later, or lift early and coast the crest. Coasting is a real and rewarded skill (fuel score).
3. **Managing the machine** (intermittent): watching brake pipe pressure, traction motor temperature, dynamic brake heat, fuel/charge level, sand reservoir, wheel-slip indicator.
4. **Managing the cargo/people** (intermittent): reading the comfort meter, the cargo condition bar, the livestock welfare dial, the hazmat compliance panel, the time window countdown.
5. **Communicating** (intermittent): answering the radio, acknowledging DRA, making PA announcements, arguing with a passenger, negotiating a contract change mid-run.

**Worked example — the first 30 seconds of a passenger haul from Kestrel Cross:**

- `t=0.0s` — Dwell timer hits zero. HUD shows a departure checklist: doors closed ✓, reverser F, brake released, DRA acknowledged.
- `t=1.5s` — Player raises the regulator to notch 3. Traction steps up over 2.4 s (real electric motors ramp; instant application is a wheelslip risk on wet rail).
- `t=4.0s` — The comfort meter dips slightly (a 0.4 m/s³ jerk spike from notch 3 on a wet rail). The player could have used notch 2.
- `t=9.0s` — Three AI commuter cars cross the level crossing ahead; the player sounds a long horn out of habit, and the game notes it as a regulatory-compliant horn event (no fine either way, small reputation nudge with the Regulator faction).
- `t=14.0s` — 300 m ahead the gradient bar shows the line climbing to +2.1%. The player raises to notch 5 pre-emptively; a good player lifts to notch 4 at the crest at `t≈55s` and coasts to the first station approach.
- `t=27.0s` — A radio call from dispatch queues: *"3781, you've got a farm tractor on the line at Kingsbury crossing, three minutes."* It will not play until the player is below 60 km/h or stationary (§2.4).

### 2.3 Minute-to-minute: the haul cycle

A "haul" (one contract, one run) has the same seven-beat structure regardless of cargo. Typical durations given for a mid-game 30–60 km run.

| Beat | Typical duration | What happens | Player decisions |
|---|---|---|---|
| **1. Plan** | 1–3 min | At a job board or cab terminal: read contract (origin, destination, cargo, mass, deadline, penalties), open the map, compare 2–4 viable routes, check weather, check which train you own. | Route, train, control model, whether to chain a second contract. |
| **2. Prepare** | 0.5–2 min | Couple to the consist in the yard. Load cargo (see 2.5). Complete pre-departure checks. | Consist composition, load distribution, whether to take extra fuel. |
| **3. Depart** | 20–40 s | Pull out of the station/yard, accelerate to line speed, observe the first signals. | Notch discipline, wheelslip avoidance. |
| **4. Line running** | 4–25 min | The core. Gradient and curve management, speed limits, TSRs, signals, AI traffic, weather, radio. 1–3 dynamic events per 10 minutes by design. | Energy budgeting, braking points, overtaking at passing loops, hazard response. |
| **5. Approach** | 1–2 min | Speed restrictions into station limits, platform selection, brake planning, final signal. | Where to begin braking (the single highest-skill moment in the game). |
| **6. Dwell** | 1–4 min | Stop on the mark, doors, announcements, loading/unloading, boarding, refuelling, quick maintenance. | Precision score, whether to run a coupling shunt, whether to take a hot-swap cargo. |
| **7. Settle** | 20–60 s | The Scorecard (§2.6). Payout, reputation deltas, wear applied, new story beats unlocked. | Spend, repair, upgrade, chain, or stop for the night. |

### 2.4 Session-to-session: the arc

A typical 40-minute real-time session contains **two to four hauls** plus a settle beat.

- **Session shape (target):** 8 min ramp-up (plan + easy first leg) → 20 min engaged running (2 hauls, at least one meaningful decision) → 8 min cluster (settle, upgrade, story beat, real progress) → 4 min "one more haul?" temptation (a contract with a genuine reason to accept it: a story NPC waiting, a rare cargo, a rival company about to take it).
- **Persistence of intent:** the player always has 1–3 *long-term goals* visible on the map (a train they're saving for, a licence test, a story thread they're mid-way through, a region they want to see). These are never blocking, always optional.
- **Soft session endings:** the game never demands a stopping point. It offers one — a "night at the depot" action that banks progress, advances the clock, and gives a written summary of the day (§3.5).

### 2.5 The delivery loop in detail

**Contract lifecycle states:** `offered → accepted → prepared → loaded → undocked → in_transit → arrived → unloaded → settled`, with `failed`, `abandoned`, `diverted`, and `renegotiated` as exits from any state after `accepted`.

- **Loading is a mechanic, not a timer.** At a freight yard, loading opens a small allocation interface: drag/place cargo units into wagon slots respecting mass limits, axle load limits, and fragility adjacency rules (e.g., don't place the piano crate next to the loose pipe load). A correct, well-balanced load measurably improves handling (lower centre of gravity, better braking) and pays a small *Cargo Handling* bonus. A botched load raises derailment risk in curves. Loading takes 40 s–3 min of game time depending on mass and method, and can be skipped for a small fee and a handling penalty.
- **Passengers are different.** Boarding is a dwell activity: you select which doors open per platform side, whose destination cards you accept (overcapacity is allowed but the standing passengers' comfort drains fast), and you may refuse a passenger — which is a story action, not a system one.
- **Unloading failures are real.** Arriving late on a perishable contract doesn't fail it; it de-rates the payout progressively along a curve (§6.2), and past a threshold the receiving party refuses part of the load, generating a *Claim* event with dialogue (and often a story hook).
- **Chaining.** A contract can be chained into a second contract that starts at or near the destination. Chaining within 10 game-minutes of arrival grants a **Continuity bonus** (+6% payout on both). This is the primary incentive for route planning that thinks two steps ahead.

### 2.6 Quest interruption rules (hard-coded pacing rules for the director)

Story and system events are never allowed to hijack an engaged player badly. The `EventDirector` obeys:

1. **A new interruption may only *start* if:** the player is stationary, **or** travelling below 40 km/h, **or** the event is a genuine hazard (safety-relevant).
2. Non-urgent narrative beats are **queued** and fire at the next station approach, the next stop, or the next time the player lifts off the regulator below 25 km/h.
3. Any interruption that requires reading more than 12 words is delivered while stationary. Radio dialogue may play while moving; choices that affect state wait for a stop.
4. **No new hazard within 3 minutes of haul start.** No second major hazard within 4 game-minutes of a major hazard. Maximum **one major hazard per 12 game-minutes.**
5. If the player has ignored three consecutive non-urgent beats, the director raises the delay until the next dwell and marks them for delivery during Dwell (where they can't be missed).
6. **Interruptions arrive with a reason to exist:** a personal request from a recurring NPC, a weather deterioration, a signalling conflict with an AI train, or a contract change. Never "because it's time for an event."

### 2.7 The Scorecard (worked example, fully specified)

> **Contract #4471 — "Greybridge Commuter Service"**
> Class P *Peregrine* · Kestrel Cross → Marlbrook Halt via Ridge Line · 46.2 km · 96 passengers
> Control model: Engineer · Base payout: **₡3,600**

| Line item | Value | Multiplier / amount |
|---|---|---|
| Timeliness | Arrived 1:12 early | ×1.05 |
| Comfort index | 0.94 (three jerk spikes > 0.5 m/s³) | ×(0.70 + 0.30 × 0.94) = ×0.982 |
| Platform precision | 4 of 5 stops within ±2 m | ×1.02 |
| Fuel | 78 L used vs 85 L allowance | +₡120 |
| Reputation (Passenger Assoc. 62, Band B) | — | ×1.08 |
| Violations | none | ×1.00 |
| **Subtotal** | 3,600 × 1.05 × 0.982 × 1.02 × 1.08 | **₡4,088** |
| Fuel bonus | — | **+₡120** |
| **Final payout** | — | **₡4,208** |

Wear applied this run: bogie wear +0.4%, brake shoe −2.1%, traction motor hours +0.3 h. Three passengers left favourable reviews (Comfort > 0.9); one wrote a complaint about the 14-second late departure at Kingsbury.

---

# 3. World & Open-World Design

### 3.1 Scale and structure (locked numbers)

| Property | Value |
|---|---|
| Total land area | **4,096 km²** — 4 regions of 32 km × 32 km, arranged in a 2×2 grid with overlapping borders (a single continuous landmass, no loading seams) |
| Continent name | **Aurelia** |
| Total track length at 1.0 | **~1,250 km** of drivable rail (main lines ~620 km, secondary ~410 km, branches/sidings/yards ~220 km) |
| Stations at 1.0 | **48** (4 Hubs, 12 Depots, 32 Halts) |
| Track graph nodes | ~**1,450** (including junctions, signals, switches, buffer stops, siding ends) |
| Named settlements | 61 (of which 22 are station-served) |
| Elevation range | 0 m to 2,340 m above sea level |
| Longest single edge (no node) | 34.6 km (Saltmarsh coastal straight) |
| Scale | **1 unit = 1 metre.** Y-up. Metric throughout. Real-world-consistent distances, so speed in km/h and mass in tonnes mean what they say. |
| Map grid cell | 1 km² terrain chunks; 16 chunks per region-superchunk; 1,024 chunks total |
| Fastest route across the continent | ~4 h 10 min game-time in Class E express on the electrified trunk; ~9 h in heavy freight |

**Why 1:1 scale works here:** at line speeds of 60–160 km/h with a default 4× game clock, a 40 km haul feels like ~10 real minutes — the same felt duration as a 300 km truck run in *Euro Truck Simulator 2*, without any distance fudging that would break the illusion of a train's inertia, braking distance, or timetable. **Do not compress distances.** Inertia is the game.

### 3.2 The four regions

Each region has: a distinct palette (§8.2), a distinct dominant hazard profile, a distinct economic identity, a signature engineering feature, and one unique systemic rule.

| | **I. VERDANT BASIN** | **II. IRONREACH HIGHLANDS** | **III. SALTMARSH COAST** | **IV. ASHFALL CORRIDOR** |
|---|---|---|---|---|
| **Geography** | Rolling farmland, river valleys, broadleaf woodland, low viaducts | Folded mountains, granite escarpments, spiral loops, snow line at 1,400 m | Mudflats, reed marsh, sea walls, long low bridges, tidal inlets | Stripped industrial valleys, spoil heaps, refineries, shale pits, night-shift towns |
| **Elevation** | 20–340 m | 180–2,340 m | 0–90 m | 40–620 m |
| **Track character** | Gentle 1:200 grades, wide curves, 100–140 km/h | 1:40 grades, 300 m radius curves, spirals, 14 tunnels, 22 km of double-headed-graded line | 1:500 grades, 18 bridges, 1 sea crossing (2.1 km), 60–110 km/h | 1:60 grades, complex yards, industrial branches, 70–90 km/h |
| **Dominant hazards** | Flooding, fallen trees, harvest traffic at crossings | Snow, rockfall, ice, avalanche zones, sharp curves | Fog, wind, tide-driven flooding, bridge restrictions | Chemical incidents, night visibility, heavy axle loads, worn rail |
| **Economy** | Grain, timber, dairy, livestock, regional commuting | Ore, quarries, timber, hydro power, tourism | Fishing, salt, import/export freight, seasonal tourism | Coal, chemicals, steel, manufactured goods, hazmat |
| **Signature feature** | The Greybridge curved viaduct, 640 m, 11 arches | The Ashcombe Spiral — a 1,100 m double-loop gaining 84 m | The Saltmarsh Sea Wall — 6 km of track 2 m above the tide line | The Cinderworks — a 9-track classification yard with a working hump |
| **Unique systemic rule** | **Crop Cycles:** seasonal freight availability shifts dramatically; harvest weeks create time-window contract floods | **Altitude Effects:** above 900 m, engine output falls ~2% per 300 m (§4.3); above 1,400 m snow is permanent in winter | **Tide Windows:** some coastal branches are only passable within a tide window; the schedule panel shows the window | **Curfew Zones:** residential areas impose night-time noise restrictions on hazardous and heavy freight |
| **Starting region** | ✅ Player begins here | Unlocked content-wise by Licence, never geographically gated | | |

### 3.3 Station hierarchy

| Tier | Count | Provides | Assets | Typical dwell |
|---|---|---|---|---|
| **Hub** (Tier 1) | 4 — one per region (Kestrel Cross, Highmoor Junction, Saltmere, Cinderworks) | Full job board (12–20 contracts), depot with garage slots, shop, upgrades, licence testing, crew hiring, hotel/sleep, crew-transfer point | 4–7 platforms, station building, footbridge, yard, turntable | 6–25 min |
| **Depot** (Tier 2) | 12 | Job board (4–8 contracts), basic maintenance, refuelling, small shop, hotel/sleep | 2–3 platforms, goods shed, siding | 3–10 min |
| **Halt** (Tier 3) | 32 | No job board. Contracts may *originate* or *terminate* here (pickup/dropoff only). Platform, shelter, siding, water/fuel drum maybe. | Single platform, shelter, lamp | 40 s – 3 min |

**Distribution rule:** no point on the map is more than **14 km of track from a Halt** and no more than **38 km from a Depot**. This bounds the worst-case "I ran out of fuel in the middle of nowhere" walk-of-shame to something recoverable, and guarantees the player can always find a contract source.

### 3.4 How free will is preserved (implementation guarantees)

These are testable properties of the track graph, not aspirations. The build pipeline must run a **graph validation tool** after every content change and fail the build if violated:

1. **Route redundancy rule.** Between any two Hub nodes, there are **at least 2 edge-disjoint paths** whose travel times differ by **at least 15%**. Between any two Depot nodes, at least 2 paths exist (they may share edges, but must differ by at least 3 edges).
2. **Traction-class completeness rule.** For every pair of stations, there is at least one path valid for **every** traction class (diesel, electric, and — if steam ever ships — steam), at least one path with a **maximum gradient ≤ 1:80**, and at least one path with **maximum gradient ≤ 1:33** for short consists.
3. **No dead ends (except deliberate ones).** Every station is on a through-line or loop, except 6 intentional branch terminals (each with a visible buffer stop, a turntable loop, or a short spur to reverse into). Deliberate dead ends are diegetic, signposted, and never traps.
4. **Physical bounds are diegetic.** The world is an island. Boundaries are coastline, cliffs and water — never invisible walls. Rails always terminate at a visible buffer stop or a visible poorly-maintained end-of-line. There is no invisible geometry anywhere in the game; a debug pass must confirm this by driving the perimeter.
5. **No content gates on geography.** Every station is reachable and enterable from hour one. One region (Ashfall's north-east quarries) has its rail access physically severed at start by a collapsed retaining wall; this is repaired as a *story outcome*, and even then, an alternate 41 km mountain branch remains passable from hour one.
6. **The Ring Line.** A 96 km perimeter ring connects all four regions on low-grade secondary track. The player is never more than one route-choice away from a way home, and the ring is always the "safe, slow" option that makes dangerous shortcuts a choice rather than a necessity.
7. **The map never nags.** The route planner suggests, never commands. Ignoring the suggestion changes nothing except the numbers on the scorecard.

### 3.5 Time: day/night cycle, schedules, and fast travel policy

**Clocks.** Exactly two, and they must never be conflated in code:

- `simTick` — fixed 60 Hz simulation step. Physics, signals, collisions, wear. Unaffected by anything the player does.
- `gameClock` — the world clock, `gameClock += dt × timeScale × 4`. At the default `timeScale = 1`, **4 game-minutes pass per real minute**. 1 game day = 6 real hours.

**Time scale options:** `1×` (default) · `2×` (only on straight track, clear signals, >20 km from any hazard) · `4×` (cruise; the standard "long haul" setting) · `8×` (requires an unlocked **Cruise Assist** upgrade; auto-drops to 1× on any hazard telegraph) · `60×` (parked at a station or sleeping only — it is the diegetic "I've finished for the night" action).

**Day/night effects (all enforced by the sim, not just lighting):**

| Period | Game hours | Effects |
|---|---|---|
| Dawn | 04:00–06:30 | Fog risk ×1.8, livestock calmer, empty passenger schedule |
| Day | 06:30–17:00 | Peak freight throughput, harvest traffic at crossings, highest passenger demand at 07:00–09:00 and 16:00–18:00 |
| Dusk | 17:00–19:30 | Animal-on-line risk ×2.4 (this is the single most common hazard in the game), visibility falling |
| Night | 19:30–04:00 | Headlight discipline required (fined if off on a running line), trespass risk ×1.7, curfew zones active in Ashfall, snow-clearing crews deploy, passenger demand collapses except on the Ashfall night shift |

### 3.6 Weather and seasons

**Seven weather states**, driven per-region by a Markov chain with a configurable transition matrix (`content/weather.json`), blended over 90–240 game-seconds so nothing ever "snaps."

| State | Adhesion μ | Visibility | Wind | Primary gameplay effect |
|---|---|---|---|---|
| Clear | 1.00 | 12 km | 0–12 km/h | Baseline |
| Overcast | 0.97 | 9 km | 5–20 km/h | Baseline; better for hazmat (no solar heat) |
| Light rain | 0.86 | 6 km | 10–30 km/h | +18% braking distance; open-wagon cargo takes wetness |
| Storm | 0.74 | 3 km | 45–95 km/h | Falling trees (hazard), bridge crosswind limits, sea spray on coast, roof leaks in old coaches |
| Fog | 0.95 | **180 m** | 0–10 km/h | The signature hazard: signals invisible until close. Requires disciplined speed reduction. TSR often imposed. |
| Snow | 0.58 | 1.5 km | 15–40 km/h | Traction loss on grades; points freeze (junctions may be set wrong — the player must verify); passenger delays |
| Blizzard | 0.42 | **90 m** | 60–120 km/h | Mountain passes may close entirely; whiteout requires stopping and waiting; livestock welfare drains fast |

**Weather is never pure decoration.** Adhesion μ multiplies available traction and all braking forces, which changes stopping distances enough to invalidate memorised braking points. A validated reference table (part of the test suite) must hold: *for a 600 t Class F consist at 80 km/h, braking distance in Clear = 980 m, in Snow = 1,690 m, in Blizzard = 2,335 m.*

**Seasons.** Four seasons, **14 game days each** (56-day year, configurable). Seasonality is primarily a content multiplier, not a difficulty gate:

| Season | Visual | Freight | Passenger | Systemic |
|---|---|---|---|---|
| Spring | Fresh greens, high water | Fertiliser, seed, machinery | Low | Flooding events peak; landslide risk at 1:40 grades |
| Summer | Hot haze, dry gold | Grain, livestock, timber | Peak tourism to Highlands | Track buckling TSRs on >32 °C days, midday; livestock heat stress |
| Autumn | Amber rust, low sun | Harvest surge, root crops, ore | Moderate | Fallen-leaf adhesion penalty (−0.06 μ) on wooded gradients; fog peak |
| Winter | Snow line down to 220 m, ice | Coal, salt, heating oil, hazmat | Holiday peaks | Snow clearing delays, frozen points, longest braking distances |

### 3.7 Fast travel policy — "the journey is the point, but respect the player's evening"

There is **no teleportation of trains** and **no free instant travel**. There are exactly three mechanisms, all diegetic, all priced, and all of which still run the simulation:

| Mechanism | Unlocked | What it does | Cost | Limits |
|---|---|---|---|---|
| **Crew Transfer** | Hub visit (Tier 1 station) | Moves *the player character* to any other Hub you have visited, to take charge of a train already parked there or to accept a contract that starts there. Your other locomotive stays exactly where you left it. | ₡800–₡2,400 (distance-scaled) + 3–8 game-hours elapsed | Requires an empty garage slot at the destination, or a stored train. Your current train must be safely stabled and handbraked. |
| **Deadhead Time-Skip** | Depot visit | Auto-drives your *empty* train along the real track at 8× time scale, running the full simulation: signals, AI traffic, weather, and hazards all still apply, and it can be interrupted at any moment by an event that demands the player take control. | Costs real fuel + crew wage for the elapsed hours | Cannot be used with cargo or passengers aboard. Cannot be used in Blizzard, on TSR-affected edges, or on any edge with a gradient >1:50. |
| **Cruise Assist** | Mid-game unlock (₡48,000) | Allows 8× time scale while moving *with cargo*, but caps the run's payout at 60% and disables all scorecard bonuses. Purely an "I want to get there and I've already mastered this line" tool. | –40% payout | Auto-disengages on any hazard telegraph, any TSR, any signal at caution, and within 2 km of any station. |

**Rationale (this must be honoured):** any time-skip still costs fuel, still risks wear, and can still be interrupted. Skipping is a *trade*, never a bypass. And because the player physically cannot teleport a loaded consist, they always have a reason to actually drive the interesting routes — the ones the game is about.

### 3.8 Points of interest & non-rail content

The world is worth looking at from the cab, and occasionally worth getting out of it:

- **Viewpoints (24)** — a siding, a viaduct parapet, a platform end. Reaching one at a specific time/weather unlocks a **Postcard**: a framed low-poly diorama render of the view, added to a collection album. Pure collector content, no gameplay power, high emotional value.
- **Walkable spaces (12)** — the 4 Hub station concourses, 4 Depot yards, 3 incident sites, and the inside of the player's home depot. Small, ~2 minutes to cross, first-person, low-poly, deliberately calm. This is where walk-up NPC encounters happen.
- **Hidden sidings (9)** — unmarked spurs leading to abandoned works, a quarry, a chapel, a lake. Discovered by driving past them, not by map markers. Each contains a story vignette or a rare cosmetic.
- **Wildlife & scenery systems** — birds over the marsh rising when your train passes, deer that cross the line at dusk (and *are* a real hazard), sheep on upland lines, an NPC fishing boat in the saltmarsh. Cheap, high-impact, low-poly-native content.

---

# 4. Train Types & Mechanics

Seven classes. Each has **one signature management system** that no other class has, and that system is the reason to own the train. All values are launch-balance targets in `content/trains.json` and are expected to be tuned.

### 4.0 Master comparison table

| # | Class | Role | Mass (light/laden) | Top speed | Capacity | Signature mechanic | Unlock cost | Upkeep / 1,000 km |
|---|---|---|---|---|---|---|---|---|
| 1 | **S — "Sable"** | Starter light mixed-use railcar | 22 t / 44 t | 90 km/h (laden 80) | 12 t freight **or** 24 pax, hot-swappable | **Versatility Module** — the only train that can switch role mid-day; cheapest to run | Start (owned) | ₡310 |
| 2 | **P — "Peregrine"** | Regional passenger DMU | 96 t / 132 t | 140 km/h | 180 pax (72 seated, 108 standing allowed) | **Comfort Index** — a live 0.00–1.00 satisfaction meter with per-passenger simulation | ₡64,000 | ₡980 |
| 3 | **F — "Foundry"** | Heavy freight diesel-electric | 128 t / 728 t (with 600 t payload) | 80 km/h laden, 100 km/h light | 600 t / 40 TEU | **Mass & Adhesion Management** — brake pipe charge, coupler slack, wheelslip on grade, sand budget | ₡118,000 | ₡1,640 |
| 4 | **M — "Courier"** | Express mail / high-value parcels | 74 t / 164 t | 160 km/h | 90 t, 6 sealed mail cars | **Chain of Custody** — timed handoff windows, sealed cars, tamper events, no unscheduled stops | ₡156,000 | ₡1,220 |
| 5 | **L — "Pastoral"** | Livestock & agricultural | 108 t / 428 t | 70 km/h | 320 t **or** 400 head of cattle | **Welfare & Freshness** — animal stress and produce spoilage clocks | ₡88,000 | ₡1,180 |
| 6 | **H — "Cinder"** | Hazmat / tanker | 142 t / 642 t | 60 km/h **hard cap** | 500 t (8 tank cars) | **Regulated Compliance** — zone speed caps, manifest accuracy, incident severity ×4 | ₡204,000 | ₡2,400 |
| 7 | **E — "Aurora"** | Electric high-speed express | 210 t / 306 t | 200 km/h | 240 pax, 40 t parcels | **Catenary & Regenerative Power** — only runs on electrified track; earns credits via regenerative braking | ₡372,000 + Renewables Licence | ₡2,050 |

### 4.1 Class S — "Sable" (starter light railcar)

**Fantasy:** *Your first machine. Rattles, honest, gets you paid.*
**Role:** Tutorialisation-by-ownership. It teaches coupling, loading, gradients, weather, and the scorecard in the lowest-stakes possible frame.

- **Specs:** 22 t empty. Two-axle plus one trailer bogie. Traction: 34 kN max. Max brake force: 42 kN. Service braking from 90 km/h: **410 m** (laden 545 m). Fuel: 320 L diesel, ~4.1 L/100 t·km.
- **Handling:** Reasonably punchy, very light, so it is unusually sensitive to load changes — the player *feels* the difference between empty and full. Prone to bouncing on jointed track. Cannot haul more than two additional wagons (coupler strength limit).
- **Signature mechanic — Versatility Module:** the only class where the rear module can be hot-swapped at a Halt or Depot in 4 game-minutes for a fee of ₡90: **Passenger Module** (24 seats, comfort 0.62 baseline), **Freight Module** (12 t open or covered), **Utility Module** (1,200 L fuel, repair kit, 2 t tools — enables field repairs at reduced quality). This makes the Sable genuinely useful for the entire game as a "short-notice contract chaser" — it can accept the odd 8 km passenger hop that no other train would profit from.
- **Upgrade path (5 tiers, each ₡1,200–₡7,500, total ₡18,400):** ① Heavy coupler (+2 wagon limit) ② Improved bogies (−22% bounce, +0.04 comfort) ③ Turbocharger (+9% power, −4% fuel) ④ Reinforced brake block (+14% brake force) ⑤ Auxiliary power unit (Utility Module can also carry 8 pax).

### 4.2 Class P — "Peregrine" (regional passenger)

**Fantasy:** *A timetable in your hands. Ninety-six people are judging your smoothness, and they're right to.*

- **Specs:** 96 t empty (3-car), expandable to 5 cars (156 t). Traction 168 kN. Service braking from 140 km/h: **1,180 m**. Boarding: doors both sides, 12 s per side per stop. Fuel 1,400 L.
- **Handling:** High power-to-weight, punchy acceleration (0–100 km/h in 42 s), and a light body that transmits every braking error straight into the passenger compartment. Dwell times are tight; a slow stop costs the schedule.
- **Signature mechanic — Comfort Index.** A live `0.00–1.00` value, computed per game-second and reported per stop, per passenger class. Formula:

```
comfortRate = 1.0
  − clamp(jerkExcess, 0, 1)          × 0.45   // jerk = |d²v/dt²| over 0.30 m/s³ counts fully
  − clamp(latAccelExcess, 0, 1)      × 0.28   // lat accel = v²/R ; over 1.1 m/s² counts fully
  − clamp(|temp − 21°C| / 10°C, 0, 1)× 0.10   // cab heater setting affects the whole train
  − clamp(dwellOverrun / 120s, 0, 1) × 0.09   // running late makes people anxious
  − (occupancy > seatedCapacity ? 0.06 : 0)   // standees cost you heavily
  − (announcementsMissed × 0.02)
comfort = rolling average of comfortRate over the whole run, weighted by occupancy
```

  Each passenger has four hidden traits (`impatience`, `temperature_sensitivity`, `jerk_sensitivity`, `chatty`) that weight their contribution and drive their individual reviews. A `comfort ≥ 0.90` run generates favourable reviews that raise Passenger Association reputation **1.6× faster**; a run `< 0.65` generates complaints that cost reputation and can trigger a regulator inspection at high frequency.
- **Signature mechanic 2 — Punctuality Windows:** each stop has a scheduled arrival ±90 s. On-time arrivals build a "Reliability Streak"; 5 consecutive on-time stops grant +8% payout; a miss resets it to zero. This makes the entire class about time discipline rather than cargo safety.
- **Upgrade path (total ₡52,000):** ① Fifth car (+60 pax, +9% fuel burn) ② Air suspension (−30% jerk transmission) ③ Buffet car (+₡0.40/pax, +0.05 comfort, −12 seats) ④ Quiet bogies (latAccel coefficient ×0.7) ⑤ Automated announcements (+0.03 comfort, removes announcement penalty entirely).

### 4.3 Class F — "Foundry" (heavy freight diesel-electric)

**Fantasy:** *Seven hundred tonnes and a hill that does not care.*

- **Specs:** 128 t locomotive + up to 600 t payload across 24 wagons. Traction 412 kN starting, 268 kN continuous. Service braking from 80 km/h laden: **980 m dry / 1,690 m snow / 2,335 m blizzard.** Fuel 3,800 L, ~1.9 L/100 t·km average, **3.6 L/100 t·km on a 1:40 grade**.
- **Handling:** Slow, deliberate, immense. Acceleration from rest to 80 km/h laden takes ~4 minutes. The consist is a physical object with slack action: braking too abruptly causes wagons to bunch and then rebound ("run-in / run-out"), which is the primary cause of damage and derailment for this class.
- **Signature mechanic — Mass & Adhesion Management:**
  - **Brake pipe charge.** Pipe pressure recharges at 0.9 bar/s per 100 m of consist. After a heavy application, you may not have brakes again for 40–90 seconds. The HUD shows a charge bar; a player who plans their braking around it never gets caught out.
  - **Wheelslip.** When requested tractive effort exceeds `μ × adhesive weight (≈ 0.28 × loco mass)` the wheels slip. Slip costs time, damages rail, and on curve+grade combinations escalates to derailment risk. Mitigations: sanding (+0.08–0.15 μ for 20 s), reducing notch, or splitting the consist with a **banking locomotive**.
  - **Dynamic brake heat.** The retarder turns 20 tonnes of kinetic energy per 100 km/h into heat. Above 480 °C, braking force falls and the unit is damaged. Managing a long descent with retarder + air + coasting in alternation is the class's single best set-piece.
  - **Altitude derate:** engine output ×(1 − 0.02 per 300 m above 900 m) — see §3.2 Ironreach.
  - **Distributed power:** the player may add a mid-consist or rear locomotive (upgrade ⑤) to cut coupler forces by up to 45% on the steepest grades.
- **Upgrade path (total ₡74,000):** ① Traction motor blower cooling ② Brake-shoe composite (brake distance −11%) ③ Extra sand hopper (2.5× capacity) ④ AAR-style slack-reducing drawgear (−28% run-in energy) ⑤ Second locomotive unit / distributed power.

### 4.4 Class M — "Courier" (express mail & high-value parcel)

**Fantasy:** *The clock is the cargo.*

- **Specs:** 74 t, up to 6 sealed mail cars (90 t total). Traction 148 kN (excellent power-to-weight for a freight-adjacent class). Top speed 160 km/h. Service braking from 160 km/h: **1,610 m**. Fuel 1,900 L.
- **Handling:** Fast, smooth, and *tall* — mail vans ride high and catch crosswind, so exposed viaducts in a storm require speed reduction. Fast enough that a missed braking point on the wrong route is unrecoverable, which is exactly the class's tension.
- **Signature mechanic — Chain of Custody:**
  - Each mail car has a **seal ID**. Seals are verified at origin, at every **handoff window** (a station where the car is exchanged, 90–240 s window on the game clock), and at destination. A missed window is a contract breach: −25% payout and −6 reputation with the Postal Service.
  - **Tamper events** occur rarely (weighted by stopping in unlit sidings at night, and by route through Ashfall). An opened seal mid-run invites a decision: continue and declare (small penalty, honesty reputation bonus) or conceal (risk of a random audit, large penalty, and a story consequence in the Postal Service thread).
  - **No unscheduled stops** without declaring a reason over the radio. Emergency stops are permitted and free; a leisurely coffee stop in a siding is a violation.
  - **High-value consignments** (₡90,000+ declared value) attract **rival interest** — the Courier is the only class where the thief/hijack hazard flavour applies, resolved non-violently as tampering, mis-signalling, and route sabotage (§7).
- **Upgrade path (total ₡61,000):** ① Sealed vestibule connections ② Composite brake discs (braking −14%) ③ Weather-sealed door mechanisms (tamper resistance ×2) ④ Auxiliary generator car (cold-chain for parcel refrigeration) ⑤ Tilt-lite bogies (curve speed limit +20%, comfort +0.05).

### 4.5 Class L — "Pastoral" (livestock & agricultural)

**Fantasy:** *You are responsible for four hundred living things that cannot complain in words.*

- **Specs:** 108 t, up to 320 t produce or 400 head of cattle across 16 vehicles. Traction 246 kN. Top speed 70 km/h (higher speeds measurably raise animal stress). Fuel 2,400 L. Includes a 1,100 L water tank and a ventilation generator.
- **Handling:** Heavy, top-heavy with full stock cars, and deliberately underpowered. Braking must be gentle: a hard application doesn't just damage cargo, it *hurts* it.
- **Signature mechanic — Welfare & Freshness:**
  - **Animal stress** (`0–100` per car) rises with: jerk above 0.25 m/s³ (+3/s), lateral accel above 0.9 m/s² (+2/s), interior temperature above 24 °C (+1.2/s, mitigated by ventilation), stationary dwell beyond 8 game-minutes (+0.6/s), and ambient noise (tunnels, horn use nearby, sharp curve squeal). It falls at −1.4/s while moving smoothly below 55 km/h.
  - At stress `>60` the load is "distressed" (payout −12%); at `>85` a **Veterinary Hold** triggers on arrival: the load is refused, a vet inspects, a fine of ₡1,400 is levied, and the Animal Welfare Board reputation drops 8.
  - **Ventilation generator** burns fuel: 4.6 L/h while running. In summer, running it is mandatory above 24 °C; forgetting it is the single most common beginner failure for this class and is fully telegraphed by a rising temperature gauge.
  - **Produce freshness** uses a per-commodity clock: `freshness -= rate(commodity) × temperatureFactor × wetnessFactor × elapsed`. Cold-chain produce (dairy, soft fruit) needs the refrigerated cars; grain and root crops are durable but heavy.
- **Upgrade path (total ₡56,000):** ① Sprung stock-car floor (jerk transmission ×0.6) ② Powered drinking troughs (water consumption doubled → stress gain ×0.7) ③ Shade/side-screening (heat gain ×0.5) ④ Refrigerated compartment set (2 cars, −14 °C) ⑤ Low-noise bogies (noise contribution ×0.4).

### 4.6 Class H — "Cinder" (hazmat / tanker)

**Fantasy:** *Everything about this train is a form you have to fill in correctly.*

- **Specs:** 142 t + 500 t (8×62.5 t tank cars). Traction 388 kN. **Top speed hard-capped at 60 km/h by regulation** — this cap cannot be upgraded away, only *modified by zone* (some lines permit 70 km/h for Class III goods). Braking from 60 km/h laden: **1,050 m**. Fuel 3,400 L.
- **Handling:** Heavy and long. The dominant feel is *inertia plus rule*. This is the slowest, most deliberate, highest-margin class in the game — and the one with the most to lose.
- **Signature mechanic — Regulated Compliance:**
  - **Zone classes** assigned per Edge in content data: `Z1 urban/tunnel` (max 30 km/h with Class I goods), `Z2 populated rural` (45 km/h), `Z3 open line` (60 km/h), `Z4 industrial` (60 km/h, but mandatory stop-and-check at yard entry). A regulator timer logs every violation; 3 violations in one run escalate to a licence review.
  - **Manifest accuracy.** Before departure the player completes a manifest (commodity, UN class, quantity, emergency contact). The game may inject an error (a wrong quantity, a mis-classified commodity) that the player is expected to catch by cross-checking the load sheet against the tank car placards. Catching it: +4 Regulator reputation. Missing it: if an incident occurs, penalties multiply by 1.8.
  - **Pressure & temperature.** Pressurised gases (chlorine, ammonia, LPG) build pressure in heat; the player must open vents, avoid prolonged sun-side parking, and never exceed a temperature threshold. Cryogenic loads must be topped up at specified stations.
  - **Incident severity ×4.** Any derailment, collision, or cargo damage event involving this class scales consequences far beyond any other: multi-day line closure (which affects *future contracts on that route* — real systemic consequence), evacuation of a nearby town (visible in the world: a cordon, emergency vehicles), an inquiry event with authored dialogue, insurance premium increase, and possible licence suspension for 10 game days.
- **Upgrade path (total ₡92,000):** ① Secondary containment pans ② Emergency pressure-relief valves (enables faster venting decisions) ③ Fire-suppression onboard (containment score +12, reduces incident severity ×4 → ×2.6) ④ Grade-control brake set (allows 70 km/h in Z3) ⑤ Telemetry suite (auto-detects manifest errors before departure, removes the check minigame but halves the reputation reward).

### 4.7 Class E — "Aurora" (electric high-speed express)

**Fantasy:** *The route itself becomes the constraint. This machine only works where somebody else has already spent the money.*

- **Specs:** 210 t (4-car set), expandable to 6 cars (306 t). Traction 520 kN. Top speed 200 km/h. Regenerative braking recovers 31% of braking energy as credits. Service braking from 200 km/h: **3,120 m**. **No fuel tank at all** — it draws from the catenary.
- **Handling:** Instant, immense, and silent apart from a rising electrical whine. It is by far the smoothest and the fastest thing in the game — and it is completely immobilised the instant the wire ends.
- **Signature mechanic — Catenary & Regenerative Power:**
  - **Electrification is a per-Edge property.** The map has an electrified trunk (~340 km), plus three electrified regional branches. **The route planner is therefore a hard constraint, not a suggestion** — and this is the class's entire strategic identity: the Aurora turns route choice into a topology puzzle, and makes the expensive "keep the diesel on the roster" decision worthwhile.
  - **Catenary limits.** Power draw is capped per section; on steep grades at high speed the unit can trip a section breaker, forcing a 30-second reset and a very embarrassing hill start. Acceleration must be planned against available supply (shown on a live ammeter).
  - **Regenerative credits.** Each regenerative braking event adds credits at ₡0.08 per recovered kWh. A well-driven mountainous electric run can earn ₡900–₡1,800 in credits *in addition* to the fare — the only class that pays you for driving well in the middle of the run.
  - **Upgrade path (total ₡128,000):** ① Fifth and sixth car sets ② Battery tender (allows 14 km of off-wire running at max 60 km/h — unlocks the whole non-electrified network in a limited way and transforms the class's usefulness) ③ Section-breaker tolerance software (removes trip events) ④ Regenerative optimiser (+31% → +38% recovery) ⑤ Tilt suspension (curve speed +25%).

### 4.8 Consist construction rules (all classes)

The player assembles trains in a yard with a live weight/brake/length calculator. Rules the code must enforce:

1. **Coupler compatibility.** Three coupler types exist in the world: `auto` (modern, all classes except Class S starter), `buffer_chain` (heritage rolling stock, Class S has an adapter), and `scharfenberg` (Class P and E multiple units). Incompatible pairs require an **adapter wagon** (available at Depots, ₡340, 14 t).
2. **Brake continuity.** Every vehicle must be connected to the brake pipe. Unbraked vehicles are permitted only as a maximum of 1 tail wagon, and they reduce permitted speed by 25% and raise scorecard risk.
3. **Axle load & bridge class.** Each Edge has a `bridge_class` (1–5). A consist exceeding the class limit cannot legally traverse it — the route planner removes the option, and attempting it triggers a **Network Rail refusal** at the signal and a Regulator violation (this is the *only* place the game says "no," and it says it before you leave, never as an invisible wall).
4. **Gradient competence.** The planner computes `max grade climbable = f(traction, μ, mass)`. If a route contains a grade beyond the consist's ability, it is flagged in red with the exact reason ("1:40 climb at Ashcombe requires 318 kN; this consist produces 271 kN").
5. **Length limits.** Halt platforms accept max 6 vehicles; Depot platforms max 12; Hub platforms max 20. Exceeding it means a *partial platform stop* — some doors can't open, and passengers for those carriages get off during an unscheduled extra stop, costing time and comfort.

---

# 5. Quest & Storyline System

### 5.1 Three concurrent content layers

The game runs three systems at once. They are separate in code, separate in UI, and never conflated in the player's head.

| Layer | Source | Count at 1.0 | Player agency | Persistence |
|---|---|---|---|---|
| **A. Authored Quests** | Hand-written JSON + dialogue | ~8 arcs / 42 chapters / ~120 quest steps | Full order freedom; arcs advance on completion, not on schedule | Permanent, save-persistent, one-shot |
| **B. Contracts** | Procedurally generated from economy data | Effectively infinite; ~30 offered at any moment | Total — accept, refuse, abandon, chain, renegotiate | Ephemeral; expire |
| **C. Emergent Beats** | `EventDirector` reacting to state | Continuous | Response or ignore | Logged to Chronicle (§5.7) |

**Balance target:** in a 2-hour session, roughly **70% of playtime on contracts**, **20% on quest content** (as deliveries that *are* the quests, not detours from them), and **10% on emergent events**. Quests are never a separate game mode bolted on the side — **a quest in `IRONHORIZON` is a contract with a story attached, and its completion is always a delivery you could have made anyway.**

### 5.2 How stories are discovered

Four discovery vectors, all diegetic:

1. **Passengers you carry.** A recurring NPC boards your train (their appearance is scheduled by arc state, not by randomness). The first occurrence usually has no marker at all — just a person who says something memorable. Only after the second encounter does the arc register as a **Thread** in the Chronicle.
2. **The radio.** Dispatcher calls, regional gossip, a colleague's complaint. Some arcs start because you answered the radio when you didn't have to.
3. **Physical places.** A Platform 3 noticeboard, a note taped to a signal box door, a works plate on a bridge, an abandoned siding you noticed. Requires the player to be out of the cab (§3.8 walkable spaces).
4. **Contract anomalies.** A freight contract with an unusual note ("do not open the third wagon"), a passenger contract that specifies a route no sane person would choose, a mail contract with a stop that isn't a station.

**Hard rule:** the game never uses a floating exclamation mark over an NPC. Story presence is indicated by lighting (a personal lamp left on), sound (a radio channel has static), and a subtle Chronicle pulse. Optional means optional and also means findable-by-the-curious.

### 5.3 Recurring NPC framework (the eight arcs at 1.0)

Each NPC has: a **home station/route**, an **arc state machine** (5–8 nodes), **trigger conditions** (where/what/when), **dialogue sets per state**, **a mechanical bias** (how their quests alter the game), and **a consequence web** (what their arc changes in the world).

| # | NPC | Role | Introduced | Arc spine | Mechanical bias | Consequence |
|---|---|---|---|---|---|---|
| 1 | **Mira Halloway** | District nurse, Saltmarsh | Riding the Sable, Chapter 1 | Needs urgent passage to a cut-off village → trusts you with a medical consignment → asks you to break a rule (a night-time unscheduled stop) → you either do or don't | Unlocks **Halt-to-Halt emergency medical contracts**, and passenger-comfort margins relax when she's aboard | Medical contracts pay little but build **Civic reputation**, which lowers insurance premiums for the whole game |
| 2 | **Dovren Aelst** | Rival haulier, Ashfall | Cinderworks yard | Starts ahead of you, mocks your operation → loses a client to you → proposes a cartel → or goes under | Unlocks the **Rival Contracts** pool: high-paying, high-risk jobs he's refused | Choose to help him or bury him; affects the **Industrial Guild** reputation track permanently |
| 3 | **Superintendent Rhea Vance** | Network Rail regulator | Inspection event after 3 violations | Investigates you → either becomes your advocate or your auditor | Either way, unlocks **Licence Testing at Hub stations**; an advocating Vance gives test-day conditions in your favour | Gates the Hazmat and Express licences behind trust; creates an "inquiry" story arc if you take the corrupt option |
| 4 | **The Drifter** (Tobias Kenn) | Unlicensed night-shift character | Found in an abandoned siding at 2am | Bums a ride → asks for jobs that shouldn't be legal → reveals he is a former Aurora test engineer | Unlocks **Rare Route Intel**: hidden sidings, shortcuts, and the electrified spur map | His arc is the gateway to the Class E Aurora and the "behind the railway" side of the story |
| 5 | **Ilse Marren** | Owes you nothing / farm owner, Verdant Basin | Livestock contract chain | Her herd is dying of a disease; needs transport she can't afford → becomes the largest livestock shipper in the region if you help | Detunes livestock stress penalties on her consignments; unlocks **Seasonal Harvest Surge** contracts | Regional food supply changes; a town's economic status visibly improves/deteriorates |
| 6 | **Old Kestrel** (Aksel Kestrel, 74) | Retired driver, Kestrel Cross | 10 hours in, always on Platform 4 bench | Tells you about the line's history → asks you to recreate his last run → his health declines → a memorial run | Unlocks **Heritage Services**: Class S-hauled vintage passenger runs at ₡0.30/pax extra and a Nostalgia reputational track | The memorial run is the emotional capstone of the Verdant Basin and unlocks a unique cosmetic livery |
| 7 | **Nia Okonjo** | Children's author, riding your commuter service obsessively | Any passenger run, Chapter 2 | Writes a book about the line; asks to ride specific routes at specific weathers and times → eventually asks for a route that no longer exists | Unlocks **Postcard collection** objectives with bonus flavour text per viewpoint | Completing her arc unlocks the **Free Roam photography mode** with a proper photo toolset |
| 8 | **Quartermaster Voss** | Depot foreman, Highmoor | Visiting a Depot in Ironreach | Offers you cheap, slightly-illegal upgrades → asks for a favour (transporting a locomotive with no paperwork) → or turns you in | Unlocks the **Grey Market upgrade tree** (cheaper, slightly failure-prone parts) | Surfaces the game's moral spine: fast money and fragile machines, or slow money and solid ones |

**Recurrence mechanics in code:** each NPC has a `spawn_schedule` evaluated against world state (region, station, hour, weather, arc node, time-since-last-meeting with a minimum of 3 game-days and a maximum of 12). The scheduler prefers to place them on contracts the player is *already likely to take*, so story feels like coincidence rather than instruction. **Guaranteed fallback:** if an NPC hasn't spawned in 12 game-days, they will appear at the next eligible Hub, because a silent arc is worse than an authored coincidence.

### 5.4 Arc structure template (identical for all eight arcs)

```
ACT I  — Encounter (3–4 steps, ~40 min)
  ├─ step: coincidental meeting, no system pressure, ~3 min of dialogue in-cab
  ├─ step: a small ask that is trivially easy (carry me to X, deliver this parcel)
  ├─ step: the ask has a cost (a detour, a lost contract, a risk)
  └─ step: the arc's premise is stated; a Thread appears in the Chronicle

ACT II — Complication (4–6 steps, ~3–6 hours)
  ├─ two parallel tracks: a PERSONAL problem (their life) and a RAILWAY problem (a system constraint)
  ├─ 2–3 steps require a real skill test (a gradient, a timetable, a hazmat route, a night run)
  ├─ 1 step is a genuine dilemma with no clean answer, branching state
  └─ midpoint reversal: information changes the meaning of Act I

ACT III — Consequence (3–5 steps, ~2–4 hours)
  ├─ the arc's system changes the world (a contract pool opens, a station reopens, a region shifts)
  ├─ the final step is a long, difficult, high-value haul with a stated risk
  ├─ ending branches: 2 or 3 outcomes based on accumulated choices
  └─ epilogue: a written entry in the Chronicle, a reward (train, upgrade, cosmetic, route), and a permanent small world change
```

**Chapter 1 is always the Sable, always in Verdant Basin, always within 20 minutes of first play.** Total authored content volume target: **~48,000 words** of dialogue, all in `content/dialogue/*.json` with speaker IDs, conditions, and per-state variants. Localisation must be assumed from day one (no concatenated strings; use ICU message format).

### 5.5 Branching and consequence system (concrete, not vibes)

Five consequence channels, each with visible feedback:

1. **Reputation tracks** (6, see §6.5) — mechanical, numeric, reversible over time.
2. **World state flags** — booleans/quaternions on the world: `station_kingsbury_reopened`, `quarry_access_repaired`, `voss_arrested`, `mira_trust = 3`. Stored in the save; ~400 flags at 1.0.
3. **Route state** — some edges physically change: a repaired branch, a lifted branch, a new siding, a line closed by a hazmat incident. **Route state changes are always additive or substitutive, never purely subtractive** — if a line closes, an alternative is always available, and the closure is a story beat with a reason.
4. **Contract pool modulation** — arcs and reputations change weights in the contract generator (e.g., `we_trust_you` raises the frequency of high-value fragile cargo; `blacklisted_by_guild` removes the industrial pool entirely for 20 game-days).
5. **NPC condition** — some NPCs can die, leave, forgive, or retire. **No NPC death is a failure state.** Deaths are always the result of a long chain of player choices, never one moment, and never a "you picked the wrong dialogue option" — the design rule is: *a player should be able to explain, in one sentence, why what happened, happened.*

**Three-ending precedent:** every arc has at least two endings; two arcs (Mira, Dovren) have three. Endings are **not** signposted with a "this will matter" warning, but **all irreversible choices are preceded by unmistakable tone shift** — the HUD's Chronicle icon engages, the music drops to a single instrument, and the dialogue camera slows down. Invisible consequences are fine; invisible *irreversibility* is not.

### 5.6 Contract generation (layer B in detail)

The generator is a scored, weighted, constrained selection process over a candidate space. It runs on a per-Hub/per-Depot tick and refills boards to 12–20 (Hub) and 4–8 (Depot).

```
candidates = all valid (origin, destination, commodity, mass, urgency)
  filtered by:
    - origin in this station's node set OR reachable within 3 km of track
    - destination reachable by at least one path valid for the player's licence class
    - mass within the player's total fleet capability OR a rented/loaned train is offered
    - commodity's seasonal window (if any) is active
    - not blocked by current route state (line closures, TSRs, curfew)
  then weighted by:
    - reputation modifiers (see §6.5)
    - contract_pool_modulation flags from world state
    - a "freshness" penalty for contracts similar to the last three completed
    - a "novelty bonus" for destinations the player has not visited in 10 game-days
    - a "route interest" score: contracts whose best path includes a new
      feature (viaduct, spiral, tunnel, new region) are boosted ×1.6
  and finally 2-4 slots per board are RESERVED for:
    - exactly one quest-linked contract when an arc is awaiting a delivery
    - exactly one "awkward" contract (hard, low pay, high reputation) to make the board feel human
```

**Anti-degeneracy rules the generator must respect:** never offer more than 3 contracts on the same route in a row; never offer a contract whose payout-per-hour is below ₡180 unless it has a story or reputation justification; never offer a contract that the player's *only* train physically cannot complete without a stated warning label; always guarantee at least one contract is acceptable with zero preparation.

### 5.7 The Chronicle (the game's memory and journal)

A single in-game document, opened with `J`, that records:

- **Threads** — active and completed arcs, with the step the player is on and a one-line "what happened so far" in plain language.
- **Ledger** — the last 60 runs with full scorecards, searchable by route and commodity.
- **Registry** — trains owned, upgrades installed, licenses held, stations visited, halts stopped at, viewpoints collected, postcards earned.
- **World** — active route closures, TSRs, weather outlook, seasonal status, current reputation bands.
- **Log** — a plain-text timeline of significant events with timestamps (the player's own story of their game, autogenerated).

The Chronicle is the single most important UX artefact after the HUD. It is what makes a 60-hour open-world game comprehensible after a two-week break. It must be fully implemented by Phase 13, not bolted on at the end.

---

# 6. Progression & Economy

### 6.1 Currency overview

| Currency | Symbol | Earned by | Spent on | Notes |
|---|---|---|---|---|
| **Credits** | ₡ | Contract payouts, regenerative credits, selling old trains | Trains, fuel, repairs, upgrades, crew transfers, licences, accessories, fines | Single primary currency. No premium currency, no real-money purchases, no loot boxes, ever. |
| **Reputation** | 6 tracks, 0–100 | Run quality, punctuality, incident handling, quest choices | Unlocks contract tiers, better rates, test waivers | Not spendable — it *modulates* everything. |
| **Licence Points** | 0–5 | Passed licence tests | Endorsements: Passenger, Freight, Livestock, Hazmat, Express | Each is a *gate on contracts*, never on geography. |
| **Wear** | 0–100% per asset | Accumulates automatically | Repairs at Depot/Hub | Poor maintenance raises the failure-event rate — the main source of tension in the economy. |
| **Chronicle completion** | 0–100% | Collection goals | Cosmetics, postcards, paint schemes, horn packs | The "comfort" progression track for players who don't want to optimise money. |

### 6.2 Payout formula (the single source of truth)

Every contract's settlement is computed by this formula. Implement it in `sim/economy/payout.ts` and expose every term in the scorecard UI.

```
base_payout        = commodityRate[commodity] × (mass_t)
                   + passengerRate[class] × (pax_count)
                   + distancePay ₡1.90 × route_km
                   + urgencyBonus (see table below)

progress           = 1.0 − 0.5 × clamp(overdue_fraction, 0, 1)   // progressive, never binary
timeliness         = 1.0 + 0.30 × clamp(early_fraction, 0, 1) − (late ? 0.5 × late_fraction : 0)
comfort_mult       = 0.70 + 0.30 × comfort_index      // passenger contracts only
condition_mult     = Π over cargo units of (1 − damagePct × severity[commodity])
compliance_mult    = 1.0 − 0.12 × violations − 0.4 × uncaught_manifest_errors
reputation_mult    = 0.85 + 0.0035 × relevant_reputation   // 0.85 at rep 0, 1.20 at rep 100
mode_mult          = 0.80 (Assisted) | 1.00 (Engineer)
chaining_mult      = 1.06 if chained within 10 game-minutes of arrival
control_assist_max = 0.60 if Cruise Assist was engaged at any point

gross              = base_payout × progress × timeliness × comfort_mult × condition_mult
                     × compliance_mult × reputation_mult × mode_mult × chaining_mult
net                = gross − fuel_cost_paid − wage_cost − fines_and_penalties
```

**Urgency tiers:** Standard `+0%`, Express `+35%`, Critical `+90%` (with a hard deadline and a real chance of failure). **Rare commodities** (concert grand piano, racehorse, prototype turbine, 1880s signal cabin clock) exist as one-off contracts with base payouts of ₡6,000–₡22,000 and bespoke damage models.

**Anti-grind guarantee:** the payout curve has diminishing returns on repetition. Repeating an identical route+commodity yields `×0.94` per repeat within 3 game-days, bottoming at `×0.70`. Combined with the generator's freshness penalty, the highest-paying move is almost always to explore something new. There is no "optimal farming loop," by design.

### 6.3 Costs (what money disappears into)

| Cost | Amount | Notes |
|---|---|---|
| Diesel | ₡2.10 / L | Fuel is the single largest ongoing cost. Class F burns ₡1,500 per full run; Class E burns nothing but pays catenary access charges. |
| Catenary access | ₡0.42 / kWh drawn | Applies to Class E. Regenerative credits offset roughly 40–55% of this on mountainous routes. |
| Crew wage | ₡180 / game-hour beyond the first 6 | Discourages marathon single sessions in-fiction and creates a real cost to Deadhead Time-Skip. |
| Routine maintenance | ₡0.34 per km + per-component wear cost | Bogies, brake shoes, traction motors, injectors. Skipping it is possible and profitable until it isn't. |
| Repairs | ₡350–₡18,000 | Scales with damage severity and class. Hazmat incident repairs can exceed ₡60,000. |
| Upgrades | ₡1,200–₡38,000 per tier | See each class's path in §4. |
| Licence tests | ₡4,000–₡22,000 | Plus a practical exam run. |
| Insurance premium | Percentage of contract value, 3.5%–11% | Drops with Civic reputation and a clean incident record; rises sharply after any hazmat event. |
| Fines | ₡400–₡9,500 | Speed violations, horn at forbidden locations, curfew breaches, manifest errors, unhandbraked rollaways. All fully avoidable and fully telegraphed. |
| Trains (used/new) | ₡22,000–₡372,000 | A used Class P with 61% wear costs ₡41,000 and will need ₡9,000 of work — a real, interesting decision. |

**Balance checkpoint:** a competent player should net **₡9,000–₡16,000/hour** in the first 5 hours, **₡26,000–₡38,000/hour** at mid-game with 2–3 trains, and **₡58,000–₡85,000/hour** late-game with a fleet and high reputation. Total cost to acquire and fully upgrade all 7 classes: **₡1.42M**, i.e. realistically 35–55 hours. Progression is designed to complete *before* burnout, not after.

### 6.4 Licence structure (5 endorsements)

| Licence | Cost | Requires | Test | Grants |
|---|---|---|---|---|
| **Light Rail** | — | Owned at start | Auto-completed | Class S, freelance passenger/freight contracts up to 12 t |
| **Passenger** | ₡4,000 | 12 completed contracts, Passenger Assoc. rep ≥ 25 | A 3-stop practical run scored on punctuality and comfort ≥ 0.80, plus a 12-question regulation quiz | Class P and E operation; passenger contracts |
| **Freight** | ₡7,500 | 20 contracts, Industrial Guild rep ≥ 30 | A laden 600 t hill start, a descent using dynamic braking only, and a brake-pipe loss recovery drill | Class F; heavy contracts above 60 t |
| **Livestock** | ₡6,000 | 8 livestock-adjacent contracts, Animal Welfare ≥ 40 | A 20 km welfare-graded run with stress never exceeding 45 on any car; plus a veterinary handling questionnaire | Class L; livestock and cold-chain produce |
| **Hazmat** | ₡22,000 | Freight licence, Regulator rep ≥ 55, no incidents in last 40 game-days | A written exam (24 questions), a supervised route through Z1 and Z2 zones at exact limits, and a manifest audit drill | Class H; regulated and hazardous commodities |
| **Express** | ₡18,000 | Passenger licence, Comfort ≥ 0.90 on three consecutive runs, Postal Service rep ≥ 50 | A timed 90 km parcel run with a 4-minute handoff window and zero unscheduled stops | Class M and Class E; express and mail contracts |

**Tests are playable, not quizzes with a side of driving.** Each is a real run with an examiner NPC in the cab who comments in real time, and each is retakeable after a 2 game-day cooldown, without losing progress. Failure is instructive; the examiner tells you exactly what went wrong.

### 6.5 Reputation tracks (6) and their concrete mechanical effects

| Track | Raised by | Bands (D 0–25 / C 26–45 / B 46–70 / A 71–90 / S 91–100) |
|---|---|---|
| **Passenger Association** | Comfort > 0.85, on-time arrivals, clean announcements, refusing to overload | ×0.85 / ×0.95 / ×1.08 / ×1.18 / ×1.30 on passenger fares; boarding time decreases; standees tolerated more readily |
| **Industrial Guild** | Heavy freight on time, low damage, refusing shoddy loading, helping Dovren | Unlocks higher-tonnage contracts; depot discounts 0–22%; hazmat referrals |
| **Postal Service** | Custody windows met, no stops, honest tamper declarations | Express contract availability ×3 at band A+; rare high-value consignments |
| **Regulator (Network Rail)** | Zero violations, clean inquires, honest manifest correction | TSR grace periods, faster licence approval, insurance −1.8%, optional "trusted operator" status that removes some inspections |
| **Animal Welfare Board** | Low stress runs, ventilation discipline, refusing cruelty | Livestock rate ×1.25 at S; cheaper veterinary support; unlocks the largest herds |
| **Civic / Regional** (per region, 4 sub-tracks) | Emergency medical runs, reconnecting cut-off towns, quest choices | Regional grants, station upgrade events, lower insurance, and *visible world changes* — a town's lights stay on, a station reopens, a factory restarts |

**Reputation decays** at 0.4 points per game-day toward a floor 25% below its peak if you don't operate in that channel. It is recoverable at 1.5× the decay rate — reaching a band is a commitment, not a one-off. **No reputation band ever removes content.** A bad reputation makes contracts rarer and cheaper; it never locks a train, route, or region.

### 6.6 Unlock structure (everything, in order, with zero geographic gating)

| Gate | Unlocks | Spends | Player feels |
|---|---|---|---|
| Hour 0 | Class S, all 48 stations reachable, Contract layer, Chronicle | — | Freedom from minute one |
| Contract 5 | Depot job boards, first upgrade tier | ₡ | "I can improve my machine" |
| Contract 12 + ₡4,000 | **Passenger Licence** → Class P | ₡ | "I can carry people properly now" |
| 20 contracts + ₡7,500 | **Freight Licence** → Class F | ₡ | Scale: mass becomes the game |
| ₡48,000 | **Cruise Assist** | ₡ | Long-haul quality of life |
| First quest arc complete | Region reputation unlocks + a free cosmetic livery | Story | "The world noticed me" |
| ₡88,000 | Class L (Livestock) | ₡ | A different kind of responsibility |
| ₡118,000 | Class F full upgrade path (or ₡118k toward it) | ₡ | Mastery of mass |
| ₡156,000 | Class M (Courier) + Express Licence | ₡ | Precision and time |
| ₡204,000 + Regulator ≥55 | Class H (Hazmat) | ₡ + Reputation | Seriousness and consequence |
| ₡372,000 + Renewables endorsement (from the Drifter arc) | Class E (Aurora) | ₡ + Story | Endgame speed and network puzzle |
| Chronicle 100% | Cosmetic rewards, a special memorial run (Kestrel) | Play | Completion, not dominance |

**Structural guarantee:** at every stage, the *entire map* and *every route* remains available. What unlocks is the vocabulary — new cargos, new constraints, new schedules — never new territory.

---

# 7. Hazards, Events & Systemic Depth

### 7.1 The event taxonomy

Every event in the game belongs to exactly one class, has one **telegraph**, two or more **responses**, and a defined **outcome range**. Nothing kills the player; everything costs them.

| Class | Cadence | Purpose | Telegraph lead time | Can end a run? |
|---|---|---|---|---|
| **Micro** | Every 30–120 s | Texture, feedback, life | 2–5 s | No |
| **Ambient** | Continuous | Atmosphere, information | Passive | No |
| **Minor** | 1 per 8–15 game-min | Attention demand | 8–20 s | No (costs time/money) |
| **Major** | 1 per 20–40 game-min | Real tension, planning failure | 15–60 s | Yes, in extreme cases (derailment) |
| **Crisis** | Rare, ~1 per 6–10 game-hours | Story, memorability, world change | 30 s – 3 min | Yes (or forces a recovery sequence) |
| **Opportunity** | 1 per 10–20 game-min | Agency, profit, risk | 10–30 s | No |

### 7.2 Micro events (the texture layer — high frequency, zero punishment)

Bird flocks rising from the marsh ahead; a hare running along the ballast; a level-crossing barrier sequence triggering on your approach; a station cat; cab radio chatter on an unrelated channel; a sun-flare through a viaduct arch; the platform lamp that flicks on as you approach; a *clickety-clack* rhythm change over jointed rail; a dog that barks at the train in a village you pass 200 times. These exist to make the world feel inhabited and to reward attention. None of them affect state.

### 7.3 Minor events

| Event | Telegraph | Player response | Cost |
|---|---|---|---|
| **Sheep on the line** | White specks on the far-side embankment, bleating audio at 900 m | Horn (long) — they scatter; braking; slow pass | 40 s lost or a welfare-regulator note if you strike one |
| **Level crossing misbehaving** | Flashing amber before barriers drop, or barrier stuck | Stop and report (₡90 fine waived, +1 Regulator), or pass at <10 km/h with horn and risk a strike event | 2 min or a 1-in-8 strike |
| **Signal at caution** | Aspect visible from 600 m | Slow to caution speed, prepare to stop | Blocked by AI traffic; use of the passing loop, +90 s |
| **Unattended trolley on track** | Visible from 500 m, out of gauge | Two-tone horn and stop; get out and clear it | 3 min, +1 Industrial Guild rep |
| **Late passenger running for the train** | Silhouette sprinting on the platform | Hold doors 15 s | −petulance risk vs +0.01 Civic rep |
| **Track defect (visible)** | Foreman's yellow mark, audible thump | Slow to 20 km/h and report it | +2 Regulator rep |
| **Wet rail leaf mulch** | μ gauge drops to 0.90, audio changes | Sand early, notch down | Slight time loss |
| **Rapidly deteriorating weather** | Barometer, sky colour shift, distant thunder | Route re-planning, speed reduction | Real, measured braking-distance changes |

### 7.4 Major events (the tension layer)

**1. Wheelslip on a grade (Class F, L, H).** Telegraph: slip-warning lamp flickers, audio pitch of the motors rises, speed falls while power is applied. Responses: (a) sand — costs 20 s of sand reservoir, fixes it; (b) reduce notch to the adhesion limit shown on the HUD; (c) take a run-up by backing off and re-attempting. Escalation to derailment requires ignoring the warning for ~12 s on a curve above 1:60.

**2. Runaway on a descent.** Telegraph: brake-pipe pressure not recovering, speed rising while you're in full brake, retarder temperature red. Responses: dynamic braking in alternation, sand, deliberate derailment into a **catch point siding** at a known location (a real, marked, survivable escape route — the game tells you where these are on the map), or accept a cost. This is the single most dramatic scenario in the game and must be designed as a set-piece.

**3. Derailment.** Rolled against a hidden derailment-risk score derived from: curve radius vs speed excess, track condition, consist loading quality, recent maintenance, and lateral forces. Telegraph: a build of lateral-accel warnings before the incident point. **When it happens it is a sequence, not a fail state:** emergency brake → collision/roll → inspection on foot (first-person, walk the consist, tag damage with a survey tool) → call for relief via radio (a crane and recovery crew take 45–180 game-minutes to arrive) → an insurance call with an NPC assessor (your reputation and honesty choices matter here) → an incident report (an authored document you fill out) → a fine, a repair bill, and a *story hook* because something always happened. Typical total cost: ₡6,000–₡40,000, one to three game-days of downtime, and a permanent entry in the Chronicle.

**4. Mechanical failure.** Six failure modes: traction motor overheating, brake rigging binding, injector/pump failure, axle bearing run-hot (**detected by stopping and touching the axle — a real pre-departure ritual**), traction motor flashover after water ingress, and a coupler shear. Telegraph: audio, gauge, and smell/vibration cues (yes, use audio cues). Responses: limp to a siding at reduced speed, field repair with a Utility Module (60% quality), or call for relief. Preventive maintenance is the counterplay, and it's priced.

**5. Signal passed at danger (SPAD).** Telegraph: countdown to the signal, DRA alarm escalating, the game lets you fail. Response: acknowledge late and brake hard. Consequence: emergency brake applies, a Regulator inquiry event fires, and this is one of the few events that can cost a licence temporarily.

**6. Rival operator conducts a job on the same block.** Telegraph: radio traffic, the AI train appears on the occupancy map 3–6 km ahead. Response: wait, use the loop, or reroute. The rivals are *not* hostile — they are competitors; the game is not a combat game.

**7. Cargo damage in transit.** Telegraph: the condition bar dips as the accelerometer records an event. Responses: drive more gently (this is entirely in the player's hands), reroute to a smoother line, or accept a partial payout. The *piano problem* is the archetype: a 480 kg instrument in a wooden crate occupying one wagon, whose damage model is a cumulative-energy integral — you can carry it across the continent intact if you are gentle for four hours.

**8. Livestock panic.** Telegraph: stress dial rising fast, audible animal noise. Responses: stop (stress starts falling only when stationary), reduce speed, open vents, use water troughs. A panic event can force an unscheduled 4-minute stop at a Halt, which costs a schedule window.

**9. Catenary fault (Class E).** Telegraph: ammeter spikes, lights flicker. Responses: coast, drop the pantograph, wait for the section to re-energise, or (with the battery tender) continue at 60 km/h on battery for 14 km.

**10. Bridge strike or structural suspicion.** Telegraph: signage, a load-limit plate, a fresh crack indicator visual. Responses: stop, inspect (a short first-person sequence), or divert via the Ring Line. A bridge event that closes a route is a *world* consequence, not just a personal one.

**11. Level crossing vehicle incident (road user).** Telegraph: none — it is instant and is supposed to be, but it occurs only where the player had a horn cue and a sightline. Responses: emergency braking (may or may not be sufficient given real stopping distances — and the game does not cheat; if the physics says you cannot stop, you were warned 1.2 km back). Consequence: minor derailment risk, an inquiry, emotional weight in the story layer, and, crucially, **the game never blames the player for physics they could not beat** — the event is generated only in situations where the player had a legitimate choice.

**12. Line closure from weather (avalanche, flood, sea surge).** Telegraph: forecast in the Chronicle's weather outlook, a warning broadcast on the radio 1–4 game-hours ahead. Responses: don't go; go the long way; go and wait in a siding. A closed line stays closed for 1–6 game-days and reshapes contract supply across a whole region — the best systemic consequence in the game.

### 7.5 Opportunity events (the agency layer)

| Event | Offer | Risk | Reward |
|---|---|---|---|
| **Hot charter** | A last-minute unscheduled passenger charter, 20 min notice | Punctuality streak reset if you miss any stop | ₡2,400–₡9,000, +Civic rep |
| **Rush consignment** | An extra wagon offered for free if you can reach its origin in 40 game-min | Requires a shunt and a coupling in the yard, +22 min | +38% on the run |
| **Salvage run** | A derailed wagon's cargo needs recovering from an abandoned siding | Access is via poor track at 15 km/h | Rare commodity, +Guild rep |
| **Regulator audit** | A voluntary audit request | Costs 30 min | +6 Regulator rep, future inspection relief |
| **Rival's refusal** | Dovren offers you a job he can't or won't do | Genuinely dangerous (curfew breach, overweight, no paperwork) | Very high payout, reputation risk |
| **Night shift** | Ashfall refineries need constant night haulage | Curfew rules, night visibility, deer risk | ×1.4 payscale |
| **Heritage service** | A vintage passenger run behind your old Sable | Slow, but must be punctual and gentle | Nostalgia reputation, a rare cosmetic, one of the best-written quests |

### 7.6 Systemic depth: things the player will discover on hour twenty

These are not signposted anywhere and exist to reward deep play. Deliberately undocumented in-game:

- **Coupler slack as a tool.** Accelerating and coasting in phase lets you "stretch" a consist before a climb, recovering ~15% of grade-climbing energy on rolling stock with loose drawgear — the classic railroad technique, modelled properly.
- **The hump.** Cinderworks' classification hump is a real, working miniature gravity yard. You can use it to sort your own consist with no locomotive power.
- **Weather-led routes.** Some contracts are 40% cheaper to run in a specific weather because of air resistance and adhesion. A player who learns to pick routes *by forecast* saves thousands of litres.
- **Track knowledge as a skill.** The game never marks curves. After a few runs, players know that the approach to Greybridge dips at the 6th arch and that Marlbrook needs braking at the white post. This is real mastery and must be preserved: **do not add braking-point markers in Engineer mode, ever.**
- **Reputation synergies.** Animal Welfare band A raises Civic rep in Ironreach (the agricultural region), which lowers the insurance premium.
- **Emergency sluicing.** Opening and closing doors at a specific platform manoeuvre lets a Hazmat train cool its tank cars by 8 °C. It's real physics, it's fiddly, and it saves a pressure event.

---

# 8. Art & Audio Direction

### 8.1 The visual thesis

**"A model railway photographed by somebody who loves them."**

The target is not crude low-poly; it is **deliberate, plane-count-limited, colour-coded, and lit with intent.** Every asset must look like a design decision, not a low-budget compromise. The three tests each asset must pass:

1. **The Silhouette Test** — squint at it (or view it at 40% size): is the object still identifiable? If a locomotive's silhouette isn't instantly readable at 200 m in fog, it is wrong.
2. **The Flat Test** — render it unlit with flat vertex colours and a black outline. Does it still look good? If the asset only reads with shading and texture, it has no form and must be rebuilt.
3. **The Noise Test** — at the player's normal camera distance, is there any visual noise? Low-poly fails when small geometry clusters into visual static. Reduce, simplify, or remove.

### 8.2 Geometry budget (hard limits, enforced by an asset-lint tool in CI)

| Asset class | Triangle budget | Material count | Texture policy |
|---|---|---|---|
| Locomotive (hero, Class A–E) | 6,000–14,000 | 2 | **No texture maps.** Vertex colours + 1 shared palette atlas for decals only. |
| Coach / wagon / tank car | 1,200–3,600 | 1 | Vertex colours |
| Station building (Hub) | 3,000–9,000 | 2 | Vertex colours + atlas decals (signage) |
| Station building (Halt) | 400–1,200 | 1 | Vertex colours |
| Trackside prop (signal, sign, hut, lamp) | 60–600 | 1 | Vertex colours |
| Vegetation cluster (tree/shrub/reed group) | 120–800 | 1 | Vertex colours, alpha for foliage cards only |
| Terrain chunk (1 km²) | 12,000–45,000 | 1 | Vertex colour + procedural detail (no baked AO beyond cheap vertex AO) |
| Distant LOD impostor | 40–300 | 1 | None |
| Character (NPC, ~120 on screen max) | 900–2,600 | 1 | Vertex colours; face features via 4-flat-plane palette swap |
| Whole-screen budget at 1080p | ≤ 1.8M tris drawn, ≤ 1,100 draw calls | — | — |

**Polygon philosophy:** straight lines are straight; curves are **chamfered** with 4–8 segments, not smoothed. Circles (wheels, tanks, chimneys) get 12–16 segments. Never use a triangle where a quad works. Bevel every hard corner by 2–4 cm with a chamfer — this catches the specular highlight that makes low-poly look intentional rather than cheap. **Bolts, rivets, cabling, and grilles are geometry, not textures** (at 40–200 tris each).

### 8.3 Colour palette philosophy

Four rules, applied per region:

1. **Rails and rolling stock are near-neutral.** Locomotives are in livery colours (see below); the permanent way is grey-brown-black. This makes the moving thing the most colourful object in every frame, which is exactly the point of a train game.
2. **Every region has one dominant hue, one accent, and one forbidden hue.** Forbidden hues are reserved exclusively for signals, hazards, and UI so they always pop.
3. **Atmosphere does the heavy lifting.** Colour comes from fog, sun angle, and sky gradient — not from textures. Two regions can share a great deal of geometry and feel completely different because of a single sky gradient shift.
4. **Saturation is capped at 78% and floors at 12%.** Nothing is fluorescent; nothing is mud.

| Region | Dominant | Accent | Forbidden (reserved) | Sky/fog gradient |
|---|---|---|---|---|
| **Verdant Basin** | Sage / leaf green `#6F8F5E` | Honey gold `#C9A44C` | — | Pale blue → warm cream haze |
| **Ironreach** | Slate blue `#5A6B7C` | Granite rose `#9C7C72` | — | Steel grey → cold violet dusk |
| **Saltmarsh** | Reed beige `#B9AC8C` | Muted teal `#5E8A8E` | — | Silver-white → apricot |
| **Ashfall** | Charcoal `#3A3A3E` | Furnace orange `#B4552B` | — | Smog amber → black |
| **All regions (reserved)** | Signal green `#2ECC5A`, caution amber `#FFB020`, danger red `#E03A2F`, hazard yellow `#F5D90A` | | | Signs, aspects, hazards, UI — **never** in world geometry |

**Livery system (7 base schemes, all player-swappable, ~18 unlockable cosmetics):** Sable (weathered olive), Peregrine (deep blue/cream stripe), Foundry (railway maroon), Courier (postal cream/scarlet), Pastoral (dun with a green band), Cinder (regulation white-tank/graphite-frame with hazard placards), Aurora (electric white/silver with a warm amber trim). Livery changes are free after purchase and are the primary cosmetic reward channel.

### 8.4 Lighting approach

- **One sun, one moon, static-forward-plus-cascades.** 3 shadow cascades (near 0–60 m, mid 60–220 m, far 220–650 m), 2048/2048/1024 resolution. Beyond 650 m: no shadows, rely on baked vertex AO.
- **Time-of-day is a single interpolated gradient plus a sun-transform.** Store the day as keyframed values in `content/lighting.json`: sun azimuth/elevation, sun colour, ambient sky/ground colours, fog colour, fog density, exposure. 24 keyframes per season (96 total). **This is the entire mood system — treat it as content, not code.**
- **God rays via a cheap screen-space radial blur** on the sun sprite, only enabled when the sun is within 35° of the camera forward and above 5° elevation. Cost: one full-screen pass at quarter res. Never on in Blizzard or at night.
- **Emissive at dusk and night**, always: station lamps, signal aspects, locomotive marker lights, window glass, the headlight cone, a lit cabin in a far-off farmhouse, a quay crane in Saltmarsh. The single biggest factor in making a low-poly night look expensive is having 15–40 small emissive sources visible at all times.
- **Headlight cones are real.** Pre-baked cone mesh with a soft additive falloff plus a spotlight — the player's primary night tool, readable at 1,200 m by oncoming AI traffic.
- **Weather passes:** rain as a full-screen overlay with 3 parallax layers + wet-surface specularity boost + a windscreen wiper mask (the wiper *is* animated geometry because the player is in the cab; this is the detail that sells the whole game); snow as a world-space particle field with wind-driven drift that accumulates on flat surfaces via a shader uniform; fog as exponential-squared with a directional bounce light; blizzard as fog + heavy particles + wind streaks + a hard vignette that shrinks with speed.
- **Post-processing, exactly four effects and no more:** ACES tonemapping, a 4-tap bloom (threshold 1.1), a subtle chromatic aberration on the screen edge (0.12 px max), and a fast approximate anti-aliasing pass. **No motion blur, ever.** A stretched frame hides the one thing the player is reading: forward motion and curve geometry.

### 8.5 Camera direction

| Camera | Use | Behaviour |
|---|---|---|
| **Cab** (default) | Mastery | First-person seated. Parallax on head-turn. Windscreen with animated wiper, rain droplets, vibration at speed, and an interior modelled at 2,000–4,000 tris with real gauges reading real sim values. |
| **Chase** | Approach and station work | 12 m behind, 4.5 m up, spring-damped, speed-scaled FOV (60°→72°), slight roll into curves (max 2.5°), dust/ballast spray at speed |
| **Cinematic** | Long straights and beauty | Auto-cuts every 8–22 s between 6 preset shot types (3/4 front, low wheel-level, side pan, drone-follow, platform-wait, rear-departure). Only engages above 30 km/h on a straight with 2 km of clear track. |
| **Free-fly / Drone** | Photography mode (unlocked via Nia's arc) | Full 6-DOF, no clip, 120 m leash, held-position mode, and a proper photo tool (FOV, filter LUT, framing guides, save-to-album at 4K). |
| **Walk** | Walkable spaces, incident sites, cab-to-platform | First-person, 1.6 m eye height, 3.1 m/s walk, no jump, a genuine walk cycle with weight. |

### 8.6 UI / diegetic design

- **Gauges are geometry.** The HUD's primary readouts (speed, brake pipe, traction current, next signal, gradient) exist as physical instruments in the cab. The screen-space HUD is a secondary, optional, minimal overlay: the gradient strip, the next-signal box, the comfort/condition meter, and the objective line. Three lines of text maximum on screen at any time.
- **Typography:** one humanist sans (Inter or Söhne Display) for UI, one condensed grotesque (Roboto Condensed) for signage and timetables, one handwritten/typewriter face (Special Elite) for manifests, notes, and the Chronicle. Signage must use a condensed face because it is world geometry.
- **Map style:** topographic, 2–3 colours, contour lines every 50 m, track drawn in a heavier weight than roads, gradient shown as a colour band along each edge (4 bands), block occupancy as a pulse. Should look like a printed railway timetable, not a video-game minimap.
- **Sound is UI.** A change in the audio bed is a legitimate information channel (§8.7).

### 8.7 Audio direction

**Engine and rolling stock (the game's core sound):**

| Layer | Source | Behaviour |
|---|---|---|
| Traction (diesel) | Recorded engine at 4 load bands | 3–4 looped layers crossfaded by RPM/load; ±4% randomised pitch per vehicle to avoid phase-lock on multi-unit consists |
| Traction (electric) | Synthetic + recorded hum | Harmonics rise with speed; a distinct "choir" tone above 120 km/h; inverter whine on regen |
| Rolling noise | Wheel-on-rail | Speed-dependent broadband layer + a jointed-rail click layer whose *tempo is the speedometer* — the player hears their speed before seeing the gauge |
| Curve squeal | Positional, per-bogie | Above 1.05 m/s² lateral accel; this is a real gameplay cue (reduce speed) and also affects livestock stress |
| Brake | Air release, shoe friction, retarder growl | Positional along the consist; a delayed hiss travelling back through the train is exactly how braking *feels* |
| Coupler slack | Recorded impact set | Plays on run-in/run-out with a force-linked volume. On a 600 t consist it should be felt, not heard. |
| Cab ambience | Per-loco | Ancient DMU rattle, freight loco deep idle, electric near-silence. This is characterisation. |
| Environment | Positional and reverb-zoned | Station concourse, valley (long delay), tunnel (short hard reverb + doppler + air-thunder), viaduct (wind-only, exposed), marsh (birds, water) |

**Reverb:** one convolution reverb with 5 switchable impulse responses (outside, station, valley, tunnel, cab-interior), crossfaded by zone trigger volume. Cheap, and it is 60% of "place."

**Adaptive music:** a **stem-based, intensity-layered** system. Four stems: *rhythm* (hand percussion, soft pump), *bass* (upright, muted), *lead* (a single instrument — guitar, harmonica, fiddle, or vibraphone depending on region), *pad* (strings/synth). Intensity `0–1` is a continuous value driven by:

```
intensity = 0.30 × speedNormalised
          + 0.25 × hazardProximity      // any warned hazard, ramping as risk grows
          + 0.15 × schedulePressure     // lateness and deadline tightness
          + 0.15 × weatherSeverity
          + 0.15 × narrativeWeight      // set explicitly by story beats
```

with hard overrides: hazard events spike intensity to ≥0.85 within 2 s and decay over 20 s; the score **drops to a single solo instrument** during any irreversible story choice; music fully fades to ambience when the player is stationary at a Hub for 60+ s (the "you're home now" cue). Musical key and instrument palette change per region: Verdant = warm acoustic folk; Ironreach = sparse modal fiddle and drone; Saltmarsh = airy, wide, choral pads; Ashfall = industrial percussion, muted brass, minor-key synth. **Total soundtrack target: ~38 minutes of stems across 5 regional palettes.** Music must never start on a haul's first 90 seconds — let the player hear the engine first.

**Accessibility in audio:** a "quiet cab" mode that reduces engine bed by 12 dB for players who need dialogue clarity; no audio-only information (every cue has a visual twin); subtitles on by default with speaker colour-coding; a "narrative pace" mode for the visually-impaired-by-text that converts the Chronicle to fully voiced summary text.

---

# 9. Technical Architecture Recommendation

### 9.1 Engine decision

Three candidates were weighed. **The recommendation is Godot 4.x (.NET/C#) for the shipping build, with a browser/Three.js prototype allowed for Phase 1–2 only if the user explicitly prefers it.** Full reasoning, so the decision is auditable:

| Criterion | Three.js / WebGL | Unity 6 | **Godot 4.x (.NET)** |
|---|---|---|---|
| Suitability for large open world | Weak: no built-in streaming, LOD, occlusion, or scene tooling. Must be hand-built. | Strong: HDRP, Addressables, terrain tools, mature streaming. | Moderate–strong: 4.x has `MultiMesh`, built-in LOD, occlusion culling, and a workable terrain approach; less turnkey than Unity but entirely sufficient at this scale. |
| Train physics on rails | Must write everything; JS float precision is fine at 1:1 scale. | Excellent, full control over fixed-step RigidBody or custom solver. | Excellent; a fully custom kinematic solver is idiomatic in C#/GDScript and avoids fighting a physics engine. |
| Build/iteration speed for AI-assisted incremental dev | Excellent: no compile, browser preview, tiny diffs. | Poor: long import/compile cycles, license activation, heavy store assets, big binary. | Good: small editor, fast iteration, no license gate, project is plain text. |
| Team-scale ceiling | Low: at 1,250 km of track + 48 stations + 7 train classes, a hand-rolled WebGL renderer becomes the project's main cost. | High: the ceiling is irrelevant; you'll never approach it. | Medium–high: comfortable for a small team and for a 2–3 year scope. |
| Licensing / cost | Free | Runtime fee risk and per-seat cost; historically volatile terms. | MIT, no fees, no seat cost, no fees on revenue, forever. |
| Shipping on desktop | Awkward (Electron/Tauri wrapper or browser-only). | Yes, native. | Yes, native, small binaries. |
| Rendering fit for stylized low-poly | Good — you control everything. | Overkill; fighting HDRP defaults for a stylized look is a real time cost. | **Best fit.** The Forward+ renderer with a custom shader for vertex-colour/flat-shaded materials produces exactly the target look with almost no fight. |

**Verdict: Godot 4.x (.NET) is the shipping target.** It wins on four axes that matter most for a stylized, deterministic, incrementally built, self-published game: licensing clarity, iteration speed, art-pipeline fit for vertex-colour low-poly, and freedom to write a custom kinematic train solver without fighting an engine's physics assumptions.

**Concession:** if the user prefers a browser-first build (immediate playable links, easier sharing), the architecture in §9.2–§9.7 is **engine-agnostic by construction** — `packages/sim/` is plain TypeScript with no engine dependency, and a Three.js renderer can consume it (`packages/render-three/`) while Godot consumes the same logic via a thin C# port, or via a WASM-compiled sim module. **The recommendation: build `sim/` in TypeScript, run the first two phases in Three.js to validate the feel fast, then port `sim/` to C# for Godot at Phase 3.** The sim's test suite ("golden runs") moves with it and is the port's correctness proof.

**Explicitly rejected:** Unreal (wrong art-fit, wrong iteration cadence, wrong cost/benefit for a stylized indie simulation), custom C++ engine (waste of the entire budget), and any engine whose working file format is a binary blob (this project's content must be diffable text for AI-assisted development).

### 9.2 Repository structure

```
ironhorizon/
├─ docs/
│  ├─ README.md                 # what this is, how to run it
│  ├─ PROGRESS.md               # per-phase log: built / verified / cut / next
│  ├─ DECISIONS.md              # every architectural decision + date + reasoning
│  ├─ OPEN_QUESTIONS.md         # pending decisions, linked to §11 IDs
│  ├─ DATA_SCHEMA.md            # generated from the schema sources
│  └─ ART_BIBLE.md              # this doc's §8, expanded, with reference images
├─ packages/
│  ├─ sim/                      # ZERO engine dependencies. Pure logic + math. (see §9.3)
│  │  ├─ src/
│  │  │  ├─ core/               # clock, RNG, fixed-step loop, event bus, state hashing
│  │  │  ├─ rail/               # graph, ribbon, switches, signals, blocks, TSRs
│  │  │  ├─ rolling/            # consist, couplers, brakes, traction, adhesion, wear
│  │  │  ├─ cargo/              # load plan, mass, fragility, comfort, welfare, hazmat
│  │  │  ├─ world/              # weather, seasons, day/night, route state, POIs
│  │  │  ├─ economy/            # payout, costs, licences, reputations, insurance
│  │  │  ├─ quest/              # quest state machines, NPC schedules, event director
│  │  │  ├─ generate/           # contract generator, NPC scheduler, world content gen
│  │  │  ├─ save/               # serialize / deserialize / migrate / verify
│  │  │  └─ api.ts              # the ONLY surface the renderer may import
│  │  └─ test/
│  │     ├─ golden/             # replayed input sequences + expected state hashes
│  │     └─ *.spec.ts
│  ├─ render/                   # engine-side: scene graph, materials, LOD, streaming,
│  │  │                         # cameras, VFX, post-processing, lighting
│  ├─ audio/                    # stem mixer, positional emitters, reverb zones
│  ├─ ui/                       # HUD, map, Chronicle, job boards, dialogue, scorecard
│  ├─ tools/                    # CLI: content validators, map builder, asset lint,
│  │  │                         # golden-run recorder, balance sim, screenshot harness
│  └─ content-pipeline/         # terrain gen, track laying helpers, prop scatter,
│                               # station kit assembly, LOD/impostor baker
├─ content/                     # ALL designer-facing data. Diffable JSON. (see §9.4)
│  ├─ trains/  commodities/  stations/  track/  contracts/  quests/  dialogue/
│  ├─ weather/  lighting/  palette/  audio/  npcs/  licences/  events/
└─ runs/                        # recorded replays for regression + balance analysis
```

**Dependency rule, enforced by a lint script (`tools/lint-deps`):** `sim` imports nothing outside itself. `content-pipeline` and `tools` may import `sim`. `render`, `audio`, `ui` may import `sim`'s `api.ts` only — never its internals. Any violation fails CI. This rule is the single most important structural decision in the project; it is what makes the sim testable, portable, saveable, and balanceable.

### 9.3 The simulation core

**Fixed step, deterministic, order-independent.**

```
while (accumulator >= FIXED_DT) {            // FIXED_DT = 1/60 s exactly
  sim.step(inputFrame, FIXED_DT);            // inputFrame is frozen & serialisable
  accumulator -= FIXED_DT;
}
render(interpolate(prevState, currState, accumulator / FIXED_DT));
```

- **RNG:** a single splittable generator; every subsystem draws from a **named, seeded stream** (`weather`, `events:region:ironreach`, `contracts:kestrel`, `npc:spawn`). Adding a new draw never perturbs existing streams — this is what allows a save file to survive a content patch without desyncing.
- **State hash:** after every 60 ticks, compute a 64-bit FNV-1a hash over the canonical state vector. Log it in dev. Golden tests assert on it. A save file stores the hash, and a mismatch on load means corruption (offer the previous autosave).
- **Determinism must be tested, not assumed:** a Nightly Sim test runs 10,000 ticks of a scripted haul on two threads and asserts identical hashes. Floating-point must use the same order of operations and no `Math.sin` approximations in hot paths (precompute or use fixed-point for anything that feeds back into state).
- **Event bus:** systems emit and subscribe to typed events (`TractionSlipStarted`, `BlockEntered`, `CargoDamageAccumulated`, `DwellStarted`, `SignalPassed`). No direct cross-system calls in `sim/`. This is what makes both the scorecard and the quest triggers trivial to implement later.

**The train solver (the heart of the game).** A train is not a rigid body — it is a chain of point masses with longitudinal couplers and 2-axle bogie constraints, moving on a 1-D arc-length track. Full method:

1. **Arc-length parameterisation.** Each `Ribbon` provides, for `s ∈ [0, length]`: `position(s)`, `tangent(s)`, `up(s)`, `curvature(s)` (signed, 1/m), `grade(s)` (signed, dy/ds), `speedLimit(s)`, `adhesionOverride(s)`, `bridgeClass(s)`, `zoneClass(s)`. Precomputed at ~2 m intervals with cubic interpolation between samples, or analytically for straight/arc segments.
2. **Per-vehicle state:** `{ s, v, mass, length, bogieSpacing, brakeForceMax, brakePipeConnected, brakesApplied, couplerState }`.
3. **Longitudinal dynamics per vehicle:** `F = F_traction − F_brake − F_gravity(grade) − F_rolling − F_davis(v) − F_curve − F_coupler`.
   - Davis equation: `F_davis = A + B·v + C·v²`, per-vehicle coefficients from `content/trains/*.json`.
   - Gravity: `F_gravity = m · g · sin(θ)` where `θ = atan(grade)`.
   - Curve resistance: `F_curve = m · g · 0.0005 · (curveRadius→ resistance coefficient)`, calibrated against published railway data.
4. **Traction:** `F_traction = min(requestedNotchForce, adhesionLimit)` where `adhesionLimit = μ(weather, s) × m_loco × 0.28`. If requested exceeds the limit, wheelslip begins: `F_traction` drops to `0.72 × limit`, a slip event fires, rail damage accrues, and `v` falls. Class F's sanding raises `μ`.
5. **Brakes:** three-part model. (a) **Propagation:** a brake application sets a target pipe pressure; it travels back through the consist at `2.4 s / 100 m`, so each vehicle's braking ramps in on a delay. (b) **Charge:** pipe recharges at `0.9 bar/s / 100 m` after release. (c) **Force:** `F_brake = brakeMass × m × g × brakeCoefficient × applicationFraction`, capped per-vehicle. Dynamic brake bypasses all this (instant, locomotive-only, heat-limited).
6. **Couplers:** modelled as springs with slack and damping. `F_coupler = k × Δs + c × Δv` when extended/compressed, clipped at a shear force. Slack of 4–14 cm per coupler per vehicle type. Run-in and run-out emerge naturally from this, and are *the* source of Class F's tension during braking.
7. **Integration:** semi-implicit Euler at 1/60 s with per-vehicle update order derived from position (so coupler forces propagate correctly along the consist in one pass), with a two-pass correction for stability. Verify the solver against the §3.6 reference braking-distance table — that test is mandatory and its numbers are a contract.

### 9.4 Content schemas (the data contracts)

Every schema lives in `content/`, validated by `tools/content-validate` in CI, with a JSON Schema file per type. Illustrative excerpts (the implementer must extend these, not restructure them):

```jsonc
// content/track/edge.kestrel_to_kingsbury.json
{
  "id": "e_kestrel_kingsbury_1",
  "from": "n_kestrel_p1", "to": "n_kingsbury_p1",
  "lengthM": 8412.6,
  "traction": "diesel",              // diesel | electric | both | none
  "electrified": false,
  "maxGradient": 0.0125,             // rise/run, signed peak
  "minRadiusM": 410,
  "bridgeClass": 4,
  "zoneClass": "Z2",
  "speedLimitKmh": 110,
  "signals": ["s_kk_1", "s_kk_2"],
  "ribbon": { "samples": 4207, "file": "e_kestrel_kingsbury_1.ribbon.bin" },
  "hazards": ["level_crossing_3", "curve_squeal_zone"],
  "sceneryDensity": 0.8,
  "services": ["p_commuter_2", "f_grain_11"]     // AI traffic assigned to this edge
}
```

```jsonc
// content/trains/class_f_foundry.json
{
  "id": "class_f_foundry",
  "name": "Foundry", "designation": "Class F",
  "role": "heavy_freight",
  "massEmpty_t": 128, "maxPayload_t": 600,
  "topSpeedLadenKmh": 80, "topSpeedLightKmh": 100,
  "traction": { "startKn": 412, "continuousKn": 268, "notches": 8, "rampSeconds": 3.2 },
  "braking": { "serviceKn": 340, "dynamicKn": 260, "emergencyFactor": 1.2,
               "pipePropagationSPer100m": 2.4, "pipeChargeBarSPer100m": 0.9 },
  "adhesion": { "baseMu": 1.0, "adhesiveWeightFraction": 0.28, "sandBonusMu": 0.11 },
  "fuel": { "capacityL": 3800, "idleLPerHour": 9.4, "rateLPer100tKm": 1.9 },
  "davis": { "A": 1650, "B": 42.7, "C": 6.1 },
  "altitudeDeratePer300m": 0.02, "altitudeDerateFromM": 900,
  "signatureMechanic": "mass_adhesion",
  "upgrades": ["tm_blower","composite_shoes","sand_hopper","drawgear","dp_unit"],
  "crew": 1, "runtimeCostPerKm": 1.64
}
```

```jsonc
// content/contracts/templates/standard_freight.json
{
  "templateId": "standard_freight",
  "commodity": "grain", "massRange_t": [180, 520],
  "urgency": "standard",
  "ratePerTonne": 4.6, "distancePay": 1.9,
  "deadlineWindowMin": [90, 300],
  "conditionModel": "bulk",           // none|fragile|perishable|livestock|hazmat
  "requiredLicence": "freight",
  "allowedClasses": ["S","F","L"],
  "reputationTrack": "industrial_guild",
  "routeRequirements": { "minBridgeClass": 3, "maxGradient": 0.020 }
}
```

```jsonc
// content/quests/arc_mira_halloway.json  (structure only; dialogue lives separately)
{
  "arcId": "arc_mira_halloway",
  "npcId": "npc_mira",
  "steps": [
    { "id": "m1_encounter", "type": "ride_along",
      "trigger": { "region": "saltmarsh", "hour": [6,20], "trainClass": ["S","P"], "minContractsCompleted": 3 },
      "dialogue": "dlg_mira_01", "autoAdvance": false,
      "next": ["m2_delivery"] },
    { "id": "m2_delivery", "type": "contract",
      "contractOverride": { "commodity": "medical_supplies", "conditionModel": "fragile",
                            "urgency": "express", "payoutBonus": 0.0, "civicRep": 4 },
      "onFail": "m2_retry", "onSuccess": "m3_request" },
    { "id": "m3_request", "type": "dilemma",
      "options": [
        { "id": "halt", "text": "Make the unscheduled halt.", "set": { "mira_trust": "+2", "regulator_violation": 1 } },
        { "id": "refuse", "text": "Keep to the schedule.", "set": { "mira_trust": "-1" } }
      ] }
  ],
  "endings": { "a": { "requires": { "mira_trust": ">=3" } }, "b": {}, "c": { "requires": { "mira_trust": "<=-3" } } }
}
```

### 9.5 Save system

- **Format:** a single versioned binary blob in a documented layout (header: magic, version, gameClock, simTick, RNG stream states, then Zstd-compressed JSON sections). **Deterministic, no engine serialisation** — the save is produced by `sim/save` from the same canonical state the hashes are computed on, so a save can be loaded and re-hashed to verify integrity.
- **Sections:** `world` (route state, weather, season, closed lines), `fleet` (trains owned, wear, upgrades, location, handbrake state), `player` (position, station, licences, money, reputations, flags), `quests` (arc states, NPC conditions, Chronicle log), `contracts` (active, offered, expired), `stats` (aggregates for telemetry and the Chronicle Registry), `runs` (last 20 scorecards).
- **Cadence:** autosave on every Dwell start, every settle, every route-state change, and every 5 game-minutes of running. **Ring buffer of 5 autosaves plus 3 manual slots per profile.**
- **Crash safety:** write to `save.tmp`, fsync, verify by re-hashing, then atomically rename. Never overwrite a good save with an unverified one.
- **Migration:** a `migrate(fromVersion → toVersion)` chain with unit tests for every historical version. **Saves must survive content patches.** Because RNG streams are named and additive, and because content is referenced by ID with a documented tombstone list for removed content, a v1.0 save loads in v1.4 with content added freely.
- **Mid-run saves are first-class.** The player can save at any moment, including mid-descent. This is a comfort game; do not punish real life.

### 9.6 Quest state, event director, and persistence of story

- **Quest state** is a per-arc state machine with a `steps` DAG. Step transitions are pure functions of `(worldState, playerState, eventLog)`. No step may mutate the world directly; it emits events that other systems consume. This keeps the story auditable and testable (a test can drive a whole arc through its branches).
- **The EventDirector** is a priority queue plus a set of hard pacing rules (§2.6). On every tick it re-evaluates candidate events against eligibility, weights them, and fires at most one at a time. It has a `budget` per game-hour (weighted, e.g. 6 minor units, 2 major units) so pacing can be tuned without touching event code.
- **Chronicle writer:** subscribes to the event bus and appends plain-text entries with game timestamps. This gives the player a readable history and gives *the developer* an invaluable tuning log — every balance session starts by reading a Chronicle from a 3-hour playtest.

### 9.7 Performance budgets and open-world rendering strategy

**The world is streamed, but the sky is not the problem — the ground is.** Terrain at 1:1 scale means a 12 km view distance with meaningful geometry, which must be handled entirely by LOD and impostors, never by "draw everything."

| Budget | 1080p / 60 fps target | Notes |
|---|---|---|
| Frame time | ≤ 16.6 ms; ≤ 8.0 ms GPU, ≤ 5.0 ms CPU sim+game logic | Leave 3.6 ms headroom; do not build a game that only runs on the dev machine |
| Triangle throughput | ≤ 1.8M / frame | Terrain is 60–70% of this |
| Draw calls | ≤ 1,100 / frame | Batching and instancing are mandatory, not optional |
| Active terrain chunks | 121 (11×11 at 1 km²) at full detail; 289 at impostor level | See LOD ladder below |
| Ground clutter instances | ≤ 60,000 batched (all grass/ballast/debris via `MultiMesh`) | Zero per-object nodes |
| Dynamic lights | ≤ 24 shadow-casting (sun + headlights + signals) | Emissives are unlit, unlimited |
| Particles | ≤ 3,000 sim particles/frame (weather is done with overlays and shaders, not particles) | |
| Memory | ≤ 3.2 GB resident | |
| Streaming hitch | ≤ 1 ms stall, never a visible stutter | Hard requirement; background-load and budget the upload across frames |

**Terrain LOD ladder (the single most important rendering decision):**

| Distance | Representation | Triangles per chunk | Detail |
|---|---|---|---|
| 0–250 m | Full mesh, 4 m grid | 12,000–45,000 | Full colour, cliff geometry, rail ballast, all props |
| 250–800 m | Half mesh, 8 m grid | 4,000–12,000 | Simplified, small props culled, trees as billboards |
| 800–2,500 m | Coarse mesh, 16 m grid | 900–3,000 | Colour fields, no props |
| 2,500–8,000 m | Coarse mesh, 32 m grid + terrain "shape" overlay | 200–800 | Silhouette only, vertex-colour grading |
| 8,000–12,000 m | Skybox-integrated impostor ring, baked per-region | 40–300 per sector | A single ring mesh at the horizon; this is what makes the horizon look infinite for almost no cost |
| > 12,000 m | Not drawn | 0 | Distance fog reaches full opacity at 9–11 km and hides the boundary entirely |

**Additional techniques, all mandatory:**
- **Temporal LOD hysteresis** — chunks must promote one level *before* they are needed and demote only after 4 seconds of stability, to eliminate LOD popping (popping in low-poly is the most conspicuous possible failure).
- **Occlusion culling for tunnels and cuttings** — a train game is full of long enclosed spaces; per-region precomputed portal rooms for the 14 tunnels and 22 deep cuttings.
- **Instanced everything.** Platform lamps, fences, telegraph poles, signal posts, bushes, rails (rail is one instanced mesh per chunk, not 400 objects), sleepers (`MultiMesh` at 0.65 m spacing), ballast as a shader on the terrain rather than geometry.
- **Texture-free materials everywhere.** Vertex colours + a single 2048×2048 palette/decal atlas means essentially zero texture memory and zero texture bandwidth. This is why the style is low-poly: **it is also the cheapest possible rendering strategy at this scale**, and it will hold 60 fps on a mid-range 2019 laptop.
- **Fixed sun-transform and shadow cascade update only when the sun moves** (every ~4 game-minutes), not per frame.
- **Physics is 1-D.** The train solver does not use the engine's 3D physics at all — it is arc-length math. This is a colossal performance and stability win and must not be abandoned for convenience. 3D physics is used only for debris, the walkable spaces, and incident-site props.

---

# 10. Phased Development Roadmap

**28 phases.** Each is designed as 1–3 development sessions of work, ends in a runnable build, and has an explicit Definition of Done (DoD). Complexity: **S** ≈ 1 session, **M** ≈ 2–3 sessions, **L** ≈ 4–6 sessions. **Do not skip the DoD.** Phases may be re-ordered only if a phase's inputs already exist; they may never be merged to "save time."

| # | Phase | Goal | Definition of Done | Effort |
|---|---|---|---|---|
| **1** | **Project skeleton & determinism harness** | Repo, packages, CI, content validator, fixed-step loop, state hashing. | `npm run test` passes. A 10,000-tick null simulation produces an identical hash on two runs and two threads. `docs/PROGRESS.md` exists with a template. Content validator rejects a deliberately malformed test file. | S |
| **2** | **Rail primitives: ribbon, graph, and a train on rails** | A single 2 km straight edge, one vehicle, throttle and brake, arc-length motion. | A capsule-shaped "train" accelerates, coasts and stops on a straight track with correct longitudinal physics. Braking distance matches the analytic reference within 2%. Rendered in *any* simple renderer (even 2-D top-down) with an interpolated camera. | S |
| **3** | **Curves, gradients, and the ribbon interpolation contract** | Multi-segment track: straights, arcs, gradients, and the `Ribbon` API finalised. | A 6 km test track with a 1:60 climb, a 300 m radius curve, and a summit. Ribbon returns analytically correct position/tangent/curvature/grade at 2 m resolution. Tests assert closed-form values on known geometry. Camera follows smooth through curves. | M |
| **4** | **Consist, couplers, and multi-vehicle dynamics** | A locomotive plus wagons as a coupled chain. | Slack action is visible and measurable: a hard brake on a 10-wagon consist produces run-in, and an acceleration from rest produces run-out, with coupler force graphs dumped to a debug file. Shear force is enforced. Wheelslip fires above the adhesion limit on a 1:45 wet grade. | M |
| **5** | **Brake systems in full** | Automatic brake with pipe propagation and charge; direct and dynamic brakes; emergency. | The four reference braking distances (600 t @ 80 km/h in Clear/Snow/Blizzard; Class P @ 140 km/h dry) match the §3.6/§4 tables within 3%. A deep application measurably removes braking authority for 40–90 s. | M |
| **6** | **Signals, blocks, switches, and the track graph** | A working signalled railway: blocks, aspects, junctions, route setting. | A 12 km test line with 4 stations, 6 signals, 2 junctions, and 2 switches. AI trains and the player share blocks correctly; a following train is held at red; junction routes can be set; SPAD is detected and fires an event. | L |
| **7** | **HUD v0 and the cab reading surface** | Speed, notch, brake pipe, gradient strip, next-signal box, objective line. | Every value on the HUD is read live from `sim` (no duplicated state). Gradient strip correctly previews the next 2 km. Debug overlay toggles with one key and shows block occupancy. | M |
| **8** | **The first station: Kestrel Cross** | One fully playable Hub: platform, station building, yard, job board, Dwell. | Player departs, runs 8 km, arrives, stops, dwells, and departs again. Platform stop precision is scored. Dwell timers work. The job board lists and accepts one hard-coded contract. | M |
| **9** | **The delivery loop v1** | Contract lifecycle end to end: offered → accepted → loaded → run → arrived → settled. Scorecard v1. | A full contract can be completed and settled with a real payout computed by the §6.2 formula. Abandon, fail, and late paths all produce distinct, correct settlements. Persistence of the contract through the whole loop is testable. | L |
| **10** | **Loading, mass, and condition models** | Cargo loading interface, mass distribution, fragility, perishable, and condition accumulation. | Placing cargo incorrectly measurably raises derailment risk in curves. A fragile cargo's condition bar responds to jerk integrals. A perishable cargo's freshness clock runs on the game clock and modulates payout. | M |
| **11** | **Class S "Sable" complete** | The starter train as a real machine: fuel, wear, service, module swap. | Fuel consumption matches a reference curve within 5%. Wear accrues per km and per event. The Versatility Module hot-swap works at a station. Refuelling and light repairs work. | M |
| **12** | **Class P "Peregrine" + Comfort Index** | The first "real" class, with passengers. | Every term in the comfort formula (§4.2) is implemented and independently testable. Passenger boarding/alighting works, standees penalise comfort, per-passenger traits alter reviews, and reviews print at settlement. | L |
| **13** | **Chronicle, save/load, and profile** | The game becomes resumable. | Save, quit, relaunch, and reload mid-run with the exact sim state (verified by hash). Chronicle shows threads, ledger, registry, world state, and log. Saves migrate across a deliberately-added content patch without loss. | M |
| **14** | **The open map: world streaming and the first region** | 32×32 km of Verdant Basin streaming with real terrain, LOD ladder, and the Ring Line. | Driving 30 km at 100 km/h never drops below 55 fps and never stalls during streaming. The LOD ladder produces no visible popping during a specified test route. 6 stations exist with correct geometry and data. | L |
| **15** | **Full track graph: all four regions, 48 stations** | The complete continent, with the graph validation tool passing. | `tools/validate-graph` passes every rule in §3.4: route redundancy, traction completeness, no invisible bounds, dead-end accounting. Track is drivable end to end in both directions. A 4-hour continent crossing is completable without a game-breaking issue. | L |
| **16** | **Weather, seasons, day/night, and adhesion coupling** | The environmental simulation, fully wired into physics and gameplay. | Adhesion from weather measurably alters braking distance (test asserts the reference table). Seasons rotate and alter contract supply. The lighting keyframe system drives all four regions through all four seasons without an art pass. Time scale options work with the correct gating rules. | L |
| **17** | **Contract generator v1 and the economy loop** | Infinite contracts, the full cost model, licences, and the first two reputation tracks. | 30 contracts generate on a Hub board in <20 ms. Anti-degeneracy rules from §5.6 hold under a 500-contract stress test. Payout formulas verified against hand-computed cases. Licence tests for Passenger and Freight are playable and gated correctly. | L |
| **18** | **Class F "Foundry" and the mass game** | Heavy freight, distributed power, the full traction model. | A 600 t consist completes the Ashcombe Spiral with a correctly managed ascent and a dynamic-brake descent. Runaway is possible and recoverable. Catch-point sidings work. Altitude derate applies in Ironreach. | L |
| **19** | **Hazards v1: the minor and major event sets** | Weather hazards, animal strikes, TSRs, mechanical failures, SPADs. | All 8 minor and all 12 major events are implemented, each with its telegraph firing 5–60 s ahead, at least two responses, and a measurable outcome. A telemetry pass confirms no player-fatal dead ends. | L |
| **20** | **Quest framework + the first two arcs** | The authored layer: arc state machines, dialogue, radio delivery, story contracts. | Mira's and Kestrel's full arcs are playable end to end with all branches reachable in tests. Dialogue fires only under the pacing rules of §2.6. Irreversible choices get the tone shift. Chronicle threads display correctly. | L |
| **21** | **NPC recurrence & the Chronicle of people** | Scheduled NPC appearance, individual review writing, passenger memory. | All 8 NPCs spawn on schedule with the 12-game-day fallback. NPCs remember the player across sessions. Passenger reviews reference real events from the run. | M |
| **22** | **Classes L, M, H, E** | The remaining four classes, each with its signature system fully realised. | Livestock welfare, chain-of-custody, regulated compliance, and regenerative power each implemented, tested, and individually balanced. All four licence tests playable. Regenerative credits verified against an energy-conservation test. | L |
| **23** | **Derailments, incidents, and recovery** | The consequence layer, including insurance, inquiry, and route closure. | A full derailment sequence is completable: inspect on foot, tag damage, request relief, receive a bill, file a report. A hazmat incident closes a route for a real duration and reshapes the contract pool. Recovery always leaves the player with a path forward. | L |
| **24** | **The world economy: rivals, world state, and route change** | AI rail services, rival hauler behaviour, station reopening, route repairs from story. | Rival trains run real schedules sharing blocks with the player. Rival contracts appear as Dovren's arc advances. A route closure and a route reopening are both visible in the world and reflected in contract supply. | M |
| **25** | **Art pass 1: the low-poly identity** | Final materials, palette system, livery system, prop kits, station kits, vegetation, and the asset-lint tool. | Every asset passes the Silhouette, Flat, and Noise tests. Triangle budgets enforced in CI. All four regions are visually distinct at a glance. A screenshot from each region at dawn/day/dusk/night goes into `docs/ART_BIBLE.md` as the reference. | L |
| **26** | **Art pass 2: light, weather, and mood** | The full lighting keyframe set, weather visuals, VFX, headlight cones, emissive night pass, post-processing stack. | The four lighting keys per season are authored. Rain, snow, fog, blizzard, and storm each have a distinct and readable look. Night is expensive-looking and navigable. All within the frame budget. | L |
| **27** | **Audio: engine beds, adaptive score, and ambience** | The full audio system with the region palettes and the intensity model. | All seven classes have distinct engine beds that change with speed and load. The intensity model drives stems smoothly with the specified overrides. Five reverb zones work. Music never starts in the first 90 s of a haul. | L |
| **28** | **Polish, balance, accessibility, and ship** | Tutorial flow, difficulty (Assisted/Engineer) tuning, key rebinding, colourblind modes, subtitles, performance validation on minimum spec, and the final content pass. | A new player completes a first contract within 6 minutes of launching with no external help. 60 fps at 1080p on the minimum spec. Zero crashes in a 4-hour automated soak. All §11 questions resolved. Release build exported for Windows, Linux, macOS. | L |

**Post-launch candidate phases (do not begin before shipping):** multiplayer co-op free run (the deterministic sim makes lockstep co-op plausible but it is a separate 6-phase project), a fifth region, steam traction as an eighth class, mod tools (already 80% built by the data-driven architecture), photo mode expansions, and a Scenario Editor that exports to contract JSON.

### 10.1 Cross-phase standing requirements

These apply to **every** phase from the date they are introduced, and a phase is not Done if it regresses them:

1. **No regression in the determinism test suite.** Every phase re-runs all golden runs.
2. **No regression in the frame budget.** Every phase re-runs the capture route benchmark.
3. **Every new system has a debug visualisation** toggled by a single key and documented in `docs/DECISIONS.md`.
4. **Every new tunable value is in `content/`,** never hard-coded.
5. **`docs/PROGRESS.md` is updated with: what was built, how it was verified, what was cut, what is next.**

---

# 11. Open Design Questions (decisions the designer — you — should make before Phase 1)

Each question has a fixed ID (`Q1`–`Q10`), a recommended default, and the phase by which the answer is needed. If an answer never arrives, **implement the recommended default** and log it as a decision, not a blocker.

| ID | Question | Why it matters | Recommendation / default | Needed by |
|---|---|---|---|---|
| **Q1** | **Fictional world (Aurelia) or real-world-inspired setting?** A real-country setting (e.g., a fictionalised India, Norway, or Switzerland) carries instant atmosphere, existing cultural texture, and audience familiarity — but also research cost, sensitivity risk, and expectations of accuracy you must then meet. | Affects every station name, every crop, every NPC, every palette, and how much research is needed before content authoring can even start. | **Fictional Aurelia.** It gives total freedom to design the track graph for *gameplay* (redundant routes, dramatic grades) rather than for geographic plausibility, and it avoids accuracy obligations. Regional flavours can still be drawn from real railways (Nordic, Alpine, Indian hill lines, Cornish branch lines) as *inspiration*. | Before Phase 14 |
| **Q2** | **Target session length and default game-clock scale?** | Determines contract deadline windows, itinerary lengths, and whether the game is a 20-minute-burst game or a 3-hour-sit-down game. | **Default 20–45 minute sessions, game clock at 1× = 4× real time.** Offer a "Commuter" preset (2× everything, shorter contracts) and a "Long Haul" preset (heavier contracts, 6-hour itineraries for streamers). | Before Phase 9 |
| **Q3** | **Single-player only, or ambitions toward multiplayer co-op?** | Co-op (two drivers sharing a long consist, or a dispatcher/driver pair) is genuinely attractive here and *is* enabled by the deterministic sim — but it roughly doubles the networking scope and constrains pause, save, and time-scale design. | **Single-player at launch, determinism preserved as a co-op option.** Do not implement networking, but never break determinism. Revisit only after shipping. | Before Phase 1 (as a constraint), before Phase 13 (save design) |
| **Q4** | **Monetisation: premium only, or premium plus cosmetic DLC / content packs?** | Affects whether cosmetic content has to be built to a schedule and whether the Chronicle collection is designed as a storefront. | **Premium, single payment, no microtransactions, no paid currency.** Post-launch DLC = one new region, one new class, and one scenario/arc pack, each sold as a content pack with a stated price. Cosmetics stay earnable in-game. | Before Phase 26 |
| **Q5** | **How hard is the simulation allowed to be? Should Engineer mode be the *default*, or Assisted?** | This is the single largest determinant of audience size. A hard default will lose the comfort audience; a soft default will lose the sim audience. | **Assisted is the default for the first three contracts only, then the game asks once and remembers.** Frame it as a professional standard, not a difficulty slider. Never shame either choice. | Before Phase 7 |
| **Q6** | **Can the player ever truly lose? (bankruptcy, licence revocation, permanent train loss)** | Full failure states are dramatically powerful but contradict the "comfort" and "no dead ends" pillars. A soft failure loop is safer but can feel consequence-free. | **No permanent loss. No bankruptcy.** Failure = debt (a repayment plan with the Industrial Guild), a suspended licence (recoverable in 10 game-days, during which you can still drive the Sable), and a permanently damaged reputation. Trains can be repo'd if debt goes unpaid for 20 game-days — and re-earned. | Before Phase 17 |
| **Q7** | **Steam traction: an eighth class, launch content, or post-launch?** | Steam is the most requested feature in this genre and the most expensive to simulate (boiler, water, firing, injectors, coal). It is a huge amount of fun and a huge amount of work. | **Post-launch as a content pack.** Model it properly or not at all. Reserve the simulation hooks now (a `power_source` enum that already supports it, and boiler/water fields in the train schema). | Before Phase 18 |
| **Q8** | **Language and localisation strategy from day one?** | Retrofitting localisation is far more expensive than building for it; but authoring 48,000 words × 6 languages is a real budget line. | **Build localisation-ready (ICU strings, no concatenation, no hard-coded text), ship in English only, sell localisation as later content.** Text-to-speech radio is acceptable as a fallback for accessibility, never as the primary voice. | Before Phase 20 |
| **Q9** | **Voiced dialogue or text?** | Voiced recurring NPCs dramatically raise the emotional ceiling (this is the game's whole story strategy) and dramatically raise cost and localisation complexity. | **Hybrid: fully voiced radio calls and station announcements (short, high-frequency, low word count); text-plus-vocalisation-stub for in-cab conversations.** Write all dialogue assuming it will be voiced later. | Before Phase 20 |
| **Q10** | **Which is the primary platform and minimum spec?** | Determines the LOD ladder aggressiveness, whether the 12 km view distance survives, and whether console ports are viable at all. | **PC first (Windows/Linux/macOS), min spec = mid-range 2019 laptop with integrated graphics at 1080p/30 fps, target 60 fps on a GTX 1060-class GPU.** Console ports only if the streaming budget holds comfortably. | Before Phase 14 |

**Additional smaller questions worth answering early** (each cheap to decide, expensive to change):

- **Q11** — Should the player character be customisable (appearance, name, voice) or a fixed named protagonist? *(Default: fixed named protagonist with configurable pronouns. Named characters write better stories.)*
- **Q12** — Is there a fatigue/sleep system, and does it punish or merely nag? *(Default: exists, diegetic, never fatal — drowsiness raises reaction delay on audible cues and lowers scorecard, nothing more.)*
- **Q13** — Should AI trains be followable pace-setters (overtaking gameplay) or purely obstructions? *(Default: both — 30% of AI services are slow enough to overtake at loops, 70% run faster than the player's class and must be yielded to.)*
- **Q14** — Do we ship a Scenario Editor at launch? *(Default: no, but export contracts as JSON from the start so the community tools arrive naturally.)*
- **Q15** — Should there be a "safe mode" with no derailments whatsoever for young players? *(Default: yes — a family preset that caps incidents at "delay and expense" and disables hazard severity. Costs almost nothing; broadens the audience significantly.)*

---

# Appendix A — Launch Content Manifest (the checklist that defines "content complete")

Nothing here is optional for v1.0. Phase 27 cannot be marked Done until every line is ticked.

| Content type | Target count | Notes |
|---|---|---|
| Regions | 4 | Fully authored palettes and lighting keys |
| Stations | 48 | 4 Hub / 12 Depot / 32 Halt, each with unique platform art and at least 3 distinguishing props |
| Track edges | ~420 | ~1,250 km total, all validated by `validate-graph` |
| Junctions / switches | ~180 | At least 12 set-piece junctions (spirals, flyovers, yard throats) |
| Tunnels | 14 | With portal geometry and reverb zones |
| Major bridges / viaducts | 22 | Including the Greybridge curved viaduct and the Saltmarsh Sea Wall |
| Train classes | 7 | Fully modelled, cab interiors, all upgrades |
| Vehicles (coaches/wagons) | 34 types | Including 6 heritage vehicles for Kestrel's arc |
| Commodities | 41 | With mass ranges, damage models, seasonal windows, and hazmat flags |
| Contract templates | 24 | All generator-backed |
| Licences | 6 | All with playable practical tests |
| Reputation tracks | 6 + 4 regional | All with band effects wired |
| Authored quest arcs | 8 | 42 chapters, ~120 steps, ~48,000 words |
| Recurring NPCs | 8 | Fully modelled at ~2,000 tris each, with 3–5 dialogue states per arc node |
| Minor NPC types | 12 | Commuters, yard workers, a station cat |
| Hazards | 8 minor + 12 major + 7 opportunity | All telegraphed and dual-response |
| Weather states | 7 | Fully coupled to physics |
| Viewpoints / postcards | 24 | With album UI |
| Hidden sidings | 9 | With a story vignette each |
| Livestries | 7 base + 18 unlockable | Free after acquisition |
| Music stems | ~38 min | 5 regional palettes |
| Engine sound beds | 7 classes × 4 load bands | Plus cab ambience per class |
| Radio stations | 6 | Dispatcher, 4 regional, one music/weather channel |
| Debug tools | 12 | Map inspector, consist force graph, event director log, balance sim, golden-run recorder, teleport-to-node, block occupancy view, affinity inspector, payout calculator, spawn inspector, weather forcetool, time scrubber |
| Tutorialisation | 1 flow | First contract in under 6 minutes without external help |

---

# Appendix B — Anti-patterns (things this project must not become)

Explicitly forbidden. If a phase appears to require any of these, the phase is wrong, not the rule.

1. **No invisible walls, ever.** Physical bounds must be diegetic: coastline, cliff, buffer stop, closed gate.
2. **No teleporting loaded trains.** Fast travel moves the *person*, never the cargo.
3. **No rubber-band difficulty.** Do not secretly weaken hazards when the player is struggling. Instead, offer a lower-risk contract or a different route.
4. **No RNG-only outcomes.** Every random element produces a *situation* the player resolves; never a coin-flip loss.
5. **No cutscene that takes the controls.** Story plays over the driver, not instead of them. The camera may move; the throttle stays with the player.
6. **No XP bar.** Progression is expressed as licences, reputation bands, owned machines, and world changes — not a number on a bar.
7. **No quest markers floating in the world.** Findable by context, indicated diegetically.
8. **No grinding.** No behaviour should be worth repeating 20 times for money; the generator and the payout curve actively prevent it.
9. **No texture-mapped detail that betrays the style.** If it needs a texture, it needs a shape.
10. **No motion blur, ever.**
11. **No "press X to skip the journey."** The journey is the product.
12. **No paid currency, no loot boxes, no battle pass, no limited-time-FOMO events.** The game is a place, not a treadmill.
13. **No engine-physics for the train.** Asserted in §9.3 and repeated here because it is the most tempting mistake to make in Phase 4.
14. **No silent state.** Anything simulated must be inspectable in-game or in a debug view.
15. **No content gating of geography.** Ever. At any stage of development.

---

# Appendix C — "Is this a good haul?" evaluation rubric

Use this to review any finished run, and to evaluate builds during playtesting. Score 1–5 per line. **A haul scoring below 18/30 is a design bug, not a player failure.**

| # | Criterion | 5 (excellent) | 1 (broken) |
|---|---|---|---|
| 1 | **Anticipation** | The player made at least one decision ≥30 s before its consequence landed (gradient, brake point, route, load) | Everything was reactive |
| 2 | **Legibility** | Every hazard had a readable warning the player could have acted on | Something happened for no visible reason |
| 3 | **Machine specificity** | The player's train's signature system mattered to the outcome | They could have been driving anything |
| 4 | **Cost of error** | A mistake cost time/money/reputation in a way that felt fair and informative | Punishment felt arbitrary, or errors were free |
| 5 | **Freedom exercised** | The player chose their route (even if it was the suggested one) with a stated reason | The route was forced by design |
| 6 | **Texture** | Something beautiful, funny, or human happened that had nothing to do with the contract | The run was pure system |

**Playtest protocol:** record every full session as a replay file (`runs/`) with the Chronicle log attached. After each phase from 9 onward, complete three runs with three different classes and score them. Any recurring 1 or 2 is the next phase's headline bug.

---

# Appendix D — Telemetry spec (local only, opt-in, no network)

Written to `user://telemetry/<profile>.jsonl`, one event per line, with the game clock and seed. Off by default; enabled by a single setting; never transmitted. Purpose: balancing and regression, and giving the player a real stat page.

Events to log: `run_start`, `run_end` (with the full scorecard), `contract_offered`, `contract_refused`, `contract_abandoned`, `hazard_telegraphed`, `hazard_resolved` (with the response chosen), `wheelslip`, `spad`, `derailment`, `cargo_damage`, `dwell_start`, `dwell_end`, `comfort_sample` (perstop), `fuel_sample` (per 5 game-min), `reputation_change`, `quest_step`, `dialogue_shown`, `dialogue_skipped`, `death_of_train_downtime`, `crash`, `route_state_change`, `unlock`, `purchase`.

**Analysis jobs to build by Phase 21 (in `tools/balance-sim`):** payout-per-hour by class and phase of progression; hazard frequency vs. game-hour; refusal rate per contract type (a high refusal rate on a type means it's badly tuned or badly explained); comfort distribution (the target median is 0.84 — if it's below 0.70, comfort is too harsh; above 0.93, too forgiving); derailment rate (target: 1 per 6–10 hours of Engineer-mode play, 1 per 25+ hours in Assisted).

---

# Appendix E — The first six minutes (onboarding script, phase-gated)

Because "a new player must complete a first contract within 6 minutes with no external help" is a DoD in Phase 28, the intended flow is specified now so systems can be designed to accommodate it:

| Time | Beat | Design intent |
|---|---|---|
| 0:00–0:25 | Black screen. Rain on a roof, a distant shunting horn. Title. The player's own name signed on a contract at the bottom of the screen. | Tone, immediately: quiet competence, not spectacle. |
| 0:25–1:10 | Player character wakes in a bunk above a depot office. Walk out to the platform (walkable space). A short, warm radio conversation establishes they're broke and licensed. | Teach the walk camera and that spaces matter. |
| 1:10–2:00 | On the platform: the Class S Sable is already coupled to one covered wagon. An NPC hands over a manifest without ceremony. | No tutorial pop-ups. No "press W to accelerate." |
| 2:00–2:30 | **Assisted mode is silently on.** A single line in the departure checklist: doors, reverser, brake, DRA. The HUD shows the gradient strip and next signal for the first time. | Teach the reading surface by making it visible, not by explaining it. |
| 2:30–4:30 | An 8 km run. One gentle curve, one level crossing (with the horn cue), one small gradient. Nothing else. A radio call tells a story about the town. | Teach the core loop with zero pressure. |
| 4:30–5:30 | Arrival, stop, dwell, unload. The station master says one memorable sentence about Kestrel Cross. | Mechanical completion plus world-building. |
| 5:30–6:00 | **The Scorecard.** First payout: ₡420. Then the job board appears with three contracts, one of which is for a person with a name. | Close the loop and immediately present the choice that defines the game. |

**Rule:** the words "tutorial," "objective," "mission," and "quest" must not appear on screen in the first six minutes.

---

# Appendix F — Environment implementation notes (avoiding the five classic failure modes)

Concrete, non-obvious traps that this specific project will otherwise hit:

1. **The god-gradient problem.** Natural terrain rarely offers a continuous 1:40 grade over 4 km. Do not rely on your heightmap generator to produce the gameplay grades you need. **Author the track's vertical profile FIRST** (as a spline through deliberate gradients and curves), then deform the terrain to meet it, then place water and biomes. The rail line must be the primary authoring object of the world, not an afterthought drawn on a heightmap.
2. **The straight-line problem.** Real railways have curves because of terrain; a procedural valley network produces long straight lines. Handle this by mandating a curve every 1,200 m of authored main line (an authored data rule, validated by the graph tool) so the player always has the tactile sense of a railway built to fit land.
3. **The invisible-block problem.** Block signalling between the player and AI trains is the source of most "why am I stopped" frustration. Mitigate by (a) drawing block occupancy on the map (always, not as a debug option), (b) a radio explanation whenever the player is held, and (c) a maximum hold time — after 3 game-minutes, the AI traffic yields and the player is given the road, with an apologetic radio call. Never let a player sit at a red without knowing exactly why.
4. **The empty-station problem.** A station with no people reads as a bug. Every station must have: 1–4 ambient NPCs, a lamp that reacts to the train, a sound bed, and one small unique detail (a bicycle, a timetable board with real data drawn from the schedule, a cat). Cost: ~200 tris each. Impact: enormous.
5. **The far-horizon problem.** At 12 km, fog hides everything and the world feels small and boxed. Fix it structurally: **every region must have one visible landmark over 40 km away** — a mountain, a gas flare, a radio mast, a city glow — rendered as a skybox-integrated impostor at effectively zero cost. Depth cues at distance are what make an open world feel open, and low-poly makes them cheap.

---

# Appendix G — Exact kickoff instruction

When you (the coding AI) begin work, your first message back should contain: (1) the repo skeleton from §9.2 created on disk, (2) a `docs/PROGRESS.md` with a filled Phase 1 entry, (3) the determinism test passing, and (4) any `DECISION-NEEDED` items found, listed by their question ID.

Then proceed phase by phase, in order, without asking for permission between phases unless a DoD cannot be met or a Q-question is blocking. Report at the end of each phase in this exact format:

```
PHASE <n> — <name>                          [STATUS: DONE | BLOCKED | PARTIAL]
Built:      <what now exists that did not before>
Verified:   <the specific test/demo that proves the DoD is met, with numbers>
Cut:        <anything dropped, and why>
Decisions:  <DECISIONS.md entries added>
Questions:  <OPEN_QUESTIONS.md entries, with IDs>
Next:       <phase n+1 goal>
Budget:     <sim ms | gpu ms | draw calls | tris>
```

**Start at Phase 1. Build the deterministic core first. Everything else is decoration on top of it.**
