# EightOS (∞OS) — Deep Analysis

**Date:** 2026-04-06
**URL:** https://8os.io/
**Creators:** Dmitry Paranyushkin + Kirikoo Des (NSDOS)
**Org:** Nodus Labs / Special Agency (Berlin-based)
**Active since:** 2015 (v1: 2018, v2: 2019–present)
**License:** CC BY-SA 4.0, code is open source (GNU AGPL v3)

---

## What It Is

EightOS calls itself an "open-source operating system for the bodymind." It's a practice-based framework that combines contemporary dance, Systema martial art, complex systems theory, network science, and technology to create feedback loops between movement and sound/visuals/haptics.

The core idea: use **Detrended Fluctuation Analysis (DFA)** to measure the fractal properties of a dancer's movement in real-time, classify the movement into one of four dynamical states, and feed that classification back as sound, visuals, and haptic vibration — creating a biofeedback loop that shapes how people move.

---

## The Technical Pipeline

```
xSens DOT sensors (IMU, Bluetooth)
    → xSens DOT Server (Node.js + Noble BLE)
        → EightOS Fractal Analysis Server (Python: NumPy, SciPy, Pandas)
            → DFA calculates alpha coefficient from acceleration data
                → OSC messages sent to:
                    → Usine (sound + haptic feedback via Wi-Fi tripod sensor)
                    → Orca / HundredRabbits (sonic soundscape generation)
                    → Projection display (visual state indicator)
```

### OSC Message Channels
| Channel | Data |
|---------|------|
| `/alpha/sensor_id` | Individual sensor alpha value (float ~0.3–1.5) |
| `/alpha_note/sensor_id` | Categorical state: A, B, C, or D |
| `/alpha_avrg/all` | Average alpha across all sensors |
| `/alpha_history/sensor_id` | Last 3 states averaged |
| `/alpha_history_signal/sensor_id` | Recommendation codes (V, L, R, F, W, S, T, M) |

**Update rate:** Every 30–40 seconds (DFA needs a window of data to compute)

### Hardware
- **Sensors:** xSens DOT wireless IMUs (Bluetooth 4.0+)
- **Processing:** Laptop or Raspberry Pi
- **Haptic feedback:** Custom Wi-Fi-enabled tripod sensor
- **Supported platforms:** Windows, macOS, Raspberry Pi

---

## The Four Movement States (Alpha Coefficient)

DFA analyzes acceleration time series and produces an alpha exponent that classifies movement quality:

| State | Alpha Range | Meaning |
|-------|------------|---------|
| **Random** | 0 < α < 0.6 (ideal: 0.5) | Movements fluctuate around an average; repetitive or idle |
| **Regular** | 0.6 < α < 0.85 | Auto-correlated; predictable patterns, large fluctuations likely over time |
| **Fractal** | 0.85 < α < 1.15 (ideal: 1.0) | Scale-free; small-scale patterns mirror large-scale patterns |
| **Complex** | α > 1.15 (~1.5 max) | Non-stationary; movement quality changes drastically |

**Key insight:** Natural, adaptive movement tends toward fractal (α ≈ 1.0). Tension or pathology pushes toward randomness. The practice aims to help people move through ALL four states rather than getting stuck in one.

---

## The Seven Operating Principles

1. **Tensivity** — Removing physical/mental blocks to increase freedom
2. **Polysingularity** — Supporting multiple simultaneous perspectives while maintaining stability
3. **Conductivity** — Iterative assimilation of external structures
4. **Adaptive Fluidity** — Redirecting incoming impulses beneficially
5. **Integrity** — Dynamic interconnectedness maintaining system stability
6. **Intra-Contextual Interoperability** — Cross-disciplinary application
7. **Programmability** — Customizable, open-source modification

---

## SomaSync App (Related Product)

A companion iOS/Apple Watch app that uses the same DFA framework for HRV (Heart Rate Variability) training:
- Measures DFA alpha from heart rate data via Polar H10 chest strap
- Four training protocols: Deep Breathing, Performance Prep, Recovery Mode, Sleep Preparation
- Uses "fractal breathing" — variable rhythms rather than fixed patterns
- Maps HRV alpha to same states: recovering (<0.8), regular, resilient (0.9–1.1), tension (>1.3)
- Currently in free TestFlight beta
- URL: https://somasync.app

---

## GitHub Repos

