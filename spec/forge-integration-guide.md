# Forge Integration Guide

**Applies to:** Commander Decklist Spec v1.2.0+
**Forge version tested:** Forge 1.6.x (open-source MTG AI simulator)

---

## 1. What Forge Expects

Forge reads decks in a plain-text `.dck` format. It does not consume JSON directly.
The commander-decklist-spec JSON is your **source of truth** — you convert it to `.dck`
on export. The `deck_rules.simulation` block carries the simulation parameters that
cannot be expressed inside `.dck` and must be applied manually (or by the MaMo
connector) when launching Forge.

---

## 2. The `.dck` File Format

A Forge Commander `.dck` file looks like this:

```
[metadata]
Name=Atraxa Superfriends

[Commander]
1 Atraxa, Praetors' Voice

[Main]
1 Sol Ring
1 Arcane Signet
1 Doubling Season
1 Smothering Tithe
1 Rhystic Study
... (95 more lines)

[Sideboard]
1 Grafdigger's Cage
```

### Rules

- Section headers are `[metadata]`, `[Commander]`, `[Main]`, `[Sideboard]`.
- Each card line is `<quantity> <Exact English card name>`.
- The `[metadata]` block supports `Name=`, `Filename=`, `Description=`, and the
  MaMo-specific keys below.
- `[Sideboard]` is optional; `[Commander]` is required for Commander format.
- Maybeboard has no equivalent in `.dck` — omit those cards.
- Card names must match Forge's internal card database exactly (same as Scryfall
  English names).

### MaMo-specific metadata keys (v1.2.0+)

| Key | Description |
|-----|-------------|
| `DeckURL` | Source URL the deck was imported from (e.g. Moxfield link). Stored in `meta.source_url` of the companion JSON. Populated in the replay export as `deck_link`. |
| `EvalScenario` | Comma-separated list of `eval_sequence` scenario IDs to activate. References `deck_rules.scenarios[].id` in the companion JSON. |

Example:

```ini
[metadata]
Name=Red Aggro Test
DeckURL=https://moxfield.com/decks/abc123
EvalScenario=eval_7plus3_aggro,eval_14_flat

[Commander]
1 Krenko, Mob Boss

[Main]
4 Lightning Bolt
...
```

---

## 3. Converting the JSON Spec to `.dck`

Given the commander-decklist-spec JSON, extract sections as follows:

```
[metadata]
Name=<meta.deck_name>

[Commander]
<quantity> <name>    ← for each entry in "commander"

[Main]
<quantity> <name>    ← for each entry in "main"

[Sideboard]
<quantity> <name>    ← for each entry in "sideboard" (omit section if empty)
```

- `edition` and `collector_number` are **ignored** by `.dck` — Forge resolves its own
  printing internally.
- `primary_mechanic` and `additional_mechanics` are **ignored** — they are MaMo metadata
  only.
- `maybeboard` entries are **omitted**.

### Example Conversion

**Input JSON (excerpt):**

```json
{
    "meta": { "deck_name": "Atraxa Superfriends" },
    "commander": [
        { "quantity": 1, "name": "Atraxa, Praetors' Voice" }
    ],
    "main": [
        { "quantity": 1, "name": "Sol Ring" },
        { "quantity": 1, "name": "Doubling Season" }
    ],
    "sideboard": []
}
```

**Output `.dck`:**

```
[metadata]
Name=Atraxa Superfriends

[Commander]
1 Atraxa, Praetors' Voice

[Main]
1 Sol Ring
1 Doubling Season
```

---

## 4. Applying the Simulation Config

The `deck_rules.simulation` block maps to Forge's game setup options.
These are applied in Forge's **New Game** dialog (or via command-line flags if using
Forge in headless mode).

```json
{
    "simulation": {
        "target": "forge",
        "play_order": "random",
        "difficulty": "ultimate",
        "starting_life": 40,
        "eval_scenario_ids": ["eval_7plus3_aggro", "eval_14_flat"]
    }
}
```

