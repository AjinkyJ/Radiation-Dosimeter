# Radiation-Dosimeter
# BLE Radiation Dosimeter — Hub & Peripheral Device

## Project Overview

A portable, wireless radiation monitoring system comprising a **peripheral radiation dosimeter** and a **central hub receiver** communicating over **BLE 4.2**. The system provides real-time radiation dose measurements with wireless data transfer, designed for field deployment in industrial and research environments.

**Status:** Functional Prototype  
**Duration:** Feb 2025 – Present  
**Organization:** C-DAC CINE, IIT Guwahati (MeitY Program)

---

## Hardware Stack

| Component | Specification |
|---|---|
| **Primary MCU** | Vega Aries V3 (RISC-V based) |
| **Wireless Module** | BLE 4.2 – RN4871 Module |
| **Radiation Sensor** | Scintillator + SiPM / PIN diode detector |
| **Display** | 10.1" Capacitive Touch TFT Display |
| **Development Environment** | Eclipse IDE on Linux |

---

## Software Architecture

The firmware runs **bare-metal** on the Vega Aries V3 with a lightweight application-level scheduler managing the core functional blocks.

```
┌──────────────────────────────────────────┐
│           Application Layer              │
│  (Measurement, Display, Logging)         │
├──────────────────────────────────────────┤
│       Bare-Metal Scheduler               │
│  ┌───────────────────────────────────┐   │
│  │ Module 1: ADC Acquisition         │   │
│  │ • ISR-driven sampling             │   │
│  │ • Raw pulse capture from sensor   │   │
│  ├───────────────────────────────────┤   │
│  │ Module 2: Signal Processing       │   │
│  │ • Peak detection                  │   │
│  │ • Dose accumulation / filtering   │   │
│  ├───────────────────────────────────┤   │
│  │ Module 3: BLE TX (RN4871)         │   │
│  │ • UART-based AT command interface │   │
│  │ • Peripheral / Central role mgmt  │   │
│  ├───────────────────────────────────┤   │
│  │ Module 4: Display (TFT)           │   │
│  │ • Real-time dose readout via LVGL │   │
│  │ • Touch input handling            │   │
│  └───────────────────────────────────┘   │
├──────────────────────────────────────────┤
│      Hardware Abstraction Layer          │
│  • ADC / GPIO / UART / SPI / Interrupts  │
└──────────────────────────────────────────┘
```

---

## Firmware Implementation

### 1. ADC-Based Radiation Pulse Acquisition

The ADC is configured in interrupt-driven mode. Each pulse event from the radiation detector frontend triggers an ISR, which captures the peak amplitude and flags it for processing in the main loop.

```c
// ISR: captures pulse amplitude on each detector event
void ADC_IRQHandler(void) {
    if (ADC->STATUS & ADC_INT_FLAG) {
        raw_sample = ADC->DATA;
        pulse_ready = 1;
        ADC->STATUS = ADC_INT_FLAG;  // clear flag
    }
}

// Main loop polling
void main_loop(void) {
    while (1) {
        if (pulse_ready) {
            pulse_ready = 0;
            process_pulse(raw_sample);
        }
        ble_service_run();
        display_update();
    }
}
```

**Note:** Bare-metal ISR + flag polling avoids RTOS overhead. Suitable for pulse rates well within the ADC's interrupt latency budget.

---

### 2. Signal Processing — Peak Detection & Dose Accumulation

```c
// Simplified peak detection: threshold-based pulse counting
void process_pulse(uint16_t adc_val) {
    if (adc_val > DETECTION_THRESHOLD) {
        pulse_count++;
        dose_uSv = (float)pulse_count * CALIBRATION_FACTOR;
    }
}

// Calibration: applied once during init from stored flash constant
// Dose [µSv] = pulse_count × calibration_factor
// calibration_factor derived from known Cs-137 source during setup
```

---

### 3. BLE Communication via RN4871 (UART AT Interface)

The RN4871 module communicates with the MCU over UART using ASCII AT commands. The firmware operates the module in both **peripheral** (dosimeter) and **central** (hub) roles.

```c
// Send dose data as BLE notification (peripheral mode)
void ble_send_dose(float dose_uSv) {
    char tx_buf[32];
    snprintf(tx_buf, sizeof(tx_buf), "SHW,0072,%04X\r\n",
             (uint16_t)(dose_uSv * 100));
    UART_Send(BLE_UART, (uint8_t *)tx_buf, strlen(tx_buf));
}

// Hub: scan and connect to peripheral dosimeters (central mode)
void ble_start_scan(void) {
    UART_Send(BLE_UART, (uint8_t *)"F\r\n", 3);  // Start scan
}
```

**Roles supported:**
- **Peripheral:** Dosimeter advertises and notifies dose data to connected hub
- **Central:** Hub scans, connects, and reads data from 1–4 peripheral dosimeters

---

