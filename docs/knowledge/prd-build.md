# Product requirements and build guide

Source: trinetra_prd_build. Binding for requirements and acceptance; see corrections for the build steps.

## Vision and scope

Estimate position, velocity and orientation continuously when GPS is unavailable, degraded or denied, by fusing inertial, magnetic and barometric data (quantum magnetometer, visual and LiDAR later) with learned error models and absolute magnetic map matching.

| Goals | Non-goals |
|---|---|
| Drift-bounded GPS-denied state estimation | Physical quantum hardware |
| AI modules that denoise sensors and adapt the filter | Path planning / mission autonomy |
| Absolute anchoring via magnetic map matching | Multi-robot swarms |
| Reproducible, ROS 2-deployable software | Certified avionics |
| Rigorous evaluation on public + simulated data | A commercial product |

**Success in one sentence:** on a held-out GPS-denied trajectory, position error grows substantially slower than a classical inertial baseline, runs in real time on an edge device, and every number is reproducible from versioned code + data + config.

Users: the developer/researcher; a robot/vehicle; a reviewer who clones the repo and runs one command; future sensors behind the interface. Scenarios: vehicle in a tunnel; pedestrian indoors with IMU + magnetometer; drone under jamming using inertial + baro + magnetic.

## Functional requirements

| ID | Requirement |
|---|---|
| FR-1 | Ingest multi-rate streams (IMU 100–200 Hz, mag, GPS 1 Hz, baro) from datasets, bags, drivers |
| FR-2 | Time-sync and resample to a common grid; mask gaps |
| FR-3 | Calibrate bias, scale, hard/soft-iron |
| FR-4 | Windowed features (statistical, spectral, lags) |
| FR-5 | EKF/UKF fusion into pose + velocity + bias state |
| FR-6 | Learned denoising of inertial signals |
| FR-7 | Predict and correct drift/bias |
| FR-8 | Detect faults/anomalies; score confidence |
| FR-9 | Adapt Q, R from confidence/anomaly |
| FR-10 | Absolute correction via magnetic map matching |
| FR-11 | Simulate GPS denial; enter/exit denied mode cleanly |
| FR-12 | ROS 2 nodes/topics with a correct TF tree |
| FR-13 | Record/replay sessions (MCAP) |
| FR-14 | Visualise trajectory, error, sensor health |
| FR-15 | Track experiments reproducibly |

## Non-functional requirements

| ID | Attribute | Target |
|---|---|---|
| NFR-1 | Accuracy | denied-mode error grows markedly slower than strapdown INS / naive double integration |
| NFR-2 | Real-time | keeps up with sensor rate (200 Hz ⇒ < 5 ms/sample) |
| NFR-3 | Reproducibility | commit SHA + data version + config ⇒ same result |
| NFR-4 | Modularity | add/swap a sensor or AI module without editing the core |
| NFR-5 | Portability | identical run on laptop and Jetson |
| NFR-6 | Reliability | graceful degradation when a sensor drops; no NaNs/crashes |
| NFR-7 | Observability | health/anomaly signals and logs for every run |

## Metrics

| Metric | Definition | Target |
|---|---|---|
| ATE | RMSE of estimated vs ground-truth positions | clearly beats INS/naive baseline |
| RTE | relative error over fixed windows (e.g. 60 s) | below baseline, stable across trajectories |
| Drift rate | position-error growth per minute in denied mode | sub-linear vs unaided inertial |
| Attitude error | roll/pitch/yaw RMSE | within a few degrees |
| Real-time factor | processing time / data time | ≤ 1.0 |
| Reproducibility | re-run with fixed seed/data | identical |
| AI confidence calibration | predicted vs actual error | reliability diagram |

## Acceptance criteria (record all three in the README)

On **at least two held-out GPS-denied trajectories**: (i) ATE and drift rate beat a documented inertial baseline; (ii) the full stack runs in real time in simulation, ideally on a Jetson; (iii) every headline number is reproduced by one scripted command from versioned code and data.

## Milestones (vertical slices, each ending tested and tagged)

| M | Milestone | Exit criterion |
|---|---|---|
| M0 | Environment & repo | repo scaffolded; tooling works |
| M1 | Data + EDA | ≥ 1 dataset in canonical schema; EDA report; quality checks pass |
| M2 | Baseline fusion | EKF/UKF end-to-end; baseline ATE/RTE recorded |
| M3 | AI modules | each beats its no-AI ablation |
| M4 | GPS-denied navigation | magnetic map match + denied mode; error bounded vs unaided INS |
| M5 | ROS 2 + simulation | node graph runs in Gazebo; denial simulated |
| M6 | Evaluation + visualisation | benchmark table, ablations, dashboards |
| M7 | Deployment | Docker on Jetson, or documented sim deployment |

**Ship M2 (working baseline on real data) before polishing any AI module.** A mediocre end-to-end system beats a perfect component that connects to nothing.

**⚠ Correction (M0):** the PRD asks for Docker, CI, DVC and MLflow before touching data. Per CLAUDE.md, M0 is git + pytest + ruff + Parquet; add the rest when they are needed.

## Risks

| Risk | Mitigation |
|---|---|
| No magnetic ground data | start inertial (RoNIN/TLIO); MagNav for the magnetic path; simulate fields |
| AI doesn't beat baseline | always ablate; keep classical fallback; held-out subjects/trajectories |
| Timing/sync bugs read as sensor error | rigorous sync; unit-test alignment; never interpolate across long gaps |
| Scope creep (planning, SLAM, quantum HW) | interfaces only for future sensors |
| Real-time budget blown on Jetson | profile early; ONNX/TensorRT; compose nodes; lighter models |
| Irreproducible results | seed control; tag every result |
| Dataset licences | verify each; document provenance |

