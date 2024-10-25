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

### Known Limitations

- **Inverter Specific**: The flow is specifically designed for a Deye inverter using Sunsynk integration. If you have a different inverter, adjustments may be required.
- **Custom Setup**: Some parts of the flow are specific to the user’s setup (e.g., entity IDs, automation settings) and will need to be adapted for other environments.

### Contributions

Feel free to submit issues or pull requests to help improve this integration. We welcome feedback and contributions from the community!
