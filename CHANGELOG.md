# Changelog

All notable changes to the MTG Replay Notation specification will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.9.6] - 2026-09-21

### Changed
- **Streamlined Optical Card Recognition Architecture (§3.1.1 in `MTG-REPLAY-NOTATION.md`)**:
  - Decoupled physical card markers into a 3-tier architecture:
    1. **Deck ID Card:** QR code encoding permanent deck URL (`https://mamo.app/deck/<deck_id>`) scanned once at pre-game setup ("latest is greatest" active revision model).
    2. **Player ID Card:** QR/Data Matrix identifying player seat.
    3. **Card Footer Markers:** 8×18 Data Matrix encoding strictly steady slot numbers (`1..100` / `0..255`) with maximum ECC200 error correction redundancy.
  - Minimalist badge design: strictly Data Matrix and `#Slot` number (e.g. `[ 8x18 ] #15`) with zero card names, set codes, or descriptive elements.
  - Lower footer placement: positioned at `y = 882..920` on 672×936 master template, placing markers safely in the black margin below copyright text while clearing corner cuts and trimming tolerances.
  - Maintained backward compatibility for 24-bit 6-hex decoders.
- **`commander-decklist-spec.md` bumped to v1.6.1**:
  - Updated §5.4.2 to document streamlined slot-only Data Matrix architecture and lower margin placement (`y = 882..920`).
  - Updated `optical_ids` in `CardEntry` and `commander-decklist-schema.json` to accept numeric slot strings (`"15"`) and legacy 6-hex strings.
- **`live-game-capture-spec.md`**:
  - Updated §3.2.1 to reflect decoupled slot-only vision recognition and setup-stage Deck ID resolution.

## [1.9.5] - 2026-09-20

### Added
- **Steady Card Slot Allocation System (§3.1.2 in `MTG-REPLAY-NOTATION.md`)**:
  Specifies deterministic slot number (`1..100` / `1..255`) allocation across deck revisions:
  - Commander is assigned Slot #1 (Partner/Background #2).
  - Main deck cards assigned in order of addition (`revisionadded ASC`).
  - Card cuts free slot numbers into an available pool without shifting remaining cards down, preserving physical sleeve stability and proxy card validity.
  - Card additions claim the lowest available freed slot number from the pool, or $\max(\text{occupied}) + 1$.
  - Re-added cards claim a fresh lowest freed slot.
  - Multi-copy cards (basic lands) receive distinct individual slots (e.g. 12 Islands = `[81..92]`).
  - Revision snapshots copy `slot_numbers` forward automatically.
  - Deterministic lazy backfill via historical revision replay from Rev 1.
- **`commander-decklist-spec.md` bumped to v1.6.0**:
  - Added `meta.optical_deck_id` (integer `1..255`) for 8-bit deck identification.
  - Added `CardEntry.slot_numbers` (integer array) and `CardEntry.optical_ids` (6-hex string array).
  - Added §5.4 (Steady Sleeve Slot Numbers & Optical Barcodes) documenting the 24-bit Data Matrix encoding (`[deck: 8b][slot: 8b][player: 4b][0: 4b]`), footer margin placement (`y=872..910`), and MaMo proxy XML export schema (`<card><name>...</name><slot>...</slot><optical_id>...</optical_id></card>`).
- **`commander-decklist-schema.json`**:
  - Added `optical_deck_id` to `DecklistMeta`.
  - Added `slot_numbers` and `optical_ids` to `DecklistCard`.
- **`live-game-capture-spec.md`**:
  - Added §3.2.1 (Optical Barcode Scanning & Physical Card Identification) and updated §5.2 (Optical Barcode Ground-Truth Recognition) for direct card identity and player owner resolution.

## [1.9.4] - 2026-09-18

