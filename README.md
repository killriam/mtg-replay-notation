# MTG Replay Notation

A standardized JSON-based format for recording and analyzing Magic: The Gathering game sessions.

[![Version](https://img.shields.io/badge/version-1.1.0-blue.svg)](./spec/MTG-REPLAY-NOTATION.md)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](./LICENSE)

## Overview

MTG Replay Notation is a comprehensive format designed for:

- **Deterministic Replay** — Games can be replayed exactly with the same seed
- **AI/ML Training** — Structured data suitable for machine learning
- **Decision Analysis** — Evaluate player choices with before/after state snapshots
- **Human Readable** — Clear JSON structure for manual inspection

## Documentation

| Document | Description |
|----------|-------------|
| [Format Specification](./spec/MTG-REPLAY-NOTATION.md) | Complete v1.1.0 format specification |
| [State Evaluation Framework](./spec/mtg-state-evaluation-spec.md) | Multi-dimensional game state evaluation |
| [JSON Schema](./schema/replay-schema.json) | JSON Schema for validation |
| [Examples](./examples/) | Sample replay files |

## Quick Start

### File Structure

```json
{
    "format": "mtg-replay",
    "version": "1.1.0",
    "meta": { /* Game metadata */ },
    "seed": 1234567890,
    "card_index": { /* Card definitions */ },
    "initial_state": { /* Starting state */ },
    "log_l1": [ /* Event log */ ],
    "views_l2": [ /* Learning units */ ]
}
```

### Two-Level Architecture

| Level | Purpose | Use Case |
|-------|---------|----------|
| **L1** (Event Log) | Complete, lossless event history | Replay engine, debugging |
| **L2** (Learning View) | Decision-focused snapshots | AI training, coaching |

## Features (v1.1.0)

### Metadata
- Game type, players, winner
- `win_condition` — How the game ended (life_zero, commander_damage, etc.)
- `deck_name` and `deck_hash` — Deck identification

### Events
- 8 Player Decision Events (CAST, ACTIVATE, PLAY_LAND, etc.)
- 12 System Events (MOVE, DAMAGE, LIFE, RESOURCES, etc.)
- Human-readable `card_name` fields

### State Tracking
- Complete game state snapshots
- Object states with counters, attachments, status flags
- Zone contents with visibility rules

## State Evaluation Framework

The evaluation framework provides multi-dimensional game state assessment:

| Dimension | Description |
|-----------|-------------|
| Resources | Mana production, fixing, capacity |
| Board Presence | Battlefield strength and permanence |
| Tempo | Initiative and mana efficiency |
| Card Advantage | Net access to cards |
| Life Pressure | Clocks toward victory/defeat |
| Inevitability | Long-term advantage |
| Flexibility | Available options |
| Risk/Information | Exposure and information asymmetry |
| Synergy/Gameplan | Strategy cohesion |
| Explosiveness | Burst potential |

See [State Evaluation Specification](./spec/mtg-state-evaluation-spec.md) for formulas and details.

## Installation

### npm (coming soon)
```bash
npm install mtg-replay-notation
```

### Manual
Clone this repository and reference the specification documents.

## Validation

Use the JSON Schema to validate replay files:

```javascript
const Ajv = require('ajv');
const schema = require('./schema/replay-schema.json');

const ajv = new Ajv();
const validate = ajv.compile(schema);
const valid = validate(replayData);
```

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Submit a pull request

For major changes, please open an issue first.

## Related Projects

- [Forge MTG](https://github.com/Card-Forge/forge) — Open source MTG game engine
- [Scryfall API](https://scryfall.com/docs/api) — Card data source

## License

MIT License — see [LICENSE](./LICENSE)

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.1.0 | 2026-02-08 | Added win_condition, deck_name, RESOURCES event, card_name in events |
| 1.0.0 | 2025-12-20 | Initial specification |

---

**Magic: The Gathering is a trademark of Wizards of the Coast LLC.**
