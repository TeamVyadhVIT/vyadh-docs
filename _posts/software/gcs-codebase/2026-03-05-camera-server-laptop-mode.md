---
title: "Camera Server — Laptop Mode"
date: 2026-03-05
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, camera, video]
---

## Overview

The laptop server is the instance of the camera server that runs on the **laptop mounted on the rover**. It is identical to the Pi server in every way except for the `SERVER_NAME` constant at the top of the file, which cascades into port allocation and camera detection behaviour — specifically, it adds a filter to exclude the laptop's built-in webcam from being detected as a rover camera.

For full documentation of the server's internals, see the [Camera Server Overview]({{ site.baseurl }}/posts/camera-server-overview/).

---

## The One Difference That Changes Everything

```python
SERVER_NAME = "laptop"
SERVER_ID = os.environ.get("SERVER_ID", SERVER_NAME)  # → "laptop"
```

`SERVER_ID` is used in two places that directly affect behaviour:

1. **Port allocation** — determines which UDP ports video is streamed on
2. **Camera detection** — enables the integrated webcam exclusion filter

---

## Port Allocation

```python
PORT_RANGES = {"laptop": 5000, "pi": 5100}
BASE_PORT = PORT_RANGES.get(SERVER_ID, 5000)   # → 5000
PORTS = [BASE_PORT + i for i in range(MAX_VIEWS)]
# → [5000, 5001, 5002, 5003, 5004, 5005]
```

The laptop server streams on ports **5000–5005**. Each camera gets its own port:

| Tile | Camera | UDP Port |
|---|---|---|
| 0 | First detected camera | 5000 |
| 1 | Second detected camera | 5001 |
| 2 | Third detected camera | 5002 |
| 3 | Fourth detected camera | 5003 |
| 4 | Fifth detected camera | 5004 |
| 5 | Sixth detected camera | 5005 |

The GCS laptop must have firewall rules open for UDP on ports 5000–5005. The GCS dashboard is configured to listen on these ports for laptop cameras.

> **Port 5005 Conflict**
>
> The GPS receiver GUI also uses UDP port 5005 to receive GPS coordinates from the serial bridge. If the laptop server assigns a camera to tile 5 (port 5005), and the GPS receiver is also running on the same GCS machine, there will be a port conflict. Reduce `MAX_VIEWS` to 5 on the laptop, or change the GPS receiver port, if this is an issue.
{: .prompt-warning }

---

## Camera Detection on the Laptop

The laptop has a built-in webcam. This webcam appears as a `/dev/video*` device and will pass the video format filter — it is a real camera. However, the team does not want the laptop's built-in webcam to be used as a rover camera. The laptop server adds an explicit exclusion for it.

### The Integrated Webcam Filter

```python
if SERVER_ID == "laptop" and is_integrated_webcam(dev):
    log(f"    ✗ Skipping {dev} (laptop built-in webcam)")
    continue
```

`is_integrated_webcam(device)` runs `v4l2-ctl -d <device> --info` and searches the device info string for known manufacturer keywords associated with built-in cameras:

```python
integrated_keywords = [
    "integrated camera", "integrated_webcam", "integrated webcam",
    "built-in", "hp truevision", "lenovo integrated",
    "thinkpad integrated", "dell integrated", "asus integrated",
    "acer integrated", "macbook", "facetime", "Intel(R) RealSense(TM)"
]
```

If any of these strings appear in the device info (case-insensitive), the device is considered an integrated webcam and is skipped.

> **Keyword Coverage**
>
> The keyword list covers most common laptop manufacturers. If the team's laptop has a built-in camera from a manufacturer not on this list, it will not be filtered and will be detected as a rover camera. Run `v4l2-ctl -d /dev/video0 --info` to check the "Card type" field of your laptop's camera and add the relevant keyword if needed.
{: .prompt-warning }

> **Intel RealSense in the Keyword List**
>
> `Intel(R) RealSense(TM)` appears in the integrated keywords list. This means if a RealSense camera is connected to the laptop (rather than the Pi), it will be excluded from the streaming server. If the RealSense is intended to stream via this server rather than through the RGBD Point Selector node, remove this keyword.
{: .prompt-info }

