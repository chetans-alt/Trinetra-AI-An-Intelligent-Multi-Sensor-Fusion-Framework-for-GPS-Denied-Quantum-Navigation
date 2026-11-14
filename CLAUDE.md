# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

**Trinetra-AI: An Intelligent Multi-Sensor Fusion Framework for GPS-Denied Quantum Navigation.**

It estimates position, velocity and orientation when GPS is unavailable. **Magnetic-anomaly navigation is the core** (decided 2026-11-14):
- **Compensate:** remove the vehicle's own magnetic interference from the magnetometer, first with Tolles-Lawson, then with a learned neural correction on top.
- **Match:** match the compensated anomaly signal against a crustal magnetic-anomaly map with a particle filter.
- **Navigate:** the particle filter is the navigator during GPS outages. It runs over INS error states and is driven by a simulated INS of declared grade (ADR-0001). Feeding the PF fix into an ESKF is kept only as a measured ablation, because it double-counts INS information.

Exactly two learned modules wrap this: learned compensation and learned trust (R) for map fixes. The "quantum" angle is that quantum magnetometers make this map matching more precise. The project covers software and AI only, not hardware.

This is the author's single portfolio and research project, aimed at sensor-fusion and navigation ML roles at deep-tech companies such as QuBeats, which builds navigation on the Earth's magnetic field. QuBeats sells its own fusion product also named "Trinetra"; this repo is independent of it and must never imply affiliation.

## Current state

- **Repository:** restarted from scratch on 2026-11-14 with a new `git init`. There is no code yet. `origin` is `github.com/chetans-alt/Trinetra-AI-An-Intelligent-Multi-Sensor-Fusion-Framework-for-GPS-Denied-Quantum-Navigation` (branch `main`).
- **Old history:** a previous version (data adapters, preprocessing and features, but no fusion code) still exists in the GitHub repo `chetan7659/TrinetraAI-An-Intelligent-Multi-Sensor-Fusion-Framework-for-GPS-Denied-Quantum-Navigation`. That repo also holds the knowledge-source PDFs under `Knowledge Source/`. Never push this new history to it, or force-push, without the user's explicit confirmation.
- **Purpose:** a job portfolio, not a thesis. The author has 8 h/week and no GPU (CPU-only training); deadline not yet set.
- **Environment:** development is on Windows. The old `.venv` is broken: its Python 3.14 base interpreter no longer exists. The target is **Python 3.12 managed by uv with a lockfile** (S0 rebuilds it). ROS 2 work, post-v1, needs WSL2 or Docker.
- **Commands:** add build, lint and test commands here once they exist.

## Knowledge sources

Read these before planning work. Where they disagree with this file, this file wins.

- **[docs/superpowers/specs/2026-11-14-trinetra-project-plan-design.md](docs/superpowers/specs/2026-11-14-trinetra-project-plan-design.md):** the **approved project plan** (2026-11-14). It contains the panel rulings, system design, evaluation protocol, engineering standards, milestones S0–S7, and the risk register. It is the most binding source after this file.
- **[docs/literature-review.md](docs/literature-review.md):** reviewed papers and the design decisions taken from them.
- **`deep-research-report.md`** (in the repo): overview, reading list and 16-week plan. See the corrections below.
- **[docs/knowledge/](docs/knowledge/README.md):** condensed markdown notes of every knowledge-source document below, with corrections marked inline. Read these instead of the PDFs.
- **Knowledge-source PDFs, provided by the user (originals not in the repo):**
  - **Phase notes:** `trinetra_math_notes` (phase 1), `phase4_ts`, `phase5_dsp`, `phase6_sensors`, `phase7_fusion`, `phase8_navigation`, `phase9_ai`, `phase10_ros2`, `phase11_data` and `phase13_se`.
  - **`trinetra_architecture`:** the software design.
  - **`trinetra_prd_build`:** the product requirements and the step-by-step build guide.
  - The two most binding are `trinetra_architecture` and `trinetra_prd_build`; their decisions are summarised below.

## Architecture (from trinetra_architecture.pdf)

- **Clean/hexagonal layers.** Dependencies point inward: Frameworks (ROS 2, sensors, Gazebo, storage, PyTorch) → Interface adapters → Application/use cases → Domain core.
  - The domain core is pure Python: estimators (EKF/UKF/PF), navigation and map matching, AI model *interfaces*, and the state model.
  - The core must never import ROS, a database driver, a dataset reader or PyTorch.
  - ROS 2 is just one adapter. Bag replay and unit tests drive the same core.
