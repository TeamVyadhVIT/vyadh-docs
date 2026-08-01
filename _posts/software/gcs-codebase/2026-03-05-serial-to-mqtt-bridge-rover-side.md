---
title: "Serial-to-MQTT Bridge — Rover Side"
date: 2026-03-05
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, sensors, mqtt]
---

## Overview

This Python script acts as a **middleware layer** between two ESP32 sensor nodes and an MQTT broker. It reads serial output from each ESP32 over USB, parses sensor values using regex patterns, and publishes them as structured MQTT topics.

---

## Dependencies

| Package | Purpose |
|---|---|
| `pyserial` | Reading data from serial ports |
| `paho-mqtt` | Publishing to MQTT broker |
| `threading` | Concurrent serial readers for two ESP32s |
| `re` | Regex-based line parsing |
| `time` | Main loop keepalive |

Install with:

```bash
pip install pyserial paho-mqtt
```

---

## System Architecture

```
┌─────────────┐   USB Serial     ┌──────────────────┐
│  ESP32 #1   │ ──────────────▶  │                  │
│ BME + NPK   │  /dev/bme        │  serial_mqtt_    │
│ + Spectral  │                  │  bridge.py       │
└─────────────┘                  │                  │   MQTT Publish
                                 │  Thread 1 (ESP1) │ ─────────────▶ MQTT Broker
┌─────────────┐   USB Serial     │  Thread 2 (ESP2) │               192.168.1.25
│  ESP32 #2   │ ──────────────▶  │                  │               Port 1883
│  GPS + MQ   │  /dev/gps        └──────────────────┘
└─────────────┘
```

---

## Configuration

| Constant | Value | Description |
|---|---|---|
| `ESP1_PORT` | `/dev/bme` | Serial port for BME680 + NPK + Spectral ESP32 |
| `ESP2_PORT` | `/dev/gps` | Serial port for GPS + MQ sensors ESP32 |
| `BAUD_RATE` | `115200` | Serial baud rate for both devices |
| `MQTT_BROKER` | `192.168.1.25` | MQTT broker IP address |
| `MQTT_PORT` | `1883` | MQTT broker port |
| `MQTT_BASE` | `esp_sensors` | Root MQTT topic namespace |

> **Tip:** The `/dev/bme` and `/dev/gps` device names are persistent symlinks. Set these up with udev rules to avoid port reassignment on reconnect.

---

## MQTT Topic Map

### ESP1 — BME680 Readings

| Serial Line | MQTT Topic | Example Payload |
|---|---|---|
| `Temp: 24.5 °C` | `esp_sensors/esp1/bme/temp` | `24.5` |
| `Pressure: 1013.2 hPa` | `esp_sensors/esp1/bme/pressure` | `1013.2` |
| `Humidity: 55.1 %` | `esp_sensors/esp1/bme/humidity` | `55.1` |
| `Gas: 120.3 KΩ` | `esp_sensors/esp1/bme/gas` | `120.3` |
| `VOC: 11.84` | `esp_sensors/esp1/bme/voc` | `11.84` |

### ESP1 — NPK / Soil Readings

| Serial Line | MQTT Topic | Example Payload |
|---|---|---|
| `Nitrogen: 45` | `esp_sensors/esp1/npk/nitrogen` | `45` |
| `Phosphorous: 30` | `esp_sensors/esp1/npk/phosphorous` | `30` |
| `Potassium: 55` | `esp_sensors/esp1/npk/potassium` | `55` |
| `Moisture: 18` | `esp_sensors/esp1/npk/moisture` | `18` |
| `Temperature: 21` | `esp_sensors/esp1/npk/temperature` | `21` |
| `EC: 300` | `esp_sensors/esp1/npk/ec` | `300` |
| `pH: 55` | `esp_sensors/esp1/npk/ph` | `55` |

### ESP1 — Spectral Sensor

| Topic | Payload Format |
|---|---|
| `esp_sensors/esp1/spectral` | `val0,val1,val2,val3,val4,val5,val6,val7` |

