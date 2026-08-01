---
title: "GStreamer — Common Pipelines"
date: 2026-03-05
author: mahi
categories: [software, gstreamer]
tags: [gcs, gstreamer, video]
---

## Pipeline Building Blocks

Before building full pipelines, here are the most commonly used elements:

| Element | Type | Purpose |
|---------|------|---------|
| `v4l2src` | Source | Capture from USB/V4L2 camera |
| `nvarguscamerasrc` | Source | Jetson CSI camera |
| `videotestsrc` | Source | Generate test video pattern |
| `videoconvert` | Filter | Convert pixel formats |
| `videoscale` | Filter | Resize video |
| `videorate` | Filter | Change framerate |
| `x264enc` | Encoder | H.264 software encoder |
| `v4l2h264enc` | Encoder | H.264 hardware encoder (Pi) |
| `nvv4l2h264enc` | Encoder | H.264 hardware encoder (Jetson) |
| `h264parse` | Parser | Parse H.264 bitstream |
| `avdec_h264` | Decoder | H.264 software decoder |
| `rtph264pay` | Payloader | Pack H.264 into RTP |
| `rtph264depay` | Depayloader | Unpack RTP to H.264 |
| `rtpjitterbuffer` | Buffer | Reorder and buffer RTP packets |
| `udpsink` | Sink | Send data via UDP |
| `udpsrc` | Source | Receive data via UDP |
| `autovideosink` | Sink | Auto-detect display output |
| `filesink` | Sink | Write to file |
| `tee` | Splitter | Duplicate stream to multiple branches |
| `queue` | Buffer | Decouple pipeline threads |

---

## Camera Capture → Encode → RTP Transmit (Rover Side)

### Basic UDP Stream (Software Encoder)

```bash
gst-launch-1.0 \
  v4l2src device=/dev/video0 ! \
  video/x-raw,width=1280,height=720,framerate=30/1 ! \
  videoconvert ! \
  x264enc tune=zerolatency speed-preset=ultrafast bitrate=4000 key-int-max=30 ! \
  h264parse ! \
  rtph264pay config-interval=1 pt=96 ! \
  udpsink host=192.168.1.50 port=5000
```

**Element breakdown:**
- `v4l2src device=/dev/video0` — Capture from camera
- `video/x-raw,...` — Enforce resolution and framerate caps
- `videoconvert` — Convert to a format x264enc accepts
- `x264enc tune=zerolatency` — Low-latency software encode
- `h264parse` — Normalizes H.264 bitstream
- `rtph264pay config-interval=1` — Send SPS/PPS with every keyframe
- `udpsink host=... port=...` — Send to GCS IP and port

### Hardware Encoder Pipeline (Raspberry Pi)

```bash
gst-launch-1.0 \
  v4l2src device=/dev/video0 ! \
  video/x-raw,width=1280,height=720,framerate=30/1 ! \
  v4l2h264enc extra-controls="controls,repeat_sequence_header=1" ! \
  video/x-h264,level=(string)4 ! \
  h264parse ! \
  rtph264pay config-interval=1 pt=96 ! \
  udpsink host=192.168.1.50 port=5000
```

### Hardware Encoder Pipeline (NVIDIA Jetson)

```bash
gst-launch-1.0 \
  nvarguscamerasrc sensor-id=0 ! \
  video/x-raw(memory:NVMM),width=1280,height=720,framerate=30/1,format=NV12 ! \
  nvv4l2h264enc bitrate=4000000 iframeinterval=30 ! \
  h264parse ! \
  rtph264pay config-interval=1 pt=96 ! \
  udpsink host=192.168.1.50 port=5000
```

---

## RTP Receive → Decode → Display (GCS Side)

### Basic Receive and Display

```bash
gst-launch-1.0 \
  udpsrc port=5000 ! \
  application/x-rtp,media=video,clock-rate=90000,encoding-name=H264,payload=96 ! \
  rtph264depay ! \
  h264parse ! \
  avdec_h264 ! \
  videoconvert ! \
  autovideosink sync=false
```

> `sync=false` on `autovideosink` disables clock-based synchronization, reducing display latency.

### With Jitter Buffer

```bash
gst-launch-1.0 \
  udpsrc port=5000 ! \
  application/x-rtp,media=video,clock-rate=90000,encoding-name=H264,payload=96 ! \
  rtpjitterbuffer latency=100 ! \
  rtph264depay ! \
  h264parse ! \
  avdec_h264 ! \
  videoconvert ! \
  autovideosink sync=false
```

- `rtpjitterbuffer latency=100` — Buffer 100ms of packets to handle out-of-order delivery

### Hardware Decode (Jetson)

```bash
gst-launch-1.0 \
  udpsrc port=5000 ! \
  application/x-rtp,media=video,clock-rate=90000,encoding-name=H264,payload=96 ! \
  rtpjitterbuffer ! \
  rtph264depay ! \
  h264parse ! \
  nvv4l2decoder ! \
  nvvidconv ! \
  autovideosink sync=false
```

---

## Using udpsink and udpsrc

### udpsink Properties

```bash
udpsink \
  host=192.168.1.50 \    # Destination IP
  port=5000 \            # Destination port
  sync=false \           # Don't sync to pipeline clock
  async=false            # Don't wait for preroll
```

### udpsrc Properties

```bash
udpsrc \
  address=0.0.0.0 \      # Listen on all interfaces (or specific IP)
  port=5000 \            # Port to listen on
  buffer-size=2097152    # Receive buffer size (2MB) — important for high bitrate
```

