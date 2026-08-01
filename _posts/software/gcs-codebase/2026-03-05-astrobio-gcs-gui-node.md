---
title: "AstroBIO Ground Control Station — GUI Node"
date: 2026-03-05
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, astrobio, ros2]
---

## Overview

`gui_node.py` is the Ground Control Station (GCS) desktop application for the AstroBIO rover. It combines a **ROS 2 publisher node** with a **PyQt5 graphical interface**, allowing operators to send real-time commands to all rover subsystems over ROS topics.

---

## Dependencies

| Package | Purpose |
|---|---|
| `rclpy` | ROS 2 Python client library |
| `std_msgs.msg.String` | ROS message type for all commands |
| `PyQt5` | GUI framework (widgets, layouts, timers) |
| `threading` | Runs `rclpy.spin()` in a background thread |

---

## Architecture

```
┌─────────────────────────────────────┐
│          FullControlGUI (QWidget)   │
│  ┌──────────────┐  ┌─────────────┐  │
│  │DiscreteServo │  │Incremental  │  │
│  │Box (Beaker)  │  │ServoBox     │  │
│  └──────────────┘  │(Test Tube)  │  │
│  ┌──────────────┐  └─────────────┘  │
│  │  Steppers    │  ┌─────────────┐  │
│  │  Box         │  │  Drill +    │  │
│  └──────────────┘  │  Auger Box  │  │
│                    └─────────────┘  │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐        │
│  │ P1 │ │ P2 │ │ P3 │ │ P4 │        │
│  └────┘ └────┘ └────┘ └────┘        │
└──────────────┬──────────────────────┘
               │ send_cmd(str)
               ▼
        ┌─────────────┐
        │  GUINode    │  (rclpy.Node)
        │  (ROS 2)    │
        └──────┬──────┘
               │ publishes to ROS topics
    ┌──────────┴───────────────────┐
    │  /rover/servo/beaker/cmd     │
    │  /rover/servo/test_tube/cmd  │
    │  /rover/steppers/cmd         │
    │  /rover/drill/cmd            │
    │  /rover/auger_shutter/cmd    │
    │  /rover/pumps/cmd            │
    └──────────────────────────────┘
```

---

## Classes

### `GUINode(Node)`

The ROS 2 node responsible for publishing commands to the rover.

**Publishers**

| Publisher | Topic | Message Type |
|---|---|---|
| `beaker_servo_pub` | `/rover/servo/beaker/cmd` | `std_msgs/String` |
| `test_tube_servo_pub` | `/rover/servo/test_tube/cmd` | `std_msgs/String` |
| `steppers_pub` | `/rover/steppers/cmd` | `std_msgs/String` |
| `drill_pub` | `/rover/drill/cmd` | `std_msgs/String` |
| `auger_pub` | `/rover/auger_shutter/cmd` | `std_msgs/String` |
| `pumps_pub` | `/rover/pumps/cmd` | `std_msgs/String` |

**Method: `send(cmd: str)`**

Routes a command string to the correct publisher based on its prefix:

| Command Prefix | Routed To |
|---|---|
| `BEAKER_SERVO` | `beaker_servo_pub` |
| `TESTTUBE_SERVO` | `test_tube_servo_pub` |
| `UP`, `DOWN`, `FWD`, `BACK`, `STOP STEPPERS` | `steppers_pub` |
| `DRILLM` | `drill_pub` |
| `AUGER` | `auger_pub` |
| `P` | `pumps_pub` |

---

### `PumpPanel(QFrame)`

A self-contained card widget for controlling a single pump.

**Constructor Parameters**
- `pump_id (int)` — Pump number (1–4)
- `send_cmd (callable)` — Callback to dispatch commands

**Commands Sent**
- `P{id} ON` — Activates the pump
- `P{id} OFF` — Deactivates the pump

---

### `DiscreteServoBox(QGroupBox)`

Servo control panel for absolute angle positioning (Beaker Carousel).

**Constructor Parameters**
- `title (str)` — GroupBox label
- `cmd_prefix (str)` — Command prefix (e.g., `BEAKER_SERVO`)
- `send_cmd (callable)` — Callback to dispatch commands

**UI Elements**
- Text input for manual angle entry (0–180°)
- Preset buttons: `0`, `45`, `90`, `135`, `180`
- **SEND** button and Enter key both trigger dispatch

