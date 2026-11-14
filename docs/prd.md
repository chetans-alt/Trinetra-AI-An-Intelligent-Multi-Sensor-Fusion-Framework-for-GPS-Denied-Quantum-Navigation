# Trinetra-AI: Product Requirements Document

| | |
|---|---|
| **Version** | 1.0 (2026-11-14) |
| **Status** | Baseline. Changes to scope or requirements are recorded in the decisions log (§15) or as ADRs |
| **Source of truth** | This PRD derives from the approved plan, [docs/superpowers/specs/2026-11-14-trinetra-project-plan-design.md](superpowers/specs/2026-11-14-trinetra-project-plan-design.md), and from [CLAUDE.md](../CLAUDE.md). Where they differ, CLAUDE.md wins, then the plan, then this PRD. The old PRD notes in [docs/knowledge/prd-build.md](knowledge/prd-build.md) are superseded. |
| **Releases** | **v0.1**: end of S4, about week 17; the minimum shippable portfolio. **v1.0**: end of S7, about week 29. Schedule assumes 8 h/week on CPU only. |
| **Requirement IDs** | FR-<area>-NN (§5), NFR-<area>-NN (§6), UC-NN (§3), RISK-NN (§14), D-N and Q-N (§15) |

## Contents

1. Product overview
2. Users and personas
3. User journeys / use cases
4. Scope
5. Functional requirements
6. Non-functional requirements
7. Success metrics and KPIs
8. Quality assurance and acceptance testing
9. Data governance
10. Release plan
11. Milestone schedule
12. Dependencies
13. Assumptions and constraints
14. Risk register
15. Open questions and decisions log
16. Glossary
17. Traceability matrix

---

---

## 1. Product overview

### 1.1 The problem

An aircraft that loses GPS through jamming, spoofing or an outage falls back on its inertial navigation system (INS). INS position error grows without bound, so after tens of minutes the solution may no longer be usable. The Earth's crustal magnetic anomaly field is a passive, hard-to-jam signal that is mapped over large areas. Matching a measured anomaly against a map can bound INS drift. Two practical obstacles stand in the way:

1. **Platform interference.** The aircraft's own magnetic field, from permanent, induced and eddy-current sources, contaminates the magnetometer. Classical Tolles-Lawson (TL) compensation removes much of it, but not all, especially for cabin-mounted sensors.
2. **Trust.** The map-match measurement has errors that are correlated over time and vary with the terrain. A filter that trusts it too much becomes inconsistent, and one that trusts it too little loses the benefit.

Published work, mainly the MagNav challenge and Canciani & Raquet, shows this approach is feasible. However, few open, reproducible end-to-end pipelines exist that measure, with ablations, whether *learned* components actually reduce navigation error.

### 1.2 Product vision

Trinetra-AI is an open, reproducible Python framework for GPS-denied aircraft navigation using crustal magnetic anomalies. It compensates the aircraft's magnetic interference, first with Tolles-Lawson and optionally with a small learned residual correction. A particle filter running over INS error states then matches the cleaned anomaly against a map, which bounds inertial drift throughout a GPS outage. Every headline number comes from public MagNav flight data with GPS masked on held-out flights, and a single command regenerates it. The product's value is a careful, honest 2×2 ablation answering one question: do learned compensation and learned measurement trust reduce position error compared with a well-tuned classical system?

### 1.3 What the product is and is not

| Trinetra-AI **is** | Trinetra-AI **is not** |
|---|---|
| A research and portfolio codebase with a reproducible benchmark | A certified, operational or commercial navigation product |
| Offline processing of MagNav HDF5 flight data on a laptop CPU | A live, real-time system wired to sensors, or a ROS 2 stack (v1) |
| A PF navigator over INS error states, aided by magnetic map matching | A strapdown INS built from raw IMU data (MagNav has none; INS errors are simulated) |
| Exactly two learned modules (residual compensator, learned R), each ablated | A collection of many AI modules (denoising, drift, health and trajectory modules were cut) |
| A system that **bounds** drift during outages | "Drift-free" navigation |
| Software that processes data from MagNav's optically pumped caesium (atomic) magnetometers | Quantum hardware. The project adds no quantum sensor; a noise sweep shows where better sensors would matter |
| An independent project | Affiliated with, endorsed by or a reimplementation of any company's product |

---

## 2. Users and personas

### P1: Hiring navigation or sensor-fusion engineer (primary)

- **Context:** reviews the repository in 5–15 minutes, often only the README, one figure and one or two source files.
- **Goals:**
  - Judge quickly whether the author understands estimation (consistency, correlated errors, leakage), not just ML.
  - See real numbers on real data, compared with a baseline.
- **Pains:**
  - Portfolio repos that overclaim ("drift-free", "quantum").
  - Results on shuffled windows.
  - No baseline.
  - Code that doesn't run.
- **Must be able to:**
  - Read the headline figure and table T2 in the README in under 2 minutes.
  - See the research question, the baselines and honest caveats, including negative results.
  - Open `src/trinetra/core/` and find clean, typed, tested filter code.
  - Find ADRs that explain key decisions (the PF as navigator, truth definitions, licence).

### P2: Researcher reproducing or extending results

- **Context:** has hours to days; may compare against MagNav.jl or their own method.
- **Goals:**
  - Regenerate T1–T3 exactly from a fresh clone.
  - Plug in their own compensator, R model or dataset.
  - Rerun the outage protocol.
- **Pains:**
  - Unpinned dependencies.
  - Unclear data versions.
  - Hidden test-set tuning.
  - IGRF or GPS leakage.
  - Licence ambiguity.
- **Must be able to:**
  - Run `uv sync`, then the verified download, then `uv run trinetra reproduce`.
  - Trace every number to a run directory holding the config, git SHA and data-manifest hash.
  - Implement a `Compensator` or `RModel` protocol without editing `core/`.
  - Read the splits, excluded inputs and licence position in the docs.

### P3: The author as developer

- **Context:** works alone, 8 h/week, CPU only, across about 29 weeks with gaps.
- **Goals:**
  - Every week ends in working, tested software.
  - Reach the v0.1 portfolio at about week 17.
  - Avoid infrastructure creep.
- **Pains:**
  - Losing context between sessions.
  - Slow experiments on CPU.
  - A previous attempt that stalled on infrastructure.
- **Must be able to:**
  - Run fast, data-free tests locally and in CI.
  - Launch one outage experiment from a config and get a self-describing run directory.
  - Add a slice without refactoring the core.
  - Rely on tests that check the mathematics (NEES/NIS, Schuler drift, IGRF identity).

---

## 3. User journeys / use cases

> CLI command names other than `trinetra version` and `trinetra reproduce` are **provisional** and will be fixed when each slice is built.

**UC-01: Clone, download and reproduce the headline** (P2, P1)
1. Clone the repository and run `uv sync` (Python 3.12, locked).
2. Read `datasets/README.md` and accept the data terms. Run `uv run python scripts/download_data.py --include <glob>`.
3. The script downloads Zenodo record 12723700 and the MagNav.jl map artifact, verifies the per-file MD5 values, and checks the files against the committed SHA-256 `datasets/manifest.json`.
4. Run `uv run trinetra reproduce`.
- **Outcome:** a new `runs/<date>_reproduce/` directory containing tables T1–T3 (T1–T5 at v1.0), the headline figure and `metrics.json`. The numbers match the README within the documented Monte-Carlo tolerance. A manifest mismatch stops the run with `ManifestMismatchError`.

**UC-02: Run an outage experiment with a chosen configuration** (P2, P3)
1. Copy a config in `configs/` and set the test flight(s), outage lengths (10/30/60 min), IMU grade, number of seeds, compensator, R model and diurnal mode (base station or FOGM).
2. Run `uv run trinetra outage --config <file>` (provisional).
3. The harness slides outage starts every 5 min along the flight, requires at least 60 s of valid map coverage at each start, and runs at least 20 INS seeds per start.
- **Outcome:** a run directory containing the config, git SHA, manifest hash, per-outage metrics, error-vs-time curves and the divergence rate.

**UC-03: Swap a compensator or R model** (P2)
1. Implement `Compensator.fit(frames)` / `.predict(frame) -> (anomaly_nT, var_nT2)`, or implement `RModel.scale(context)`.
2. Register it under a config name. Nothing in `core/` changes.
3. Rerun UC-02 or the ablation command with the new name.
- **Outcome:** the new variant appears as a row in T2, with paired differences and block-bootstrap 95% CIs against B1 on identical outages and seeds.

**UC-04: Inspect consistency diagnostics** (P1, P2, P3)
1. Open a run directory, or run `uv run trinetra report <run>` (provisional).
2. View the NEES and NIS time series against χ² bands, the N_eff trace, multi-peak and divergence flags, and the NIS-gate rejection log.
3. For learned modules, view the PIT histogram and the 68%/95% coverage.
- **Outcome:** the user can tell whether the filter is overconfident, where it diverged, and whether the learned variance is calibrated (table T3).

**UC-05: Add a new dataset adapter** (P2)
1. Write `src/trinetra/adapters/<name>.py`. This is the only place allowed to import the dataset's reader or name.
2. Map the data to the canonical (extended) schema in Parquet (zstd), with monotonic `t`, SI and nT units, and gap masks.
3. Add the dataset's licence to `datasets/README.md` and its checksums to a manifest.
4. Run the adapter round-trip and import-rule tests.
- **Outcome:** the downstream pipeline, compensators and harness run on the new data unchanged.

**UC-06: Inject faults and check the gate** (P2, P3)
1. Enable synthetic fault injection (bias steps, spikes or drift on the anomaly) in an outage config.
2. Run the experiment.
- **Outcome:** a report of detection and false-alarm rates for the χ² NIS gate, and the position error with and without gating.

**UC-07: Five-minute review** (P1)
1. Open the README.
2. Read the pitch, the headline figure (median and 5–95% band, log y-axis, B0/B1/+NN/+NN+learned R, 1σ envelope) and T2.
3. Follow the links to the ADRs and `core/`.
- **Outcome:** the reviewer understands the research question, the result (including whether it was negative or mixed) and the honest-claims caveats.

---

## 4. Scope

### 4.1 In scope: v0.1 (S0–S4, about week 17; minimum shippable portfolio)

