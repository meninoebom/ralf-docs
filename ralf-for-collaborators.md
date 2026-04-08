# Ralf

## What Is This?

Ralf is a system I've been building that lets a dancer's body shape music in real time. A camera watches the dancer, extracts movement qualities — how fast, how smooth, how symmetric, how jerky — and an engine interprets those qualities to make decisions about sound. Not just "move your hand to change volume" — more like "when the movement becomes flowing and coherent, the harmonic layers open up; when it gets agitated and sharp, the rhythm intensifies."

The dancer doesn't control individual parameters. They move, and the system *reads* the movement — the way a musician reads the room. The mapping between body and sound is configured per piece, so the same movement can mean different things in different compositions.

The sound side is completely open. Right now there's a built-in browser synth (Tone.js), but the engine speaks OSC — so it can talk to Ableton, Max/MSP, SuperCollider, anything. The composer works in their own environment. Ralf just sends messages like "trigger this" or "set this parameter to 0.7" based on what the dancer is doing.

## How It Works

```
                            ┌─────────────────────────────────────┐
                            │        PERFORMANCE CONSOLE          │
                            │     scene editor + live monitor     │
                            └──────────────┬──────────────────────┘
                                           │ edit/monitor
                                           │
  ┌──────────┐    ┌────────────┐    ┌──────┴──────┐    ┌─────────────────┐
  │          │    │            │    │             │    │                 │
  │  DANCER  │───▶│  CAPTURE   │───▶│   ENGINE    │───▶│   TRANSLATOR    │
  │          │    │            │    │   (Ralf)    │    │                 │
  └──────────┘    └────────────┘    └─────────────┘    └─────────────────┘
                                                              │
   body motion     camera/phone      reads movement,          │
                   extracts:         makes decisions:     ┌───┴───────────┐
                   · velocity        · is it flowing?     │  ANY AUDIO    │
                   · symmetry        · is it agitated?    │  ENGINE       │
                   · coherence       · did it just stop?  │               │
                   · jerkiness       ──────────────▶      │  · Ableton    │
                   · expansion       trigger/unmute_track  │  · Max/MSP    │
                   · etc.            set/filter_cutoff     │  · Tone.js    │
                                                          │  · SuperCol.  │
                                                          └───────────────┘
```

### The Three Layers

**Capture** — A camera (MediaPipe) or phone (IMU sensors) watches the dancer and produces continuous streams of movement qualities: velocity, symmetry, coherence, jerkiness, expansion. These are numbers between 0 and 1, updated ~30 times per second.

**Engine** — The core of Ralf. It takes raw qualities and interprets them through a *scene* — a configuration file that defines:
- **Readings**: named combinations of qualities with thresholds ("flowing" = high coherence + moderate velocity + low jerkiness)
- **Intents**: what should happen when a reading activates ("when flowing, open the harmonic layers")
- Each intent picks from a pool of possible actions, so the system isn't perfectly predictable

**Translator** — A separate process that receives Ralf's decisions as simple messages and converts them into commands for a specific audio engine. Two message types:
- `trigger/` — do this now (unmute a track, fire a one-shot)
- `set/` — smoothly move to this value (filter cutoff, send level, volume)

The translator is the only part that needs to know about the audio engine. Everything upstream is engine-agnostic.

### Multiple Dancers

Ralf supports crowd mode — multiple dancers tracked simultaneously via phone IMU sensors. The engine automatically computes relational qualities between dancers: synchrony, contrast, aggregate energy. A piece can respond to how dancers relate to each other, not just individual movement.

### The Console

A browser-based tool for composing and monitoring:
- **Scene Editor** — build and tweak the reading/intent configuration visually
- **Signal Strip** — live display of all qualities, readings, and intents as they fire
- **Scene Library** — save, load, and switch between different configurations
