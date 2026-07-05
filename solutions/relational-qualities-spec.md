# RALF Relational Quality Layer: Design Spec

**Issue #3, RALF Ultracode Roadmap.** Implementation contract for the multi-dancer quality layer and compositional vocabulary. A future session should be able to build the whole layer from this document without rerunning the design work.

Generated via judge-panel-then-synthesize workflow (4 designs × 4 judges × 1 synthesis, Opus 4.8). Winner: Design 2 (Crowd-first, score 36/40). Ideas from all four designs grafted into the synthesis.

---

## 0. The one decision everything descends from

"Cooperation over domination" is not a vibe to gesture at in scene-author discretion. It is a property the **computation layer** makes structurally easy to honor and structurally hard to violate. The synthesis fuses four ideas, each of which encodes that property at a different layer:

1. **Mean-field reframing** (winner, Design 2): compute one shared group reference signal per tick, measure every dancer against it. O(n), and the group reference literally *is* the "virtual crowd entity" the thesis names.
2. **The monotonicity test** (Design 3): every relational quality must be non-monotonic in any single dancer's signal. If a number rises when one body overpowers the rest, it is a domination reward by construction. This is a *screening rule* for the quality set and a *validator lint rule* for scenes.
3. **Min-reduction for shared-floor qualities** (Design 1): where a quality means "the group is doing X together," reduce by `min`, not `mean`, so one dancer cannot raise it alone.
4. **Remove the negative-correlation clamp** (Design 4): anti-synchrony is content the thesis explicitly values. The sign survives; counterpoint becomes gateable.

---

## 1. Relational Quality Set

All qualities live on the virtual `_crowd` dancer, written each tick in `computeRelational()`. Every one passes the **monotonicity test**: a quality that rises when one dancer overpowers the rest is a domination reward by construction.

Core computational primitive: the **mean field**. Build one group reference signal per tick, then measure each dancer against it.

```
meanVel[t]   = (1/n) Σ_d velHistory[d][t]   // group pulse, 20-frame window
centroid[q]  = (1/n) Σ_d dancer[d].qualities[q]   // group shape, 13 solo qualities
```

| Quality | Range | Reduction | Monotonicity |
|---|---|---|---|
| `cohesion` | **−1..1** | mean-field corr | pass |
| `dissent` | 0..1 | fraction | pass |
| `unison` | 0..1 | 1 − dispersion | pass |
| `fragmentation` | 0..1 | largest-gap | pass |
| `aggregate_energy` | 0..1 | **min** (shared floor) | pass (changed from mean) |
| `energy_spread` | 0..1 | std | pass |
| `convergence` | 0..1 (0.5=steady) | slope of cohesion magnitude | pass |
| `proximity` | 0..1 | mean-field density | pass |
| `lead_strength` | 0..1 | max lag-corr asymmetry | pass |
| `lead_id` | string | argmax | n/a (meta channel, not gateable) |
| `field_intensity` | 0..1 | mean (validator-restricted) | monotonic (named honestly) |

### 1.1 `cohesion` — signed, the spine

How each dancer's velocity correlates with the group pulse, signed, averaged.

```
for each dancer d (with full window):
  corr_d = pearson(velHistory[d], meanVel)   // NO clamp
cohesion = mean(corr_d)                       // −1..1
```

+1 = the group moves as one body. 0 = unrelated. **−1 = the group is anti-phase, which is dialogue in its contrastive mode, not the absence of relationship.** The negative half is the single most important expressive unlock.

**Self-correlation fix:** use leave-one-out field unconditionally (correlate dancer A against `meanVel` computed without A). Cheap, removes self-correlation bias at all n including duet (n=2).

**Monotonicity: pass.** One dancer flailing alone cannot raise group correlation; it actually lowers cohesion.

### 1.2 `dissent` — anti-domination made first-class

```
dissent = count(corr_d < -0.3) / n        // 0..1
```

Fraction of dancers whose correlation with the field is strongly negative. The structural anti-domination signal: how many bodies are pushing back. Input to the `productive-dissent` reading that *rewards* deliberate counterpoint.