- Reproducible environment (uv, lockfile, Python 3.12), src layout, import-rule test, ADR-0001 to ADR-0003, plus an ADR for splits.
- Verified MagNav download, adapter to Parquet, line segmentation, gap masking, field inventory, free-inertial verdict on `ins_*`, anomaly-floor table.
- Map loader with bicubic interpolation, gradient, FFT upward continuation and a filled-cell mask.
- 18-term TL and ridge-TL compensation, table T1, and a MagNav.jl cross-check (optional; plan D6).
- Pinson INS error simulator at navigation grade, with seeds.
- Outage harness, metrics and B0 (INS-only) curves.
- PF navigator with a NIS gate, fault injection and fixed or PF-spread R, giving B1.
- `trinetra reproduce` regenerating T2 and the headline figure for B0 and B1.
- GitHub Actions CI (ruff, pyright, data-free tests).

### 4.2 In scope: v1.0 (S5–S7, about week 29)

- AI-1: residual NN compensator ensemble (5 members, ≤50k parameters, NLL loss, CPU), with calibration.
- AI-2: learned R, compared with fixed, PF-spread and Sage-Husa R.
- The full 2×2 ablation plus the reference rows (B0, stinger Mag 1 upper bound, PF-spread R, Sage-Husa R).
- IGRF/diurnal variants: base-station diurnal (headline) and diurnal as a FOGM state (ablation).
- Tables T1–T5, error budget, sensor-noise ("quantum") sweep, real-time factor at 10 Hz on a laptop CPU.
- Final README, verified literature numbers, and a fresh-clone reproduction on a second machine.

### 4.3 Out of scope

- **Not in v1 (possible later):**
  - 15-state ESKF and the ESKF→PF cascade ablation.
  - Tightly coupled EKF (H = ∇map).
  - Tactical-grade headline. Tactical grade may appear as a sweep; its release is unverified.
  - FOGM state for compensation error.
  - ROS 2, Docker, C++, Jetson.
  - RoNIN and other datasets.
  - UKF, ECEF frame, coning/sculling corrections, Allan variance.
  - Streamlit or live dashboards; live ingestion.
- **Cut entirely:** learned denoising, drift/bias prediction, health monitoring and trajectory-prediction modules; GANs/VAEs, RL, GNNs, XAI; TensorFlow.
- **Never:** DVC, TimescaleDB, Redis, Kafka, Spark, Hydra. Also never: committing raw or derived data or trained weights; fusing GPS during evaluation; latitude, longitude, time or map value as NN inputs; tuning on test flights; picking the best seed; claims of affiliation.

---

## 5. Functional requirements

Priority uses MoSCoW. Release "post" means after v1.0.

### FR-DATA: Data acquisition and preparation

| ID | Requirement | Priority | Release | Acceptance criterion |
|---|---|---|---|---|
| FR-DATA-01 | The system shall download the MagNav data (Zenodo 12723700) and the MagNav.jl map artifact through `scripts/download_data.py`, with `--list` and `--include` options. | Must | v0.1 | On a clean machine, the selected files land in the gitignored `data/`. A `git status` afterwards shows no data files. |
| FR-DATA-02 | The system shall verify every downloaded file against Zenodo's per-file MD5 and the committed SHA-256 manifest, and refuse to continue on mismatch. | Must | v0.1 | `test_manifest_round_trip` passes. Altering one MD5 raises `ManifestMismatchError`. The manifest is never silently overwritten. |
| FR-DATA-03 | The system shall provide a MagNav adapter that writes per-flight Parquet (zstd) in the canonical schema, extended per ruling R10 (scalar magnetometers in nT, fluxgate vectors, `ins_*` outputs, currents, line IDs, GPS truth). | Must | v0.1 | The round-trip test passes. `t` is strictly monotonic. Units are SI or nT. No module outside `adapters/` imports h5py or dataset names (import-rule test). |
| FR-DATA-04 | The system shall segment flights into lines using the line IDs, and mark the calibration segments. | Must | v0.1 | Segment counts per flight match the field inventory. TL fit code can select calibration segments only. |
| FR-DATA-05 | The system shall mask gaps longer than a configured threshold and never interpolate across them. | Must | v0.1 | A unit test with a synthetic gap shows it stays masked after resampling. |
| FR-DATA-06 | The system shall verify the IGRF identity `mag_1_dc − mag_1_igrf = igrf` on the real data. | Must | v0.1 | The data test passes within 0.01 nT on every downloaded flight, and is skipped when data is absent. |
| FR-DATA-07 | The system shall test whether the recorded `ins_*` stream is free-inertial or GPS-aided, and record a written verdict. | Must | v0.1 | A verdict document exists. The simulated INS (FR-INS) remains the primary baseline unless the test shows `ins_*` is free-inertial. |
| FR-DATA-08 | The system shall produce an EDA field inventory (names only) and an anomaly-floor table per flight. | Should | v0.1 | Both are generated by a script. The inventory contains field names only, never values or derived data. |

### FR-MAP: Anomaly map model

| ID | Requirement | Priority | Release | Acceptance criterion |
|---|---|---|---|---|
| FR-MAP-01 | The system shall load the MagNav anomaly maps through `adapters/magnav_maps.py` into a core `MapModel`. | Must | v0.1 | The core map type has no h5py dependency (import-rule test). The map extent and resolution match the source metadata. |
| FR-MAP-02 | `MapModel.value()` shall interpolate bicubically. | Must | v0.1 | On an analytic test field, the interpolation error is below a tolerance stated in the test. |
| FR-MAP-03 | `MapModel.gradient()` shall return the spatial gradient of the anomaly. | Must | v0.1 | It matches the analytic gradient of the test field within the stated tolerance. |
| FR-MAP-04 | The system shall upward-continue the map to flight altitude using an FFT. | Must | v0.1 | Continuing an analytic field (for example a dipole) reproduces the closed form within tolerance. Continuing by 0 m is the identity. |
| FR-MAP-05 | `MapModel.valid()` shall report cells filled by k-NN interpolation, or outside coverage, as invalid. | Must | v0.1 | The mask matches the fill flags. The harness rejects outage starts with less than 60 s of valid coverage. |

### FR-COMP: Magnetic compensation

| ID | Requirement | Priority | Release | Acceptance criterion |
|---|---|---|---|---|
| FR-COMP-01 | The system shall fit an 18-term Tolles-Lawson model (permanent, induced, eddy) on calibration segments only (Flt1006 and the Flt1002 calibration segment). | Must | v0.1 | A unit test checks the design matrix against hand-computed values. TL on Mag 1 reproduces `mag_1_c` to within a few nT. |
| FR-COMP-02 | The system shall provide ridge-regularised TL using fluxgate B and band-pass filtering. | Must | v0.1 | Table T1 (nT RMSE per flight and magnetometer) is produced by one command. Cabin magnetometers are within a stated factor of MagNav.jl; a gap above 2× is explained in `docs/validation`. |
| FR-COMP-03 | Every compensator shall implement `predict(frame) -> (anomaly_nT, var_nT2)`. | Must | v0.1 | A protocol conformance test passes for TL, ridge-TL and NN. Classical compensators return a constant calibrated variance. |
| FR-COMP-04 | The system shall compute the compensation reference as stinger Mag 1 (`mag_1_c`) minus diurnal minus IGRF, and never use the map as a training target. | Must | v0.1 | ADR-0002 documents this. A test asserts that map values are absent from training targets. |
| FR-COMP-05 | The system shall provide AI-1, a residual NN ensemble on top of TL (5 members, ≤50k parameters each, NLL loss, CPU), trained on flights 1002 and 1004 and validated on 1005. | Must | v1.0 | It beats ridge-TL in nT on validation, or the loss is reported. Inference takes under 1 ms per sample on CPU. At least 5 training seeds. |
| FR-COMP-06 | The NN shall exclude latitude, longitude, time and map value from its inputs, and share one feature function between training and evaluation. | Must | v1.0 | A test fails if any excluded input appears. The training and evaluation feature paths call the same function. |
| FR-COMP-07 | The NN's predicted variance shall be calibrated. | Must | v1.0 | 95% coverage within ±5 points on validation. A PIT histogram is produced. |

### FR-INS: INS error simulation

| ID | Requirement | Priority | Release | Acceptance criterion |
|---|---|---|---|---|
| FR-INS-01 | The system shall simulate INS errors on the GPS truth trajectory using a Pinson error model in local-level NED (WGS-84, Somigliana gravity, transport rate). | Must | v0.1 | Unit tests on gravity and Earth radii match hand-computed values. |
| FR-INS-02 | The simulator shall take a declared IMU grade from config, with navigation grade as the headline. | Must | v0.1 | The grade parameters are listed in the config and README. The specific grade values are **unverified** until checked against a primary source. |
| FR-INS-03 | The simulator shall be seeded and reproducible. | Must | v0.1 | The same seed gives bit-identical trajectories. The harness supports at least 20 seeds per outage start. |
| FR-INS-04 | The simulator shall reproduce Schuler-period drift. | Must | v0.1 | A Monte-Carlo test matches analytic drift within 10%. |
| FR-INS-05 | The system shall support a tactical-grade sweep. | Could | post | The sweep runs; results are reported separately from the headline. |

### FR-NAV: Navigation (PF navigator)

