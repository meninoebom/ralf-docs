# Artist Materials

Reusable copy for applications, websites, press kits, and introductions. Last updated March 2026.

## Bio (150 words)

I'm an engineer, musician, and dancer. I'm somewhat obsessed with a particular phenomenon: what happens when someone plays records through a sound system for a large group of people and it becomes a responsive, interactive, self-organizing artistic expression. I think that phenomenon is a piece with what happens in a capoeira roda, a bombazo, an assam. These are forms of collective, embodied, real-time musical coordination, and I believe they represent a kind of human intelligence we've mostly failed to recognize as such.

My current project, Ralf, is infrastructure for creating real-time dialogue between movement and music. It's input agnostic (a webcam, a phone, a pressure mat, a LiDAR sensor) and output agnostic (Ableton, a browser synth, SuperCollider, anything). The dancer influences the music, the music shapes the dancer, and neither is in full control. I'm also writing a book, Technologies of Remembering, exploring why these embodied practices matter now more than ever.

## Project Description (500 words)

Ralf is infrastructure for creating real-time, mutually transformative dialogue between movement and music. A dancer moves. The system reads their movement as meaningful qualities: velocity, jerkiness, spatial extent, contraction, coherence between limbs. Those qualities feed into a compositional layer where a technologist has configured readings, thresholds, and intents. When conditions are met, the system sends messages to a music engine that shapes the sound. The dancer hears the response and the loop tightens. We call this the Listening Loop.

The architecture is deliberately agnostic at both ends. Any motion capture input (webcam, phone accelerometers, LiDAR, pressure mats, wristbands) feeds through an adapter that translates raw data into a universal vocabulary of movement qualities. Output goes through translators to any sound engine (Ableton, Tone.js, SuperCollider, anything that speaks OSC). The runtime in the middle evaluates 11 compositional primitives and uses weighted probability to select responses, so the system influences rather than dictates. The dancer shapes the music, the music shapes the dancer, and neither is in full control.

The core system is built and working end to end. The runtime implements 11 compositional primitives (sense, recognize, gate, accumulate, combine, draw, act, smooth, delay, map, latch) with 90+ passing tests. I've built a Performance Console for real-time scene composition, a crowd mode that computes relational qualities between multiple dancers using phone accelerometers, and trajectory detection that distinguishes building energy from releasing it. The architecture has been reviewed by experts in motion capture, interaction design, and signal processing. What I have not yet done is the thing that matters most: sustained rehearsal with dancers and a real performance.

## Working Style (150 words)

I'm a builder. I work best when I have a clear target and the freedom to figure out how to get there. Over the past year I've maintained a weekly rhythm on RALF. Technical sprints during the week, integration testing on weekends, writing and reflection when I hit a wall. I use a lightweight documentation system that keeps my future self oriented when I pick something back up after a break.

What keeps me accountable is having something to show. Demos, even rough ones, create their own momentum. I'd plan to set monthly milestones: first rehearsal with a dancer, first rough-cut performance, first public showing, first documentation piece.

## Application History

### Gray Area Cultural Incubator (March 2026)

Track: Generative Media. Full application drafts preserved in Claude memory at `gray_area_application.md` (includes motivation essay and program-specific answers).
