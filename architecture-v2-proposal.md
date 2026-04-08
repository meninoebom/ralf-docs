# Ralf Architecture v2

> Status: Active. Supersedes architecture.md.
>
> Key changes from v1: Input-agnostic adapter layer, explicit four-layer architecture,
> quality vocabulary as the universal currency, Gesture Studio repositioned
> as offline training tool.
>
> Key changes from v2 draft: Expanded primitives (7 -> 11), smoothing pipeline,
> hysteresis on gates, continuous modulation mode, trajectory as first-class concept,
> bidirectional state protocol, operational essentials, revised roadmap.
>
> Informed by expert review (motion capture, interaction design, media programming,
> signal processing/ML, software architecture) — Feb 2026.

> Mutual transformative engagement through programmable audio responsiveness

## The One-Sentence Pitch

Ralf is a system that lets dancers influence music through movement — where a technologist configures the connection, a dancer can feel it responding, and a musician brings the sound.

---

## How It Works (Plain English)

A dancer moves. The system watches through a camera, phone, wristband, LiDAR sensor, or pressure mat — it doesn't matter what. An **adapter** translates the raw data into a common vocabulary of **qualities**: how fast, how jerky, how contracted, how symmetric.

The runtime reads two things at once:

1. **Qualities** — continuous measurements of *how* they're moving (fast? jerky? still? contracted?)
2. **Gestures** — specific recognized movements (*that was a jack, that was a wave*)

These feed into **Readings** — interpretations of what the movement means ("she's hitting the beat," "he's in flow," "they're mirroring each other").

Readings trigger **Intents** — what the system wants to do ("strip energy," "build tension"). Each intent has a **pool** of possible actions — the system **draws** one, weighted by tendencies. Same intent might produce a filter sweep one time, a percussion drop the next. This randomness is what keeps it feeling like a conversation, not a remote control.

The chosen action reaches into the **Sonic World** — the music a composer created — through a **Translator** that speaks whatever language the audio engine uses (Ableton, Tone.js, SuperCollider, anything).

The dancer hears the change. Their next move is shaped by what they heard. This is the **Listening Loop** — both sides listening, neither in full control.

---

## Core Concepts

### The Listening Loop

```
Dancer moves
    -> Adapter translates raw input into qualities
        -> Runtime smooths and reads qualities + gestures
            -> Readings interpret meaning (including trajectory)
                -> Intents express what should happen
                    -> Draw chooses how (weighted random from pool)
                        -> Translator speaks to the audio engine
                            -> Audio engine reports state back
                                -> Sound changes
                                    -> Dancer hears and responds
                                        -> (loop continues)
```

### Two Surfaces

The system reads movement through two complementary surfaces:

| Surface | Type | Example | Analogy |
|---------|------|---------|---------|
| **Quality** | Continuous stream, 0-1, every frame | Velocity: 0.7, Jerkiness: 0.2 | A fader — always moving |
| **Gesture** | Discrete event, fires once | `/gesture/jack` -> fire! | A button press |

Most interactive dance systems have one or the other. Ralf has both, and they can feed each other.

**What falls between these two surfaces:** State transitions (floor-to-standing over 2-5 seconds) and sustained poses (held for duration). These are handled by the Accumulate primitive (tracking duration) and on_exit intents (detecting release). See "Two Intent Modes" below.

### The Abstraction Boundary

The most important architectural decision: **qualities are the universal currency.**

Raw input data varies wildly — a skeleton has 33 named joints, a LiDAR produces point clouds, an IMU sends acceleration vectors, a pressure mat gives a 2D grid. These cannot be unified at the raw data level.

But the qualities we care about converge. See the Quality Vocabulary section below.

The adapter computes whichever qualities it can from its input. The runtime works with whatever arrives. A wristband produces 3 qualities. A full skeleton produces 12. Same runtime, same scenes — graceful degradation.

**Important caveat:** Two adapters reporting velocity 0.7 may not mean the same thing perceptually. A MediaPipe adapter at 30fps with 100ms latency produces different velocity curves than an IMU at 200Hz with 5ms latency. AdaptiveRange gives *statistical* equivalence (both use their full 0-1 range), not *perceptual* equivalence. This is fine for dance-music coupling — but don't expect cross-adapter readings to feel identical.

### Two Dimensions of Composition

| Dimension | What it is | Who creates it | Example |
|-----------|-----------|----------------|---------|
| **Sonic World** | The musical material — melodies, rhythms, sounds, loops, scenes | The composer/producer, in their DAW | An Ableton session, a set of samples |
| **Tendencies** | Weights that shape how the sonic world responds to movement | The interaction designer (might be the same person) | "When energy is stripped, filter sweep is 3x more likely than mute" |

The sonic world without tendencies is a static piece. Tendencies without a sonic world have nothing to act on.

**Semantic coherence rule:** Intent pools should contain actions that are perceptually similar — all variations on "more energy" or all variations on "pull back." The *domain* should be consistent even when the specific action varies. A pool containing both "unmute percussion" and "fire next scene" is hostile to the dancer's mental model because the outcomes are too different. Randomness enhances the conversation when the dancer can learn "this area of movement means more energy" without needing to predict the exact sonic result.

