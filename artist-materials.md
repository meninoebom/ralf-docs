# Artist Materials

Reusable copy for applications, websites, press kits, and introductions. Last updated May 2026.

> See also [`ralf-research-prospectus.md`](./ralf-research-prospectus.md) for the full strategic articulation of the work, the three-dimensions framework, and the three-tier roadmap. The sections below are the short-form, reusable versions of that material.

## Bio (150 words)

I'm an engineer, musician, and dancer. I'm somewhat obsessed with a particular phenomenon: what happens when someone plays records through a sound system for a large group of people and it becomes a responsive, interactive, self-organizing artistic expression. I think that phenomenon is a piece with what happens in a capoeira roda, a bombazo, an assam. These are forms of collective, embodied, real-time musical coordination, and I believe they represent a kind of human intelligence we've mostly failed to recognize as such.

My current project, Ralf, is infrastructure for creating real-time dialogue between movement and music. It's input agnostic (a webcam, a phone, a pressure mat, a LiDAR sensor) and output agnostic (Ableton, a browser synth, SuperCollider, anything). The dancer influences the music, the music shapes the dancer, and neither is in full control. I'm also writing a book, *Relational Musicality: A Technology of Remembering*, exploring why these embodied practices matter now more than ever.

## Project Description (500 words)

Ralf is infrastructure for creating real-time, mutually transformative dialogue between movement and music. A dancer moves. The system reads their movement as meaningful qualities: velocity, jerkiness, spatial extent, contraction, coherence between limbs. Those qualities feed into a compositional layer where a technologist has configured readings, thresholds, and intents. When conditions are met, the system sends messages to a music engine that shapes the sound. The dancer hears the response and the loop tightens. We call this the Listening Loop.

The architecture is deliberately agnostic at both ends. Any motion capture input (webcam, phone accelerometers, LiDAR, pressure mats, wristbands) feeds through an adapter that translates raw data into a universal vocabulary of movement qualities. Output goes through translators to any sound engine (Ableton, Tone.js, SuperCollider, anything that speaks OSC). The runtime in the middle evaluates 11 compositional primitives and uses weighted probability to select responses, so the system influences rather than dictates. The dancer shapes the music, the music shapes the dancer, and neither is in full control.

The core system is built and working end to end. The runtime implements 11 compositional primitives (sense, recognize, gate, accumulate, combine, draw, act, smooth, delay, map, latch) with 90+ passing tests. I've built a Performance Console for real-time scene composition, a crowd mode that computes relational qualities between multiple dancers using phone accelerometers, and trajectory detection that distinguishes building energy from releasing it. The architecture has been reviewed by experts in motion capture, interaction design, and signal processing. What I have not yet done is the thing that matters most: sustained rehearsal with dancers and a real performance.

## Working Style (150 words)

I'm a builder. I work best when I have a clear target and the freedom to figure out how to get there. Over the past year I've maintained a weekly rhythm on RALF. Technical sprints during the week, integration testing on weekends, writing and reflection when I hit a wall. I use a lightweight documentation system that keeps my future self oriented when I pick something back up after a break.

What keeps me accountable is having something to show. Demos, even rough ones, create their own momentum. I'd plan to set monthly milestones: first rehearsal with a dancer, first rough-cut performance, first public showing, first documentation piece.

## Research Vision (115 words)

I work as a practitioner-theorist. My primary text is not books. It's the roda, the dance floor, the jazz combo, the bombazo. These forms are practical philosophies of human relating — carriers of insight, expressed in body and sound, about how people cooperate under pressure, metabolize conflict, keep their systems alive over time, and a great deal else I'm still drawing out. Most public intellectual work flattens what it cannot translate into text. My work translates carefully, and it also builds. RALF is the system I'm using to bring what I find in these forms into new performance contexts and new collaborations. Thinking and making inform each other. Neither is the source.

## Three Patterns: A Case-Study Reading (175 words)

When I read these forms as practical philosophies of human relating, more than three patterns appear. As a case study in what this kind of reading yields, here are three I keep returning to.

First, cooperation over domination — the system penalizes dominators and rewards mutual attentiveness, which frees up energy for creativity rather than for conflict.

Second, conflict resolved without a winner — disagreement gets metabolized into musical content rather than into combat. The energy of conflict becomes the next move.

Third, continuous self-correction at multiple timescales — inside a tune and across decades, inside a game and across generations.

Versions of these same patterns also organize liberal institutions at their best. That parallel is, for me, corroboration — evidence that what these forms carry is worth taking seriously as practical wisdom about how humans can relate, not just as cultural materials to admire. The list isn't exhaustive. More patterns are there to be found. The method — taking these forms seriously as practical philosophy — is what matters.

## Three-Tier Roadmap (175 words)

RALF supports three kinds of work. They reinforce each other.

**Tier 1 — Documentation and analysis with practitioners in established traditions.** Capoeira mestres, bomba *bomberas*, house dancers, rumberos, jazz musicians. Collaborative pieces and sessions that make visible what is usually tacit in the form. The practitioners co-author the work; they are partners, not subjects.

**Tier 2 — Experimental collaborations across traditions.** Cross-tradition pieces where RALF is the connective tissue. A capoeirista and a house dancer with the system as the third collaborator. A jazz composer writing for a bomba dancer who has not seen the piece in advance. New compositional ideas, new audiences. Most of the formally innovative work happens here.

**Tier 3 — North Star.** A piece commissioned by Alonzo King LINES Ballet — or a company of similar caliber — with a composer in the range of Jason Moran, running through RALF. Dancers in real-time dialogue with the score, inside a constraint grammar the composer has set. The argument made inside the most formally trained Western performance institutions. A stretch target, concrete enough to orient by.

## One-Liners

**Practitioner-theorist.** I work as a practitioner-theorist: my primary text is the roda, the dance floor, the jazz combo — and the apparatus I use to read those texts includes my own embodied training in them.

**The system as argument.** RALF is designed to operate by some of the principles I keep finding when I read these forms closely — cooperation over domination, conflict-as-content, continuous self-correction among them. Using it isn't a demonstration of those principles. It's the principles in operation.

**What endures.** The technology will keep shifting. What does not shift is the thing I'm pointing at: the relationality of moving bodies and music, the way these forms carry political and social philosophy, the way they've long been laboratories for what good human relating can actually look like.

**The work in one breath.** Afro-diasporic musicking traditions can be read as practical philosophies of human relating. Among the patterns I keep finding: cooperation rewarded over domination, conflict turned into content rather than combat, correction built into the system — and more I am still drawing out. RALF is designed to operate by what these forms know. The work is to bring practitioners, composers, and eventually major institutions into collaboration inside this kind of practice.

## Application History

### Gray Area Cultural Incubator (March 2026)

Track: Generative Media. Full application drafts preserved in Claude memory at `gray_area_application.md` (includes motivation essay and program-specific answers).