| ID | Requirement | Priority | Release | Acceptance criterion |
|---|---|---|---|---|
| FR-NAV-01 | The system shall provide a PF navigator over INS error states (δN, δE, δv_N, δv_E, map/sensor bias) behind the `Navigator` protocol (`init`, `propagate`, `update`, `nis`). | Must | v0.1 | With GPS masked, error stays bounded through 30-minute outages on both test flights (1003, 1007). `core/` imports only numpy and scipy. |
| FR-NAV-02 | The PF shall use 1–5k particles (configurable) and systematic resampling when N_eff < N/2. | Must | v0.1 | Unit tests: resampling preserves weights in expectation, and a zero-information update leaves the state unchanged. |
| FR-NAV-03 | The PF shall raise divergence flags for N_eff collapse and for a multi-peaked particle cloud. | Must | v0.1 | Flags are logged per step. The divergence rate (error > 500 m) is reported per configuration. |
| FR-NAV-04 | The system shall gate map measurements with a χ² test on NIS. | Must | v0.1 | Gate decisions are logged. Mean NIS is reported against χ² bands (T3). |
| FR-NAV-05 | The system shall inject synthetic faults (for example bias steps and spikes) into the anomaly measurement. | Must | v0.1 | Detection and false-alarm rates for the gate are reported (UC-06). |
| FR-NAV-06 | The system shall provide R models behind `RModel.scale(context)`: fixed tuned R and PF-spread R (v0.1); Sage-Husa and learned R, AI-2 (v1.0). | Must | v0.1 / v1.0 | Each is selectable by config. Learned R keeps mean NIS inside the χ² band, and its paired CIs appear in T2. |
| FR-NAV-07 | The system shall evaluate IGRF at the *estimated* position and use base-station diurnal as the headline, with a FOGM diurnal state as an ablation. | Must | v1.0 | Both variants appear in the results. A test checks that the navigator never reads the GPS-position IGRF field. |
| FR-NAV-08 | The system shall provide a 15-state ESKF, the ESKF→PF cascade ablation and a tightly coupled EKF. | Could | post | An ADR records the double-counting result of the cascade ablation. |

### FR-EVAL: Evaluation

| ID | Requirement | Priority | Release | Acceptance criterion |
|---|---|---|---|---|
| FR-EVAL-01 | The system shall provide an outage harness with 10, 30 and 60 min outages and starts every 5 min, each needing at least 60 s of valid map coverage, with initial P set to the declared GPS/INS covariance. | Must | v0.1 | The number of outages per flight is reported. GPS is never fused during evaluation (test). |
| FR-EVAL-02 | The system shall compute these metrics: horizontal error vs time; DRMS at 10/30/60 min; CEP50 and CEP95; ATE; drift (m/min and % of distance); final error; divergence rate; NEES and NIS; compensation RMSE (nT); PIT and coverage; real-time factor. | Must | v0.1 (PIT/coverage v1.0) | Metric functions pass hand-computed unit tests. Every run writes `metrics.json`. |
| FR-EVAL-03 | The system shall enforce flight-level splits: TL fit on calibration boxes; NN training on 1002 and 1004; validation on 1005; test on 1003 and 1007; never shuffled windows. | Must | v0.1 | Splits are recorded in an ADR. The test set is touched once per frozen configuration, and that use is logged. |
| FR-EVAL-04 | The system shall run the 2×2 ablation (TL or TL+NN × fixed or learned R) plus the reference rows (B0, stinger Mag 1, PF-spread R, Sage-Husa R) on identical outages and seeds. | Must | v1.0 (B0/B1 at v0.1) | T2 lists every cell. Results are reported whatever their sign. |
| FR-EVAL-05 | The system shall compute statistics as paired differences with block-bootstrap 95% CIs and medians with IQR, and make no claim based on a single flight. | Must | v1.0 | T2 shows the CIs. A claim appears in the README only if it holds on both test flights. |
| FR-EVAL-06 | The system shall produce the headline figure (log-y horizontal error vs time since GPS loss; median and 5–95% band; B0, B1, +NN, +NN+learned R; 1σ envelope) and tables T1–T5. | Must | v0.1 (fig, T1–T3 partial) / v1.0 (all) | Figure and tables are generated by `reproduce`, with no manual editing. |
| FR-EVAL-07 | The system shall report a leave-one-flight-out spread, an error budget (T4) and a sensor-noise ("quantum") sweep. | Should | v1.0 | The sweep shows the noise floor below which better magnetometers would matter, relative to platform, map and diurnal errors. |

### FR-CLI: Commands, configuration and runs

| ID | Requirement | Priority | Release | Acceptance criterion |
|---|---|---|---|---|
| FR-CLI-01 | The system shall install a `trinetra` console entry point, with `version` and `reproduce` subcommands. | Must | v0.1 | `uv run trinetra version` prints the package version. Before v0.1, `reproduce` exits with a clear "not yet available" message. |
| FR-CLI-02 | `trinetra reproduce` shall regenerate every headline number and figure from versioned code, data and config. | Must | v0.1 | On a fresh clone after a verified download, it regenerates T2 and the headline figure (v0.1), and T1–T3 on a second machine (v1.0). |
| FR-CLI-03 | The system shall drive experiments from version-controlled config files in `configs/`, selecting the compensator, R model, navigator parameters, IMU grade, seeds, outage lengths and diurnal mode. | Must | v0.1 | Swapping a compensator or R model (UC-03) needs only a config change plus a registered class. |
| FR-CLI-04 | Every run shall write `runs/<date>_<name>/` containing the resolved config, git SHA, data-manifest hash, `metrics.json` and figures. | Must | v0.1 | A test checks that all of these are present. `runs/` is gitignored. |
| FR-CLI-05 | The system shall provide subcommands for single outage experiments, ablations and run reports (names provisional). | Should | v0.1 | UC-02 and UC-04 can be completed with CLI commands alone. |
| FR-CLI-06 | Runs shall refuse dirty or unverified data. | Should | v0.1 | Running with a manifest mismatch exits non-zero with a clear message. |

### FR-DOC: Documentation

| ID | Requirement | Priority | Release | Acceptance criterion |
|---|---|---|---|---|
| FR-DOC-01 | The README shall contain the pitch, the research question, a "Reproduce" section (`uv sync` → download → `trinetra reproduce`), the headline figure and T2. | Must | v0.1 | A reviewer finds the figure and T2 without scrolling past the first two sections. The commands work as written. |
| FR-DOC-02 | The README shall follow the honest-claims policy: "bounds drift", never "drift-free"; the approved quantum wording; negative or mixed results reported; no affiliation implied. | Must | v0.1 | A text check finds no "drift-free" and no company names. The quantum wording matches plan §2. |
| FR-DOC-03 | The system shall maintain ADRs in `docs/adr/`: 0001 (PF navigator), 0002 (truth and reference definitions), 0003 (data and licence policy), and 0004 onward (splits, cascade, and so on). | Must | v0.1 | Each ADR has the sections Status, Context, Decision and Consequences. |
| FR-DOC-04 | `datasets/README.md` shall record every dataset's licence, the stricter-terms policy (no redistribution of data, derived files or weights) and the status of the licence enquiry to the data maintainers. | Must | v0.1 | The file exists before the first download. The licence position is stated, and is marked unresolved until a reply arrives. |
| FR-DOC-05 | `docs/validation/` shall document the MagNav.jl cross-check and explain any gap above 2×. | Should | v0.1 | The document exists if the oracle was run. If it was skipped (plan D6), the skip is stated. |
| FR-DOC-06 | Every literature number cited shall be verified against its primary source; unverifiable claims (including all 2025–2026 papers and industry claims) shall be marked or removed. | Must | v1.0 | A checklist in the write-up lists each citation with its verification status. "Thompson et al. 2019" never appears. |
| FR-DOC-07 | `CLAUDE.md` shall list the current build, lint and test commands. | Should | v0.1 | The commands listed run successfully. |

---

**Unverified items in this section:**
- The IMU grade parameter values.
- The exact count and naming of MagNav calibration segments and line IDs (pending S1 EDA).
- Whether the flight splits survive EDA. The plan marks them provisional.
- CLI subcommand names other than `version` and `reproduce`.
- The release of the tactical-grade sweep.

---

## 6. Non-functional requirements

These tables follow the approved project plan (`docs/superpowers/specs/2026-11-14-trinetra-project-plan-design.md`), which overrides the old PRD NFR-1 to NFR-7 where they differ. Two examples: the plan has no Jetson in v1 (R9), and it sets a 10 Hz MagNav rate rather than the PRD's 200 Hz IMU budget.

Releases map to milestones: **v0.1** is S4 (the minimum shippable portfolio), **v1.0** is S7, and **post** means after v1.0. A number marked *proposed* is this document's suggestion. The approved plan does not fix it, and the author should confirm it or drop it.

### NFR-PERF: Performance

| ID | Requirement | Priority | Release | Measure / verification method |
|---|---|---|---|---|
| NFR-PERF-01 | The system shall process MagNav data at 10 Hz with a real-time factor (RTF = wall time / data time) below 0.1 on a laptop CPU. This covers the full PF navigator with up to 5,000 particles plus compensation. | Must | v0.1 | Table T5 reports the measured RTF, with the CPU model, particle count and flight line. It comes from timing inside `reproduce`. |
| NFR-PERF-02 | Each learned compensator shall have at most 50k parameters, and inference shall take under 1 ms per sample on a CPU. | Must | v1.0 | Count parameters in a unit test. Benchmark the mean and p95 latency per sample on the test flights and report them in T5. |
| NFR-PERF-03 | Every learned model shall train on a CPU only, with the PyTorch CPU wheel and no GPU step. | Must | v1.0 | `uv.lock` contains the CPU wheel only. Training is run and timed on the author's laptop, and the time is logged in the run directory. |
| NFR-PERF-04 | The wall-clock runtime of the full `uv run trinetra reproduce` shall be measured and documented in the README. | Should | v0.1 | Report the time from the reproduce log. Set a target once S4 profiling exists; it is unknown today. |
| NFR-PERF-05 | Peak memory of `reproduce` shall fit on a typical developer laptop, and the README shall report it. | Should | v1.0 | Peak RSS is logged per run. Report only; no target is set (unverified). |

### NFR-ACC: Accuracy and consistency