### Three Tiers of Portability

| Tier | What | Portable? | Format |
|------|------|-----------|--------|
| **Interaction design** | Readings, intents, draws, tendencies | Yes — works with any audio engine | Scene config (JSON) |
| **Musical material as samples** | Stems, loops, one-shots (e.g. from the Blender) | Yes — wav files + category metadata | Samples + perf.json |
| **Full production** | A complete Ableton session with synths, effects, routing | No — native to the DAW, and that's fine | Ableton project |

Tiers 1 and 2 together = a portable composition. Tier 3 is where a producer goes deeper in their native tool.

---

## The Four Layers

```
INPUTS           ADAPTERS          RUNTIME           TRANSLATORS
                 (separate         (one brain,        (separate
                  processes,        TS)                processes,
                  any language)                        any language)

MediaPipe -----> mediapipe/ -----> Smooth             Tone.js --------> browser
LiDAR ---------> lidar/ --------> -> Readings   ---> Max4Live -------> Ableton
IMU -----------> imu/ ----------> -> Intents         SuperCollider --> SC
Phone ---------> phone/ --------> -> Acts            OSC passthru ---> anything
Pressure mat --> pressure/ ----->
                                        |     ^
                                   WebSocket  |  /ralf/state/*
                                        |     |  (translator -> runtime)
                                  PERFORMANCE
                                  CONSOLE
                                  (browser UI)

                 GESTURE STUDIO
                 (offline training tool, Rust/Tauri)
                 Exports gesture templates the runtime loads at startup
```

### Layer 1: Inputs

The physical source of movement data. Could be anything — camera, sensor, phone, LiDAR.
Ralf doesn't control this layer. It just needs an adapter that speaks the input's language.

### Layer 2: Adapters

Each adapter is a **standalone process** that:
1. Receives raw data from its input (WebSocket, serial, OSC, whatever)
2. Computes whichever qualities it can from that data
3. Sends `/ralf/quality/<dancerId>/<quality> <float>` OSC to the runtime
4. Optionally recognizes gestures and sends `/ralf/gesture/<dancerId> <string>`

Adapters live in the monorepo:

```
adapters/
  shared/
    quality-math.ts       # Shared quality computation (velocity, jerkiness, etc.)
                          # Same math regardless of sensor — factor out early
  mediapipe/
    adapter.ts            # Receives 33-point skeleton, computes qualities
    manifest.json         # Declares name, qualities produced, input/output config
  imu/
    adapter.ts
    manifest.json
  lidar/
    adapter.py            # Can be any language — only contract is OSC output
    manifest.json
```

The manifest is metadata for the Performance Console, not enforced by the runtime:

```json
{
  "name": "mediapipe",
  "description": "33-point pose estimation from webcam",
  "qualities": ["velocity", "jerkiness", "contraction",
                "verticality", "symmetry", "coherence"],
  "input": { "type": "websocket", "defaultPort": 8765 },
  "output": { "type": "osc", "defaultPort": 9000 }
}
```

**Why separate processes, not plugins:**
- **Language agnostic.** Python LiDAR SDK? Write the adapter in Python. Rust IMU library? Write it in Rust. The only contract is "send OSC to this port."
- **Crash isolation.** A buggy adapter crashes itself, not the runtime. In a live performance, this matters.
- **No plugin architecture to build.** Plugin systems are complex. Separate processes give you the same extensibility with zero infrastructure.

**Quality math portability:** Factor quality computation into `adapters/shared/` early. The math for velocity/jerkiness/etc. from a sequence of positions is identical regardless of sensor. Implementing it slightly differently in each adapter will cause confusion when they behave differently.

**Signal processing guidance for adapter authors:**
- **Velocity**: 3-frame central difference, not frame-to-frame (reduces quantization noise at 30fps).
- **Acceleration**: Smooth before differentiating (Savitzky-Golay filter, order 2, window 5-7 frames).
- **Jerkiness**: Compute as windowed variance of acceleration, NOT as literal third derivative (which is pure noise at 30fps from MediaPipe). See Quality Vocabulary section.
- **Coherence**: Windowed Pearson correlation of left-side vs right-side velocity profiles, 20-frame window (~0.67s). Incrementally computable.
- **Outlier rejection**: If any joint moves more than 30cm in a single frame (9 m/s — physically impossible in dance), hold the previous frame's position.
- **MediaPipe confidence**: When a joint's visibility score drops below 0.5, hold last confident value. Critical for self-occlusion (dancer turns sideways).

### Layer 3: Runtime

The brain. A single TypeScript process that:
1. Receives quality and gesture OSC from adapters
2. Smooths incoming qualities (one-euro filter)
3. Runs the eleven primitives at 30fps
4. Evaluates readings, resolves intents (edge or continuous), fires actions
5. Sends `/ralf/act/*` OSC to translators
6. Receives `/ralf/state/*` OSC from translators
7. Serves a WebSocket for the Performance Console
8. Logs all acts, connections, and scene changes with timestamps

