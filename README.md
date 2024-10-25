# Qilowatt Node-RED Integration for Home Assistant

This repository contains a Node-RED flow designed to integrate Home Assistant with the Qilowatt platform for managing solar inverters, batteries, and energy usage. This specific flow works with a Deye inverter and uses the Sunsynk Home Assistant integration to fetch relevant data. The flow is customized to handle different modes of operation, including normal operation and Fusebox (Manual Frequency Restoration Reserve) mode.


## Setup Overview

This Node-RED flow communicates with the Qilowatt platform via MQTT, providing real-time sensor data and responding to control commands. The flow consists of various components that handle sensor data collection, MQTT payload construction, and command processing. The following sections describe the major components of the flow.

### Key Features

- **MQTT Communication**: Sends energy and system status data to Qilowatt via MQTT and listens for incoming control commands.
- **Home Assistant Integration**: Retrieves real-time data from Home Assistant entities for grid power, battery state of charge (SOC), current, voltage, and more.
- **Customizable WORKMODE**: Supports dynamic handling of `WORKMODE` based on the current system mode, whether it’s running on a timer (normal operation) or Fusebox (buy/sell) commands.
- **Deye Inverter Specific**: The flow is tailored to a Deye inverter using the Sunsynk integration but can be adapted for other setups.

### Prerequisites

- **Node-RED**: Make sure Node-RED is installed and running on your Home Assistant or external server.
- **MQTT Broker**: A running MQTT broker (e.g., Mosquitto). The flow assumes connection to a broker, such as `test-mqtt.qilowatt.it`.
- **Home Assistant**: The flow assumes integration with Home Assistant to fetch sensor data.

### Entities Used

The flow uses the following Home Assistant entities (specific to the Deye inverter setup):
- **Power sensors**: `sensor.ss_grid_l1_power`, `sensor.ss_grid_l2_power`, `sensor.ss_grid_l3_power`
- **Current and Voltage sensors**: `sensor.ss_grid_l1_current`, `sensor.ss_grid_l2_current`, `sensor.ss_grid_l3_current`, `sensor.ss_grid_l1_voltage`, `sensor.ss_grid_l2_voltage`, `sensor.ss_grid_l3_voltage`
- **Battery data**: `sensor.ss_battery_soc`, `sensor.ss_battery_power`, `sensor.ss_battery_voltage`, `sensor.ss_battery_current`, `sensor.ss_battery_temperature`
- **Inverter data**: `sensor.ss_radiator_temperature`
- **PV data**: `sensor.ss_pv1_power`, `sensor.ss_pv2_power`, `sensor.ss_pv1_voltage`, `sensor.ss_pv2_voltage`, `sensor.ss_pv1_current`, `sensor.ss_pv2_current`
- **Control selectors**: `input_select.battery_automation_selector`, `input_select.battery_mode_selector`, `select.ss_grid_peak_shaving`

### Flow Breakdown

#### 1. **MQTT Publish Flow (`/SENSOR` and `/STATE`)**

This section of the flow collects real-time data from Home Assistant entities, formats the data into an MQTT payload, and publishes it to Qilowatt topics.

- **Sensor Data**: Retrieves sensor data (power, voltage, current) from Home Assistant.
- **WORKMODE Handling**: Depending on whether the system is in normal mode or receiving commands from Fusebox, WORKMODE values are adjusted accordingly.
- **Ping Data**: A periodic ping to a public DNS server (e.g., `8.8.8.8`) is included in the payload to monitor network connectivity.

#### 2. **MQTT Listener for Control Commands (`/cmnd/backlog`)**

This part of the flow listens for control commands from Qilowatt. It processes `WORKMODE` commands (either from a timer or Fusebox) and updates the system accordingly.

- **WORKMODE Parsing**: Extracts the `WORKMODE` command from incoming MQTT messages and updates Home Assistant settings based on the control signal.
- **Timer Mode**: Updates the system to operate in regular management mode.
- **Fusebox Mode**: Switches the system to either "buy" or "sell" mode based on signals received from Fusebox, and adjusts battery charge/discharge settings and grid export limits.