### 4. Display — LVGL on 10.1" Capacitive Touch TFT

The hub device drives a 10.1" TFT display using LVGL for real-time dose visualization and touch-based user interaction.

```c
// LVGL label update (called periodically from display_update())
void display_update_dose(float dose_uSv) {
    char buf[32];
    snprintf(buf, sizeof(buf), "Dose: %.2f µSv", dose_uSv);
    lv_label_set_text(dose_label, buf);
}
```

---

## BLE Debugging

BLE communication issues were identified and resolved using:
- **BLE Sniffer** (Wireshark + hardware sniffer) — packet-level inspection of advertisement and connection events
- **AT command trace** — UART log of all commands/responses to/from RN4871
- **Mobile app testing** — end-to-end validation of peripheral notifications with a custom BLE client app

Resolved issues included incorrect characteristic handle references, connection parameter mismatches, and UART framing errors in the AT command interface.

---

## System Roles

### Peripheral Device (Dosimeter)
- Vega Aries V3 + RN4871 BLE module
- ADC-based radiation pulse detection from sensor frontend
- Advertises and sends dose data to hub over BLE 4.2

### Hub Device (Receiver)
- Vega Aries V3 in BLE central role
- Connects to 1–4 peripheral dosimeters simultaneously
- Displays real-time dose from all connected devices on 10.1" TFT
- Logs data locally

---

## Calibration Procedure

```
1. Expose sensor to known reference source (e.g., Cs-137 at known dose rate)
2. Record ADC pulse count over a fixed measurement window
3. Compute: calibration_factor = known_dose_uSv / measured_pulse_count
4. Store calibration_factor in non-volatile flash memory
5. Apply at runtime: Dose [µSv] = pulse_count × calibration_factor
```

---

## Testing & Validation

### Firmware-Level Tests

| Test | Description | Result |
|---|---|---|
| ADC Sampling | Verify pulse capture consistency under known source | PASS |
| BLE Peripheral Role | Notify dose data to mobile app / hub | PASS |
| BLE Central Role | Scan, connect, and receive from peripheral | PASS |
| Display Rendering | Real-time LVGL update with touch input | PASS |
| BLE Sniffer Validation | Packet integrity, connection stability | PASS |

### Field Validation
- **Location:** IIT Guwahati Radiation Lab
- **Reference:** Calibrated dosimeter
- **Duration:** Extended continuous monitoring sessions
- **Outcome:** Prototype dose readings in close agreement with reference instrument

---

## Software Dependencies

| Component | Purpose |
|---|---|
| **Eclipse IDE** | Firmware development (Linux environment) |
| **Embedded C (bare-metal)** | Application and driver firmware |
| **LVGL** | GUI framework for TFT display |
| **RN4871 AT Firmware** | BLE stack (handled by module) |
| **Vega SDK / BSP** | Board support package for Aries V3 |

---

## Project File Structure

```
radiation-dosimeter/
├── firmware/
│   ├── src/
│   │   ├── main.c                  # Entry point, main loop, scheduler
│   │   ├── adc_handler.c           # ADC init, ISR, pulse capture
│   │   ├── signal_processor.c      # Peak detection, dose accumulation
│   │   ├── ble_rn4871.c            # UART AT command interface for RN4871
│   │   └── display.c               # LVGL init, dose display, touch handler
│   ├── include/
│   │   └── *.h                     # Header files
│   └── Makefile
├── hardware/
│   ├── schematic.pdf               # Electrical schematic
│   └── BOM.csv                     # Bill of Materials
├── calibration/
│   ├── calibration_procedure.txt   # Step-by-step calibration guide
│   └── reference_data.csv          # Calibration measurements
├── test/
│   └── validation_log.txt          # Field test results
└── docs/
    ├── ARCHITECTURE.md             # System design overview
    └── BLE_PROTOCOL.md             # RN4871 command reference used
```

---

## Key Achievements

- **BLE Peripheral & Central roles** implemented on Vega Aries V3 with RN4871 module
- **Real-time dose display** on 10.1" capacitive touch TFT using LVGL
- **BLE debugging** using sniffer tools — resolved communication issues with mobile app and hub
- **Bare-metal firmware** — efficient, deterministic execution without RTOS overhead
- **Dual-role system** — same hardware platform used as dosimeter (peripheral) and hub (central)

---

## Future Enhancements

- Migrate to RTOS if concurrency requirements increase (e.g., multi-sensor high-rate logging)
- Add SD card or cloud logging on hub (MQTT over Wi-Fi)
- Upgrade to BLE 5.0 for extended range and higher throughput
- Add on-device anomaly detection for dose spike alerting
- Formal certification compliance (IEC 61526)

---

## Contact

**Developer:** Ajinkya Jadhav  
**Organization:** C-DAC CINE, IIT Guwahati  
**Program:** MeitY  
**Code:** Private government repository — details available upon request

*Last Updated: April 2026*
