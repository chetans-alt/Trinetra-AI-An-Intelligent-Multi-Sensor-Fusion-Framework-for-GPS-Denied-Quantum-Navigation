# Literature review

How each paper informs Trinetra-AI's three stages: **compensate → match → fuse**. Numbers are quoted from the papers themselves; nothing here is a Trinetra-AI result.

| # | Paper | Theme |
|---|---|---|
| A0 | Kalman, *A New Approach to Linear Filtering and Prediction Problems*, Trans. ASME J. Basic Eng. 82(D), 1960 | estimation foundations: the original Kalman filter |
| A1 | Pei, Biswas, Fussell & Pingali, *An Elementary Introduction to Kalman Filtering*, arXiv:1710.04055 (v5, 2019) | estimation foundations: fusion (ESKF) |
| A2 | Arulampalam, Maskell, Gordon & Clapp, *A Tutorial on Particle Filters for Online Nonlinear/Non-Gaussian Bayesian Tracking*, IEEE TSP 50(2), 2002 | estimation foundations: map matching (particle filter) |
| A3 | Thrun, *Probabilistic Algorithms in Robotics*, CMU-CS-00-126, 2000; and Thrun, *Probabilistic Robotics*, Comm. ACM 45(3), 2002 | framing: Bayes filters, Monte Carlo localisation |
| B1 | Canciani & Raquet, *Absolute Positioning Using the Earth's Magnetic Anomaly Field*, NAVIGATION (ION), 2016 | magnetic navigation (airborne) |
| B2 | Hanley et al., *MagPIE: A Dataset for Indoor Positioning with Magnetic Anomalies*, IPIN 2017 | magnetic navigation (indoor dataset) |
| C1 | Saputra, Lu, de Gusmão, Wang, Markham & Trigoni, *Graph-based Thermal-Inertial SLAM with Probabilistic Neural Networks*, arXiv:2104.07196 (v3, 2021) | learned measurement uncertainty inside a classical estimator |
| C2 | Kipf & Welling, *Semi-Supervised Classification with Graph Convolutional Networks*, ICLR 2017 | graph neural networks (future work) |

The two Thrun documents overlap heavily: the CACM article is a short version of the technical report. They are reviewed together as A3.

---

# Part A: Estimation foundations

## A0. Kalman (1960): a new approach to linear filtering and prediction problems

**Citation.** R. E. Kalman, "A New Approach to Linear Filtering and Prediction Problems," *Transactions of the ASME, Journal of Basic Engineering*, 82 (Series D):35–45, 1960.

The copy we have is a 2002 retyped transcription by J. Lukesh. The transcriber notes that equation (32) may contain an ordering error, so cite the original and take the equations from it or from A1.

### Problem

The **Wiener filter** was the standard method for estimating a signal from noisy measurements. Kalman lists four practical limits:
1. The filter is specified only by its impulse response, which is hard to turn into a working filter.
2. Computing that impulse response numerically is involved and poorly suited to computers.
3. Every extension, such as nonstationary statistics or finite data, needs a new derivation.
4. The underlying assumptions are hidden in the mathematics.

### Method

The paper makes three moves.

- **Optimal estimation as conditional expectation.**
  - **Theorem 1:** for symmetric, unimodal conditional distributions (Gaussian ones included), the conditional mean minimises *any* reasonable loss.
  - **Theorem 2:** the estimate is the **orthogonal projection** of the state onto the space spanned by the measurements. This holds if the processes are Gaussian, *or* if the estimator is restricted to be linear with squared-error loss.
  - Only first- and second-order statistics (means and covariances) are ever needed.
- **State-space models.** A random signal is modelled as the output of a linear system driven by white Gaussian noise: x(t+1) = Φx(t) + u(t), y(t) = Mx(t), with noise covariance Q.
  - **Coloured noise is handled by adding it to the state.** In Example 2, the measurement noise is a Gauss–Markov state (x₃ with coefficient φ₃₃), estimated alongside position and velocity.
- **Theorem 3: the recursive filter.** It is written as a one-step predictor:
  - gain Δ* = ΦP*Mᵀ(MP*Mᵀ)⁻¹;
  - closed-loop transition Φ* = Φ − Δ*M;
  - covariance update P*(t+1) = Φ*P*Φᵀ + Q.
  - Eliminating the gain gives a nonlinear (Riccati) equation for P*, which replaces the Wiener–Hopf integral equation.

