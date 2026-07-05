# SOMI-1 Integration for RALF

**Status:** Decision made — Path A (MIDI→OSC bridge as new adapter). Implementation pending.
**Date:** 2026-05-06

## Why this matters

RALF's pipeline is built around an OSC bus on `:6449` with input-agnostic adapters (`mediapipe/`, `imu/`). Adding the **SOMI-1** as a first-class movement source gives us:

- Wearable, body-worn input that doesn't depend on a camera
- 6 sensors per hub, sub-10ms latency, 50m range
- Semantic motion data (action triggers, 3 inclination axes, 3 acceleration axes per sensor) — not just raw accel
- A second movement modality alongside MediaPipe's vision-based skeletal tracking, which expands the "mutually transformative engagement" surface

## What we figured out

### SOMI-1 is much more open than it looks

| Layer | Status |
|---|---|
| **Editor app** | Open source (C++/JUCE, GPL-3) — github.com/instrumentsofthings/SOMI-1-Editor |
| **Hub output** | MIDI 1.0 over USB-MIDI / TRS-MIDI / MIDI Host. No native OSC. |
| **Hub-to-sensor link** | Custom Bluetooth 5 protocol, <10ms latency, 6 sensors max |
| **Underlying sensors** | Off-the-shelf **Movesense** modules with documented BLE GATT (IMU notify on UUID `0x2BF2`) and an open Python real-time SDK |
| **Auth / cloud lock-in** | None. The hub is a self-contained device. |

### Three viable integration paths

**Path A — MIDI→OSC bridge as a new adapter** *(chosen)*
- Plug SOMI-1 hub via USB-MIDI; new `adapters/somi-1/` reads MIDI and republishes as OSC to `:6449`
- ~30 lines once the OSC schema is decided
- Keeps SOMI-1's optimized BLE path and 6-sensor multiplexing for free
- Bottleneck: 7-bit CC (128 values) per parameter; 14-bit available for some

**Path B — Fork the editor to add OSC output** *(ruled out by evidence, 2026-05-07)*
- Originally considered as a way to get higher-fidelity data than Path A's MIDI stream
- **Source review of github.com/instrumentsofthings/SOMI-1-Editor proves this is a chimera:**
  - `MainComponent.cpp:166-177` — editor uses standard `juce::MidiInput`, no custom USB driver or serial channel
  - `MainComponent.cpp:286-360` — every hub→editor message is either standard MIDI CC (live data) or SysEx (configuration only); no streaming SysEx command exists in the `SysExCmd` enum (`SomiDefines.h:64-79`)
  - `MainComponent.cpp:308` — the editor's own activity UI divides `getControllerValue()` by 127.0f, consuming the same 7-bit CC values we'd receive in Path A; comment explicitly states "For high resolution values, only high part is used"
- The hub's only data-output protocol is MIDI. The editor isn't a higher-bandwidth client — it's a JUCE-wrapped MIDI receiver
- 14-bit CC resolution (`midi_cc_t.high_resolution` in `data_types.h`) is configurable per-CC via the released editor binary and works just fine in Path A — no fork required

**Path C — Bypass the hub entirely via Movesense BLE** *(parked, probably never)*
- Read sensors directly using the open Movesense Python SDK
- Maximum fidelity (raw quaternions, full sample rate) but loses the custom multi-sensor sync, increases pairing complexity, likely *worse* latency than the optimized hub
- Only worth it if we hit a real wall in the other paths

### Why Path A first

- **Validates the expensive question first:** "does SOMI-1 data shape make RALF more alive in rehearsal?" — not "can we replace MIDI with OSC?"
- **Lowest level of effort** — testable tonight; no rebuilding of pairing/auth/BLE/firmware
- **Path B's work isn't wasted later** — the OSC schema designed in Path A is reusable when/if the editor fork happens
- **Fits existing adapter pattern** — sister to `adapters/imu/` and `adapters/mediapipe/`, declared via `manifest.json`, no runtime changes needed

## What was ruled out and why

- **Notch sensors** (separate research thread): Android-only SDK, cloud-gated license keys, dormant since ~2020, oriented toward capture/replay rather than realtime streaming, custom hardware not bypassable. Possible but high-effort, and largely redundant with what MediaPipe already delivers in joint space. **Parked.**

## Open design questions (to be resolved in `/workflow:plan`)

1. **OSC schema:** what address pattern? Probably `/somi/<sensor_id>/<param>` matching the existing adapter conventions, but the exact param names need to mirror SOMI-1's model (`action`, `pitch`, `roll`, `yaw`, `accel_x`, `accel_y`, `accel_z`).
2. **Resolution handling:** 7-bit vs 14-bit CC — start with 7-bit, surface 14-bit for the params SOMI-1 supports it on, normalize to `0.0–1.0` floats on the OSC side.
3. **Sensor identity:** how do we surface "which sensor is which limb" to the runtime? SOMI-1 channels them on different MIDI channels; we map MIDI channel → sensor_id in the adapter.
4. **Manifest declarations:** what `qualities` does the SOMI-1 source claim? Likely a different list than the raw IMU adapter — closer to `["inclination", "acceleration", "trigger"]`.
5. **Lifecycle:** does the adapter need to handle hub disconnect/reconnect gracefully? (Yes — performers will move out of range.)

## Next step

Run `/workflow:plan` against this document to produce a detailed implementation plan, then `/workflow:issues` to decompose into ordered tickets in `meninoebom/ralf-adapters`.

## References

- [SOMI-1 product page](https://instrumentsofthings.com/products/somi-1)
- [SOMI-1 Editor — open source](https://github.com/instrumentsofthings/SOMI-1-Editor) (C++/JUCE, GPL-3)
- [Movesense launch announcement confirming SOMI-1 sensor lineage](https://www.movesense.com/news/2021/09/instruments-of-things-launches-somi-1-the-sound-of-me/)
- [Movesense Python real-time IMU script](https://www.movesense.com/news/2024/10/python-script-for-collecting-real-time-ecg-and-imu-data/)
- [Hackster.io overview](https://www.hackster.io/news/instruments-of-things-somi-1-bluetooth-wearables-turn-your-movements-into-music-51d4e08b607c)
