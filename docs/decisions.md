# Technical decisions

## Real IAQ via BSEC2, not raw `bme680`

**Alternatives considered:** (a) native `bme680`, raw `gas_resistance` in
ohms only, no computed index; (b) middle ground, `gas_resistance` + a
custom heuristic; (c) `bme68x_bsec2`, real 0-500 index via Bosch's BSEC2
algorithm.

**Chosen:** (c). Home Assistant automations that react to air quality
expect numeric thresholds on a 0-500 scale: switching to raw
`gas_resistance` would break that logic. The cost (Bosch's BSEC2 license,
implicitly accepted by using the component, plus ~30-60min of baseline
recalibration on every reflash) was accepted knowingly. It ended up
simpler than expected: the component is native to ESPHome, no
`external_components` or external library source needed.

## LVGL instead of raw display + lambda drawing

The intended UI (colored readouts, multiple sensor rows, a small on-screen
menu) was rich enough that drawing everything via `it.printf`/`it.image`
in a lambda would have been more work and less flexible than using the
LVGL framework (widgets, declarative updates). The initial worry about
blowing the RAM budget didn't pan out: the PLUS2 has 2MB of PSRAM, and a
135×240×16bpp framebuffer is only ~64KB.

## M5Launcher abandoned: dedicated device

Original idea: run the environmental firmware *through* M5Launcher (a
multi-app launcher already used on other sticks), so this wouldn't need to
be a fully dedicated device. Tested live: M5Launcher's WebUI hangs on
local upload ("Total 0B" of storage). Decision: not worth debugging a
third-party launcher: this StickC becomes dedicated to this firmware
only, flashed directly via USB/OTA. Simplifies the project (no launcher
partition sizing to worry about, no external storage dependency).

## 4 color themes instead of day/night automation

**Motivation:** full brightness at night in a bedroom is disruptive, and a
single fixed color palette felt limiting.

**Decision:** a `select` with 4 color themes ("Verde"/"Azul"/"Vermelho"/
"Âmbar") and a `select` with two brightness levels (**"Fraco" 40% /
"Forte" 100%**, default "Fraco" because 100% measurably heats the BME688):
on the physical screen, finer gradations weren't perceptible enough to be
worth it. Until 2026-10-09 both could also be changed from an on-screen
menu (physical buttons A/B); since the landscape layout they are set from
Home Assistant only, and the buttons are left for paging and the alert
settings.

Night dimming was originally a schedule inside the firmware (fixed 22h/7h,
not sunrise/sunset detection): drop to 5% at night, restore the manual
level in the morning, independent of the selected color theme. That was
**removed**. Once Home Assistant also started turning the screen off as
part of its sleep mode, two owners were writing the same state on
different clocks, and the firmware's schedule relit a screen that the
house had deliberately turned off. The firmware now runs no schedule at
all and simply obeys whatever the `light` entity is told; the "when" lives
entirely in Home Assistant, where the rest of the sleep routine already
lives. (A `time: homeassistant` clock came back on 2026-10-08, used only
for the "Último Boot" timestamp; nothing is scheduled on it.)

## Battery: direct ADC reading, no fuel gauge

The PLUS2 has no AXP192 (the power-management chip with a fuel gauge that
the original Plus had). Battery is voltage-only via ADC (GPIO38) + linear
calibration (3.27V-4.15V → 0-100%): less accurate than a real fuel gauge,
but it's what the hardware allows without an extra chip.

## Sound and LED alerts, each independently toggleable from Home Assistant

**Motivation:** a single mute switch isn't granular enough: sometimes
the beeps (boot chime, button clicks, bad-air-quality alert) are the
annoyance, sometimes the blinking LED is, and sometimes both are wanted.

**Decision:** three independent switches, all exposed as HA entities:
a general sound mute (gates every beep centrally, inside the shared `beep`
script), a bad-air-quality beep toggle (only gates the IAQ alert sound,
still respects the general mute), and a bad-air-quality LED toggle (the
onboard red LED, GPIO19, blinks at 500ms while IAQ stays "Ruim").

The two bad-air-quality toggles (beep and LED) can also be flipped on
screen page 3; the general mute ("Som Geral") stays in Home Assistant only.

## Landscape, three pages, buttons only for paging

Lying down with the USB-C on the right, the 240×135 screen fits the three
most-read values (temperature, humidity, air quality) at 32 px, legible
from across the room. Less-read data (pressure, gas resistance, IAQ number,
IAQ accuracy) went to page 2 and the alert toggles to page 3. Both return
to page 1 on their own (30 s and 15 s), so the device always rests on the
main readings. Button A pages through the screen; button B only acts on
page 3. Brightness and color theme left the device: they are set from Home
Assistant.

## Self-heating: placement first, offset last

The compensation is `temperature_offset` in the `bme68x_bsec2_i2c` block,
not a sensor filter or a Home Assistant template, because BSEC2 derives
humidity and IAQ from the corrected temperature: one setting fixes all
three. It is measured only after the mounting, CPU (160 MHz), Wi-Fi
(`power_save_mode: light`) and screen brightness (40%) are final, so it
does not bake in an error that could be fixed at the source. It must be
re-measured if any of those change. See
[lessons-learned.md](lessons-learned.md) for the measurements.

## Restart alert on "Último Boot", not on the reset reason

"Motivo do Reset" is re-sent every time Home Assistant restarts, so an
automation triggered by it fires falsely. The `uptime` sensor published as
a timestamp ("Último Boot") only changes when the device really restarts,
so it is the trigger for the unexpected-restart alert.

## USB power inferred from the battery voltage

The PLUS2 has no pin or power chip that reports USB power, so "Cabo USB" is
derived from the battery ADC with hysteresis (on at 4.13 V or more, off
below 4.10 V, checked every 5 min). Features that cost power or heat, like
polling the IMU every 200 ms for the motion wake, only run while it is on;
on battery the IMU drops back to one read per minute.

## Fewer, averaged updates to Home Assistant

BSEC2 in LP mode outputs every 3 s. A `throttle_average` filter (60 s for
temperature, humidity and IAQ; 5 min for pressure and gas resistance) and
5-minute battery and Wi-Fi reads cut the traffic from about 110 to about 5
updates per minute and smooth the readings. The screen and the IAQ alert
read the averaged values. The heap and loop-time diagnostics were removed
once the reboot investigation ended; only the reset reason stays.
