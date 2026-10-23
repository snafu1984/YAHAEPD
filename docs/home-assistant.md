# Home Assistant Configuration

This document describes the Home Assistant integrations, template sensors, and configuration required by YAHAEPD (Yet Another Home Assistant E-Paper Display).

The configuration provides current weather data, hourly and daily forecasts, sunrise and sunset times, moon phase and moonrise/moonset times, rainfall measurements, and birthday calendar data.

## Requirements

The following components are required for the complete setup:

- Home Assistant
- ESPHome integration
- A weather integration providing `weather.forecast_home`
- The built-in Sun integration
- The built-in Moon integration
- The Lunar Phase integration for moonrise and moonset times
- An MQTT broker for the birthday calendar and Plantage rainfall sensors
- Node-RED for the birthday calendar, Plantage rainfall calculations, and MQTT Discovery
- InfluxDB 1.x for Plantage rainfall storage and calculations

The weather, Sun and Moon sensors can be configured independently of the birthday calendar and precipitation sensor.

## Required Integrations

### Weather

The configuration uses the Home Assistant weather entity:

`weather.forecast_home`

It retrieves hourly and daily forecasts through the `weather.get_forecasts` action.

If your weather entity has a different name, update the entity references in `configuration.yaml`.

### Sun

The built-in Sun integration provides the next sunrise and sunset through the `sun.sun` entity.

The following template sensors convert the timestamps to local time in `HH:MM` format:

- `sensor.display_sunrise`
- `sensor.display_sunset`

If the corresponding timestamp is unavailable, the template returns `unavailable`.

### Moon Phase

The built-in Home Assistant Moon integration provides the moon phase through:

`sensor.moon_phase`

No additional custom integration is required for the moon phase itself.

### Lunar Phase

The Lunar Phase custom integration provides the moonrise and moonset times.