- **Twelve components.**

  | ID | Component |
  |---|---|
  | C1 | Sensor ingestion |
  | C2 | Data pipeline (sync, calibrate, gap-mask) |
  | C3 | Feature engine |
  | C4 | AI modules |
  | C5 | Fusion engine |
  | C6 | Navigation engine |
  | C7 | State / world model |
  | C8 | Orchestration |
  | C9 | Data store / registry |
  | C10 | Visualisation |
  | C11 | ROS 2 adapter |
  | C12 | Config and model registry |

  The domain core is the C4 interfaces plus C5, C6 and C7.
- **Two feedback loops, which are the contribution.**
  - Filter innovations (C5) feed the AI anomaly and confidence modules (C4).
  - The magnetic map-match fix (C6) feeds back into fusion (C5) as an absolute measurement.
  - All other data flows go forward only.
- **AI modules are advisory.** Their outputs enter the filter as corrected measurements and as adaptive noise or gating (Q/R). The classical core must keep working when every module is disabled.
- **Estimators are interchangeable.** EKF, UKF and PF are strategies behind one interface. Sensors and AI models are plugins built from config (a factory pattern).
- **GPS-denied is a mode, not a separate system.** The navigation engine switches its correction source from GPS to the magnetic map match when GPS drops, and switches back when it returns.
- **One core, two pipelines.** The offline (training) and online (live) pipelines share the same core. Feature code must be a single function used by both (train/serve parity).

## Requirements and acceptance (from trinetra_prd_build.pdf)

- **Acceptance criteria:**
  - On at least two held-out GPS-denied trajectories, ATE and drift rate beat a documented inertial baseline.
  - The system runs in real time. For v1 this is shown as a real-time factor on a laptop CPU at 10 Hz; Jetson comes after v1.
  - Every headline number is reproduced by one scripted command from versioned code, data and config.
- **Milestones**, each a vertical slice that ends in working, tested, tagged software:

  | Milestone | Scope |
  |---|---|
  | M0 | Setup |
  | M1 | Data + EDA |
  | M2 | Baseline EKF |
  | M3 | AI modules |
  | M4 | GPS-denied navigation |
  | M5 | ROS 2 + simulation |
  | M6 | Evaluation + visualisation |
  | M7 | Deploy |

  Ship M2 (a working baseline on real data) before polishing any AI module.
- **Canonical schema.** One adapter per dataset maps into it, and code after the adapters never imports HDF5 or dataset names. Storage is Parquet with zstd compression.

  | Column(s) | Unit |
  |---|---|
  | `t` | seconds, monotonic |
  | `ax ay az` | m/s² |
  | `gx gy gz` | rad/s |
  | `mx my mz` | µT |
  | `gt_x gt_y gt_z` | ground-truth position (optional) |
  | `lat lon alt` | optional |
  | `baro` | Pa (optional) |

  For airborne data the core signal is the **scalar anomaly in nT**. The schema gains an airborne extension: scalar magnetometers, fluxgates, INS outputs, currents and line IDs. Define it after inspecting a real MagNav file.

- **Preprocessing order** (pedestrian/IMU data; for airborne data, Tolles-Lawson replaces steps 3–4, and gap masking still applies):
  1. Mask long gaps; never interpolate across them.
  2. Resample to a common grid.
  3. Remove bias using a static segment.
  4. Apply hard- and soft-iron magnetometer calibration.
  5. Band-limit, with cut-offs chosen from the EDA spectra.
