---
title: "Traversal — GCS Joystick UDP Sender"
date: 2026-03-05
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, traversal, joystick, udp]
---

## Overview

This script runs on the **Ground Control Station (GCS)** laptop. It reads a connected gamepad/joystick using `pygame` and sends single-character command codes over **UDP** to the rover at a fixed IP and port. Each physical input (stick, trigger, D-pad, or button) maps to a unique letter that the rover interprets as a drive or arm command.

---

## Dependencies

| Package | Purpose |
|---|---|
| `pygame` | Joystick input reading |
| `socket` | UDP packet transmission |
| `time` | Loop rate control |

Install with:

```bash
pip install pygame
```

---

## Configuration

| Constant | Value | Description |
|---|---|---|
| `ROVER_IP` | `192.168.1.25` | Rover's IP address on the shared network |
| `ROVER_PORT` | `5006` | UDP port the rover listens on |

---

## Input → Command Mapping

### Left Stick (Axes 0, 1) — Drive / Movement

| Physical Input | Axis | Threshold | Letter | Action |
|---|---|---|---|---|
| Left stick up | Axis 1 | < −0.5 | `y` | Forward |
| Left stick down | Axis 1 | > +0.5 | `h` | Backward |
| Left stick left | Axis 0 | < −0.5 | `g` | Turn left |
| Left stick right | Axis 0 | > +0.5 | `t` | Turn right |

### Right Stick (Axes 3, 4) — Arm Control

| Physical Input | Axis | Threshold | Letter | Action |
|---|---|---|---|---|
| Right stick up | Axis 4 | < −0.5 | `u` | Arm up |
| Right stick down | Axis 4 | > +0.5 | `j` | Arm down |
| Right stick left | Axis 3 | < −0.5 | `r` | Arm left |
| Right stick right | Axis 3 | > +0.5 | `f` | Arm right |

### Analog Triggers (Axes 2, 5)

| Physical Input | Axis | Threshold | Letter | Action |
|---|---|---|---|---|
| Left trigger (L2) | Axis 2 | > +0.5 | `b` | Left trigger pulled |
| Right trigger (R2) | Axis 5 | > +0.5 | `H` | Right trigger pulled |

> **Note:** `b` is the mode-toggle signal on the rover side — pressing the left trigger will switch between Serial and ROS mode on the receiver.

### D-Pad (Hat 0)

| Hat Value | Letter | Action |
|---|---|---|
| `(0, 1)` | `w` | D-pad up |
| `(0, -1)` | `s` | D-pad down |
| `(-1, 0)` | `a` | D-pad left |
| `(1, 0)` | `d` | D-pad right |

### Special Axis Combinations (Exact Values)

These detect axis positions set to exactly `1` with all others at `0` — likely mapped to digital button presses on certain controllers:

| Condition | Letter | Action |
|---|---|---|
| Axis 3 == 1, all others 0 | `V` | — |
| Axis 0 == 1, all others 0 | `Q` | — |
| Axis 2 == 1, all others 0 | `P` | — |
| Axis 1 == 1, all others 0 | `O` | — |

> These exact-value checks are fragile — floating-point axis values rarely land at exactly `1.0`. Consider using threshold comparisons (`> 0.9`) for reliability.

### Buttons

| Button Index | Letter | Controller Label |
|---|---|---|
| 0 | `3` | Bottom face button |
| 1 | `2` | Right face button |
| 2 | `4` | Left face button |
| 3 | `1` | Top face button |
| 4 | `u` | Y |
| 5 | `E` | R1 |
| 6 | `Z` | L2 (digital click) |
| 7 | `C` | R2 (digital click) |
| 8 | `U` | Select |
| 9 | `O` | Start |
| 10 | `N` | L3 (left stick click) |
| 11 | `P` | R3 (right stick click) |

---

## Input Priority Order

Since all inputs are evaluated with `if / elif`, the priority is strictly top-to-bottom:

```
1. Left stick
2. Right stick
3. Exact-axis combinations
4. Analog triggers
5. D-pad
6. Buttons (fallthrough)
```

If a stick is deflected at the same time as a button is pressed, the **stick takes priority**.

---

## Main Loop

```
pygame.event.pump()          ← process OS events
  ↓
Evaluate inputs top-to-bottom
  ↓
letter = matched command (or "x" if idle)
  ↓
sock.sendto(letter.lower(), (ROVER_IP, ROVER_PORT))
  ↓
time.sleep(0.03)             ← ~33 Hz send rate
```

All letters are sent as **lowercase** (`letter.lower()`), regardless of the case defined in the map. This means uppercase mappings like `H`, `E`, `V` etc. become `h`, `e`, `v` on the wire.

Loop rate: **~33 Hz** (30 ms sleep).

---

## Running

```bash
python3 gcs_joystick_udp_fullmap.py
```

Ensure the joystick is connected before launching. The script will crash on init if no joystick is found at index `0`.

---

## Notes

- Only the **first joystick** (`Joystick(0)`) is used — multi-controller setups are not supported
- The idle letter `"x"` is sent as `"x"` every cycle when no input is active, resulting in constant UDP traffic even at rest — add an idle guard if this is undesirable
- Axis numbering varies by controller model and OS; verify axis assignments with a tool like `jstest` or the pygame joystick example before deployment
- On Linux, plug the controller in before running; on Windows, the axis layout may differ from the mapping above
