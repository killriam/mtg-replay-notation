# Live Game Capture & Multi-Modal Gamelog Reconstruction Specification

## Architecture & Protocol Specification v1.0.0

**Status:** Draft / Active Specification  
**Published:** August 2026  
**Purpose:** Standardized specification and architecture for converting physical, live-played Magic: The Gathering games (audio, video, and companion telemetry) into structured, deterministic [MTG Replay Notation](./MTG-REPLAY-NOTATION.md) logs.

**Related Specifications:**
- [MTG Replay & Learning Notation v1.6.0](./MTG-REPLAY-NOTATION.md)
- [Commander Decklist Notation v1.2.0](./commander-decklist-spec.md)
- [MTG State Evaluation Framework](./mtg-state-evaluation-spec.md)

---

## 1. Executive Overview & Problem Statement

Physical tabletop Magic: The Gathering (specifically Commander and 1v1 formats) generates rich strategic decisions, but lacks automated digital telemetry. While digital platforms (e.g. Arena, Forge) produce native event logs, live games traditionally rely on manual scorekeeping or memory.

The **Live Game Capture Service** bridges this physical-to-digital gap. By fusing:
1. **Verbal audio declarations** (spells cast, triggers announced, phase changes, targets declared),
2. **Visual playmat state tracking** (overhead camera detecting card movements, tap/untap orientation, facedown cards),
3. **Decklist domain priors** (known 100-card Commander decklists with Scryfall/BigQuery Oracle IDs, card names, and mechanics),
4. **Companion app telemetry** (life changes, turn counters, manual card markings), and
5. **Game Notes / Scribe artifacts** (in-game shorthand notes, life checkpoints, player annotations, post-game debrief notes),

the service reconstructs a deterministic, timestamped **MTG Replay Notation** document (`MtgReplayFile` with `log_l1` events and `views_l2` decision snapshots) directly compatible with `new-backend`, `MaMoFrontend` replay viewers, and `mamo-story-engine`.

---

