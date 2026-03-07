# MTG Replay Notation — Copilot Instructions

> Repo-specific instructions for the `mtg-replay-notation` specification.
> See the workspace-level `.github/copilot-instructions.md` for cross-repo architecture.

## Project Overview

This is a **specification repository**, not a code project. It defines a standardized JSON
format for recording and replaying Magic: The Gathering games. The spec supports two levels
of detail (L1 event log, L2 learning view) and is consumed by multiple MaMo components.

## Content & Structure

```
spec/
├── MTG-REPLAY-NOTATION.md          # Main specification document (v1.5.0, ~2145 lines)
└── mtg-state-evaluation-spec.md    # State evaluation framework (10 dimensions)

schema/
└── replay-schema.json              # JSON Schema (draft-07) for validation (~478 lines)

examples/
└── simple-game.json                # Reference implementation example

CHANGELOG.md                        # Version history (v1.0.0 → v1.5.0)
CONTRIBUTING.md                     # Change process, PR guidelines, style rules
README.md                           # Overview, features, quick start
LICENSE                              # MIT
```

## Cross-Repo Consumers

This spec is consumed by multiple MaMo components:

| Consumer | How it uses the spec |
|----------|---------------------|
| `mamo-Connector` | Captures game logs from Forge, validates against schema, uploads |
| `new-backend` | Stores replay JSON, validates uploads, serves to frontend |
| `mamo-story-engine` | Parses replays as input for narrative generation |
| `MaMoFrontend` | Displays replay data in UI, replay viewer component |

**Breaking changes to this spec affect ALL consumers.** Always follow the versioning process.

## Domain Knowledge

### Two-Level Architecture

- **L1 (Events)**: Raw chronological event log — each game action is an event object
  with `type`, `turn`, `player`, and action-specific fields
- **L2 (Learning View)**: Higher-level analysis units grouping related events for
  educational/review purposes

### Object ID Convention

| Prefix | Meaning | Example |
|--------|---------|---------|
| `c` + number | Card | `c1`, `c42` |
| `t` + number | Token | `t1`, `t5` |
| `s` + number | Stack item | `s1` |
| `P` + number | Player | `P1`, `P4` |

### Top-Level JSON Structure

```json
{
    "format": "mtg-replay",
    "version": "1.5.0",
    "spec_version": "1.5.0",
    "meta": { "game_id": "", "timestamp": "", "players": [], "winner": "", "turns": 0 },
    "seed": 12345,
    "game_start": { "starting_player": "P1", "mulligans": [] },
    "card_index": { "c1": { "name": "", "type_line": "", "oracle_id": "" } },
    "initial_state": { "libraries": {}, "hands": {}, "life_totals": {} },
    "events": [ { "type": "DRAW", "turn": 1, "player": "P1", "card": "c1" } ],
    "views_l2": [],
    "learning_markers": [],
    "per_turn_summary": [],
    "game_summary": {}
}
```

### Event Types

Core events: `GAME_START`, `DRAW`, `PLAY`, `CAST`, `ACTIVATE`, `ATTACK`, `BLOCK`,
`DAMAGE`, `DESTROY`, `DISCARD`, `EXILE`, `RETURN`, `COUNTER`, `TRIGGER`, `TAP`,
`UNTAP`, `MULLIGAN`, `PASS_TURN`, `GAME_END`

### State Evaluation Framework

10 dimensions for evaluating game state:
Board Presence, Card Advantage, Mana Efficiency, Threat Level, Life Total Delta,
Commander Damage, Synergy Activation, Removal Ratio, Tempo, Political Position

## Development Guidelines

### This is a SPEC repo — not code

- Changes are to Markdown documents, JSON Schema, and examples
- No source code compilation or build step
- "Testing" means validating examples against the schema
- Changes go through the CONTRIBUTING.md process

### Change Process (from CONTRIBUTING.md)

| Change type | Version bump | Examples |
|-------------|-------------|---------|
| Patch (x.x.+1) | Typos, clarifications, formatting | Fix wording in spec |
| Minor (x.+1.0) | New optional fields, new event types | Add `SCRY` event |
| Major (+1.0.0) | Breaking changes, field removals | Rename `log_l1` → `events` |

### Style Rules

- JSON: 4-space indentation, `snake_case` keys, double quotes
- Markdown: Max 100 chars per line (where reasonable), ATX headings (`#`)
- Examples: Must validate against `schema/replay-schema.json`
- Spec sections: clear heading hierarchy, one concept per section

### Schema Updates

When modifying `schema/replay-schema.json`:
1. Validate existing examples still pass
2. Add new required fields to the `required` array only if truly mandatory
3. Use `"additionalProperties": true` to allow extension
4. Add `"description"` to every new property
5. Update the version in both spec and schema

### Writing Spec Changes

- Be precise and unambiguous — this document is a contract
- Include examples for every new concept
- Document edge cases explicitly
- Reference related sections with internal links
- Update `CHANGELOG.md` with every change

## Testing

Validate examples against the schema:

```bash
# Using Node.js (ajv)
npx ajv validate -s schema/replay-schema.json -d examples/simple-game.json

# Using Python (jsonschema)
python -c "
import json, jsonschema
schema = json.load(open('schema/replay-schema.json'))
example = json.load(open('examples/simple-game.json'))
jsonschema.validate(example, schema)
print('Valid!')
"
```

## Documentation Policy

This repo IS documentation. All changes are to docs/spec/schema.
Keep the writing precise, technical, and consistent with existing style.

## AI Agent Guidelines

1. **This is a specification** — treat it like a standards document, not code
2. **Breaking changes affect 4+ consumers** — use minor/patch versions whenever possible
3. **Examples must validate** — always check `simple-game.json` against the schema after changes
4. **Object ID conventions are sacred** — `c` for cards, `t` for tokens, `P` for players
5. **`snake_case` for JSON keys** — never camelCase in the spec or schema
6. **Update CHANGELOG.md** with every spec change
7. **Follow CONTRIBUTING.md** — especially the version bump rules
8. **Be precise** — ambiguity in specs causes bugs in all consumers
9. **Add examples** for every new concept or field
10. **English only** — all spec text, examples, and documentation
