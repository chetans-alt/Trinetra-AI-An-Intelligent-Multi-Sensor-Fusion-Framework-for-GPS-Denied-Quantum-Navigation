# Phase 6 — Sensors

Source: trinetra_phase6_sensors.

A sensor implements `z = h(x) + errors`. Fusion is only optimal with a correct error model of each sensor, so this phase supplies Q, R, h(·) and bias states.

## Metrology

- Reading `z = x + b + ε`: accuracy ↔ bias b (remove by calibration); precision ↔ σ (averaging N samples cuts σ by √N, not b). MSE = b² + σ².
- Sensitivity `∂z/∂x`, scale-factor error `(1+k)x`, resolution, dynamic range `20 log10(x_max/x_min)`.
- First-order sensor: bandwidth `1/(2πτ)`; wider tracks fast motion but admits noise. Latency matters when fusing different rates.
- Non-idealities: bias, drift, noise, cross-sensitivity, hysteresis, saturation, dead zone, nonlinearity, quantisation, temperature.
- Uncertainty propagation: `σ_y² ≈ Σ (∂f/∂x_i)² σ_i²`.

## Accelerometer

Measures **specific force** (reads +g up at rest, 0 in free fall). Spring–mass `mẍ + cẋ + kx = −ma`; static `x = −a/ω_n²` (sensitivity vs bandwidth trade-off). MEMS read x capacitively (down to DC); piezo can't do DC.
Roll/pitch: `φ = atan2(a_y, a_z)`, `θ = atan2(−a_x, √(a_y² + a_z²))`; no yaw.
Errors: bias (→ ½b t²), VRW, scale, misalignment, temperature. Six-position tumble test calibrates bias and scale.

## Gyroscope

Coriolis on a vibrating mass: `a_c = −2 ω × v`. Bias → heading error ∝ t; white noise → ARW ∝ √t.

**Allan deviation (log–log):** slope −½ = angle random walk (read N at τ = 1 s); flat minimum = bias instability; slope +½ = rate random walk. These coefficients populate Q.

Grades: consumer MEMS 10–100 °/h; tactical 1–10 °/h; navigation (FOG/RLG) < 0.01 °/h. The project targets MEMS, so software drift compensation is where the value is.

## Magnetometer

Earth field 25–65 µT; declination δ (magnetic vs true north), inclination (dip).
Tilt-compensated heading:
```
m_hx = m_x cosθ + m_z sinθ
m_hy = m_x sinφ sinθ + m_y cosφ − m_z sinφ cosθ
ψ_true = atan2(−m_hy, m_hx) + δ      (δ from WMM)
```
Hard-iron offset b, soft-iron matrix A: `m_meas = A m_true + b` ⇒ `m_cal = A⁻¹(m_meas − b)`. Rotating through all orientations traces an ellipsoid; fit `(m−b)ᵀM(m−b) = 1` by least squares (SVD null-space), `M = A⁻ᵀA⁻¹`. **The most important calibration for GPS-denied heading.**

Indoor distortion and crustal anomalies (WDMAM) are spatially stable → a map to localise against. The magnetometer is the drift **anchor**; a quantum magnetometer makes the anchor sharper without changing the pipeline.

## IMU

6-DOF (accel + gyro), 9-DOF (+ mag, MARG), 10-DOF (+ baro). Attitude as Euler (gimbal lock at θ = ±90°), rotation matrix, or quaternion (singularity-free, preferred). `q̇ = ½ q ⊗ [0, ω]`, renormalise after each step.
Strapdown: ω → attitude → `R a − g` → v → p. Gyro bias → heading ∝ t → misrotated gravity → v ∝ t² → p ∝ t³; accel bias alone → p ∝ t².

## GPS

Pseudorange `ρ_i = ‖p − s_i‖ + c δt_u + ε`; four unknowns ⇒ ≥ 4 satellites; Gauss–Newton with rows `H_i = [−û_iᵀ 1]`; geometry sets DOP. Frames: geodetic → ECEF → ENU.
Errors: multipath (non-Gaussian jumps), atmosphere, clocks, ephemeris, signal loss, urban canyon, jamming/spoofing. **Gate before fusion** with a χ² test on the innovation `d² = yᵀS⁻¹y < threshold`.

## Barometer

`h = (T0/L)[1 − (P/P0)^{RL/gM}]` (standard atmosphere). Weak absolute accuracy (weather moves P0), excellent relative altitude. Re-reference P0 at start or from GPS. Bounds the weakest axis (vertical) in fusion. Canonical unit is Pa.

## Error taxonomy → mitigation

| Error | Detection | Correction |
|---|---|---|
| Bias | static mean | subtract; KF bias state |
| Noise | PSD | filter; set R |
| Drift | innovation trend, CUSUM | KF bias state; learned prediction |
| Random walk | Allan | aiding; tune Q |
| Scale / misalignment | multi-point calibration | `K⁻¹T⁻¹(z − b)` |
| Temperature | correlate with temperature | b(T) regression |
| Magnetic / EMI | spectral anomaly | hard/soft-iron calibration |
| Vibration | spectrogram peaks | notch / low-pass |

General IMU model: `z_meas = T K z_true + b + ε`. Calibration regimes: factory, offline (controlled motions), online (extra KF states).

## Drift by sensor

Gyro: heading ∝ t (mag + accel aiding, learned bias). Accel: ∝ t² (gravity reference, KF bias state). Mag: environmental change (recalibrate, map matching). GPS: jumps, not drift (gating). Baro: weather (re-reference).

## Practices

Characterise noise on **static** data; calibrate every sensor before fusion; version calibration parameters; unit-test frame transforms; keep raw copies and a classical dead-reckoning baseline. Common mistakes: skipping hard/soft-iron, ignoring misalignment, forgetting declination, not gating multipath, deg/rad and g/m·s⁻² mix-ups.

## Derivations to know

Spring–mass response; Coriolis; Allan slopes; tilt-compensated heading; ellipsoid calibration; quaternion kinematics; t/t²/t³ growth; pseudorange least squares; barometric formula; Q, R from Allan variance.
