# Literature review

How each paper informs Trinetra-AI's three stages: **compensate → match → fuse**. Numbers are quoted from the papers themselves; nothing here is a Trinetra-AI result.

| # | Paper | Status |
|---|---|---|
| 1 | — | not yet received (Drive link private) |
| 2 | — | not yet received (Drive link private) |
| 3 | Hanley et al., *MagPIE: A Dataset for Indoor Positioning with Magnetic Anomalies*, IPIN 2017 | reviewed |
| 4 | Canciani & Raquet, *Absolute Positioning Using the Earth's Magnetic Anomaly Field*, NAVIGATION (ION), 2016 | reviewed |

---

## 4. Canciani & Raquet (2016): absolute positioning with the magnetic anomaly field

**Citation.** A. Canciani and J. Raquet, "Absolute Positioning Using the Earth's Magnetic Anomaly Field," *NAVIGATION: Journal of the Institute of Navigation*, 2016 (Air Force Institute of Technology). The copy we have is an early-view print marked "Vol. 0, No. 0", so the final volume and pages need checking before citing.

### Problem

Find a GPS alternative for aircraft that works everywhere, at any time, in any weather. The Earth's crustal magnetic field is a candidate: it is static, it is mapped worldwide, and it is hard to jam. But a magnetometer measures the *total* field, so the crustal signal has to be separated from:
- the core field;
- external time-varying sources, which the paper calls **temporal variations (TV)**.

### Key physics

| Fact | Value | Why it matters to us |
|---|---|---|
| Core (main) field | 95–99 % of the measured field | removed with the IGRF model; any leftover acts as a constant bias |
| Crustal anomaly | 1–5 % of the field, i.e. several hundred nT; effectively static | this is the navigation signal |
| Solar-quiet (Sq) daily variation | 20–40 nT at mid-latitudes; 100–200 nT near the equator (equatorial electrojet) | **India lies under the equatorial electrojet**, so temporal variations are larger there |
| Auroral currents | about 1500 nT | navigation near the poles is hard |
| Magnetosphere ring current | 20–30 nT when quiet; hundreds of nT in storms | storm robustness |
| Shortest crustal wavelength | about equal to the altitude *h* | altitude acts as a low-pass filter on the map |
| Highest temporal frequency seen from the crustal field | F = V/h (0.005–1 Hz for typical aircraft) | sets the band-pass limits in our preprocessing |
| Map aliasing (Reid 1980) | F_aliased = exp(−2πh/Δx); line spacing ≤ h/2 captures 99.97 % of the signal | a properly sampled map can be interpolated continuously |

### Method

The navigation filter is a **five-state particle filter** over INS *error* states: δlat, δlon, δv_n, δv_e and TV.
- **Dynamics:** the Pinson INS error model, truncated to the horizontal states. TV is modelled as a first-order Gauss–Markov (FOGM) process.
- **Altitude:** barometric aiding is applied inside the INS mechanisation, not in the filter.
- **Setup:** N = 1000 particles, initialised from the GPS/INS covariance at the moment GPS is lost.
- **Weighting:** a Gaussian likelihood of the measured value Z(t) against the expected value. The expected value is the reference field IGRF(x_ins), plus a 24-hour diurnal baseline O_B(t), plus the interpolated map value, plus the particle's TV.
- **Map lookup:** bivariate spline interpolation (SciPy).
- **Resampling:** sequential importance resampling after **every** measurement.
- **Removing temporal variations:**
  1. Subtract the monthly mean of the five quietest days recorded at an observatory.
  2. Model the remainder as FOGM.
  3. Fit σ and τ by autocorrelation over 2920 three-hour segments from one year at Boulder; about 98 % of segments have σ < 25 nT. The best predictor of the next segment's σ and τ was the previous segment's values; the Dst storm index correlated poorly.

### Data and results

The test is a **simulation built on real data**:
- the USGS Ohio aeromagnetic map at 500 m;
- INS and GPS truth from a real flight;
- *synthetic* magnetometer readings. These were built from map truth, an instrument error model (heading-dependent scale factor, moving bias and white noise) and real observatory temporal-variation records.

