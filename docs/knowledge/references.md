# Datasets and papers

Collected from all knowledge sources. "Verify" means confirm DOI/venue/licence on the official page before citing or downloading. Per CLAUDE.md, 2025–2026 papers and industry claims are **unverified** until checked.

## Datasets

| Dataset | Sensors | Ground truth | Best for | Note |
|---|---|---|---|---|
| **RoNIN** | phone IMU + mag, 200 Hz | 3-D trajectory | **start here**; learned inertial | research licence, ~50 % public, 42.7 h, >100 subjects (Herath et al., ICRA 2020) |
| **MagNav (DAF-MIT AIA)** | aeromagnetic + INS | flight reference | **required** for the magnetic path | Gnadt et al., 2023; github.com/MIT-AI-Accelerator/MagNav.jl |
| TLIO | headset IMU | VIO | learned displacement + EKF | Liu et al., RA-L 2020; github.com/CathIAS/TLIO |
| OxIOD | IMU (accel/gyro/mag) | Vicon/VI | inertial baselines | Chen et al., 2018 |
| EuRoC MAV | IMU + stereo | Vicon/Leica | VIO, fusion | Burri et al., 2016 |
| KITTI | IMU + GPS/RTK + LiDAR + cam | RTK/INS | driving fusion; no magnetometer | Geiger et al. |
| comma2k19 | IMU + GNSS + CAN | GNSS + fusion | GPS/INS driving | ~33 h, ~100 GB |
| MagPIE | IMU + magnetometer | cm-level | indoor magnetic map matching | Hanley et al., IPIN 2017; **MagPIE2 unverified** |
| TUM VI | IMU + stereo | partial MoCap | precise IMU–camera sync | Schubert et al., 2018 |
| KAIST Urban | IMU + GPS + LiDAR | fused | urban-canyon GNSS | hard |
| NASA C-MAPSS | turbofan run-to-failure | failure cycle | RUL/health practice | |
| UCI HAR | phone IMU, no mag | activity labels | pipeline practice | |

Synthetic options: sample WMM along a ground-truth path to fake a magnetometer track; Gazebo/AirSim for controlled GPS-denial ablations.

Record each used dataset's licence in `datasets/README.md`.

## Papers — estimation and navigation foundations

| Work | Authors, year |
|---|---|
| A New Approach to Linear Filtering and Prediction | Kalman, 1960 |
| A New Extension of the KF to Nonlinear Systems (UKF) | Julier & Uhlmann, 1997 |
| The Unscented KF for Nonlinear Estimation | Wan & van der Merwe, 2000 |
| Novel approach to nonlinear/non-Gaussian Bayesian state estimation (PF) | Gordon, Salmond & Smith, 1993 |
| Probabilistic Robotics | Thrun, Burgard & Fox, 2005 |
| Strapdown Inertial Navigation Technology | Titterton & Weston, 2004 |
| Principles of GNSS, Inertial & Multisensor Integrated Navigation | Groves, 2013 |
| Global Positioning System: Signals, Measurements & Performance | Misra & Enge, 2006 |
| Analysis & Modeling of Inertial Sensors Using Allan Variance | El-Sheimy, Hou & Niu, IEEE T-IM 2008 |
| MEMS Inertial Sensors: A Tutorial Overview | Shaeffer, IEEE Comm. Mag. 2013 |
| Nonlinear Complementary Filters on SO(3) | Mahony, Hamel & Pflimlin, 2008 |
| Estimation of IMU/MARG Orientation (Madgwick) | Madgwick et al., ICORR 2011 |
| Monte Carlo Localization | Dellaert, Fox, Burgard & Thrun, ICRA 1999 |
| Hidden Markov Map Matching Through Noise and Sparseness | Newson & Krumm, ACM GIS 2009 |
| Quaternion kinematics for the error-state KF | Solà, arXiv 1711.02508 (from CLAUDE.md plan) |
| Kalman and Bayesian Filters in Python (book) | Labbe (from CLAUDE.md plan) |

## Papers — learned inertial / fusion (most relevant)

| Work | Authors, year |
|---|---|
| KalmanNet: NN-Aided Kalman Filtering | Revach et al., IEEE TSP 2022 |
| TLIO: Tight Learned Inertial Odometry | Liu et al., RA-L 2020 |
| RoNIN | Herath, Yan & Furukawa, ICRA 2020 |
| IONet | Chen, Lu, Markham & Trigoni, AAAI 2018 |
| Denoising IMU Gyroscopes with Deep Learning | Brossard, Bonnabel & Barrau, RA-L 2020 |
| AI-IMU Dead-Reckoning | Brossard, Barrau & Bonnabel, IEEE T-IV 2020 |
| VINet | Clark et al., AAAI 2017 |
| AirIMU / EqNIO | Qiu et al., 2023 / Jayanth et al., 2024 — verify |

## Papers — ML foundations used by the modules

LSTM (Hochreiter & Schmidhuber, 1997); GRU (Cho et al., 2014); Transformer (Vaswani et al., 2017); TCN (Bai, Kolter & Koltun, 2018); Informer (Zhou et al., AAAI 2021); denoising autoencoders (Vincent et al., 2008); VAE (Kingma & Welling, 2013); Isolation Forest (Liu, Ting & Zhou, 2008); MC Dropout (Gal & Ghahramani, 2016); Deep Ensembles (Lakshminarayanan et al., 2017); SHAP (Lundberg & Lee, 2017); DDPM (Ho et al., 2020); PINNs (Raissi, Perdikaris & Karniadakis, 2019); TimeGAN (Yoon et al., NeurIPS 2019).

## Papers — DSP foundations

Shannon 1949 / Nyquist 1928 (sampling); Cooley & Tukey 1965 (FFT); Butterworth 1930; Savitzky & Golay 1964; Welch 1967 (PSD); Allan 1966; Mallat 1989 (wavelets); Huang et al. 1998 (EMD/HHT); Widrow & Hoff 1960 (LMS).

## Papers — ROS 2

Macenski et al., Science Robotics 2022 (ROS 2); Quigley et al. 2009 (ROS); Macenski et al., IROS 2020 (Nav2, "The Marathon 2"); Macenski et al., RAS 2023; Macenski et al., RA-L 2023 (composition); Moore & Stouch, IAS-13 2014 (robot_localization); Koenig & Howard, IROS 2004 (Gazebo); Macenski & Jambrecic, JOSS 2021 (SLAM Toolbox).

## Papers — software engineering

Wilson et al. 2014, 2017; Sandve et al. 2013; Sculley et al., NeurIPS 2015; Amershi et al., ICSE-SEIP 2019; Martin, Clean Code 2008; Gamma et al. 1994; Fowler, Refactoring 2018.

## Unverified — do not cite until checked

- Muradoglu et al. 2025, "Quantum-assured magnetic navigation" (arXiv:2504.08167) / Q-CTRL Ironstone Opal
- Boeing 2024 GPS-free quantum-sensor flight tests
- SandboxAQ AQNav
- MagPIE2
- NavIC operational-satellite figures for 2024–2026
- Huang 2026 (named in CLAUDE.md)

## Never cite

- "Thompson et al. 2019" (the deep-research report itself labels it hypothetical).
