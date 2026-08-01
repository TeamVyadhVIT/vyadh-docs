---
title: "AstroBIO Rover — Arduino Firmware"
date: 2026-03-05
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, astrobio, firmware, arduino]
---

## Overview

This firmware runs on an **Arduino Mega 2560** and controls all electromechanical subsystems of the AstroBIO rover's science module. Commands are received over **Serial (9600 baud)** as plain-text strings terminated by a newline (`\n`), typically relayed from the ROS 2 bridge node.

---

## Hardware Summary

| Subsystem | Driver | Interface |
|---|---|---|
| Pumps 1 & 2 | L298N Driver 1 | Digital (EN + IN pins) |
| Pumps 3 & 4 | L298N Driver 2 | Digital (EN + IN pins) |
| Drill Stepper (Z-axis) | TB6600 | STEP/DIR digital pulses |
| Gantry Stepper (X-axis) | TB6600 | STEP/DIR digital pulses |
| Beaker Carousel Servo | — | PWM via `Servo.h` |
| Test Tube Carousel Servo | — | PWM via `Servo.h` |
| Auger Shutter Motor | BTS7960 | Dual PWM (LPWM / RPWM) |
| Drill Motor | MD10C | PWM + Direction pin |

---

## Pin Assignments

### L298N Driver 1 — Pumps 1 & 2

| Pin | Arduino | Function |
|---|---|---|
| ENA1 | 3 | Pump 1 Enable (PWM) |
| IN1A | 22 | Pump 1 Direction A |
| IN1B | 24 | Pump 1 Direction B |
| ENB1 | 4 | Pump 2 Enable (PWM) |
| IN2A | 26 | Pump 2 Direction A |
| IN2B | 28 | Pump 2 Direction B |

### L298N Driver 2 — Pumps 3 & 4

| Pin | Arduino | Function |
|---|---|---|
| ENA2 | 5 | Pump 3 Enable (PWM) |
| IN3A | 31 | Pump 3 Direction A |
| IN3B | 33 | Pump 3 Direction B |
| ENB2 | 6 | Pump 4 Enable (PWM) |
| IN4A | 35 | Pump 4 Direction A |
| IN4B | 37 | Pump 4 Direction B |

### TB6600 Steppers

| Pin | Arduino | Function |
|---|---|---|
| STEP1 | 40 | Drill stepper pulse |
| DIR1 | 41 | Drill stepper direction |
| STEP2 | 42 | Gantry stepper pulse |
| DIR2 | 43 | Gantry stepper direction |

### Servos

| Pin | Arduino | Function |
|---|---|---|
| SERVO_BEAKER_PIN | 13 | Beaker carousel servo signal |
| SERVO_TESTTUBE_PIN | 9 | Test tube carousel servo signal |

### BTS7960 — Auger Shutter

| Pin | Arduino | Function |
|---|---|---|
| AUGER_LPWM | 8 | Left PWM (open direction) |
| AUGER_RPWM | 7 | Right PWM (close direction) |

### MD10C — Drill Motor

| Pin | Arduino | Function |
|---|---|---|
| DRILL_PWM | 11 | Motor speed (PWM) |
| DRILL_DIR | 53 | Motor direction |

---

## Global State Variables

| Variable | Type | Default | Description |
|---|---|---|---|
| `run1` | `bool` | `false` | Whether drill stepper is currently running |
| `run2` | `bool` | `false` | Whether gantry stepper is currently running |
| `speed1` | `int` | `200` | Drill stepper step interval (µs) |
| `speed2` | `int` | `300` | Gantry stepper step interval (µs) |
| `lastStep1` | `unsigned long` | `0` | Timestamp of last drill stepper pulse (µs) |
| `lastStep2` | `unsigned long` | `0` | Timestamp of last gantry stepper pulse (µs) |
| `augerState` | `AugerState` | `AUGER_IDLE` | Current auger motor state |
| `augerStartTime` | `unsigned long` | `0` | Timestamp when auger motion started (ms) |
| `AUGER_PWM` | `int` | `255` | Auger motor power (full) |
| `AUGER_MOVE_TIME_MS` | `unsigned long` | `600` | Duration of timed auger move (ms) |

---

## Initialization — `setup()`

1. Configures all motor driver pins as `OUTPUT`
2. Sets pump motor direction pins to a default forward state (`IN_A = HIGH`, `IN_B = LOW`)
3. Disables all pump enable lines (motors off at startup)
4. Attaches both servos and moves them to 0°
5. Initializes auger PWM outputs to 0
6. Initializes drill PWM to 0 and direction to `LOW`
7. Starts `Serial` at **9600 baud**

---

## Main Loop — `loop()`

The loop performs two concurrent tasks:

### 1. Serial Command Parsing

Reads a newline-terminated command string from Serial, trims whitespace, and dispatches to the appropriate handler.

### 2. Non-Blocking Stepper Stepping

Uses `micros()` timestamps to fire step pulses at the configured interval without blocking:

```cpp
if (run1 && now - lastStep1 >= speed1) {
    lastStep1 = now;
    pulseStep(STEP1);
}
```

The same pattern applies to stepper 2 (gantry).

### 3. Timed Auger Auto-Stop

If the auger is moving and `AUGER_MOVE_TIME_MS` has elapsed, `augerStop()` is called automatically:

```cpp
if (augerState != AUGER_IDLE && millis() - augerStartTime >= AUGER_MOVE_TIME_MS) {
    augerStop();
}
```

---

## Command Protocol

All commands are plain ASCII strings sent over Serial at 9600 baud, terminated with `\n`.

### Servo Commands

| Command | Action |
|---|---|
| `BEAKER_SERVO <angle>` | Move beaker carousel servo to `angle` (0–180°) |
| `TESTTUBE_SERVO <angle>` | Move test tube carousel servo to `angle` (0–180°) |

Both angles are clamped via `constrain(angle, 0, 180)`.

---

### Stepper Commands

| Command | Action |
|---|---|
| `UP SPEED <µs>` | Run drill stepper upward; step interval = `<µs>` |
| `DOWN SPEED <µs>` | Run drill stepper downward; step interval = `<µs>` |
| `FWD SPEED <µs>` | Run gantry stepper forward; step interval = `<µs>` |
| `BACK SPEED <µs>` | Run gantry stepper backward; step interval = `<µs>` |
| `STOP STEPPERS` | Halt both steppers (`run1 = run2 = false`) |

> **Speed Note:** The interval is in **microseconds between steps** — a lower value means faster rotation.

**Parsing offsets used:**

| Command | `substring()` offset | Reason |
|---|---|---|
| `UP SPEED` | 9 | `"UP SPEED "` = 9 chars |
| `DOWN SPEED` | 11 | `"DOWN SPEED "` = 11 chars |
| `FWD SPEED` | 10 | `"FWD SPEED "` = 10 chars |
| `BACK SPEED` | 11 | `"BACK SPEED "` = 11 chars |

---

### Drill Motor Commands

| Command | Action |
|---|---|
| `DRILLM SPEED <value>` | Set drill motor speed; range −255 to +255 |

Speed is parsed from offset 13 (`"DRILLM SPEED "` = 13 chars) and clamped to [−255, 255]. Values with absolute magnitude < 5 are treated as 0 (dead-band).

**Direction logic:**

| Speed | DIR pin | PWM |
|---|---|---|
| > 0 | HIGH (forward) | `speed` |
| < 0 | LOW (reverse) | `-speed` |
| 0 | — | 0 (off) |

---

### Pump Commands

| Command | Action |
|---|---|
| `P<1–4> ON` | Enable pump (raise EN pin HIGH) |
| `P<1–4> OFF` | Disable pump (pull EN pin LOW) |

Pump number is parsed from `cmd.charAt(1)`. Direction pins remain fixed (set in `setup()`), so pump speed is always full when enabled.

---

### Auger Shutter Commands

| Command | Action |
|---|---|
| `AUGER OPEN` | Drive LPWM=255, RPWM=0; start timer |
| `AUGER CLOSE` | Drive LPWM=0, RPWM=255; start timer |
| `AUGER STOP` | Set both PWM outputs to 0; state → `AUGER_IDLE` |

The auger runs for `AUGER_MOVE_TIME_MS` (default 600 ms) then stops automatically. Tune this value to match the physical ~180° travel of the shutter.

---

## Helper Functions

### `pulseStep(int pin)`

Generates a single STEP pulse (HIGH → 2 µs delay → LOW) on the given pin. Called by the non-blocking stepper logic in `loop()`.

```cpp
void pulseStep(int pin) {
    digitalWrite(pin, HIGH);
    delayMicroseconds(2);
    digitalWrite(pin, LOW);
}
```

### `pumpOn(int p)` / `pumpOff(int p)`

Enables or disables the EN pin for pump `p` (1–4). Direction is fixed forward.

### `augerOpen()` / `augerClose()` / `augerStop()`

Manages the BTS7960 dual-PWM output and updates `augerState` + `augerStartTime`.

---

## Auger State Machine

```
         AUGER OPEN cmd          AUGER CLOSE cmd
              │                        │
              ▼                        ▼
        AUGER_OPENING           AUGER_CLOSING
              │                        │
     timeout or STOP          timeout or STOP
              └──────────┬─────────────┘
                         ▼
                    AUGER_IDLE
```

---

## Notes & Tuning

- **Stepper speed:** Lower µs interval = faster. The TB6600 requires a minimum pulse width; do not set below ~50 µs.
- **Auger timing:** `AUGER_MOVE_TIME_MS = 600` is an initial estimate. Adjust for your physical shutter travel.
- **Drill dead-band:** Speeds with `abs(speed) < 5` are zeroed to prevent motor stutter at near-zero commands.
- **Serial baud rate:** Both firmware and ROS bridge must be set to **9600 baud** (can be increased if latency is a concern).