| Condition | Horizontal RMS error |
|---|---|
| 500 m altitude, quiet conditions, navigation-grade INS | **20 m** |
| 1 km / 3.5 km / 5 km / 10 km altitude | 45 / 139 / 156 / 171 m |
| Quiet vs storm conditions (Kp ≥ 3; 13 % of runs) | 21.51 m vs 30.66 m |
| Temporal variations *not* estimated | about 10× worse |

### Limitations

- **The aircraft's own magnetic interference is absent.** The magnetometer data is synthetic and assumes a stinger-mounted optically pumped caesium sensor with about 1 nT accuracy. Platform noise, which Tolles-Lawson compensation exists to remove, is never modelled.
- **Narrow test conditions.** Only one map region, one flight and a navigation-grade INS were used. The flight crossed a "relatively benign" part of the map.
- **Map error and altitude are not modelled.** The likelihood treats the map as perfect, and only horizontal states are estimated.
- **Coverage.** The temporal-variation model is stated to be limited to non-polar regions.

### What Trinetra-AI takes from it

1. **Blueprint for C6 map matching:**
   - state vector, Pinson dynamics and FOGM temporal-variation state;
   - likelihood (their eq. 11) and spline map lookup;
   - SIR resampling and N = 1000 particles.

   Implement this first as the classical map-matching baseline.
2. **Expected accuracy:**
   - About 20 m is achievable at low altitude with clean data, and accuracy degrades with altitude.
   - MagNav flights vary in altitude, so we report error grouped by altitude.
3. **Temporal variations are mandatory.** Ignoring them costs about 10× in accuracy. **Action:** check whether MagNav includes base-station or diurnal data. If it doesn't, fall back to the FOGM state.
4. **The gap our AI module 1 fills.** This paper assumes the platform interference problem away. Testing on *real* platform magnetometer data, with Tolles-Lawson plus learned compensation, is the step it leaves open.
5. **Inputs for adaptive R (AI module 2).** Error depends strongly on map gradient ("benign" areas), altitude and storm level. These are natural inputs for deciding how much to trust each map-match fix.
6. **Design question raised.** In this paper the particle filter *is* the navigation filter. In our design, the particle filter's fix feeds an error-state Kalman filter (ESKF) as a measurement, so the two filters share errors. Treating the fix as independent would make the ESKF overconfident.
   - **Mitigation:** check consistency with NEES/NIS. If it fails, inflate R or move to a tighter coupling.

---

## 3. Hanley et al. (2017): the MagPIE dataset

**Citation.** D. Hanley, A. B. Faustino, S. D. Zelman, D. A. Degenhardt and T. Bretl, "MagPIE: A Dataset for Indoor Positioning with Magnetic Anomalies," *IPIN 2017* (University of Illinois at Urbana-Champaign). Dataset: bretl.csl.illinois.edu/magpie.

### Problem

Indoor magnetic positioning methods couldn't be compared fairly, because earlier datasets had three gaps:
- they covered a single small building (e.g. 185 m²);
- they used handheld recordings only;
- they assumed the indoor field never changes.

### Dataset

| Item | Value |
|---|---|
| Size | 723 trials; more than 13 h, 51 km and 960 m² |
| Buildings | 3 (CSL 195 m², Talbot 295 m², Loomis 470 m²) |
| Platforms | handheld walking (WLK) and a wheeled ground robot (UGV) |
| Sensors | Moto Z Play phone: magnetometer about 50 Hz, accelerometer and gyroscope about 200 Hz |
| Ground truth | Google Tango (Lenovo Phab 2 Pro) with AprilTags; RMS ATE **2.9–4.2 cm**, checked against motion capture |
| Trial length | mean 90 s / 101 m; range 21–245 s / 23–297 m |
| Splits | training trials (for building maps) and test trials, split into *dead-load* (unchanged field) and *live-load* (objects moved) |
| Files | per trial: `Initial_DCM`, `Output_accel`, `Output_gyro`, `Output_mag`, `Output_gt` (.txt). Column 1 is time in seconds; then body-frame x, y, z in m/s², rad/s and µT; ground truth is position plus quaternion |
| Extras | Allan-deviation data from a 3908 s static test (for tuning Q); magnetometer-norm σ = 0.495 µT |

### Methods shown

