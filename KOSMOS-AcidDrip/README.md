# KOSMOS-AcidDrip

KOSMOS v1.3.4 ported to the **Acid Drip V2** main board (RP2040 / Raspberry Pi
Pico, ILI9341 320x240, 16 Cherry MX pads).

This variant replaces the original I2S DAC output with **mono PWM audio on
GP15**, and the Waveshare 240x240 ST7789 driver with an **ILI9341 driver that
centres the 240x240 UI in the 320x240 panel**.

Compiled and linked with arduino-pico 6.0.0 and being tested on real Acid Drip
V2 hardware. Display orientation is matched to the original firmware; audio
gain and PWM noise may still need adjustment (see below).

---

## Hardware mapping

| Function | Acid Drip V2 pin | Notes |
|---|---|---|
| TFT SCK | GP6 | SPI0 |
| TFT MOSI | GP7 | SPI0 |
| TFT RST | GP8 | |
| TFT DC | GP10 | |
| TFT CS | GP13 | |
| TFT backlight | — | hard-wired on |
| Audio (mono) | GP15 | 100R + 10nF -> `level` pot -> mono TS jack |
| A button | GP16 | pad 4 |
| B button | GP17 | pad 5 |
| X button | GP19 | pad 6 |
| Y button | GP21 | pad 8 |
| Pots | GP26 / GP28 / GP27 | not used yet (CUT / RES / DECAY) |
| GP2 | — | left for the original sync/drums circuit |
| GP22 | — | pad 16, no longer driven as DAC mute |

Pad numbering follows the Acid Drip README: top row pads 1-8 are
`GP14, GP12, GP11, GP16, GP17, GP19, GP20, GP21`; bottom row pads 9-16 are
`GP0, GP1, GP3, GP4, GP5, GP9, GP18, GP22`.

> **Important:** the V2 `level` pot crossfades the GP15 (acid) path against the
> GP2 (drums) path. KOSMOS only drives GP15, so turn the pot toward the
> **ACID** side or you will hear silence.

---

## Build

Arduino IDE 2.x with the **Raspberry Pi Pico/RP2040** core (arduino-pico,
tested with 6.0.0):

| Setting | Value |
|---|---|
| Board | Raspberry Pi Pico |
| USB Stack | **Adafruit TinyUSB** (required for USB MIDI) |
| CPU Speed | **150 MHz** (recommended for PWM audio) |
| Flash Size | 2 MB (no FS) |
| Optimize | Smallest (-Os) default is fine |

The `pra32-u-*.h` headers are vendored into this folder so the Arduino builder
can compile the sketch on its own. They are copies of `../src/pra32-u-*.h` —
when updating PRA32-U, copy the headers again.

arduino-cli equivalent:

```
arduino-cli compile \
  --fqbn rp2040:rp2040:rpipico:usbstack=tinyusb,freq=150 \
  KOSMOS-AcidDrip
```

`bin/KOSMOS-AcidDrip.uf2` is a prebuilt image with those settings.

To flash: hold BOOTSEL, plug the Pico's USB port in, drag the UF2 onto the
`RPI-RP2` drive.

---

## Audio

* Mono, 48 kHz, 16-bit PWM. The synth's left channel is used by default
  (`KOSMOS_MONO_SUM_LR 0`). Set it to `1` to average L+R instead; the stereo
  chorus/delay can comb-filter when summed, so listen before choosing.
* Output trim is `7/10` (`KOSMOS_MASTER_GAIN_NUM` / `_DEN`). Increase it if the
  result is too quiet, decrease it if the output clips.
* PRA32-U runs its PWM error-diffusion branch, which is tuned for a ~3125-step
  PWM carrier. That is exactly what 150 MHz / 48 kHz gives. Other CPU speeds
  still work, but the dither amplitude is slightly off; the build prints a
  warning when `F_CPU` is not 150 MHz.
* GP2 (drums/sync) is untouched. The sync SPDT switch can stay wherever it is.
* The board's 100R + 10nF RC is an EMI filter, not a reconstruction filter. If
  you can hear PWM carrier hiss, try a larger capacitor (e.g. 22-47 nF) or a
  proper RC low-pass after GP15, and/or lower `KOSMOS_MASTER_GAIN`.
* Upstream PRA32-U documents a known PWM issue: a small signal discontinuity
  roughly every 60-80 ms, most audible on high notes and sine-like patches.
  It is a limitation of this PWM path, not of this port.

---

## Display

The UI keeps its native 240x240 coordinate system. The panel is addressed as
320x240, and `lcdSetWindow()` applies the X offset; the full 320x240 area is
cleared to black during init.

This port uses the Acid Drip-style landscape address space (320x240) and applies a
software 90° clockwise rotation to the logical UI in `lcdSetWindow()`:

```cpp
#define KOSMOS_LCD_MADCTL 0x20
#define LCD_PANEL_W 320
#define LCD_PANEL_H 240
#define LCD_X_OFFSET 40
#define LCD_Y_OFFSET 0
#define KOSMOS_LCD_ROTATE_CONTENT_90 1
```

`0x20` is landscape with RGB order (no BGR bit). The ILI9341 has no column/row
start offset, so centring the 240 px UI in the 320 px width needs a 40 px margin
on each side: `LCD_X_OFFSET = (320 - 240) / 2 = 40`. The 320x240 panel is cleared
fully to black on init to avoid side noise.

---

## Controls

| Button | Action |
|---|---|
| A (pad 4) | next program for synth A |
| B (pad 5) | next program for synth B |
| A + B | MIDI Start / Stop |
| X (pad 6) | cycle mute: A / B / ALL / A un-mute / B un-mute |
| Y (pad 8) | next BPM (short press) / repeat (hold) |

USB MIDI and the TouchOSC CC map are unchanged (CC20 density, CC22 speed,
CC81 scale, CC82 B volume, CC83 transpose, CC100 program, CC106 save).

The joystick code is compiled out (`KOSMOS_HAS_JOYSTICK 0`) because GP6/GP7 are
the display's SPI pins. `readJoystick()` is a no-op.

---

## Differences from upstream KOSMOS v1.3.4

* PWM mono audio on GP15 instead of I2S on GP26/27/28.
* ILI9341 init using landscape MADCTL `0x20` (RGB), full 320x240 clearing, a
  40 px X offset to centre the 240x240 UI, and a 90° clockwise software rotation
  instead of the ST7789 240x240 display.
* A button moved from GP15 to GP16 (GP15 is audio on Acid Drip).
* Joystick disabled; GP22 no longer used as DAC mute.
* Splash delay reduced from 10 s to 1.5 s.
* `lcdFillRect()` now clips to 240x240 (upstream could send windows past the
  panel edge).

## TODO

* Map the three parameter pots (GP26 CUT, GP28 RES, GP27 DECAY) to filter /
  resonance / decay CCs.
* Use the remaining 12 pads for Euclidean editing / pattern selection.
* Verify panel orientation and PWM noise on hardware.
