# Phase 4 — Time series

Source: trinetra_phase4_ts (PDF and markdown versions, same content).

## Why it matters

Order carries information; navigation integrates sequences. A constant accel bias b gives position error `½bt²`. Unbounded drift under GPS denial is a temporal, forecastable phenomenon.

## Fundamentals

- `f_s` sampling rate, `T_s = 1/f_s`; Nyquist `f_s > 2f` or high-frequency content aliases and can masquerade as drift.
- **Time synchronisation** of multi-rate, multi-clock sensors is the most common silent failure of a fusion stack.
- Autocorrelation `ρ(k) = γ(k)/γ(0)`; physical signals are autocorrelated at short lags, white noise is not.
- Decomposition `x = trend + seasonal + cyclic + noise`. Sensor meanings: trend = gyro bias drift; seasonal = loop-repeating magnetic distortion; cyclic = gait; outliers = GPS multipath, magnetometer spikes near motors; missing = dropped packets / GPS loss (mask, triggers denied mode).
- Stationarity (weak: constant mean, variance, γ(k)); test with ADF. Moving-vehicle signals are non-stationary; **drift is a non-stationarity**.

## Sensor error model

`z = s + b + k·s + n + w` (true + bias + scale error + white noise + random walk). Growth laws: white gyro noise integrated → angle error ∝ √t (ARW); bias → ∝ t; double-integrated accel bias → ∝ t².

| Sensor | Units | Rate | Dominant error |
|---|---|---|---|
| Accelerometer | m/s² | 100–1000 Hz | bias, vibration; |a| ≈ g at rest → roll/pitch, not yaw |
| Gyroscope | rad/s | 100–1000 Hz | **bias drift**, ARW |
| Magnetometer | µT | 10–100 Hz | hard/soft-iron `m_cal = A⁻¹(m_raw − b)`; anomalies are a map |
| GPS | deg, m | 1–10 Hz | multipath, dropouts |
| Barometer | hPa, m | 10–100 Hz | weather drift |
| Quantum mag (future) | nT | 10–1000 Hz | shot noise, low drift |

Preprocessing order: sync → calibrate → denoise (without eating dynamics) → handle gaps (mask, never interpolate long outages) → normalise per channel.

## Windows

`N = ⌊(T − L)/s⌋ + 1`, overlap `1 − s/L`. Sliding (s < L), tumbling (s = L), expanding. Choose L from the physics (longest relevant timescale; ~1 s gait at 100 Hz ⇒ L ≳ 100), stride from the latency budget. **Split train/test by trajectory or time, never shuffled windows** (overlap leaks the future). Recipes: 200-sample IMU window → 3-D displacement (RoNIN/TLIO); mag window → map match; GPS residual window → spoofing detector.

## Forecasting

One-step vs multi-step; recursive (errors compound ~`Σλʲε`, exponential if λ > 1) vs direct vs seq2seq. Teacher forcing causes exposure bias; mitigate with scheduled sampling.

| Target | Forecastable? |
|---|---|
| Drift / bias | **yes** — the ideal target; subtract before the predict step |
| Position | short horizon only; GPS-denied position is the filter's job, not a forecaster's |
| Velocity, orientation | short horizon |
| Magnetic field | only given map + position |
| Sensor failure | as classification |

## Sequence models

| Model | Notes | Fit |
|---|---|---|
| RNN | vanishing/exploding gradients (product of Jacobians) | baseline only |
| LSTM | additive cell update fixes vanishing gradients; O(T), sequential | drift, denoise (primary) |
| GRU | two gates, fewer params | edge |
| TCN | dilated causal convs; RF = `1 + Σ(k−1)d_l`, `d_l = 2^l`; parallel, deterministic latency | real-time drift/displacement |
| Transformer | `softmax(QKᵀ/√d)V` + positional encoding; O(L²), data-hungry | long-range multi-sensor |
| Informer | ProbSparse, O(L log L) | long-horizon forecasting |

**Stance:** start with LSTM or TCN (proven by RoNIN/TLIO); Transformers only with enough data and a genuine long-range need.

## Drift

Physical drift types: bias, temperature, gyro (∝ t), accel (∝ t²), magnetic. ML drift: data, concept, model drift — detect to trigger retraining or fall back to the classical filter.

Detection families: statistical (CUSUM, Page–Hinkley, ADWIN, KS), traditional (high-pass, temperature regression), Kalman (bias states + innovation monitoring), ML (one-class SVM, isolation forest), DL (LSTM/TCN forecast, autoencoder residual), adaptive (LMS/RLS, adaptive Q/R).

CUSUM on innovations: `S_t = max(0, S_{t−1} + (r_t − μ₀ − κ))`, alarm if `S_t > h`.

Trinetra's angle: the KF estimates bias **reactively** through innovations; the learned module **predicts** it ahead and pre-corrects.

## Features (per window)

Lags, rolling mean/std/min/max, autocorrelation, cross-correlation (lead–lag, sync), FFT dominant frequency and band power, wavelet coefficients, peaks, skew/kurtosis/RMS/energy/ZCR.

| Module | Features |
|---|---|
| Drift | rolling mean/std, low-freq band, temperature cross-corr, lags |
| Fault | std spikes, min/max, ZCR, reconstruction error |
| Anomaly | Mahalanobis on features, AE residual, forecast residual |
| Health | SNR, rolling-std trend, autocorrelation decay |
| Adaptive weighting | per-sensor variance, innovation magnitude, confidence |

Engineered features + gradient-boosted trees give an interpretable baseline for health/fault modules.

## Libraries

numpy/scipy/pandas (data layer), statsmodels/tsfresh (classical baselines, features), PyTorch (custom models); darts or pytorch-forecasting for quick baselines.

## Common mistakes

Shuffled overlapping windows; ignoring sync; over-filtering; interpolating across long GPS gaps; no well-tuned classical baseline; fabricated citations.

## Derivations to know

Position error from accel and gyro biases; window length from physics; KF as a recursive Bayesian time-series method; why drift is forecastable but denied-mode position isn't; leakage-free evaluation.
