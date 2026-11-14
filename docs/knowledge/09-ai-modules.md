# Phase 9 — AI modules

Source: trinetra_phase9_ai.

## Why AI

The KF is optimal only for linear dynamics, Gaussian noise with known stationary covariance, correct models and no faults. Real GPS-denied navigation breaks all of these.

| KF assumption | Reality → module |
|---|---|
| Gaussian known noise | structured/nonlinear noise → denoising |
| Stationary bias | time/temperature-varying drift → drift prediction |
| No faults | hard/soft/intermittent faults → fault detection |
| Fixed R | context-dependent reliability → confidence + adaptive weighting |
| Known model | mismatch → neural-Kalman, PINNs |

**Modules wrap the core, never replace it.** Features flow up into the modules; corrected measurements, adaptive Q/R and weights flow into fusion; innovations flow back to fault/anomaly detection.

| Module | Input | Output → destination |
|---|---|---|
| Noise reduction | noisy window | clean signal → fusion input |
| Drift prediction | IMU/temperature window | bias forecast → pre-correct predict step |
| Fault detection | sensor features | fault flag → exclude/isolate |
| Sensor health | long-term features | health index → weighting/maintenance |
| Anomaly detection | measurement/innovation | score → gate outliers |
| Confidence | measurement + context | uncertainty → adaptive R/weights |

**⚠ Priority (CLAUDE.md):** confidence → adaptive R first, then drift/bias, then denoising, then one anomaly/fault module, then health, then trajectory. GANs, VAEs, RL, GNNs and XAI are future work.

## Noise reduction

Classical filters handle out-of-band, linear, stationary noise; learned denoisers target in-band, nonlinear, signal-dependent noise (vibration coupling, platform magnetic signature).
DAE loss `E‖x − g(f(x̃))‖²` with x̃ a corrupted input. 1-D conv encoder–decoder; Brossard 2020 uses a dilated CNN for gyros.
Options: CNN/TCN (real-time, primary), LSTM/GRU, Transformer, diffusion (offline, heavy).
Train on clean signals + synthetic sensor noise, or paired low/high-grade recordings. Metrics: SNR gain, RMSE, and **downstream ATE**. Highest-value target: the magnetometer platform signature.

## Drift prediction

Learn `b̂_{t+1} = f(X_{t−L+1:t})` with targets from a high-grade IMU or filter-estimated bias; correct `z_corr = z − b̂`, shrinking effective Q. Models: ridge (temperature → bias baseline), LSTM/GRU, TCN (dilated), Transformer/Informer. Evaluate forecast error vs horizon and drift reduction during simulated outages. Proactive complement to the filter's reactive bias estimation, attacking the t and t² terms.

## Fault detection

Hard, soft, intermittent faults; detection → isolation → recovery.
Residual test: NIS `yᵀS⁻¹y ~ χ²`; above threshold ⇒ fault. Learned: AE reconstruction error (soft/nonlinear), one-class SVM / isolation forest (novel), GNN (isolation across correlated sensors), Bayesian/GP. Gives graceful degradation.

## Sensor health

Gradual condition: health index h ∈ [0,1] from degradation features (noise floor, drift rate, innovation magnitude, autocorrelation decay), e.g. `h = σ(wᵀφ)`; RUL forecast with LSTM. Health lowers fusion weight (raises R) before a hard fault.

## Anomaly detection

Point, contextual, collective. Unsupervised. Methods: isolation forest, one-class SVM, LOF, AE, VAE, forecast residual, AnoGAN. Scores: reconstruction error; Mahalanobis `D² = (φ−μ)ᵀΣ⁻¹(φ−μ)` vs a χ² quantile. Evaluate with precision/recall, PR-AUC, point-adjusted F1 — not accuracy. Operates per measurement (vs fault detection per sensor over time). Inject synthetic faults (dropout, bias step, spikes) when no labels exist.

## Confidence scoring

Aleatoric (data noise, irreducible) vs epistemic (model ignorance, e.g. novel magnetic zone).
- Aleatoric: heteroscedastic NLL `‖y − μ(x)‖²/(2σ(x)²) + ½ log σ(x)²`.
- Epistemic: MC dropout (T stochastic passes, predictive variance) or deep ensembles (often strongest).
- **Calibration is non-negotiable:** reliability diagram, ECE, temperature scaling. An over-confident module tells the filter to trust bad data — worse than no AI.

## Model catalogue

Linear/ridge, RF, XGBoost/LightGBM (tabular health/fault baselines), SVM/one-class, KNN, MLP, **CNN/TCN** (denoise, drift, real-time), RNN, **LSTM/GRU** (drift, trajectory), Transformer, Informer, TFT, GNN, BNN, **KalmanNet** (learned gain in KF structure), **PINN** (`L_phys = ‖ṗ − v‖² + ‖v̇ − (Ra − g)‖²`). Start simple (trees, TCN); escalate only when data and need justify it.

## Future-work tracks (summarised)

- **Generative:** GAN `min_G max_D`, VAE ELBO with reparameterisation, diffusion noise-prediction loss. Useful for augmentation (TimeGAN), imputation, fault simulation, sim→real (CycleGAN). Never the estimator.
- **RL:** adaptive sensor management (state = health/confidence/innovation, action = weights/mode, reward = −error − compute). Supervised confidence weighting is the lower-risk path.
- **XAI:** SHAP, LIME, attention maps, integrated gradients, counterfactuals — to audit why a sensor was flagged.

## Training and evaluation rules

Split by trajectory/time, never shuffled windows. Early stopping on a trajectory-split validation set. Export TorchScript/ONNX; run light TCN/GRU on Jetson. Metrics: SNR/RMSE (denoise), forecast error vs horizon (drift), PR-AUC (fault/anomaly), ECE (confidence), and **end-to-end ATE/RTE and drift rate after fusion, ablated against the system without the module**.

Mistakes: training and testing on the same trajectory; optimising a module in isolation; trusting uncalibrated confidence.

## Research contributions (thesis-scale)

1. Confidence-driven adaptive fusion (calibrated uncertainty head sets R; vs fixed and innovation-adaptive baselines). — **the project's central question**
2. AI-denoised magnetic navigation on MagNav.
3. Neural-Kalman for magnetic-inertial navigation.
4. Physics-informed drift/denoise.
5. Explainable fault-robust fusion.

Niche: most learned navigation replaces or ignores the classical filter; this project keeps a physics core and wraps it with calibrated, ablated modules.

## Derivations to know

DAE loss; heteroscedastic NLL; MC dropout as approximate Bayes; NIS χ² test; Mahalanobis gate; ECE and temperature scaling; KalmanNet; PINN loss; confidence → R mapping.
