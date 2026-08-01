---
title: "MQTT — Concepts & Protocol"
date: 2026-03-01
author: mahi
categories: [software, mqtt]
tags: [gcs, mqtt]
---

## Overview

MQTT (Message Queuing Telemetry Transport) is a lightweight, publish-subscribe messaging protocol designed for constrained environments and low-bandwidth, high-latency networks. In the rover system, it serves as the primary transport layer for sensor data between the rover and the Ground Control Station (GCS).

MQTT uses **TCP** as its underlying transport layer, which provides reliable, ordered delivery of messages.

Official Specification: [MQTT v5.0 — OASIS Standard](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.pdf)

---

## Why MQTT for Rover Telemetry

| Feature | Benefit |
|---|---|
| Low Bandwidth | Operates efficiently over weak wireless links |
| Low Latency | Enables near real-time sensor monitoring and control |
| Reliable Delivery | QoS levels ensure messages are not silently dropped |
| Lightweight | Minimal CPU and memory overhead on embedded hardware |

---

## Publish-Subscribe Model

MQTT follows a publish-subscribe pattern. Devices do not communicate directly with each other. Instead:

- A **publisher** sends a message to a named channel called a **topic**.
- The **broker** receives the message and forwards it to all clients that have subscribed to that topic.
- A **subscriber** receives messages from topics it has registered interest in.

This decouples the sender and receiver — the publisher does not need to know who is listening, and the subscriber does not need to know who is sending.

---

## Topics

A topic is a named channel used to route messages through the broker. Topics are UTF-8 strings structured hierarchically using `/` as a separator.

**Example topic structure for the rover:**

```
rover/sensors/imu
rover/sensors/gps
rover/battery/voltage
rover/battery/current
rover/status/connection
```

### Wildcards

Subscribers can use wildcard characters to subscribe to multiple topics at once.

| Wildcard | Scope | Example |
|---|---|---|
| `+` | Single level | `rover/sensors/+` matches `rover/sensors/imu` and `rover/sensors/gps` but not `rover/sensors/imu/x` |
| `#` | Multi level | `rover/#` matches everything under `rover/` |

---

## Broker

The broker is the central server in MQTT. It is responsible for:

- Receiving messages from publishers
- Routing and delivering messages to matching subscribers
- Managing client connections and session state
- Handling QoS guarantees, retained messages, and Last Will and Testament

All communication passes through the broker. Clients never communicate directly with each other.

**Mosquitto** is the broker used by the team. See [Broker Setup]({{ site.baseurl }}/posts/mqtt-broker-setup-mosquitto/) for installation and configuration.

---

## Quality of Service (QoS)

QoS defines the delivery guarantee for a message between sender and broker, and between broker and subscriber.

| QoS Level | Name | Guarantee | Rover Use Case |
|---|---|---|---|
| 0 | At most once | No acknowledgement. Message may be lost. | High-frequency sensor streams (e.g. IMU at 100 Hz) |
| 1 | At least once | Acknowledged. Message may be delivered more than once. | Battery status, GPS coordinates |
| 2 | Exactly once | Four-step handshake. Guaranteed single delivery. | Critical commands or alerts |

In practice, QoS 0 is used for high-rate telemetry and QoS 1 for lower-frequency but important data. QoS 2 is rarely used due to the overhead of the four-step handshake.

---

## Retained Messages

When a message is published with the **retained** flag set, the broker stores the last value for that topic. Any new subscriber will immediately receive it on connection, rather than waiting for the next publish event.

This is useful for status topics — for example, publishing the rover's last known GPS position as retained ensures the GCS always has a value on startup.

---

## Last Will and Testament (LWT)

LWT allows a client to register a message with the broker at connection time. If the client disconnects unexpectedly (without sending a proper DISCONNECT packet), the broker automatically publishes the LWT message on the client's behalf.

**Team usage — detecting rover disconnect:**

```
Topic:   rover/status/connection
Payload: offline
Retain:  true
```

When the rover connects cleanly, it publishes `online` to the same topic. If the link drops without a clean disconnect, the broker publishes `offline` automatically.

---

## Sessions

| Session Type | Behaviour |
|---|---|
| Clean Session | Broker discards all state on disconnect. Subscriptions must be re-established on reconnect. |
| Persistent Session | Broker retains subscriptions and queued QoS ≥ 1 messages while the client is offline, delivering them on reconnect. |

For the GCS subscriber, a persistent session ensures no telemetry is missed during brief wireless dropouts.