### Key findings

- **One derivation covers everything:** stationary and nonstationary statistics, and finite or infinite data.
- **The filter's gain follows from its covariance P\*.** Once P\* is known, no further calculation is needed.
- **Innovations are white (Remark m).** The innovation sequence ỹ(t|t−1) = y − Mx\* is uncorrelated over time. The optimal filter is a feedback system driven by white noise.
- **Linear estimation is hard to beat.** It can be improved on *only* when the processes are non-Gaussian, and even then only by using third- or higher-order statistics.
- **Conditions for the recursion to work:**
  - **P\*(t₀) and Q must be specified.** If Q is positive definite, every P\* stays positive definite.
  - **No measurement may be a linear combination of the others,** or MP\*Mᵀ cannot be inverted.
- **Duality (Theorem 4).** Estimation is the mathematical dual of the noise-free optimal regulator (control) problem; both reduce to the same Riccati equation.
- **The model is assumed, not learned.** Obtaining Φ and Q from data is called "a subtle and presently largely unsolved problem". Kalman knows of no study that optimises the model and the filter jointly.

### Limitations

- The paper covers linear systems only. The extended Kalman filter (EKF) came later; see A1 and A2.
- There is no separate measurement-noise term. Noise is modelled as part of the state, unlike the modern form, which adds R.
- Stability and convergence of the filter are deferred to a later paper.
- Φ and Q are assumed known exactly.
- The examples are analytical; there is no real data.

### What Trinetra-AI takes from it

1. **The research gap is 60 years old.** Kalman calls identifying the noise model "largely unsolved" and notes that nobody had optimised the model and the filter together. Learned compensation and learned R are modern attempts at exactly that joint problem, which makes this a strong framing citation for the thesis introduction.
2. **White innovations are the basis of our feedback loop.** If the filter's model is right, the innovations are white. Any structure left in them (correlation over time, wrong magnitude) is a signal that Q or R is wrong. Two consequences:
   - **Diagnostics:** add innovation-whiteness checks (autocorrelation) alongside NIS.
   - **Justification for AI module 2:** using innovations as its input (the C5 → C4 loop) learns from exactly the signal Kalman's theory says should be white.
3. **Coloured noise becomes a state.** Example 2 is the same trick as Canciani's FOGM temporal-variation state (B1) and the IMU bias states in the ESKF. This paper is the original source for that design.
4. **Numerical rules for the code:**
   - Keep Q positive definite and never set it to zero; this keeps P positive definite (already in the test plan) and avoids the particle collapse A2 warns about.
   - Don't feed linearly dependent measurement channels, such as duplicate magnetometer components, unless R > 0 keeps the innovation covariance invertible.
5. **Nonlinear filtering needs a non-Gaussian problem to pay off.** A nonlinear filter beats a linear one only when the problem is non-Gaussian. Magnetic map matching has several peaks, so it is non-Gaussian; this supports the particle filter for matching (A2) while keeping the ESKF for single-peaked inertial fusion.
6. **Out of scope:** the estimation–control duality, noted only as background.

---

## A1. Pei et al. (2019): an elementary introduction to Kalman filtering

**Citation.** Y. Pei, S. Biswas, D. S. Fussell and K. Pingali, "An Elementary Introduction to Kalman Filtering," arXiv:1710.04055v5, 2019. If a peer-reviewed version is cited instead, check its venue first.

### Problem

Most introductions derive the Kalman filter from Bayesian inference with Gaussian noise. That hides which assumptions the filter actually needs, and leads to the common belief that the noise must be Gaussian.

### Method

The paper builds the Kalman filter from two elementary ideas.

1. **Optimal linear fusion of uncorrelated estimates.** For two scalar estimates, the minimum-variance weight is K = σ₁²/(σ₁²+σ₂²); in other words, **each estimate is weighted by its precision**, 1/σ².
   - The fused precision is the sum of the input precisions.
   - Fusing n estimates one at a time loses nothing; this is why a *recursive* filter is optimal.
   - For vectors: K = Σ₁(Σ₁+Σ₂)⁻¹ and Σ = (I−K)Σ₁.