The runtime doesn't know what input devices exist. It doesn't know what audio engine is connected. It only knows qualities, gestures, readings, intents, and acts.

**Staleness policy:** If no quality update arrives for a dancer for 90 frames (3 seconds), decay all their quality values toward 0 at a rate of 0.02/frame. This prevents a disconnected dancer from leaving the system stuck at "high energy" forever. Log a warning when staleness decay begins.

### Layer 4: Translators

Each translator is a **standalone process** that:
1. Listens for `/ralf/act/*` OSC messages from the runtime
2. Converts them to engine-specific commands
3. Sends `/ralf/state/*` OSC back to the runtime (tempo, transport, scene index)

The runtime outputs two types of act messages:

```
# Discrete — translator should execute immediately
/ralf/act/trigger/fire_next_scene  0.91
/ralf/act/trigger/unmute_track     0.85  "perc"

# Continuous — translator should slew/interpolate to target
/ralf/act/set/filter_cutoff        0.73
/ralf/act/set/send_level           0.45  "reverb"
```

The `trigger/` vs `set/` prefix tells the translator whether to jump or interpolate. For `set/` messages, the translator handles smoothing (e.g., `linearRampToValueAtTime` in Tone.js, parameter interpolation in Ableton).

The reading's continuous 0-1 value is always the first argument. This means the dancer's body shapes the parameter, not just triggers a preset.

