# Software architecture

Source: trinetra_architecture (pre-implementation design). Binding.

## Core rule

Clean / hexagonal (ports and adapters). **Source-code dependencies point inward only.** The domain core never imports ROS 2, a database driver, a dataset reader, a sensor SDK or PyTorch. Outer layers depend on interfaces the inner layers define.

| Layer | Holds | Depends on |
|---|---|---|
| Domain core | estimators (EKF/UKF/PF), navigation and map matching, AI model *interfaces*, state model | nothing |
| Application / use cases | orchestrators: run-fusion, run-navigation, denied-mode, train, evaluate | core only |
| Interface adapters | ROS 2 nodes, data adapters, visualisation adapters | application + core interfaces |
| Frameworks and drivers | ROS 2/DDS, sensors, Gazebo, storage, PyTorch/ONNX Runtime | adapters |

Rejected: microservices (a single vehicle does not need network-partitioned services). ROS 2 is one adapter among several; bag replay and unit tests drive the same core.

## Twelve components

| ID | Component | Single responsibility | Layer |
|---|---|---|---|
| C1 | Sensor ingestion | acquire raw multi-rate data from drivers, bags or datasets; present uniformly | edge |
| C2 | Data pipeline | sync, resample, calibrate, clean, gap-mask onto a common grid | edge |
| C3 | Feature engine | windowed statistical/spectral/temporal features | edge |
| C4 | AI modules | denoise, drift/bias, anomaly/fault, confidence (interfaces in core, models as plugins) | core (interfaces) |
| C5 | Fusion engine | measurements + AI outputs → one probabilistic state (EKF/UKF/PF) | core |
| C6 | Navigation engine | INS mechanisation, dead reckoning, magnetic map matching, denied-mode logic | core |
| C7 | State / world model | current pose, velocity, attitude, biases; frames | core |
| C8 | Orchestration | compose components into use cases | application |
| C9 | Data store / registry | persist raw data, features, models, run metadata | edge |
| C10 | Visualisation & monitoring | trajectory, error, health views | edge |
| C11 | ROS 2 adapter | expose core as nodes/topics/services/actions + TF | edge |
| C12 | Config & model registry | typed parameters, calibration, model selection | cross-cutting, injected |

## Inputs and outputs (the ones that matter)

- C4 in: feature vectors **and filter innovations from C5**. Out: clean signals, bias corrections, anomaly flags, confidence.
- C5 in: measurements, AI corrections and confidence, **map fix from C6**. Out: state mean + covariance, innovations.
- C6 in: fused state, magnetic/terrain map, GPS-availability flag. Out: absolute fix, navigation solution, denied-mode status.

## The two feedback loops (the contribution)

1. **Innovations → AI** (C5 → C4): anomaly/fault and confidence modules read the filter's innovations.
2. **Map fix → fusion** (C6 → C5): magnetic map-match position enters the filter as an absolute measurement.

All other flows are forward only.

```
Sensors → Sync&Clean → Features → AI → Fusion → Navigation → State → Viz
                          └────────────→ Fusion
Fusion ┄innovations┄> AI        Navigation ┄map fix┄> Fusion
Sync&Clean → Store → Viz
```

## Communication

| Mechanism | Used for |
|---|---|
| Interfaces / ports (in-process) | core calls; Fusion depends on a `MeasurementSource` interface, not ROS |
| Dependency injection | C8/C12 supply concrete adapters and models at startup |
| Topics | streaming data between nodes |
| Services | occasional commands (calibrate, reset, get health) |
| Actions | long tasks with feedback |
| TF tree | frame relationships |
| Events / callbacks | innovations, anomaly, mode-change notifications |

## Pipelines

- **Offline (research loop):** download → adapt to canonical schema → EDA & quality → preprocess & features → train → evaluate & ablate (iterate). Batch, reproducible, hardware-free.
- **Online (live):** live sensors → real-time sync → features → AI inference → fusion → navigation → state + monitoring, within a latency budget.
- **Same core, two drivers.** Feature, AI, fusion and navigation code is shared → train/serve parity.

### AI pipeline (C4)

Models are **advisory**: the classical core must work with every model disabled.

| Model | Input | Effect on fusion |
|---|---|---|
| Denoiser | raw/windowed inertial | cleaner measurements |
| Drift/bias predictor | feature window | bias correction to state/measurement |
| Anomaly/fault detector | features + innovations | gate a measurement; raise R |
| Confidence estimator | feature window | adaptive measurement-noise scaling |

### Fusion pipeline (C5)

predict (motion model) → gate & adapt (from AI) → update (measurements) → map-fix update → state + covariance; innovation loops back. EKF, UKF and PF are **interchangeable strategies** behind one interface.

### Navigation pipeline (C6)

`GPS available?` yes → GPS update; no → denied mode (INS + magnetic map match), whose fix feeds back into fusion. **GPS-denied is a mode, not a separate system**; it switches back on reacquisition.

## ROS 2 integration points

| Point | Mechanism | Exposes |
|---|---|---|
| Sensor streams | topics | C1: IMU/mag/GPS/baro |
| Processed data | topics | C2/C3 |
| AI outputs | topics | C4: clean/bias/anomaly/confidence |
| Fused state | topic + TF | C5/C7 |
| Navigation solution | topic; action for goals | C6 |
| Commands | services | calibrate, reset filter, get health |
| Frames | TF | map → odom → base_link → sensor frames |
| Record/replay | bag (MCAP) | deterministic capture |
| Configuration | parameters | C12 |

## Extension points (Open/Closed)

Quantum magnetometer, LiDAR/camera/wheel encoder → sensor interface (C1) + new measurement. Atomic clock → timing/sync (C2). Transformer / neural-KF → AI model interface (C4). UKF/PF/learned filter → estimator interface (C5). Terrain/gravity/signals-of-opportunity maps → map interface (C6). Nav2, multi-robot, digital twin → additive adapters. None require core changes.

## Design patterns in use

Strategy (estimators, AI models), Factory (sensor/AI plugins built from config), Adapter (ROS 2 around the core). Observer is provided by ROS 2 topics.

**⚠ Correction:** the layer diagram lists TimescaleDB as a store. Parquet only for now (CLAUDE.md).
