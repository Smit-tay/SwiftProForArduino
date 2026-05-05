# Swift Pro Firmware V4.9.1-jss.5 — fork of [uArm-Developer/SwiftProForArduino V4.9.0](https://github.com/uArm-Developer/SwiftProForArduino), based on [Grbl v0.9j](https://github.com/grbl/grbl)

----------
## Update Summary

**V4.9.1-jss.5 — zero warnings, hardware-verified, smoother motion.**

This fork started life inheriting 112 compiler warnings from upstream. As of jss.5, that count is zero, and the cleanup turned up real bugs along the way — not just cosmetic noise.

### Real bugs found and fixed

- **Silent-skip in `mc_line`.** Illegal segments mid-trajectory were silently skipped, letting the planner stitch legal segments together via step-space interpolation through the forbidden joint region — and report success. Now returns `UARM_COORD_ERROR` honestly.
- **Memory corruption in M2211/M2212.** `sscanf` with `%d` was writing int-sized values into `unsigned char` destinations, clobbering adjacent stack bytes on every invocation. Fixed by switching to `%hhd` (avr-libc support hardware-verified on this toolchain).
- **EEPROM checksum rotate.** `||` (logical OR) where `|` (bitwise OR) was intended. Beautifully self-concealing — identically broken on read and write paths, so every EEPROM ever written validated against its own broken algorithm and passed.
- **Undefined behaviour in `mc_line` check-mode.** Bare `return;` in a non-void function propagated whatever was in the return register as G-code execution status. Usually benign on AVR, formally UB.

### Performance and behavioural improvements

- **Junction deviation** raised from 0.01mm to 0.05mm. The original value was a CNC-router inheritance forcing near-stops at every direction change — audibly jerky motion and peak-torque events on plastic gears at every junction. New value allows continuous motion through corners while remaining well below the parallel linkage's dynamics envelope. Verified smooth at F3000 zigzags with no oscillation.
- **Acceleration ramp resolution** doubled (`ACCELERATION_TICKS_PER_SECOND` 100 → 200). Finer velocity control during ramps, smoother low-feedrate motion.
- **Look-ahead depth** increased (`BLOCK_BUFFER_SIZE` 18 → 24). Better planner decisions on short-segment sequences.
- **Encoder averaging** reduced from 5 to 3 samples. Cuts I²C read time per cycle from ~21ms to ~12.6ms, enabling clean 25-30Hz position reporting.
- **Defensive returns** added against future enum extensions in `check_encoder`, `get_point_b_angle`, and `uarm_cmd_p2244`. `NaN` return from `get_point_b_angle` integrates with existing `is_angle_legal()` checks for fail-safe behaviour.
- **Dead code excluded** from the build (`step_lowlevel.c` — never called, latent Timer4 conflict).
- **Redundant work eliminated** (`mc_line` was re-reading full EEPROM settings on every move command).

### Build

| Metric   | Phase 4 baseline | jss.5        | Delta            |
|----------|------------------|--------------|------------------|
| Flash    | 66,966 bytes     | 65,480 bytes | −1,486 bytes     |
| RAM      | 5,112 bytes      | 5,351 bytes  | +239 bytes       |
| Warnings | 19               | 0            | −19 (−112 from upstream) |

Hardware verified post-flash on uArm Swift Pro: settings restore correctly, all known-good moves succeed, deliberately-illegal moves return E22 cleanly, check-mode regression clean, junction smoothness retained.

Tagged `V4.9.1-jss.5`.

## Caution
#### The current firmware is not perfect and will be updated periodically
#### Not support app control 
#### For uArmStudio control

- input and Grove module in BLOCKLY are not supported.
- 3D Printing is not supported.

## Communication protocol
#### For serial terminal control

First, connect to uArm using the serial terminal of your choice.Set the baud rate to 115200 as 8-N-1 (8-bits, no parity, and 1-stop bit.) .Cmd list reference to [protocol documents](doc/).

