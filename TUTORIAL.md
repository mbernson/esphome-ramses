# Getting Started with ESPHome RAMSES II: A Beginner's Guide

Welcome! If you recently acquired a ready-made **ESP32 + CC1101 (868.3 MHz)** board (such as the Elecram ESP32-C6 wireless interface board) and want to integrate your heating or ventilation system into **Home Assistant**, this step-by-step guide is for you.

You do **not** need prior knowledge of the RAMSES II wireless protocol or deep expertise with ESPHome. By the end of this tutorial, your heating zones, ventilation fans, and sensors will be neatly organized and controllable in Home Assistant.

---

## Table of Contents
1. [The Basics: What is RAMSES II and ESPHome?](#1-the-basics-what-is-ramses-ii-and-esphome)
2. [What You Need Before Starting](#2-what-you-need-before-starting)
3. [Step 1: Flash the Discovery Engine (`ramses_discovery`)](#step-1-flash-the-discovery-engine-ramses_discovery)
4. [Step 2: Find Your Equipment with the Web Dashboard](#step-2-find-your-equipment-with-the-web-dashboard)
5. [Step 3: Create Your Permanent Setup (`ramses_devices`)](#step-3-create-your-permanent-setup-ramses_devices)
6. [Step 4: View and Control Your Devices in Home Assistant](#step-4-view-and-control-your-devices-in-home-assistant)
7. [Step 5 (Optional): Pair a Virtual Remote to Control Your Ventilation Unit](#step-5-optional-pair-a-virtual-remote-to-control-your-ventilation-unit)
8. [Troubleshooting & FAQs](#troubleshooting--faqs)

---

## 1. The Basics: What is RAMSES II and ESPHome?

### What is RAMSES II?
RAMSES II is a wireless radio protocol operating at **868.3 MHz**. It is used by many European home climate and ventilation devices, including:
- **Honeywell Evohome** multi-zone heating systems, wall controllers, and Round RF thermostats.
- **Radiator Valves (TRVs)** such as the HR92 that regulate radiator water flow in each room.
- **Mechanical Ventilation with Heat Recovery (MVHR)** units from brands like **Orcon, Vasco, Itho Daalderop, and Zehnder**.
- **Boiler switches and relays**, such as the Honeywell BDR91 or OpenTherm bridge.

Every device in your house has a unique 2-digit device type and 6-digit identifier (for example, `01:145678` for a main controller, `04:089123` for a radiator TRV, or `32:155617` for an MVHR ventilation fan).

### What is ESPHome?
[ESPHome](https://esphome.io) is a system that allows you to control ESP32 microcontrollers using simple YAML configuration files. Home Assistant discovers ESPHome devices on your local Wi-Fi network automatically—no cloud required, 100% local.

### How this project works:
1. **`ramses_discovery`** is the discovery engine: it listens to the RAMSES II signal in your home, queries your devices for names and zones, and gives you a visual dashboard in your web browser.
2. **`ramses_devices`** creates clean, official Home Assistant climate controls, fan sliders, and temperature sensors on your ESP32.

---

## 2. What You Need Before Starting

- An **ESP32 or ESP32-C6 board with a CC1101 transceiver** tuned to 868.3 MHz with an antenna connected.
  *(e.g., the pre-assembled Elecram ESP32-C6 wireless board).*
- A standard **USB cable** to plug the ESP32 into your computer or Home Assistant server for initial flashing.
- A running **Home Assistant** installation with the **ESPHome Dashboard** add-on installed.
- Your home 2.4 GHz Wi-Fi network name (SSID) and password.

---

## Step 1: Flash the Discovery Engine (`ramses_discovery`)

The first step is to let the ESP32 scan the airwaves to identify which devices are installed in your home.

### 1. Create a New Configuration in ESPHome
1. In Home Assistant, open the **ESPHome** dashboard from the left sidebar.
2. Click **+ New Device**, give it a name like `ramses-discovery`, and select your board type (for ESP32-C6, select **ESP32-C6**).
3. Open the newly created configuration by clicking **Edit**.
4. Replace its contents with the pre-built discovery template:

```yaml
substitutions:
  name: "ramses-discovery"
  friendly_name: "RAMSES Discovery Engine"

esphome:
  name: ${name}
  friendly_name: ${friendly_name}

esp32:
  board: esp32-c6-devkitc-1
  variant: esp32c6
  framework:
    type: esp-idf

# Pull the RAMSES components directly from GitHub
external_components:
  - source: github://wlcrs/esphome-ramses
    components: [ ramses_esp, ramses_devices, ramses_discovery ]

api:

ota:
  - platform: esphome

logger:
  level: DEBUG

wifi:
  ssid: "YOUR_WIFI_SSID"
  password: "YOUR_WIFI_PASSWORD"

# Web server for the discovery dashboard
web_server:
  port: 80

# Physical CC1101 pinout (pre-configured for Elecram ESP32-C6)
ramses_esp:
  id: ramses_hub
  sck_pin: GPIO6
  miso_pin: GPIO2
  mosi_pin: GPIO7
  cs_pin: GPIO18
  gdo0_pin: GPIO15
  gdo2_pin: GPIO14
  port: 6638

# Discovery component: actively queries zone names and schedules
ramses_discovery:
  ramses_esp_id: ramses_hub
  active_probing: true
  probing_interval: 30s
```

> [!NOTE]
> If your hardware is not an Elecram ESP32-C6 board, update the `board:`, `variant:`, and `GPIO` pin numbers to match your specific ESP32 wiring.

### 2. Install to the ESP32
1. Connect your ESP32 board to your computer using a USB cable.
2. In the ESPHome editor, click **Install** $\rightarrow$ **Plug into this computer** (via Web Serial in Chrome/Edge) or **Plug into the computer running ESPHome Dashboard**.
3. Follow the on-screen prompts to flash the firmware.
4. Once completed, the ESP32 will connect to your Wi-Fi network and display its IP address (e.g., `192.168.1.120`).

---

## Step 2: Find Your Equipment with the Web Dashboard

Once connected to your Wi-Fi, the discovery engine starts listening immediately.

1. Open a web browser and navigate to:
   ```
   http://<YOUR-ESP-IP>/ramses
   ```
   *(For example: `http://192.168.1.120/ramses`)*

2. You will be greeted by the **RAMSES Discovery Dashboard**:
   - The table displays every heating controller, radiator valve, ventilation unit, and sensor that transmits on your network.
   - The active probe engine periodically pings controllers to automatically fetch **friendly zone names** (like *"Living Room"* or *"Master Bedroom"*) and schedules.

3. **Speed up discovery**:
   RAMSES devices sleep to conserve battery. You can prompt them to transmit right away:
   - Walk over to your Evohome controller and adjust the target temperature by half a degree.
   - Turn the wheel on any radiator valve (HR92).
   - Press a speed button on your ventilation remote.
   As soon as a device speaks, it immediately appears in your table.

4. **Copy Your Generated YAML**:
   At the bottom or side of the dashboard, you will see a section titled **Generated YAML Configuration**. Click **Copy Configuration** (or download it). This YAML contains all the device IDs, zone numbers, and sensors that were found.

> [!TIP]
> **Can I keep this discovery firmware permanently?**  
> Yes! In the web dashboard, you can click **Save to NVS**. This saves all detected devices to the ESP32's non-volatile memory so that all entities are created automatically whenever the device boots up—without you needing to flash it again.  
>  
> **However, there is a catch**: The discovery engine, embedded web pages, and probing logic consume a significant chunk of flash storage and memory. If you want a lightweight setup or plan on adding other sensors/components to this ESP32 later, we recommend proceeding to **Step 3** to flash a dedicated, lean configuration.

---

## Step 3: Create Your Permanent Setup (`ramses_devices`)

Now that we know the exact IDs of all your devices, we can set up the lean, permanent firmware.

1. In the ESPHome Dashboard, click **+ New Device** and name it something permanent, like `heating-gateway`.
2. Click **Edit** and paste in your configuration:
   - Keep the core setup (`esp32:`, `wifi:`, `api:`, and `ramses_esp:`).
   - In `external_components:`, you only need `components: [ ramses_esp, ramses_devices ]`.
   - Paste the `climate:`, `fan:`, and `sensor:` blocks that you copied from the discovery dashboard in Step 2.

Here is what a typical finished configuration looks like:

```yaml
substitutions:
  name: "heating-gateway"
  friendly_name: "RAMSES Heating Gateway"

esphome:
  name: ${name}
  friendly_name: ${friendly_name}

esp32:
  board: esp32-c6-devkitc-1
  variant: esp32c6
  framework:
    type: esp-idf

external_components:
  - source: github://wlcrs/esphome-ramses
    components: [ ramses_esp, ramses_devices ]

api:

ota:
  - platform: esphome

logger:
  level: INFO

wifi:
  ssid: "YOUR_WIFI_SSID"
  password: "YOUR_WIFI_PASSWORD"

ramses_esp:
  id: ramses_hub
  sck_pin: GPIO6
  miso_pin: GPIO2
  mosi_pin: GPIO7
  cs_pin: GPIO18
  gdo0_pin: GPIO15
  gdo2_pin: GPIO14

# ==========================================================
# Discovered RAMSES Entities
# ==========================================================

# 1. Heating Zones (Climate)
climate:
  - platform: ramses_devices
    ramses_esp_id: ramses_hub
    name: "Living Room Heating"
    controller_address: "01:145678" # Your controller ID
    zone_index: 0
    zone_name: "Living Room"

  - platform: ramses_devices
    ramses_esp_id: ramses_hub
    name: "Bedroom Heating"
    controller_address: "01:145678"
    zone_index: 1
    zone_name: "Bedroom"

# 2. Ventilation Fan (MVHR)
fan:
  - platform: ramses_devices
    ramses_esp_id: ramses_hub
    name: "Ventilation Unit"
    device_address: "32:155617"     # Your ventilation unit ID
    scheme: orcon                   # Or: vasco, itho, zehnder, hopper

# 3. Sensors
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
    name: "Living Room Heat Demand"
    device_address: "04:089123"     # Your radiator TRV ID

  - platform: ramses_devices
    ramses_esp_id: ramses_hub
    type: filter_remaining_percent
    name: "Ventilation Filter Health"
    device_address: "32:155617"
```

3. Click **Install**. Since your ESP32 is already on your Wi-Fi network, you can select **Wirelessly (Over The Air)** to flash it without needing to plug it into your computer!

---

## Step 4: View and Control Your Devices in Home Assistant

Once the firmware finishes flashing:

1. Open **Home Assistant** and go to **Settings** $\rightarrow$ **Devices & Services**.
2. Under the **Discovered** section at the top, you will see your new ESPHome device (`heating-gateway`).
3. Click **Configure** $\rightarrow$ **Submit**.
4. That's it! Home Assistant will now display all your entities:
   - **Thermostats**: Each heating zone has its own standard Climate card showing current temperature and allowing target temperature adjustments.
   - **Ventilation**: A native Fan entity with speed preset controls.
   - **Sensors**: Clean graphable entities for room temperatures, heat demand percentage, filter lifetime, and battery levels.

---

## Step 5 (Optional): Pair a Virtual Remote to Control Your Ventilation Unit

Many Mechanical Ventilation (MVHR) units (such as Orcon, Itho, Vasco, or Zehnder) will monitor their sensors automatically, but will **only accept speed commands from paired wireless remotes**.

Your ESP32 can emulate a real 4-button wireless remote control (a "fake remote") and register itself with the ventilation unit over the air!

### 1. Add the Pairing Button to your `fan:` block
Open your ESPHome configuration and add `pairing_button:` to your `fan:` section:

```yaml
fan:
  - platform: ramses_devices
    ramses_esp_id: ramses_hub
    name: "MVHR Ventilation Fan"
    device_address: "32:155617"
    scheme: orcon   # Match your brand: orcon, vasco, itho, zehnder, hopper
    pairing_button:
      name: "Pair Ventilation Remote"
```

Flash the update to your ESP32. In Home Assistant, a new button called **`button.pair_ventilation_remote`** will appear alongside your fan.

### 2. Put the Ventilation Unit into Pairing Mode
Ventilation units only accept new remotes when placed in "Bind" or "Pairing" mode:
- **Power-cycle method (most common)**: Unplug the ventilation unit (or flip its circuit breaker) for 15 seconds, then plug it back in. Most units stay in pairing mode for the first **3 to 10 minutes** after boot.
- **Button method**: Some units have a small pairing button on the main casing or behind the front cover. Hold it until an LED begins blinking or you hear a confirmation beep.

### 3. Click the Pair Button
1. In Home Assistant, go to your ESPHome device page and click **Press** on the **Pair Ventilation Remote** button.
2. The ESP32 will transmit the special RAMSES binding sequence.
3. The ventilation unit will acknowledge the pair (usually with a beep or a change in the LED pattern).

### 4. Test Fan Speed Control
Now test changing the fan speed in Home Assistant:
- Select **Low / Away / 1**, **Medium / 2**, or **High / Boost / 3**.
- You will hear the ventilation fan speed up or slow down accordingly!

---

## Troubleshooting & FAQs

### The Activity LED is not blinking
- Verify your CC1101 wiring against the pinout table.
- Make sure an 868 MHz antenna is screwed securely into the CC1101 connector.
- Confirm your hardware is running the correct firmware variant (`esp-idf`).

### No devices show up in the discovery table
- **Give it a few minutes**: Battery-powered radiator valves only transmit every 3 to 10 minutes to preserve battery life.
- **Trigger an update manually**: Turn the dial on a radiator valve or change the temperature setting on your wall thermostat to force an immediate transmission.
- Check the distance between your ESP32 and the nearest RAMSES device. While 868 MHz penetrates walls well, start by placing the ESP in the same room or hallway to confirm reception.

### What is the difference between `ramses_devices` and `ramses_cc`?
- **`ramses_devices` (used in this tutorial)** is built directly into ESPHome. It decodes frames inside the ESP32 chip and creates standard ESPHome entities. It is 100% self-contained and does not require any third-party Home Assistant integrations.
- **`ramses_cc`** is a custom Home Assistant integration written in Python. It uses the ESP32 purely as a "dumb" radio gateway (HGI80 emulation) and performs all decoding on the Home Assistant server. If you prefer `ramses_devices`, you do **not** need `ramses_cc`.

---

Congratulations! You now have full local visibility and control over your RAMSES II heating and ventilation system. If you have questions or want to explore advanced automations, check out the main [README.md](README.md) or open an issue on GitHub.