| ID | Requirement | Priority | Release | Measure / verification method |
|---|---|---|---|---|
| NFR-ACC-01 | During simulated GPS outages of up to 30 minutes on both held-out test flights (1003, 1007), horizontal position error under B1 (INS + TL map matching) shall stay bounded and shall be lower than B0 (INS only). Neither the requirement nor the docs shall say "drift-free". | Must | v0.1 | Outage harness (spec §4): start times every 5 minutes, at least 20 INS seeds per start. Headline figure plus T2: median DRMS at 10/30/60 minutes with block-bootstrap 95% CIs, and paired differences between B1 and B0. |
| NFR-ACC-02 | The navigator shall be statistically consistent: mean NEES and mean NIS shall fall inside their χ² confidence bands for the headline configuration. | Must | v0.1 (B1), v1.0 (2×2) | Monte-Carlo consistency test, plus table T3. If a configuration falls outside its band, report it as a finding and do not hide it. |
| NFR-ACC-03 | The learned compensator's predicted variance shall be calibrated: empirical coverage of the 95% interval shall be within ±5 percentage points of 95%, and 68% coverage shall also be reported. | Must | v1.0 | PIT histogram and coverage table on validation flight 1005, and again on the test flights. |
| NFR-ACC-04 | Tolles-Lawson compensation shall reproduce MagNav.jl-level results on the same flight lines. On Mag 1 it shall reproduce `mag_1_c` to within a few nT. The cabin magnetometers shall come within a stated factor of MagNav.jl; a gap above 2× triggers a written explanation. | Must | v0.1 | Table T1 (RMSE in nT per flight and per magnetometer). Cross-check in `docs/validation/`. The size of "a few nT" is still to be fixed in S2 (unverified). |
| NFR-ACC-05 | The INS error simulator shall reproduce analytic Schuler-oscillation drift to within 10%. | Must | v0.1 | Monte-Carlo consistency test against the closed-form Pinson response. |
| NFR-ACC-06 | Each AI module's effect shall be stated as an end-to-end change in position error, with confidence intervals, and never only as a change in nT or validation loss. | Must | v1.0 | The 2×2 ablation in T2, with paired CIs on identical outages and seeds. |

### NFR-REPRO: Reproducibility

| ID | Requirement | Priority | Release | Measure / verification method |
|---|---|---|---|---|
| NFR-REPRO-01 | One sequence of commands shall regenerate every headline number and the headline figure from a fresh clone: `uv sync` → `scripts/download_data.py` → `uv run trinetra reproduce`. | Must | v0.1 (T2 + figure), v1.0 (T1–T3) | v1.0 exit check: a fresh clone on a second machine reproduces T1–T3. |
| NFR-REPRO-02 | Dependencies shall be pinned in a committed `uv.lock`, with Python pinned to 3.12 in `.python-version`. | Must | v0.1 (from S0) | Inspect the lockfile in the repo. `uv sync --locked` succeeds. |
| NFR-REPRO-03 | Every stochastic step shall take an explicit seed from config: INS error simulation, particle initialisation and resampling, NN initialisation and data ordering. | Must | v0.1 | Determinism test (NFR-REL-01). Seeds are recorded in each run directory. |
| NFR-REPRO-04 | Input data shall be identified by a committed SHA-256 manifest (`datasets/manifest.json`), and every run shall record the manifest hash. | Must | v0.1 (from S0) | Data test: checksums match the manifest. `metrics.json` contains the manifest hash. |
| NFR-REPRO-05 | Each run shall write `runs/<date>_<name>/` containing the resolved config, git SHA, data-manifest hash, `metrics.json` and figures. | Must | v0.1 | Integration test on a synthetic run checks the directory contents. |
| NFR-REPRO-06 | Every result quoted in the README shall point to a tagged commit (one tag per milestone). | Should | v0.1 | Review at release: each README number links to a tag. |

### NFR-REL: Reliability

| ID | Requirement | Priority | Release | Measure / verification method |
|---|---|---|---|---|
| NFR-REL-01 | Runs shall be deterministic: the same code, data, config and seed give bit-identical `metrics.json` on the same platform. | Must | v0.1 | Regression test runs a short configuration twice and compares the outputs. Cross-platform equality is not required; cross-platform agreement is to a tolerance (NFR-PORT-01). |
| NFR-REL-02 | The data download shall be idempotent and safe: files already verified are skipped; partial downloads are never promoted (`.part` then verify then rename); a checksum mismatch raises an error naming the file and leaves no file behind; a manifest mismatch fails loudly and never overwrites. | Must | v0.1 (from S0) | Unit tests from the S0 plan, Task 3 (`test_partial_download_not_promoted`, `test_md5_mismatch_raises_and_leaves_no_file`, `test_existing_verified_file_is_skipped`, `test_manifest_mismatch_raises`). |
| NFR-REL-03 | The navigator shall detect and flag divergence. Flags shall cover N_eff collapse, a multi-modal particle cloud and NIS-gate rejections. The run shall continue and record the flags, and shall never crash or emit NaNs silently. | Must | v0.1 | Fault-injection tests: synthetic map bias, a spike, a flat-map region. Assert that the flags are raised and the outputs are finite. |
| NFR-REL-04 | Divergence rate (horizontal error > 500 m) shall be computed for every configuration and reported. A rate above 20% triggers the risk-register mitigation. | Must | v0.1 | Outage harness metric, reported in T2/T3. |
| NFR-REL-05 | Map lookups shall respect validity: cells filled by k-NN interpolation report as invalid, and the PF shall not take map updates from invalid cells. | Must | v0.1 | Unit test on a synthetic map with masked cells. |
| NFR-REL-06 | Gaps in the input shall be masked and never interpolated across. | Must | v0.1 | Unit test: a known input with a gap produces a masked span on the output grid. |

### NFR-MAINT: Maintainability

| ID | Requirement | Priority | Release | Measure / verification method |
|---|---|---|---|---|
| NFR-MAINT-01 | `trinetra.core` shall import only numpy, scipy, the standard library and `trinetra.core`. It shall never import torch, h5py, rclpy, pandas, `trinetra.adapters`, `trinetra.compensation` or a dataset name. | Must | v0.1 (from S0) | AST-based architecture test (`tests/architecture/test_core_imports.py`). The S0 exit criterion requires it to catch a deliberate violation. |
| NFR-MAINT-02 | Code shall pass pyright, in strict mode for `core/` and standard mode elsewhere. | Must | v0.1 | `uv run pyright` reports 0 errors. Enforced in CI from S3. |
| NFR-MAINT-03 | Code shall pass `ruff check` and `ruff format --check` with the configured rule set. | Must | v0.1 | `uv run ruff check .` reports 0 issues. Enforced in CI from S3. |
| NFR-MAINT-04 | Each significant design decision shall be recorded as an ADR in `docs/adr/` (0001–0003 at S0, then numbered from 0004). The data split and the ESKF-cascade ablation each need one. | Must | v0.1 | Review at release: ADRs exist for R1, R7/R8, R13 and the splits. |
| NFR-MAINT-05 | Interfaces (`Compensator`, `MapModel`, `Navigator`, `RModel`) shall be `typing.Protocol`s, introduced only once a second implementation exists. | Should | v1.0 | Code review. |
| NFR-MAINT-06 | Source modules shall stay small enough to review. *Proposed:* a file over about 400 lines triggers a refactor review. | Could | v1.0 | Line-count check in review (proposed threshold). |

### NFR-PORT: Portability

| ID | Requirement | Priority | Release | Measure / verification method |
|---|---|---|---|---|
| NFR-PORT-01 | The system shall run on Windows (development) and Linux (CI), and the headline metrics shall agree across both to a stated tolerance. | Must | v0.1 (Windows), v1.0 (both) | Data-free tests run on the Linux CI runner. The v1.0 second-machine reproduction checks agreement; the tolerance is set then (unverified). |
| NFR-PORT-02 | The project shall target Python 3.12 only (`>=3.12,<3.13`). | Must | v0.1 | `pyproject.toml` and `.python-version`. |
| NFR-PORT-03 | No step shall need a GPU, CUDA, Julia, ROS 2, Docker or a Jetson. MagNav.jl is an optional offline oracle and never a runtime dependency. | Must | v1.0 | A fresh-clone reproduction on a CPU-only machine with no Julia installed. |
| NFR-PORT-04 | Paths shall work in directories that contain spaces, as the author's repo path does. | Should | v0.1 | Tests use `tmp_path` and `pathlib`. Manual run in the author's path. |

### NFR-SEC: Security, licence and privacy

| ID | Requirement | Priority | Release | Measure / verification method |
|---|---|---|---|---|
| NFR-SEC-01 | The repository shall never contain raw MagNav data, derived Parquet, map tiles or trained weights. | Must | v0.1 (from S0) | `data/` and `runs/` are in `.gitignore`. Release check: `git ls-files` lists no `.h5`, `.parquet`, `.pt` or map files. |
| NFR-SEC-02 | Use of the data shall follow the stricter of the two conflicting licence terms (see Data governance), and each dataset's licence shall be recorded in `datasets/README.md`. | Must | v0.1 | `datasets/README.md` and ADR-0003 are present and current. |
| NFR-SEC-03 | The repository shall contain no secrets, tokens or credentials. The data download needs none. | Must | v0.1 | Manual scan before each release. *Proposed:* add a secret-scanning step when CI arrives. |
| NFR-SEC-04 | Published material shall contain no personal data of the dataset providers or aircrew. Publish aggregates and figures only, with no record-level data. | Must | v0.1 | Release review of `docs/`, README and figures. |
| NFR-SEC-05 | Nothing in the repository shall imply affiliation with, or endorsement by, any company or product, including any product with a similar name. | Must | v0.1 | Wording review of README, docs and repo description at each release. |

### NFR-USAB: Usability

| ID | Requirement | Priority | Release | Measure / verification method |
|---|---|---|---|---|
| NFR-USAB-01 | The README shall let a stranger go from clone to the headline table using only the documented commands, with the time-to-first-result stated. | Must | v0.1 | A timed fresh-clone walkthrough recorded at release. The phase-13 notes aim for under 10 minutes excluding the download; whether that is achievable is unverified. |
| NFR-USAB-02 | A reviewer shall be able to find the headline figure and result within 2 minutes of opening the repo. | Must | v0.1 | Headline figure and T2 sit above the fold in the README. Checked by an informal timed review. |
| NFR-USAB-03 | User-facing failures shall give actionable messages and documented exit codes: download exits 0 on success, 1 on `FetchError` naming the file, 2 for a network error; `reproduce` explains a missing-data state. | Must | v0.1 | CLI unit tests on messages and exit codes. |
| NFR-USAB-04 | Public functions shall have NumPy-style docstrings with units (nT, m, rad, s) stated. | Should | v1.0 | Review. *Proposed:* ruff docstring rules on `core/`. |
| NFR-USAB-05 | `docs/validation/` shall explain each comparison with MagNav.jl and each known limitation. | Should | v1.0 | Present at release. |

### NFR-HONEST: Honest claims

