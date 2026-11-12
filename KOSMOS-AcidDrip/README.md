# KOSMOS-AcidDrip

KOSMOS v1.3.4 ported to the **Acid Drip V2** main board and to the **V5
modification** (RP2040 / Raspberry Pi Pico, ILI9341 320x240, 16 Cherry MX
pads). On the V5 mod the LCD RST line is tied to 3V3 and GP8 is the sync jack,
so this port leaves GP8 alone; see `KOSMOS_LCD_RST_PIN` below.

This variant replaces the original I2S DAC output with **mono PWM audio on
GP15**, and the Waveshare 240x240 ST7789 driver with an **ILI9341 driver that
centres the 240x240 UI in the 320x240 panel**.

Compiled and linked with arduino-pico 6.0.0 and tested on real Acid Drip V2 and
V5 hardware (display orientation, audio output). Audio gain and PWM noise may
still need adjustment per board (see below).

---

## Hardware mapping

| Function | Acid Drip pin | Notes |
|---|---|---|
| TFT SCK | GP6 | SPI0 |
| TFT MOSI | GP7 | SPI0 |
| TFT RST | GP8 (V2) / 3V3 (V5 mod) | on the V5 mod GP8 is sync and is **not** driven; see below |
| TFT DC | GP10 | |
| TFT CS | GP13 | |
| TFT backlight | — | hard-wired on |
| Audio (mono) | GP15 | V2: 100R + 10nF -> `level` pot -> jack. V5: 10K/3.3K divider + 10nF -> 10µF -> jack |
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

> **Important (V2):** the `level` pot crossfades the GP15 (acid) path against
> the GP2 (drums) path. KOSMOS only drives GP15, so turn the pot toward the
> **ACID** side or you will hear silence. On the V5 mod GP15 feeds the mixing
> node directly and the pot only trims the GP2 side, so this does not apply.

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
arduino-cli config add board_manager.additional_urls \
  https://github.com/earlephilhower/arduino-pico/releases/download/6.0.0/package_rp2040_index.json
arduino-cli core update-index
arduino-cli core install rp2040:rp2040@6.0.0
arduino-cli compile \
  --fqbn rp2040:rp2040:rpipico:usbstack=tinyusb,freq=150 \
  KOSMOS-AcidDrip
```

> The Pico core is **not** in the official Arduino package index, it comes from
> Earle Philhower's third-party index — without that `additional_urls` entry
> the build fails with `Platform 'rp2040:rp2040' not found`. The core bundles
> Adafruit TinyUSB, so no extra library is needed for USB MIDI.

Prebuilt images are produced by the **Build** workflow
(`.github/workflows/build-arduino.yml`): every push that touches this folder
compiles the sketch with exactly the FQBN above and uploads the resulting
`KOSMOS-AcidDrip.uf2` as a workflow artifact
(*Actions → Build → Artifacts → KOSMOS-AcidDrip-uf2*).

To flash: hold BOOTSEL, plug the Pico's USB port in, drag the UF2 onto the
`RPI-RP2` drive.

### LCD reset (V2 vs V5 mod)

The port does **not** drive GP8 by default (`KOSMOS_LCD_RST_PIN = -1`). On the
V5 modification the panel RST is tied to 3V3 and GP8 is the sync jack, so a
hardware reset pulse there would go out of the sync connector on every boot and
could fight an external clock source. The panel is reset with `SWRESET (0x01)`
instead, which is enough because its RST pin is already released.

On a stock V2 board (panel RST on GP8) build with:

```
arduino-cli compile \
  --fqbn rp2040:rp2040:rpipico:usbstack=tinyusb,freq=150 \
  --build-property "compiler.cpp.extra_flags=-DKOSMOS_LCD_RST_PIN=8" \
  KOSMOS-AcidDrip
