# Embedded — M5Stack, ESP32, microcontrollers

**Status:** unprobed.

- **Serial log (text, unattended):** `pio device monitor -b 115200` or `esptool`-based monitors,
  captured to `.evidence/<topic>/serial.txt` with a timestamp per line. This is the primary
  evidence: boot, the state change, errors.
- **Display:** no known way to dump an M5Stack screen buffer over serial yet (open question). Until
  probed, the image is a photo the owner takes, marked manual.
- **Flash and version:** record the firmware build hash or commit with the log.

The first M5Stack project that needs evidence probes the display question and writes the answer here.