### Increasing UDP Receive Buffer

For high-bitrate streams, the OS default socket buffer may be too small:

```bash
# Increase system UDP buffer (Linux)
sudo sysctl -w net.core.rmem_max=26214400
sudo sysctl -w net.core.rmem_default=26214400

# Set per-socket in GStreamer
udpsrc buffer-size=26214400 port=5000
```

---

## Using rtspsrc and RTSP Server

### Receiving an RTSP Stream (Client)

```bash
gst-launch-1.0 \
  rtspsrc location=rtsp://192.168.1.20:8554/stream latency=0 ! \
  rtph264depay ! \
  h264parse ! \
  avdec_h264 ! \
  videoconvert ! \
  autovideosink sync=false
```

- `latency=0` — Minimize jitter buffer latency (increase if stream is unstable)

### Serving RTSP (Python + GstRTSPServer)

```python
import gi
gi.require_version('Gst', '1.0')
gi.require_version('GstRtspServer', '1.0')
from gi.repository import Gst, GstRtspServer, GLib

Gst.init(None)

server = GstRtspServer.RTSPServer.new()
server.set_service("8554")  # Port

mounts = server.get_mount_points()

factory = GstRtspServer.RTSPMediaFactory.new()
factory.set_launch(
    "( v4l2src device=/dev/video0 ! videoconvert ! "
    "x264enc tune=zerolatency speed-preset=ultrafast ! "
    "rtph264pay name=pay0 pt=96 )"
)
factory.set_shared(True)
mounts.add_factory("/stream", factory)

server.attach(None)
print("RTSP server running at rtsp://0.0.0.0:8554/stream")

GLib.MainLoop().run()
```

---

## Adding rtpjitterbuffer for Stability

The `rtpjitterbuffer` element smooths out packet arrival timing caused by network jitter.

### How It Works

```
Network:  P1 ──► P3 ──► P2 ──► P4   (out-of-order, variable timing)
                     ↓
          rtpjitterbuffer (buffers and reorders)
                     ↓
Output:   P1 ──► P2 ──► P3 ──► P4   (correct order, smooth timing)
```

### Key Properties

```bash
rtpjitterbuffer \
  latency=100 \          # Jitter buffer depth in ms (default: 200ms)
  drop-on-latency=true \ # Drop late packets instead of stalling
  do-lost=true           # Emit signal when packet is considered lost
```

### Tuning latency

| Network Condition | Recommended latency |
|-------------------|---------------------|
| Local WiFi (stable) | 50–100 ms |
| Local WiFi (variable) | 100–200 ms |
| LTE / internet | 200–500 ms |
| SRT (use SRT latency instead) | 0 (bypass jitterbuffer) |

---

## Recording While Displaying (Simultaneous)

Use `tee` to split the decoded stream — one branch for display, one for file recording.

```bash
gst-launch-1.0 \
  v4l2src device=/dev/video0 ! \
  video/x-raw,width=1280,height=720,framerate=30/1 ! \
  videoconvert ! \
  tee name=split \
  \
  split. ! queue ! autovideosink sync=false \
  \
  split. ! queue ! \
    x264enc tune=zerolatency speed-preset=ultrafast bitrate=4000 ! \
    h264parse ! \
    mp4mux ! \
    filesink location=/home/user/recording.mp4
```

### Record Received Stream While Displaying

```bash
gst-launch-1.0 \
  udpsrc port=5000 ! \
  application/x-rtp,media=video,clock-rate=90000,encoding-name=H264,payload=96 ! \
  rtpjitterbuffer ! \
  rtph264depay ! \
  h264parse ! \
  tee name=split \
  \
  split. ! queue ! avdec_h264 ! videoconvert ! autovideosink sync=false \
  \
  split. ! queue ! mp4mux ! filesink location=/home/user/received.mp4
```

> **Important:** Always put a `queue` element after `tee`. Each branch runs in its own thread; `queue` is the thread boundary and prevents one branch from blocking the other.

---

## Muxing Multiple Streams

### Combine Video and Audio into a Container

```bash
gst-launch-1.0 \
  v4l2src ! videoconvert ! x264enc ! h264parse ! mux. \
  alsasrc ! audioconvert ! audioresample ! avenc_aac ! aacparse ! mux. \
  matroskamux name=mux ! filesink location=output.mkv
```

### Multiple Camera Streams Over Separate Ports

```bash
# Camera 1 — port 5000
gst-launch-1.0 v4l2src device=/dev/video0 ! videoconvert ! \
  x264enc tune=zerolatency ! rtph264pay ! udpsink host=192.168.1.50 port=5000 &

# Camera 2 — port 5002
gst-launch-1.0 v4l2src device=/dev/video1 ! videoconvert ! \
  x264enc tune=zerolatency ! rtph264pay ! udpsink host=192.168.1.50 port=5002 &
```

### Composite Multiple Cameras Into One Stream

```bash
gst-launch-1.0 \
  videomixer name=mix \
    sink_0::xpos=0 sink_0::ypos=0 \
    sink_1::xpos=640 sink_1::ypos=0 ! \
  videoconvert ! autovideosink \
  \
  v4l2src device=/dev/video0 ! video/x-raw,width=640,height=480 ! videoconvert ! mix.sink_0 \
  v4l2src device=/dev/video1 ! video/x-raw,width=640,height=480 ! videoconvert ! mix.sink_1
```