## 2. System Architecture

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                       1. SENSORY INGESTION                                       │
├───────────────────┬───────────────────┬──────────────────────────┬───────────────────────────────┤
│    Audio Input    │    Video Input    │ Companion Telemetry Sync │   Game Notes / Scribe Stream  │
│(Room/Phone Mic WAV│(Overhead Playmat) │(Life/Turns/Drawn Check)  │(In-Game Notes/Debrief/Scribe) │
└─────────┬─────────┴─────────┬─────────┴────────────┬─────────────┴───────────────┬───────────────┘
          │                   │                      │                             │
          ▼                   ▼                      │                             │
┌───────────────────┐┌───────────────────┐           │                             │
│ 2. Biased ASR     ││ 3. Vision Tracker │           │                             │
│ (Whisper + Prompt)││(OCR, Taps, Zones) │           │                             │
└─────────┬─────────┘└────────┬──────────┘           │                             │
          │ Transcripts       │ Card Detections      │ Telemetry Anchors           │ Semantic Landmarks
          └─────────┬─────────┴──────────────────────┼─────────────────────────────┘
                    │                                │
                    ▼                                ▼
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                          4. MULTI-MODAL FUSION & STATE INFERENCE ENGINE                          │
│                                                                                                  │
│  ┌─────────────────────────┐  ┌──────────────────────────┐  ┌─────────────────────────────────┐  │
│  │   Decklist Priors &     │  │  NLP Action & Intent     │  │ Temporal Event Synchronizer &   │  │
│  │   BigQuery Oracle Index │  │  Extractor (German/EN)   │  │ Scribe Landmark Cross-Validator │  │
│  └────────────┬────────────┘  └────────────┬─────────────┘  └────────────────┬────────────────┘  │
│               │                            │                                 │                   │
│               ▼                            ▼                                 ▼                   │
│  ┌────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                            MTG Game State Machine & Rules Solver                           │  │
│  │  - Cross-validates verbal claims against written scribe notes and life totals               │  │
│  │  - Validates legal transitions (mana, card ownership, priority, triggers)                   │  │
│  │  - Disambiguates facedown morphs/manifests using decklist identity + scribe annotations     │  │
│  │  - Fills unstated implicit game actions (untap step, state-based actions)                  │  │
│  └────────────────────────────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────┬─────────────────────────────────────────────────┘
                                                 ▼
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                              5. REPLAY NOTATION EMITTER & CONSUMERS                              │
│                                                                                                  │
│   MTG Replay Notation v1.6.0 (`meta`, `card_index`, `initial_state`, `log_l1`, `views_l2`)      │
│   - POST /api/gamelog/upload (new-backend Postgres + BigQuery association)                       │
│   - MaMoFrontend: Replay Player, Turn Explorer, Strategy Evaluation                              │
│   - mamo-story-engine: AI Narrative Prose & Scene Image Generation                               │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Sensory Ingestion Layer

### 3.1 Audio Capture & Chunking
- **Audio Channels**: 
  - Room ambient audio (e.g. 16kHz/48kHz WAV mono/stereo) or direct device recording from `mamo-companion`.
  - Continuous streaming or sliding chunks (e.g. 2-minute rolling buffers with 10-second overlap to prevent sentence boundary cutoff).
- **Acoustic Environment Handling**:
  - Voice Activity Detection (VAD) to filter background room noise and table shuffling.
  - Speaker diarization to attribute spoken declarations to Player seats (e.g. Seat 1: David, Seat 2: Opponent).

### 3.2 Video Capture & Coordinate Normalization
- **Camera Position**: Top-down or high-angle angled webcam/phone camera covering the playmat.
- **Playmat Segmentation & Homography**:
  - Automatic or 4-point manual perspective correction warping the playmat into a canonical 2D coordinate grid.
  - Dedicated bounding zones per player:
    - `Zone_Battlefield_Active` (Creatures / Permanents)
    - `Zone_Lands` (Mana pool / Lands)
    - `Zone_Command` (Commander cards)
    - `Zone_Graveyard` & `Zone_Exile`
    - `Zone_Stack` (Center board for currently resolving spells)

### 3.3 Companion Telemetry Stream
- Real-time event timestamps sent from `mamo-companion`:
  - `TURN_CHANGED` (turn number, active player index)
  - `LIFE_UPDATED` (player ID, old life, new life, delta)
  - `POISON_ADDED`, `COMMANDER_DAMAGE_ADDED`
  - `CARD_DRAWN_MARKED`, `CARD_USEFUL_MARKED`
- These telemetry points serve as **high-confidence anchor timestamps** to synchronize audio-visual event inference.

### 3.4 Game Notes & Scribe Artifact Ingestion
Players frequently jot down in-game shorthand, turn notes, or post-match summaries (e.g. *"Warte, ich muss erst mal was schreiben: Plateau, Soaring... "* or logging commander casts and life milestones). These notes serve as a critical quality multiplier:
- **Formats Supported**:
  - **In-App Companion Notes**: Timestamped turn notes typed during play (`CompletedGame.notes`, `sessionNotes`).
  - **Structured Scribe Text / Markdown**: Scribe logs containing turn markers (`T1: Plateau -> Sol Ring`, `T4: Davros cast`, `T7: Phyrexian Metamorph -> Dalek Drone`).
  - **Post-Game Debrief Annotations**: Retrospective notes recorded immediately after match completion (e.g. winner, final turns, key combos).
- **Signal Value**:
  - Provides **ground-truth sequence landmarks** to disambiguate periods of quiet play, background noise, or overlapping speech.
  - Exposes **hidden/private information** intentionally recorded by the player (e.g. card tutored to hand, facedown morph card identity).

---

## 4. Domain Prior & Lexicon Biasing

Natural speech in Magic games is filled with jargon, shorthand, and multi-lingual mixing (e.g. German conversational syntax mixed with English MTG card names and keywords).

### 4.1 Dynamic Decklist Lexicon Injection
Before game ingestion begins, the session resolves the active decklists for all seats from `new-backend` / `mamo-types`:
```json
{
  "players": {
    "P1": {
      "deck_id": "deck-uuid-1",
      "deck_name": "Kadena Morph",
      "cards": ["Trail of Mystery", "Reluctant Roleworker", "Qarsi Deceiver", "Ainok Survivalist", "Deserted Beach", "Underground Sea", "Ugin's Mastery"]
    },
    "P2": {
      "deck_id": "deck-uuid-2",
      "deck_name": "Davros Dalek Clones",
      "cards": ["Davros, Dalek Creator", "Dalek Drone", "Phyrexian Metamorph", "Court of Vantress", "Training Center", "Sulfur Falls"]
    }
  }
}
```

### 4.2 ASR Context Biasing
1. **Vocabulary Prompt Injection**: The combined unique list of card names, mechanic keywords (`morph`, `manifest`, `monarch`, `cascade`, `connive`), and player names are fed into the speech-to-text decoder prompt (e.g. Whisper `initial_prompt` or decoding vocabulary constraint).
2. **Fuzzy Phonetic Matching**:
   - Phonetic representations (Double Metaphone / Soundex) match mispronounced or accented card names (e.g., German pronunciation of *"Trail of Mystery"*, *"Court of Vantress"*, *"Blasphemous Act"* / *"Blasphälose"*).

### 4.3 Notes-Driven Disambiguation & Cross-Validation
Written notes act as an authoritative tie-breaker for ambiguous phonetic transcriptions:
- If ASR transcribes *"Soaring"* or *"Radgau"*, but the game note contains `Sol Ring` and `Ragavan`, the fusion engine automatically resolves the card entity to `Sol Ring` / `Ragavan, Nimble Pilferer`.
- If audio volume drops during combat, written life changes (e.g. `P2: 40 -> 34`) confirm combat damage amounts.

---

## 5. Perception & Action Extraction

### 5.1 Verbal Dialogue Parsing
The Natural Language Action Extractor converts transcribed utterances into candidate intent objects:

| Spoken German Dialogue Example | Extracted MTG Action Candidate | Target Event Type |
|---|---|---|
| *"Ich spiele eine Flooded Grove, tappe die beiden um Trail of Mystery reinzubringen"* | Player 1 plays land `Flooded Grove`, taps 2 mana, casts `Trail of Mystery` | `PLAY_LAND`, `CAST` |
| *"Wenn ich Face-down Karten spiele, kann ich mir Basic Land suchen"* | Trigger declaration for `Trail of Mystery` | `TRIGGER` |
| *"2 im Blau und Morph aufdecken, dann counte ich Juna"* | Unmorph cost {U}{U}, unflip trigger targeting spell | `SPECIAL_ACTION`, `COUNTER` |
| *"Für drei Mana kaste ich Phyrexian Metamorph als Kopie von Dalek Drone"* | Cast `Phyrexian Metamorph`, choose clone target `Dalek Drone` | `CAST`, `CHOICE` |
| *"Ich greife mit den beiden an, diese beiden fliegen"* | Declare attackers with 2 flying creatures | `ATTACK` |
| *"End of turn trigger... du verlierst drei Leben"* | End step trigger resolution, life loss | `STEP`, `LIFE` |

### 5.2 Visual Board Tracking
- **Object Detection**: Detects rectangular card tokens, card orientation ($0^\circ$ untapped vs. $90^\circ$ tapped).
- **Facedown & Token Resolution**: Recognizes facedown cards on the battlefield; associates them with secret identity once revealed by audio or hand state.
- **Card Art Identification**: Feature matching (ORB/SIFT/CNN embedding) against the known card art pool of the loaded decklists.

---

## 6. Multi-Modal Fusion & MTG State Inference Engine

The Fusion Engine correlates timeline streams into a single unified event graph:

```
Audio Timestamp 20:45:12 ────────┐
"Für drei Mana Phyrexian         │
 Metamorph Kopie von Dalek Drone"│
                                 ├─────► [Fusion & Legality Engine] ─────► L1Event: CAST
Vision Timestamp 20:45:13 ───────┤       - Validates P2 has 3 mana open            (Phyrexian Metamorph)
Card placed into battlefield     │       - Validates Dalek Drone on board          L1Event: CHOICE
and tapped state change          │       - Emits L1 CAST + CHOICE + ETB TRIGGER    (Copy: Dalek Drone)
                                 │
Companion Timestamp 20:45:30 ────┘
Life total P1 decremented -3
```

### Rules & State Consistency Solver
- **Mana Verification**: Ensures mana pool or untapped lands can pay costs before confirming spell resolution.
- **Implicit Actions**: Automatically inserts required system events that players don't explicitly announce (e.g. `UNTAP_STEP`, `DRAW_STEP`, graveyard movement when creature dies).
- **Confidence Scoring**: Each generated event receives a confidence score ($0.0 - 1.0$). Events below threshold are flagged for interactive human review in `MaMoFrontend`.

---

## 7. Output Format: Replay Notation Mapping

The service outputs a standard `mtg-replay` JSON structure:

```json
{
  "format": "mtg-replay",
  "version": "1.6.0",
  "meta": {
    "game_id": "live-capture-20260828-01",
    "timestamp": "2026-08-28T19:32:00+02:00",
    "game_type": "Commander",
    "source": "live_multimodal_capture",
    "players": {
      "P1": {
        "name": "David",
        "deck_name": "Kadena Morph",
        "deck_id": "80d573fb-4e12-4c22-b918-123456789abc",
        "starting_life": 40
      },
      "P2": {
        "name": "Brother",
        "deck_name": "Davros Clones",
        "deck_id": "d510bdcc-7f89-4b11-a890-abcdef123456",
        "starting_life": 40
      }
    },
    "winner": "P2",
    "win_condition": "life_zero"
  },
  "card_index": {
    "Trail of Mystery": {
      "oracle_id": "963c65e8-fb9e-4ebc-a81d-6169542a1766",
      "name": "Trail of Mystery",
      "cost": "{1}{G}",
      "type": "Enchantment"
    },
    "Phyrexian Metamorph": {
      "oracle_id": "14f2e519-7e44-42f3-8f0a-48d89e4726f1",
      "name": "Phyrexian Metamorph",
      "cost": "{3}{U/P}",
      "type": "Artifact Creature — Shapeshifter"
    }
  },
  "log_l1": [
    {
      "id": "evt_001",
      "timestamp": "19:32:15.200",
      "type": "PLAY_LAND",
      "player": "P1",
      "card_name": "Upland Palace",
      "status": "tapped"
    },
    {
      "id": "evt_002",
      "timestamp": "19:32:45.100",
      "type": "CAST",
      "player": "P1",
      "card_name": "Trail of Mystery",
      "mana_spent": "{1}{G}"
    }
  ]
}
```

---

## 8. Evaluation & Ground-Truth Benchmark Protocol

To benchmark and iterate on the reconstruction pipeline, the project maintains an authoritative test suite in:
`testing/fixtures/live-capture/2026-08-28-session/`

### Evaluation Metrics:
1. **Card Entity Recognition Precision & Recall**: Accuracy of card name detection against known decklist.
2. **Turn & Phase Alignment Accuracy**: Correct identification of active turn player and phase boundaries.
3. **Action Event Accuracy**: Precision of `CAST`, `ACTIVATE`, `ATTACK`, `DAMAGE`, `LIFE` events compared against human ground truth.
4. **State Consistency Score**: Percentage of game turns where reconstructed board state remains 100% legal under MTG Comprehensive Rules.
