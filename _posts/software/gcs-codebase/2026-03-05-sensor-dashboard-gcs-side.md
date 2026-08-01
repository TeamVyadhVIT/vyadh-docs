---
title: "Sensor Dashboard — GCS Side"
date: 2026-03-05
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, sensors, mqtt]
---

## Overview

A **PyQt5 + Matplotlib** desktop application that subscribes to MQTT topics published by the serial bridge and displays live, scrolling plots for all sensor channels. Includes a **CSV logging toggle** for data recording.

---

## Dependencies

| Package | Purpose |
|---|---|
| `PyQt5` | GUI framework (main window, widgets, timer) |
| `matplotlib` | Embedded live plots (Qt5Agg backend) |
| `paho-mqtt` | Subscribing to sensor data from the broker |
| `csv` | Writing sensor readings to file |
| `time`, `math` | Timestamps and VOC calculation |

Install with:

```bash
pip install PyQt5 matplotlib paho-mqtt
```

---

## Architecture

```
MQTT Broker (192.168.1.25:1883)
        │
        │  Subscribed topics:
        │  _dev_ttyUSB0/esp2/#
        │  _dev_ttyUSB1/esp2/#
        ▼
   on_message()  ←── paho MQTT thread (loop_start)
        │
        │  Updates shared `data` dict
        ▼
   QTimer (500ms)  ←── Qt main thread
        │
        │  Reads `data` dict
        ▼
   update_plots()
        │
        ├── Redraws 10 line plots (one per sensor)
        └── Redraws 1 spectral bar chart
```

---

## Global State

### `sensors` list

```python
sensors = [
    'temp', 'humidity', 'pressure', 'gas',
    'nitrogen', 'phosphorous', 'potassium',
    'gas2', 'gas3', 'gas4'
]
```

### `data` dictionary

Each sensor key maps to a rolling buffer:

```python
data = {
    'temp':     { "values": [], "time": [] },
    'humidity': { "values": [], "time": [] },
    ...
    'spectral': { "values": [0]*8, "time": [] }
}
```

| Parameter | Value |
|---|---|
| `max_points` | 100 — maximum samples retained per sensor |
| `start_time` | Epoch time at launch, used to compute elapsed seconds |
| `logging_enabled` | `False` by default |

---

## CSV Logging

A file `sensor_data.csv` is created on startup with a header row. When logging is active, one row is appended per incoming MQTT message containing the latest value of every sensor channel.

**Header:**
```
Temperature, Humidity, Pressure, Gas, Nitrogen, Phosphorous, Potassium, Gas2, Gas3, Gas4, Spectral_Data
```

**Spectral column** is stored as a comma-separated string of 8 values within the cell.

---

## `SensorDashboard` Class

Extends `QMainWindow`. Builds the full GUI and drives the plot refresh cycle.

### Layout

```
┌─────────────────────────────────────────────┐
│  [ Start Logging ]                          │
├─────────────────────────────────────────────┤
│  temp  humidity  pressure  gas  nitrogen    │
│  phosphorous  potassium  gas2  gas3  gas4   │
│  spectral                                   │
├─────────────────────────────────────────────┤
│                                             │
│         3×4 Matplotlib subplot grid         │
│   (10 line plots + 1 spectral bar chart)    │
│                                             │
└─────────────────────────────────────────────┘
```

### `__init__()`

- Sets window to **1400×800 px**
- Applies dark theme stylesheet (see Theme section)
- Creates `QLabel` status indicators for each sensor in a `QGridLayout` (6 per row)
- Embeds a **3×4 Matplotlib figure** via `FigureCanvasQTAgg`
- Starts a `QTimer` at **500 ms** intervals connected to `update_plots()`

### `toggle_logging()`

Flips `logging_enabled` and updates the button label between `"Start Logging"` and `"Stop Logging"`.

### `update_plots()`

Called every 500 ms by the Qt timer. For each of the 10 sensors:

1. Clears the corresponding subplot
2. Re-applies dark theme (facecolor, tick colors, grid)
3. Plots `data[sensor]["values"]` vs `data[sensor]["time"]` in accent blue
4. Sets axis labels, title, and tick locators (max 5 ticks per axis)

For the **spectral channel** (subplot index 10):

- Renders a **bar chart** with 8 colored bars, one per wavelength band
- Wavelength labels: `415, 445, 480, 515, 555, 590, 630, 680` nm
- Bar colors (violet → red): `#7c3aed, #4f46e5, #2563eb, #16a34a, #84cc16, #facc15, #fb923c, #ef4444`

---

## MQTT Subscription

### Connection

```python
client.connect("192.168.1.25", 1883, 60)
```

### Subscribed Topics

```python
client.subscribe("_dev_ttyUSB0/esp2/#")
client.subscribe("_dev_ttyUSB1/esp2/#")
```

The sensor key is extracted from the **last segment** of the topic path:

```python
sensor = msg.topic.split("/")[-1]
```

### `on_message()` Callback

Runs on the **paho background thread** — updates the shared `data` dict directly (no Qt calls):

- **Spectral:** payload is a comma-separated string of 8 floats → stored as list in `data["spectral"]["values"]`
- **All other sensors:** payload is a single float → appended to rolling buffer; oldest value dropped when `max_points` is exceeded

If `logging_enabled`, appends the current snapshot of all sensor values to `sensor_data.csv`.

---

## Theme

| Token | Hex | Used For |
|---|---|---|
| `BG` | `#05060a` | Main window background |
| `CARD` | `#0b1220` | Button background |
| `CARD_IN` | `#0f172a` | Label and plot background |
| `TEXT` | `#e5e7eb` | All text and tick labels |
| `ACCENT` | `#38bdf8` | Line plot color, button hover |
| `GRID` | `#1e293b` | Axis grid lines and spine color |

---

## Running

```bash
python3 sensor_dashboard.py
```

The MQTT client connects and begins receiving immediately. The dashboard will show `--` on labels and empty plots until data arrives.

---

## Notes

- `matplotlib.use("Qt5Agg")` must be called **before** importing `pyplot` to avoid backend conflicts with PyQt5
- The `main_layout.setStretch()` calls ensure the canvas expands to fill the window while the button and labels remain fixed height
- `client.loop_start()` runs the MQTT receive loop on a **background thread**, keeping the Qt event loop unblocked
- The `data` dict is accessed from both the MQTT thread (writes) and the Qt timer thread (reads) — for production use, protect shared state with a `threading.Lock()`
- The topic subscription pattern (`_dev_ttyUSB0/esp2/#`) should be updated to match the actual topic structure published by the serial bridge (`esp_sensors/esp1/...`)
