# RALF — One-Claim Spec (Version 1)

*The smallest buildable version that demonstrates one claim. Turns RALF from "art project with a manifesto" into "an instrument that produces measurements about a stated hypothesis." Everything grander is version N.*

---

## The one claim this version demonstrates

From the Claims Ledger, **Claim 2** (load-bearing):

> Two people can be brought into stable phase-locking through a shared rhythmic medium, and the system can detect when locking occurs.

That is it. Not generative choreography. Not call-and-response synthesis. Not bonding measurement. Just: **can I measure two people falling into rhythm, and detect the moment they lock.**

---

## What v1 is a slice *of* (so it doesn't become the whole identity)

This version measures phase-locking between two bodies. That is the narrowest, most measurable slice of the project — and it is the slice most easily mistaken for the entire project. State plainly what it is *not*:

- **v1 is not a downscaled statement of the thesis.** The thesis is wide: musicking as an antidote to the fragmenting modes of cognition (abstraction, language, analysis, goal-directedness). v1 instruments the one corner of that with a clean signal — two rhythms locking. Building the slice is not narrowing the argument; it is finding the argument's testable handle.
- **Detecting locked limbs is the *measurement*, not the *point*.** The point is re-grounding; phase-locking is just what re-grounding looks like when you reduce it to something a camera and a Hilbert transform can see. Keep the wide argument in the writing even while the build is this narrow, or RALF really does collapse into "make two people move in time."
- **The activity v1 instruments is itself a non-goal-directed, infinite game.** Even as the metric (PLV, lock/no-lock) is binary and goal-shaped, what the two people are *doing* is open-ended musicking — play for the sake of continuing play. v1 measures the goal-directedness handle while staging the non-goal-directed antidote. Don't let the metric's shape leak back into how the activity is designed or described.

---

## Why this version, politically

A tool that generates data about embodied social synchrony is a research apparatus, and research apparatuses get PhDs and positions built around them. Version 1 produces a *measurement* about a *stated hypothesis* — which is exactly the thing that lets a design-research program say "this fits here." Resist the temptation to make version 1 grander; grandeur is what makes it read as art-with-manifesto instead of instrument.

---

## Minimum system (the pipeline)

1. **Capture** two people's motion.
   - MediaPipe pose estimation, two bodies (two cameras, or two regions of one frame to start).
   - Output: time series of landmark positions per person.

2. **Extract a rhythmic signal** from each person.
   - This is where the *frequency-of-a-real-quantity* discipline kicks in. Pick one clean periodic quantity per body — e.g. vertical displacement of the torso/center-of-mass, or a single joint angle that swings periodically in the chosen movement.
   - Output: one (or a few) clean oscillating signal(s) per person.

3. **Compute the phase relationship** between the two signals over time.
   - Extract instantaneous phase of each signal (Hilbert transform is the standard tool).
   - Compute phase difference over time.
   - Phase-locking value (PLV) over a sliding window is the standard metric for "how locked are they right now."

4. **Detect and display** when locking occurs.
   - Threshold/visualize PLV: show the moment two independent rhythms become a stable phase relationship.
   - Make it *felt* — the demo's power is letting people watch (and ideally hear/see in real time) the lock happen.

---

## Definition of done (Version 1)

- Two people move freely; system extracts a clean rhythmic signal from each.
- System computes phase difference and a locking metric in (near) real time.
- System reliably detects and displays the transition from unlocked to locked.
- Latency low enough to feel live (target the sub-300ms range that makes interaction feel responsive).

That is a complete, demonstrable, fundable artifact. Stop there for v1.

---

## Explicitly OUT of scope for Version 1

- Generative choreography / movement synthesis
- Call-and-response *generation* (v1 only *detects* relationship structure)
- Bonding / trust / cooperation measurement (Claim 3)
- Modeling Afrodiaspora forms as coupled-oscillator systems (Claim 5) — *this is the exciting version N, not v1*
- Multi-person (>2) Kuramoto-style collective synchronization
- Any audio synthesis

Each of these is a later version. Writing them here is to *park* them, not to build them.

---

## Technical notes / things to learn as you build

- **Hilbert transform** for instantaneous phase — the key signal-processing step. Learn this; it's the heart of step 3.
- **Phase-locking value (PLV)** — the standard synchrony metric. Borrowed from neuroscience (where it measures EEG channel synchrony); applies directly here.
- **Kuramoto model** — not needed for two oscillators, but read it now so the path to multi-person collective sync (version N) is clear.
- **MediaPipe** pose — you likely know this from prior RALF work; v1 just needs stable landmarks for the chosen quantity.

---

## Reading, timed to the build (not before)

- **Christopher Small, *Musicking*** — music as activity/relationship enacted in real time. Possibly the single most important text; it *is* RALF's thesis statement.
- **Steven Mithen, *The Singing Neanderthals*** — music as cognitively/evolutionarily fundamental.
- These land harder when you are already wrist-deep in the problem. Read as the build proceeds.

---

## The trap to avoid

The temptation will be to keep writing the framework, because writing feels like progress and building is scary. The framework is already articulated well enough to start. The bottleneck now is not clarity of thought — it is a working version-one of RALF. **Build the smallest thing that detects two people locking. Everything else follows from there.**