Repository: [Lunar Phase](https://github.com/ngocjohn/lunar-phase)

The configuration expects the following source entities:

- `sensor.winsen_luhe_moon_rise`
- `sensor.winsen_luhe_moon_set`

The template sensors convert these timestamps to local time in `HH:MM` format:

- `sensor.display_moonrise`
- `sensor.display_moonset`

If the source entity is `unknown`, `unavailable`, `none`, or empty, the corresponding display sensor returns `--:--`. Otherwise, the template converts the source value to local time in `HH:MM` format.

If your Lunar Phase integration uses different entity IDs, update the source references in `configuration.yaml`.

## Weather Forecasts

The configuration creates two forecast template sensors:

- `sensor.weather_forecast_hourly`
- `sensor.weather_forecast_daily`

Both are updated every hour and when Home Assistant starts.

### Hourly Forecast

The hourly forecast is retrieved using:

```yaml
action: weather.get_forecasts
data:
  type: hourly
```

The template filters out forecast entries that are not in the future and stores up to four entries.

Each entry contains:

- Datetime
- Local time in `HH:MM` format
- Weather condition
- Temperature
- Precipitation probability
- Precipitation amount

The following display sensors provide the data in a pipe-separated format:

- `sensor.display_forecast_hour_1`
- `sensor.display_forecast_hour_2`
- `sensor.display_forecast_hour_3`
- `sensor.display_forecast_hour_4`

Format:

`Time|Condition|Temperature|Precipitation probability|Precipitation amount`

If a forecast entry is missing, a fallback value is used.

### Daily Forecast

The daily forecast is retrieved using:

```yaml
action: weather.get_forecasts
data:
  type: daily
```

The template processes entries `forecast[1:5]`, skipping the first entry and selecting up to four subsequent entries.

Each entry contains:

- Local date in `DD.MM.` format
- German weekday abbreviation (`Mo`, `Di`, `Mi`, `Do`, `Fr`, `Sa`, `So`)
- Weather condition
- High temperature
- Low temperature
- Precipitation probability
- Precipitation amount

The display sensors are:

- `sensor.display_forecast_day_1`
- `sensor.display_forecast_day_2`
- `sensor.display_forecast_day_3`
- `sensor.display_forecast_day_4`

Format:

`Date|Weekday|Condition|High|Low|Precipitation probability|Precipitation amount`

Missing entries use fallback values.

## Precipitation Sensors

The Plantage rain gauge uses the RANLWE01 sensor. Its uplinks are received through The Things Stack (TTN) and processed by Node-RED.

The raw `rainlevel` value represents the number of tipping events since the last uplink. Each tipping event corresponds to `0.5 mm` of rainfall.

Node-RED calculates the rainfall amount for each uplink and stores the measurement in InfluxDB 1.x. The measurement is stored as `plantage_rain`, with the following fields:

- `tip_count` — Number of tipping events reported by the device
- `rain_mm` — Calculated rainfall amount in millimeters
- `frame_counter` — TTN uplink frame counter

The device ID is stored as a tag, and the TTN `received_at` timestamp is used as the measurement timestamp.

### Daily Rainfall Totals

Node-RED calculates the daily rainfall totals by querying InfluxDB. The rainfall day runs from **8:00 AM to 8:00 AM the following day**, using the `Europe/Berlin` time zone, including daylight saving time changes.

The flow calculates two totals:

- **Today:** The current rainfall period, from the most recent 8:00 AM boundary to the next 8:00 AM boundary.
- **Yesterday:** The previous completed rainfall period, from 8:00 AM to 8:00 AM.

The totals are recalculated every 14 minutes and published as retained MQTT messages:

| MQTT topic | Description |
|---|---|
| `precipitation/today` | Total rainfall for the current rainfall period |
| `precipitation/yesterday` | Total rainfall for the previous completed rainfall period |

Separate MQTT Discovery messages register the corresponding sensors in Home Assistant. The flow defines the following discovery configuration topics:

- `homeassistant/sensor/precipitation/today/config`
- `homeassistant/sensor/precipitation_yesterday/config`

The actual sensor entity IDs are assigned by Home Assistant and may differ depending on existing entities and naming conflicts.

The calculation is performed by Node-RED and InfluxDB, not by a Home Assistant template sensor or utility meter. See [Node-RED Configuration](node-red.md#plantage-rainfall-calculation) for the flow details and installation instructions.

## Birthday Calendar via MQTT

The birthday calendar is provided by Node-RED. It retrieves calendar entries and publishes the results to MQTT topics. A separate Node-RED flow registers the corresponding MQTT sensors in Home Assistant through MQTT Discovery.

See [Node-RED Configuration](node-red.md) for setup instructions and flow details.

### MQTT Topics

The birthday flow publishes up to five calendar entries using four topic groups:

| Topic group | Content | Example |
|---|---|---|
| `Display/A1`–`Display/A5` | Weekday | `Display/A1` |
| `Display/B1`–`Display/B5` | Date | `Display/B1` |
| `Display/C1`–`Display/C5` | Event name | `Display/C1` |
| `Display/D1`–`Display/D5` | Age | `Display/D1` |

The birthday flow calculates the age from the birth year entered in the calendar event's comment. The calculated value represents the age the person will turn on their birthday in the current year. If the comment is empty and contains no birth year, the corresponding age field remains empty.

The MQTT messages are retained so that the latest values remain available to subscribers.

### MQTT Discovery

The separate discovery flow registers the MQTT sensors automatically in Home Assistant.

For example, the discovery configuration for `Display/A1` uses the topic:

`homeassistant/sensor/display_a1/config`

The corresponding state topic is:

`Display/A1`

The resulting entity is:

`sensor.display_a1`

The same naming pattern applies to the remaining sensors:

- `sensor.display_a1`–`sensor.display_a5`
- `sensor.display_b1`–`sensor.display_b5`
- `sensor.display_c1`–`sensor.display_c5`
- `sensor.display_d1`–`sensor.display_d5`

The discovery flow groups these entities under the device **Display Matrix**.

The MQTT sensor definitions are not configured directly in `configuration.yaml`; they are created through MQTT Discovery by Node-RED.

The display uses the first four rows (`A1`–`D4`). The fifth row remains available in Home Assistant but is not displayed by the current ESPHome layout.

## ESPHome Integration

ESPHome reads the Home Assistant entities and uses them to render the display.

The Home Assistant entities referenced by the ESPHome configuration must exist and have valid states. Check the entity IDs if values are missing from the display.

The current display implementation refreshes the e-paper display as a whole. It does not independently refresh individual screen regions.

## Troubleshooting

### Weather Data Is Missing

- Verify that `weather.forecast_home` exists.
- Check that the weather integration is returning forecast data.
- Check the state and attributes of `sensor.weather_forecast_hourly` and `sensor.weather_forecast_daily`.
- Verify that the display forecast sensors contain the expected pipe-separated values.

### Sunrise or Sunset Is Unavailable

- Verify that the built-in Sun integration is available.
- Check the `next_rising` and `next_setting` attributes of `sun.sun`.

### Moon Phase Is Missing

- Verify that the built-in Moon integration provides `sensor.moon_phase`.
- Check that the entity is available in Home Assistant.

### Moonrise or Moonset Is Missing

- Verify that the Lunar Phase integration is installed and working.
- Check the source entities `sensor.winsen_luhe_moon_rise` and `sensor.winsen_luhe_moon_set`.
- Confirm that the source entity IDs match the template configuration.

### Rainfall Values Are Incorrect

- Check the raw source entities for the Zeyn and Plantage rain gauges.
- Confirm that each tipping event corresponds to `0.5 mm`.
- For Plantage, verify that TTN uplinks are reaching Node-RED and that the measurements are being stored in InfluxDB.
- Check the Node-RED logs for invalid uplinks or InfluxDB query errors.
- Verify that the MQTT topics `precipitation/today` and `precipitation/yesterday` receive retained numeric values.
- Remember that the Plantage rainfall day resets at 8:00 AM, not at midnight.

### Birthday Sensors Are Missing

- Verify that Node-RED is running and that both birthday and discovery flows are deployed.
- Check the MQTT broker connection.
- Confirm that the discovery messages are published to `homeassistant/sensor/.../config`.
- Check that the corresponding state topics (`Display/A1`, `Display/B1`, and so on) are receiving messages.
- Verify that MQTT Discovery is enabled and that the entities have been registered in Home Assistant.

## Related Documentation

- [Node-RED Configuration](node-red.md)
- [Project README](../README.md)