Published only once **all 8 channel values have been received** for the current cycle.

### ESP2 — MQ Gas Sensors

| Serial Line | MQTT Topic | Example Payload |
|---|---|---|
| `MQ-4 (CH4): 1024` | `esp_sensors/esp2/mq4` | `1024` |
| `MQ-8 (H2): 856` | `esp_sensors/esp2/mq8` | `856` |
| `MQ-135 (Air): 2048` | `esp_sensors/esp2/mq135` | `2048` |

---

## Regex Patterns

| Pattern | Matches |
|---|---|
| `r'Spectral\s+(\d+):\s*(-?\d+)'` | Spectral channel index and value |
| `r'(Temp\|Pressure\|Humidity\|Gas\|VOC):\s*(-?\d+\.?\d*)'` | BME680 key-value pairs |
| `r'(Moisture\|Temperature\|EC\|pH\|Nitrogen\|Phosphorous\|Potassium):\s*(-?\d+)'` | NPK/soil key-value pairs |
| `r'MQ-(\d+).*:\s*(\d+)'` | MQ sensor number and raw ADC value |

All matching is case-sensitive and done on individual lines.

---

## Functions

### `handle_esp1_line(line: str)`

Processes a single line from ESP32 #1. Tries each regex in priority order:

1. **Spectral** — updates `spectral_values[idx]`, publishes when all 8 are filled
2. **BME** — publishes to `esp_sensors/esp1/bme/<key>`
3. **NPK** — publishes to `esp_sensors/esp1/npk/<key>`

### `handle_esp2_line(line: str)`

Processes a single line from ESP32 #2:

1. **MQ** — publishes to `esp_sensors/esp2/mq<number>`

### `publish_spectral()`

Checks if all 8 spectral channel values (`spectral_values`) are non-None. If complete, joins them as a comma-separated string, publishes to `esp_sensors/esp1/spectral`, and resets the buffer.

### `serial_reader(port: str, handler: callable)`

Daemon thread function that opens a serial port and continuously reads newline-terminated strings. Each non-empty line is printed and passed to `handler`.

```python
def serial_reader(port, handler):
    with serial.Serial(port, BAUD_RATE, timeout=1) as ser:
        while True:
            line = ser.readline().decode(errors='ignore').strip()
            if line:
                handler(line)
```

Errors are caught and printed without crashing the thread.

---

## Spectral Buffer Logic

The spectral sensor outputs 8 readings sequentially, one channel at a time. A shared list `spectral_values = [None] * 8` accumulates them:

```
Spectral 0: 512  →  spectral_values[0] = 512
Spectral 1: 480  →  spectral_values[1] = 480
...
Spectral 7: 390  →  spectral_values[7] = 390  →  PUBLISH + RESET
```

This ensures the MQTT broker always receives a complete, synchronised 8-channel reading.

---

## Threading Model

Two **daemon threads** run concurrently, one per serial port:

```python
threads = [
    threading.Thread(target=serial_reader, args=(ESP1_PORT, handle_esp1_line), daemon=True),
    threading.Thread(target=serial_reader, args=(ESP2_PORT, handle_esp2_line), daemon=True)
]
```

The main thread runs an infinite `time.sleep(1)` loop to keep the process alive. On `KeyboardInterrupt`, the MQTT loop is stopped and the client disconnects cleanly.

---

## Running

```bash
python3 serial_mqtt_bridge.py
```

Stop with `Ctrl+C`.

Ensure both ESP32 devices are connected before starting. If a device is absent, the thread will print a serial error and exit without affecting the other thread.

---

## Notes

- `decode(errors='ignore')` silently drops non-UTF-8 bytes — useful when ESP32 sends partial or garbled frames during startup
- The MQTT client uses **CallbackAPIVersion.VERSION2** — ensure `paho-mqtt >= 2.0` is installed
- No QoS or retained flags are set; all messages are published at **QoS 0**
- GPS output from ESP2 is not currently forwarded to MQTT; extend `handle_esp2_line()` with a LAT/LNG regex if needed
