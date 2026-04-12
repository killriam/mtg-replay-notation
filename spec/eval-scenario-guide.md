# Evaluation Scenario Guide

**Applies to:** Commander Decklist Spec v1.2.0+
**Related:** [Commander Decklist Spec](./commander-decklist-spec.md) · [Forge Integration Guide](./forge-integration-guide.md)

---

## 1. What Is an Eval Scenario?

An **eval scenario** is a deck-level definition that describes how an opponent
(test deck) should behave during a simulation or evaluation run. It captures:

- Which cards are drawn in the opening hand
- Which card is drawn each subsequent turn
- Which cards are played each turn and in what order
- Optionally: what the board state should look like at a given point

Scenarios are defined inside the `deck_rules.scenarios` array of the
Commander Decklist Notation JSON, then referenced by ID from
`deck_rules.simulation.eval_scenario_ids`.

The same deck can carry multiple eval scenarios for different draw shapes,
strategies, or testing angles.

---

## 2. The Two Draw Shapes

All eval scenarios reduce to one of two draw shapes:

### Shape A: 7+3 (opening hand + 3 turns)

```
Total cards seen: 10
Structure: opening_hand[7] + turns[3]
```

The player draws 7 cards at game start, then draws 1 additional card each
turn for 3 turns.

```json
{
    "id": "eval_7plus3",
    "type": "eval_sequence",
    "mode": "forced",
    "opening_hand": [ ... 7 entries ... ],
    "turns": [
        { "turn": 1, "drawn": "...", "played": [...] },
        { "turn": 2, "drawn": "...", "played": [...] },
        { "turn": 3, "drawn": "...", "played": [...] }
    ]
}
```

### Shape B: 14 (flat draw)

```
Total cards seen: 14
Structure: opening_hand[14] + no turns
```

Useful for evaluating card quality in isolation without turn sequencing.

```json
{
    "id": "eval_14",
    "type": "eval_sequence",
    "mode": "forced",
    "opening_hand": [ ... 14 entries ... ]
}
```

---

## 3. Card References

Cards in `opening_hand`, `turns[].drawn`, and `turns[].played` can be
specified in two ways:

### Exact name

```json
"Lightning Bolt"
```

Matches one specific card. Forge will put that exact card in the forced
library order.

### Group reference

```json
{ "group": "ramp" }
```

Matches **any card** whose `primary_mechanic` or `additional_mechanics`
includes the named mechanic group key. Useful when the scenario should work
regardless of which specific ramp piece is drawn.

In `forced` mode, Forge resolves a group reference to the first matching
card in the deck list. In `look_for` mode, the game engine checks whether
the actual drawn card belongs to the named group.

---

## 4. Execution Modes

Every eval scenario has a `mode` field (default: `"look_for"`):

| Mode | What happens |
|------|-------------|
| `"forced"` | Forge sets a deterministic library order so the exact draw sequence is executed every game. Group references are resolved to concrete card names before play starts. |
| `"look_for"` | Forge runs normally. After each turn, the game state is checked against the scenario definition. A match event is logged in the replay when all conditions are satisfied. |

Use `"forced"` when you want reproducible data (benchmarking, regression
tests). Use `"look_for"` when you want to know whether a natural game
ever reaches a target state (probability analysis).

---

## 5. Scenario Definition Reference

```json
{
    "id": "eval_7plus3_aggro",
    "type": "eval_sequence",
    "name": "7+3 Aggro Curve",
    "mode": "forced",

    "opening_hand": [
        "Mountain",
        "Mountain",
        { "group": "land" },
        "Lightning Bolt",
        { "group": "aggro" },
        { "group": "aggro" },
        { "group": "aggro" }
    ],

    "turns": [
        {
            "turn": 1,
            "drawn": { "group": "land" },
            "played": ["Mountain", "Lightning Bolt"]
        },
        {
            "turn": 2,
            "drawn": { "group": "aggro" },
            "played": [{ "group": "land" }, { "group": "aggro" }]
        },
        {
            "turn": 3,
            "drawn": { "group": "aggro" },
            "played": [{ "group": "aggro" }]
        }
    ],

    "focus": {
        "mechanic_groups": ["aggro"],
        "card_names": ["Lightning Bolt"],
        "description": "Verify consistent aggro curve reaches T3 with 3 threats"
    }
}
```

### Field summary

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Unique within the deck's `scenarios` array |
| `type` | Yes | Must be `"eval_sequence"` |
| `name` | Yes | Human-readable label |
| `mode` | No | `"forced"` or `"look_for"`. Default: `"look_for"` |
| `opening_hand` | Yes | Array of card references (7 for 7+3, 14 for flat-14) |
| `turns` | No | Per-turn draw + play sequence (omit for flat-14 shape) |
| `board_state` | No | Explicit zone snapshot at a given turn (see §6) |
| `focus` | No | Which cards / groups this scenario is demonstrating |

### Turn entry fields

| Field | Required | Description |
|-------|----------|-------------|
| `turn` | Yes | Turn number (1-indexed) |
| `drawn` | Yes | Card reference drawn at start of this turn |
| `played` | No | Card references played this turn, in cast order |

---

## 6. Board State (optional)

`board_state` captures what the battlefield and other zones look like at a
given point. It complements the turn sequence and is particularly useful for
`"look_for"` mode.

