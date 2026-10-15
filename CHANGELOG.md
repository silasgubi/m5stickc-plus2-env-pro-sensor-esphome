# Changelog

## 2.0.1 (2026-10-15)

- **Fix: screen dark after an IAQ alert beep**: the backlight and the
  buzzer were on auto-assigned LEDC channels 0 and 1, which share one
  timer, so every beep retuned the backlight PWM. The 2500 Hz IAQ alert
  left the screen dark while `Tela Brilho` still read "on", and neither
  the M5 button nor motion woke it (only toggling the light in Home
  Assistant did). The backlight is now on `channel: 0` and the buzzer on
  `channel: 2`, one timer each.

## 2.0.0 (2026-10-14)

Breaking UI change: the screen is now landscape with three pages, and the
on-screen brightness/theme menu is gone. Set brightness and theme from Home
Assistant (`Brilho` and `Tema` selects). New entities: `Temperatura Placa`,
`Último Boot`, `Cabo USB` and `Acordar com Movimento`.

- **Self-heating offset as a substitution**: `temperature_offset` now sits
  in `substitutions:` next to `motion_threshold`. The 1.8 °C value fits
  this mounting only; on another build start at 0 and measure.
- **Fix: Cabo USB actions at boot** (2026-10-13): the bolt never showed
  and the IMU stayed at 60 s (so the motion wake did not really run) until
  the first power change, because `on_state` skips the first state after
  boot. Added `trigger_on_initial_state: true`.
- **USB bolt on the status bar** (2026-10-13): a lightning bolt between
  the Wi-Fi dots and the battery % while `Cabo USB` is on, in the theme
  color. Wi-Fi dots moved 12 px left.
- **Motion wake** (2026-10-13): with the screen off, moving the device
  wakes it like an M5 press (MPU6886 accelerometer every 200 ms, threshold
  1.0 m/s² against a ~2 s moving reference). New switch
  `Acordar com Movimento`. The board temperature is now averaged to one
  value per minute, since the IMU is read more often.
- **USB power detection** (2026-10-13): new `Cabo USB` binary sensor,
  inferred from the battery voltage (the PLUS2 has no pin for it): on at
  >= 4.13 V, off below 4.10 V. On the cable the ADC read 4.15-4.26 V, flat
  for hours, over ten days. Motion wake and the 200 ms IMU polling only
  run on USB; on battery the IMU drops to 60 s. The unplugged side still
  needs one measured test.
- **Wake on M5** (2026-10-09): with the screen off (sleep mode), the M5
  button wakes it on page 1; it turns off again 30 s after the last press,
  with click beeps silent meanwhile.
- **Self-heating compensation** (2026-10-09): `temperature_offset: 1.8` in
  the BSEC2 block (corrects temperature, humidity and IAQ together).
  Measured in the new LEGO case, ENV Pro ~2 cm from the StickC: median
  error +1.83 °C against the reference (p10 +1.78, p90 +1.84, AC off).
  Without the case it was +1.25 °C on three separate windows.
- **Landscape screen** (2026-10-09): the device now lies down with the
  USB-C on the right. Three pages (readings; pressure/gas/IAQ/accuracy;
  sound and light alert settings) plus a status bar on the LVGL top layer
  with Wi-Fi dots and battery. M5 button pages through and walks the
  settings cursor, the side button applies. Pages 2 and 3 return to page 1
  on their own. IAQ accuracy is back on screen (page 2, in Portuguese).
  Brightness and theme moved to Home Assistant only. Every text and shape
  follows the theme color; red theme changed to 0xFF3B30 (0xE84C4C looked
  wine on the panel). Layout in `docs/display-ui.md`.
- **Network diet** (2026-10-08): removed the `Heap Livre`,
  `Heap Maior Bloco`, `PSRAM Livre`, `Loop Time` and `Uptime` (seconds)
  sensors, unused since the reboot investigation ended; battery, WiFi
  signal, pressure and gas resistance now every 5 min. From about 110
  state updates per minute to about 5.
- **Self-heating work, step 2a** (2026-10-08): WiFi `power_save_mode` back
  to `light` and `cpu_frequency` 240 → 160 MHz. Before (AC off): reading
  +1.19 °C above the reference, board (MPU6886) 48.2 °C. After: board
  38.2 °C, reading error unchanged (+1.25 °C), so the ENV Pro is already
  thermally decoupled from the board; the remaining offset is for
  `temperature_offset`.