### Changed
- **`commander-decklist-spec.md` bumped to v1.4.0**: lowered `mulligan.card_values.mv4`-
  `mv7Plus` defaults from `0.45`/`0.4`/`0.35`/`0.3` to a flat `0.2` (mana value 4+ is worth
  meaningfully less to see in an opening hand), and added two new scoring rules to §6.1.1 —
  §6.1.1a (`{X}`-cost cards count `X=2` for curve lookup, scoring only, never the card's real
  mana value) and §6.1.1b (a land producing 2+ colors gets a 1.0-1.4× value multiplier scaled
  by how well its colors match the deck's own colored-mana-pip distribution). Updated
  `schema/commander-decklist-schema.json`'s description strings and both worked examples
  (the full example in the spec, and `examples/commander-decklist.json`) to match. §6.1.4
  (compact `.dck` `AiHints=` encoding) still never carries `card_values`, so real Forge games
  remain unaffected — only the JSON-driven consumers (MaMoFrontend, mamo-sim) apply these
  rules, same boundary as before.

## [1.9.3] - 2026-09-16

### Documentation
- **`MTG-REPLAY-NOTATION.md` — documentation caught up to the Forge fork's actual v1.9.2 generator
  output, which had shipped several fields this spec never documented.** Closes out item #4 of
  the Forge fork's `FORGE_REPLAY_REMAINING_CHANGES.md` process (the last outstanding item; #1-#3
  and #5 were the generator changes that shipped in the fork's v1.9.2, this doc just hadn't caught
  up): decided documentation, not the generator, is the source of truth to reconcile toward, since
  the generator behavior is what 4+ downstream consumers already read.
  - §7.3 `LIFE`: documented the real `cause` enum (`"damage_or_loss"` / `"gain"` / `"lifelink"` —
    the previous "card name or description" text implied free-form text, which was never
    accurate) and the `source`/`source_name` fields carried for `"gain"` and `"lifelink"`.
  - §7.3 `TRIGGER`: documented `granted_by`/`granted_by_name`, present when a triggered ability's
    host card received it from a different card's static ability.
  - §7.3 `DRAW`: added `source`/`source_name`. Also corrected the 1.9.1 discrepancy note, which
    turned out to itself be wrong: production `DRAW` events were never missing `obj`/`from`/`to`/
    `pos`/`visibility` — they carry those *and* `owner`/`controller` together. Only the "consumer
    must parse the player out of `to`" assumption was actually wrong.
  - §7.3 `RESOLVE`: corrected the 1.9.1 discrepancy note the same way — `stack` is not absent from
    production events, it is present and always the literal string `"unknown"`. Root cause: the
    exporter's `logPutOnStack()` method (which would populate a real stack-ID map consumed by
    `RESOLVE`) is fully implemented but has zero callers in any production code path — confirmed
    by instrumenting a real simulation (75/75 `RESOLVE` events carried `stack: "unknown"`, 0
    `PUT_ON_STACK` events emitted). Documented as inert rather than wired up: no downstream
    consumer needs a stack ID (`card`/`card_name` already identify what resolved), and threading
    real stack IDs through every "goes on the stack" code path in Forge's engine (casts,
    activations, triggers alike) was judged a disproportionate, higher-risk change for a field
    nothing reads.
  - No schema change: `schema/replay-schema.json`'s `data` field is already unconstrained per
    event type, so none of the above required a JSON Schema edit.

## [1.9.2] - 2026-09-14

### Documentation
- **`MTG-REPLAY-NOTATION.md` §12.3 — the triggered-ability ordering flagged in 1.9.1 is confirmed
  permanent, not a fixable generator bug.** Investigated by the Forge fork implementation team per
  the change-request process in `MaMo-Base`: `MagicStack.resolveStack()` runs
  `AbilityUtils.resolve(sa)` synchronously — which fires a triggered ability's `DRAW`/`LIFE`/etc.
  effects as a direct side effect — *before* `game.fireEvent(new GameEventSpellResolved(...))` is
  called a few lines later. Reordering this would mean deferring every effect class's
  event-firing until after resolution completes across the whole engine, not a formatter tweak.
  §12.3's documented "typical sequence" is now corrected to `TRIGGER → effect → RESOLVE` (matching
  reality) instead of carrying only a discrepancy callout against the old, wrong sequence, with the
  mechanism and its implication (a direct `source`/`source_name` field on effect events, tracked as
  a proposed Forge fork change, is the real fix — not anything resolvable by event ordering) spelled
  out inline. Still documentation-only, no schema change.

## [1.9.1] - 2026-09-14

