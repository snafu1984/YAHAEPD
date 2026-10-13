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
- An MQTT broker for the birthday calendar
- Node-RED for the birthday calendar and MQTT Discovery setup

The weather, Sun, Moon, and rainfall sensors can be configured independently of the birthday calendar.

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

Repository: https://github.com/ngocjohn/lunar-phase

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

## Rainfall Sensors

The configuration supports two rainfall sources: Zeyn and the Plantage rain gauge.

### Zeyn Rainfall

The template sensor `sensor.zeyn_rain_interval` calculates the rainfall amount since the last uplink.

The source entity is:

`sensor.zeyn_regen_rainlevel`

The raw value is multiplied by `0.5` to convert it to millimeters. Invalid source values result in `0`.

The resulting sensor is used by four utility meters:

- `sensor.zeyn_rain_today`
- `sensor.zeyn_rain_this_week`
- `sensor.zeyn_rain_this_month`
- `sensor.zeyn_rain_this_year`

These utility meters use `delta_values: true` to accumulate the reported interval amounts over their respective periods.

### Plantage Rainfall

The Plantage rain gauge uses the RANLWE01 sensor.

The raw rain-level value represents the number of tipping events since the last uplink. Each tipping event corresponds to `0.5 mm` of rainfall.

The template sensor `sensor.plantage_niederschlag_intervall` calculates the rainfall amount for the reporting interval by multiplying the raw value by `0.5`.

The sensor is updated when a `state_changed` event occurs for:

`sensor.plantage_niederschlag_rainlevel`

The new state from the event is used for the calculation. Invalid values result in `0`.

### Daily Rainfall Total

The utility meter `sensor.plantage_niederschlag_heute` accumulates the interval values from `sensor.plantage_niederschlag_intervall`.

The rainfall day runs from **8:00 AM to 8:00 AM the following day**, using:

```yaml
cycle: daily
offset: "08:00:00"
delta_values: true
always_available: true
```

The template sensor `sensor.plantage_niederschlag_letzter_tag` reads the `last_period` attribute of `sensor.plantage_niederschlag_heute` to provide the rainfall total for the previous completed period.

### Plantage Temperature and Battery Voltage

Two additional template sensors convert the raw values from the Plantage device:

| Template sensor | Source entity | Conversion | Unit |
|---|---|---|---|
| `sensor.plantage_niederschlag_temperatur` | `sensor.plantage_niederschlag_temperature` | Raw value ÷ 10 | °C |
| `sensor.plantage_niederschlag_batteriespannung` | `sensor.plantage_niederschlag_bat_voltage` | Raw value ÷ 10 | V |

The temperature and battery voltage templates return no value when their source values are not numeric.

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

This behavior has been confirmed in regular use.

The MQTT messages are retained so that the latest values remain available to subscribers.

### MQTT Discovery

The separate discovery flow registers the MQTT sensors automatically in Home Assistant.

For example, the discovery configuration for `Display/A1` uses the topic:

`homeassistant/sensor/display_a1/config`

and the state topic:

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

The display uses the first four rows (A1–D4). The fifth row remains available in Home Assistant but is not displayed by the current ESPHome layout.

## ESPHome Integration

ESPHome reads the Home Assistant entities and uses them to render the display.

The Home Assistant entities referenced by the ESPHome configuration must exist and have valid states. Check the entity IDs if values are missing from the display.

The current display implementation refreshes the e-paper display as a whole. It does not independently refresh individual screen regions.

## Troubleshooting

### Weather data is missing

- Verify that `weather.forecast_home` exists.
- Check that the weather integration is returning forecast data.
- Check the state and attributes of `sensor.weather_forecast_hourly` and `sensor.weather_forecast_daily`.
- Verify that the display forecast sensors contain the expected pipe-separated values.

### Sunrise or sunset is unavailable

- Verify that the built-in Sun integration is available.
- Check the `next_rising` and `next_setting` attributes of `sun.sun`.

### Moon phase is missing

- Verify that the built-in Moon integration provides `sensor.moon_phase`.
- Check that the entity is available in Home Assistant.

### Moonrise or moonset is missing

- Verify that the Lunar Phase integration is installed and working.
- Check the source entities `sensor.winsen_luhe_moon_rise` and `sensor.winsen_luhe_moon_set`.
- Confirm that the source entity IDs match the template configuration.

### Rainfall values are incorrect

- Check the raw source entities for the Zeyn and Plantage rain gauges.
- Confirm that the raw rainfall values represent tipping events and that each event corresponds to `0.5 mm`.
- For Plantage, verify that the source entity changes state when a new report arrives.
- Remember that the Plantage daily total resets at 8:00 AM rather than midnight.

### Birthday sensors are missing

- Verify that Node-RED is running and that both birthday and discovery flows are deployed.
- Check the MQTT broker connection.
- Confirm that the discovery messages are published to `homeassistant/sensor/.../config`.
- Check that the corresponding state topics (`Display/A1`, `Display/B1`, and so on) are receiving messages.
- Verify that MQTT Discovery is enabled and that the entities have been registered in Home Assistant.

## Related Documentation

- [Node-RED Configuration](node-red.md)
- [Project README](../README.md)
