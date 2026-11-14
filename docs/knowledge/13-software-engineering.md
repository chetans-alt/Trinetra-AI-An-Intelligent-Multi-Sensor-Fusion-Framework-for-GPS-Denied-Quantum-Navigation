# Phase 13 — Software engineering

Source: trinetra_phase13_se.

## Quality attributes

Maintainability, scalability, reliability, **reproducibility (first among equals)**, portability. A research result you cannot reproduce is not a result. References: Wilson et al. 2014/2017; Sandve et al. 2013.

## Architecture verdict

Modular, layered, clean/hexagonal core + plugin pattern for sensors and AI modules + ROS 2 adapters. Not microservices. The load-bearing split is **pure core (no ROS import) vs `ros2/` adapters**. Details in [architecture.md](architecture.md).

## Suggested repo layout (from the notes)

```
src/trinetra_core/   sensor_fusion/  navigation/  ai_modules/  pipelines/   # pure, no ROS
src/ros2/            # thin adapters
configs/  datasets/  models/  visualization/  tests/  docs/  research/  scripts/
pyproject.toml  README.md  LICENSE  CITATION.cff
```

**⚠ Correction:** `docker/`, `.github/workflows/`, `dvc.yaml` and Hydra configs are added only when their trigger arrives (CLAUDE.md). The final package layout is decided at M0, not fixed by this sketch.

## Git

Conventional Commits (`feat:`, `fix:`, `docs:`, `test:`); short-lived feature branches off `main`; small, frequent commits; tag every result that appears in a write-up with its commit SHA. Never commit large data or weights.

**⚠ Project rule:** never push or force-push this history to the old GitHub repo without explicit confirmation.

## GitHub (when a remote exists)

PRs, issues per task, releases per milestone, branch protection with green CI, README + LICENSE + CITATION.cff at the root.

## Testing

Pyramid: many unit tests, fewer integration (nodes on bags), few E2E (Gazebo GPS-denied run). AI-specific tests:
- **Metric gates:** ATE below a budget on a held-out trajectory.
- **Invariance checks:** rotate the input, the trajectory rotates.
- **Sanity:** no NaNs; P stays symmetric positive-definite; a measurement update must not increase tr(P).
- Regression tests lock in fixed drift bugs; parametrise over trajectories.

## Docker, Linux, deployment (later milestones)

Multi-stage Dockerfile on a pinned ROS base; volumes for datasets/weights; GPU passthrough for TensorRT; Compose for the stack. On Jetson: headless SSH, systemd unit with `Restart=on-failure`, journalctl, bringup/record/deploy scripts, `dialout`/`video` groups for device access.

## Documentation

README that lets a stranger reproduce the headline number in under ten minutes; NumPy-style docstrings on public functions; ADRs for decisions (why clean architecture, why Humble/Jazzy); MkDocs/Sphinx only if needed.

## MLOps

Reproducibility triangle: Git (code) + DVC (data/models) + MLflow (runs), each run logging code SHA, data version, config, metrics. References: Sculley et al. 2015; Amershi et al. 2019. ⚠ Added when triggered, not up front.

## CI/CD (when ROS 2 arrives)

ruff + pytest with coverage gate; colcon build; docker image; docs. A research pipeline can re-run a small fixed benchmark and fail on accuracy regression.

## Coding standards

PEP 8 via ruff; DRY (one feature function), KISS/YAGNI (no premature abstraction); SOLID — dependency inversion is what makes the core depend on a `SensorSource`/`MeasurementSource` port. Patterns: Strategy, Factory, Adapter; Observer via ROS 2 topics.

## Best practices / mistakes

Pure core; one feature/experiment per branch; version code, data and models together; pin dependencies; seed control; tag every result. Mistakes: data in Git, train/serve config drift, untested glue scripts, a core that imports ROS, unpinned deps, over-engineering.
