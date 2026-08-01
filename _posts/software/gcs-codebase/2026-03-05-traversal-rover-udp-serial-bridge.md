---
title: "Traversal — Rover UDP–Serial/ROS Bridge"
date: 2026-03-05
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, traversal, ros2, udp]
---

## Overview

This script runs **on the rover**. It listens for single-character UDP packets from the GCS joystick sender, and depending on the active **mode**, either:

- **Mode 0 (SERIAL):** Translates commands into 4-byte PWM packets and writes them to the rover's motor controller over UART
- **Mode 1 (ROS):** Publishes the command letter as a ROS 2 `std_msgs/String` message on the `user_input` topic

The mode is **toggled at runtime** by the `b` command (left trigger on the GCS gamepad), with no restart required.

---

## Dependencies

| Package | Purpose |
|---|---|
| `socket` | UDP receive |
| `pyserial` | UART communication with motor controller |
| `rclpy` | ROS 2 Python client library |
| `std_msgs.msg.String` | ROS message type |

---

## Configuration

| Constant | Value | Description |
|---|---|---|
| `UDP_IP` | `0.0.0.0` | Bind to all interfaces |
| `UDP_PORT` | `5006` | Must match `ROVER_PORT` in the GCS script |
| Serial port | `/dev/ttyUSB1` | Motor controller serial device |
| Baud rate | `115200` | Serial baud rate |

---

## Architecture

```
GCS Laptop
  └── gcs_joystick_udp_fullmap.py
        └── UDP packet (1 byte letter) ──────────▶ Port 5006
                                                        │
                                              rover_udp_serial.py
                                                        │
                                          sock.recvfrom(1024)
                                                        │
                                              letter = decoded char
                                                        │
                                        ┌───────────────┴──────────────┐
                                        │                              │
                                    Mode 0                         Mode 1
                                   SERIAL                           ROS 2
                                        │                              │
                              ser.write(4-byte PWM)       ros_node.publish_letter()
                                        │                              │
                              /dev/ttyUSB1                   topic: user_input
```

---

## `UserInputPublisher` (ROS 2 Node)

A minimal ROS 2 publisher node.

| Property | Value |
|---|---|
| Node name | `user_input_publisher` |
| Topic | `user_input` |
| Message type | `std_msgs/String` |
| QoS depth | 10 |

**Method: `publish_letter(letter: str)`**

Wraps the letter string in a `String` message and publishes it, then logs the event.

---

## Mode System

| Mode | Value | Description |
|---|---|---|
| SERIAL | `0` | Commands drive motor controller via UART |
| ROS | `1` | Commands are published to ROS 2 topic |

### Toggle Mechanism

Mode switching is **edge-triggered** on the `b` letter to prevent repeated toggles from a held trigger:

```python
if letter == 'b':
    if not b_prev:       # only act on the rising edge
        mode ^= 1        # XOR toggle between 0 and 1
    b_prev = True
    continue
else:
    b_prev = False
```

This means holding the left trigger does **not** repeatedly toggle the mode — it only switches once per press.

---

## Mode 0 — Serial PWM Commands

Commands are sent as **4-byte packets** to the motor controller. Each byte represents:

```
[ LEFT_DIR, LEFT_SPEED, RIGHT_DIR, RIGHT_SPEED ]
```

| Byte | Value | Meaning |
|---|---|---|
| Direction byte | `1` | Forward |
| Direction byte | `0` | Reverse |
| Speed byte | `255` | Full speed |

### Command → Byte Map

| Letter | Command | Bytes Sent | Effect |
|---|---|---|---|
| `y` | Forward | `[1, 255, 1, 255]` | Both motors forward, full speed |
| `h` | Backward | `[0, 255, 0, 255]` | Both motors reverse, full speed |
| `g` | Turn left | `[1, 255, 0, 255]` | Left motor forward, right reverse |
| `t` | Turn right | `[0, 255, 1, 255]` | Left motor reverse, right forward |

> All unrecognised letters in Mode 0 are silently ignored — no bytes are written to serial.

---

## Mode 1 — ROS 2 Publishing

All letters (except `b`) are published verbatim to the `user_input` ROS topic as `std_msgs/String`. Downstream ROS nodes subscribe to this topic and act on the received letter.

---

## Main Loop Flow

```
sock.recvfrom(1024)          ← blocks until a UDP packet arrives
        │
   decode + strip
        │
   letter == 'b'?
   ┌────┴────┐
  YES        NO
   │          │
  Toggle    Mode 0?
  mode     ┌──┴──┐
           YES   NO
            │     │
         Serial  ROS
         write   publish
```

---

## Shutdown

On `KeyboardInterrupt`:

1. Serial port is closed (`ser.close()`)
2. ROS 2 is shut down (`rclpy.shutdown()`)

---

## Running

Ensure the ROS 2 environment is sourced, then:

```bash
source /opt/ros/<distro>/setup.bash
python3 rover_udp_serial.py
```

---

## Full Command Reference

| Letter | Mode 0 (Serial) | Mode 1 (ROS topic) |
|---|---|---|
| `y` | Forward `[1,255,1,255]` | Published as `"y"` |
| `h` | Backward `[0,255,0,255]` | Published as `"h"` |
| `g` | Left `[1,255,0,255]` | Published as `"g"` |
| `t` | Right `[0,255,1,255]` | Published as `"t"` |
| `b` | Mode toggle (consumed, not forwarded) | Mode toggle |
| All others | Ignored | Published verbatim |

---

## Notes

- The script initialises `rclpy` and creates the ROS node **unconditionally**, even if Mode 1 is never used — this requires a sourced ROS 2 environment even for serial-only operation
- `sock.recvfrom()` is **blocking** — the loop only advances when a UDP packet arrives; there is no timeout, so the loop rate is entirely driven by the GCS sender (~33 Hz)
- The serial write is fire-and-forget with no acknowledgement or error handling; dropped bytes are silent
- To add more drive commands in Mode 0, extend the `if/elif` block with additional letter-to-byte-packet mappings
- For production use, consider adding a **watchdog**: if no UDP packet is received within N ms, send a stop command to the motor controller to prevent runaway on network loss
