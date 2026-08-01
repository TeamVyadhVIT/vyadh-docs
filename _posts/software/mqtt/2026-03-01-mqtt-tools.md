---
title: "MQTT — Tools"
date: 2026-03-01
author: mahi
categories: [software, mqtt]
tags: [gcs, mqtt, tools]
---

## mosquitto_sub

Subscribe to topics from the terminal. Part of the `mosquitto-clients` package.

**Subscribe to all rover topics (verbose — shows topic + payload):**

```bash
mosquitto_sub -h 192.168.1.1 -t "rover/#" -v
```

**Subscribe to a specific topic:**

```bash
mosquitto_sub -h 192.168.1.1 -t "rover/sensors/gps" -v
```

**Log everything to a file for post-run analysis:**

```bash
mosquitto_sub -h 192.168.1.1 -t "#" -v >> session_log.txt
```

**With authentication:**

```bash
mosquitto_sub -h 192.168.1.1 -t "rover/#" -v -u <username> -P <password>
```

---

## mosquitto_pub

Publish a message to a topic from the terminal. Useful for testing subscribers or simulating sensor data.

**Publish a single value:**

```bash
mosquitto_pub -h 192.168.1.1 -t "rover/sensors/mq4" -m "1023"
```

**Publish a JSON payload:**

```bash
mosquitto_pub -h 192.168.1.1 -t "rover/sensors/gps" -m '{"lat": 12.576, "lng": 76.905}'
```

**Publish with QoS 1 and retain flag:**

```bash
mosquitto_pub -h 192.168.1.1 -t "rover/status/connection" -m "online" -q 1 -r
```

---

## MQTT Explorer

A GUI tool for visualising the live topic tree, inspecting payloads, and publishing test messages without writing code.

Download: [mqtt-explorer.com](https://mqtt-explorer.com)

**What it shows:**
- Live tree of all active topics on the broker
- Payload of each topic updated in real time
- History graph for numeric values
- Ability to publish to any topic manually

This is the fastest way to verify that sensor data is flowing correctly from the rover to the broker during bench testing and field setup.

---

## Quick Reference

| Task | Command |
|---|---|
| Watch all topics | `mosquitto_sub -h <ip> -t "#" -v` |
| Watch rover topics only | `mosquitto_sub -h <ip> -t "rover/#" -v` |
| Log session to file | `mosquitto_sub -h <ip> -t "#" -v >> log.txt` |
| Publish test message | `mosquitto_pub -h <ip> -t "test/hello" -m "world"` |
| Check broker is reachable | Publish + subscribe simultaneously on `test/#` |
