# Phase 5 — Digital signal processing

Source: trinetra_phase5_dsp.

## Why DSP comes before fusion

The KF is optimal only for zero-mean, white, Gaussian noise with known R. DSP makes that roughly true (removes vibration, spikes, out-of-band noise) and **measures** the residual noise so Q and R are honest. Integration amplifies low-frequency error, so it must be characterised before integrating.

## Signal classes (decide which tools are legal)

Continuous/discrete, deterministic/random (`z = s + η`: filter s, characterise η → R), stationary/non-stationary (static bench → FFT/Allan; moving vehicle → STFT/wavelets), periodic/aperiodic (rotor hum → notch; pothole → wavelet/outlier), energy/power (transients vs ongoing noise → PSD).

## Sampling

- Nyquist–Shannon: `f_s > 2 f_max`. Sampled spectrum is replicated every f_s; under-sampling folds `f` to `|f − f_s·round(f/f_s)|`.
- **Aliasing cannot be removed digitally.** Anti-alias in analog before the ADC.
- Quantisation noise variance `Δ²/12`; SQNR ≈ `6.02b + 1.76` dB. 16-bit ≈ 98 dB, so MEMS noise, not quantisation, dominates.

| Sensor | f_s | Band of interest |
|---|---|---|
| Accel / gyro | 100–1000 Hz | 0–50 Hz motion |
| Magnetometer | 10–100 Hz | 0–10 Hz |
| GPS | 1–10 Hz | < 1 Hz (up-interpolate) |
| Barometer | 10–100 Hz | < 5 Hz |

Synchronisation: timestamp alignment, clock drift, interpolation onto the IMU grid.

## Noise

Types: white, Gaussian, thermal, shot (quantum magnetometer), quantisation, magnetic, environmental, random walk, bias, drift. Metrics: SNR (dB), noise power σ², noise density (µg/√Hz, °/√hr), covariance R = E[ηηᵀ]. **Estimate R from a static recording** (`np.cov(z_static.T)`).

White noise integrated: `Var = nσ² ∝ t` ⇒ σ ∝ √t (ARW). Bias → t; double-integrated bias → ½bt².

AI handles noise via learned denoising (Brossard 2020 dilated CNN), noise-robust training (augmentation) and predicted variance → adaptive R. Classical filters take the cheap out-of-band noise; the network takes the structured in-band residual.

## Frequency domain

PSD = FT of autocorrelation (Wiener–Khinchin); spectrogram = |STFT|². Uses: noise colour, vibration tones, fault peaks, health (rising noise floor), separating motion vs vibration bands to pick cut-offs. Offline → filter design; online → features (dominant freq, band power, noise floor).

## Fourier

DFT bin `f_k = k f_s / N`, O(N²); radix-2 FFT O(N log N). Resolution `Δf = f_s/N`. Finite windows leak; taper (Hann). FFT assumes stationarity within the window, periodic extension, uniform sampling. Use Welch for PSD.

## Filters

| Type | Properties | Use |
|---|---|---|
| FIR `y = Σ b_k x[n−k]` | always stable; symmetric taps ⇒ **linear phase** (constant delay (M−1)/2, compensable in timestamps); many taps | IMU pre-filtering |
| IIR (feedback) | efficient; can be unstable (poles inside unit circle); nonlinear phase ⇒ use `filtfilt` offline | band-limiting |
| Adaptive LMS `w ← w + μ e x` | tracks changing noise | cancel throttle-dependent rotor tone |

- **Low-pass:** the most-used filter; strip vibration before integration. Too low a cut-off adds lag and eats dynamics; pick fc from the measured motion band.
- **High-pass:** remove DC/bias/slow drift (e.g. baro weather drift). Complementary filter = HP gyro + LP accel.
- **Band-pass / notch:** isolate gait (1–3 Hz) or a fault frequency; notch the rotor tone.
- **Butterworth** `|H|² = 1/(1 + (Ω/Ω_c)^{2N})`: maximally flat passband, −3 dB at Ω_c for every order, −20N dB/decade, poles on a circle. **Project default** for IMU/magnetometer band-limiting. Use Bessel when timing/waveform must be preserved; Chebyshev/Elliptic for sharper cuts with ripple.
- **Moving average:** SMA is FIR with a Dirichlet response (leaky side-lobes, lag (M−1)/2); EMA is a one-pole IIR `y = αx + (1−α)y_prev`. Cheap smoothing and rolling features, not precise band-limiting.
- **Savitzky–Golay:** local polynomial least-squares fit = fixed FIR; preserves peaks; gives clean derivatives. Use for magnetic-anomaly profiles and jerk/angular-acceleration features.

## Advanced

STFT (time–frequency trade-off), wavelets CWT/DWT (multi-resolution; denoise non-stationary IMU, catch transients), Hilbert envelope (bearing faults, instantaneous frequency), cepstrum (harmonic families), EMD/HHT (non-stationary decomposition). Kalman = model-based adaptive low-pass whose cut-off is set by Q vs R — the seam to Phase 7.

## Pipeline

sample (anti-alias) → sync → noise analysis (PSD, SNR, Allan ⇒ Q, R) → filter → FFT/TF → features → signal-quality gate (SNR, outlier reject) → AI modules + fusion. Quality feeds fusion weights; innovations feed health/anomaly.

## Libraries

NumPy + SciPy (`butter`, `sosfiltfilt`, `firwin`, `welch`, `savgol_filter`, `spectrogram`), PyWavelets, statsmodels, FilterPy (KF prototyping).

## Mistakes and practices

Mistakes: filtering after aliasing; forgetting a causal filter's delay in fusion timing; cut-off too low; global FFT on manoeuvres; R from moving data. Practices: analog anti-alias; zero-phase offline; cut-offs from measured bands; noise from static data; version filter configs; keep raw copies.

Evaluate a filter by SNR gain, RMSE vs reference, attenuation at target frequency, phase delay, and ultimately downstream ATE/drift.

## Derivations to know

Aliasing spectrum replication; white noise → random walk; FFT butterfly O(N log N); Butterworth flatness and poles; SG as a fixed convolution; LMS update.
