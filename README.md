# Speech-Based Parkinson's Disease Detection on STM32F401RE

**Real-time, on-chip PD screening — bare-metal firmware, no OS, no cloud.**

[![Platform](https://img.shields.io/badge/Platform-STM32F401RE-03234B?style=for-the-badge&logo=stmicroelectronics&logoColor=white)](https://www.st.com/en/microcontrollers-microprocessors/stm32f401re.html)
[![Language](https://img.shields.io/badge/Language-Embedded_C-00599C?style=for-the-badge&logo=c&logoColor=white)]()
[![Status](https://img.shields.io/badge/Status-Patent_Pending-red?style=for-the-badge)]()

> Source code is proprietary and patent-pending. This repository documents the system architecture, methodology, hardware setup, and validated results. Code is available on request for academic review.

---

## Overview

This project implements a complete Parkinson's Disease (PD) speech screening system on a single STM32F401RE Nucleo-64 board running at 84 MHz with 96 kB of SRAM. The entire pipeline — audio acquisition, dual feature extraction, and on-chip AI inference — runs as bare-metal firmware with no operating system, no network connection, and no external processing hardware.

The system captures a 224 ms voice sample, extracts a 29-dimensional hybrid feature vector via two complementary pipelines (MFCC and STFT/NMF), and classifies the result on-chip using a NanoEdge AI model. The final diagnosis is displayed directly on an integrated SSD1306 OLED display.

To the best of our knowledge, this is the first demonstration of the complete PD-validated NMF-TF pipeline running as real-time firmware on a single Cortex-M4 device.

---

## Why This Matters

Parkinson's disease affects approximately 6.1 million people worldwide and is projected to reach 12 million by 2040. Hypokinetic dysarthria — the distinctive speech degradation caused by PD — affects 70–90% of patients and is one of the earliest clinical indicators. However, current diagnostic tools (MDS-UPDRS rating scales) are subjective, time-intensive, and largely unavailable in rural or low-resource clinical settings.

All prior research on NMF-based speech feature extraction for PD processed audio offline on desktop computers. This work is the first to demonstrate that the same validated pipeline can run in real time on hardware affordable enough to deploy at the point of care.

---

## System Architecture

The system is partitioned into three physical layers: the analogue input chain, the STM32F401RE digital processing core, and the output peripherals.

```
Patient Voice
     |
 MAX4466 Electret Amplifier (Gain ~40 dB, Bias VCC/2)
     |
 ADC1 — PA0 — 12-bit, 16 kHz, 3584 samples (224 ms)
     |
 Pre-processing: DC Removal + Pre-emphasis (α=0.97)
     |
     +——————————————+——————————————+
     |                              |
 MFCC Branch                  STFT/NMF Branch
 (Cepstral)                   (Time-Frequency)
     |                              |
 Hamming Window              VAD Gate (θ=10⁻⁴)
 FFT 1024-pt (CMSIS-DSP)     STFT × 6 frames
 Mel Filterbank (26)         TFM: T ∈ R^(6×512)
 Log + DCT                   Rank-1 NMF: T ≈ wh^T
 F[00–12]: 13 MFCCs          F[13–28]: 16 NMF Stats
     |                              |
     +——————————————+——————————————+
                    |
          29-Dimensional Feature Vector
                    |
          NanoEdge AI — 2-Class KNN
          (IPD / Healthy)
                    |
     +——————————————+——————————————+
     |                              |
 SSD1306 OLED Display         USART2 Debug Log
 (I2C @ 400 kHz)              (115200 baud)
```

---

## Hardware

| Component | Specification |
|---|---|
| Microcontroller | STM32F401RETx, ARM Cortex-M4 @ 84 MHz, FPU, 96 kB SRAM, 512 kB Flash |
| Microphone | MAX4466 electret amplifier, gain ~40 dB, bias at VCC/2 ≈ 1.65 V |
| Display | SSD1306 0.96" OLED, 128×64 px, I2C (PB6/PB7) |
| ADC | 12-bit, ADC1 Channel 0 (PA0), 480-cycle sample time |
| Debug/Logging | USART2 @ 115200 baud (PA2/PA3) |
| AI Library | NanoEdge AI — 2-class classifier compiled into firmware |
| DSP Library | ARM CMSIS-DSP v1.0.0 — arm_rfft_fast_f32 |
| Trigger | User button PC13, rising-edge EXTI |
| IDE / Toolchain | STM32CubeIDE, Cube FW_F4 V1.28.2 |

### Wiring

| Signal | STM32 Pin | Component Pin |
|---|---|---|
| Audio input (analog) | PA0 | MAX4466 OUT |
| VCC | 3.3 V | MAX4466 VCC, SSD1306 VCC |
| Ground | GND | MAX4466 GND, SSD1306 GND |
| I2C clock (SCL) | PB6 | SSD1306 SCL |
| I2C data (SDA) | PB7 | SSD1306 SDA |
| UART TX | PA2 | USB-Serial RX |
| UART RX | PA3 | USB-Serial TX |
| Trigger button | PC13 | Nucleo user button (built-in) |
| Status LED | PA5 | Nucleo green LED (built-in) |

---

## Feature Extraction

### MFCC Branch — F[00]–F[12]

A six-stage pipeline operating on a single 1,024-sample (64 ms) frame:

1. DC removal and first-order pre-emphasis (α=0.97)
2. Hamming windowing (N=1024)
3. Real FFT — arm_rfft_fast_f32 (FPU-accelerated)
4. Mel filterbank — 26 triangular filters
5. Logarithmic compression
6. Type-II DCT — first 13 coefficients retained

Captures the vocal-tract spectral envelope and articulatory configuration.

### STFT/NMF Branch — F[13]–F[28]

Operating over the full 224 ms acquisition window:

1. Voice activity detection (energy threshold θ=10⁻⁴)
2. Per-frame DC removal
3. Hamming windowing + FFT × 6 frames (hop H=512)
4. Linear magnitude spectrum per frame
5. Time-frequency matrix construction — T ∈ R^(6×512)
6. Rank-1 NMF decomposition — T ≈ wh^T (max 10 iterations)
7. Eight statistical descriptors extracted from each of w and h

**Eight descriptors per vector (16 total):** Mean · Std Dev · Skewness · Kurtosis · 1st Moment · 2nd Central Moment · Sparsity · Discontinuity

These descriptors capture phonatory instabilities — irregular vocal-fold vibration, glottal noise, and tremor-induced energy bursts — that frame-level MFCC representations fundamentally cannot resolve.

### Why Both Branches Together

| Aspect | MFCC Branch | NMF Branch |
|---|---|---|
| Domain | Cepstral (perceptual) | Time-frequency (structural) |
| Output | 13 coefficients | 16 statistical descriptors |
| Captures | Spectral envelope | Temporal phonatory dynamics |
| PD sensitivity | Articulatory imprecision | Phonatory instability |

---

## Results

### Latency

| Stage | Cycles | Time |
|---|---|---|
| VAD gate | ~7,168 | ~85 µs |
| DC removal | ~7,168 | ~85 µs |
| Frame + Window × 6 | ~36,864 | ~439 µs |
| FFT × 6 (FPU) | ~73,728 | ~878 µs |
| Magnitude × 6 | ~18,432 | ~219 µs |
| TFM accumulate | ~3,072 | ~37 µs |
| NMF + 16 stats | ~50,000 | ~595 µs |
| **STFT/NMF total** | **~196k** | **< 3 ms** |
| MFCC pipeline | — | < 5 ms |
| **End-to-end total** | — | **< 8 ms** |

The complete pipeline processes each 224 ms audio window in under 8 ms — less than 3.6% of the acquisition window.

### Statistical Validation (PC-GITA Corpus)

All 16 NMF descriptors were validated at p<0.001 on the PC-GITA corpus (50 PD subjects, 50 healthy controls). The weakest result (skewness of h) returned χ²=56.69, p=5.1×10⁻¹⁴. The strongest (std dev of h) returned χ²=570.81.

### Classification Accuracy vs. Baselines (Karan et al., 2021 — PC-GITA)

| Task | NMF-TF | Acoustic | MFCC |
|---|---|---|---|
| Vowel /a/ | 0.85 ± 0.04 | 0.65 ± 0.07 | 0.55 ± 0.05 |
| Vowel /e/ | **0.92 ± 0.03** | 0.62 ± 0.07 | 0.57 ± 0.09 |
| Vowel /i/ | 0.90 ± 0.05 | 0.62 ± 0.05 | 0.61 ± 0.07 |
| Vowel /o/ | 0.91 ± 0.03 | 0.59 ± 0.08 | 0.51 ± 0.10 |
| Vowel /u/ | 0.91 ± 0.03 | 0.60 ± 0.09 | 0.58 ± 0.07 |
| Word /petaka/ | **0.97 ± 0.02** | 0.61 ± 0.04 | 0.61 ± 0.04 |

NMF-TF features outperform MFCCs by 25–35 percentage points across all vowels.

### On-Device Validation

Mathematical correctness of the embedded computation was verified by comparing UART-logged feature values against an independent Python reference implementation of the identical pipeline applied to the same 224 ms audio segment. Maximum absolute deviation across all 29 features was below 10⁻³, consistent with 32-bit floating-point rounding.

---

## Memory Footprint

| Buffer | Size | Purpose |
|---|---|---|
| mic_buffer[3584] | 14,336 B | Raw ADC samples |
| spectrogram[30][512] | 61,440 B | Time-frequency matrix |
| window[1024] | 4,096 B | Hamming coefficients |
| fft_out[1024] | 4,096 B | FFT scratch |
| mfcc_frame[1024] | 4,096 B | MFCC frame buffer |
| W[512] | 2,048 B | NMF basis w |
| H[30] | 120 B | NMF activation h |
| feature_vector[29] | 116 B | Classifier input |
| **Total** | **~90 kB** | **Within 96 kB budget** |

---

## Comparison with Prior Work

| Study | Features | Platform | Real-Time? |
|---|---|---|---|
| Little et al. (2015) | Dysphonia + PPE | Desktop | No |
| Tsanas et al. (2012) | 132 dysphonia features | Desktop | No |
| Karan et al. (2021) | NMF-TF (17 features) | Desktop | No |
| Vasquez-Correa et al. (2019) | STFT image | GPU | No |
| Oukil et al. (2025) | Sound classification | STM32F401RE | Yes (not PD-specific) |
| **This work** | **13 MFCC + 16 NMF = 29 features** | **STM32F401RE (bare-metal)** | **Yes** |

---

## Source Code

This project is patent-pending. The source code is maintained in a private repository and is not publicly available at this time.

For academic review or collaboration inquiries, please contact:

**Rohan Kumar Singh**
rohankumar17362@gmail.com
[LinkedIn](https://linkedin.com/in/rohan-kumar-singh-b31465268)

---

## Reference

R. K. Singh, "Hardware Implementation of the Speech-Based Parkinson's Disease Detection System on STM32 Nucleo Board," Department of Electronics and Communication Engineering, Babasaheb Bhimrao Ambedkar University, Lucknow, India, 2025.