**Monotonicity: pass.** A single dominator cannot raise dissent alone.

### 1.3 `unison` — distribution width, not pairwise distance

```
dispersion = mean_over_dancers( ||qualityVec_d − centroid|| ) / sqrt(13)
unison     = 1 − min(1, dispersion)        // 0..1
```

How tightly all dancers cluster in 13-dimensional quality space. O(n), not O(n²). The inverse (`1 − unison`) replaces the old pairwise `contrast` for the "are we apart" question.

### 1.4 `fragmentation` — the group splits into camps

```
project each dancer onto velocity (or centroid's dominant axis)
sort the projections
fragmentation = largest_gap_between_adjacent / total_range     // 0..1
```

Cheap 1-D gap proxy (no k-means in the hot loop). A mean cannot see "two sub-crowds." Fragmentation can.

### 1.5 `aggregate_energy` — shared floor (min), not mean

```
aggregate_energy = min_over_dancers( dancer.qualities.velocity )   // 0..1
```

**Deliberate change from the current `mean`.** The mean is a dominance leak: one thrashing dancer drags the aggregate up even if everyone else is still. The `min` makes "high energy together" require *everyone* to bring energy. Scale-invariant: at any n, it is "the floor the least-energetic body sets."

**`field_intensity` = mean** is kept separately for scenes that genuinely want room-loudness (it is honestly monotonic because it describes the *room*, not a *relationship*). It is validator-restricted so it can never gate a reward action alone.

### 1.6 `energy_spread` — texture at equal mean

```
energy_spread = stddev_over_dancers( velocity ) normalized to 0..1
```

Distinguishes "all medium" from "half still, half wild" at the same `field_intensity`.

### 1.7 `convergence` — the trend toward/away from relation

```
maintain ring buffer of last W values of |cohesion|   (relatedness magnitude)
slope = windowed_linear_regression_slope(buffer)        // REUSE existing trajectory slope code
convergence = 0.5 + clamp(slope * k, -0.5, 0.5)         // 0..1, 0.5 = steady, >0.5 = coming together
```

Reuses the existing reading-`trajectory` slope regression. Do not write a second one.

### 1.8 `proximity` — dance-floor geometry

```
proximity = mean_over_dancers( neighbors_within(d, r) / (n − 1) )   // 0..1
```

Requires position data. **Degrades gracefully:** when position is unavailable (current IMU-only state), fall back to heading closeness `1 − mean|heading_i − heading_j|` over the mean field, documented as a proxy. When MoveNet pelvis position is wired in, swap to normalized inverse Euclidean distance; the interface and range are identical, so scenes do not change.

### 1.9 `lead_strength` + `lead_id` — leadership as circulation

```
for each dancer d:
  best_lag_corr = max over lag in [−5..+5] of pearson(velHistory[d], shift(meanVel, lag))
  lead_score_d  = peak correlation at negative lag (field follows d) × consistency
lead_strength = max_d(lead_score_d)        // 0..1
lead_id       = argmax_d(lead_score_d)     // string, parallel meta channel
```

`lead_strength` measures *that there is a clear lead structure right now*, never crowning a permanent leader. `lead_id` is expected to rotate. Leadership is a circulation property, not a role.

**`lead_id` type problem.** Identity is not a number. `lead_strength` is the gateable quality; `lead_id` rides a parallel `_crowd.meta: Record<string, string>` surfaced in the state broadcast for the translator's voice assignment.

**`lead_id` hysteresis:** reassigns only when a new candidate beats the current leader by a margin (0.1) for ≥10 frames. Makes rotation feel musical, not per-frame jitter.

---

## 2. Scaling Algorithm

**Algorithm: mean-field approximation with three tiers.**

### Mean-field core (default, all n)

Replace both O(n²) loops with a single O(n) pass:

- Build `meanVel` + `centroid`: O(n·w) and O(n·q)
- Measure each dancer: O(n·w) for cohesion/dissent; lead search O(n·w·L), L=11
- Distribution shape: O(n·q) for unison/energy_spread; O(n log n) for fragmentation

**Total: O(n·(w+q) + n·w·L + n log n). No n² term.** Both O(n²) loops in the current code are eliminated.

