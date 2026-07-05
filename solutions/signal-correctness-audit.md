# Signal-Correctness Audit

A cross-repo verification that every "quality" RALF emits actually computes what its name claims, checked against math first principles and the constraints in `expert-review.md`. This is a correctness audit of the signal-processing chain, not artistic tuning.

## Bottom line

The audit examined **36 formula units** across the pipeline (adapters, runtime, both pose layers). It produced:

- **7 confirmed findings** that withstood adversarial refutation (4 high, 1 medium, 2 low).
- **11 candidate findings refuted away** (examined, judged not to be real problems, or judged out of scope).
- **18 units verified correct.**

The single most important pattern: the **SOMI sensor path computes several qualities with different formulas than the MediaPipe pose path, under the same field names.** `jerkiness` and `velocity` are materially different quantities on the two paths, so anything downstream that consumes those qualities gets inconsistent numbers depending on which sensor produced them. The pose path is correct against the expert spec; the SOMI path is not.

The good news: the expert spec's hardest constraints already hold on the pose path. MediaPipe confidence gating (hold last value when visibility < 0.5) is correctly implemented in the adapter's `cleanSkeleton`. Coherence is a correct windowed L-vs-R Pearson correlation. The One Euro filter, the trajectory regression, and the core kinematic chain (velocity, acceleration, jerkiness on the pose path) are all correct.

## How the audit was run

Three stages, structured so that the danger mode (a plausible-but-wrong formula that throws no error) gets caught by independent skeptics rather than rationalized in a single pass:

1. **Inventory.** Enumerate every emitted or computed quantity across the repos, with exact `file:line` and a plain-language read of what the code actually does.
2. **Verify.** One agent per formula. Derive from first principles plus the expert spec what the quantity *should* compute, compare to the implementation, and flag any discrepancy with a category and severity.
3. **Refute.** For every flagged finding, three independent skeptics each try to *refute* it, one through a mathematical-correctness lens, one through a numerical-stability lens, one through a naming-vs-intent lens. Each skeptic defaults to "refuted" unless the case is solid. A finding is **confirmed** only if at least two of three skeptics fail to refute it.

Categories used:

- **wrong-math**: the formula computes a materially different or incorrect quantity than its name and docs claim.
- **design-choice-undocumented**: a defensible modeling choice that silently discards information or diverges from the spec, and should be documented.
- **naming-smell**: the math may be intentional, but the field name or doc misleads.

Out of scope, by design: the numeric *values* of empirically-chosen tuning constants (`VARIANCE_TO_FULLY_UNSTILL = 0.25`, `JERK_NORMALIZE = 10`, buffer sizes, the `0.6` and `0.4` thresholds). Those are rehearsal-gated and belong to the separate SOMI-tuning milestone. The audit flags formula *shape* and naming, never a constant's value.

---

## Confirmed findings (ranked)

### 1. SOMI `jerkiness` is a literal derivative, not windowed variance

- **Severity: HIGH. Category: wrong-math. Confidence 0.95 (3/3 skeptics upheld).**
- **Location:** `adapters/shared/somi-quality-math.ts:82-95`

**Should compute:** windowed variance of acceleration over a short recent window. The expert spec is explicit: jerkiness MUST be windowed variance of acceleration, NOT a literal derivative. The pose path is the correct reference (`adapters/shared/quality-math.ts:320-343`): it builds an acceleration series and returns `variance(recent)` over the last ~10 frames.

**Actually computes:** the mean absolute first derivative of acceleration magnitude: `mean(|(magnitude_i - magnitude_{i-1}) / dt|)`, normalized by `JERK_NORMALIZE`. This is the literal rate of change of acceleration magnitude (jerk magnitude), which is exactly the construction the spec forbids.

**Why it matters:** the two formulas are not monotonic transforms of each other. A smooth acceleration ramp produces large mean-derivative but small variance; a noisy signal with zero net drift produces large variance but modest mean-derivative. So the field named `jerkiness` emits materially different numbers on the SOMI path versus the pose path, feeding the same downstream readings.

**Recommended fix:** replace the mean-absolute-derivative loop (lines 84 to 95) with windowed variance of acceleration magnitude, mirroring the pose path. Concretely: `const mags = samples.map(magnitude); const recent = mags.slice(-N); jerkiness = clamp01(variance(recent) / JERK_VARIANCE_NORMALIZE)`. The `variance()` helper is already imported (it is used three lines later for stillness). If a literal rate-of-change measure is genuinely wanted as a separate quality, give it a different name (for example `jolt` or `accel_roughness`) rather than overloading `jerkiness`.

