# Ralf Architecture

> Mutual transformative engagement through programmable audio responsiveness

## The One-Sentence Pitch

Ralf is a visual mapper between body and sound where the technologist configures, the dancer can read, and the musician plugs in their own sonic world.

## Core Concepts

### The Listening Loop

The system listens to the body through qualities and gestures. The dancer listens to the system through sound. Neither is in control. Both are responding. Every turn through the loop changes the next turn.

### Two Communication Surfaces

| Surface | What it reads | Signal type | Example |
|---------|--------------|-------------|---------|
| **Gesture** | Discrete movement patterns | Events (fire / don't fire) | A jack, a wave, a spin |
| **Quality** | Continuous movement character | Streams (0.0–1.0, always flowing) | Jerkiness, velocity, stillness |

These are complementary, not competing. Most interactive dance systems have one or the other. Ralf has both.

### Two Dimensions of Composition

A composition has two dimensions:

- **The Sonic World** — melodies, rhythms, harmonics, sound design, loops, scenes. This is the composer's musical intent. What they make in their DAW.
- **The Tendencies** — weights that shape how the sonic world responds to movement. Which scenes get triggered, which effects engage, what's probable vs rare. This is the interaction design.

The sonic world without tendencies is a static piece. Tendencies without a sonic world have nothing to act on. A composer authors both, but they're different creative acts — one is musicianship, the other is interaction design.

---

## The Seven Primitives

Everything in the system is built from seven node types. Each maps to something already proven in the existing codebase.

| # | Node | What it does | Already built in |
|---|------|-------------|-----------------|
| 1 | **Sense** | Extract a continuous 0-1 value from body data | States of Being (velocity, jerkiness, contraction, verticality, symmetry, coherence) |
| 2 | **Recognize** | Pattern-match frames against trained examples → fire event | Gesture Studio (DTW + VAD state machine) |
| 3 | **Accumulate** | Track a value over time — windowed rate or running count | Max4Live (streams + stacks) |
| 4 | **Combine** | Weighted mix of inputs + gate → derived meaning (a **Reading**) | States of Being (emotion formulas with smoothstep gates) |
| 5 | **Roll** | Weighted random selection from a pool of actions | Max4Live (`weightedRandom()` in intent resolution) |
| 6 | **Gate** | Pass or block based on condition (threshold, state, time) | Max4Live (signals), States of Being (smoothstep gates) |
| 7 | **Act** | Do something to the sonic world | Max4Live (action library), States of Being (Tone.js parameter changes) |

### How Data Flows Through Them

```
BODY (skeleton frames or sensor data)
  │
  ├──→ [Sense] ──→ continuous qualities (0-1 floats, every frame)
  │       │
  │       ├──→ [Combine] ──→ named readings (e.g. "flow", "hitting the beat")
  │       │       │
  │       │       └──→ [Intent] ──→ [Roll] ──→ [Act]
  │       │
  │       └──→ [Accumulate] ──→ [Gate] ──→ [Intent] ──→ [Roll] ──→ [Act]
  │
  └──→ [Recognize] ──→ discrete gesture events
          │
          ├──→ [Accumulate] ──→ rate/count ──→ [Gate] ──→ [Intent]
          │
          ├──→ [Roll] ──→ weighted action selection ──→ [Act]
          │
          └──→ [Gate] ──→ conditional direct commands ──→ [Act]
```

Two paths: **continuous** (Sense → Combine → Act, every frame, smooth) and **discrete** (Recognize → Roll → Act, event-driven, sparse). They can cross-pollinate — a quality can gate a gesture, a gesture hit can feed into an accumulator that a reading uses.

---

## Glossary

**Quality** — A continuous measurement of how a body moves. Velocity, jerkiness, contraction, verticality, symmetry, coherence. Always flowing, always 0-1. The letters of the movement alphabet.

**Gesture** — A recognized discrete movement pattern. A jack, a wave, a pull-back. Fires once, like pressing a key.

**Reading** — The system's interpretation of what movement means. Created by combining qualities through weighted formulas with gate conditions. "High jerkiness + rhythmic periodicity = hitting the beat" is a reading. Different readings can be applied to the same body — a house reading, a contemporary reading, a duo reading.

**Intent** — What the system wants to do in response, expressed symbolically. "Strip energy," "build tension," "add texture." The intent names the desire, not the specific action.

**Roll** — The weighted random selection from an intent's pool of possible actions. Same intent, different outcome each time. The weights are the composer's Tendencies — they shape probability without dictating outcome. This is what makes the system feel alive.

**Scene** — The full configuration of a performance. Who is here, how many, what inputs, which readings, which tendencies, what sonic world. A scene is a complete setup that can be saved, loaded, and shared.

**Listening Loop** — The continuous cycle: body moves → surfaces read → readings interpret → rolls choose → audio responds → body hears and adapts. Both sides listen. Neither controls.

**Sonic World** — The musical material: melodies, rhythms, harmonics, sound design, samples, loops, scenes. What the composer creates.

**Tendencies** — The weights that shape how the sonic world responds to movement. The probability space. A different dimension of composition from the sonic world itself.

---

## System Architecture

### Browser Console + Runtime Server

```
┌──────────────────────────────────────────────────┐
│           PERFORMANCE CONSOLE (Browser)            │
│                                                    │
│  ┌──────────┐ ┌──────────┐ ┌─────────┐ ┌───────┐ │
│  │  Node    │ │  Dancer  │ │  Scene  │ │Monitor│ │
│  │  Patcher │ │  View    │ │  Mgmt   │ │/ Debug│ │
│  └────┬─────┘ └──────────┘ └─────────┘ └───────┘ │
│       └──── all connected via WebSocket ──────────│
└─────────────────────┬────────────────────────────-┘
                      │
         ┌────────────┴────────────┐
         │     RUNTIME SERVER       │
         │     (Node.js / TS)       │
         │                          │
         │  Executes the 7 nodes    │
         │  Routes OSC in/out       │
         │  Manages scenes + state  │
         │  Headless — won't crash  │
         │  if you close a tab      │
         └────┬──────────┬─────────┘
              │          │
       OSC ↕  │          │  OSC / MIDI
   ┌──────────┴┐   ┌─────┴──────────┐
   │  Inputs   │   │  Translators   │
   │  camera   │   │                │
   │  phone    │   │  ┌───────────┐ │
   │  watch    │   │  │ Tone.js   │ │
   │  lidar    │   │  │ (browser) │ │
   └───────────┘   │  ├───────────┤ │
                   │  │ Max4Live  │ │
                   │  │ (Ableton) │ │
                   │  ├───────────┤ │
                   │  │ SuperCol. │ │
                   │  │ Pd, etc.  │ │
                   │  └───────────┘ │
                   └────────────────┘
```

### Why This Architecture

- **The browser is the UI, not the runtime.** Refresh it anytime; the server keeps running. Three people can open three different views of the same system.
- **The server is the brain.** ~2000 lines of TypeScript. Your 7 primitives executing at 30fps on arrays of floats. JavaScript handles this trivially.
- **Sound stays in whatever the musician already uses.** The runtime server sends generic messages (`/ralf/act/filter_cutoff 0.7`). A translator converts them to the target's language.
- **Translators are swappable.** Monday: prototype with Tone.js in browser. Tuesday: plug into a producer's Ableton set. Same primitives, same scene, different last mile.

### The Translator Pattern

The runtime server outputs generic semantic messages:

```
/ralf/act/filter_cutoff  0.7
/ralf/act/scene           3
/ralf/act/mute_track      "perc"
/ralf/intent/fired        "strip_energy"
```

Each translator converts these to its target's language:

| Translator | Receives | Sends |
|------------|----------|-------|
| **Tone.js** (browser) | WebSocket from server | `synth.filter.frequency.rampTo(10000, 0.1)` |
| **Max4Live** (Ableton) | OSC from server | Live API calls: `live_set.tracks[3].mixer_device.volume.value = 0.7` |
| **SuperCollider** | OSC from server | `s.sendMsg('/n_set', nodeID, 'freq', 700)` |
| **Any OSC target** | OSC from server | Pass-through — the target handles it |

### Three Sonic World Scenarios

**A. Ableton as the Sonic World** — A producer builds Session View with clips, scenes, instruments, effects. The M4L translator receives messages from the runtime server and manipulates Live. This is the most powerful scenario. The M4L device is lightweight (~200 lines) — all the intelligence is in the runtime server.

**B. The Blender** — Drop a track in, Demucs splits stems, the blender slices and categorizes (foundation, groove, bass, hook, texture, accent). Out comes samples + config that loads into either Ableton or the Tone.js engine.

**C. Standalone Browser** — For workshops, installations, Burning Man, quick jams. Tone.js plays samples and synths directly. Less powerful than Ableton, but zero setup.

All three use the same runtime server and the same primitives. Only the translator changes.

---

## Qualities (Sense Nodes)

### Individual Qualities (per dancer)

Proven in States of Being. Each is a continuous 0-1 value, self-calibrating via Adaptive Range (no calibration step needed).

| Quality | What it measures | Physical meaning |
|---------|-----------------|-----------------|
| Velocity | Average joint speed | How fast you're moving |
| Jerkiness | 3rd derivative of position | Abruptness — shaking vs smooth |
| Contraction | Limb distance from center of mass | Arms pulled in vs reaching out |
| Verticality | Head height relative to hips | Upright vs slumped |
| Symmetry | Left vs right positional balance | Even-sided vs one-sided |
| Coherence | Left vs right timing variance | Coordinated vs chaotic |

### Relational Qualities (between dancers — not yet built)

| Quality | What it measures | Physical meaning |
|---------|-----------------|-----------------|
| Proximity | Distance between centers of mass | How close |
| Mirroring | Correlation of joint positions | How alike |
| Synchrony | Correlation of velocity profiles | How in time |
| Contrast | Divergence of quality profiles | How different |
| Convergence | Rate of change of proximity | Approaching or departing |

### From Any Input

Qualities are computed from whatever data is available:

| Input | What you get |
|-------|-------------|
| Camera (MediaPipe) | Full skeleton → all individual qualities |
| Phone accelerometer | Velocity, jerkiness (no contraction/verticality without skeleton) |
| Wristband | Velocity, jerkiness of that limb |
| LiDAR | Full 3D skeleton → all qualities with depth |
| Multiple phones in a crowd | Aggregate velocity, aggregate synchrony, spatial patterns |

---

## Scene Configuration Format

A Scene is a JSON file that declares the full configuration of a performance:

```json
{
  "name": "House Session with Maya",
  "version": "0.1",

  "inputs": [
    { "type": "mediapipe", "source": "camera", "port": 6448 }
  ],

  "dancers": [
    { "id": "maya", "input": 0 }
  ],

  "senses": [
    { "id": "velocity", "type": "velocity", "dancer": "maya" },
    { "id": "jerkiness", "type": "jerkiness", "dancer": "maya" },
    { "id": "contraction", "type": "contraction", "dancer": "maya" }
  ],

  "recognizers": [
    { "id": "house-vocab", "vocabulary": "house-foundations.ralf", "dancer": "maya" }
  ],

  "readings": [
    {
      "id": "hitting-the-beat",
      "type": "combine",
      "inputs": { "jerkiness": 0.6, "velocity": 0.3 },
      "gate": { "jerkiness": { "above": 0.2 } }
    },
    {
      "id": "flow",
      "type": "combine",
      "inputs": { "velocity": 0.3, "contraction": -0.4, "jerkiness": -0.3 },
      "gate": { "velocity": { "above": 0.1 }, "jerkiness": { "below": 0.5 } }
    }
  ],

  "intents": {
    "add_energy": {
      "pool": [
        { "action": "fire_next_scene", "weight": 2 },
        { "action": "unmute_track", "target": "perc", "weight": 3 },
        { "action": "emphasis_track", "target": "bass", "weight": 1 }
      ]
    },
    "strip_energy": {
      "pool": [
        { "action": "filter_sweep_down", "weight": 3 },
        { "action": "hush_master", "weight": 2 },
        { "action": "mute_track", "target": "perc", "weight": 2 }
      ]
    }
  },

  "sonic_world": {
    "type": "ableton",
    "translator": "max4live",
    "port": 12000
  }
}
```

Swap `"sonic_world"` to `{ "type": "browser", "engine": "tonejs", "samples": "./blended/" }` and the same scene runs standalone.

---

## What's Not Yet Built (Identified Gaps)

### Transitions
No crossfade between states. Temporal actions in Max4Live are a workaround. Need a first-class concept for "how do we get there" — easing, blend time, transition curves.

### History / Trajectory
The system knows where you *are* but not where you *came from*. Pulling back from peak energy should feel different than pulling back from silence. Need a node that exposes the derivative of a reading — not just its value, but its direction.

### Cross-Pollination Between Surfaces
Currently the gesture path and quality path are parallel. They should be able to feed each other: a quality reading could gate when gestures are recognized, and gesture hits could feed into accumulators that quality readings use.

### Relational Qualities
The between-dancer qualities (proximity, mirroring, synchrony, contrast, convergence) are conceptually defined but not yet implemented. These are computed from pairs of individual quality streams.

---

## Build Roadmap

Target: Burning Man (summer 2026, ~6 months)

| Month | What | Deliverable |
|-------|------|-------------|
| 1-2 | Runtime server + scene config | TypeScript server executing 7 primitives, OSC in/out, WebSocket API, scene load/save |
| 3 | Browser Performance Console | Node patcher (reactflow), live data visualization, dancer view, scene management |
| 4 | Integration + first piece | Work with a real dancer. This phase rewrites half of month 3. Expected. |
| 5 | Crowd mode | Lightweight phone web app sending IMU data to server. Aggregate qualities. The Burning Man scenario. |
| 6 | Hardening | Launcher script, stability, fallback modes, the actual event |

### What Already Exists (reuse, don't rebuild)

- **Gesture recognition** → Gesture Studio (Rust/Tauri). Keep it for training. Export .ralf files that the runtime server loads.
- **Quality analysis** → States of Being (browser). Port the 8 primitives + AdaptiveRange into the runtime server.
- **Mapping logic** → Max4Live gesture-controller.js. Port streams/stacks/intents/signals into the runtime server as Accumulate/Roll/Gate/Act nodes.
- **Sound engine** → sound-engine/ (Tone.js). Becomes the browser translator.
- **Blender** → sound-engine/ (Python stem separation). Keep as-is.
- **Pose tracking** → movenet/ and mediapipe/. Keep as-is, they already output OSC.
- **iOS input** → gesture-input-ios/. Keep as-is, already outputs WebSocket/OSC.
- **M4L device** → max4live/. Simplify to a thin translator that receives /ralf/act/* messages.

---

## Design Principles

1. **The "hello world" must work in under 2 minutes.** Launch one command, browser opens, camera activates, default sounds respond to movement. Configure later.

2. **The primitives are the product.** Everything else is plumbing. Build plumbing in the fastest, most boring technology available (TypeScript, React) so you can spend time on what matters.

3. **The naming of things is half the design.** Quality, Gesture, Reading, Intent, Roll, Scene, Listening Loop, Sonic World, Tendencies. These words are the framework. Protect them.

4. **Separate the brain from the sound.** The runtime server owns intelligence. The translator owns audio. They connect via OSC. Swap the translator, swap the sonic world.

5. **Three views, one system.** The technologist sees the patcher. The dancer sees quality feedback. The musician sees what's being triggered. All views into the same state.

6. **Influence, not control.** The dancer's movement influences the sonic world through weighted probability. Same input, different outcome. The tendencies shape the space; they don't dictate the path.

7. **Start by making a specific piece.** The framework emerges from practice. Build one piece with one dancer, then extract. Every successful creative framework (openFrameworks, Pure Data, Sonic Pi, VCV Rack) followed this path.
