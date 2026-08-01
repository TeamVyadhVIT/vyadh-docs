---
title: "Camera Server — Overview"
date: 2026-03-05
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, camera, video]
---

## What This Server Does

This is the rover-side video streaming server. It runs on each onboard computer (Raspberry Pi and/or a laptop mounted on the rover), detects attached cameras, starts a GStreamer pipeline for each one that encodes video as H.264 and sends it over UDP RTP to the GCS, and accepts WebSocket commands from the GCS dashboard to control streams in real time.

One instance of this server runs on each machine. The Pi instance and the laptop instance are the same codebase — the only difference is the `SERVER_NAME` constant at the top of the file, which controls port allocation and camera detection behaviour.

---

## Pages in This Section

- [Pi Server]({{ site.baseurl }}/posts/camera-server-pi-mode/) — configuration, camera detection behaviour, and deployment on the Raspberry Pi
- [Laptop Server]({{ site.baseurl }}/posts/camera-server-laptop-mode/) — configuration, integrated webcam exclusion, and deployment on the rover laptop

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                  CAMERA SERVER (Pi or Laptop)                        │
│                                                                       │
│  ┌──────────────────────┐    ┌──────────────────────────────────┐   │
│  │   Camera Detection   │    │   asyncio Event Loop             │   │
│  │   (startup, once)    │    │                                   │   │
│  │                      │    │  ┌─────────────────────────┐     │   │
│  │  glob /dev/video*    │    │  │  WebSocket Server        │     │   │
│  │  v4l2-ctl checks     │    │  │  handle_ws()            │     │   │
│  │  → CAMERA_DEVICES    │    │  │  port 8765              │     │   │
│  └──────────────────────┘    │  └─────────────────────────┘     │   │
│                               │                                   │   │
│  ┌──────────────────────┐    │  ┌─────────────────────────┐     │   │
│  │  Per-Camera State    │    │  │  monitor_streams()       │     │   │
│  │                      │    │  │  polls every 3s          │     │   │
│  │  ACTIVE_DEVICES[i]   │    │  │  auto-restart on crash  │     │   │
│  │  STREAM_INTENT[i]    │    │  └─────────────────────────┘     │   │
│  │  STREAMS[i]          │    │                                   │   │
│  │  STREAM_LOCKS[i]     │    │  ┌─────────────────────────┐     │   │
│  │  RESTART_FAILURES[i] │    │  │  health_check_task()     │     │   │
│  └──────────────────────┘    │  │  polls every 30s         │     │   │
│                               │  │  CPU usage check        │     │   │
│  ┌──────────────────────┐    │  └─────────────────────────┘     │   │
│  │  GStreamer Processes  │    └──────────────────────────────────┘   │
│  │  (one per camera)     │                                            │
│  │                       │                                            │
│  │  v4l2src → videoscale │                                            │
│  │  → videoconvert       │                                            │
│  │  → x264enc            │                                            │
│  │  → rtph264pay         │                                            │
│  │  → udpsink → GCS      │                                            │
│  └──────────────────────┘                                            │
└─────────────────────────────────────────────────────────────────────┘
         ▲ WebSocket commands              ▼ UDP RTP video
         │ ws://0.0.0.0:8765              │ to GCS IP, per-camera port
         │                                │
┌────────┴────────────────────────────────┴──────────────┐
│                    GCS Laptop                            │
│  GCS Dashboard ────────────────────────────────────────  │
└─────────────────────────────────────────────────────────┘
```

---

## Dependencies

| Package | Purpose | Install |
|---|---|---|
| `websockets` | Async WebSocket server | `pip install websockets` |
| `psutil` | CPU usage monitoring in health check | `pip install psutil` |
| `asyncio` | Async event loop | Python stdlib |
| `subprocess` | Launching and managing GStreamer processes | Python stdlib |
| `signal` | SIGINT / SIGTERM graceful shutdown | Python stdlib |
| `v4l2-ctl` | Camera device querying (system tool) | `sudo apt install v4l-utils` |
| `fuser` | Device release checking (system tool) | `sudo apt install psmisc` |
| `gst-launch-1.0` | GStreamer pipeline runner | `sudo apt install gstreamer1.0-tools gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-libav` |

```bash
# Full install
sudo apt install v4l-utils psmisc gstreamer1.0-tools \
    gstreamer1.0-plugins-base gstreamer1.0-plugins-good \
    gstreamer1.0-plugins-bad gstreamer1.0-libav

