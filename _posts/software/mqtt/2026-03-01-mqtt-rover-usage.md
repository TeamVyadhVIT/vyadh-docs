---
title: "MQTT — Rover Usage"
date: 2026-03-01
author: mahi
categories: [software, mqtt]
tags: [gcs, mqtt, rover]
---

## Overview

This page covers how MQTT is used in practice on the rover system — how sensor data is published from the rover, how the GCS subscribes to it, payload format conventions, and latency considerations.

For protocol fundamentals, see [Concepts & Protocol]({{ site.baseurl }}/posts/mqtt-concepts-and-protocol/). For broker setup, see [Broker Setup]({{ site.baseurl }}/posts/mqtt-broker-setup-mosquitto/).

---

## Topic Naming Convention

All rover topics follow a consistent hierarchical structure:

```
rover/<subsystem>/<measurement>
```

| Topic | Data |
|---|---|
| `rover/sensors/gps/lat` | GPS latitude (decimal degrees) |
| `rover/sensors/gps/lng` | GPS longitude (decimal degrees) |
| `rover/sensors/mq4` | MQ-4 raw ADC value (methane) |
| `rover/sensors/mq8` | MQ-8 raw ADC value (hydrogen) |
| `rover/sensors/mq135` | MQ-135 raw ADC value (air quality) |
| `rover/battery/voltage` | Battery voltage (V) |
| `rover/status/connection` | `online` / `offline` — LWT topic |

Keep topic names lowercase with no spaces. Consistency here makes wildcard subscriptions and log parsing much easier.

---

## Python — Installing paho-mqtt

```bash
pip install paho-mqtt
```

---

## Publishing — Rover Side (Sensor Node)

A publisher connects to the broker and sends sensor readings on a topic. This runs on the rover's onboard computer (e.g. Raspberry Pi or Jetson).

```python
import paho.mqtt.client as mqtt
import json
import time

BROKER_IP = "127.0.0.1"  # If broker runs on the rover itself
BROKER_PORT = 1883

client = mqtt.Client()

# Register LWT before connecting
client.will_set("rover/status/connection", payload="offline", qos=1, retain=True)

client.connect(BROKER_IP, BROKER_PORT)

# Announce online
client.publish("rover/status/connection", payload="online", qos=1, retain=True)

while True:
    # Build payload
    payload = json.dumps({
        "lat": 12.576480,
        "lng": 76.905350
    })

    client.publish("rover/sensors/gps", payload=payload, qos=1)
    client.publish("rover/sensors/mq4", payload="1023", qos=0)
    client.publish("rover/sensors/mq8", payload="654", qos=0)
    client.publish("rover/sensors/mq135", payload="2187", qos=0)

    time.sleep(2)
```

> GPS and battery data use QoS 1 (delivery guaranteed). High-frequency sensor data like MQ readings use QoS 0 (no overhead) since occasional loss is acceptable.
{: .prompt-info }

---

## Subscribing — GCS Side

A subscriber connects to the broker and receives messages matching a topic pattern. This runs on the GCS laptop.

```python
import paho.mqtt.client as mqtt

BROKER_IP = "192.168.1.1"  # Rover IP on the field network
BROKER_PORT = 1883

def on_connect(client, userdata, flags, rc):
    if rc == 0:
        print("Connected to broker")
        client.subscribe("rover/#")  # Subscribe to all rover topics
    else:
        print(f"Connection failed with code {rc}")

def on_message(client, userdata, msg):
    topic = msg.topic
    payload = msg.payload.decode()
    print(f"[{topic}] {payload}")

client = mqtt.Client()
client.on_connect = on_connect
client.on_message = on_message

client.connect(BROKER_IP, BROKER_PORT)
client.loop_forever()  # Blocking — runs until interrupted
```

To handle specific topics separately rather than printing everything:

```python
def on_message(client, userdata, msg):
    if msg.topic == "rover/status/connection":
        print(f"Rover status: {msg.payload.decode()}")
    elif msg.topic.startswith("rover/sensors/"):
        sensor = msg.topic.split("/")[-1]
        print(f"Sensor [{sensor}]: {msg.payload.decode()}")
```

---

## Payload Format

MQTT message payloads are raw bytes. The format is a team convention, not enforced by the protocol.

| Format | Pros | Cons | Recommendation |
|---|---|---|---|
| Plain string | Simple | Not structured | Fine for single values |
| JSON | Structured, human-readable, easy to parse | Slightly larger payload | Recommended for multi-field messages |
| Binary / Struct | Compact, fast | Harder to debug | Only if bandwidth is severely constrained |

**Use JSON for multi-field messages, plain strings for single values:**

```python
# Single value — plain string
client.publish("rover/sensors/mq4", payload="1023")

# Multiple fields — JSON
client.publish("rover/sensors/gps", payload=json.dumps({"lat": 12.576, "lng": 76.905}))
```

---

## Latency Considerations

MQTT over a wireless link introduces latency from both the TCP connection and the radio link. For the rover system:

- **High-frequency data (IMU, raw ADC)** — use QoS 0. No acknowledgement overhead. Occasional message loss is acceptable.
- **Low-frequency important data (GPS, battery)** — use QoS 1. Delivery is confirmed. The small acknowledgement overhead is acceptable at these rates.
- **Critical one-off events** — use QoS 2 only if exactly-once semantics are essential. The four-step handshake adds significant round-trip overhead.

If MQTT latency becomes a bottleneck for time-critical data (e.g. real-time motor feedback), consider a direct UDP stream running in parallel. MQTT and UDP are not mutually exclusive — the team can use both simultaneously for different data types.
