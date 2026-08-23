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
- **§10, "Constructed-Match Scenario Toggle" — a concept/gap document, NOT a third implemented
  pipeline.** Product-side, MaMo presents scenario playback in Forge as two options: (1) §9's
  dedicated single-scenario playtest ("just set up this scenario and let me play on from
  there" — real, described above) and (2) attaching a scenario's forced draw order to a normal
  constructed match, with a further human-plays-it-as-a-hint vs. AI-plays-it-scripted split
  depending on who controls that seat. Only (1) exists today. §10 defines (2) precisely, audits
  what's already reusable from §9 vs. what's a genuine gap in every layer (schema, backend,
  connector, Forge), and specifies the requirements each non-Forge layer would need to meet —
  it does not include a Forge-side implementation, which is out of this document's scope.

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

### 9.6 CAST event `cost`/`x`/`choices` — recorded by Forge's Demo Play, not yet consumed on replay

**Status: recording works and is spec-conformant; replay of a scripted sequence still ignores
these fields (soft-enforcement matches by `card_name` only, same as it always has).**

Forge's "Demo Play" feature (Investigate Scenarios → select a scenario → "Demo Play (record
actions)") applies a scenario's forced draw order to the human seat, lets them actually play the
hand out, and records the playthrough — the intended workflow for *authoring* a scenario's
`events[]` array by playing a good line once instead of hand-writing timestamps and card names.
Until now, the recorder (`ReplayEventLogger`) only captured `card_name` for CAST/ACTIVATE events —
nothing about what a spell targeted, what it cost beyond mana, or what X was chosen. For a spell
like Metamorphosis (`sacrifice a creature; add X mana … where X = the sacrificed creature's
toughness`), the exported snippet had no way to say *which* creature or *what* X — an author had
to reconstruct that by hand from the full recording.

