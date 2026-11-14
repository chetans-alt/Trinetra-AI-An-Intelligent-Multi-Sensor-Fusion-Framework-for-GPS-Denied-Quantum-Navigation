# Trinetra-AI: project plan and design (v1)

**Status:** approved by the author on 2026-11-14. The requirements view is in [docs/prd.md](../../prd.md).

**How it was made:** a five-member planning panel produced it.
- **Supervisor:** chair, who ruled on every disagreement.
- **Two AI/CSE specialists:** machine learning for sensors; state estimation and software engineering.
- **Two physicists:** geomagnetism and magnetic compensation; inertial navigation.

**Fixed inputs from the author:**
- The purpose is a **job portfolio**.
- **8 hours a week** of work.
- **No GPU.**
- The deadline is not set.

Where this plan departs from `CLAUDE.md`, the change is listed in section 9. `CLAUDE.md` is updated only after the plan is approved.

---

## 1. Panel rulings

| # | Issue | Ruling | Why |
|---|---|---|---|
| R1 | How map fixes enter navigation | **The particle filter (PF) is the navigator.** It runs over INS *error* states: δN, δE, δv_N, δv_E and a map/sensor bias, as in Canciani & Raquet and MagNav.jl. The ESKF→PF "cascade" from CLAUDE.md becomes a measured ablation, documented in an ADR. A tightly coupled EKF (H = ∇map) is a stretch goal. | Feeding the PF fix into an ESKF counts the same INS information twice, which makes the filter overconfident. All four specialists raised this independently. Measuring the problem shows more competence than hiding it. |
| R2 | INS baseline | **Simulate INS errors on the GPS truth trajectory** using a Pinson error model, a declared IMU grade and swept random seeds. The recorded `ins_*` stream is used only if a test shows it is free-inertial, i.e. not GPS-aided. | MagNav has **no raw high-rate IMU data** (verified from the docs), so building a strapdown INS from raw IMU is impossible. Canciani and MagNav.jl both use this simulation method, which makes our numbers comparable with theirs. |
| R3 | IMU grade for the headline | **Navigation grade** is the headline. Tactical grade is a later sweep. The grade specs are unverified. | It matches the literature. Tactical-grade drift can push the PF outside the range where it can still lock on. |
| R4 | MagNav.jl | **Use it as an oracle only.** Optionally install Julia once and reproduce its TL and INS numbers on the same flight lines. It is never a runtime dependency. | Reviewers will ask what this project adds. The answer is: reproduce MagNav.jl's numbers first, then show the improvement. |
| R5 | Python | **Python 3.12 via uv, with a lockfile.** Delete the broken `.venv`. | The 3.14 base interpreter for `.venv` no longer exists, and PyTorch wheels for Windows on cp314 are unverified. |
| R6 | Number of AI modules | **Exactly two.** AI-1 is a residual neural compensator on top of Tolles-Lawson (TL). AI-2 is a learned trust scale R for map fixes. Fault detection is a classical χ² test on NIS plus synthetic fault injection. Drift, denoising, health and trajectory modules are **cut**. | Two rigorously ablated modules beat seven shallow ones. There are no IMU-bias labels, so drift prediction would be circular. |
| R7 | What counts as truth | **Navigation truth** is GPS. **The compensation reference** is the stinger Mag 1 field: `mag_1_c` minus diurnal minus IGRF. It is a *reference*, not ground truth. **The map is never a training target.** | Training on the map would leak it into the model and make map matching circular. |
| R8 | IGRF and diurnal leakage | **Headline run:** IGRF evaluated at the *estimated* position, plus the supplied base-station diurnal (assumes a base station is available). **Ablation:** diurnal modelled as a first-order Gauss–Markov (FOGM) state. Report both. | The supplied IGRF uses the GPS position, which leaks truth. |
| R9 | Real time | **No Jetson in v1.** Report the real-time factor (RTF) on a laptop CPU at 10 Hz. | The PRD allows "documented simulation". |
| R10 | Schema and preprocessing | The core signal is the **scalar anomaly in nT**. Extend the schema with scalar magnetometers, fluxgates, INS outputs, currents and line IDs. For airborne data, TL replaces the static-bias and hard/soft-iron calibration steps. Gap masking stays. | The PRD schema was written for pedestrian IMU data. |
| R11 | Week-1 toy filters | **Dropped.** | Every week must end in project software. |
| R12 | Infrastructure | **Added only when a trigger occurs** (section 5). DVC: never. MLflow: only if parameter sweeps become unmanageable. | Avoid repeating the previous attempt's stall. |
| R13 | Data licence | **Follow the stricter terms.** Zenodo says public domain or attribution; the MagNav.jl data-sharing agreement says non-profit research only, no redistribution. So: never commit data, derived files or trained weights. Email magnav-admins@mit.edu. | The two sources conflict, and only the data owner can settle it. |

