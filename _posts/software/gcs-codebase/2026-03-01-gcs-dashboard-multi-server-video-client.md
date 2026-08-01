---
title: "GCS Dashboard — Multi-Server Video Client"
date: 2026-03-01
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, video, python]
---

## Overview

This is the main Ground Control Station dashboard application. It is a Python desktop GUI that runs on the GCS laptop and does three things simultaneously:

1. Maintains persistent **WebSocket connections** to one or more onboard servers (laptop, Raspberry Pi) to send camera control commands (start, stop, restart, invert, resolution change).
2. Receives **H.264 RTP video streams** from each camera over UDP and renders them in real time using GStreamer.
3. Displays all camera feeds in a **grid layout** built with PySide6 (Qt), with per-camera controls.

The application is designed to be hardened — WebSocket connections auto-reconnect with exponential backoff, GStreamer pipelines auto-restart on error, and all cross-thread communication is handled safely.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     GCS DASHBOARD (this file)                    │
│                                                                   │
│  ┌──────────────┐    ┌──────────────────────────────────────┐   │
│  │  Qt Main     │    │  Background Thread                    │   │
│  │  Thread      │    │  (asyncio event loop)                 │   │
│  │              │    │                                        │   │
│  │  Dashboard   │    │  websocket_handler(laptop_uri)        │   │
│  │  └─ Grid     │    │  websocket_handler(pi_uri)            │   │
│  │     ├─ Cam0  │◄───┤  (auto-reconnect, exponential        │   │
│  │     ├─ Cam1  │    │   backoff)                            │   │
│  │     ├─ Cam2  │    │                                        │   │
│  │     └─ ...   │    │  send_message_to_server()             │   │
│  │              │───►│  (called via run_coroutine_threadsafe)│   │
│  └──────────────┘    └──────────────────────────────────────┘   │
│         │                                                         │
│  ┌──────┴──────────────────────────────────────────────────┐    │
│  │  GStreamer (per camera)                                    │    │
│  │  udpsrc → rtph264depay → avdec_h264 → videoconvert        │    │
│  │  → videoscale → xvimagesink (rendered into Qt label)      │    │
│  └───────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
         ▲                            ▲
         │ WebSocket (commands)       │ UDP RTP (video)
         ▼                            ▼
┌──────────────────┐       ┌─────────────────────┐
│  Laptop Server   │       │  Pi Server           │
│  192.168.1.25    │       │  192.168.1.27        │
│  ws port 8765    │       │  ws port 8765        │
│  video ports     │       │  video ports         │
│  5000, 5001      │       │  5100, 5101, 5102,   │
│                  │       │  5103                │
└──────────────────┘       └─────────────────────┘
```

---

## Dependencies

| Library | Purpose | Install |
|---|---|---|
| `PySide6` | Qt GUI framework — windows, widgets, layouts, signals | `pip install PySide6` |
| `websockets` | Async WebSocket client | `pip install websockets` |
| `gi` (PyGObject) | Python bindings for GStreamer | `sudo apt install python3-gi gir1.2-gst-1.0` |
| `Gst` (via gi) | GStreamer pipeline — video decode and render | Included with PyGObject |
| `GstVideo` (via gi) | GStreamer video overlay API | Included with PyGObject |
| `GLib` (via gi) | GLib main context — required for GStreamer event processing | Included with PyGObject |
| `asyncio` | Async event loop for WebSocket handling | Python stdlib |
| `threading` | Background thread for asyncio loop | Python stdlib |
| `json`, `logging`, `time` | Payload serialisation, logging, timing | Python stdlib |

### Full Installation

```bash
# Qt and WebSocket
pip install PySide6 websockets

# GStreamer Python bindings and plugins
sudo apt install \
    python3-gi \
    python3-gst-1.0 \
    gir1.2-gst-1.0 \
    gstreamer1.0-plugins-base \
    gstreamer1.0-plugins-good \
    gstreamer1.0-plugins-bad \
    gstreamer1.0-libav \
    gstreamer1.0-x
