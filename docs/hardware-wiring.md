# Wiring: M5StickC PLUS2 + ENV Pro (BME688)

Every pin below was confirmed on real hardware (not just from third-party
docs/examples; see the caution note at the end).

## Chip

ESP32-PICO-V3-02, dual-core, up to 240MHz (this firmware runs it at 160MHz
to cut self-heating near the BME688). **2MB Quad PSRAM + 8MB Flash.**
**No AXP192** (unlike the original M5StickC Plus): no dedicated power
management chip; `power_hold` is done directly via GPIO4.

## I2C: two separate buses, don't mix them

| Bus | SDA | SCL | Devices (confirmed via `i2c: scan: true`) |
|---|---|---|---|
| Grove (external, ENV Pro) | GPIO32 | GPIO33 | 0x77 = BME688 |
| Internal (factory IMU + RTC) | GPIO21 | GPIO22 | 0x51 = BM8563 (RTC, unused), 0x68 = MPU6886 (IMU: board temperature and motion wake) |

The ENV Pro connects to the side Grove port (HY2.0-4P connector), **not**
the HAT pins on the bottom of the device.

**Mounting:** put the ENV Pro on a Grove cable (20 to 50 cm), beside or
below the StickC, never above it or inside the same closed case: otherwise
the board's heat reads as room temperature. See the README section
[Mounting and self-heating](../README.md#mounting-and-self-heating).

## Display: ST7789V2, 135×240

| Signal | Pin |
|---|---|
| CS | GPIO5 |
| DC | GPIO14 |
| RESET | GPIO12 |
| SPI CLK | GPIO13 |
| SPI MOSI | GPIO15 |
| Backlight (PWM/ledc) | GPIO27 |

**Confirmed hardware quirk**: without `offset_width: 52` /
`offset_height: 40` in the `dimensions:` block, the screen stays black/blank:
the controller has a RAM window bigger than the panel's physically
visible area. See `esphome/stickc-env-quarto.yaml` for the full block.
`invert_colors` was **not** needed with this rotation: tested both `true`
(white background) and `false` (dark background, correct).

## Buttons

| Button | Pin | Firmware use |
|---|---|---|
| A (M5 logo, right of the screen when lying down) | GPIO37 | Next page; on page 3, next ON/OFF option. Hold 5 s = restart |
| B (side, on top when lying down) | GPIO39 | Page 3: applies the option under the cursor. Hold 8 s = factory reset |
| C (PWR, bottom when lying down) | GPIO35 | Wake, handled by hardware: no ESPHome config |

Positions are for the landscape layout (USB-C on the right); see
[display-ui.md](display-ui.md).

GPIO34-39 on the ESP32 are **input-only** and have no internal pull-up
circuit. The M5Stack board already has external pull-ups on these lines:
use plain `mode: INPUT`, `INPUT_PULLUP` fails ESPHome's validation.

## Battery

ADC on GPIO38, no dedicated fuel gauge (no AXP192). Hardware voltage
divider = factor 2 (`multiply: 2.0` in the filter). Linear calibration
3.27V → 0%, 4.15V → 100%.

## USB power

No pin or chip reports it. The firmware infers it from the battery voltage
("Cabo USB", with hysteresis); see [lessons-learned.md](lessons-learned.md).

## Buzzer

GPIO2, driven as `ledc` PWM with `inverted: true` (frequency set per beep).
Used for the boot chime, button clicks and the bad-air-quality alert.

## LED / IR

GPIO19, shared between the onboard red LED and the IR transmitter. This
project drives the LED (blinks on bad air quality) and doesn't use IR.
Polarity (`inverted: false`) is not confirmed on hardware yet: if the LED
is lit when it should be off, flip `inverted`.

---

## Caution with third-party sources

During development, a community repository (`WarC0zes/m5stickc-plus2-esphome`)
listed the Grove port as GPIO0/GPIO26: **incorrect** for this hardware.
The real confirmation came from running `i2c: scan: true` on both buses and
reading the boot log, which showed the BME688 responding at 0x77 on the
GPIO32/33 bus. Always prefer testing on real hardware over trusting a
third-party example when they disagree.
