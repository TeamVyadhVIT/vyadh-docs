---
title: "Camera Server — Pi Mode"
date: 2026-03-05
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, camera, raspberry-pi]
---

## Overview

The Pi server is the instance of the camera server that runs on the **Raspberry Pi mounted on the rover**. It is identical to the laptop server in every way except for the `SERVER_NAME` constant at the top of the file, which cascades into port allocation and camera device filtering behaviour.

For full documentation of the server's internals, see the [Camera Server Overview]({{ site.baseurl }}/posts/camera-server-overview/).

---

## The One Difference That Changes Everything

```python
SERVER_NAME = "pi"
SERVER_ID = os.environ.get("SERVER_ID", SERVER_NAME)  # → "pi"
```

`SERVER_ID` is used in two places that directly affect behaviour:

1. **Port allocation** — determines which UDP ports video is streamed on
2. **Camera detection** — determines which `/dev/video*` devices are filtered out

Everything else — the WebSocket server, GStreamer pipeline, stream monitor, health check, restart logic — is identical.

---

## Port Allocation

```python
PORT_RANGES = {"laptop": 5000, "pi": 5100}
BASE_PORT = PORT_RANGES.get(SERVER_ID, 5000)   # → 5100
PORTS = [BASE_PORT + i for i in range(MAX_VIEWS)]
# → [5100, 5101, 5102, 5103, 5104, 5105]
```

The Pi streams on ports **5100–5105**. Each camera gets its own port:

| Tile | Camera | UDP Port |
|---|---|---|
| 0 | First detected camera | 5100 |
| 1 | Second detected camera | 5101 |
| 2 | Third detected camera | 5102 |
| 3 | Fourth detected camera | 5103 |
| 4 | Fifth detected camera | 5104 |
| 5 | Sixth detected camera | 5105 |

The GCS laptop must have firewall rules open for UDP on ports 5100–5105 to receive these streams. The GCS dashboard is configured to listen on these ports for Pi cameras — see the `SERVERS` dict in the GCS dashboard code.

---

## Camera Detection on the Pi

The Pi runs a specialised Linux kernel for the Raspberry Pi hardware. Its V4L2 device tree includes both real camera devices and internal system video devices used for hardware acceleration, ISP pipelines, and codec interfaces. These internal devices appear as `/dev/video*` entries but are not cameras — querying them as cameras will fail or return garbage.

### The Pi Internal Device Filter

```python
dev_num = int(dev.replace("/dev/video", ""))
if dev_num >= 19:
    log(f"    ✗ Skipping {dev} (Pi internal device)")
    continue
```

On a Raspberry Pi running the standard camera stack, real camera devices appear at low device numbers (`/dev/video0`, `/dev/video2`, etc.), while internal ISP and codec devices appear at `/dev/video19` and above. The filter skips anything at `/dev/video19` or higher.

> **Device Numbers Are Not Guaranteed**
>
> The `>= 19` threshold was determined empirically for the specific Pi OS and camera driver version in use. If the Pi OS is updated, if a USB hub with multiple cameras is connected, or if additional kernel modules are loaded, real camera devices could potentially appear at higher numbers or internal devices could appear at lower numbers. If cameras stop being detected after a system update, check `v4l2-ctl --list-devices` and adjust this threshold.
{: .prompt-warning }

### The Video Format Filter

Even after the device number filter, the detection code runs `v4l2-ctl --list-formats-ext` on each device and checks for real video format strings:

```python
has_video_formats = any(
    fmt in output.upper()
    for fmt in ["YUYV", "MJPEG", "MJPG", "H264", "H265", "YUV", "NV12", "RGB"]
)
```

A device that passes the device number filter but reports no video formats (only control or metadata formats) is still excluded. This is a second layer of defence.

### Integrated Webcam Check

The Pi server **does not skip integrated webcams** — the `is_integrated_webcam()` check only runs when `SERVER_ID == "laptop"`:

```python
if SERVER_ID == "laptop" and is_integrated_webcam(dev):
    log(f"    ✗ Skipping {dev} (laptop built-in webcam)")
    continue
```

Since Raspberry Pis do not have built-in webcams, this check is irrelevant and is skipped.

---

## Supported Camera Types on Pi

