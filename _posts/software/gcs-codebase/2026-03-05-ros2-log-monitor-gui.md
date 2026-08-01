---
title: "ROS2 Log Monitor GUI"
date: 2026-03-05
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, ros2, logs]
---

## Overview

A full-screen **PyQt5 desktop application** that provides real-time monitoring of a ROS 2 system. It subscribes to `/rosout` for log messages and a comprehensive set of `/system/*` diagnostic topics published by a companion system monitor node. All data is visualised in a GitHub-dark-themed dashboard with live graphs, htop-style bars, analytics charts, and a searchable, filterable log table.

---

## Dependencies

| Package | Purpose |
|---|---|
| `rclpy` | ROS 2 Python client library |
| `rcl_interfaces.msg.Log` | `/rosout` log message type |
| `diagnostic_msgs.msg.DiagnosticArray` | System diagnostic message type |
| `std_msgs.msg.Float32` | CPU / memory / disk / GPU percentage topics |
| `std_msgs.msg.UInt32` | System uptime in seconds |
| `PyQt5` | GUI framework — widgets, painting, threading |
| `matplotlib` (Qt5Agg) | Embedded charts in the Analytics tab |
| `collections.deque` | Rolling fixed-length data buffers |
| `collections.defaultdict` | Auto-initialised statistics counters |
| `re` | Regex-based log filtering |
| `datetime` | Timestamp generation |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        LogMonitorGUI (QMainWindow)               │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │gHeader: Stat Cards (Total / Errors / Warns / Nodes / Uptime)│  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────┐  ┌───────────────────────────────────────┐ │
│  │  Left Column     │  │  Right Column                         │ │
│  │  System Vitals   │  │  Search + Filter Bar                  │ │
│  │  CPU/Mem/Disk/GPU│  │                                       │ │
│  │  Progress Bars   │  │  ┌─────────┬───────────┬───────────┐  │ │
│  │                  │  │  │Logs  │Analytics│Diagnost.       │  │ │
│  │  System Info     │  │  │ Table   │ Charts     │htop bars │  │ │
│  │  Panel           │  │  └─────────┴───────────┴───────────┘  │ │
│  └──────────────────┘  └───────────────────────────────────────┘ │
│                                                                  │
│  Status Bar                                                      │
└──────────────────────────────────────────────────────────────────┘
         ▲
         │ Qt signals (pyqtSignal)
         │