#### 3. **Dynamic Handling of Battery Charge Current**

This flow dynamically adjusts the battery charge current based on real-time voltage values from the battery. The default fallback is 52V, ensuring more current is supplied when the actual voltage is unavailable.

### Installation

1. **Clone the Repository**:
   ```bash
   git clone https://github.com/qilowatt/node-red-hass.git
2. **Get MQTT Credentials from Qilowatt**:
   - Contact Qilowatt support to obtain your MQTT username and password.
   - Register your device with Qilowatt to get a unique device ID. This ID will be used in the MQTT topics for communication with Qilowatt. Replace the placeholder device ID in the flow with your own.

3. **Import the Flow**:
   - Open Node-RED.
   - Click on the menu and select “Import”.
   - Paste the JSON content from the provided flow file (`qilowatt_node_red_flow.json`).
   - Deploy the flow.

4. **Configure MQTT Broker**:
   - Set up your MQTT broker details, including the username and password provided by Qilowatt, in the `mqtt-broker` node.
   - Ensure Home Assistant is configured correctly with the necessary entities for data retrieval.

5. **Customize for Your Setup**:
   - If you’re using a different inverter or have a different Home Assistant setup, modify the entity IDs in the flow to match your system.

### Usage

- The flow will automatically start fetching data from Home Assistant and sending it to Qilowatt once deployed.
- Use the inject nodes to test various Fusebox and Timer WORKMODE scenarios.
- The system will respond to WORKMODE commands sent via the `Q/<your-device-id>/cmnd/backlog` topic.

### Payload Structure

#### **Q/<serial_number>/SENSOR** JSON Object Explanation

**Example:**
```json
{
  "PING": {
    "IP": "8.8.8.8",
    "AvgTime": 35,
    "MaxTime": 100,
    "MinTime": 12,
    "Success": 4,
    "Timeout": 0,
    "Reachable": true
  },
  "Time": "2024-10-17T11:00:58",
  "ESP32": {
    "Temperature": 65.0
  },
  "ENERGY": {
    "Power": [3.00, 1.00, 2.00],
    "Today": 0.000,
    "Total": 10.201,
    "Current": [0.013, 0.004, 0.008],
    "Voltage": [239.70, 238.10, 237.80],
    "Frequency": 50.00
  },
  "METRICS": {
    "PvPower": [985, 648],
    "BatterySOC": 24,
    "LoadPower": [346, 2205, 245],
    "BatteryPower": [2773, 0],
    "BatteryCurrent": [51.93, 0],
    "BatteryVoltage": [53.6, 0],
    "InverterStatus": 2,
    "GridExportLimit": 3000,
    "BatteryTemperature": [33],
    "InverterTemperature": 41.5
  },
  "VERSION": {
    "fs": "24.7.1",
    "inverter": "24.10.2",
    "qilowatt": "24.8.1"
  },
  "WORKMODE": {
    "Mode": "buy",
    "BatterySoc": 100,
    "PowerLimit": 3000,
    "ChargeCurrent": 100,
    "DischargeCurrent": 185
  }
}