### 2. MoveNet confidence gating is render-only, not stream-gating

- **Severity: HIGH. Category: design-choice-undocumented. Confidence 0.92 (2/3 skeptics upheld).**
- **Location:** `movenet/src/public/js/app.js:4-75` (and the emit path at `app.js:128-131`, `movenet/src/server/osc.js:57-90`)

**Should compute:** per the expert constraint, confidence gating should act on the data stream before it leaves the browser. For each keypoint whose score is below threshold, hold the last known-good x/y rather than forwarding the noisy estimate, so that downstream Sense, Smooth, readings, and DTW gesture matching never see sub-threshold noise.

**Actually computes:** `SCORE_THRESHOLD = 0.4` is consulted only in the visualization code (which skeleton edges and dots to draw). The actual stream is ungated. `detectPoses()` emits the full raw keypoint array, and the server forwards every enabled keypoint without ever reading `kp.score`. There is no hold-last-value logic on the data path. A near-zero-confidence joint's jittery coordinate is forwarded identically to a high-confidence one.

**Why it matters:** this is the silent-poisoning mode. No error is thrown, but raw sub-threshold coordinates flow straight into trajectory, velocity, and gesture computations, degrading results invisibly. The presence of a threshold constant misleadingly implies gating is happening. Note the contrast: the MediaPipe path satisfies this same expert constraint correctly in the adapter (`cleanSkeleton`, see Verified Correct below); only the MoveNet path is missing it.

**Recommended fix:** move confidence gating onto the emit path. Maintain a per-keypoint last-good buffer; before `socket.emit('poses', ...)`, replace the x/y of any keypoint with `score <= SCORE_THRESHOLD` with its last accepted value (carry-forward), and update the buffer for keypoints above threshold. Equivalently, do the same hold-last gating server-side before `sendOsc()`. Keep the existing draw-time threshold as is. If raw pass-through is intentional, document that and rename `SCORE_THRESHOLD` to read as a render-only visualization threshold.

### 3. `computeGroundedness` lacks the peak-and-arrest detection it documents

- **Severity: HIGH. Category: wrong-math. Confidence 0.9 (3/3 skeptics upheld).**
- **Location:** `adapters/shared/quality-math.ts:599-634`

**Should compute:** per its own docstring, groundedness should detect impulsive weight-into-floor events: downward vertical-velocity peaks followed by deceleration (the body mass accelerates downward, then the floor arrests it). From first principles this needs (a) hip-centroid vertical velocity, (b) detection of local peaks of downward velocity, and (c) confirmation of the arrest (a sharp negative vertical acceleration right after the downward phase).

**Actually computes:** half-wave rectified, time-summed downward hip velocity: `downwardSum = sum of vy over frames where vy > 0`, divided by `totalVelocitySum` (the per-frame mean 3D speed across up to 33 joints), then rescaled by `/0.6`. There is no peak picking and no acceleration or arrest detection.

**Why it matters:** the defining behavior (impulsive plant-and-arrest) is entirely absent, not merely simplified. A dancer drifting steadily downward accumulates the same score as one performing repeated weight-drops, which is close to the opposite of the intended semantics. The numerator (hip centroid, vertical axis, 1 joint) and denominator (mean 3D speed over up to 33 joints) are also different physical aggregates, so the ratio is not a clean fraction; it is structurally diluted by the joint averaging and then rescaled by an ad hoc `/0.6`.

**Recommended fix:** implement the documented event-based definition. From the existing `vy[]` series, compute vertical acceleration `ay[t] = (vy[t+1] - vy[t-1]) / (2*dt)`, detect grounding events as local maxima of `vy` (vy > 0) immediately followed by sufficiently negative `ay` (the arrest), accumulate the impulse energy or count of such events, and normalize against a denominator of the same quantity (for example total vertical kinematic energy of the hip) so the result is a genuine 0..1 fraction. If the rectified-fraction heuristic is intentionally kept for now, rewrite the docstring to say plainly that it is "fraction of total full-body speed attributable to downward hip translation (no peak or arrest detection)" and file a follow-up to implement the real detector.

### 4. SOMI `velocity` is an accumulator of acceleration magnitude, not speed

- **Severity: HIGH. Category: wrong-math. Confidence 0.9 (3/3 skeptics upheld).**
- **Location:** `adapters/shared/somi-quality-math.ts:72-80`