```

> **GStreamer on non-Linux**
>
> The pipeline uses `xvimagesink` which is X11-specific and only works on Linux. On other platforms, replace `xvimagesink` with `autovideosink` (cross-platform) or `glimagesink`.
{: .prompt-warning }

---

## Configuration

All configuration is at the top of the file. Edit this section before running.

### SERVERS Dictionary

```python
SERVERS = {
    "laptop": {
        "uri": "ws://192.168.1.25:8765",
        "port_base": 5000,
        "cameras": [0, 1]
    },
    "pi": {
        "uri": "ws://192.168.1.27:8765",
        "port_base": 5100,
        "cameras": [0, 1, 2, 3]
    }
}
```

| Field | Description |
|---|---|
| `uri` | WebSocket address of the server (IP must match the server's static IP) |
| `port_base` | Base UDP port for video streams from this server |
| `cameras` | List of camera IDs available on this server |

The UDP port for each camera is calculated as `port_base + camera_id`. So for the Pi, camera 2 receives video on port `5100 + 2 = 5102`.

To add a new server, add a new entry to `SERVERS`. To remove a server, delete its entry. The camera mapping and grid layout are built automatically at startup from this dictionary.

### Camera Mapping (Auto-Generated)

At startup, the code iterates through `SERVERS` and builds `CAMERA_MAPPING` — a flat dictionary that assigns each camera a sequential display index:

```
Display 0: laptop cam0 @ port 5000
Display 1: laptop cam1 @ port 5001
Display 2: pi cam0 @ port 5100
Display 3: pi cam1 @ port 5101
Display 4: pi cam2 @ port 5102
Display 5: pi cam3 @ port 5103
```

This display index is used to position each `CameraWidget` in the grid (3 cameras per row).

### Timing Constants

```python
WS_RECONNECT_DELAY = 2        # Initial reconnect delay in seconds
WS_MAX_RECONNECT_DELAY = 32   # Maximum reconnect delay (exponential backoff cap)
WS_PING_INTERVAL = 10         # WebSocket keepalive ping interval (seconds)
WS_PING_TIMEOUT = 5           # Time to wait for pong before declaring connection dead

PIPELINE_RESTART_DELAY = 1500 # Delay before first pipeline start on launch (ms)
OVERLAY_SET_DELAY = 100        # Delay before setting GStreamer overlay (ms)
EXPANDED_CLOSE_DELAY = 150     # Delay before re-attaching overlay after close (ms)
```

---

## How It Works — Component by Component

### 1. WebSocket Layer

The WebSocket layer runs entirely in a **background thread** with its own `asyncio` event loop. This is necessary because `asyncio` and Qt each have their own event loops that cannot run on the same thread.

```
Main Thread (Qt)          Background Thread (asyncio)
     │                           │
     │                    websocket_handler(laptop_uri)
     │                    websocket_handler(pi_uri)
     │                           │
     │◄── status_signals ────────┤  (thread-safe Qt signals)
     │                           │
     │──── run_coroutine_threadsafe ──► send_message_to_server()
