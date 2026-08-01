---
title: "ESP32 Sensor Node 1 — BME680 + Soil NPK Firmware"
date: 2026-03-05
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, sensors, esp32, firmware]
---

## Overview

This firmware runs on an **ESP32** and interfaces with two sensor systems:
- A **BME680** environmental sensor over I2C for temperature, pressure, humidity, and gas resistance
- A **Modbus RTU soil sensor** over RS485 for NPK (nitrogen, phosphorous, potassium), moisture, pH, EC, and soil temperature

Readings are printed over **USB Serial at 115200 baud** and consumed downstream by the serial-to-MQTT bridge.

---

## Dependencies

| Library | Purpose |
|---|---|
| `Wire.h` | I2C communication for BME680 |
| `Adafruit_Sensor.h` | Unified sensor abstraction layer |
| `Adafruit_BME680.h` | BME680 driver |
| `HardwareSerial.h` | UART2 for RS485 communication |

---

## Hardware Configuration

### BME680

Connected via I2C using the ESP32's default SDA/SCL pins (GPIO 21 / GPIO 22).

### RS485 Interface

| Signal | ESP32 GPIO |
|---|---|
| RXD2 | 16 |
| TXD2 | 17 |
| RE (Receive Enable, active LOW) | 18 |
| DE (Driver Enable, active HIGH) | 19 |

The RE and DE pins control the RS485 transceiver direction. Both are held LOW during idle (receive mode).

---

## Modbus RTU Request Frames

The soil sensor communicates via **Modbus RTU** over RS485. Each request is an 8-byte frame. The currently defined (but commented-out) frames are:

| Parameter | Register | Frame |
|---|---|---|
| Nitrogen | `0x001E` | `01 03 00 1E 00 01 E4 0C` |
| Phosphorous | `0x001F` | `01 03 00 1F 00 01 B5 CC` |
| Potassium | `0x0020` | `01 03 00 20 00 01 85 C0` |
| Moisture | `0x0012` | `01 03 00 12 00 01 24 0F` |
| Soil Temp | `0x0013` | `01 03 00 13 00 01 75 CF` |
| EC | `0x0014` | `01 03 00 14 00 01 C4 0E` |
| pH | `0x0006` | `01 03 00 06 00 01 64 0B` |

All frames use **device address `0x01`**, **function code `0x03`** (read holding registers), and read **1 register** (2 bytes). CRC is included as the last 2 bytes (little-endian).

> **Note:** Soil sensor code is currently commented out. See the `readSoil()` section below for the implementation.

---

## `setup()`

1. Starts USB Serial at **115200 baud**
2. Initialises the **BME680** via I2C — halts with error if not found
3. Starts **UART2** (RS485Serial) at **9600 baud** on GPIO 16/17
4. Sets RE and DE pins as OUTPUT, both LOW (receive mode)

---

## `loop()`

### BME680 Reading

Calls `bme.performReading()` and prints all four values:

| Field | Unit | Serial Output |
|---|---|---|
| Temperature | °C | `Temp: X.XX °C` |
| Pressure | hPa | `Pressure: X.XX hPa` |
| Humidity | % | `Humidity: X.XX %` |
| Gas Resistance | kΩ | `Gas: X.XX KΩ` |
| VOC Index | — | `voc: X.XX` |

**VOC approximation formula:**

```
VOC ≈ ln(gas_resistance) + (0.04 × humidity)
```

This is a simplified proxy for IAQ (Indoor Air Quality) index without requiring the full Bosch BSEC library.

### Soil Sensor Reading (Commented Out)

When enabled, reads 7 parameters sequentially and applies scaling:

| Parameter | Raw Divisor | Example |
|---|---|---|
| Moisture | ÷ 10 | `183 → 18.3 %` |
| Soil Temp | raw float | `210 → 21.0 °C` |
| EC | none | whole number µs/cm |
| pH | ÷ 10 | `55 → 5.5` |
| Nitrogen | none | mg/kg |
| Phosphorous | none | mg/kg |
| Potassium | none | mg/kg |

Loop delay: **3000 ms**

---

## `readSoil(const uint8_t* frame)` (Commented Out)

Sends a Modbus RTU request frame over RS485 and reads the response.

**Sequence:**
1. Pull DE and RE HIGH → switch to transmit mode
2. Write 8-byte frame via `RS485Serial`
3. Flush transmit buffer
4. Pull DE and RE LOW → switch back to receive mode
5. Read up to 16 bytes within a **500 ms timeout**
6. Return `response[5]` if ≥ 7 bytes received (Modbus data byte position)

**Response frame structure:**

```
[0] Device Address   = 0x01
[1] Function Code    = 0x03
[2] Byte Count       = 0x02
[3] Data High Byte
[4] Data Low Byte    ← combined value
[5] CRC Low
[6] CRC High
```

> The current implementation reads only `response[5]` as a single byte. For full 16-bit register values, bytes [3] and [4] should be combined: `(response[3] << 8) | response[4]`.

---

## Serial Output Format

The bridge script (`serial_mqtt_bridge.py`) parses output using the following expected formats:

```
Temp: 24.5 °C
Pressure: 1013.2 hPa
Humidity: 55.1 %
Gas: 120.3 KΩ
voc: 11.84
```

---

## Notes

- The soil sensor section is fully implemented but **disabled via comments** — uncomment all blocks prefixed with `//` to enable
- Ensure the RS485 transceiver module supports **3.3V logic** for ESP32 compatibility
- The 5 ms delay after asserting DE/RE HIGH gives the transceiver time to switch before transmitting
- Increase `AUGER_MOVE_TIME_MS`-equivalent timeouts if sensor response is intermittent at 9600 baud