2. **The best linear unbiased estimator (BLUE)** for parts of the state that are never measured directly: ŷ = μ_y + Σ_yx Σ_xx⁻¹(x − μ_x).
   - With an observation matrix H, this produces the standard update K = ΣHᵀ(HΣHᵀ+R)⁻¹. It is how hidden states, such as sensor biases, get corrected from measurements of other states.

**The Kalman filter** = prediction (x̂ = Fx̂ + Bu, Σ = FΣFᵀ + Q) followed by this fusion step. The paper also gives the covariance update in its general form, (I−KH)Σ(I−KH)ᵀ + KRKᵀ, before simplifying it to (I−KH)Σ.

### Key findings

- **Gaussian noise is not required** for the Kalman filter to be the best *linear* unbiased estimator. With Gaussian noise it is the best of *all* unbiased estimators. With non-Gaussian noise, a nonlinear estimator can do better.
- **The optimality rests on stated assumptions:**
  - sensors are **unbiased** (no systematic error);
  - process and measurement noise are **uncorrelated** with each other and over time;
  - the covariances Q and R are known.
- **Nonlinear extensions.** The EKF and UKF are described as heuristic extensions for nonlinear systems; the EKF linearises with Jacobians.

### Limitations

- It is a tutorial: the only worked example is a linear, two-state falling body.
- There is no treatment of error-state filters, rotations or consistency testing.

### What Trinetra-AI takes from it

1. **Adaptive R is a precision weight.** The fusion weight is the measurement's precision, so a wrong R shifts trust to the wrong source. That is exactly what AI module 2 (adaptive R) exists to correct, and this paper gives the cleanest justification for it in the write-up.
2. **Every violated assumption needs explicit handling:**

   | Assumption | Where it breaks in Trinetra-AI | How the design handles it |
   |---|---|---|
   | Unbiased sensors | IMU biases; temporal magnetic variations | accel and gyro bias states in the ESKF; temporal-variation state in the particle filter (B1) |
   | Uncorrelated noise | the particle filter's map fix and the ESKF share the same INS errors | inflate R or couple more tightly; verify with NEES/NIS (see B1) |
   | Known Q and R | real noise drifts over time | tune Q from Allan variance (B2 data); learn R (AI module 2) |
   | Gaussian, single-peaked belief | the map-matching likelihood has several peaks | use a particle filter for matching, not an EKF (see A2) |

3. **Implementation:** use the general covariance update (Joseph form) for numerical stability, and build the first linear filters in week 1 from this paper's derivation.

---

## A2. Arulampalam et al. (2002): a tutorial on particle filters

**Citation.** M. S. Arulampalam, S. Maskell, N. Gordon and T. Clapp, "A Tutorial on Particle Filters for Online Nonlinear/Non-Gaussian Bayesian Tracking," *IEEE Trans. Signal Processing*, 50(2):174–188, 2002.

### Problem

Online estimation when the dynamics or the measurements are nonlinear, or the noise is non-Gaussian. In those cases the exact Bayesian recursion has no closed form.

### Content

- **Optimal Bayesian recursion.** Prediction uses the Chapman–Kolmogorov equation; the update uses Bayes' rule.
  - It is tractable only in two cases: linear-Gaussian systems (the Kalman filter) and finite state spaces (grid filters).
- **Approximate methods:**
  - **EKF:** linearises the models with Jacobians and keeps the belief Gaussian.
  - **Approximate grid filters:** cost grows rapidly with state dimension, and resolution is fixed in advance.
  - **Particle filters:** represent the belief with weighted samples.
- **Sequential importance sampling (SIS).** Weights are updated as w ∝ w·p(z|x)p(x|x₋₁)/q(x|x₋₁,z).
- **Degeneracy is unavoidable.** Weight variance only grows over time. Its severity is measured by the **effective sample size, N̂_eff = 1/Σwᵢ²**.
- **Two remedies for degeneracy:**
  1. A good **importance density** (the distribution new particles are drawn from). The optimal one is p(x|x₋₁,z); drawing from the motion model alone (the "prior") is common but wasteful.
  2. **Resampling when N̂_eff < N_T.** The authors prefer **systematic resampling**, which runs in O(N) and adds the least Monte Carlo variance.
