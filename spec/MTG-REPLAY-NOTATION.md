# MTG Replay & Learning Notation

## Format Specification v1.5.0

**Status:** Stable  
**Published:** February 2026  
**Purpose:** Human-readable specification for understanding MTG game replay files

**Version History:**
- **1.0.0** (December 2025): Initial specification
- **1.1.0** (February 2026): Added `win_condition`, `conceded`, `deck_name`, `deck_hash` algorithm, `RESOURCES` event, `card_name` in events
- **1.2.0** (February 2026): Added `game_start` section (toss winner, starting player, play/draw choice), enhanced `MULLIGAN` event with full details
- **1.2.1** (February 2026): Added `ACTIVE_PLAYER_CHANGE` event for turn transitions, major impact turn/play identification guide
- **1.3.0** (February 2026): Added `LEARNING_MARKER` event and `learning_markers` top-level section for player-placed game state bookmarks
- **1.4.0** (February 2026): Added `deck_link` with revision anchor to player metadata
- **1.5.0** (February 2026): Renamed `log_l1` to `events`; added `spec_version`, `per_turn_summary`, `game_summary`; new `DRAW` and `GAME_START` event types; extended player metadata with `is_ai`, `player_type`, `starting_life`
- **1.6.0** (March 2026): Added scenario replay mode (`mode: "scenario"` + `scenario` object) for interaction checks, rules clarification, and combo outcome analysis

---

## 1. Introduction

This document specifies the MTG Replay & Learning Notation format, a JSON-based format for recording complete Magic: The Gathering game sessions. The format is designed to be:

- **Deterministic** — Games can be replayed exactly
- **Comprehensive** — All game actions and state changes are captured
- **Analyzable** — Suitable for AI training and decision analysis
- **Human-readable** — Clear structure for manual inspection

---

## 2. File Structure Overview

A replay file contains:

```json
{
    "format": "mtg-replay",
    "version": "1.5.0",
    "spec_version": "1.5.0",
    "meta": {
        /* Game metadata */
    },
    "seed": 1234567890,
    "game_start": {
        /* Toss/mulligan info */
    },
    "card_index": {
        /* Card definitions */
    },
    "initial_state": {
        /* Starting game state */
    },
    "events": [
        /* Event log (renamed from log_l1 in v1.5.0) */
    ],
    "views_l2": [
        /* Level 2: Learning view */
    ],
    "learning_markers": [
        /* Player-placed bookmarks for review */
    ],
    "per_turn_summary": [
        /* Pre-computed per-turn statistics (v1.5.0+) */
    ],
    "game_summary": {
        /* Pre-computed game-wide statistics (v1.5.0+) */
    },
    "mode": "game", /* "scenario" for interactive/rules-check scenarios (v1.6.0+) */
    "scenario": {
        "description": "P1 has both Card A and Card B in play and attacks with X; does the replacement effect apply before damage assignment?",
        "purpose": "rules_clarification",
        "question": "Does Card A's replacement effect interact with Card B to prevent combat damage this turn?",
        "expected_outcomes": ["Yes, damage is prevented", "No, damage is assigned normally"],
        "notes": "Designed to validate edge-case ruling for layered continuous effects in combat damage step."
    }
}
```

### 2.1 Two-Level Architecture

**Level 1 (L1) — Event Log**

- Chronological sequence of all game events
- Lossless, authoritative record
- Required for deterministic replay

**Level 2 (L2) — Learning View**

- Decision-focused snapshots
- Before/after game states
- Stack contents with full context
- Derived from L1 events

---

## 3. Core Concepts

### 3.1 Object Identifiers

Every game object has a unique, immutable identifier:

| Type           | Format       | Example    | Description              |
| -------------- | ------------ | ---------- | ------------------------ |
| Card/Permanent | `c` + number | `c42`      | Physical card in game    |
| Token          | `t` + number | `t7`       | Token creature/permanent |
| Stack Object   | `s` + number | `s1`       | Spell/ability on stack   |
| Player         | `P` + number | `P1`, `P2` | Player in the game       |

**Important:** IDs never change. A card keeps its ID when moving between zones.

---

### 3.2 Time Markers

Format: **`T<turn>.<phase>[:<priority_pass>]`**

Examples:

- `T1.UP` — Turn 1, Upkeep Phase
- `T3.MP1:2` — Turn 3, Main Phase 1, Priority Pass 2
- `T4.COMBAT:0` — Turn 4, Combat Phase, Priority Pass 0

**Phase Codes:**

| Code      | Phase        | Description            |
| --------- | ------------ | ---------------------- |
| `UP`      | Upkeep       | Upkeep phase           |
| `DRAW`    | Draw         | Draw step              |
| `MP1`     | Main Phase 1 | Pre-combat main phase  |
| `COMBAT`  | Combat       | Combat phase           |
| `MP2`     | Main Phase 2 | Post-combat main phase |
| `END`     | End          | End step               |
| `CLEANUP` | Cleanup      | Cleanup step           |

---

### 3.3 Zone Notation

**Shared Zones:**

- `battlefield` — Permanents in play
- `stack` — Spells and abilities resolving
- `exile` — Exiled cards

**Player-Specific Zones:**

- `P1:hand` — Player 1's hand
- `P1:library` — Player 1's library
- `P1:graveyard` — Player 1's graveyard
- `P1:command` — Player 1's command zone (Commander)

---

## 4. Metadata Section

The `meta` section contains information about the game:

```json
{
    "game_id": "game-uuid-here",
    "timestamp": "2026-02-08T15:30:45Z",
    "game_type": "Commander",
    "players": {
        "P1": {
            "name": "Alice",
            "deck_name": "Atraxa Superfriends",
            "deck_hash": "a3f8c2d1e9b7f604",
            "deck_link": "https://mamo.games/deck/abc123-uuid#22022026_a3f8c2d1e9b7f604"
        },
        "P2": {
            "name": "Forge AI",
            "deck_name": "Mono-Red Aggro",
            "deck_hash": "7b2e5a9c4d1f8306",
            "deck_link": null
        }
    },
    "winner": "P1",
    "win_condition": "life_zero",
    "conceded": false,
    "turns": 12,
    "duration_seconds": 720
}
```

**Fields:**

- `game_id` — Unique identifier for this game
- `timestamp` — When game started (ISO 8601)
- `game_type` — Format (Constructed, Limited, Commander, etc.)
- `players` — Map of player IDs to player information
- `winner` — Player ID of winner, `"draw"`, or `null` for ongoing
- `win_condition` — How the game ended (see below)
- `conceded` — Boolean indicating if any player conceded
- `turns` — Total number of turns played
- `duration_seconds` — Real-time duration of game

### 4.1 Player Metadata

Each player entry contains:

- `name` — Player display name
- `deck_name` — Name of the deck used
- `deck_hash` — SHA-256 based deck identifier (16 hex characters)
- `deck_link` — URL to the deck page with revision anchor (optional, `null` for AI players)
- `is_ai` — Boolean indicating whether this player is an AI (v1.5.0+)
- `player_type` — String: `"Human"` or `"AI"` (v1.5.0+)
- `starting_life` — Starting life total for this player (v1.5.0+)

### 4.2 Deck Link Format

The `deck_link` field provides a direct URL to the deck as it existed at game time. The URL includes a **fragment anchor** encoding the game date and the deck hash:

```
https://mamo.games/deck/<deck-uuid>#<DDMMYYYY>_<deck_hash>
```

**Anchor components:**
- `DDMMYYYY` — Date the game was played (from `meta.timestamp`), e.g. `22022026` for February 22, 2026
- `deck_hash` — The 16-character deck hash at game time

**Examples:**
```
https://mamo.games/deck/abc123-uuid#22022026_a3f8c2d1e9b7f604
https://mamo.games/deck/xyz789-uuid#13022026_7b2e5a9c4d1f8306
```

**Purpose:**
- Links the replay to the exact deck version used
- The anchor allows the frontend to resolve the correct deck revision even if the deck has been updated since the game
- AI players or unknown decks use `null`

### 4.3 Deck Hash Calculation

The `deck_hash` provides a stable identifier for a deck based solely on its card contents, independent of the deck name. This allows identifying the same deck even if renamed.

**Algorithm:**
1. For Main and Commander sections only (Sideboard excluded):
   - Collect all cards as "CardName:Quantity" pairs
   - Sort alphabetically
2. Concatenate all entries into a canonical string
3. Calculate SHA-256 hash
4. Return first 16 hex characters (64 bits)

**Why exclude Sideboard?**
- Sideboards may vary between games/tournaments
- The core deck identity is defined by Main + Commander
- Same maindeck with different sideboards = same hash

### 4.4 Win Condition Values

