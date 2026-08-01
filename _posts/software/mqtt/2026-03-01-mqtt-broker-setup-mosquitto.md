---
title: "MQTT — Broker Setup (Mosquitto)"
date: 2026-03-01
author: mahi
categories: [software, mqtt]
tags: [gcs, mqtt, mosquitto]
---

## Overview

The MQTT broker is the central hub through which all sensor data flows. The team uses **Mosquitto** — a lightweight, open-source MQTT broker that runs as a system service on Linux.

---

## Broker Placement

The broker can run on either the rover or the GCS machine. The recommended architecture is to **run the broker on the rover**:

- All sensor publisher nodes connect to the broker locally over loopback (`127.0.0.1`) — zero network latency on the publish side
- The GCS connects remotely over the wireless link as a subscriber
- If the wireless link drops, the rover continues collecting and logging data internally

Running the broker on the GCS is simpler to set up but means every sensor publish must cross the wireless link twice (rover → GCS broker → GCS subscriber), which adds latency and creates a single point of failure at the radio link.

---

## Installation

```bash
sudo apt update
sudo apt install mosquitto mosquitto-clients
```

`mosquitto` is the broker daemon. `mosquitto-clients` installs the `mosquitto_pub` and `mosquitto_sub` CLI tools used for testing.

---

## Running as a System Service

```bash
# Enable on boot
sudo systemctl enable mosquitto

# Start immediately
sudo systemctl start mosquitto

# Check status
sudo systemctl status mosquitto
```

---

## Configuration

The configuration file is located at:

```
/etc/mosquitto/mosquitto.conf
```

After editing, restart the service:

```bash
sudo systemctl restart mosquitto
```

### Anonymous Access (Development / Field Use)

```
listener 1883
allow_anonymous true
```

This allows any client on the network to connect without credentials. Acceptable for a closed competition network with no external access.

### Authenticated Access

```
listener 1883
allow_anonymous false
password_file /etc/mosquitto/passwd
```

Create a user and password:

```bash
sudo mosquitto_passwd -c /etc/mosquitto/passwd <username>
```

To add additional users without overwriting the file, omit the `-c` flag:

```bash
sudo mosquitto_passwd /etc/mosquitto/passwd <another-user>
```

### Logging (Optional but Recommended)

```
log_dest file /var/log/mosquitto/mosquitto.log
log_type all
```

This logs all broker activity — connections, disconnections, subscriptions, and errors — to a file. Useful for post-run debugging.

---

## Verifying the Broker is Running

From any machine on the same network, subscribe to a test topic:

```bash
mosquitto_sub -h <broker-ip> -t "test/#" -v
```

Then from another terminal, publish to it:

```bash
mosquitto_pub -h <broker-ip> -t "test/hello" -m "broker is alive"
```

If the subscriber terminal prints the message, the broker is running and reachable.

---

## Common Issues

| Issue | Cause | Fix |
|---|---|---|
| `Connection refused` on port 1883 | Mosquitto not running, or firewall blocking | Check `systemctl status mosquitto`, check `ufw` rules |
| Client connects but receives no messages | Topic mismatch or wrong broker IP | Verify topic strings and broker address |
| Broker starts then immediately stops | Config file syntax error | Run `mosquitto -c /etc/mosquitto/mosquitto.conf` manually to see the error |
| Authentication failure | Wrong password or `allow_anonymous false` with no credentials in client | Check client connection code for username/password |
