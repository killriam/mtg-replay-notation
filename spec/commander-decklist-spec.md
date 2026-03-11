# Commander Decklist Notation

## Companion Specification v1.0.0

**Status:** Stable
**Published:** March 2026
**Purpose:** Standardized JSON format for describing Commander format decklists,
including deck composition, card mechanic roles, mulligan evaluation rules,
and combo/anti-synergy declarations.

**Related Specification:** [MTG Replay & Learning Notation](./MTG-REPLAY-NOTATION.md)

---

## 1. Introduction

The Commander Decklist Notation defines a JSON structure for completely describing a
Commander format deck. It serves as both a deck registry entry and a behavioral
configuration that downstream tools (replay viewer, learning engine, connector) can
consume to:

- Identify the exact artwork for each card (edition + collector number)
- Understand the strategic role every card plays in the deck
- Evaluate opening hands using configurable mulligan scoring
- Recognize synergistic combinations and known anti-synergies

This format is designed to complement the replay format. A replay file may reference
a decklist via the `meta.players[N].deck_link` field; the decklist file itself is a
separate document.

---

## 2. File Structure Overview

A decklist file contains:

```json
{
    "format": "mtg-commander-decklist",
    "version": "1.0.0",
    "meta": {
        /* Deck metadata */
    },
    "commander": [
        /* Commander zone cards */
    ],
    "main": [
        /* Main deck cards */
    ],
    "sideboard": [
        /* Sideboard cards */
    ],
    "maybeboard": [
        /* Considered but not included cards */
    ],
    "deck_rules": {
        /* Mulligan rules and combo declarations */
    }
}
```

---

## 3. Deck Metadata (`meta`)

```json
{
    "meta": {
        "deck_id": "abc123-uuid",
        "deck_name": "Atraxa Superfriends",
        "format": "Commander",
        "colors": ["W", "U", "B", "G"],
        "created": "2026-03-11",
        "updated": "2026-03-11",
        "author": "Alice",
        "description": "Superfriends deck built around Atraxa's proliferate ability."
    }
}
```

### 3.1 Meta Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `deck_id` | string | No | Unique identifier for this deck (UUID recommended) |
| `deck_name` | string | **Yes** | Display name for the deck |
| `format` | string | **Yes** | Must be `"Commander"` |
| `colors` | array | No | Color identity as WUBRG letters (e.g., `["W","U","B","G"]`) |
| `created` | string | No | Creation date (ISO 8601 date, e.g., `"2026-03-11"`) |
| `updated` | string | No | Last modification date (ISO 8601 date) |
| `author` | string | No | Name or username of the deck builder |
| `description` | string | No | Free-text description of the deck strategy |

---

## 4. Deck Sections

A Commander decklist is divided into four sections. All sections use the same
**Card Entry** structure (see Section 5).

### 4.1 Commander Section (`commander`)

Contains all cards that start in the command zone:

- The primary commander (exactly 1 copy)
- Partner commanders (when the commander has the Partner keyword)
- Background enchantments (when the commander has "Choose a Background")
- Companion cards (when using the optional Companion mechanic)

```json
{
    "commander": [
        {
            "quantity": 1,
            "name": "Atraxa, Praetors' Voice",
            "edition": "C16",
            "collector_number": "35",
            "primary_mechanic": "counters",
            "additional_mechanics": ["proliferate", "multicolor", "win-condition"]
        }
    ]
}
```

### 4.2 Main Section (`main`)

The 99 cards (or 98 when using a companion) that form the main library. This section
should contain exactly 99 entries when card quantities are summed, unless a companion
is declared in the `commander` section, in which case 98 entries are expected.

```json
{
    "main": [
        {
            "quantity": 1,
            "name": "Sol Ring",
            "edition": "C21",
            "collector_number": "263",
            "primary_mechanic": "ramp",
            "additional_mechanics": ["mana-rock"]
        },
        {
            "quantity": 1,
            "name": "Doubling Season",
            "edition": "BBD",
            "collector_number": "183",
            "primary_mechanic": "counters",
            "additional_mechanics": ["combo-piece", "token"]
        }
    ]
}
```