**Root cause, not just a missing feature:** the fix needed wasn't new capture logic — it already
existed and had for a while. `ReplayNotationExporter` (Forge's separate JSON-notation exporter,
attached via `Match.enableReplayNotation`) has had `getAdditionalCosts()`/`getAlternativeCostType()`
methods computing exactly `kicker`/`buyback`/`X=<n>`/etc. from a real `SpellAbility` object since
before this fix. The problem: `GameEventSpellAbilityCast` (the event both this exporter and Demo
Play's recorder subscribe to) only ever carried `SpellAbilityView`/`StackItemView` — lightweight
Trackable views for the UI — never the real `SpellAbility`. Its one call site
(`GameLogFormatter.visit(GameEventSpellAbilityCast)`) always passed `null` for it, so that cost
logic was dead code in practice: correct, tested-in-isolation, and never actually reachable with
real data.

**Fix (`killriam/forge`, `replay-Features`, commit `9e02627a51d`):** `GameEventSpellAbilityCast`
gained a `realSa` field, populated at its single construction site
(`MagicStack.push()`, which already had the real `SpellAbility` right there). Both consumers now
use it — `ReplayNotationExporter`'s cost logic finally runs, and `ReplayEventLogger` builds a CAST
event's `cost`/`x`/`choices` following this spec's own §CAST Event schema (`MTG-REPLAY-NOTATION.md`):

```json
{
  "type": "CAST",
  "data": {
    "card_name": "Metamorphosis",
    "cost": { "mana": "0", "additional": ["X=4"], "alternative": null },
    "x": 4,
    "choices": { "sacrifice": ["c5"] }
  }
}
```

`choices.sacrifice` is a Forge-specific addition (not yet in the core spec's documented `choices`
shape) — cards sacrificed as an additional cost, tracked via a new `GameEventCardSacrificed`
listener, buffered and attached to the *next* CAST/ACTIVATE event within the same phase. This is a
heuristic, not a guaranteed causal link: it can misattribute if an unrelated sacrifice happens in
the same phase before the next cast. Good enough for the common "one action at a time" case Demo
Play is meant for; not something to build automated tooling on top of without re-verifying.

**What's still simplified vs. this spec's full schema:**
- `targets` — a flat array of resolved names, not the spec's `{"slot": "...", "obj": "P2"}` object
  form. No slot/role information is captured, just *what* was targeted.
- `modes` — never populated (Forge doesn't currently expose modal-spell choices through this path).
- The extracted scenario `events[]` snippet (what `DemoPlaySequenceExtractor` actually writes for
  an author to paste back into a scenario file) further flattens all of this into
  `data.targets`/`data.x`/`data.sacrifice` — simpler than the raw recording's nested `cost`/
  `choices`, since scenario files are meant to be hand-edited.
- **None of `targets`/`x`/`choices.sacrifice` are consumed when a forced `events[]` sequence is
  replayed** (`AiController`'s soft-enforcement still matches purely on `card_name` — see §9.4).
  Recording ahead of consumption is deliberate: it makes the data visible to a human author now,
  without requiring the replay-side work (and its own precision questions — e.g. what happens
  when a recorded target is no longer legal) to land first.

Verified: `forge-gui/src/test/java/forge/game/DemoPlaySequenceExtractorTest
.testExtractPlayerEvents_resolvesTargetsXAndSacrificeFromRealLogShape` mirrors this exact log
shape end-to-end (card-id → name resolution via `card_index`, `x` passthrough, `choices.sacrifice`
→ `data.sacrifice`) — added specifically because an earlier draft of the extractor read these
fields from the wrong JSON nesting level and would never have resolved anything from a real
recording; the test caught it before release.

---

## 10. Constructed-Match Scenario Toggle — Concept, Definition & Gap

> **⚠️ This section's "NOT YET IMPLEMENTED" status (below) needs re-verification against
> `killriam/forge`'s current `replay-Features` branch.** Evidence from a later, separate work
> session on the Forge side suggests §10.4.3's hand-off has at least partially landed:
> `PlayerPanel.java` has a per-seat scenario picker (`scenarioPickerComboBox`,
> `populateScenarioComboBox`) reading a deck's `Scenario=` metadata key,
> `docs/SCENARIO_STARTING_HAND_FORMAT.md` documents a "Von einem Deck referenzieren" workflow
> matching §10.1's per-seat/per-controller definition, and `HostedMatch`/`GameLobby` show related
> wiring. This has **not** been independently re-audited end-to-end for this changelog entry the
> way §9.4's fix was — treat the implementation-status claims below as unverified until someone
> does that audit and updates this section properly (or confirms it's still accurate and removes
> this note).

**Status: none of this exists yet. This section defines the concept precisely, audits what §9
already gives you for free, and specifies requirements for the non-Forge layers. Forge-side
implementation is explicitly out of scope here — see §10.4.4, which hands that off rather than
designs it.**

### 10.0 Why this section exists

MaMo's Playbook page frames two distinct ways a scenario ends up in Forge:

1. **"Scenario playout"** — §9's Live Scenario Viewer. Launches a dedicated, single-purpose
   session for exactly this scenario; the user picks it from Forge's own "Replay Scenario"
   submenu once Forge is open. **Real, working, described above.**
2. **"Constructed match with a scenario toggle"** — start a normal constructed match (two real
   decks, either can be a real opponent) and additionally tick "use this deck's Perfect Game /
   Best Starting Hand scenario" so that seat's draws follow the scripted line instead of a
   random shuffle. **Does not exist as a working pipeline anywhere.** The UI that looks like it
   configures this (`ForgeSimulationEditor.tsx`'s "Use best starting hand"/"Use perfect game"
   checkboxes, MaMoFrontend) only edits a config block (`deck_rules.simulation`) that gets
   written into the manually-downloaded `mtg-commander-decklist` JSON (§§1–8) — nothing
   downstream of that download ever reads it back. See
   `MaMoFrontend/specs/gap-forge-constructed-match-scenario-toggle.md` for the UI-side detail.

This section is about (2). It also introduces a distinction that doesn't exist in either
pipeline today: **who is actually driving the scripted seat.**

### 10.1 Definition — what "played" should mean, per seat, per controller

Every scenario turn (`turns[].drawn` / `.played` / `.actions` — see AC-EXP-007/AC-EXP-008 in
MaMoFrontend's `playbook.spec.md` for how these are built from `ScenarioCard.turnAdded` /
`eventTiming`) already answers "what happened this turn" for the scenario's own author. What it
should *cause to happen* in a live Forge match depends on who's sitting in that seat:

| Seat controller | Draw order (`opening_hand` + `turns[].drawn`) | Play sequence (`turns[].played` + `.actions`) |
|---|---|---|
| **Human** | **Forced.** Same mechanism §9 already uses (`ScenarioLibrarySetup.reorderLibraries()`) — the scripted cards are stacked to the top of that seat's library, so the human draws exactly what the scenario says. | **Hint only, never forced.** The human sees (however Forge's UI chooses to surface it — §10.4.4) what the scenario *intended* to be played this turn, but decides and executes every action themselves. Nothing about their input is blocked, auto-played, or overridden. |
| **AI** | **Forced.** Identical mechanism, no seat-specific difference. | **Scripted — the AI performs it.** Reuses the exact `GameRules.setForcedPlaySequence()` + `AiController` soft-enforcement path §9.3/§9.4 already built and verified: at each priority, if the next queued event's card is currently castable, the AI plays it; if not, it's left queued and the AI falls back to its own judgment for that priority (never blocks, never forces an illegal play). An empty turn/phase in `turns[]`/`actions[]` means "no scripted preference — AI decides normally," exactly as §9.3 already documents. |

The right column is the entire concept this section is closing a gap on. The left column (draw
order) is **not** a gap — it's the same mechanism §9 already ships, for either seat, unchanged.

### 10.2 What already exists vs. what's missing

**Already built and reusable, unchanged:**
- Forced library reordering (`ScenarioLibrarySetup`) — seat-agnostic per the Forge integration
  doc pointer in §9.1; works for whichever seat it's told to target.
- Forced play-sequence + soft enforcement (`GameRules.setForcedPlaySequence()` /
  `AiController`) — already verified end-to-end for P1 in a scenario-playout launch (§9.4).
  Architecturally this is exactly the "AI-scripted" behavior column 10.1 wants — the open
  question (§10.4.4) is only whether it's already AI-only by construction (it's driven through
  `AiController`, which a human-controlled seat's own input never goes through) or whether an
  explicit guard needs adding so it's never mistakenly applied to force a human's actions.

**Missing in every layer:**
- **No seat-controller concept anywhere.** Nothing in the schema, UI, backend, or connector
  records or asks "is this seat a human or the AI?" `events[].a` (§9.3) identifies *which seat*
  a scripted action belongs to, never *how* that seat is driven.
- **No opponent-seat scripting at all, for either controller type.** `getForgeScenarioExport`
  hardcodes `players.P2: { starting_hand: [], first_draws: [] }` unconditionally
  (`CRUDDeckController.ts`), and `buildEventsFromCards` hardcodes every event's actor to
  `"P1"` with an explicit code comment that Playbook has no concept of scripting a second seat
  (`scenarioEventBuilder.ts`, quoted in the research this section is based on). This blocks the
  "AI plays out *its own* Perfect Game deck as my opponent" case entirely, not just the
  human-hint half.
- **No cross-deck scenario reference.** Validation Rule 14 (`commander-decklist-spec.md` §9)
  restricts `eval_scenario_ids` to scenarios inside the *same* deck's own `scenarios` array.
  There's no way today to say "attach *this other deck's* Perfect Game scenario to the seat
  playing that other deck."
- **No hint-surfacing mechanism.** Nothing anywhere — schema, Connector, or (as far as this
  workspace can determine) Forge itself — defines how a human player would see "the scenario
  says X was meant to be played this turn" while playing. This is a genuine open UI question,
  not just a missing field.
- **No real launch path for a normal constructed match with a scenario attached at all.** Every
  existing deeplink either launches a plain constructed match with no scenario data
  (`playtest/{deckId}`) or a dedicated single-scenario session that isn't a real match against
  an opponent's own deck (`playtest-scenario/{deckId}`, §9). Nothing launches "constructed match,
  deck A vs. deck B, with an optional scenario attached to either seat."

### 10.3 Proposed data-model shape (schema — proposed, not a version bump yet)

Extend the `scenarioJson` shape (§9.3) so each populated player entry can declare who's driving
it. This section proposes the shape; it is **not** a ratified schema change — no version bump
has been made, and this shouldn't be treated as stable until implemented and verified the same
way §9 was.

```json
{
    "scenario": {
        "players": {
            "P1": {
                "controlled_by": "human",
                "commanders": ["..."],
                "starting_hand": ["..."],
                "first_draws": ["..."]
            },
            "P2": {
                "controlled_by": "ai",
                "commanders": ["..."],
                "starting_hand": ["..."],
                "first_draws": ["..."]
            }
        }
    },
    "events": [
        { "i": 1, "t": "T1.MP1:1", "a": "P2", "type": "PLAY_LAND", "data": { "card_name": "Forest" } }
    ]
}
```

- `controlled_by` is deliberately **not** a per-event field — it's per-player, since a seat's
  controller doesn't change mid-game. `events[].a` continues to just mean "which seat," as it
  does today; the consuming side looks up that seat's `controlled_by` to decide whether the
  matching event is a hint or a forced action.
- **Open question, not resolved here:** should `controlled_by` live in the JSON at all, or
  should the launcher (mamo-Connector) simply already know it — since it's whoever launched the
  match choosing "I'll play this seat, AI plays that one" in Forge's own New Game dialog — and
  pass it as launch context instead of baking it into the scenario file? Embedding it in the
  JSON is simpler to reason about (self-contained, replay-able, testable in isolation); passing
  it as launch context avoids ever having a JSON file whose `controlled_by` claim disagrees with
  who Forge actually assigned to that seat when the match starts. Whoever implements this needs
  to pick one — not something to guess at from outside Forge's own New Game flow.

### 10.4 Requirements for each layer

None of the following is implemented by this change. Each is a requirement the corresponding
repo/team would need to satisfy; this document does not implement any of them.

#### 10.4.1 new-backend

- A new or extended export endpoint that, unlike `getForgeScenarioExport` (single deck, always
  scenario-owner-as-P1, P2 always empty), accepts **two** deck+scenario pairs — one per seat —
  and returns both `players.P1`/`players.P2` populated when a scenario is attached to that seat.
- A relaxed, explicitly-scoped version of Validation Rule 14 for this use case only: a scenario
  ID may reference a scenario belonging to a *different* deck when that deck is the one assigned
  to that seat in this launch. The existing same-deck-only rule should stay as-is for
  `eval_scenario_ids` inside a single deck's own simulation config — this is a new, additional
  allowance, not a loosening of the existing rule.

#### 10.4.2 mamo-Connector

- A new or extended deeplink for "real constructed match, optionally with a scenario attached
  per seat" — distinct from `playtest-scenario` (§9, always a dedicated single-scenario session)
  and from `resolve_opponent_deck_path`'s existing tiered opponent-resolution (explicit deck2 →
  archenemy → curated pool → none), which was built for scenario-playout, not for a symmetric
  two-real-decks match.
- Must be able to write **both** seats' populated scenario bundles to Forge's gamelog directory
  when both sides have a scenario attached (today's writer, `create_deck_and_scenario_for_forge`
  in `deck.rs`, only ever populates one).
- Needs to resolve `controlled_by` per §10.3's open question before it can decide what to write
  or how to launch Forge — this is a prerequisite, not something the Connector can route around.

#### 10.4.3 Forge (spec-only — hand-off to the Forge team, no implementation here)

- **Human seat:** apply forced library order only (`ScenarioLibrarySetup`, already built and
  reusable as-is). Confirm — don't assume — whether `setForcedPlaySequence`/`AiController` soft
  enforcement already only ever engages for AI-driven decision-making (in which case a
  human-controlled seat is naturally unaffected by forced play-sequence data with zero new
  guard code) or whether an explicit "skip forced-play-sequence application for a human seat"
  check needs adding. This is the single most load-bearing open question in this whole section —
  everything else in the human-hint column of §10.1 depends on the answer.
- **A hint surface for the human seat.** Needs a Forge-side UI decision (sidebar panel, tooltip
  on the current turn, log line — any of these would satisfy the requirement). The only
  requirement from this side of the integration is that the human seat's current-turn
  `drawn`/`played`/`actions` data reaches wherever Forge decides to display it; the visual design
  is entirely Forge's call.
- **AI seat:** reuse `setForcedPlaySequence` + soft enforcement exactly as already built and
  verified for P1 in §9.3/§9.4 — no new Forge mechanism needed here, only extending which seat
  it's allowed to key off (today implicitly whichever seat scenario-playout always assigns as
  P1; this needs to work for whichever seat `controlled_by: "ai"` actually lands on).