| Value | Description |
|-------|-------------|
| `life_zero` | Opponent's life reduced to 0 or less |
| `commander_damage` | 21+ commander damage from single commander |
| `decked` | Opponent attempted to draw from empty library |
| `poison` | Opponent received 10+ poison counters |
| `concession` | Opponent conceded |
| `alternate_win` | Card effect (e.g., Laboratory Maniac, Thassa's Oracle) |
| `draw` | Game ended in a draw |
| `unknown` | Win condition could not be determined |

### 4.5 Game Start Information

The `game_start` section captures pre-game decisions:

```json
{
    "game_start": {
        "toss_winner": "P1",
        "play_draw_choice": "play",
        "starting_player": "P1",
        "mulligans": [
            {
                "player": "P1",
                "starting_hand_size": 7,
                "mulligans_taken": 1,
                "final_hand_size": 6,
                "cards_to_bottom": 1
            },
            {
                "player": "P2",
                "starting_hand_size": 7,
                "mulligans_taken": 0,
                "final_hand_size": 7,
                "cards_to_bottom": 0
            }
        ]
    }
}
```

**Fields:**

- `toss_winner` — Player who won the die roll/coin toss
- `play_draw_choice` — Choice made: `"play"` (go first) or `"draw"` (go second)
- `starting_player` — Player who takes the first turn
- `mulligans` — Array of mulligan decisions per player

**Mulligan Entry Fields:**

| Field | Description |
|-------|-------------|
| `player` | Player ID |
| `starting_hand_size` | Initial hand size (usually 7) |
| `mulligans_taken` | Number of times player mulliganed |
| `final_hand_size` | Cards in hand after all mulligans |
| `cards_to_bottom` | Cards put on bottom (London mulligan rule) |

---

## 4.6 Scenario Analysis Mode (v1.6.0+)

Scenario mode is an additional notation style for describing focused game states and interactions without requiring a full competitive match. It is useful for rules clarification and combo outcome analysis.

- `mode`: `"scenario"` (optional; if omitted, defaults to `"game"`)
- `scenario`: structured scenario metadata object

Required fields in `scenario`:
- `description`: human-readable summary of the playing situation
- `purpose`: one of `"rules_clarification"`, `"combo_outcome"`, `"interaction_check"`, or `"general"`

Optional fields in `scenario`:
- `question`: the specific ruling/analysis prompt
- `expected_outcomes`: list of anticipated answers or possible results
- `notes`: additional context for judges, rules engines, or AI analysis

Example:

```json
{
  "format": "mtg-replay",
  "version": "1.6.0",
  "spec_version": "1.6.0",
  "mode": "scenario",
  "meta": {
    "game_id": "scenario-12345",
    "timestamp": "2026-03-31T12:00:00Z",
    "game_type": "Scenario",
    "players": {
      "P1": {"name": "Test Player", "is_ai": false, "player_type": "Human", "starting_life": 20}
    }
  },
  "scenario": {
    "description": "P1 controls Hullbreacher and multiple Counterspells in hand at the end of opponent's draw step.",
    "purpose": "rules_clarification",
    "question": "How many cards does P1 draw when opponent attempts to draw for turn and Spellstutter Sprite triggers?",
    "expected_outcomes": ["0", "1", "2"],
    "notes": "Use this to validate delayed replacement effects and Gitrog interaction rules."
  },
  "card_index": {
    "Hullbreacher": {"name": "Hullbreacher", "type": "Creature - Elf", "oracle_text": "..."},
    "Spellstutter Sprite": {"name": "Spellstutter Sprite", "type": "Creature - Faerie Wizard", "oracle_text": "..."}
  },
  "initial_state": {
    "turn": 5,
    "phase": "DRAW",
    "players": {
      "P1": {"life": 20, "hand": ["c1", "c2"]},
      "P2": {"life": 18}
    }
  }
}
```

## 5. Card Index

The `card_index` maps card names to their definitions:

```json
{
    "Lightning Bolt": {
        "name": "Lightning Bolt",
        "cost": "{R}",
        "type": "Instant",
        "oracle_id": "scryfall-uuid"
    },
    "Grizzly Bears": {
        "name": "Grizzly Bears",
        "cost": "{1}{G}",
        "type": "Creature — Bear"
    }
}
```

**Fields:**

- `name` — English card name
- `cost` — Mana cost in standard notation
- `type` — Full type line
- `oracle_id` — (Optional) Scryfall Oracle ID

### 5.1 How Card Index Lookup Works

The replay format uses a **two-level identification system** to keep events compact while enabling full card information lookup:

1. **Card IDs** (`c1`, `c5`, `c42`, etc.) — Unique identifiers for each physical card instance in the game
2. **Card Index** — A lookup table mapping card names to their definitions

**Linking Cards to the Index:**

Each card object in the `objects` map contains a `card_ref` field that references a key in the `card_index`:

```json
{
    "objects": {
        "c5": {
            "card_ref": "Grizzly Bears",
            "controller": "P1",
            "zone": "battlefield"
        }
    },
    "card_index": {
        "Grizzly Bears": {
            "name": "Grizzly Bears",
            "cost": "{1}{G}",
            "type": "Creature — Bear"
        }
    }
}
```

**Lookup Flow:**

1. An event references a card ID: `"card": "c5"`
2. Find `c5` in the `objects` map → `"card_ref": "Grizzly Bears"`
3. Look up `"Grizzly Bears"` in `card_index` → full card definition

**Why This Design?**

- **Compact events** — Events only need short IDs (`c5`) instead of full card data
- **Deduplication** — Cards appearing multiple times share one `card_index` entry
- **Tracking** — Card IDs are immutable, so you can follow a specific card through zone changes
- **Flexibility** — Events can optionally include `card_name` for human readability without duplicating full definitions

---

## 6. Initial State

The `initial_state` captures the game configuration at start:

```json
{
    "turn": 0,
    "phase": "PREGAME",
    "step": "PREGAME",
    "priority": null,
    "active_player": null,
    "players": {
        "P1": {
            "life": 20,
            "mana_pool": [],
            "counters": {},
            "lands_played_this_turn": 0,
            "max_hand_size": 7
        },
        "P2": {
            /* ... */
        }
    },
    "zones": {
        "battlefield": [],
        "stack": [],
        "exile": [],
        "P1:hand": [],
        "P1:library": { "count": 60 },
        "P1:graveyard": [],
        "P2:hand": [],
        "P2:library": { "count": 60 },
        "P2:graveyard": []
    },
    "objects": {}
}
```

---

## 7. Level 1 Events (`events`)

### 7.1 Event Structure

Each event in the `events` array has this structure:

```json
{
    "i": 42,
    "t": "T3.MP1:2",
    "a": "P1",
    "type": "CAST",
    "data": {
        /* Event-specific payload */
    }
}
```

**Fields:**

- `i` — Event index (starts at 0, increases by 1)
- `t` — Time marker
- `a` — Actor: `"P1"`, `"P2"`, or `"SYS"` (system)
- `type` — Event type (see section 7.2)
- `data` — Event-specific data (see section 7.3)

---

### 7.2 Event Types

Events are categorized into two groups:

#### Player Decision Events

Events where a player makes a strategic choice:

| Type                | When Used                                  |
| ------------------- | ------------------------------------------ |
| `CAST`              | Player casts a spell                       |
| `ACTIVATE`          | Player activates an ability                |
| `PLAY_LAND`         | Player plays a land                        |
| `DECLARE_ATTACKERS` | Player declares attacking creatures        |
| `DECLARE_BLOCKERS`  | Player declares blocking creatures         |
| `PASS_PRIORITY`     | Player passes priority                     |
| `MULLIGAN`          | Player mulligan decision (keep/mull)       |
| `CHOOSE`            | Player makes a choice (mode, target, etc.) |
| `DISCARD`           | Player discards a card                     |
| `LEARNING_MARKER`   | Player bookmarks current game state for later review |

#### System Events

Automatic game actions and state changes:

| Type           | When Used                       |
| -------------- | ------------------------------- |
| `PUT_ON_STACK` | Object is placed on the stack   |
| `TRIGGER`      | Triggered ability goes on stack |
| `RESOLVE`      | Stack object resolves           |
| `MOVE`         | Card changes zones              |
| `DAMAGE`       | Damage is dealt                 |
| `LIFE`         | Life total changes              |
| `COUNTERS`     | Counters are added/removed      |
| `TAP`          | Permanent taps or untaps        |
| `PHASE_CHANGE` | Game advances to new phase      |
| `ACTIVE_PLAYER_CHANGE` | Active player changes (turn transition) |
| `RESOURCES`    | Player resources at upkeep      |
| `STATE_BASED`  | State-based action occurs       |
| `RANDOM`       | Random event (shuffle, reveal)  |
| `DRAW`         | Card is drawn from library (v1.5.0+) |
| `GAME_START`   | Game initialization event (v1.5.0+) |

---

### 7.3 Event Data Schemas

#### MULLIGAN Event

```json
{
    "i": 1,
    "t": "T0.PREGAME",
    "a": "P1",
    "type": "MULLIGAN",
    "data": {
        "decision": "mulligan",
        "hand_size_before": 7,
        "hand_size_after": 6,
        "mulligan_count": 1,
        "cards_seen": ["c1", "c2", "c3", "c4", "c5", "c6", "c7"],
        "cards_to_bottom": ["c3"],
        "cards_to_bottom_names": ["Swamp"]
    }
}
```

**Data Fields:**

- `decision` — `"keep"` or `"mulligan"`
- `hand_size_before` — Hand size before this decision
- `hand_size_after` — Hand size after (same if keep, -1 if mulligan)
- `mulligan_count` — How many mulligans taken so far (0 = first look)
- `cards_seen` — Card IDs in hand when decision made (optional, for analysis)
- `cards_to_bottom` — Card IDs put to bottom (London mulligan, only on keep)
- `cards_to_bottom_names` — Human-readable names of cards put to bottom

**Keep Decision Example:**

```json
{
    "i": 2,
    "t": "T0.PREGAME",
    "a": "P1",
    "type": "MULLIGAN",
    "data": {
        "decision": "keep",
        "hand_size_before": 6,
        "hand_size_after": 6,
        "mulligan_count": 1,
        "cards_to_bottom": ["c8"],
        "cards_to_bottom_names": ["Forest"]
    }
}
```

---

#### CAST Event

```json
{
    "i": 10,
    "t": "T1.MP1:1",
    "a": "P1",
    "type": "CAST",
    "data": {
        "card": "c5",
        "card_name": "Lightning Bolt",
        "cost": {
            "mana": ["{R}"],
            "additional": [],
            "alternative": null
        },
        "modes": [],
        "x": null,
        "targets": [
            {
                "slot": "any target",
                "obj": "P2"
            }
        ],
        "choices": {}
    }
}
```

**Data Fields:**

- `card` — ID of card being cast
- `card_name` — Human-readable card name
- `cost.mana` — Mana paid
- `cost.additional` — Additional costs (sacrifice, discard, etc.)
- `cost.alternative` — Alternative cost used (if any)
- `modes` — Modal choices made
- `x` — Value chosen for X
- `targets` — Array of targets
- `choices` — Other choices made

---

#### MOVE Event

```json
{
    "i": 45,
    "t": "T3.MP1:3",
    "a": "SYS",
    "type": "MOVE",
    "data": {
        "obj": "c17",
        "card_name": "Lightning Bolt",
        "from": "stack",
        "to": "P1:graveyard",
        "pos": "top",
        "visibility": "public"
    }
}
```

**Data Fields:**

- `obj` — Object ID being moved
- `card_name` — Human-readable card name
- `from` — Source zone
- `to` — Destination zone
- `pos` — Position in destination (`"top"`, `"bottom"`, `"random"`)
- `visibility` — Whether move is public or hidden

---

#### PUT_ON_STACK Event

```json
{
    "i": 11,
    "t": "T1.MP1:1",
    "a": "SYS",
    "type": "PUT_ON_STACK",
    "data": {
        "stack": "s1",
        "kind": "SPELL",
        "source": "c5",
        "controller": "P1",
        "card": "c5",
        "card_name": "Lightning Bolt",
        "targets": [
            {
                "slot": "any target",
                "obj": "P2"
            }
        ],
        "choices": {}
    }
}
```

**Data Fields:**

- `stack` — Stack object ID
- `kind` — `"SPELL"`, `"ABILITY"`, or `"TRIGGER"`
- `source` — Source object ID
- `controller` — Controlling player
- `card` — Card ID (for spells)
- `card_name` — Human-readable card name
- `targets` — Targets declared
- `choices` — Choices made

---

#### DAMAGE Event

```json
{
    "i": 78,
    "t": "T5.COMBAT:8",
    "a": "SYS",
    "type": "DAMAGE",
    "data": {
        "source": "c12",
        "source_name": "Grizzly Bears",
        "target": "c24",
        "target_name": "Elvish Mystic",
        "amount": 3,
        "type": "combat",
        "prevented": 0
    }
}
```

**Data Fields:**

- `source` — Source of damage (object ID or `"unknown"`)
- `source_name` — Human-readable source name
- `target` — Target receiving damage (object or player ID)
- `target_name` — Human-readable target name (card name or player name)
- `amount` — Amount of damage
- `type` — `"combat"` or `"noncombat"`
- `prevented` — Amount prevented

---

#### LIFE Event

```json
{
    "i": 79,
    "t": "T5.COMBAT:9",
    "a": "SYS",
    "type": "LIFE",
    "data": {
        "player": "P2",
        "delta": -3,
        "new_total": 14,
        "cause": "combat damage"
    }
}
```

**Data Fields:**

- `player` — Player ID
- `delta` — Change amount (negative = loss, positive = gain)
- `new_total` — New life total
- `cause` — Reason for change (card name or description)

---

#### PHASE_CHANGE Event

```json
{
    "i": 6,
    "t": "T1.MP1",
    "a": "SYS",
    "type": "PHASE_CHANGE",
    "data": {
        "phase": "MAIN_1",
        "step": "MAIN",
        "active_player": "P1"
    }
}
```

**Data Fields:**

- `phase` — New phase name
- `step` — New step name
- `active_player` — Active player ID

---

#### RESOLVE Event

```json
{
    "i": 44,
    "t": "T3.MP1:3",
    "a": "SYS",
    "type": "RESOLVE",
    "data": {
        "stack": "s1"
    }
}
```

**Data Fields:**

- `stack` — Stack object ID being resolved

---

#### RESOURCES Event

Recorded at upkeep to track player resource state:

```json
{
    "i": 50,
    "t": "T4.UP",
    "a": "SYS",
    "type": "RESOURCES",
    "data": {
        "player": "P1",
        "land_count": 4,
        "available_mana": 5
    }
}
```

**Data Fields:**

- `player` — Player ID
- `land_count` — Number of lands on battlefield
- `available_mana` — Total available mana from untapped sources

---

#### ACTIVE_PLAYER_CHANGE Event

Recorded when the active player changes, typically at the start of a new turn:

```json
{
    "i": 100,
    "t": "T5.UP",
    "a": "SYS",
    "type": "ACTIVE_PLAYER_CHANGE",
    "data": {
        "previous_player": "P1",
        "new_player": "P2",
        "turn_number": 5
    }
}
```

**Data Fields:**

- `previous_player` — Player ID of the previous active player
- `new_player` — Player ID of the new active player
- `turn_number` — The turn number that is starting

---

#### LEARNING_MARKER Event

Placed by a player to bookmark the current game state for later review:

```json
{
    "i": 63,
    "t": "T4.MP1:1",
    "a": "P1",
    "type": "LEARNING_MARKER",
    "data": {
        "marker_id": "lm-1",
        "label": "Should I have bolted the creature instead?",
        "category": "decision_review",
        "created_at": "2026-02-21T10:30:00Z"
    }
}
```

**Data Fields:**

- `marker_id` — Unique marker identifier (`lm-` prefix + sequential number)
- `label` — Player-written description or question
- `category` — One of: `"decision_review"`, `"mistake"`, `"turning_point"`, `"interesting_interaction"`, `"sideboard_note"`, `"general"`
- `created_at` — ISO 8601 timestamp when marker was placed

**Note:** This event does not affect game state or deterministic replay. It is skipped during replay execution.

---

#### DRAW Event (v1.5.0+)

Recorded when a card is drawn from the library:

```json
{
    "i": 1,
    "t": "T0.PREGAME",
    "a": "SYS",
    "type": "DRAW",
    "data": {
        "obj": "c1",
        "card_name": "Lightning Bolt",
        "from": "P1:library",
        "to": "P1:hand",
        "pos": "top",
        "visibility": "private"
    }
}
```

**Data Fields:**

- `obj` — Object ID of the card drawn
- `card_name` — Human-readable card name
- `from` — Source zone (always `<Player>:library`)
- `to` — Destination zone (always `<Player>:hand`)
- `pos` — Position drawn from (`"top"`)
- `visibility` — Whether the draw is public or private

---

#### GAME_START Event (v1.5.0+)

Recorded as the first event (index 0) to capture game initialization:

```json
{
    "i": 0,
    "t": "T0.PREGAME",
    "a": "SYS",
    "type": "GAME_START",
    "data": {
        "players": ["P1", "P2", "P3", "P4"],
        "game_type": "Constructed",
        "first_player": "P1"
    }
}
```

**Data Fields:**

- `players` — Array of player IDs participating in the game
- `game_type` — Game format (e.g., `"Constructed"`, `"Commander"`)
- `first_player` — Player ID who takes the first turn

---

#### PLAY_LAND Event

Recorded when a player plays a land for the turn:

```json
{
    "i": 3,
    "t": "T1.MP1:0",
    "a": "P1",
    "type": "PLAY_LAND",
    "data": {
        "card": "c1",
        "card_name": "Mountain",
        "player": "P1"
    }
}
```

**Data Fields:**

- `card` — Card ID of the land being played
- `card_name` — Human-readable card name
- `player` — Player who played the land (v1.5.0+)

---

#### ACTIVATE Event

Recorded when a player activates an ability on a permanent or card:

```json
{
    "i": 30,
    "t": "T3.MP1:2",
    "a": "P1",
    "type": "ACTIVATE",
    "data": {
        "card": "c12",
        "card_name": "Birthing Pod",
        "ability": "{1}{G/P}, {T}, Sacrifice a creature: Search...",
        "controller": "P1",
        "cost": {},
        "targets": [],
        "choices": {}
    }
}
```

**Data Fields:**

- `card` — Card ID of the source permanent
- `card_name` — Human-readable card name
- `ability` — Text description of the activated ability (v1.5.0+)
- `controller` — Player activating the ability (v1.5.0+)
- `cost` — Costs paid (mana, tap, sacrifice, etc.)
- `targets` — Array of targets chosen
- `choices` — Other choices made

---

#### TRIGGER Event

Recorded when a triggered ability goes on the stack:

```json
{
    "i": 31,
    "t": "T3.MP1:3",
    "a": "SYS",
    "type": "TRIGGER",
    "data": {
        "source": "c12",
        "source_name": "Rhystic Study",
        "trigger": "Whenever an opponent casts a spell, you may pay {1}. If you don't, that player draws a card.",
        "controller": "P1"
    }
}
```

**Data Fields:**

- `source` — Card ID of the source permanent
- `source_name` — Human-readable card name (v1.5.0+)
- `trigger` — Text of the trigger condition (v1.5.0+)
- `controller` — Player who controls the triggered ability

---

#### TAP Event

Recorded when a permanent taps or untaps:

```json
{
    "i": 20,
    "t": "T2.MP1:1",
    "a": "SYS",
    "type": "TAP",
    "data": {
        "obj": "c5",
        "card_name": "Forest",
        "tapped": true
    }
}
```

**Data Fields:**

- `obj` — Object ID of the permanent
- `card_name` — Human-readable card name
- `tapped` — `true` if the permanent is now tapped, `false` if untapped

---

#### COUNTERS Event

Recorded when counters are added to or removed from a permanent:

```json
{
    "i": 35,
    "t": "T4.MP1:2",
    "a": "SYS",
    "type": "COUNTERS",
    "data": {
        "obj": "c12",
        "card_name": "Luminarch Aspirant",
        "counter_type": "+1/+1",
        "delta": 1,
        "new_total": 3
    }
}
```

**Data Fields:**

- `obj` — Object ID of the permanent
- `card_name` — Human-readable card name (v1.5.0+)
- `counter_type` — Type of counter (e.g., `"+1/+1"`, `"loyalty"`, `"charge"`)
- `delta` — Change amount (positive = added, negative = removed)
- `new_total` — New counter total on the permanent

---

#### DECLARE_ATTACKERS Event

Recorded when a player declares attacking creatures:

```json
{
    "i": 40,
    "t": "T3.COMBAT:1",
    "a": "P1",
    "type": "DECLARE_ATTACKERS",
    "data": {
        "attackers": {
            "c12": "P2",
            "c15": "P3"
        }
    }
}
```

**Data Fields:**

- `attackers` — Map of attacker card IDs to their attack targets (player IDs or planeswalker card IDs)

---

#### DECLARE_BLOCKERS Event

Recorded when a player declares blocking creatures:

```json
{
    "i": 41,
    "t": "T3.COMBAT:2",
    "a": "P2",
    "type": "DECLARE_BLOCKERS",
    "data": {
        "blockers": {
            "c30": ["c12"]
        }
    }
}
```

**Data Fields:**

- `blockers` — Map of blocker card IDs to arrays of attacker card IDs they are blocking

---

#### DISCARD Event

Recorded when a player discards a card:

```json
{
    "i": 55,
    "t": "T3.END:1",
    "a": "P1",
    "type": "DISCARD",
    "data": {
        "obj": "c7",
        "card_name": "Cancel",
        "from": "P1:hand",
        "to": "P1:graveyard",
        "forced": false
    }
}
```

**Data Fields:**

- `obj` — Object ID of the discarded card
- `card_name` — Human-readable card name
- `from` — Source zone (always `<Player>:hand`)
- `to` — Destination zone (typically `<Player>:graveyard`)
- `forced` — `true` if discarded by an opponent's effect, `false` for hand-size cleanup or voluntary

---

#### PASS_PRIORITY Event

Recorded when a player passes priority without acting:

```json
{
    "i": 7,
    "t": "T1.MP1:2",
    "a": "P1",
    "type": "PASS_PRIORITY",
    "data": {}
}
```

**Data Fields:**

- *(No data fields — empty object)*

---

#### CHOOSE Event

Recorded when a player makes a choice (mode selection, target assignment, etc.):

```json
{
    "i": 32,
    "t": "T3.MP1:3",
    "a": "P1",
    "type": "CHOOSE",
    "data": {
        "choice_type": "mode",
        "options": ["Destroy target artifact", "Destroy target enchantment"],
        "chosen": "Destroy target artifact",
        "source": "c14",
        "source_name": "Reclamation Sage"
    }
}
```

**Data Fields:**

- `choice_type` — Type of choice (e.g., `"mode"`, `"target"`, `"order"`, `"distribution"`)
- `options` — Available options
- `chosen` — The option selected
- `source` — Card ID of the source requiring the choice
- `source_name` — Human-readable source card name

---

#### STATE_BASED Event

Recorded when a state-based action occurs (e.g., creature dies from lethal damage):

```json
{
    "i": 80,
    "t": "T5.COMBAT:9",
    "a": "SYS",
    "type": "STATE_BASED",
    "data": {
        "action": "destroy",
        "reason": "lethal_damage",
        "obj": "c24",
        "card_name": "Elvish Mystic"
    }
}
```

**Data Fields:**

- `action` — What happened (`"destroy"`, `"sacrifice"`, `"exile"`, `"discard_to_hand_size"`)
- `reason` — Why the SBA triggered (`"lethal_damage"`, `"zero_toughness"`, `"legend_rule"`, `"deathtouch"`)
- `obj` — Object ID affected
- `card_name` — Human-readable card name

> **Note:** Most SBA results also produce a corresponding `MOVE` event. The `STATE_BASED` event provides the *reason* for the zone change.

---

#### RANDOM Event

Recorded for random game actions (shuffle, scry, coin flip, etc.):

```json
{
    "i": 25,
    "t": "T2.MP1:1",
    "a": "SYS",
    "type": "RANDOM",
    "data": {
        "action": "shuffle",
        "player": "P1",
        "reason": "tutor"
    }
}
```

**Data Fields:**

- `action` — Type of random event (`"shuffle"`, `"scry"`, `"coin_flip"`, `"reveal"`)
- `player` — Player whose library/selection is affected
- `reason` — What caused the random event (e.g., card name or game rule)
- `result` — Outcome for binary events (e.g., `"heads"`, `"tails"`)

---

## 8. Level 2 Learning View (views_l2)

### 8.1 L2 Unit Structure

Each L2 unit captures a decision context with before/after states:

```json
{
    "u": 5,
    "t_start": "T3.MP1:2",
    "t_end": "T3.MP1:4",
    "l1_range": [57, 89],
    "decision_events": [57],
    "before": {
        /* GameState */
    },
    "stack": [
        /* StackItem[] */
    ],
    "after": {
        /* GameState */
    },
    "annotations": {
        /* Learning data */
    }
}
```

**Fields:**

- `u` — Unit index
- `t_start` — Starting time marker
- `t_end` — Ending time marker
- `l1_range` — Range of L1 event indices `[start, end]`
- `decision_events` — Array of L1 event indices containing decisions
- `before` — Game state before actions
- `stack` — Stack contents during unit
- `after` — Game state after actions
- `annotations` — Analysis and teaching notes

---

### 8.2 Game State Snapshots

Both `before` and `after` fields contain complete game state:

```json
{
    "turn": 3,
    "phase": "MAIN_1",
    "step": "MAIN",
    "priority": "P1",
    "active_player": "P1",
    "players": {
        "P1": {
            "life": 18,
            "mana_pool": ["R", "R", "G"],
            "counters": {},
            "lands_played_this_turn": 1,
            "max_hand_size": 7
        },
        "P2": {
            "life": 20,
            "mana_pool": [],
            "counters": {},
            "lands_played_this_turn": 0,
            "max_hand_size": 7
        }
    },
    "zones": {
        "battlefield": ["c1", "c5", "c12", "c18", "c22"],
        "stack": [],
        "exile": ["c8"],
        "P1:hand": ["c3", "c7", "c11"],
        "P1:library": { "count": 52 },
        "P1:graveyard": ["c9", "c15"],
        "P2:hand": { "count": 6 },
        "P2:library": { "count": 50 },
        "P2:graveyard": ["c4"]
    },
    "objects": {
        "c1": {
            "card_ref": "Forest",
            "controller": "P1",
            "owner": "P1",
            "zone": "battlefield",
            "tapped": true,
            "flipped": false,
            "face_down": false,
            "counters": {},
            "damage_marked": 0,
            "attached_to": null,
            "notes": {}
        },
        "c5": {
            "card_ref": "Grizzly Bears",
            "controller": "P1",
            "owner": "P1",
            "zone": "battlefield",
            "tapped": false,
            "flipped": false,
            "face_down": false,
            "counters": {
                "+1/+1": 1
            },
            "damage_marked": 0,
            "attached_to": null,
            "notes": {}
        }
    }
}
```

---

### 8.3 Stack Items

The `stack` array in an L2 unit details what was on the stack:

```json
{
    "stack": "s1",
    "kind": "SPELL",
    "controller": "P1",
    "source": "c17",
    "card": "c17",
    "card_name": "Lightning Bolt",
    "targets": [
        {
            "slot": "any target",
            "obj": "c42",
            "name": "Grizzly Bears",
            "valid": true
        }
    ],
    "choices": {},
    "linked_decision_event": 57,
    "mana_paid": ["R"],
    "outcome": "resolved"
}
```

**Fields:**

- `stack` — Stack object ID
- `kind` — `"SPELL"`, `"ABILITY"`, or `"TRIGGER"`
- `controller` — Controlling player
- `source` — Source object
- `card` — Card object (for spells)
- `card_name` — Human-readable card name
- `targets` — Detailed target information
- `choices` — All choices made
- `linked_decision_event` — L1 event index of the decision
- `mana_paid` — Mana used to cast/activate
- `outcome` — Result: `"resolved"`, `"countered"`, `"fizzled"`, `"exiled"`

---

### 8.4 Target Information

Each target in a stack item includes:

```json
{
    "slot": "target creature",
    "obj": "c42",
    "name": "Grizzly Bears",
    "valid": true
}
```

**Fields:**

- `slot` — Target slot name from the spell/ability
- `obj` — Target object ID
- `name` — Human-readable name
- `valid` — Whether target was legal when chosen

---

### 8.5 Annotations

The `annotations` field provides learning context:

```json
{
    "decision_quality": null,
    "alternative_lines": [
        "Could have saved bolt for the flyer",
        "Attacking first would have been better"
    ],
    "key_moment": true,
    "teaching_notes": "This was a critical decision point. Player chose to remove the blocker before attacking."
}
```

**Fields:**

- `decision_quality` — Quality score (optional, format varies)
- `alternative_lines` — Array of alternative plays to consider
- `key_moment` — Boolean indicating if this was a pivotal moment
- `teaching_notes` — Free-form notes for learning

### 8.6 Learning Markers

Players can place **learning markers** at any point during a game to bookmark the current game state for later review. Markers are recorded as `LEARNING_MARKER` events in the L1 log and additionally collected in a top-level `learning_markers` array for quick navigation.

#### LEARNING_MARKER Event (L1)

Recorded inline in the event log when the player places a marker:

```json
{
    "i": 63,
    "t": "T4.MP1:1",
    "a": "P1",
    "type": "LEARNING_MARKER",
    "data": {
        "marker_id": "lm-1",
        "label": "Should I have bolted the creature instead of going face?",
        "category": "decision_review",
        "created_at": "2026-02-21T10:30:00Z"
    }
}
```

**Data Fields:**

- `marker_id` — Unique identifier for this marker (`lm-` prefix + sequential number)
- `label` — Player-written description or question about this moment
- `category` — Classification of the marker (see table below)
- `created_at` — ISO 8601 timestamp when the marker was placed

**Marker Categories:**

| Category | Description |
|----------|-------------|
| `decision_review` | Re-examine a specific decision made |
| `mistake` | Player believes they made an error here |
| `turning_point` | Moment the game shifted |
| `interesting_interaction` | Notable rules or card interaction |
| `sideboard_note` | Insight for sideboarding |
| `general` | Uncategorized bookmark |

**Note:** `LEARNING_MARKER` events do **not** affect game state or replay determinism. They are purely annotation events and are skipped during deterministic replay.

#### Top-Level `learning_markers` Summary

The top-level `learning_markers` array provides a quick-access index of all markers in the file, enriched with snapshot context so viewers can jump directly to bookmarked moments:

```json
{
    "learning_markers": [
        {
            "marker_id": "lm-1",
            "event_index": 63,
            "t": "T4.MP1:1",
            "player": "P1",
            "label": "Should I have bolted the creature instead of going face?",
            "category": "decision_review",
            "created_at": "2026-02-21T10:30:00Z",
            "snapshot": {
                "turn": 4,
                "phase": "MAIN_1",
                "active_player": "P1",
                "life_totals": { "P1": 16, "P2": 11 },
                "cards_in_hand": { "P1": 3, "P2": 5 },
                "battlefield_count": { "P1": 4, "P2": 3 },
                "stack_empty": true
            },
            "notes": "I had Bolt in hand. Used it on P2 face but the 4/4 ended up killing me two turns later."
        },
        {
            "marker_id": "lm-2",
            "event_index": 91,
            "t": "T6.COMBAT:3",
            "player": "P1",
            "label": "Bad blocks — lost two creatures for nothing",
            "category": "mistake",
            "created_at": "2026-02-21T10:35:00Z",
            "snapshot": {
                "turn": 6,
                "phase": "COMBAT",
                "active_player": "P2",
                "life_totals": { "P1": 8, "P2": 11 },
                "cards_in_hand": { "P1": 1, "P2": 3 },
                "battlefield_count": { "P1": 3, "P2": 5 },
                "stack_empty": false
            },
            "notes": ""
        }
    ]
}
```

**Summary Fields:**

- `marker_id` — Matches the L1 event `marker_id`
- `event_index` — L1 event index for direct lookup
- `t` — Time marker of the bookmarked moment
- `player` — Player who placed the marker
- `label` — Short description
- `category` — Marker category
- `created_at` — When marker was placed
- `snapshot` — Lightweight game state at the marked moment:
  - `turn` — Current turn number
  - `phase` — Current phase
  - `active_player` — Who is the active player
  - `life_totals` — Life totals for all players
  - `cards_in_hand` — Hand sizes
  - `battlefield_count` — Number of permanents per player
  - `stack_empty` — Whether the stack is empty
- `notes` — Extended free-form notes (can be added/edited after the game)

#### Usage Patterns

**During Game (Live Markers):**
Player presses a "bookmark" button during gameplay. The system records a `LEARNING_MARKER` L1 event at the current time marker and adds a summary entry to `learning_markers`.

**Post-Game Review (Retrospective Markers):**
Player reviews the replay and adds markers at interesting moments. These are only added to the `learning_markers` array (no L1 event inserted, to preserve log integrity). Retrospective markers have `event_index` pointing to the nearest L1 event.

**UI Recommendations:**
- Display markers as pins/flags on a game timeline
- Allow filtering by `category`
- Clicking a marker jumps to the full game state at that `event_index`
- Provide a summary panel listing all markers with labels
- Enable export of markers as a standalone study list

---

## 8.7 Event Log Key Name Change (v1.5.0)

Starting with v1.5.0, the event log key was renamed from `log_l1` to `events` at the top level of the replay file. Consumers should check for both keys for backward compatibility:

```
events = file.log_l1 || file.events || []
```

---

## 8.8 Pre-computed Statistics (v1.5.0+)

### Per-Turn Summary (`per_turn_summary`)

An array of per-turn statistics, indexed by turn. Each entry provides per-player stats for that turn:

```json
{
    "turn": 1,
    "active_player": "P1",
    "players": {
        "P1": {
            "lands_played": 1,
            "land_drop_rating": "good",
            "cards_drawn": 1,
            "spells_cast": 0,
            "abilities_activated": 0,
            "land_count": 1,
            "available_mana": 0,
            "life": 40,
            "cards_in_hand": 7,
            "creatures_on_battlefield": 0,
            "permanents_on_battlefield": 1,
            "damage_dealt": 0,
            "damage_taken": 0
        }
    }
}
```

**Per-Player Fields:**

| Field | Description |
|-------|-------------|
| `lands_played` | Number of lands played this turn |
| `land_drop_rating` | `"bad"` (0 lands), `"good"` (1 land), `"super"` (2+ lands) |
| `cards_drawn` | Cards drawn this turn |
| `spells_cast` | Spells cast this turn |
| `abilities_activated` | Abilities activated this turn |
| `land_count` | Total lands on battlefield at end of turn |
| `available_mana` | Available mana at end of turn |
| `life` | Life total at end of turn |
| `cards_in_hand` | Hand size at end of turn |
| `creatures_on_battlefield` | Creatures on battlefield at end of turn |
| `permanents_on_battlefield` | Total permanents on battlefield at end of turn |
| `damage_dealt` | Total damage dealt this turn |
| `damage_taken` | Total damage taken this turn |

### Game Summary (`game_summary`)

A single object with game-wide statistics per player:

```json
{
    "total_turns": 70,
    "duration_seconds": 8881,
    "winner": "P1",
    "win_condition": "unknown",
    "players": {
        "P1": {
            "total_cards_drawn": 39,
            "card_draw_rate": 0.56,
            "total_spells_cast": 24,
            "spell_velocity": 0.34,
            "total_abilities_activated": 31,
            "missed_land_drops": 4,
            "total_lands_played": 15,
            "peak_mana": 71,
            "total_damage_dealt": 230,
            "total_damage_received": 4,
            "total_creatures_played": 13,
            "starting_life": 40,
            "ending_life": 44,
            "life_delta": 4,
            "total_counters_placed": 0
        }
    }
}
```

**Top-level Fields:**

| Field | Description |
|-------|-------------|
| `total_turns` | Total number of turns in the game |
| `duration_seconds` | Game duration in seconds |
| `winner` | Winning player ID |
| `win_condition` | How the game was won |

**Per-Player Fields:**

| Field | Description |
|-------|-------------|
| `total_cards_drawn` | Total cards drawn during the game |
| `card_draw_rate` | Cards drawn per turn (average) |
| `total_spells_cast` | Total spells cast |
| `spell_velocity` | Spells cast per turn (average) |
| `total_abilities_activated` | Total abilities activated |
| `missed_land_drops` | Turns where no land was played |
| `total_lands_played` | Total lands played |
| `peak_mana` | Maximum available mana during the game |
| `total_damage_dealt` | Total damage dealt |
| `total_damage_received` | Total damage received |
| `total_creatures_played` | Total creatures played |
| `starting_life` | Life total at game start |
| `ending_life` | Life total at game end |
| `life_delta` | Net life change (ending - starting) |
| `total_counters_placed` | Total counters placed on permanents |

---

## 9. Random Seed

The `seed` field contains the random number generator seed:

```json
{
    "seed": 1734701432000
}
```

This enables deterministic replay of the game. With the same seed and the same initial state, the game can be replayed exactly.

---

## 10. Complete Example

Here's a minimal complete replay file:

```json
{
    "format": "mtg-replay",
    "version": "1.5.0",
    "spec_version": "1.5.0",
    "meta": {
        "game_id": "game-abc123",
        "timestamp": "2025-12-20T14:30:00Z",
        "game_type": "Constructed",
        "players": {
            "P1": { "name": "Alice" },
            "P2": { "name": "Bob" }
        },
        "winner": "P1",
        "turns": 5,
        "duration_seconds": 180
    },
    "seed": 1234567890,
    "card_index": {
        "Mountain": {
            "name": "Mountain",
            "cost": "",
            "type": "Basic Land — Mountain"
        },
        "Lightning Bolt": {
            "name": "Lightning Bolt",
            "cost": "{R}",
            "type": "Instant"
        }
    },
    "initial_state": {
        "turn": 0,
        "phase": "PREGAME",
        "players": {
            "P1": { "life": 20 },
            "P2": { "life": 20 }
        },
        "zones": {
            "P1:library": { "count": 60 },
            "P2:library": { "count": 60 }
        }
    },
    "events": [
        {
            "i": 0,
            "t": "T1.UP",
            "a": "SYS",
            "type": "PHASE_CHANGE",
            "data": {
                "phase": "UPKEEP",
                "active_player": "P1"
            }
        },
        {
            "i": 1,
            "t": "T1.MP1",
            "a": "SYS",
            "type": "PHASE_CHANGE",
            "data": {
                "phase": "MAIN_1",
                "active_player": "P1"
            }
        },
        {
            "i": 2,
            "t": "T1.MP1:0",
            "a": "P1",
            "type": "PLAY_LAND",
            "data": {
                "card": "c1"
            }
        }
    ],
    "views_l2": [
        {
            "u": 0,
            "t_start": "T1.MP1",
            "t_end": "T1.MP1:1",
            "l1_range": [1, 2],
            "decision_events": [2],
            "before": {
                "turn": 1,
                "phase": "MAIN_1",
                "players": {
                    "P1": { "life": 20, "mana_pool": [] },
                    "P2": { "life": 20, "mana_pool": [] }
                }
            },
            "stack": [],
            "after": {
                "turn": 1,
                "phase": "MAIN_1",
                "players": {
                    "P1": { "life": 20, "mana_pool": [] },
                    "P2": { "life": 20, "mana_pool": [] }
                },
                "zones": {
                    "battlefield": ["c1"]
                }
            },
            "annotations": {
                "decision_quality": null,
                "alternative_lines": [],
                "key_moment": false,
                "teaching_notes": ""
            }
        }
    ]
}
```

---

## 11. Reading Replay Files

### 11.1 Chronological Reading

To understand what happened in a game:

1. Read the **metadata** for context (who played, who won)
2. Check the **card_index** to understand what cards were in the game
3. Read **events** sequentially from start to finish
4. Use **time markers** to understand when events occurred
5. Follow **object IDs** to track specific cards through the game

### 11.2 Learning-Focused Reading

To analyze player decisions:

1. Browse **views_l2** array
2. Each unit represents a decision point
3. Compare **before** and **after** states to see changes
4. Review **stack** contents to understand what was being played
5. Check **annotations** for insights and alternatives
6. Use **linked_decision_event** to find the original L1 event

### 11.3 Specific Event Lookup

To find a specific event:

1. Use **time markers** to locate approximate position
2. Search for specific **event types**
3. Filter by **actor** (P1, P2, or SYS)
4. Follow **object IDs** to track specific cards

---

## 12. Common Patterns

### 12.1 Casting a Spell

Typical sequence:

1. `CAST` event (Player decision)
2. `PUT_ON_STACK` event (System places spell)
3. `PASS_PRIORITY` events (Players pass)
4. `RESOLVE` event (Spell resolves)
5. Various effect events (DAMAGE, MOVE, etc.)
6. `MOVE` event (Spell to graveyard)

### 12.2 Combat Damage

Typical sequence:

1. `PHASE_CHANGE` to COMBAT
2. `DECLARE_ATTACKERS` (Player decision)
3. `DECLARE_BLOCKERS` (Player decision)
4. Multiple `DAMAGE` events
5. Multiple `LIFE` events (if players damaged)
6. `STATE_BASED` events (if creatures die)
7. Multiple `MOVE` events (dead creatures to graveyard)

### 12.3 Triggered Abilities

Typical sequence:

1. Triggering event occurs
2. `TRIGGER` event (System)
3. `PUT_ON_STACK` event
4. `CHOOSE` events (if targets/modes needed)
5. Priority passes
6. `RESOLVE` event
7. Effect events

---

## 13. Validation Rules

A well-formed replay file must satisfy:

1. **Format version** is supported
2. **All object IDs** referenced in events exist in card_index or are valid player IDs
3. **Event indices** are sequential starting from 0
4. **Time markers** progress forward (except for simultaneous events)
5. **L2 l1_range** values reference valid L1 event indices
6. **Zone changes** reference valid zone names
7. **Player IDs** match those in metadata

---

## 14. Best Practices for Reading

### 14.1 Understanding Actor Field

- `"P1"` or `"P2"` — Deliberate player action, can be analyzed for quality
- `"SYS"` — Automatic game rule, not a strategic choice

### 14.2 Following the Stack

When the stack is active:

1. Look for `PUT_ON_STACK` events to see what's added
2. Track multiple stack objects by their `stack` IDs
3. `RESOLVE` events process from top (last added)
4. Stack empties when no more objects remain

### 14.3 Tracking Mana

Player state includes `mana_pool`:

- Empty array `[]` means no mana
- Array of strings `["R", "G", "G"]` shows available mana
- Mana empties at step/phase boundaries

### 14.4 Hidden Information

Some information is hidden during gameplay:

- Library counts may show `{"count": 52}` instead of card list
- Hand contents may show `{"count": 6}` for opponent
- Face-down cards have `face_down: true` in object state

---

## 15. Glossary

**Actor** — The entity performing an action (player or system)

**Decision Event** — An event representing a strategic player choice

**Event Index** — Sequential number identifying an event's position

**L1** — Level 1 event log, the authoritative game record

**L2** — Level 2 learning view, derived decision contexts

**Object ID** — Unique identifier for a game object

**Stack ID** — Unique identifier for an object on the stack

**Time Marker** — Human-readable timestamp for when an event occurs

**Zone** — A game area where cards can exist (battlefield, hand, etc.)

---

## 16. Learning Helper Statistics

The replay format enables calculation of key performance indicators (KPIs) for analyzing player decision-making and game efficiency. These statistics help identify strengths, weaknesses, and learning opportunities.

### 16.1 Card Efficiency Metrics

**Card Draw/Turn Ratio**
- **Definition:** Average number of cards drawn per turn
- **Formula:** `(Total cards drawn from L1 DRAW events) / (Total turns played)`
- **Calculation:** Count all DRAW events (excluding opening hands), divide by final turn number
- **Interpretation:** Higher values indicate better card advantage generation
- **Data Requirements:**
  - ✅ **Available Now:** DRAW events from L1 log
  - ✅ **Available Now:** Turn numbers from turn_start events
  - ⚠️ **Context:** Baseline is 1 card/turn; ≥2.0 is excellent, <0.8 is poor
- **Rating Scale:**
  - ≥ 2.0: 🌟 Excellent (multiple extra cards/turn)
  - 1.5–1.99: 🟢 Good (consistent extra cards)
  - 0.8–1.49: 🟡 Normal
  - < 0.8: 🔴 Poor (falling behind)

**Most Played Cards Efficiency**
- **Definition:** Ratio of cards cast vs. opportunities to cast them
- **Formula:** `(Times card was cast) / (Times card was available to cast)`
- **Calculation:** 
  - Count CAST events for each card
  - Count turns where card was in hand with sufficient mana
  - Calculate ratio per card
- **Interpretation:** Low ratios may indicate suboptimal card selection or missed opportunities
- **Data Requirements:**
  - ✅ **Available Now:** CAST events for each card from L1 log
  - ✅ **Available Now:** Cumulative hand zone tracking from MOVE events
  - ✅ **Requires Card DB:** Card mana costs (from `extendedCardInfo.cardfaces[0].mana_cost`)
  - ⚠️ **Limitation:** Cannot determine summoning sickness for creatures yet
  - ⚠️ **Limitation:** Cannot detect if instant-speed cards were deliberately held
- **Rating Scale:**
  - ≥ 0.8: 🟢 Excellent (played nearly every opportunity)
  - 0.5–0.79: 🟡 Moderate (sometimes held for timing)
  - 0.2–0.49: 🟠 Low (often not cast when available)
  - < 0.2: 🔴 Very Low (consider cutting from deck)
- **Special Cases:**
  - Counterspells/Interaction: <0.5 is normal (held for threats)
  - Win Conditions: Low efficiency acceptable if card wins when cast

### 16.2 Mana Utilization Metrics

**Land Drop Rating**
- **Definition:** Whether player made their land drop this turn
- **Formula:** `if lands_played == 0: "bad", if == 1: "good", if >= 2: "super"`
- **Data Requirements:**
  - ✅ **Available Now:** `lands_played_this_turn` counter from player state
  - ⚠️ **Enhancement:** Could compare cumulative lands to `turn_number` to detect deficit
- **Rating Scale:**
  - 0 lands: 🔴 Bad (missed drop, falling behind)
  - 1 land: 🟢 Good (on curve)
  - ≥ 2 lands: 🌟 Super (accelerated, ahead on curve)

**Available Mana**
- **Definition:** Total mana pool by color available this turn
- **Display:** `{W: 2, U: 1, B: 0, R: 0, G: 3, C: 1} = 7 total`
- **Calculation:** 
  - **Method 1 (Preferred):** Extract from `mana_pool` state or `mana_tap` events with `manaProduced`
  - **Method 2 (Estimation):** Count lands on battlefield and estimate colors from names
  - Add mana dorks/rocks if tagged in card database
- **Data Requirements:**
  - ✅ **Available Now:** Cumulative battlefield zone tracking
  - ✅ **Requires Card DB:** `is_manaproducing` + `mana_producing_colors` fields
  - ⚠️ **Limitation:** Estimation only works for recognized land names (basic, shock, fetch, dual)
  - ❌ **Not Tracked Yet:** Actual `mana_tap` events with `manaProduced` values

**Cast Options in Hand**
- **Definition:** Which cards in hand can be cast with current mana
- **Calculation:**
  - Parse each card's mana cost (e.g., `{2}{W}{U}` → generic: 2, W: 1, U: 1)
  - Check if colored requirements met: `mana_pool[color] >= cost[color]` for each color
  Spell Velocity**
- **Definition:** Mean number of spells cast per turn
- **Formula:** `(Total CAST events for player) / (Total turns)`
- **Calculation:** Count all CAST events where actor is the player, divide by turns
- **Interpretation:** Higher values indicate faster, more aggressive gameplay
- **Data Requirements:**
  - ✅ **Available Now:** CAST events from L1 log
  - ✅ **Available Now:** Turn count from turn_start events
  - ⚠️ **Enhancement:** Compare to expected velocity based on deck average CMC
- **Archetype Expectations:**
  - ≥ 3.0: Storm, Combo, Spellslinger
  - 2.0–2.99: Tempo, Prowess, Cantrip-heavy
  - 1.0–1.99: Midrange, Typical Commander
  - 0.5–0.99: Control (early game)
  - < 0.5: Mana-screwed or extremely slow deck

**Effective Turn to Boardstate Impact**
- **Definition:** Turn where player establishes meaningful board presence
- **Formula:** First turn where `BoardPresenceScore >= Threshold`
- **Calculation:**
  - `BoardPresenceScore = (creature power × 1.0) + (creature toughness × 0.3) + (planeswalker loyalty × 0.8) + (game-enders × 5.0)`
  - Threshold varies by archetype (e.g., 6 for aggro, 10 for midrange)
  - Identify first turn exceeding threshold
- **Data Requirements:**
  - ✅ **Available Now:** Cumulative battlefield tracking
  - ✅ **Requires Card DB:** Creature power/toughness
  - ✅ **Requires Card DB:** Planeswalker loyalty values
  - ❌ **Not Available:** "Game-ending permanent" tagging
  - ⚠️ **Limitation:** Cannot fully assess non-combat threats (combo pieces, engines)
- **Rating by Archetype:**
  - Aggro (threshold 6): Fast if turn ≤2, On Curve if turn 3, Slow if turn ≥4
  - Tempo (threshold 8): Fast if turn ≤3, On Curve if turn 4, Slow if turn ≥5
- **Data Requirements:**
  - ✅ **Available Now:** `lands_played_this_turn` counter from player state
  - ✅ **Available Now:** Hand contents from cumulative zone tracking
  - ✅ **Requires Card DB:** Land type identification (though basic patterns can work)
  - ⚠️ **Limitation:** Cannot detect deliberate holds (e.g., holding fetchland for shuffle)
  - ⚠️ **Limitation:** Multiple land drop abilities (e.g., Azusa) not currently tracked
  - Midrange (threshold 10): Fast if turn ≤4, On Curve if turn 5, Slow if turn ≥6
  - Control (threshold 12): Fast if turn ≤5, On Curve if turn 6-7, Slow if turn ≥8

**Critical Turn Number**
- **Definition:** Turn where game outcome became effectively decided
- **Formula:** `min(CriticalTurn_Life, CriticalTurn_Board, CriticalTurn_Concede)`
- **Calculation Methods:**
  - **Method 1 (Life Swing):** Turn with largest `|LifeTotal(you,t) - LifeTotal(you,t-1)| + |LifeTotal(opp,t) - LifeTotal(opp,t-1)|`
  - **Method 2 (Board Dominance):** Turn with largest `|BoardPresenceScore(you,t) - BoardPresenceScore(opp,t)|` change
  - **Method 3 (Game End):** Turn before concession or turn of win_condition event
- **Data Requirements:**
  - ✅ **Available Now:** Life total changes from DAMAGE events
  - ✅ **Available Now:** Concession turn from `conceded` in meta
  - ✅ **Available Now:** Win condition turn from `win_condition` event
  - ⚠️ **Partial:** Board dominance calculation (see Effective Turn requirements)
- **Usage:** Focal point for deep analysis — what decisions led here? Were there alternatives 2-3 turns earlier?
  - ✅ **Requires Card DB:** Flash keyword detection
  - ⚠️ **Limitation:** Cannot detect situational reasons to hold mana (e.g., waiting for specific threat)
  - ⚠️ **Context Dependency:** Tap-out decks (aggro/ramp) naturally have higher waste
- **Rating Scale:**
  - 0–1 per turn: 🟢 Efficient
  - 2–3 per turn: 🟡 Moderate waste
  - 4–5 per turn: 🟠 High waste
  - >5 per turn: 🔴 Excessive waste

**Mana Color Coverage**

*A. Commander Color Coverage*
- **Definition:** Percentage of commander colors currently available
- **Formula:** `|AvailableColors ∩ CommanderColors| / |CommanderColors|`
- **Data Requirements:**
  - ✅ **Requires Decklist:** Commander color identity
  - ✅ **Available Now:** Available colors from mana pool
- **Rating Scale:**
  - 100%: 🟢 Full (all colors accessible)
  - 50–99%: 🟡 Partial (some colors missing)
  - <50%: 🔴 Poor (major color screw)

*B. Deck Spell Castability Percentage*
- **Definition:** What % of deck spells have color requirements met (ignoring generic mana)
- **Formula:** Count spells where colored mana requirements ≤ available colored mana, divide by total spells
- **Calculation:**
  - For each non-land card in deck
  - Check only colored mana (ignore generic/colorless)
  - E.g., `{4}{W}{W}` is castable if W ≥ 2, regardless of total mana
- **Data Requirements:**
  - ✅ **Requires Decklist:** All cards with counts
  - ✅ **Requires Card DB:** Mana cost for each card to extract colored requirements
  - ⚠️ **Note:** This measures COLOR FIXING quality, not mana quantity
- **Rating Scale:**
  - ≥90%: 🟢 Excellent (mana base covers almost everything)
  - 70–89%: 🟡 Adequate (some spells color-locked)
  - <70%: 🔴 Deficient (significant color problems)

### 16.3 Game Tempo Metrics

**AverageData Requirements Summary

**Computation Requirements Table:**

| Statistic | Log Events Only | Card DB Required | Decklist Required | Current Status |
|-----------|----------------|------------------|-------------------|----------------|
| Land Drop Rating | ✅ Yes | No | No | ✅ Fully Computable |
| Card Draw Efficiency | ✅ Yes | No | No | ✅ Fully Computable |
| Spell Velocity | ✅ Yes | No | No | ✅ Fully Computable |
| Critical Turn Number | ✅ Yes | No | No | ✅ Fully Computable |
| Missed Land Drops | ✅ Yes | ⚠️ Helpful | No | ✅ Basic Version Works |
| Available Mana | ⚠️ Partial | ✅ Yes | No | ⚠️ Estimation Only |
| Cast Options in Hand | No | ✅ Yes | No | ✅ With Card DB |
| Unused Mana at Opponent Turn | ⚠️ Partial | ✅ Yes | No | ⚠️ With Card DB + Estimates |
| Commander Color Coverage | ⚠️ Partial | No | ✅ Yes | ⚠️ Requires Decklist |
| Deck Spell Castability % | No | ✅ Yes | ✅ Yes | ⚠️ Requires Both |
| Most Played Cards Efficiency | ✅ Yes | ✅ Yes | No | ⚠️ Requires Card DB |
| Effective Turn to Boardstate Impact | ⚠️ Partial | ✅ Yes | No | ❌ Partial (P/T only) |

**Data Sources:**
- **L1 Event Log:** CAST, DRAW, MOVE, DAMAGE, turn_start events (authoritative)
- **Player State:** `mana_pool`, `lands_played_this_turn`, `life_total` counters
- **Card Database:** Mana costs, power/toughness, types, keywords (Flash), mana production
- **Decklist:** Commander color identity, all cards with counts
- **L2 Views:** Decision context and turn summaries

**Current Limitations:**
1. ❌ **Mana Tap Events:** `mana_tap` events with `manaProduced` values not consistently logged
2. ❌ **Permanent Tags:** Game-ending permanents, combo pieces not tagged
3. ❌ **Ability Tracking:** Flash, multiple land drops, alternative costs not fully tracked
4. ❌ **Summoning Sickness:** Cannot determine if creatures could attack/tap abilities
5. ⚠️ **Hand Visibility:** Relies on cumulative tracking (may have gaps if player hides information)

### 16.6 Implementation Guidelines

**Best Practices:**
- Calculate statistics for both players separately
- Compare against deck archetype averages
- Consider game format and matchup context
- Correlate with game outcome (win/loss)
- Highlight outlier games for detailed review
- **Graceful Degradation:** Show simplified statistics when card DB unavailable

**Examples of Graceful Degradation:**
- **Without Card DB:**
  - Show land drop rating, card draw rate, spell velocity, critical turn
  - Estimate available mana from basic land names only
  - Cannot show cast options or color coverage
- **Without Decklist:**
  - Cannot show commander color coverage or deck castability %
  - All other statistics still available if card DB present

**Example Use Cases:**
- Identify consistent mana problems → mulligan strategy adjustment
- Low spell velocity → deck curve optimization
- High unused mana → add more instant-speed interaction
- Low card efficiency → deck building: replace underperforming cards
- Critical turn at turn 3 → analyze turns 1-2 for better play
- **Interpretation:** Helps identify key decision points for analysis

### 16.4 Strategic Efficiency Metrics

**Missed Land Drops**
- **Definition:** Number of turns without playing a land when lands were available
- **Formula:** `Count of turns (land in hand AND lands_played_this_turn == 0)`
- **Calculation:**
  - Each turn, check player hand for land cards
  - Verify lands_played_this_turn counter
  - Count missed opportunities
- **Interpretation:** Indicates unforced play errors or mana flood management decisions

### 16.5 Implementation Guidelines

**Data Sources:**
- Extract from L1 event log for authoritative data
- Use L2 views for decision context
- Reference card_index for card types and costs
- Access player state for mana_pool and counters

**Best Practices:**
- Calculate statistics for both players separately
- Compare against deck archetype averages
- Consider game format and matchup context
- Correlate with game outcome (win/loss)
- Highlight outlier games for detailed review

**Example Use Cases:**
- Identify consistent mana problems → mulligan strategy adjustment
- Low spell velocity → deck curve optimization
- High unused mana → add more instant-speed interaction
- Low card efficiency → deck building: replace underperforming cards

### 16.7 Identifying Major Impact Turns and Plays

To help players focus on the most important moments in a game, identify and highlight turns and individual plays that had significant impact on the game outcome. This enables efficient review and learning from key decision points.

#### Turn-Level Impact Scoring

**High Impact Turn Indicators:**

1. **Life Swing Magnitude**
   - **Formula:** `ΔLife = |Life(t) - Life(t-1)| for all players`
   - **Threshold:** Sum of life changes ≥ 10 = High Impact
   - **Example:** Turn where 15 damage dealt = Major impact turn
   - **Detection:** Sum absolute value of all LIFE events in turn

2. **Board State Swing**
   - **Formula:** `ΔBoard = |BoardScore(you,t) - BoardScore(you,t-1)| + |BoardScore(opp,t) - BoardScore(opp,t-1)|`
   - **Threshold:** Change ≥ 8 = High Impact
   - **Example:** Turn where 3 creatures die and 2 are played
   - **Detection:** Compare cumulative board presence scores

3. **Critical Turn Match**
   - If turn equals Critical Turn Number (from 16.3) → Highest Impact
   - This turn decided the game outcome
   - Should be prominently marked

4. **Game-Ending Events**
   - Turn contains `win_condition` event → Maximum Impact
   - Turn has `conceded` in metadata → Maximum Impact
   - Last turn of game → High Impact

5. **Multiple High-Value Plays**
   - Turn with 3+ significant events (see Play-Level scoring below)
   - "Combo turn" indicator
   - Dense decision-making turn

**Turn Impact Rating Scale:**

```
Critical (Score 10):  Critical Turn or Game-Ending Turn
Major (Score 7-9):    Life swing ≥15 OR Board swing ≥12
High (Score 4-6):     Life swing 10-14 OR Board swing 8-11 OR 3+ significant plays
Medium (Score 2-3):   Life swing 5-9 OR Board swing 4-7 OR 2 significant plays
Low (Score 0-1):      Routine development, no major swings
```

#### Play-Level Impact Scoring

**High Impact Play Indicators:**

1. **Removal/Destruction**
   - **Detection:** MOVE event with `from: "battlefield"`, `to: "graveyard"` or `"exile"`
   - **Impact Score:** Based on destroyed permanent's value
     - Creature with power ≥4: Score 3
     - Planeswalker: Score 4
     - Enchantment/Artifact affecting multiple cards: Score 3-5
   - **Enhancement:** Check if permanent was key threat (highest power/toughness)

2. **Board Wipes**
   - **Detection:** Single event causing 3+ permanents to leave battlefield
   - **Impact Score:** 8-10 (extremely high)
   - **Example:** Wrath of God removes 5 creatures

3. **Large Life Swings**
   - **Detection:** Single LIFE event with `|delta| ≥ 5`
   - **Impact Score:** Scale with magnitude
     - 5-9 life: Score 3
     - 10-14 life: Score 5
     - ≥15 life: Score 7

4. **Combat - Major Attacks**
   - **Detection:** DECLARE_ATTACKERS with 3+ attackers OR total attacking power ≥10
   - **Impact Score:** 4-6
   - **Enhancement:** Check if attack was decisive (reduced opponent below safe threshold)

5. **Card Draw Bursts**
   - **Detection:** Multiple DRAW events in single turn (3+ cards drawn beyond normal draw)
   - **Impact Score:** 3-4
   - **Example:** Sphinx's Revelation for 5 cards

6. **Expensive Spells**
   - **Detection:** CAST event with mana cost ≥6
   - **Impact Score:** 2-4 (high CMC often means high impact)
   - **Enhancement:** Check card type (creatures, planeswalkers score higher)

7. **Stack Interactions**
   - **Detection:** Counterspell on expensive/critical spell
   - **Impact Score:** Based on countered spell's cost
   - **Formula:** `CounterspellImpact = CounteredSpellCMC × 0.5`

8. **First Threat Resolved**
   - **Detection:** First creature with power ≥3 entering battlefield (Effective Turn)
   - **Impact Score:** 4
   - **Context:** Establishing board presence

**Play Impact Rating Scale:**

```
Game-Winning (Score 9-10):  Play directly wins the game
Critical (Score 7-8):       Board wipe, major threat removal, game-saving counter
High (Score 4-6):           Significant creature/planeswalker, large life swing, key removal
Medium (Score 2-3):         Moderate threat, card advantage, expensive spell
Low (Score 0-1):            Routine play, minor creature, basic land
```

#### Implementation Guidance for UI Display

**Turn View Highlighting:**

```typescript
interface TurnHighlight {
    turnNumber: number;
    impactScore: number;  // 0-10
    impactRating: "critical" | "major" | "high" | "medium" | "low";
    reasons: string[];    // ["Life swing: 15", "Board wipe", "Critical turn"]
    iconBadge: string;    // "⚠️" for critical, "🔥" for major, etc.
    borderColor: string;  // "#dc3545" for critical, "#ff9800" for major
}
```

**Recommended Visual Indicators:**

- **Critical Turns:** Red border, "⚠️" badge, expanded by default
- **Major Impact Turns:** Orange border, "🔥" badge, highlighted header
- **High Impact Turns:** Yellow border, "⭐" badge
- **Medium/Low:** Standard styling, collapsed by default

**Play-Level Display:**

```typescript
interface PlayHighlight {
    eventIndex: number;
    playDescription: string;  // "Cast Wrath of God (board wipe)"
    impactScore: number;
    impactCategory: "board-wipe" | "removal" | "threat" | "life-swing" | "card-advantage";
    tooltipDetails: string;   // Extended explanation
}
```

**Turn Summary with Impact:**

```
Turn 5 [⚠️ CRITICAL TURN - Game Decided]
├─ Life Swing: -12 (opponent)
├─ 🔥 Cast Craterhoof Behemoth (game-winning threat)
└─ Impact Score: 10/10
```

#### Automatic Event Filtering

**"Show Only Important Moments" Feature:**

Filter display to turns with `impactScore ≥ 4`, allowing players to:
- Skip routine development turns
- Focus on pivotal decisions
- Quickly review game-deciding moments

**Example Filter Implementation:**

```typescript
const majorImpactTurns = parsedGame.turns.filter(turn => {
    const score = calculateTurnImpactScore(turn, parsedGame);
    return score >= 4;  // Show only High, Major, or Critical turns
});
```

#### Detection Algorithm Summary

```typescript
function calculateTurnImpactScore(turn: GameTurn, game: ParsedGame): number {
    let score = 0;
    
    // Check if critical turn
    if (turn.turnNumber === game.criticalTurn) {
        return 10;  // Maximum impact
    }
    
    // Check for game-ending events
    if (turn.events.some(e => e.type === "win_condition")) {
        return 10;
    }
    
    // Calculate life swing
    const lifeSwing = calculateLifeSwing(turn.events);
    if (lifeSwing >= 15) score += 5;
    else if (lifeSwing >= 10) score += 3;
    else if (lifeSwing >= 5) score += 1;
    
    // Calculate board swing
    const boardSwing = calculateBoardSwing(turn, getPreviousTurn(turn));
    if (boardSwing >= 12) score += 5;
    else if (boardSwing >= 8) score += 3;
    else if (boardSwing >= 4) score += 1;
    
    // Count significant plays
    const significantPlays = countSignificantPlays(turn.events);
    score += Math.min(significantPlays, 3);  // Cap at +3 for multiple plays
    
    return Math.min(score, 10);  // Cap at 10
}
```

#### User Experience Recommendations

1. **Timeline View:** Show impact scores as vertical bars (higher = more important)
2. **Color Coding:** Use heat map colors (red = critical, orange = major, yellow = high)
3. **Expandable Details:** Click high-impact turn to see breakdown of why it's important
4. **Quick Navigation:** "Jump to next critical moment" button
5. **Impact Summary:** Game overview showing all high-impact turns in sequence
6. **Comparison View:** Side-by-side "What happened" vs "What could have happened"

#### Example Impact Annotation

```json
{
    "turn": 5,
    "impactScore": 9,
    "impactRating": "critical",
    "impactReasons": [
        "Life swing: 15 damage dealt to opponent",
        "Board wipe: 4 creatures destroyed",
        "Game-winning threat deployed: Craterhoof Behemoth"
    ],
    "keyMoments": [
        {
            "eventIndex": 67,
            "description": "Cast Wrath of God",
            "impact": "board-wipe",
            "score": 8
        },
        {
            "eventIndex": 72,
            "description": "Cast Craterhoof Behemoth",
            "impact": "game-winning-threat",
            "score": 9
        }
    ]
}
```

This impact scoring system enables replay viewers to quickly identify and focus on the turns and plays that mattered most, improving learning efficiency and game analysis quality.

---

## 17. Commander Decklist Notation

A companion specification defines the JSON format for recording Commander format
decklists. The decklist format extends the replay notation with four deck sections,
artwork identification, deck mechanic role labeling, and a strategic rule set for
opening hand evaluation and combo tracking.

### 17.1 Companion Specification

See **[Commander Decklist Notation](./commander-decklist-spec.md)** for the full
specification, including:

- Deck sections: `commander`, `main`, `sideboard`, `maybeboard`
- Card entry fields: `edition` + `collector_number` for artwork identification,
  `primary_mechanic` and `additional_mechanics` for strategic roles
- Mulligan scoring model with per-category card values and per-round thresholds
- Combo and anti-synergy (`dont_combos`) declarations

### 17.2 Inline Decklist in Replay Files (v1.6.0+)

A replay file may embed decklist data directly using the optional top-level
`decklist` field. The value is a map from player ID to a full `CommanderDecklist`
object (as defined in the companion spec):

```json
{
    "format": "mtg-replay",
    "version": "1.6.0",
    "decklist": {
        "P1": {
            "format": "mtg-commander-decklist",
            "version": "1.0.0",
            "meta": { "deck_name": "Atraxa Superfriends", "format": "Commander" },
            "commander": [ /* ... */ ],
            "main": [ /* ... */ ],
            "sideboard": [],
            "maybeboard": [],
            "deck_rules": { /* ... */ }
        }
    }
}
```

When both `deck_link` (in `meta.players`) and an inline `decklist` entry are present
for the same player, consumers should prefer the inline `decklist`.

---

## 18. Version History

| Version | Date       | Changes               |
| ------- | ---------- | --------------------- |
| 1.0.0   | 2025-12-20 | Initial specification |
| 1.1.0   | 2026-02-08 | Added `win_condition`, `conceded`, `deck_name` in meta; `deck_hash` algorithm; `RESOURCES` event; `card_name` in CAST, MOVE, PUT_ON_STACK, DAMAGE events |
| 1.2.0   | 2026-02-12 | Added Learning Helper Statistics section with KPIs for game analysis |
| 1.2.1   | 2026-02-14 | Added section 16.7 on identifying major impact turns and plays for UI display highlighting |
| 1.3.0   | 2026-02-21 | Added `LEARNING_MARKER` event type and `learning_markers` top-level section for player-placed bookmarks |
| 1.4.0   | 2026-02-21 | Added `deck_link` with revision anchor to player metadata |
| 1.5.0   | 2026-02-22 | Renamed `log_l1` to `events`; added `spec_version`, `per_turn_summary`, `game_summary`; new `DRAW` and `GAME_START` events; extended player metadata |
| 1.6.0   | 2026-03-11 | Added Commander Decklist Notation companion spec; `deck_rules` with mulligan scoring, combos, and anti-synergies; optional inline `decklist` in replay files |

---

## 19. Legal

This format is designed for Magic: The Gathering gameplay recording and analysis. Magic: The Gathering is trademark of Wizards of the Coast LLC.

---

**End of Specification**

For questions or suggestions about this format, please refer to the MTG Forge project documentation.
