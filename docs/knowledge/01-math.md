# Phase 1 — Mathematics for the Kalman filter

Source: trinetra_math_notes. Only the load-bearing subset.

## Linear algebra

- **State is a vector**, e.g. `x = [px py pz vx vy vz]ᵀ`. Position error in metres is a norm: `‖p̂ − p_true‖`.
- Dot product `a·b = ‖a‖‖b‖cosθ` (alignment, angles); cross product (rigid-body rotation, gyro ↔ frame rate); unit vectors for headings and rotation axes.
- **Matrices are linear maps.** `(AB)ᵀ = BᵀAᵀ`; `AB ≠ BA`; inverse exists iff `det A ≠ 0`.
- Special matrices: covariances are **symmetric positive semi-definite**; rotations satisfy `RᵀR = I`, `det R = +1` (SO(3)), so `R⁻¹ = Rᵀ`.
- **Eigen-decomposition of P:** eigenvectors = axes of the uncertainty ellipsoid; √eigenvalues = 1σ lengths. P is PD ⇔ all eigenvalues > 0 — the sanity check that P is still a legal covariance. Eigenvalues of F govern whether errors grow; conditioning explains silent divergence.

## Calculus

- Dynamics `ẋ = f(x, u)`: `ṗ = v`, `v̇ = a`. Propagating the state = numerically integrating this; small biases integrate into drift.
- **Jacobian** `J = ∂f/∂x` (m×n) is the best local linear approximation. The EKF linearises at the estimate: `F = ∂f/∂x|x̂`, `H = ∂h/∂x|x̂`. If you can't hand-derive a small Jacobian you can't debug an EKF. UKF avoids Jacobians with sigma points.
- **Gradient descent** `θ ← θ − α∇L` trains every learned module (backprop = chain rule) and underlies least-squares calibration.

## Probability

- Mean, variance, σ (sensor noise level).
- Multivariate Gaussian `N(μ, Σ)`; `(x−μ)ᵀΣ⁻¹(x−μ)` is the squared **Mahalanobis distance** — used to gate anomalies (fault detection).
- **Bayes:** posterior ∝ likelihood × prior. Every recursive filter is Bayes once per step: predict = prior through the motion model (uncertainty grows); update = fold in the likelihood (uncertainty shrinks). Linear + Gaussian ⇒ closed form = Kalman filter. Non-Gaussian / multimodal ⇒ particle filter.

## Statistics

- Sample mean; unbiased variance with N−1.
- Covariance and correlation; off-diagonals of P couple states (e.g. position ↔ velocity error).
- Sensor noise = white noise (sets R) + bias (integrates to drift) + random walk (variance ∝ t). **Allan variance** separates them and justifies Q and R.
- Evaluation: position RMSE, orientation error (deg), drift rate (error per distance) — the honest GPS-denied metric.

## One Kalman step, annotated

```
Predict:  x̂⁻ = F x̂ + B u            P⁻ = F P Fᵀ + Q
Update:   K = P⁻Hᵀ (H P⁻ Hᵀ + R)⁻¹
          x̂ = x̂⁻ + K (z − H x̂⁻)     P = (I − K H) P⁻
```

| Element | Math |
|---|---|
| x̂, z | vectors |
| F, H, B, products, transposes | matrices |
| inverse in K | matrix inverse |
| F, H as Jacobians (EKF) | partial derivatives |
| P, Q, R | Gaussians / covariance |
| predict → update | Bayes prior → posterior |
| values in Q, R; RMSE | noise statistics |
| PD of P, stability of F | eigenvalues |
