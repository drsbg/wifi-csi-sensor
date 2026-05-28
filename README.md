# Wi-Fi CSI Dataset — Passive Motion Sensing in Educational Environments

> Dataset companion for the paper **"Wi-Fi CSI como Sensor Passivo: Monitoramento de Movimento em Salas de Aula com ESP32"**, submitted to SBSeg 2026.

---

## Overview

This repository contains the anonymized CSI (Channel State Information) datasets collected in two real Brazilian school environments, as well as the preprocessing and classification pipeline described in the paper.

The data supports research on **passive Wi-Fi sensing** for presence detection, movement classification, and spatial zone tracking using low-cost ESP32 devices — without cameras, wearables, or any participant-carried hardware.

## Experiments at a Glance

| | Experiment 1 | Experiment 2 |
|---|---|---|
| **Environment** | Public school classroom | Private school lab |
| **Participants** | 30 adults (18–60 yrs) | 43 adolescents (14–18 yrs) |
| **Gender** | 18M / 12F | 18M / 25F |
| **Height range** | 152–190 cm | 150–184 cm |
| **Zones** | 4 (A–D) | 6 (A–F) |
| **Samples / participant** | 44 | 66 |
| **Total records** | 1,320 | 2,838 |
| **Simultaneous presence** | No (individual sessions) | Yes (up to 43 concurrent) |
| **Hardware** | ESP32 TX+RX pair | ESP32 TX+RX pair |

**Total across both experiments: 73 participants · 4,158 CSI records.**

---

## Data Format

Each record in the processed datasets is a CSV row with the following fields:

| Field | Type | Description |
|---|---|---|
| `record_id` | string | Unique identifier (anonymized) |
| `experiment` | int | `1` or `2` |
| `zone` | string | Ground-truth zone label (`A`–`D` or `A`–`F`) |
| `movement` | int | `0` = static posture · `1` = active movement (arm raises) |
| `phase` | string | `individual` or `collective` (Experiment 2 only) |
| `group_size` | int | Number of participants present during capture |
| `subcarrier_{1..56}_amp_mean` | float | Mean amplitude per subcarrier |
| `subcarrier_{1..56}_amp_std` | float | Amplitude std deviation per subcarrier |
| `subcarrier_{1..56}_energy` | float | Signal energy per subcarrier |
| `subcarrier_{1..56}_phase_var` | float | Phase variance per subcarrier |
| `subcarrier_{1..56}_phase_mean` | float | Sanitized phase mean per subcarrier |
| `rssi` | float | Received Signal Strength Indicator (dBm) |
| `snr` | float | Signal-to-Noise Ratio (dB) |
| `noise_floor` | float | Estimated noise floor (dBm) |

The feature vector has **280 dimensions** (5 features × 56 active subcarriers), plus RSSI, SNR, and noise floor.

Raw files contain the complex-valued CSI output from the ESP32-CSI-Tool, structured as `(real, imaginary)` pairs per subcarrier per packet.

---

## Pipeline Summary

```
Raw CSI (ESP32-CSI-Tool)
        │
        ▼
1. Phase sanitization      — corrects systematic hardware offsets between subcarriers
        │
        ▼
2. Hampel filter           — window=10; replaces outliers with local median
        │
        ▼
3. Savitzky-Golay filter   — window=11; smooths high-frequency noise preserving peaks
        │
        ▼
4. FFT                     — 256-sample windows, 50% overlap; converts to frequency domain
        │
        ▼
5. Feature extraction      — 280-dim vector (amplitude, phase, energy per subcarrier)
        │
        ▼
6. Classification          — KNN (k=5, Euclidean) · Random Forest (100 trees)
                             Stratified 5-fold cross-validation
```

---

## Key Results

### Binary Movement Detection (static vs. active)

| Experiment | Algorithm | F1-score | Precision | Recall |
|---|---|---|---|---|
| Exp 1 — 30 participants | KNN | **100.00%** | 100.00% | 100.00% |
| Exp 1 — 30 participants | Random Forest | 99.92% | 99.93% | 99.92% |
| Exp 2 — 43 participants | KNN | 99.89% | 99.88% | 99.90% |
| Exp 2 — 43 participants | Random Forest | **99.96%** | 99.96% | 99.97% |

### Spatial Zone Tracking

| Experiment | Algorithm | F1-score | Precision | Recall |
|---|---|---|---|---|
| Exp 1 — 4 zones | KNN | 97.12% | 97.17% | 97.12% |
| Exp 1 — 4 zones | Random Forest | **100.00%** | 100.00% | 100.00% |
| Exp 2 — 6 zones | KNN | 45.24% | 45.21% | 45.55% |
| Exp 2 — 6 zones | Random Forest | 46.85% | 46.70% | 47.28% |

Performance degradation in Experiment 2 is expected and explained by the larger environment, greater TX–RX distance, and higher number of adjacent zones — not by model failure. A single ESP32 pair is sufficient for smaller deployments (≤4 zones); multi-pair setups are recommended beyond that scale.

---

## Hardware & Collection Setup

- **Device**: Espressif ESP32 (standard, unmodified)
- **Firmware / CSI extraction**: [ESP32-CSI-Tool](https://github.com/StevenMHernandez/ESP32-CSI-Tool)
- **Channel width**: 20 MHz · **Active subcarriers**: 56
- **TX mode**: continuous broadcast
- **RX placement**: central position in the monitored environment
- **Movement protocol**: participant stops at each zone marker, then performs 6 arm raises