| Spec field | Where to set in Forge |
|---|---|
| `play_order: "play"` | New Game → "Player goes first" |
| `play_order: "draw"` | New Game → "AI goes first" |
| `play_order: "random"` | New Game → default (leave unset) |
| `difficulty` | New Game → AI Difficulty dropdown |
| `starting_life` | New Game → Starting life (default 40 for Commander) |
| `eval_scenario_ids` | See §5 below — replaces `use_best_starting_hand` / `use_perfect_game` |

Forge does not accept these as `.dck` metadata — they must be set in the UI or applied
programmatically via `GameReplaySimulation.applyForcedLibraryOrder()` (see §5.1).

---

## 5. Using Scenarios in Forge

### 5.1 `eval_sequence` scenario (v1.2.0+)

The `eval_sequence` type is the primary scenario type for evaluation runs.
It defines a complete draw + play sequence with an optional board state snapshot.

Two execution modes are supported:

**Forced mode** — Forge uses a fixed library order so the exact sequence is
reproduced every game:

```java
// In mamo-Connector or simulation harness:
List<String> ids = GameReplaySimulation.getEvalScenarioIds(deck);
// ids = ["eval_7plus3_aggro", "eval_14_flat"] (from EvalScenario metadata)

// For each forced-mode scenario:
//   1. Load the companion JSON and find the scenario by id
//   2. Resolve group refs {"group":"ramp"} → first matching card in deck list
//   3. Build the ordered list: opening_hand cards + turns[].drawn cards
List<String> cardOrder = resolveScenario(deck, scenarioJson);

GameReplaySimulation.applyForcedLibraryOrder(rules, playerName, cardOrder);
// This calls rules.setReplayMode(true) and populates forcedLibraryOrder
```

**Look_for mode** (default) — Forge runs normally. After each turn the game
engine evaluates whether the actual state matches the scenario's `board_state`
and `turns` conditions. A match event is logged in the replay when satisfied.

#### Card reference resolution

Group references (`{"group": "ramp"}`) must be resolved before calling
`applyForcedLibraryOrder`. Resolution rule: pick the first card in the deck's
`main` section whose `primary_mechanic` equals the group key.

If no matching card is found, skip that slot (do not crash — the library order
will be shorter than expected).

#### Draw shapes

| Shape | `opening_hand` size | `turns` count | Total drawn |
|-------|--------------------:|:-------------:|:-----------:|
| 7+3   | 7 | 3 | 10 |
| 14    | 14 | 0 | 14 |

### 5.2 `best_starting_hand` scenario

The `best_starting_hand` scenario documents the ideal 7-card opening hand. When
`use_best_starting_hand: true` is set in `simulation`:

1. Find the scenario entry in `deck_rules.scenarios` with `"type": "best_starting_hand"`.
2. Read `opening_hand` — this is the ordered list of card names to use as the opening hand.
3. In Forge's mulligan dialog, use **"No Mulligan"** and manually set the hand to these
   seven cards if using a scripted simulation, or note them as the reference hand.
4. The `turns` array documents which card was drawn each subsequent turn and what was
   played — useful for validating replay fidelity, not required by Forge directly.

**Example scenario entry:**

```json
{
    "id": "scenario_best_hand_1",
    "type": "best_starting_hand",
    "name": "Ideal Doubling Season Opener",
    "opening_hand": [
        "Sol Ring",
        "Arcane Signet",
        "Command Tower",
        "Forest",
        "Plains",
        "Doubling Season",
        "Atraxa, Praetors' Voice"
    ],
    "turns": [
        { "turn": 1, "drawn": "Smothering Tithe", "played": ["Sol Ring", "Command Tower"] },
        { "turn": 2, "drawn": "Rhystic Study",    "played": ["Arcane Signet", "Forest"] },
        { "turn": 3, "drawn": "Deepglow Skate",   "played": ["Plains", "Doubling Season"] }
    ]
}
```

### 5.2 `perfect_game` scenario

Identical structure to `best_starting_hand` but with 10 `turns` entries instead of 3.
This represents the full strategic ideal from Turn 1 through Turn 10 — a reference line
for evaluating how far a real game deviates.