* move cmd support **G0,G1,G2004,G2201,G2202,G2204,G2205**.
* setting cmd support **M17,M204,M2019,M2120,M2121,M2122,M2201,M2202,M2203,M2210,M2215,M2220,M2221,M2222,M2231,M2232,M2233,M2400,M2401,M2410,M2411,M2240,M2241**.                                                                                                                                                                           
not support currently **M2211,M2212,M2213,M2234,M2245**.
* query cmd support 
**P2200,P2201,P2202,P2203,P2204,P2205,P2206,P2220,P2221,P2231,P2231,P2232,P2233,P2234,P2240,P2241,P2242,P2400**.                                                                  
## How to upgrade uArm

### 1、Flashing Firmware to uArm

#### To Determine your uArm's COM port:

**Windows:**
* Windows XP: Right click on "My Computer", select "Properties", select "Device Manager".
* Windows 7: Click "Start" -> Right click "Computer" -> Select "Manage" -> Select "Device Manager" from left pane.
* In the tree, expand "Ports (COM & LPT)".
* Your uArm will be the **Arduino Mega 2560 (COMX)**, the "X" represents the COM number, for example COM6.
* If there are multiple USB serial ports, right click each one and check the manufacturer, the Arduino will be "FTDI".

**Linux:**
* Plug the uArm in via USB.
* Open a terminal and run `ls /dev/ttyACM*` (or `ls /dev/ttyUSB*` on older boards). The uArm will typically appear as `/dev/ttyACM0`.
* To confirm, run `dmesg | tail` immediately after plugging in — you should see a line identifying the device, e.g. `cdc_acm 1-1:1.0: ttyACM0: USB ACM device`.
* If your user account doesn't have permission to access the port, add yourself to the `dialout` group (`sudo usermod -aG dialout $USER`) and log out/in.

#### To flash hex to Swift Pro:

**Windows (XLoader):**
* Download the [hex](hex/)
* Download and extract [XLoader](https://raw.githubusercontent.com/uArm-Developer/SwiftProForArduino/Version_V4.0/imagaes/XLoader.zip).
* Open XLoader and select your uArm's COM port from the drop down menu on the lower left.
* Select the appropriate device from the dropdown list titled "Device".
* Check that Xloader set the correct baud rate for the device: 115200 for Mega (ATMEGA2560).
* Now use the browse button on the top right of the form to browse to your hex file.
* Once your hex file is selected, click "Upload".

The upload process generally takes about 10 seconds to finish. Once completed, a message will appear in the bottom left corner of XLoader telling you how many bytes were uploaded. If there was an error, it would show instead of the total bytes uploaded.

**Linux (avrdude):**

Most distributions ship `avrdude` in their package manager (`sudo dnf install avrdude` on Fedora, `sudo apt install avrdude` on Debian/Ubuntu). To flash a hex file:

```bash
avrdude -c wiring -p atmega2560 -P /dev/ttyACM0 -b 115200 -D -U flash:w:firmware.hex:i
```

Replace `firmware.hex` with the path to the hex you want to flash, and `/dev/ttyACM0` with the port identified above. Flags explained:

* `-c wiring` — programmer protocol used by the Mega 2560 bootloader.
* `-p atmega2560` — target MCU.
* `-P /dev/ttyACM0` — serial port.
* `-b 115200` — baud rate.
* `-D` — disable auto-erase (the bootloader handles erase).
* `-U flash:w:firmware.hex:i` — write the hex file to flash memory.

The upload takes ~10 seconds and ends with `avrdude done. Thank you.` on success.

**Linux (PlatformIO, if building from source):**

If you're building this firmware from source rather than flashing a prebuilt hex, PlatformIO handles flashing directly:

```bash
pio run --target upload --upload-port /dev/ttyACM0
```
Run from the repository root. PlatformIO invokes `avrdude` under the hood with the correct flags.

### 2、Control your uArm
you have three ways to control your uArm:

* using the serial terminal [example](example)
* using the [Python library 2.0](https://github.com/uArm-Developer/uArm-Python-SDK/tree/2.0 "Python library 2.0")
* using the [https://github.com/uArm-Developer/uArm-SDK](https://github.com/uArm-Developer/uArm-SDK "C++ library")
* using the uArmStudio


## License

Swift Pro Firmware is published under the [GPL license](/LICENSE) 