pip install websockets psutil
```

---

## Configuration Constants

These are the values to check and update before deploying.

| Constant | Default | Meaning |
|---|---|---|
| `SERVER_NAME` | `"pi"` or `"laptop"` | Identity of this server — controls port base and camera filtering |
| `GCS_HOST` | `"192.168.1.26"` | IP of the GCS laptop — where UDP video is sent |
| `WEBSOCKET_PORT` | `8765` | Port the WebSocket server listens on |
| `MAX_VIEWS` | `6` | Maximum number of simultaneous camera streams |
| `DEFAULT_RES` | `640x480` | Resolution used on startup if no other is specified |
| `MAX_RESTART_FAILURES` | `3` | How many times a crashed stream is restarted before being disabled |
| `STREAM_HEALTH_CHECK_INTERVAL` | `30` | Seconds between CPU health checks |
| `MIN_CPU_PERCENT` | `0.5` | CPU usage below this triggers a "frozen stream" warning |
| `MAX_MESSAGES_PER_MINUTE` | `60` | Rate limit per WebSocket client |

---

## Global State

The server tracks all runtime state in module-level globals. Understanding these is essential for reading the rest of the code.

| Variable | Type | Purpose |
|---|---|---|
| `CAMERA_DEVICES` | `List[str]` | Detected camera device paths (e.g. `["/dev/video0", "/dev/video2"]`). Populated at startup, not changed afterward. |
| `ACTIVE_DEVICES[i]` | `Optional[str]` | Which camera device is assigned to tile `i`. `None` if tile is unassigned. |
| `STREAM_INTENT[i]` | `str` | **The single source of truth** for whether tile `i` should be streaming. Either `"streaming"` or `"paused"`. The monitor uses this to decide whether to auto-restart. |
| `STREAMS[i]` | `Dict` | Metadata for the running stream on tile `i`: process handle, device, resolution, port, epoch, start time, restart count. |
| `STREAM_LOCKS[i]` | `asyncio.Lock` | Per-tile async lock preventing concurrent start/stop operations on the same tile. |
| `STREAM_EPOCH[i]` | `int` | Incremented each time a stream starts. Allows detection of stale operations targeting an old stream instance. |
| `RESTART_FAILURES[i]` | `int` | Count of consecutive failed restart attempts. Resets to 0 on success. Disables the stream when it reaches `MAX_RESTART_FAILURES`. |
| `STREAM_CONFIG[i]` | `Dict` | Persistent configuration for tile `i`: current width, height, and invert flag. Survives stream restarts. |
| `RESOLUTION_CACHE` | `Dict[str, List[Tuple]]` | Caches supported resolutions per device path so `v4l2-ctl` is not called repeatedly. |

---

## How It Works — Component by Component

### 1. Camera Detection — `detect_cameras()`

Called once at startup. Scans all `/dev/video*` devices and filters out non-camera devices:

**Step 1 — List all video devices:**
```python
all_devices = sorted(glob.glob("/dev/video*"), key=lambda x: int(x.replace("/dev/video", "")))
```

**Step 2 — Filter each device:** For every device found:
- Verify it exists and has read/write permissions
- Run `v4l2-ctl --list-formats-ext` and check if the output contains real video format strings (`YUYV`, `MJPEG`, `H264`, `NV12`, etc.). Devices that are metadata or control nodes (common on Pi) will not report any video formats and are excluded.
- Apply server-specific exclusions (see Pi and Laptop pages)

**Step 3 — Auto-assign to tiles:** After detection, cameras are assigned to tiles 0, 1, 2... in order:
```python
for i, device in enumerate(CAMERA_DEVICES[:MAX_VIEWS]):
    ACTIVE_DEVICES[i] = device