- **Calibration:** hard-iron and soft-iron correction, m_c = Λ(m_raw − b_m), projecting readings onto a sphere of about 53 µT.
- **Initial orientation:** 30 s quasi-static start; the starting direction-cosine matrix is computed from gravity.
- **Map:** Gaussian-process regression (GPR) per field component, with a squared-exponential kernel and constant mean. The sparse approximation used 75k points, 2,220 inducing points and variational free energy. The map gives a **mean and a variance** at every point.
- **Localisation demo:** grid search over (x, y, azimuth) with a Gaussian likelihood built from the GPR mean and covariance, giving a global maximum-likelihood estimate (MLE) and a local maximum a posteriori (MAP) estimate with a 3 m prior.
- **Findings:**
  - Results differ strongly between buildings: the share of local-MAP likelihoods above threshold was 10.8–34.5 % for Loomis and 49.6–83.1 % for Talbot.
  - Live loads clearly distort the field. A table and chairs, a toolbox and an industrial fan all did; a bicycle and a metal bin barely did.

### Limitations

- The paper reports no navigation error in metres, only likelihood-threshold percentages from a grid search.
- Trials are short, under about 4 minutes, so long-outage drift can't be studied.
- There is no barometer and no GPS, and the magnetometer is consumer phone grade.
- The ground truth is itself an estimate (Tango, about 3–4 cm), validated in one motion-capture arena.
- Indoor anomalies come from building steel, not crustal geology. That physics differs from the airborne MagNav case.

### What Trinetra-AI takes from it

1. **Optional indoor extension.** MagPIE maps directly onto our canonical schema (m/s², rad/s, µT, time in seconds), so it is a cleaner choice than RoNIN for the indoor magnetic path.
   - The magnetometer is about 50 Hz and the IMU about 200 Hz, which exercises the resampling step.
   - Split by building or trial, never by shuffled windows.
2. **Live-load trials are a ready-made test set for AI module 4 (fault detection).** They are labelled "With Outliers". They also test whether adaptive R lowers trust in a disturbed fix.
3. **GPR map variance → measurement noise R.** A map that reports its own uncertainty gives AI module 2 a principled input: trust a fix less where the map is uncertain or flat. This matches Canciani's observation about benign map regions.
4. **Calibration formula and Allan data.** They confirm preprocessing step 4 (hard- and soft-iron calibration) and give real values for tuning Q.
5. **Evaluation practice.** Ground truth is validated with ATE (Sturm et al. 2012), the metric we already use. The building-to-building differences support testing on held-out environments.

---

## Synthesis

| | Canciani & Raquet 2016 | Hanley et al. 2017 | **Trinetra-AI** |
|---|---|---|---|
| Domain | airborne, crustal anomaly | indoor, building anomaly | airborne first (MagNav); indoor optional (MagPIE) |
| Magnetometer data | **synthetic** (map + error model) | real, phone grade | **real platform data** (MagNav) |
| Platform interference | ignored | handled by calibration only | **Tolles-Lawson + learned compensation** |
| Estimator | particle filter over INS error states | grid-search MLE/MAP | particle-filter map matching → ESKF |
| Map uncertainty | none | GPR variance | used as an input to adaptive R |
| Reported metric | horizontal RMS error (m) | likelihood percentages | ATE/RTE, drift rate, NEES/NIS, with ablations |
| Disturbances | temporal variations, storms | live loads | fault detection; adaptive trust |

**The research gap.** Canciani shows that map matching works when the magnetometer signal is clean. MagPIE shows that real fields are disturbed and that maps have uncertainty. Neither paper measures how much **learned compensation of real platform interference**, combined with **uncertainty-aware trust in each fix**, improves end-to-end navigation error during a GPS outage. That is Trinetra-AI's central question.

**Concrete decisions from this review:**
- **Baseline:** Canciani's five-state particle filter (with temporal variations) is the classical map-matching baseline.
- **Metrics:** report horizontal RMS error grouped by altitude, alongside ATE/drift, so results are comparable with Canciani's.
- **MagNav data check:** confirm whether MagNav includes temporal-variation or base-station data.
- **Architecture risk:** the particle filter → ESKF cascade must pass NEES/NIS consistency checks.
- **Indoor extension:** MagPIE replaces RoNIN, and its live-load trials become the fault-detection test set.

Papers 1 and 2 will be added when received.
