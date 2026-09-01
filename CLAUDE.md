# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

K3NG Rotator Controller: an Arduino-based interface that sits between a computer (logging/contest/tracking software) and an antenna rotator, emulating Yaesu GS-232A/B, Easycom, or DCU-1 control protocols. All source lives in [k3ng_rotator_controller/](k3ng_rotator_controller/), with the entire application built from `#include`d `.h`/`.cpp` files into one sketch, [k3ng_rotator_controller.ino](k3ng_rotator_controller/k3ng_rotator_controller.ino) (~22k lines). Full user documentation is the [GitHub wiki](https://github.com/k3ng/k3ng_rotator_controller/wiki), not this repo.

## Build commands (PlatformIO)

`pio` is **not on PATH** — it lives in the PlatformIO venv:

```bash
~/.platformio/penv/bin/pio run -e uno_r4_minima         # build (the working target)
~/.platformio/penv/bin/pio run -e uno_r4_minima -t upload
~/.platformio/penv/bin/pio device monitor -b 9600       # serial console
```

`uno_r4_minima` builds clean (~34% flash, ~17% RAM of the RA4M1). Shared options (`framework`, `monitor_speed`, `lib_deps`) live in a common `[env]` section of [platformio.ini](platformio.ini) so every environment inherits them — note that `lib_deps` is *not* valid under `[platformio]`, which only takes project-level options like `src_dir`.

**`-e nanoatmega328` still fails**, now on `analogReadResolution' was not declared in this scope` — the active profile sets `FEATURE_ADC_RESOLUTION14` ([rotator_features.h:49](k3ng_rotator_controller/rotator_features.h#L49)), and `analogReadResolution()` exists only on the Uno R4 / Due / Teensy families, not on AVR. The Nano would need that feature off, and this feature set would still be a tight fit in the ATmega328's 2KB RAM. Bare `pio run` therefore reports one failure — pass `-e uno_r4_minima` for a clean run. Don't "repair" the Nano env unless asked.

There is no test suite — this is firmware. Validation is build success plus on-hardware testing over the serial console (see below).

## Validating changes over the serial console

The control port (`Serial`, 9600 baud, `CONTROL_PORT_MAPPED_TO` in [rotator_settings.h](k3ng_rotator_controller/rotator_settings.h)) accepts two command families, and this is the primary way to exercise a change without a computer-side program:

- **Yaesu GS-232B protocol commands** (dispatched around line 18000 of the `.ino`): `C` report azimuth, `C2` report az+el, `M###` rotate to azimuth, `W### ###` rotate to az+el, `B` report elevation, `R`/`L` manual CW/CCW, `U`/`D` elevation up/down, `A`/`S`/`E` stop, `X1`–`X4` speed, `O`/`F` offset and full-scale calibration. `H` prints the built-in help (`print_help()`, gated on `OPTION_SERIAL_HELP_TEXT`).
- **`\` backslash commands** — the controller's own extended/diagnostic set, handled by `process_backslash_command()` (~lines 14700–16806). Notably `\D` toggles debug output, `\E` initializes EEPROM to defaults, `\Q` saves settings to EEPROM and restarts, `\Ax[xx]` manually sets azimuth, `\Bx[xx]` manually sets elevation, `\Ix[xx]` sets the az starting point, `\Jx[xxxx]` sets az rotation capability, `\Nxx`/`\Fxx`/`\Wxxyyy` directly drive a pin on/off/PWM (very useful for bench-testing wiring), `\P` park, `\?` status queries.

`\E` followed by `\Q` is the reset-to-known-state sequence after changing anything EEPROM-backed.

Debug output is compiled in per-subsystem via `#define DEBUG_*` flags in [rotator_debug_log_activation.h](k3ng_rotator_controller/rotator_debug_log_activation.h) (currently active: `DEBUG_DUMP`, `DEBUG_VARIABLE_OUTPUTS`, `DEBUG_POSITION_PULSE_INPUT`, `DEBUG_LIMIT_SENSE`), then enabled at runtime with `\D`. Set `DEFAULT_DEBUG_STATE 1` there to capture debug from startup. Enabling many at once will blow the flash/RAM budget — turn on only the subsystem you're investigating.

## Configuration model — read this before editing anything

Behavior is selected entirely at **compile time** via `#define FEATURE_*` / `OPTION_*` macros, not runtime config. There is no single settings file — there's a family of parallel files, and the active set is chosen by which `HARDWARE_*` macro (if any) is uncommented in [rotator_hardware.h](k3ng_rotator_controller/rotator_hardware.h):

| Concern | Default (no HARDWARE_* set) | Per-hardware-profile variants |
|---|---|---|
| Feature toggles | [rotator_features.h](k3ng_rotator_controller/rotator_features.h) | `rotator_features_m0upu.h`, `rotator_features_wb6kcn.h`, `rotator_features_wb6kcn_k3ng.h`, `rotator_features_test.h` |
| Pin assignments | [rotator_pins.h](k3ng_rotator_controller/rotator_pins.h) | `rotator_pins_m0upu.h`, `rotator_pins_wb6kcn.h`, `rotator_pins_wb6kcn_k3ng.h`, `rotator_pins_k3ng_g1000.h`, `rotator_pins_test.h`, `rotator_pins_wb6kcn_az_test_setup.h` |
| Tunable settings/constants | [rotator_settings.h](k3ng_rotator_controller/rotator_settings.h) | `rotator_settings_m0upu.h`, `rotator_settings_wb6kcn.h`, `rotator_settings_wb6kcn_k3ng.h`, `rotator_settings_test.h` |

The `.ino` includes the right trio based on which `HARDWARE_*` macro is defined ([the features chain at ~line 1123](k3ng_rotator_controller/k3ng_rotator_controller.ino#L1123), pins at ~1246, settings at ~1270). If you're customizing a personal build, edit the *default* `rotator_features.h` / `rotator_pins.h` / `rotator_settings.h` — do not touch the hardware-profile variants unless you're explicitly working on that profile.

[rotator_dependencies.h](k3ng_rotator_controller/rotator_dependencies.h) enforces cross-feature constraints at preprocess time (e.g. you can't enable both `FEATURE_YAESU_EMULATION` and `FEATURE_EASYCOM_EMULATION`; `FEATURE_MOON_TRACKING` requires `FEATURE_ELEVATION_CONTROL` and `FEATURE_CLOCK`) and implies certain macros from others (e.g. any I2C LCD feature implies `FEATURE_WIRE_SUPPORT`). When adding a new `FEATURE_*`/`OPTION_*`, add the relevant conflict/requirement checks here too.

Macro-based constants (state machine values, protocol codes, debug IDs) live in [rotator.h](k3ng_rotator_controller/rotator.h) — shared infrastructure, not per-build config.

## Runtime architecture

`loop()` (~line 1899) is a cooperative round-robin of `service_*` / `check_*` functions — nothing blocks, everything is a state machine polled each pass. The rotation pipeline is the core of the program and spans several functions you should read together:

1. **Sensors → position.** `read_headings()` (~2554) calls `read_azimuth()` (~8125) / `read_elevation()` (~9562). Every supported sensor type (potentiometer, rotary/incremental encoder, pulse input, I2C compass/accelerometer) is `#ifdef`-selected inside these two functions and funnels into the same globals: `azimuth`, `raw_azimuth`, `target_azimuth`, `target_raw_azimuth` (declared ~1314). `raw_azimuth` is the uncorrected sensor reading; `convert_raw_azimuth_to_real_azimuth()` and `correct_azimuth()` apply the overlap/starting-point/calibration-table math to produce `azimuth`.

2. **Intent → request.** Anything that wants motion — serial commands (`check_serial()` ~3389), buttons (`check_buttons()` ~3874), preset pots/encoders, park, tracking — never drives pins directly. They all call `submit_request(axis, request, parm, called_by)` (~11295), which stores an `REQUEST_*` code (`REQUEST_AZIMUTH`, `REQUEST_CW`, `REQUEST_STOP`, `REQUEST_KILL`, …) into `az_request`/`el_request` and sets the queue state to `IN_QUEUE`. The `called_by` argument is a `DBG_*` traceback ID used only by debug output — pass a meaningful one when adding a call site.

3. **Request → motion.** `service_request_queue()` (~11957) consumes the queued request and transitions `az_state`/`el_state` through the state machine defined in [rotator.h](k3ng_rotator_controller/rotator.h#L13) (`IDLE`, `INITIALIZE_SLOW_START_CW`, `NORMAL_CW`, `SLOW_DOWN_CCW`, `INITIALIZE_DIR_CHANGE_TO_CW`, …). `service_rotation()` (~11350) then advances that state machine each pass and is the only thing that calls `rotator()` (~10150), the single choke point that actually asserts direction pins and calls `update_az_variable_outputs()` / `update_el_variable_outputs()` for PWM speed.

**When adding a behavior that moves the rotator, go through `submit_request()`** — bypassing it skips slow-start/slow-down ramping, direction-change handling, brake sequencing, and the safety checks below.

### Motion profile (`FEATURE_MOTION_PROFILE`, enabled in this build)

Stock K3NG ramps speed in *timed PWM steps* (`AZ_SLOW_START_UP_TIME`, `AZ_SLOW_DOWN_STEPS`) and starts braking at a *fixed distance* (`SLOW_DOWN_BEFORE_TARGET_AZ`), never knowing its actual speed. A new target arriving mid-rotation either did nothing (same direction — `// if we're already rotating CW, don't do anything`) or forced a full timed stop before reversing.

`service_motion_profile()` (~11395, called from `loop()` between `service_request_queue()` and `service_rotation()`) replaces that with a trapezoidal velocity profile per axis. Each pass it measures velocity from the position sensor, computes the fastest it may travel and still stop on target (`v = sqrt(2·a·d)`), ramps the commanded velocity toward that within the acceleration bound, and maps the result to PWM. Because the target only enters through the distance term, **a new target mid-rotation just changes `d`** — current velocity is preserved and the ramp is re-derived, so there is no jump; a reversal falls out of the same maths as the velocity ramps down through zero.

Consequences when editing this code:
- The axis stays in `NORMAL_CW`/`NORMAL_CCW` for the whole move. The `SLOW_START_*`/`SLOW_DOWN_*` states and `INITIALIZE_DIR_CHANGE_TO_*` are bypassed for *targeted* moves (they still run for manual `REQUEST_CW`/`REQUEST_CCW`, which have no target to brake toward). The old ramp blocks in `service_rotation()` are guarded by `if (!az_profile_active)`.
- `update_az_variable_outputs()` only writes PWM when `az_state` is a rotation state, so anything that leaves `az_state` latched in a braking state silently swallows the profile's output. This is why the same-direction retarget path forces `SLOW_DOWN_CW → NORMAL_CW`.
- Inside `AZIMUTH_TOLERANCE`/`ELEVATION_TOLERANCE` the profile commands zero and returns, letting `service_rotation()`'s target check stop the axis. Removing that bail-out makes the axis hunt around the target in a limit cycle.
- Tuning lives in `rotator_settings.h`: `AZ`/`EL_MAX_ACCELERATION_DPSS`, `AZ`/`EL_MAX_DECELERATION_DPSS` (degrees/sec²), `AZ`/`EL_FULL_SPEED_DEG_PER_SEC` (**must be measured on the hardware** — it maps deg/s onto PWM), and `AZ`/`EL_MOTION_PROFILE_MIN_PWM` (stiction floor). Arrival speed is `sqrt(2·decel·tolerance)`, so deceleration and tolerance together set how hard the axis is still moving when it cuts — that, not the profile, is where residual overshoot comes from.
- `DEBUG_MOTION_PROFILE` in `rotator_debug_log_activation.h` prints distance, commanded and measured velocity per update.

Safety and limit logic runs as independent pollers that call `submit_request(..., REQUEST_KILL/REQUEST_STOP, ...)`: `check_limit_sense()` (~13134), `check_az_manual_rotate_limit()` / `check_el_manual_rotate_limit()` (~3178/3202), `az_check_rotation_stall()` / `el_check_rotation_stall()` (~7917/7959), and `az_check_operation_timeout()` / `el_check_operation_timeout()` (`OPERATION_TIMEOUT`, 120 s).

## Code organization

- **k3ng_rotator_controller.ino** — everything listed above plus EEPROM read/write, LCD display updates, encoder handling, Nextion touchscreen protocol, and debug output. Functions are typically guarded by the `FEATURE_*` macro they belong to, so grep for the feature macro to find all code paths it affects.
- **rotator_debug.cpp/h**, **rotator_debug_log_activation.h** — debug output plumbing and per-subsystem debug flags.
- **rotator_k3ngdisplay.cpp/h** — LCD display abstraction (classic 4-bit and various I2C displays).
- **rotator_language.h** — string tables per `LANGUAGE_*` macro for LCD/serial messages.
- **Nextion/** — `.HMI` project files and font data for the optional Nextion touchscreen UI (edited in the separate Nextion Editor tool, not by hand).
- **rotator_ethernet.h**, **rotator_stepper.h**, **rotator_clock_and_gps.h**, **rotator_command_processing.h**, **rotator_moon_and_sun.h** — empty stub headers reserved for future refactoring; that logic is still inline in the `.ino`.
- **libraries/** — vendored third-party Arduino libraries (bundled because most aren't in the Library Manager index, or need a specific fork/version). PlatformIO/Arduino IDE picks these up from this path.
- **tle/** — TLE orbital element data for `FEATURE_SATELLITE_TRACKING`.

## Other things to know

- Almost all logic is wrapped in `#ifdef FEATURE_X` / `#if defined(...)`. When changing behavior, check what combination of features must be active for your code path to compile and run.
- Use the `*Enhanced` pin wrappers (`pinModeEnhanced`, `digitalWriteEnhanced`, `digitalReadEnhanced`, `analogWriteEnhanced`, ~13378–13442) rather than raw Arduino calls — they honor the "pin 0 = disabled" convention used throughout the pins files and route through remote-unit/I2C expanders where those features are active.
- EEPROM-backed configuration (calibration tables, saved settings) is versioned by `CONFIGURATION_STRUCT_VERSION` in [rotator.h](k3ng_rotator_controller/rotator.h#L2) and `calculated_configuration_struct_subversion()` (~7172). Bump/handle these if you change what's persisted, or old EEPROM contents will be misread. Writes are deferred: setting `configuration_dirty = 1` lets `check_for_dirty_configuration()` (~12562) batch the write.
- Master/remote-slave operation (`FEATURE_MASTER_WITH_SERIAL_SLAVE` / `FEATURE_MASTER_WITH_ETHERNET_SLAVE` / `FEATURE_REMOTE_UNIT_SLAVE`) lets one Arduino host sensors physically remote from the control unit; see `rotator_dependencies.h` for what's mutually exclusive.
- The repo tracks the maintainer's personal build config (pulse-input AZ/EL sensors, RFRobot I2C display, Uno R4 Minima). Don't "fix" these back to upstream defaults unless asked.

## Current build profile (default, no HARDWARE_* set)

AZ/EL rotator with pulse-input position sensors (`FEATURE_AZ_POSITION_PULSE_INPUT`, `FEATURE_EL_POSITION_PULSE_INPUT`, both with `OPTION_*_POSITION_PULSE_HARD_LIMIT`), Yaesu GS-232B emulation (`OPTION_GS_232B_EMULATION`), RFRobot I2C LCD, stall detection on both axes, manual rotate limits, limit-sense inputs with calibration, timed buffer, 14-bit ADC (`FEATURE_ADC_RESOLUTION14`, Uno R4 family). Pins are in [rotator_pins.h](k3ng_rotator_controller/rotator_pins.h); every pin not listed below is `0` (disabled).

| Pin | Macro | Function |
|---|---|---|
| 2 | `az_position_pulse_pin` / interrupt 0 | Azimuth position pulse input (interrupt-driven) |
| 3 | `el_position_pulse_pin` / interrupt 1 | Elevation position pulse input (interrupt-driven) |
| 4 | `button_up` | Manual elevation UP button (to ground) |
| 5 | `button_down` | Manual elevation DOWN button (to ground) |
| 6 | `rotate_cw_pwm` | PWM output, azimuth CW speed |
| 7 | `rotate_ccw_pwm` | PWM output, azimuth CCW speed |
| 8 | `rotate_up_pwm` | PWM output, elevation UP speed |
| 9 | `rotate_down_pwm` | PWM output, elevation DOWN speed |
| 10 | `az_limit_sense_pin` | Azimuth limit sense input (active low stops AZ) — `FEATURE_LIMIT_SENSE` |
| 11 | `el_limit_sense_pin` | Elevation limit sense input (active low stops EL) — `FEATURE_LIMIT_SENSE` |
| A2 | `button_cw` | Manual azimuth CW button (to ground) |
| A3 | `button_ccw` | Manual azimuth CCW button (to ground) |
| SDA / SCL | (hardware I2C, not in `rotator_pins.h`) | RFRobot I2C LCD via `Wire` / `LiquidCrystal_I2C`; no explicit address set in source, so the library default is used. `FEATURE_WIRE_SUPPORT` (auto-enabled by `rotator_dependencies.h`) calls `Wire.begin()`. |

Notes:
- `az_rotation_stall_detected` / `el_rotation_stall_detected` are still `0` in `rotator_pins.h` — stall detection is enabled as a *feature* (tuned by `STALL_CHECK_FREQUENCY_MS_AZ`/`_EL` and `STALL_CHECK_DEGREES_THRESHOLD_AZ`/`_EL` in `rotator_settings.h`) but no pin is wired yet. Assign a pin here if a physical stall sensor is connected.
- Classic 4-bit LCD pins (`lcd_4_bit_*`, pins 2/3/4/5/11/12) are defined but unused while `FEATURE_RFROBOT_I2C_DISPLAY` is active — and note they collide with the pulse-input and limit-sense pins above, so don't re-enable that display without reassigning.
- `rotate_cw`/`rotate_ccw`/`rotate_up`/`rotate_down` (on/off direction relays) are `0`; this build drives rotation via PWM pins only. Brakes, speed/preset pots, park, and joystick pins are all unused.
- Tuning currently in effect: `AZIMUTH_TOLERANCE 1.0`, `ELEVATION_TOLERANCE 0.33`, manual rotate limits AZ −1…360 and EL −1…120.