```

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
320x240, and `lcdSetWindow()` rotates the UI into the panel's address space
and applies the centring margin; the full 320x240 area is cleared to black
during init.

```cpp
#define KOSMOS_LCD_MADCTL 0x20
#define LCD_PANEL_W 320
#define LCD_PANEL_H 240
#define LCD_X_OFFSET 0    // RAM X -> the 240 px tall axis
#define LCD_Y_OFFSET 40   // RAM Y -> the 320 px wide axis
```

`0x20` is landscape with RGB order (no BGR bit). The ILI9341 has no column/row
start offset, so centring the 240 px UI in the 320 px width needs a 40 px margin
on each side: `(320 - 240) / 2 = 40`. The 320x240 panel is cleared fully to
black on init to avoid side noise.

**The 90° rotation is required, and the margin must go on the other axis.**
With MADCTL's MV bit set the panel's RAM axes are transposed relative to the
screen: the 320 px wide axis is addressed by the row address (RAM Y) and the
240 px tall axis by the column address (RAM X). A 240x240 UI therefore has to
be written rotated 90° clockwise — logical `(x, y)` → RAM `(239 - y, x)` — to
come out upright, and the 40 px margin belongs to RAM Y, the axis that ends up
horizontal.

Getting either half wrong is visible immediately:

* margin on RAM X (the tall axis) → the UI sits against one edge / is clipped
  and reads as "not centred";
* no rotation at all → the picture comes out sideways.

The original Acid Drip firmware draws the full 320x240 panel in one go, so it
never needs either of these; this port has to place a smaller square inside
that space.

### Keeping the black margins clean

The two 40 px margins are the only pixels the UI never redraws — the 240x240
area is repainted continuously, the margins are not. Anything that is not
written correctly at boot therefore stays on screen for as long as the synth
is powered, which is exactly how the speckles in the margin appeared. Four
things guard against that:

* the boot clear uses the panel's real axis order, CASET 0..239 / PASET
  0..319, which covers every cell of the RAM in one window (clearing with the
  opposite order points the column counter past the end of the RAM);
* both address commands get **four** parameter bytes — start high/low and end
  high/low. Sending only the end address leaves the parameter fetch half
  finished: the following bytes are swallowed as parameters, and the window
  that is actually cleared is not the one that was asked for;
* the panel is cleared a second time after the splash screen, when the USB
  MIDI stack has settled and the boot time interrupt storm is over;
* both margins are repainted black in 10 row slices, one slice every 200 ms.
  Painting black over black does not flicker, and it wipes out any speckle a
  single flipped SPI bit would otherwise leave behind. The slices keep the
  panel clean without blocking the synth for more than ~3 ms at a time, which
  would otherwise show up as MIDI timing jitter.

For the same reason the LCD SPI clock is 12 MHz instead of the 24 MHz maximum:
the Acid Drip V2 panel hangs off long jumper wires, and a flipped bit in a
one-shot write is permanent.

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
* ILI9341 init using landscape MADCTL `0x20` (RGB), a full 320x240 clear in the
  panel's real axis order with four parameter bytes per address command, a
  90° clockwise rotation of the 240x240 UI plus a 40 px margin on the axis that
  ends up horizontal, a 12 MHz SPI clock and a periodic repaint of the black
  margins, instead of the ST7789 240x240 display.
* A button moved from GP15 to GP16 (GP15 is audio on Acid Drip).
* LCD reset: GP8 is only driven when `KOSMOS_LCD_RST_PIN` is set (default is
  `-1`, for the V5 mod where RST is tied to 3V3 and GP8 is sync).
* Joystick disabled; GP22 no longer used as DAC mute.
* Splash delay reduced from 10 s to 1.5 s.
* `lcdFillRect()` now clips to 240x240 (upstream could send windows past the
  panel edge).

## TODO

* Map the three parameter pots (GP26 CUT, GP28 RES, GP27 DECAY) to filter /
  resonance / decay CCs.
* Use the remaining 12 pads for Euclidean editing / pattern selection.
