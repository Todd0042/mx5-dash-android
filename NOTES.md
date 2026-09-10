# MX-5 Dash Android App — Working Notes

This is the standalone repo for the **MX-5 ND2 Telemetry Dashboard Android OBD-II
companion app** (Bluetooth Classic RFCOMM), split out of the `mx5-dash` ESP32
monorepo on 2026-09-10. Read this before continuing any work.

For project-wide engineering rules that still apply, see `GEMINI.md` in the
**parent `mx5-dash` repo** (Rule 2 covers the harmonized dual-platform UI;
Rules 3-7 cover CAN PIDs, polling, transmission state machine, and multi-target
builds).

---

## ⚠️ CRITICAL — The repo is NOT yet self-contained (do this first)

The native engine (`app/src/main/cpp/CMakeLists.txt`) currently resolves its
shared telemetry sources relative to the OLD monorepo layout:

```
get_filename_component(REPO_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/../../../../.." ABSOLUTE)
get_filename_component(MX5_UI     "${REPO_ROOT}/lib/mx5_ui"    ABSOLUTE)  # fonts
get_filename_component(MX5_CONFIG "${REPO_ROOT}/lib/mx5_config" ABSOLUTE) # core telemetry
```

Three levels up was the old `mx5-dash/OBD2Android/app/src/main/cpp`, so
`../../../../..` landed on the repo root with `lib/mx5_ui` + `lib/mx5_config`.
Now that the app lives at the root of this repo, that path resolves **one level
ABOVE this repo** (e.g. `/home/todd/Documents/GitHub`), where nothing exists.
The Gradle/CMake build will fail until this is fixed.

### Files the Android engine NEEDS from parent `mx5-dash/lib/`

From **`lib/mx5_config/`** (shared, pure-host headers — same `VehicleData.h`
used by ESP32 + native preview; destined for cross-repo sync):
- `Config.h`          (only `MX5_UNITS_US`, `MX5_BLE_DEVICE_PREFIX`,
                       `MX5_TPMS_ENABLED`, `MX5_LCD_ROTATION` are read by the
                       Android engine via `android_bridge/UserPrefs.cpp`)
- `VehicleData.h`     (telemetry snapshot struct)
- `ObdSource.h`       (BleDeviceInfo etc.)
- `DtcDatabase.h`     (+ `DtcDatabase.cpp` — compiled into the engine)
- `UserPrefs.h`       (interface; Android impl is `android_bridge/UserPrefs.cpp`)
- `SdLogger.h`        (interface; Android impl is `android_bridge/SdLogger.cpp`)

From **`lib/mx5_ui/`** (compiled into the engine):
- `lv_font_mono_48.c` / `.h`
- `lv_font_mono_96.c` / `.h`
- `lv_font_mono_120.c` / `.h`

### Recommended fix options
1. **Vendor-by-copy (simplest):** copy the files above into this repo under
   e.g. `app/src/main/cpp/shared/mx5_config/` and
   `app/src/main/cpp/shared/mx5_ui/`, then point `MX5_UI` / `MX5_CONFIG` at
   those local dirs. No parent-repo dependency at all.
2. **CMake path fix only:** change `REPO_ROOT` so it resolves to this repo root
   (1 fewer `..`), and add a `lib/` tree with just the needed files. Same end
   state as option 1, different layout.
3. **Git submodule / subtree** sharing `mx5-config` + `mx5-ui` from the parent
   repo. Cleaner long-term for the shared types, but heavier setup.

Chosen approach should still let the ESP32 repo and this one be built on the
same laptop without collisions. The `native_preview` env in the parent repo
`lib_ignore`s `mx5_config`... actually verify which way you went.

---

## Current State & Last Verified Behavior

- Android app is **working as of the pre-split snapshot** (imported from
  `mx5-dash/OBD2Android/`). Import commit: `8c6534d`.
- Repo pushed to GitHub: **https://github.com/Todd0042/mx5-dash-android**
  (`main` branch, tracks `origin/main`).
- The split intentionally left behind all ESP32/PlatformIO firmware changes
  (Waveshare bring-up, BLE scan filters, ST7796 etc.) that were mixed into the
  parent repo — they are irrelevant to this app.