- **The cost of resampling.** It causes **sample impoverishment**: particles collapse onto a few points, which is worst when process noise is small.
- **Variants:**
  - **SIR (bootstrap):** draws from the motion model and resamples every step. It is simple, but sensitive to outliers and loses diversity.
  - **ASIR (auxiliary particle filter):** better when process noise is small.
  - **RPF (regularised particle filter):** resamples from a kernel-smoothed posterior (Epanechnikov kernel, h_opt) to fight impoverishment.

### Results

A benchmark scalar nonlinear model with 50 particles, RMSE averaged over 100 Monte Carlo runs:

| Filter | RMSE |
|---|---|
| EKF | 23.19 |
| approximate grid (50 cells) | 6.09 |
| SIR | 5.54 |
| RPF | 5.55 |
| ASIR | 5.35 |
| "likelihood" particle filter | 5.30 |

- **EKF failure mode.** On this two-peaked problem the EKF picks the wrong peak or sits between the peaks, and is **highly overconfident about a biased estimate**. The authors note this inconsistency is hard to detect online.
- **The key design choice.** The authors' main lesson is that the importance density matters more than any other refinement.

### Limitations

- The tutorial uses only one scalar toy example; there are no navigation results.
- The regularised filter's kernel is optimal only for equal weights and a Gaussian posterior, and works less well as the state dimension grows.

### What Trinetra-AI takes from it

1. **Why map matching uses a particle filter.** A magnetic map repeats similar anomaly values in many places, so the position belief has several peaks: exactly the case where the EKF failed (23.19 vs about 5.5). This is the justification for the particle filter in C6.
2. **Improving on Canciani's resampling.** B1 resamples (SIR) after *every* measurement. The tutorial warns this loses diversity and is outlier-sensitive.
   - Design: resample systematically only when N̂_eff falls below a threshold.
   - Fall back to the regularised filter if particles collapse, which is likely because INS-error process noise is small.
   - Log N̂_eff as a health signal.
3. **Gate the fix before sending it to the ESKF.** The ESKF assumes a single-peaked Gaussian fix. Only send it when the particle cloud has a single peak (compact spread, healthy N̂_eff), and derive its covariance R from the particle spread. AI module 2 can then refine that R.
4. **Outlier sensitivity sets a role for AI module 4.** Residual platform interference, or live loads as in B2, produces outlier measurements that SIR handles badly. A fault-detection module, or a heavy-tailed likelihood, protects the particle weights.
5. **An evaluation idea.** Report EKF-vs-particle-filter matching on the same data as an ablation; the tutorial predicts the EKF will be overconfident, which NEES will expose.

---

## A3. Thrun (2000, 2002): probabilistic robotics

**Citations.**
- S. Thrun, "Probabilistic Algorithms in Robotics," Tech. Rep. CMU-CS-00-126, Carnegie Mellon University, 2000.
- S. Thrun, "Probabilistic Robotics," *Communications of the ACM*, 45(3):52–57, 2002.

The textbook of the same name (Thrun, Burgard & Fox, 2005) is already in [references.md](knowledge/references.md).

### Problem

Robots must act despite uncertain sensing, imperfect models and dynamic environments.

### Central claim

> "A robot that carries a notion of its own uncertainty and that acts accordingly, will do better than one that does not."

### Content

- **Probabilistic models.** Motion is modelled as p(x′|u,x) and sensing as p(z|x), replacing deterministic kinematics.
- **The Bayes filter recursion,** b_t(s) = η p(o|s,m) ∫ p(s|a,s′,m) b_{t−1}(s′) ds′.
  - It covers Markov localisation, Kalman filters, HMMs and dynamic Bayes networks, and rests on the Markov ("static world") assumption.
- **Monte Carlo localisation (MCL).** A particle filter over pose given a map. It handles:
  - **global localisation**, where the initial position is unknown;
  - the **kidnapped-robot** case, recovering after the robot is moved without being told;
  - beliefs with several peaks.

  It works with crude models, because the uncertainty absorbs model error, and needs as few as about 100 samples with refinements. It can also adapt its sample count to the computing budget available.
- **Mapping:**
  - EKF-SLAM (O(N²) memory, O(N³) time; needs uniquely identifiable features);
  - EM, which treats the path as a missing variable;
  - occupancy grids with log-odds updates.
