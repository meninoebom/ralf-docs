
**Why:** Applying to Gray Area Cultural Incubator, Generative Media track. Application drafted March 2026.

**How to apply:** Reference these drafts when working on grant applications, artist statements, or bios. The bio and project description have been carefully tuned to Brandon's voice and accurate framing of Ralf.

---

## Short bio (150 words)

I'm an engineer, musician, and dancer. I'm somewhat obsessed with a particular phenomenon: what happens when someone plays records through a sound system for a large group of people and it becomes a responsive, interactive, self-organizing artistic expression. I think that phenomenon is a piece with what happens in a capoeira roda, a bombazo, an assam. These are forms of collective, embodied, real-time musical coordination, and I believe they represent a kind of human intelligence we've mostly failed to recognize as such.

My current project, Ralf, is infrastructure for creating real-time dialogue between movement and music. It's input agnostic (a webcam, a phone, a pressure mat, a LiDAR sensor) and output agnostic (Ableton, a browser synth, SuperCollider, anything). The dancer influences the music, the music shapes the dancer, and neither is in full control. I'm also writing a book, Technologies of Remembering, exploring why these embodied practices matter now more than ever.

## Motivation / what I hope to gain (250 words)

I've been building Ralf largely in isolation for the past year, and while the technical system is working, the hardest remaining questions aren't engineering problems. They're questions about how a performer actually experiences this thing in rehearsal and on stage, how audiences receive it, and how to talk about work that sits between so many disciplines without flattening it into something it's not.

Gray Area feels like the right place to wrestle with those questions because the community seems to genuinely hold space for work that's technically rigorous and culturally grounded at the same time. I'm not building a research prototype or an art installation exactly. Ralf is a performance instrument, and I need to develop it in conversation with other practitioners who are navigating similar tensions between building and performing, between the technical and the embodied.

What I hope to gain is honest feedback from people whose instincts I trust, access to performance contexts where I can test Ralf with real dancers and real audiences, and the kind of creative pressure that comes from being around people who are actually making things. I also think my work on relational musicality and the theoretical framework underneath it could be useful to other artists in the cohort who are thinking about embodiment, collective experience, or the politics of whose intelligence gets recognized as intelligence. I'm looking for collaborators and co-conspirators as much as resources.

## Project description (500 words)

Ralf is infrastructure for creating real-time, mutually transformative dialogue between movement and music. A dancer moves. The system reads their movement as meaningful qualities: velocity, jerkiness, spatial extent, contraction, coherence between limbs. Those qualities feed into a compositional layer where a technologist has configured readings, thresholds, and intents. When conditions are met, the system sends messages to a music engine that shapes the sound. The dancer hears the response and the loop tightens. We call this the Listening Loop.

The architecture is deliberately agnostic at both ends. Any motion capture input (webcam, phone accelerometers, LiDAR, pressure mats, wristbands) feeds through an adapter that translates raw data into a universal vocabulary of movement qualities. Output goes through translators to any sound engine (Ableton, Tone.js, SuperCollider, anything that speaks OSC). The runtime in the middle evaluates 11 compositional primitives and uses weighted probability to select responses, so the system influences rather than dictates. The dancer shapes the music, the music shapes the dancer, and neither is in full control.

**(1) Current stage:** The core system is built and working end to end. The runtime implements 11 compositional primitives (sense, recognize, gate, accumulate, combine, draw, act, smooth, delay, map, latch) with 90+ passing tests. I've built a Performance Console for real-time scene composition, a crowd mode that computes relational qualities between multiple dancers using phone accelerometers, and trajectory detection that distinguishes building energy from releasing it. The architecture has been reviewed by experts in motion capture, interaction design, and signal processing. What I have not yet done is the thing that matters most: sustained rehearsal with dancers and a real performance.

**(2) What I need to accomplish during the program:** Three things. First, I need to move from building to rehearsing. The system works technically but I haven't yet developed the compositional and performative intuitions that come from hours of use with real dancers. I need rehearsal time and willing collaborators. Second, I want to develop at least one complete performance piece that demonstrates what the Listening Loop feels like from the audience's perspective. Third, I need to produce documentation, a demo video, writing, maybe a talk, that communicates why this work matters in terms that reach beyond the technical community.

**(3) What resources or support would be most helpful:** Studio space with room to move is the biggest need. Ralf requires nothing more than a laptop and a webcam to get started, so the hardware requirements are minimal, but the physical space to dance, experiment, and rehearse is essential. Connections to dancers, particularly people with backgrounds in improvisational or participatory forms, would be transformative. I'd also benefit from mentorship around the performance and presentation side: how to frame this work for live audiences, how to document it well, and how to navigate the grant landscape for projects that don't fit neatly into "tech art" or "dance" or "music" categories. The theoretical framework (relational musicality, technologies of remembering) is developed, the technical system is built. What I need now is the embodied, social, performative context to bring them together.

## Working style / maintaining momentum (150 words)

I'm a builder. I work best when I have a clear target and the freedom to figure out how to get there. Over the past year I've maintained a weekly rhythm on RALF. Technical sprints during the week, integration testing on weekends, writing and reflection when I hit a wall. I use a lightweight documentation system that keeps my future self oriented when I pick something back up after a break.

What keeps me accountable is having something to show. Demos, even rough ones, create their own momentum. I'd plan to set monthly milestones: first rehearsal with a dancer, first rough-cut performance, first public showing, first documentation piece. The four-month timeline actually maps well to what I need: enough time to develop real fluency with the system in a performative context, not so much time that I lose urgency. External deadlines and a cohort of people doing ambitious work would be ideal structure for me.

## Program track

Generative Media
