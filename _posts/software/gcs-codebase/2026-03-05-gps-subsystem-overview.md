---
title: "GPS Subsystem — Overview"
date: 2026-03-05
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, gps]
---

## What This Subsystem Does

The GPS subsystem reads location data from a NEO GPS module connected to an ESP32, forwards it over the network, and displays it live on the GCS laptop. It is made up of three separate pieces of code that form a chain:

```
┌─────────────────────────────────────────────────────────────────┐
│                        ROVER                                     │
│                                                                   │
│  [NEO GPS Module] ──UART──► [ESP32 Firmware]                     │
│                              └─ Parses NMEA via TinyGPSPlus      │
│                              └─ Prints LAT/LNG to Serial (USB)   │
│                                        │                         │
│                                        │ USB Serial              │
│                                        ▼                         │
│                          [Serial-to-UDP Bridge]                  │
│                           └─ Reads Serial lines                  │
│                           └─ Sends UDP packet → GCS              │
└─────────────────────────────────────────────────────────────────┘
                                     │
                               UDP (port 5005)
                               over WiFi link
                                     │
┌─────────────────────────────────────────────────────────────────┐
│                        GCS LAPTOP                                │
│                                                                   │
│                    [GPS Receiver GUI]                            │
│                     └─ Listens on UDP 5005                       │
│                     └─ Displays LAT / LNG live                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## The Three Components

| File | Language | Runs On | Role |
|---|---|---|---|
| `esp32_gps_mq.ino` | Arduino C++ | ESP32 (rover) | Reads GPS + MQ sensors, prints to Serial |
| `serial_udp_bridge.py` | Python | Rover onboard computer (connected to ESP32 via USB) | Reads Serial, forwards GPS over UDP |
| `gps_receiver_gui.py` | Python | GCS laptop | Receives UDP, displays coordinates |

---

## Data Flow

```
NEO GPS Module
    │ NMEA sentences (9600 baud UART)
    ▼
ESP32 (UART1 on GPIO 22/23)
    │ TinyGPSPlus parses NMEA
    │ Prints "LAT: xx.xxxxxx" and "LNG: xx.xxxxxx" to USB Serial (115200 baud)
    ▼
Serial-to-UDP Bridge (Python, runs on rover computer)
    │ Reads Serial lines, pairs LAT + LNG
    │ Sends "lat,lng" as UTF-8 UDP datagram to GCS IP:5005
    ▼
GPS Receiver GUI (Python PyQt6, runs on GCS laptop)
    │ Listens on UDP 0.0.0.0:5005
    │ Splits payload on comma → lat, lng
    ▼
Live display on GCS screen
```

---

## Network Configuration

| Setting | Value |
|---|---|
| UDP port | 5005 |
| GCS IP (bridge target) | `192.168.1.26` — update in `serial_udp_bridge.py` if GCS IP changes |
| Receiver bind address | `0.0.0.0` — listens on all interfaces |

---

## Pages in This Section

- [ESP32 Firmware — GPS + MQ Sensors]({{ site.baseurl }}/posts/esp32-firmware-gps-and-mq-sensors/)
- [Serial to UDP Bridge]({{ site.baseurl }}/posts/gps-serial-to-udp-bridge-rover-side/)
- [GPS Receiver GUI]({{ site.baseurl }}/posts/gps-receiver-gui-gcs-side/)