- **Self-heating work, steps 2b/2c** (2026-10-08): `throttle_average: 60s`
  on all five BME688 sensors (BSEC2 LP published every 3 s); new
  diagnostic sensors `Temperatura Placa` (board temperature from the
  MPU6886 on the internal I2C bus) and `Último Boot` (uptime as a
  timestamp, the trigger for an unexpected-restart alert). Adds a
  `time: homeassistant` clock used only by that timestamp; no schedules
  run on it. Pressure and gas resistance later moved to 5 min averages
  (see Network diet).
- Fixed: after a reboot the `Brilho` and `Tema` selects showed their
  `initial_option` instead of the restored values (Home Assistant said
  "Forte" while the screen was at 40%). `on_boot` now syncs them. Default
  brightness is now `Fraco`: 100% adds about 1 °C to the BME688 reading.

## 1.1.0 (2026-09-27)

- **Diagnostics for the recurring reboots/freezes** (2026-09-27): added
  the `debug:` component plus `Motivo do Reset` (reset reason),
  `Heap Livre`, `Heap Maior Bloco`, `PSRAM Livre`, `Loop Time` and an
  `Uptime` sensor, all as diagnostic entities. The reset reason tells
  watchdog, brownout and panic apart; the heap numbers test the memory
  fragmentation theory directly. *Superseded in 2.0.0: only
  `Motivo do Reset` remains; the heap, PSRAM, loop time and uptime sensors
  were removed (`Último Boot` replaces the uptime sensor).*
- BSEC2 `state_save_interval` lowered from 6h to 1h, so a reboot no longer
  throws away up to 6h of IAQ calibration.
- WiFi `power_save_mode: none` (device lives on the charger).
  *Superseded in 2.0.0: back to `light`, since `none` keeps the radio awake
  and raised the temperature error.*
- Built with ESPHome 2026.9.0 (was 2026.7.x). This is also the first build
  to carry the removal of the night brightness schedule below.
- **Removed the built-in night brightness schedule** (the whole `time:`
  block: sntp platform plus the `on_time:` handlers at 22h and 7h). The
  firmware no longer decides when the screen dims; Home Assistant owns
  the screen state through its sleep mode, turning `light.…_tela_brilho`
  off when the house goes to sleep and back on at the manually selected
  level when it wakes. Reason: the two schedules fought each other. With
  the firmware still on its own clock, the screen relit itself at 22h
  (5%) and again at 7h (full manual brightness) regardless of whether
  the house was still asleep. Nothing else referenced the `time:`
  component, so removal is self-contained. Note that `on_boot` still
  calls `apply_manual_brightness` and the light is `restore_mode:
  ALWAYS_ON`, so a reboot still brings the screen back lit.
- Screen freeze after hours/days of uptime: mitigated (`logger` from
  `DEBUG` to `INFO`, removed `std::string` `+`-concatenation in the IAQ
  accuracy lambda in favor of `str_sprintf`): suspected heap
  fragmentation. See [docs/lessons-learned.md](docs/lessons-learned.md).
- Manual brightness simplified from 5 levels (20/40/60/80/100%) to 2
  (**Fraco 40% / Forte 100%**): on the physical screen only two levels
  were actually perceptible.
- Sensor readout font enlarged (11 → 16pt) and columns realigned so
  values are legible at arm's length; labels abbreviated (Temp/Umid/
  Pres/Gás/Ar/Bat) to fit the 135px screen width. *Superseded by the
  landscape screen in 2.0.0.*
- Wifi status changed from a single connect/disconnect dot to 4
  signal-strength dots (like phone signal bars), lit in the current
  theme color based on RSSI.
- Added a short on-screen button legend (`<-PWR M5 Menu ->OK`) using the
  device's real physical button labels instead of firmware letters
  (A/B/C). *Superseded by the landscape screen in 2.0.0.*
- Added 3 independent switches, exposed to Home Assistant: general sound
  mute, bad-air-quality beep toggle, bad-air-quality LED toggle.
- Added blinking red LED (onboard, GPIO19) while IAQ stays in the "Ruim"
  (bad) range.
- Removed the on-screen IAQ calibration-accuracy readout (still available
  as a Home Assistant text sensor). *Superseded in 2.0.0: back on page 2.*

## 1.0.0

First fully working version.

- ESPHome firmware for M5StickC PLUS2 + ENV Pro (BME688) via BSEC2
- Sensors: temperature, humidity, pressure, gas resistance, IAQ (0-500),
  IAQ accuracy, battery (V + %), wifi signal
- LVGL screen: single page, sensor list, physical-button navigation
- Automatic night mode (22h-7h, brightness only) + 4 selectable color
  themes + dimmer
- Device added to Home Assistant
- M5Launcher abandoned (WebUI hung on upload on this hardware),
  replaced by a dedicated device, direct USB/OTA flashing
