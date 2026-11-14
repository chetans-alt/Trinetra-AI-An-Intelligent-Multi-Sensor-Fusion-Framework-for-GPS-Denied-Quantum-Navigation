# Phase 8 — Navigation systems

Source: trinetra_phase8_navigation.

## Framing

Navigation = PNT (position, navigation, timing). The core tension: smooth-but-drifting sources (inertial) vs absolute-but-sparse/fragile ones (GPS, magnetic map). Under GPS denial, bounding drift is the central problem.

## Coordinates

Geodetic (φ, λ, h) → ECEF: `N = a/√(1 − e² sin²φ)`; `x = (N+h)cosφcosλ`, `y = (N+h)cosφsinλ`, `z = (N(1−e²)+h)sinφ` (WGS-84 a = 6378137, e² = 6.69437999e-3). ECEF → ENU via rotation about the reference point. NED reorders and flips sign.

## GPS / GNSS

Three segments (space ~31 MEO satellites with atomic clocks; control; user). Pseudorange least squares; DOP; carrier-phase RTK needs integer ambiguity resolution. Errors: multipath, atmosphere, clocks/ephemeris, receiver noise, urban canyon, indoor/underwater, jamming/spoofing.

Constellations: GPS, GLONASS, Galileo, BeiDou, NavIC (regional, India), QZSS, SBAS (WAAS, EGNOS, GAGAN). Multi-GNSS improves geometry and availability but **does not solve denial**: all share weak, blockable, jammable signals.

In the project: when available, a χ²-gated GPS fix resets drift. Otherwise the system falls back to inertial + magnetic map matching + baro.

**⚠ Unverified:** the notes state that by early 2026 NavIC had only ~3–4 fully operational satellites after clock failures and NVS-02 missing orbit. Verify against primary sources before citing.

## INS

Strapdown: `q̇ = ½q⊗[0,ω]`, `v̇ⁿ = R(q)aᵇ − g`, `ṗⁿ = vⁿ`. Errors: gyro bias → heading ∝ t → v ∝ t² → p ∝ t³; accel bias → ½b t².

| Grade | Gyro bias | Unaided |
|---|---|---|
| Consumer MEMS | 10–100 °/h | metres in seconds |
| Tactical | 1–10 °/h | minutes |
| Navigation (FOG/RLG) | < 0.01 °/h | km over hours |

INS is the only fully self-contained, high-rate, unjammable motion source — the backbone whose drift everything else bounds.

## Dead reckoning (2-D)

`x += v cosψ Δt`, `y += v sinψ Δt`. Constant heading error δψ ⇒ cross-track error ≈ v t δψ. In the filter, DR is the high-rate predict step between fixes that may be seconds apart.

## Magnetic navigation

Field: intensity 25–65 µT, declination, inclination. Main field from WMM/IGRF; crustal anomalies (WDMAM) are a stable spatial fingerprint.

Map matching likelihood: `p(z|p) ∝ exp(−(z − B_map(p))²/(2σ²))`, fused with inertial motion by a Bayesian filter. Passive, unjammable, available indoors/underwater. Main challenge: the platform's own magnetic signature swamps the anomaly — where AI denoising earns its place (the core MagNav problem).

**⚠ Correction:** the notes call it "drift-free". It **bounds** drift. Industry examples (Boeing 2024, SandboxAQ AQNav, Q-CTRL Ironstone Opal) are unverified until checked against primary sources.

## Map matching

Variants: road, magnetic, terrain (TERCOM), vision. HMM formulation: hidden positions, emission `p(z|x)`, transition `p(x_t|x_{t−1})`, Viterbi decoding (Newson & Krumm 2009). For continuous fields use a **particle filter**: inertial motion propagates particles, the field weights them, resample; the weighted mean becomes the position fix to the EKF — the map→odom correction. Handles multimodality a KF can't.

## Localization

Global (kidnapped robot, multimodal) vs local (pose tracking). MCL = particle filter over pose. Project: Gaussian EKF/UKF for smooth tracking, PF for the multimodal magnetic sub-problem. The covariance is the reported navigation confidence.

## Path planning

A* (`f = g + h`, admissible h; h = 0 → Dijkstra), RRT/RRT*, PRM, potential fields, Hybrid A*. **Optional and downstream** — the project is a state-estimation framework. Add simple A* only if a demo needs motion.

## Navigation modes

| Mode | Drift | GPS-denied? |
|---|---|---|
| INS only | unbounded (t³) | yes, drifts |
| GPS only | none | no |
| INS + GPS | bounded while GPS present | partial |
| Magnetic | bounded by map | yes |
| **Magnetic-inertial (MINS)** | bounded | yes |
| Visual / VIO | bounded with loop closure | yes, needs light/texture |
| Quantum (mag + INS + AI) | very low, bounded | frontier |

**Project mode:** magnetic-inertial navigation with AI; baro bounds altitude; GPS resets drift when present; visual-inertial and quantum magnetometer are future channels into the same core.

## AI in navigation

Learned measurement models, trajectory prediction, drift prediction (LSTM/TCN), sensor health (autoencoder/classifier), learned map matching, adaptive weighting (attention), fault detection, calibrated confidence. AI wraps the core; the Kalman/particle core keeps interpretability and guarantees.

## Pipeline

sensors → sampling → calibration → filtering → fusion → state → localization → map matching (fix loops back to fusion) → navigation engine (mode select, drift bounding) → trajectory → confidence (loops back) → visualisation.

## Evaluation

ATE/RTE, RMSE, heading error, **drift rate in GPS-denied segments**, fix availability, NEES/NIS. Error-vs-time plots with outage windows shaded. Simulate outages by masking the GNSS channel on a dataset that has GPS truth.

Debugging: ENU/NED signs, units, sync, Q/R scale; biased innovation = model/calibration error. Validate each mode against DR-only and GPS-only baselines. Mistakes: forgetting declination, ungated multipath, ignoring platform magnetic signature, evaluating only with GPS.

## Research directions

AI-denoised magnetic map matching on MagNav; learned adaptive mode and Q/R selection; neural-Kalman magnetic-inertial navigation; confidence-aware mode switching; simulated quantum-magnetometer channel; magnetic SLAM.

## Derivations to know

Pseudorange solution and DOP; INS mechanisation and error orders; tilt-compensated heading; map-matching likelihood; HMM/Viterbi; MCL recursion; geodetic → ECEF → ENU; mode-switching logic.