- **Datasets:**
  - **Start with MagNav (DAF-MIT AIA).** It provides flight data with magnetometers, INS and GPS truth, plus anomaly maps. It is the primary dataset (this overrides the PRD's "start with RoNIN").
  - **MagNav has no raw high-rate IMU data,** so strapdown mechanisation from raw IMU is impossible. The INS baseline is a Pinson error-model simulation on the GPS truth trajectory, with navigation grade as the headline.
  - **MagNav licence conflict:** Zenodo says attribution/public domain; the MagNav.jl data-sharing agreement says non-profit research only, no redistribution. Follow the stricter terms: **never commit raw data, derived Parquet, map tiles or trained weights.** Use a download script with a SHA-256 manifest.
  - **RoNIN** is out of v1. MagPIE is the preferred indoor extension, if one is ever added.
  - **Others:** TLIO, OxIOD, EuRoC, KITTI and comma2k19.
  - **Licences:** record each dataset's licence in `datasets/README.md`.
- **Evaluation rules:**
  - Judge every AI module by its effect on end-to-end ATE and drift, not by its own validation loss, and ablate it against the system without it.
  - Split data by trajectory or subject, never by shuffled windows.
  - Test with GPS masked as well as available.
  - Report filter consistency (NEES/NIS).

## Approved plan (29 weeks at 8 h/week)

Approved 2026-11-14. It replaces the earlier 16-week plan; the full plan is in the spec linked above.

| Weeks | Slice | Exit criterion (summary) |
|---|---|---|
| 1–2 | **S0 Setup:** uv + Python 3.12, skeleton, ADR-0001 to 0003, download script, licence email | pytest and ruff pass; the core import-rule test works |
| 3–5 | **S1 Data + EDA:** MagNav adapter, maps, line segmentation, `ins_*` free-inertial test, anomaly-floor table | IGRF identity holds within 0.01 nT; splits fixed in an ADR |
| 6–8 | **S2 Classical compensation:** 18-term Tolles-Lawson and ridge-TL, cross-checked against MagNav.jl | Mag 1 reproduces `mag_1_c` to within a few nT |
| 9–11 | **S3 INS simulator + outage harness + CI** | Pinson drift matches theory within 10% |
| 12–17 | **S4 Magnetic navigation (v0.1, the minimum shippable portfolio):** PF navigator, map model, NIS gate, headline figure | bounded error over 30-min outages on test flights 1003 and 1007 |
| 18–22 | **S5 AI-1:** learned residual compensation (ensemble, ≤50k params, CPU) | beats ridge-TL; calibrated |
| 23–26 | **S6 AI-2:** learned R vs fixed, PF-spread and Sage-Husa R; full 2×2 | NIS within band |
| 27–29 | **S7 Write-up (v1.0):** error budget, sensor-noise ("quantum") sweep | a fresh clone reproduces the results |

After v1: the ESKF and the cascade ablation, tight coupling, the tactical-IMU sweep, the 2021 flights, then ROS 2 and C++.

## Corrections and scope rules (override the knowledge sources)

- **Right-size the infrastructure.**
  - The PRD's M0 asks for DVC, MLflow, Docker and CI before any data is touched. Phase 11 also proposes TimescaleDB, Postgres and Redis.
  - Do not build these up front: the previous attempt stalled on infrastructure for 22 commits without reaching fusion.
  - Start with uv, git, pytest, hypothesis, ruff, pyright and Parquet files. Add more only on its trigger, per the approved plan (§5):
    - GitHub Actions CI with the first filter-consistency test (S3).
    - Per-run directories at the first NN, with MLflow only if sweeps become unmanageable.
    - Docker when ROS 2 arrives (after v1).
  - Never add DVC (the data can't be redistributed anyway), TimescaleDB, Redis, Kafka, Spark or Hydra.
- **Research framing.**
  - The central question: does learned magnetic compensation, with adaptive trust in map-match fixes, reduce position error during GPS outages compared with INS plus Tolles-Lawson-based map matching in a well-tuned classical filter?
  - It is answered as a 2×2 ablation: Tolles-Lawson vs Tolles-Lawson + NN, crossed with fixed vs learned R.
  - Reference rows: INS only; the stinger Mag 1 upper bound; R from PF spread; Sage-Husa adaptive R. Learned modules must beat the classical baselines, not only fixed R.
  - GPS serves as ground truth and as a source of simulated outages, not as a fused input during evaluation.
  - **Compensation reference:** `mag_1_c` minus diurnal minus IGRF. It is a reference, not ground truth. **The map is never a training target.**
  - **Never NN inputs:** latitude, longitude, time, map value, GPS or stinger data.
  - IGRF is evaluated at the *estimated* position.
- **Exactly two AI modules:**
  1. Learned residual compensation on top of Tolles-Lawson.
  2. Learned R scale for map fixes.

  Fault detection is a classical χ² test on NIS. Drift prediction, denoising, health monitoring and trajectory prediction are cut. GANs and VAEs, reinforcement learning, GNNs and XAI are future work. Use PyTorch only; no TensorFlow.
- **No Week-1 toy filters.** Every week ends in project software.
- **No overclaims.** Magnetic aiding bounds drift; it does not make navigation "drift-free", even though the phase notes and the report say so.
- **Citations.**
  - **Never cite:** "Thompson et al. 2019", which the report itself labels hypothetical.
  - **Verify against primary sources before citing:** every 2025–2026 paper and every industry claim. This covers Huang 2026, MagPIE2, the Boeing 2024 quantum flight, AQNav, Ironstone Opal / Muradoglu 2025, and the NavIC outage figures.
- **ROS 2:** target Humble (PRD) or Jazzy. Foxy and Galactic, suggested in the report, are end-of-life.
- **Dates:** the report's Gantt dates (from 2026-07-19) are in the past.
- **Verify the PRD sample code against the real data.**
  - The MagNav field names and the RoNIN HDF5 keys (`synced/acce`, `pose/tango_pos`, etc.) need checking against an actual file.
  - The "error-state EKF skeleton" in the PRD is really a linear KF.
