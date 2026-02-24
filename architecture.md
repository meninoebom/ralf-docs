# Ralf Architecture

> Mutual transformative engagement through programmable audio responsiveness

## The One-Sentence Pitch

Ralf is a system that lets dancers influence music through movement — where a technologist configures the connection, a dancer can feel it responding, and a musician brings the sound.

---

## How It Works (Plain English)

A dancer moves. The system watches through a camera, phone, or wristband. It reads two things at once:

1. **Qualities** — continuous measurements of *how* they're moving (fast? jerky? still? contracted?)
2. **Gestures** — specific recognized movements (*that was a jack, that was a wave*)

These feed into **Readings** — interpretations of what the movement means ("she's hitting the beat," "he's in flow," "they're mirroring each other").

Readings trigger **Intents** — what the system wants to do ("strip energy," "build tension"). Each intent has a **Roll** — a weighted random pick from a pool of possible actions. Same intent might produce a filter sweep one time, a percussion drop the next. This randomness is what keeps it feeling like a conversation, not a remote control.

The chosen action reaches into the **Sonic World** — the music a composer created — through a **Translator** that speaks whatever language the audio engine uses (Ableton, Tone.js, SuperCollider, anything).

The dancer hears the change. Their next move is shaped by what they heard. This is the **Listening Loop** — both sides listening, neither in full control.

---

## Core Concepts

### The Listening Loop

```
Dancer moves
    → System reads qualities + gestures
        → Readings interpret meaning
            → Intents express what should happen
                → Rolls choose how (weighted random)
                    → Translator speaks to the audio engine
                        → Sound changes
                            → Dancer hears and responds
                                → (loop continues)
```

### Two Surfaces

The system reads movement through two complementary surfaces:

| Surface | Type | Example | Analogy |
|---------|------|---------|---------|
| **Quality** | Continuous stream, 0-1, every frame | Velocity: 0.7, Jerkiness: 0.2 | A fader — always moving |
| **Gesture** | Discrete event, fires once | `/gesture/jack` → fire! | A button press |

Most interactive dance systems have one or the other. Ralf has both, and they can feed each other.

### Two Dimensions of Composition

| Dimension | What it is | Who creates it | Example |
|-----------|-----------|----------------|---------|
| **Sonic World** | The musical material — melodies, rhythms, sounds, loops, scenes | The composer/producer, in their DAW | An Ableton session, a set of samples |
| **Tendencies** | Weights that shape how the sonic world responds to movement | The interaction designer (might be the same person) | "When energy is stripped, filter sweep is 3× more likely than mute" |

The sonic world without tendencies is a static piece. Tendencies without a sonic world have nothing to act on.

### Three Tiers of Portability

| Tier | What | Portable? | Format |
|------|------|-----------|--------|
| **Interaction design** | Readings, intents, rolls, tendencies | Yes — works with any audio engine | Scene config (JSON) |
| **Musical material as samples** | Stems, loops, one-shots (e.g. from the Blender) | Yes — wav files + category metadata | Samples + perf.json |
| **Full production** | A complete Ableton session with synths, effects, routing | No — native to the DAW, and that's fine | Ableton project |

Tiers 1 and 2 together = a portable composition. Tier 3 is where a producer goes deeper in their native tool.

---

## The Seven Primitives

Everything is built from seven node types. Think of them as Lego bricks — simple individually, powerful when combined.

| Node | What it does | One-line example |
|------|-------------|-----------------|
| **Sense** | Extracts a 0-1 value from the body | "How jerky is this dancer right now? → 0.6" |
| **Recognize** | Matches a movement pattern, fires an event | "That was a jack → fire!" |
| **Accumulate** | Tracks something over time | "3 gestures in the last 5 seconds" or "12 total moves" |
| **Combine** | Mixes multiple values with weights to produce a Reading | "jerkiness × 0.6 + velocity × 0.3 = hitting the beat: 0.48" |
| **Roll** | Picks one action from a weighted pool | "strip_energy → [filter sweep ×3, mute perc ×1] → filter sweep wins" |
| **Gate** | Passes or blocks based on a condition | "Only fire if energy > 0.5 AND we haven't acted in 10 seconds" |
| **Act** | Sends a message to the audio engine | "/ralf/act/filter_cutoff 0.7" |

### Where each primitive already exists in the codebase

| Node | Built in | File/module |
|------|----------|-------------|
| Sense | States of Being | `index.html` — computePrimitives(), AdaptiveRange |
| Recognize | Gesture Studio | `engine/recognizer.rs` — DTW + VAD state machine |
| Accumulate | Max4Live | `gesture-controller.js` — streams (windowed) + stacks (counting) |
| Combine | States of Being | `index.html` — computeEmotions() with weights + smoothstep gates |
| Roll | Max4Live | `gesture-controller.js` — weightedRandom() in resolveIntents() |
| Gate | Max4Live + States of Being | `gesture-controller.js` — signals; `index.html` — smoothstep gates |
| Act | Max4Live + States of Being | `gesture-controller.js` — action library; `index.html` — Tone.js params |

