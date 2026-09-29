# Live Game Capture & Multi-Modal Reconstruction — Progress & Roadmap

## Overview

Tracking implementation milestones, benchmark datasets, and progress for the **Live Game Capture Service** (`mamo-live-capture`).

---

## Milestone Roadmap

| Phase | Milestone | Status | Key Deliverables |
|---|---|---|---|
| **Phase 1** | Ground-Truth Dataset & Fixture Archival |  **Completed** | 13 audio WAV clips + raw ASR transcripts + combined chronological transcript preserved in `testing/fixtures/live-capture/2026-08-28-session/`. |
| **Phase 2** | Architecture & Protocol Specification |  **Completed** | [Live Game Capture Spec v1.0.0](./live-game-capture-spec.md) detailing ingestion, decklist priors, ASR biasing, fusion, and replay emission. |
| **Phase 3** | Ground-Truth Replay Target (`ground_truth_replay.json`) | 🟡 **In Progress** | Manual human-annotated MTG Replay Notation L1 events for the 2026-08-28 games to serve as gold standard evaluation. Extended by a second, independently-sourced ground truth for `2026-09-26-session` — see Phase 3b. |
| **Phase 3b** | Third-Party Tracker Ground Truth & Second Dataset | ✅ **Completed** | `2026-09-26-session` archived (5-player Commander, ~6hr ambient capture) with a Playgroup.gg tracker export cross-referenced into `playgroup_ground_truth.json`. Drove three new spec patterns: §3.5 Third-Party Digital Life-Tracker Cross-Validation, the diarization-fallback note in §3.1, and §3.6 Non-Game Content Segmentation & Redaction. |
| **Phase 3c** | Multiplayer Synthesis & Frontend Recap Integration | 🟡 **In Progress, replanning** | N-player synthesis logic is built and tested, but was wired into the wrong host (`gameLogsController`/`game_logs`) — see [Live Capture Service Architecture](./live-capture-service-architecture.md) for the correction before continuing. See detail section below. |
| **Phase 4** | Decklist-Biased ASR & Dialogue Intent Parser | ⚪ *Planned* | Python/FastAPI module extracting card entities & game actions from spoken dialogue with Whisper vocabulary prompting. |
| **Phase 5** | Companion App & Live Telemetry Integration | ⚪ *Planned* | Audio capture stream / upload button & telemetry synchronization hooks in `mamo-companion`. |
| **Phase 6** | Computer Vision Playmat Tracker | ⚪ *Planned* | OpenCV/YOLO top-down playmat card detector, zone classifier, and tap/untap state detector. |

---

## Dataset Inventory (`testing/fixtures/live-capture/2026-08-28-session/`)

- **Date**: 2026-08-28 (~19:32–21:04 CEST)
- **Format**: 2-Player Commander Game Session
- **Participants**: David (Seat 1) vs. Brother (Seat 2)
- **Deck Archetypes Identified**:
  - **Game 1**: Kadena / Morph & Manifest vs. Counter-Stealing & Double-Pass triggers (`Trail of Mystery`, `Reluctant Roleworker`, `Cavern of Souls`, `Ugin's Mastery`, `Flooded Grove`, `Underground Sea`, etc.)
  - **Game 2**: Davros, Dalek Creator / Dalek Clones (`Davros`, `Dalek Drone`, `Phyrexian Metamorph`, `Court of Vantress`, `Training Center`, `Sulfur Falls`) vs. Morph/Manifest deck.
- **Audio Chunks**: 13 files, 2 minutes each (~120 MB total uncompressed WAV audio).
- **Transcript**: `transcript_combined.md` containing full German conversational dialogue with MTG card names and timing notes.

## Dataset Inventory (`testing/fixtures/live-capture/2026-09-26-session/`)