| **Field**             | **Type**   | **Description**                                                      | **Example**                      | **Optional/Required** |
|-----------------------|------------|----------------------------------------------------------------------|----------------------------------|------------------------|
| **PING**                  | -          | Contains the results of a ping test                                  |                                  | Optional               |
| IP                    | String     | The IP address being pinged                                           | 8.8.8.8                          | Optional               |
| AvgTime               | Integer    | Average ping time in milliseconds                                     | 35                               | Optional               |
| MaxTime               | Integer    | Maximum ping time in milliseconds                                     | 100                              | Optional               |
| MinTime               | Integer    | Minimum ping time in milliseconds                                     | 12                               | Optional               |
| Success               | Integer    | Number of successful ping attempts                                    | 4                                | Optional               |
| Timeout               | Integer    | Number of ping timeouts                                               | 0                                | Optional               |
| Reachable             | Boolean    | Indicates whether the target is reachable (true/false)                | true                             | Optional               |
| Time                  | String     | Timestamp of the report                                               | 2024-10-17T10:59:57              | Required               |
| **ENERGY**                | -          | Contains energy data                                                  |                                  | Required               |
| Power                 | Array      | An array of power values for different phases                         | [2, 0, 0]                        | Required               |
| Current               | Array      | An array of current values for different phases                       | [0.008, 0, 0]                    | Required               |
| Voltage               | Array      | An array of voltage values for different phases                       | [240, 238.7, 238.7]              | Required               |
| Frequency             | Float      | The electrical frequency, typically 50 or 60 Hz                      | 50.00                            | Required               |
| **METRICS**               | -          | Contains various power and battery metrics                            |                                  | Required               |
| PvPower               | Array      | An array of photovoltaic power values                                 | [981, 662, 0, 0]                 | Required               |
| GenPower              | Array      | An array of generator power values                                    | [0, 0, 0]                        | Optional               |
| LoadPower             | Array      | An array of load power values                                         | [333, 187, 240]                  | Required               |
| AlarmCodes            | Array      | An array of alarm codes                                               | [0, 0, 0, 0, 0, 0]               | Optional               |
| BatterySOC            | Array      | An array showing the battery state of charge percentage               | [24]                             | Required               |
| GenVoltage            | Array      | An array of generator voltage values                                  | [12, 8, 3]                       | Optional               |
| LoadCurrent           | Array      | An array of load current values                                       | [1.3, 0.8, 0.9]                  | Required               |
| BatteryPower          | Array      | An array of battery power values                                      | [676, 0]                         | Required               |
| BatteryCurrent        | Array      | An array of battery current values                                    | [13.14, 0]                       | Required               |
| BatteryVoltage        | Array      | An array of battery voltage values                                    | [53.12, 0]                       | Required               |
| InverterStatus        | Integer    | Represents the status of the inverter                                 | 2                                | Required               |
| GridExportLimit       | Integer    | Power limit for grid export                                           | 8000                             | Required               |
| BatteryTemperature    | Array      | An array showing the battery temperature                              | [33]                             | Required               |
| InverterTemperature   | Float      | Inverter temperature in °C                                            | 41.3                             | Required               |
| **VERSION**               | -          | API version information                                               |                                  | Required               |
| api                   | String     | Version of the API                                                    | 1.0                              | Required               |
| **WORKMODE**              | -          | Contains operational data                                             |                                  | Required               |
| Mode                  | String     | Operational mode                                                      | normal                           | Required               |
| BatterySoc            | Integer    | Battery state of charge in percentage                                 | 10                               | Required               |
| PowerLimit            | Integer    | Power limit setting in watts                                          | 8000                             | Required               |
| PeakShaving           | Integer    | Peak shaving setting                                                  | 0                                | Required               |
| ChargeCurrent         | Integer    | Battery charge current in amperes                                     | 100                              | Required               |
| DischargeCurrent      | Integer    | Battery discharge current in amperes                                  | 185                              | Required               |

<br>