- **[deemeetree/xsens_dot_server](https://github.com/deemeetree/xsens_dot_server)** — Modified xSens DOT server with DFA + OSC output (Node.js + Python)
- **[deemeetree/dfa](https://github.com/deemeetree/dfa)** — JavaScript DFA library, zero dependencies, available on npm as `dfa-variability`
- **[noduslabs/8os](https://github.com/noduslabs/8os)** — Main project repo

---

## Where They've Presented

- Triennale Milano
- Movement Masterclass
- Various workshops/activations across Europe
- Active practice community (WhatsApp/Telegram groups, Facebook)

---

## RALF vs. EightOS — Comparison

### What They Share
- Movement → Sound feedback loop
- OSC as the signal transport protocol
- IMU/motion sensors on the body
- Interest in collective/group dynamics
- Open source ethos
- Performance + practice context
- Philosophy of embodied computation

### Where They Diverge

| Dimension | EightOS | RALF |
|-----------|---------|------|
| **Analysis approach** | Single metric: DFA alpha coefficient (fractal analysis of acceleration) | Multiple qualities: velocity, jerkiness, coherence, spread, symmetry, etc. |
| **Update rate** | ~30-40 seconds (DFA needs windowed data) | Real-time at ~30fps |
| **Movement vocabulary** | Four states (random, regular, fractal, complex) | Open-ended: readings, gates, intents, action pools |
| **Sound mapping** | State → soundscape (4 sonic worlds) | Quality → reading → intent → action (composable, per-scene) |
| **Sensing** | xSens DOT IMUs (Bluetooth, dedicated hardware) | MediaPipe (camera, no hardware) + phone IMUs (crowd mode) |
| **Compositional model** | Implicit: each state has a sonic identity | Explicit: scene editor, action manifests, weighted pools, trigger/set modes |
| **Primary metaphor** | Operating system / software for the body | Design grammar / language for translating movement |
| **Theoretical grounding** | Dynamical systems theory, fractal analysis, network science | African diasporic aesthetics, collective musical experience, technologies of remembering |
| **Crowd dynamics** | Group sessions mentioned, multi-sensor support | Explicit relational qualities: synchrony, contrast, aggregate_energy, virtual _crowd dancer |
| **Hardware dependency** | Requires xSens DOT sensors (~$100+ each) | Camera-only for solo, phone-only for crowd |
| **Haptic feedback** | Yes (custom Wi-Fi tripod sensor) | Not yet |
| **HRV/biometric** | Yes (SomaSync app, Polar H10) | Not yet |

### Key Differences in Philosophy

**EightOS reduces movement to one number (alpha) and classifies it into four states.** This is elegant and scientifically grounded but coarse — you know the fractal quality of movement but not *what* the movement is. A dancer waving their arm randomly and a dancer shuffling their feet randomly produce the same alpha.

**RALF extracts multiple qualities and lets the composer wire them to specific sonic responses.** This is more expressive and composable but requires more design work. The composer decides what matters and how it sounds.

**EightOS is a practice framework with technology.** The tech serves the practice — DFA validates the movement quality the practitioner is cultivating.

**RALF is a composition tool with practice implications.** The tech IS the creative medium — the scene file is the score.

---

## Opportunities

### Learn From
- DFA as an *additional* quality in RALF (add a "fractality" reading that computes alpha over a sliding window)
- Haptic feedback as an output modality for translators
- HRV integration — SomaSync's approach to biometric feedback could complement movement data
- Their framework for talking about movement states is clear and teachable

### Differentiate On
- Real-time responsiveness (30fps vs 30-40 sec updates)
- Compositional depth (scene editor, action manifests, weighted pools)
- Zero-hardware entry point (camera-based via MediaPipe)
- Crowd-native architecture (relational qualities, virtual crowd dancer)
- Cultural grounding (African diasporic aesthetics vs European systems theory)

### Potential Collaboration
- RALF could consume EightOS's DFA alpha as an input quality (they already output OSC)
- Shared performances: EightOS providing macro-state feedback while RALF handles micro-gestural sound
- Cross-pollination of practice communities

---

## Key People to Know

- **Dmitry Paranyushkin** — Creator, based in Berlin. Born in Moscow. Visual artist, performer, web systems designer. Runs Nodus Labs (network analysis tools). Portfolio: https://paranyushkin.com/
- **Kirikoo Des (NSDOS)** — Co-creator. Artist working across dance, music, and digital art. Handles the sound/music side.
- **Special Agency** (specialagency.co) — Management entity

---

## Sources

- https://8os.io/
- https://8os.io/docs/
- https://8os.io/fractal-choreography-motion-capture-haptic-sensors/
- https://8os.io/chaotic-fractal-movement-variability/
- https://polysingularity.com/8os/
- https://noduslabs.com/research/8os-bodymind-operating-system/
- https://paranyushkin.com/portfolio-item/eightos-version-2/
- https://somasync.app
- https://github.com/deemeetree/xsens_dot_server
- https://github.com/deemeetree/dfa
