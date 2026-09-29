# Live Capture Preprocessor — Interface Specification

**Status:** Draft / Active Specification
**Published:** 2026-09-29
**Purpose:** Defines the contract for a standalone, interchangeable "media-to-text"
component that sits upstream of Live Capture Synthesis in the pipeline described in
[Live Game Capture Spec v1.0.0](./live-game-capture-spec.md). This doc is
implementation-agnostic on purpose — any service that satisfies this contract can be
"pointed at" by MaMo, whether that's the reference implementation described here or a
different one entirely.

**Related Specifications:**
- [Live Game Capture Spec v1.0.0](./live-game-capture-spec.md)
- [Live Capture Service Architecture](./live-capture-service-architecture.md)

---

## 1. Purpose & Scope

This component's only job is turning raw media into raw text: **audio → text**,
**image → text**. Nothing more. It does not resolve card names, does not cross-
validate against other sources, does not anchor anything to a turn or round, does not
persist anything, and does not know what a "deck" or a "player" is beyond an opaque
vocabulary hint it's handed and passes through unexamined.

## 2. Relationship to §5 of the Live Game Capture Spec

`live-game-capture-spec.md` §5 ("Perception & Action Extraction") originally described
one combined job: raw media → structured, MTG-semantic claims. This spec splits that
into two components, refining §5 rather than contradicting it:

- **S5a — Live Capture Preprocessor (this document).** media → raw text (+ optional
  per-segment confidence). No MTG semantics.
- **S5b — Live Capture Synthesis** (already scoped in
  [live-capture-service-architecture.md](./live-capture-service-architecture.md)).
  raw text → `ExtractedClaim[]` / synthesized replay / recap. Owns card-name
  resolution, decklist cross-referencing, turn/round anchoring, and the
  cross-validation described in §6 of the main spec.

