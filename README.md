<p align="center">
  <a href="https://ko-fi.com/todd0042">
    <img src="https://raw.githubusercontent.com/Todd0042/creator-assets/main/banners/todd_creator_banner_v1.0.png" alt="Support Todd on Ko-fi" width="100%" />
  </a>
</p>

---

# 📱 MX-5 Dash Android: Companion Telemetry App

[![Version](https://img.shields.io/badge/Version-1.0.0-ff2a85.svg)](https://github.com/Todd0042/mx5-dash-android/releases)
[![License: MIT](https://img.shields.io/badge/License-MIT-purple.svg)](LICENSE)
[![Platform: Android 8.0+](https://img.shields.io/badge/Platform-Android%208.0%2B%20(API%2026%2B)-3DDC84.svg?logo=android&logoColor=white)](https://developer.android.com)
[![Hardware Link](https://img.shields.io/badge/Hardware-ESP32--S3%20Companion-orange.svg)](https://github.com/Todd0042/mx5-dash)
[![Engine: C++ & Native LVGL](https://img.shields.io/badge/Engine-C%2B%2B%20%26%20Native%20LVGL-blue.svg)]()
[![Support on Ko-fi](https://img.shields.io/badge/Ko--fi-Support%20Creator-F16061?style=flat&logo=ko-fi&logoColor=white)](https://ko-fi.com/todd0042)

> **The standalone Android companion application for the Mazda MX-5 Miata (ND2 / SkyActiv-G 2.0L) Telemetry Dashboard — rendering the exact high-contrast LVGL vector gauge cluster natively on Android smartphones, tablets, and infotainment head units via Bluetooth Classic (RFCOMM) and Bluetooth Low Energy (BLE).**

```
       _________________________________________________________________________
      |  MAZDA MX-5 MIATA (ND2)                     [GEAR: 4]     SPEED: 62 MPH |
      |  TACH [████████████████████░░░░░] 5,200 RPM  COOLANT: 198°F  OIL: 215°F |
      |_________________________________________________________________________|
      |  TPMS [FL: 30 PSI] [FR: 30 PSI]             0-60 MPH: 5.68s [BEST]      |
      |       [RL: 30 PSI] [RR: 30 PSI]             FUEL: 64%       AFR: 14.7   |
      |_________________________________________________________________________|
```

---

## 📖 Table of Contents

* **Part 1 — Consumer Overview & Quick Start**
  * [🏎️ What Is MX-5 Dash Android?](#-what-is-mx-5-dash-android)
  * [⚡ Quick Start in 3 Steps](#-quick-start-in-3-steps)
  * [📱 Automotive Cockpit Glanceability](#-automotive-cockpit-glanceability)
  * [🚀 Quick Download & Installation (Android APK)](#-quick-download--installation-android-apk)
* **Part 2 — Consolidated Technical Documentation & Architecture Reference**
  * [1. Unified C++ LVGL Engine & Android JNI Bridge](#1-unified-c-lvgl-engine--android-jni-bridge)
  * [2. Bluetooth Classic (RFCOMM) & BLE Transport Stack](#2-bluetooth-classic-rfcomm--ble-transport-stack)
  * [3. SkyActiv-G 2.0L Shared PID Pipeline](#3-skyactiv-g-20l-shared-pid-pipeline)
  * [4. Standalone ESP32 Hardware Sister Project](#4-standalone-esp32-hardware-sister-project)
  * [5. Automated Build Verification & Clean-Room Licensing](#5-automated-build-verification--clean-room-licensing)

---

# Part 1 — Consumer Overview & Quick Start

## 🏎️ What Is MX-5 Dash Android?

**MX-5 Dash Android** is the native Android implementation of the **[MX-5 Dash](https://github.com/Todd0042/mx5-dash)** telemetry dashboard.

If you don't have the standalone Waveshare ESP32-S3 physical display hardware in your vehicle, MX-5 Dash Android allows you to mount any Android phone, tablet, or aftermarket head unit to your ND2 dashboard and experience the exact same 40 Hz sports car gauge cluster:
- **🏎️ 3-Foot Driving Glanceability**: Massive 120px speed numbers, redline indicators, and color-coded status beacons.
- **🛞 4-Wheel OEM TPMS**: Individual tire pressures (FL, FR, RL, RR in PSI) and tire temperatures (°F).
- **🌡️ SkyActiv Fluid Temps**: Real Engine Oil Temp (`221310`), Coolant Temp (ECT), and Intake Air Temp (IAT).
- **⏱️ Automated 0–60 MPH Launch Timer**: Performance timer and live pedal response telemetry.

---

## ⚡ Quick Start in 3 Steps

1. **Plug in Wireless OBD Scanner**: Plug a standard Bluetooth (SPP) or BLE OBD-II adapter (e.g. *OBDLink MX+*, *vLinker MS/MC*, *Veepeak*) into your Miata's OBD-II port.
2. **Mount Your Device**: Place your Android phone or tablet on your dashboard or phone mount.
3. **Launch & Connect**: Open MX-5 Dash Android and tap your scanner in the device list. Live gauges start streaming immediately.

---

## 🚀 Quick Download & Installation (Android APK)

1. Download **`mx5-dash-android-v1.0.0-debug.apk`** from [Releases](https://github.com/Todd0042/mx5-dash-android/releases).
2. Install the APK on any Android 8.0+ device (API 26 or higher).
3. Grant **Bluetooth / Nearby Devices** permission when prompted.

---

# Part 2 — Consolidated Technical Documentation & Architecture Reference

## 1. Unified C++ LVGL Engine & Android JNI Bridge

MX-5 Dash Android shares the exact same core C++ graphics engine as the ESP32 physical display:
- Compiled via **CMake** and **Android NDK** (`app/src/main/cpp/`).
- Directly executes the native C++ **LVGL 9** vector rendering engine into an Android native `SurfaceView` buffer for zero-overhead 60 FPS gauge rendering.

---

## 2. Bluetooth Classic (RFCOMM) & BLE Transport Stack

- Supports standard Android Bluetooth SPP serial sockets (`UUID 00001101-0000-1000-8000-00805F9B34FB`).
- Supports Bluetooth Low Energy (GATT) communication for modern ultra-low-power adapters.

---

## 3. SkyActiv-G 2.0L Shared PID Pipeline

The Android engine implements the identical tiered interleaving scheduler:
- Polling Speed (`010D`) at ~40 Hz.
- Cycling secondary powertrain metrics (RPM, Throttle, Coolant, Oil Temp, Fuel Rail PSI) with Header `7E0`.
- Staggered BCM/Cluster queries with Header `720` for 4-wheel tire pressures and fuel level %.

---

## 4. Standalone ESP32 Hardware Sister Project

Prefer a permanent physical dash display built into your Miata's air vent pod?  
Check out the dedicated ESP32-S3 firmware repository:  
👉 **[https://github.com/Todd0042/mx5-dash](https://github.com/Todd0042/mx5-dash)**

---

## 5. Automated Build Verification & Clean-Room Licensing

```bash
# Build debug APK:
./gradlew assembleDebug
```

Licensed under the **MIT License** — see [LICENSE](LICENSE).

---

<p align="center">
  If you enjoy using MX-5 Dash Android, consider supporting open-source development:<br/><br/>
  <a href="https://ko-fi.com/todd0042" target="_blank">
    <img src="https://img.shields.io/badge/☕_Support_MX--5_Dash_on_Ko--fi-F16061?style=for-the-badge&logo=ko-fi&logoColor=white" alt="Support on Ko-fi">
  </a>
</p>

---

<p align="center">
  <a href="https://ko-fi.com/todd0042">
    <img src="https://raw.githubusercontent.com/Todd0042/creator-assets/main/banners/todd_creator_banner_v2.0.png" alt="Support Todd on Ko-fi" width="100%" />
  </a>
</p>
