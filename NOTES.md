# MX-5 Dash Android App — Working Notes

This is the standalone repo for the **MX-5 ND2 Telemetry Dashboard Android OBD-II
companion app** (Bluetooth Classic RFCOMM), split out of the `mx5-dash` ESP32
monorepo on 2026-09-10. Read this before continuing any work.

For project-wide engineering rules that still apply, see `GEMINI.md` in the
**parent `mx5-dash` repo** (Rule 2 covers the harmonized dual-platform UI;
Rules 3-7 cover CAN PIDs, polling, transmission state machine, and multi-target
builds).

---

## ✅ FIXED — The repo is now self-contained

The critical CMake external-dependency gap has been resolved (vendor-by-copy
approach). Shared files from the parent `mx5-dash/lib/` are now vendored locally
under `app/src/main/cpp/shared/` and CMake resolves them there. Build verified
with `./gradlew assembleDebug` — **BUILD SUCCESSFUL**.

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

**Chosen approach: Option 1 — vendor-by-copy.** Shared files from
`mx5-dash/lib/mx5_config/` and `mx5-dash/lib/mx5_ui/` are copied into
`app/src/main/cpp/shared/mx5_config/` and `app/src/main/cpp/shared/mx5_ui/`
respectively. CMake resolves them locally with no parent-repo dependency.
The ESP32 repo and this one build independently on the same laptop.
The parent repo's `native_preview` env does NOT `lib_ignore` `mx5_config`
(ESP32 native_preview also uses those shared headers), so each repo carries
its own copy — be mindful of keeping them in sync if the shared types change.

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
- Shared telemetry headers + font sources vendored locally under
  `app/src/main/cpp/shared/` (see FIXED section above).
- **`local.properties` is NOT committed** (machine-specific). Recreate it on
  any machine before building:
  `sdk.dir=/home/todd/Android/Sdk` (or your own SDK path).
- Build command: `./gradlew assembleDebug` (this repo is at root).
  **Verified BUILD SUCCESSFUL.**

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
   ├─ shared/
   │  ├─ mx5_config/                   vendored from parent: Config.h, VehicleData.h,
   │  │                                 ObdSource.h, DtcDatabase.h/.cpp, UserPrefs.h,
   │  │                                 SdLogger.h
   │  └─ mx5_ui/                       vendored fonts: lv_font_mono_{48,96,120}.c/.h
   ├─ AndroidObdSource.h               source abstraction used by the engine
   ├─ NativeBridge.cpp                 JNI bridge (JavaVM + engine lifecycle)
   ├─ lv_conf.h, lvgl_v9compat.h
   └─ third_party/lvgl/                vendored LVGL v9
```

- UI engine is **360x800 logical portrait** (`AndroidMx5UI::WIDTH/HEIGHT`),
  designed for a Pixel-class ~2:1.2 portrait screen (2410x1080 physical).
  Originally 800x360 landscape; ALL 17 screens redesigned for portrait on
  2026-09-10 (primary/large metrics at top, smaller supporting elements below).
  Horizontal swipe navigation retained. This DIFFERS from the ESP32 dash's
  480x320 — the screens are shared in *layout/styling concept* only, not
  geometry (per GEMINI.md Rule 2).
- Screen carousel indices (same order as ESP32): 0 Speed, 1 TPMS, 2 Temps,
  3 Diag, 4 Track, 5 RPM, 6 Trip, 7 Menu, 8-12 Diag sub-dashboards
  (fuel/cyl/chassis/smog/logs), 13 Settings, 14 BLE config.
- Transmission logic, PRND decoding, PID math mirror GEMINI.md Rules 3-6.

## Known Issues / Next Steps

1. ~~**Fix the CMake external-dependency gap (blocker)**~~ ✅ Fixed — vendor-by-copy.
2. ~~Verify a clean `./gradlew assembleDebug`~~ ✅ Verified BUILD SUCCESSFUL.
3. Re-test pairing (Classic BT) + live telemetry on device.
4. Continue whatever vehicle/test work was in flight in the parent repo; the
   two repos now evolve independently:
   - Dashboard UI changes → this repo (`android_ui/AndroidMx5UI.cpp`).
   - Shared telemetry types (`VehicleData.h`, `DtcDatabase.*`, `ObdSource.h`,
     `UserPrefs.h`, `SdLogger.h`, `Config.h`, mono fonts) → vendored locally
     under `shared/`. Keep in sync with parent repo if shared types change.

## Git

- `origin` → `https://github.com/Todd0042/mx5-dash-android.git` (`main`).
- Repo style: conventional commits (`fix(bt):`, `feat(track):`, ...).
- After any change: `git add -A && git commit && git push`.