- **Control:**
  - POMDPs, which are intractable beyond small problems;
  - **coastal navigation**, which plans with only (most likely state, entropy) and keeps the robot near features that keep it localised. It trades a longer path for a higher success rate.
- **Evidence.** The museum robots Rhino and Minerva covered more than 60 km. Rhino had one major localisation failure in six days, which coincided with a sonar fault.

### Limitations

- These are survey and position papers with no controlled benchmarks.
- The mild-violation claim about the Markov assumption is asserted, not quantified; the report itself lists its theory as an open problem.
- The examples are indoor robots with laser and sonar sensors, not inertial or magnetic navigation.

### What Trinetra-AI takes from it

1. **Framing for the write-up.** Trinetra-AI *is* a Bayes filter, and magnetic map matching is MCL in which the map m is the anomaly map and p(z|s,m) is the magnetic likelihood. That places the project in a well-established line of work.
2. **Where the static-world assumption fails.** Temporal magnetic variations (B1) and live loads (B2) both break it. The design answers each explicitly: a temporal-variation state for the first, fault detection for the second.
3. **Initialisation.** Starting with map-wide particles (the global-localisation and kidnapped-robot cases) is a stress test worth adding after the baseline, which starts from the GPS/INS covariance as in B1.
4. **Learning p(z|x) is the open area this project works in.** Thrun names learning probabilistic models as a key open area. Learned compensation (AI module 1) and learned R (AI module 2) are exactly that: learning parts of p(z|x).
5. **Future work: magnetic coastal navigation.** Plan routes over magnetically informative regions (high map gradient) so the vehicle stays localised. This is listed as future work only, because the project scope is estimation, not planning.
6. **Out of scope:** mapping and SLAM. The project uses existing anomaly maps.

---

# Part B: Magnetic-anomaly navigation

## B1. Canciani & Raquet (2016): absolute positioning with the magnetic anomaly field

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
- **Resampling:** sequential importance resampling after **every** measurement (see A2 for why we change this).
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
   - N = 1000 particles.

   Implement this first as the classical map-matching baseline, with the resampling changes from A2.
2. **Expected accuracy:**
   - About 20 m is achievable at low altitude with clean data, and accuracy degrades with altitude.
   - MagNav flights vary in altitude, so we report error grouped by altitude.
3. **Temporal variations are mandatory.** Ignoring them costs about 10× in accuracy. **Action:** check whether MagNav includes base-station or diurnal data. If it doesn't, fall back to the FOGM state.
4. **The gap our AI module 1 fills.** This paper assumes the platform interference problem away. Testing on *real* platform magnetometer data, with Tolles-Lawson plus learned compensation, is the step it leaves open.
5. **Inputs for adaptive R (AI module 2).** Error depends strongly on map gradient ("benign" areas), altitude and storm level. These are natural inputs for deciding how much to trust each map-match fix.
6. **Design question raised.** In this paper the particle filter *is* the navigation filter. In our design, the particle filter's fix feeds an error-state Kalman filter (ESKF) as a measurement, so the two filters share errors. That breaks A1's uncorrelated-noise assumption, and treating the fix as independent would make the ESKF overconfident.
   - **Mitigation:** check consistency with NEES/NIS. If it fails, inflate R or move to a tighter coupling.

---

## B2. Hanley et al. (2017): the MagPIE dataset

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

# Part C: Learned models inside estimators

## C1. Saputra et al. (2021): graph-based thermal-inertial SLAM with probabilistic neural networks

**Citation.** M. R. U. Saputra, C. X. Lu, P. P. B. de Gusmão, B. Wang, A. Markham and N. Trigoni, "Graph-based Thermal-Inertial SLAM with Probabilistic Neural Networks," arXiv:2104.07196v3, 2021 (University of Oxford). The PDF uses a journal template but names no venue, so confirm where it was published before citing. Code: github.com/risqiutama/ti-slam.

### Problem

Navigate where cameras and lidar fail: darkness, smoke and dust. Thermal cameras still work there, but their images are low-contrast and nearly textureless, so classical feature tracking loses track. The camera also pauses for 0.5–1 s now and then to recalibrate itself (non-uniformity correction).

### Method

The system has a learned front end and a classical back end.

