# MTG Replay Notation — Claude Code Instructions

> See `.github/copilot-instructions.md` for full conventions. This file adds Claude-specific guidance.

## Key Rules

1. **Specification repo** — not code. Changes are to Markdown, JSON Schema, and examples
2. **Breaking changes affect 4+ consumers** — prefer optional fields and minor versions
3. **`snake_case` JSON keys** — never camelCase
4. **Examples must validate** against `schema/replay-schema.json`
5. **Update CHANGELOG.md** with every change
6. **Follow CONTRIBUTING.md** for version bump rules
7. **English only**

## Consumers

Changes here affect: `mamo-Connector` (capture), `new-backend` (storage/validation),
`mamo-story-engine` (narrative input), `MaMoFrontend` (replay viewer).

## Object IDs

`c` + number = card, `t` + number = token, `s` + number = stack, `P` + number = player.
These conventions are used across ALL consumers — never change them without a major version bump.