| Camera Type | How It Connects | Typical Device Path | Notes |
|---|---|---|---|
| USB Webcam | USB port | `/dev/video0`, `/dev/video2` | Standard V4L2 device, detected automatically |
| Raspberry Pi Camera Module (v2, v3) | CSI ribbon cable | `/dev/video0` (varies by driver) | Requires `libcamera` or `bcm2835-v4l2` driver loaded |
| Intel RealSense (colour stream) | USB 3.0 | `/dev/video0` or similar | Depth stream uses a separate device — see RGBD Point Selector |

> **Pi Camera Module and V4L2**
>
> The Raspberry Pi Camera Module (connected via CSI) appears as a V4L2 device only when the appropriate driver is enabled. Run `sudo modprobe bcm2835-v4l2` or configure the Pi camera via `raspi-config`. Without this, the camera will not appear as `/dev/video*` and will not be detected.
{: .prompt-info }

---

## Deployment on the Raspberry Pi

### System Requirements

- Raspberry Pi 4 or 5 (Pi 3 is marginal for H.264 encoding — `x264enc` is CPU-intensive)
- Raspberry Pi OS (Bookworm or Bullseye)
- Python 3.9+
- GStreamer and plugins installed (see [Overview — Dependencies]({{ site.baseurl }}/posts/camera-server-overview/))

### Install Dependencies

```bash
sudo apt update
sudo apt install v4l-utils psmisc \
    gstreamer1.0-tools \
    gstreamer1.0-plugins-base \
    gstreamer1.0-plugins-good \
    gstreamer1.0-plugins-bad \
    gstreamer1.0-libav

pip3 install websockets psutil
```

### Verify Cameras Are Detected

Before running the server, verify V4L2 can see your cameras:

```bash
v4l2-ctl --list-devices
```

Expected output for a USB webcam:
```
USB2.0 HD UVC WebCam (usb-0000:01:00.0-1.3):
    /dev/video0
    /dev/video1
```

Verify the camera produces a real video stream:
```bash
v4l2-ctl --list-formats-ext -d /dev/video0
```

If you see `YUYV`, `MJPEG`, or similar — the camera will be detected by the server.

### Run the Server

```bash
python3 camera_server_pi.py
```

Expected startup output:
```
[PI] ============================================================
[PI] Multi-Camera Server - PI
[PI] ============================================================
[PI] GCS Host: 192.168.1.26
[PI] WebSocket: ws://0.0.0.0:8765
[PI] Port Range: 5100-5105
[PI] Scanning for cameras...
[PI]   Checking /dev/video0...
[PI]     ✓ REAL CAMERA FOUND: /dev/video0
[PI] DETECTED 1 REAL CAMERA(S): ['/dev/video0']
[PI] Auto-assigned /dev/video0 to tile 0
[PI] Starting initial streams...
[PI] ✓ Stream 0 started (PID 12345, epoch 1)
[PI] Server ready
```

### Run as a Service (Auto-Start on Boot)

Create a systemd service so the camera server starts automatically when the Pi boots:

```bash
sudo nano /etc/systemd/system/camera-server.service
```

```ini
[Unit]
Description=GCS Camera Server (Pi)
After=network.target

[Service]
ExecStart=/usr/bin/python3 /home/pi/camera_server_pi.py
WorkingDirectory=/home/pi
Restart=on-failure
RestartSec=5
User=pi

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable camera-server
sudo systemctl start camera-server
sudo systemctl status camera-server
```

---

## Troubleshooting Pi-Specific Issues

| Symptom | Likely Cause | Fix |
|---|---|---|
| No cameras detected | Pi camera module driver not loaded | Run `sudo modprobe bcm2835-v4l2` or enable camera in `raspi-config` |
| Camera detected but stream fails to start | Camera format not YUY2 — Pi cameras often output YUV420 | Change `format=YUY2` to `format=I420` in `build_gst_command` or use `videoconvert` earlier in the pipeline |
| High CPU causing dropped frames | Pi 3/4 struggling with x264 software encoding | Reduce resolution to 640x480, or use `omxh264enc` (Pi 4 hardware encoder) instead of `x264enc` |
| Video19+ devices appearing as cameras | Kernel update changed device numbering | Run `v4l2-ctl --list-devices` to see actual layout and update the `>= 19` threshold |
| GStreamer fails with "resource busy" | Previous stream process not cleaned up | Run `pkill -9 gst-launch-1.0` manually, then restart server |