- **Front end:** three neural networks turn sensor data into constraints for the optimiser.
  - **Neural thermal-inertial odometry** estimates the motion between frames.
    - **Inputs:** 14-bit thermal images and the IMU.
    - **Network:** a feature extractor, a "hallucination" branch that imitates RGB-camera features, a selective-fusion mask that re-weights features by how reliable each source is at that moment, and two stacked LSTM layers.
    - **Output:** a **mixture density network (MDN)**. Each pose component is a Gaussian mixture, with a mean, a variance and mixing weights.
  - **Neural embedding** (ResNet50, 128-dimensional vector per image, trained with a triplet loss) detects when the robot revisits a place. Its training labels come from the RGB camera.
  - **Neural loop closure** estimates the relative pose between revisited places, using the same mixture-density output.
- **Back end:** robust pose-graph optimisation (g2o, Levenberg–Marquardt).
  - **Objective (their eq. 7):** Σ‖odometry residual‖² weighted by ϱΣ̂, plus Σ‖loop-closure residual‖² weighted by ρΛ̂. In other words, **the network's predicted variances become the measurement covariances**.
  - **Outlier rejection:** pairs of loop closures are checked for geometric consistency (an extension of GraphTinker), and any that disagree are dropped.
- **Training:**
  - A Huber loss protects the hallucination branch from outliers during the camera's recalibration pauses.
  - The mixture-density networks are trained by negative log-likelihood.
  - The predicted variances get **no direct supervision**.

### Results

| Test | Odometry only ATE | Full SLAM ATE | Gain |
|---|---|---|---|
| Ground robot, 13 sequences | 0.609 m | **0.281 m** | 53.9 % |
| Handheld, 8 sequences | 4.397 m | **1.334 m** | 62.8 % |
| SubT tunnel, 54 min | 32.05 m | **19.28 m** | about 40 % |

**Ablation: learned covariance vs identity covariance** (sequence 43, odometry ATE 0.592 m):

| Covariance used | SLAM ATE | Gain |
|---|---|---|
| identity for both odometry and loops | 0.348 m | 41.3 % |
| learned for odometry only | 0.172 m | 71.1 % |
| learned for loop closures only | 0.221 m | 62.2 % |
| **learned for both** | **0.121 m** | **79.5 %** |

**Other findings:**
- **Number of mixture components (K):** 10 gave the best ATE (0.609 m), compared with 0.793 m for a single Gaussian (K = 1). Above 10, accuracy got worse.
- **Uncertainty tracks error.** Predicted variance correlates with the actual error and rises during sharp turns and U-turns; errors mostly stay inside the 3σ bounds.
- **Selective fusion adapts to conditions.** When the thermal image has no contrast (a poster at wall temperature), the fusion mask shifts weight to the IMU.

### Limitations

- **The learned variances are not calibrated as they come out.** They need hand-tuned scale factors (ϱ = 0.01, ρ = 3). The authors show that results depend on ρ, and they name "training a model to predict the back-end parameters" as future work.
  - Uncertainty is checked only by correlation and 3σ plots; there is no NEES or NIS test.
  - The covariance is diagonal, assuming the pose components are independent.
- **Not real time.** The model has 273 M parameters (547 MB) and takes about 0.5 s per inference on a CPU, so the system runs offline. It reaches 26 Hz only on a TITAN V GPU.
- **Poor transfer.** The networks must be fine-tuned whenever sensor placement changes (domain shift).
- **Weak ground truth.** Most test data uses lidar SLAM or wheel odometry as stand-in ground truth; the smoke test is qualitative only.
- **Rotations use Euler angles,** which is safe only for near-planar motion.

### What Trinetra-AI takes from it

1. **The strongest direct evidence for AI module 2.** Replacing fixed (identity) covariances with learned, input-dependent ones cut the optimised ATE from 0.348 m to 0.121 m. That is the mechanism Trinetra-AI proposes: learned R for map-match fixes inside the ESKF. Cite it as precedent, in a different sensor domain.
2. **Its gap is our contribution.** Their variances needed hand-tuned scaling and were never checked for consistency. Trinetra-AI should:
   - calibrate the learned R, by fitting the network on NIS or innovation likelihood, or with a single learned scale;
   - report NEES/NIS to show the variances are consistent, not just correlated with error.

   This answers the authors' own future-work item on learning the back-end weights.