┌────────┴──────────────┐
│     ROS2Thread        │  (QThread — background)
│  ┌─────────────────┐  │
│  │ LogMonitorNode  │  │  (rclpy.Node)
│  │ rclpy.spin_once │  │
│  └─────────────────┘  │
└───────────────────────┘
         ▲
         │ ROS 2 DDS
    /rosout, /system/*
```

---

## Classes

---

### `ROS2Thread(QThread)`

Runs the ROS 2 event loop in a **background thread** so it never blocks the Qt main thread.

**Signals emitted to the GUI thread:**

| Signal | Type | Description |
|---|---|---|
| `log_received` | `object` | A `/rosout` `Log` message |
| `diagnostic_received` | `object` | A `DiagnosticArray` from legacy topics |
| `cpu_received` | `float` | CPU usage percentage |
| `mem_received` | `float` | Memory usage percentage |
| `disk_received` | `float` | Disk usage percentage |
| `gpu_received` | `float` | GPU utilisation percentage |
| `uptime_received` | `int` | System uptime in seconds |
| `diagnostics_array_received` | `object` | Detailed `DiagnosticArray` from system monitor |

**`run()`** — Initialises `rclpy`, creates a `LogMonitorNode`, then loops `rclpy.spin_once()` every 100 ms until stopped.

**`stop()`** — Sets `self.running = False` to cleanly exit the loop.

---

### `LogMonitorNode(Node)`

ROS 2 node that subscribes to all monitoring topics and re-emits data via the parent thread's Qt signals.

**Node name:** `log_monitor_gui`

**Subscriptions:**

| Topic | Type | Purpose |
|---|---|---|
| `/rosout` | `Log` | All ROS 2 node log messages (queue: 100) |
| `/diagnostics` | `DiagnosticArray` | Standard diagnostics |
| `/diagnostics_agg` | `DiagnosticArray` | Aggregated diagnostics |
| `/system/network` | `DiagnosticArray` | Network diagnostics |
| `/system/thermal` | `DiagnosticArray` | Thermal diagnostics |
| `/system/cpu/usage` | `Float32` | Overall CPU % |
| `/system/memory/usage` | `Float32` | Memory usage % |
| `/system/disk/usage` | `Float32` | Disk usage % |
| `/system/gpu/utilization` | `Float32` | GPU utilisation % |
| `/system/uptime` | `UInt32` | Seconds since boot |
| `/system/cpu/cores` | `DiagnosticArray` | Per-core CPU usage |
| `/system/memory/detailed` | `DiagnosticArray` | RAM/cache/swap breakdown |
| `/system/disk/detailed` | `DiagnosticArray` | Per-partition disk info |
| `/system/network/detailed` | `DiagnosticArray` | Overall network throughput |
| `/system/network/interfaces` | `DiagnosticArray` | Per-interface stats |
| `/system/gpu/memory` | `DiagnosticArray` | GPU memory breakdown |
| `/system/battery` | `DiagnosticArray` | Battery charge and status |
| `/system/processes/top` | `DiagnosticArray` | Top 5 processes by CPU |
| `/system/io/stats` | `DiagnosticArray` | Disk read/write throughput |

---

### `GraphWidget(QWidget)`

A custom-painted real-time **line graph** resembling htop's CPU history view.

**Constructor Parameters**
- `title (str)` — Label displayed at top-left
- `color (str)` — Hex colour for the line and fill
- `max_points (int)` — Rolling buffer length (default: 60)

**`add_value(value)`** — Appends a new data point and triggers a repaint.

**`paintEvent()`** — Draws:
- Dark background (`#1e1e2e`)
- Title and current percentage (colour-coded: green < 60%, orange 60–80%, red > 80%)
- Dotted horizontal grid lines (5 divisions)
- Filled area under the line with 50% alpha
- Solid line in `self.color`

---

### `PieChartWidget(QWidget)`

A custom-painted **pie chart** for log level distribution in the Analytics tab.

**`set_data(data_dict)`** — Accepts `{label: count}` and triggers repaint.

**`paintEvent()`** — Draws:
- Pie slices proportional to count
- Legend below the chart with percentage labels
- Black (`#11111b`) separator lines between slices

---

### `BarChartWidget(QWidget)`

A custom-painted **horizontal bar chart** showing the top 10 most active ROS nodes.

**`set_data(data_dict)`** — Sorts by count descending, keeps top 10, triggers repaint.

**`paintEvent()`** — Draws:
- Node names on the left (truncated to 25 characters)
- Proportional coloured bars (10 rotating Catppuccin Mocha colours)
- Count values to the right of each bar

---

### `TimelineWidget(QWidget)`

A **bar chart timeline** showing log messages per second over the last 60 seconds.

**`add_count(count)`** — Appends a new per-second count.

**`paintEvent()`** — Draws vertical bars colour-coded by activity level: blue (low), orange (medium > 40% of max), red (high > 70% of max).

---

### `HTOPStyleBar(QWidget)`

A single **htop-style labelled progress bar** used throughout the Diagnostics tab.

**Constructor Parameters**
- `label_text (str)` — Fixed-width label on the left
- `color (str)` — Bar fill colour

**`set_value(value)`** — Updates the percentage and triggers repaint.

**`paintEvent()`** — Draws:
- Monospace label (120px wide)
- Background bar (`#313244`)
- Fill bar (colour switches to orange > 60%, red > 80%)
- Right-aligned percentage value

---

### `LogMonitorGUI(QMainWindow)`

The main application window. Composes all panels, manages data storage, starts the ROS thread, and drives the statistics timer.

---

## GUI Layout

### Header — Stat Cards

Five cards displayed across the top:

| Card | Metric | Colour |
|---|---|---|
| 📋 Total | Total log count | Blue `#58a6ff` |
| ❌ Errors | ERROR + FATAL count | Red `#f85149` |
| ⚠️ Warns | WARN count | Yellow `#d29922` |
| 🔗 Nodes | Unique active node count | Green `#3fb950` |
| ⏱️ Uptime | System uptime `HH:MM:SS` | Purple `#a371f7` |

---

### Left Column — System Vitals (max 380px wide)

Four compact metric cards (CPU, Memory, Disk, GPU), each showing:
- Name and current percentage value
- A slim 6px `QProgressBar` with colour-coded fill
- Threshold colours: green → orange (> 60%) → red (> 80%)

Below these, a **System Info** panel displays:
- Hostname, OS + kernel, architecture, logical/physical core count

(Populated from `System/Info` diagnostic status values.)

---

### Right Column — Search, Filters, Tabs

#### Search and Filter Bar

- **Search box** — real-time text filter applied across all 5 table columns simultaneously
- **Level checkboxes** — individually toggle visibility of DEBUG / INFO / WARN / ERROR / FATAL
- **Node filter input** — type a node name then:
  - `+ Include` — show ONLY logs from this node
  - `- Exclude` — hide all logs from this node
- **Active filters label** — compact summary of all active filter rules

#### Tab 1 — 📋 Logs

A `QTableWidget` with 5 columns:

| Column | Width | Style |
|---|---|---|
| Time | 100px | Dimmed `#6c7086`, 10pt |
| Level | 80px | Bold, level colour, 12pt |
| Node | 220px | Cyan bold `#89dceb` |
| Function | 180px | Muted `#a6adc8`, 10pt |
| Message | Stretch | Light `#cdd6f4`, 11pt |

- Maximum 1000 rows retained (oldest removed automatically)
- Auto-scrolls to bottom unless filtered
- Monospace font (`Consolas`) throughout

#### Tab 2 — 📊 Analytics

- **Log Distribution** — `PieChartWidget` showing share of each log level
- **Top Active Nodes** — `BarChartWidget` for top 10 nodes by message count
- **Activity Timeline** — `TimelineWidget` showing logs/second over last 60s
- **Most Frequent Errors** — table of top 10 repeated error messages with occurrence count

#### Tab 3 — 🔧 Diagnostics

**Status Cards Row** (5 cards, colour left-border accented):
CPU · Memory · Disk · Network · Thermal

**Left sub-column:**
- **CPU Cores** — dynamically created `HTOPStyleBar` per core (auto-scales to core count)
- **Memory Breakdown** — Used / Cache / Swap bars
- **Disk I/O** — Read / Write bars (scaled to 100 MB/s max)
- **Network Interfaces** — per-interface TX/RX throughput with status icon

**Right sub-column:**
- **Top 5 Processes** — table with PID, name, CPU%, Mem% (CPU% colour-coded)
- **Temperature Sensors** — dynamically created bars per thermal zone (scaled to 100°C)
- **GPU Status** — Utilisation + Memory bars + temperature label
- **Battery Status** — Charge bar + status text with charging/discharging icons

---

## Data Storage

| Attribute | Type | Capacity | Description |
|---|---|---|---|
| `all_logs` | `deque` | 10,000 | All received log messages as dicts |
| `cpu_history` | `deque` | 60 | Rolling CPU % samples |
| `mem_history` | `deque` | 60 | Rolling memory % samples |
| `disk_history` | `deque` | 60 | Rolling disk % samples |
| `gpu_history` | `deque` | 60 | Rolling GPU % samples |
| `log_counts` | `defaultdict(int)` | — | Per-level total counts |
| `node_counts` | `defaultdict(int)` | — | Per-node total counts |
| `error_messages` | `defaultdict(int)` | — | Per-message error frequency |
| `logs_per_second` | `deque` | 60 | Log rate timeline data |
| `detailed_diagnostics` | `dict` | — | Nested category → name → data |

---

## Log Level Reference

| ROS Level Code | Name | Colour |
|---|---|---|
| 10 | DEBUG | `#89b4fa` (blue) |
| 20 | INFO | `#a6e3a1` (green) |
| 30 | WARN | `#f9e2af` (yellow) |
| 40 | ERROR | `#fab387` (orange) |
| 50 | FATAL | `#f38ba8` (red) |

---

## Key Methods

### `add_log(msg)`

Called on every `/rosout` message. Extracts timestamp, level, node, function, and message text into a dict, appends to `all_logs`, updates all counters, and calls `add_row_to_table()` if the log passes current filters.

### `add_row_to_table(table, log_data)`

Inserts a styled row into the given `QTableWidget`. Applies the active search filter to the new row immediately. Enforces the 1000-row cap by removing the oldest row.

### `add_detailed_diagnostic(msg)`

Processes `DiagnosticArray` messages from the system monitor. Organises statuses into `self.detailed_diagnostics[category][name]` and triggers `update_diagnostic_visualizations()`.

### `update_diagnostic_visualizations()`

Dispatcher that calls all eight individual updaters: status cards, CPU cores, memory, disk I/O, network interfaces, processes, thermal sensors, GPU, and battery.

### `update_cpu_cores()`

Parses per-core usage values from diagnostic key-value pairs using regex to extract core numbers. Dynamically creates or destroys `HTOPStyleBar` instances to match the actual core count.

### `filter_logs()`

Iterates every row in the log table and hides/shows it based on whether the search text appears in any column. Connected to `search_box.textChanged` for real-time filtering.

### `passes_advanced_filters(log_data)`

Returns `True` if a log dict satisfies all four active filter rules in order: node include list → node exclude list → level checkboxes → regex pattern.

### `apply_advanced_filters()`

Reapplies all advanced filters to every existing row in all log tables. Called when filter checkboxes or node filter rules change.

### `update_statistics()`

Called every 1000 ms by `stats_timer`. Updates all stat cards, pie chart, bar chart, timeline, and top errors table from the accumulated counter dicts.

### `update_uptime(seconds)`

Converts a raw second count to `HH:MM:SS` format and updates the uptime stat card.

### `closeEvent(event)`

Stops the ROS thread cleanly before closing (`ros_thread.stop()` + `ros_thread.wait()`).

---

## Statistics Timer

```python
self.stats_timer = QTimer()
self.stats_timer.timeout.connect(self.update_statistics)
self.stats_timer.start(1000)   # Every 1 second
```

Drives all aggregate chart and card updates independently of incoming message rate.

---

## Theme

The application uses the **GitHub Dark** colour palette for the overall chrome and **Catppuccin Mocha** accents for data visualisation:

| Role | Colour |
|---|---|
| Window background | `#0d1117` |
| Card / panel background | `#161b22` |
| Input background | `#0d1117` |
| Border / grid | `#30363d` |
| Primary text | `#e6edf3` |
| Dimmed text | `#8b949e` |
| Accent / link blue | `#58a6ff` |
| Plot background | `#1e1e2e` |
| Plot text | `#cdd6f4` |

---

## Running

```bash
source /opt/ros/<distro>/setup.bash
python3 ros2_log_monitor.py
```

The window opens maximised. It begins receiving logs immediately if any ROS 2 nodes are publishing to `/rosout`. System metric cards and diagnostic panels populate once the companion `RoverSystemMonitor` node is running and publishing to the `/system/*` topics.

---

## Known Limitations & Notes

- `clear_all_logs()` references `self.debug_table`, `self.info_table` etc. which are not instantiated in the current layout — these references will raise `AttributeError` if triggered. The per-level tab tables were removed from the layout but the clear method was not updated.
- `filter_logs()` contains a reference to a local variable `table` in the debug print that is out of scope — remove or fix before production use.
- The `detailed_diagnostics` dict is written from the ROS background thread and read from the Qt main thread without a lock. For production robustness, protect it with `threading.Lock()`.
- Exact-value axis checks in the diagnostics parser (`== 1`, `== 0`) may miss values due to floating-point precision; use threshold comparisons (`> 0.9`) for reliability.
- `check_diagnostic_topics()` uses `subprocess.run(['ros2', 'topic', 'list'])` which requires `ros2` to be on the system PATH.
