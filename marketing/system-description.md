
## Four Layers of the System

### 1. Motion Capture Layer
Captures movement from dancing bodies. Designed to be **agnostic** about capture method — simple webcam + software (MediaPipe), or advanced tech like LIDAR. The point is: any way of sensing a moving body can feed RALF.

### 2. Adapter Layer
Translates raw motion capture data into a format the RALF brain can understand. Sits between the camera and the runtime. Repackages data for the brain.

### 3. Runtime ("The Brain")
Where **interactive design** happens — the process of deciding how movement input gets interpreted and mapped to musical output. This is where the choreographer/composer/designer makes creative decisions.

The brain uses a **Reading → Intent → Action** framework:
- **Readings**: Qualities of movement being watched at all times (jerkiness, stillness, smoothness, speed, distance between hands, openness/closedness of body)
- **Intents**: Meaning assigned to movement qualities ("when jerkiness goes over 70%, the dancer wants to raise the energy")
- **Actions**: Musical responses triggered by intents (unmute a track, add echo to an instrument, trigger the next section, add a wobble to a synth)

These are configured into **scenes** — an emergent universe the dancer inhabits and plays in.

### 4. Sound Output
Speakers through which the dancer-composed music emerges.

## The Creative Workflow (Collaborative Use Case)

Example: Working with Alonzo King LINES Ballet and a composer on a piece about grief.

1. **Workshop movements** with dancers
2. **Composer builds a reconfigurable library of sound** — loops, motifs, riffs, layers for different instruments and emotional sections (not a single fixed piece, but material that can be assembled and triggered)
3. **Wire movement to music** in the RALF brain — "if arm going up = expansiveness, link to ocean sounds (70% weight) or wind sounds (30% weight)"
4. **Dancers perform in the scene** — their movement triggers, affects, and influences the musical material in real time
5. **The score responds** — and the dancer responds to the score — creating mutually transformative dialogue

The composer's role: create material like scoring a film — loops, motifs, each maybe a few bars long, using a limited palette of instruments/synthesized sounds applied differently across sections. A digital audio workstation (DAW) is the primary tool.

## The Open/Accessible Use Case

No choreography needed. Take a piece of music, computationally break it into pieces, and let anyone walk up to a webcam and explore that music through movement — "as if it was a kind of space." You don't need to know anything. Music starts playing when you step in front of the camera.

## What RALF Has Built (as of March 2026)

The **primitives** — the legos/building blocks that allow flexible configuration. Not a finished product, but a toolkit that enables many different applications.

## Beyond Dance: Rehabilitation

The RALF brain is an approach that could be applied to physical rehabilitation:
- Scientific literature supports using rhythm as internal support for walking in neurodegenerative conditions
- RALF could interpret walking as the salient movement, providing rhythmic feedback
- Could target specific movements (arm mobility, gait) and connect them to musical outcomes
- Makes rehabilitation **playful** — patients choose music that inspires their brain and bodies
- Each application is a different scene configuration, not a different system