#### **Q/<serial_number>/STATE JSON Object Explanation
**Example:**
```json
{
  "Heap": 56,
  "Time": "2024-10-17T11:00:57",
  "Wifi": {
    "RSSI": 92,
    "SSId": "Salutee",
    "Signal": -54,
    "Channel": 7
  },
  "Uptime": "0T03:49:59",
  "LoadAvg": 28,
  "POWER1": "OFF",
  "POWER2": "OFF",
  "POWER3": "OFF"
}
| **Field**             | **Type**   | **Description**                                                      | **Example**                      | **Optional/Required** |
|-----------------------|------------|----------------------------------------------------------------------|----------------------------------|------------------------|
| Time                  | String     | Timestamp when the report was generated                               | "2024-10-17T10:59:57"            | Required               |
| Wifi                  | -          | Contains information about the WiFi connection                        |                                  | Required               |
| AP                    | Integer    | The access point number                                               | 1                                | Optional               |
| Mode                  | String     | The WiFi mode, such as "HT20"                                         | HT20                             | Optional               |
| RSSI                  | Integer    | Received Signal Strength Indicator (RSSI) value indicating signal strength | 92                           | Required               |
| SSId                  | String     | The name of the WiFi network                                          | "Salutee"                        | Required               |
| BSSId                 | String     | The Basic Service Set Identifier (BSSId) of the WiFi network          | "32:87:BA:D7:89:76"              | Required               |
| Signal                | Integer    | Signal strength in dBm                                                | -54                              | Required               |
| Channel               | Integer    | The WiFi channel in use                                               | 7                                | Required               |
| Downtime              | String     | Duration of WiFi downtime                                             | "0T00:00:03"                     | Optional               |
| LinkCount             | Integer    | Number of reconnections or link counts                                | 1                                | Optional               |
| POWER1                | String     | The power state of the first power relay                              | "OFF"                            | Required               |
| LoadAvg               | Integer    | System load average as a percentage                                   | 33                               | Required               |
| UptimeSec             | Integer    | The system uptime in seconds                                          | 13739                            | Required               |

#### **Q/<serial_number>/CMND/BACKLOG JSON Object Explanation
**Example:**
```json
{
  "backlog": {
    "WORKMODE": {
      "Mode": "buy",
      "BatterySoc": 100,
      "PowerLimit": 3000,
      "PeakShaving": 0,
      "ChargeCurrent": 100,
      "DischargeCurrent": 185
    }
  }
}

| **Field**             | **Type**   | **Description**                                                      | **Example**                      | **Optional/Required** |
|-----------------------|------------|----------------------------------------------------------------------|----------------------------------|------------------------|
| **WORKMODE**              | -          | Contains operational settings for managing power modes                |                                  | Required               |
| Mode                  | String     | Operational mode for energy management. Possible values:              | "normal"                         | Required               |
|                       |            | **buy**: Limits grid buy power                                        |                                  |                        |
|                       |            | **sell**: Limits export power                                         |                                  |                        |
|                       |            | **nobattery**: Operates without battery support                       |                                  |                        |
|                       |            | **limitexport**: Limits the amount of power exported to the grid      |                                  |                        |
|                       |            | **normal**: Standard operating mode without specific grid buy/export restrictions |                           |                        |
| BatterySoc            | Integer    | The state of charge of the battery, represented as a percentage. Range: 0-100 | 100                       | Required               |
| PowerLimit            | Integer    | Power limit in watts. It can limit grid buy power or export power     | 3000                             | Required               |
| PeakShaving           | Integer    | Integer value for peak shaving to reduce peak power consumption       | 0                                | Required               |
| ChargeCurrent         | Integer    | Maximum current allowed for charging the battery, in amperes          | 100                              | Required               |
| DischargeCurrent      | Integer    | Maximum current allowed for discharging the battery, in amperes       | 185                              | Required               |

