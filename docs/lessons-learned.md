# Lessons learned

## ESPHome compilation on Windows: PowerShell only

Running `esphome compile`/`upload`/`run` via Git Bash fails **silently**:
the command finishes, prints "Successfully compiled program", but never
actually produces a `.bin`/`.elf`, just a discreet "MSys/Mingw is no
longer supported" warning buried in the output. ESP-IDF 5.5 (used by
ESPHome since 2026.7) only works with a native Windows shell. Always use
PowerShell for these commands.

## M5Launcher doesn't work on this StickC Plus2

Trying to load the firmware via M5Launcher's WebUI (v2.7.2) hung
indefinitely on upload, with the storage card showing "Free 0B / Used 0B /
Total 0B", suggesting the internal FAT partition (~1MB, that M5Launcher
creates for apps on SD-less devices) never mounted/formatted on this
device. There's an open issue in that project's repo
(`bmorcelli/M5Stick-Launcher#95`) about OTA failures specifically on
StickCPlus2, a similar (not identical) symptom. Decision: abandon
M5Launcher, dedicate this device to this firmware, flash directly via
USB/OTA through ESPHome.

## Black/white screen without `offset_width`/`offset_height`

This StickC Plus2's ST7789V2 panel has a controller RAM window bigger than
the physically visible area. Without `offset_width: 52` /
`offset_height: 40` in the `display: dimensions:` block, content gets
drawn outside the visible area: the screen shows black or white
(depending on `invert_colors`), even though the firmware is running
perfectly (confirmed via serial log: sensor reading, LVGL initialized, no
errors). Found by cross-checking a community repo
(`WarC0zes/m5stickc-plus2-esphome`) that used the same values for a
different (non-LVGL) display use case.

## `align: CENTER` + `y:` in LVGL is not an absolute position

In ESPHome's LVGL, combining `align: CENTER` with an explicit `y:` turns
`y` into an **offset from the parent's center**, not an absolute
coordinate from the top. This completely broke the first layout attempt
(overlapping text, misplaced widgets). Fix: don't combine `align` with a
`y:` you meant as absolute. Either use plain `x`/`y` with no `align`, or
use `align` on purpose and treat `x`/`y` as offsets. The current layout
does the latter: each row is its own box, the label is `LEFT_MID` with
`x: 6`, the value `RIGHT_MID` with `x: -6`, and no `y:` at all.

## LVGL: a container's `text_color` doesn't reach labels in nested boxes

The 2026-10-09 landscape layout recolored the text by updating
`text_color` on one box per page, expecting the labels to inherit it. On
the device only the status bar changed color. LVGL's default theme gives
every `obj` a "card" style that sets its own `text_color`, so labels
inside the row boxes inherited the card color, not the page box's. The
status bar worked only because its labels sat directly in the updated box.
Fix: put `text_color` in a named style (`box_style`) used by every box and
recolor with a single `lvgl.style.update`, which also redraws every widget
using it. A named style outranks the theme's card style; a local color
(set on the widget itself) outranks the named style, so the boxes must not
set one.

Also from that change: `rotation` in the display config is refused when
LVGL drives the display; set it in the `lvgl:` block, and LVGL passes it
to drivers that rotate in hardware (`mipi_spi` does).

## `lv_color_hex()` in a lambda renders inverted

On this panel, colors set from a lambda with `lv_color_hex()` show up as
their complement: the green theme `0x57D97C` came out magenta. Colors
written directly in YAML (`text_color: 0x57D97C`) are fine. The firmware
works around it by storing pre-complemented values (`0xFFFFFF - color`)
for everything set from a lambda: green is stored as `0xA82683`, blue
`0x4FA8E8` as `0xB05717`, red `0xFF3B30` as `0x00C4CF`, amber `0xF2B84B`
as `0x0D47B4`, and `0xFFFFFF` in a lambda shows as black. This covers
`apply_theme_color`, the Wi-Fi dots and the page 3 pills. If you add a
lambda that sets a color, complement it the same way. If a future ESPHome
version stops inverting, every theme will look wrong at once: then drop
the complement.

## `on_time_sync` can fire with the wrong time at the very first sync