| ID | Requirement | Priority | Release | Measure / verification method |
|---|---|---|---|---|
| NFR-HONEST-01 | Documentation shall say magnetic aiding **bounds** drift, and shall never say "drift-free" or similar. | Must | v0.1 | Release check: grep the repo for "drift-free" and "drift free" and get no hits outside quoted corrections. |
| NFR-HONEST-02 | Quantum wording shall follow the policy text: MagNav's scalar magnetometers are optically pumped caesium (atomic) sensors; this project adds no quantum hardware; a sensor-noise sweep shows where better sensors would matter. | Must | v0.1 (wording), v1.0 (sweep) | Wording review. Sensor-noise sweep in S7. |
| NFR-HONEST-03 | Negative or mixed results, including an AI module that loses its ablation, shall be reported in the README as findings and not removed. | Must | v1.0 | The ablation design is fixed before the test flights are run. T2 shows every 2×2 cell. |
| NFR-HONEST-04 | No result shall come from a selected best seed, from tuning on test flights, or from a single flight. The test flights are touched once per frozen configuration. | Must | v0.1 | The split ADR plus a test-access log (config hash and date) in the run directories. |
| NFR-HONEST-05 | Every literature number and industry claim shall be checked against its primary source before it is cited. Hypothetical citations shall never be cited. | Must | v1.0 | Citation checklist in `docs/`. S7 exit criterion: "literature numbers verified". |
| NFR-HONEST-06 | Known leakage paths shall be excluded and documented: IGRF evaluated at the GPS position, map value used as an input or target, latitude/longitude/time used as NN inputs, GPS fused during evaluation. | Must | v0.1 | ADR-0002 excluded-input list. Unit test asserting the NN feature list contains none of these. |

---

## 7. Success metrics and KPIs

### Product metrics

| Metric | Definition | How measured | Target |
|---|---|---|---|
| Horizontal error vs time since GPS loss (headline) | Median and 5–95% band of horizontal error across all outages and seeds, for B0, B1, +NN and +NN+learned R, with the filter's 1σ envelope | Outage harness: 10/30/60-minute outages, starts every 5 minutes, at least 20 seeds | B1 below B0 and bounded through 30-minute outages on both test flights (v0.1). No numeric target is set in advance. |
| DRMS at 10/30/60 minutes | Distance RMS of horizontal error at a fixed time after GPS loss | T2, block-bootstrap 95% CIs, paired differences | Report only, with CIs. The direction of the B1 vs B0 difference is the criterion. |
| CEP50 / CEP95, ATE, final error, drift (m/min and % of distance) | Standard position-error statistics over each outage | T2 / supplementary | Report only |
| Compensation RMSE | RMSE (nT) of compensated anomaly against the Mag 1 reference (`mag_1_c` − diurnal − IGRF), per line and per magnetometer | T1 | Mag 1 TL within a few nT of `mag_1_c`. Cabin magnetometers within a stated factor of MagNav.jl; above 2× needs an explanation. AI-1 beats ridge-TL on validation. |
| Filter consistency | Mean NEES and mean NIS compared with χ² bands | T3, Monte-Carlo | Inside the bands for headline configurations. Report all of them. |
| Calibration coverage | Empirical 68%/95% coverage and PIT histogram of the compensator's variance | Validation and test flights | 95% coverage within ±5 points |
| Divergence rate | Fraction of outage runs with horizontal error > 500 m | Outage harness | Report only. Above 20% triggers mitigation. |
| Real-time factor | Processing time / data time at 10 Hz | T5 | < 0.1 on a laptop CPU |
| NN latency and size | Inference time per sample; parameter count | Benchmark plus unit test | < 1 ms per sample; ≤ 50k parameters |

### Portfolio and process metrics

| Metric | Definition | How measured | Target |
|---|---|---|---|
| Clone-to-table time | Wall time from `git clone` to the T2 table, split into download time and compute time | Timed fresh-clone run at each release | Report only. A target is set after S4 profiling (unverified). |
| Headline discoverability | Time for a first-time reader to find the headline figure and its one-sentence claim | Informal timed review by 1–2 people | < 2 minutes |
| Second-machine reproduction | Whether T1–T3 reproduce on another machine within tolerance | v1.0 exit check | Yes |
| Test counts by layer | Number of unit, property, consistency, data and regression tests | `pytest --collect-only` per marker or directory | Report only, with every layer non-empty by v0.1 |
| `core/` coverage | Line and branch coverage of `src/trinetra/core` from data-free tests | pytest-cov, added when CI arrives | Report only. *Proposed:* aim high and do not gate on it in v0.1. |
| Milestone timeliness | Actual vs planned week for each slice's tag (S0–S7, at 8 h/week) | Tag dates against the plan | v0.1 at about week 17 and v1.0 at about week 29. Report slips; slices are independent. |
| Infrastructure discipline | Infrastructure added without its trigger firing | Review against spec §5 | Zero |

---

## 8. Quality assurance and acceptance testing

### Test layers

| Layer | Location | What it proves | Data needed |
|---|---|---|---|
| **Architecture** | `tests/architecture` | The hexagonal import rule holds. `core/` imports no frameworks, dataset readers or PyTorch, including aliased and `from` imports. | None |
| **Unit** | `tests/unit` | Mathematics matches hand-computed values: Somigliana gravity, WGS-84 radii, transport rate, the 18-term TL design matrix, bicubic interpolation and gradient on an analytic field, upward continuation of a known field, metrics. Download safety (S0 Task 3). CLI exit codes. | None |
| **Property** | `tests/property` (hypothesis) | Invariants over random inputs: P stays symmetric positive-definite; a zero-information update changes nothing; an update never increases tr(P); particle weights stay normalised and finite; resampling preserves the particle count. | None |
| **Monte-Carlo consistency** | `tests/consistency` | Estimators are statistically honest: NEES and NIS fall in χ² bands on synthetic truth; the Pinson simulator reproduces Schuler drift within 10%; injected faults trip the NIS gate. | None (synthetic) |
| **Data** | `tests/data` (skipped when data is absent) | Real files are what we think they are: checksums match the manifest; `mag_1_dc − mag_1_igrf = igrf` within 0.01 nT; adapter round-trip; monotonic time; gaps masked; written verdict on whether `ins_*` is free-inertial. | MagNav, local only |
| **Regression** | `tests/regression` | Headline numbers do not change silently: ATE and DRMS on a frozen short configuration stay within tolerance of recorded values; determinism (NFR-REL-01). | Synthetic in CI; real data run locally |

Unit, property, architecture and consistency tests never touch the network or real data. Network tests carry the `network` marker and are deselected by default.

### Release acceptance checklist: v0.1 (S4)

- [ ] All test layers pass locally. Data tests pass with the data present.
- [ ] ruff, ruff format and pyright pass with 0 issues, and CI is green on `main`.
- [ ] The architecture test passes, and it was shown to catch a deliberate violation.
- [ ] `uv run trinetra reproduce` regenerates T2 and the headline figure from a clean `runs/`.
- [ ] B1 is bounded and below B0 through 30-minute outages on **both** test flights (1003, 1007), with CIs.
- [ ] NEES/NIS (T3) and divergence rate are reported, including any results outside the bands.
- [ ] T1 is compared with MagNav.jl, and any gap above 2× is explained in `docs/validation/`.
- [ ] RTF < 0.1 is reported in T5 with the hardware stated.
- [ ] Each run directory contains config, git SHA, manifest hash, seeds and metrics.
- [ ] `git ls-files` shows no data, derived files or weights. `datasets/README.md` shows the current licence status.
- [ ] README: headline above the fold; Reproduce section; quantum wording per policy; no "drift-free"; no affiliation implied.
- [ ] ADRs exist for the PF navigator, truth definitions, data licence and splits.
- [ ] The test flights were touched once per frozen configuration (access log checked).
- [ ] Tag `v0.1`.

### Release acceptance checklist: v1.0 (S7)

- [ ] Everything in v0.1 still passes.
- [ ] The full 2×2 ablation plus reference rows (INS only, Stinger Mag 1 bound, PF-spread R, Sage-Husa R) are in T2, with paired CIs, at least 20 INS seeds and at least 5 NN seeds.
- [ ] AI-1: ≤ 50k parameters, < 1 ms per sample, 95% coverage within ±5 points, and nT compared against ridge-TL. The end-to-end effect is reported whatever its sign.
- [ ] AI-2: mean NIS inside the χ² band, compared with fixed, PF-spread and Sage-Husa R.
- [ ] Error budget (T4), sensor-noise ("quantum") sweep, and runtime (T5) are complete.
- [ ] Every literature number is verified against its primary source, and the citation checklist is complete.
- [ ] A fresh clone on a second machine (Linux or another Windows box) reproduces T1–T3 within tolerance.
- [ ] Negative and mixed results appear in the README as findings.
- [ ] Tag `v1.0`.

### CI policy

- **Before S3:** no CI. Checks run locally (`uv run pytest; uv run ruff check .; uv run ruff format --check .; uv run pyright`), as spec §5 sets out.
- **Trigger:** the first filter consistency test, planned in S3. At that point add GitHub Actions on Linux with Python 3.12 via uv and `uv sync --locked`. It runs ruff, ruff format check, pyright, and the data-free tests (architecture, unit, property, consistency, synthetic regression).
- **Never in CI:** data download, data tests, or anything that needs MagNav files. The licence and the size rule them out. Data tests run locally before each release.
- **Gate:** `main` must be green before a milestone tag. Coverage is reported, not gated (*proposed*).
- **Later:** Docker builds join CI only when ROS 2 arrives after v1. DVC, MLflow-in-CI and Hydra are not planned.

---

## 9. Data governance

**Licence policy.**
- MagNav (Zenodo record 12723700) has conflicting terms. Zenodo lists "Other (Attribution)" and US-government public domain. The MagNav.jl Data Sharing Agreement says non-profit research only, no redistribution, a one-year term, and publication links sent to the Air Force.
- The project follows the **stricter** terms until the data owner rules otherwise. The enquiry goes to magnav-admins@mit.edu (D7), and its status ("pending", "sent <date>", the reply) is recorded in `datasets/README.md` and ADR-0003.
- If there is no reply by S4, publish the code and method and state the restriction clearly.
- The interpretation of the agreement's terms (for example, whether a public portfolio counts as non-profit research use) is unverified until the owner replies.

