# Forge Integration Guide

**Applies to:** Commander Decklist Spec v1.2.0+
**Forge version tested:** Forge 1.6.x (open-source MTG AI simulator)

---

## 0. Two Separate Pipelines — Read This First

This guide covers **two distinct handoffs** to Forge. Don't conflate them:

- **§§1–8, "Export JSON" pipeline** — the user-triggered `mtg-commander-decklist` v1.2.0 JSON
  download from MaMo's Playbook page. This is a **reference document**, not something Forge
  reads automatically. As §7 already states, Forge's AI does not "follow" a `perfect_game`/
  `best_starting_hand` turn sequence on its own — a human (or a custom script) applies it via
  Forge's debug/cheat mode.
- **§9, "Live Scenario Viewer" pipeline** — a **separate, already-implemented, machine-driven**
  handoff: `mamoConnector://playtest-scenario/<deckId>?scenarioId=<id>` fetches a purpose-built
  bundle from MaMo's backend, mamo-Connector writes it straight into Forge's own deck/gamelog
  directories, then launches Forge. This is what "Perfect Game" and "Starting Hand" scenarios
  built in Playbook's guided wizard actually use today. **This is the pipeline to implement
  against in Forge** if the goal is "the scenario just plays out when Forge opens" rather than
  "a human sets it up by hand." See §9 below — it does not use the `mtg-commander-decklist`
  JSON shape described in §§1–6 at all; it has its own format.

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

---

## 9. The Live Scenario Viewer Pipeline (`opening_hand_test`, format v1.8.0)

**Status: already implemented and tested on the Forge side — this is not a "please build this"
brief, it's a "here's the contract, here's what to verify" one.**

### 9.1 Where the Forge-side implementation actually lives

Not in this repo, and not in `MaMo-Base` at all — it's in a **separate custom Forge fork**
checked out locally at `C:\SWProjects\Forge` (`github.com/killriam/forge`, branch
`replay-Features`, tracking upstream `Card-Forge/forge`). Key files there:

- `docs/SCENARIO_STARTING_HAND_FORMAT.md` — the **authoritative field reference** for this format
  (in German). Treat it as the source of truth over this section if the two ever disagree —
  this section is a pointer to it, not a fork of it.
