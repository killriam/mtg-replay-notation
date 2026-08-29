# Live Game Capture & Multi-Modal Reconstruction — Progress & Roadmap

## Overview

Tracking implementation milestones, benchmark datasets, and progress for the **Live Game Capture Service** (`mamo-live-capture`).

---

## Milestone Roadmap

| Phase | Milestone | Status | Key Deliverables |
|---|---|---|---|
| **Phase 1** | Ground-Truth Dataset & Fixture Archival |  **Completed** | 13 audio WAV clips + raw ASR transcripts + combined chronological transcript preserved in `testing/fixtures/live-capture/2026-08-28-session/`. |
| **Phase 2** | Architecture & Protocol Specification |  **Completed** | [Live Game Capture Spec v1.0.0](./live-game-capture-spec.md) detailing ingestion, decklist priors, ASR biasing, fusion, and replay emission. |
| **Phase 3** | Ground-Truth Replay Target (`ground_truth_replay.json`) | 🟡 **In Progress** | Manual human-annotated MTG Replay Notation L1 events for the 2026-08-28 games to serve as gold standard evaluation. |
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

---

## Recent Updates

### 2026-08-29
- Archived 13 WAV audio clips and transcripts into permanent repo fixture path `testing/fixtures/live-capture/2026-08-28-session/`.
- Authored initial [Live Game Capture Spec v1.0.0](./live-game-capture-spec.md).
- Updated [MTG Replay Notation README](../README.md) documentation index.