**Command Format**
```
BEAKER_SERVO <angle>
```

---

### `IncrementalServoBox(QGroupBox)`

Servo control panel for relative (delta) angle adjustments (Test Tube Carousel).

**Constructor Parameters**
- `title (str)` — GroupBox label
- `cmd_prefix (str)` — Command prefix (e.g., `TESTTUBE_SERVO`)
- `send_cmd (callable)` — Callback to dispatch commands

**UI Elements**
- Text input showing current angle
- Delta buttons: `−22`, `−13`, `+13`, `+22`
- **SEND** button and Enter key both trigger dispatch

**Command Format**
```
TESTTUBE_SERVO <angle>
```

---

### `FullControlGUI(QWidget)`

The main application window. Composes all subsystem panels into a unified layout.

**Constructor Parameters**
- `ros_node (GUINode)` — The active ROS 2 node instance

**Key Attributes**

| Attribute | Type | Description |
|---|---|---|
| `selected_drill_speed` | `int` | Tracks current drill speed selection |
| `drill_speeds` | `list[int]` | Preset speed values |
| `timer` | `QTimer` | 200ms heartbeat timer |

**Layout Structure**
- Top row: Beaker Servo · Test Tube Servo · Steppers · Drill + Auger
- Bottom row: Pump 1 · Pump 2 · Pump 3 · Pump 4

---

#### Steppers Box

Controls two TB6600 stepper motors via delay-based speed.

| Button | Command Sent |
|---|---|
| UP | `UP SPEED <delay_µs>` |
| DOWN | `DOWN SPEED <delay_µs>` |
| FWD | `FWD SPEED <delay_µs>` |
| BACK | `BACK SPEED <delay_µs>` |
| STOP STEPPERS | `STOP STEPPERS` |

- Delay slider range: **50–800 µs** (lower = faster)
- Separate sliders for Drill Stepper (UP/DOWN) and Gantry Stepper (FWD/BACK)

---

#### Drill Motor Box

Controls a DC drill motor (MD10C driver).

| Button | Command Sent |
|---|---|
| ON | `DRILLM SPEED <-selected_speed>` |
| OFF | `DRILLM SPEED 0` |

- Speed slider range: **−255 to +255**
- Note: Speed is **inverted** at send time (`-selected_drill_speed`) to match hardware polarity convention

**Direction Label Logic**

| Effective Speed | Label |
|---|---|
| > 0 | FWD |
| < 0 | REV |
| 0 | STOP |

---

#### Auger Shutter Box

Controls a BTS7960 motor driver for the auger shutter.

| Button | Command Sent |
|---|---|
| OPEN | `AUGER OPEN` |
| CLOSE | `AUGER CLOSE` |
| STOP | `AUGER STOP` |

---

## Command Reference

| Command | Description |
|---|---|
| `BEAKER_SERVO <0–180>` | Move beaker carousel servo to absolute angle |
| `TESTTUBE_SERVO <0–180>` | Move test tube carousel servo to absolute angle |
| `UP SPEED <µs>` | Run drill stepper upward at given delay |
| `DOWN SPEED <µs>` | Run drill stepper downward at given delay |
| `FWD SPEED <µs>` | Run gantry stepper forward at given delay |
| `BACK SPEED <µs>` | Run gantry stepper backward at given delay |
| `STOP STEPPERS` | Halt both steppers |
| `DRILLM SPEED <-255–255>` | Set drill motor speed and direction |
| `AUGER OPEN` | Open the auger shutter |
| `AUGER CLOSE` | Close the auger shutter |
| `AUGER STOP` | Stop the auger motor |
| `P<1–4> ON` | Turn pump on |
| `P<1–4> OFF` | Turn pump off |

---

## Entry Point

```python
def main():
    rclpy.init()
    node = GUINode()
    threading.Thread(target=rclpy.spin, args=(node,), daemon=True).start()
    app = QApplication(sys.argv)
    gui = FullControlGUI(node)
    gui.show()
    sys.exit(app.exec_())
```

`rclpy.spin()` runs on a **daemon thread** so it exits automatically when the GUI closes. The Qt event loop runs on the **main thread**.

---

## Running

```bash
python3 gui_node.py
```

Ensure a ROS 2 environment is sourced before running:

```bash
source /opt/ros/<distro>/setup.bash
python3 gui_node.py
```
