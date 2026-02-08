# Changelog

All notable changes to the MTG Replay Notation specification will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