### 4.3 Sideboard Section (`sideboard`)

Optional. Cards in the sideboard for tournament or casual swaps between games. The
sideboard is excluded from the `deck_hash` calculation (see the main replay spec,
Section 4.3).

```json
{
    "sideboard": [
        {
            "quantity": 1,
            "name": "Grafdigger's Cage",
            "edition": "M20",
            "collector_number": "228",
            "primary_mechanic": "removal",
            "additional_mechanics": ["hate-piece"]
        }
    ]
}
```

### 4.4 Maybeboard Section (`maybeboard`)

Optional. Cards that were considered for the deck but were not included. Useful for
deckbuilding reviews and upgrade tracking. Maybeboard cards do not count toward deck
totals and are excluded from the `deck_hash`.

```json
{
    "maybeboard": [
        {
            "quantity": 1,
            "name": "Parallel Lives",
            "edition": "INN",
            "collector_number": "203",
            "primary_mechanic": "token",
            "additional_mechanics": ["combo-piece"],
            "note": "Budget alternative to Doubling Season for tokens"
        }
    ]
}
```

---

## 5. Card Entry Format

Every card across all four sections uses the following structure:

```json
{
    "quantity": 1,
    "name": "Smothering Tithe",
    "edition": "RNA",
    "collector_number": "22",
    "primary_mechanic": "ramp",
    "additional_mechanics": ["card-draw", "synergy"]
}
```

### 5.1 Card Entry Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `quantity` | integer | **Yes** | Number of copies (usually 1 in Commander) |
| `name` | string | **Yes** | Exact English card name as printed |
| `edition` | string | **Yes** | Set code identifying the specific printing (e.g., `"RNA"`, `"C16"`, `"MH2"`) |
| `collector_number` | string | **Yes** | Collector number within the edition (e.g., `"22"`, `"263a"`) |
| `primary_mechanic` | string | **Yes** | Main strategic role this card fills in the deck (see Section 5.2) |
| `additional_mechanics` | array | No | Additional roles or synergies (array of strings from Section 5.2) |
| `note` | string | No | Free-text note about this card's role or budget considerations |

### 5.2 Edition and Collector Number

The combination of `edition` and `collector_number` uniquely identifies a specific
printing of a card, including the exact artwork, frame, and treatment (foil, showcase,
extended art, borderless, etc.).

**Edition codes** follow Scryfall set codes (3–4 uppercase or lowercase letters):

| Edition Code | Set Name |
|-------------|----------|
| `DOM` | Dominaria |
| `C16` | Commander 2016 |
| `MH2` | Modern Horizons 2 |
| `RNA` | Ravnica Allegiance |
| `C21` | Commander 2021 |
| `SLD` | Secret Lair Drop |

**Collector numbers** are strings to support suffixes for double-faced cards
(`"123a"`, `"123b"`) and variant treatments (`"263p"` for promo).

**Example — same card, different artworks:**

```json
{ "name": "Sol Ring", "edition": "C16", "collector_number": "263" },
{ "name": "Sol Ring", "edition": "LTR", "collector_number": "1" },
{ "name": "Sol Ring", "edition": "SLD", "collector_number": "141" }
```

### 5.3 Mechanic Categories

The `primary_mechanic` and `additional_mechanics` fields use string labels from the
following standard vocabulary. Custom labels are permitted; consumers should handle
unknown values gracefully.

#### Mana and Resources

| Category | Description |
|----------|-------------|
| `ramp` | General mana acceleration |
| `mana-rock` | Artifact that produces mana |
| `mana-dork` | Creature that produces mana |
| `land-ramp` | Searches for or puts extra lands into play |
| `fixing` | Produces multiple colors of mana |

#### Card Advantage

| Category | Description |
|----------|-------------|
| `card-draw` | Draws or replaces cards |
| `tutor` | Searches the library for specific cards |
| `looting` | Draws then discards (or discards then draws) |
| `recursion` | Returns cards from the graveyard to hand/battlefield |

