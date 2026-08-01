---
title: "ESP32 Sensor Node 2 — GPS + MQ Gas Sensors Firmware"
date: 2026-03-05
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, sensors, esp32, gps, firmware]
---

## Overview

This firmware runs on an **ESP32** and reads:
- **GPS location data** (latitude/longitude) from a NEO-series GNSS module via UART
- **Raw ADC values** from three MQ-series gas sensors (MQ-4, MQ-8, MQ-135)

All data is printed over **USB Serial at 115200 baud** and consumed by the serial-to-MQTT bridge.

---

## Dependencies

| Library | Purpose |
|---|---|
| `Wire.h` | I2C (included but not actively used in this sketch) |
| `Adafruit_Sensor.h` | Sensor abstraction (included for compatibility) |
| `TinyGPSPlus.h` | NMEA sentence parsing for GPS module |
| `HardwareSerial.h` | UART1 for GPS communication |

---

## Hardware Configuration

### NEO GPS Module (UART1)

| Signal | ESP32 GPIO |
|---|---|
| RX (ESP32 receives) | 22 |
| TX (ESP32 transmits) | 23 |

Baud rate: **9600**

### MQ Gas Sensors (ADC)

| Sensor | GPIO | Detects |
|---|---|---|
| MQ-4 | 33 | Methane (CH₄) |
| MQ-8 | 34 | Hydrogen (H₂) |
| MQ-135 | 35 | Air quality / CO₂ / NH₃ |

All three pins are used as **analog inputs** (`analogRead`). GPIO 33–35 are ADC1 channels, compatible with ESP32 WiFi active.

> **Note:** Raw 12-bit ADC values (0–4095) are output — no conversion to PPM is applied. Calibration curves must be applied externally if PPM readings are needed.

---

## `setup()`

1. Starts USB Serial at **115200 baud**
2. Starts `neogps` (HardwareSerial on UART1) at **9600 baud** on GPIO 22/23

---

## `loop()`

### GPS Reading

Reads all available bytes from `neogps`, echoes raw NMEA data to Serial, and feeds each character to the `TinyGPSPlus` parser:

```cpp
while (neogps.available()) {
    char c = neogps.read();
    Serial.write(c);
    gps.encode(c);
}
```

When a valid location fix is obtained (`gps.location.isUpdated()`), the coordinates are printed to 6 decimal places:

```
LAT: 12.971599
LNG: 77.594566
```

### MQ Sensor Reading

Three `analogRead()` calls sample the gas sensors and print raw ADC values:

```
MQ-4 (CH4): 1024
MQ-8 (H2): 856
MQ-135 (Air): 2048
----------------------
```

Loop delay: **2000 ms**

---

## Serial Output Format

The bridge script (`serial_mqtt_bridge.py`) parses this output with the regex:

```python
r'MQ-(\d+).*:\s*(\d+)'
```

Expected output lines:

```
MQ-4 (CH4): <value>
MQ-8 (H2): <value>
MQ-135 (Air): <value>
```

GPS coordinates are not currently forwarded to MQTT by the bridge — extend `handle_esp2_line()` with a GPS regex if needed.

---

## NMEA Data

`TinyGPSPlus` processes standard NMEA sentences (`$GPRMC`, `$GPGGA`, etc.) from the NEO module. The raw sentences are also echoed directly to Serial for debugging. Ensure the GPS module has a clear sky view for a valid fix.

---

## Notes

- GPIO 34 and 35 are **input-only** on the ESP32 and cannot be used as outputs — they are suitable for ADC only
- MQ sensors require a **warm-up period** (typically 20–60 seconds after power-on) before readings stabilise
- If the GPS module outputs at a different baud rate, update `neogps.begin()` accordingly
- Raw NMEA echo to Serial may interleave with MQ output lines — the bridge regex is designed to match specific prefixes and will ignore unrecognised lines
