# Smart Hand Rehabilitation System — Hardware

> ⚠️ **This repository is under active development.** Hardware schematics, firmware code, and documentation will be updated continuously as the project evolves.

---

## About the Project

Neurological disorders such as stroke, traumatic brain injury, and Parkinson's disease frequently impair hand motor function — manifesting as muscle weakness, tremor, and loss of coordination. Traditional clinical tools like handheld dynamometers capture only peak grip strength, offering an incomplete picture of neuromuscular recovery.

This project is a **low-cost, wearable hand rehabilitation system** that integrates three sensor modalities — pressure, EMG, and IMU — into a portable, Bluetooth-enabled dual-device platform. The system targets two clinical populations:

- **Stroke / Brain Injury** — emphasis on force control assessment (EMG + Pressure)
- **Parkinson's / Tremor-Dominant** — emphasis on movement quality analysis (IMU + Pressure)

The device streams real-time sensor data via Bluetooth Low Energy (BLE) to a companion web application, which guides the patient through rehabilitation exercises, computes clinically meaningful features, and logs sessions for remote clinician review.

> This repository covers the **hardware design**: electrical schematics, sensor integration, firmware, and device prototyping. The web application and signal processing backend are maintained separately and briefly referenced here for context.

---

## System Architecture

The system is split into two main blocks.

### Device Block (this repo)

Two independent wearable units communicate with each other and relay all data to the computer:

```
[Forearm Unit]                         [Wrist Unit]
 ├─ NRF52840 board                      ├─ NRF52840 board
 ├─ EMG-LAB sensor (analog, 3.3V)       ├─ Pressure sensor ABP-DRRV060MGAA5
 ├─ Li-Po 5V battery (JST)              ├─ On-board IMU (3-axis accel + gyro)
 └─ USB-B connector (UART TX/RX)        ├─ Li-Po 5V battery (JST)
                                        └─ USB-A connector (UART TX/RX)
                 │                                   │  │
                 └──────────── UART ─────────────────┘  │
                                                        │
                                                   BLE (2.4 GHz)
                                                        │
                                                    [Computer]
```

The forearm unit collects EMG data, applies preliminary digital filtering on-chip, and forwards the processed samples to the wrist unit over a UART serial link. The wrist unit aggregates EMG, IMU, and pressure data and transmits everything to the computer over Bluetooth.

### Computer Block (external repos)

- **FastAPI backend** — receives BLE data via WebSocket, runs the signal processing and feature extraction pipeline
- **Web application** — real-time sensor visualization, exercise guidance, session logging, and longitudinal progress tracking
- **ML pipeline** — SVM state classifier pre-trained on public datasets + k-NN longitudinal progress tracker (in development)

---

## Hardware Design

### Microcontroller: XIAO Seeed NRF52840

Both units are built around the **NRF52840** board (NRF52 family). Key properties that motivated this choice:

| Feature | Value |
|---|---|
| Processor | 32-bit ARM Cortex-M4 with FPU |
| Clock speed | 64 MHz |
| Flash | 2 MB |
| BLE | Built-in (2.4 GHz) |
| IMU | Built-in 3-axis accelerometer + gyroscope |
| I/O | Digital + analog pins, I²C, UART |
| Power | Onboard Li-Po charging module (5V input via JST) |
| Size | Compact form factor |