## 2. Project charter

**Pitch** (safe to use in the README). Trinetra-AI is an open, reproducible framework for GPS-denied aircraft navigation using the Earth's crustal magnetic anomaly field.
- **How it works:** it removes the aircraft's own magnetic interference (Tolles-Lawson, with an optional learned correction). A particle filter then matches the cleaned signal against an anomaly map, which aids an inertial solution throughout a GPS outage.
- **Data:** results come from the public MagNav flight data, with GPS masked on held-out flights.
- **Reproducibility:** one command regenerates every headline number.
- **The contribution** is a careful, ablated measurement of whether learned compensation and learned trust reduce position error compared with a well-tuned classical system.

**Research question** (a 2×2 ablation):

| | Fixed tuned R | Learned R |
|---|---|---|
| **TL** | classical baseline (B1) | effect of learned trust only |
| **TL + NN** | effect of learned compensation only | full system |

**Reference rows** reported alongside the 2×2:
- INS only (B0).
- Stinger Mag 1 as an upper bound.
- R taken from the PF particle spread.
- Classical Sage-Husa adaptive R.

**Goals:**
1. Reproduce MagNav.jl-level TL and INS numbers.
2. Show that drift stays bounded, with consistent NEES/NIS, on at least two held-out flights.
3. Measure each learned module end to end, with confidence intervals.
4. Get from a fresh clone to the results table with one command.

**Non-goals (v1):** ROS 2, Jetson, C++, live ingestion, Streamlit, RoNIN, UKF, ECEF frame, coning/sculling corrections, Allan variance.

**Anti-goals (never):**
- infrastructure before fusion;
- picking the best seed;
- tuning on test flights;
- latitude, longitude, time or map value as NN inputs;
- fusing GPS during evaluation;
- implying affiliation with any company or product.

**Honest-claims policy:**
- Say magnetic aiding **bounds** drift. Never say "drift-free".
- Report negative or mixed results as findings.
- Verify every literature number against its primary source.
- **Quantum wording:** "MagNav's scalar magnetometers are optically pumped caesium (atomic) sensors; this project adds no quantum hardware. A sensor-noise sweep shows the noise floor below which better sensors would matter, relative to platform, map and diurnal errors."

## 3. System design v1

```
MagNav HDF5 ──adapter──► Parquet (per flight, line-segmented, gap-masked) ──► EDA, anomaly-floor table
TL fit (calibration segments only) ─► Compensator(TL | TL+NN) ─► anomaly_nT, var_nT²
GPS truth ─► INS error simulator (declared grade, seed) ─► INS solution
GPS loss ─► PF navigator (INS error states; map likelihood; NIS gate; R model) ─► outage harness ─► run dir
```

**Package layout:**