#### 10.4.4 MaMoFrontend

- Product decision needed on `ForgeSimulationEditor.tsx` (not made here — flagged for the
  product owner): either (a) rewire its "Use best starting hand"/"Use perfect game" checkboxes
  to trigger the new deeplink once 10.4.1–10.4.3 exist, or (b) keep it as a reference/manual-
  setup config editor only, and relabel the UI so it stops reading as a live toggle it isn't.
  See the MaMoFrontend gap file linked in §10.0 for the current-state detail this decision is
  needed against.
- A picker letting the user attach the **opponent's own** saved scenario (not just their own
  deck's), once 10.4.1's cross-deck export support exists.
- Some UI to set `controlled_by` per seat — or, if §10.3's open question resolves toward "launch
  context, not JSON," this may not need a MaMoFrontend-side field at all and could instead be
  implied by which deck the user picked as "my deck" vs. "opponent."

### 10.5 What you already get today without any of this

§9's existing pipeline already delivers most of the human-hint *feel* — forced draw order plus
non-blocking soft-enforced play-sequence hints — for the scenario owner's own seat, in a
dedicated single-scenario session. What it doesn't give you: a real match against an opponent's
own deck, an opponent who plays out *their own* scripted line, or any way to attach a scenario
to a normal constructed-match launch instead of the dedicated Scenario Viewer flow. Closing
those gaps is everything in §10.2–10.4 above.

---

## 11. Desktop Release Artifact Naming

**Status: ✅ Option A shipped (2026-08-17)** — `killriam/forge` `replay-Features` commit
`d2bc12ba3dc`. `forge-gui-desktop`'s release jar filename now carries a build-timestamp suffix
(`${snapshot-version}`, e.g. `forge-gui-desktop-2.0.14-SNAPSHOT-08.17-0700-jar-with-dependencies
.jar`) instead of the static name it had before. **See `MEMO-forge-jar-naming.md` at this repo's
root for the short, action-oriented version** (what mamo-Connector needs to do: glob instead of
hardcoding the filename). The rest of this section is kept as the original proposal/rationale for
reference — options B/C/D below were considered and not chosen.

### 11.0 The problem