---

## System Architecture

```
┌──────────────────────────────────────────┐
│      PERFORMANCE CONSOLE (Browser)        │
│                                           │
│  What you see and interact with:          │
│  • Node patcher (wire primitives)         │
│  • Dancer view (quality feedback)         │
│  • Scene management (save/load)           │
│  • Live monitoring                        │
│                                           │
│           WebSocket ↕                     │
└──────────────┬───────────────────────────-┘
               │
  ┌────────────┴────────────┐
  │     RUNTIME SERVER       │
  │     (Node.js / TS)       │
  │                          │
  │  The brain:              │
  │  • Runs the 7 primitives │
  │  • Manages scenes        │
  │  • Routes messages       │
  │  • Headless process —    │
  │    survives tab refresh  │
  └────┬──────────┬─────────┘
       │          │
OSC in │          │ OSC / MIDI out
       │          │
  ┌────┴───┐  ┌──┴────────────┐
  │ Inputs │  │  Translator   │
  │        │  │  (swappable)  │
  │ camera │  │               │
  │ phone  │  │ • Tone.js     │
  │ watch  │  │ • Max4Live    │
  │ lidar  │  │ • SuperCol.   │
  └────────┘  │ • anything    │
              └───────────────┘
```

### Why this design

**The browser is the UI, not the runtime.** Refresh it anytime; the server keeps running. Three people can open three different views.

**The server is the brain.** ~2000 lines of TypeScript. Runs the primitives at 30fps. JavaScript is more than fast enough for this.

**Sound stays native.** The server sends generic messages like `/ralf/act/filter_cutoff 0.7`. A translator converts them to whatever the audio engine needs. Swap the translator, swap the sonic world.

### The Translator Pattern

The runtime server doesn't know or care what audio engine is at the other end. It outputs generic messages:

```
/ralf/act/filter_cutoff  0.7
/ralf/act/scene           3
/ralf/act/mute_track      "perc"
```

Each translator converts these:

| Translator | When to use it |
|------------|---------------|
| **Tone.js** (browser) | Workshops, quick jams, installations, Burning Man. Zero setup. |
| **Max4Live** (Ableton) | Full production. A producer's complete session with deep sound design. |
| **OSC pass-through** | SuperCollider, Pd, or any OSC-capable software. |

Monday you prototype with Tone.js. Tuesday a producer shows up and you plug into their Ableton set. Same scene config, same primitives, different translator.

### Three Sonic World Scenarios

**A. Ableton** — A producer builds their Session View (clips, scenes, instruments, effects). The M4L translator receives Ralf messages and manipulates Live. Most powerful. The M4L device is thin (~200 lines) — all intelligence is in the runtime server.

**B. The Blender** — Drop a track in, get stems + categorized samples out (foundation, groove, bass, hook, texture, accent). Loads into either Ableton or Tone.js. A portable starting point for any sonic world.

**C. Browser standalone** — Tone.js plays samples and synths directly. Less powerful than Ableton, but zero setup. Good for when you just need sound and a body.

All three use the same runtime server and the same primitives. Only the translator changes.

---

## Qualities

### Individual (per dancer)

Self-calibrating via Adaptive Range — no setup step needed. Proven in States of Being.

| Quality | Measures | Meaning |
|---------|---------|---------|
| Velocity | Average joint speed | How fast |
| Jerkiness | Rate of change of acceleration | How abrupt |
| Contraction | Limb distance from center | How closed |
| Verticality | Head height vs hips | How upright |
| Symmetry | Left-right position balance | How even |
| Coherence | Left-right timing match | How coordinated |

### Relational (between dancers — not yet built)

| Quality | Measures | Meaning |
|---------|---------|---------|
| Proximity | Distance between centers | How close |
| Mirroring | Joint position correlation | How alike |
| Synchrony | Velocity profile correlation | How in time |
| Contrast | Quality profile divergence | How different |
| Convergence | Rate of proximity change | Approaching or departing |

### What each input gives you

| Input | Available qualities |
|-------|-------------------|
| Camera (MediaPipe) | All individual qualities |
| Phone accelerometer | Velocity, jerkiness |
| Wristband / Watch | Velocity, jerkiness of that limb |
| LiDAR | All qualities with depth |
| Crowd of phones | Aggregate velocity, aggregate synchrony |

---

## Scene Config

A Scene is a JSON file that declares everything about a performance setup. Here's a minimal example:

```json
{
  "name": "House Session with Maya",

  "dancers": [
    { "id": "maya", "input": { "type": "mediapipe", "port": 6448 } }
  ],

  "readings": [
    {
      "id": "hitting-the-beat",
      "mix": { "jerkiness": 0.6, "velocity": 0.3 },
      "gate": { "jerkiness": { "above": 0.2 } }
    }
  ],

  "intents": {
    "add_energy": [
      { "action": "fire_next_scene", "weight": 2 },
      { "action": "unmute_track", "target": "perc", "weight": 3 }
    ]
  },

  "sonic_world": { "type": "ableton", "port": 12000 }
}
```

Swap `"sonic_world"` to `{ "type": "tonejs", "samples": "./blended/" }` and the same scene runs in the browser.

---

## What's Not Yet Built

| Gap | What's needed | Why it matters |
|-----|--------------|---------------|
| **Transitions** | Crossfade/easing between states | Currently state changes are instant; need "how do we get there" |
| **Trajectory** | Expose the *direction* of a reading, not just its value | Pulling back from peak should feel different than from silence |
| **Surface cross-talk** | Let qualities gate gestures, let gestures feed accumulators | The two surfaces are parallel; they should cross-pollinate |
| **Relational qualities** | Compute between-dancer qualities from pairs of streams | Needed for duo and crowd scenarios |

---

## Build Roadmap

Target: Burning Man (summer 2026, ~6 months from Feb 2026)

| Phase | What | Deliverable |
|-------|------|-------------|
| **1-2** | Runtime server | TypeScript server executing 7 primitives. OSC in/out. WebSocket API. Scene load/save. Port quality analysis from States of Being, mapping logic from Max4Live. |
| **3** | Performance Console | Browser UI: node patcher (reactflow), live data viz on each node, dancer view, scene management. |
| **4** | First piece with a dancer | Real integration test. Expect to rewrite half of phase 3. That's the point. |
| **5** | Crowd mode | Lightweight phone web app → server. Aggregate qualities. The Burning Man scenario. |
| **6** | Hardening | One-command launcher. Stability. Fallback modes. Ship it. |

### What already exists (reuse, don't rebuild)

| Component | Where | Role in new system |
|-----------|-------|-------------------|
| Gesture recognition | `gesture-studio/` (Rust/Tauri) | Keep for training. Exports .ralf files the server loads. |
| Quality analysis | `states-of-being/` (browser) | Port 8 primitives + AdaptiveRange into server. |
| Mapping logic | `max4live/gesture-controller.js` | Port streams/stacks/intents/signals into server. |
| Tone.js sound engine | `sound-engine/` | Becomes the browser translator. |
| Blender (stem separation) | `sound-engine/` (Python) | Keep as-is. Portable sample pipeline. |
| Pose tracking | `movenet/` and `mediapipe/` | Keep as-is. Already outputs OSC. |
| iOS input | `gesture-input-ios/` | Keep as-is. Already outputs WebSocket. |
| M4L device | `max4live/` | Simplify to thin translator receiving /ralf/act/* messages. |

---

## Glossary

For quick reference. Every term earns its place.

| Term | Definition |
|------|-----------|
| **Quality** | A continuous 0-1 measurement of how a body moves (velocity, jerkiness, etc.) |
| **Gesture** | A recognized discrete movement pattern that fires as an event |
| **Reading** | An interpretation of movement — qualities combined through weights and gates into named meaning |
| **Intent** | What the system wants to do, symbolically ("strip energy," "build tension") |
| **Roll** | Weighted random selection from an intent's pool of possible actions |
| **Gate** | A condition that passes or blocks (threshold, time, state) |
| **Act** | A generic message sent to the audio engine |
| **Translator** | Converts generic Act messages into a specific audio engine's language |
| **Scene** | The full configuration of a performance — who, what inputs, which readings, which tendencies, what sonic world |
| **Sonic World** | The musical material a composer creates (samples, synths, clips, scenes) |
| **Tendencies** | The weights in the Rolls — shaping probability, not dictating outcome |
| **Listening Loop** | The continuous cycle: move → read → interpret → choose → sound → hear → move |
| **Blender** | Pipeline that takes a track, splits stems, outputs categorized samples — a portable sonic world |

---

## Design Principles

1. **"Hello world" in under 2 minutes.** One command, browser opens, camera sees you, sound responds. Configure later.

2. **The primitives are the product.** Everything else is plumbing. Use boring technology for plumbing.

3. **Protect the vocabulary.** These words are the framework. They matter more than the code.

4. **Separate brain from sound.** The server thinks. The translator speaks. Swap the translator, swap the world.

5. **Three views, one system.** Technologist sees the patcher. Dancer sees feedback. Musician sees triggers.

6. **Influence, not control.** Weighted probability, not deterministic mapping. The tendencies shape the space; they don't dictate the path.

7. **Start with one piece.** Every successful creative framework started as one person's tool for one specific project. The platform emerges from practice.
