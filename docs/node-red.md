# Node-RED Configuration

YAHAEPD uses Node-RED to retrieve upcoming birthdays from an iCal calendar and publish the data to Home Assistant via MQTT. A separate discovery flow automatically registers the MQTT sensors in Home Assistant.

## Requirements

- Node-RED
- An MQTT broker accessible by both Node-RED and Home Assistant
- An iCal calendar containing the birthday events
- The following additional Node-RED palette modules:
  - [`node-red-contrib-ical-events`](https://flows.nodered.org/node/node-red-contrib-ical-events) (version 2.2.5 used during development) — retrieves events from an iCal calendar.
  - [`node-red-contrib-simpletime`](https://flows.nodered.org/node/node-red-contrib-simpletime) (version 3.1.0) — provides additional time and date handling nodes used by the flow.

Install both palette modules before importing and deploying the flows.

## Birthday Calendar

The birthday calendar flow retrieves upcoming events from an iCal calendar and checks for updated calendar data every three hours.

The flow processes five birthday entries and publishes the following information for each entry:

| Field | Description | MQTT topics |
|---|---|---|
| Weekday | Abbreviated weekday (`Mo`, `Di`, `Mi`, etc.) | `Display/A1`–`Display/A5` |
| Date | Date in `DD.MM.` format | `Display/B1`–`Display/B5` |
| Event | Name or summary of the event | `Display/C1`–`Display/C5` |
| Age | Age the person will turn on their birthday in the current year, if the birth year is available | `Display/D1`–`Display/D5` |

The birth year is read from the calendar event's comment. The calculated age represents the age the person will turn on their birthday in the current calendar year. If the comment is empty and contains no birth year, the corresponding age field remains empty.

This behavior has been confirmed in regular use.

### Birthday Calendar Flow

The following diagram shows the Node-RED flow used to retrieve birthday events and publish them via MQTT.

![Node-RED birthday calendar flow](images/node-red-birthday-flow.png)

### Separate Processing of Entries

Each of the five entries is processed individually using separate Function nodes for the weekday, date, event name, and age.

This structure is intentional. It makes the flow easier to understand, debug, and customize. Although the processing could be consolidated into fewer nodes, readability and maintainability take priority over reducing the number of nodes.

### Calendar Configuration

Configure the iCal calendar in the `ical-config` node.

- **Calendar URL:** Enter the private iCal URL of your calendar.
- **Time zone:** Use `Europe/Berlin` for German local time, including automatic daylight saving time changes.
- **Preview period:** The current configuration retrieves events up to 90 days in advance.
- **Refresh interval:** The flow checks for updated calendar data every three hours.

Keep the private iCal URL confidential. It may grant access to calendar information.

### MQTT Topics

The birthday calendar flow publishes the data to the MQTT topics listed above.

The MQTT output nodes use QoS 2 and retained messages. This allows the most recently published values to remain available on the broker until they are replaced by newer values.

Configure the MQTT broker connection in Node-RED to match the broker used by Home Assistant.

## Home Assistant MQTT Discovery

The discovery flow automatically registers the display sensors in Home Assistant using MQTT Discovery.

An Inject node triggers the discovery function once, two seconds after Node-RED starts. The Function node generates 20 discovery messages using nested loops:

- Rows: `1` to `5`
- Columns: `A`, `B`, `C`, and `D`
- Total: 20 sensors, from `Display A1` to `Display D5`

### MQTT Discovery Flow

The following diagram shows the Node-RED flow that registers the display sensors in Home Assistant using MQTT Discovery.

![Node-RED MQTT discovery flow](images/discovery-flow.png)

Each sensor is registered using a discovery topic in the following format:

`homeassistant/sensor/display_a1/config`

The corresponding state topic is:

`Display/A1`

The discovery payload defines the sensor name, unique ID, icon, and shared device information. All sensors are grouped under the Home Assistant device **Display Matrix**, with the model `5x4 MQTT Display` and manufacturer `NodeRed`.

Discovery messages are published with QoS 1 and retained, allowing Home Assistant to rediscover the sensors after a restart.

### Importing and Configuring the Flows

1. Import the required Node-RED flow JSON files.
2. Configure the iCal URL in the calendar configuration node.
3. Configure the MQTT broker address and credentials, if required.
4. Deploy the flows.
5. Verify that birthday values arrive on the expected MQTT topics and that Home Assistant discovers the sensors.

The calendar processing and discovery logic are separate flows and can be adapted independently, provided that the MQTT topic names remain consistent with the Home Assistant and ESPHome configurations.

## Relationship to the ESPHome Display

The birthday calendar flow publishes five rows (`A1`–`D5`), while the current ESPHome display configuration uses only the first four rows (`A1`–`D4`).

The fifth row remains available as an MQTT sensor in Home Assistant but is not currently displayed on the e-paper screen.

The age field may be empty when the event comment does not contain a birth year. This does not prevent the weekday, date, or event name from being published.

## Flow Files

The `node-red` directory contains the following flow exports:

- [`birthday-calendar.json`](../node-red/birthday-calendar.json) — Retrieves upcoming birthdays from the iCal calendar and publishes weekday, date, event name, and age via MQTT.
- [`discovery.json`](../node-red/discovery.json) — Registers the 20 display sensors (`Display A1` to `Display D5`) in Home Assistant using MQTT Discovery.

Import the required JSON files into Node-RED and configure the iCal calendar URL and MQTT broker connection before deploying the flows.

## Customization

The flows are provided as a working example. Users can adjust the calendar settings, MQTT connection, and discovery metadata to suit their own installation.

The five-entry processing structure is deliberately kept explicit for readability. Users who prefer a more compact implementation can consolidate the repeated Function nodes themselves.