#### Interaction

| Category | Description |
|----------|-------------|
| `removal` | Destroys or exiles a single permanent or spell |
| `board-wipe` | Mass removal affecting multiple permanents |
| `counter` | Counterspells |
| `hate-piece` | Shuts down or punishes specific strategies |
| `protection` | Protects key pieces from removal |

#### Threats and Win Conditions

| Category | Description |
|----------|-------------|
| `win-condition` | Primary way this deck wins |
| `threat` | Creates a must-answer threat |
| `combo-piece` | Required component of a combo (see Section 6.2) |
| `token` | Creates creature tokens |
| `tribal` | Grants bonuses to a specific creature type |

#### Strategy-Specific

| Category | Description |
|----------|-------------|
| `counters` | Puts or cares about +1/+1 or other counters |
| `proliferate` | Adds counters to permanents and players |
| `synergy` | Synergizes with the commander or deck theme |
| `flicker` | Blinks permanents to reset or re-trigger effects |
| `reanimation` | Returns cards from graveyard to play |
| `stax` | Slows the game with taxing or locking effects |
| `political` | Uses political mechanics (monarch, goad, vote) |
| `multicolor` | Rewards casting or controlling multicolored spells |
| `enchantress` | Triggers off enchantments entering or abilities |
| `spellslinger` | Triggers off instants and sorceries |

---

## 6. Deck Rules (`deck_rules`)

The `deck_rules` section encodes strategic guidelines that are specific to this deck
and used by replay analysis and coaching tools.

```json
{
    "deck_rules": {
        "mulligan": { /* ... */ },
        "combos": [ /* ... */ ],
        "dont_combos": [ /* ... */ ]
    }
}
```

### 6.1 Mulligan Rule

The mulligan rule defines how to score an opening hand to decide whether to keep it
or take a mulligan. Each card in the opening hand is assigned a **value** based on its
type, and the total hand value is compared against a **threshold** for the current
mulligan round.

#### 6.1.1 Card Values

```json
{
    "mulligan": {
        "card_values": {
            "land": 1.0,
            "cmc_0_to_2": 0.8,
            "cmc_3": 0.5,
            "other": 0.3
        }
    }
}
```

| Key | Default | Applies To |
|-----|---------|------------|
| `land` | `1.0` | Any land card |
| `cmc_0_to_2` | `0.8` | Non-land cards with converted mana cost 0–2 |
| `cmc_3` | `0.5` | Non-land cards with converted mana cost exactly 3 |
| `other` | `0.3` | All other non-land cards (CMC 4+) |

All values are floating-point numbers. Consumers may override individual categories
for specific decks (e.g., a high-CMC ramp deck might raise `other` to `0.4`).

**How to compute a hand's total value:**

For each card in the opening hand, look up its value from `card_values` based on
whether it is a land and, if not, its CMC. Sum the individual card values. The result
is the hand's **total value**.

**Example:**
A 7-card hand containing 3 lands (1.0 each), 2 mana rocks with CMC 2 (0.8 each), and
2 spells with CMC 5 (0.3 each) has a total value of:

```
3×1.0 + 2×0.8 + 2×0.3 = 3.0 + 1.6 + 0.6 = 5.2
```

#### 6.1.2 Mulligan Thresholds