#### **Q/<serial_number>/CMND/STATUS0 JSON Object Explanation
**Example:**
```json
{
  "Status": {
    "DeviceName": "Deye Modbus",
    "FriendlyName": ["Home Assistant"],
    "Topic": "c4ea0c75-c784-4184-9752-6b772e709d4c"
  },
  "StatusPRM": {
    "StartupUTC": "2024-10-18T06:57:49",
    "BootCount": 1043
  },
  "StatusFWR": {
    "Version": "QW-MQTT-API-24.10.01",
    "Hardware": "ESP32-XXX"
  },
  "StatusLOG": {
    "TelePeriod": 10
  },
  "StatusNET": {
    "Hostname": "Home Server",
    "IPAddress": "192.168.1.35",
    "Gateway": "192.168.1.1",
    "Subnetmask": "255.255.255.0",
    "Mac": "E8:9F:6D:06:D2:78"
  },
  "StatusMQT": {
    "MqttHost": "test-mqtt.qilowatt.it",
    "MqttPort": 8883,
    "MqttClient": "QWAPI_XXXXXX",
    "MqttUser": "07ebd0b75956a293aa120a866e917458"
  },
  "StatusTIM": {
    "UTC": "2024-10-25T11:28:52.009Z",
    "Local": "25/10/2024, 14:28:52"
  }
}

| **Field**             | **Type**   | **Description**                                                      | **Example**                      | **Optional/Required** |
|-----------------------|------------|----------------------------------------------------------------------|----------------------------------|------------------------|
| **Status**                | -          | Contains basic device information                                     |                                  | Required               |
| DeviceName            | String     | Name of the device                                                    | "Deye Modbus"                    | Required               |
| FriendlyName          | Array      | Array containing up to 3 friendly names                               | ["My funny device", "", ""]       | Required (first), Optional (others) |
| Topic                 | String     | Unique topic identifier, typically a serial number                    | "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" | Required               |
| **StatusPRM**         | -          | Contains startup parameters                                           |                                  | Required               |
| StartupUTC            | String     | UTC timestamp of the device's startup                                 | "2024-10-18T06:57:49"            | Required               |
| BootCount             | Integer    | Number of times the device has booted                                 | 1043                             | Required               |
| **StatusFWR**         | -          | Firmware and hardware information                                     |                                  | Required               |
| Version               | String     | Firmware version                                                      | "QW-MQTT-API-24.10.01"           | Required               |
| Hardware              | String     | Hardware model                                                        | "ESP32-XXX"                      | Required               |
| **StatusLOG**         | -          | WiFi network and log information                                      |                                  | Required               |
| SSId                  | Array      | Array of up to two SSID names                                         | ["Salutee", ""]                  | Optional               |
| TelePeriod            | Integer    | Interval in seconds for telemetry data reporting                      | 10                               | Required               |
| **StatusNET**         | -          | Network information                                                   |                                  | Required               |
| Hostname              | String     | Device's hostname                                                     | "XXXX"                           | Required               |
| IPAddress             | String     | IP address of the device                                              | "192.168.10.105"                 | Required               |
| Gateway               | String     | Gateway IP address                                                    | "192.168.10.1"                   | Required               |
| Subnetmask            | String     | Subnet mask                                                           | "255.255.255.0"                  | Required               |
| DNSServer1            | String     | Primary DNS server                                                    | "195.80.119.99"                  | Optional               |
| DNSServer2            | String     | Secondary DNS server                                                  | "8.8.8.8"                        | Optional               |
| Mac                   | String     | MAC address of the device                                             | "E8:9F:6D:06:D2:78"              | Required               |
| **StatusMQT**         | -          | MQTT connection information                                           |                                  | Required               |
| MqttHost              | String     | MQTT broker hostname                                                  | "test-mqtt.qilowatt.it"          | Required               |
| MqttPort              | Integer    | MQTT port                                                             | 8883                             | Required               |
| MqttClientMask        | String     | Format mask for generating MQTT client IDs                            | "QWAPI_%06X"                     | Optional               |
| MqttClient            | String     | The actual MQTT client ID in use                                      | "QWAPI_XXXXXX"                   | Required               |
| MqttUser              | String     | MQTT username                                                         | "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" | Required              |
| MqttCount             | Integer    | Number of MQTT messages sent or received                              | 1                                | Optional               |
| **StatusTIM**         | -          | Time-related information                                              |                                  | Required               |
| UTC                   | String     | Current UTC time                                                      | "2024-10-18T07:49:12Z"           | Required               |
| Local                 | String     | Local time on the device                                              | "2024-10-18T10:49:12"            | Required               |
| StartDST              | String     | Start date and time for daylight saving time (DST)                    | "2024-03-31T03:00:00"            | Optional               |
| EndDST                | String     | End date and time for DST                                             | "2024-10-27T04:00:00"            | Optional               |
| Timezone              | Integer    | Timezone offset                                                       | 99                               | Optional               |


### Known Limitations

- **Inverter Specific**: The flow is specifically designed for a Deye inverter using Sunsynk integration. If you have a different inverter, adjustments may be required.
- **Custom Setup**: Some parts of the flow are specific to the user’s setup (e.g., entity IDs, automation settings) and will need to be adapted for other environments.

### Contributions

Feel free to submit issues or pull requests to help improve this integration. We welcome feedback and contributions from the community!
