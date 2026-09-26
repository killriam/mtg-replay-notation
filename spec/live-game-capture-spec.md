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

## 2.1 Sequential Technical Flow

```mermaid
sequenceDiagram
    autonumber
    actor User as Player / User
    participant Ingestion as 1. File & Event Ingestion
    participant Proposer as 2. Deck Proposer & Identifier
    participant Analyzer as 3. Multi-Modal Analysis Engine
    participant CardDB as BigQuery MTG Card Catalog
    participant Emitter as 4. Replay Gamelog Emitter
    participant Scribe as Scribe / Ground Truth Verifier

    %% Step 1
    Note over User,Ingestion: Step 1: Reference all capture files of event
    User->>Ingestion: Ingest audio WAVs, video frames, Companion telemetry
    Ingestion->>Ingestion: Build synchronized media timeline & chunk index

    %% Step 2
    Note over Ingestion,Proposer: Step 2: Identify decks with user proposal
    Ingestion->>Proposer: Initial audio/visual scan for commander names & mechanics
    Proposer->>User: Propose Deck A (Seat 1) & Deck B (Seat 2) from MaMo / Moxfield
    User-->>Proposer: Confirm / adjust deck selections & revision IDs

    %% Step 3
    Note over Proposer,Analyzer: Step 3: Analyzing files (90/10 Decklist Prior + CI Fallback)
    Proposer->>Analyzer: Inject confirmed 100-card lexicons & Commander Color Identities
    Analyzer->>Analyzer: ASR transcription + Vision detection of plays & damage
    alt 90% Case: Card found in Decklist
        Analyzer->>Analyzer: High-confidence card resolution from deck prior
    else 10% Case: Recent swap / Not in Decklist
        Analyzer->>CardDB: Fallback search constrained strictly to Commander Color Identity
        CardDB-->>Analyzer: Candidate card matches within legal colors
    end

    %% Step 4
    Note over Analyzer,Emitter: Step 4: Propose gamelogs with confidence score
    Analyzer->>Emitter: Generate timestamped L1Event stream (Main cards played + Damage)
    Emitter->>User: Propose MtgReplayFile JSON with per-event confidence scores (0.0 - 1.0)
    
    %% Verification
    Note over Emitter,Scribe: Independent Verification
    Emitter->>Scribe: Cross-validate proposed gamelog against handwritten Signal/Scribe notes
    Scribe-->>User: Accuracy & state-consistency score report
```

---

## 2.2 Technical Pipeline Breakdown

### Step 1: Reference All Capture Files of the Event
- Ingest and index all media components belonging to the match:
  - Audio WAV chunks with sample-rate normalization (e.g. 16kHz mono).
  - Overhead playmat video files / synchronized frame buffers.
  - Companion telemetry logs (turn increments, life total timestamps).
- Builds a unified multi-stream timeline: $T = [t_0, t_{\text{end}}]$.

### Step 2: Deck Identification & User Proposal
1. **Pre-scan**: Rapid keyword search across the initial audio/video segments for commander declarations (e.g. *"Kadena"*, *"Yuma"*, *"Davros"*, *"Mr. House"*).
2. **Proposal Generation**: Queries MaMo backend and Moxfield reference decks to present a candidate match setup:
   - Seat 1: `David` $\rightarrow$ Deck: `Kadena Morph` (Revision `rev-001`).
   - Seat 2: `Brother` $\rightarrow$ Deck: `Davros Dalek Clones` (Moxfield ID `xyz123`).
3. **User Confirmation**: The user confirms or adjusts the deck assignment before full multi-modal analysis begins.

### Step 3: Multi-Modal File Analysis (The 90/10 Rule & Color Identity Constraints)
- **90% Decklist Prior**:
  - The analysis assumes **$\ge 90\%$ of played cards exist in the confirmed decklist**.
  - ASR decoder and vision matcher prioritize the $\sim 100$ card entities of that player's deck.
