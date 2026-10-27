# Waveshare ESP32-C5-Touch-LCD-3.5 port

Status: **experimental / compile-tested; physical validation required**.

This profile (`AWOK_WAVESHARE_C5_35`) targets the Waveshare
ESP32-C5-Touch-LCD-3.5: an ESP32-C5 (dual-band 2.4/5 GHz Wi-Fi + BLE) with a
3.5-inch ST7796 320 × 480 panel and FT6336 capacitive touch — the same
display/touch combo as the Pancake C5 profile, so it reuses the native-320 × 480
ST7796 driver (`pancake_display.h`) and the capacitive-touch UI coordinate
mapping. Everything else on this board is different and is handled behind
`AWOK_WAVESHARE_C5_35`: a **CH32V006 I/O expander** owns panel power, the panel
and touch resets, and the backlight PWM (there is **no backlight GPIO**); the
FT6336, the expander, the AXP2101 PMU and the **GPS all share one I²C bus**; GPS
is an **LC76G over I²C** (not a UART); and the AXP2101 fuel gauge is read over
raw I²C.

Hardware sources (all pins and sequences below are taken from these, not
re-derived):

- [ESP32-C5-BSP](https://github.com/slickrick773/ESP32-C5-BSP) — the board's
  hardware-verified pin map, bring-up sequences, and the cross-cutting C5
  findings (`docs/BOARD-WS-C5-35.md`, `docs/HARDWARE-FINDINGS.md`).
- [GhostESP-Revival/GhostESP #433](https://github.com/GhostESP-Revival/GhostESP/pull/433)
  — a hardware-verified port of this exact board (CH32V006 expander register
  sequence, SPI/I²C pin map).
- [Official Waveshare schematic](https://github.com/waveshareteam/ESP32-C5-Touch-LCD-3.5)
  — pin map and the EXIO assignments.

## Pin map

| Function | GPIO | Notes |
| --- | ---: | --- |
| SPI SCLK | 6 | Shared by the ST7796 and the microSD |
| SPI MOSI | 7 | Shared bus |
| SPI MISO | 2 | Shared bus |
| ST7796 CS | 8 | Active low |
| ST7796 DC | 5 | Data/command |
| ST7796 RST | — | **Not a GPIO**: EXIO1 on the 0x24 expander |
| Backlight | — | **Not a GPIO**: PWM register on the 0x24 expander |
| microSD CS | 9 | On the shared display SPI bus |
| I²C SDA | 27 | Shared bus (touch + expander + PMU + GPS). **This is why there is no NeoPixel** — a status LED on GPIO27 would corrupt every I²C transaction |
| I²C SCL | 26 | Shared bus, 100 kHz |
| FT6336 touch | I²C 0x38 | Reset on EXIO0; **reads are INT-gated on GPIO3** |
| FT6336 INT | 3 | TP_INT, external 10K pull-up; low = finger down |
| CH32V006 expander | I²C 0x24 | Panel power / resets / backlight PWM |
| AXP2101 PMU | I²C 0x34 | Battery fuel gauge (raw I²C) |
| GPS (LC76G) | I²C 0x50 / 0x54 / 0x58 | On the **same** shared bus — **not a UART** |

**Never put a UART on GPIO13/14** — they are the native USB D-/D+ lines; a UART
there drops the board off USB and breaks auto-download. The J3 header UART is not
bonded to the chip as shipped, which is why GPS is on I²C. `kGpsRx`/`kGpsTx` are
`-1` to keep every UART off those pins.

## Build and flash

```bash
python3 scripts/build_firmware.py waveshare-c5-35
```

FQBN: `esp32:esp32:esp32c5:FlashSize=8M,PartitionScheme=default_8MB,PSRAM=enabled,CDCOnBoot=cdc`.
The board has 32 MB flash / 8 MB PSRAM, but the C5 Arduino FQBN tops out at 16 MB
flash, so the Pancake 8 MB layout is used (16 MB is an easy later bump).
**`CDCOnBoot=cdc` is required**: console and flashing are over the native
USB-Serial-JTAG at 115200 with no UART bridge, and without
`ARDUINO_USB_CDC_ON_BOOT=1` the sketch's own `Serial.print()` goes to
UART0/GPIO11 and is invisible on the USB port (the board still runs).

Flash the merged image at `0x0`. **Set the flash mode explicitly** — write with
`--flash-mode dio --flash-freq 80m`. Flashing the bootloader with
`--flash-mode keep` leaves a bad flash-mode byte in the image header; the ROM
rejects it (`invalid header`) and the board boot-loops to a **dark screen** that
looks exactly like a display bug. **If a flash "won't boot," read the ROM/console
log first.** If the board does not auto-enter download mode, **hold BOOT, tap
RST, release BOOT**, then upload.

## Display, touch, GPS and power choices

- **Display.** AxD's `AwokST7796` (an `Adafruit_SPITFT` subclass) drives the
  panel natively at 320 × 480 with colour inversion on (`0x21`), the same driver
  the Pancake profile uses. The push clock is 80 MHz (rated 60; holds on the
  units the BSP tested). Reset is `-1` because the expander pulses it during
  bring-up. If a unit shows SPI artifacts, drop the clock in
  `initializeDisplayAndTouch()` toward 40 MHz; if the image is mirrored/rotated,
  adjust `AwokST7796::kMadctlPortrait`.
- **Expander bring-up order.** The CH32V006 must come up on the I²C bus before
  the ST7796 and FT6336: it powers the rail with both resets released, then
  pulses the LCD reset and the touch reset (the FT6336 needs its rail up first or
  it stays unresponsive), then sets the backlight PWM. This runs once in
  `initializeDisplayAndTouch()`.
- **Touch.** The FT6336 read is **INT-gated** on GPIO3 (`G_MODE = 0` makes INT a
  stable level): with no finger down the I²C read is skipped entirely, which
  kills the phantom/offset taps a blind poll produces on this board. Each read
  re-opens the shared bus once (the C5 read quirk) and retries once. Coordinates
  are raw ST7796 panel units; `readTouch()` maps them into the 240 × 320 design
  grid.
- **GPS.** The LC76G is read over I²C using Waveshare's `LC76G_RPI_I2C` protocol,
  one atomic multi-frame cycle per `updateGps()` throttled to ~2 Hz, gated on
  touch-idle (≥ 400 ms since the last activity) so a stroke is never interrupted,
  with a 300 ms bus timeout saved/restored around the cycle and a wake nudge
  after repeated empty reads. Each NMEA byte is fed to the existing
  `TinyGPSPlus` parser, so the GPS, wardrive and clock paths are otherwise
  unchanged.
- **Battery.** The AXP2101 is read over raw I²C (no XPowersLib, so AxD's CI
  libraries are unchanged): chip id `0x03 = 0x4A`, gauge enable `0x18` bit 3,
  cell-present `0x00` bit 3, percent `0xA4`. The Status screen shows the percent
  when a cell is fitted on the MX1.25 header, otherwise "USB".
- **Single core.** The C5 is single-core, and every I²C owner (touch, GPS,
  battery) runs from `loop()`, so bus access is serialised without a mutex. Keep
  blocking work off the loop so touch stays responsive.

## Physical validation checklist

Do not mark any capability "supported" until it is observed on a real board.

- Boot artwork fills the 320 × 480 panel with correct colours; backlight dimming
  in Settings works (it drives the expander PWM, not a GPIO).
- Settings → Screen test: capacitive touch tracks with no phantom taps at boot
  (INT gate), and all regions register.
- Wi-Fi scans both 2.4 and 5 GHz; BLE starts and stops without a controller
  allocation failure.
- The microSD mounts, creates `/awokxdag`, writes a capture, and reads it back.
- GPS: a boot I²C scan shows `0x50`; the LC76G gets a fix and sets system UTC
  while touch stays smooth.
- AXP2101: the Status screen shows a rising percent while charging with a cell
  fitted, and "USB" with no cell.

If touch is mirrored or offset, capture the raw values from the screen test
before changing anything; if the panel is dark after a flash, read the console
for `invalid header` before suspecting the display.

## Future work / follow-up

- **Parlio for display pushes — likely N/A here, flagged for later.** A suggestion
  from the GhostESP community (hardware partner "The Big BG" / wiredhatters) was
  that the C5's parlio (parallel / i80-style) peripheral could smooth full-screen
  redraws. On *this* board the ST7796 is wired as **SPI** (SCLK 6 / MOSI 7 /
  MISO 2 / CS 8 / DC 5), so a parallel transport does not apply without different
  wiring — parlio is not a lever for this panel. The redraw-smoothness levers
  that do apply are the **SPI clock** (the panel holds ≤ 80 MHz), **DMA** on the
  SPI writes, and **partial / dirty-rect redraws** instead of blitting the whole
  320 × 480 buffer every frame. Captured as a down-the-line experiment, not a
  current need.
