# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

K3NG Rotator Controller: an Arduino-based interface that sits between a computer (logging/contest/tracking software) and an antenna rotator, emulating Yaesu GS-232A/B, Easycom, or DCU-1 control protocols. All source lives in [k3ng_rotator_controller/](k3ng_rotator_controller/), with the entire application built from `#include`d `.h`/`.cpp` files into one sketch, [k3ng_rotator_controller.ino](k3ng_rotator_controller/k3ng_rotator_controller.ino) (~22k lines). Full user documentation is the [GitHub wiki](https://github.com/k3ng/k3ng_rotator_controller/wiki), not this repo.

## Build commands (PlatformIO)

Configured in [platformio.ini](platformio.ini), source dir is `./k3ng_rotator_controller`.

```bash
pio run                          # build all environments
pio run -e nanoatmega328         # build for Arduino Nano (ATmega328)
pio run -e uno_r4_minima         # build for Arduino Uno R4 Minima (Renesas RA)
pio run -e nanoatmega328 -t upload   # build and flash
pio device monitor -b 9600       # serial monitor (matches monitor_speed)
```

There is no test suite — this is firmware; validation is build success plus on-hardware / serial-console testing (see below).

## Configuration model — read this before editing anything

Behavior is selected entirely at **compile time** via `#define FEATURE_*` / `OPTION_*` macros, not runtime config. There is no single settings file — there's a family of parallel files, and the active set is chosen by which `HARDWARE_*` macro (if any) is uncommented in [rotator_hardware.h](k3ng_rotator_controller/rotator_hardware.h):

| Concern | Default (no HARDWARE_* set) | Per-hardware-profile variants |
|---|---|---|
| Feature toggles | [rotator_features.h](k3ng_rotator_controller/rotator_features.h) | `rotator_features_m0upu.h`, `rotator_features_wb6kcn.h`, `rotator_features_wb6kcn_k3ng.h`, `rotator_features_test.h` |
| Pin assignments | [rotator_pins.h](k3ng_rotator_controller/rotator_pins.h) | `rotator_pins_m0upu.h`, `rotator_pins_wb6kcn.h`, `rotator_pins_wb6kcn_k3ng.h`, `rotator_pins_k3ng_g1000.h`, `rotator_pins_test.h`, `rotator_pins_wb6kcn_az_test_setup.h` |
| Tunable settings/constants | [rotator_settings.h](k3ng_rotator_controller/rotator_settings.h) | `rotator_settings_m0upu.h`, `rotator_settings_wb6kcn.h`, `rotator_settings_wb6kcn_k3ng.h`, `rotator_settings_test.h` |

The `.ino` includes the right trio based on which `HARDWARE_*` macro is defined (see the `#ifdef HARDWARE_*` chain around line 1123 for features, and the analogous chains later in the file for pins/settings). If you're customizing a personal build, edit the *default* `rotator_features.h` / `rotator_pins.h` / `rotator_settings.h` — do not touch the hardware-profile variants unless you're explicitly working on that profile.

[rotator_dependencies.h](k3ng_rotator_controller/rotator_dependencies.h) enforces cross-feature constraints at preprocess time (e.g. you can't enable both `FEATURE_YAESU_EMULATION` and `FEATURE_EASYCOM_EMULATION`; `FEATURE_MOON_TRACKING` requires `FEATURE_ELEVATION_CONTROL` and `FEATURE_CLOCK`; etc.) and implies certain macros from others (e.g. any I2C LCD feature implies `FEATURE_WIRE_SUPPORT`). When adding a new `FEATURE_*`/`OPTION_*`, add the relevant conflict/requirement checks here too.

Macro-based constants (state machine values, protocol codes, debug IDs) live in [rotator.h](k3ng_rotator_controller/rotator.h) — this is shared infrastructure, not per-build config.

## Code organization

- **k3ng_rotator_controller.ino** — everything: global state, `setup()`/`loop()`, azimuth/elevation reading and rotation state machines, serial command parsing (GS-232/Easycom/DCU-1), EEPROM read/write, LCD display updates, button/encoder handling, Nextion touchscreen protocol, debug output. Functions are typically guarded by the `FEATURE_*` macro they belong to (`#ifdef FEATURE_X ... #endif`), so grep for the feature macro to find all code paths it affects.
- **rotator_debug.cpp/h**, **rotator_debug_log_activation.h** — debug output plumbing and per-subsystem debug flags.
- **rotator_k3ngdisplay.cpp/h** — LCD display abstraction layer (classic 4-bit and various I2C displays).
- **rotator_language.h** — string tables per `LANGUAGE_*` macro (English, Spanish, Czech, Portuguese, German, French, etc.) for LCD/serial messages.
- **Nextion/** — `.HMI` project files and font data for the optional Nextion touchscreen UI (edited in the separate Nextion Editor tool, not by hand).
- **rotator_ethernet.h**, **rotator_stepper.h**, **rotator_clock_and_gps.h**, **rotator_command_processing.h** — currently empty stub headers reserved for future refactoring; don't assume logic lives there yet (it's still inline in the `.ino`).
- **libraries/** — vendored third-party Arduino libraries (bundled because most aren't in the standard Library Manager index, or need a specific fork/version). PlatformIO/Arduino IDE picks these up automatically from this path.
- **tle/** — TLE (orbital element) data files used by `FEATURE_SATELLITE_TRACKING`.

## Working with this codebase

- Almost all logic is wrapped in `#ifdef FEATURE_X` / `#if defined(...)`. When changing behavior, check what combination of features must be active for your code path to compile and run — `rotator_dependencies.h` documents the valid combinations.
- Multiple physical position-sensor types (potentiometer, rotary/incremental encoder, pulse input, several I2C compass/accelerometer chips) all funnel into the same azimuth/elevation state variables (`azimuth`, `raw_azimuth`, `target_azimuth`, etc., declared near the top of the `.ino`) — read `read_azimuth()`/`read_headings()` to see how a given sensor feature populates them.
- EEPROM-backed configuration (calibration tables, saved settings) has its own versioning: `CONFIGURATION_STRUCT_VERSION` in [rotator.h](k3ng_rotator_controller/rotator.h) and `calculated_configuration_struct_subversion()` in the `.ino`. Bump/handle these if you change what's persisted, or old EEPROM contents will be misread.
- Master/remote-slave operation (`FEATURE_MASTER_WITH_SERIAL_SLAVE` / `FEATURE_MASTER_WITH_ETHERNET_SLAVE` / `FEATURE_REMOTE_UNIT_SLAVE`) lets one Arduino host sensors physically remote from the control unit; check `rotator_dependencies.h` for what's mutually exclusive here.
- The repo currently tracks the maintainer's personal build config (uncommitted local tweaks to `rotator_features.h`/`rotator_settings.h` targeting pulse-input AZ/EL sensors, RFRobot I2C display, Uno R4 Minima). Don't "fix" these back to upstream defaults unless asked.

## Current pin assignment (default profile, no HARDWARE_* set)

Active feature set: AZ/EL rotator with pulse-input position sensors (`FEATURE_AZ_POSITION_PULSE_INPUT`, `FEATURE_EL_POSITION_PULSE_INPUT`), Yaesu GS-232B emulation, RFRobot I2C LCD (`FEATURE_RFROBOT_I2C_DISPLAY`), stall detection on both axes, manual rotate limits, limit-sense inputs. Pins are defined in [rotator_pins.h](k3ng_rotator_controller/rotator_pins.h); all pins not listed below are set to `0` (disabled) in this profile.

| Pin | Macro | Function |
|---|---|---|
| 2 | `az_position_pulse_pin` / interrupt 0 | Azimuth position pulse input (interrupt-driven pulse counting) |
| 3 | `el_position_pulse_pin` / interrupt 1 | Elevation position pulse input (interrupt-driven pulse counting) |
| 4 | `button_up` | Manual elevation UP button (to ground) |
| 5 | `button_down` | Manual elevation DOWN button (to ground) |
| 6 | `rotate_cw_pwm` | PWM output, azimuth CW rotation speed |
| 7 | `rotate_ccw_pwm` | PWM output, azimuth CCW rotation speed |
| 8 | `rotate_up_pwm` | PWM output, elevation UP speed |
| 9 | `rotate_down_pwm` | PWM output, elevation DOWN speed |
| 10 | `az_limit_sense_pin` | Azimuth limit sense input (active low stops AZ rotation) — `FEATURE_LIMIT_SENSE` |
| 11 | `el_limit_sense_pin` | Elevation limit sense input (active low stops EL rotation) — `FEATURE_LIMIT_SENSE` |
| A2 | `button_cw` | Manual azimuth CW button (to ground) |
| A3 | `button_ccw` | Manual azimuth CCW button (to ground) |
| SDA / SCL | (hardware I2C bus, not in `rotator_pins.h`) | RFRobot I2C LCD data/clock — `FEATURE_RFROBOT_I2C_DISPLAY` drives the display over `Wire` via the `LiquidCrystal_I2C` library (see `lib_deps` in [platformio.ini](platformio.ini)); no explicit LCD I2C address is set in source, so the library default is used. On the Nano (`nanoatmega328`) SDA/SCL are A4/A5; on the Uno R4 Minima they're the dedicated SDA/SCL pins. `FEATURE_WIRE_SUPPORT` (auto-enabled by `rotator_dependencies.h` whenever an I2C LCD feature is active) calls `Wire.begin()`. |

Notes:
- `az_rotation_stall_detected` / `el_rotation_stall_detected` (stall detection input pins, used by `FEATURE_AZ_ROTATION_STALL_DETECTION`/`FEATURE_EL_ROTATION_STALL_DETECTION`) are still set to `0` in `rotator_pins.h` — the stall-detection *feature* is enabled but no pin is wired yet. Assign a pin here if the physical stall sensor is connected.
- The classic 4-bit LCD pins (`lcd_4_bit_*`, pins 2/3/4/5/11/12) are defined but unused while `FEATURE_RFROBOT_I2C_DISPLAY` is active (I2C display uses `SDA`/`SCL` via `Wire`, not these pins) — ignore them for this build.
- `rotate_cw`, `rotate_ccw`, `rotate_up`, `rotate_down` (digital on/off direction relays) are `0`/disabled; this build drives rotation via the PWM pins only.
- Brake lines (`brake_az`, `brake_el`), speed/preset pots, park pins, and joystick pins are all `0`/unused in this build.