### Pi Internal Device Filter Does Not Apply

The laptop server does **not** apply the `>= 19` device number filter. That filter is Pi-specific:

```python
if dev_num >= 19:
    log(f"    ✗ Skipping {dev} (Pi internal device)")
    continue
```

This check runs unconditionally for both servers — it is not gated by `SERVER_ID`. On a laptop, USB cameras will never appear at `/dev/video19` or above in practice, so this filter has no effect. But be aware it exists — if unusual hardware creates high-numbered device paths on a laptop, they would be filtered out.

---

## Deployment on the Rover Laptop

### System Requirements

- Ubuntu 22.04 or later (or compatible Linux distribution)
- Python 3.9+
- GStreamer and plugins installed
- USB cameras connected

### Install Dependencies

```bash
sudo apt update
sudo apt install v4l-utils psmisc \
    gstreamer1.0-tools \
    gstreamer1.0-plugins-base \
    gstreamer1.0-plugins-good \
    gstreamer1.0-plugins-bad \
    gstreamer1.0-libav

pip install websockets psutil
```

### Verify Cameras Are Detected

```bash
v4l2-ctl --list-devices
```

Check which device corresponds to the built-in webcam vs the external USB cameras. The server will automatically exclude the built-in one.

### Run the Server

```bash
python3 camera_server_laptop.py
```

Expected startup output showing the built-in webcam being excluded:
```
[LAPTOP] ============================================================
[LAPTOP] Multi-Camera Server - LAPTOP
[LAPTOP] ============================================================
[LAPTOP] GCS Host: 192.168.1.26
[LAPTOP] WebSocket: ws://0.0.0.0:8765
[LAPTOP] Port Range: 5000-5005
[LAPTOP] Scanning for cameras...
[LAPTOP]   Checking /dev/video0...
[LAPTOP]     ✗ Skipping /dev/video0 (laptop built-in webcam)
[LAPTOP]   Checking /dev/video2...
[LAPTOP]     ✓ REAL CAMERA FOUND: /dev/video2
[LAPTOP]   Checking /dev/video4...
[LAPTOP]     ✓ REAL CAMERA FOUND: /dev/video4
[LAPTOP] DETECTED 2 REAL CAMERA(S): ['/dev/video2', '/dev/video4']
[LAPTOP] Auto-assigned /dev/video2 to tile 0
[LAPTOP] Auto-assigned /dev/video4 to tile 1
[LAPTOP] ✓ Stream 0 started (PID 23456, epoch 1)
[LAPTOP] ✓ Stream 1 started (PID 23457, epoch 1)
[LAPTOP] Server ready
```

### Run as a Service (Auto-Start on Boot)

```bash
sudo nano /etc/systemd/system/camera-server.service
```

```ini
[Unit]
Description=GCS Camera Server (Laptop)
After=network.target

[Service]
ExecStart=/usr/bin/python3 /home/user/camera_server_laptop.py
WorkingDirectory=/home/user
Restart=on-failure
RestartSec=5
User=user

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable camera-server
sudo systemctl start camera-server
```

---

## Troubleshooting Laptop-Specific Issues

| Symptom | Likely Cause | Fix |
|---|---|---|
| Built-in webcam is being detected as a rover camera | Laptop manufacturer not in keyword list | Run `v4l2-ctl -d /dev/video0 --info`, check "Card type" field, add the keyword to `integrated_keywords` |
| External USB camera is being excluded | Its device info contains a keyword from the list (e.g. a RealSense) | Remove the conflicting keyword from `integrated_keywords` |
| Both cameras on the same port | Two cameras auto-assigned tiles 0 and 1 on the same port range | Ports 5000 and 5001 — check GCS dashboard is listening on both |
| GPS receiver on port 5005 conflicts with camera tile 5 | Both use UDP 5005 | Set `MAX_VIEWS = 5` in the laptop server to avoid allocating port 5005 |
| Server detects no cameras | All devices failed the format check or are excluded | Run `v4l2-ctl --list-formats-ext -d /dev/videoX` for each device to diagnose |
