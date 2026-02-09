# Evaluation Framework for Learning-Oriented Magic: The Gathering Replays

> A unified, learning-oriented framework for representing, replaying, and evaluating games of Magic: The Gathering (MTG).

---

## Abstract

This paper proposes a unified, learning-oriented framework for representing, replaying, and evaluating games of Magic: The Gathering (MTG). We introduce a two-level notation system that separates a lossless, rule-complete event log from a derived, pedagogical representation. On top of this notation, we define a vector-based state evaluation model inspired by game AI, decision theory, and traditional MTG strategic concepts. The framework is designed to support deterministic replays, explainable evaluation of decisions, and future integration with machine learning systems.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Design Goals](#2-design-goals)
3. [Two-Level Notation System](#3-two-level-notation-system)
    - [Level 1: Full-Detail Event Log](#31-level-1-full-detail-event-log)
    - [Level 2: Learning-Oriented Units](#32-level-2-learning-oriented-units)
4. [State Representation](#4-state-representation)
5. [Evaluation Model](#5-evaluation-model)
    - [Philosophy](#51-philosophy)
    - [Evaluation Dimensions Overview](#52-evaluation-dimensions-overview)
    - [Common Helpers](#53-common-helpers)
6. [Dimension Formulas](#6-dimension-formulas)
    - [Resources](#61-resources)
    - [Board Presence](#62-board-presence)
    - [Tempo](#63-tempo)
    - [Card Advantage](#64-card-advantage)
    - [Life Pressure](#65-life-pressure)
    - [Inevitability](#66-inevitability)
    - [Flexibility](#67-flexibility)
    - [Risk / Information](#68-risk--information)
    - [Synergy / Gameplan / Reach](#69-synergy--gameplan--reach)
    - [Explosiveness](#610-explosiveness)
7. [Card Formations](#7-card-formations)
    - [Overview](#71-overview)
    - [Formation Graph Structure](#72-formation-graph-structure)
    - [Node Roles](#73-node-roles)
    - [Edge Types and Axes of Interaction](#74-edge-types-and-axes-of-interaction)
    - [Formation Output Calculation](#75-formation-output-calculation)
    - [Redundancy and Cumulative Types](#76-redundancy-and-cumulative-types)
    - [Formation Templates](#77-formation-templates)
    - [Formation Evaluation Integration](#78-formation-evaluation-integration)
    - [Example Formation Analysis](#79-example-formation-analysis)
8. [Learning Helper Statistics](#8-learning-helper-statistics)
    - [Overview](#81-overview)
    - [Land Drop Rating](#82-land-drop-rating)
    - [Available Mana](#83-available-mana)
    - [Cast Options in Hand](#84-cast-options-in-hand)
    - [Mana Color Coverage](#85-mana-color-coverage)
    - [Integration with Evaluation Framework](#86-integration-with-evaluation-framework)
9. [Evaluation Before and After Decisions](#9-evaluation-before-and-after-decisions)
10. [Applications](#10-applications)
11. [Limitations and Future Work](#11-limitations-and-future-work)
12. [Conclusion](#12-conclusion)
13. [Appendix A: Notation Syntax and Semantics](#appendix-a-notation-syntax-and-semantics)

---

## 1. Introduction

Magic: The Gathering is a strategically complex game combining hidden information, stochastic elements, and a highly expressive rules engine. Unlike games such as chess or Go, MTG lacks a compact, standardized notation suitable for replay, analysis, and learning. This work addresses that gap by proposing a formal notation and evaluation framework focused on player decision quality rather than outcomes alone.

---

## 2. Design Goals

The framework is designed around the following principles:

- **Deterministic replay** of games
- **Explicit representation** of player decisions
- **Separation** of rules execution and learning abstraction
- **Explainable, multi-dimensional** state evaluation with purpose of play optimization
- **Extensibility** to different formats and future mechanics

---

## 3. Two-Level Notation System

The proposed notation consists of two strictly separated layers:

| Level  | Purpose                 | Characteristics                                                                                                                                                                                          |
| ------ | ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **L1** | Full-Detail Event Log   | Complete, lossless event log capturing all game actions, rule resolutions, state-based actions, and randomness. Sole authoritative source for replay.                                                    |
| **L2** | Learning-Oriented Units | Derived learning view with stable state snapshots before and after significant decision windows, together with structured representation of stack, targets, and player choices. Fully derivable from L1. |

### 3.1 Level 1: Full-Detail Event Log

Each L1 event is represented as a tuple consisting of:

- **Time** - Temporal marker
- **Actor** - Player or system
- **Event Type** - Classification of action
- **Payload** - Event-specific data

Events are classified into:

| Category                   | Examples                                    | Description                       |
| -------------------------- | ------------------------------------------- | --------------------------------- |
| **Player Decision Events** | `CAST`, `ACTIVATE`, `DECLARE_ATTACKERS`     | Strategic choices made by players |
| **System Events**          | `MOVE`, `RESOLVE`, `TRIGGER`, `STATE_BASED` | Automatic rule execution          |

> **Note:** All randomness is explicitly recorded with seeds to ensure deterministic replays. This should be without hidden information.

### 3.2 Level 2: Learning-Oriented Units

L2 groups L1 events into **Units**, each representing a complete decision context. A Unit contains:

1. A **stable state snapshot** before the decision
2. All **spells, abilities, triggers, targets, and choices** placed on the stack
3. A **stable state snapshot** after full resolution

This structure aligns with how players reason about decisions and enables localized evaluation.

For evaluation purposes it additional information layer with information about thought process of decision maker if there where multiple relevant options to choose from. Hereby reasoning about decision makers analysis of play state and anticipated changes should be part evaluation in quantifiying comparism.

---

## 4. State Representation

Game states are represented as structured objects including:

### Turn Information

- Turn number
- Phase and step
- Priority information

### Player State

- Life totals
- Mana pools
- Counters

### Zone Contents

- Battlefield
- Stack
- Hand
- Graveyard
- Library
- Exile
- Command zone

### Per-Object Attributes

- Power / Toughness
- Counters
- Attachments
- Status flags (tapped, summoning sickness, etc.)

> **Visibility:** Rules allow separation between training views (full information) and spectator or shared views.

---

## 5. Evaluation Model

### 5.1 Philosophy

Instead of assigning a single scalar score to a game state, we adopt a **vector-based evaluation model**. Each dimension corresponds to a well-established MTG strategic concept.

**Benefits:**

- Nuanced assessment of decisions
- Supports explainable feedback
- Avoids conflating short-term and long-term advantages

### 5.2 Evaluation Dimensions Overview

| #   | Dimension                      | Description                                         |
| --- | ------------------------------ | --------------------------------------------------- |
| 1   | **Resources**                  | Mana production, fixing, and future action capacity |
| 2   | **Board Presence**             | Current battlefield strength and permanence         |
| 3   | **Tempo**                      | Initiative, mana efficiency, and time advantage     |
| 4   | **Card Advantage**             | Net and repeatable access to cards                  |
| 5   | **Life Pressure**              | Effective clocks toward victory or defeat           |
| 6   | **Inevitability**              | Long-term advantage if the game continues           |
| 7   | **Flexibility**                | Number and quality of available options             |
| 8   | **Risk / Information**         | Exposure to variance and information asymmetry      |
| 9   | **Synergy / Gameplan / Reach** | Cohesion of strategy and ability to close games     |
| 10  | **Explosiveness**              | Potential for sudden, dramatic state changes        |

### 5.3 Common Helpers

Let `S` be the snapshot, `you` the evaluated player, `opp` the opponent.

#### Normalization Functions

```
clamp(x, lo, hi)        → Clamps x to range [lo, hi]
norm(x, C) = clamp(x, -C, C) / C   → Output in [-1, +1]
```

> Recommended caps `C` are listed per dimension.

#### Card Metadata Tags

You'll want a lightweight card DB mapping `card_ref` / `oracle_id` to tags:

| Category | Tags                                    |
| -------- | --------------------------------------- |
| Mana     | `mana_producer`, `fixer`                |
| Engines  | `engine_draw`, `engine_tokens`          |
| Control  | `lock_piece`, `removal`, `counterspell` |
| Aggro    | `reach_burn`, `combat_trick`            |
| Combo    | `combo_piece`, `payoff`, `enabler`      |
| Utility  | `tutor`                                 |

> If you don't have this yet, start with type-based heuristics and add tags later.

---

## 6. Dimension Formulas

### 6.1 Resources

**Purpose:** Measure mana production, fixing, and future action capacity.

#### Features

- `land_count(you)`, `rock_count(you)`, `dork_count(you)`
- `fix_sources(you)` — Lands/rocks producing ≥2 colors or any-color
- `colors_accessible(you)` — WUBRG present on untapped sources
- _Optional:_ `hand_playable_next_turn(you)` — Spells you can cast next turn

#### Formula

**Mana Production Potential:**

```
MPP(you) = lands + 0.9 × rocks + 0.8 × dorks_active
MPPΔ = MPP(you) - MPP(opp)
```

**Fixing:**

```
Fix(you) = min(1, colors_accessible(you) / colors_needed_est(you))
```

> If unknown, use `colors_needed_est = 2` for 2-color, `3` for 3+ color, or infer from commander identity / deck.

**Raw Value:**

```
R_raw = 1.0 × MPPΔ + 0.8 × (Fix(you) - Fix(opp))
```

**Normalized:** `Resources = norm(R_raw, C=6)`

---

### 6.2 Board Presence

**Purpose:** Measure current battlefield strength and permanence.

#### Features

For each permanent `p` on battlefield:

| Type        | Attributes                                                                         |
| ----------- | ---------------------------------------------------------------------------------- |
| Creature    | P/T, keywords (flying, trample, first strike, deathtouch, lifelink, hexproof/ward) |
| Noncreature | Tag category (engine/lock/anthem/utility), planeswalker loyalty                    |
| Status      | Tapped, summoning sickness, counters, attachments                                  |

#### Formula

**Creature Value:**

```
kw = 0.8×flying + 0.4×trample + 0.4×first_strike + 0.6×deathtouch + 0.4×lifelink + 0.8×hexproof_or_ward

stat = P + 0.8×T + kw

mult = (0.85 if summoning_sick else 1.0) × (0.90 if tapped else 1.0)

V_creature = stat × mult
```

**Noncreature Value:**

| Permanent Type     | Value                                              |
| ------------------ | -------------------------------------------------- |
| Planeswalker       | `V_pw = 1.2×loyalty + 1.5×is_engine + 1.5×is_lock` |
| Engine (draw)      | +3.0                                               |
| Engine (tokens)    | +2.0                                               |
| Lock piece         | +3.5                                               |
| Anthem/buff        | +1.5                                               |
| Utility (ramp/fix) | +0.7 _(mostly counted in Resources)_               |

**Raw Value:**

```
BP_raw = ΣV_you - ΣV_opp
```

**Normalized:** `BoardPresence = norm(BP_raw, C=12)`

---

### 6.3 Tempo

**Purpose:** Measure time advantage, mana efficiency, and initiative.

#### Features

- `unspent_mana(you)` — If tracked, else approximate from mana pool + untapped sources
- `DNT_you_to_opp` and `DNT_opp_to_you` — See [Life Pressure](#65-life-pressure)
- Time-walk effects — Key permanents tapped/frozen/bounced

#### Formula

> **Note:** Tempo is best computed as a delta metric if you have per-unit info. For snapshot-only estimate:

**Initiative Proxy:**

```
Init_raw = (1 / ClockOpp) - (1 / ClockYou)
```

_(Using clocks from Life Pressure, capped)_

**Efficiency Proxy:**

```
Eff_raw = (untapped_mana_sources(you) - untapped_mana_sources(opp)) × 0.3
```

_(Crude but works early)_

**Raw Value:**

```
T_raw = 1.2 × Init_raw + 0.8 × Eff_raw
```

**Normalized:** `Tempo = norm(T_raw, C=2)` _(tempo is naturally smaller-scale)_

---

### 6.4 Card Advantage

**Purpose:** Measure net and repeatable access to cards.

#### Features

- `hand_count(you)` — Or exact list
- `hand_count(opp)` — May be hidden → count only
- Engine draw permanents
- Recastable resources (flashback, escape, etc.) via tags

#### Formula

```
RawCA = hand_you - hand_opp

EngΔ = (3.0 × engine_draw_you + 2.0 × tutor_engine_you) - (same for opp)

RecastΔ = 0.5 × (recastable_you - recastable_opp)  // Optional

CA_raw = 1.0 × RawCA + 0.8 × EngΔ + 0.5 × RecastΔ
```

**Normalized:** `CardAdvantage = norm(CA_raw, C=8)`

---

### 6.5 Life Pressure

**Purpose:** Measure clocks, not life totals.

#### Features

Estimate **Damage Next Turn (DNT)**:

- `attackers` = creatures that can plausibly attack next combat
- `evasion_mult`:
    - Flying with no flying/reach blockers: **1.2**
    - Menace: **1.1**
    - Trample: **1.1**

**Block Estimate (simple):**

```
block_est = min(sum_blocker_toughness, sum_attacker_power) × 0.5
```

_(Very rough; good enough for v1)_

#### Formula

**Damage Next Turn:**

```
DNT_you = max(0, Σ(attacker_power × evasion_mult) - block_est_by_opp)
DNT_opp = max(0, Σ(attacker_power × evasion_mult) - block_est_by_you)
```

**Clocks:**

```
ClockOpp = opp_life / max(1, DNT_you + Reach2_you)
ClockYou = you_life / max(1, DNT_opp + Reach2_opp)
```

**Pressure:**

```
LP_raw = (1 / ClockOpp) - (1 / ClockYou)
```

**Normalized:** `LifePressure = norm(LP_raw, C=1)`

---

### 6.6 Inevitability

**Purpose:** Measure long-run advantage if game drags.

#### Features (tag-based)

- `engine_draw`, `engine_tokens`, `mana_engine`, `planeswalker_threat`, `lock_piece`
- **Resilience:** ward/hexproof on engines, recursion density

#### Formula

**Weights:**
| Tag | Weight |
|-----|--------|
| `w_draw` | 2.0 |
| `w_token` | 1.2 |
| `w_mana` | 1.3 |
| `w_pw` | 1.5 |
| `w_lock` | 2.2 |

**Engine Strength Index:**

```
ESI(you) = w_draw × draw_engines + w_token × token_engines + w_mana × mana_engines + w_pw × pw_engines + w_lock × locks
```

**Resilience:**

```
Res(you) = 0.5 × protected_engines + 0.3 × recursion_sources
```

**Raw Value:**

```
IN_raw = (ESI(you) + Res(you)) - (ESI(opp) + Res(opp))
```

**Normalized:** `Inevitability = norm(IN_raw, C=10)`

---

### 6.7 Flexibility

**Purpose:** Measure "How many good lines do I have?"

#### Features

- `playable_spells_in_hand_next_turn`
- `instant_speed_interaction_count`
- `modal_count`
- `card_selection` (scry, surveil, rummage) engines

#### Formula

**Option Count:**

```
OC(you) = playable + 0.6 × instant + 0.3 × modal + 0.4 × selection_sources
```

**Raw Value:**

```
F_raw = OC(you) - OC(opp)
```

**Normalized:** `Flexibility = norm(F_raw, C=6)`

---

### 6.8 Risk / Information

**Purpose:** Mix of info advantage and fragility.

#### Features

**Information:**

- `known_cards_in_opp_hand_to_you` (revealed)
- `known_cards_in_you_hand_to_opp`

**Exposure Indicators:**

- Wide board without protection
- Single point of failure engine
- Low life vs reach
- Behind on cards with no engine (high topdeck risk)

#### Formula

**Information:**

```
Info_raw = known_opp - known_you
```

**Exposure (negative):**

```
Exp_raw = 0.7 × wide_unprotected + 0.7 × single_engine + 0.7 × low_life_vs_reach + 0.4 × no_engine_topdeck_mode
```

**Raw Value:**

```
RI_raw = Info_raw - Exp_raw
```

**Normalized:** `RiskInformation = norm(RI_raw, C=4)`

---

### 6.9 Synergy / Gameplan / Reach

**Purpose:** Measure cohesion of strategy and ability to close games.

> This is easiest if you already compute synergy tags.

#### Features

**Synergy:**

- Enablers and payoffs present on board/hand
- Overlap in mechanic/archetype tags

**Gameplan Alignment:**

- Compare state profile to archetype template (aggro/control/combo/midrange)

**Reach:**

- `Reach2_you`: Expected non-combat damage/drain in next 2 turns

#### Formula

**Synergy:**

```
Sy(you) = 0.8 × match_count(enablers, payoffs) + 0.2 × redundant_enablers
```

**Alignment:**

Make archetype templates as feature vectors:

| Archetype | Profile                                                    |
| --------- | ---------------------------------------------------------- |
| Aggro     | High LifePressure, medium BoardPresence, low Inevitability |
| Control   | High CardAdvantage + Inevitability, low LifePressure early |
| Combo     | High Flexibility + combo_piece_count + protection          |

```
Align(you) = cosine_similarity(state_features, archetype_profile)  // in [0..1]
```

**Reach:**

```
Reach(you) = min(1, Reach2_you / max(1, opp_life))
```

**Combined:**

```
SGR_raw = (0.9 × Sy(you) + 1.1 × Align(you) + 1.0 × 2 × Reach(you)) - (same for opp)
```

**Normalized:** `SynergyGameplanReach = norm(SGR_raw, C=6)`

---

### 6.10 Explosiveness

**Purpose:** Measure potential for sudden, dramatic state changes within 1-2 turns.

#### Concept

Explosiveness captures the "powder keg" nature of certain board states—positions where a player can rapidly transform a stable situation into a decisive advantage. Unlike Tempo (which measures current initiative) or Life Pressure (which measures steady clocks), Explosiveness measures **latent burst potential**.

#### Features

**Burst Damage Sources:**

- Creatures with haste enablers available
- Pump effects (anthems, combat tricks in hand)
- Direct damage spells/abilities
- Sacrifice outlets with fodder

**Board Multiplication:**

- Token generators with payoffs
- "Go wide" enablers (Overrun effects, mass pump)
- Untap effects for mana/attack doubling

**Combo Proximity:**

- Pieces in play vs. pieces needed
- Tutors available to find missing pieces
- Protection available for combo turn

**Storm/Chain Potential:**

- Mana generation exceeding spell costs
- Card draw engines online
- Recursion loops available

#### Formula

**Burst Damage Potential:**

```
Burst(you) = haste_damage_available + pump_ceiling + direct_damage_in_hand_or_ability
```

Where:

- `haste_damage_available` = Σ(power of creatures with haste or haste-enabler present)
- `pump_ceiling` = estimated max power boost from available effects
- `direct_damage_in_hand_or_ability` = burn spells + activated damage abilities

**Multiplication Factor:**

```
Mult(you) = 1.0 + 0.5×token_doublers + 0.3×untap_effects + 0.4×extra_combat
```

**Combo Proximity Index:**

```
CPI(you) = (pieces_in_play / pieces_needed) × protection_modifier × tutor_boost
```

Where:

- `protection_modifier` = 1.0 + 0.3×counterspells_in_hand + 0.2×hexproof_pieces
- `tutor_boost` = 1.0 + 0.4×tutors_available (if missing pieces ≤ tutors)

**Storm Index:**

```
Storm(you) = min(1, (net_mana_generation × draw_rate) / threshold)
```

> Threshold typically 5-10 depending on format.

**Raw Value:**

```
EX_raw = (Burst(you) × Mult(you) + 4.0×CPI(you) + 3.0×Storm(you)) - (same for opp)
```

**Normalized:** `Explosiveness = norm(EX_raw, C=15)`

#### Interpretation

| Value       | Meaning                                          |
| ----------- | ------------------------------------------------ |
| > 0.7       | Threatening lethal burst; opponent must react    |
| 0.3 to 0.7  | Significant burst potential; influences blocking |
| -0.3 to 0.3 | Neutral; game proceeds on established axes       |
| < -0.3      | Opponent has burst advantage; defensive posture  |

---

## 7. Card Formations

### 7.1 Overview

Card Formations represent structured synergy relationships between cards using a **directed graph-based syntax**. Unlike simple tag matching, formations model the actual flow of effects, triggers, and resources between cards, enabling precise calculation of synergy output.

### 7.2 Formation Graph Structure

A formation is a directed acyclic graph (DAG) where:

- **Nodes** represent cards or card-like objects (tokens, emblems)
- **Edges** represent effect relationships with typed connections
- **Weights** represent strength/frequency of the connection

```json
{
    "formation_id": "aristocrats_core",
    "nodes": [
        { "id": "n1", "card_ref": "Viscera Seer", "role": "enabler" },
        { "id": "n2", "card_ref": "Blood Artist", "role": "payoff" },
        { "id": "n3", "card_ref": "Reassembling Skeleton", "role": "fuel" }
    ],
    "edges": [
        { "from": "n1", "to": "n3", "type": "sacrifice", "axis": "creature_death" },
        { "from": "n3", "to": "n2", "type": "trigger", "axis": "creature_death" },
        { "from": "n3", "to": "n3", "type": "recursion", "axis": "graveyard_to_battlefield" }
    ],
    "output": {
        "type": "damage_drain",
        "per_activation": 1,
        "repeatable": true,
        "mana_per_cycle": 2
    }
}
```

### 7.3 Node Roles

| Role          | Description                         | Examples                            |
| ------------- | ----------------------------------- | ----------------------------------- |
| **Enabler**   | Initiates the synergy chain         | Sacrifice outlets, tap effects      |
| **Payoff**    | Generates value from the synergy    | Death triggers, +1/+1 counter lords |
| **Fuel**      | Consumed or cycled by the formation | Tokens, recursive creatures         |
| **Amplifier** | Multiplies formation output         | Doublers, additional triggers       |
| **Protector** | Guards critical formation pieces    | Hexproof granters, counterspells    |
| **Tutor**     | Can find missing formation pieces   | Tutors, card selection              |

### 7.4 Edge Types and Axes of Interaction

Edges are typed by their mechanical relationship and classified along **axes of interaction**:

#### Edge Types

| Type          | Description                             | Symbol |
| ------------- | --------------------------------------- | ------ |
| `trigger`     | A triggers B                            | `→T`   |
| `sacrifice`   | A sacrifices B                          | `→S`   |
| `target`      | A targets B with effect                 | `→X`   |
| `buff`        | A grants stats/abilities to B           | `→B`   |
| `produce`     | A creates B (tokens, mana)              | `→P`   |
| `recursion`   | A returns B from graveyard/exile        | `→R`   |
| `cost_reduce` | A reduces cost of B                     | `→C`   |
| `enable`      | A unlocks B's ability (threshold, etc.) | `→E`   |
| `protect`     | A protects B from interaction           | `→Π`   |
| `copy`        | A copies B (spell or permanent)         | `→Δ`   |

#### Axes of Interaction

Axes describe the game-mechanical domain of the interaction:

| Axis                       | Description                            | Typical Cards                    |
| -------------------------- | -------------------------------------- | -------------------------------- |
| `creature_death`           | Death triggers and sacrifice synergies | Blood Artist, Grave Pact         |
| `creature_etb`             | Enter-the-battlefield triggers         | Panharmonicon, Essence Warden    |
| `creature_ltb`             | Leave-the-battlefield triggers         | Reveillark, Fiend Hunter loops   |
| `spell_cast`               | Cast triggers (prowess, storm)         | Monastery Mentor, Guttersnipe    |
| `noncreature_spell_cast`   | Subset of spell_cast                   | Prowess, Young Pyromancer        |
| `mana_production`          | Mana generation synergies              | Birgi, Nyxbloom Ancient          |
| `life_gain`                | Lifegain triggers                      | Soul Warden, Ajani's Pridemate   |
| `life_loss_opponent`       | Opponent life loss triggers            | Vito, Sanguine Bond              |
| `card_draw`                | Draw triggers                          | Teferi's Ageless Insight         |
| `discard`                  | Discard synergies                      | Madness, Waste Not               |
| `graveyard_to_battlefield` | Reanimation axis                       | Persist, Unearth                 |
| `graveyard_to_hand`        | Recursion to hand                      | Regrowth effects                 |
| `counter_placement`        | +1/+1 or other counter synergies       | Hardened Scales, Winding Constr. |
| `token_creation`           | Token generation axis                  | Anointed Procession, Doubling S. |
| `combat_damage`            | Combat damage triggers                 | Coastal Piracy, Reconnaissance   |
| `attack_trigger`           | Attack declaration triggers            | Aurelia, Hellrider               |
| `tap_untap`                | Tap/untap synergies                    | Intruder Alarm, Seedborn Muse    |

### 7.5 Formation Output Calculation

Formation output is calculated by traversing the graph and accumulating effects:

#### Output Schema

```json
{
  "output_type": "damage" | "life_gain" | "card_draw" | "mana" | "tokens" | "counters" | "mill" | "composite",
  "value_per_activation": number,
  "activations_per_turn": number | "unlimited",
  "limiting_factor": "mana" | "cards" | "life" | "once_per_turn" | "none",
  "net_resource_cost": {
    "mana": number,
    "cards": number,
    "life": number
  },
  "repeatability": "one_shot" | "per_turn" | "unlimited"
}
```

#### Calculation Algorithm

```
function calculateFormationOutput(formation, gameState):
    // 1. Identify active nodes (cards in play/hand)
    activeNodes = formation.nodes.filter(n => isAvailable(n, gameState))

    // 2. Check formation completeness
    completeness = activeNodes.length / formation.nodes.length
    if completeness < formation.min_completeness:
        return NULL_OUTPUT

    // 3. Find limiting resource
    limiter = findLimitingFactor(formation, gameState)

    // 4. Calculate iterations possible
    iterations = calculateMaxIterations(formation, limiter, gameState)

    // 5. Traverse graph and accumulate output
    totalOutput = 0
    for each path in topologicalSort(formation.edges):
        pathOutput = calculatePathOutput(path, iterations)
        totalOutput += pathOutput × getAmplifierMultiplier(path)

    return {
        output_type: formation.output.type,
        value: totalOutput,
        iterations: iterations,
        limiter: limiter
    }
```

### 7.6 Redundancy and Cumulative Types

Formations can have multiple cards filling the same role, providing **redundancy** or **cumulative** effects:

#### Redundancy Types

| Type              | Description                                          | Example                       |
| ----------------- | ---------------------------------------------------- | ----------------------------- |
| **Substitutable** | Multiple cards can fill same role (OR relationship)  | Any sac outlet works          |
| **Cumulative**    | Multiple copies stack for greater effect (AND bonus) | Multiple Blood Artists        |
| **Threshold**     | Need minimum count to function                       | Tribal critical mass          |
| **Diminishing**   | Each additional copy provides less value             | Third mana dork less valuable |

#### Redundancy Schema

```json
{
    "role": "payoff",
    "redundancy_type": "cumulative",
    "cards": ["Blood Artist", "Zulaport Cutthroat", "Bastion of Remembrance"],
    "cumulative_formula": "linear",
    "base_effect": 1,
    "per_additional": 1,
    "cap": null
}
```

#### Cumulative Formulas

| Formula       | Description                            | Output for n copies             |
| ------------- | -------------------------------------- | ------------------------------- |
| `linear`      | Each copy adds full value              | `base × n`                      |
| `diminishing` | Each copy adds less                    | `base × (1 + 0.5 + 0.33 + ...)` |
| `exponential` | Multipliers stack (rare, often banned) | `base × 2^(n-1)`                |
| `threshold`   | Binary once threshold met              | `0 if n < k else base`          |
| `synergistic` | Copies interact with each other        | `base × n × (n-1) / 2`          |

### 7.7 Formation Templates

Common archetypical formations:

#### Aristocrats Formation

```
[Sac Outlet] →S [Fodder] →T [Death Payoff]
     ↑                         |
     └─────── [Recursion] ←────┘
```

#### Blink Formation

```
[Blink Engine] →X [ETB Value] →T [Resource Gain]
      ↑              |
      └── [Untapper] ┘
```

#### Storm Formation

```
[Cost Reducer] →C [Cantrips] →T [Storm Payoff]
       ↓              ↑
  [Mana Engine] →P ───┘
```

#### Tokens-Matter Formation

```
[Token Producer] →P [Tokens] →E [Payoff]
       ↑               |
  [Doubler] →×2 ───────┘
```

### 7.8 Formation Evaluation Integration

Formations integrate with the main evaluation framework:

```
FormationValue(you) = Σ formations × (completeness × output_value × repeatability_factor)

Where:
  - completeness = active_nodes / required_nodes
  - output_value = calculated output per turn
  - repeatability_factor = {one_shot: 0.5, per_turn: 1.0, unlimited: 1.5}
```

This value feeds into:

- **Synergy dimension** (formation alignment)
- **Explosiveness dimension** (burst formations)
- **Inevitability dimension** (engine formations)

### 7.9 Example Formation Analysis

**Board State:**

- Viscera Seer (sac outlet)
- Blood Artist (death payoff)
- 3x 1/1 Zombie tokens (fodder)
- 5 mana available
- Opponent at 8 life

**Formation Detection:**

```json
{
    "formation": "aristocrats_burst",
    "completeness": 1.0,
    "nodes_active": ["Viscera Seer", "Blood Artist", "Zombie×3"],
    "missing": [],
    "output_calculation": {
        "per_sacrifice": {
            "scry": 1,
            "damage_to_opponent": 1,
            "life_gain": 1
        },
        "available_sacrifices": 3,
        "total_damage": 3,
        "total_life_gain": 3,
        "mana_cost": 0
    },
    "analysis": {
        "lethal": false,
        "damage_percentage_of_life": 0.375,
        "repeatability": "limited_by_fodder"
    }
}
```

---

## 8. Learning Helper Statistics

### 8.1 Overview

Learning Helper Statistics are **per-turn, at-a-glance indicators** designed to help players quickly assess whether their game is developing healthily. Unlike the full evaluation dimensions in Section 6, these statistics are intentionally simple, require no card metadata database, and can be computed from the game log alone (or enriched with deck data when available).

They answer four practical questions every turn:

1. **Did I make my land drop?**
2. **What mana do I have available?**
3. **What can I cast from hand?**
4. **How well does my mana base cover my deck's color needs?**

### 8.2 Land Drop Rating

**Purpose:** Immediately flag turns where the player missed a land drop (falling behind on curve) or made multiple land drops (acceleration).

#### Rating Scale

| Lands Played This Turn | Rating | Symbol | Interpretation |
|------------------------|--------|--------|----------------|
| 0 | **Bad** | 🔴 | Missed land drop — falling behind on mana curve |
| 1 | **Good** | 🟢 | Normal development — on curve |
| ≥ 2 | **Super** | 🌟 | Accelerated development — ahead on curve |

#### Formula

```
LandDropRating(turn) =
    if landsPlayed == 0: "bad"
    if landsPlayed == 1: "good"
    if landsPlayed >= 2: "super"
```

#### Context Enrichment

When cumulative land count is available:

```
ExpectedLands(turn) = turn_number  (in early game, turns 1-4)
LandDeficit = ExpectedLands - CumulativeLandsOnBattlefield
```

A running deficit compounds the severity of a missed drop.

### 8.3 Available Mana

**Purpose:** Show the player exactly what colors and total amount of mana they can produce this turn.

#### Data Sources

- **Mana-producing lands** on battlefield (from cumulative zone tracking)
- **Mana dorks / rocks** on battlefield (if tagged via deck data)
- **Mana produced events** from the log (`mana_tap` events with `manaProduced`)

#### Representation

Mana is displayed as a color-coded pool:

```
Available Mana: {W: 2, U: 1, B: 0, R: 0, G: 3, C: 1} = 7 total
```

Where WUBRG follows the standard MTG color ordering:
- **W** = White (☀️)
- **U** = Blue (💧)
- **B** = Black (💀)
- **R** = Red (🔥)
- **G** = Green (🌿)
- **C** = Colorless (◇)

#### Estimation from Lands

When exact mana data is unavailable, estimate from land names on battlefield:

```
EstimateManaFromLands(lands):
    for each land in lands:
        if land is basic or recognized dual/fetch/shock:
            add known color(s)
        else:
            // Use deck card data if available for is_manaproducing + mana_producing_colors
            add inferred color(s)
    return color_pool
```

### 8.4 Cast Options in Hand

**Purpose:** Show which spells in hand can actually be cast given current available mana.

#### Prerequisites

- Hand contents (from cumulative zone tracking)
- Available mana pool (from Section 8.3)
- Card mana costs (from deck card data: `extendedCardInfo.cardfaces[0].mana_cost`)

#### Castability Check

For each non-land card in hand:

```
CanCast(card, mana_pool):
    cost = ParseManaCost(card.mana_cost)  // e.g., "{2}{W}{U}" → {generic: 2, W: 1, U: 1}

    // 1. Check colored requirements
    for each color C in cost.colored:
        if mana_pool[C] < cost[C]: return false

    // 2. Check total mana (generic can use any color)
    remaining = total(mana_pool) - sum(cost.colored)
    if remaining < cost.generic: return false

    return true
```

#### Display

Cards are split into two groups:

| Group | Style | Meaning |
|-------|-------|---------|
| **Castable** | Green highlight, full opacity | Can be cast with current mana |
| **Not Castable** | Red/grey, reduced opacity | Cannot be cast — shows what's missing |

### 8.5 Mana Color Coverage

**Purpose:** Assess how well the current mana base covers the deck's color requirements.

Two sub-metrics are computed:

#### 8.5.1 Commander Color Coverage

For Commander format decks:

```
CommanderColors = deck.ColorIdentity  // e.g., ["W", "U", "B"]
AvailableColors = colors present in mana pool with count ≥ 1

Coverage_Commander = |AvailableColors ∩ CommanderColors| / |CommanderColors|
```

| Coverage | Rating | Meaning |
|----------|--------|---------|
| 100% | 🟢 Full | All commander colors accessible |
| 50–99% | 🟡 Partial | Some colors missing — potential casting problems |
| < 50% | 🔴 Poor | Major color screw — most spells uncastable |

#### 8.5.2 Deck Spell Castability Percentage

Independent of what's in hand — measures what **percentage of all spells in the deck** could theoretically be cast with current mana:

```
CastablePercentage(mana_pool, deck):
    castable_count = 0
    spell_count = 0

    for each card in deck.MainCards:
        if card is land: continue
        spell_count++

        cost = ParseManaCost(card.mana_cost)
        // Only check color requirements (ignore generic/total mana)
        // This measures COLOR FIXING, not mana quantity
        color_satisfied = true
        for each color C in cost.colored:
            if C != "C" and mana_pool[C] < cost[C]:  // Ignore generic colorless
                color_satisfied = false
                break

        if color_satisfied:
            castable_count++

    return castable_count / spell_count
```

> **Note:** This deliberately ignores colorless/generic mana requirements. A `{4}{W}{W}` spell counts as castable if `W ≥ 2`, regardless of total mana. This isolates **color fixing quality** from **mana quantity**.

| Percentage | Rating | Meaning |
|------------|--------|---------|
| ≥ 90% | 🟢 Excellent | Mana base covers nearly all spells |
| 70–89% | 🟡 Adequate | Some spells require colors not yet available |
| < 70% | 🔴 Deficient | Significant portion of deck is color-locked |

### 8.6 Integration with Evaluation Framework

Learning Helper Statistics feed into the main evaluation dimensions as lightweight proxies:

| Statistic | Feeds Into | Relationship |
|-----------|-----------|--------------|
| Land Drop Rating | Resources (6.1) | Direct component of MPP |
| Available Mana | Resources (6.1), Flexibility (6.7) | Determines action capacity |
| Cast Options | Flexibility (6.7) | Directly maps to option count |
| Color Coverage | Resources (6.1) | Directly maps to Fix(you) |

These statistics can be computed without any card metadata database, making them suitable for quick feedback even when full evaluation is unavailable.

---

## 9. Evaluation Before and After Decisions

For each L2 Unit, the evaluation vector is computed **both before and after resolution**. The difference between these vectors represents the effect of the decision **independent of game outcome variance**.

This delta-centric approach enables feedback such as:

- Tempo gain vs. card disadvantage
- Increased inevitability at the cost of risk
- Board presence improvement vs. flexibility reduction

---

## 10. Applications

The framework enables multiple applications:

| Application              | Description                              |
| ------------------------ | ---------------------------------------- |
| **Interactive Replay**   | Step-by-step learning tools              |
| **Automated Coaching**   | Mistake detection and feedback           |
| **Dataset Generation**   | For supervised or reinforcement learning |
| **Comparative Analysis** | Analysis of alternative lines of play    |

---

## 11. Limitations and Future Work

The proposed heuristics are approximations and require tuning per format.

**Future work includes:**

- Learning-based value functions
- Archetype inference
- Multiplayer extensions
- Empirical validation using large-scale replay data

---

## 12. Conclusion

This paper presents a unified notation and evaluation framework for Magic: The Gathering that bridges rules-accurate replay and learning-oriented analysis. By separating full-detail logs from pedagogical abstractions and adopting a vector-based evaluation model, the framework provides a foundation for explainable analysis, coaching, and AI research in complex card games.

---

## Appendix A: Notation Syntax and Semantics

### A.1 Overview

The MTG Replay & Learning Notation is a **JSON-based formal language** designed to represent complete games of Magic: The Gathering in a deterministic, analyzable, and learning-oriented manner. The notation employs a two-level architecture separating rule-complete execution data from decision-centric analytical abstractions.

### A.2 Top-Level File Structure

A replay file is a single JSON object containing:

```json
{
  "format": "mtg-replay",
  "version": "1.0.0",
  "meta": { ... },
  "seed": number,
  "card_index": { ... },
  "initial_state": { ... },
  "log_l1": [ ... ],
  "views_l2": [ ... ]
}
```

| Field           | Role                                                           |
| --------------- | -------------------------------------------------------------- |
| `meta`          | Contextual metadata (does not affect replay determinism)       |
| `seed`          | Fixes all stochastic processes, guaranteeing replayability     |
| `card_index`    | Static card universe referenced by object identifiers          |
| `initial_state` | Complete starting configuration of the game                    |
| `log_l1`        | Authoritative chronological record of the game                 |
| `views_l2`      | Derived, learning-oriented representation of decision contexts |

### A.3 Object Identity Model

All game entities are represented by **immutable identifiers** that never change throughout the game, even when objects move between zones.

| Entity Type    | Prefix | Example |
| -------------- | ------ | ------- |
| Card/Permanent | `c`    | `c42`   |
| Token          | `t`    | `t7`    |
| Stack Object   | `s`    | `s1`    |
| Player         | `P`    | `P1`    |

### A.4 Temporal Syntax

Each event is annotated with a human-readable time marker:

```
T<turn>.<phase>[:<priority_pass>]
```

**Examples:**

- `T1.UP` — Turn 1, Upkeep
- `T3.MP1:2` — Turn 3, Main Phase 1, second priority window

These markers impose a total ordering on events while preserving semantic information about turn structure.

### A.5 Zones and State Semantics

Zones are explicitly represented containers. All zone transitions are explicit and no implicit state changes are permitted.

| Category                  | Zones                                                 |
| ------------------------- | ----------------------------------------------------- |
| **Shared Zones**          | `battlefield`, `stack`, `exile`                       |
| **Player-Specific Zones** | `P#:hand`, `P#:library`, `P#:graveyard`, `P#:command` |

### A.6 Level 1 Event Structure

```json
{
  "i": number,
  "t": string,
  "a": "P1" | "P2" | "SYS",
  "type": string,
  "data": { ... }
}
```

| Field  | Meaning                    |
| ------ | -------------------------- |
| `i`    | Strict chronological order |
| `t`    | Temporal marker            |
| `a`    | Actor (player or system)   |
| `type` | Semantic meaning of event  |
| `data` | Event-specific payload     |

### A.7 Event Typology

#### Player Decision Events

Strategic choices — primary targets of evaluation:

| Event               | Description                 |
| ------------------- | --------------------------- |
| `CAST`              | Cast a spell                |
| `ACTIVATE`          | Activate an ability         |
| `PLAY_LAND`         | Play a land                 |
| `DECLARE_ATTACKERS` | Declare attacking creatures |
| `DECLARE_BLOCKERS`  | Declare blocking creatures  |
| `MULLIGAN`          | Take a mulligan             |
| `CHOOSE`            | Make a choice               |
| `PASS_PRIORITY`     | Pass priority               |

> **Key rule:** Decision events declare intent but do not directly change game state.

#### System Events

Automatic consequences mandated by rules:

| Event          | Description              |
| -------------- | ------------------------ |
| `PUT_ON_STACK` | Object placed on stack   |
| `TRIGGER`      | Triggered ability        |
| `RESOLVE`      | Stack object resolves    |
| `MOVE`         | Zone transition          |
| `DAMAGE`       | Damage dealt             |
| `LIFE`         | Life total change        |
| `COUNTERS`     | Counter modification     |
| `STATE_BASED`  | State-based action       |
| `PHASE_CHANGE` | Phase/step transition    |
| `RANDOM`       | Random event (with seed) |

> System events are not subject to strategic evaluation.

### A.8 Stack and Resolution Model

Every spell, ability, or trigger placed on the stack is assigned a stack object ID via a `PUT_ON_STACK` event.

**Stack objects explicitly record:**

- Source
- Controller
- Targets
- Choices

Resolution order is determined solely by the sequence of `RESOLVE` events, ensuring deterministic execution of even deeply nested stack interactions.

### A.9 State Snapshots

Game states are explicit snapshots consisting of:

- Turn, phase, step, and priority
- Player state (life, mana pool, counters)
- Zone contents
- Object attributes (power, toughness, counters, attachments, status flags)

> Hidden information is represented via counts or visibility flags, enabling both training-mode and spectator-mode interpretations.

### A.10 Level 2 Learning Units

L2 Units correspond closely to how players conceptualize gameplay:

> "Given this position, I chose to do X, which resulted in Y."

This abstraction enables:

- Before/after evaluation
- Delta-based learning feedback
- Comparison of alternative lines

### A.11 Determinism and Validation

All random processes are explicitly seeded. Given identical initial state, event log, and seed, a replay is guaranteed to produce the same outcome.

**Validation Rules:**

1. Sequential event indices starting at zero
2. Monotonic time progression
3. Valid object and player references
4. Explicit zone transitions
5. Referential integrity between L1 and L2
6. Supported format version

### A.12 Non-Functional Requirements

- Must be doable easily for a human while at a tournament
- Must be derivable from video capture
