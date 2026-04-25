# 🌊 M5Tough Balboa Spa Monitor

An ESPHome project that turns an M5Stack M5Tough into a Home Assistant–integrated monitor and controller for Balboa BP-series spa systems, talking to the spa's main panel bus over RS-485.

![M5Tough Balboa Spa Wiring](documentation/hw-setup-final.png)

## 🛠️ Required Components

### 💻 Core hardware

- **[M5Stack Tough ESP32 IoT Development Board](https://shop.m5stack.com/products/m5stack-tough-esp32-iot-development-board-kit?variant=40644956160172)** — main controller (ESP32, 2.4" TFT + touch).
- **[M5Tough Extension Board](https://shop.m5stack.com/products/m5tough-ext-board)** — adds four labeled HY2.0-4P (Grove) slots: GPIO, UART, I²C, RS485. We use the **UART** slot.
- **[M5Stack RS485 Unit (U034)](https://docs.m5stack.com/en/unit/rs485)** — Grove TTL ↔ RS-485 transceiver (SP485EEN, auto-direction). Plug into the Extension Board's **UART** slot.
- **4-pin cable** to splice into the spa's J34 / J35 main panel port. Any 4-conductor cable terminated in a Molex 43025-0400 (TE 794617-4) works; many builders just buy a Balboa main-panel extension cable and cut it.

> ⚠️ **Power:** the M5Tough is powered from the spa's J34 +12V / GND pins via the RS485 Unit's VIN terminal. **No USB power needed in normal operation.** All four wires (12V, GND, A, B) go from J34 to the U034.

### 🛁 Spa side

- A Balboa **BP-series controller** with a spare main panel port. This build is verified on **BP6013G1** (BP21 platform, software M100_226 V65.0); wiring/protocol is shared across the BP21 family.
- The panel bus is RS-485 at **115200 8-N-1**.
- **Do not** disconnect the topside panel — tap into J34 or J35 (whichever is unused). Both are interchangeable MAIN ports on the same RS-485 bus. AUX panels live on J5 / J8 (separate bus, do not use).
- Network connection (Wi-Fi) for Home Assistant and OTA updates.

## ⚙️ Hardware configuration

### Spa connector pinout (J34 / J35)

4-pin Molex 43025-0400 (TE 794617-4). Looking at the connector with the **clip facing up**:

```
   ┌─────────┐
   │ 4   3   │   ← top row
   │ 2   1   │   ← bottom row
   └─────────┘
       │
       clip (key) on this side
```

| Pin | Signal             | Voltage to GND | Goes to U034 terminal |
|----:|--------------------|----------------|-----------------------|
|   1 | **+12–15 VDC**     | ~12–15 V       | VIN                   |
|   2 | **RS-485 B (D+)**  | ~2–3 V         | B                     |
|   3 | **RS-485 A (D−)**  | ~2–3 V         | A                     |
|   4 | **GND / Return**   | 0 V            | GND                   |

#### Wire colors are not standardized — verify with a multimeter

Cable colors vary by manufacturer and year. Two real-world cables encountered for the same connector have *inverted* color codes:

| Pin | Function | Cable A (community writeup) | Cable B (this build)  |
|----:|----------|-----------------------------|-----------------------|
|   1 | +12V     | yellow                      | red                   |
|   2 | B        | black w/ purple tape        | black                 |
|   3 | A        | black                       | white                 |
|   4 | GND      | red                         | yellow                |

Identify pins by **physical position relative to the clip**, or **verify with a multimeter** (12V on power, 0V on GND, ~2-3V on A and B). Color-only mapping has misled multiple builders.

### M5 RS485 Unit (U034) connections

- Chip: **SP485EEN** — half-duplex with auto-direction control. No DE/RE GPIO required.
- Grove HY2.0-4P input from the Extension Board (yellow=UART_RX, white=UART_TX, red=5V, black=GND).
- 4-position screw terminal block to the spa: **B, A, GND, VIN** (12-24V power input).
- Onboard 120Ω termination.

Tying GND between U034 and the spa is required as a common reference. If A/B end up reversed at the spa connector, the symptom is *garbled bytes / CRC errors* — swap A↔B at the U034 terminals.

### M5Tough Extension Board slot map

The Extension Board exposes four HY2.0-4P (Grove) ports. Per its schematic, each routes the Grove pins to different ESP32 GPIOs:

| Slot       | Pin 2 (Yellow)  | Pin 3 (White)            | Notes                                              |
|------------|-----------------|--------------------------|----------------------------------------------------|
| GPIO       | GPIO 26 (out)   | **GPIO 36 (input-only!)** | TX cannot drive — won't work as UART output       |
| **UART**   | **GPIO 14 (TX)** | **GPIO 13 (RX)**        | **Use this for the U034**                          |
| I²C        | GPIO 32 (SDA)   | GPIO 33 (SCL)            | I²C only                                           |
| RS485      | B− (differential) | A+ (differential)      | Onboard SP485EEN — bypasses the U034 if used; differential output only, not for the U034 Grove cable |

The corresponding ESPHome UART block:

```yaml
uart:
  id: spa_uart_bus
  tx_pin: GPIO14
  rx_pin: GPIO13
  baud_rate: 115200
  data_bits: 8
  parity: NONE
  stop_bits: 1
  rx_buffer_size: 1024
```

## 📊 Current status

### ✅ Implemented

- Real-time spa monitoring via Home Assistant (encrypted ESPHome API, no MQTT broker required)
- Climate / thermostat with current and target temperature
- Single massage pump as a `fan` entity (more stable than `switch` on BP-series boards)
- Spa light as a switch
- Filter cycle 1 + 2 configuration — read + write from HA
- Spa time read + write; **automatic time sync from HA on boot** (Balboa has no battery-backed RTC; resets to 12:00 after power loss)
- Fault log diagnostics: fault code/total/current/days-ago, fault message, fault log time, request-fault-log button
- Heartbeat & status: `connected` and `highrange` binary sensors
- Reminder text + component firmware version exposed
- WiFi diagnostics (signal, IP, SSID, MAC, uptime)
- OTA firmware updates (password-protected)

### 📋 Todo

- ⬜ M5Tough TFT touch UI for local control
- ⬜ Home Assistant blueprints / dashboard examples
- ⬜ Investigate switching to [`jhenkens/esphome-balboa-spa`](https://github.com/jhenkens/esphome-balboa-spa) once it's compatible with current ESPHome (currently breaks due to a removed `UNIT_FAHRENHEIT` constant) — its central command-retry queue should improve toggle reliability further.

## 🧩 Software

- **Component:** [`brianfeucht/esphome-balboa-spa`](https://github.com/brianfeucht/esphome-balboa-spa) — pulled as an `external_components` source.
- **Integration:** ESPHome native API to Home Assistant (encrypted). MQTT is no longer used.
- **Time:** `time: homeassistant` pulls time from HA; `on_boot` waits 15 s and presses a `sync_time` button to push the current time to the spa.

## 🚀 Getting started

### Prerequisites

- ESPHome installed (`pip install esphome` or `uv tool install esphome`)
- M5Tough + Extension Board + U034 RS485 Unit, Grove cable in the **UART** slot
- 4-wire cable spliced to spa J34 / J35 with the topside panel still connected on the other port
- Wi-Fi credentials and Home Assistant for the API + time

### Configure secrets

Copy `secrets_template.yaml` → `secrets.yaml` and fill in:

- `device_name`, `friendly_name`
- `wifi_ssid`, `wifi_password`
- `api_encryption_key` (generate with `esphome wizard` or any 32-byte base64 value)
- `ota_password`

### Compile and flash

```bash
# Validate the YAML
esphome config esphome-m5tough-balboa-spa.yaml

# First time — over USB (M5Tough connected via USB-C)
esphome run esphome-m5tough-balboa-spa.yaml --device COMx     # Windows
esphome run esphome-m5tough-balboa-spa.yaml --device /dev/ttyUSB0  # Linux/Mac

# Subsequent updates — over the air
esphome run esphome-m5tough-balboa-spa.yaml --device <hostname-or-ip>.local

# Tail logs
esphome logs esphome-m5tough-balboa-spa.yaml --device <hostname-or-ip>.local
```

### Add to Home Assistant

After the device is on Wi-Fi, HA's ESPHome integration will auto-discover it. Provide the encryption key from `secrets.yaml` when prompted.

## 🔧 Troubleshooting

Most issues observed during this build are physical-layer.

| Symptom on M5             | Symptom on panel | Root cause                                                                       |
|---------------------------|------------------|-----------------------------------------------------------------------------------|
| 0 frames received         | Normal           | Cable in **GPIO** slot — pin 3 = GPIO 36 (input-only), TX never reaches U034 DI   |
| 0 frames received         | NO COMM          | Topside panel unplugged to make room — keep it connected, use the *other* main port |
| 0 frames received         | Normal           | Wires not on data pair — e.g. mapped to GND/12V instead of pins 2/3              |
| Garbled CRC errors        | NO COMM          | A/B reversed — swap them at the U034                                              |
| 0 frames, panel works     | —                | U034 GND not tied (no common reference)                                           |
| Boot loop on Wi-Fi connect| n/a              | A `restart` button entity is being externally pressed (HA automation or stale MQTT command); we removed it from the YAML |

**Counter-intuitive lesson:** silence ≠ polarity bug. A/B reversed produces *garbled* bytes. Pure silence almost always means wrong pins or a dead/ungrounded transceiver.

## 📁 Project structure

```
├── esphome-m5tough-balboa-spa.yaml    # Main ESPHome configuration
├── secrets_template.yaml              # Template for secrets.yaml
├── secrets.yaml                       # Local secrets (gitignored)
├── README.md                          # This file
└── documentation/                     # Photos / diagrams
```

## 🙏 Credits and references

Builds on years of community work decoding the Balboa panel protocol:

- **[brianfeucht/esphome-balboa-spa](https://github.com/brianfeucht/esphome-balboa-spa)** — the ESPHome component this project consumes.
- **[jhenkens/esphome-balboa-spa](https://github.com/jhenkens/esphome-balboa-spa)** — rewritten fork with a typed message layer and central command-retry queue (currently incompatible with ESPHome 2026.4.0).
- **[ccutrer/balboa_worldwide_app](https://github.com/ccutrer/balboa_worldwide_app/wiki)** — protocol & physical-layer wiki, the canonical reference for connector pinouts.
- **[Dakoriki/ESPHome-Balboa-Spa](https://github.com/Dakoriki/ESPHome-Balboa-Spa)** — earlier ESPHome integration.
- **[mhetzi/esphome-balboa-spa](https://github.com/mhetzi/esphome-balboa-spa)** — alternative maintained fork.
- **[Reddit /r/hottub: "Finally made my tub smart"](https://www.reddit.com/r/hottub/comments/1rbvkhu/)** — practical wiring writeup that helped resolve the J34 pin mapping during this build.
- **[Balboa BP6013G1 tech sheet (PN 56611-08)](https://www.balboawater.com/wp-content/uploads/2025/03/BP6013G1-Current.pdf)** — official wiring diagram showing J34/J35 as MAIN panel ports.
- **[M5Tough Extension Board schematic](https://m5stack-doc.oss-cn-shenzhen.aliyuncs.com/647/Tough-Ext-Board-Schematics-PDF.pdf)** — definitive slot-to-GPIO mapping.
- **[Original repo by dhWasabi](https://github.com/dhWasabi/M5Tough-BalboaSpa-esphome)** — the M5Tough-specific starting point.
