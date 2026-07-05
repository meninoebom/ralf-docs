# Song Space Learnings: What the Lab Proved

Song Space (`~/dev/song-space`, [repo](https://github.com/meninoebom/song-space)) is a browser app where movement drives how a blended song remixes in real time. It is two things at once: a standalone product with its own commercial track, and Ralf's fastest iteration lab. Because it compresses the whole pipeline into one page of vanilla JS (MediaPipe → qualities → readings → mappings → Tone.js), interaction ideas can be tested there in minutes that would take a five-process session to test in Ralf proper.

This note captures what the lab established, so the concepts live here and not only in that repo's code. The validated implementations are in `song-space/frontend/js/` (`score.js`, `readings.js`, `movement.js`, `trigger-engine.js`, `arc.js`).

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

## Arc: what Ralf still lacks

Ralf's scene system is reactive: the same movement produces the same response all night. Song Space's arc adds temporal composition on top: phases that unlock categories over time, durations modulated by engagement (movement extends a phase, stillness can compress it), and an opening AWAIT phase so the piece begins when the dancer begins. The concept is specified in [score-concept.md](../score-concept.md); Song Space has the only working implementation (`arc.js`, plus arc-aware triggers in `trigger-engine.js`). Porting arcs into the runtime's scene system is the single highest-value transfer remaining.

## AdaptiveRange pinning

Directly applicable to `runtime/src/primitives/adaptive-range.ts`. Qualities with absolute bounds (velocity min 0, coherence min 0) need their AdaptiveRange pinned before AND after normalize, or decay collapses the range and the gate chatter begins. The constraint that must hold: `noise_floor / max_pin < gate_threshold − HYSTERESIS_BAND`. Song Space's velocity numbers: `0.002 / 0.05 = 0.04 < 0.07`.

## Deliberate forks (documented, not debt)

- Song Space's backend is a cloud port of the [Blender](https://github.com/meninoebom/ralf-blender) pipeline (Demucs + allin1 via Replicate instead of local processing). The standalone Blender is the canonical algorithm reference.
- `song-space/frontend/js/movement.js` is a JS fork of `adapters/shared/quality-math.ts` (8 qualities + relational metrics). Improvements found in one should be evaluated for the other; the adapters version is canonical for Ralf.