A `time: on_time_sync:` handler that decided day/night mode from the
current time applied night mode (5% brightness) at around 9:37pm local
time (should have been day mode). Suspicion: right at that exact moment,
just after the first NTP sync, `time.now()` may not yet reliably reflect
the applied timezone (a race between UTC and local time). Fix at the
time: removed the auto-detect on first sync, so the device always booted
lit (`restore_mode: ALWAYS_ON`) and only the fixed schedule (`on_time:` at
22h/7h) handled the transition once the system was already stable. Today
the device boots at the restored `Brilho` level (`Fraco`, 40%, by default)
and has no schedule at all.

The schedule itself was later removed too, for a different reason. Home
Assistant also began turning the screen off at bedtime, so two owners were
writing the same state on separate clocks, and the firmware kept relighting
a screen the house had deliberately turned off. The broader lesson: a
device's local clock is a poor home for a policy the hub already has an
opinion about. Keep the "when" on one side and let the firmware just obey
the entity.

## `gfonts://` in ESPHome doesn't include accents or every symbol

Fonts downloaded via `gfonts://` default to the `GF_Latin_Kernel` glyphset
(basic, no accents). For Portuguese, you need `glyphsets: [GF_Latin_Core]`
plus an explicit `glyphs:` list with the accented characters used.
Beyond that, some "decorative" symbols simply don't exist in the font at
all (e.g. "●" U+25CF doesn't exist in Montserrat, even with
`GF_Latin_Core`): in that case the fix isn't finding the right glyphset,
it's not depending on a font at all: use an `obj` widget (a drawn
circle/rectangle) instead of a character.

## Screen freezes after hours/days of uptime: fragmented heap, not a crash

Symptom: the screen freezes (buttons stop responding, last values stay
stuck), but sensors keep reporting to Home Assistant normally (wifi
connected, device doesn't disappear), and it only happens after hours/
days of uptime, never right after boot. That's the classic signature of
**heap fragmentation**, not a crash/reboot: since LVGL needs a contiguous
block for its draw buffer, a heap that's fragmented (but still has "spare"
memory in small pieces) makes LVGL's allocation silently fail and the
screen freezes, while wifi/API/sensors (which only need small allocations)
keep working fine.

Two contributors identified in the `interval: 10s` block (which runs
forever) and in the global config:

1. `logger: level: DEBUG`: generates continuous verbose logging, each
   line allocates a string. Over days of uptime, this fragments the heap
   little by little. Reduced to `INFO`.
2. `+`-concatenation in a lambda (`"precisão: " + id(...).state`) creates
   a temporary `std::string` every cycle, instead of a single allocation.
   Replaced with `str_sprintf(...)`.

## Self-heating: "it doesn't feel warm" is not a measurement

An early check ruled out self-heating because the device didn't feel warm
to the touch. That was wrong. Compared against the air conditioner's own
temperature sensor with the AC off, the BME688 read **about +4.9 °C high**
for ten days straight (10th to 90th percentile: +4.5 to +5.7 °C), and
relative humidity read about 26 points low, since humidity is computed
from the same (too warm) die temperature. A few degrees of warming is
invisible to a hand but enormous for a room sensor.

What made it worse here: the ENV Pro sitting close to the StickC, the
ESP32 at 240 MHz, Wi-Fi power saving disabled (`power_save_mode: none`,
the radio never sleeps; the error rose about 2 °C after that change),
and the screen backlight always at 100%. Moving the sensor alone cut the
error from +4.9 to +1.4 °C within half an hour, so physical placement
dominates.

Order of fixes: placement first (Grove cable, sensor beside or below the
board, never above it, away from sun, AC airflow and the USB charger),
then firmware power, and only then a `temperature_offset` in the
`bme68x_bsec2_i2c` block. The offset belongs in the BSEC block, not in a
sensor filter or a Home Assistant template: BSEC uses the corrected
temperature to compute humidity and IAQ too, so a downstream filter would
fix the temperature and leave humidity and IAQ wrong. Measure the offset
only once the other changes are stable, otherwise it just bakes the
current mistake in.

Where this build ended up (AC off, 30 min settle + 60 min, median of sensor
minus reference, one change at a time):

