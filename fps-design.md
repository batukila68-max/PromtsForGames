# PROJECT SANDSTORM — Technical Design Document
## A CS2-Inspired Tactical First-Person Shooter with Fly-Brain Bot AI

**Document version:** 1.0 · **Status:** Concept / pre-production · **Scope:** Playable 5v5
tactical FPS slice (one map, one mode, bots + human players)

> Relation to this repository: the AI chapter (§4) is a direct scale-up of the
> systems already proven in `src/`: a shared FlyWire-derived `GraphReservoir`
> (`controller.ts`), one independent `FlyPilot` state per agent (`game-brain.ts`),
> navigation over a room/corridor graph (`game-map.ts`), and deterministic match
> rules (`game.ts`). Nothing in this document claims full emulation of a living
> brain — the connectome supplies controller topology; match rules and decisions
> stay ordinary, auditable code.

---

## Table of contents

1. [High concept & pillars](#1-high-concept--pillars)
2. [Character design — terrorist faction](#2-character-design--terrorist-faction)
3. [Gameplay mechanics](#3-gameplay-mechanics)
4. [Advanced AI system — fly-brain bots](#4-advanced-ai-system--fly-brain-bots)
5. [Maps, modes & content plan](#5-maps-modes--content-plan)
6. [Technical architecture](#6-technical-architecture)
7. [UX, onboarding & accessibility](#7-ux-onboarding--accessibility)
8. [Balancing, telemetry & live ops](#8-balancing-telemetry--live-ops)
9. [Milestones & risks](#9-milestones--risks)
10. [Appendix A — data tables](#appendix-a--data-tables)
11. [Appendix B — glossary](#appendix-b--glossary)

---

## 1. High concept & pillars

**One-line pitch.** A round-based 5v5 tactical shooter in the CS2 lineage — tight
gunplay, an economy meta-game, and site-execution strategy — where the opposing
force is driven by a biologically inspired neurocontroller: bots that react fast,
move organically, and stay readable.

**Design pillars.**

| # | Pillar | What it means in practice |
|---|--------|---------------------------|
| P1 | **Readable lethality** | Every death is explainable: tracer, sound, and kill-feed tell the story. No hidden damage. |
| P2 | **Economy = strategy** | Money converts round outcomes into future options (force-buy, save, full-buy). |
| P3 | **Movement is skill** | Counter-strafing, jiggle-peeking, and silent walking separate ranks. |
| P4 | **Bots with bodies** | AI obeys the same physics, vision, hearing, and reaction budgets as humans. |
| P5 | **Deterministic core** | Simulation is seeded and replayable; randomness is explicit, logged, and tunable. |

**Non-goals for the slice:** battle royale, hero abilities, wall-running, or
pay-to-win monetization. Cosmetics only, zero gameplay effect.

---

## 2. Character design — terrorist faction

### 2.1 Faction identity: the "Dust Vipers"

A fictional, non-state paramilitary cell built for a desert-industrial theater.
The design brief demands three readings at a glance: **silhouette** (role),
**palette** (team), and **gear** (threat level). Terrorists read warm/dusty
(sand, rust, olive, worn black) against the cool blues/grays of the
counter-terrorists.

**Design rules (all T models).**

- **Silhouette first:** head > torso bulk > leg stance must be distinguishable
  at 30+ m against skybox and wall textures. No capes, no spikes, nothing that
  breaks the hitbox read.
- **Head hitbox honesty:** helmets, hoods, and goggles never extend beyond the
  head capsule by more than 3 cm.
- **Palette discipline:** ≥60% dusty warm tones, ≤15% saturated accent (scarf,
  patch, glove trim) so the accent marks identity without camouflage abuse.
- **Gear tells the kit:** backpack/pouch bulk scales with the player's primary
  (rifle bulk vs. light SMG loadout); the bomb carrier gets a distinct satchel.
- **First-person consistency:** sleeves/gloves match the third-person model so
  viewmodels never contradict the world model.

### 2.2 The five T operatives (playable skins, one shared rig)

All five share one humanoid rig (58 bones), one animation set, and one hitbox
layout; they differ in headgear, torso gear, palette, and voice barks.

#### T-1 «JACKAL» — cell leader / rifleman (default T)

- **Look:** weathered sand shemagh over a black balaclava; matte olive plate
  carrier with three STANAG pouches; rust-red shoulder patch (Viper sigil);
  fingerless gloves, dusty tan fatigues, scuffed boots.
- **Palette:** sand `#C2A878` / olive `#5A5B3A` / worn black `#23211E`, accent
  rust `#A33F27`.
- **Gear read:** medium bulk, rifle-length mag pouches — "standard rifle threat".
- **Animation flavor:** confident, upright idle; hand-signal gesture on round start.
- **Voice:** terse, calm callouts ("Site B. Go, go.").

#### T-2 «SCARAB» — breacher / entry fragger

- **Look:** scarred half-mask + tinted goggles pushed up; heavy front plate with
  bolt-cutter scars; shotgun-shell bandolier across the chest; knee pads,
  reinforced boots.
- **Palette:** charcoal `#2E2C29` / dust `#9A8A6E`, accent amber `#D08A2D`
  (shell brass + visor tint).
- **Gear read:** widest torso, chest bandolier — "close-range threat, watch corners".
- **Animation flavor:** forward-leaning sprint, shoulder-check door breach.
- **Voice:** loud, aggressive ("Breaching! Move!").

#### T-3 «MIRAGE» — scout / lurker

- **Look:** light hood + wrap sunglasses; minimal chest rig (two pouches), slim
  backpack; silent-sole boots; coiled tuft of antenna wire on the pack
  (non-gameplay cosmetic).
- **Palette:** pale dust `#D3C49C` / gray-olive `#6E6E58`, accent teal `#3E8E8A`.
- **Gear read:** slimmest profile, quiet footstep mix — "could be anywhere".
- **Animation flavor:** crouch-friendly transitions, slow deliberate plant/defuse… 
  (defuse is CT-side; Mirage gets the fastest ladder/climb set instead).
- **Voice:** whispers, late-round info ("One mid. Holding.").

#### T-4 «ANVIL» — support / AWPer

- **Look:** long canvas duster over armor; ghillie-trimmed hood (shoulder only,
  hitbox-clean); tall backpack with antenna mast; magnifier optic on the rifle.
- **Palette:** brown `#6B4E35` / canvas `#A68D5F`, accent bone `#E4DCC8`.
- **Gear read:** tallest pack silhouette — "long-angle threat".
- **Animation flavor:** slow scope-in, deliberate bolt/scope cycle.
- **Voice:** patient, positional ("Holding long. Don't peek me.").

#### T-5 «COURIER» — bomb carrier specialist

- **Look:** compact civilian-jacket shell over a light vest; the **bomb satchel**
  (olive drab, red LED blink, coiled wire) worn cross-body — instantly readable;
  sneakers over Moreau-style wraps, light gloves.
- **Palette:** faded denim `#4A5A6E` / sand `#B9A071`, accent hazard-yellow
  `#D9B13B` on the satchel strap.
- **Gear read:** the ONLY satchel in the game — "kill priority when planted/exposed".
- **Animation flavor:** satchel-clutching sprint, kneeling plant animation (3.2 s).
- **Voice:** nervous-efficient ("Planting. Cover me.").

### 2.3 Visual-concept notes (for the art pass)

- **Turnaround spec:** front/side/back orthographic sheets + 3/4 action pose +
  first-person sleeve study per operative; palette swatches with hex values.
- **Material language:** cloth (high roughness, frayed edges), metal (mid
  metallic, edge wear), polymer (satin), glass (visor, low opacity).
- **Damage states (cosmetic):** three dirt/blood overlays driven by remaining HP
  thresholds (100/60/30) — readability aid, never hitbox-changing.
- **Performance budget:** T model ≤ 14k tris third-person, ≤ 8k first-person
  arms; single 2k albedo/roughness/normal set per operative.

### 2.4 Counter-terrorist foils (summary)

Five CT counterparts ("Sentinels") in cool tones (navy, slate, ice-gray) with
helmets/visors and a defuse-kit pouch. Same rig and hitboxes as the T side —
symmetry is a competitive requirement, identity is carried by art, barks, and
the CT defuse-kit read.

---

## 3. Gameplay mechanics

### 3.1 Core loop (round level)

```text
BUY (15–20 s) → EXECUTION / DEFAULT (plant-or-pick) → SITE TAKE or PICK →
PLANT (3.2 s channel) → POST-PLANT (defense of ticking bomb, 40 s) or
RET AKE/DEFUSE (CT 5 s, 10 s without kit) → ROUND END → ECONOMY UPDATE → repeat
```

- **Win condition:** first to 13 round wins (MR12), 6-round halves, side swap;
  12–12 triggers overtime (MR3, $10k start, repeat until decided).
- **Round timer:** 1 min 55 s; bomb timer 40 s; defuse 10 s (5 s with kit).
- **Bomb (C4):** drops on carrier death, pickup 1 s channel, plant restricted to
  site zones (3.2 s), beeps accelerate as detonation nears (audio telegraph).

### 3.2 Moment-to-moment loop (duel level)

```text
INFO (sound/utility/vision) → POSITIONING (angle, cover, crosshair) →
DUEL (peek, spray/burst, strafe) → TRADE or RESET → ECONOMY CONSEQUENCE
```

Every mechanic below is tuned to keep this loop under ~8 seconds per duel and
fully scrutable after death (killcam + damage report).

### 3.3 Movement

| Mechanic | Spec | Skill expression |
|----------|------|------------------|
| Walk / run | 250 u/s run, 130 u/s walk (silent), 175 u/s crouch-walk | Shift-walk to deny audio |
| Counter-strafe | accuracy recovers ≤120 ms after full stop | tap-`A`/`D` duels |
| Jiggle / shoulder-peek | no accuracy penalty for ≤150 ms exposure | info without commitment |
| Jump | single jump, 1.05× height crouch-jump assist; stamina-free but inaccurate airborne | no bunny-hop chaining (hard cap) |
| Ladders / vaults | fixed ladder speed, 0.9 s vault over ≤1.1 m rails | timing windows |
| Noise model | footsteps audible ~18 m (run), ~6 m (walk); landing thump ~22 m; scope zoom audible ~4 m | audio = second sight |

Movement is **server-authoritative** with client prediction; speed and spread
are functions of the same velocity vector, so "running accuracy" exploits are
structurally impossible (§6.3).

### 3.4 Combat

- **Hitscan rifles, projectile utility.** Headshot multipliers ×4 (rifles),
  ×2 (SMGs/pistols); helmets halve headshot damage through the first bullet.
- **Recoil & spray:** deterministic base pattern per weapon + small random
  perturbation; pattern resets after 0.45 s pause. Spray transfer between
  targets is the premier skill; patterns ship in-client (practice range shows them).
- **Damage falloff:** linear from 100% (0–15 m) to 70% (30 m+) for rifles;
  damage report shows per-hit numbers post-mortem.
- **Armor & helmets:** $650 armor (50% body bleed-through reduction), +$350
  helmet (headshot mitigation). Armor never changes TTK breakpoints silently —
  breakpoints are published in the buy menu.
- **Penetration:** wallbang multipliers per material (wood 0.7, thin metal 0.5,
  concrete 0.0); tracers + impact decals always show penetration.
- **Utility:** smoke (18 s bloom, one-way-proof volumes), flash (2.1 s max,
  LOS + angle falloff, team-flash 50%), HE (≤52 damage, distance falloff),
  Molotov/incendiary (7 s area denial, 8 dps), decoy (fake radar ping + sound).
- **Weapons (slice roster):** AK-47 / M4A4 / M4A1-S, AWP, Desert Eagle, Glock /
  USP-S, MP9 / MAC-10, Nova, FAMAS / Galil, SSG-08; prices mirror the CS2
  lineage (see Appendix A) so economy intuition transfers.

### 3.5 Economy

- Kill reward: $300 (rifle) / $600 (SMG) / $100 (AWP), plant +$300, defuse +$300.
- Round outcome: win $3,250 · loss $1,900 → +$500 per consecutive loss (cap $3,400).
- Pistol round: everyone $800; max carry $16,000; no interest, no debt.
- Design intent: force-buys are viable but punishable; full saves recur every
  ~4 rounds; the scoreboard always shows team money to legitimize "eco" calls.

### 3.6 UX & "playable" requirements

- **TTK band:** 0.25–0.7 s for rifle duels at <20 m — lethal but tradeable.
- **Netcode honesty:** 128-tick servers, lag compensation ≤200 ms, damage numbers
  optional, hit marker + sound always.
- **Anti-frustration:** spawn protection 3 s (no damage in/out), AFK auto-kick
  after 2 rounds, surrender vote at 0–8, team-damage reflection beyond 300/round.
- **Practice tooling:** offline range with bots frozen/mirroring, spray-pattern
  overlay, grenade-trajectory preview (offline only), demo/replay from tick files.

---

## 4. Advanced AI system — fly-brain bots

### 4.1 Why a fly brain (honest framing)

Real flies combine **~30 ms sensorimotor loops**, wide-field motion vision, and
compact recurrent circuits (notably the fan-shaped body, FB) that arbitrate
between competing drives (approach/avoid) with tiny neuron counts. We do not
simulate a living brain. We borrow three ideas: (1) a **fixed recurrent
topology** derived from the real FlyWire FAFB v783 FB subgraph as the
controller's wiring; (2) **fast, small state updates** (few ms, tens of floats);
(3) **trained linear readouts** over rich reservoir dynamics instead of a giant
policy network. Deterministic game rules sit on top, exactly as `game.ts` sits
atop `game-brain.ts` in this repo.

### 4.2 Architecture overview

```text
SENSORS (20-ch vector) ──► GRAPH RESERVOIR (FlyWire FB subgraph, tanh, 8 ticks)
        │                              │
        │                     per-bot recurrent STATE (Float64Array, N+1)
        │                              ▼
        │                     LINEAR READOUTS (trained offline, frozen in prod)
        │                      ├─ steer vector (x, y)      [NeuralPolicy]
        │                      ├─ caution / aggression / focus gates [FlyPilot.channels]
        │                      └─ confidence (mission-vs-threat logistic)
        ▼
SYMBOLIC LAYER (ordinary code): role FSM, utility scoring, squad comms,
  aim controller, pathfinding over the map graph, reaction scheduler
        ▼
ACTUATORS: move/strafe/crouch/walk, look (yaw/pitch), fire/burst, utility,
  plant/defuse channels, radio callouts
```

**Key numbers (carried over from `src/`).**

| Component | Parameter | Value / source |
|-----------|-----------|----------------|
| Sensor vector | dimension | **20** (`controller.ts` `inputSize`) |
| Reservoir ticks | per decision | **8** tanh relaxation steps |
| Nonlinearity | node update | `tanh(drive + Σw·state)`, ring-buffered CSR edges |
| Training samples | readout fit | 512 (policy) / 384+128 (drawing head), CG ≤96 iters |
| Recurrence | per bot | one `Float64Array(N+1)` state, bias unit at index 0 |
| Sharing | weights | **shared** reservoir + readouts; **private** state per bot (9 in `among.html`, 10 in the slice) |

### 4.3 Sensor vector (bot → brain, every think tick)

Adapted from `PilotSensors` to FPS needs; all values normalized to [−1, 1]:

| # | Channel | Meaning | Fly analogue |
|---|---------|---------|--------------|
| 0 | bias | constant 1 | — |
| 1–2 | goal dir ×1.2 | unit vector to current objective | celestial-heading input |
| 3 | goal distance | dist/12 ×0.3 | optic-flow magnitude |
| 4 | allies near | n/8 ×0.3 | social context |
| 5 | threat | visible enemies, LOS-weighted ×0.3 | looming detectors |
| 6 | role flag | attacker +0.2 / defender −0.2 | state-dependent gating |
| 7 | objective progress | plant % / site control ×0.3 | satiety-like drive |
| 8 | flash/dark | flashed −0.25 / clear +0.25 | light adaptation |
| 9–10 | time sin/cos | `sin/cos(t·0.2 + id)` | circadian/oscillatory drive |
| 11 | personality | id/8−0.5 ×0.4 (per-bot seed → playstyle) | individuality |
| 12 | alive/round | +0.2 / −0.2 | — |
| 13 | team share | round-win share ×0.25 | colony state |
| 14 | bomb nearby | +0.25 | odor-gated approach |
| 15 | utility cooldown | cd/10 ×0.25 | refractory period |
| 16–18 | caution/aggr/focus | recurrent gates ×0.25 (feedback) | neuromodulation |
| 19 | jitter | `sin(t·0.07 + id·1.9)` ×0.15 | motor noise / exploration |

**Vision model (same physics as players, §4.6):** 110° FOV cone, LOS raycast on
the map graph (`lineOfSight` equivalent), distance falloff, muzzle-flash and
footstep events injected as transient threat/bearing channels. Bots never see
through walls; "hearing" is the same noise model from §3.3.

### 4.4 Readouts and arbitration (brain → behavior)

1. **Steer head** (`NeuralPolicy.readout`): 2-D vector blended 85/15 with the
   symbolic goal direction; anti-reversal guard (falls back to goal dir if the
   blend opposes it) — ported verbatim from `FlyPilot.think`.
2. **Gate channels** (`channel(offset)`): caution, aggression, focus are tanh
   readouts of staggered reservoir slices + the logistic `mathSignal`
   (mission-vs-threat). They scale, never override: caution widens clear-angles,
   aggression shortens burst delays, focus narrows aim noise.
3. **Action selection (symbolic, utility-based):** every 150–250 ms the bot
   scores `hold / peek / rotate / execute / save / hunt` from
   `{gates, sensors, squad state, economy}`; the reservoir proposes, the FSM
   disposes. A hard rules layer vetoes illegal acts (firing during freeze time,
   planting outside zones) — determinism and anti-cheat (§6.4).
4. **Aim controller:** two-stage — saccade (ballistic yaw jump with fly-like
   30–60 ms latency + overshoot) then smooth pursuit; per-bot noise σ scaled by
   `(1 − focus)`, movement penalty, and distance. Reaction budget below.

### 4.5 Fly-like behavior palette (the fun part)

| Behavior | Biological inspiration | Game implementation |
|----------|------------------------|---------------------|
| **Saccadic peeks** | fly body saccades (~100 ms turns) | jiggle-peek with 90–140 ms exposure, overshoot corrected next tick |
| **Looming flinch** | giant-fiber escape | on sudden close threat: instant crouch-strafe away, 250 ms cooldown |
| **Optic-flow hugging** | wall-following in corridors | steering bias toward mid-corridor flow balance; reduces open-field exposure |
| **Erratic approach** | zigzag odor tracking | execute paths add bounded lateral sine (≤0.8 m) at 1–2 Hz — harder to pre-aim, still fair |
| **Freezing** | motion camouflage | lurker role holds still 1–3 s after a teammate's contact; audio-only presence |
| **Swarm spacing** | collision-avoidance | separation steering keeps 2–4 m between executing bots; no stacking in smokes |
| **State persistence** | recurrent memory | per-bot state carries round-to-round (decayed 50% at round start): bots "tilt" or "heat up" like players |

Difficulty is **not** aimbot scaling: Rookie/Regular/Elite change **reaction
budget** (420/300/210 ms), **saccade noise** (×1.5/×1.0/×0.7), **utility usage**
(0/50/100% of scripted executes), and **comms richness** — never damage, HP, or
wallhack vision.

### 4.6 Fairness constraints (bots obey P4)

- Same tick rate, same spread/recoil tables, same noise model, same economy.
- Reaction timer starts at *first perceptible frame* (LOS + FOV + unflashed);
  timers are logged per duel for tuning.
- No omniscient rotates: off-screen enemies exist only as last-known ghosts
  with timestamps; ghosts older than 5 s are deleted.
- Bot thinking costs real server ms: 10 bots × (8 reservoir ticks × N edges)
  must fit in **<1.5 ms** per think slice — trivially true for N ≤ 1024.

### 4.7 Training & calibration pipeline

1. **Topology:** export the FB subgraph (`scripts/prepare_connectome.py` →
   `public/data/connectome.json`, FlyWire FAFB v783, CC-BY-4.0; attribution in
   `public/THIRD_PARTY_LICENSES.txt`).
2. **Readout fit (offline):** 512 seeded samples mapping sensor→steer targets;
   ridge-regularized Gram + Conjugate Gradient ≤96 iterations
   (`NeuralPolicy`); report `calibrationError` (RMSE) into telemetry.
3. **Behavior cloning (optional):** record human demo ticks, fit additional
   heads (burst timing, utility choice) with the same CG solver.
4. **Bot personalities:** fixed per-id input seed (`id/8−0.5` channel) yields
   stable lurker/entry/support archetypes without separate models.
5. **Regression gate:** `npm test`-style checks — calibration RMSE below
   threshold, deterministic replays bit-identical, reaction budgets respected.

---

## 5. Maps, modes & content plan

**Launch map: "Sirocco" (de_dust lineage, original layout).**

- 3 lanes (Long / Mid / Short-to-B), 2 sites, CT spawn ↔ A/B split ~12 s rotate.
- Callout grid fixed at design time (Long Corner, X-Box, Goose, …); bots and
  barks share the same callout IDs.
- Performance: ≤120k tris visible, 2k PBR sets, baked GI + 4 dynamic lights max.
- Map graph ships as data (`rooms`/`connections` pattern): nav nodes, LOS
  edges, walkability margin, per-node tags (site, choke, boost, plant-zone).

**Modes (slice):** Competitive MR12 (flagship), Casual 10v10 (short rounds, free
armor), Deathmatch (respawn, aim training vs. Elite bots), Offline-vs-bots
(new-player funnel).

**Content cadence (post-slice):** one map/year, two operative-skin drops/season,
one weapon-finish collection/season — all cosmetic.

---

## 6. Technical architecture

### 6.1 Client / server split

```text
CLIENT (prediction + render)            SERVER (authority, 128 tick)
 Render (Three.js/WebGPU→UE5 target)    Simulation: movement, combat, utility
 Input → predicted move                 Lag compensation + anti-cheat review
 Reservoir echo for bot *ghosts*        Full GraphReservoir + 10 FlyPilot states
 Demo recorder (tick file)              Replay log + seeded RNG stream
```

Web slice can run the whole match in-browser (as `among.html` does today);
production moves simulation server-side with zero protocol change — sensors in,
actuators out.

### 6.2 Determinism & replays

- Seeded RNG (`seededRandom`, xorshift) per match; every random call draws from
  the match stream — replays are bit-identical inputs + seed.
- Fixed-point-safe math policy: `Float64Array` reservoirs, no `Math.random()`
  in simulation code (validated by lint rule).
- Tick file format: header (seed, map hash, build id) + per-tick deltas;
  <2 MB per 40-round match.

### 6.3 Anti-cheat posture

Server-authoritative movement/combat/economy; client sends intents only.
Reaction-time telemetry flags inhuman consistency; bot logic never ships
privileged information to clients (ghosts stay server-side).

### 6.4 Performance budgets

| System | Budget |
|--------|--------|
| Frame (client, 1080p) | 8 ms CPU / 8 ms GPU (120 fps target on mid hardware) |
| Server tick (128 Hz) | 6 ms total; AI ≤1.5 ms; physics ≤2 ms |
| Memory | <1.5 GB client; reservoir N≤1024 ×10 bots ≈ <5 MB |
| Network | <30 kbps/client; snapshot 20 Hz + delta ticks |

---

## 7. UX, onboarding & accessibility

- **First 15 minutes:** boot camp (move → counter-strafe → spray → plant) vs.
  Rookie bots → first Casual match with a Coach-bot squadmate calling plays.
- **HUD:** minimal (HP/armor/ammo/round-timer/minimap-callouts); damage direction,
  killfeed with weapon icons, economy strip always visible in buy time.
- **Accessibility:** color-blind palettes (3 presets), separate volume sliders
  (footsteps boosted by default), hold-to-crouch toggle, 40–110 FOV slider,
  text-to-speech callouts, reduced-flash option (no competitive disadvantage —
  flash still blinds, effect only softened visually).
- **Comms:** ping wheel (Enemy here / Need smoke / Rotating), voice with
  live transcription, mute-by-default for non-party in Casual.

---

## 8. Balancing, telemetry & live ops

- **Telemetry per duel:** reaction ms, first-bullet accuracy, trade rate,
  utility damage, economy state — segmented by rank and bot tier.
- **Balance levers (no client patch needed):** weapon price/damage tables,
  loss-bonus curve, bot reaction/noise scalars, map-graph node tags.
- **Rank system:** Elo-per-round (CT/T split) with placement (10 games) and
  visible progress; smurf detection via duel-stats fingerprinting.
- **Live-ops rules:** no mid-season weapon stat changes; bot-behavior tuning
  ships with patch notes citing telemetry deltas.

---

## 9. Milestones & risks

| Milestone | Exit criteria |
|-----------|---------------|
| M1 Graybox + human duel | 1v1 on graybox Sirocco, TTK band verified |
| M2 Bot slice | 5 Rookie bots hold/execute, reaction logs green |
| M3 Economy + full rounds | MR12 loop playable, saves/forces emerge |
| M4 Art + audio pass | T/CT reads at 30 m, footsteps mix final |
| M5 Closed beta | retention D1/D7, duel-fairness survey ≥4/5 |

**Top risks:** (1) reservoir tuning feels "drunk" not "organic" → mitigation:
symbolic guardrails + noise caps; (2) 128-tick cost → mitigation: headless
profile gate in CI; (3) scope creep on weapons → mitigation: roster freeze at M2.

---

## Appendix A — data tables

**A1. Slice weapon roster (price / damage / notes).**

| Weapon | Price | Base dmg | Notes |
|--------|-------|----------|-------|
| Glock-18 / USP-S | — (spawn) | 28 / 32 | T/CT pistols |
| Desert Eagle | $700 | 53 | headshot cannon |
| MAC-10 / MP9 | $1050 / $1250 | 29 / 26 | high kill reward |
| Nova | $1050 | 26×8 | — |
| Galil / FAMAS | $1800 / $1950 | 30 / 30 | eco rifles |
| AK-47 | $2500 | 36 | one-tap helmetless |
| M4A4 / M4A1-S | $3100 / $2900 | 33 / 33 | CT rifles |
| SSG-08 | $1700 | 88 | mobile scope |
| AWP | $4750 | 115 | 5-round mag, slow scope |

**A2. Bot tiers.**

| Tier | Reaction | Saccade noise | Utility | Comms |
|------|----------|---------------|---------|-------|
| Rookie | 420 ms | ×1.5 | none | pings only |
| Regular | 300 ms | ×1.0 | smokes + flashes | callouts |
| Elite | 210 ms | ×0.7 | full executes | strats + fakes |

**A3. Round economy.**

Win $3,250 · Loss $1,900 (+$500/consecutive loss, cap $3,400) · Plant/Defuse
+$300 · Kill $300 rifle / $600 SMG / $100 AWP · Pistol round $800 · Cap $16,000.

---

## Appendix B — glossary

- **Connectome / FlyWire FAFB v783:** real mapped fruit-fly brain wiring
  (CC-BY-4.0); we use an FB-region subgraph as fixed controller topology.
- **GraphReservoir:** recurrent network whose weights come from the connectome
  subgraph; only linear readouts are trained (`controller.ts`).
- **FlyPilot:** one bot's private recurrent state + gates + confidence over the
  shared reservoir (`game-brain.ts`).
- **Readout:** trained linear map from reservoir features to an action signal.
- **Saccade:** fast ballistic turn (fly-like), vs. smooth pursuit tracking.
- **MR12:** max 12 rounds per half; first to 13 wins.
- **Trade:** killing the enemy who just killed your teammate (core FPS concept).

---

*End of document — PROJECT SANDSTORM TDD v1.0*
