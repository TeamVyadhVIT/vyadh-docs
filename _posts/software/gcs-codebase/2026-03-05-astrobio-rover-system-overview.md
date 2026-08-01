---
title: "AstroBIO Rover — System Overview"
date: 2026-03-05
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, traversal, astrobio]
---

## What Is This?

AstroBIO is a **field science rover** designed for autonomous and teleoperated soil and atmospheric data collection. It combines a ground vehicle with a mounted science module, controlled over a local WiFi network from a laptop-based Ground Control Station (GCS). The full system spans embedded firmware, ROS 2 middleware, desktop GUIs, sensor pipelines, and real-time teleoperation.

---

## The Big Picture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        GROUND CONTROL STATION (Laptop)                  │
│                                                                         │
│   ┌─────────────────────────┐     ┌─────────────────────────────────┐   │
│   │  gcs_joystick_udp.py    │     │       sensor_dashboard.py       │   │
│   │  (pygame + UDP sender)  │     │  (PyQt5 + Matplotlib + MQTT)    │   │
│   └────────────┬────────────┘     └──────────────┬──────────────────┘   │
│                │ UDP letter cmds                  │ MQTT subscribe       │
└────────────────┼──────────────────────────────────┼─────────────────────┘
                 │ 192.168.1.25:5006                │ 192.168.1.25:1883
                 │                                  │
┌────────────────┼──────────────────────────────────┼─────────────────────┐
│                │          ROVER (Onboard PC)       │                     │
│                ▼                                  ▼                     │
│   ┌─────────────────────────┐     ┌─────────────────────────────────┐   │
│   │  rover_udp_serial.py    │     │       MQTT Broker               │   │
│   │  UDP → Serial / ROS 2   │     │       (Mosquitto :1883)         │   │
│   └────────────┬────────────┘     └──────────────▲──────────────────┘   │
│                │                                  │                     │
│        ┌───────┴──────┐               ┌───────────┴──────────┐          │
│        │   Mode 0     │               │  serial_mqtt_bridge  │          │
│        │   SERIAL     │               │  (2 threads, USB)    │          │
│        │  /dev/ttyUSB1│               └───────┬──────────────┘          │
│        └──────┬───────┘                       │                         │
│               │           ┌───────────────────┴───────────┐             │
│        ┌──────▼───────┐   │  /dev/bme        /dev/gps     │             │
│        │ Motor Driver │   │  ESP32 #1        ESP32 #2      │             │
│        │ (Drive Base) │   │  BME680+NPK      GPS+MQ Gas   │             │
│        └──────────────┘   └───────────────────────────────┘             │
│                                                                         │
│   ┌──────────────────────────────────────────────────────────────────┐  │
│   │                     ROS 2 Network (DDS)                          │  │
│   │                                                                  │  │
│   │   user_input topic ──▶ [downstream arm / science nodes]         │  │
│   │                                                                  │  │
│   │   gui_node.py (Science GCS GUI)                                  │  │
│   │     publishes to:                                                │  │
│   │       /rover/servo/beaker/cmd    /rover/servo/test_tube/cmd      │  │
│   │       /rover/steppers/cmd        /rover/drill/cmd                │  │
│   │       /rover/auger_shutter/cmd   /rover/pumps/cmd                │  │
│   │                                                ▼                 │  │
│   │                                      Arduino Mega 2560           │  │
│   │                                      (Science Module Firmware)   │  │
│   └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## System Layers

### Layer 1 — Drive & Teleoperation

The operator uses a **gamepad on the GCS laptop**. `gcs_joystick_udp.py` reads stick and button inputs via pygame and sends a single-letter command over **UDP** to the rover at ~33 Hz.

On the rover, `rover_udp_serial.py` receives these letters and operates in one of two modes:

- **Mode 0 (Serial):** Translates direction commands directly into 4-byte PWM packets sent over UART to the **drive motor controller**, enabling immediate movement with zero ROS overhead.
- **Mode 1 (ROS):** Publishes letters onto the `user_input` ROS 2 topic, allowing downstream arm and science nodes to react.

The operator **switches modes live** by pressing the left trigger (letter `b`), without any restart.

---

### Layer 2 — Science Module Control

A separate desktop GUI (`gui_node.py`) runs on the GCS and communicates with the rover's **science payload** through ROS 2 topics. The operator uses it to control:

- **Two servo-driven carousels** — beaker (absolute angle) and test tube (incremental delta)
- **Two stepper motors** — a Z-axis drill lift and an X-axis gantry, with adjustable step delay
- **A DC drill motor** — variable speed and direction via PWM
- **An auger shutter** — timed open/close using a BTS7960 driver
- **Four peristaltic pumps** — individual on/off control

ROS 2 messages from the GUI are consumed by a **ROS–Serial bridge node** on the rover, which relays them as plain-text commands to the **Arduino Mega 2560** running the science module firmware.

---

### Layer 3 — Environmental Sensing

Two ESP32 nodes collect field data and publish it via MQTT:

**ESP32 #1** (`/dev/bme`) reads:
- BME680 — temperature, pressure, humidity, gas resistance, VOC index
- RS485 soil sensor — NPK (nitrogen, phosphorous, potassium), moisture, EC, pH, soil temperature *(currently staged, not yet active)*

**ESP32 #2** (`/dev/gps`) reads:
- NEO GNSS module — latitude and longitude
- MQ-4, MQ-8, MQ-135 — methane, hydrogen, and air quality (raw ADC)

The `serial_mqtt_bridge.py` script runs on the rover, reads both serial streams in parallel threads, parses each line with regex, and publishes structured values to the **Mosquitto MQTT broker**.

Back on the GCS laptop, `sensor_dashboard.py` subscribes to all MQTT topics and renders **live scrolling plots** for every channel across a 3×4 subplot grid, with an optional **CSV logging** toggle for data recording.

---

## Data Flows Summary

| Flow | Transport | From | To |
|---|---|---|---|
| Joystick → Rover | UDP (port 5006) | GCS laptop | Rover onboard PC |
| Drive commands | UART serial | Rover PC | Drive motor controller |
| Arm/science commands | ROS 2 DDS | GCS GUI node | Rover ROS bridge → Arduino |
| Sensor readings | USB Serial | ESP32s | Rover PC (bridge script) |
| Sensor readings | MQTT | Rover broker | GCS dashboard |

---

## Network

All devices share a local WiFi network. The rover is assigned a **static IP of `192.168.1.25`** and runs:
- UDP listener on port **5006**
- MQTT broker (Mosquitto) on port **1883**
- ROS 2 DDS (multicast or configured peers)

---

## Key Design Decisions

**UDP for drive, ROS for science.** Drive commands need the lowest possible latency and must survive a ROS crash. UDP with a raw serial fallback (Mode 0) provides that resilience. The science module is less latency-sensitive and benefits from ROS tooling (logging, introspection, modularity).

**MQTT for sensor telemetry.** Sensor data flows independently of ROS — any device on the network can subscribe to the broker without being part of the ROS graph. This makes the dashboard portable and decoupled.

**Single-letter protocol.** The joystick → UDP → rover pipeline uses single ASCII characters to keep packets tiny, parsing instant, and the protocol human-readable during debugging.

**Mode toggle without restart.** The live `b`-key mode switch means the operator can seamlessly hand off from driving to arm control and back without interrupting any process.