All analog signal processing (filtering, ADC) happens directly on the board, eliminating the need for external op-amp circuits. This is a deliberate departure from the original design (see [Design Evolution](#design-evolution) below).

---

## Sensors

### 1. Pressure Sensor — ABP-DRRV060MGAA5

**Purpose:** Measures grip force during exercises. Connected via a plastic tube to a rubber bulb held in the patient's hand.

| Property | Value |
|---|---|
| Supply voltage | 3.3 V |
| Output | Analog (connected to pin `D0` on wrist unit) |
| Placement | On-board the wrist device; tube routed to patient's palm |

**Why this sensor?** A commercial dynamometer capable of real-time wireless data transmission was the original target, but budget constraints led to this pressure sensor solution. It reliably captures the force waveform over time, enabling extraction of features beyond simple peak force.

**Signal preprocessing:** Butterworth low-pass filter at 20 Hz cutoff to remove high-frequency noise from hand vibrations. Optional DC baseline correction when the hand is at rest.

**Extracted features:**

| Feature | Formula / Description |
|---|---|
| Maximum Voluntary Contraction (MVC) | Peak force during a maximal squeeze |
| Rate of Force Development (RFD) | `dF/dt` — slope of the force-time curve |
| Fatigue Index | `(F_initial − F_final) / F_initial` |
| Force Variability (CV) | `SD / Mean Force` — coefficient of variation during submaximal holds |
| Tremor amplitude | Force oscillation amplitude during steady holds (Parkinson group) |
| Spectral power in tremor band | PSD integrated over 4–12 Hz |

---

### 2. EMG Sensor — EMG-LAB (custom lab board)

**Purpose:** Measures electrical activity of the flexor digitorum superficialis muscle to assess neuromuscular activation, fatigue, and tremor modulation.

| Property | Value |
|---|---|
| Supply voltage | 3.3 V |
| Output | Analog (connected to pin `D0` on forearm unit) |
| Electrode configuration | Both measurement electrodes embedded directly on board; virtual ground reference on-board |
| Placement | Inner forearm, fixed with silver chloride gel electrodes |

**Why this sensor?** The previous design used a Muscle Sensor V3.0, which required a ±9 V bipolar supply (two 9 V batteries = 90 g of bulk). The EMG-LAB sensor runs at 3.3 V from the same Li-Po battery powering the board, dramatically reducing weight and complexity.

> ⚠️ The EMG-LAB sensor was borrowed from a previous laboratory project and must be returned after completion. The sensor is mounted in a removable socket soldered onto the forearm PCB.

**Electrode placement:** Electrodes must be aligned along the muscle fibers of the flexor digitorum superficialis, spaced ≤ 2 cm apart. Good-quality silver chloride gel electrodes are required to minimize skin-electrode impedance.

**Signal preprocessing (two-stage pipeline):**

*On-chip (NRF52840):*
- Digital notch filter at 50 Hz — removes power-line interference
- DC offset removal — subtracts signal mean to center around zero

*Offline in Python (backend):*
- Band-pass filter 20–450 Hz — removes motion artifacts and high-frequency noise
- Full-wave rectification
- Moving RMS window — generates smooth muscle activation envelope

**Extracted features:**

| Feature | Formula / Description |
|---|---|
| EMG RMS | `sqrt(1/N * Σ e²_i)` — overall muscle activation level |
| Median Frequency Shift | Change in spectral median between first and second half of contraction — fatigue indicator |
| Tremor Band Power | PSD of EMG envelope integrated over 2–12 Hz |
| Peak Tremor Frequency | `argmax PSD` over [2, 12] Hz — identifies dominant tremor frequency |
| Electromechanical Delay (EMD) | `t_force_onset − t_EMG_onset` — neuromuscular transmission efficiency |

---

### 3. IMU Sensor — On-board NRF52840

**Purpose:** Measures linear acceleration and angular velocity in 3 axes to quantify hand tremor, orientation stability, and movement quality.

| Property | Value |
|---|---|
| Type | 3-axis accelerometer + 3-axis gyroscope |
| Supply | Powered by the NRF52840 board (no extra wiring) |
| Placement | Wrist unit, outer side of the hand |
| Interface | Accessed programmatically via the NRF52 SDK |

**Signal preprocessing:**
- Butterworth low-pass filter at 20 Hz cutoff (accelerometer + gyroscope)
- Sensor fusion (accelerometer + gyroscope) for accurate orientation estimation and drift reduction
- FFT / PSD for frequency-domain tremor analysis

**Clinically relevant frequency bands:**

| Band | Range | Meaning |
|---|---|---|
| Physiological tremor | 8–12 Hz | Normal, non-pathological oscillations |
| Parkinson's rest tremor | 4–6 Hz | Pathological resting tremor |
| Essential tremor | 5–12 Hz | Postural or action tremor |
| Voluntary / postural | < 4 Hz | Excluded from tremor analysis |

**Extracted features:**

| Feature | Formula / Description |
|---|---|
| Tremor Power (4–12 Hz) | `Σ PSD(a_mag)` over 4–12 Hz |
| Tremor Frequency | `argmax PSD(a_mag)` over [4, 12] Hz |
| Orientation Stability | `Var(θ_quaternion)` during hold phase |
| Range of Motion (ROM) | `θ_max − θ_min` during grip/dynamic tasks |
| Hold Stability | `SD(ω_mag)` during plateau phase |
| Drift Rate | `(θ_end − θ_start) / t_hold` |

---

## Inter-Device Communication

The two boards communicate via **UART** over pins `D6`–`D7`, routed through an external USB cable (USB-A on the wrist unit, USB-B on the forearm unit). UART was chosen because it is a digital interface — it introduces zero noise into the data, and all analog signal paths are kept as short as possible within each unit.

Bluetooth transmission is intentionally implemented only from the **wrist unit** to minimize electromagnetic interference with the sensitive EMG module on the forearm unit. Two interference mechanisms were identified and mitigated:

1. The BLE 2.4 GHz carrier — kept physically separated from the EMG analog path
2. Uneven BLE power consumption — current spikes during active transmission can cause voltage supply fluctuations that corrupt EMG readings

---

## Design Evolution

### Why We Moved Away from the Original Design

The first prototype used a **Muscle Sensor V3.0** EMG sensor with an analog filter cascade (band-stop at 50 Hz + low-pass at 1000 Hz). Two critical problems emerged:

**Battery problem:** The Muscle Sensor V3.0 requires a ±9 V bipolar supply. Generating this from two 9 V batteries (each 45 g) mounted on the wrist added significant mass — enough to mechanically affect tremor measurements and distort baseline muscle load:

```
Greater load → higher permanent muscle strain → signal offset relative to unloaded baseline
```

**Analog filtering problem:** Component availability in the lab was limited; the exact resistor-capacitor values for the target filter frequencies were not always obtainable. Compromise values resulted in poorly attenuated noise, requiring additional digital post-processing anyway. Since digital filtering on the NRF52840 is more precise and flexible than any analog cascade achievable with available parts, analog filtering was abandoned entirely.

### Current Design Improvements

| Aspect | Old Design | Current Design |
|---|---|---|
| EMG sensor | Muscle Sensor V3.0 | EMG-LAB (3.3V, on-board electrodes) |
| Power supply | 2× 9V batteries (90g) | 2× Li-Po 5V (10g each) |
| Analog filtering | Band-stop + low-pass RC circuits | Removed — fully digital |
| Physical form | Single unit on wrist | Two units: forearm (EMG) + wrist (IMU/Pressure) |
| Weight (forearm) | — | 19 g (25 g with battery) |
| Weight (wrist) | — | 14 g (20 g with battery) |
| IMU placement | Wrist | Wrist (enables future forearm tremor localization) |

---

## Device Specifications

### Forearm Unit
- **Weight:** 19 g (25 g including battery)
- **Dimensions:** 3.0 × 4.6 × 2.2 cm
- **Mounting:** Silver chloride gel electrodes adhered directly to the inner forearm skin
- **Battery:** 300 mAh Li-Po, 5V, rechargeable via JST connector

### Wrist Unit
- **Weight:** 14 g (20 g including battery)
- **Dimensions:** 2.7 × 5.8 × 1.4 cm
- **Mounting:** Sports fabric strap on the outer wrist
- **Battery:** 300 mAh Li-Po, 5V, rechargeable via JST connector

---

## Component List

| Component | Model / Type | Role |
|---|---|---|
| Main board ×2 | NRF52840 (NRF52 family) | Core controller, BLE, IMU |
| IMU | On-board (3-axis accel + gyro) | Motion and tremor sensing |
| Pressure sensor | ABP-DRRV060MGAA5 | Grip force measurement |
| EMG sensor | EMG-LAB (lab-sourced) | Muscle activation measurement |
| Forearm connector | USB-A (4-pin) | UART inter-device link |
| Wrist connector | USB-B (4-pin) | UART inter-device link |
| Batteries ×2 | 5V Li-Po, 300 mAh | Board power supply |
| Electrode pads | Silver chloride gel | EMG signal acquisition |

---

## Future Hardware Work

- **Enclosure:** 3D-printed or workshop-fabricated housing to protect components and prevent patient contact with circuit elements
- **Power switch:** Currently absent — boards run until battery depletion
- **Unified charging:** Single charging port / switch for both units instead of two separate JST connectors
- **Extended sensor coverage:** Additional IMU unit on the forearm to localize tremor by segment
- **Analog filter revisit:** At a later stage, when minimizing acquisition latency becomes critical, a minimal analog pre-filter stage may be reintroduced

---

## Repository Structure

```
/
├── schematics/          # KiCad / PDF schematics for both units
├── firmware/            # NRF52840 firmware (C / Zephyr RTOS)
│   ├── wrist/
│   └── forearm/
├── hardware/            # PCB layout files, perfboard assembly notes
├── docs/                # Additional documentation and figures
└── README.md
```

> 📌 Schematics and firmware will be uploaded as the repository is populated. Check back for updates.

---

## Authors

This project was developed as part of a university course project.

- **Iaroslav Petrishchev** — Hardware design, electrical schematics, component selection and procurement, device assembly and testing *(this repository)*
- **Naya Nasr** — NRF52 sensor programming, data acquisition firmware, web application
- **Lucía Pérez Sáez** — Exercise protocols, signal processing design, backend integration, FSM and WebSocket logic

---

## License

To be determined.
