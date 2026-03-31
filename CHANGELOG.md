# Changelog

All notable changes to the MTG Replay Notation specification will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.6.2] - 2026-03-31

### Added
- **Scenario Notation** — Optional top-level `scenario` block for rules clarification and combo outcome analysis.
- **Schema** — Added `ReplayScenario` definition and `scenario` root property in `schema/replay-schema.json` (v1.6.1+).
- **Spec** — Added documentation of scenario format in `spec/MTG-REPLAY-NOTATION.md`.
- **Example** — Added `examples/scenario.json`.

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