```json
"board_state": {
    "turn": 3,
    "active_player": "self",
    "zones": {
        "hand": [
            { "group": "aggro" }
        ],
        "battlefield": [
            "Mountain",
            "Mountain",
            { "group": "land" },
            { "group": "aggro" }
        ],
        "graveyard": [
            "Lightning Bolt"
        ]
    }
}
```

Zone names: `hand`, `battlefield`, `graveyard`, `library`, `exile`, `commandZone`.

---

## 7. Source URL

Each test deck (and any deck in the system) can carry a `meta.source_url`
pointing to the external page the deck was imported from:

```json
{
    "meta": {
        "deck_name": "Red Aggro",
        "source_url": "https://moxfield.com/decks/abc123",
        ...
    }
}
```

This is also stored as `DeckURL` in the Forge `.dck` `[metadata]` block:

```ini
[metadata]
Name=Red Aggro
DeckURL=https://moxfield.com/decks/abc123
EvalScenario=eval_7plus3_aggro
```

`EvalScenario` is a comma-separated list of scenario IDs to activate. Forge
uses this to know which scenario definition to load from the companion JSON.

---

## 8. Wiring Multiple Test Decks

Multiple test decks = multiple `.dck` files (one per deck), each referencing
its own scenario IDs:

```
res/evaluation/
├── aggro_red.dck          DeckURL=… EvalScenario=eval_7plus3_aggro
├── control_blue.dck       DeckURL=… EvalScenario=eval_14_control
└── midrange_golgari.dck   DeckURL=… EvalScenario=eval_7plus3_midrange,eval_14_midrange
```

A single deck can carry multiple eval scenarios (e.g., one `"forced"` for
benchmarking and one `"look_for"` for probability tracking).

---

## 9. How to Test

### 9.1 Validate the JSON spec

Open the decklist JSON and confirm:

- [ ] Every entry in `simulation.eval_scenario_ids` matches a scenario `id`
      in `deck_rules.scenarios`
- [ ] Every referenced scenario has `"type": "eval_sequence"`
- [ ] `opening_hand` has 7 entries (7+3 shape) or 14 entries (flat-14 shape)
- [ ] Group references (`{"group": "..."}`) name a mechanic group key that
      appears in at least one card's `primary_mechanic` or `additional_mechanics`
- [ ] `mode` is `"forced"` or `"look_for"` (or absent, defaulting to `"look_for"`)

### 9.2 Verify the `.dck` file

Open the exported `.dck` and confirm:

```ini
[metadata]
Name=<deck name>
DeckURL=<url>               ← populated from meta.source_url
EvalScenario=<id1>,<id2>   ← comma-separated eval scenario IDs
```

### 9.3 Test forced mode in Forge

1. Load the `.dck` in Forge.
2. In `GameReplaySimulation.java` (or the connector), call:
   ```java
   List<String> ids = GameReplaySimulation.getEvalScenarioIds(deck);
   // for each forced-mode scenario:
   //   resolve group refs → concrete card names
   //   call:
   GameReplaySimulation.applyForcedLibraryOrder(rules, playerName, resolvedOrder);
   ```
3. Run the simulation and observe the draw sequence.
4. **Expected:** cards appear in the exact order defined by `opening_hand` + `turns[].drawn`.
5. **Check the replay JSON:** the exported `deck_link` field should equal the `DeckURL`
   value from `[metadata]` (not `null`).

### 9.4 Test look_for mode in Forge

1. Run a simulation without forced library order.
2. After each turn, the game engine checks whether the actual board state
   satisfies the scenario's `board_state` + `turns` conditions.
3. **Expected:** a match event is logged in the replay at the turn where
   all conditions are first satisfied.
4. Run multiple games (e.g. 100) and count how often the match fires —
   this gives a probability estimate for reaching the scenario state.

### 9.5 Test the frontend export

1. In MaMoFrontend, open a deck that has a `sourceUrl` set.
2. Export the decklist JSON (Scenarios page → Export).
3. Open the downloaded file and verify:
   - `meta.version` is `"1.2.0"`
   - `meta.source_url` equals the deck's source URL
   - Any `eval_sequence` scenario is present with the correct `opening_hand`
     and `turns` arrays
   - Group-ref cards appear as `{"group": "..."}` objects, not plain strings
4. Also verify `simulation.eval_scenario_ids` lists the correct scenario IDs
   (not the deprecated `use_best_starting_hand` boolean).

### 9.6 Test the backend source_url round-trip

```
1. Create a deck via POST /api/addDeck  with { sourceUrl: "https://moxfield.com/..." }
2. Retrieve it via POST /api/getDeck    → response.sourceUrl should match
3. Retrieve via POST /api/getDeckWithCards → response.sourceUrl should match
```

You can run the migration first if the column does not exist:
```bash
node src/migrations/005_add_deck_source_url.js
```

---

## 10. Quick Reference

| Concept | JSON key | `.dck` key | DB column |
|---------|----------|------------|-----------|
| Deck import URL | `meta.source_url` | `DeckURL` | `source_url` |
| Eval scenario IDs | `simulation.eval_scenario_ids` | `EvalScenario` | — |
| Scenario type | `scenarios[].type: "eval_sequence"` | — | — |
| Execution mode | `scenarios[].mode` | — | — |
| Group card ref | `{"group": "ramp"}` | — | — |
| Board snapshot | `scenarios[].board_state` | — | — |