### Documentation
- **`MTG-REPLAY-NOTATION.md` — flagged three confirmed spec-vs-reality discrepancies, no schema
  or behavior change.** Found while root-causing a real card-draw/life-gain misattribution bug in
  `new-backend`'s `gameLogsService.ts` (a downstream consumer): its original heuristic was built
  by reading this spec rather than real replay data, and got the event ordering backwards as a
  direct result.
  - **§7.3 `RESOLVE` event**: spec says `data: {"stack": "s1"}`, expecting a lookup through an
    earlier `PUT_ON_STACK` event. Every production replay inspected instead carries
    `data: {"card": "...", "card_name": "..."}` directly, and `PUT_ON_STACK` is never emitted.
  - **§7.3 `DRAW` event**: spec says `data: {"obj", "from", "to", "pos", "visibility"}`, expecting
    the drawing player parsed out of `to` (e.g. `"P1:hand"`). Every production replay inspected
    instead carries `data: {"owner": "P1", "card_name": "..."}` directly.
  - **§12.3 Triggered Abilities pattern**: spec documents `TRIGGER → RESOLVE → effect events`.
    Verified by tracing several real games turn-by-turn that the actual order is
    `TRIGGER → effect events → RESOLVE` for triggered (not directly-cast) abilities — the effect
    is logged *before* the `RESOLVE` that closes it, not after. §12.1 (directly-cast spells) is
    unaffected; verified separately to still follow the documented `RESOLVE → effect` order.
  - Added ⚠️ **Known Discrepancy** callouts at each location rather than changing the documented
    schema outright, since it's not yet decided whether the generator or this spec is the
    intended source of truth for each — tracked via the Forge fork's `FORGE_REPLAY_CHANGE_REQUEST.md`
    / `FORGE_REPLAY_REMAINING_CHANGES.md` process in `MaMo-Base`. No JSON Schema change:
    `schema/replay-schema.json`'s `data` field is already an unconstrained object per event type,
    so nothing there was ever enforcing the (incorrect) documented shape.

## [1.7.0] - 2026-09-12

### Changed
- **`commander-decklist-spec.md` §6.1.1 (breaking): `mulligan.card_values` widened from 4
  CMC-bucketed keys to a full per-mana-value curve.** Was `{land, cmc_0_to_2, cmc_3, other}`;
  now `{land, mv0, mv1, mv2, mv3, mv4, mv5, mv6, mv7Plus}` — one value per exact mana value
  0-6 plus a 7+ catch-all, so a deck's standard baseline can distinguish a 1-drop from a
  2-drop instead of lumping "CMC 0-2" together. `schema/commander-decklist-schema.json` and
  both worked examples (§6.1.3, the full decklist example, and
  `examples/commander-decklist.json`) updated to match. §6.1.4 (compact `.dck` inline
  encoding) is unaffected — it never carried `card_values`. Bumped the companion spec's own
  version to v1.3.0 (§11 Version History). No production decks had ever saved a `mulligan`
  block at the time of this change, so no migration path is documented — this is a genuine
  breaking change for any future consumer of the old 4-key shape, not a compatible extension.

## [1.6.9] - 2026-09-11

### Added
- **`commander-decklist-spec.md` §6.1.4 (new): Compact Inline Encoding (`.dck` `AiHints=`).**
  Documents the plain-text `MulliganThreshold$`/`MulliganOverride$` token format that carries
  §6.1's mulligan rule (`card_values`/`thresholds`/`card_overrides`) on a Forge `.dck` file's
  own `AiHints=` metadata line, for decks played without a companion `mtg-commander-decklist`
  JSON file present. Cross-references `forge-integration-guide.md` §12.5.5 for the underlying
  `AiHints`/`DecklistSpecPath` mechanism. Reflects an already-shipped implementation
  (`new-backend`'s `getPublicDeckForgeExport` as writer; Forge's own
  `forge.deck.DeckRulesConfig.fromInlineHints()` / `forge.ai.ComputerUtil.wantMulligan()` as
  reader, verified directly against Forge fork source) — this entry documents existing behavior
  rather than proposing new behavior. No change to the `mtg-commander-decklist` JSON schema or
  its own version (still v1.2.0) — `card_values` overrides remain JSON-only (§6.1.1), not
  expressible in this compact form.

## [1.6.8] - 2026-08-17

### Added
- **`MEMO-forge-jar-naming.md`** — short, action-oriented notice for mamo-Connector: §11's
  proposed Option A (build-timestamp-suffixed jar filename) shipped in `killriam/forge`
  `replay-Features` commit `d2bc12ba3dc`. States what changed, what mamo-Connector needs to do
  (glob instead of hardcoding the jar filename), and what didn't change (download URL, zip
  structure). §11 itself now points at this commit as resolved rather than proposed.

## [1.6.7] - 2026-08-17

### Added
- **`forge-integration-guide.md` §11 (new, proposal only): desktop release jar naming.** Forge's
  release jar filename is currently static across every release
  (`forge-gui-desktop-2.0.14-SNAPSHOT-jar-with-dependencies.jar`, unchanged until the next
  `versionCode` bump) - the direct cause of an already-hit "mamo-Connector launched a stale
  cached build" confusion, since nothing in the filename distinguishes one release from the next.
  Proposes 4 options (build-timestamp suffix / git-hash suffix / both / a sidecar version file)
  with a recommendation, and specifies the pattern-matching change mamo-Connector needs regardless
  of which option ships, since the filename becoming variable is the point. Nothing in Forge's
  build has changed yet - this is flagged for mamo-Connector's team before the Forge side moves.

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