```
src/trinetra/
  core/          numpy/scipy only (enforced by a test): state, earth (WGS-84, Somigliana gravity,
                 transport rate), ins_error (Pinson), pf, eskf (later), maps (bicubic interpolation,
                 gradient, FFT upward continuation, filled-cell mask), gating, metrics, consistency, protocols
  adapters/      magnav.py, magnav_maps.py   ← the only place that imports h5py or dataset names
  pipeline/      segment, preprocess, features (one function shared by training and evaluation)
  compensation/  tolles_lawson, ridge_tl, nn (PyTorch only here)
  experiments/   outage, ablation, reproduce
  viz/
configs/  scripts/download_data.py  data/ (gitignored)  runs/ (gitignored)
tests/{unit,property,consistency,data,regression}  docs/{adr,validation,knowledge}
```

**Interfaces.** Each is a `typing.Protocol`, added only once a second implementation exists.
- `Compensator.fit(frames)`, `.predict(frame) -> (anomaly_nT, var_nT2)`
- `MapModel.value()`, `.gradient()`, `.valid()`. Cells filled in by k-NN interpolation report as invalid.
- `Navigator.init(state, P)`, `.propagate(dt, ins)`, `.update(z, R)`, `.nis()`
- `RModel.scale(context)`. Implementations: fixed, PF spread, Sage-Husa, learned.

**Estimators:**
- **PF:**
  - About 5 states, 1–5k particles.
  - Systematic resampling when N_eff < N/2.
  - A χ² gate on NIS.
  - Divergence flags for N_eff collapse and for multiple peaks in the particle cloud.
- **Compensation error** is correlated over time. v1 inflates R to cover it; a FOGM state is a stretch goal.
- **ESKF (15 states, local-level NED, Joseph form):** built later. See decision D3.

**Data policy:**
- `scripts/download_data.py` fetches Zenodo record 12723700 and the MagNav.jl map artifact, and checks them against a committed SHA-256 manifest.
- Licences are recorded in `datasets/README.md`.
- Only figures and aggregate metrics are published.

## 4. Evaluation protocol

**Splits** (provisional until EDA):
- TL is fit on calibration boxes only: Flt1006 and the calibration segment of Flt1002.
- NN training: 1002 and 1004.
- Validation: 1005.
- **Test: 1003 and 1007.** These are also the comparison flights in the literature. The test set is touched once per frozen configuration.
- Leave-one-flight-out spread as a robustness check.
- Splits are by flight or line, **never by shuffled windows**.

**Outage protocol:**
- GPS outages of 10, 30 and 60 minutes.
- Outage start times slide every 5 minutes along each test flight; each start needs at least 60 s of valid map coverage.
- Initial covariance P is the declared GPS/INS covariance.
- At least 20 Monte-Carlo INS seeds per start; at least 5 NN training seeds.

**Metrics:**
- **Position error:** horizontal error vs time since GPS loss; DRMS at 10, 30 and 60 minutes; CEP50 and CEP95; ATE; drift in m/min and as a percentage of distance; final error.
- **Robustness:** divergence rate (error above 500 m).
- **Consistency:** NEES and NIS against χ² bands.
- **Compensation:** RMSE in nT, per line.
- **Calibration:** PIT histogram and 68%/95% coverage.
- **Speed:** real-time factor.

**Headline figure.** Horizontal error vs time since GPS loss, on a log y-axis. It shows the median and the 5–95% band over all outages and seeds, for B0, B1, +NN and +NN+learned R, with the filter's 1σ envelope overlaid.

**Tables:**
- **T1:** compensation error (nT) by flight and magnetometer, against MagNav.jl.
- **T2:** median error at 10/30/60 minutes with 95% confidence intervals, for the 2×2 and the reference rows.
- **T3:** filter consistency.
- **T4:** error budget.
- **T5:** runtime.

**Statistics:**
- Paired differences on identical outages and seeds.
- Block-bootstrap confidence intervals.
- Medians with interquartile range.
- No claim based on a single flight.

## 5. Engineering standards: "production grade"

**Definition:** `uv sync` → `scripts/download_data.py` → `uv run trinetra reproduce` regenerates the headline table and figure, and the tests check the mathematics, not just that the code runs.

