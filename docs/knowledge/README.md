# Knowledge base

These notes condense the 13 knowledge-source documents the user supplied on 2026-11-14 (phase notes, the architecture design and the PRD/build guide). They are rewritten as project reference notes, not copies: each file keeps the decisions, equations and design rules the project depends on, and drops the textbook padding.

Where a source conflicts with [CLAUDE.md](../../CLAUDE.md), CLAUDE.md wins. Known conflicts are marked **⚠ Correction** inline.

## Precedence

1. [CLAUDE.md](../../CLAUDE.md): scope rules and corrections.
2. [architecture.md](architecture.md) and [prd-build.md](prd-build.md): binding design and acceptance criteria.
3. The phase notes: background and reference.

## Index

| File | Source | Use it for |
|---|---|---|
| [architecture.md](architecture.md) | trinetra_architecture | Layers, 12 components, data flow, pipelines, ROS 2 seams |
| [prd-build.md](prd-build.md) | trinetra_prd_build | Requirements, acceptance criteria, milestones, step-by-step build |
| [01-math.md](01-math.md) | trinetra_math_notes | Linear algebra, calculus, probability, statistics behind the KF |
| [04-time-series.md](04-time-series.md) | trinetra_phase4_ts (+ .md) | Windowing, forecasting, sequence models, drift, features |
| [05-dsp.md](05-dsp.md) | trinetra_phase5_dsp | Sampling, noise, FFT, filters, preprocessing pipeline |
| [06-sensors.md](06-sensors.md) | trinetra_phase6_sensors | Sensor physics, error models, calibration, Allan variance |
| [07-fusion.md](07-fusion.md) | trinetra_phase7_fusion | Frames, quaternions, KF/EKF/UKF/PF, fusion architecture |
| [08-navigation.md](08-navigation.md) | trinetra_phase8_navigation | GNSS, INS, magnetic map matching, navigation modes |
| [09-ai-modules.md](09-ai-modules.md) | trinetra_phase9_ai | Denoise, drift, fault, health, anomaly, confidence modules |
| [10-ros2.md](10-ros2.md) | trinetra_phase10_ros2 | ROS 2 node graph, TF, robot_localization, deployment |
| [11-data.md](11-data.md) | trinetra_phase11_data | Formats, schema, sync, quality, feature parity, versioning |
| [13-software-engineering.md](13-software-engineering.md) | trinetra_phase13_se | Repo layout, testing, Git, CI, MLOps |
| [references.md](references.md) | all | Datasets and papers, with verification status |

The original PDFs also contain interview, viva, research and coding question banks (roughly 50–70 per phase). Those lists are not reproduced here; each file ends with the derivations most worth being able to do from memory.

## Cross-cutting corrections (apply everywhere)

- **⚠ "Drift-free" magnetic navigation:** several notes say magnetic aiding is "drift-free". It bounds drift; it does not remove it.
- **⚠ Infrastructure:** TimescaleDB, Postgres, Redis, Kafka, Spark, Flink, Hydra, Great Expectations and up-front DVC/MLflow/Docker/CI are out of scope at the start. Start with git, pytest, ruff and Parquet (see CLAUDE.md).
- **⚠ Unverified claims:** Boeing 2024 quantum flight, SandboxAQ AQNav, Q-CTRL Ironstone Opal / Muradoglu 2025, MagPIE2, and the NavIC outage figures must be checked against primary sources before citing.
- **⚠ PRD sample code:** the RoNIN HDF5 keys are unverified, and the "error-state EKF skeleton" is a plain linear KF.
- **⚠ ROS 2 distro:** Humble or Jazzy only.