- `forge-gui-desktop/.../screens/home/replay/CSubmenuScenario.java` — scans
  `%AppData%\Forge\games\gamelogs\` for `*.json` files (any filename, no naming convention
  required), keeps the ones `ReplayLogParser.isScenario()` accepts, lists them in a **"Replay
  Scenario"** submenu (sibling to Puzzle Mode) for the user to pick and click **Start** —
  this is a manual step in the GUI today, not auto-launched from the deeplink.
- `forge-gui/.../ReplayLogParser.java` — parses the JSON.
- `forge-game/.../log/model/Scenario.java`, `ScenarioLibrarySetup.java`,
  `mulligan/ScenarioKeepMulligan.java` — apply the forced library order / opening hand / AI
  mulligan-skip.
- `docs/example_scenario_forced_sequence.json`, `ScenarioStartingHandTest.java` — worked,
  passing examples.

**§§1–8 of this guide (the `eval_sequence`/`best_starting_hand`/`perfect_game`
`mtg-commander-decklist` export) describe a different, older, manual-setup mechanism and do not
apply here** — see [§0](#0-two-separate-pipelines--read-this-first).

### 9.2 What MaMo already hands off

`new-backend`'s `getForgeScenarioExport` (`GET /api/deck/:deckId/forge-scenario/:scenarioId`,
`CRUDDeckController.ts`) returns `{ deckName, dck, scenarioJson }`. `mamo-Connector`'s
`create_deck_and_scenario_for_forge` (`deck.rs`) writes:

| What | Where | Convention |
|---|---|---|
| `dck` | `%AppData%\Forge\decks\commander\<sanitized deckName>.dck` (Win); `~/.forge/Forge/decks/commander/...` (mac/Linux) | Same directory/writer every deck download uses |
| `scenarioJson` | `%AppData%\Forge\games\gamelogs\Scenario_<sanitized deckName>.json` (Win); `~/.forge/Forge/games/gamelogs/...` (mac/Linux) | Filename is decorative — Forge scans the whole directory for any `.json`, so no collision risk even with the `Scenario_` prefix |

Then launches Forge via `gui --format commander --deck <name> [--deck2 <name2>]` (`forge.rs`) —
no scenario-specific CLI flag; the user opens the Replay Scenario submenu manually once Forge is
up.

### 9.3 The `scenarioJson` shape (matches Forge's parser as of this writing)

```json
{
    "format": "mtg-replay",
    "version": "1.8.0",
    "mode": "scenario",
    "meta": { "game_id": "string", "timestamp": "ISO 8601", "game_type": "commander" },
    "scenario": {
        "type": "opening_hand_test",
        "title": "string",
        "description": "string",
        "question": "string",
        "answer": "string",
        "tags": ["string"],
        "ruling_references": ["string"],
        "player_count": 2,
        "players": {
            "P1": {
                "commanders": ["string"],
                "starting_hand": ["string"],
                "first_draws": ["string"],
                "battlefield": ["string"],
                "starting_life": 40
            },
            "P2": { "starting_hand": [], "first_draws": [], "starting_life": 40 }
        }
    },
    "events": [
        { "i": 1, "t": "T1.MP1:1", "a": "P1", "type": "PLAY_LAND", "data": { "card_name": "string", "targets": [] } }
    ]
}
```

- **`events[].a` is a plain seat id (`"P1"`, `"P2"`, …), matching the `scenario.players` keys —
  not a constructed lobby-name string.** This is a **contract change from earlier drafts of
  this guide** (see §9.4): Forge now resolves the id to whatever runtime name it actually
  assigns that seat at launch time, so `buildEventsFromCards` should emit the same `"P1"`/`"P2"`
  token it already uses to key `scenario.players` — no username/deck-name/date construction
  needed, and no dependency on `.dck` naming conventions at all.
- `scenario.players.PX.starting_hand` / `first_draws` go to the **front of that player's
  library** (`ScenarioLibrarySetup.reorderLibraries()`), so the `.dck`'s `[Main]` card ordering
  MaMo already produces (starting hand, then first draws, then the rest) isn't itself load-bearing
  — Forge reorders by name lookup against these two arrays, not by `.dck` line order. A card
  named here that isn't in the `.dck`'s card pool is skipped with a `WARN` log, not a crash.
- `events[].t` = `"T<turn>.<phase>:<priority>"`. Only `MP1`/`BC`/`MP2` are ever emitted by the
  current TurnWizard UI; `UP`/`DP`/`DC`/`EP` exist in the type system but nothing produces them
  today.
- **Empty turns are the intended way to say "no preference, AI decides."** Forge's own docs call
  this **soft enforcement**: at each priority it checks whether the next queued event's card is
  currently castable — if not, it's left in the queue and normal AI/human play happens for that
  priority instead of forcing or blocking anything. A turn/phase with zero scripted events simply
  has nothing in the queue, which is exactly "AI's own choice." (This directly confirms what we
  already told the user about the "leave it empty = AI decides" behavior.)
- `data.targets` is defined but **Forge's own parser doesn't consume it yet either** ("Phase 2 —
  noch nicht implementiert") — so the fact that `buildEventsFromCards` never populates it from the
  frontend's `selectedAbility`/`choiceText`/`targetSelections` fields is not currently a
  functional gap on our side; it will become one once Forge's target support ships, worth
  revisiting then.

### 9.4 ✅ Resolved (2026-08-10) — actor identity + GUI wiring fixed on the Forge side

An earlier draft of this section flagged two problems after auditing the Forge fork's actual
code (not just its docs). Both are now fixed in `killriam/forge` on `replay-Features`:

1. **The GUI "Replay Scenario" submenu — the pipeline mamo-Connector actually drives per §9.2 —
   never consumed `events` at all.** `CSubmenuScenario.launchScenario()` wired
   `starting_hand`/`first_draws`/`commanders`/`battlefield`/`starting_life` but never called
   anything that read the `events` array or set `GameRules.forcedPlaySequence`. Forced play
   sequence only ran through the CLI `-s` flag (`SimulateMatch.java`), a headless entry point
   MaMo doesn't use in production.
2. **Even where `events` *was* read (the CLI `-s` path), the actor id was used as the lobby
   name with no translation** (`String lobbyName = actor;` — a literal bug, not a doc gap): the
   AI's forced-sequence lookup keys by `player.getLobbyPlayer().getName()`
   (`Ai(N)-<deckName>` for CLI seats, or whatever name the GUI assigns), which never equals a
   raw `"P1"`/`"P2"` token. So even the one path that ran forced sequences at all silently
   never matched.

**The fix, rather than chasing a lobby-name string mamo-Connector would have to predict:**
`events[].a` is now always a plain seat id (`"P1"`, `"P2"`, …) — see the updated §9.3 shape.
Both `CSubmenuScenario` (GUI) and `SimulateMatch` (CLI `-s`) now parse `events` via a shared
`ReplayLogParser.parseForcedSequenceEvents()`, then translate each id to whatever lobby name
that launcher actually assigned the seat this run, before calling
`GameRules.setForcedPlaySequence()` — the same field and the same `AiController` "soft
enforcement" consumption logic already used and trusted by full-game `-r` Replay mode. Details
and worked examples: `docs/SCENARIO_STARTING_HAND_FORMAT.md` (updated alongside this fix) and
`forge-gui-desktop/src/test/java/forge/game/scenario/ScenarioForcedPlaySequenceTest.java` (new
unit coverage for the id→lobby-name translation and event parsing — 8 tests, all passing; no
existing scenario test regressed).

**✅ `new-backend` side applied (2026-08-10, commit `a34a607`):** `buildEventsFromCards` now
hardcodes `a: 'P1'` — Playbook has no concept of scripting another seat's plays, so every
event's actor is always the scenario owner's own `"P1"` seat id, matching the key
`getForgeScenarioExport` already uses for that player's `scenario.players` entry. The
now-dead username/deckDate DB lookup that only existed to build the old constructed string was
removed along with it (one less query per export). The legacy `castOrder` fallback path already
emitted `"P1"` directly and needed no change. New test asserts the actor is always `"P1"`.

**This closes the loop end-to-end**, verified independently on the Forge side: a live
`sim -d ... -scenario ...` CLI run with a scripted `PLAY_LAND` event applied correctly — the
saved replay JSON shows `{"a":"P1","type":"PLAY_LAND","data":{"card_name":"Swamp"}}` on the
correct player's first turn, out of a hand where the AI had no independent reason to prefer
that card. Scripted plays from the ▶ Play in Forge (scenario) flow should now actually apply
instead of silently no-oping.

### 9.5 Practical checklist for testing this pipeline manually

- [ ] `%AppData%\Forge\games\gamelogs\Scenario_<deckName>.json` exists and has
      `"mode": "scenario"` at the top level (Forge's own troubleshooting note: this is the #1
      reason a scenario doesn't show up in the Replay Scenario list at all)
- [ ] Every `starting_hand`/`first_draws` card name matches the `.dck`'s card names exactly
      (case differences are tolerated; the actual string must still resolve)
- [ ] Commander cards are in `players.PX.commanders`, never in `starting_hand`/`first_draws`
- [ ] `events[].a` is a plain `"P1"`/`"P2"` seat id, not a constructed lobby-name string —
      `new-backend` emits this correctly as of commit `a34a607` (see §9.4); only relevant if
      inspecting an export cached/generated before that fix
- [ ] If testing forced play sequence: watch Forge's log for `Scenario: forced play sequence
      set — N event(s) for M player(s)` (GUI) / `Scenario: Loaded forced play sequence — N
      event(s) for M player(s)` (CLI) to confirm the array was even parsed, before assuming a
      "skipped" scripted turn is a data bug — an uncastable next card is expected to stay
      queued (soft enforcement), not an error
- [ ] Forge → **Replay Mode → Scenario Viewer** to pick and start the scenario (this step isn't
      automated by the deeplink today)