Nothing about Live Capture Synthesis's design changes because of this split — it still
consumes text (now, optionally, this component's output) and still owns everything
downstream of "I have raw text."

## 3. Design Principles

These were explicit product decisions, not defaults — recorded here so they don't get
re-litigated by a future implementer:

1. **Interchangeable microservice, not a library.** MaMo (or any caller) points at a
   deployment of this contract over HTTP. Different deployments can run different
   engines, different providers, even different implementations entirely, as long as
   they satisfy §4.
2. **Zero coupling to MaMo's other services.** No MaMo user authentication, no
   deck/user ID lookups, no calls back into `new-backend`. Stateless. Any domain
   context it needs (a card-name vocabulary) is passed in as plain data by the caller
   — never fetched by this component itself. Optional deployment-level hardening (an
   API key header) is out-of-band HTTP access control, not part of this contract, and
   is not user identification.
3. **Vocabulary hint is data-in, not a lookup.** `vocabulary_hint` biases recognition
   (analogous to Whisper's `initial_prompt`, per §4.2 of the main spec) but is never
   matched, resolved to an Oracle ID, or cross-referenced by this component. That
   stays entirely in Live Capture Synthesis.
4. **Engine-agnostic, dual-engine by design.** The contract supports multiple
   interchangeable engines per media type, selectable per request or by deployment
   default — not one hardcoded approach.

## 4. Interface Contract

### `POST /api/preprocess`

**Request:**
```json
{
  "media_type": "image",
  "media_base64": "<base64-encoded raw file bytes, no data-URL prefix>",
  "media_filename": "note_sheet_page1.jpg",
  "vocabulary_hint": ["Sol Ring", "Niv-Mizzet Reborn", "Kenrith, the Returned King"],
  "engine": "vision_llm"
}
```

| Field | Required | Notes |
|---|---|---|
| `media_type` | yes | `"audio" \| "image"` |
| `media_base64` | yes | Raw bytes, base64-encoded. Format is sniffed from content, not trusted from `media_filename`. |
| `media_filename` | no | Diagnostic/logging only. |
| `vocabulary_hint` | no | Plain `string[]` — card names, mechanic keywords, player names. Passed through opaquely (§3.3). |
| `engine` | no | One of the engine ids registered for `media_type`. Omitted → the deployment's configured default for that `media_type`. |

**Response (success):**
```json
{
  "success": true,
  "media_type": "image",
  "engine_used": "vision_llm",
  "extracted_text": "T1: Sol Ring\nT6: BOOM\n...",
  "confidence": 0.74,
  "segments": [
    { "text": "Sol Ring", "confidence": 0.97 },
    { "text": "[illegible]", "confidence": 0.10 }
  ],
  "processing_time_ms": 1830,
  "error": null
}
```

`segments` exists because legibility is bimodal, not uniform (`live-game-capture-spec.md`
§3.4.1: isolated key-action words read reliably, dense multi-item cells mostly don't).
A single scalar `confidence` would erase exactly that signal. No bounding boxes, no
turn/player labels — adding those would be re-absorbing the S5b job into S5a.

**Response (error):**
- `400` — malformed request or unknown `engine` id for the given `media_type`.
- `501` — no engine registered for the given `media_type` at all:
  ```json
  { "success": false, "error": "engine_not_configured", "message": "No engine registered for media_type 'audio'." }
  ```
  This is how an implementation that doesn't support a media type yet (e.g. this
  reference implementation's v1 audio deferral, §6) surfaces cleanly rather than
  crashing.

### `GET /health`

```json
{ "status": "ok", "engines": { "image": { "vision_llm": true, "tesseract": true }, "audio": {} } }
```

Lets a caller discover capabilities before calling `/api/preprocess` — which engines
are actually configured and reachable for which media types.

## 5. Engine Registry & Selection Semantics

An implementation registers zero or more engines per `media_type`. Each `media_type`
has a deployment-configured default engine id, used when the request omits `engine`.
A request naming an unregistered `engine` for that `media_type` gets `400`; a
`media_type` with no engines registered at all gets `501` regardless of whether
`engine` was specified.

## 6. Reference Implementation Scope (v1)

A reference implementation (`mamo-preprocessor`) exists for this contract with the
following honest v1 scope:

- **Image (`media_type: "image"`)**: two engines shipped —
  - `vision_llm` — a vision-capable LLM call, prompted to return strict per-line JSON
    (so `segments` reflects real model output) and to mark unreadable spans
    `[illegible]` rather than guess. Expected to be the stronger engine for this
    project's actual use case (handwritten Commander Game Note Sheets) — already
    informally validated this session (§3.4.1's legibility finding was produced by
    exactly this kind of read).
  - `tesseract` — a traditional OCR engine, included so the interface is genuinely
    swappable (not vision-LLM-with-a-vestigial-second-enum-value), as a free/local/
    no-API-key fallback, and as the likely better choice later for printed-text inputs
    this pipeline doesn't have yet. `vocabulary_hint` is wired via Tesseract's
    `--user-words` dictionary-biasing flag — documented honestly as having much less
    effect on cursive handwriting than on printed text.
- **Audio (`media_type: "audio"`)**: **no engine shipped in v1.** The contract fully
  supports it; a call returns the `501 engine_not_configured` response above. Real
  audio-to-text already happens today in this project's production path via an
  external tool ("Emmah"), outside this codebase, whose output text flows directly
  into Live Capture Synthesis unchanged. Building a real ASR engine here would be
  speculative infrastructure with nothing concrete to verify against yet.

## 7. Non-Goals

Explicitly not this component's job (all of it is Live Capture Synthesis's, per §2):
card-name / Oracle-ID resolution, decklist cross-referencing, turn/round anchoring,
cross-source validation or contested-fact flagging, MaMo user authentication,
persistence of any kind.

## 8. Deployment & Tech Stack (reference implementation)

Python 3.11 + FastAPI, matching this workspace's existing `mamo-story-engine`
submodule's toolchain (`pydantic-settings`, `httpx`, `uvicorn`) — OCR/vision tooling
(`pytesseract`, `Pillow`) is materially stronger in the Python ecosystem than the
Node/TypeScript equivalents, and `new-backend` has no image-processing dependencies to
build on today. Deployed as its own repository/submodule (`mamo-preprocessor`),
structurally separate from `new-backend` — not a module inside it — so "no MaMo
coupling" (§3.2) is enforced by the deployment boundary, not just convention.

## 9. Verification / Benchmark Protocol

Benchmarked against `testing/fixtures/live-capture/2026-09-26-session/recap-inputs/
note_sheet_scan.pdf`, which already has a careful hand-derived ground-truth
transcription to check recall against (`recap-inputs/notes.md`,
`turn_by_turn_audit.md`). Metric shape matches `live-game-capture-spec.md` §8's own
"Card Entity Recognition Precision & Recall": case-insensitive substring recall against
a fixed list of already-confirmed facts (Sol Ring/T1, BOOM/T6, the Niv-Mizzet
commander cast/T9, World Tree, Scion of Draca, Command Tower/Bloom Tender, Esper
Sentinel), not exact-transcription matching — the ground truth itself is partial. See
`mamo-preprocessor/tests/verify_against_note_sheet_fixture.py` for the runnable
version of this protocol.

## 10. Open Questions / Future Work

- **Audio engine shape.** Sketch, not built: an `external_http` engine that forwards
  audio bytes to a configured `AUDIO_ASR_ENDPOINT_URL` and normalizes the response —
  cheap to add once a concrete ASR backend (a documented Emmah endpoint, or a real
  Whisper deployment) exists to point it at.
- **Alternate vision providers.** The reference `vision_llm` engine is written against
  one provider; the contract doesn't assume which. A deployment could swap providers
  without changing this spec.
- **Barcode/Data-Matrix identification.** `mamo-companion` already has a working
  optical card-identification path (`components/DataMatrixScanner.tsx` +
  `proxy-printing/barcode_stamper.py`) for proxy cards with a printed badge — a
  separate, already-solved problem from free-hand note-sheet OCR, not folded into this
  contract.