### Tier 2: spatial hash for proximity (when position exists)

Bucket dancers into a grid of cell size `r`. Each dancer checks only its 9 neighboring cells. Expected O(n) for uniform density.

### Tier 3: k-nearest sketch (escape hatch for pairwise texture)

Restrict pairwise computation to each dancer's k spatial neighbors (k≈3), O(n·k). **Duet falls out for free at n=2:** no special-case branch needed.

### Graceful degradation by group size

- **n ≤ 8:** full fidelity, including lag-search and leave-one-out field
- **8 < n ≤ ~40:** drop lag-search; `lead_strength` falls back to lag-0 asymmetry
- **n > 40:** this is a future `computeField()` with genuinely different math. Mean-field already degrades correctly here ethically: no single body can move any group statistic much, which *is* the thesis.

### Dynamic entry/exit

1. Mean-field is naturally robust to n changing: no reindexing, no realloc.
2. **Cold-history newcomers:** exclude dancers with `history.length < w/2` from the field reference and correlation stats. A newcomer cannot spike `dissent` or steal `lead_id` on frame one.
3. **Leavers:** drop from the field the moment they go `stale`.
4. `lead_id` hysteresis prevents spotlight strobing on entry/exit.

---

## 3. Compositional Vocabulary for Relational Intents

The reading → intent → action grammar is reused **unchanged**. Relational qualities become valid keys in `mix`/`gate`. Two additions: a `scope` field and a validator warning.

### 3.1 The `scope` routing extension

```ts
interface ReadingConfig {
  // ...existing...
  scope?: "per_dancer" | "crowd" | "broadcast";   // default "per_dancer"
}
```

- **`per_dancer`** (default): evaluated once per real dancer, exactly as today. Existing scenes untouched.
- **`crowd`**: evaluated once against `_crowd`. Makes explicit what `crowd-demo.json` does implicitly.
- **`broadcast`**: evaluated against `_crowd`, but the firing dancer context is the **outlier the quality identifies** (`lead_id` or max-dissent dancer). This is how "a dancer's personal response to the group" gets expressed.

### 3.2 The anti-domination validator rule

> **A reward-bearing action may not be gated by `field_intensity`, `cohesion` (positive), or `unison` alone. At least one of `aggregate_energy`, `dissent`, `proximity`, `convergence`, or a `below`-threshold on those must co-gate it.**

Those three "alone" qualities are the ones that can rise under domination. Co-gating means the reward only fires when the energy is *shared*. Ships as a **warning, not a hard error**. Lives in `scenes/validator.ts`.

### 3.3 Seven worked intent conditions

| # | Reading | `scope` | Gate | Thesis principle |
|---|---|---|---|---|
| 1 | ensemble-converge | `crowd` | `cohesion above 0.5` + `convergence above 0.55` | dialogue / "lock in around a phrase" |
| 2 | counterpoint-alive | `crowd` | `cohesion below -0.3` | conflict-as-content; the negative half |
| 3 | productive-dissent | `broadcast` | `dissent above 0.3` | anti-domination; conflict attributed to the dissenting body |
| 4 | leader-spotlight | `broadcast` | `lead_strength above 0.6` | "leadership rotated because the music asked for it" |
| 5 | floor-splits | `crowd` | `fragmentation above 0.5` | two sub-crowds; hear the split |
| 6 | shared-lift | `crowd` | `aggregate_energy above 0.6` | min-floor: fires only when everyone brings energy |
| 7 | someone-carrying | `crowd` | `field_intensity above 0.4` + `dissent below 0.1` + `cohesion below 0.2` | the "boring game"; responds by withdrawal |

Two non-negotiable conventions:

- **Chance stays.** Every relational intent resolves through a weighted `pool`. Domination shifts weights; it never seizes the output.
- **Withdrawal over punishment.** Anti-domination readings respond by *removing* reward (thin a layer, drop gain), never by triggering a "you lose" sound.

---

## 4. Anti-Synchrony Handling