```

**`websocket_handler(uri)`** — a coroutine that:
1. Connects to the given WebSocket URI
2. Sets `ws_connected[uri]` (a `threading.Event`) so other threads can check connection status
3. Stores the live connection in `ws_connections[uri]`
4. Listens for incoming messages in a loop
5. On disconnect, clears the event, waits with exponential backoff, and retries

Backoff starts at `WS_RECONNECT_DELAY` (2s) and doubles on each failure up to `WS_MAX_RECONNECT_DELAY` (32s). On successful reconnect, it resets to 2s.

**`send_message_to_server(uri, payload)`** — an async function that serialises `payload` to JSON and sends it over the WebSocket. Called from the Qt thread using `run_coroutine_threadsafe`, which safely schedules the coroutine on the background asyncio loop and returns a `Future`.

The Qt thread calls `.result(timeout=5.0)` on the Future to block briefly and get the send result.

### 2. GStreamer Pipeline (Per Camera)

Each `CameraWidget` owns and manages one GStreamer pipeline. The pipeline receives H.264 video over UDP RTP and renders it into a Qt widget using the video overlay API.

**Pipeline string:**

```
udpsrc port=<PORT> caps="application/x-rtp,media=video,encoding-name=H264,payload=96"
! rtph264depay
! avdec_h264
! videoconvert
! videoscale
! video/x-raw,width=<W>,height=<H>
! xvimagesink name=video_sink sync=false
```

| Element | Role |
|---|---|
| `udpsrc` | Listens on UDP port, receives RTP packets |
| `rtph264depay` | Strips RTP headers, outputs raw H.264 NAL units |
| `avdec_h264` | Decodes H.264 to raw video frames (uses libav/FFmpeg) |
| `videoconvert` | Converts colour space if needed |
| `videoscale` | Scales to the selected resolution |
| `xvimagesink` | Renders video into an X11 window (the Qt label's window ID) |

The `sync=false` on `xvimagesink` disables clock synchronisation — the sink renders frames as fast as they arrive rather than buffering to a clock. This reduces latency at the cost of potentially uneven frame timing.

**Overlay binding** — GStreamer renders into a Qt `QLabel` by passing the label's native window handle (`winId()`) to `GstVideo.VideoOverlay.set_window_handle()`. This tells `xvimagesink` to draw directly into that window. The overlay must be re-set whenever:
- The pipeline is restarted
- The expanded view is opened (overlay moves to the expanded label)
- The expanded view is closed (overlay moves back to the main label)

This is why small `QTimer.singleShot` delays exist around expand/collapse — the window must be visible and fully initialised before its `winId()` is valid.

### 3. Qt UI and Threading

The Qt main thread runs the UI event loop. All UI updates must happen on this thread. Two mechanisms ensure this:

- **`StatusSignals` / `Signal`** — a `QObject` subclass with Qt signals. The background WebSocket thread emits `connection_changed` to update the connection indicator dots. Qt automatically queues signal delivery to the correct thread.
- **`QTimer.singleShot`** — used to schedule actions (pipeline restart, overlay re-attach) on the Qt event loop from within Qt callbacks.

**`GLib.MainContext` polling** — GStreamer's bus signals (error, warning, state-changed, EOS) are delivered through GLib's main context, not Qt's event loop. A `QTimer` fires every 50 ms and calls `glib_context.iteration(False)` to drain the GLib event queue. Without this, GStreamer bus messages would never be delivered and error handling would not work.

---

## Camera Widget — Controls

Each camera in the grid has its own set of controls.

| Control | Action Sent to Server | Local Effect |
|---|---|---|
| **Restart** | `{"action": "restart", "camera_id": N}` | Waits 500 ms, then restarts local GStreamer pipeline |
| **Stop** | `{"action": "stop", "camera_id": N}` | Sets `user_paused = True`, stops local pipeline |
| **Invert** | `{"action": "invert", "camera_id": N}` | None — server flips the stream |
| **Resolution dropdown** | `{"action": "set_resolution", "camera_id": N, "width": W, "height": H}` | Rebuilds local pipeline at new resolution |
| **Expand** | None | Opens a 1024×768 popup window with the stream |
| **Collapse** | None | Closes popup, re-attaches stream to main grid label |

The status dot (●) on each widget shows the WebSocket connection state for that camera's server — green = connected, red = disconnected. It updates every 2 seconds via the connection monitor timer.

---

## Auto-Restart Behaviour

The application handles failures automatically without user intervention.

**WebSocket disconnect** — `websocket_handler` loops forever with exponential backoff. No manual action needed.

**GStreamer errors** — `on_gst_error` checks the error string for keywords indicating a recoverable condition:

```python
if any(keyword in error_str for keyword in ["could not read", "streaming stopped", "internal data"]):
    QTimer.singleShot(3000, self.request_restart_stream)