| When | What |
|---|---|
| Day 1 | `pyproject.toml` (src layout); uv and `uv.lock`; Python 3.12; ruff; pytest and hypothesis; pyright (strict on `core/`); a test that `core/` never imports torch, h5py, rclpy or dataset names; a "Reproduce" section in the README; ADR-0001 (PF navigator), ADR-0002 (truth definitions), ADR-0003 (data licence) |
| First filter consistency test | GitHub Actions: ruff, pyright and the data-free tests |
| First NN | Per-run directories (MLflow only if needed) |
| Post-v1 ROS 2 | Docker |
| Never | DVC, TimescaleDB, Redis, Kafka, Spark, Hydra |

**Tests:**
- **Unit tests** against hand-computed values: gravity, Earth radii, the TL design matrix, interpolation on an analytic field.
- **Property tests:** P stays symmetric positive-definite; a zero-information update changes nothing.
- **Monte-Carlo consistency:** NEES and NIS inside their bands; the Pinson simulator reproduces Schuler drift within 10%.
- **Data tests** (skipped when data is absent):
  - the identity `mag_1_dc − mag_1_igrf = igrf` holds within 0.01 nT;
  - data checksums match the manifest.
- **Regression tests** on ATE.

**Run outputs:** each run writes `runs/<date>_<name>/` containing the config, git SHA, data-manifest hash, `metrics.json` and figures.

**Repository practice:** Conventional Commits, and a git tag for each milestone.

## 6. Milestones at 8 hours a week, CPU only

| Slice | Weeks | Deliverables | Exit criteria |
|---|---|---|---|
| **S0 Setup** | 1–2 | rebuilt environment; skeleton; ADR-0001 to 0003; download script and manifest; licence email sent | pytest and ruff pass; the import-rule test catches a deliberate violation |
| **S1 Data + EDA** | 3–5 | MagNav adapter → Parquet; map loader; line segmentation; field inventory; free-inertial test of `ins_*`; anomaly-floor table | IGRF identity within 0.01 nT; round-trip test passes; a written verdict on `ins_*`; splits recorded in an ADR |
| **S2 Classical compensation** | 6–8 | 18-term TL and ridge-TL (fluxgate B, band-pass); nT table produced by one command; MagNav.jl cross-check | TL on Mag 1 reproduces `mag_1_c` to within a few nT; cabin magnetometers within a stated factor of MagNav.jl |
| **S3 INS + harness** | 9–11 | Pinson error simulator; outage harness; GitHub Actions CI; INS-only curves | simulator matches analytic drift within 10%; B0 curves on both test flights |
| **S4 Magnetic navigation** ★ | 12–17 | PF navigator; map model with upward continuation and a filled-cell mask; NIS gate and fault injection; B1 vs B0; README with headline figure and reproduce command | error stays bounded through 30-minute outages on both test flights; NEES/NIS and divergence rate reported; `reproduce` regenerates T2 and the figure; tag **v0.1** (**the minimum shippable portfolio**) |
| **S5 Learned compensation (AI-1)** | 18–22 | NN residual ensemble (5 members, ≤50k parameters, NLL loss, CPU); calibration; end-to-end ablation | beats ridge-TL in nT on validation; 95% coverage within ±5 points; under 1 ms per sample; result reported whatever its sign |
| **S6 Learned trust (AI-2)** | 23–26 | learned R compared with fixed, PF-spread and Sage-Husa R; the full 2×2 | mean NIS inside the χ² band; paired confidence intervals in T2 |
| **S7 Write-up** | 27–29 | error budget; sensor-noise ("quantum") sweep; final README; literature numbers verified | a fresh clone on another machine reproduces T1–T3; tag **v1.0** |

**Changes from the panel's 12-hour plan (author's adjustment for 8 hours, flagged in D3):**
- The pre-outage ESKF is **deferred**. The PF starts from a declared GPS/INS covariance, as Canciani did.
- This brings the minimum shippable portfolio forward from about week 20 to **about week 17**.
- The ESKF returns after v1.0, together with the cascade ablation and the tight-coupling stretch goal.

**No GPU:** every model is at most 50k parameters and trains on CPU, using the PyTorch CPU wheel. No step needs a GPU.

