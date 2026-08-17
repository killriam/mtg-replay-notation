# Changelog

All notable changes to the MTG Replay Notation specification will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.6.6] - 2026-08-17

### Fixed
- **`forge-integration-guide.md` §9.6 (new): Forge's CAST event `cost`/`x`/`choices` capture was
  wired but unreachable.** `GameEventSpellAbilityCast` (the event both Forge's separate
  `ReplayNotationExporter` and its Demo Play recorder subscribe to) only ever carried
  `SpellAbilityView`/`StackItemView`, never the real `SpellAbility` - so the one call site that
  could have supplied real cost/X-value data always passed `null`, making
  `ReplayNotationExporter.getAdditionalCosts()`/`getXManaCostPaid()` dead code despite being
  correct and already implemented. Fixed on the Forge side (`killriam/forge`,
  `replay-Features`, commit `9e02627a51d`) by carrying the real `SpellAbility` on the event; Demo
  Play recordings now populate `cost.mana`/`cost.additional`/`cost.alternative`/`x` per this
  spec's §CAST Event schema, plus a Forge-specific `choices.sacrifice` extension for cards
  sacrificed as an additional cost (e.g. Metamorphosis). Not yet consumed when a scripted
  `events[]` sequence is replayed - see §9.6 for the full writeup, known simplifications
  (flat `targets` names instead of `{slot, obj}` objects, no `modes` capture), and test coverage.