```

---

### 2. The `STREAM_INTENT` System

`STREAM_INTENT` is the most important design decision in the entire codebase. It prevents a subtle but serious bug: the auto-restart monitor should not restart a stream that was deliberately stopped by the operator.

```
STREAM_INTENT[i] = "streaming"  → this tile should be streaming
STREAM_INTENT[i] = "paused"     → this tile was intentionally stopped
```

**The rule:** Before attempting any auto-restart, the monitor checks `STREAM_INTENT`. If it is `"paused"`, it skips the tile entirely — even if the GStreamer process is dead. This means:
- Operator clicks Stop → `STREAM_INTENT = "paused"` → monitor never restarts it
- GStreamer crashes → `STREAM_INTENT` remains `"streaming"` → monitor restarts it
- Operator clicks Restart → `STREAM_INTENT = "streaming"` is set before starting → monitor will maintain it

Every function that starts a stream sets `STREAM_INTENT = "streaming"` first. Every function that stops at the user's request sets `STREAM_INTENT = "paused"` first, before touching the process.

---

### 3. GStreamer Pipeline — `build_gst_command()`

Each camera stream runs as a separate `gst-launch-1.0` subprocess. The pipeline:

```
v4l2src device=/dev/videoX
! videoscale
! video/x-raw,format=YUY2,width=W,height=H,framerate=30/1
[! videoflip method=rotate-180]   ← only if invert=True
! videoconvert
! x264enc tune=zerolatency bitrate=2000 speed-preset=ultrafast key-int-max=30
! rtph264pay config-interval=1 pt=96
! udpsink host=GCS_IP port=PORT sync=false
```

| Element | Purpose |
|---|---|
| `v4l2src` | Captures raw frames from the Linux V4L2 camera device |
| `videoscale` | Scales to requested resolution |
| `video/x-raw,format=YUY2` | Caps negotiation — forces YUY2 colour format |
| `videoflip method=rotate-180` | 180° rotation for inverted cameras (optional) |
| `videoconvert` | Converts YUY2 to I420 (required by x264enc) |
| `x264enc` | H.264 software encoder — `zerolatency` tune minimises encode buffering |
| `rtph264pay` | Wraps H.264 NAL units into RTP packets |
| `udpsink` | Sends RTP packets to the GCS over UDP |

**`tune=zerolatency`** is critical for live streaming — without it, x264 buffers many frames before outputting, introducing several seconds of latency.

**`key-int-max=30`** forces a keyframe every 30 frames (at 30fps, every 1 second). This limits how long the receiver must wait before it can display the stream after starting or resuming.

**`preexec_fn=os.setsid`** starts the GStreamer process in its own process group. This allows `os.killpg(os.getpgid(proc.pid), signal.SIGKILL)` to kill the entire process group (GStreamer + any child processes it spawned) cleanly.

---

### 4. Stream Lifecycle — Start, Stop, Restart

**`start_stream(cam_id)`:**
1. Checks `STREAM_INTENT` — refuses to start if not `"streaming"` or `"paused"`
2. Kills any zombie GStreamer processes still holding the device
3. Launches the GStreamer subprocess
4. Polls `proc.poll()` three times over 1.5 seconds to verify the process stays alive
5. On success, stores metadata in `STREAMS[cam_id]` and increments `STREAM_EPOCH`

**`stop_stream(cam_id, user_initiated=True)`:**
1. If `user_initiated`, sets `STREAM_INTENT = "paused"` **before** touching the process — this is the critical ordering
2. Sends `SIGTERM` and waits up to 3 seconds
3. If still running after 3s, kills the entire process group with `SIGKILL`
4. Sets `stream_info["process"] = None`

**`restart_stream_clean(cam_id)`** — the canonical restart path used by all code paths:
1. Acquires `STREAM_LOCKS[cam_id]` — prevents concurrent restarts
2. Checks `STREAM_INTENT` — aborts if `"paused"`
3. Stops existing stream (non-user-initiated, so intent is not changed)
4. Waits 500ms then calls `wait_for_device_release()` — polls `fuser` until no process holds the device
5. Kills any remaining zombies
6. Re-checks `STREAM_INTENT` before starting — guards against a race where intent changed during the wait
7. Calls `start_stream()`

---

### 5. Stream Monitor — `monitor_streams()`

An async task that polls every 3 seconds. For each tile:

1. Tries to acquire `STREAM_LOCKS[cam_id]` with a 100ms timeout — skips the tile if locked (a restart is already in progress)
2. Skips if `STREAM_INTENT != "streaming"`
3. Checks if `proc.poll() is None` (process still running) — if yes, skips
4. If process is dead and intent is `"streaming"`, schedules a restart
5. Applies **exponential backoff** before restarting: `backoff = min(2 ** RESTART_FAILURES[cam_id], 16)` — first retry after 1s, then 2s, 4s, 8s, 16s max
6. After `MAX_RESTART_FAILURES` consecutive failures, sets `STREAM_INTENT = "paused"` and broadcasts `"disabled"` to connected clients

The monitor deliberately releases the lock before doing the restart — holding the lock during a multi-second restart would block all other operations on that tile.

---

### 6. Health Check — `health_check_task()`

Runs every 30 seconds. For each running stream, uses `psutil` to check the CPU usage of the GStreamer process:

```python
p = psutil.Process(proc.pid)
cpu_percent = p.cpu_percent(interval=1.0)
```

A GStreamer process encoding and sending video will consistently use CPU. If CPU drops below `MIN_CPU_PERCENT` (0.5%), the stream is likely frozen — the process is alive but not doing any work. This is logged as a warning. The health check does not currently trigger an automatic restart on low CPU — it only warns. A future TODO is to add automatic recovery here.

---

### 7. WebSocket Server — `handle_ws()`

The WebSocket handler manages one connected client at a time (though multiple clients can connect simultaneously — `connected_clients` is a set). On connection, it immediately sends an `init` message:

```json
{
  "action": "init",
  "server_id": "pi",
  "server_name": "pi",
  "base_port": 5100,
  "max_views": 6,
  "devices": [{"name": "/dev/video0", "device": "/dev/video0"}],
  "active_devices": [
    {"tile": 0, "device": "/dev/video0", "port": 5100},
    ...
  ]
}
```

The GCS dashboard uses this to know which cameras are available and on which ports.

**Incoming message protocol:**

All messages are JSON objects with an `action` field and usually a `camera_id` field.

| Action | Required Fields | Effect |
|---|---|---|
| `get_status` | none | Returns current stream state for all tiles |
| `assign_device` | `camera_id`, `device` | Assigns a specific `/dev/videoX` to a tile and starts streaming |
| `start` | `camera_id` | Starts stream on an already-assigned tile |
| `stop` | `camera_id` | Stops stream, sets intent to paused |
| `restart` | `camera_id` | Restarts stream via `restart_stream_clean` |
| `invert` | `camera_id` | Toggles 180° flip on the stream and restarts |
| `set_resolution` | `camera_id`, `width`, `height` | Validates resolution then restarts at new size |

**Rate limiting:** Each client is limited to 60 messages per minute. The count resets every 60 seconds. Exceeding this returns an error message and introduces a 1-second delay.

**Message size limit:** Messages over 10KB are rejected. Prevents abuse.

---

### 8. Resolution Validation — `validate_resolution()`

Before restarting at a new resolution, the server checks whether the camera actually supports it:

```python
result = subprocess.run(["v4l2-ctl", "--list-formats-ext", "-d", device], ...)
```

Supported resolutions are parsed from the output and cached in `RESOLUTION_CACHE[device]`. Subsequent resolution changes for the same camera use the cache, avoiding repeated `v4l2-ctl` calls.

If the requested resolution is not in the supported list, the server returns an error and does not restart the stream — avoiding a GStreamer startup failure.

---

### 9. Shutdown Handling

Signal handlers for `SIGINT` and `SIGTERM` call `shutdown()`, which sets `shutdown_event`. The main coroutine is `await shutdown_event.wait()` — when the event is set, it cancels all background tasks and calls `cleanup()`, which stops all streams gracefully before the process exits.

---

## Logging

Every run produces a timestamped log file:

```
/tmp/camera_server_<server_id>_YYYYMMDD_HHMMSS.log
```

Log levels:

| Level | Used for |
|---|---|
| `INFO` | Startup, stream starts/stops, WebSocket connections |
| `WARN` | Disconnections, crashed streams, health warnings |
| `ERROR` | Pipeline failures, device errors, unexpected exceptions |
| `DEBUG` | Every WebSocket TX/RX, state transitions, resolution cache hits |

To enable DEBUG output, change `level=logging.INFO` to `level=logging.DEBUG` in `logging.basicConfig`.

---

## How to Run

```bash
# Install dependencies first (see Dependencies section above)