## 7. Learning path, just in time

| Slice | What to learn |
|---|---|
| S0–S1 | Geomagnetism basics (total field, crustal anomaly, IGRF, diurnal variation); the MagNav paper (Gnadt et al.); MagNav.jl docs |
| S2 | Tolles-Lawson derivation (permanent, induced and eddy-current terms); direction cosines; band-pass filtering; ridge regression |
| S3 | Solà §1–4 (rotations); NED frame, WGS-84, gravity, transport rate; IMU error models; Pinson error model and Schuler oscillation; NEES/NIS |
| S4 | Arulampalam particle-filter tutorial; Canciani & Raquet 2016 in depth; map upward continuation; why correlated measurements make filters overconfident |
| S5 | Heteroscedastic regression, β-NLL, deep ensembles, PIT and coverage calibration |
| S6 | Adaptive Kalman filtering (Sage-Husa, Mehra); learning noise covariances from innovations |
| Later | ESKF predict/update/reset (Solà); Joseph form |

## 8. Risk register

| Risk | Likelihood | Impact | Mitigation and trigger |
|---|---|---|---|
| Licence forbids portfolio use | M | H | Email the maintainers early; publish aggregates only. No reply by S4 → publish code and method, and state the restriction |
| `ins_*` is GPS-aided | H | M | The simulated INS is the primary baseline anyway; the S1 test decides |
| Fewer than two usable test flights | M | H | Split by line; leave-one-flight-out; add the 2021 flights. Triggered by the S1 EDA |
| PF diverges over flat or filled map areas | H | M | Mask invalid map cells; N_eff and multiple-peak checks; report the divergence rate. Triggered if more than 20% of runs diverge |
| A gain in nT doesn't reduce ATE | M | M | The ablation is fixed in advance; report the result honestly |
| Leakage (map, GPS, IGRF at the true position) | M | H | Excluded-input list; the R8 variants; review against ADR-0002 |
| Can't match MagNav.jl | M | M | Same flight lines, same terms; explain any gap in `docs/validation`. Triggered by a gap above 2× |
| Infrastructure creep | M | H | The trigger table in section 5 |
| Schedule slip | H | M | Slices are independent; S4 is the shippable point |

## 9. Decisions for the author

| # | Decision | Recommendation |
|---|---|---|
| D1 | Make the PF the navigator (R1); this replaces CLAUDE.md's design of feeding the PF fix into the ESKF | Yes |
| D2 | Replace CLAUDE.md's 16-week plan with this roughly 29-week plan at 8 hours a week (toy filters and strapdown INS dropped) | Yes |
| D3 | Defer the ESKF until after v1.0, so the minimum shippable portfolio arrives at about week 17 instead of about week 20 | Yes |
| D4 | Rebuild the environment with Python 3.12 and uv; delete `.venv` | Yes |
| D5 | Navigation-grade IMU for the headline; tactical grade as a sweep | Yes |
| D6 | Install Julia once to run the MagNav.jl oracle | Optional; skip if it takes more than 2 hours |
| D7 | Send the licence email this week | Yes |
| D8 | Accept a negative or mixed AI result in the README | Yes |
| D9 | Deadline | Not yet given |
| D10 | Drop Jetson and RoNIN from v1 | Yes |

## 10. Next five actions, once approved

1. Email magnav-admins@mit.edu about using the data in a public portfolio; record the licence position in `datasets/README.md`.
2. Install uv and Python 3.12; delete `.venv`; create `pyproject.toml` and `uv.lock`; add ruff, pytest, hypothesis and pyright; update the CLAUDE.md commands.
3. Commit the S0 skeleton: src layout, import-rule test, ADR-0001 to 0003, and a "Reproduce" stub in the README.
4. Write `scripts/download_data.py` and the SHA-256 manifest; download the data, inspect it with h5py, and commit a field inventory (field names only).
5. Write the first two data tests: the IGRF identity, and the free-inertial check on `ins_*`.