- Nothing in this repo is known-affected by the ESP32 work; the two sides share
  only the files listed above.

### Build state of the import
- `2838` tracked files imported (source only; `.gradle/`, `build/`,
  `.kotlin/`, `lib/cpp/build`, `local.properties`, `.cxx` are gitignored —
  see `.gitignore`).
- **`local.properties` is NOT committed** (machine-specific). Recreate it on
  any machine before building:
  `sdk.dir=/home/todd/Android/Sdk` (or your own SDK path).
- Build command: `cd OBD2Android && ./gradlew assembleDebug` (this repo is at
  root, so it's just `./gradlew assembleDebug`). Toolchain: AGP 8.7.3,
  Kotlin 2.0.21, NDK 27.2.12479018, compileSdk 35 / minSdk 26 / targetSdk 34,
  Java 17, CMake 3.22.1 + `-DANDROID_STL=c++_static`. 16 KB ELF alignment
  flags are set in CMake for Android 15+ compliance.

## Architecture (so you can pick up fast)

```
app/src/main/
├─ java/com/mx5dash/obd2android/
│  ├─ Mx5Application.kt            app entry, foreground service wiring
│  ├─ bluetooth/
│  │  ├─ BluetoothSerialManager.kt RFCOMM serial layer (SPP) + connection mgmt
│  │  └─ BluetoothService.kt       foreground service / lifecycle
│  ├─ bridge/
│  │  ├─ NativeBridge.kt            JNI entry points
│  │  └─ PreferencesBridge.kt       prefs from Kotlin->C++
│  ├─ ui/
│  │  ├─ MainActivity.kt            host activity + screen/touch plumbing
│  │  └─ Mx5RenderView.kt           GL surface, LVGL rendering loop
│  ├─ storage/                      DataLogger.kt, IniExporter.kt, PreferencesManager.kt
│  └─ simulation/TelemetrySimulator.kt  mock data (preview/debug only)
└─ cpp/
   ├─ android_ui/AndroidMx5UI.cpp/.h   the ENTIRE dashboard UI engine (LVGL v9)
   ├─ android_bridge/                  Arduino.h, IPAddress.h stubs; UserPrefs.cpp,
   │                                    SdLogger.cpp (JNI-backed implementations)
   ├─ AndroidObdSource.h               source abstraction used by the engine
   ├─ NativeBridge.cpp                 JNI bridge (JavaVM + engine lifecycle)
   ├─ lv_conf.h, lvgl_v9compat.h
   └─ third_party/lvgl/                vendored LVGL v9
```

- UI engine is **800x360 logical** (`AndroidMx5UI::WIDTH/HEIGHT`), designed for
  a Pixel-class 19.5:9 screen (2410x1080 physical). This DIFFERS from the ESP32
  dash's 480x320 — the screens are shared in *layout/styling concept* only, not
  geometry (per GEMINI.md Rule 2).
- Screen carousel indices (same order as ESP32): 0 Speed, 1 TPMS, 2 Temps,
  3 Diag, 4 Track, 5 RPM, 6 Trip, 7 Menu, 8-12 Diag sub-dashboards
  (fuel/cyl/chassis/smog/logs), 13 Settings, 14 BLE config.
- Transmission logic, PRND decoding, PID math mirror GEMINI.md Rules 3-6.

## Known Issues / Next Steps

1. **Fix the CMake external-dependency gap (blocker)** — see top of file.
2. Verify a clean `./gradlew assembleDebug` after the fix; create
   `local.properties` first.
3. Re-test pairing (Classic BT) + live telemetry on device.
4. Continue whatever vehicle/test work was in flight in the parent repo; the
   two repos now evolve independently:
   - Dashboard UI changes → this repo (`android_ui/AndroidMx5UI.cpp`).
   - Shared telemetry types (`VehicleData.h`, `DtcDatabase.*`, `ObdSource.h`,
     `UserPrefs.h`, `SdLogger.h`, `Config.h`, mono fonts) → ideally shared via
     your chosen vendoring/submodule approach; avoid editing them in two places.

## Git

- `origin` → `https://github.com/Todd0042/mx5-dash-android.git` (`main`).
- Repo style: conventional commits (`fix(bt):`, `feat(track):`, ...).
- After any change: `git add -A && git commit && git push`.