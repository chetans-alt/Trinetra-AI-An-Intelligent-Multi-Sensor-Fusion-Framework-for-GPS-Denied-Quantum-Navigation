# Phase 7 — Sensor fusion

Source: trinetra_phase7_fusion.

## Why fuse

Each sensor alone fails (accel ∝ t², gyro ∝ t heading, mag distorted, GPS absent/jammed, baro weather drift) but the failures are complementary. Five pillars: complementarity, redundancy, accuracy, reliability, bias/drift compensation.

Inverse-variance fusion of two independent estimates: `x̂ = (σ₂²x₁ + σ₁²x₂)/(σ₁² + σ₂²)`, `1/σ² = 1/σ₁² + 1/σ₂²` — never worse than either input. The Kalman gain is its matrix generalisation.

## Frames and rotations

Body, navigation (ENU or NED), ECEF, geodetic. `vⁿ = R_bⁿ v_b`; chains compose by multiplication; pose via 4×4 homogeneous T. ENU ↔ NED: `[E,N,U] ↔ [N,E,−U]` — a frequent bug.

Rotation matrices: `RᵀR = I`, `det R = 1`, non-commutative, drift from orthogonality under integration. Euler ZYX `R = Rz(ψ)Ry(θ)Rx(φ)`; the Euler-rate Jacobian contains tanθ, secθ → gimbal lock at ±90°. **Keep state as a quaternion**, Euler only for display.

Quaternion: `q = [cos(α/2), u sin(α/2)]`; Hamilton product `[w1w2 − v1·v2, w1v2 + w2v1 + v1×v2]`; rotate `q⊗[0,v]⊗q*`; kinematics `q̇ = ½ q⊗[0,ω]`, renormalise.

## Dead reckoning

`q̇ = ½q⊗[0,ω]`, `v̇ = R(q)a − g`, `ṗ = v`. Gyro bias → p ∝ t³; accel bias → ½b t². Necessary between fixes, never sufficient.

## Estimation

State in this project: error-state with **position, velocity, quaternion, accel bias, gyro bias** — biases estimated online. Bayes filter: predict via Chapman–Kolmogorov `p(x_t|z_{1:t−1}) = ∫p(x_t|x_{t−1})p(x_{t−1}|z_{1:t−1})dx`, update `∝ p(z_t|x_t) p(x_t|z_{1:t−1})`. Multiplying Gaussians sharpens the belief.

## Kalman filter

Model `x_t = F x_{t−1} + B u + w (Q)`, `z = H x + v (R)`.
```
x̂⁻ = F x̂ + B u          P⁻ = F P Fᵀ + Q
y = z − H x̂⁻             S = H P⁻ Hᵀ + R
K = P⁻ Hᵀ S⁻¹
x̂ = x̂⁻ + K y             P = (I − K H) P⁻
```
Gain minimises tr(P) (Joseph form `(I−KH)P⁻(I−KH)ᵀ + KRKᵀ` is the numerically safer covariance update). R → 0 trusts the measurement; R → ∞ ignores it.

Worked check: F=[[1,1],[0,1]], H=[1,0], x̂=[0,1], P=I, Q=0.1I, R=0.5, z=1.2 → x̂⁻=[1,1], P⁻=[[2.1,1],[1,1.1]], S=2.6, K=[0.808,0.385], x̂=[1.162,1.077], position variance 2.1 → 0.40. Good unit test.

In the project: Q from Allan variance; R from measured noise, adapted online by the confidence module; **innovation y feeds the AI modules**.

## EKF, UKF, PF

- **EKF:** nonlinear f for the mean, Jacobians F = ∂f/∂x, H = ∂h/∂x for covariance. Workhorse for INS/GNSS and quaternion attitude. Weak under strong nonlinearity or large uncertainty; hand Jacobians are error-prone.
- **UKF:** 2n+1 sigma points `x̂ ± (√((n+λ)P))_i`, `λ = α²(n+κ) − n`; propagate through the true f, recombine with weights. Second-order accurate, no Jacobians, moderate cost.
- **Particle filter (SIR):** propagate particles with noise, weight by likelihood, resample (systematic) when `N_eff = 1/Σw² ` falls below a threshold. Handles multimodality. Curse of dimensionality ⇒ used only for the **low-dimensional magnetic map-matching sub-problem**, handing a position fix to the EKF/UKF.

## Algorithm comparison

KF (linear baseline), EKF (main INS/GNSS/mag fusion), UKF (strongly nonlinear attitude), PF (map matching), complementary `θ = α(θ_prev + ωΔt) + (1−α)θ_accel` (cheap edge attitude, a fixed-gain KF), Mahony/Madgwick (attitude backup), federated (per-sensor local filters + master), adaptive KF (online Q, R), deep / transformer fusion (learned displacement, long-range).

## Fusion architecture

Levels: low (raw, tight), feature, decision (loose). Topology: centralised, distributed/federated, hybrid. **Chosen: hybrid** — centralised error-state EKF/UKF tightly coupling IMU/mag/baro/GPS, wrapped by decision-level health/anomaly modules that can isolate a faulty sensor, plus a feature-level path for learned displacement and map matching.

## AI around the core (not instead of it)

Learned measurements (displacement, map match); adaptive Q, R and per-sensor weights from confidence/health; drift pre-correction; innovations → anomaly detection. KalmanNet keeps predict/update structure but learns K with an RNN.

Research gaps: learned adaptive Q/R vs fixed and innovation-adaptive baselines; uncertainty-aware learned measurements (TLIO-style); neural-Kalman for magnetic nav; learned vs χ² fault gating on injected faults. Transformers are the clear fit; GANs/VAEs for augmentation/anomaly; diffusion for imputation; RL for policy-level decisions only.

## Evaluation and debugging

Metrics: ATE/RTE, position RMSE, orientation error, drift rate, **NEES** and **NIS** (is the reported covariance honest?). Plots: trajectory vs truth, covariance ellipses, innovation sequences, per-sensor weights.

Divergence checklist: frame conventions, Q/R scale (too-small R over-trusts noise), timestamps, biased innovation ⇒ model or calibration error. Practices: well-tuned classical filter before AI; unit-test transforms; error-state formulation; validate with NEES/NIS; keep a dead-reckoning baseline. Mistakes: unnormalised quaternions, no GPS gating, R from moving data, unit mix-ups, forgetting gravity removal.

## Derivations to know

Kalman gain and covariance update; inverse-variance weighting; EKF from KF; sigma-point weights; PF weight update and systematic resampling; quaternion kinematics; gimbal lock; complementary filter as a KF special case; Joseph form; NEES/NIS.