**Implementation.** `cohesion` is signed, range **−1..1**. The current code's `Math.max(0, pearson(a, b))` clamp on line 49 of `relational.ts` is **deleted**. Negative correlation is preserved bit-for-bit.

**What it means artistically.** Two bodies in deliberate opposition are *in dialogue*, in its contrastive mode. Clamping anti-phase to 0 says opposition is the absence of relationship. It is the opposite: it is relationship as counterpoint, call-and-response, "conflict channeled into expression rather than into combat."

**How composers use it:**

1. **Gate the negative half directly.** `{ "cohesion": { "below": -0.3 } }` fires on deliberate counterpoint. No new grammar.
2. **`dissent` as the population-level anti-phase signal.** When you want "how *many* are pushing back," gate `dissent`.

**Important:** relational values bypass `AdaptiveRange` entirely (they are written straight onto `_crowd.qualities`, not sensed). Signed values flow through `combine`/`gate` as raw numbers; the `above`/`below` comparisons handle negatives correctly because they are plain numeric comparisons. This is *not* because AdaptiveRange normalizes them — it does not touch the relational path.

---

## 5. Migration Path from Current `relational.ts`

Five independently shippable, backwards-compatible PRs. Only Step 1 changes any existing behavior; Step 5's `aggregate_energy` redefinition is the one deliberate, named scene-behavior change.

**Step 1 — Remove the clamp (isolated, first PR).**
- `relational.ts:49`: delete `Math.max(0, ...)`. In the mean-field rewrite: no clamp on `corr_d`.
- Add test: anti-correlated histories now produce negative `cohesion`.
- Update `runtime/CLAUDE.md` "Relational qualities" note.
- Keep deprecated `synchrony = max(0, cohesion)` alias on `_crowd` for one release so `crowd-demo.json` keeps working unchanged.

**Step 2 — Rewrite to mean-field core.**
- Replace both O(n²) upper-triangle loops with single-pass mean-field computation.
- Add `cohesion`, `dissent`, `unison`, `fragmentation`, `energy_spread`, `field_intensity` to `RelationalQualities` and the `QualityName` union in `types.ts`.
- Implement leave-one-out field unconditionally.
- Redefine the `<2 dancers` early return: cohesion 0, dissent 0, unison 1, convergence 0.5.

**Step 3 — Add `convergence`.**
- Maintain `|cohesion|` ring buffer on `_crowd`. Reuse existing reading-`trajectory` slope regression.

**Step 4 — Add `lead_strength` + `lead_id` + `scope` routing.**
- `lead_strength` via lag-search; `lead_id` on new `_crowd.meta: Record<string,string>`.
- Add `scope?: "per_dancer" | "crowd" | "broadcast"` to `ReadingConfig`; default `per_dancer`.
- Branch in reading-eval loop in `runtime.ts` (~lines 148–185) for `crowd`/`broadcast`. `broadcast` reroutes firing dancer context to `lead_id`/max-dissent dancer; this touches `resolveIntents`/`fireIntent` edge-state keying and is the highest-risk piece, so it ships last.
- Add `lead_id` hysteresis (0.1 margin, ≥10 frames).

**Step 5 — Redefine `aggregate_energy` to `min`; expose `mean` as `field_intensity`.**
- `aggregate_energy = min`. Add `field_intensity = mean`.
- **This changes `crowd-demo.json` behavior:** the `crowd-energy` reading fires less often (requires everyone, not someone). Named, not a surprise.
- Add the anti-domination validator warning to `scenes/validator.ts`.
- Delete the `synchrony` alias from Step 1. Migrate `crowd-demo.json` `synchrony` → `cohesion`.

**File map:**
- `runtime/src/engine/relational.ts` — rewrite to mean-field; delete clamp; new qualities; aggregate→min
- `runtime/src/types.ts` — `QualityName` additions; `ReadingConfig.scope`; `SceneSettings.relational_window`; `_crowd.meta` shape
- `runtime/src/engine/runtime.ts` — tick integration (~148–185), scope routing, join/leave window guard, lead hysteresis, `_crowd.meta` write
- `runtime/src/scenes/validator.ts` — anti-domination warning
- `runtime/scenes/crowd-demo.json` — migrate
- `runtime/scenes/ensemble-dialogue.json` — new (§6)
- `runtime/CLAUDE.md` — update relational note

