# Song Space Learnings: What the Lab Proved

Song Space (`~/dev/song-space`, [repo](https://github.com/meninoebom/song-space)) is a browser app where movement drives how a blended song remixes in real time. It is two things at once: a standalone product with its own commercial track, and Ralf's fastest iteration lab. Because it compresses the whole pipeline into one page of vanilla JS (MediaPipe → qualities → readings → mappings → Tone.js), interaction ideas can be tested there in minutes that would take a five-process session to test in Ralf proper.

This note captures what the lab established, so the concepts live here and not only in that repo's code. The validated implementations are in `song-space/frontend/js/` (`score.js`, `readings.js`, `movement.js`, `runtime.js`, `arc.js`). The score is now authored as JSON (`frontend/schema/scores/*.json`) and loaded through a validating loader (`score-loader.js`); `runtime.js` (the `RalfRuntime` class) replaced the old `mapping.js` / `trigger-engine.js` / `trigger-actions.js` trio.

## The three interaction modes

Song Space's working claim: every way a body can shape music is one of three modes, or a combination of them.

| Mode | Shape | Feel | Example |
|------|-------|------|---------|
| **Impulse** | A single punctuating moment, fire-and-forget | Hitting a drum | Sudden arm shoot triggers a vocal accent hit |
| **Gate** | A state you inhabit; entering changes the music, leaving changes it back | Stepping into a room | Arms up = reverb washes in; arms down = dry |
| **Continuous** | Proportional tracking; more of the quality = more of the effect | A fader | More movement opens the master filter |

Treat "three and only three" as a strong hypothesis, not settled fact. But as a design vocabulary it immediately clarifies scene configs: for each reading, ask which mode it is. Temporal modifiers compose with any mode: accumulating (`rampSeconds: N`, value grows over N seconds) and delayed edge (`after: N`, fires only after N seconds sustained).

## Volume is the composer's domain

The strongest revision to come out of the dance tests: **no reading should control volume.** Tying volume to a noisy movement signal made the mix pulse randomly, and multiple readings fighting over faders felt chaotic rather than intentional. The fix that worked:

1. The composer fixes the mix levels. Readings never touch them.
2. The dancer shapes the sound through *effects and events only* (filter, reverb, one-shots, mute/unmute of whole categories at bar boundaries).
3. **Each mode owns its own effect domain** so modes cannot fight each other: continuous → filter, gate → reverb, impulse → one-shots.

This supersedes the mapping examples in [score-concept.md](../score-concept.md) that route readings to loudness ("harmonic_bed: loud"). Route them to timbre, space, and events instead.

## The proof score method

Before layering complexity, validate with a minimal score containing exactly one interaction per mode (`PROOF_SCORE` in `score.js`): one continuous (energy → filter), one gate (arms up → reverb), one impulse (arm raise → accent hit). Each must be detectable, audible, and musical on its own before anything is added. This is the same instinct as the gesture recognizer's F1 rehearsal gate: prove the primitive in the body before trusting compositions built on it.

## Recorded sessions: making the tuning loop reproducible

The lab's tuning loop was live-webcam-only: every quality formula and gate threshold was validated by a person dancing in front of a camera, and the evidence vanished when the tab closed. Unrepeatable, unshareable, absent from CI. The fix (song-space `frontend/js/session-recorder.js`, `session-replay.js`, `replay-cli.js`): capture raw landmarks at the input-adapter boundary with segment labels, then replay them deterministically through the real pipeline (MovementDetector → ReadingsEngine → RalfRuntime) in node or in the lab. One recorded dance becomes a permanent fixture that validates any later change offline, and it gave `movement.js` its first headless test coverage.

Why this matters for Ralf: the runtime is *harder* to tune live than the lab (a multi-process session with a somi-1 body suit you physically wear, not a webcam), so the same record-at-the-boundary / replay-deterministically pattern is higher leverage there. The code does not port (TypeScript, OSC, `Skeleton[]` buffers), but the architecture does, and the runtime is already shaped for it: `SceneConfig` takes typed `ReadingValue[]` and is server-authoritative with a WS broadcast. Capture at the OSC/skeleton boundary. Because the replay also logs fired intents and actions with timestamps, it can regression-test runtime *decisions*, not just perception — which is exactly what porting the arc into the scene system (the highest-value transfer below) needs to be validated against.

The session format follows gesture-studio's conventions (`tracking_system: mediapipe-pose-33-xy`, `coordinate_system: normalized-0-1-xy`), so a MediaPipe-recorded session is a shared corpus both repos' MediaPipe adapters can replay. What transfers is the method, the separation metric, and the pin-math for deriving constants. What does not transfer is the constants themselves, which are calibrated to each adapter's noise floor and framerate.

## Arc: what Ralf still lacks

Ralf's scene system is reactive: the same movement produces the same response all night. Song Space's arc adds temporal composition on top: phases that unlock categories over time, durations modulated by engagement (movement extends a phase, stillness can compress it), and an opening AWAIT phase so the piece begins when the dancer begins. The concept is specified in [score-concept.md](../score-concept.md); Song Space has the only working implementation (`arc.js`, plus arc-aware intents in `runtime.js`). Porting arcs into the runtime's scene system is the single highest-value transfer remaining.

## AdaptiveRange pinning

Directly applicable to `runtime/src/primitives/adaptive-range.ts`. Qualities with absolute bounds (velocity min 0, coherence min 0) need their AdaptiveRange pinned before AND after normalize, or decay collapses the range and the gate chatter begins. The constraint that must hold: `noise_floor / max_pin < gate_threshold − HYSTERESIS_BAND`. Song Space's velocity numbers: `0.002 / 0.05 = 0.04 < 0.07`.

**Second failure mode (found 2026-07-23 by the replay harness, before any real data):** `normalize()` also short-circuits to `0.5` whenever its live range falls below a degenerate-range epsilon. If a max pin is set near that epsilon, internal decay keeps the pinned range perpetually "degenerate" and the quality reads a constant mid-scale — even at rest. In song-space this bit `jerkiness` (max pin `1e-4`, equal to the old `1e-4` epsilon), so a motionless dancer would have shown a half-full jerkiness bar; the fix made the epsilon a pure divide-by-zero guard (`1e-9`). **Ralf's `runtime/src/primitives/adaptive-range.ts` is the same primitive and likely carries the same latent bug — worth an explicit check before it corrupts a live tuning session.**

## Deliberate forks (documented, not debt)

- Song Space's backend is a cloud port of the [Blender](https://github.com/meninoebom/ralf-blender) pipeline (Demucs + allin1 via Replicate instead of local processing). The standalone Blender is the canonical algorithm reference.
- `song-space/frontend/js/movement.js` is a JS fork of `adapters/shared/quality-math.ts` (11 qualities in `QUALITY_KEYS` plus lab-only `jerkiness`, plus relational metrics). Improvements found in one should be evaluated for the other; the adapters version is canonical for Ralf. **The fork has drifted on `jerkiness`:** it shares Ralf's shape (windowed variance of acceleration, 10-frame window) but not its acceleration estimator — Ralf uses a Savitzky-Golay smoothed derivative, song-space uses raw frame-differencing of torso-normalized velocity. A "jerkiness separates staccato from flowing" verdict from the lab supports the design choice but must be re-validated against Ralf's own formula and input before its numbers are trusted.