- **§10's "NOT YET IMPLEMENTED" status flagged as needing re-verification** - later Forge-side
  work appears to have at least partially implemented the constructed-match scenario toggle
  (`PlayerPanel.java` scenario picker, `docs/SCENARIO_STARTING_HAND_FORMAT.md`'s "Von einem Deck
  referenzieren"), but this hasn't been independently audited end-to-end the way §9.4 was. Added
  a warning note pending that audit rather than asserting either status without verification.

### Housekeeping
- Forge's local submodule checkout of this repo (`killriam/forge/mtg-replay-notation`) had drifted
  3 months behind this canonical copy (pinned at `d023626`, 2026-05-19). Synced to `e8f0cd8`
  (2026-08-16) and documented submodule init/refresh steps in Forge's `GETTING_STARTED.md`, which
  previously mentioned the submodule in passing but never explained how to populate or refresh it.

## [1.6.5] - 2026-08-16

### Fixed
- **`schema/commander-decklist-schema.json` didn't validate `deck_rules.scenarios` at all** — the
  markdown spec (§6.4) has documented `id`/`type`/`name` as required per scenario since v1.2.0,
  but the JSON Schema file (still self-described as "v1.0.0") had no `scenarios` property on
  `DeckRules` and no `DecklistScenario` definition, so a scenario missing its type or name (or
  the whole `scenarios` array being malformed) would silently pass schema validation. Added
  `DecklistScenario`, `CardRef` (string or `{"group": "..."}`), `ScenarioTurnEntry`,
  `ScenarioZoneRequirement`, `ScenarioPreconditions`, `ScenarioFocus`, and `ScenarioBoardState`
  definitions, wired `scenarios` into `DeckRules`. Verified against the spec's own three
  documented examples (`best_starting_hand`, `mid_game`, `eval_sequence`) — all validate; a
  scenario missing `type`/`name` correctly fails.
- **`turns[].drawn` schema/spec mismatch** — the field table said `drawn` was required, but
  `MaMoFrontend`'s own implementation (and its `playbook.spec.md` AC-EXP-007) had already
  relaxed this to optional, since a turn scripting only an attack/activate action against an
  existing permanent has no new card to report. Schema and §6.4.2's field table both now mark
  `drawn` optional, matching what's actually shipped.
- **`turns[].actions`** (attack/activate actions, `{type, source}`) — implemented in
  MaMoFrontend's export pipeline and documented there as a "local extension… not yet coordinated
  into the shared spec" (`playbook.spec.md` AC-EXP-007). Now formally part of both the schema and
  §6.4.2's field table.

### Added
- **`DecklistScenario.deck_id`** (optional) — a scenario's owning deck reference. Normally
  implicit (a scenario embedded in a deck's own exported document belongs to that document's
  `meta.deck_id`) and omitted; only meaningful once a scenario reference can cross into another
  deck's context, e.g. attaching an opponent's own Perfect Game scenario to a constructed match.
  See `forge-integration-guide.md` §10 (proposed, not yet a live pipeline) for the use case this
  is meant to support.

## [1.6.4] - 2026-08-15

### Documentation
- **`forge-integration-guide.md`** — Added §10, a concept/gap document (not an implemented
  pipeline) for a third scenario-in-Forge use case: attaching a scenario's forced draw order to
  a normal constructed match, with a human-plays-it-as-a-hint vs. AI-plays-it-scripted split
  depending on who controls the scripted seat. Audits what §9's already-built mechanisms
  (`ScenarioLibrarySetup`, `setForcedPlaySequence` + `AiController` soft enforcement) already
  cover for free vs. what's a genuine gap in every layer — no `controlled_by`/seat-role concept
  exists anywhere yet, `getForgeScenarioExport`/`buildEventsFromCards` hardcode the opponent
  seat (P2) empty with no scripting support at all, and there's no cross-deck scenario reference
  (Validation Rule 14 stays same-deck-only for the existing `eval_scenario_ids` use case; §10
  proposes a scoped, additional allowance for this new one). Specifies requirements for
  `new-backend`, `mamo-Connector`, and `MaMoFrontend`; Forge-side implementation is explicitly
  handed off, not designed here (§10.4.3 lists open questions for that team, doesn't answer
  them). §0 updated with a one-paragraph pointer distinguishing this from §9's real pipeline.
  Proposed JSON shape in §10.3 is not a ratified schema change — no version bump beyond this
  changelog/doc entry.

## [1.6.3] - 2026-08-10

### Fixed
- **`forge-integration-guide.md` §9.4** — The `events[].a` actor-string mismatch flagged in
  1.6.2 turned out to be worse than "unverified": the GUI Replay Scenario submenu (the pipeline
  mamo-Connector actually drives) didn't consume `events` at all, and the one path that did
  (CLI `-s`) used the raw player id as a lobby name with no translation, so it never matched
  either. Fixed on the Forge side (`killriam/forge`, `replay-Features`) by redefining
  `events[].a` as a plain seat id (`"P1"`/`"P2"`, matching `scenario.players`' own keys) that
  each launcher translates internally to its actual runtime lobby name, instead of requiring
  the exporter to predict/reconstruct that name. §9.3 and §9.4 rewritten accordingly; §9.5
  checklist updated.
- **`new-backend`/mamo-Connector — applied 2026-08-10, commit `a34a607`**: `buildEventsFromCards`
  now emits the plain seat id `"P1"` for `events[].a` instead of the constructed
  `Ai(1)-{username} - {deckName} ({date})` string, closing the loop from the fix above. The
  dead username/deckDate DB lookup that only existed to build that string was removed too. New
  test asserts the actor is always `"P1"`. Verified end-to-end on the Forge side via a live CLI
  scenario run: the saved replay JSON shows the scripted `PLAY_LAND` event applying correctly.
  Real fix to a live feature — scripted plays from ▶ Play in Forge (scenario) were being
  silently dropped before this.

## [1.6.2] - 2026-08-09

### Documentation
- **`forge-integration-guide.md`** — Added §0 (clarifying this guide covers two unrelated
  pipelines) and §9, documenting the live, already-implemented "Scenario Viewer" pipeline
  (`format: "mtg-replay"`, `version: "1.8.0"`, `mode: "scenario"`,
  `scenario.type: "opening_hand_test"`) that `new-backend`'s `getForgeScenarioExport` and
  `mamo-Connector`'s `playtest-scenario` deeplink actually use today. This format was previously
  undocumented here — §§1–8 describe an older, unrelated, manual-setup mechanism
  (`eval_sequence`/`best_starting_hand`/`perfect_game` via the `mtg-commander-decklist` export)
  that this repo's own text already says Forge does not auto-follow. The new §9 points to the
  authoritative field reference (`docs/SCENARIO_STARTING_HAND_FORMAT.md` in the Forge fork at
  `github.com/killriam/forge`, branch `replay-Features`) rather than duplicating it, and flags an
  unverified actor-string mismatch risk (§9.4) between what `new-backend` currently generates for
  `events[].a` and what the scenario `.dck`'s actual filename/in-game lobby name would be.
- No schema or field changes — `schema/replay-schema.json` and existing examples are unaffected.

## [1.6.1] - 2026-03-28

### Fixed
- **Replay Schema** — Brought `replay-schema.json` into full parity with v1.5.0+ spec:
  - Added `spec_version`, `events`, `learning_markers`, `per_turn_summary`,
    `game_summary` top-level properties
  - Backward compatibility: both `events` (v1.5.0+) and `log_l1` (pre-v1.5.0) accepted;
    neither is strictly required so parsers can handle both key names
  - Extended `PlayerMeta` with `deck_link`, `is_ai`, `player_type`, `starting_life`
  - Extended `CardDefinition` with `oracle_text`, `power`, `toughness`, `subtypes`
  - Added `DISCARD` to `L1Event.type` enum
  - Added `"unknown"` to `win_condition` enum
  - Added `LearningMarker`, `LearningMarkerSnapshot`, `PerTurnSummary`,
    `PerTurnPlayerStats`, `GameSummary`, `GameSummaryPlayerStats` definitions
- **Specification** — Added missing event data schemas for: `PLAY_LAND`, `ACTIVATE`,
  `TRIGGER`, `TAP`, `COUNTERS`, `DECLARE_ATTACKERS`, `DECLARE_BLOCKERS`, `DISCARD`,
  `PASS_PRIORITY`, `CHOOSE`, `STATE_BASED`, `RANDOM`
- **Specification** — Fixed `log_l1` references to `events` in §7, §10, §11
- **Specification** — Added `DISCARD` to player decision event type table
- **Specification** — Added `unknown` to win condition values table
- **Example** — Updated `simple-game.json` from v1.1.0 to v1.5.0:
  - Uses `events` key, includes `spec_version`, `game_start`, `game_summary`,
    `per_turn_summary`, `learning_markers` sections
  - Extended card_index entries with `oracle_text`, `power`, `toughness`, `subtypes`
  - Added `GAME_START` and `ACTIVE_PLAYER_CHANGE` events
  - Extended player metadata with `deck_link`, `is_ai`, `player_type`, `starting_life`

## [1.6.0] - 2026-03-11

### Added
- **Commander Decklist Notation** — New companion specification
  (`spec/commander-decklist-spec.md`) defining a JSON format for Commander decklists
  with four sections: `commander`, `main`, `sideboard`, and `maybeboard`
- **Card Entry Fields** — Each card entry records `quantity`, `name`, `edition`,
  `collector_number` (together uniquely identifying the artwork), `primary_mechanic`,
  and `additional_mechanics`
- **Deck Rules** — New `deck_rules` block in decklist files with:
  - `mulligan` — Opening hand scoring model: configurable per-category card values
    (`land`, `cmc_0_to_2`, `cmc_3`, `other`), per-card overrides, and
    per-round keep thresholds
  - `combos` — Array of named combo declarations (pieces, result, tags)
  - `dont_combos` — Array of anti-synergy declarations (pieces, reason, severity)
- **Inline Decklist in Replay Files** — Optional top-level `decklist` map in replay
  files allows embedding full decklist objects keyed by player ID
- **New JSON Schema** — `schema/commander-decklist-schema.json` for validating
  standalone decklist files
- **Schema Updates** — `schema/replay-schema.json` updated to v1.6.0 with new
  `CommanderDecklist`, `DecklistMeta`, `DecklistCard`, `DeckRules`, `MulliganRule`,
  `CardValueOverride`, `MulliganThreshold`, `ComboDeclaration`, and
  `DontComboDeclaration` definitions
- **Example** — `examples/commander-decklist.json` — reference Atraxa Superfriends
  Commander decklist demonstrating all new fields

## [1.5.0] - 2026-02-22

### Changed
- **Event Log Key Renamed**
  - `log_l1` renamed to `events` at top level
  - Consumers should check for both keys for backward compatibility

### Added
- **New Top-Level Fields**
  - `spec_version` — Explicit spec version (may differ from `version`)
  - `per_turn_summary` — Pre-computed per-turn statistics array
  - `game_summary` — Pre-computed game-wide statistics object

- **New Event Types**
  - `DRAW` — Card draw event with `obj`, `card_name`, `from`, `to`, `pos`, `visibility`
  - `GAME_START` — Game initialization event with `players`, `game_type`, `first_player`

- **Extended Player Metadata**
  - `is_ai` — Boolean indicating whether player is an AI
  - `player_type` — String: `"Human"` or `"AI"`
  - `starting_life` — Starting life total for the player

- **Extended Event Data**
  - `CAST` — Added `total_mana_value` and `play_mode` fields to cost data
  - `PLAY_LAND` — Added `player` field
  - `TRIGGER` — Added `trigger` (text) and `source_name` fields
  - `ACTIVATE` — Added `ability` (text) and `controller` fields
  - `COUNTERS` — Added `card_name` field

- **New Phase Code**
  - `END_OF_TURN` — End of turn phase (in addition to existing `END`)

## [1.4.0] - 2026-02-21

### Added
- **Deck Link** — `deck_link` field in player metadata with revision anchor format

## [1.3.0] - 2026-02-21

### Added
- **Learning Markers**
  - `LEARNING_MARKER` event type for player-placed game state bookmarks
  - `learning_markers` top-level section for quick marker navigation

## [1.2.0] - 2026-02-08

### Added
- **Game Start Section**
  - `toss_winner` — Player who won the die roll/coin toss
  - `play_draw_choice` — Whether toss winner chose to play or draw
  - `starting_player` — Player who takes the first turn
  - `mulligans` — Array with mulligan summary per player

- **Enhanced MULLIGAN Event**
  - `decision` — "keep" or "mulligan"
  - `hand_size_before` / `hand_size_after` — Hand sizes
  - `mulligan_count` — Number of mulligans taken
  - `cards_seen` — Card IDs in hand when decision made (optional)
  - `cards_to_bottom` — Cards put to bottom (London mulligan)
  - `cards_to_bottom_names` — Human-readable names

## [1.1.0] - 2026-02-08

### Added
- **Metadata Enhancements**
  - `win_condition` field in meta section with values: `life_zero`, `commander_damage`, `decked`, `poison`, `concession`, `alternate_win`, `draw`
  - `conceded` boolean field to indicate if any player conceded
  - `deck_name` field in player metadata
  - Documented `deck_hash` calculation algorithm (SHA-256 based, 16 hex chars)

- **New Event Type**
  - `RESOURCES` event for tracking player resources at upkeep (land_count, available_mana)

- **Human-Readable Fields**
  - `card_name` field added to CAST event data
  - `card_name` field added to MOVE event data
  - `card_name` field added to PUT_ON_STACK event data
  - `source_name` and `target_name` fields added to DAMAGE event data

### Changed
- Version number updated from 1.0.0 to 1.1.0
- Updated all JSON examples to reflect new fields

## [1.0.0] - 2025-12-20

### Added
- Initial specification release
- Two-level architecture (L1 Event Log, L2 Learning View)
- Core concepts: Object IDs, Time Markers, Zone Notation
- Metadata section with game info and player data
- Card index for card definitions
- Initial state representation
- Level 1 Events:
  - Player Decision Events: CAST, ACTIVATE, PLAY_LAND, DECLARE_ATTACKERS, DECLARE_BLOCKERS, PASS_PRIORITY, MULLIGAN, CHOOSE
  - System Events: PUT_ON_STACK, TRIGGER, RESOLVE, MOVE, DAMAGE, LIFE, COUNTERS, TAP, PHASE_CHANGE, STATE_BASED, RANDOM
- Level 2 Learning Units with before/after state snapshots
- Stack item representation with targets and choices
- Annotations for learning context
- Validation rules
- JSON Schema for file validation

---

## Upgrading

### From 1.0.0 to 1.1.0

The 1.1.0 release is **backward compatible** with 1.0.0. New fields are optional.

**Recommended updates for producers:**
1. Add `card_name` to CAST, MOVE, PUT_ON_STACK events for readability
2. Add `source_name` and `target_name` to DAMAGE events
3. Include `win_condition` in metadata when game ends
4. Add `deck_name` to player metadata
5. Generate `RESOURCES` events at each upkeep

**For consumers:**
- Handle missing new fields gracefully (they're optional)
- Check version field to determine available features
