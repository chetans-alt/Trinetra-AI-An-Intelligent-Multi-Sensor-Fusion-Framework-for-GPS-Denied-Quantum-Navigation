# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

**Trinetra-AI: An Intelligent Multi-Sensor Fusion Framework for GPS-Denied Quantum Navigation.**

It estimates position, velocity and orientation when GPS is unavailable. It fuses IMU, magnetometer and barometer data, plus GPS when present, in a classical estimator. Learned modules wrap that estimator and adapt it. The "quantum" angle is magnetic-anomaly map matching, the technique quantum magnetometers enable. The project covers software and AI only, not hardware.

This is the author's single portfolio and research project, aimed at sensor-fusion and navigation ML roles at deep-tech companies such as QuBeats. QuBeats sells its own fusion product also named "Trinetra"; this repo is independent of it and must never imply affiliation.

## Current state

- **Repository:** restarted from scratch on 2026-11-14 with a new `git init`. There is no code yet and no remote.
- **Old history:** a previous version (data adapters, preprocessing and features, but no fusion code) still exists in the GitHub repo `chetan7659/TrinetraAI-An-Intelligent-Multi-Sensor-Fusion-Framework-for-GPS-Denied-Quantum-Navigation`. That repo also holds the knowledge-source PDFs under `Knowledge Source/`. Never push this new history to it, or force-push, without the user's explicit confirmation.
- **Environment:** development is on Windows. The `.venv` uses Python 3.14 and has numpy, pandas, h5py and pytest; activate it with `.venv\Scripts\Activate.ps1`. ROS 2 work needs WSL2 or Docker.
- **Commands:** add build, lint and test commands here once they exist.

## Knowledge sources

Read these before planning work. Where they disagree with this file, this file wins.

- **`deep-research-report.md`** (in the repo): overview, reading list and 16-week plan. See the corrections below.
- **Knowledge-source PDFs, provided by the user and not yet in the repo:**
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
  - The system runs in real time, on a Jetson or in a documented simulation.
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

- **Preprocessing order:**
  1. Mask long gaps; never interpolate across them.
  2. Resample to a common grid.
  3. Remove bias using a static segment.
  4. Apply hard- and soft-iron magnetometer calibration.
  5. Band-limit, with cut-offs chosen from the EDA spectra.
- **Datasets:**
  - **Start with RoNIN.** It is research-licence only, with about half of it public.
  - **MagNav (DAF-MIT AIA)** is required for the magnetic path.
  - **Others:** TLIO, OxIOD, EuRoC, KITTI, comma2k19 and MagPIE.
  - **Licences:** record each dataset's licence in `datasets/README.md`.
- **Evaluation rules:**
  - Judge every AI module by its effect on end-to-end ATE and drift, not by its own validation loss, and ablate it against the system without it.
  - Split data by trajectory or subject, never by shuffled windows.
  - Test with GPS masked as well as available.
  - Report filter consistency (NEES/NIS).

## Proposed learn-by-building plan (16 weeks)

Claude proposed this plan on 2026-11-14 and the user has not yet confirmed it. It replaces the user's original sequence, which spent weeks 1–6 relearning Python, ML and deep learning, which the user already uses at work.

| Weeks | Learn | Build |
|---|---|---|
| 1 | Probability refresher; Labbe *Kalman and Bayesian Filters in Python*, ch. 1–6 | repo skeleton; 1-D and 2-D KF on simulated data |
| 2–3 | EKF and UKF, Jacobians; Labbe ch. 7–11 | 2-D EKF fusing simulated IMU and GPS; GPS-outage simulation |
| 4–5 | Rotations and quaternions (Solà, arXiv 1711.02508); sensor error models; Allan variance | 3-D strapdown INS and pure dead-reckoning baseline |
| 6–7 | Error-state KF; NEES/NIS; ATE/RTE metrics | ESKF fusing INS, magnetometer and barometer on real data; evaluation harness |
| 8–9 | DSP: Butterworth, Savitzky–Golay, resampling, sensor sync | preprocessing module feeding the filter |
| 10–11 | Magnetic navigation, Tolles-Lawson, particle filters | magnetic map matching on MagNav |
| 12–13 | AI-IMU (Brossard 2020), KalmanNet, uncertainty estimation | AI module 1: learned noise covariances inside the ESKF |
| 14 | Time-series anomaly detection | AI module 2: drift prediction or fault detection |
| 15 | Streamlit, Plotly | dashboard with ablation results |
| 16 | — | write-up, benchmark README, apply to QuBeats |

ROS 2 and C++ come afterwards, as a port of the finished ESKF.

## Corrections and scope rules (override the knowledge sources)

- **Right-size the infrastructure.**
  - The PRD's M0 asks for DVC, MLflow, Docker and CI before any data is touched. Phase 11 also proposes TimescaleDB, Postgres and Redis.
  - Do not build these up front: the previous attempt stalled on infrastructure for 22 commits without reaching fusion.
  - Start with git, pytest, ruff and Parquet files. Add MLflow when the first model trains, DVC when data needs versioning, and Docker or CI when ROS 2 arrives. TimescaleDB, Redis, Kafka and Spark are out of scope.
- **Research framing.**
  - The central question: do learned noise and drift models (adaptive Q/R, drift prediction) reduce position error during GPS outages compared with a well-tuned classical filter?
  - GPS serves as ground truth and as a source of simulated outages, not as a fused input during evaluation.
- **AI module priority:**
  1. Confidence → adaptive R and weights.
  2. Drift / bias prediction.
  3. Denoising.
  4. Anomaly and fault detection (one module).
  5. Health monitoring.
  6. Trajectory prediction.

  GANs and VAEs, reinforcement learning, GNNs and XAI are future work. Use PyTorch only; no TensorFlow.
- **No overclaims.** Magnetic aiding bounds drift; it does not make navigation "drift-free", even though the phase notes and the report say so.
- **Citations.**
  - **Never cite:** "Thompson et al. 2019", which the report itself labels hypothetical.
  - **Verify against primary sources before citing:** every 2025–2026 paper and every industry claim. This covers Huang 2026, MagPIE2, the Boeing 2024 quantum flight, AQNav, Ironstone Opal / Muradoglu 2025, and the NavIC outage figures.
- **ROS 2:** target Humble (PRD) or Jazzy. Foxy and Galactic, suggested in the report, are end-of-life.
- **Dates:** the report's Gantt dates (from 2026-07-19) are in the past.
- **Verify the PRD sample code against the real data.**
  - The RoNIN HDF5 keys (`synced/acce`, `pose/tango_pos`, etc.) need checking against an actual file.
  - The "error-state EKF skeleton" in the PRD is really a linear KF.
