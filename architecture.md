# HealthyFocus — Architecture

## Overview
HealthyFocus is an embedded Edge AI desk companion that monitors environmental and behavioral factors to help detect unhealthy work habits. It runs entirely on local compute — an STM32 microcontroller plus a laptop — with no cloud dependency, keeping the design privacy-first.

## System Diagram

```
┌─────────────────────────────┐         ┌──────────────────────────────┐
│         STM32 (Nucleo-F401RE)         │  UART   │           Laptop               │
│                              │────────▶│                                │
│  I2C Bus:                   │         │  Python (pyserial)             │
│   - BME280 (temp/humidity/  │         │   - Reads STM32 sensor data    │
│     pressure)                │         │                                │
│   - BH1750 (ambient light)  │         │  OpenCV (laptop webcam)        │
│                              │         │   - Posture / presence /      │
│  ADC Pin:                   │         │     focus detection            │
│   - MQ-135 (air quality) or │         │                                │
│     sound level sensor      │         │  Decision Logic                │
│                              │◀────────│   - Combines env + CV signals │
│  Acts on decision:           │  UART   │   - Flags "unhealthy habit"   │
│   - LED / buzzer / display  │         │     events                     │
└──────────────────────────────┘         └──────────────────────────────┘
```

## Component Breakdown

### STM32 side (firmware)
**Board:** Nucleo-F401RE (Cortex-M4F, 84MHz, 96KB SRAM, 512KB flash, built-in ST-LINK debugger)

**Responsibilities:**
- Read sensors reliably at a fixed interval
- Package sensor readings into a simple serial data frame
- Send data to laptop over UART
- Receive simple decisions/commands back from laptop
- Act on those decisions (LED, buzzer, or small display output)

**Sensors (phased build order):**
1. BME280 (I2C) — temperature, humidity, pressure — build and verify first
2. BH1750 (I2C, same bus as BME280) — ambient light — adds multi-device I2C addressing
3. MQ-135 or sound sensor (ADC, analog) — air quality or ambient noise — adds analog input handling

**Skills this exercises:** I2C (single and multi-device), ADC/analog reading, UART communication, buffering/framing sensor data (DSA-in-C relevant), timing/interrupts.

### Laptop side (Python/CV)
**Responsibilities:**
- Read incoming UART data from STM32 (`pyserial`)
- Run computer vision on the laptop's own webcam (`opencv-python`) — posture, presence, focus-related detection
- Combine environment signals (from STM32) with CV signals (from webcam) into a single "unhealthy habit" decision
- Send simple feedback back to STM32 over UART

**Skills this exercises:** Python fundamentals, OpenCV/CV basics, serial communication, combining multi-source signals into a decision.

## Data Contract (STM32 → Laptop)
Start simple — plain text lines over UART, one reading per line, before considering anything binary/structured:
```
temp:24.3,hum:55.2,press:1012.4,light:320
```
Format can evolve once the basic pipeline works end-to-end.

## Repo Structure
```
Healthy-focus/
├── firmware/           ← STM32CubeIDE project
├── cv-python/          ← Python/OpenCV script + venv (gitignored)
├── .gitignore
├── README
└── LICENSE
```

## Design Principles
- No cloud dependency — all compute stays local (STM32 + laptop)
- Privacy-first framing — camera feed and sensor data never leave the local machine
- Build order: get one sensor working end-to-end before adding the next
- STM32 never touches the camera — CV runs entirely on the laptop's own webcam, keeping firmware scope realistic
