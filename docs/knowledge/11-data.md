# Phase 11 — Data engineering

Source: trinetra_phase11_data.

**⚠ Scope correction (CLAUDE.md):** the notes propose TimescaleDB, Postgres, Redis, Kafka, Spark, Flink, Airflow, Hydra, Great Expectations and LakeFS. All are out of scope. Use Parquet files, pandas, and plain Python validation. The notes themselves say a single-vehicle project should stay lean; the principles below are what to keep.

## Volume

9-DOF IMU at 200 Hz as float64: `200 × (9×8 + 8) B = 16 kB/s ≈ 57.6 MB/h`. Parquet + zstd compresses 5–10×. At 200 Hz the ingest budget is 5 ms per sample.

## Two regimes

Offline/batch (logs → Parquet → clean → version → train) and online/streaming (Jetson: capture → sync → infer → fuse in real time). Same feature and model code in both.

## Formats

| Format | Use here |
|---|---|
| **MCAP** | record live/sim ROS 2 sessions (default since Iron; indexed, chunk-compressed) |
| **Parquet (+zstd)** | the ML/compute layer; columnar, read only needed channels |
| JSON | run metadata, config |
| HDF5 | only for raw datasets that ship in it (RoNIN, MagNav) — read by adapters only |
| CSV | tiny human-readable exports |

"MCAP for capture, Parquet for compute."

## Canonical schema and adapters

Every dataset gets one adapter into the canonical schema (see [prd-build.md](prd-build.md) Step 2). Downstream code sees only the canonical form. Keep an immutable raw layer; derived layers: raw → interim (synced/cleaned Parquet) → features (versioned). Data model principle if a DB is ever added: normalised small metadata tables (sessions, sensors, calibration), denormalised wide readings partitioned by time and session.

## Time-series engineering

| Operation | Concern |
|---|---|
| Timestamps | hardware vs receipt time; monotonicity |
| Sync | IMU 200 Hz vs GPS 1 Hz |
| Resample / interpolate | onto the fastest sensor's grid; linear/spline for slow channels |
| Missing data | **mask, never fabricate across long gaps** — the filter enters denied mode instead |
| Outliers | flag/gate (feed anomaly module) |
| Windowing | training tensors |

Sub-millisecond misalignment appears downstream as sensor error that is really timing error.

## Data quality (validate at ingestion, also gate live measurements)

| Check | Rule |
|---|---|
| Range | |a| < 16 g, |ω| < 2000 °/s |
| Rate | IMU ≈ nominal, no long gaps |
| Monotonic time | catches clock resets |
| Completeness | all required channels non-null |
| Duplicates | no repeated (t, sensor) |
| Consistency | units and frames agree (m/s², µT) |
| Drift | distribution stable |

Dimensions: validity, completeness, consistency, accuracy, uniqueness, timeliness.

## Feature store (lightweight)

The non-negotiable principle is **train/serve parity**: the exact function that builds training features builds live features. A versioned features Parquet table is the offline store; no Feast/Redis. Point-in-time correctness: no future leakage when joining labels.

## Versioning and reproducibility

Git for code and config. DVC when data needs versioning; MLflow when the first model trains (CLAUDE.md). Every experiment must record code SHA, data version, config and seed.

## Security (proportionate)

Trajectories reveal location. Keep raw data out of Git, back it up, checksum archives. Don't over-engineer for a public-dataset research project.

## Datasets: format and ingestion

| Dataset | Native format | Ingestion |
|---|---|---|
| RoNIN | HDF5 | HDF5 reader → canonical |
| TLIO | numpy/HDF5 | array store |
| OxIOD | CSV | CSV → Parquet |
| EuRoC | ROS bag / CSV | CSV → Parquet |
| KITTI | text + binary + PNG | parse OXTS IMU/GPS |
| comma2k19 | numpy chunks (~100 GB) | chunked loader, sample first |
| MagPIE | CSV / MAT | → Parquet |
| MagNav | HDF5 / CSV | flight lines → Parquet |

## Best practices / mistakes

One canonical schema; train/serve parity; version data and runs; mask missing data; immutable raw copy. Mistakes: silently interpolating GPS gaps; feature skew; unversioned datasets; wrong units/frames on ingest; clock confusion.
