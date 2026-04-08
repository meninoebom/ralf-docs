# Composing a Score for Ralf

## The Idea in Brief

A "score" for an interactive dance piece needs to answer two questions: *what musical materials are available?* and *how does the dancer's body navigate through them?* The approach I've been developing treats a score as a grid of materials organized by function (rhythm, harmony, texture, melody...) crossed with sections of a piece (intro, build, peak, release...) — plus a set of rules that map movement qualities to mixing decisions. The composer fills the grid. The interaction designer writes the rules. The dancer brings it to life.

---

## The Grid: Your Palette of Materials

A piece needs different *kinds* of sound — not just "track 1, track 2" but functional roles. A starting vocabulary might be:

| Category | What It Does |
|---|---|
| **texture** | Ambient bed, atmosphere — sounds good on its own for a full minute |
| **harmonic_bed** | Sustained harmony, pads, drones |
| **bass** | Low-end foundation |
| **foundation** | Minimal rhythmic skeleton (kick + hat) |
| **groove** | Full rhythmic energy (drums, percussion) |
| **hook** | Melodic feature — the part you'd hum |
| **accent** | Sparse punctuation, one-shots, FX |

These aren't fixed — a piece might only need four of these, or might introduce new ones (vocal, field_recording, noise). They're a starting point for thinking about what functional roles the music needs.

Now cross that with **sections** — the emotional arc of the piece:

```
              intro     verse     chorus    bridge    outro
  texture    [  ◆  ]   [  ◆  ]   [  ◆  ]   [  ◆  ]   [  ◆  ]
  harmonic   [     ]   [  ◆  ]   [  ◆  ]   [  ◆  ]   [     ]
  bass       [     ]   [  ◆  ]   [  ◆  ]   [     ]   [  ◆  ]
  foundation [     ]   [  ◆  ]   [  ◆  ]   [     ]   [  ◆  ]
  groove     [     ]   [     ]   [  ◆  ]   [     ]   [  ◆  ]
  hook       [     ]   [     ]   [  ◆  ]   [     ]   [     ]
  accent     [     ]   [     ]   [  ◆  ]   [  ◆  ]   [     ]
```

Each ◆ is one or more loops/clips that fill that role for that section. Not every cell needs content — a sparse grid makes for a more minimal piece. The grid is the **composer's domain**: what does each layer sound like, and what's available in each section?

In a DAW like Ableton, this maps naturally to **groups of tracks** (one group per category) with **scenes or clip slots** (one per section).

## The Arc: Dramatic Shape Over Time

A piece isn't just reactive — it has a journey. The **arc** is a sequence of phases, each with a duration range and a set of categories it unlocks:

```
  AWAIT ──▶ EMERGE ──▶ BUILD ──▶ PEAK ──▶ BREAKDOWN ──▶ RESOLVE
  (silent)   texture    +bass     +groove    texture      +bass
             only       +found.   +hook      +harmony     +groove
                        +harmony  +accent                 +hook
```

- **AWAIT** — silence until sustained movement is detected (the piece begins when the dancer begins)
- **Phases progress over time**, but duration is modulated by engagement — more movement extends a phase, stillness can compress it
- **Early triggers** are possible — e.g., sustained stillness during PEAK can jump to BREAKDOWN

The arc means the same movement produces different results at different points in the piece. Early on, flowing movement might only affect texture. At the peak, it shapes everything.

## Readings: Interpreting the Body

Raw movement qualities (velocity, symmetry, coherence, jerkiness) are combined into **readings** — named interpretations:

- **"flowing"** = high coherence + moderate velocity + low jerkiness
- **"agitated"** = high jerkiness + high velocity
- **"still"** = very low velocity for a sustained period
- **"expansive"** = high expansion + moderate velocity

Each reading produces a value from 0 to 1 and an active/inactive state (based on threshold gates). You can define as many readings as the piece needs.

## Mappings: Body → Mix

Mappings connect readings to the audio mix — this is the **taste layer**, where the artistic decisions live:

```
  "flowing" active at 0.8  →  harmonic_bed: loud, texture: moderate, foundation: quiet
  "agitated" active at 0.6 →  groove: loud, bass: loud, accent: present
  "still" active           →  everything fades except texture
```

Multiple readings can be active simultaneously — their contributions blend. The result is a continuously shifting mix driven by the body.

## Triggers: Dramatic Punctuation

Triggers watch for specific moments — state changes, sustained conditions — and fire discrete actions:

- "Dancer has been still for 3 seconds → mute all drums (0.5s fade)"
- "Dancer exits stillness → restore full mix"
- "Agitation reading crosses threshold → fire a one-shot percussion hit"

These create surprise, contrast, and dramatic weight that continuous mapping alone can't achieve.

## How This Maps to a DAW Workflow

Imagine building a piece in Ableton:

1. **Organize tracks by category** — a group for texture, a group for groove, etc.
2. **Fill clip slots by section** — intro clips, verse clips, chorus clips in each group
3. **A small Max for Live device** receives messages from Ralf over OSC:
   - `set/` messages smoothly adjust track volumes, filter cutoffs, send levels
   - `trigger/` messages launch clips, fire one-shots, mute/unmute tracks
4. **The scene configuration** (in Ralf's console) defines what readings exist, what triggers exist, and what actions they map to — referencing the category names from the Ableton session

The composer doesn't need to know how MediaPipe works. The interaction designer doesn't need to know Ableton's routing. The dancer doesn't need to know either. Each person works in their own domain.

## The Roles

| Role | Domain | Tool |
|---|---|---|
| **Composer** | Musical materials, arc design, sound palette | Ableton / DAW |
| **Interaction Designer** | Readings, mappings, triggers, scene config | Ralf Console |
| **Dancer** | Movement, performance, embodied expression | Their body |

These roles can overlap — a composer might also design interactions, a dancer might tweak mappings in rehearsal. But the system is designed so they *can* be separate, which makes collaboration possible without everyone needing to understand everything.
