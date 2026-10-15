# 🌡️ M5StickC PLUS2 + ENV Pro: ESPHome Environmental Sensor

> A bedside/desk air-quality monitor built from off-the-shelf M5Stack
> hardware: real IAQ (0-500) via Bosch's BSEC2 algorithm, a legible
> landscape LVGL screen, and native Home Assistant integration: no cloud,
> no companion app.

![ESPHome](https://img.shields.io/badge/ESPHome-2026.9-blue)
![License: MIT](https://img.shields.io/badge/license-MIT-green)
![Home Assistant](https://img.shields.io/badge/Home%20Assistant-native%20API-41BDF5)

---

## Why this exists

Most BME680/BME688 ESPHome guides expose raw `gas_resistance` in ohms and
leave you to invent your own "good/bad air" heuristic. That's noisy and
hard to threshold sensibly. This project uses ESPHome's native
`bme68x_bsec2` component instead, which runs Bosch's actual BSEC2
algorithm on-device and outputs a real **0-500 IAQ index** (the same
scale used by commercial air-quality monitors), plus a calibration-
confidence indicator so you know when to trust the reading.

It also drives the M5StickC's screen with a proper LVGL UI (not
`it.printf` in a lambda), so the device is useful standalone, on a
nightstand or desk, without needing Home Assistant open on a phone.

## Features

- Real IAQ (0-500) via BSEC2, plus temperature, humidity, pressure and gas
  resistance, published to Home Assistant over the native API (no MQTT
  broker required). Readings are averaged on the device (60 s; pressure
  and gas 5 min) instead of being sent every 3 s.
- Self-heating handled: 160 MHz CPU, Wi-Fi power saving on, backlight at
  40% by default, and a `temperature_offset` inside the BSEC2 block that
  corrects temperature, humidity and IAQ together. See
  [Mounting and self-heating](#mounting-and-self-heating).
- Landscape screen (device lying down, USB-C on the right) with three
  pages:
  1. temperature, humidity, air quality (`Bom` / `Médio` / `Ruim`);
  2. pressure, gas resistance, IAQ number, IAQ calibration accuracy;
  3. settings: bad-air sound alert and light alert, ON/OFF.

  A status bar on every page shows 4 Wi-Fi signal dots, a lightning bolt
  while on USB power, and the battery % and icon. Pages 2 and 3 return to
  page 1 on their own.
- 4 color themes and 2 brightness levels (40% / 100%), set from Home
  Assistant. Every text and shape follows the theme color.
- No clock and no schedule on the device: Home Assistant owns the screen
  light (turn it off at bedtime from your own automation).
- Wake on demand while the screen is off: the M5 button, or moving or
  tapping the device (accelerometer, USB power only), lights the screen on
  page 1 for 30 s, with click beeps silent.
- Bad-air-quality alerts (IAQ 200 and above): one beep when the air turns
  bad and a blinking onboard red LED while it lasts, each toggleable from
  Home Assistant or page 3, plus a switch that mutes every sound.
- Protection kit: board temperature (MPU6886 IMU), reset reason, last boot
  timestamp (for an unexpected-restart alert) and USB power state, as
  diagnostic entities.
- No companion app or launcher: a single dedicated ESPHome firmware,
  flashed once via USB, updated over-the-air after that.

## Hardware

| Item | Detail |
|---|---|
| Board | M5StickC PLUS2 (ESP32-PICO-V3-02, 2MB Quad PSRAM, 8MB flash, **no** AXP192), run at 160 MHz |
| Sensor | M5Stack ENV Pro (BME688), Grove port, I2C address 0x77 |
| Display | Built-in ST7789V2, 135×240, used in landscape (240×135) |
| Onboard parts used | MPU6886 IMU (board temperature, motion wake), buzzer (GPIO2), red LED (GPIO19), battery ADC (GPIO38) |
| Firmware | ESPHome (built with 2026.9.1, `min_version` 2026.6.0), esp-idf framework |

Full pinout, confirmed on real hardware, in
[docs/hardware-wiring.md](docs/hardware-wiring.md).

## Mounting and self-heating

The ESP32, the Wi-Fi radio and the backlight warm the board, and a BME688
a few centimeters away reads that heat as room temperature (humidity and
IAQ then go wrong too, since BSEC2 derives them from the same
temperature). On this build the first readings were about +4.9 °C high,
and the device did not feel warm to the touch.

1. Put the ENV Pro on a Grove cable (20 to 50 cm), beside or below the
   StickC, never above it or inside the same closed case. Keep it away
   from sun, AC airflow and the USB charger. This alone does most of the
   work.
2. Keep the firmware defaults: `cpu_frequency: 160MHz`, Wi-Fi
   `power_save_mode: light`, brightness at 40% (`Fraco`).
3. Only then set the `temperature_offset` substitution. The shipped value
   was measured for the author's mounting and is not right for yours.
   Start at `0`, let the device settle for 30 min, then take the median of
   (sensor minus a trusted thermometer) over 60 min with the AC or heating
   off, and use that. Change one thing at a time, and re-measure after any
   change to mounting, CPU, Wi-Fi or brightness.

The offset is applied in the `bme68x_bsec2_i2c` block, not in a sensor
filter or a Home Assistant template, because BSEC2 uses the corrected
temperature for humidity and IAQ too. Numbers and the full story in
[docs/lessons-learned.md](docs/lessons-learned.md).

## Getting started

1. Install [ESPHome](https://esphome.io/) (CLI or the Home Assistant
   add-on).
2. Copy `secrets.yaml.example` to `esphome/secrets.yaml` (next to the
   firmware YAML: ESPHome looks for it there) and fill in your own
   values (wifi credentials, API encryption key, OTA password, fallback
   access point password; see the comments in that file for how to
   generate the encryption key).
3. Review the `substitutions:` at the top of
   `esphome/stickc-env-quarto.yaml`:
   - `name` / `friendly_name`: the device hostname and the name shown in
     Home Assistant. Rename **before** adding the device to Home
     Assistant: entity IDs are created from these names on first
     discovery.
   - `temperature_offset`: see
     [Mounting and self-heating](#mounting-and-self-heating).
   - `motion_threshold`: motion wake sensitivity in m/s² (default `1.0`,
     about a 6 degree tilt or a light tap); raise it if bumping the
     nightstand wakes the screen.

   The `stickc_env_quarto_*` secret keys can keep their names; if you
   rename them, do it in both files.
4. Compile and flash over USB the first time:

   ```sh
   esphome run esphome/stickc-env-quarto.yaml
   ```

5. After the first flash, subsequent updates can go out over-the-air
   (OTA) with the same command, no cable needed.
6. Add the device in Home Assistant (it is discovered automatically).
   BSEC2 can take days of running before `IAQ Precisão` reaches
   `Calibrated`; don't trust the IAQ number before that. The calibration
   is saved every hour, so a reboot keeps most of it.

> **Windows users:** ESP-IDF's toolchain silently fails to produce a
> working binary under Git Bash / MSYS: always run ESPHome commands
> from PowerShell or cmd.exe. See
> [docs/lessons-learned.md](docs/lessons-learned.md) for the full story.

## Using the device

Device lying down, USB-C on the right:

| Button | Click | Hold |
|---|---|---|
| A (M5 logo, right of the screen) | Next page (1 → 2 → 3). On page 3 it moves the cursor through the four ON/OFF options, then back to page 1. With the screen off: wake it | 5 s: restart |
| B (side, on top) | Page 3: apply the option under the cursor. No action on pages 1 and 2 | 8 s: factory reset (erases every saved setting: theme, brightness, switches and the BSEC2 calibration; Wi-Fi comes from `secrets.yaml`, so it reconnects. On purpose there is no Home Assistant button for it) |
| PWR (bottom) | Hardware power button only, no firmware action | |

Pages 2 and 3 go back to page 1 after 30 s / 15 s untouched. When the
screen was woken (button or motion), it goes dark again 30 s after the
last press or movement. Screen layout and colors:
[docs/display-ui.md](docs/display-ui.md).

## Home Assistant side

The firmware has no schedule. Suggested automations (not included;
replace the placeholder IDs with yours):

- **Night screen:** turn `light.stickc_env_quarto_tela_brilho` off at
  bedtime and on in the morning. While it is off, the M5 button or a
  nudge (on USB power) lights it for 30 s.
- **Unexpected restart:** trigger on `Último Boot` (a timestamp that only
  changes when the device really boots), not on `Motivo do Reset`, which
  is re-sent every time Home Assistant restarts. Skip reset reasons that
  contain `esphome.ota` (your own OTA updates):

  ```yaml
  - alias: "Air sensor: unexpected restart"
    mode: restart
    triggers:
      - trigger: state
        entity_id: sensor.stickc_env_quarto_ultimo_boot
        not_to: ["unknown", "unavailable"]
    conditions:
      # A real restart has a fresh boot time; a Home Assistant restart
      # re-sends the old one.
      - condition: template
        value_template: >-
          {{ now() - as_datetime(trigger.to_state.state) < timedelta(minutes=10) }}
    actions:
      # Give the new reset reason time to reach Home Assistant.
      - delay: "00:01:30"
      - variables:
          reason: "{{ states('sensor.stickc_env_quarto_motivo_do_reset') }}"
      - condition: template
        value_template: "{{ 'esphome.ota' not in reason }}"
      - action: notify.mobile_app_YOUR_PHONE
        data:
          message: "Air sensor restarted. Reason: {{ reason }}"
  ```

- **Board too hot:** alert if `Temperatura Placa` stays above about 50 °C
  for 10 min. On the author's build (160 MHz, Wi-Fi power saving, 40%
  backlight) the board runs at about 38 °C out of a case and about 42 °C
  in one, roughly 14 °C above the room.

## Home Assistant entities

Exposed over ESPHome's native API (device `stickc-env-quarto`, friendly
name `StickC Env Quarto`). The IDs below are the defaults Home Assistant
derives from the friendly name and the entity names; yours may differ
(for example with an area name in front, or if you renamed the device).
Entity names are in Portuguese, see [Language / Idioma](#language--idioma).
ESPHome text sensors show up under `sensor.` in Home Assistant.

| Entity | What it is |
|---|---|
| `sensor.stickc_env_quarto_temperatura` | Temperature (°C), self-heating compensated, 60 s average |
| `sensor.stickc_env_quarto_umidade` | Relative humidity (%), 60 s average |
| `sensor.stickc_env_quarto_pressao` | Atmospheric pressure (hPa), 5 min average |
| `sensor.stickc_env_quarto_resistencia_de_gas` | Raw gas resistance (Ω), 5 min average |
| `sensor.stickc_env_quarto_iaq` | Air quality index, 0-500 (BSEC2), 60 s average |
| `sensor.stickc_env_quarto_iaq_precisao` | IAQ calibration confidence: `Stabilizing` / `Uncertain` / `Calibrating` / `Calibrated` (text sensor) |
| `sensor.stickc_env_quarto_bateria` | Battery (%), estimated from the voltage (3.27 V = 0%, 4.15 V = 100%) |
| `sensor.stickc_env_quarto_bateria_voltagem` | Battery (V), every 5 min |
| `sensor.stickc_env_quarto_wifi_sinal` | Wi-Fi signal (dBm), every 5 min |
| `sensor.stickc_env_quarto_temperatura_placa` | Diagnostic: board temperature from the MPU6886 IMU (°C), 60 s average |
| `sensor.stickc_env_quarto_ultimo_boot` | Diagnostic: last boot time (timestamp), the trigger for an unexpected-restart alert |
| `sensor.stickc_env_quarto_motivo_do_reset` | Diagnostic: reset reason (text sensor) |
| `binary_sensor.stickc_env_quarto_cabo_usb` | Diagnostic: on USB power, inferred from the battery voltage (on at ≥ 4.13 V, off below 4.10 V) |
| `light.stickc_env_quarto_tela_brilho` | Screen backlight (on/off and dimmer); on at boot |
| `select.stickc_env_quarto_tema` | Color theme: `Verde` / `Azul` / `Vermelho` / `Âmbar` (green, blue, red, amber) |
| `select.stickc_env_quarto_brilho` | Brightness: `Fraco` (40%) / `Forte` (100%) |
| `switch.stickc_env_quarto_som_geral` | All sounds on/off (off mutes every beep: boot chime, clicks, alert) |
| `switch.stickc_env_quarto_aviso_sonoro_qualidade_ar` | Bad-air-quality beep on/off (also on screen page 3) |
| `switch.stickc_env_quarto_aviso_luminoso_qualidade_ar` | Bad-air-quality red LED blink on/off (also on screen page 3) |
| `switch.stickc_env_quarto_acordar_com_movimento` | Wake the screen on motion (USB power only) on/off |
| `button.stickc_env_quarto_reiniciar` | Restart the device |

The four switches default to on. Theme and brightness survive a reboot.

## Project layout

```
esphome/
├── stickc-env-quarto.yaml   # the firmware (source of truth)
└── poc/                     # early proof-of-concept builds

docs/
├── hardware-wiring.md   # confirmed pinout
├── display-ui.md        # screen layout, coordinates, fonts, buttons
├── decisions.md         # why things were built this way
└── lessons-learned.md   # real bugs hit during development, and the fix

CHANGELOG.md
secrets.yaml.example     # copy to esphome/secrets.yaml (git-ignored)
```

## Language / Idioma

Code, comments and docs are in English. The text a person sees, on the
screen and in Home Assistant, is in Brazilian Portuguese (pt-BR), the
language of the household this was built for. To translate it, edit
`esphome/stickc-env-quarto.yaml`:

| What | pt-BR strings | Where in the YAML |
|---|---|---|
| Screen labels | `Temperatura`, `Umidade`, `Qualidade do ar`, `Pressão`, `Gás`, `Precisão`, `Aviso sonoro`, `Aviso luminoso` | `text:` of the labels under `lvgl:` → `pages:` |
| Screen values | `Bom` / `Médio` / `Ruim` (air quality), `Estabilizando` / `Incerto` / `Calibrando` / `Calibrado` (IAQ accuracy) | the `p1_air` and `p2_acc` lambdas in the `interval: 10s` block |
| Entity names | `Temperatura`, `Umidade`, `Pressão`, `Resistência de Gás`, `IAQ Precisão`, `Bateria`, `Bateria Voltagem`, `Wifi Sinal`, `Temperatura Placa`, `Último Boot`, `Motivo do Reset`, `Cabo USB`, `Tela Brilho`, `Tema`, `Brilho`, `Som Geral`, `Aviso Sonoro Qualidade Ar`, `Aviso Luminoso Qualidade Ar`, `Acordar com Movimento`, `Reiniciar` | `name:` of each entity |
| Device name | `StickC Env Quarto` ("quarto" = bedroom), hostname `stickc-env-quarto`, fallback access point `StickC-Env-Quarto Fallback` | `friendly_name` and `name` under `substitutions:`; the access point name is the `ssid:` under `wifi:` → `ap:` (not driven by the substitutions) |
| Select options | `Tema`: `Verde`, `Azul`, `Vermelho`, `Âmbar`; `Brilho`: `Fraco`, `Forte` | `options:` and `initial_option:` of each select, **and** the same strings in its `set_action` lambda and in the `on_boot` sync lambdas (under `esphome:`), or the select stops matching |

### pt-BR to English

| pt-BR | English | Used for |
|---|---|---|
| `Temperatura` | Temperature | sensor, screen label |
| `Umidade` | Humidity | sensor, screen label |
| `Pressão` | Pressure | sensor, screen label |
| `Resistência de Gás` | Gas resistance | sensor |
| `Gás` | Gas | screen label |
| `Qualidade do ar` | Air quality | screen label |
| `Precisão` / `IAQ Precisão` | Accuracy / IAQ accuracy | screen label, sensor |
| `Bateria` | Battery | sensor |
| `Bateria Voltagem` | Battery voltage | sensor |
| `Wifi Sinal` | Wi-Fi signal | sensor |
| `Temperatura Placa` | Board temperature | sensor |
| `Último Boot` | Last boot | sensor (timestamp) |
| `Motivo do Reset` | Reset reason | sensor |
| `Cabo USB` | USB cable (on USB power) | binary sensor |
| `Tela Brilho` | Screen backlight | light |
| `Tema` | Theme | select |
| `Brilho` | Brightness | select |
| `Som Geral` | All sounds | switch |
| `Aviso Sonoro Qualidade Ar` / `Aviso sonoro` | Air quality sound alert | switch, screen label |
| `Aviso Luminoso Qualidade Ar` / `Aviso luminoso` | Air quality light alert | switch, screen label |
| `Acordar com Movimento` | Wake on motion | switch |
| `Reiniciar` | Restart | button |
| `Bom` / `Médio` / `Ruim` | Good / Moderate / Bad | air quality on screen |
| `Estabilizando` / `Incerto` / `Calibrando` / `Calibrado` | Stabilizing / Uncertain / Calibrating / Calibrated | IAQ accuracy on screen |
| `Fraco` / `Forte` | Low (40%) / High (100%) | brightness options |
| `Verde` / `Azul` / `Vermelho` / `Âmbar` | Green / Blue / Red / Amber | theme options |
| `Quarto` | Bedroom | device name |

Rename entities before adding the device to Home Assistant: entity IDs are
created from these names on first discovery. The fonts carry an explicit
`glyphs:` list for Portuguese accents (`font:` block); add the accented
characters your language needs there, or they render blank. The
`IAQ Precisão` state in Home Assistant stays in English (it comes from
BSEC2); only the screen translates it.

## Known issues and open points

- `Cabo USB` is inferred from the battery voltage and only re-checked
  every 5 min; the "unplugged" threshold (below 4.10 V) has not been
  confirmed with a measured unplugged run yet.
- If Home Assistant turns the screen on during a 30 s button/motion wake,
  the firmware still turns it off when the 30 s end.
- The onboard red LED polarity (`inverted: false` on GPIO19) is not
  confirmed on hardware: if the LED is lit when it should be off, flip
  `inverted`.

The screen freezing after days of uptime (suspected heap fragmentation,
not a crash: sensors kept reporting while the UI locked up), the
self-heating story, the inverted `lv_color_hex()` colors in lambdas, the
`align: CENTER` + `y:` LVGL gotcha, the ST7789V2 black-screen offset
quirk, and a few other real bugs hit during development are written up
in detail in [docs/lessons-learned.md](docs/lessons-learned.md): worth a
look if you're debugging something similar on an M5StickC PLUS2.

## License

[MIT](LICENSE), © 2026 Silas Gubitoso.

## Português

Firmware ESPHome para M5StickC PLUS2 com ENV Pro (BME688): IAQ real pelo
BSEC2, tela em paisagem com três páginas e integração nativa com o Home
Assistant. O código e a documentação estão em inglês; a tela e os nomes
das entidades no Home Assistant estão em português do Brasil. Antes de
gravar, monte o sensor num cabo Grove ao lado ou abaixo da placa e meça o
seu próprio `temperature_offset` (ver
[Mounting and self-heating](#mounting-and-self-heating)).
