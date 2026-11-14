# Trinetra-AI: An Intelligent Multi-Sensor Fusion Framework for GPS-Denied Quantum Navigation

> **Status:** early development. The repository holds the design and reference material; code and results will follow. Nothing below is a measured result yet.

## Abstract

Satellite navigation fails underground, underwater, in urban canyons and under jamming or spoofing. Without it, inertial navigation drifts without bound: a small accelerometer bias alone grows into a position error that rises with the square of time. The Earth's crustal magnetic field offers a passive reference that cannot be jammed. It varies from place to place, so a magnetometer reading matched against an anomaly map gives an absolute position. The obstacle is that a magnetometer mounted on a vehicle mostly measures the vehicle itself.

Trinetra-AI is an open software framework for GPS-denied navigation built around magnetic-anomaly navigation. It works in three stages:
- **Compensate:** remove platform interference from the magnetometer with the Tolles-Lawson model, plus a learned neural correction for what Tolles-Lawson misses.
- **Match:** match the compensated anomaly signal against a magnetic-anomaly map with a particle filter.
- **Fuse:** feed the resulting position fixes into an error-state Kalman filter that fuses inertial and barometric data, and uses GPS only when it is present.

Further learned modules turn the filter's own innovations into calibrated confidence scores that adapt how much each fix is trusted. Every learned module is advisory, and the filter keeps working if any of them is turned off. Quantum magnetometers are expected to make this map-matching technique more precise.

The central research question is whether learned magnetic compensation and adaptive trust in map-match fixes reduce position error during GPS outages, compared with inertial navigation aided by classical Tolles-Lawson compensation. To answer it, GPS is masked on held-out flight lines from the public MagNav dataset. Inertial-only navigation, classical compensation and learned compensation are then compared by end-to-end trajectory error (ATE/RTE) and drift rate. Each learned module is ablated against the system without it, and filter consistency is checked with NEES/NIS. A clean, hexagonal architecture keeps the estimation core independent of ROS 2, storage and the deep-learning runtime. As a result, offline experiments, recorded-data replay and real-time deployment all run the same code, and every reported number can be reproduced with one command.

**Keywords:** GPS-denied navigation, magnetic anomaly navigation, magnetic compensation, Tolles-Lawson, particle filter map matching, error-state Kalman filter, adaptive noise estimation, inertial navigation.

## Documentation

- [docs/knowledge/](docs/knowledge/README.md): architecture, requirements, and notes on sensors, DSP, fusion, navigation and AI modules.
- [deep-research-report.md](deep-research-report.md): background research and reading list.

## Scope

This project covers software and AI only, not sensor hardware. It is an independent research project and is not affiliated with any company or product.
