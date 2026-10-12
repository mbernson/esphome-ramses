# ESPHome RAMSES II Component Suite

[![CI](https://github.com/wlcrs/esphome-ramses/actions/workflows/ci.yml/badge.svg)](https://github.com/wlcrs/esphome-ramses/actions/workflows/ci.yml)
[![ESPHome](https://img.shields.io/badge/ESPHome-External%20Component-blue.svg)](https://esphome.io)
[![Target](https://img.shields.io/badge/Target-ESP32%20%7C%20ESP32--C6-green.svg)](https://espressif.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Native Honeywell Evohome & RAMSES II RF (868.3 MHz) integration for [ESPHome](https://esphome.io) running on **ESP32** and **ESP32-C6** microcontrollers paired with a Texas Instruments **CC1101** transceiver.

This component suite brings complete, standalone RAMSES II device support directly into ESPHome. Your heating, ventilation (MVHR), domestic hot water, and sensor devices are exposed as **native ESPHome entities directly to Home Assistant** via the standard ESPHome API—with zero middleware, zero external daemons, and no dependency on custom Home Assistant integrations.

> [!TIP]
> **New to ESPHome or RAMSES II?** Check out the step-by-step [Beginner's Tutorial](TUTORIAL.md) for a walkthrough on discovering your devices, setting up Home Assistant, and pairing ventilation remotes.

---

## Core Components

The suite consists of three modular components:

1. **[`ramses_discovery`](#device-auto-discovery-ramses_discovery)**: An automated discovery engine with an **interactive web dashboard** (`http://<esp-ip>/ramses`). It listens for RF packets and actively probes your network to identify controllers, heating zones, radiator TRVs, ventilation units, and sensors, generating ready-to-use YAML configurations.
2. **[`ramses_devices`](#native-esphome-entities-ramses_devices)**: Native ESPHome entity platforms (`climate`, `fan`, `water_heater`, `sensor`, `binary_sensor`) that communicate directly with discovered devices over Sub-1GHz RF.
3. **`ramses_esp`**: The underlying radio transceiver component that handles CC1101 SPI control, hardware UART bitstream framing, and real-time Manchester encoding/decoding. *(Can also optionally be used as a standalone [HGI80 gateway with `ramses_cc`](#alternative-gateway-mode-ramses_esp-with-ramses_cc)).*

---

## Recommended Workflow

Setting up your RAMSES devices follows a simple two-step process:

```
┌─────────────────────────────────┐       ┌─────────────────────────────────┐
│   Step 1: ramses_discovery      │       │     Step 2: ramses_devices      │
│  Flash discovery firmware       │ ────> │  Paste generated YAML into      │
│  Open http://<esp-ip>/ramses    │       │  production configuration and   │
│  Inspect & copy generated YAML  │       │  flash native ESPHome entities  │
└─────────────────────────────────┘       └─────────────────────────────────┘
```

1. **Run Discovery**: Flash [`esp32-c6-ramses-discovery.yaml`](esp32-c6-ramses-discovery.yaml) to your ESP32. Navigate to `http://<esp-ip>/ramses` in your browser. As heating and ventilation units transmit (or respond to active probes), they populate the live dashboard table.
2. **Deploy Native Entities**: Copy the generated YAML fragment from the discovery dashboard, paste it into your production config (such as [`esp32-c6-ramses-devices.yaml`](esp32-c6-ramses-devices.yaml)), and flash your device. All climate zones, ventilation controls, and sensors appear natively in Home Assistant.

> [!TIP]
> **Zero-Reflash Mode (NVS Storage)**:  
> `ramses_discovery` can also save the configuration with the detected devices directly into **Non-Volatile Storage (NVS)** from the web dashboard, automatically restoring and registering the entities on every boot without needing to flash the device a second time.  
>  
> **The Storage Downside**: Keeping `ramses_discovery` enabled permanently takes up a considerable chunk of flash storage (for the embedded web UI, active probing routines, and dynamic JSON parsers) that cannot be used to add other ESPHome components to the ESP32. Exporting the generated YAML to a clean `ramses_devices` firmware frees this storage for additional components, sensors, or automations.

---

## Device Auto-Discovery (`ramses_discovery`)

The `ramses_discovery` component acts as an automated network inspector. It continuously analyzes demodulated RAMSES frames and proactively interrogates controllers and remotes to identify the topology of your home.

### Key Features
- **Live Web Dashboard**: An embedded, responsive web UI hosted directly on the ESP at `http://<esp-ip>/ramses` (or `/discovery`).
- **Active Probing**: Periodically transmits discovery queries (`0x0008`, `0x10E0`, `0x2309`, etc.) to prompt controllers and ventilation units to announce their zone schedules, zone names, and device parameters.
- **YAML Generation**: Formats all identified devices and zones into complete, copy-paste `ramses_devices` YAML blocks.
- **Persistent Storage (NVS) & Zero-Reflash**: Saves the detected configuration into NVS so entities dynamically register on boot without re-flashing.
- **REST API Endpoints**:
  - `GET /ramses`: Interactive discovery dashboard.
  - `GET /ramses/devices.json`: Discovered devices, zones, and telemetry in JSON format.
  - `POST /ramses/probe`: Triggers an immediate active probe sweep.
  - `POST /ramses/api/config`: Retrieves the generated ESPHome YAML configuration or saves it to NVS.
  - `POST /ramses/api/reset`: Clears the discovery cache in NVS.

### NVS Configuration & Storage Overhead

`ramses_discovery` includes two ways to operate your devices:

1. **Static YAML Workflow (Recommended)**: Use `ramses_discovery` to find your device addresses, export the YAML snippet, and flash a static `ramses_devices` configuration. This removes the discovery web UI and probing logic, minimizing memory footprint and maximizing available flash storage for other ESPHome components.
2. **Dynamic NVS Storage**: Save the discovered configuration directly to Non-Volatile Storage (NVS) from the web dashboard. `ramses_discovery` will automatically re-instantiate all entities each time the device boots, **making it unnecessary to flash the device again**.

> [!WARNING]
> **Flash Storage Cost**: The major downside of keeping `ramses_discovery` permanently on your device is that the discovery engine—along with its embedded web interface, active probing state machines, and dynamic JSON allocators—takes up a substantial chunk of flash storage. That memory cannot be used to add other ESPHome components (such as Bluetooth Proxies, display drivers, or additional sensors) to the ESP32.

### Discovery Configuration Example

```yaml
external_components:
  - source: github://wlcrs/esphome-ramses
    components: [ ramses_esp, ramses_devices, ramses_discovery ]

esp32:
  board: esp32-c6-devkitc-1
  framework:
    type: esp-idf

# Enable ESPHome Web Server for the /ramses dashboard
web_server:
  port: 80

ramses_esp:
  id: ramses_hub
  cs_pin: GPIO18
  sck_pin: GPIO6
  mosi_pin: GPIO7
  miso_pin: GPIO2
  gdo0_pin: GPIO15
  gdo2_pin: GPIO14

ramses_discovery:
  ramses_esp_id: ramses_hub
  active_probing: true       # Proactively queries device parameters (default: true)
  probing_interval: 30s      # Probing frequency (default: 30s)
```

See [`esp32-c6-ramses-discovery.yaml`](esp32-c6-ramses-discovery.yaml) for a complete, ready-to-flash discovery configuration with WiFi captive portal and activity LED pulsing.

### Configuration Reference: `ramses_discovery`

| Option | Type | Default | Description |
|---|---|---|---|
| **`ramses_esp_id`** | **Required**, ID | — | ID of the `ramses_esp` physical transceiver hub. |
| **`active_probing`** | Optional, Boolean | `true` | Enables active query frames to request zone names, schedules, and ventilation state. |
| **`probing_interval`** | Optional, Time | `30s` | Interval between active probing sweeps. |
| **`web_server_base_id`** | Optional, ID | auto | Explicit ID of the `web_server_base` component if multiple exist. |

---

## Native ESPHome Entities (`ramses_devices`)

Once device addresses and zone mappings are known, `ramses_devices` provides native ESPHome platforms for them. These entities report their states and accept commands locally on the ESP32, integrating seamlessly with Home Assistant via the ESPHome Native API.

### Supported Platforms

#### 1. Climate (`climate.ramses_devices`)
Controls Honeywell Evohome heating zones and thermostats (e.g. Round RF).
- Supports target temperature adjustment, current temperature display, and preset modes.
- Each climate entity corresponds to one zone on the controller. Multi-zone systems simply define multiple climate entities sharing the same `ramses_esp_id`.

```yaml
climate:
  - platform: ramses_devices
    ramses_esp_id: ramses_hub
    name: "Living Room Heating"
    controller_address: "01:145678"
    zone_index: 0
    zone_name: "Living Room"
    device_id: ramses_01_145678
```

#### 2. Ventilation Fan (`fan.ramses_devices`)
Controls Mechanical Ventilation with Heat Recovery (MVHR) units and extractor fans.
- Supports vendor-specific protocol schemes: `orcon`, `vasco`, `itho`, `zehnder`, `hopper` (or `generic`).
- Supports speed presets, timed boost modes, and automated remote pairing.
- An optional `pairing_button` automates RF binding between the ESP32 and the ventilation unit.

```yaml
fan:
  - platform: ramses_devices
    ramses_esp_id: ramses_hub
    name: "MVHR Ventilation Fan"
    device_address: "32:155617"
    scheme: orcon
    fake_remote_address: "29:123456" # Optional virtual remote address for RF pairing
    pairing_button:
      name: "Pair Ventilation Remote"
    device_id: ramses_32_155617
```

#### 3. Water Heater (`water_heater.ramses_devices`)
Controls Domestic Hot Water (DHW) cylinders linked to an Evohome controller.

```yaml
water_heater:
  - platform: ramses_devices
    ramses_esp_id: ramses_hub
    name: "Domestic Hot Water"
    controller_address: "01:145678"
    device_id: ramses_01_145678
```

#### 4. Sensors (`sensor.ramses_devices`)
Monitors temperatures, demands, ventilation telemetry, and OpenTherm boiler data.

| `type` | Description | Typical Source Device |
|---|---|---|
| `zone_temperature` | Current measured temperature of a zone (`zone_index`) | Evohome Controller / Thermostat |
| `zone_setpoint` | Target setpoint temperature of a zone (`zone_index`) | Evohome Controller |
| `outdoor_temperature` | External ambient outdoor temperature | Outdoor Sensor / Weather Bridge |
| `heat_demand` | Radiator / zone actuator heat demand percentage (0–100%) | Radiator TRV (`04:xxxxxx`) |
| `relay_demand` | BDR91 wireless relay demand percentage (filtered by `relay_index`) | BDR91 Relay (`13:xxxxxx`) |
| `co2` | Indoor carbon dioxide level (ppm) | Ventilation Sensor (`32:xxxxxx`) |
| `indoor_humidity` | Relative humidity percentage (%) | Ventilation Sensor / Thermostat |
| `outdoor_humidity` | External relative humidity percentage (%) | Outdoor Sensor |
| `air_quality_temperature` | Indoor temperature reading from ventilation sensor | MVHR Unit / Sensor |
| `bypass_position` | Heat exchanger bypass damper position (%) | MVHR Unit |
| `filter_remaining_days` | Estimated days before ventilation filter replacement | MVHR Unit |
| `filter_lifetime_days` | Total rated filter lifetime in days | MVHR Unit |
| `filter_remaining_percent` | Remaining ventilation filter health (%) | MVHR Unit |
| `opentherm_modulation` | Boiler burner modulation percentage (0–100%) | OpenTherm Bridge (`10:xxxxxx`) |
| `flow_temperature` | Central heating water flow temperature (°C) | OpenTherm Bridge |
| `return_temperature` | Central heating return water temperature (°C) | OpenTherm Bridge |
| `battery_level` | Battery level percentage (%) | Radiator TRVs (`04:xxxxxx`) |
| `supply_temperature` | MVHR fresh supply air temperature (°C) | MVHR Unit |
| `exhaust_temperature` | MVHR exhaust air temperature (°C) | MVHR Unit |
| `supply_fan_speed` | Measured supply fan RPM / speed | MVHR Unit |
| `exhaust_fan_speed` | Measured exhaust fan RPM / speed | MVHR Unit |
| `remaining_mins` | Remaining duration on temporary boost or ventilation timer | MVHR Unit / Controller |
| `actuator_modulation` | Underfloor heating / valve actuator modulation (%) | UFH Controller |
| `ufh_min_temp` / `ufh_max_temp` | Underfloor heating flow water temperature limits | UFH Controller |
| `spider_temperature` | Temperature reading from Itho Spider units | Spider Thermostat |
| `fault_code` | Active equipment diagnostic error or fault code | Boiler / Controller |

#### 5. Binary Sensors (`binary_sensor.ramses_devices`)
Monitors binary equipment states and alerts.

| `type` | Description |
|---|---|
| `filter_alarm` | True when the ventilation filter requires cleaning or replacement. |
| `flame_active` | True when the boiler burner flame is active. |
| `fault_alarm` | True if a general fault condition is present on the controller/appliance. |
| `window_open` | True when a radiator TRV detects an open window. |
| `bypass_active` | True when the ventilation summer bypass is open. |
| `battery_low` | True when a battery-powered device (TRV/sensor) indicates low battery. |
| `actuator_relay` | True when an actuator relay contact is energized. |

### Complete `ramses_devices` Example

```yaml
external_components:
  - source: github://wlcrs/esphome-ramses
    components: [ ramses_esp, ramses_devices ]

ramses_esp:
  id: ramses_hub
  cs_pin: GPIO18
  sck_pin: GPIO6
  mosi_pin: GPIO7
  miso_pin: GPIO2
  gdo0_pin: GPIO15
  gdo2_pin: GPIO14

climate:
  - platform: ramses_devices
    ramses_esp_id: ramses_hub
    name: "Living Room Heating"
    controller_address: "01:145678"
    zone_index: 0
    zone_name: "Living Room"

fan:
  - platform: ramses_devices
    ramses_esp_id: ramses_hub
    name: "Ventilation Unit"
    device_address: "32:155617"
    scheme: orcon

water_heater:
  - platform: ramses_devices
    ramses_esp_id: ramses_hub
    name: "Domestic Hot Water"
    controller_address: "01:145678"

sensor:
  - platform: ramses_devices
    ramses_esp_id: ramses_hub
    type: zone_temperature
    name: "Living Room Temperature"
    device_address: "01:145678"
    zone_index: 0

  - platform: ramses_devices
    ramses_esp_id: ramses_hub
    type: heat_demand
    name: "Radiator Heat Demand"
    device_address: "04:089123"

  - platform: ramses_devices
    ramses_esp_id: ramses_hub
    type: opentherm_modulation
    name: "Boiler Modulation"
    device_address: "10:045678"

binary_sensor:
  - platform: ramses_devices
    ramses_esp_id: ramses_hub
    type: flame_active
    name: "Boiler Flame"
    device_address: "10:045678"

  - platform: ramses_devices
    ramses_esp_id: ramses_hub
    type: filter_alarm
    name: "MVHR Filter Dirty"
    device_address: "32:155617"
```

For full multi-device setups, inspect [`esp32-c6-ramses-devices.yaml`](esp32-c6-ramses-devices.yaml) and [`esp-c6-ramses-devices.yaml`](esp-c6-ramses-devices.yaml).

---

## Hardware & Pinout

The CC1101 connects via SPI (for register configuration and transmit control) and routes its `GDO2` pin to an ESP32 hardware UART RX pin (for continuous demodulated bitstream reception), and `GDO0` for transmission.

### Typical ESP32-C6 Pinout
| CC1101 Pin | ESP32-C6 Pin | Function |
|---|---|---|
| **CSN (CS)** | `GPIO 18` | SPI Chip Select |
| **SCK (CLK)** | `GPIO 6` | SPI Clock |
| **MOSI (SI)** | `GPIO 7` | SPI MOSI |
| **MISO (SO)** | `GPIO 2` | SPI MISO |
| **GDO0** | `GPIO 15` | CC1101 GDO0 (TX Packet Data) |
| **GDO2** | `GPIO 14` | CC1101 GDO2 (Demodulated RX Bitstream to UART RX) |

*(Pins are fully configurable to match your hardware layout).*

Compatible ready-made hardware includes custom ESP32-C6 Sub-1GHz boards by IMMRMKW ([Elecram Shop](https://elecram.com/products/esp32-c6-wifi-zigbee-to-855-925mhz-wireless?variant=52605637099853) / [Tindie](https://www.tindie.com/products/elecram/esp32-c6-wifizigbee-to-855-925mhz-wireless/)).

---

## Advanced: Radio Time-Sharing (`cc1101_multiplexer`)

If you have a **single CC1101 transceiver** on your board and want to use it **both** for continuous RAMSES II monitoring (868.3 MHz, 38.4 kbps Manchester) **and** to transmit commands to or receive replies from another 868 MHz device (such as an extractor hood, roller blinds, or custom FSK remote using ESPHome's standard `cc1101` component), use `cc1101_multiplexer`.

### How it works:
1. `ramses_esp` monitors RAMSES II traffic >99% of the time.
2. When you send a command to your secondary device, `cc1101_multiplexer` temporarily pauses RAMSES listening, switches the CC1101 to the secondary configuration (e.g. 64 kbps FSK with custom sync word and CRC), transmits the packet, and keeps the radio in RX mode for a brief configurable window (`rx_window: 75ms`) to catch ACKs.
3. Once the window expires, RAMSES monitoring resumes automatically without impacting heating or ventilation control.

```yaml
external_components:
  - source: github://wlcrs/esphome-ramses
    components: [ ramses_esp, cc1101_multiplexer ]

cc1101:
  id: hood_radio
  cs_pin: GPIO18
  gdo0_pin: GPIO15
  frequency: 868.3MHz
  modulation_type: 2-FSK
  symbol_rate: 64000
  packet_mode: true
  crc_enable: true

ramses_esp:
  id: ramses_hub
  cs_pin: GPIO18
  sck_pin: GPIO6
  mosi_pin: GPIO7
  miso_pin: GPIO2
  gdo0_pin: GPIO15
  gdo2_pin: GPIO14

cc1101_multiplexer:
  id: radio_mux
  ramses_id: ramses_hub
  cc1101_id: hood_radio
  rx_window: 75ms

# Transmit action in automations or buttons
button:
  - platform: template
    name: "Hood Fan Up"
    on_press:
      - cc1101_multiplexer.send_packet:
          id: radio_mux
          data: [ 0x0E, 0x46, 0x12, 0x34, 0x56, 0x78, 0xFF, 0xFF, 0xFF, 0xFF, 0x01, 0x00, 0x27, 0x5A, 0x02 ]
```

---

## Alternative Gateway Mode: `ramses_esp` with `ramses_cc`

If you already use Home Assistant's custom [`ramses_cc`](https://github.com/zxdavids/ramses_cc) integration (powered by [`ramses_rf`](https://github.com/ramses-rf/ramses_rf)) and prefer running the protocol engine entirely within Home Assistant, `ramses_esp` can act as an **HGI80-compatible RF Gateway**.

In this mode, the ESP acts as a bridge, exposing a bidirectional ASCII stream server over a TCP socket or MQTT.

### 1. TCP Socket Mode (Default)

Configure `ramses_esp` with a TCP server port (default: `6638`):

```yaml
external_components:
  - source: github://wlcrs/esphome-ramses
    components: [ ramses_esp ]

ramses_esp:
  id: ramses_hub
  cs_pin: GPIO18
  sck_pin: GPIO6
  mosi_pin: GPIO7
  miso_pin: GPIO2
  gdo0_pin: GPIO15
  gdo2_pin: GPIO14
  port: 6638 # TCP port for HGI80 stream server
```

Then configure `ramses_cc` in Home Assistant's `configuration.yaml` using the socket URI:

```yaml
ramses_cc:
  serial_port: "socket://192.168.1.100:6638"
  packet_log: /config/ramses_packets.log
  restore_cache: true
```

### 2. MQTT Gateway Mode (`ramses-mqtt` Parity)

If you prefer communicating with `ramses_cc` / `ramses_rf` over MQTT, ESPHome achieves **100% feature parity** with the standalone `ramses-mqtt` firmware:
- Availability topic (`online`/`offline` on `<root>/<device_id>`)
- RX Packets published to `<root>/<device_id>/rx` as JSON `{"msg": "<hgi80_frame>", "ts": "<timestamp>"}`
- TX Packets subscribed on `<root>/<device_id>/tx`
- Handshake / version query (`!V` on `<root>/<device_id>/cmd/cmd`)

See [`esp32-c6-ramses-mqtt.yaml`](esp32-c6-ramses-mqtt.yaml) for a complete MQTT gateway configuration.

### Configuration Reference: `ramses_esp`

| Option | Type | Default | Description |
|---|---|---|---|
| **`cs_pin`** | **Required**, Pin | — | SPI Chip Select pin for CC1101. |
| **`sck_pin`** | **Required**, Pin | — | SPI Clock pin. |
| **`mosi_pin`** | **Required**, Pin | — | SPI MOSI pin. |
| **`miso_pin`** | **Required**, Pin | — | SPI MISO pin. |
| **`gdo0_pin`** | **Required**, Pin | — | CC1101 GDO0 pin for packet TX. |
| **`gdo2_pin`** | **Required**, Pin | — | CC1101 GDO2 pin connected to ESP32 UART RX for continuous bitstream reception. |
| **`uart_num`** | Optional, Integer | `1` | ESP32 hardware UART peripheral number (0–2). |
| **`port`** | Optional, Integer | `6638` | TCP port for the HGI80 ASCII stream server (used by `ramses_cc`). |
| **`on_message`** | Optional, Automation | — | Automation trigger fired whenever a valid RAMSES frame is received (passes the HGI80 string `x`). |

---

## Testing & Verification

This project includes **native host C++ unit tests** for protocol logic and **ESPHome compile verification**:

### 1. Native C++ Unit Tests (Host Machine)
Tests the Manchester encoder/decoder, RAMSES address parsing, HGI80 parsing/formatting, checksum calculation, and raw radio frame generation on Linux/macOS without hardware:

```bash
cmake -B build -S tests
cmake --build build
ctest --test-dir build --output-on-failure
```

### 2. ESPHome Compilation Test
Validates YAML schemas and compiles the firmware using the ESP-IDF toolchain:

```bash
esphome compile esp32-c6-ramses-devices.yaml
```

### 3. Continuous Integration
All PRs and commits are automatically verified via GitHub Actions against:
- The native C++ unit test suite.
- ESPHome YAML validation and ESP32-C6 ESP-IDF compilation.

---

## Origins & Credits

This project ports and builds upon the foundational work of several community members and projects:

- **Peter Price ([IndaloTech](https://github.com/IndaloTech/ramses_esp))**: Creator of the original standalone `ramses_esp` ESP-IDF firmware.
- **[IMMRMKW](https://github.com/IMMRMKW/ramses_esp/)**: Added ESP32-C6 single-core support, optimized CC1101 interrupt timing/FIFO thresholds, and created custom ESP32-C6 wireless boards ([Shop](https://elecram.com/products/esp32-c6-wifi-zigbee-to-855-925mhz-wireless?variant=52605637099853) / [Tindie](https://www.tindie.com/products/elecram/esp32-c6-wifizigbee-to-855-925mhz-wireless/)).
- **David Bonnes ([zxdavids](https://github.com/zxdavids/ramses_cc)) & the [ramses-rf](https://github.com/ramses-rf/ramses_rf) team**: Creators of the RAMSES RF protocol decoding library and Home Assistant integration.
- **The [ESPHome](https://esphome.io) team**: For the embedded framework and core abstractions.

---

## Contributing

Contributions, bug reports, and enhancements are welcome!

1. Fork the repository and create a feature branch (`git checkout -b feature/my-feature`).
2. Implement your changes and ensure code style is clean (`clang-format` for C++, `ruff` for Python). Run Python and C++ checks before submitting.
3. Run the unit tests (`ctest --test-dir build`) and add tests for any new protocol logic.
4. Verify compilation with `esphome compile esp32-c6-ramses-devices.yaml`.
5. Open a Pull Request.

---

## License

This project is licensed under the [MIT License](LICENSE).  
Portions derived from `ramses_esp` are copyright &copy; 2023–2025 Peter Price & IMMRMKW.
