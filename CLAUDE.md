# SwiftProForArduino — Claude Context

## What this is
Firmware for the **uArm Swift Pro** robotic arm (v4.9.0). It is Grbl v0.9j
(open-source CNC motion controller) ported to the ATmega2560 and extended with
uArm-specific kinematics, a proprietary serial command protocol, and peripheral
drivers (pump, gripper, laser, end-effector, angle encoders).

## Target hardware
- **MCU**: ATmega2560 @ 16 MHz (Arduino Mega 2560)
- **Serial**: USART0, 115200 8-N-1

## Build system
**PlatformIO** — official installer puts the CLI at `~/.platformio/penv/bin/pio`.

`platformio.ini` is at the repo root:
```ini
[env:megaatmega2560]
platform = atmelavr
board = megaatmega2560
framework = arduino
upload_speed = 115200
```

Build: `pio run`  
Flash: `pio run --target upload`  
Output: `.pio/build/megaatmega2560/firmware.hex`

Pre-built release hexes live in `hex/` for comparison or end-user flashing.

The Arduino IDE path also works via `src/grblUpload/grblUpload.ino`, but
PlatformIO is the canonical build.

### Why `framework = arduino`
`uarm_common.h` and `step_lowlevel.h` both `#include <Arduino.h>`. No Arduino
runtime APIs (millis, Serial) are used — just the headers for AVR type
definitions. All I²C is bit-banged directly against AVR port registers
(`uarm_iic.c`); no Wire library.

### `main()` override
Grbl provides `int main(void)` in `main.c`. The Arduino core also provides
`main()` in `main.cpp`, but that file is compiled into `core.a` (an archive).
Standard AVR-GCC linker behaviour: direct `.o` files take precedence over
archive members, so Grbl's `main()` wins and the Arduino one is dead code.
This is intentional — there is no `setup()`/`loop()` anywhere.

## Source layout
All firmware source is flat in `src/`. Two interleaved layers:

### Grbl v0.9j core (`src/`)
| Files | Role |
|---|---|
| `main.c` | Entry point; inits Grbl then calls `uarm_swift_init()` |
| `gcode.c/h` | G-code parser |
| `motion_control.c/h` | High-level move commands |
| `planner.c/h` | Look-ahead motion planner |
| `stepper.c/h`, `step_lowlevel.c/h` | Step-pulse ISR and low-level stepper I/O |
| `serial.c/h`, `protocol.c/h` | Serial I/O and command loop |
| `limits.c/h`, `probe.c/h` | Limit-switch and probe handling |
| `settings.c/h`, `eeprom.c/h` | Persistent settings in AVR EEPROM |
| `report.c/h`, `print.c/h` | Status/error reporting |
| `spindle_control.c/h`, `coolant_control.c/h` | Spindle PWM and coolant |
| `system.c/h`, `nuts_bolts.c/h` | System state and utility functions |
| `config.h` | Compile-time configuration (selects `CPU_MAP_ATMEGA2560`) |
| `cpu_map.h`, `cpu_map/` | Per-MCU pin mapping headers |
| `defaults.h`, `defaults/` | Machine-profile defaults |

### uArm extensions (`src/uarm_*`)
| Files | Role |
|---|---|
| `uarm_swift.c/h` | Top-level init/tick; version strings (`V4.9.0` / API `V4.0.5`) |
| `uarm_protocol.c/h` | uArm G/M/P command families (G2xxx, M2xxx, P2xxx) |
| `uarm_coord_convert.c/h` | Kinematics: Cartesian ↔ joint-angle ↔ step |
| `uarm_angle.c/h` | AS5600 magnetic encoder reading via bit-bang I²C |
| `uarm_iic.c/h` | Bit-bang I²C for arm encoders and external EEPROM |
| `uarm_peripheral.c/h` | Pump, gripper, laser, end-effector control |
| `uarm_timer.c/h` | Software timers using AVR Timer2/3/4/5 |
| `uarm_common.h` | Shared `uarm_state_t`, work-mode enum, EEPROM address map |
| `uarm_debug.h` | Debug print macros |

## Joint → Grbl axis mapping
The three robot joints are mapped onto Grbl's X/Y/Z axes in `cpu_map_atmega2560.h`:
- **ARML** (left arm link) → X axis
- **ARMR** (right arm link) → Y axis
- **BASE** (rotation) → Z axis

Each joint has independent step/dir/enable pins across AVR ports F, L, K, D.
The standard Grbl step/dir port block is commented out and replaced by these
per-axis macros.

## Serial command protocol
Full reference: `doc/uArm Swift Pro Protocol V4.0.5_en.pdf`

Connect at 115200 8-N-1. Key command families:
- **Move**: `G0`, `G1`, `G2004`, `G2201`, `G2202`, `G2204`, `G2205`
- **Set**: `M17`, `M204`, `M2019`, `M2120`–`M2122`, `M2201`–`M2203`,
  `M2210`, `M2215`, `M2220`–`M2222`, `M2231`–`M2233`, `M2400`, `M2401`,
  `M2410`, `M2411`, `M2240`, `M2241`
- **Query**: `P2200`–`P2206`, `P2220`–`P2221`, `P2231`–`P2234`,
  `P2240`–`P2242`, `P2400`

Example script: `example/pick_square.txt`

## EEPROM layout (key addresses)
Defined in `uarm_common.h`:
- `1U` — angle calibration map (< 600 bytes)
- `800U` / `820U` — angle reference (old/new)
- `900U` — work mode
- `1002U` — BT MAC address
- `3200U` — parameter block (from v3.2.0 firmware)
- `3202U` — effect angle offset

## Variants
Controlled by defines in `uarm_common.h` (all commented out = SwiftPro default):
- `UARM_MINI` — uArm Mini
- `UARM_2500` — uArm3Plus

## Git history note
`platformio.ini` was absent from `Version_V4.0` but existed in an orphan commit
`5104434` ("travis build test"). It has been restored to the repo root.
The `.travis.yml` references a `travis` / `develop` branch workflow that no
longer exists on the remote.