**Optional:** add `relational_window?: number` to `SceneSettings` (default 20) and thread into `computeRelational`'s existing `windowSize` param.

---

## 6. Worked Example Scene

`ensemble-dialogue.json` — four dancers. Rewards finding each other (cohesion + convergence co-gated), treats dissent as content attributed to the dissenting body, gives the rotating leader a stochastic spotlight no one can monopolize, hears the floor split, and *withdraws* when one body carries the room.

```json
{
  "version": 2,
  "name": "ensemble-dialogue",
  "settings": {
    "hysteresis_band": 0.06,
    "staleness_frames": 90,
    "adaptive_range_decay": 0.002,
    "relational_window": 30
  },
  "dancers": [
    { "id": "d1", "adapter": "imu" },
    { "id": "d2", "adapter": "imu" },
    { "id": "d3", "adapter": "imu" },
    { "id": "d4", "adapter": "imu" },
    { "id": "_crowd" }
  ],
  "readings": [
    {
      "id": "ensemble-converge",
      "scope": "crowd",
      "mix": { "cohesion": 0.6, "convergence": 0.4 },
      "gate": { "cohesion": { "above": 0.5 }, "convergence": { "above": 0.55 } },
      "trajectory": { "window": 12, "above": 0.05 },
      "intents": [{ "intent": "bloom_harmony", "mode": "continuous" }],
      "on_exit": ["dissolve_harmony"]
    },
    {
      "id": "counterpoint-alive",
      "scope": "crowd",
      "mix": { "cohesion": 1.0 },
      "gate": { "cohesion": { "below": -0.3 } },
      "intents": ["answer_phrase"]
    },
    {
      "id": "productive-dissent",
      "scope": "broadcast",
      "mix": { "dissent": 1.0 },
      "gate": { "dissent": { "above": 0.3 } },
      "intents": [{ "intent": "open_countervoice", "mode": "edge" }]
    },
    {
      "id": "leader-spotlight",
      "scope": "broadcast",
      "mix": { "lead_strength": 1.0 },
      "gate": { "lead_strength": { "above": 0.6 } },
      "intents": [{ "intent": "spotlight", "mode": "continuous" }]
    },
    {
      "id": "floor-splits",
      "scope": "crowd",
      "mix": { "fragmentation": 1.0 },
      "gate": { "fragmentation": { "above": 0.5 } },
      "intents": ["widen_stereo"]
    },
    {
      "id": "shared-lift",
      "scope": "crowd",
      "mix": { "aggregate_energy": 1.0 },
      "gate": { "aggregate_energy": { "above": 0.6 } },
      "intents": ["lift_section"]
    },
    {
      "id": "someone-carrying",
      "scope": "crowd",
      "mix": { "field_intensity": 0.5 },
      "gate": {
        "field_intensity": { "above": 0.4 },
        "dissent": { "below": 0.1 },
        "cohesion": { "below": 0.2 }
      },
      "intents": [{ "intent": "withdraw_layer", "mode": "continuous" }],
      "on_exit": ["restore_layer"]
    },
    {
      "id": "field-stills",
      "scope": "crowd",
      "mix": { "aggregate_energy": 1.0 },
      "gate": { "aggregate_energy": { "below": 0.15 } },
      "intents": [{ "intent": "rest", "mode": "continuous" }]
    }
  ],
  "intents": {
    "bloom_harmony":    { "pool": [{ "action": "set/reverb_wet", "weight": 3 }, { "action": "set/chord_richness", "weight": 2 }]},
    "dissolve_harmony": { "pool": [{ "action": "set/chord_richness", "args": { "target": 0 }, "weight": 1 }]},
    "answer_phrase":    { "pool": [{ "action": "trigger/call_response", "args": { "phrase": "up" }, "weight": 6 }, { "action": "trigger/call_response", "args": { "phrase": "turn" }, "weight": 4 }]},
    "open_countervoice":{ "pool": [{ "action": "trigger/unmute_track", "args": { "track": "counter" }, "weight": 3 }, { "action": "trigger/arp_burst", "weight": 2 }]},
    "spotlight":        { "pool": [{ "action": "set/lead_send", "weight": 3 }, { "action": "set/filter_cutoff", "weight": 2 }]},
    "widen_stereo":     { "pool": [{ "action": "set/stereo_width", "weight": 3 }, { "action": "set/delay_pingpong", "weight": 2 }]},
    "lift_section":     { "pool": [{ "action": "trigger/unmute_track", "args": { "track": "bass" }, "weight": 3 }, { "action": "trigger/fire_next_scene", "weight": 1 }]},
    "withdraw_layer":   { "pool": [{ "action": "set/volume", "args": { "track": "perc", "target": 0.3 }, "weight": 3 }, { "action": "trigger/mute_track", "args": { "track": "lead" }, "weight": 1 }]},
    "restore_layer":    { "pool": [{ "action": "set/volume", "args": { "track": "perc", "target": 1.0 }, "weight": 1 }]},
    "rest":             { "pool": [{ "action": "set/volume", "weight": 3 }, { "action": "set/reverb_wet", "weight": 2 }]}
  },
  "translator": { "type": "tonejs", "port": 12000 }
}
```

