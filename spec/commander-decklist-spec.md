# Commander Decklist Notation

## Companion Specification v1.5.1

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
        /* Mulligan rules, combo declarations, scenarios, and simulation config */
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
        "source_url": "https://moxfield.com/decks/abc123",
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
| `source_url` | string | No | URL of the external deck page this list was imported from (e.g. Moxfield, Archidekt) |
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
        "dont_combos": [ /* ... */ ],
        "scenarios": [ /* ... */ ],
        "simulation": { /* ... */ }
    }
}
```

### 6.1 Mulligan Rule

The mulligan rule defines how to score an opening hand to decide whether to keep it
or take a mulligan.

> **Two coexisting scoring models as of v1.5.0.** §§6.1.1–6.1.4 below define the original
> model: each card in the opening hand is assigned a **value** based on its type and exact
> mana value, and the total hand value is compared against a **threshold** for the current
> mulligan round. This remains the *only* model the compact `.dck` `AiHints=` encoding
> (§6.1.4) understands, and is still what real Forge games decide mulligans with.
> §6.1.5 (**Mana Base Band**, new in v1.5.0) defines a second, independent model — a
> formula-only score of just a hand's lands and cheap mana-producing cards, compared
> against a fixed band — that a consumer may implement *instead of* §§6.1.1–6.1.3's
> curve-and-threshold decision for the JSON form of this rule. A consumer should treat
> `mana_base_min`/`mana_base_max` (§6.1.5), if present, as authoritative for the
> keep/mulligan decision, and `card_values`/`thresholds` as informational/legacy in that
> case — see §6.1.5 for exactly which consumers currently do this.

#### 6.1.1 Card Values

> **Changed in v1.3.0** (previously 4 CMC-bucketed keys — `cmc_0_to_2`/`cmc_3`/`other`;
> that shape is no longer valid). Widened to a full per-mana-value curve so a deck's
> standard baseline can distinguish, say, a 1-drop from a 2-drop rather than lumping
> "CMC 0–2" together. No production decks had ever saved a `mulligan` block under the
> old shape at the time of this change, so no migration note is needed for existing data.
>
> **Changed in v1.4.0**: `mv4`–`mv7Plus` defaults lowered to a flat `0.2` (previously
> `0.45`/`0.4`/`0.35`/`0.3`) — mana value 4+ is worth meaningfully less to see in an
> opening hand. Also added §6.1.1a (X-cost mana-value adjustment) and §6.1.1b (multicolor
> land bonus), two new rules in "how to compute a hand's total value" below.

```json
{
    "mulligan": {
        "card_values": {
            "land": 1.0,
            "mv0": 0.85,
            "mv1": 0.8,
            "mv2": 0.75,
            "mv3": 0.6,
            "mv4": 0.2,
            "mv5": 0.2,
            "mv6": 0.2,
            "mv7Plus": 0.2
        }
    }
}
```

| Key | Default | Applies To |
|-----|---------|------------|
| `land` | `1.0` | Any land card |
| `mv0` | `0.85` | Non-land cards with mana value exactly 0 |
| `mv1` | `0.8` | Non-land cards with mana value exactly 1 |
| `mv2` | `0.75` | Non-land cards with mana value exactly 2 |
| `mv3` | `0.6` | Non-land cards with mana value exactly 3 |
| `mv4` | `0.2` | Non-land cards with mana value exactly 4 |
| `mv5` | `0.2` | Non-land cards with mana value exactly 5 |
| `mv6` | `0.2` | Non-land cards with mana value exactly 6 |
| `mv7Plus` | `0.2` | Non-land cards with mana value 7 or higher |

All values are floating-point numbers. All 9 keys are required (a consumer should treat
a missing key as the default shown above, not as an error). Consumers may override
individual entries for specific decks (e.g., a high-curve ramp deck might raise
`mv7Plus`), and a per-card `card_overrides` entry (§6.1.3) always wins over this curve
for the named card.

**How to compute a hand's total value:**

For each card in the opening hand, look up its base value from `card_values`: `land` if
it's a land, else the entry matching its mana value (adjusted per §6.1.1a if the card has
an `{X}` cost) rounded to the nearest integer and clamped to `[0, 7]` (so mana value 7 and
anything higher both use `mv7Plus`). If the card is a land producing 2 or more colors,
multiply its value by §6.1.1b's coverage multiplier. Sum every card's (possibly adjusted)
value. The result is the hand's **total value**.

**Example:**
A 7-card hand containing 3 lands (1.0 each, none multicolor), 2 mana rocks with mana
value 2 (0.75 each), and 2 spells with mana value 5 (0.2 each) has a total value of:

```
3×1.0 + 2×0.75 + 2×0.2 = 3.0 + 1.5 + 0.4 = 4.9
```

##### 6.1.1a Mana Value Adjustment for `{X}` Costs

*(New in v1.4.0.)* A non-land card whose mana cost includes one or more `{X}` symbols
(e.g. `{X}{R}`, Fireball) has its mana value increased by 2 **for this curve lookup
only**, following the common convention of treating `X = 2` as a rough average. This
does **not** change the card's real mana value anywhere else a consumer might use it
(casting cost, curve statistics, etc.) — it's a scoring-only adjustment applied just
before the §6.1.1 bucket lookup. Cards with more than one `{X}` symbol are **not**
double-adjusted — the +2 applies once per card regardless of how many `{X}` symbols its
cost contains, matching every existing implementation of this rule.

Fireball (`{X}{R}`, real mana value 1 by convention) is therefore scored at effective
mana value 3 (`mv3`), not `mv1`.

##### 6.1.1b Multicolor Land Bonus

*(New in v1.4.0.)* A land producing 2 or more colors (mono-color and colorless lands are
unaffected — their value is exactly `card_values.land`, no multiplier) has its `land`
value multiplied by a factor from **1.0 to 1.4**, scaled by how well the colors it
produces match the deck's own colored-mana-pip distribution. The idea: a land fixing for
colors the deck's spells actually need heavily is more valuable to open with than one
fixing for a barely-used splash color.

**Step 1 — compute the deck's colored-pip weights** (once per deck, not per hand): for
each of the 5 colors, sum the number of that color's mana symbols across every **non-land**
card in the deck (each card's symbols counted once per physical copy — i.e. weighted by
however many copies of that card the deck runs). Divide each color's total by the grand
total across all 5 colors to get that color's *weight* (0.0–1.0, summing to 1.0 across
all five). If the deck has no colored pips at all, every weight is 0 and no land ever
receives a bonus.

**Step 2 — score a land producing colors `C`:** if `|C| < 2`, the multiplier is `1.0`
(no bonus). Otherwise, sum the deck's pip-weight for each color in `C` (call this
*coverage*, capped at `1.0` since a land covering every color used could otherwise
exceed it due to floating-point rounding), and compute:

```
multiplier = 1.0 + 0.4 × coverage
```

**Example:** A deck's non-land cards use only blue and black mana, in a 60/40 split
(`pipWeight.U = 0.6`, `pipWeight.B = 0.4`, all others `0`). A dual land producing both
blue and black has `coverage = 0.6 + 0.4 = 1.0`, so `multiplier = 1.0 + 0.4×1.0 = 1.4` —
the maximum bonus, since it perfectly covers the deck's only two colors. A land producing
blue and red instead has `coverage = 0.6 + 0 = 0.6`, so `multiplier = 1.0 + 0.4×0.6 = 1.24`
— a smaller bonus, since red isn't used by any non-land card in this deck at all.

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
            "mv0": 0.85,
            "mv1": 0.8,
            "mv2": 0.75,
            "mv3": 0.6,
            "mv4": 0.2,
            "mv5": 0.2,
            "mv6": 0.2,
            "mv7Plus": 0.2
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
                "reason": "High mana value but crucial for combo; better than the curve default"
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

#### 6.1.4 Compact Inline Encoding (`.dck` `AiHints=`)

**Purpose:** §§6.1.1–6.1.3 above define the mulligan rule as part of the full
`mtg-commander-decklist` JSON file. That file is not always present at the point a deck is
actually played — Forge's real game client loads decks from plain-text `.dck` files, which
carry no JSON payload by default. This compact, non-JSON encoding lets the same `card_values`
and `thresholds` data ride along on the `.dck` file itself, as one `AiHints=` line in its
`[metadata]` section, so a deck's mulligan tuning reaches a real game with no companion file
required. `card_overrides` (§6.1.3) are supported in this form too; `card_values` (§6.1.1) are
**not** — a consumer that also needs to override the base tier weights must use the full JSON
form via a `DecklistSpec$<path>` `AiHints` token instead (a path to an external
`mtg-commander-decklist` JSON file; not detailed further here — see
`forge-integration-guide.md` §12.5.5 for that mechanism).

**Location:** One `AiHints=` line inside the `.dck` file's `[metadata]` section. If more than
one hint token is present (e.g. alongside a future `Combo$`/`DontCombo$` token), they are
joined with `" | "`.

**Tokens:**

```
AiHints=MulliganThreshold$0:3.5;1:3.0;2:2.5;3:2.0 | MulliganOverride$Sol Ring:1.2;Doubling Season:0.6
```

| Token | Format | Maps to |
|-------|--------|---------|
| `MulliganThreshold$` | `<round>:<min_value>;<round>:<min_value>;...` | One entry per `thresholds[]` item (§6.1.2) — `hand_size`/`description` are dropped, since they're informational-only in the JSON form |
| `MulliganOverride$` | `<CardName>:<value>;<CardName>:<value>;...` | One entry per `card_overrides[]` item (§6.1.3) — `reason` is dropped |

`round` is an integer, `min_value`/`value` are decimal numbers, `CardName` is the exact card
name (must not contain `:` or `;`). Entries within one token are `;`-separated; there is no
escaping mechanism for a `;` or `:` inside a card name — such a card cannot be expressed in
this compact form (falls back to the JSON `DecklistSpec$` route instead).

**Interpretation is identical to §6.1.2's decision procedure**, substituting the tokens above
for the JSON fields, and falling back to §6.1.1's default `card_values` curve
(`land: 1.0, mv0: 0.85, mv1: 0.8, mv2: 0.75, mv3: 0.6, mv4: 0.2, mv5: 0.2, mv6: 0.2,
mv7Plus: 0.2`) and this spec's default thresholds (`0:3.5, 1:3.0, 2:2.5, 3:2.0`) for any
round not listed. §6.1.1a (X-cost mana-value adjustment) and §6.1.1b (multicolor land
bonus) are still part of this same fallback procedure — this compact form only omits the
*override* mechanism for `card_values` itself, not the rest of §6.1.1's scoring rules:

1. Score each card in the current hand: an override value if one matches its name, else the
   default curve value for land / its exact mana value (clamped to `mv7Plus` at 7+, and
   adjusted per §6.1.1a/§6.1.1b as applicable).
2. Sum every card's value → hand score.
3. Find the `MulliganThreshold$` entry for the current round (0 = initial 7-card hand); if
   none exists for that round, use the default listed above.
4. Keep if `hand score ≥ min_value`; otherwise mulligan (London mulligan: draw 7, bottom
   `round + 1` cards).

**Absence is valid:** a `.dck` file with no `MulliganThreshold$`/`MulliganOverride$` tokens
(or no `AiHints=` line at all) carries no opinion on mulligan tuning — a consuming program
should fall back to its own default mulligan behavior rather than treating this as an error.

**Reference implementation:** `new-backend`'s `getPublicDeckForgeExport` (writer) and Forge's
`forge.deck.DeckRulesConfig.fromInlineHints()` / `forge.ai.ComputerUtil.wantMulligan()` via
`forge.ai.mulligan.DecklistMulliganEvaluator` (reader) — see `forge-integration-guide.md`
§12.5.5 for the surrounding `AiHints`/`DecklistSpecPath` mechanism this token family extends.
**Unaffected by §6.1.5** — Mana Base does not extend to this compact form; real Forge games
played from a `.dck` file are decided by §§6.1.1–6.1.3 exactly as before, regardless of what a
deck's JSON `mulligan` block says about Mana Base.

#### 6.1.5 Mana Base Band

*(New in v1.5.0.)* A second, independent way to decide keep-or-mulligan, alongside §§6.1.1–6.1.4
rather than replacing them in the schema. Where §6.1.1's `card_values` scores *every* card in the
hand by a curve, Mana Base scores **only** lands and cheap mana-producing cards — every other
card, however good, contributes nothing — and compares the sum against a fixed band instead of a
per-round threshold table.

```json
{
    "mulligan": {
        "mana_base_min": 3,
        "mana_base_max": 4
    }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mana_base_min` | number | No (default `3`) | Below this total, the hand has too little mana |
| `mana_base_max` | number | No (default `4`) | Above this total, the hand has too much mana (see "Two verdicts" below) |

**Per-card Mana Base value** (independent of `card_values` — does not use the §6.1.1 curve, is
**not** affected by `card_overrides` §6.1.3 at all, and is unaffected by the §6.1.1a/§6.1.1b
adjustments):

| Card | Value |
|------|-------|
| Basic land | `1.0` |
| True non-mana utility land (produces no mana at all, e.g. Maze of Ith, Dark Depths) | `0.0` |
| Non-basic land producing exactly one color, enters tapped or under a condition (rules text matches `enters tapped unless`, `unless you`, `you may pay...{`, `enters tapped`, or `enters the battlefield tapped`) | `0.8` |
| Non-basic land producing exactly one color, otherwise | `1.0` |
| Land producing 2 or more colors | `1.0` to `1.4`, via the **same** coverage formula as §6.1.1b |
| Non-land card that produces mana, mana value `0` | `1.0` |
| Non-land card that produces mana, mana value `1` | `0.9` |
| Non-land card that produces mana, mana value `2` | `0.6` |
| Anything else (mana value 3+ mana producers; any card with no mana ability at all) | `0.0` |

**Decision procedure:**

1. For each card in the hand, look up its Mana Base value from the table above (lands via the
   §6.1.1b coverage formula where applicable; non-lands by mana value).
2. Sum every card's value → the hand's **Mana Base score**.
3. **Two verdicts, for two different audiences:**
   - **Playable** (`score >= mana_base_min`) — the human-facing "can this hand function at all"
     check. Does not penalize a mana-flooded hand.
   - **Good AI hand** (`mana_base_min <= score <= mana_base_max`) — the stricter, double-sided
     check used when a program needs to pick or redraw an opening hand *for* a simulated
     player, so it lands neither mana-screwed nor mana-flooded. A consumer implementing
     automated hand selection (not just showing a keep/mulligan pill to a human) should use
     this check, not the single-sided one above.

**Example:** A 7-card hand containing 3 basic lands (`1.0` each), 1 dual land covering 100% of
the deck's colored pips (`1.4`), 1 mana rock at mana value 1 (`0.9`), and 2 non-mana spells
(`0.0` each) has a Mana Base score of:

```
3×1.0 + 1×1.4 + 1×0.9 + 2×0.0 = 3.0 + 1.4 + 0.9 = 5.3
```

Against the default band (`3`–`4`), this hand is **Playable** (`5.3 >= 3`) but **not a Good AI
hand** (`5.3 > 4` — too much mana for automated hand selection to prefer, even though a human
would likely still keep it).

**Which consumers implement this (as of v1.5.0):** MaMoFrontend's Mulligan Decision tab
(`MulliganValueEditor.tsx`) computes and displays both verdicts; `mamo-sim`'s "Simulate AI"
batch-statistics tool uses the **Good AI hand** check as its actual opening-hand-redraw
criteria (`game_engine.rs`'s `run_game`, reached via `new-backend`'s
`GET /api/simulation/deck-input/:deckId` → `mamo-Connector`'s `encode_deck_input` →
mamo-sim's wire format). **Does not reach real Forge games** — those are decided by §6.1.1's
curve via the compact `.dck` encoding (§6.1.4) exactly as before; Mana Base has no `.dck`
representation. A consumer with no opinion on Mana Base should simply ignore
`mana_base_min`/`mana_base_max` and use §6.1.1's total-value model unchanged — their presence
in a `mulligan` block is never required.

**Known limitation:** mamo-sim's per-card wire encoding has no bit available to distinguish a
true non-mana utility land (e.g. Maze of Ith) from a normal untapped land, so that one card
category scores `1.0` there instead of the `0.0` a fuller implementation (like
`MulliganValueEditor.tsx`'s) gives it — a disclosed, not-yet-fixed gap in that one consumer,
not a schema ambiguity.

#### 6.1.6 Starting Hand Quality (informational — not a schema field, most of it can't be one)

*(Documented for completeness, not added as a new field.)* MaMoFrontend's Mulligan Decision tab
also computes a **Starting Hand Quality** number, shown alongside Mana Base (§6.1.5) but purely
informational — it never affects the keep/mulligan decision. Unlike every other rule in §6.1,
it is **not** exposed as a `mulligan` JSON field here, because two of its three components
depend on data this notation format does not model at all:

| Component | Formula | Representable in this schema? |
|-----------|---------|-------------------------------|
| Mana Curve bonus | Flat `+0.5`, all-or-nothing, if the hand has a non-land permanent (creature/artifact/enchantment/planeswalker) at mana value exactly 1, another at 2, another at 3, and another at 4 | **Yes** — uses only each card's type line and mana value, both already in §5 |
| Tier ranking bonus | `+0.40` / `+0.25` / `+0.10` per hand card assigned Mechanic Graph tier S / A / B (best tier across every mechanic group it's in; unranked/lower tiers add `+0`) | **No** — tier assignment (`S`/`A`/`B`/`C`/`D`/`E`/`F`/`Maybe`) is a property of MaMo's internal `CardsDeckMechanicAssignment` table, with no equivalent field anywhere in this notation. §6.4's `mechanic_groups` (used by scenario zone requirements) are plain string keys — no per-card tier, no per-group rank |
| Deckmechanic synergy bonus | `+0.15` per mechanic group the hand can chain into (holds that group's card *and* a card from a group that enables it), capped at `+0.6` | **No** — requires the enabler→dependent relationship between mechanic groups (MaMo's internal `synergyDetails.enablingFormations`). This notation's `mechanic_groups` (§6.4.2/§6.4.3, and the schema's `mechanic_groups: string[]`) are a flat, unordered list of keys — no dependency/enabling-formation concept exists here at all |

Because 2 of the 3 components can only be computed by reaching into MaMo's own database (not
from a standalone `mtg-commander-decklist` JSON file), Starting Hand Quality stays a
MaMoFrontend-only display rather than a documented `mulligan` field — adding just the Mana
Curve third of it as a JSON field without the other two would misrepresent the real feature.
A future version of this notation that grows a per-card tier field and an inter-group
dependency field on `mechanic_groups` could reopen this; not proposed here.

**Full description:** `MaMoFrontend/specs/playbook.spec.md` AC-MULL-001i,
`MaMoFrontend/specs/evaluation.spec.md` §Mulligan Decision.

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

### 6.4 Scenarios

The optional `scenarios` array documents strategic game states captured by the deck
builder. Two sub-formats are supported: **hand-based** (opening hand + drawn turns) and
**precondition-based** (arbitrary game state defined by conditions).

#### 6.4.1 Scenario Types

| `type` | Description |
|--------|-------------|
| `best_starting_hand` | 7-card opening hand + 3 drawn turns (guided wizard) |
| `perfect_game` | 7-card opening hand + 10 drawn turns (full strategic ideal) |
| `mid_game` | Arbitrary game state defined by preconditions |
| `free_build` | Freeform board state with optional preconditions and focus |
| `eval_sequence` | Multi-turn draw + play sequence for evaluation; can be forced or matched |

#### 6.4.1a Scenario Mode

All scenario types support an optional `mode` field that controls execution behaviour:

| `mode` | Description |
|--------|-------------|
| `"forced"` | Forge sets a fixed library order so the exact draw sequence is executed. Card group references are resolved to concrete card names before play. |
| `"look_for"` | Forge runs normally; the scenario is a pattern — the game is monitored and a match event is logged when the game state satisfies the scenario. Default. |

#### 6.4.1b Card Reference Type

Wherever a card name is expected (in `opening_hand`, `turns[].drawn`, `turns[].played`,
`board_state.zones`, and `zone_requirements`) a **card reference** may be either:

- a plain string `"Lightning Bolt"` — matches the exact card name
- an object `{"group": "ramp"}` — matches any card whose `primary_mechanic` or
  `additional_mechanics` includes the named mechanic group key

Group references allow `look_for` scenarios to describe patterns independent of the
specific card drawn, and allow `forced` scenarios to say "resolve this group slot to
the first matching card in the deck list".

#### 6.4.1c Scenario Category

Scenarios are classified into three functional categories to prevent goldfishing / showcase scenarios from corrupting AI guidance evaluation:

| `category` | Included Types | Purpose & Lifecycle Role |
| :--- | :--- | :--- |
| `"aspirational"` | `best_starting_hand`, `perfect_game` | Showcase dream openings and goldfishing ceilings (0 opponent interaction). **Excluded from AI guidance policy calibration.** |
| `"tactical_benchmark"` | `decision_puzzle`, `threat_triage`, `eval_sequence`, `mid_game` | Deterministic unit-test fixtures with `decision_question` and `evaluation_criteria` used to verify AI guidance and policy rules. |
| `"replay_snapshot"` | `blunder_snapshot`, `critical_turn`, `free_build` | Game-state fixtures extracted directly from match replays for regression testing and blunder review. |

#### 6.4.2 Hand-Based Scenario Format

Used for `best_starting_hand` and `perfect_game` types. Documents which cards were in
the opening hand and what was drawn and played each subsequent turn.

```json
{
    "scenarios": [
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
                {
                    "turn": 1,
                    "drawn": "Smothering Tithe",
                    "played": ["Sol Ring", "Command Tower"]
                },
                {
                    "turn": 2,
                    "drawn": "Rhystic Study",
                    "played": ["Arcane Signet", "Forest"]
                },
                {
                    "turn": 3,
                    "drawn": "Deepglow Skate",
                    "played": ["Plains", "Doubling Season"]
                }
            ]
        }
    ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | **Yes** | Unique identifier for this scenario within the deck |
| `type` | string | **Yes** | One of the scenario types (see §6.4.1) |
| `name` | string | **Yes** | Human-readable scenario name |
| `deck_id` | string | No | Owning deck's identifier. Normally implicit and omitted — a scenario embedded in a deck's own exported document belongs to that document's own `meta.deck_id`. Only needed when a scenario reference crosses into another deck's context (e.g. attaching an opponent's own Perfect Game scenario to a constructed match — see [forge-integration-guide.md](./forge-integration-guide.md) §10, a proposed, not-yet-implemented pipeline). |
| `mode` | string | No | `"forced"` or `"look_for"` (see §6.4.1a). Default: `"look_for"` |
| `opening_hand` | array | For hand-based | Card references in the opening hand (in draw order). See §6.4.1b. |
| `turns` | array | No | Drawn and played cards per turn (see below) |
| `board_state` | object | No | Explicit zone snapshot for this scenario (see §6.4.4) |

**Turn entry fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `turn` | integer | **Yes** | Turn number (1 = first turn of the game) |
| `drawn` | string or object | No | Card reference drawn at the start of this turn (see §6.4.1b). Omit for a turn that only scripts `actions` (below) against an existing permanent, with no new card drawn. |
| `played` | array | No | Card references played this turn, in cast order (see §6.4.1b) |
| `actions` | array | No | Attack/activate actions against a permanent already in play this turn — `{"type": "ATTACK" \| "ACTIVATE", "source": <card reference>}`. Local extension originating in MaMoFrontend's export pipeline; not yet implemented by every consumer of this spec. |

`drawn` was documented as required in earlier drafts of this spec; relaxed 2026-08-16 to match
what every real implementation already does — a turn-by-turn placement UI can legitimately
schedule a turn with only an attack/activate action and no newly drawn/played card at all.

#### 6.4.3 Precondition-Based Scenario Format

Used for `mid_game` and `free_build` types. Describes the game state conditions under
which a scenario applies rather than documenting a specific card sequence.

```json
{
    "scenarios": [
        {
            "id": "scenario_turn4_engine",
            "type": "mid_game",
            "name": "Turn 4 Counter Engine Active",
            "preconditions": {
                "description": "Turn 4+, commander in play, 6 mana available, 2 ramp pieces on board",
                "mana_available": 6,
                "mana_colors": ["W", "U", "B", "G"],
                "turn_number": 4,
                "min_hand_size": 2,
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
            },
            "focus": {
                "mechanic_groups": ["counters", "proliferate"],
                "card_names": ["Doubling Season", "Deepglow Skate"],
                "description": "Demonstrates how the counter engine doubles with Doubling Season"
            }
        }
    ]
}
```

**Preconditions fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `description` | string | No | Human-readable summary of required game state |
| `mana_available` | integer | No | Total mana available (any color) |
| `mana_colors` | array | No | Specific mana colors required (WUBRG letters) |
| `turn_number` | integer | No | Turn number this scenario typically applies to |
| `min_hand_size` | integer | No | Minimum number of cards in hand |
| `zone_requirements` | array | No | Per-zone card or mechanic requirements (see below) |

**Zone requirement fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `zone` | string | **Yes** | One of: `battlefield`, `hand`, `graveyard`, `commandZone`, `library`, `exile` |
| `mechanic_groups` | array | No | Deck mechanic group keys satisfying this requirement |
| `card_names` | array | No | Specific card names required in this zone |
| `min_count` | integer | No | Minimum matching cards required (default: count of listed items) |

Either `mechanic_groups` or `card_names` (or both) must be present in a zone requirement.

**Focus fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mechanic_groups` | array | No | Deck mechanic group keys this scenario demonstrates |
| `card_names` | array | No | Specific card names that are focal in this scenario |
| `description` | string | No | Free-text description of what the scenario demonstrates |

#### 6.4.4 Board State

The optional `board_state` object captures an explicit (partial) zone snapshot. It
complements `preconditions`: where `preconditions` defines conditions that must be met,
`board_state` defines a concrete snapshot of zones and their contents.

```json
"board_state": {
    "turn": 3,
    "active_player": "self",
    "zones": {
        "hand": [
            "Sol Ring",
            {"group": "ramp"}
        ],
        "battlefield": [
            "Command Tower",
            {"group": "land"},
            {"group": "land"}
        ],
        "graveyard": []
    }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `turn` | integer | No | Turn number this board state applies to |
| `active_player` | string | No | `"self"` \| `"opponent"` \| player name |
| `zones` | object | No | Map of zone name → array of card references (see §6.4.1b) |

Zone names: `hand`, `battlefield`, `graveyard`, `library`, `exile`, `commandZone`.

#### 6.4.5 Eval Sequence Scenario Format

The `eval_sequence` type combines a full opening hand + per-turn draw/play sequence
with an optional board state. It is the primary scenario type for evaluation runs.

The `7+3` shape is expressed as `opening_hand` of 7 + 3 `turns` entries.
The `14` shape is expressed as `opening_hand` of 14 with no `turns`.

```json
{
    "scenarios": [
        {
            "id": "eval_7plus3_aggro",
            "type": "eval_sequence",
            "name": "7+3 Aggro Opening",
            "mode": "forced",
            "opening_hand": [
                "Mountain",
                "Mountain",
                {"group": "land"},
                "Lightning Bolt",
                {"group": "aggro"},
                {"group": "aggro"},
                {"group": "aggro"}
            ],
            "turns": [
                {
                    "turn": 1,
                    "drawn": {"group": "land"},
                    "played": ["Mountain", "Lightning Bolt"]
                },
                {
                    "turn": 2,
                    "drawn": {"group": "aggro"},
                    "played": [{"group": "land"}, {"group": "aggro"}]
                },
                {
                    "turn": 3,
                    "drawn": {"group": "aggro"},
                    "played": [{"group": "aggro"}]
                }
            ],
            "focus": {
                "mechanic_groups": ["aggro"],
                "card_names": ["Lightning Bolt"],
                "description": "Verify consistent aggro curve over 3 turns"
            }
        }
    ]
}
```

In `forced` mode Forge resolves each group reference to the first matching card in
the deck list and builds the `forcedLibraryOrder` accordingly.
In `look_for` mode (default) the turn sequence serves as a pattern: a match event is
logged in the replay whenever the actual game state satisfies the sequence up to the
current turn.

### 6.5 Forge Simulation Config

The optional `simulation` object configures how the Forge AI simulation tool should
use this deck when running simulations. Forge consumes `.dck` plain-text decklists;
this config is a companion layer that carries simulation parameters separately.

```json
{
    "deck_rules": {
        "simulation": {
            "target": "forge",
            "play_order": "random",
            "difficulty": "ultimate",
            "starting_life": 40,
            "eval_scenario_ids": ["eval_7plus3_aggro", "eval_14_midrange"]
        }
    }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `target` | string | **Yes** | Must be `"forge"` (reserved for future simulation targets) |
| `play_order` | string | No | `"play"` (go first) \| `"draw"` (go second) \| `"random"` (default) |
| `difficulty` | string | No | AI difficulty: `"easy"` \| `"medium"` \| `"hard"` \| `"ultimate"` (default) |
| `starting_life` | integer | No | Starting life total (default: `40` for Commander) |
| `eval_scenario_ids` | array | No | IDs of `eval_sequence` scenarios to run. `forced` scenarios use a fixed library order; `look_for` scenarios run as pattern matchers. |
| ~~`use_best_starting_hand`~~ | boolean | *Deprecated* | Replaced by `eval_scenario_ids`. Kept for v1.x compatibility. |
| ~~`use_perfect_game`~~ | boolean | *Deprecated* | Replaced by `eval_scenario_ids`. Kept for v1.x compatibility. |

`eval_scenario_ids` entries must reference scenario `id` values present in the
`scenarios` array (see §6.4). Each referenced scenario must be of type `eval_sequence`.

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
9. **Scenario `id` values must be unique** within the `scenarios` array.
10. **Hand-based scenarios** (`best_starting_hand`, `perfect_game`) must include
    `opening_hand` with 7 entries. `perfect_game` scenarios should have 10 turn entries;
    `best_starting_hand` should have 3.
11. **Precondition-based zone requirements** must include at least one of
    `mechanic_groups` or `card_names`.
12. **`use_best_starting_hand: true`** in `simulation` requires a `best_starting_hand`
    scenario to be present; `use_perfect_game: true` requires a `perfect_game` scenario.
    *(Deprecated — prefer `eval_scenario_ids`.)*
13. **`eval_sequence` scenarios** must include `opening_hand`. `turns` is optional.
14. **`eval_scenario_ids`** entries must each match an `id` in the `scenarios` array,
    and the referenced scenario must be of type `eval_sequence`.
15. **Group references** (`{"group": "..."}`) in card references must name a
    `mechanic_group` key that exists in at least one card in the deck's `main` or
    `commander` section.
16. **`mode`** must be `"forced"` or `"look_for"` when present. In `forced` mode,
    all group references in `opening_hand` and `turns[].drawn` must be resolvable to
    at least one concrete card in the deck list.
17. **`mulligan.mana_base_min` must be ≤ `mulligan.mana_base_max`** when both are present
    (§6.1.5).

---

## 10. Complete Example

```json
{
    "format": "mtg-commander-decklist",
    "version": "1.2.0",
    "meta": {
        "deck_id": "atraxa-superfriends-v1",
        "deck_name": "Atraxa Superfriends",
        "format": "Commander",
        "colors": ["W", "U", "B", "G"],
        "created": "2026-03-11",
        "updated": "2026-04-12",
        "author": "Alice",
        "source_url": "https://moxfield.com/decks/atraxa-superfriends-v1",
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
                "mv0": 0.85,
                "mv1": 0.8,
                "mv2": 0.75,
                "mv3": 0.6,
                "mv4": 0.2,
                "mv5": 0.2,
                "mv6": 0.2,
                "mv7Plus": 0.2
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
            ],
            "mana_base_min": 3,
            "mana_base_max": 4
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
        ],
        "scenarios": [
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
                    { "turn": 2, "drawn": "Rhystic Study", "played": ["Arcane Signet", "Forest"] },
                    { "turn": 3, "drawn": "Deepglow Skate", "played": ["Plains", "Doubling Season"] }
                ]
            },
            {
                "id": "scenario_turn4_engine",
                "type": "mid_game",
                "name": "Turn 4 Counter Engine Active",
                "mode": "look_for",
                "preconditions": {
                    "description": "Turn 4+, commander in play, 6 mana available, 2 ramp pieces on board",
                    "mana_available": 6,
                    "turn_number": 4,
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
                },
                "focus": {
                    "mechanic_groups": ["counters", "proliferate"],
                    "description": "Demonstrates how the counter engine activates at full speed"
                }
            },
            {
                "id": "eval_7plus3_superfriends",
                "type": "eval_sequence",
                "name": "7+3 Superfriends Opener",
                "mode": "forced",
                "opening_hand": [
                    "Sol Ring",
                    "Arcane Signet",
                    "Command Tower",
                    {"group": "land"},
                    {"group": "proliferate"},
                    {"group": "counters"},
                    "Atraxa, Praetors' Voice"
                ],
                "turns": [
                    {
                        "turn": 1,
                        "drawn": {"group": "ramp"},
                        "played": ["Sol Ring", "Command Tower"]
                    },
                    {
                        "turn": 2,
                        "drawn": {"group": "proliferate"},
                        "played": ["Arcane Signet", {"group": "land"}]
                    },
                    {
                        "turn": 3,
                        "drawn": {"group": "counters"},
                        "played": [{"group": "land"}, {"group": "proliferate"}]
                    }
                ],
                "focus": {
                    "mechanic_groups": ["counters", "proliferate"],
                    "card_names": ["Atraxa, Praetors' Voice"],
                    "description": "Verify 7+3 superfriends curve with ramp into Atraxa"
                }
            }
        ],
        "simulation": {
            "target": "forge",
            "play_order": "random",
            "difficulty": "ultimate",
            "starting_life": 40,
            "eval_scenario_ids": ["eval_7plus3_superfriends"]
        }
    }
}
```

---

## 11. Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.5.1 | 2026-09-20 | Document §6.1.6 (Starting Hand Quality) — no schema change. Clarifies, for completeness, that MaMoFrontend's informational (non-keep/mulligan-deciding) Mana Curve/tier-ranking/synergy-chain bonuses are **not** offered as `mulligan` JSON fields, because two of the three (tier ranking, synergy chains) depend on MaMo-internal data (per-card Mechanic Graph tier assignments, inter-group enabling relationships) that this notation's `mechanic_groups` (a flat string-key list) has no field for at all. |
| 1.5.0 | 2026-09-20 | Add `mulligan.mana_base_min`/`mana_base_max` (§6.1.5) and validation rule 17 — a second, independent keep/mulligan model (Mana Base Band) that scores only lands and cheap mana-producing cards against a fixed band, with two separate verdicts (`Playable` vs. `Good AI hand`). Additive: does not remove or reshape any existing field, and does not extend to the compact `.dck` `AiHints=` encoding (§6.1.4), which is unaffected and still governs real Forge games via §§6.1.1–6.1.3 alone. |
| 1.4.0 | 2026-09-18 | Lower `mulligan.card_values.mv4`-`mv7Plus` defaults from `0.45`/`0.4`/`0.35`/`0.3` to a flat `0.2` (mana value 4+ is worth meaningfully less to see in an opening hand); add §6.1.1a (`{X}`-cost cards count `X=2` for curve lookup, scoring only — never the card's real mana value) and §6.1.1b (a land producing 2+ colors gets a 1.0-1.4× value multiplier scaled by how well its colors match the deck's own colored-pip distribution). Both are new scoring rules within "how to compute a hand's total value," not new top-level fields — no schema shape change beyond the `card_values` default-number updates. |
| 1.3.0 | 2026-09-12 | **Breaking:** `mulligan.card_values` (§6.1.1) widened from 4 CMC-bucketed keys (`cmc_0_to_2`/`cmc_3`/`other`) to a full 9-key per-mana-value curve (`mv0`-`mv6`, `mv7Plus`, plus `land`); add §6.1.4 note and worked example updates to match. No production decks had ever saved a `mulligan` block under the old shape, so no migration path is documented. |
| 1.2.0 | 2026-04-12 | Add `meta.source_url`; add `eval_sequence` scenario type (§6.4.5); add scenario `mode` field (`forced`/`look_for`, §6.4.1a); add card reference type `{"group":...}` (§6.4.1b); add `board_state` field (§6.4.4); add `simulation.eval_scenario_ids`; deprecate `use_best_starting_hand`/`use_perfect_game`; add validation rules 13–16 |
| 1.1.0 | 2026-03-31 | Add §6.4 Scenarios (hand-based + precondition-based) and §6.5 Forge Simulation Config; extend `deck_rules` with `scenarios[]` and `simulation`; add validation rules 9–12 |
| 1.0.0 | 2026-03-11 | Initial specification |

---

## 12. Legal

This specification is designed for Magic: The Gathering gameplay recording and
analysis. Magic: The Gathering is a trademark of Wizards of the Coast LLC.