# Run the Pi server
python3 camera_server_pi.py

# Run the laptop server
python3 camera_server_laptop.py

# Or override SERVER_ID at runtime without editing the file
SERVER_ID=pi python3 camera_server.py
```

---

## Known Limitations and TODOs

| Issue | Detail |
|---|---|
| Health check warns but does not recover | `health_check_task` logs frozen streams but does not trigger a restart. Add `restart_stream_clean()` call when CPU stays below threshold for two consecutive checks. |
| `STREAM_EPOCH` is tracked but not used for stale operation detection | `STREAM_EPOCH` is incremented and stored but nothing currently checks it to discard responses from stale restart attempts. |
| No TLS on WebSocket | The WebSocket server runs unencrypted on `ws://`. Anyone on the network can connect and send commands. Acceptable for a closed field network. |
| `stop_stream` calls `time.sleep(0.2)` in `finally` block | This is a synchronous sleep inside an async context. It blocks the event loop for 200ms on every stream stop. Replace with `await asyncio.sleep(0.2)`. |
| `wait_for_device_release` is synchronous | Called inside `restart_stream_clean` which is async. The blocking `time.sleep` calls inside it block the event loop. Should use `await asyncio.sleep` or be run in a thread executor. |
| No audio support | The pipeline is video-only. Adding audio would require a separate `alsasrc` branch merged with `muxer`. |
| Resolution cache is never invalidated | If a camera is physically swapped for a different model at the same `/dev/videoX` path (unusual but possible), the cached resolutions would be wrong until the server restarts. |