- **10% Fallback (Color-Identity-Constrained BigQuery Search)**:
  - If a player plays a card not in the decklist (e.g. a recent card swap or tech choice):
  - The system initiates a fallback search against the complete BigQuery MTG Card Catalog (`magicmoments-442919.MtgCards.cards_transformed`).
  - **Constraint Rule**: The search is strictly filtered by the commander's **Color Identity**:
    $$\text{Card.ColorIdentity} \subseteq \text{Commander.ColorIdentity}$$
- **Critical Action Guarantee**:
  - Even if minor micro-actions (e.g. unannounced mana floating, priority passes) are omitted, the engine guarantees capturing:
    1. **Every main card played** (`PLAY_LAND`, `CAST`, `ACTIVATE`).
    2. **All damage created and life total changes** (`DAMAGE`, `LIFE`, `ATTACK`).

### Step 4: Propose Gamelog with Per-Entry Confidence Scores
- Reconstructs the game as a standard `mtg-replay` v1.6.0 document.
- **Confidence Scoring per Entry**: Every L1 event includes a `confidence` field ($0.0 - 1.0$) and `source` flags:
  ```json
  {
    "id": "evt_042",
    "timestamp": "20:45:12.300",
    "type": "CAST",
    "player": "P2",
    "card_name": "Phyrexian Metamorph",
    "confidence": 0.96,
    "source": "audio_declaration_and_vision_placement",
    "in_decklist": true
  }
  ```
- **Role of Game Notes**:
  - Game notes (from Signal or hand-written pads) are **not used during initial perceptual investigation**; they serve purely as **independent post-analysis verification** to measure reconstruction quality and benchmark model accuracy.

---

## 3. Sensory Ingestion Layer

### 3.1 Audio Capture & Chunking
- **Audio Channels**: 
  - Room ambient audio (e.g. 16kHz/48kHz WAV mono/stereo) or direct device recording from `mamo-companion`.
  - Continuous streaming or sliding chunks (e.g. 2-minute rolling buffers with 10-second overlap to prevent sentence boundary cutoff).
- **Acoustic Environment Handling**:
  - Voice Activity Detection (VAD) to filter background room noise and table shuffling.
  - Speaker diarization to attribute spoken declarations to Player seats (e.g. Seat 1: David, Seat 2: Opponent).
- **Known Limitation — No Working Diarization in Practice**: both archived room-mode datasets to date (`2026-08-28-session`, `2026-09-26-session`) were captured with a single ambient mic and carry **zero speaker labels** in the raw ASR output; diarization above is aspirational until Phase 6 hardware/software lands. Until then, the fusion engine MUST fall back to **Name-Mention Anchoring**: attribute an action to a player only when that player's name (or an unambiguous "du"/"you" addressed to a previously-named target) appears in the same or an adjacent utterance; otherwise emit the event with `player: "unattributed"` rather than guessing from turn-order heuristics or vocal-tone continuity. A 5-player, 3+ hour real session (`2026-09-26-session`) showed that guessing here produces confidently wrong attributions (see 3.5.1) that are worse than an honest "unattributed" tag.

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

### 3.2.1 Optical Barcode Scanning & Physical Card Identification
- **8×18 Data Matrix (ECC200) Detection**:
  - Top-down playmat camera scans detect the compact 8×18 rectangular Data Matrix badge located in the card's lower footer margin (`y = 879..917`, below copyright notice).
  - Decodes either the primary slot numeric string (e.g. `"15"`) or legacy 24-bit 6-hex payload (`"<DECK:2><CARD:2><PLAYER:1><FLAGS:1>"`, e.g. `"2A0F00"`).
- **Direct Card & Owner Resolution**:
  - Maps `card_in_deck` directly into the deck's steady slot allocation table (`slot_numbers`), achieving $100\%$ card identity confidence without visual art ambiguity or OCR error.
  - Resolves player ownership and deck context from the pre-game Deck ID QR card scan (`https://mamo.app/deck/<deck_id>`) or match lobby seat assignment.

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