- **Date**: 2026-09-26 (~11:59-17:55 CEST, ~6hr continuous ambient capture, 746 speech segments)
- **Format**: 5-player Commander session (2 physical games; audio pacing suggested 3 before tracker cross-check corrected this to 2 — see below)
- **Participants**: David (`Killiram`, MaMo-account-confirmed) + 3-4 others without MaMo accounts (real-name-to-tracker-username mapping not yet confirmed)
- **Deck Archetypes Identified**: Superfriends/Oko/Karn/Teferi control (Game 1); a 5-player game spanning Niv-Mizzet Reborn, Kenrith the Returned King, Urza Chief Artificer, Niko Light of Hope (David's confirmed MaMo deck), and Esika God of the Tree (Game 2 — the eventual winner)
- **Artifacts**:
  - `transcript_combined.md` — combined ambient transcript, no speaker diarization.
  - `game_notes_scribe.md` — first-pass narrative summary (originally 3 games; superseded by the corrected 2-game structure).
  - `game_log.md` — corrected, timestamped event-log reconstruction with an explicit corrections/limitations section.
  - `playgroup_ground_truth.json` — independent Playgroup.gg tracker export, structured as round-by-round life/damage/elimination events with an `audio_cross_references` block showing exactly where it corrected the audio-only reconstruction.
- **Why this dataset matters**: it is the first dataset where a second, independent ground-truth source (rather than manual annotation) was cross-referenced against the audio reconstruction, and it caught real structural errors (game-boundary mis-split, elimination mis-timing) that per-event ASR accuracy metrics alone would not have surfaced. See [Live Game Capture Spec §3.5](./live-game-capture-spec.md#35-third-party-digital-life-tracker-cross-validation-eg-playgroupgg).

---

## Phase 3c: Multiplayer Synthesis & Frontend Recap Integration

**Confirmed scope decisions**: this feature is **playgroup-scoped, not deck-scoped**, and its
recap data lives alongside the existing **`game_feedback_*` (live game feedback) tables**,
not `game_logs` (which stays the technical replay layer) or a new top-level table.

**Correction (2026-09-29)**: the section immediately below, "build on this, don't replace
it," was wrong about *what* to build on. `/api/gamelog/synthesize` existing and being
reachable made it the path of least resistance, not the right architectural home -
`gameLogsController`/`game_logs` is for parsing already-structured Forge replay uploads,
not for fusing messy multi-source live-capture data. See
[Live Capture Service Architecture](./live-capture-service-architecture.md) for the
corrected design (a distinct Live Capture Synthesis component) before doing any more
work against this endpoint. The fusion logic itself (N-player support, real
life/elimination tracking) is correct and is being re-homed, not rewritten.

### The existing connection point (build on this, don't replace it)

Direct code inspection found this is further along than expected:
- **Frontend**: `MaMoFrontend/src/components/gamelogs/UploadReplayModal.tsx` already has a
  "Synthesize from References" tab (`notesText` + `transcriptText` inputs, per-player
  name/commander fields) wired to `synthesizeGameLog()` in `src/services/gameLogsApi.ts`.
- **Backend**: that calls `POST /api/gamelog/synthesize` (`gameLogsController.ts:1155`) →
  `new-backend/src/services/liveCaptureSynthesizer.ts`'s `synthesizeGameLogFromReferences()`,
  which already has a `PHONETIC_HOMOPHONE_MAP` (the same idea as this project's ASR-homophone
  tables, built independently) and a note-line parser, emitting a standard `MtgReplayFile`.

**This module is real and routed, but its logic is a 2-player demo stub** — literally seeded
with `2026-08-28-session`'s example data (Kadena/Davros/Gwen Stacy/Mr. House). Hardcoded:
`P1`/`P2` only, `winner: 'P1'` unconditionally, `life: 40` fixed throughout (not computed from
actions), fabricated per-turn stats. The request type
(`SynthesizeGameLogRequest.players: Record<string, {...}>`) is already keyed by arbitrary
string, so generalizing to N players is a service-logic change, not a type/schema break.

### Gaps confirmed by direct exploration (not assumed)

- No multiplayer support, identity-resolution concept, round-by-round tracker
  cross-referencing, note-sheet ingestion, table-dynamics capture, contested/unresolved
  flagging, or source-trail concept anywhere in this chain.
- No page for any of this reachable from top-level navigation — the only existing path
  (`Evolvement/Learning` → `EvaluationTab` → `GameLogReviewPanel` → `Evolvement/Assessment` →
  `GameLogViewer`) is deck-scoped and also bakes in a 1v1 assumption
  (`GameLogReviewPanel.deriveGameLogSummary()` hardcodes `players[1]` as "the opponent").
- `PlaygroupDetail.tsx` (the confirmed target home) has exactly 4 tabs
  (`overview | members | rules | decks`) and zero games/sessions concept, even though
  `game_feedback_sessions.playgroup_id` already exists as an unused nullable FK.
- **Anti-pattern to avoid**: `MaMoFrontend/src/components/gamelogs/GameLogList.tsx` is a
  fully-built, fully-typed, barrel-exported table component that is never imported by any
  page — confirmed dead code. Whatever ships here needs its navigation entry in the same
  change that adds it, not as a follow-up.

### Plan

1. ✅ **Done (2026-09-27).** **Data model** (additive): added `game_feedback_games.recap_data
   jsonb` for the identity-resolution roster, table-dynamics entries, contested/unresolved
   notes, and source trail. `game_logs.parsed_replay` keeps holding the technical
   `MtgReplayFile` round/turn events — no change to that split.
   **Correction to this plan's original wording**: the actual self-applying mechanism isn't a
   true server-boot hook — it's `GameFeedbackService.ensureTables()`
   (`new-backend/src/services/gameFeedbackService.ts`), a lazy idempotent
   `CREATE TABLE IF NOT EXISTS` / `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` block called at
   the top of every `GameFeedbackController` request handler. Same practical effect
   (self-applying, no manual migration step to run), just triggered per-request rather than
   once at process start. Verified against a real local Postgres
   (`docker compose -f docker-compose.local.yml`), not assumed. Type added:
   `GameFeedbackGameRow.recap_data?: unknown` in `new-backend/src/types/gameFeedback.ts`
   (shape intentionally left loose until step 2's writer exists; not added to `mamo-types`).
   **Deliberately not done yet**: populating `game_feedback_sessions.playgroup_id` — scoped
   out of step 1 on purpose, not forgotten.
2. **Backend**: generalize `liveCaptureSynthesizer.ts` from `P1`/`P2` to
   `Object.keys(request.players)`, compute life/elimination from parsed actions instead of
   hardcoding, and add an optional tracker-export / note-sheet-OCR input alongside
   `notesText`/`transcriptText`, mirroring the 3-source fusion this project proved out by hand
   for `2026-09-26-session`. Persist the new `recap` object into `recap_data`.
3. **Frontend**: new `GameRecapView.tsx` in `src/components/gamelogs/`, composing a new
   roster/identity section, a new outcome/elimination summary, `GameLogViewer`'s existing
   `TurnAccordion`/`groupTurnEvents` for the round-by-round log (reuse, don't reinvent), and a
   new dynamics/contested/sources block. Wrap in `PageLayout`, reuse `Button`/`Modal` from
   `src/components/common/` — there is no shared Badge/Table/Tabs component in this codebase
   (confirmed); hand-rolled badge styling is the existing convention, not a gap to fix first.
4. **Navigation**: add a 5th tab to `PlaygroupDetail.tsx`'s `TabType` union
   (`"overview" | "members" | "rules" | "decks" | "games"`) — the first actually-discoverable
   top-level entry point for any of this.

**Sequencing**: (1) schema + type extensions first (small, safe, no UI change) → (2) generalize
the synthesizer to N real players (the biggest self-contained logic lift) → (3) build
`GameRecapView` reachable by direct logId for testing → (4) wire the playgroup tab once the
view is solid.

---

## Recent Updates

### 2026-09-27
- Investigated real MaMoFrontend/new-backend integration points for a "Gamelog by Transcript
  & Notes" recap feature and found the connection point already exists and is routed
  (`UploadReplayModal.tsx` → `/api/gamelog/synthesize` → `liveCaptureSynthesizer.ts`), just
  implemented as a 2-player demo stub rather than a general N-player synthesizer.
- Wrote up the integration plan as **Phase 3c** (above), with scope confirmed as
  playgroup-scoped (not deck-scoped) and recap data living on the `game_feedback_*` tables
  (not `game_logs`, not a new table).
- Built a static design mockup of the resulting recap page (an Artifact, not committed to
  either repo) to validate the information architecture before writing real code against it.
- **Started implementation**: Phase 3c step 1 done — `game_feedback_games.recap_data jsonb`
  added and verified against a real local Postgres via the `local-dev-environment` subagent
  (`new-backend` commits `c14cdd6`, `9058796`, local only, not pushed). Corrected this doc's
  own description of the self-applying schema mechanism in the process — see the plan's
  step 1 for the accurate version.
- Assembled `testing/fixtures/live-capture/2026-09-26-session/recap-inputs/` — a
  self-contained, re-runnable input bundle (transcript, transcribed note-sheet text, tracker
  export, confirmed player roster, README) for regenerating this session's recap later,
  whether by hand or once Phase 3c's synthesizer supports it.

### 2026-09-26
- Archived a second real-world dataset (`2026-09-26-session`): 5-player Commander, ~6hr ambient capture, no diarization.
- Cross-referenced an independent Playgroup.gg tracker export (`playgroup_ground_truth.json`) against the audio-only reconstruction and used the disagreement to correct it: two audio-perceived "games" were actually one continuous game, and an elimination line ("Joni, du bist tot") had been placed at the wrong point in the timeline relative to the eventual winner's near-death moment.
- Confirmed one player's decklist directly against the MaMo production database (`mamo.commanderdeckbase` / `deckrevision` / `deckcardlist`) rather than guessing it from audio — the only one of five seats with a MaMo account.
- Derived and wrote up three new patterns into [Live Game Capture Spec v1.0.0](./live-game-capture-spec.md): §3.5 Third-Party Digital Life-Tracker Cross-Validation (+ §3.5.1 case study), a diarization-fallback note in §3.1 ("Name-Mention Anchoring"), and §3.6 Non-Game Content Segmentation & Redaction. Also added a 5th evaluation metric (§8: Cross-Source Agreement Rate) to `live-game-capture-spec.md`.
- **Open gap, not yet fixed**: the root MaMo-Base repo's `.gitignore` explicitly un-ignores `.md` files under `testing/**`, meaning this session's raw transcript (which contains off-topic personal/family/political content captured by the always-on ambient mic, per §3.6) is currently stageable/committable as-is. Needs a redaction pass or a tighter ignore rule before that risk is closed.

### 2026-08-29
- Archived 13 WAV audio clips and transcripts into permanent repo fixture path `testing/fixtures/live-capture/2026-08-28-session/`.
- Created structured session game notes fixture [`game_notes_scribe.md`](../../testing/fixtures/live-capture/2026-08-28-session/game_notes_scribe.md).
- Authored [Live Game Capture Spec v1.0.0](./live-game-capture-spec.md) incorporating Game Notes / Scribe stream ingestion and notes-driven disambiguation.
- Updated [MTG Replay Notation README](../README.md) documentation index.