**Should compute:** speed derived from accelerometer data is the magnitude of the time-integral of the acceleration *vector*, with gravity removed. Its defining property: it rises when acceleration aligns with motion and falls when acceleration opposes motion, returning toward zero when the limb stops or reverses. For an oscillating limb it should oscillate around zero, not accumulate.

**Actually computes:** `velocityRaw = sum of magnitude(sample) * dt` over the window, the time-integral of scalar acceleration *magnitude*, then clamped. Because magnitude is always non-negative, this is monotonically non-decreasing within a window of motion: it can only grow, never decrease, so it cannot represent slowing or reversing. For oscillatory motion the positive and negative accelerations do not cancel (they would in a vector integral), so the value ramps up every half-cycle and saturates at the clamp. It also integrates the gravity component baked into `magnitude()`, so on real (gravity-present) sensor data even a motionless sensor accumulates toward the clamp. The tests pass only because their fixtures use synthetic gravity-removed `acc=(0,0,0)` data.

**Why it matters:** the field, the OSC channel `/somi/<part>/velocity`, and the docstring all claim "velocity," but an integral of `|a|` is a different quantity (an exertion or activity accumulator). It drives downstream sound under the wrong name, and on real sensor data it pins to 1 for sustained movement and accumulates gravity bias at rest.