| Step | Sensor error | Board |
|---|---|---|
| Sensor in a case next to the board, 240 MHz, Wi-Fi `none` | +4.93 °C | not measured |
| Sensor out of the case, backlight 40%, 60 s averaging | +1.19 °C | 48.2 °C |
| + 160 MHz and Wi-Fi `light` | +1.25 °C | 38.2 °C |
| New case, ENV Pro about 2 cm from the StickC | +1.83 °C | ~42 °C |

The board dropped 10 °C at the third step but the error did not move, so
the sensor was already thermally decoupled from the board. The rest likely
comes from the BME688's own gas heater and its position relative to the
reference. The last row is why the firmware has `temperature_offset: 1.8`.
That value belongs to this mounting: measure your own.

## USB power on the PLUS2 can only be inferred from the battery voltage

The PLUS2 has no power chip (the original Plus had an AXP192) and no pin
that reports USB power, so "on the cable" comes from the battery ADC. On the
cable it read 4.15-4.26 V and stayed flat for hours (2026-10-03 to 10-13);
on battery it falls under load. `Cabo USB` uses hysteresis (on at >= 4.13 V,
off below 4.10 V) so it doesn't flap, and it reacts at the battery read pace
(5 min). The unplugged side still needs one measured test. Anything that
should only run on USB (like the 200 ms IMU polling for the motion wake)
hangs off this sensor, and `component.resume` with `update_interval`
changes a polling component's rate at runtime.

## Template `select` starts at `initial_option`, not at the restored value

The `Brilho` and `Tema` selects are `optimistic` templates whose real
state lives in `restore_value` globals. After a reboot the globals came
back, but the selects showed their `initial_option`, so Home Assistant said
`Forte` while the screen was at 40%. Fix: in `on_boot`, `select.set` each
select from its restored global.

## Binary sensor `on_state` skips the first state after boot

Since ESPHome 2025, a binary sensor's `on_state` does not fire for the
first state after boot (unknown to on/off) unless the sensor has
`trigger_on_initial_state: true`. `Cabo USB` showed "on" in Home
Assistant, but the actions hanging off it (bolt icon, 200 ms IMU polling)
never ran until this was added (2026-10-13). Checking the entity state in
Home Assistant is not enough; check the effect on the device.

## Self-heating check: compare against the AC state

With `temperature_offset: 1.8` applied (ENV Pro about 2 cm from the
StickC, in a case), the board still runs ~14 °C above the room, but the
remaining sensor error is +0.1 to +0.3 °C with the AC off. With the AC
cooling, the sensor reads 0.2 to 0.5 °C below the reference (the AC unit's
own temperature sensor), likely because the two sit in different parts of
the airflow. That gap is small and not a reason to change the offset. Only
compare windows with the AC in the same state for 30 min or more.

## GPIO 34-39 have no internal pull-up on the ESP32

Buttons A (GPIO37) and B (GPIO39) on the StickC Plus2 are in the 34-39
range, which on the ESP32 is input-only with no internal pull-up/pull-down
circuit: using `mode: INPUT_PULLUP` fails ESPHome's validation
("GPIO37 (34-39) does not support pullups"). The M5Stack board already has
external pull-ups on these lines; use plain `mode: INPUT` with
`inverted: true`.

## Two LEDC outputs on adjacent channels share one timer

ESPHome picks the LEDC timer as `channel / 2`, so auto-assigned channels 0
and 1 run on the same timer. The backlight (GPIO27) and the buzzer (GPIO2)
landed there: every `output.ledc.set_frequency` for a beep retuned the
backlight PWM as well, and its duty no longer matched the new bit depth
(1000 Hz = 16 bits, 2500 Hz = 14 bits, so 40% brightness is out of range).
On 2026-10-15 the 2500 Hz IAQ alert beeps (11:19 to 11:35) left the screen
dark while the light entity still read "on"; button A and motion wake did
nothing because both check `light.is_off`. Toggling the light in Home
Assistant brought it back. Fix: fixed `channel: 0` (backlight) and
`channel: 2` (buzzer), one timer each. Any time one LEDC output changes
frequency at runtime, give it a channel on its own timer.
