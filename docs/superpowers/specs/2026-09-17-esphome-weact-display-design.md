# Design: ESP8266 + WeAct 2.13" e-ink (B&W&R) + Home Assistant

Date: 2026-09-17
Status: approved by the user at the time, since implemented and
extended — see [README.md](../../../README.md) for the current,
as-built design (the battery status + grid consumption display shown
there superseded the original "house consumption only" goal below).

## Goal

Test the integration and behavior of two components with Home
Assistant, using ESPHome:

1. An ESP8266 module (NodeMCU-type) connected over USB.
2. A WeAct 2.13" tricolor e-ink module (Black/White/Red).

The end result is an ESPHome device that shows the household's power
consumption (in watts), read from an existing Home Assistant sensor,
updating the screen at least once a minute.

This is a proof-of-concept (PoC) project, not a production deployment:
validating that the integration works end-to-end is prioritized over
long-term robustness.

## Hardware

### ESP8266

- NodeMCU-type board (exact model to confirm once detected over
  USB — see "Open decisions").
- Framework: Arduino (via ESPHome's `esp8266:` platform).
- GPIOs usable without boot restrictions: GPIO4, GPIO5, GPIO12,
  GPIO13, GPIO14, GPIO16. Avoided: GPIO0/2/15 (boot mode), GPIO1/3
  (UART), GPIO6-11 (internal flash).

### WeAct 2.13" B&W&R e-ink panel

- Controller: SSD1680.
- Panel: GDEY0213Z98, 122×250 pixels, tricolor (black/white/red).
- Only supports full refresh (no reliable partial refresh) — a full
  refresh cycle takes ~15-20s and causes visible flicker.
- Source: WeAct's official ESP8266 example
  (`WeActStudio.EpaperModule/Example/EpaperModuleTest_Arduino_ESP8266`)
  and the SSD1680 datasheet included in the same repository
  (`Doc/SSD1680_Datasheet.pdf`).

### Proposed wiring (ESP8266 hardware SPI)

| E-ink signal | ESP8266 GPIO | NodeMCU pin (reference) |
|---|---|---|
| CS   | GPIO15 | D8 |
| SCK/CLK | GPIO14 | D5 |
| MOSI/SDA | GPIO13 | D7 |
| BUSY | GPIO16 | D0 |
| RST  | GPIO5  | D1 |
| DC   | GPIO4  | D2 |
| VCC  | 3V3 | 3V3 |
| GND  | GND | GND |

Uses the ESP8266's hardware SPI bus (HSPI: GPIO12/13/14) for CLK/MOSI,
as recommended in the manufacturer's own example — avoids software SPI
and is more reliable at the panel's refresh speed. MISO (GPIO12) isn't
used (the panel doesn't need it) but stays reserved in case a future
ESPHome SPI component requires it in the bus configuration.

GPIO15 (CS) needs to be LOW during boot so it doesn't interfere with
boot mode selection; since it's an output actively driven by SPI, this
isn't an issue in practice, but it's documented here as something to
watch if erratic boots show up.

## Software

- **ESPHome** (CLI + YAML), installed via `pip`/a Python virtual
  environment. Minimum version: whichever includes the `epaper_spi`
  component with the `weact-2.13in-3c` model — available since release
  **2026.9.0** (published 2026-09-16). Don't use versions older than
  February 2026.
- **PlatformIO**: used internally by ESPHome to compile the firmware;
  no Arduino/PlatformIO code is written by hand. Installed as an
  environment dependency; the PlatformIO VSCode extension is only used
  as a helper (serial monitor, browsing the build), not as the main
  compile flow.
- **ESPHome VSCode extension**: YAML syntax highlighting and
  validation.
- ESPHome display platform for this panel: `platform: epaper_spi`,
  `model: weact-2.13in-3c` (do NOT use `waveshare_epaper`, which
  doesn't support this tricolor 2.13" variant).

## Repository structure

```
esphome-weact-2.13/
├── README.md              # main project documentation
├── CLAUDE.md               # conventions for working with Claude Code
├── .gitignore               # excludes secrets.yaml, .esphome/, .pio/
├── secrets.yaml             # WiFi password, etc. (NOT committed)
├── secrets.yaml.example     # secrets.yaml template, committed
└── weact-display.yaml       # single ESPHome config
```

A single ESPHome YAML file: this is one device with one purpose. No
`packages:` or split into multiple files — only reconsider this if the
project grows to more than one device or display.

## ESPHome configuration (key components)

- `esphome:` — device name, `substitutions:` for configurable values
  (see below).
- `esp8266:` — `board:` (to confirm), `framework: arduino`.
- `spi:` — hardware SPI bus, `clk_pin: GPIO14`, `mosi_pin: GPIO13`.
- `display:`
  - `platform: epaper_spi`
  - `model: weact-2.13in-3c`
  - `cs_pin`, `dc_pin`, `busy_pin`, `reset_pin` per the wiring table.
  - `update_interval:` controlled by a substitution (see refresh
    strategy).
  - `full_update_every:` to force a periodic clean and avoid ghosting
    (exact value to tune empirically, starting point: every 30
    refreshes).
- `wifi:` — `ssid` and `password: !secret wifi_password`.
- `api:` — ESPHome's native API; Home Assistant discovers the device
  via mDNS, no manual token needed.
- `logger:` — UART output (hardware_uart: UART0), for serial-port
  debugging.
- `sensor:` — `platform: homeassistant`, `entity_id:` (placeholder
  until the real house-consumption sensor is confirmed), with a
  `throttle` filter so the screen doesn't redraw more than once per
  configured interval.

### Proposed substitutions

```yaml
substitutions:
  display_refresh_interval: "60s"   # "configurable later", item 1
  power_entity_id: "sensor.TBD_house_consumption"
```

## Data flow

1. The power sensor already exists in Home Assistant (a consumption
   measurement integration, not yet identified — see open decisions).
2. HA pushes the value to the ESP8266 over ESPHome's native API
   whenever the sensor's state changes (push, not polling).
3. A `throttle: ${display_refresh_interval}` filter on the sensor
   limits how often per minute the screen update is triggered.
4. A full e-ink refresh only fires if the displayed value actually
   changes (e.g. rounded to whole watts).
5. The e-ink keeps showing the last drawn image between refreshes
   (native e-ink behavior, no extra logic needed).

## Error handling

- **WiFi down**: ESPHome's native auto-reconnect. The last valid image
  on the e-ink isn't overwritten.
- **API/HA down**: `api:`'s native auto-reconnect. The sensor filter
  ignores `NaN`/unknown values, avoiding a redraw with garbage or
  "unavailable" data.
- **Panel ghosting**: mitigated with `full_update_every`.
- **Compile/flash failure**: validated at each phase with
  `esphome compile` and `esphome upload` before considering the phase
  done (see the phase plan).

## Testing and validation

There are no traditional automated tests in an ESPHome project —
validation happens phase by phase, each with its own "done" criteria:

1. Environment (PlatformIO + ESPHome installed and working).
2. ESP8266 detected over USB (serial port visible).
3. Basic firmware compiled and flashed successfully.
4. Logs readable via the ESPHome serial monitor.
5. Physical wiring documented and validated with continuity/a
   multimeter if needed.
6. "Hello world!" text visible on the e-ink panel.
7. Device connected to the WiFi network.
8. HA consumption value visible on the panel, updating per the
   defined refresh strategy.

Each phase is tracked as a GitHub issue in the
`MarceloSalazar/esphome-weact-2.13` repository, with its acceptance
criteria.

## Secrets management

- `secrets.yaml` (gitignored) holds `wifi_password` and any other
  sensitive value.
- `secrets.yaml.example` is committed as a template, with placeholder
  values.
- `.gitignore` also excludes `.esphome/` and `.pio/` (build
  directories generated by ESPHome/PlatformIO).

## Open decisions (deliberately deferred)

- **Exact ESP8266 board model** (`board:` in the config): pending
  confirmation once the device is detected over USB. Detection
  troubleshooting in progress.
- **`entity_id` of the house-consumption sensor in Home Assistant**:
  the user indicated this isn't a priority yet; it'll be resolved
  before implementing step 10 (final HA integration). Until then, the
  YAML uses a clearly marked placeholder.
- **Exact `full_update_every` value**: will be tuned empirically once
  the panel is operational, based on observed ghosting.

## Out of scope

- Power/battery management (the device is permanently USB-powered).
- Partial refresh of the panel (not reliably supported by the SSD1680
  controller in tricolor mode).
- Multiple displays or additional devices.
- Manual Home Assistant authentication/token (native discovery via the
  ESPHome API is used instead).