**Recommended fix:** preferred path (A), integrate the acceleration *vector* with gravity removed and a leaky decay to bound drift: maintain a per-sensor running velocity vector, subtract estimated gravity (a slow low-pass of the signal, or the SOMI hub's gravity-removed stream if available), apply `v *= leak` each step, and report `clamp01(|v| / SCALE)`. A simpler honest proxy is windowed RMS of gravity-removed acceleration magnitude, which at least does not monotonically accumulate. Minimum acceptable path (B): if the monotonic accumulator is genuinely wanted, rename the field and OSC channel to something like `exertion` and update the docstring; then fix the test that currently locks in monotonic growth. In all cases, enforce the gravity contract: if `magnitude()` includes gravity, the still-sensor case is wrong, so subtract gravity first.

### 5. `computeStillness` uses a relative threshold while its name and doc say absolute

- **Severity: MEDIUM. Category: design-choice-undocumented. Confidence 0.78 (2/3 skeptics upheld).**
- **Location:** `adapters/shared/quality-math.ts:511-542`

**Should compute:** normalized duration the body has stayed below a velocity threshold. The counting and `stillFrames * dt / maxDuration` normalization are correct. For the 0..1 scalar to be comparable across performers and to behave monotonically (holding stiller raises stillness toward 1), the parameter named `threshold = 0.1` and the doc "below threshold" imply an absolute speed bound.

**Actually computes:** the threshold is self-scaling, not absolute. `absThreshold = 0.1 * max(recentMax, 0.001)`, where `recentMax` is the peak speed over the last ~90 frames. So a frame counts as "still" only if its speed is below 10% of the recent peak.

**Why it matters:** two effects, both undocumented. First, a decaying-floor pathology: when the dancer truly holds still, `recentMax` decays toward the `0.001` floor, the threshold collapses toward ~0.0001, and ordinary pose jitter then exceeds it and resets the still-count, so sustained perfect stillness can fail to register, which is backwards. Second, it is scale-relative, not absolute: two performers at identical absolute speeds get opposite stillness values depending on recent peak, breaking cross-performer comparability. The counting math itself is fine, so this is a documentation and naming issue, not wrong-math. (This is not a flag on the value `0.1`, which is out of scope; it is a flag on the relative-vs-absolute semantics and the misleading parameter name.)

**Recommended fix:** either (a) make the threshold absolute to match the name (`absThreshold = threshold`, drop the `recentMax` scaling), or (b) if the adaptive relative threshold is intentional, rename the parameter to `peakSpeedFraction`, update the doc to "fraction of recent peak speed below which motion counts as still," and add a sensible absolute floor (not the near-zero `0.001`) so sustained true stillness can still saturate to 1. Either way, document the decision and the comparability implication next to the function.

### 6. `synchrony` divides by the wrong pair count

- **Severity: LOW. Category: wrong-math. Confidence 0.9 (3/3 skeptics upheld).**
- **Location:** `runtime/src/engine/relational.ts:41-54`

**Should compute:** the mean of the available positive pairwise Pearson correlations. Pairs that cannot be evaluated (missing or too-short history) must enter neither the numerator nor the denominator. The Pearson kernel itself and the negative clamp are correct and match the documented spec.

**Actually computes:** `syncSum` accumulates only for pairs whose histories both reach length >= 2 (line 49), but `syncCount` increments for *every* pair unconditionally (line 51, outside the guard). The result is `syncSum / syncCount`, so the divisor counts pairs that contributed nothing, biasing synchrony downward whenever any pair lacks history.

**Why it matters in practice (why it is LOW):** velocity histories fill one sample per tick at ~30fps, so within roughly two frames (~66ms) of a dancer joining, every present pair satisfies the guard and the formula self-corrects. The dilution only manifests transiently on the first frame or two after a new dancer connects, and the downstream Smooth pipeline damps even that. It is a genuine correctness defect (numerator and denominator are over mismatched sets) but it does not poison steady-state output.

**Recommended fix:** move the count increment inside the guard so the denominator matches the contributing set:

```ts
if (ha && hb && ha.length >= 2 && hb.length >= 2) {
  const a = ha.slice(-windowSize);
  const b = hb.slice(-windowSize);
  syncSum += Math.max(0, pearson(a, b));
  syncCount++;
}
```

and drop the unconditional `syncCount++` at line 51. The existing `syncCount > 0 ? ... : 0` fallback already handles the all-unready case.

### 7. `aggregate_energy` aggregates velocity, not the distinct `energy` quality

- **Severity: LOW. Category: naming-smell. Confidence 0.72 (2/3 skeptics upheld).**
- **Location:** `runtime/src/engine/relational.ts:74-78`

**Should compute:** the math (arithmetic mean of each real dancer's already-normalized `velocity` quality) is correct and matches its own inline comment ("mean velocity"); it stays in 0..1 with no instability. The issue is purely the name.

**Actually computes:** `mean over dancers of qualities.velocity`, exposed as the `_crowd` field `aggregate_energy`.

**Why it matters:** `energy` is a genuinely distinct quality in this system (a separate enum member in `runtime/src/types.ts`, a separate `QUALITY_KEYS` entry, and a separately computed quantity in the adapter where `computeEnergy` returns the *sum* of per-joint speeds while `computeVelocity` returns the *average*). Naming a field `aggregate_energy` while it aggregates velocity collides with that existing concept: a scene author would reasonably expect it to aggregate `qualities.energy`. The local comments are honest, which is why this is a naming smell rather than wrong-math.

**Recommended fix:** rename the field to `aggregate_velocity` (or `mean_velocity`) and update the interface comment, inline comment, CLAUDE.md, and any `_crowd`-targeting scenes and the console dashboard. Alternatively, if a crowd-level energy reading is the real intent, change the body to aggregate `qualities.energy`. Either way, make name and computed quantity agree.

---

## Examined and not promoted to a confirmed finding

These were flagged during verification but did not clear the two-of-three adversarial bar, or were judged out of scope. Several correspond to suspects named in the original issue, so they are recorded here with the reasoning rather than dropped silently.

| Unit | `file:line` | Original flag | Why not confirmed |
|---|---|---|---|
| MediaPipe per-joint visibility gating | `mediapipe/src/public/js/app.js:64-71` | hold-last-value missing in browser | The gating *is* implemented, in the adapter's `cleanSkeleton` (verified correct), not the browser. The expert constraint is satisfied for MediaPipe. |
| SOMI `acceleration` naming | `adapters/shared/somi-quality-math.ts:69-70` | field named `acceleration` is instantaneous magnitude | Individually defensible (an accelerometer measures acceleration; the vector magnitude is a reasonable scalar). See the note below: it still creates the same cross-path naming inconsistency as `jerkiness`, so it is worth a doc comment even though it was refuted as a standalone finding. |
| MediaPipe outlier filter (`MAX_JUMP_THRESHOLD`) | `mediapipe/src/public/js/app.js:324-347` | rejection predicate conjoins jump and visibility-decrease | Skeptics judged the conjunction defensible (it only holds a previous value when a joint both jumps and loses confidence). Not a clear wrong-math case. |
| MediaPipe whole-frame hold-last-good | `mediapipe/src/public/js/app.js:468-503` | undocumented continuity mechanism | Sound continuity behavior for DTW (no gaps); correctly implemented; nothing misnamed. |
| MoveNet path: no browser smoothing/outlier/hold | `movenet/src/public/js/app.js:128-148` | diverges from MediaPipe sibling | Smoothing still happens downstream in runtime `Smooth` (One Euro). The most material gap (no stream confidence gate) is captured by confirmed finding 2. |
| Coordinate contract: MoveNet raw pixels vs MediaPipe normalized | `movenet/src/server/osc.js:57-90` | no unifying contract | Refuted as a standalone defect because the two pose layers are alternatives, never mixed in one session, and each is internally consistent. Still a real architectural divergence worth a documented contract. See note below. |
| `video.js` duplicate MediaPipe pipeline | `mediapipe/src/public/js/video.js:178-219` | omits the outlier-filter stage app.js runs | Real parity gap between the two MediaPipe entry points, but no formula is miscomputed; documentation-level. |
| `computeEnergy` (sum of per-joint speeds) | `adapters/shared/quality-math.ts:348-355` | "energy" is linear in speed, not kinetic energy | Math matches its docstring exactly. Naming is a defensible movement-vocabulary convention. |
| `computeSymmetry` | `adapters/shared/quality-math.ts:396-435` | vertical-mirror assumptions | Core reflection math is correct for the claimed quantity. |
| `computeVerticality` | `adapters/shared/quality-math.ts:491-505` | name implies uprightness, computes hip height | Arithmetic matches its docstring; naming judged acceptable. |
| `computeRelational.contrast` | `runtime/src/engine/relational.ts:57-71` | L2 normalization | Computes exactly "mean pairwise L2 distance normalized by sqrt(numQualities)," bounded in 0..1, matching name and doc. |

### Two refuted items still worth a one-line doc comment

1. **SOMI `acceleration` cross-path meaning.** On the pose path, `acceleration` is the derivative of speed (`computeAcceleration`). On the SOMI path, `acceleration` is the raw accelerometer vector magnitude. Both are individually reasonable, but the same field name carries two different quantities system-wide, exactly the pattern that makes confirmed finding 1 (`jerkiness`) dangerous. Worth a comment in `somi-quality-math.ts` stating that SOMI `acceleration` is raw sensor magnitude, not a derivative of velocity.

2. **Pose-layer coordinate contract.** MediaPipe emits normalized 0..1 coordinates; MoveNet emits raw pixels, on the same OSC address shape. They are never used together, so it is not an active bug, but a short documented contract ("pose layers emit normalized 0..1; any raw-pixel source must normalize before emit") would prevent a future third source from silently breaking the convention.

Also noted: the SOMI path is **scalar-magnitude throughout**, so directional information from the 3-axis accelerometer is discarded before any quality is computed. This is an intentional modeling choice, not an error, but it is the root reason the `velocity` integral cannot cancel (confirmed finding 4) and is worth stating explicitly in the module header.

---

## Verified correct (18)

These were checked against first principles and the expert spec and found to compute what they claim. Listing them so the audit's coverage and the system's confidence floor are both visible.

| Unit | `file:line` | Verified as |
|---|---|---|
| `cleanSkeleton` | `adapters/shared/quality-math.ts:115-152` | Correct hold-last-value gating: visibility < 0.5 holds the last confident position. This satisfies the expert confidence-gating constraint for MediaPipe. |
| `pearsonCorrelation` | `adapters/shared/quality-math.ts:173-193` | Standard Pearson r, raw-score-sum form. |
| `variance` | `adapters/shared/quality-math.ts:195-203` | Population variance, stable two-pass. |
| `savitzkyGolayDerivative` | `adapters/shared/quality-math.ts:210-240` | Correct SG derivative coefficients for window 5 and 7. |
| `computeVelocity` | `adapters/shared/quality-math.ts:290-295` | Mean per-joint central-difference speed over visible joints. |
| `computeAcceleration` | `adapters/shared/quality-math.ts:301-314` | Derivative of the velocity series. Correct on the pose path. |
| `computeJerkiness` (pose) | `adapters/shared/quality-math.ts:320-343` | Windowed variance of acceleration over the last ~10 frames. This is the correct reference the SOMI path should match. |
| `computeSpatialExtent` | `adapters/shared/quality-math.ts:360-376` | Visibility-gated mean distance of the five limb endpoints from the hip centroid. |
| `computeContraction` | `adapters/shared/quality-math.ts:383-387` | Inverse of spatial extent. |
| `computeCoherence` | `adapters/shared/quality-math.ts:441-484` | Windowed L-vs-R velocity Pearson over a 20-frame window. Matches the expert spec exactly. |
| `computePeriodicity` | `adapters/shared/quality-math.ts:548-592` | Autocorrelation-based periodicity over the mean-speed series. |
| `updateSomiQuality.stillness` | `adapters/shared/somi-quality-math.ts:97-107` | 1 minus normalized variance of acceleration magnitude. |
| Trajectory regression slope | `runtime/src/engine/runtime.ts:210-234` | Textbook OLS slope. Checked specifically for numerical stability and judged fine at the window sizes used (x = 0..n-1, small windows), so the "naive formulation" suspicion does not bite here. |
| `OneEuroFilter` | `runtime/src/primitives/smooth.ts:35-81` | Standard 1-euro filter, the expert-endorsed smoothing choice. |
| One Euro filter (MediaPipe browser) | `mediapipe/src/public/js/app.js:27-71` | Correct 1-euro implementation. |
| Pose plausibility check | `mediapipe/src/public/js/app.js:73-99` | Correct torso-visibility and shoulder-distance plausibility gate. |
| Coordinate contract: MediaPipe normalized | `mediapipe/src/server/osc.js:99-135` | Emits normalized 0..1 coordinates per the convention. |
| MoveNet keypoint averaging | `movenet/src/public/js/app.js:53-75` | Correct online arithmetic mean for synthetic skeleton vertices. |

---

## Recommended fix priority

1. **SOMI `jerkiness` and SOMI `velocity` (findings 1 and 4).** Both are wrong-math on a path feeding live sound, and both are the same root cause: the SOMI module reimplemented qualities with different formulas than the pose reference. Fix them together, and add the cross-path naming comment for SOMI `acceleration` while in the file. These are the highest-value, clearest fixes.
2. **MoveNet stream confidence gating (finding 2).** A real signal-hygiene gap on the data path. Add hold-last-value before emit, matching what MediaPipe already does in `cleanSkeleton`.
3. **`computeGroundedness` (finding 3).** The largest rewrite. The current heuristic is not just simplified, it omits the defining behavior. If a full rewrite is not wanted now, at minimum correct the docstring so it stops claiming peak-and-arrest detection.
4. **`computeStillness` semantics (finding 5)** and the two low-severity items (**`synchrony` divisor, `aggregate_energy` rename**). The synchrony fix is a one-line change with existing tests; the rename touches consumers so coordinate it with scene configs and the console.

## Remediation tracking

Each confirmed finding has a GitHub issue, labeled `signal-audit` in its home repo so the audit's work is identifiable as a set. Autonomy verdict (`agent-ready` / `needs-eyes`) is from the `/workflow:issues` triage.

| # | Finding | Repo | Issue | Verdict |
|---|---|---|---|---|
| 1 | SOMI `jerkiness` → windowed variance (+ `acceleration` doc comment) | ralf-adapters | [#25](https://github.com/meninoebom/ralf-adapters/issues/25) | agent-ready |
| 2 | MoveNet stream confidence gating | ralf-mocap-movenet | [#1](https://github.com/meninoebom/ralf-mocap-movenet/issues/1) | agent-ready |
| 3 | `computeGroundedness` peak-and-arrest | ralf-adapters | [#28](https://github.com/meninoebom/ralf-adapters/issues/28) | needs-eyes |
| 4 | SOMI `velocity` accumulator | ralf-adapters | [#27](https://github.com/meninoebom/ralf-adapters/issues/27) | needs-eyes |
| 5 | `computeStillness` absolute threshold | ralf-adapters | [#26](https://github.com/meninoebom/ralf-adapters/issues/26) | agent-ready |
| 6 | `synchrony` divisor | ralf-runtime | [#1](https://github.com/meninoebom/ralf-runtime/issues/1) | agent-ready |
| 7 | `aggregate_energy` rename | ralf-runtime | [#2](https://github.com/meninoebom/ralf-runtime/issues/2) | needs-eyes |
| — | Doc comments (scalar-magnitude header, coordinate contract, video.js parity) | ralf-adapters | [#29](https://github.com/meninoebom/ralf-adapters/issues/29) | agent-ready |

## Scope and caveats

- This audit covers whether each *formula* computes the right quantity. It does not tune the *values* of empirically-chosen constants; those are rehearsal-gated and belong to the SOMI-tuning milestone.
- The findings are recommendations with `file:line` and concrete fixes. Applying them is a separate change set that touches three repos (`adapters`, `runtime`, `movenet`) and their tests, and some (SOMI velocity, groundedness) alter runtime behavior that should be re-checked against rehearsal data.
- No data dependency: this audit was runnable purely from the source and the expert spec.