### 3.4.1 Standardized Printable Telemetry Sheet (Commander Game Note Sheet)
For players opting for low-friction physical analog logging during tabletop Commander games (especially when life totals, pod damage, and standings are managed digitally via companion apps such as **Playgroup.gg**), MaMo provides an integrated, printable personal telemetry half-sheet:

- **Format & Layout**: Standardized **DIN A5 half-sheet (148 × 210 mm)**. Available as:
  - *Page 1 (2-Up)*: Two complete game logs side-by-side on standard DIN A4 landscape (297 × 210 mm) with a central dashed cut line (`x = 148.5 mm`).
  - *Page 2 (Standalone)*: Single DIN A5 portrait layout formatted for small clipboards, A5 paper trays, or digital e-ink tablets (e.g. iPad, reMarkable).
- **Application Integration**: Downloadable directly within `MaMoFrontend` via **Deck Finishing** > **Command Tower** dropdown (`Data & Print` > `📋 Game Note Sheet (PDF)`), and permanently hosted at `/templates/commander_game_note_sheet_a5.pdf`.
- **Recorded Fields & Schema Mapping**:
  1. **Minimal Header**: `DECK`, `DATE`, `MATCH #`.
  2. **Starting Hand Box (8 Cards)**:
     - `Mulligan down to: [   ]`: Write-in box for final kept card count.
     - `Opening Lands: [   ]`: Initial mana base count.
     - `Slots 1–8`: Two 4-column rows (Slots 1–4 and 5–8) capturing the 7-card opening hand plus the guaranteed Turn 1 draw step (or partner commanders).
  3. **Turn-by-Turn Action Table (Turns 1–14)**:
     - `TURN`: Round indicator (`T 1` .. `T 14`).
     - `CARDS DRAWN`: Dotted handwriting baseline for drawn cards or velocity (`+1`, `+3`). Cards in hand are derived deterministically as $(\text{Total Drawn} - \text{Total Played})$.
     - `LAND PLAYED`: Dotted baseline for land drops.
     - `CARDS PLAYED (Name / #)`: Dual baselines for cast spells or 8×18 Data Matrix slot numbers (`#1..#100`).
     - `KEY ACTIONS / COMBAT / NOTES`: Dual baselines for combat attacks, targets, triggers, and counterspells.
  4. **Post-Game Notes & Deck Tuning**:
     - `MVP (Name or Slot #)`: Card that overperformed or enabled the win condition.
     - `Cut Candidate (Name or Slot #)`: Underperforming card or dead draw.
- **Multimodal Reconstruction Role**:
  - The rigid, bounding-box-aligned geometry enables mobile camera capture and OCR segmentation (`scripts/ocr_signal_game_notes.py`), feeding verified turn anchors directly into the Multimodal Fusion & State Inference Engine.

### 3.4.2 Game State Evaluation Sheet (End-of-Round Table Discussion & Vector Rankings)
To evaluate multiplayer board progression, relative advantages, and game dynamics without cluttering personal deck notes, MaMo provides a dedicated companion sheet for end-of-round table discussions:

- **Format & Layout**: Standardized **DIN A5 Landscape (210 × 148 mm)**. Formatted as:
  - *Page 1 (2-Up)*: Two complete game state sheets top-and-bottom on standard DIN A4 portrait (210 × 297 mm) with a central dashed cut line (`y = 148.5 mm`).
  - *Page 2 (Standalone)*: Single DIN A5 landscape page for native A5 printing or digital stylus tablets.