3. **A training recipe for learned compensation.**
   - Fit the noise model by negative log-likelihood. A network that outputs mean plus variance (heteroscedastic regression) gives AI module 1 a corrected field *and* its uncertainty, which can feed straight into R.
   - Use a Huber loss for transient interference events, such as manoeuvres or switching on a radio, much as they used it for the camera's calibration pauses.
4. **Mixture outputs for map ambiguity.** An MDN can represent position beliefs with several peaks. It is a possible learned alternative to the particle filter; this is out of scope for now, so the particle filter stays (A2).
5. **Gate fixes before fusing them.** Their consistency-based loop-closure rejection is the same idea as gating map-match fixes. It supports a geometric or innovation gate before fusion, and reinforces AI module 4.
6. **Keep models small.** Their 273 M-parameter network ran offline, while Trinetra-AI must run in real time (acceptance criterion). Compensation and noise models should be small (an MLP or small temporal CNN), and inference timing should be reported.
7. **Expect poor transfer between platforms.** Changing sensor placement forced retraining. MagNav's sensors sit in different positions on the aircraft and the flights differ, so split by flight, and report whether a compensation model trained on one sensor transfers to another.
8. **A future-work option: factor graphs.** Pose-graph and factor-graph back ends are a recognised alternative to filtering. Smoothing an ESKF trajectory with a factor graph is listed as future work, not baseline scope.

---

## C2. Kipf & Welling (2017): semi-supervised classification with graph convolutional networks

**Citation.** T. N. Kipf and M. Welling, "Semi-Supervised Classification with Graph Convolutional Networks," *ICLR 2017* (arXiv:1609.02907).

### Problem

Classify the nodes of a graph, such as documents in a citation network, when only a small fraction of nodes are labelled (0.1–5 %). Earlier methods either assumed that linked nodes share labels (Laplacian regularisation) or used multi-step embedding pipelines (DeepWalk, Planetoid).

### Method

- **Starting point: spectral graph convolution,** g_θ ⋆ x = U g_θ Uᵀ x. Computing it is expensive (O(N²), plus an eigendecomposition).
- **Simplification in three steps:**
  1. Approximate the filter with Chebyshev polynomials (Hammond et al. 2011).
  2. Truncate to first order (K = 1) and set λ_max ≈ 2.
  3. Use a single shared parameter, plus a **renormalisation trick** that adds self-loops for numerical stability.
- **Propagation rule:** H⁽ˡ⁺¹⁾ = σ(D̃^−½ Ã D̃^−½ H⁽ˡ⁾ W⁽ˡ⁾).
  - Each layer mixes every node's features with its neighbours' features, normalised by node degree.
  - Cost is linear in the number of edges.
- **Model:** two layers, softmax output, cross-entropy on the labelled nodes only, full-batch training.

### Results

| Dataset | GCN accuracy | Best baseline |
|---|---|---|
| Citeseer | 70.3 % | ICA 69.1 % |
| Cora | 81.5 % | Planetoid 75.7 % |
| Pubmed | 79.0 % | Planetoid 77.2 % |
| NELL | 66.0 % | Planetoid 61.9 % |

- **Speed:** GCN also trains faster than Planetoid on three of the four datasets.
- **Renormalisation:** the trick beat the Chebyshev and plain first-order variants.
- **Depth:** 2–3 layers is best. Beyond about 7 layers, training needs residual connections.

### Limitations

- **Setting:** the tests are transductive (training and test nodes sit in one fixed graph) and use only undirected graphs.
- **Memory:** full-batch training needs memory that grows with the graph.
- **Weighting:** a node's own features and its neighbours' features are weighted equally.
- **Domain:** there is no time-series, sensor or navigation application.

### What Trinetra-AI takes from it

1. **Scope.** GNNs are **future work** under the project's rules (CLAUDE.md), so this paper does not change the baseline plan. It is kept as the foundational GNN citation for the future-work section.
2. **Candidate future uses**, each untested and with an obvious baseline to beat:
   - **A graph of the aircraft's sensors.** MagNav records several magnetometers at different positions. A GCN over a sensor graph could share interference information between them during compensation. Compare against a plain MLP on concatenated channels: the paper's own MLP baseline did far worse (55.1 % vs 81.5 % on Cora), but only where the graph carries real information.
   - **The anomaly map as a grid graph,** for learning which regions are informative for navigation (input to adaptive R). Compare against the simple map-gradient feature from B1.
