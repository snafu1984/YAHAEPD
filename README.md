# YAHAEPD — Yet Another Home Assistant E-Paper Display

YAHAEPD is a Home Assistant-powered e-paper dashboard built with ESPHome for the **Seeed reTerminal E1001**.

The goal is to create a compact, energy-efficient display for essential household information. It combines weather data, calendar information, and astronomical details in a single, easy-to-read dashboard.

The project uses Home Assistant as the central source of data, with ESPHome handling the display, hardware interfaces, and power management. Node-RED, MQTT, and InfluxDB provide additional processing for the birthday calendar and Plantage rainfall data.

## Display Overview

The YAHAEPD display shows current weather conditions, weather forecasts, moon phase, sunrise and sunset times, moonrise and moonset times, and upcoming birthdays.

![YAHAEPD display overview](docs/images/display-overview.jpeg)

### Moon Phase Icons

The moon phase icons use the **Noto Emoji** font instead of Material Design Icons.

- **Font file:** `fonts/NotoEmoji-Regular.ttf`
- **ESPHome font ID:** `sym_moon`
- **Font configuration:** `config/packages/symbol.yaml`
- **Font size:** `90`
- **Unicode range:** `U+1F311`–`U+1F318`

The font configuration defines the eight moon phase symbols used by the display:

| Unicode | Moon Phase |
|---|---|
| `U+1F311` | New Moon |
| `U+1F312` | Waxing Crescent |
| `U+1F313` | First Quarter |
| `U+1F314` | Waxing Gibbous |
| `U+1F315` | Full Moon |
| `U+1F316` | Waning Gibbous |
| `U+1F317` | Last Quarter |
| `U+1F318` | Waning Crescent |

The font file must be present in the `fonts/` directory. The glyphs defined in `config/packages/symbol.yaml` must match the Unicode code points used in `config/packages/display.yaml`.

If you change the font or its glyphs, update both configuration files accordingly.

## Features

- **Date and time** — Displays the current date and time.
- **Current weather** — Provides an overview of current weather conditions.
- **Hourly forecast** — Shows the forecast for the next four hours.
- **Daily forecast** — Shows the forecast for the next four days.
- **Moon phase** — Displays the current lunar phase with a symbol and German description.
- **Sunrise and sunset** — Displays local sunrise and sunset times provided by Home Assistant.
- **Moonrise and moonset** — Displays local moonrise and moonset times using the optional Lunar Phase integration.
- **Birthday calendar** — Displays birthday information supplied by Node-RED via MQTT and made available through Home Assistant sensors.
- **Rainfall monitoring** — Supports Zeyn rainfall measurements and daily Plantage rainfall totals.
- **Battery monitoring** — Monitors battery voltage.
- **Deep sleep** — Reduces power consumption between display updates.
- **Manual refresh** — Allows manual wake-up and display refresh using the onboard KEY0 button.

The project is under active development. Features, configuration, and documentation may change as development continues.

## Hardware

YAHAEPD is developed for the following hardware:

| Component | Specification |
|---|---|
| Device | Seeed reTerminal E1001 |
| Microcontroller | ESP32-S3 |
| Display | 7.5-inch e-paper |
| Resolution | 800 × 480 pixels |
| Display technology | Monochrome e-paper with grayscale support |

The display is controlled directly by ESPHome. Home Assistant provides the information displayed on the screen.

## Software Requirements

