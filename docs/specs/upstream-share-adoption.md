# Selective adoption from upstream share

This specification supports the [commit-by-commit review](../research/upstream-share-adoption-review-2026-09-26.md) and the SHARE tickets in [tickets.md](../../tickets.md). It is a proposal for implementation, not a claim that the features below have shipped.

## Baseline and architecture

Compare fork c7a28c4facf50f67942221d8185ca2ae4dfb4401 with upstream 95e82ada5e2513484dccf68295a5ddde6d223730. Keep RecordingSession, RecognitionModelLifecycle, and the SpeechBackend/StreamingSpeechBackend interfaces as the starting points. Preserve Orukeet as the new-install default and all saved selections.

Do not import the upstream shared Android module wholesale. A reusable library becomes justified when a second maintained application actually consumes our implementation. FUTO Keyboard can select the standalone app using its activity protocol without that extraction.

Preserve audio history and retranscription, cleanup ordering, personal vocabulary, diagnostics content exclusions, permission recovery, and truthful recognition-activity results. Shared module adoption or replacement of Sherpa requires a separate decision that supersedes ADR 0001.

## Implementation order

| Plan ID | Work | Dependency |
| --- | --- | --- |
| SHARE-01 | FUTO Keyboard external-provider setup | None |
| SHARE-02 | Session-safe IME insertion | None |
| SHARE-03 | Versioned manual model updates; reopen existing ticket | Existing model catalog |
| SHARE-04 | Reactive readiness/upgrade notices | SHARE-03 |
| SHARE-05 | Model-specific language controls | None; coordinate with existing Model Options ticket |
| SHARE-06 | Explicit Bluetooth microphone selection | None |
| SHARE-07 | Coherent waveform, partial-text, and status presentation | None |
| SHARE-08 | Bounded recovery for recoverable streaming failures | None |
| SHARE-09 | Reproducible transcribe.cpp adapter experiment | Source availability gate before integration |
| SHARE-10 | ASR4ALL Small evaluation and conditional opt-in integration | Successful SHARE-09 feasibility |
| SHARE-11 | Parakeet 110M evaluation and conditional opt-in integration | Successful SHARE-09 feasibility |
| SHARE-12 | Help and branch-specific comparison documentation | None for factual corrections; shipped-feature text waits for its feature |

Prioritize SHARE-01, SHARE-02, and SHARE-05. Engine/model experiments must not block those improvements.

## Behavior constraints

### Keyboard and IME

Use the actual application ID as targetPackage and explicitly address the detected supported keyboard package. An explicit user choice initiates switching. Checking status must not silently change the keyboard's preference. Missing activity, cancellation, stale results, and older keyboards have clear fallback behavior.

Insertion owns per-input-session state, surrounding-text decisions, successful composing updates, and final delivery together. Preserve the editor's normal selected-text replacement behavior. If composition ownership is lost after a cursor move or edit, prefer preserving user text over forcibly moving the cursor to a guessed position. Test empty partials, repeated partials, final-only results, punctuation, both sides of the cursor, new sessions with identical text, editor changes, cancellation, and unavailable connections. Surrounding text stays out of diagnostics.

### Model updates

Work only with immutable versions explicitly known to the app. Do not add a remote model discovery service, background polling, or automatic transfers. Readiness must distinguish usable installed data from a required repair and an optional known upgrade.

Stage and validate a candidate without destroying the working installation. Coordinate activation with runtime/session ownership, then publish state changes. A canceled download or failed activation retains the working model. Startup and composition use cheap validated markers and metadata, not full-file hashing.

The former implementation in 683be27 was intentionally removed by 47566e2. Use it as evidence and a test reference, not as a patch to restore wholesale. Test with two pinned fixtures; show a real update only when the production catalog contains a genuine supported successor.

### Capabilities and presentation

Distinguish inference-time vocabulary hints from our app's post-recognition vocabulary corrections. A model that lacks hinting still supports our correction stage. Do not import upstream's blanket personal-dictionary warning.

Language controls must follow the active model. Keep Nemotron's existing Auto-detect option where supported; upstream's manual-language restriction applies to its implementation. Whisper training-hour copy and model download effects must remain within Whisper settings.

Retain the waveform and accessible text components. Upstream's Canvas-based partial-text rendering is a visual reference, not required implementation. Do not claim this work resolves the separate intermittent-waveform report without reproducing that report.

### Recovery and microphone routing

Distinguish a recoverable stream failure from cancellation, exhausted memory, bad model files, and ordinary backlog. At most one same-model replay is allowed after a classified recoverable failure, using the complete retained recording and a clean runtime state. Do not silently switch models, retry cancellation, or deliver partial text as a final result.

Route choice must be explicit and handle supported Android versions, connection failure, disconnects, and cleanup. Show the actual route, not merely the requested route. Preserve saved audio/transcript continuity when changing route; if a route change requires a new utterance, say so and do not silently discard the current one.

## Engine and model evaluation gate

FUTO's transcribe.cpp pin b9e8a8e348db9a34052b72f8643bbe4597522181 was not anonymously retrievable during review. The public handy-computer repository does not contain that commit. Before writing the adapter, obtain a reproducible source revision and document its relationship to FUTO's integration. An explicitly evaluated public revision is acceptable; silently substituting one is not.

The experiment must:

- Start in an isolated experimental build and leave production defaults and existing adapters intact.
- Build on ARM64 with a pinned toolchain and complete license notices. Test coexistence with legacy GGML and S1's llama.cpp, including duplicate targets, native symbols, and library packaging.
- Implement load, transcribe, streaming where supported, cancel/close, errors, and language behavior through the current interfaces. Preserve full audio and exactly one final result.
- Treat CPU as the baseline because upstream explicitly selects it. Any GPU experiment is separately identified and needs fallback and device evidence.
- Compare at least one model family available in both implementations before proposing replacement. Record different quantization/model versions as confounders rather than calling them pure engine comparisons.

Measure on the user's phone with fixed audio and reference transcripts: first usable partial, partial revisions, Stop-to-final median and p95, total processing time, peak memory, APK/install/model size, and thermal behavior during sustained dictation. Include short phrases, numbers/names, silence, approximately 30-second samples, and at least two-minute recordings with pauses. Report raw recognition separately from cleanup. Use repeated warm and cold runs with the settings and build revision recorded.

ASR4ALL Small and Parakeet 110M must demonstrate a useful measured speed/resource benefit or a clearly documented accuracy tradeoff against the existing small-model choices. English-only candidates cannot replace multilingual Orukeet by default. A non-shipping decision with recorded evidence is a valid evaluation result. Source or hardware unavailability is an unresolved validation gap, not a successful result.

## Verification

For production code, add focused checks at the affected interface and run the relevant existing unit, instrumentation, assembly, and lint checks. Keyboard routing and microphone behavior require device testing; fake-result tests alone do not prove interoperability. UI changes check TalkBack, font scaling, orientation, and system-bar padding without duplicating the theme's existing safe-area handling.

This planning change itself requires ticket dependency/link consistency checks and a documentation diff check. It does not require an APK build.