- **Application Integration**: Downloadable via **Deck Finishing** > **Command Tower** dropdown (`Data & Print` > `🎯 Game State Evaluation Sheet (PDF)`), hosted at `/templates/commander_game_state_evaluation_sheet_a5.pdf`.
- **Recorded Fields & Schema Mapping**:
  1. **Pod Seating & Identification**: Assigns seat numbers: `P1 (You)`, `P2 (Left)`, `P3 (Across)`, `P4 (Right)` with player/commander names.
  2. **Early Game Development Block (Rounds 1–2 Side-by-Side)**:
     - Focuses exclusively on **Resource Potential** (mana base, lands, rocks, dorks, ramp acceleration).
     - R1 & R2 are placed side-by-side to optimize sheet space since complex combat, engines, and lethal threats have not yet developed.
     - Inside each round: 4 ranked rows (`1: P[ ]`, `2: P[ ]`, `3: P[ ]`, `4: P[ ]`) with generous note space for ramp cards (e.g. *Sol Ring*, *Mana Crypt*, *dork*, *missed land*).
  3. **Full State Evaluation Table (Rounds 3–10)**:
     - Four full state vector columns corresponding to the [MTG State Evaluation Framework](./mtg-state-evaluation-spec.md):
       - `RESOURCE POTENTIAL`: Mana production capacity & card advantage / hand size (Dimension 1 & 4).
       - `BOARD PRESENCE`: Battlefield footprint, power/toughness, blocker density (Dimension 2).
       - `DECK SYNERGY`: Engine assembly stage, active formations, archetype cohesion (Dimension 9 & Section 7).
       - `THREAT POTENTIAL`: Lethality clock, burst damage, inevitability / archenemy status (Dimension 5 & 10).
     - Inside each category cell: 4 dedicated rows for player ranking (`1: P[ ]`, `2: P[ ]`, `3: P[ ]`, `4: P[ ]`) with dotted lines for key reasons / permanents (e.g. `Rhystic`, `8 pwr`, `sac loop`, `lethal clock`).
- **Evaluation Framework Alignment**:
  - The turn-by-turn ordinal rankings $[\text{Rank}_{\text{Res}}, \text{Rank}_{\text{Brd}}, \text{Rank}_{\text{Syn}}, \text{Rank}_{\text{Thr}}]$ feed directly into the State Evaluation Framework's vector dominance and relative player advantage models.

### 3.5 Third-Party Digital Life-Tracker Cross-Validation (e.g. Playgroup.gg)

**Pattern derived from the `2026-09-26-session` dataset.** A large share of Commander pods already run a phone/tablet life-counter app (Playgroup.gg, or similar) alongside physical play, purely for life totals and turn timing — independent of any MaMo integration. Where the host or a player exports that app's own timeline/event log, it is a **near-zero-cost, high-confidence telemetry source** and should be ingested on par with (arguably above) Companion Telemetry Sync (§3.3), because it already exists without any capture-pipeline effort:

- **What it reliably provides**: exact life deltas per event, elimination order, the declared winner, and — critically — an unambiguous **game/match boundary** (one tracker "game" record = one physical game, regardless of how the room's conversation flows around it).
- **What it does not provide**: card names, spell/ability causality, or turn phase — it is a life/round skeleton, not a replay. It complements rather than replaces audio/vision perception.
- **Primary use in the Fusion Engine (§6)**:
  1. **Life-total resolution**: when audio/vision disagree or are silent on a life change, the tracker's logged delta is authoritative.
  2. **Elimination order & winner**: use the tracker's `ELIMINATED`/`GAME_END`/`lost the game` events as ground truth in preference to inferring "who died when" from combat banter.
  3. **Game-boundary correction**: treat a single tracker game record as one game, full stop. Do not let a lull in audio, a snack break, or an off-topic tangent (§3.6) be misread as a new game starting — that mistake is exactly what happened on first pass with the `2026-09-26-session` dataset, which was originally (and wrongly) split into two separate "games" around a conversational gap that a Playgroup.gg export later proved was mid-game.
- **Player-identity caveat**: third-party tracker usernames are **not** pre-mapped to MaMo user IDs or real names. Only a player who is independently confirmed (e.g. their own MaMo account, matched by email/login) can be resolved automatically; other seats' usernames should be carried through as opaque labels and only resolved to real names via explicit user confirmation (Step 2 of §2.1), never guessed from audio nicknames alone.
- **Reference fixture**: `testing/fixtures/live-capture/2026-09-26-session/playgroup_ground_truth.json` — a structured extraction of one such export, including an `audio_cross_references` block showing exactly where the audio-only reconstruction was corrected once this source was available. Treat this file the same way `replays/*.json` is used for the `2026-08-28-session` dataset (§8): as a benchmark target, not a training input.

#### 3.5.1 Case Study: What Went Wrong Without It

Before the tracker export was cross-referenced, the audio-only reconstruction of `2026-09-26-session`:
1. Split one continuous 5-player game into two separate "games" at a conversational lull, because Niv-Mizzet/Kenrith (present the whole game) only became audibly central late, and no audio signal marked a new game starting (no mulligan/shuffle dialogue, no reset chatter).
2. Misattributed a heard elimination line — `"Joni, du bist tot übrigens"` — to the wrong point in the game's actual timeline, purely because it had no way to anchor a spoken line to a specific tracker-confirmed round without diarization or a name→username mapping.
3. Assumed the player heard saying `"du hast noch zwei Leben"` (2 life) had lost, when the tracker showed that player's subsequent small-heal pattern and eventual win — audio alone captures a moment, not its outcome.

None of these were ASR quality failures (the actual card names and quotes transcribed correctly, see §4.3's homophone table for confirmed accurate resolutions from this same session: Karn/"Kahn", Pernicious Deed/"Precious Deeds", Niv-Mizzet/"Nif", Kenrith/"Ken Riff", Fatestitcher/"Fade Stitcher", Sigarda's Aid/"Zigardas-Aid"). They were **structural** failures — the kind only a second, independent event source can catch. This is the core argument for building tracker cross-validation into the pipeline rather than treating audio+decklist-prior ASR as sufficient on its own.

### 3.6 Non-Game Content Segmentation & Redaction

Room-mode ambient capture, by construction, keeps recording after the game ends and during any lull inside it. The `2026-09-26-session` dataset (a ~6-hour continuous capture) contained substantial stretches of pre-game banter, mid-session personal/family/political conversation, and — after the final game ended — several minutes of multilingual family conversation (English/Russian/Portuguese) unrelated to Magic in any way. None of this is useful signal for state inference, and some of it is content the players and bystanders did not consent to having retained as a "Magic session" artifact.

- **Pipeline requirement**: before persistence or display, classify each transcript window as `IN_GAME` vs. `OFF_TOPIC` (a cheap classification pass — presence of card names via decklist-prior lexicon, MTG action verbs, or turn-structure cues is a strong signal) and exclude `OFF_TOPIC` windows from both the state-inference engine's input and any long-term storage/display surface.
- **Do not use topic changes as game-boundary signals** — see §3.5's game-boundary correction above; a switch to off-topic conversation and back does not imply a new game.
- **Storage/consent implication**: raw audio and full transcripts of an ambient capture are a superset of "the game" and must be handled accordingly (e.g. not swept into a shared repo's git history via a broad `.gitignore` exception) until a redaction pass has run. This is a process gap identified during `2026-09-26-session` intake, not yet a pipeline stage — see [live-capture-progress.md](./live-capture-progress.md).

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
- **Optical Barcode Ground-Truth Recognition**: When proxy or printed cards with 8×18 Data Matrix barcodes are detected in the footer, the vision module bypasses heuristic feature matching and emits deterministic card identity with confidence `1.0`, cross-referencing the deck's `slot_numbers` table.

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
5. **Cross-Source Agreement Rate** (added after `2026-09-26-session`): where an independent source is available (third-party life tracker per §3.5, Forge replay JSON per `replays/*.json`), the percentage of life-total, elimination, and game-boundary events on which the audio/vision reconstruction agrees with that source before any manual correction. This is the metric that caught the `2026-09-26-session` game-boundary and elimination-attribution errors described in §3.5.1, and is the only metric of the five that scores *structural* correctness rather than per-event perception accuracy.
