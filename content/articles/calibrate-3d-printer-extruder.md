---
publish: true
created: 2026-07-31
modified: 2026-08-07T09:58:51.329Z
---

# Manually calibrate an FLSUN Delta extruder

This procedure is for an older FLSUN Delta printer whose firmware does not provide a calibration wizard or other built-in calibration tools. Instead, the extruder must be calibrated manually by connecting to the printer over USB and sending G-code commands.

The setting being adjusted is the extruder's **steps per millimeter**: the number of motor steps used to feed one millimeter of filament. An incorrect value can cause thin walls, inconsistent infill, and over- or under-extrusion. To correct it, request 100 mm of filament, measure how much the printer actually feeds, then update the setting with the measured result.

## Measure the extrusion

1. Heat the nozzle to 210 °C, or to the normal printing temperature of the loaded filament.

2. Mark the filament 120 mm before it enters the extruder. The extra 20 mm keeps the mark measurable if the printer feeds too much.

3. Connect to the printer over USB at 115200 baud using the Arduino IDE serial monitor or a terminal, as described below.

4. Put the extruder in relative positioning mode:

   ```gcode
   M83
   ```

5. Slowly extrude 100 mm of filament:

   ```gcode
   G1 E100 F100
   ```

6. Measure the distance from the mark to the extruder. The actual extrusion is `120 mm - remaining distance`.

## Calculate and save the correction

Ask the FLSUN firmware to display its EEPROM settings, then find `Extr.1 steps per mm` in the response:

```gcode
M205
```

Calculate the corrected value:

```text
new steps/mm = current steps/mm × 100 / actual extrusion in mm
```

Apply the result, replacing `XXX.XX` with the calculated value, then save it:

```gcode
M92 EXXX.XX
M500
```

Repeat the measurement before printing to confirm that the extruder now feeds 100 mm.

## Example

On 4 May 2026, the printer fed only 45 mm with an existing setting of 440 steps/mm:

```text
440 × 100 / 45 = 977.78 steps/mm
```

The theoretical correction would therefore be:

```gcode
M92 E977.78
```

This is an unusually large correction. Before saving it, check for a slipping drive gear, restricted filament path, or partially blocked nozzle, then repeat the test. Calibration should not compensate for a mechanical fault.

## Connect to the FLSUN Delta from a macOS terminal

Because the printer has no on-device calibration interface, the commands above must be sent through a USB serial connection. First, list the available serial ports:

```sh
ls /dev/cu.*
```

After identifying the printer port, check whether another process is using it:

```sh
lsof /dev/cu.usbserial-2120 /dev/tty.usbserial-2120
```

Stop the process with `kill PID` if necessary, then open the connection:

```sh
picocom -b 115200 /dev/cu.usbserial-2120
```

If the printer repeatedly returns `Resend:1` and `ok`, reset the line number:

```gcode
N1 M110 N1*125
```

Test the connection with `M115`. A response containing `FIRMWARE_NAME:Robin` and `PrinterMode:FFF` confirms communication. Exit Picocom with `Ctrl+A`, then `Ctrl+X`.