**Manifest.**
- `datasets/manifest.json` is the only committed record of the data. It holds, per file: Zenodo source, key, size, MD5 and SHA-256.
- `scripts/download_data.py` checks the Zenodo record against it before downloading, verifies every file, and refuses to overwrite it silently.
- Each run records the manifest hash. A changed upstream record is a hard failure, and it needs a deliberate, reviewed manifest update. The MagNav.jl map artifact is covered by the same mechanism.

**What may be published.**

| Allowed | Not allowed |
|---|---|
| Source code, configs, tests | Raw HDF5, maps or map tiles |
| Field names (inventory, no values) | Derived Parquet or any per-sample export |
| Aggregate metrics and tables (T1–T5) | Trained weights or checkpoints (they may encode the data) |
| Figures of errors, distributions and maps at publication scale | Record-level samples in notebooks, fixtures or issues |
| Checksum manifest, licence notes, ADRs | Anything implying endorsement by the data owner or any company |

Test fixtures must be synthetic or hand-written, never cut from real files. Whether map figures count as redistribution is unverified. Keep them coarse and attributed until the owner replies.

**Retention.**
- Raw data lives only in the gitignored `data/raw/magnav/` on the author's machine, and stays there for the licence term.
- If the one-year term in the data-sharing agreement applies, delete local raw and derived data at its end unless it is renewed. The start date is recorded in `datasets/README.md` (unverified whether the term binds Zenodo downloads).
- `runs/` is local and disposable. Only the numbers quoted in the README are kept, as tagged commits plus `metrics.json` content copied into docs.
- Weights are regenerated from seeds and are not archived publicly.

---

## 10. Release plan

Every release is a git tag on `main` that passes its exit gate. Week numbers count from the start of S0. If work starts on Monday 2026-11-16 and there are no breaks, week 17 ends about 2027-03-14 and week 29 about 2027-06-06. These calendar dates are **indicative only**, because no deadline has been set (D9).

| Release | Weeks | Slices | Contents | Exit / acceptance gate | Tag |
|---|---|---|---|---|---|
| **S0 Setup** | 1–2 | S0 | Python 3.12 via uv with `uv.lock`; src layout; ruff, pytest, hypothesis, pyright; core import-rule test; `scripts/download_data.py` checking a SHA-256 manifest; ADR-0001 to 0003; README "Reproduce" stub; licence email sent | `pytest`, `ruff check`, `ruff format --check` and `pyright` all pass; the import-rule test catches a deliberate violation; no data is tracked | `s0-setup` |
| Data checkpoint | 3–5 | S1 | MagNav adapter → Parquet; map loader; line segmentation; field inventory; verdict on whether `ins_*` is free-inertial; anomaly-floor table; splits ADR | IGRF identity holds within 0.01 nT; round-trip test passes; written `ins_*` verdict; splits frozen in an ADR | `s1-data` (name proposed) |
| Compensation checkpoint | 6–8 | S2 | 18-term TL and ridge-TL; table T1 (nT) from one command; optional MagNav.jl cross-check | TL on Mag 1 reproduces `mag_1_c` to within a few nT; cabin magnetometers within a stated factor of MagNav.jl | `s2-compensation` (name proposed) |
| INS checkpoint | 9–11 | S3 | Pinson INS-error simulator; outage harness; GitHub Actions CI; B0 curves | Simulator matches analytic drift within 10%; B0 curves exist for both test flights; CI passes | `s3-ins` (name proposed) |
| **v0.1, the minimum shippable portfolio** ★ | 12–17 | S4 | PF navigator; map model with upward continuation and a filled-cell mask; NIS gate and fault injection; B1 vs B0; README with the headline figure and the reproduce command | Error stays bounded through 30-minute outages on both test flights (1003, 1007); NEES/NIS and divergence rate reported; `uv run trinetra reproduce` regenerates T2 and the headline figure | `v0.1` |
| v0.2 (optional) | 18–22 | S5 | AI-1: residual NN ensemble on top of TL; calibration; end-to-end ablation | Beats ridge-TL in nT on validation; 95% coverage within ±5 points; under 1 ms per sample; the end-to-end result is reported whatever its sign | `v0.2` (proposed) |
| v0.3 (optional) | 23–26 | S6 | AI-2: learned R compared with fixed, PF-spread and Sage-Husa R; full 2×2 | Mean NIS inside the χ² band; paired CIs in T2 | `v0.3` (proposed) |
| **v1.0** | 27–29 | S7 | Error budget (T4); runtime/RTF (T5); sensor-noise ("quantum") sweep; final README; literature numbers checked against primary sources | A fresh clone on a second machine reproduces T1–T3 with one command | `v1.0` |

**Post-v1 roadmap.** All effort figures are rough, **unverified** estimates. Order follows value to the portfolio.

| Release | Contents | Gate | Est. effort |
|---|---|---|---|
| v1.1 | 15-state ESKF (local-level NED, Joseph form) for the pre-outage solution; **cascade ablation** (PF fix → ESKF) to measure double-counting, documented in an ADR | NEES within bands with GPS on; cascade vs PF-navigator difference reported with CIs | 5–7 wk |
| v1.2 | FOGM compensation-error state in the PF, replacing R inflation | NIS improves or stays in band; ATE change reported | 2–3 wk |
| v1.3 | Tight-coupled EKF (H = ∇map) as an alternative navigator | Compared with the PF on the same outages | 3–4 wk |
| v1.4 | Tactical-grade IMU sweep | Divergence rate and ATE reported for each grade | 1–2 wk |
| v1.5 | 2021 MagNav flights as extra test data (availability and licence **unverified**) | Adapter covers them; T2 extended | 2–3 wk |
| v2.0 | ROS 2 (Humble or Jazzy) adapter and C++ port of the filter core; Docker | Bag replay matches Python results within tolerance | 8+ wk |
| v2.1 | Jetson deployment | RTF ≤ 1 on device; accuracy parity check | 3–4 wk (needs hardware) |

## 11. Milestone schedule (8 h/week, CPU only)

**Budget:** 29 weeks × 8 h is about 232 h. S0 16 h; S1 24 h; S2 24 h; S3 24 h; S4 48 h; S5 40 h; S6 32 h; S7 24 h.

```
Week            12345678901234567890123456789
S0 Setup        ##...........................
S1 Data+EDA     ..###........................
S2 TL comp      .....###.....................
S3 INS+harness  ........###..................
S4 MagNav       ...........######............   <- v0.1 at end of week 17 (minimum shippable)
S5 AI-1         .................=====.......
S6 AI-2         ......................====...
S7 Write-up     ..........................###   <- v1.0 at end of week 29
# = critical path (mandatory)    = = cuttable (see scope-cut ladder)
```

**Critical path:** S0 → S1 → (S2 ∥ S3) → S4 → S7. S2 and S3 do not depend on each other, but one author works them in sequence, so if S2 is blocked (for example by a MagNav.jl mismatch) S3 can be pulled forward. With a solo author there is no slack: any slip before week 17 delays v0.1 week for week. S5 and S6 sit on the path to v1.0 but not to v0.1, and they are the first things to cut.

| Wk | Slice | Focus | Weekly checkpoint |
|---|---|---|---|
| 1 | S0 | uv + Python 3.12, `pyproject.toml`, linters, import-rule test | Licence email sent (D7) |
| 2 | S0 | Download script + manifest; ADR-0001 to 0003; first download inspected with h5py | **tag `s0-setup`** |
| 3 | S1 | Field inventory; MagNav adapter → Parquet | IGRF identity test passes |
| 4 | S1 | Map loader (grid metadata, filled-cell mask); line segmentation; gap masking | Round-trip test |
| 5 | S1 | `ins_*` free-inertial test; anomaly-floor table; splits ADR | `ins_*` verdict written |
| 6 | S2 | TL design matrix (18 terms), unit-tested; fit on calibration segments | Design-matrix tests pass |
| 7 | S2 | Ridge-TL, band-pass; Mag 1 vs `mag_1_c` | Within a few nT |
| 8 | S2 | T1 from one command; MagNav.jl cross-check (optional, D6) | T1 committed (aggregates only) |
| 9 | S3 | Earth model (WGS-84, Somigliana gravity, transport rate); Pinson dynamics | Unit tests vs hand values |
| 10 | S3 | Monte-Carlo seeds; Schuler-drift check; outage harness | Drift within 10% |
| 11 | S3 | B0 curves on 1003/1007; GitHub Actions | CI passes |
| 12 | S4 | Map model: bicubic interpolation, gradient, validity mask | Analytic-field tests |
| 13 | S4 | FFT upward continuation | Continuation tests |
| 14 | S4 | PF navigator: ~5 states, systematic resampling, N_eff | Runs on validation flight 1005 |
| 15 | S4 | NIS gate, divergence flags, fault injection; B1 tuning on 1005 only | Config frozen |
| 16 | S4 | Single test-set run; T2, T3, headline figure | Bounded 30-min error |
| 17 | S4 | Reproduce command, README, buffer | ★ **tag `v0.1`** |
| 18 | S5 | Shared feature function; residual target; excluded-input test | Parity test |
| 19 | S5 | Ensemble (5 × ≤50k params), β-NLL, CPU training | Trains in under 1 h (target, **unverified**) |
| 20 | S5 | PIT and coverage calibration; latency | Coverage ±5 pts |
| 21 | S5 | NN compensator inside the navigator; ablation on 1005 | End-to-end delta |
| 22 | S5 | Frozen test run; write-up of the result | tag `v0.2` (proposed) |
| 23 | S6 | `RModel`: fixed, PF-spread, Sage-Husa | Classical rows done |
| 24 | S6 | Learned R from innovations and map informativeness | Model trains |
| 25 | S6 | Full 2×2 on validation | NIS in band |
| 26 | S6 | Frozen test 2×2; paired CIs | tag `v0.3` (proposed) |
| 27 | S7 | Error budget (T4); RTF at 10 Hz (T5) | Tables generated |
| 28 | S7 | Sensor-noise sweep; check every cited number | Citations checked |
| 29 | S7 | Fresh-clone reproduction on a second machine; final README | **tag `v1.0`** |

