# Live Capture Synthesis — Component Architecture

**Status:** Draft — supersedes the "wire into `/api/gamelog/synthesize`" approach taken in
the first two Phase 3c commits (`new-backend` `b33e65d`, `815d383`). Written after
recognizing that those commits coupled live-capture synthesis to the wrong existing
endpoint rather than questioning whether that endpoint was the right home at all.

---

## 1. Why the current placement is wrong

`liveCaptureSynthesizer.ts` and its `structuredEvents` request field currently live
inside `gameLogsController`/`game_logs` — the component whose actual job is **parsing
and storing well-formed digital replay files** (Forge exports: a complete,
already-structured `mtg-replay` JSON, uploaded and parsed as-is).

Live capture synthesis is a different job: **fusing messy, multi-source, confidence-laden
human/audio/photo data into something structured**, per this repo's own
`live-game-capture-spec.md` — ASR + decklist-prior matching, OCR, tracker-export
parsing, cross-validation, contested-fact flagging (§3.5, §5, §6). That's synthesis
from noisy inputs, not parsing of a trusted one. Concretely, bolting it onto
`gameLogsController` already caused:

- `SynthesizeGameLogRequest` (a `game_logs`-flavored type) had to grow an
  `ExtractedClaim`/`structuredEvents` field that has nothing to do with uploading a
  Forge replay — a live-capture concept leaking into a game-logs type file.
- The `saveToDatabase` path assumed a `game_logs` row was always the right place to
  persist a result, when a live-capture *recap* output (roster, dynamics, contested
  facts, source trail — see `game_recap_instance.md`'s 7-section template) actually
  belongs on `game_feedback_games.recap_data`, not `game_logs.parsed_replay`.
- Every future addition (vision/photo processing, ASR biasing, the OCR pipeline for
  the Commander Game Note Sheet) would keep growing `gameLogsController` instead of
  having its own boundary.

The underlying fusion logic (N-player support, real life/elimination tracking from
`ExtractedClaim`s) is correct and worth keeping — it's the **placement**, not the logic,
that needs to move.

## 2. Proposed component: Live Capture Synthesis

A distinct component whose only job is the transformation described in
`live-game-capture-spec.md` §5-§6:

```
Input:  raw/structured artifacts
        - transcriptText (ambient audio, ASR'd)
        - noteSheetText (OCR'd Commander Game Note Sheet)
        - trackerExport (third-party tracker JSON, e.g. Playgroup.gg)
        - handPhotoClaims (vision-derived hand contents)
        - players (roster: name, commander, deck)

Output: - ExtractedClaim[] (the common per-source format, S3.5.1's schema)
        - a synthesized mtg-replay JSON (log_l1/events, confidence-tagged)
        - a recap payload (roster w/ identity-resolution confidence,
          elimination order, winner, table-dynamics notes, contested/
          unresolved facts, source trail)
```

**What it does not own**: persistence. It is a transformation — artifacts in,
structured output out. Where that output gets stored is the caller's decision:

```
                    ┌─────────────────────────────┐
                    │   Live Capture Synthesis     │
                    │  (extraction + cross-        │
                    │   validation + confidence)   │
                    └───────────────┬─────────────┘
                                    │ returns ExtractedClaim[] +
                                    │ replay JSON + recap payload
                    ┌───────────────┴───────────────┐
                    ▼                                ▼
          caller may persist to              caller may persist to
          game_logs.parsed_replay            game_feedback_games.recap_data
          (technical event log)              (human-facing recap)
```

This mirrors the split already designed back in Phase 3c's data-model step: technical
log vs. human recap are different tables because they're different audiences, and
Live Capture Synthesis is the one thing that produces both from the same input, without
being responsible for where either one lands.

## 3. Where it lives — two options, not yet decided

**Option A — a bounded module inside `new-backend`.**
New folder, e.g. `new-backend/src/liveCapture/` (`liveCaptureController.ts`,
`liveCaptureService.ts` — the existing `liveCaptureSynthesizer.ts` logic moves here
largely unchanged), its own route namespace (`/api/live-capture/synthesize`, not
`/api/gamelog/synthesize`), its own types file instead of `types/gameLogs.ts`. Same
process, same deployment, but a clean interface boundary — `gameLogsController` no
longer imports it or knows it exists.
*Cost*: low — no new deployment, no new repo, mostly moving files and renaming a route.
*Limit*: still shares new-backend's deploy cadence and can't be scaled or iterated on
independently.

**Option B — a fully separate deployable**, matching the `mamo-live-capture` naming
already used in `live-capture-progress.md` and the precedent of `mamo-story-engine`
(its own Python FastAPI service in this workspace).
*Cost*: higher — a new repo, its own CI/deploy, a network hop from whatever calls it
(new-backend, or eventually the Connector directly for on-device OCR/ASR).
*Benefit*: matches where this is actually headed per the spec's own roadmap — Phase 4
(ASR/dialogue parsing), Phase 6 (computer vision playmat tracker) are both
compute-heavier, python-ecosystem-friendly workloads (Whisper, OpenCV/YOLO) that don't
naturally belong in a TypeScript/Express backend at all. Building Option A now and
splitting it into Option B later is a viable, lower-risk path if the heavier phases
land — the interface (artifacts in, `ExtractedClaim[]`/replay/recap out) doesn't change
between the two options, only where the process runs.

**My recommendation**: start with **Option A**. Phase 4/6's Python-ecosystem tooling is
still unplanned/unbuilt, so a full separate service today would be infrastructure ahead
of the workload that justifies it. Option A gets the architectural separation that
actually matters right now (out of `gameLogsController`, its own types) without the
operational cost of a new deployable, and the interface is designed so Option B remains
a pure infrastructure move later, not a redesign. This is a real decision for you to
make, not something to default on quietly a second time.

## 4. Migration of what's already built

Not a rewrite — a re-home:
- `ExtractedClaim` moves from `new-backend/src/types/gameLogs.ts` to a new
  `new-backend/src/types/liveCapture.ts` (or the new component's own types, under
  Option B).
- The N-player/`structuredEvents` logic added to `synthesizeGameLogFromReferences` in
  the last two commits is the correct fusion algorithm — it moves to the new module
  as-is; only its request/response types and route stop being `gameLogs`-flavored.
- `gameLogsController`'s `synthesize` handler and the `structuredEvents` validation
  branch get removed once the new endpoint exists — `game_logs` goes back to being
  only about parsing already-structured Forge replay uploads.

## 5. Open questions for you

1. Option A or B, per §3 — I'd default to A absent other input.
2. Should the new route be reachable today (even with no frontend caller yet, same as
   the current stub), or should it stay internal/unrouted until `GameRecapView` is
   ready to call it — avoiding a second half-finished public endpoint?
3. Do the existing two commits get reverted/moved now, or left as a known-temporary
   state until this is decided and the move happens in one pass?