Forge's desktop release jar is currently named with a **static** version string that never
changes between releases: `forge-gui-desktop-2.0.14-SNAPSHOT-jar-with-dependencies.jar`. It stays
exactly this name across every build until `versionCode` in the root `pom.xml` is bumped (rare —
it's been `2.0.14` for months of releases). This is a real, already-hit problem: a stale local
copy is indistinguishable by filename from a fresh one, which directly caused a "MamoConnector
launched Forge but it was running an old build" confusion during development of the features in
this guide's §9.

By contrast, Forge's **in-app version banner** already shows something more specific —
`2.0.14-SNAPSHOT-08.16-1837` — because `forge-gui-desktop/pom.xml`'s `snapshot-version` property
(regex-derived from `revision`, suffixing `-SNAPSHOT` with a `MM.dd-HHmm` build timestamp) is used
for the jar's `Implementation-Version` manifest attribute. It's just never applied to the jar's
own **filename**. (The Android and installer builds *do* apply this same property to their output
filenames already — desktop is the odd one out.)

The GitHub Release itself is already stable and doesn't need to change: it's always published
under the `replay-features-latest` tag (force-moved on each push), so
`.../releases/download/replay-features-latest/MaMoForge-portable.zip` as a download URL is
unaffected by anything below. This proposal only changes what the jar *inside* that zip is named.

### 11.1 Naming options

| # | Example filename | Source data | Notes |
|---|---|---|---|
| A | `forge-gui-desktop-2.0.14-SNAPSHOT-08.16-1837-jar-with-dependencies.jar` | `${snapshot-version}` (already computed, already shown in-app) | Smallest Forge-side change — one `<finalName>` line. Filename always matches what the running app reports as its own version. |
| B | `forge-gui-desktop-2.0.14-SNAPSHOT-g618f7dc-jar-with-dependencies.jar` | short git commit hash (already computed at build time, e.g. the `Git short hash: ...` build log line) | Unambiguous exact-commit traceability; no timestamp-collision edge case across parallel CI runs. Less immediately human-readable ("when was this built?") without a lookup. |
| C | `forge-gui-desktop-2.0.14-SNAPSHOT-08.16-1837-g618f7dc-jar-with-dependencies.jar` | both of the above | Most information, longest name. Recommended if mamo-Connector wants to log/display build provenance, not just detect staleness. |
| D | *(jar filename unchanged)* + a small `VERSION.txt`/`latest.json` release asset with version/hash/timestamp | new file, no jar rename | Zero change to the `java -jar ...` invocation path; mamo-Connector reads the sidecar file instead. Doesn't fix the "the extracted folder always looks identical" confusion by itself, and still requires a mamo-Connector-side change (reading a new file) — so it doesn't avoid coordination, it just relocates it. |

**Recommendation: Option A**, for two reasons — it's already computed and already trusted (it's
the exact string the app prints about itself on startup, so "jar filename" and "app's own version
report" can never disagree), and it requires the smallest Forge-side diff. Option C is a
reasonable upgrade if commit-level traceability matters more than filename brevity; A and C are
not mutually exclusive to switch between later since both derive from data Forge already computes.

### 11.2 What changes for mamo-Connector regardless of which option is picked

Whichever option ships, **the jar filename becomes variable between releases** — that's the
entire point. Any code that currently assumes/hardcodes the literal string
`forge-gui-desktop-2.0.14-SNAPSHOT-jar-with-dependencies.jar` needs to switch to locating it by
pattern after extracting `MaMoForge-portable.zip`, e.g. (pseudocode, adjust to whatever language
mamo-Connector is in):

```
jar = glob("forge-gui-desktop-*-jar-with-dependencies.jar", inside=extracted_dir)[0]
run: java -jar {jar} ...
```

This is the same approach Forge's own `build-release.yml` already uses internally
(`find forge-gui-desktop/target -name "forge-gui-desktop-*-jar-with-dependencies.jar"`) — it
doesn't hardcode the exact name either, for exactly this reason.

If mamo-Connector currently determines "is my local Forge install stale?" by anything other than
comparing the jar's actual last-modified time or content hash, this is also the natural point to
switch that check to parsing the new filename (or the manifest's `Implementation-Version`, which
already carries the same information and works today, before this proposal - see
`BuildInfo.getVersionString()`/`BuildInfo.getGitCommit()` in `forge-core`, callable at runtime
without needing to touch the filename at all).

---

## 12. Forge AI Team Handover: Play Guidance & Policy Integration

**Target Audience:** Forge Java Core / AI Development Team  
**Scope:** Adding declarative strategic policy overlay (`ai_guidance`) to Forge AI without breaking existing heuristic baselines.

---

### 12.1 Overview & Design Philosophy

MaMo provides an authored / derived strategic policy (`deck.policy.json` or embedded `ai_guidance` in scenario JSON) to guide Forge AI in Commander matches. Rather than replacing Forge's existing rules engine or combat simulator (`ComputerUtilCombat`), the policy acts as a **declarative scoring and veto filter** at four specific priority interception points.

---

### 12.2 Handover Deliverables & Required Java Patches

#### 1. New Package: `forge.ai.guidance`

Create the following two classes in `forge-ai/src/main/java/forge/ai/guidance/`:

##### `AiGuidanceProfile.java`
Parses the incoming JSON policy and provides fast lookups:
- `getRoleBinding(String cardName)` $\to$ `CardRoleBinding` (returns `tactical_role`, `deployment_guard`, `phase_timing`)
- `findTargetRule(Card hostCard)` $\to$ `TargetRankingRule` (returns priority ladder steps and hard vetoes)
- `getPreferenceBonus(SpellAbility sa)` $\to$ integer score modifier

##### `PredicateEvaluator.java`
Zero-dependency recursive AST evaluator ($O(1)$) supporting:
- Logical combinators: `all_of`, `any_of`, `none_of`
- Leaf comparisons: `==`, `!=`, `>`, `>=`, `<`, `<=`
- Dynamic game-state fields: `opponent_open_mana`, `opponent_mana_colors`, `turn_number`, `target.has_indestructible`, `target.canonical_threat_tier`

```java
package forge.ai.guidance;

import forge.game.Game;
import forge.game.player.Player;
import forge.game.card.Card;
import com.google.gson.JsonObject;
import com.google.gson.JsonArray;
import com.google.gson.JsonElement;

public class PredicateEvaluator {
    public static boolean evaluate(JsonObject ast, Player aiPlayer, Game game, Card targetCard) {
        if (ast == null || ast.isJsonNull()) return true;
        if (ast.has("all_of")) {
            for (JsonElement el : ast.getAsJsonArray("all_of")) {
                if (!evaluate(el.getAsJsonObject(), aiPlayer, game, targetCard)) return false;
            }
            return true;
        }
        if (ast.has("any_of")) {
            for (JsonElement el : ast.getAsJsonArray("any_of")) {
                if (evaluate(el.getAsJsonObject(), aiPlayer, game, targetCard)) return true;
            }
            return false;
        }
        if (ast.has("none_of")) {
            for (JsonElement el : ast.getAsJsonArray("none_of")) {
                if (evaluate(el.getAsJsonObject(), aiPlayer, game, targetCard)) return false;
            }
            return true;
        }
        
        if (!ast.has("field") || !ast.has("op")) return true;
        String field = ast.get("field").getAsString();
        String op = ast.get("op").getAsString();
        JsonElement val = ast.get("value");
        return evaluateLeaf(field, op, val, aiPlayer, game, targetCard);
    }

    private static boolean evaluateLeaf(String field, String op, JsonElement val, Player aiPlayer, Game game, Card target) {
        switch (field) {
            case "opponent_open_mana":
                int maxUntapped = 0;
                for (Player opp : aiPlayer.getOpponents()) {
                    int lands = opp.getCardsIn(forge.game.zone.ZoneType.Battlefield).filter(c -> c.isLand() && c.isUntapped()).size();
                    if (lands > maxUntapped) maxUntapped = lands;
                }
                return compareInt(maxUntapped, op, val.getAsInt());
            case "target.has_indestructible":
                return target != null && (target.hasKeyword("Indestructible") || target.hasKeyword("Hexproof"));
            case "target.canonical_threat_tier":
                return target != null && target.hasKeyword(val.getAsString());
            default:
                return true;
        }
    }
    
    private static boolean compareInt(int a, String op, int b) {
        switch (op) {
            case "==": return a == b;
            case "!=": return a != b;
            case ">":  return a > b;
            case ">=": return a >= b;
            case "<":  return a < b;
            case "<=": return a <= b;
            default: return false;
        }
    }
}
```

---

#### 2. Interception Points in Existing Forge Code (Grounded)

| File to Modify | Target Method | Insertion Logic |
| :--- | :--- | :--- |
| **`forge.ai.AiController`** | `chooseSpellAbilityToPlay()` | Evaluate `CardRoleBinding.getDeploymentGuard()` before sorting candidate abilities. If false, filter out candidate (prevents naked Multiplier drops). |
| **`forge.ai.ComputerUtilCard`** | `evaluateCreature()` / `evaluatePermanent()` | Centralized threat scoring: (1) apply `target_rankings` hard vetoes to return -9999 for indestructible/invalid targets, (2) apply `target_rankings` tier score bonus (`+100` combo, `+70` engine hub). Handled automatically across all 100+ spell ability handlers. |
| **`forge.deck.DeckRulesLoader`** | `loadDeckRules()` | Ingest `deck_rules.ai_guidance` JSON block directly using Gson into `AiGuidanceProfile`. |
| **`forge.ai.logging.AiDecisionLogger`** | `logDecision()` | Structured JSON logging hook to export candidate options, evaluated guidance rules, and delta scores for Replay Coach. |

---

### 12.3 Inbound Payload Format

When a scenario is launched from MaMo, Forge will find the policy bundled inside `scenario.json` under `ai_guidance` or as a companion `<deck_name>.policy.json`:

```json
{
  "deck_id": "Ghave_Guru_of_Spores",
  "tactical_roles": {
    "Doubling Season": {
      "role": "multiplier",
      "deployment_guard": {
        "all_of": [
          { "field": "active_engine_core_count", "op": ">=", "value": 1 }
        ]
      }
    }
  },
  "target_rankings": [
    {
      "source_card": "Swords to Plowshares",
      "vetoes": [
        { "condition": { "field": "target.has_indestructible", "op": "==", "value": true }, "reason": "indestructible" }
      ],
      "ladder": [
        { "condition": { "field": "target.canonical_threat_tier", "op": "==", "value": "Tier1_Combo" }, "score": 100 },
        { "condition": { "field": "target.canonical_threat_tier", "op": "==", "value": "Tier2_EngineHub" }, "score": 70 }
      ]
    }
  ]
}
```

---

### 12.4 Backward Compatibility Guarantee

If no `ai_guidance` profile is present (or `guidanceProfile == null`), all four hooks **no-op immediately**, preserving Forge's default vanilla AI behavior with zero performance regression.

---

### 12.5 Reality Check: Contradictions With Current Forge AI Architecture

**Status: audit only — nothing in this section has been implemented or changed on the Forge side.
No Forge source file was modified to produce it.** Checked against `killriam/forge`,
`replay-Features`, 2026-08-23, by reading the actual classes named in §12.2 and in
`ai-play-guidance-spec.md` §11.4, not just their docs. Where §12.1–12.4 above (and the source spec's
§11) name a specific method, class, or config property as an integration point, this section
records whether that thing exists as described. **The instruction driving this audit: do not change
Forge's core AI principles to make the spec fit — where they don't fit cleanly, write the
contradiction down instead.** Forge's own core principles (`docs/AI_DECISION_MAKING_CONCEPT.md` §1)
are: rules engine/decision agent separation via `PlayerController`, a phase-driven priority loop, and
**decentralized per-ability heuristics** (~100 independent `SpellAbilityAi` handlers) rather than one
central scorer. Several of §12's hooks assume the opposite of that third principle.

#### 12.5.1 Named hook methods that don't exist

| §12.2 / §11.4 claims | Reality (file:line) | Contradiction |
| :--- | :--- | :--- |
| `forge.ai.AiController.getSpellAbilitiesToPlay()` | No method with this name exists anywhere in `forge-ai`. The real entry point is `AiController.chooseSpellAbilityToPlay()` (`AiController.java:1432`), which returns `List<SpellAbility>` and internally delegates to `chooseSpellAbilityToPlayFromList()` (`:1762`). | The proposed "check deployment guard, skip adding to candidates" pseudocode has nowhere to attach — there is no exposed per-candidate accumulation loop at that name to intercept. |
| `forge.ai.SpellAbilityAi.chooseTargetsAI()` | No method with this name exists on `SpellAbilityAi` or any of its ~100 subclasses (checked `DestroyAi.java` as a representative handler). Target selection is spread across heterogeneous overrides — `chooseSingleCard()`, `chooseSingleEntity()`, `chooseSinglePlayer()`, or inline inside `checkApiLogic()`/`doTriggerNoCost()`/handler-specific methods like `DestroyAi.doLandForLandRemovalLogic()` — that differ per handler. | §12.2's target-veto/scoring hook and §11.4 #3's `chooseTargetsAI()` code sample have no single call site. Reaching parity would mean editing on the order of 100 separate handler classes individually, which directly conflicts with §12.1's own promise ("without breaking existing heuristic baselines") — that's not an overlay, it's a rewrite of the decentralized-handler principle itself. |
| `SpellAbilityAi.canPlayAI()` (named in both `AI_DECISION_MAKING_CONCEPT.md`'s own class-summary and mirrored in the guidance spec's §11.1 table) | `SpellAbilityAi.java:37`'s class javadoc claims this name, but the actual methods are `canPlay()` (protected, `:70`) and `canPlayWithSubs()` (public, `:54`). | This mismatch predates the guidance spec — it's already stale inside Forge's own concept doc — and the guidance spec inherited the wrong name from it rather than the source. |

#### 12.5.2 No additive "Final Score" — real ranking is a `Comparator` + an enum-tagged rating record

Forge already has a scoring/decision type, and it isn't the one §5.1's
`Base Threat + Σ(condition weights) + Tier Bonus + Tempo Bonus` formula assumes:

- Candidate spells are ranked by `all.sort(ComputerUtilAbility.saEvaluator)` — a `Comparator<SpellAbility>` — inside `chooseSpellAbilityToPlayFromList()` (`AiController.java:1767`), not by comparing an accumulated numeric total across all candidates.
- Per-ability playability is `AiAbilityDecision` (`AiAbilityDecision.java`), a **record** of `(int rating, AiPlayDecision decision, SpellAbility sa)`. `willingToPlay()` already has its own situational rating-boost logic baked in (turn/phase-based `boosted` adjustments, `MIN_RATING = 30` threshold) — it is not a blank slate waiting for an external additive term.

Bolting `PreferenceBonus(a)` on top means either (a) running a second, disconnected scoring pass that the `Comparator`/`AiAbilityDecision` machinery never sees — so a guidance bonus can never actually change what gets played, defeating the point — or (b) changing what `AiAbilityDecision.rating` means and how `saEvaluator` compares candidates, which **is** a core-principle change (altering the vanilla heuristic baseline §12.1 promises to leave alone), not an overlay.

#### 12.5.3 `evaluation_profile` stage weights have no runtime property to map onto

§7 (and Stage 4, §13.3) describe mapping `evaluation_profile.stages[stage].weights` onto Forge `.ai`
profile properties `AggressionLevel`, `TradeThreshold`, `LifeDangerThreshold`, `MulliganModel`,
`SimulationDepth` — a table `AI_DECISION_MAKING_CONCEPT.md` §6 (Forge's own doc) also states nearly
verbatim. Checked against the real property list (`forge-ai/src/main/java/forge/ai/AiProps.java`,
82 enum constants) and the shipped profile files (`forge-gui/res/ai/*.ai`): **none of those five
names exist.** The closest real properties are things like `MULLIGAN_THRESHOLD`, `PLAY_AGGRO`,
`CHANCE_TO_ATTACK_INTO_TRADE`, `TOKEN_GENERATION_ABILITY_CHANCE` — dozens of narrow, situational
knobs, not five clean stage-weighted dials.

Worse than a naming gap: "aggression" in the real code (`AiAttackController.aiAggression`,
`AiAttackController.java:80`, computed at `:1229–1269`) is a **per-combat int (0–6), recomputed fresh
every combat** from board state (life totals, force comparison, a chance roll) — it is not a
persisted profile-level setting an external system could overwrite once per game stage. Calling
Stage 4 "Low Effort" (§13.3) is only true if this property already existed to map onto; it doesn't,
so "map weights onto AiProfile variables" would mean inventing a new parallel config surface *and*
rewiring `AiAttackController`'s per-combat computation to defer to it — a change to how combat
aggression itself is decided, which is core combat-math behavior, not an overlay on top of it.

#### 12.5.4 `AiDecisionLogger` already exists — but as a plain-text game-log writer, not an L2/JSON emitter

§12.2's 4th hook and the source spec's Stage 5 describe extending `logDecision()` to "include
candidate options, evaluated guidance rules, and delta scores in Level 2 replay log output," framed
as a light addition to an existing method. The real class
(`forge-ai/src/main/java/forge/ai/AiDecisionLogger.java`) does exist, but:

- `logDecision(Player ai, SpellAbility sa, AiPlayDecision decision)` takes an **enum** reason (`WillPlay`, `Removal`, `Tempo`, …), not a numeric score or a guidance rule id.
- It writes a human-readable string via `game.getGameLog().add(GameLogEntryType.AI_DECISION, ...)` — there is no `views_l2` concept, no ΔV breakdown, and no connection to `ReplayNotationExporter` (the actual JSON replay-notation exporter, a separate class). §9.6 of this same guide already documents, from direct experience, that reaching `ReplayNotationExporter` from a game event required a dedicated fix (`GameEventSpellAbilityCast` gaining a `realSa` field) because the event this logger's own trigger point subscribes to didn't carry enough data — the same gap applies here.

Wiring guidance rule IDs and score deltas into the actual replay JSON (not just the human-readable
game log) is a materially bigger change than "include ... in Level 2 replay log output" implies, and
routes through `ReplayNotationExporter`, which neither §12 nor the source spec's §11.4 mentions.

#### 12.5.5 An existing, closer-fitting loader precedent goes unmentioned

§11.3 of the source spec says only that "the `mamo-Connector` places the guidance configuration in
Forge's deck/profile directory or launches Forge with the guidance payload" — vague about the actual
mechanism. Forge already ships a working analog for exactly this shape of problem:
`forge-ai/src/main/java/forge/ai/DeckRulesLoader.java` + `DeckRulesConfig` — a Gson-based loader
(Gson is already a `forge-ai` dependency, confirmed in `forge-ai/pom.xml`; that part of the spec's
assumption is correct) that reads a **`deck_rules` JSON block** — `mulligan`, `combos`, `dont_combos`
— off a path recorded on the `Deck` object (`DecklistSpecPath`), attaches a parsed `DeckRulesConfig`
to it, and is consulted at decision time. This is the same `deck_rules` envelope §1/§4 of this guide
already documents for the `mtg-commander-decklist` JSON's `simulation` block. An `ai_guidance` key
sitting next to `mulligan`/`combos` in that same envelope, parsed by extending this loader, is a
smaller and more consistent change than the "deck/profile directory or launch payload" language
suggests — worth raising with whoever picks up the handover, not a blocking contradiction.

#### 12.5.6 Forced play sequence (real, existing) and "tactical sequences" (proposed) are being conflated

`ai-play-guidance-spec.md`'s own §11.1 table lists "Forced Play Sequences... Scenario script engine"
as powering the proposed `tactical_sequences`/`bait_countermagic` mechanism (§6.2's `abort_if`
stage-2 guard). §9.3/§9.4 of **this** guide already document the real mechanism accurately: it's a
pure **exact-card-name short-circuit** — `chooseSpellAbilityToPlay()`'s forced-sequence block
(`AiController.java:1443–1535`) checks the head of a lobby-name-keyed queue
(`GameRules.getForcedPlaySequence()`), and once a matching castable card is found, returns it
immediately, bypassing `canPlayAI`/target-ranking/evaluation entirely for that decision. There is no
mid-sequence condition check in that code path at all — it either plays the next queued name or
leaves it queued (soft enforcement) until it's playable or the turn ends. It cannot express "cast the
enabler now, then **abort** stage 2 if the opponent now has ≥4 untapped blue mana" — that's a
different capability (conditional, abortable, re-evaluated every priority) than "next name in a fixed
list." Reusing the forced-sequence machinery for `tactical_sequences` as the source spec's table
implies would mean adding conditional-abort logic on top of a mechanism whose own design doc
(`plan-deckRulesAiIntegration.prompt.md`, root of this Forge fork — the actual plan this code was
built from) deliberately scoped it to soft, retry-or-fall-through enforcement only, and left target
forcing as an explicit, unbuilt "Phase 2." Treating `tactical_sequences` as a genuinely separate, new
mechanism rather than an extension of forced-sequence is the safer reading, and should be called out
as such wherever this handover is picked up.

#### 12.5.7 What's left standing

Not everything contradicts. Kept for balance: the `PredicateEvaluator` AST shape itself (`all_of` /
`any_of` / `none_of` / leaf `field`/`op`/`value`) is implementable as ordinary Java with no core
changes — it's a pure function of `(Player, Game, Card)`, and nothing about it requires touching
`AiController`/`SpellAbilityAi` internals to exist as a standalone utility class. Gson as the JSON
parser is already a real `forge-ai` dependency. `ComputerUtilCard.evaluateCreature()` (real method,
`ComputerUtilCard.java:758`) is a real, single, poke-able baseline-value function a guidance modifier
could plausibly wrap. The gap is specifically in *wiring* that evaluator into the four named
interception points — none of the four are the low-risk overlay §12.1 describes; each either doesn't
exist under that name, doesn't carry the data needed, or would require changing the exact heuristic
baseline behavior this section promises to leave alone.

---

### 12.6 Slice 1 Shipped: Declarative Role Deployment Guards

**Status: implemented, tested, and passing on `killriam/forge` `replay-Features` as of 2026-08-23 —
13/13 new tests green (`mvn test`), 45/45 in a broader regression pass across existing
`forge.ai`/`forge.game.scenario`/`forge.ai.ability.*Test` suites, zero failures.** This section
covers three things: (1) a second grounding pass on §12.2/§11.1's revised "Grounded" hook table —
that table was edited after §12.5 above was written, presumably in response to it, but wasn't
independently re-verified against the source before landing here; (2) what got built, where, and
why, given that pass; (3) the V&V approach actually used, CLI/`mvn test`-first throughout, and what
is deliberately left for a human.

#### 12.6.1 Second grounding pass — checking the "Grounded" fixes against source

§12.2's interception-point table and §11.1/§11.4 of the source spec were rewritten to answer §12.5.
Re-checked against the real classes before building on top of them:

| Revised claim | Checked against source | Verdict |
| :--- | :--- | :--- |
| Hook entry point is `AiController.chooseSpellAbilityToPlay()` | That's the *public* entry point, but the actual per-candidate loop — where a veto can `continue` past one candidate without touching the others — is inside the *private* `chooseSpellAbilityToPlayFromList()` it delegates to (`AiController.java:1776`, called from `getSpellAbilityToPlay()` at `:1767`). | **Mostly right, one level too shallow.** Built against the real loop, not the public method's name. |
| Target hook is `ComputerUtilCard.evaluateCreature()` **/ `evaluatePermanent()`** | `evaluateCreature()` is real (`ComputerUtilCard.java:758`). `evaluatePermanent()` **does not exist.** `ComputerUtilCard.getBestAI()` (`:555`) has its own `// TODO - Once we get an EvaluatePermanent this should call getBestPermanent()` comment — Forge's own maintainers flagged this as a gap that doesn't exist yet, not something already there to decorate. | **Half-fabricated.** A `target_rankings` hook covering only creatures is real and buildable; covering "any permanent" needs new baseline Forge functionality first, which is out of scope for a guidance overlay to add unilaterally. Deferred — see §12.6.3. |
| `forge.deck.DeckRulesLoader` / `loadDeckRules()` | The real class is `forge.ai.DeckRulesLoader` (package `forge.ai`, confirmed via its own `package` declaration) — not `forge.deck`. It has no `loadDeckRules()` method; the real methods are `loadIfNeeded(Deck, File)` and `loadFromFile(File)`. | **Package and method name both wrong.** Extended the real class instead (see §12.6.2). |
| Forced sequences: wrap with `PredicateEvaluator.evaluate(abort_if)` "before honoring sequence" | `GameRules.forcedPlaySequence` is `Map<String, List<String>>` — a flat list of card-name strings (`GameRules.java:326`), confirmed unchanged. There is no per-step field to attach an `abort_if` predicate to. `forge-game` also has no Gson dependency (confirmed: no `gson` entry in `forge-core/pom.xml`, and `DeckRulesConfig`'s own javadoc states it "lives in forge-core so that `Deck` can hold a reference" specifically *because* JSON parsing needs to stay out of forge-core/forge-game). | **Not a wrap — a schema change.** `tactical_sequences`/`abort_if` needs a new data shape in `GameRules` (or a fully separate mechanism) before any `PredicateEvaluator` call can attach to it. Not attempted in this slice — see §12.6.3. |
| Decision logging: "structured JSON exporter added as an extension to `AiDecisionLogger`" | Still accurate as of §12.5.4 — `AiDecisionLogger` writes enum-tagged strings to the game log, not JSON, and isn't wired to `ReplayNotationExporter`. The revision doesn't add new information here, just restates the gap. | **Unchanged, still open.** Not attempted in this slice — see §12.6.3. |

One thing the "Grounded" pass got right that's worth confirming rather than just trusting: Gson
*is* already a `forge-ai` dependency (`forge-ai/pom.xml`), so no new third-party dependency was
needed for any of this.

#### 12.6.2 What actually shipped

Scope: **`role_bindings` deployment guards only** — the one interception point that survived
grounding without needing new baseline Forge functionality or a `GameRules` schema change. New
files, all in `forge-ai/src/main/java/forge/ai/guidance/` (package `forge.ai.guidance`):

| File | Role |
| :--- | :--- |
| `PredicateEvaluator.java` | AST evaluator (`all_of`/`any_of`/`none_of`, leaf `field`/`op`/`value`). Takes an `AiGuidanceProfile` parameter the spec's own reference pseudocode doesn't — `active_engine_core_count`/`battlefield.roles`/`hand.roles` need it to resolve a card name to its declared role, which a bare `(Player, Game, Card)` function can't do. Also implements the set operators (`contains`, `contains_any`, `contains_all`, `excludes_all`, `lacks`) that ai-play-guidance-spec.md §10.2's own TypeScript `PredicateOperator` union declares and §4.3's own worked examples use, but that neither spec document's Java reference implementation actually has a case for. |
| `AiGuidanceProfile.java` | Parses `role_bindings.cards` (→ `CardRoleBinding`, `primary_role` only — ability-level granularity from spec §4.2 not implemented) and `role_bindings.deployment_constraints[]` (`applies_to_role`/`condition`, matching ai-play-guidance-spec.md §4.3's shape — **not** forge-integration-guide.md §12.3's alternate per-card `tactical_roles.<card>.deployment_guard` shape; the two spec documents disagree on this and only one was implemented, deliberately, because it generalizes). Exposes `passesDeploymentGuard(Card, Player, Game)`. |
| `CardRoleBinding.java` | Small model, `primary_role` only. |

Extended, not replaced: `forge-ai/src/main/java/forge/ai/DeckRulesLoader.java` gained
`loadAiGuidanceIfNeeded(Deck, File)` — parses `deck_rules.ai_guidance` from the **same** JSON file
`loadIfNeeded()` already resolves via `Deck.getDecklistSpecPath()`, into an `AiGuidanceProfile`.
Deliberately **not** attached to `Deck`/`DeckRulesConfig` (forge-core) the way `DeckRulesConfig`
itself is — `AiGuidanceProfile` is Gson-shaped and forge-core must stay Gson-free (§12.6.1's
`abort_if` row above explains why). Instead it follows the *exact* existing precedent for exactly
this problem: `AiController` already holds a forge-ai-only `ComboTracker` object built from
`DeckRulesConfig`, populated once at game setup via `AiController.initComboTracker(Deck)`
(`AiController.java:176`), called from `PlayerControllerAi.complainCardsCantPlayWell()`
(`PlayerControllerAi.java:1410`). `initGuidanceProfile(Deck)` was added right next to it, called
from the same site, storing the result in a new `AiController.guidanceProfile` field — no change to
`Deck`, `DeckRulesConfig`, or any forge-core class at all.

The hook itself: one `continue`-guarded block inserted into `AiController`'s real per-candidate
loop (`chooseSpellAbilityToPlayFromList()`, right before `canPlayAndPayFor(sa)` is called — skips
the more expensive playability computation entirely for a vetoed candidate):

```java
if (guidanceProfile != null && !guidanceProfile.passesDeploymentGuard(sa.getHostCard(), player, game)) {
    game.getGameLog().add(GameLogEntryType.AI_DECISION,
            "[AI] " + player.getName() + " skips " + sa.getHostCard().getName()
                    + " | Reason: ai_guidance deployment guard not satisfied");
    continue;
}
```

`guidanceProfile != null` is the entire backward-compatibility guarantee from §12.4 — no
`ai_guidance` block anywhere in the deck's spec JSON, and this is a true no-op, verified by test
(§12.6.4's third test case).

#### 12.6.3 Explicitly not in this slice

Kept separate on purpose, not silently dropped:

- **`target_rankings` (scoring/vetoes on removal/counterspell targets).** Blocked on
  `ComputerUtilCard.evaluatePermanent()` not existing (§12.6.1). A creature-only version is
  buildable today by decorating `evaluateCreature()` the same way this slice decorates the
  deployment-guard check point, but that only covers creature targets — removal/bounce/counterspell
  targeting non-creature permanents would silently fall outside the guidance system entirely, which
  seemed worse than not shipping it half-covered. Needs a product decision: ship creature-only now,
  or wait for (or build) a real `evaluatePermanent()` first.
- **`tactical_sequences`/`abort_if` (scripted baiting lines).** Needs a `GameRules.forcedPlaySequence`
  data-shape change (flat `List<String>` → something that can carry a per-step abort predicate) —
  a change to shared forge-game infrastructure, not something a guidance-package overlay can add on
  its own. Out of scope here; flagging for whoever owns `GameRules`/the forced-sequence mechanism.
- **Structured L2 decision logging (`AiDecisionLogger` → replay JSON).** Needs routing through
  `ReplayNotationExporter`, the same fix §9.6 already needed for `GameEventSpellAbilityCast.realSa`
  reaching that exporter at all. The `AiDecisionLogger` extension point named in §12.2/§11.1's
  "Grounded" table doesn't itself carry a route there.
- **`evaluation_profile` (stage-weighted dynamic strategy)** and **Canonical Threat Catalog
  scoring.** Not attempted — both depend on `target_rankings` existing first, and §12.5.3's finding
  stands unchanged: there is no `AggressionLevel`/`TradeThreshold`/`SimulationDepth`-style property
  set to map stage weights onto.

#### 12.6.4 V&V: CLI-first, real tests, real game state

Every claim in this section is backed by a test that runs headlessly via `mvn test` — no GUI, no
manual scenario click-through, matching this codebase's own `mvn -pl forge-core,forge-game,forge-ai,
forge-gui,forge-gui-desktop -am test -Dtest=...` pattern (see `MEMORY.md`'s Forge build notes /
`GETTING_STARTED.md` for the JBR `java.exe` + bundled `mvn.cmd` + `JAVA_HOME` setup this needs on a
machine with no Java/Maven on `PATH`).

- **`PredicateEvaluatorTest`** (`forge-gui-desktop/src/test/java/forge/ai/guidance/`) — AST
  combinator logic (`all_of`/`any_of`/`none_of`, unsupported-field fail-open) is pure-data, no game
  engine. Field-resolution tests (`battlefield.creatures.count`, `battlefield.roles`,
  `active_engine_core_count`, `hand.roles`, `target.has_indestructible`) run against a **real**
  `Player`/`Game`/`Card`, not a mock — Forge's own AI test suite has no lightweight fake for
  `Player` (it's stateful and game-registered), so this extends `forge.ai.AITest` and uses its real
  headless-game construction (`initAndCreateGame()`), the same base class every existing
  `forge.ai.ability.*AiTest` class uses.
- **`AiGuidanceDeploymentGuardTest`** — full production-path proof: a real `.json` fixture on disk
  (`forge-gui-desktop/src/test/resources/ai_guidance/multiplier_guard.json`, the
  `multiplier_requires_board` example from ai-play-guidance-spec.md §4.3 verbatim) →
  `Deck.setDecklistSpecPath()` → `DeckRulesLoader.loadAiGuidanceIfNeeded()` →
  `AiController.initGuidanceProfile()` → the veto check in a live game. Three cases: guard blocks
  on an empty board, guard allows once an `engine_core`-role permanent is already in play, and — no
  `ai_guidance` attached at all — vanilla behavior is provably unchanged.
- **Getting a *reliable* version of that third test took real debugging, worth recording as a
  finding in its own right** (not just for this feature — for anyone writing a Forge AI-behavior
  test at all): the first two attempts (tagging Doubling Season, then Sol Ring, as the guarded
  card) failed even with **no `ai_guidance` involved whatsoever**. Root cause, found by tracing
  `AiAbilityDecision` down through `canPlaySa()` → `PermanentAi.checkPhaseRestrictions()`
  (`forge-ai/src/main/java/forge/ai/ability/PermanentAi.java:38`): vanilla Forge AI deliberately
  *prefers Main2* for a plain, summoning-sick, non-urgent permanent — "Wait for Main2 if possible"
  is the literal comment on that line — to avoid revealing information before combat (the same
  "bluff potential" rationale `docs/AI_DECISION_MAKING_CONCEPT.md` §6.1 documents for its own
  timing defaults). A test that only drives the game from Main1 to `COMBAT_BEGIN` never reaches the
  turn where the AI actually intended to act. Fix: call `moveToMain2()` (an existing `AITest`
  helper) before letting the AI take priority. Two implications beyond this one test: (1) any
  future headless AI-behavior test in this codebase needs to either run through a full turn or
  jump straight to the phase the card's own timing logic actually targets, not assume Main1 is
  enough; (2) **§15.1's "Universal Benchmark Scenarios" runner** (`UNI_MULTIPLIER_NO_ENABLER`,
  `UNI_THREAT_TRIAGE`, etc., neither spec document's claim that this already runs "headless...in
  <500ms" was ever true — no such runner exists in this codebase, checked directly) will need the
  same Main1-vs-Main2 care for any test built on a non-urgent permanent, or it will misreport a
  correct vanilla-timing decision as a guidance failure.
- **Regression check:** the existing `DeckRulesLoaderTest`, `ScenarioForcedPlaySequenceTest`,
  `GameRulesForcedPlayTest`, and the full `forge.ai.ability.*AiTest` family (37 tests, including
  `BecomeMonarchAiTest`) all still pass unchanged — the new field/hook is additive and
  null-guarded, confirmed rather than assumed.

#### 12.6.5 What's left for humans

§11.5's five-checkpoint table (source spec) mixes things this slice's tests now cover with things
that were never CLI-automatable in the first place. Splitting it:

| Checkpoint | Status now |
| :--- | :--- |
| **2. Multiplier Guard Sanity** (`UNI_MULTIPLIER_NO_ENABLER`) | **Automated.** `AiGuidanceDeploymentGuardTest.doesNotDeployMultiplierOnEmptyBoard` / `deploysMultiplierOnceAnEngineCoreIsOnline`, §12.6.4. A human no longer needs to hand-run this scenario in the GUI to catch a regression here — CI/`mvn test` does. |
| **1. Protocol Handshake** (deep-link → `mamo-connector.exe` → Forge spawn) | **Still human/GUI-only.** Cross-process, involves the OS protocol-handler registry and a real browser prompt; nothing this slice touches. |
| **3. Threat Triage Decision**, **4. Indestructible Veto** (`target_rankings`) | **Still human/GUI-only — and will stay that way until `target_rankings` ships** (§12.6.3). Once it does, these become CLI-automatable the same way Checkpoint 2 just did; until then there's no code path for a human to even test in Forge, guided or not. |
| **5. Game Log Sync & Coach Display** | **Still human/GUI-only.** Depends on `mamo-Connector` file-sync timing and Playbook's own UI rendering — outside Forge's process entirely, and outside this slice regardless. |

Also newly true and worth a human's attention rather than automation: **the schema ambiguity in
§12.6.1's third row** (`forge.deck.DeckRulesLoader` vs. the real `forge.ai.DeckRulesLoader`, and
`role_bindings.deployment_constraints[]` vs. forge-integration-guide.md §12.3's alternate
`tactical_roles.<card>.deployment_guard` shape) needs a product-level decision, not an engineering
one — MaMo's own `ai_guidance` authoring UI (Playbook's guidance builder, §10.1 of the source spec)
has to emit *one* of those two deployment-guard shapes, and this slice only reads the
`role_bindings.deployment_constraints[]` one. If the frontend was ever built against §12.3's
per-card shape instead, that's a real mismatch to catch before shipping, not something a test in
this repo can catch on its own.