## Build steps (each with a Definition of Done)

**Step 0 — Setup.** Python env, repo scaffold, `pyproject.toml` with pinned deps, first tagged commit. *(⚠ Docker/CI/DVC/MLflow deferred.)*

**Step 1 — Datasets.** Start with **RoNIN** (42.7 h, 200 Hz, accel + gyro + mag, 3-D truth, research licence, ~50 % public). TLIO for learned inertial + EKF; EuRoC/KITTI for visual/GPS paths; **MagNav** for the magnetic path. Keep `datasets/raw/` immutable. *Done:* ≥ 1 dataset downloaded, licence noted in `datasets/README.md`.

**Step 2 — Canonical schema.** One adapter per dataset → `t` (s, monotonic), `ax ay az` (m/s²), `gx gy gz` (rad/s), `mx my mz` (µT), optional `gt_x gt_y gt_z`, `lat lon alt`, `baro` (Pa). Parquet + zstd. *Done:* canonical Parquet with correct units and monotonic time; a second adapter yields the same columns; downstream code imports neither HDF5 nor dataset names. *Pitfalls:* g vs m/s², deg/s vs rad/s, per-dataset axis conventions, duplicate/non-monotonic timestamps. **⚠ Verify the RoNIN keys (`synced/acce`, `synced/gyro`, `synced/magnet`, `synced/time`, `pose/tango_pos`) against a real file.**

**Step 3 — EDA.** Median rate and max gap; ranges (|a| ≈ 9.8 at rest); time-series plots and spectra; ground-truth trajectory and path length; Allan variance on a static segment → Q/R priors. *Done:* per-dataset report stating each sensor's effective rate, noise level and anomalies, plus Q/R priors.

**Step 4 — Preprocessing, in order:** (1) mask long gaps — never interpolate across them; (2) resample to a common grid; (3) remove bias from a static segment; (4) hard/soft-iron magnetometer calibration `(m − b) @ A.T`; (5) band-limit (Butterworth, `filtfilt`) with cut-offs from EDA spectra. *Done:* unit tests prove known inputs → known grid and that gaps stay masked; gravity ≈ 9.8 after calibration. Sub-millisecond misalignment shows up as fake "sensor error".

**Step 5 — Features.** One `window_features` function (mean, std, rms, energy, dominant frequency, low/high spectral bands, jerk RMS) used offline and online; windows L=200, stride 10 over the six IMU channels. *Pitfalls:* train/serve skew, future leakage, unversioned features.

**Step 6 — Baseline EKF.** State = position, velocity, orientation, biases. Predict with IMU mechanisation; update with GPS (when present), zero-velocity, magnetic heading. Expose the innovation. *Done:* runs a whole sequence without diverging; ATE/RTE logged; a GPS-masked run shows the expected drift. **⚠ The PRD's "error-state EKF skeleton" is a linear KF (`x = F x`, no error state, no quaternion). Implement a real ESKF.**

**Step 7 — AI modules.** Denoiser (TCN/CNN, Brossard-style), drift/bias predictor, anomaly detector (gate, raise R), confidence estimator (MC dropout / ensemble → adaptive R). Judge each by end-to-end ATE on held-out trajectories, ablated against the system without it. Export to ONNX. A module that loses its ablation is cut.

**Step 8 — Navigation & magnetic map matching.** Particle filter: particles = candidate positions, predicted by inertial motion, weighted by `exp(−½((z − map(x,y))/σ)²)`, resampled when N_eff < N/2; weighted mean → position fix to the EKF. Denied-mode logic: detect GPS loss from the gap mask → inertial predict + magnetic-fix updates → resume GPS on return. *Done:* with GPS masked, error stays bounded and clearly below unaided dead reckoning; mode transitions have no jumps. *Pitfalls:* particle degeneracy, low-res map, platform magnetic signature, treating the fix as noise-free (give it a realistic R).

**Step 9 — ROS 2 + simulation.** Thin nodes that import the core (never re-implement). Topics: `/imu /mag /gps /baro → /synced → /features → /clean /bias /anomaly /confidence → /odometry/filtered`, `/mag_fix`, `/pose`. Gazebo with noisy sensors; record MCAP. *Pitfalls:* QoS mismatch, broken TF tree, noise-free sim.

**Step 10 — Evaluation.** `scripts/evaluate.py` regenerates the table: INS baseline → EKF → EKF+AI → EKF+AI+MagMatch, on identical held-out trajectories, GPS-denied and GPS-available splits, per-module ablations, seed-controlled. Plots: trajectory overlay, error over time, reliability diagram. *Pitfalls:* evaluating on training trajectories, changing test sets, cherry-picking sequences.

**Step 11 — Visualisation.** Static figures (equal-aspect trajectory with ground truth, error over time); RViz2/Foxglove live; optional Streamlit/Plotly dashboard — only after the numbers are trustworthy.

**Step 12 — Deployment.** Docker; RTF ≤ 1 on Jetson (or documented sim); TensorRT with accuracy parity check; systemd autostart.

## Consolidated pitfalls

Data: raw data in Git, unit/axis/frame mismatches, interpolating across gaps. Timing: sub-ms misalignment. ML: train/serve skew, optimising val loss instead of navigation error, subject leakage, shipping a module that loses its ablation. Filter: mis-scaled Q/R, hidden innovations, testing only with GPS on. ROS 2: re-implementing the core, QoS, TF, noise-free sim. Repro: unseeded runs, unpinned deps. Scope: planning/SLAM/quantum HW in the core.
