# Expert Review Key Findings

- Jerkiness MUST be windowed variance of acceleration, NOT literal 3rd derivative
- One-euro filter is the right smoothing choice
- Coherence: windowed Pearson correlation, left vs right velocity, 20-frame window
- Trajectory: single most important unbuilt feature (now implemented)
- MediaPipe confidence gating: hold last value when joint visibility < 0.5