3. **Practical notes if it is ever used:**
   - Use 2–3 layers.
   - Check that the method works on graphs it wasn't trained on (inductive use), since the paper only tests within one graph.
   - Implement in PyTorch. The paper's code uses TensorFlow, which this project excludes.

---

# Synthesis

## Magnetic-navigation papers compared

| | Canciani & Raquet 2016 | Hanley et al. 2017 | **Trinetra-AI** |
|---|---|---|---|
| Domain | airborne, crustal anomaly | indoor, building anomaly | airborne first (MagNav); indoor optional (MagPIE) |
| Magnetometer data | **synthetic** (map + error model) | real, phone grade | **real platform data** (MagNav) |
| Platform interference | ignored | handled by calibration only | **Tolles-Lawson + learned compensation** |
| Estimator | particle filter over INS error states, resampling every step | grid-search MLE/MAP | particle filter (N̂_eff-triggered resampling) → gated fix → ESKF |
| Map uncertainty | none | GPR variance | used as an input to adaptive R |
| Reported metric | horizontal RMS error (m) | likelihood percentages | ATE/RTE, drift rate, NEES/NIS, with ablations |
| Disturbances | temporal variations, storms | live loads | fault detection; adaptive trust |

## The research gap

Canciani shows that map matching works when the magnetometer signal is clean. MagPIE shows that real fields are disturbed and that maps have uncertainty. Neither paper measures how much **learned compensation of real platform interference**, combined with **uncertainty-aware trust in each fix**, improves end-to-end navigation error during a GPS outage. That is Trinetra-AI's central question.

The foundations papers explain *why* this gap matters:
- **A0:** learning the noise model was already an open problem in 1960, and white innovations are the test of whether it is right.
- **A1:** a filter's weights are only as good as its stated noise.
- **A2:** EKFs become overconfident on beliefs with several peaks, and particle filters are sensitive to outliers.
- **A3:** probabilistic models need only be approximately right, and learning them is the open problem.

**C1** gives the closest precedent: learned, input-dependent covariances inside a classical estimator cut ATE from 0.348 m to 0.121 m in thermal-inertial SLAM. But those covariances needed hand-tuned scaling and were never tested for consistency. Trinetra-AI moves this idea to magnetic navigation and adds the missing calibration and NEES/NIS checks.

## Design decisions from this review

| Decision | Source |
|---|---|
| Classical map-matching baseline: Canciani's five-state particle filter with a temporal-variation state | B1 |
| Resample systematically when N̂_eff < N_T, not every step; regularised particle filter if particles collapse; log N̂_eff | A2 |
| Send the particle filter's fix to the ESKF only when the belief has a single peak; R from particle spread, refined by AI module 2 | A2, A1 |
| Track IMU bias and temporal variations as states, because the filter's optimality assumes unbiased inputs | A0, A1, B1 |
| Check innovation whiteness (autocorrelation) alongside NIS; feed innovations to AI module 2 | A0 |
| Keep Q positive definite; don't fuse linearly dependent measurement channels | A0 |
| Check that the particle filter → ESKF cascade passes NEES/NIS, because its noises are correlated | A1, B1 |
| Report horizontal RMS error grouped by altitude alongside ATE/drift, so results are comparable with Canciani | B1 |
| Ablation: EKF vs particle-filter map matching | A2 |
| MagNav: confirm whether temporal-variation or base-station data is included | B1 |
| Indoor extension: MagPIE replaces RoNIN; its live-load trials become the fault-detection test set | B2 |
| Future work: plan routes over magnetically informative regions ("magnetic coastal navigation") | A3 |
| AI module 2: learned, input-dependent R, ablated against fixed R; calibrate it and report NEES/NIS (C1 did neither) | C1 |
| AI module 1: train by negative log-likelihood with mean + variance output; Huber loss for transient interference | C1 |
| Keep learned models small and report inference time (real-time acceptance criterion) | C1 |
| Test whether compensation transfers across sensors and flights; split by flight | C1 |
| Future work: factor-graph smoothing back end; GCN over the sensor graph or map graph | C1, C2 |