**How it plays.** As the four find a shared pulse, `cohesion` climbs past 0.5 *and* `convergence` confirms they are actively coming together (not just statically aligned), so `bloom_harmony` rides reverb and chord-richness up; when they scatter, `on_exit` dissolves it. If d3 deliberately works against the group, `cohesion` swings negative (`counterpoint-alive` answers the phrase) and `dissent` crosses 0.3, opening a counter-voice attributed to d3 via broadcast scope. Whoever the field follows lights up via `spotlight`, but the stochastic pool means no one can hold the spotlight as a button, and `lead_id` hysteresis lets it rotate. If the four break into two camps, `fragmentation` widens the stereo field so you hear the split. `shared-lift` fires only when *all four* bring energy, so no solo can escalate the section. And when one body carries the room, `someone-carrying` quietly withdraws a layer. The dominator gets less, not more, and `on_exit` restores it the moment the room shares the load again.

---

## 7. What Was Rejected and Why

**Design 1 (Minimal two-dancer).** Its `.together`/`.against` suffix convention was rejected: undersold as "one transform" but actually requires new parsing across `combine.ts`'s mix and gate loops plus validator updates, and the mean-field base makes it unnecessary (gate `cohesion below -0.3` already covers anti-phase in existing grammar). Its `mutuality` quality (velocity-balance as a proxy for attentiveness) was rejected as too thin a proxy for stillness-based dialogue. **Kept:** `aggregate_energy` as `min` (the shared-floor encoding), which survives wholesale.

**Design 3 (Thesis-purist).** Its `synchrony`/`opposition` pair split (keeping everything 0..1) was rejected because the winning base already carries a single signed `cohesion`, and `below: -0.3` works as a plain numeric comparison. One signed quality beats two when the grammar can already gate negatives. Its `reciprocity` lead-lag estimation was rejected as the keystone of `mutuality` (flagged by the design itself as least-confident); the lead-lag idea survives in `lead_strength`. **Kept:** the monotonicity test as a screening principle and the validator rule, grafted wholesale into §3.2.

**Design 4 (Maximal reuse).** Its architecture (keep O(n²) pairwise loops) was rejected because the mean-field base is both cheaper and more thesis-aligned. Its `unison`/`contrast` redundancy is resolved by keeping `unison` and dropping pairwise `contrast`. One factual error corrected: Design 4 claimed signed values are safe because `AdaptiveRange` normalizes them; in fact the relational path bypasses `AdaptiveRange` entirely. **Kept:** the staged backwards-compatible migration shape (5 PRs), the `scope` field (from Design 4's `source` idea), and the `relational_window` tunability setting.

---

*Output of issue [#3](https://github.com/meninoebom/ralf-docs/issues/3) in the RALF Ultracode Roadmap. Design tournament: 4 designs × 4 judges × 1 synthesis, Opus 4.8, June 2026.*