```

**End of stream (EOS)** — `on_eos` schedules a restart after 1 second, unless `user_paused` is True (meaning the user explicitly stopped the stream).

**`user_paused` flag** — set to `True` when the user clicks Stop. Prevents auto-restart from triggering in response to the EOS that results from a deliberate stop. Cleared when Restart is clicked.

---

## Logging

Every run produces a timestamped log file:

```
/tmp/gcs_client_YYYYMMDD_HHMMSS.log
```

The path is also printed to the terminal on startup and shown in the Dashboard's `__init__` log output.

Log levels:

| Level | Usage |
|---|---|
| `INFO` | Connection events, pipeline start/stop, control actions |
| `WARN` | Disconnections, unexpected states, non-fatal issues |
| `ERROR` | Pipeline failures, send failures, exceptions |
| `DEBUG` | Every WebSocket message TX/RX, every GStreamer state change |

To enable DEBUG output, change `level=logging.INFO` to `level=logging.DEBUG` in the `logging.basicConfig` call.

---

## How to Run

### Prerequisites

Ensure the rover-side servers (laptop and Pi) are running their WebSocket + GStreamer server scripts, and are reachable at the IPs in `SERVERS`.

### Launch

```bash
python3 gcs_dashboard.py
```

On startup, the terminal prints:

```
============================================================
Starting Hardened Multi-Server GCS Dashboard
============================================================
Camera Mapping:
  Display 0: laptop cam0 @ port 5000
  Display 1: laptop cam1 @ port 5001
  Display 2: pi cam0 @ port 5100
  ...
Log file: /tmp/gcs_client_20250301_143022.log
============================================================
```

The dashboard window opens, WebSocket connections are attempted immediately, and each camera pipeline starts after a 1500 ms delay.

---

## File Location in MkDocs

```
docs/codebase/gcs-dashboard.md
```

---

## Known Limitations and TODOs

### 1. `xvimagesink` is X11-Only

The pipeline uses `xvimagesink`, which requires an X11 display server. This works on Ubuntu with X11. If the team switches to Wayland or needs to run on macOS/Windows, replace `xvimagesink` with `autovideosink` or `glimagesink`.

### 2. No Incoming Message Handling

`websocket_handler` receives and logs incoming messages but does nothing with them. If the server ever sends status updates or error notifications, there is currently no handler to process them. Add a message dispatch function inside the `async for msg in ws` loop.

### 3. Grid is 3 Columns Fixed

The grid layout hardcodes 3 columns (`i % 3`). If the number of cameras changes significantly (e.g. 8+ cameras), the grid may become awkward. Make this configurable or compute columns from the number of cameras.

### 4. No Recording

There is no option to record streams to disk. To add this, insert a `tee` element in the pipeline string and branch one output to `filesink`.

### 5. No Authentication on WebSocket

WebSocket connections are unauthenticated. Anyone on the network can connect to the server and send control commands. Acceptable for a closed field network; not acceptable if the network is shared.

### 6. Pipeline Does Not Verify Server Sent Stream Before Starting

The client sends a restart command and waits a hardcoded 500 ms before starting the local pipeline. If the server takes longer to start streaming (e.g. under CPU load), the pipeline may start before RTP packets arrive, producing a brief error before auto-recovering. A proper solution would wait for a server acknowledgement before starting the pipeline.

### 7. Resolution Change Does Not Restart the Pipeline

`on_resolution_change` sends the new resolution to the server but does not restart the local GStreamer pipeline. The pipeline continues running at the old resolution until the user manually clicks Restart. Either trigger a pipeline restart automatically after the resolution command is confirmed, or add a note in the UI.

---

## Sensor Datasheet and Reference Links

| Component | Reference |
|---|---|
| PySide6 Documentation | [doc.qt.io/qtforpython](https://doc.qt.io/qtforpython/) |
| GStreamer Plugin Reference | [gstreamer.freedesktop.org/documentation](https://gstreamer.freedesktop.org/documentation/) |
| websockets Library | [websockets.readthedocs.io](https://websockets.readthedocs.io/) |
| GStreamer Video Overlay | [GstVideoOverlay API](https://gstreamer.freedesktop.org/documentation/video/gstvideooverlay.html) |
| PyGObject Documentation | [pygobject.gnome.org](https://pygobject.gnome.org/) |
