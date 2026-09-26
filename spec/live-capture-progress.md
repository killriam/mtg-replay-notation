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

## Recent Updates

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
