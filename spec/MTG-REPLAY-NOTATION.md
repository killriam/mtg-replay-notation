# MTG Replay & Learning Notation

## Format Specification v1.2.0

**Status:** Stable  
**Published:** February 2026  
**Purpose:** Human-readable specification for understanding MTG game replay files

**Version History:**
- **1.0.0** (December 2025): Initial specification
- **1.1.0** (February 2026): Added `win_condition`, `conceded`, `deck_name`, `deck_hash` algorithm, `RESOURCES` event, `card_name` in events
- **1.2.0** (February 2026): Added `game_start` section (toss winner, starting player, play/draw choice), enhanced `MULLIGAN` event with full details
- **1.2.1** (February 2026): Added `ACTIVE_PLAYER_CHANGE` event for turn transitions

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
    "version": "1.2.0",
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
    "log_l1": [
        /* Level 1: Event log */
    ],
    "views_l2": [
        /* Level 2: Learning view */
    ]
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
            "deck_hash": "a3f8c2d1e9b7f604"
        },
        "P2": {
            "name": "Forge AI",
            "deck_name": "Mono-Red Aggro",
            "deck_hash": "7b2e5a9c4d1f8306"
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

### 4.2 Deck Hash Calculation

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

### 4.3 Win Condition Values

| Value | Description |
|-------|-------------|
| `life_zero` | Opponent's life reduced to 0 or less |
| `commander_damage` | 21+ commander damage from single commander |
| `decked` | Opponent attempted to draw from empty library |
| `poison` | Opponent received 10+ poison counters |
| `concession` | Opponent conceded |
| `alternate_win` | Card effect (e.g., Laboratory Maniac, Thassa's Oracle) |
| `draw` | Game ended in a draw |

### 4.4 Game Start Information

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

## 7. Level 1 Events (log_l1)

### 7.1 Event Structure

Each event in the `log_l1` array has this structure:

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
    "version": "1.1.0",
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
    "log_l1": [
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
3. Read **log_l1** sequentially from start to finish
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

---

## 17. Version History

| Version | Date       | Changes               |
| ------- | ---------- | --------------------- |
| 1.0.0   | 2025-12-20 | Initial specification |
| 1.1.0   | 2026-02-08 | Added `win_condition`, `conceded`, `deck_name` in meta; `deck_hash` algorithm; `RESOURCES` event; `card_name` in CAST, MOVE, PUT_ON_STACK, DAMAGE events |
| 1.2.0   | 2026-02-12 | Added Learning Helper Statistics section with KPIs for game analysis |

---

## 18. Legal

This format is designed for Magic: The Gathering gameplay recording and analysis. Magic: The Gathering is trademark of Wizards of the Coast LLC.

---

**End of Specification**

For questions or suggestions about this format, please refer to the MTG Forge project documentation.
