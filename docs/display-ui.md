# Screen layout: StickC Env Quarto

## Canvas

Landscape since 2026-10-09: the device lies down with the USB-C port on the
right. 240×135px, ST7789V2, `color_depth: 16` (RGB565), black background.

The rotation (`rotation: 90`, content turned 90° clockwise) goes in the
`lvgl:` block: ESPHome refuses `rotation` in the display config when LVGL
drives it. LVGL hands it to the `mipi_spi` driver, which rotates in hardware
(MADCTL), so the display keeps its native portrait `dimensions` and offsets
(135×240, offset 52/40). If the image ever comes out upside down, use 270.

## Critical positioning rule

**Never mix `align: CENTER` with `y:` on the same widget by accident.** In
LVGL, `align` turns `x:`/`y:` into an *offset from the alignment point*, not an
absolute position. This caused the very first layout attempt to come out
broken. The current layout uses `align` only deliberately: each row is its own
box, the label is `LEFT_MID` with `x: 6` and the value is `RIGHT_MID` with
`x: -6`, with no `y:` at all.

## Structure

- **Status bar** on the LVGL `top_layer` (rows 0-16), so it shows on every
  page: `select ^` on the left (page 3 only; it points at the side button,
  which is on top when the device lies down), then 4 Wi-Fi dots, a
  lightning bolt while on USB power (`Cabo USB`; LVGL's built-in
  `montserrat_12`, symbol U+F0E7), the battery % and a battery icon,
  right-aligned. The Wi-Fi dots moved 12 px left on 2026-10-13 to fit the
  bolt. A 1px rule at 35% opacity below it.
- **Pages**, each a box from y=17 to the bottom:

| Page | Rows | Label font | Value font |
|---|---|---|---|
| `page_main` (1) | Temperatura, Umidade, Qualidade do ar (Bom/Médio/Ruim) | 14 | 32 semibold |
| `page_details` (2) | Pressão, Gás, IAQ, Precisão (IAQ accuracy, translated) | 14 | 22 semibold |
| `page_config` (3) | Aviso sonoro, Aviso luminoso, each with ON / OFF pills | 16 | pills 14 |

Row boxes: page 1 at y 0/39/79 (39-40px tall), page 2 at y 0/30/59/89
(29-30px), page 3 at y 0/59 (59px). Page 3 pills are 46×26, radius 13, ON at
`x: -60` and OFF at `x: -6` from the right edge.

IAQ accuracy is translated on screen only: Stabilizing → Estabilizando,
Uncertain → Incerto, Calibrating → Calibrando, Calibrated → Calibrado. Home
Assistant keeps the original state.

Air quality on page 1: `Bom` below IAQ 100, `Médio` from 100 to 199, `Ruim`
at 200 and above (the same 200 threshold drives the beep and LED alerts).
All pages refresh every 10 s, the hidden ones included, so switching pages
never shows stale values.

## Brightness

Two levels, set from the `Brilho` select in Home Assistant: `Fraco` = 40%
(the default, less self-heating) and `Forte` = 100%. The choice survives a
reboot. The backlight light (`Tela Brilho`) uses `gamma_correct: 1.0`, so
the requested % matches the real PWM duty cycle; with the default 2.8, 20%
came out as about 1% (backlight dark) and 40% as about 8%.

## Buttons

- **A (M5, right of the screen)**: next page (1 → 2 → 3). On page 3 it walks
  the cursor through sound ON, sound OFF, light ON, light OFF; the next press
  goes back to page 1. Hold 5 s = restart.
- **B (side button, on top)**: on page 3 applies the option under the cursor
  and stays on page 3; no action on pages 1 and 2. Hold 8 s = factory reset.
- **PWR** (labeled C in the pin table): hardware wake only, no ESPHome logic.
- Pages 2 and 3 return to page 1 after 30 s / 15 s untouched.
- **Screen off** (Home Assistant's sleep mode turns `Tela Brilho` off at
  night): the first A press only wakes the screen, on page 1, at the chosen
  brightness. From there the buttons work as usual, and the screen goes
  dark again 30 s after the last press. Click beeps stay silent while woken
  this way. Known edge: if Home Assistant turns the screen back on during
  those 30 s, the firmware still turns it off at the end.
- **Motion wake** (since 2026-10-13): with the screen off, moving or tapping
  the device does the same as that first A press (MPU6886 accelerometer,
  polled every 200 ms). Movement while woken keeps the 30 s timer going.
  Sensitivity is the `motion_threshold` substitution (1.0 m/s², about a
  6 degree tilt); the `Acordar com Movimento` switch in Home Assistant
  turns it off. It only runs on USB power (`Cabo USB`); on battery the IMU
  is read once a minute and motion is ignored.

Page 3 "Aviso sonoro" is the `Aviso Sonoro Qualidade Ar` switch (buzzer when
the air turns bad) and "Aviso luminoso" is `Aviso Luminoso Qualidade Ar`
(red LED blink). The pills follow the switches when Home Assistant changes
them. `Som Geral` (mute everything) stays in Home Assistant only, as do the
brightness and color theme selects.

Page 3 pills: active option = filled with the theme color and black text;
the other one = black with theme-colored text; cursor = 2px theme-colored
outline 3px around the pill, blinking every 500 ms.

## Colors

Every text and shape uses the theme color, with no exceptions: labels,
values, lit dots (filled) and unlit dots (ring with a black fill), battery
icon, rule and pills.

4 themes (Verde 0x57D97C, Azul 0x4FA8E8, Vermelho 0xFF3B30, Âmbar 0xF2B84B),
chosen via the `select.stickc_env_quarto_tema` entity. The red was 0xE84C4C
until 2026-10-09; on the panel at 40% backlight it looked wine/pink.

How the color reaches every label: labels have no color of their own and
inherit `text_color` from their box, and every box uses the `box_style`
style, so one `lvgl.style.update` on `box_style` recolors all text. It has to
be a style: LVGL's default theme gives each `obj` a "card" style with its own
`text_color`, so a color set only on an outer box does not reach labels
inside nested boxes. Local colors on the boxes would override the style, so
the boxes set none.

Theme colors set via `lv_color_hex()` inside a `lambda` render **inverted**
on screen (a quirk observed live): the hex values baked into the firmware
are pre-complemented to compensate (0xFFFFFF in a lambda shows as black).
Colors set directly in YAML (no lambda) are not affected. See
[lessons-learned.md](lessons-learned.md#lv_color_hex-in-a-lambda-renders-inverted).

## Wifi signal dots

4 small `obj` circles (not font glyphs; Montserrat has no bullet
character, see [lessons-learned.md](lessons-learned.md)) act as a
phone-style signal indicator. Thresholds (RSSI dBm): 4 dots ≥ -55, 3 ≥ -65,
2 ≥ -75, 1 ≥ -85, 0 below that. Updated on every Wi-Fi signal reading (every
5 min) and whenever the theme changes; all unlit on disconnect.

## Fonts

`gfonts://Montserrat`: 11 (status bar), 14 (row labels on pages 1-2, pills),
16 (row labels on page 3), and 22 / 32 at weight 600 (values). Needs
`glyphsets: [GF_Latin_Core]` + explicit `glyphs:` for Portuguese accents:
the font doesn't ship with them by default. The "●" character doesn't exist
in Montserrat at all; dot-style indicators use an `obj` widget (drawn
circle) instead of text.

The USB lightning bolt uses LVGL's built-in `montserrat_12` instead, because
it carries the `LV_SYMBOL_*` glyphs (U+F0E7 = `LV_SYMBOL_CHARGE`); the
`gfonts://` Montserrat does not.