**Scope-cut ladder.** If the deadline arrives early, apply the cuts in this order. Each rung keeps a coherent and honest story.

1. **Drop the MagNav.jl oracle (D6).** Cite its published numbers instead, marked "not reproduced".
2. **Drop learned R (S6, about 4 wk).** Keep the classical R comparison (fixed vs PF-spread vs Sage-Husa) as a one-week add-on to S4 or S7. The 2×2 collapses to TL vs TL+NN.
3. **Trim S7 (about 1 wk).** Drop the sensor-noise sweep and the full error budget. Keep the final README and the fresh-clone check.
4. **Drop AI-1 (S5, about 5 wk).** Ship ridge-TL as the compensator. The portfolio becomes "a rigorous classical magnetic-navigation baseline". This is still above the v0.1 floor.
5. **Shrink S4 evaluation.** Use 10 Monte-Carlo seeds instead of 20, drop the 60-minute outages, and drop synthetic fault injection while keeping the NIS gate.
6. **Floor (deadline before week 17).** Ship a pre-release tag with T1 (TL compensation) plus B0 INS-drift curves and the reproduce command, labelled clearly as incomplete. Never ship a PF result that has not been evaluated on both test flights.

## 12. Dependencies

**External**

| Dependency | Used in | Notes / status |
|---|---|---|
| MagNav flight data, Zenodo record 12723700 | S0 onward | About 1.2 GB; checked against per-file MD5 and a committed SHA-256 manifest. Never redistributed |
| MagNav.jl map artifact | S1, S4 | Fetched by the download script; format and grid spacing **unverified** |
| magnav-admins@mit.edu licence reply | before v0.1 publication | Outstanding; see RISK-01 |
| MagNav.jl + Julia | S2–S3 (optional oracle) | Never a runtime dependency; skip if setup takes more than 2 h (D6) |
| PyTorch CPU wheel for cp312 on Windows | S5 | Expected to exist; confirm with a spike before S5 |
| uv, Python 3.12, numpy, scipy, h5py, pyarrow | S0 onward | Pinned in `uv.lock` |
| IGRF model coefficients and a Python implementation | S1, S4 | Needed to evaluate IGRF at the *estimated* position (R8). Package choice and IGRF generation **unverified** |
| Base-station diurnal data (in MagNav) | S1, S4 | Assumed present for the test flights; **unverified** |
| GitHub + GitHub Actions (free tier) | S3 onward | CI covers data-free tests only |
| Primary literature (Canciani & Raquet 2016; Gnadt et al.; Arulampalam; Solà) | S2–S7 | Numbers must be checked before citing |

**Internal (between slices)**

| Slice | Needs | Provides |
|---|---|---|
| S1 | S0 download, manifest, import rule | Parquet, map loader, splits, `ins_*` verdict |
| S2 | S1 Parquet, calibration segments | `Compensator` (TL, ridge-TL), anomaly in nT with variance |
| S3 | S1 GPS truth trajectory | INS-error simulator, outage harness, B0 |
| S4 | S2 compensator, S3 simulator/harness, S1 maps | PF navigator, B1, T2/T3, v0.1 |
| S5 | S2 (residual target), S4 (end-to-end evaluation) | TL+NN compensator |
| S6 | S4 innovations/NIS; S5 for the full 2×2 (can start with TL only) | `RModel` implementations |
| S7 | S4, plus S5 and S6 if not cut | T4, T5, v1.0 |
| Post-v1 ESKF | S3 Earth model, S4 harness | Cascade ablation |

## 13. Assumptions and constraints

| # | Assumption | How we will validate it | When |
|---|---|---|---|
| A1 | The `ins_*` fields may be GPS-aided, so the simulated INS is the primary baseline | Free-inertial test: compare `ins_*` position with GPS over long segments. Unbounded growth with Schuler oscillation means free-inertial; bounded error means aided | S1, wk 5 |
| A2 | At least 2 usable test flights (1003, 1007) with enough map coverage | EDA: per-line map coverage, gaps, at least 60 s of valid map before each outage start | S1 |
| A3 | h5py reads the MagNav files directly, without Julia-specific encodings | First download inspected with h5py; field inventory committed | S0 wk 2 |
| A4 | Map grid spacing, reference altitude and datum are suitable after upward continuation (**unverified**) | Read the map metadata; compare map altitude with flight altitude (downward continuation is unstable) | S1 wk 4 |
| A5 | A base-station diurnal record exists for every test flight | Field inventory | S1 |
| A6 | Calibration boxes exist in Flt1006 and Flt1002 | Line segmentation in EDA | S1 |
| A7 | Flights 1003/1007 are the literature's comparison flights | Check against Gnadt et al. and MagNav.jl docs | S1 |
| A8 | Navigation-grade IMU parameters can be sourced and cited (**unverified**) | Take them from a primary source; reproduce the analytic Schuler drift | S3 |
| A9 | `mag_1_c` minus diurnal minus IGRF is a usable compensation reference | TL on Mag 1 reproduces `mag_1_c` | S2 |
| A10 | CPU is enough for ≥20 seeds × all outage starts × 1–5k particles | Time one outage in S3/S4; extrapolate total compute; RTF at 10 Hz | S4 wk 14 |
| A11 | A PyTorch CPU wheel is available for Python 3.12 on Windows | `uv add torch` spike in a branch | Before S5 |
| A12 | The licence permits publishing code, figures and aggregate metrics | Licence reply; until then follow the stricter terms (R13) | S0 onward |
| A13 | The author can sustain 8 h/week | Weekly hours log; re-plan if 3 weeks in a row fall below 6 h | Ongoing |

**Constraints:**
- Solo author, 8 h/week, no GPU. Models have at most 50k parameters.
- Development is on Windows. The repository path contains spaces.
- Never commit raw data, derived files or trained weights. Publish aggregates and figures only.
- `core/` stays numpy/scipy only, enforced by a test.
- Infrastructure is added only on its trigger (spec §5).
- The test flights are touched once per frozen configuration. No tuning on test data, no best-seed picking, and GPS is never fused during evaluation.
- Honest-claims policy: magnetic aiding bounds drift and is never described as "drift-free". Negative results are reported. The project never implies affiliation with any company or product.

## 14. Risk register

Owner is the author for every risk, because this is a solo project. The owner column names the role.

| ID | Risk | L | I | Mitigation | Trigger | Owner |
|---|---|---|---|---|---|---|
| RISK-01 | The licence forbids public portfolio use | M | H | Email early (D7); publish aggregates only; never redistribute data | No reply by start of S4 → publish code and method with the restriction stated | Author (data steward) |
| RISK-02 | `ins_*` is GPS-aided | H | M | Simulated INS is the primary baseline anyway | S1 verdict | Author |
| RISK-03 | Fewer than two usable test flights | M | H | Split by line; leave-one-flight-out; 2021 flights (**unverified**) | S1 EDA | Author |
| RISK-04 | PF diverges over flat or filled map areas | H | M | Filled-cell mask; N_eff and multiple-peak flags; report divergence rate | More than 20% of runs diverge | Author |
| RISK-05 | A gain in nT does not reduce ATE | M | M | Ablation fixed in advance; report honestly (D8) | S5 end-to-end delta CI includes 0 | Author |
| RISK-06 | Leakage (map, GPS, IGRF at true position, test tuning) | M | H | Excluded-input test; R8 variants; ADR-0002 review; test set touched once | Any review finding, or test result much better than validation | Author (reviewer role) |
| RISK-07 | Cannot match MagNav.jl numbers | M | M | Same lines and terms; explain gaps in `docs/validation` | Gap above 2× | Author |
| RISK-08 | Infrastructure creep | M | H | Trigger table; "Never" list | Any infrastructure work not tied to a trigger | Author |
| RISK-09 | Schedule slip | H | M | Independent slices; scope-cut ladder; S4 is the shippable point | Two consecutive weeks miss their checkpoint | Author (PM role) |
| RISK-10 | Map resolution or altitude mismatch (would need downward continuation) | M | H | Upward continuation only; restrict outages to compatible lines; state in error budget | S1 map metadata check | Author |
| RISK-11 | Monte-Carlo compute exceeds CPU budget | M | M | Vectorised PF; fewer particles or seeds (ladder rung 5); overnight runs | One full T2 run takes more than 24 h (threshold **unverified**) | Author |
| RISK-12 | Environment breakage (uv, Windows, torch cp312) | L | M | Lockfile; early torch spike; CI on Linux | Spike fails | Author |
| RISK-13 | Wrong or unverifiable citations damage credibility | M | H | Verify every number against its primary source; never cite "Thompson et al. 2019" | Any 2025–26 or industry claim without a primary source | Author |
| RISK-14 | Zenodo record changes or becomes unavailable | L | H | `ManifestMismatchError`; keep a private local copy; record the version | Manifest mismatch | Author |

## 15. Open questions and decisions log

**Decisions taken** (from spec §9; approved 2026-11-14 according to CLAUDE.md)

| # | Decision | Status |
|---|---|---|
| D1 | The PF over INS-error states is the navigator. The ESKF→PF cascade becomes a measured ablation (ADR-0001) | Approved; ADR-0001 is written in S0 |
| D2 | Replace the 16-week plan with the ~29-week plan at 8 h/week | Approved; CLAUDE.md updated |
| D3 | Defer the ESKF to after v1.0; v0.1 at about week 17 | Approved |
| D4 | Python 3.12 + uv; delete the broken `.venv` | Approved; carried out in S0 |
| D5 | Navigation-grade IMU for the headline; tactical grade as a sweep | Approved; grade specs **unverified** |
| D6 | Install Julia once for the MagNav.jl oracle | Optional; skip if it takes more than 2 h. Not yet attempted |
| D7 | Send the licence email this week | Approved; send is pending (S0 task) |
| D8 | Accept a negative or mixed AI result in the README | Approved |
| D9 | Deadline | **Open** |
| D10 | Drop Jetson and RoNIN from v1 | Approved |

**Open questions**