| Translator | What it does | When to use it |
|------------|-------------|---------------|
| **Tone.js** (browser) | WebSocket bridge -> Tone.js API calls | Workshops, quick jams, installations, Burning Man. Zero setup. |
| **Max4Live** (Ableton) | JS patch inside M4L -> Live API calls | Full production. Deep sound design. |
| **OSC pass-through** | Forwards /ralf/act/* as-is | SuperCollider, Pd, or any OSC-capable software. |

**The output side is deliberately unstructured.** Adapters (input) benefit from a defined protocol because Ralf controls the quality vocabulary. Translators (output) are someone else's domain — a producer maps things their way, a SuperCollider artist writes their own responder. The OSC act protocol IS the architecture. Anyone who can receive OSC can be a translator.

**What Ralf ships:**
1. The OSC protocol spec (the contract)
2. One reference translator (Tone.js — the "hello world in 2 minutes")
3. One production translator (M4L device)
4. A guide: "How to build a translator"

**M4L translator capabilities (reference):**
The M4L device should support at minimum: transport control (start/stop), scene firing, clip firing, track mute/unmute/volume, device parameter control (`live_set tracks N devices M parameters P`), send/return levels. It needs a name-to-index lookup for tracks (scan on load). Use `deferlow()` for LiveAPI calls to avoid audio dropouts. ~300-400 lines.

### Gesture Studio (Offline Tool)

Gesture Studio is **not in the live signal chain.** It's a workshop tool for training gesture templates.

| | Adapter (live) | Gesture Studio (offline) |
|---|---|---|
| **When** | Every frame, 30fps | Before the performance |
| **What** | Computes continuous qualities | Trains discrete gesture templates |
| **Output** | `/ralf/quality/*` streams | `.ralf` template files |
| **Consumer** | Runtime, in real time | Runtime loads templates at startup |

The training UI stays standalone (Rust/Tauri). The recognition engine (DTW matching) can be embedded in the adapter or run as a sidecar process — TBD. The runtime just receives `/ralf/gesture/<dancerId> <name>` and doesn't care who sent it.

**Gesture variability:** Templates should be trained on body-relative coordinates (normalized to shoulder-width and hip-height units). This handles inter-dancer body proportion differences. For temporal variability (same gesture at different speeds), DTW's time warping handles this within the Sakoe-Chiba band. For dancer populations with >30% speed variation, per-dancer template sets may be needed.

**Template metadata:** `.ralf` files should include which skeleton model produced them (e.g., "mediapipe-33"). Matching against templates from a different skeleton model will fail silently.

**Algorithm note:** DTW is the right choice for the Burning Man timeline — it's tested, interpretable, and works with few examples (6-10 per gesture). Post-Burn, a fine-tuned Temporal Convolutional Network (TCN) is the upgrade path. The OSC protocol already isolates this — the runtime doesn't care what recognition backend produced the gesture event.

---

## The Eleven Primitives

Everything is built from eleven node types. Think of them as Lego bricks — simple individually, powerful when combined.

### Core Pipeline (from v1)

| Node | What it does | One-line example |
|------|-------------|-----------------|
| **Sense** | Normalizes a raw quality to 0-1 via AdaptiveRange | "How jerky is this dancer right now? -> 0.6" |
| **Recognize** | Matches a movement pattern, fires an event | "That was a jack -> fire!" |
| **Accumulate** | Tracks something over time (windowed rate, total count, duration) | "3 gestures in the last 5 seconds" or "still for 12 seconds" |
| **Combine** | Mixes multiple values with weights to produce a Reading | "jerkiness x 0.6 + velocity x 0.3 = hitting the beat: 0.48" |
| **Draw** | Draws one action from a weighted pool (or deterministic in learning mode) | "strip_energy -> [filter sweep x3, mute perc x1] -> filter sweep wins" |
| **Gate** | Passes or blocks based on a condition, with hysteresis | "Only fire if energy > 0.5 AND we haven't acted in 10 seconds" |
| **Act** | Sends a message to the audio engine (trigger or continuous set) | "/ralf/act/set/filter_cutoff 0.7" |

### New Primitives (from expert review)

| Node | What it does | One-line example | Why it was missing |
|------|-------------|------------------|--------------------|
| **Smooth** | Temporal smoothing via one-euro filter | "Raw jerkiness 0.73 -> smoothed 0.68" | MediaPipe is noisy; without this, thresholds chatter and audio parameters zipper |
| **Delay** | Time-offsets a signal | "React to velocity from 2 seconds ago" | Common in Max/MSP (`[pipe]`); enables echo, call-and-response patterns |
| **Map** | Applies a transfer function (curve, dead zone, steps) | "Linear 0-1 -> exponential response" | Linear mapping almost never feels natural for body-to-sound |
| **Latch** | Captures a value at the moment of an event and holds it | "Velocity was 0.8 when the jack happened -> hold 0.8" | Enables "the speed of the gesture determines the intensity of the response" |

### The Smoothing Pipeline

Smoothing is not optional. Without it, the system is unusable with real sensor data.

```
Raw quality from adapter
    -> Sense (AdaptiveRange normalization to 0-1)
        -> Smooth (one-euro filter: low latency during fast movement, strong smoothing during stillness)
            -> Available for readings, gates, thresholds
```

The one-euro filter was invented for exactly this use case (noisy pose tracking at 30-120fps). Its two parameters:
- **min cutoff frequency**: controls smoothing during slow movement (lower = smoother, more latency)
- **speed coefficient**: controls how quickly smoothing reduces during fast movement (higher = more responsive)

Defaults: `minCutoff: 1.0, beta: 0.007` — tunable per quality in scene config.

**Deadband on Act emission:** Don't send a `/ralf/act/set/*` message if the value changed by less than 0.01 since the last emission. Prevents flooding the audio engine with 30 near-identical messages per second.

### Gate with Hysteresis (Schmitt Trigger)

Gates use two thresholds to prevent oscillation around boundaries:

```
Enter active state when value crosses: threshold + hysteresis_band
Exit active state when value drops below: threshold - hysteresis_band
```

Default `hysteresis_band: 0.05`. Configurable per gate in scene config.

Without this, a quality hovering at 0.49-0.51 around a 0.5 threshold produces rapid active/inactive toggling, causing spurious edge fires even with edge detection. This is the single most common bug in threshold-based interactive systems.

**Minimum activation duration:** Optionally, don't transition to active unless the value has been above the enter threshold for N consecutive frames (default: 2 frames / 66ms). The temporal equivalent of the VAD frame accumulation in Gesture Studio.

### Two Intent Modes

Intents support two firing modes:

| Mode | When it fires | Use case | Scene config |
|------|--------------|----------|-------------|
| **Edge** (default) | Once, on rising edge (inactive -> active) | Structural events: fire a scene, unmute a track | `"mode": "edge"` |
| **Continuous** | Every tick while active, passing reading value | Sustained parameter shaping: filter sweep, volume ride | `"mode": "continuous"` |

Edge mode also supports **on_exit intents** — fired on falling edge (active -> inactive). This captures the expressive moment of release: a dancer ending a sustained contraction, dropping from a held pose.

```json
{
  "id": "contracted-state",
  "mix": { "contraction": 1.0 },
  "gate": { "contraction": { "above": 0.6 } },
  "intents": ["tension_building"],
  "on_exit": ["tension_release"]
}
```

**Learning mode:** Intent pools support `"deterministic": true`, which disables randomness (highest weight always wins). Use during rehearsal so the dancer can build a reliable mental model. Switch to stochastic for performance.

### Trajectory

Trajectory is the direction and rate of change of any quality or reading — the first derivative over time. It is what enables the system to distinguish:

- **Building** (value increasing) — exciting, escalating
- **Sustaining** (value stable) — maintaining, holding
- **Releasing** (value decreasing) — resolving, winding down

Without trajectory, the system treats "accelerating from stillness" and "decelerating from explosion" identically when they pass through the same value. This is the single biggest source of "the system doesn't get me" frustration.

**Implementation:** Windowed linear regression slope over the last 5-10 frames. Output normalized to approximately [-1, 1] where positive = building, negative = releasing, near-zero = sustaining.

Trajectory is available as a meta-quality that can feed readings:

```json
{
  "id": "energy-building",
  "mix": { "velocity": 0.5, "energy": 0.5 },
  "trajectory": { "window": 10, "above": 0.2 },
  "intents": ["build_tension"]
}
```

This reading activates not when energy is *high*, but when energy is *rising*.

---

## Qualities

### Universal Quality Vocabulary

Qualities are the universal currency. Each is a 0-1 float, self-calibrating via AdaptiveRange, then smoothed via one-euro filter.

**Tier 1 — works with any tracked point(s):**

| Quality | Measures | Meaning | Computation notes |
|---------|---------|---------|-------------------|
| Velocity | Speed of tracked points | How fast | 3-frame central difference to reduce quantization noise |
| Acceleration | Rate of speed change | How much change | Savitzky-Golay (order 2, window 5-7) — smooth before differentiating |
| Jerkiness | Abruptness of movement | How abrupt | Windowed variance of acceleration (NOT literal 3rd derivative — that's pure noise at 30fps) |

**Dropped from v2 draft:** Fluidity (was defined as inverse jerkiness). If it's literally `1 - jerkiness`, it carries no information. Scene config can use negative weights or the Map primitive to invert. If a distinct smoothness measure is needed later, use spectral arc length (SPARC) over a 1-second window.

**Tier 1b — derived temporal qualities (any adapter can produce with a time buffer):**

| Quality | Measures | Meaning | Computation notes |
|---------|---------|---------|-------------------|
| Stillness | Duration below velocity threshold | How long they've been still | Timer on velocity < 0.1; normalized 0-1 over configurable max (e.g., 30 seconds). A sustained freeze is one of the most expressive moments in dance. |
| Periodicity | Rhythmic regularity of movement | How rhythmic | Autocorrelation of velocity over 2-4 second window; peak height = regularity. Directly useful for "on the beat" readings. |
| Groundedness | Weight quality of movement | How heavy/grounded vs light/floaty | Approximated from vertical acceleration patterns and contact duration. Distinguishes heavy/grounded from light/floaty. |

**Tier 2 — needs multiple tracked points:**

| Quality | Measures | Meaning |
|---------|---------|---------|
| Energy | Total kinetic energy | How much movement overall |
| Spatial extent | Spread from center | How big / open |
| Contraction | Limb distance from center | How closed |
| Symmetry | Left-right balance | How even |
| Coherence | Left-right timing match | How coordinated |

**Coherence computation:** Windowed Pearson correlation of left-side velocity (avg of left wrist, elbow, ankle, knee speeds) vs right-side velocity. Window: 20 frames (~0.67s). Output [-1, 1] mapped to [0, 1] where 1 = perfectly synchronous, 0 = uncorrelated. Incrementally computable with running covariance formula.

**Tier 3 — needs spatial reference:**

| Quality | Measures | Needs | Meaning | Clarification |
|---------|---------|-------|---------|---------------|
| Verticality | Center-of-mass height | Vertical axis | How high/low | Normalized to dancer's own standing height range, NOT spine angle. A dancer can be upright (straight spine) but low (deep plié). Height matters more for music coupling. |
| Heading | Body orientation | Defined "front" | Which direction | |

Not every adapter produces every quality. That's fine. The scene config declares which qualities each reading uses; missing qualities simply don't contribute.

### What each input gives you

| Input | Tier 1 | Tier 1b | Tier 2 | Tier 3 | Notes |
|-------|--------|---------|--------|--------|-------|
| Camera (MediaPipe, 33 joints) | All | All | All | All | Richest single source |
| Depth camera (Kinect, RealSense) | All | All | All | All | Plus real depth |
| LiDAR (point cloud) | All | All | All | Verticality | Heading depends on cluster analysis |
| Phone accelerometer | All | Stillness, Periodicity | -- | -- | Single point, no spatial |
| Wristband / Watch | All | Stillness, Periodicity | -- | -- | Single limb |
| Pressure mat (2D grid) | Velocity, Accel | Stillness, Periodicity | Extent, Symmetry | -- | No verticality. Groundedness directly measurable. |
| Crowd of phones | All (aggregate) | Periodicity (aggregate) | Synchrony (aggregate) | -- | Special case for crowd mode |

### Relational Qualities (between dancers — future)

Computed by the runtime from pairs of individual quality streams. Adapters don't need to know about other dancers.

| Quality | Measures | Meaning | Computation |
|---------|---------|---------|-------------|
| Proximity | Distance between centers | How close | L2 distance of hip centroids, normalized to room scale |
| Mirroring | Position correlation | How alike (shape) | Reflect one dancer about their vertical axis, compute mean joint-to-joint distance |
| Synchrony | Velocity profile correlation | How in time | Windowed Pearson correlation of velocity profiles, 30-45 frame window (1.0-1.5s) |
| Contrast | Quality profile divergence | How different | L2 distance between quality vectors |
| Convergence | Signed rate of proximity change | Approaching or departing | Smoothed derivative of proximity. Positive = approaching, negative = departing. Range [-1, 1], NOT [0, 1]. |
| Leader-follower | Temporal offset of velocity correlation | Who's initiating | Cross-correlation with lag parameter; sign of peak lag = who leads |

**Crowd-scale qualities (for 20+ participants):**

Pairwise relational qualities don't scale to crowds. For crowd mode, compute aggregate spatial qualities:

| Quality | Measures | Meaning |
|---------|---------|---------|
| Cluster count | Number of movement groups | How fragmented |
| Dispersion | Spread vs. clustered | How scattered |
| Formation stability | Rate of cluster change | How stable are the groups |
| Aggregate synchrony | Mean pairwise velocity correlation | How together |

---

## The Smoothing & Noise Pipeline

This section is critical. Without it, the system is unusable with real sensor data.

### End-to-End Signal Path

```
Raw sensor data (noisy)
    -> Adapter: outlier rejection (teleportation filter)
    -> Adapter: quality computation (3-frame diff for velocity, etc.)
    -> OSC transport
    -> Runtime: Sense (AdaptiveRange normalization to 0-1)
    -> Runtime: Smooth (one-euro filter)
    -> Runtime: readings, gates, thresholds, trajectory
    -> Runtime: deadband filter on Act emission (skip if delta < 0.01)
    -> OSC transport
    -> Translator: interpolation for continuous parameters
```

### Noise Amplification at Each Derivative

This is why smoothing matters more for higher-order qualities:

| Stage | Noise amplification | MediaPipe noise at this stage |
|-------|-------------------|------------------------------|
| Position (raw) | 1x | ~1-2cm std dev per joint |
| Velocity (1st derivative) | ~30x (1/dt at 30fps) | ~30-60 cm/s |
| Acceleration (2nd derivative) | ~900x | ~900-1800 cm/s^2 |
| Jerkiness (3rd derivative) | ~27000x | **Unusable without windowed variance approach** |

This is why jerkiness MUST be computed as windowed variance of acceleration, not as the literal third derivative.

### AdaptiveRange Improvements

The self-calibrating normalizer works, but needs these refinements:

**Configurable decay rate:** Expose in scene config. Default 0.001 at 30fps (~23 second half-life). For workshop demos (5 min), use 0.005. For long installations (2 hours), use 0.0005.

**Warm-up handling:** Return 0.5 (midpoint) for the first 90 frames (3 seconds) instead of normalizing against a tiny initial range. This prevents "everything saturated" during the first moments when the range hasn't established.

**Minimum range floor:** Don't let `max - min` collapse below a configurable minimum (default: 0.05 of historical max). Prevents a long still period from making the next tiny fidget read as 1.0.

**Asymmetric decay (optional):** For qualities with a physical floor near 0 (velocity, energy), the max should decay faster than the min. Max-decay of 0.003 with min-decay of 0.001 keeps the system sensitive to current dynamics rather than remembering a peak from 2 minutes ago.

---

## OSC Protocol

The protocol is the architecture. These are the only messages that cross layer boundaries.

### Adapter -> Runtime (quality stream)

```
/ralf/quality/<dancerId>/<quality>  <float 0-1>
/ralf/gesture/<dancerId>            <string>
```

Examples:
```
/ralf/quality/maya/velocity      0.73
/ralf/quality/maya/jerkiness     0.31
/ralf/quality/kai/velocity       0.55
/ralf/gesture/maya               "jack"
```

Valid quality names: `velocity`, `acceleration`, `jerkiness`, `energy`, `spatial_extent`, `contraction`, `symmetry`, `coherence`, `verticality`, `heading`, `stillness`, `periodicity`, `groundedness`.

Unknown quality names are silently ignored (forward-compatible).

### Runtime -> Translator (act messages)

Two message types distinguished by prefix:

```
/ralf/act/trigger/<action>  <readingValue float> [args...]   # Execute immediately
/ralf/act/set/<action>      <readingValue float> [args...]   # Slew/interpolate to value
```

Examples:
```
/ralf/act/set/filter_cutoff       0.73
/ralf/act/trigger/unmute_track    0.85  "perc"
/ralf/act/trigger/fire_next_scene 0.91
/ralf/act/set/send_level          0.45  "reverb"
/ralf/act/set/device_param        0.6   "synth" 2
```

### Translator -> Runtime (state messages)

Translators report audio engine state back to the runtime. This enables tempo-synced behaviors, prevents double-muting, and lets the runtime know what the audio engine is doing.

```
/ralf/state/tempo     <float bpm>
/ralf/state/playing   <int 0|1>
/ralf/state/scene     <int index>
```

The runtime stores this state and makes it available to readings and gates. For example, a gate could require `playing == 1` before firing intents, or a reading could use tempo to detect beat alignment.

### Health (all processes)

```
/ralf/ping  <string processName>
/ralf/pong  <string processName>
```

Every process responds to `/ralf/ping` with `/ralf/pong`. The Performance Console uses this to show process health. When debugging at 3 AM on the playa, this tells you *where* in the chain things stopped.

---

## Scene Config

A Scene is a JSON file that declares everything about a performance setup.

```json
{
  "version": 1,
  "name": "House Session with Maya",

  "settings": {
    "adaptive_range_decay": 0.001,
    "hysteresis_band": 0.05,
    "staleness_frames": 90
  },

  "dancers": [
    { "id": "maya", "adapter": "mediapipe", "port": 6448 }
  ],

  "readings": [
    {
      "id": "hitting-the-beat",
      "mix": { "jerkiness": 0.6, "velocity": 0.3 },
      "gate": { "jerkiness": { "above": 0.2 } },
      "intents": ["add_energy"],
      "on_exit": ["settle_down"]
    },
    {
      "id": "energy-level",
      "mix": { "velocity": 0.5, "jerkiness": 0.5 },
      "intents": [
        { "intent": "strip_energy", "below": 0.3 },
        { "intent": "add_energy", "above": 0.7 }
      ]
    },
    {
      "id": "energy-building",
      "mix": { "velocity": 0.5, "energy": 0.5 },
      "trajectory": { "window": 10, "above": 0.2 },
      "intents": ["build_tension"]
    },
    {
      "id": "sustained-intensity",
      "mix": { "energy": 1.0 },
      "gate": { "energy": { "above": 0.7 } },
      "intents": [
        { "intent": "ride_filter", "mode": "continuous" }
      ]
    }
  ],

  "intents": {
    "add_energy": {
      "deterministic": false,
      "pool": [
        { "action": "trigger/fire_next_scene", "weight": 2 },
        { "action": "trigger/unmute_track", "args": { "track": "perc" }, "weight": 3 }
      ]
    },
    "ride_filter": {
      "pool": [
        { "action": "set/filter_cutoff", "weight": 1 }
      ]
    },
    "build_tension": {
      "pool": [
        { "action": "set/send_level", "args": { "send": "reverb" }, "weight": 2 },
        { "action": "trigger/unmute_track", "args": { "track": "texture" }, "weight": 1 }
      ]
    }
  },

  "translator": { "type": "ableton", "port": 12000 }
}
```

### Scene validation

On load, the runtime should validate:
- All quality names in `mix` are valid quality names
- All intent references in readings resolve to defined intents
- All gate quality references are valid
- Version is recognized

A typo in a quality name (`"velocty"`) would silently produce a 0 reading without validation. 20 lines of code. Saves hours of debugging in a dark tent.

### Scene hot-reload

The runtime must support updating individual scene properties (reading weights, gate thresholds, intent pools) without resetting dancer AdaptiveRange calibration or edge state. Losing calibration mid-performance because someone tweaked a weight is destructive.

`loadScene()` reinitializes everything. `updateScene(patch)` applies surgical updates. Both are needed.

---

## Operational Essentials

These are not nice-to-haves. They are prerequisites for live performance.

### Logging

Log with timestamps to both stdout and a file:
- Every Act message sent
- Every scene load/reload
- Every dancer connection/disconnection
- Every staleness decay trigger
- Process startup/shutdown

When it breaks at 3 AM on the playa, logs are your only friend. 30 minutes of work.

### Decay on Disconnect

If no quality update arrives for a dancer for 90 frames (3 seconds), begin decaying all their quality values toward 0 at 0.02/frame. Log a warning. This prevents a dancer walking away from the camera from leaving the system stuck in whatever state they were in.

### Graceful Degradation

- Adapter crashes: runtime continues with last known values, then staleness decay kicks in.
- Translator crashes: runtime continues evaluating — acts go nowhere but state is preserved. Translator reconnects and picks up the stream.
- Runtime crash: all other processes keep running. Restart runtime, reload scene.

---

## Build Roadmap

Target: Burning Man (summer 2026, ~4 months from Feb 2026)

**Revised from v2 draft based on expert consensus: move dancer testing earlier, cut visual patcher, prioritize crowd mode.**

| Phase | What | Deliverable | Est. |
|-------|------|-------------|------|
| **1** | Runtime core (DONE) | TS server, 7 original primitives, OSC in/out, WebSocket, scene load. 27 tests passing. | Done |
| **2** | Smoothing + signal robustness | One-euro filter (Smooth primitive), hysteresis on gates, deadband on Act emission, staleness decay, AdaptiveRange warm-up + min range. Wire Accumulate and Gate into runtime. Add Trajectory. Scene validation. Logging. | 2 weeks |
| **3** | First piece with a dancer | MediaPipe adapter computing qualities. Real integration test. Hot-reload scene properties. Expect to discover what's actually broken. | 1-2 weeks |
| **4** | Crowd mode | Lightweight phone web app -> server via WebSocket (phones can't send OSC/UDP). Aggregate qualities. The Burning Man scenario. | 3 weeks |
| **5** | Monitoring page | Read-only browser UI: dancer states, reading values, recent acts, process health. Single HTML file, no build step. Not a visual patcher — that's post-Burn. | 3 days |
| **6** | Hardening | One-command launcher (spawns runtime + adapters). Heartbeat/ping. Stability. Fallback modes. | 2 weeks |
| **—** | Buffer | Things will break. | remaining |

**What was cut:** The visual patcher / Performance Console (previously Phase 3). Author scenes as JSON — a text editor is the patcher for now. The patcher is a post-Burn project. Expert consensus: "The visual patcher is where solo developers die."

**What was added:** Smoothing/hysteresis as its own phase (was scattered across later phases). Monitoring page as a minimal viable console.

### Post-Burn

| What | Notes |
|------|-------|
| Visual patcher (ReactFlow) | The original Phase 3. JSON becomes serialization format, not authoring format. |
| Map, Delay, Latch primitives | Build when needed for specific pieces, not speculatively. |
| Graph evaluation engine | Replace fixed pipeline with topological node evaluation. Required for visual patcher. |
| Relational qualities | Proximity, mirroring, synchrony for duo scenarios. |
| TCN gesture recognition | Drop-in replacement for DTW inside Gesture Studio. |
| Tempo/beat sync | Runtime clock concept synced to DAW transport. Enables beat-aligned behaviors. |

---

## Repo Structure

```
ralf/                           # Monorepo (not a git repo itself)
  docs/
    architecture.md             # This document
  runtime/                      # Git repo: the brain (TS)
    src/
      engine/runtime.ts         # Tick loop, reading evaluation, intent resolution
      primitives/               # Sense, Recognize, Accumulate, Combine, Draw, Gate, Act,
                                # Smooth, Delay, Map, Latch
      transport/                # OSC server, WebSocket server
      scenes/                   # Scene loader + validator
    scenes/                     # JSON scene configs
  adapters/                     # Git repo: all adapters
    shared/
      quality-math.ts           # Shared quality computation library
    mediapipe/
      adapter.ts
      manifest.json
    imu/
      adapter.ts
      manifest.json
  translators/                  # Git repo: all translators
    tonejs/                     # Reference translator (browser)
    max4live/                   # Production translator (Ableton)
  gesture-studio/               # Git repo: offline gesture training (Rust/Tauri)
  sound-engine/                 # Tone.js engine + Blender (stem separation)
  mediapipe/                    # Pose tracking frontend (existing)
```

---

## Glossary

Every term earns its place.

| Term | Definition |
|------|-----------|
| **Quality** | A continuous 0-1 measurement of how a body moves (velocity, jerkiness, etc.) — the universal currency |
| **Gesture** | A recognized discrete movement pattern that fires as an event |
| **Reading** | An interpretation of movement — qualities combined through weights and gates into named meaning |
| **Trajectory** | The direction and rate of change of a quality or reading — building, sustaining, or releasing |
| **Intent** | What the system wants to do, symbolically ("strip energy," "build tension") — edge-triggered or continuous |
| **Draw** | Weighted random selection from an intent's pool of possible actions (deterministic in learning mode) |
| **Gate** | A condition that passes or blocks, with hysteresis to prevent oscillation |
| **Act** | A message sent to the audio engine — either trigger (immediate) or set (interpolated) |
| **Adapter** | A standalone process that translates raw input data into the quality vocabulary via OSC |
| **Translator** | A standalone process that converts Act messages into a specific audio engine's language, and reports state back |
| **Manifest** | Metadata file declaring what an adapter produces (for the Performance Console, not enforced) |
| **Scene** | The full configuration of a performance — who, what inputs, which readings, which tendencies, what translator |
| **Sonic World** | The musical material a composer creates (samples, synths, clips, scenes) |
| **Tendencies** | The weights that stack the deck — shaping probability, not dictating outcome |
| **Listening Loop** | The continuous cycle: move -> read -> interpret -> choose -> sound -> hear -> move |
| **Blender** | Pipeline that takes a track, splits stems, outputs categorized samples — a portable sonic world |
| **Adaptive Range** | Self-calibrating normalizer with exponential decay, warm-up period, and minimum range floor |
| **One-Euro Filter** | Adaptive low-pass filter: strong smoothing during stillness, low latency during fast movement |
| **Hysteresis** | Schmitt trigger behavior on gates: separate enter/exit thresholds to prevent oscillation |
| **Staleness** | Quality values decay toward 0 when no updates arrive, preventing frozen state from disconnected dancers |

---

## Design Principles

1. **"Hello world" in under 2 minutes.** One command, browser opens, camera sees you, sound responds. Configure later. (The browser-only path: MediaPipe + quality computation + Tone.js all in-browser via WebSocket to runtime.)

2. **The primitives are the product.** Everything else is plumbing. Use boring technology for plumbing.

3. **Protect the vocabulary.** These words are the framework. They matter more than the code.

4. **Input agnostic, quality native.** Raw data varies wildly. Qualities converge. The adapter boundary is where diversity becomes uniformity.

5. **Separate brain from sound.** The runtime thinks. The translator speaks. Swap the translator, swap the world.

6. **Processes, not plugins.** Each layer is a standalone process. Crash isolation, language freedom, zero infrastructure. OSC is the universal glue.

7. **Three views, one system.** Technologist sees the config. Dancer sees feedback. Musician sees triggers.

8. **Influence, not control.** Weighted probability, not deterministic mapping. The tendencies shape the space; they don't dictate the path. Pools should be semantically coherent.

9. **Start with one piece.** Every successful creative framework started as one person's tool for one specific project. The platform emerges from practice.

10. **Smooth the signal, not the experience.** Apply aggressive noise filtering to the data pipeline so the interaction can be responsive and direct. The dancer should feel connection, not latency.

11. **Time is a dimension.** Movement is temporal — trajectory, duration, periodicity, and release matter as much as instantaneous state. The system must read *over time*, not just *right now*.