```json
{
    "mulligan": {
        "thresholds": [
            {
                "round": 0,
                "hand_size": 7,
                "min_value": 3.5,
                "description": "Keep 7-card hand if total value is at least 3.5"
            },
            {
                "round": 1,
                "hand_size": 6,
                "min_value": 3.0,
                "description": "Keep 6-card hand if total value is at least 3.0"
            },
            {
                "round": 2,
                "hand_size": 5,
                "min_value": 2.5,
                "description": "Keep 5-card hand if total value is at least 2.5"
            },
            {
                "round": 3,
                "hand_size": 4,
                "min_value": 2.0,
                "description": "Keep 4-card hand if total value is at least 2.0"
            }
        ]
    }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `round` | integer | **Yes** | Mulligan round (0 = initial 7-card hand; 1 = after first mulligan, etc.) |
| `hand_size` | integer | No | Expected hand size at this round (informational) |
| `min_value` | number | **Yes** | Minimum total hand value required to keep |
| `description` | string | No | Human-readable rule description |

**Decision procedure:**

1. Evaluate each card in the current hand and sum its value.
2. Look up the threshold entry matching the current `round`.
3. If `total_value >= min_value`, keep the hand.
4. Otherwise, take a mulligan (London mulligan: draw 7, then put `round+1` cards on the bottom).

#### 6.1.3 Per-Card Value Overrides

For cards that do not fit the generic CMC-based formula, individual card overrides can
be declared in the mulligan section:

```json
{
    "mulligan": {
        "card_values": {
            "land": 1.0,
            "cmc_0_to_2": 0.8,
            "cmc_3": 0.5,
            "other": 0.3
        },
        "card_overrides": [
            {
                "name": "Sol Ring",
                "value": 1.2,
                "reason": "Best turn-1 play in Commander"
            },
            {
                "name": "Doubling Season",
                "value": 0.6,
                "reason": "High CMC but crucial for combo; better than average other"
            }
        ],
        "thresholds": [ /* ... */ ]
    }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | **Yes** | Exact card name |
| `value` | number | **Yes** | Override value for this specific card |
| `reason` | string | No | Explanation for the override |

### 6.2 Combos

The `combos` array declares known synergistic combinations (including infinite combos
and powerful two- or three-card interactions) present in the deck.

```json
{
    "combos": [
        {
            "id": "combo_inf_counters",
            "name": "Doubling Season + Atraxa",
            "pieces": ["Doubling Season", "Atraxa, Praetors' Voice"],
            "result": "Each end step, Atraxa's proliferate doubles all counters",
            "tags": ["infinite-ish", "counters", "win-condition"]
        },
        {
            "id": "combo_superfriends_ult",
            "name": "Deepglow Skate + Planeswalkers",
            "pieces": ["Deepglow Skate"],
            "result": "On ETB, doubles loyalty counters on all planeswalkers, enabling immediate ultimates",
            "tags": ["one-shot", "win-condition", "counters"]
        }
    ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | **Yes** | Unique identifier for this combo within the deck |
| `name` | string | **Yes** | Human-readable name |
| `pieces` | array | **Yes** | Card names required for the combo |
| `result` | string | **Yes** | Description of what the combo achieves |
| `tags` | array | No | Labels such as `"infinite"`, `"win-condition"`, `"two-card"` |

### 6.3 Don't Combos (Anti-Synergies)

The `dont_combos` array documents known anti-synergies: pairs or groups of cards that
conflict with each other in this specific deck. This is used by the learning engine to
flag situations where both cards are in play simultaneously.

```json
{
    "dont_combos": [
        {
            "id": "dc_stax_vs_storm",
            "name": "Rule of Law vs. Storm spells",
            "pieces": ["Rule of Law", "Thousand-Year Storm"],
            "reason": "Rule of Law prevents casting more than one spell per turn, disabling the storm engine completely.",
            "severity": "critical"
        },
        {
            "id": "dc_bounce_vs_etb_tracking",
            "name": "Conjurer's Closet vs. Teferi's Protection",
            "pieces": ["Conjurer's Closet", "Teferi's Protection"],
            "reason": "If Teferi's Protection phases out the Closet, its end-step trigger is lost for that turn cycle.",
            "severity": "minor"
        }
    ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | **Yes** | Unique identifier for this anti-synergy |
| `name` | string | **Yes** | Human-readable name |
| `pieces` | array | **Yes** | Card names involved in the conflict |
| `reason` | string | **Yes** | Explanation of why these cards conflict |
| `severity` | string | No | `"critical"` \| `"major"` \| `"minor"` (default: `"major"`) |

---

## 7. Deck Hash

The `deck_hash` in the replay format (Section 4.3 of the main spec) is derived from
the commander decklist using the following rule:

- Include all cards from the `commander` and `main` sections.
- Exclude `sideboard` and `maybeboard`.
- For each included card, form the string `"CardName:Quantity"`.
- Sort the resulting strings alphabetically.
- Concatenate with newline separators.
- Compute SHA-256 and return the first 16 hex characters.

This ensures that two players with identically-composed Commander + Main decks
(regardless of chosen artworks or sideboard/maybeboard) produce the same hash.

---

## 8. Relationship to the Replay Format

A replay file references a decklist through the player metadata:

```json
{
    "meta": {
        "players": {
            "P1": {
                "name": "Alice",
                "deck_name": "Atraxa Superfriends",
                "deck_hash": "a3f8c2d1e9b7f604",
                "deck_link": "https://mamo.games/deck/abc123-uuid#11032026_a3f8c2d1e9b7f604"
            }
        }
    }
}
```

Alternatively, a replay file may embed a full decklist inline via the optional
top-level `decklist` map:

```json
{
    "format": "mtg-replay",
    "version": "1.6.0",
    "decklist": {
        "P1": { /* full CommanderDecklist object */ },
        "P2": { /* full CommanderDecklist object */ }
    }
}
```

Embedding decklists inline is optional. When present, consumers should prefer the
inline decklist over an external lookup.

---

## 9. Validation Rules

1. **`commander` section must not be empty** — every Commander deck must declare at
   least one commander.
2. **`main` card count** — the sum of `quantity` values across all `main` entries
   should total 99 (or 98 when a companion is declared in `commander`).
3. **`edition` and `collector_number` must both be present** together — they have no
   meaning in isolation.
4. **`primary_mechanic` is required** — every card must have a declared primary role.
5. **Combo `pieces` must reference cards** that exist in at least one of `commander`,
   `main`, or `sideboard`.
6. **Don't combo `pieces` must reference cards** that exist in at least one of
   `commander`, `main`, or `sideboard`.
7. **Combo and don't-combo `id` values must be unique** within their respective arrays.
8. **Mulligan threshold `round` values must be unique** within the `thresholds` array
   and monotonically increasing from 0.

---

## 10. Complete Example

```json
{
    "format": "mtg-commander-decklist",
    "version": "1.0.0",
    "meta": {
        "deck_id": "atraxa-superfriends-v1",
        "deck_name": "Atraxa Superfriends",
        "format": "Commander",
        "colors": ["W", "U", "B", "G"],
        "created": "2026-03-11",
        "updated": "2026-03-11",
        "author": "Alice",
        "description": "Proliferate planeswalkers to their ultimate abilities."
    },
    "commander": [
        {
            "quantity": 1,
            "name": "Atraxa, Praetors' Voice",
            "edition": "C16",
            "collector_number": "35",
            "primary_mechanic": "counters",
            "additional_mechanics": ["proliferate", "win-condition"]
        }
    ],
    "main": [
        {
            "quantity": 1,
            "name": "Sol Ring",
            "edition": "C21",
            "collector_number": "263",
            "primary_mechanic": "ramp",
            "additional_mechanics": ["mana-rock"]
        },
        {
            "quantity": 1,
            "name": "Arcane Signet",
            "edition": "ELD",
            "collector_number": "331",
            "primary_mechanic": "ramp",
            "additional_mechanics": ["mana-rock", "fixing"]
        },
        {
            "quantity": 1,
            "name": "Doubling Season",
            "edition": "BBD",
            "collector_number": "183",
            "primary_mechanic": "counters",
            "additional_mechanics": ["combo-piece", "token", "synergy"]
        },
        {
            "quantity": 1,
            "name": "Deepglow Skate",
            "edition": "C16",
            "collector_number": "3",
            "primary_mechanic": "counters",
            "additional_mechanics": ["combo-piece", "win-condition"]
        },
        {
            "quantity": 1,
            "name": "Smothering Tithe",
            "edition": "RNA",
            "collector_number": "22",
            "primary_mechanic": "ramp",
            "additional_mechanics": ["token"]
        },
        {
            "quantity": 1,
            "name": "Swords to Plowshares",
            "edition": "A25",
            "collector_number": "35",
            "primary_mechanic": "removal",
            "additional_mechanics": []
        },
        {
            "quantity": 1,
            "name": "Counterspell",
            "edition": "MH1",
            "collector_number": "52",
            "primary_mechanic": "counter",
            "additional_mechanics": ["protection"]
        },
        {
            "quantity": 1,
            "name": "Rhystic Study",
            "edition": "PCY",
            "collector_number": "48",
            "primary_mechanic": "card-draw",
            "additional_mechanics": ["stax"]
        },
        {
            "quantity": 1,
            "name": "Command Tower",
            "edition": "C21",
            "collector_number": "279",
            "primary_mechanic": "fixing",
            "additional_mechanics": []
        }
    ],
    "sideboard": [],
    "maybeboard": [
        {
            "quantity": 1,
            "name": "Parallel Lives",
            "edition": "INN",
            "collector_number": "203",
            "primary_mechanic": "token",
            "additional_mechanics": ["combo-piece"],
            "note": "Budget alternative to Doubling Season for token strategies"
        }
    ],
    "deck_rules": {
        "mulligan": {
            "card_values": {
                "land": 1.0,
                "cmc_0_to_2": 0.8,
                "cmc_3": 0.5,
                "other": 0.3
            },
            "card_overrides": [
                {
                    "name": "Sol Ring",
                    "value": 1.2,
                    "reason": "Best turn-1 play in Commander; always keep"
                },
                {
                    "name": "Doubling Season",
                    "value": 0.6,
                    "reason": "CMC 5 but a game-winning piece; worth keeping in opening hand"
                }
            ],
            "thresholds": [
                {
                    "round": 0,
                    "hand_size": 7,
                    "min_value": 3.5,
                    "description": "Keep 7-card hand if total value is at least 3.5"
                },
                {
                    "round": 1,
                    "hand_size": 6,
                    "min_value": 3.0,
                    "description": "Keep 6-card hand if total value is at least 3.0"
                },
                {
                    "round": 2,
                    "hand_size": 5,
                    "min_value": 2.5,
                    "description": "Keep 5-card hand if total value is at least 2.5"
                },
                {
                    "round": 3,
                    "hand_size": 4,
                    "min_value": 2.0,
                    "description": "Keep 4-card hand if total value is at least 2.0"
                }
            ]
        },
        "combos": [
            {
                "id": "combo_doubling_atraxa",
                "name": "Doubling Season + Atraxa",
                "pieces": ["Doubling Season", "Atraxa, Praetors' Voice"],
                "result": "Each end step, Atraxa's proliferate doubles all counters on the battlefield",
                "tags": ["counters", "engine", "win-condition"]
            },
            {
                "id": "combo_deepglow_walkers",
                "name": "Deepglow Skate + Planeswalkers",
                "pieces": ["Deepglow Skate"],
                "result": "On ETB doubles loyalty counters on all planeswalkers, immediately enabling most ultimates",
                "tags": ["one-shot", "win-condition", "counters"]
            }
        ],
        "dont_combos": [
            {
                "id": "dc_rhystic_teferi_time_raveler",
                "name": "Rhystic Study vs. Teferi, Time Raveler",
                "pieces": ["Rhystic Study", "Teferi, Time Raveler"],
                "reason": "Teferi prevents opponents from casting spells at instant speed, so Rhystic Study's trigger can never be paid during the draw step sequence.",
                "severity": "minor"
            }
        ]
    }
}
```

---

## 11. Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-03-11 | Initial specification |

---

## 12. Legal

This specification is designed for Magic: The Gathering gameplay recording and
analysis. Magic: The Gathering is a trademark of Wizards of the Coast LLC.