| # | Question | Needed by | Owner |
|---|---|---|---|
| Q1 | What is the deadline? It decides how far down the scope-cut ladder to go | S4 start | Author |
| Q2 | What does the licence reply say about publishing figures, metrics and derived artifacts? | S4 publication | magnav-admins |
| Q3 | Is `ins_*` free-inertial? | S1 wk 5 | Author |
| Q4 | Map grid spacing, altitude, datum, and which map covers which flight; draped or level survey? | S1 wk 4 | Author |
| Q5 | Which IGRF implementation and generation to use, and how is it validated? | S1 | Author |
| Q6 | Is base-station diurnal available for 1003 and 1007? | S1 | Author |
| Q7 | Which primary source supplies the navigation- and tactical-grade IMU parameters? | S3 | Author |
| Q8 | Do the PRD's real-time criterion ("Jetson or documented simulation") and laptop RTF at 10 Hz (R9) satisfy the acceptance bar? | v0.1 README | Author |
| Q9 | ~~CLAUDE.md vs plan on when CI and MLflow arrive~~ **Resolved 2026-11-14:** the plan wins, and CLAUDE.md was updated: CI at S3, MLflow only if needed, Docker with ROS 2, never DVC | — | Author |
| Q10 | ~~Spec header still says "draft"~~ **Resolved 2026-11-14:** header now reads "approved" | — | Author |
| Q11 | Should the intermediate tags (s1-data … v0.3) be adopted under these names? | S1 | Author |
| Q12 | Are the 2021 flights publicly available, and under which licence? | post-v1 | Author |

## 16. Glossary

| Term | Meaning |
|---|---|
| **ADR** | Architecture Decision Record: a short document recording one design decision, its context and its consequences |
| **Ablation** | Removing or swapping one component and measuring the change in end-to-end results, to isolate that component's effect |
| **ATE** | Absolute Trajectory Error: RMS of position error between the estimated and true trajectories |
| **RTE** | Relative Trajectory Error: error over fixed-length windows, which measures local drift |
| **DRMS** | Distance root-mean-square: the RMS of horizontal (2-D) position error |
| **CEP50 / CEP95** | Circular Error Probable: the radius containing 50% / 95% of horizontal errors |
| **Drift rate** | Growth of position error over time (m/min) or distance (%) |
| **Divergence rate** | Share of runs whose horizontal error exceeds 500 m |
| **EKF / ESKF** | Extended Kalman Filter / Error-State KF: an ESKF estimates the small error of a nominal state rather than the state itself |
| **Joseph form** | A numerically stable covariance update that keeps P symmetric positive-definite |
| **PF** | Particle filter: represents the state distribution with weighted samples, so it suits the multi-peak likelihoods of map matching |
| **N_eff** | Effective sample size of a PF, 1/Σwᵢ². Resample when it falls below N/2 |
| **Systematic resampling** | A low-variance way of drawing new particles in proportion to their weights |
| **NEES / NIS** | Normalised Estimation Error Squared / Normalised Innovation Squared. Consistency tests that compare actual errors with the filter's claimed covariance against χ² bands |
| **χ² gate** | Rejects a measurement whose NIS exceeds a χ² threshold; used here for fault detection |
| **INS / IMU** | Inertial Navigation System / Inertial Measurement Unit (accelerometers and gyros). An INS integrates IMU data into position, velocity and attitude |
| **Navigation / tactical grade** | IMU quality classes. Navigation grade drifts far less than tactical grade |
| **Free-inertial** | An INS solution with no external aiding such as GPS |
| **Pinson model** | Linear model of how INS position, velocity and attitude errors propagate |
| **Schuler period** | About 84.4 min. The natural oscillation period of INS horizontal errors near the Earth's surface |
| **NED / WGS-84** | North-East-Down local frame / the standard Earth ellipsoid and datum |
| **HAE** | Height Above Ellipsoid (WGS-84) |
| **TL (Tolles-Lawson)** | Linear model of an aircraft's permanent, induced and eddy-current magnetic interference, fitted on calibration manoeuvres. Ridge-TL adds L2 regularisation |
| **Compensation** | Removing platform interference from magnetometer readings |
| **Crustal anomaly** | The small field (nT) from magnetised rocks, left after the core field is removed; this is what the map records |
| **IGRF** | International Geomagnetic Reference Field: the standard model of the Earth's core field |
| **Diurnal** | Daily time variation of the field caused by the ionosphere; removed with base-station data or modelled |
| **FOGM** | First-Order Gauss-Markov: an exponentially correlated random process, used to model slowly varying errors as a filter state |
| **Upward continuation** | Mathematically transforming a map measured at one altitude to a higher one. Downward continuation is unstable |
| **Drape / level survey** | A map flown at constant height above terrain / at constant altitude (which applies to MagNav maps is **unverified**) |
| **Filled cell** | A map cell filled by interpolation rather than measured; treated as invalid |
| **OPM** | Optically Pumped Magnetometer: an atomic (e.g. caesium) scalar magnetometer, a form of quantum sensor |
| **Fluxgate** | A vector magnetometer measuring three field components; it supplies the TL direction cosines |
| **Stinger** | A boom that holds a magnetometer away from the aircraft body; its Mag 1 is the cleanest sensor |
| **nT** | Nanotesla; the Earth's field is about 50,000 nT and anomalies are typically tens to hundreds of nT |
| **Sage-Husa** | A classical adaptive filter that estimates noise covariance from innovations |
| **R / Q** | Measurement / process noise covariance in a Kalman-type filter |
| **β-NLL / deep ensemble** | A heteroscedastic regression loss / several independently trained networks whose spread estimates uncertainty |
| **PIT** | Probability Integral Transform: a histogram of predicted CDF values at the truth, which should be flat when well calibrated |
| **Coverage** | Fraction of truths inside the predicted 68% / 95% intervals |
| **RTF** | Real-Time Factor: processing time divided by data time. ≤ 1 means real time |
| **Block bootstrap** | Resampling contiguous blocks to get confidence intervals for correlated data |
| **Cascade** | Feeding the PF's position fix into a separate ESKF; it double-counts INS information |
| **Tight coupling** | Using the raw magnetic measurement in the filter, with H = map gradient, instead of a position fix |
| **MagNav / MagNav.jl** | The DAF-MIT AI Accelerator airborne magnetic-navigation dataset / its reference Julia package |
| **Hexagonal architecture** | A layering where the pure domain core has no dependencies on frameworks, I/O or ML libraries |
| **uv / lockfile** | Python package manager / a file pinning exact dependency versions for reproducibility |

---

## 17. Traceability matrix

Each row maps a requirement area to the slice that delivers it, the test layer that proves it (§8), and the risks it reduces (§14). An individual requirement inherits the row of its area unless its own Release column says otherwise.

| Requirement area | Slice(s) | Release | Proven by (test layer / artefact) | Risks addressed |
|---|---|---|---|---|
| FR-DATA-01, 02 (download, verification) | S0 | v0.1 | Unit tests (download safety); manifest data test | RISK-01, RISK-14 |
| FR-DATA-03 to 08 (adapter, segmentation, gaps, IGRF identity, `ins_*` verdict, EDA) | S1 | v0.1 | Data tests; round-trip test; architecture test; `ins_*` verdict doc | RISK-02, RISK-03, RISK-06 |
| FR-MAP-01 to 05 (map model) | S1 (loader), S4 (model) | v0.1 | Unit tests on analytic fields; validity-mask test | RISK-04, RISK-10 |
| FR-COMP-01 to 04 (TL, ridge-TL, reference) | S2 | v0.1 | Design-matrix unit test; T1; `docs/validation/` | RISK-06, RISK-07 |
| FR-COMP-05 to 07 (AI-1) | S5 | v1.0 | Excluded-input test; PIT/coverage; latency benchmark; T2 ablation | RISK-05, RISK-06 |
| FR-INS-01 to 04 (Pinson simulator) | S3 | v0.1 | Unit tests (gravity, radii); Monte-Carlo Schuler-drift test; determinism test | RISK-02, RISK-11 |
| FR-NAV-01 to 05 (PF navigator, gate, faults) | S4 | v0.1 | Property tests; Monte-Carlo NEES/NIS; fault-injection tests; T3 | RISK-04, RISK-11 |
| FR-NAV-06 (R models), 07 (IGRF/diurnal variants) | S4 (fixed, PF-spread), S6 (Sage-Husa, learned) | v0.1 / v1.0 | NIS against χ² bands; T2 paired CIs; IGRF-source test | RISK-05, RISK-06 |
| FR-EVAL-01 to 03 (harness, metrics, splits) | S3, S4 | v0.1 | Metric unit tests; split ADR; test-access log | RISK-03, RISK-06 |
| FR-EVAL-04 to 07 (2×2, statistics, figures, sweeps) | S4 (B0/B1), S5–S7 | v0.1 / v1.0 | T1 to T5; headline figure; fresh-clone reproduction | RISK-05, RISK-13 |
| FR-CLI-01 to 06 | S0 (version, stub), S3–S4 | v0.1 | CLI unit tests; run-directory test | RISK-08, RISK-14 |
| FR-DOC-01 to 07 | S0, S4, S7 | v0.1 / v1.0 | Release checklists (§8); wording and citation checks | RISK-01, RISK-13 |
| NFR-PERF | S4 (RTF), S5 (NN latency) | v0.1 / v1.0 | T5; latency benchmark | RISK-11 |
| NFR-ACC | S2–S6 | v0.1 / v1.0 | T1 to T3; Monte-Carlo consistency tests | RISK-04, RISK-05, RISK-07 |
| NFR-REPRO, NFR-REL | S0 onward | v0.1 | Determinism and regression tests; run directories; manifest | RISK-12, RISK-14 |
| NFR-MAINT, NFR-PORT | S0 onward; CI from S3 | v0.1 / v1.0 | Architecture test; ruff and pyright; Linux CI; second-machine run | RISK-08, RISK-12 |
| NFR-SEC, data governance | S0 onward | v0.1 | `git ls-files` release check; `datasets/README.md`; ADR-0003 | RISK-01 |
| NFR-USAB | S4, S7 | v0.1 / v1.0 | Timed fresh-clone walkthrough; headline discoverability check | RISK-09 |
| NFR-HONEST | S0 onward; sweep in S7 | v0.1 / v1.0 | Wording check; full 2×2 shown; citation checklist | RISK-05, RISK-13 |