When `use_perfect_game: true` is set in `simulation`, treat the `turns` array as a
scripted play order for Forge's AI to follow (via a custom AI script or manual
observation mode).

### 5.3 `mid_game` and `free_build` scenarios

These are **precondition-based** — they describe a board state rather than a card
sequence. Forge does not natively load board states, but the scenario can be used to:

- **Set up a Forge game state manually** using the debug/cheat mode
  (`Ctrl+Alt+F` in most Forge builds → "Add card to battlefield")
- **Validate a position** by describing what must be on the battlefield before the
  relevant engine fires

The `zone_requirements` array maps to board state setup:

| Spec `zone` | Forge setup location |
|---|---|
| `battlefield` | "Add card to battlefield" in cheat mode |
| `hand` | "Add card to hand" |
| `graveyard` | "Add card to graveyard" |
| `commandZone` | Commander is always in command zone by default |
| `library` | "Add card to library" (top or shuffled) |
| `exile` | "Add card to exile" |

**Example — reading a zone requirement:**

```json
{
    "zone_requirements": [
        {
            "zone": "battlefield",
            "mechanic_groups": ["ramp"],
            "min_count": 2
        },
        {
            "zone": "battlefield",
            "card_names": ["Atraxa, Praetors' Voice"],
            "min_count": 1
        }
    ]
}
```

Meaning: before the scenario fires, ensure at least 2 ramp pieces and Atraxa are on
the battlefield. The `mechanic_groups` field uses the deck's own mechanic group labels
(defined in MaMo's mechanic workshop) — you need to resolve which specific cards belong
to the `"ramp"` group in this deck.

The `focus` block tells you which cards or mechanic groups the scenario is
**demonstrating**, not requiring. Use it to decide which cards to watch during Forge
playback:

```json
"focus": {
    "mechanic_groups": ["counters", "proliferate"],
    "description": "Demonstrates how the counter engine activates at full speed"
}
```

---

## 6. Forge Game Settings Reference

Quick reference for common Commander simulation setups:

| Goal | Forge settings |
|---|---|
| Standard Commander game | Format: Commander, Life: 40, Hand: 7 |
| Test aggressive opener | Difficulty: Hard, Play Order: Play |
| Test resilience | Difficulty: Ultimate, Play Order: Draw |
| Validate combo line | Difficulty: Easy (AI does not interrupt), Play Order: Play |
| Stress-test engine | Difficulty: Ultimate, Play Order: Random, 1000 games |

---

## 7. Limitations

- Forge's `.dck` format does not support mechanic tags, MaMo oracle IDs, or scenario
  metadata — all of that lives in the JSON spec only.
- `starting_life` values other than 20 or 40 require Forge's custom variant settings
  (not all builds support this via the UI).
- Scripted hand loading (for `best_starting_hand`) requires Forge's debug mode or a
  custom game script — it cannot be done through the standard New Game dialog alone.
- Forge's AI does not "follow" a `perfect_game` turn sequence automatically; the turns
  array is a human reference, not a machine instruction for Forge.

---

## 8. Full Export Checklist

Before handing a deck to Forge:

- [ ] Exported `.dck` file has `[Commander]`, `[Main]` sections
- [ ] Card names match Forge's database (check for split cards: `Fire // Ice`)
- [ ] `deck_rules.simulation.target` is `"forge"`
- [ ] Difficulty and play order noted for New Game dialog
- [ ] If `eval_scenario_ids` is set — every listed ID exists in `deck_rules.scenarios`
      and has `"type": "eval_sequence"`
- [ ] `.dck` `[metadata]` includes `DeckURL=` and `EvalScenario=` when applicable
- [ ] Group references (`{"group": "..."}`) resolved to concrete card names for
      `"forced"` mode scenarios
- [ ] If running a `mid_game` / `free_build` scenario — resolve `mechanic_groups` to
      specific card names using the deck's mechanic workshop data before loading Forge

See [eval-scenario-guide.md](./eval-scenario-guide.md) for the full concept reference
and end-to-end testing steps.