- [Home Assistant](https://www.home-assistant.io/)
- [ESPHome](https://esphome.io/) **2026.9.1** — Version used for development and testing
- ESPHome Arduino framework
- Seeed reTerminal E1001 with ESP32-S3
- [Node-RED](https://nodered.org/)
- An MQTT broker accessible by Node-RED and Home Assistant
- InfluxDB 1.x for Plantage rainfall storage and calculations

**Optional components**

- HACS and the [Lunar Phase integration](https://github.com/ngocjohn/lunar-phase) for moonrise and moonset times. The moon phase itself uses the built-in Home Assistant Moon integration and does not require Lunar Phase.

Other ESPHome versions may work, but the project has been developed and tested with ESPHome 2026.9.1. Compatibility with other versions has not been verified.

The required Home Assistant integrations and entities must be configured before all display features can work correctly.

## How It Works

YAHAEPD uses the following architecture:

1. **Home Assistant** collects and prepares information, including weather forecasts, astronomical data, and data for the display.
2. **Node-RED** retrieves birthday events from an iCal calendar and publishes calendar data via MQTT.
3. **Node-RED and InfluxDB** process Plantage rainfall measurements received through The Things Stack (TTN). Node-RED stores individual measurements in InfluxDB and periodically calculates the rainfall totals for today and yesterday.
4. **MQTT** provides the calculated Plantage rainfall totals through the `precipitation/today` and `precipitation/yesterday` topics.
5. **ESPHome** retrieves the relevant Home Assistant entity states and handles the display logic and hardware interfaces.
6. **The e-paper display** presents the information in a readable dashboard.
7. **Deep sleep** reduces power consumption between updates, while the onboard KEY0 button allows a manual wake-up and refresh.

The Plantage rainfall day runs from **8:00 AM to 8:00 AM the following day**, using the `Europe/Berlin` time zone.

The display is refreshed as a whole rather than by independently updating individual screen regions.

The project aims to balance useful information, readability, and energy efficiency.

## Configuration

Before using YAHAEPD, configure the following:

- Wi-Fi credentials and Home Assistant connectivity.
- The required Home Assistant integrations and entities for weather, sunrise, sunset, moon phase, and rainfall.
- The Home Assistant template sensors used to prepare forecast and display-ready values.
- Node-RED, an MQTT broker, and the required flow configuration for the birthday calendar.
- Node-RED and InfluxDB for Plantage rainfall calculations.
- The MQTT Discovery configuration for the required Home Assistant sensors.

Home Assistant prepares data in the required format before ESPHome receives it. For example, sunrise and sunset times are provided as local-time strings, so the display does not need to perform time zone conversion itself.

**Important:** Do not commit Wi-Fi passwords, API encryption keys, private calendar URLs, TTN credentials, or other secrets to a public repository. Store sensitive values in an appropriate secrets configuration.

### Secrets Configuration

Copy `config/secrets.yaml.example` to `config/secrets.yaml` and replace the placeholder values with your own credentials.

The `secrets.yaml` file contains sensitive information and must not be committed to the repository. The `.gitignore` file excludes it from version control.

### Detailed Documentation

- [Home Assistant Configuration](docs/home-assistant.md) — Required integrations, template sensors, weather forecasts, sunrise and sunset, moonrise and moonset, rainfall, and display entities.
- [Node-RED Configuration](docs/node-red.md) — Birthday calendar flow, MQTT topics, MQTT Discovery, Plantage rainfall calculations, InfluxDB, and flow installation.

### Birthday Calendar

The birthday calendar retrieves upcoming events from an iCal calendar and publishes five entries to Home Assistant via MQTT. Each entry provides the weekday, date, event name, and age, if a birth year is available.

The age field represents the age the person will turn on their birthday in the current calendar year. If the calendar event's comment contains no birth year, the corresponding age field remains empty.

The Node-RED flow processes the five entries separately to keep the logic easy to understand, debug, and customize.

The current ESPHome display configuration uses the first four rows (`A1`–`D4`). The fifth row remains available in Home Assistant but is not currently displayed.

### Plantage Rainfall

The Plantage rain gauge sends measurements through The Things Stack (TTN). Node-RED processes the uplinks and stores the calculated rainfall amounts in InfluxDB.

Every 14 minutes, Node-RED queries InfluxDB to calculate the totals for the current and previous rainfall days. The results are published as retained MQTT messages:

- `precipitation/today`
- `precipitation/yesterday`

Separate MQTT Discovery messages register the corresponding sensors in Home Assistant.

See [Node-RED Configuration](docs/node-red.md#plantage-rainfall-calculation) for the calculation flow and setup instructions.

## Project Status

YAHAEPD is a personal project in active development.

The current focus is on a reliable display layout, correct Home Assistant data handling, predictable wake-up behavior, and efficient e-paper updates.

The repository includes the ESPHome configuration, supporting files, Node-RED flow exports, and documentation for the Home Assistant and Node-RED components.

## License

YAHAEPD is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
