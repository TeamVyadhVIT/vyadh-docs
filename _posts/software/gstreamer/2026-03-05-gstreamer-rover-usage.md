---
title: "GStreamer — Rover Usage"
date: 2026-03-05
author: mahi
categories: [software, gstreamer]
tags: [gcs, gstreamer, video, rover]
---

## Camera Interfaces

Rovers typically use one of three camera interfaces. Each has a different GStreamer source element.

### V4L2 (Video4Linux2) — USB and CSI Cameras

V4L2 is the standard Linux camera interface. USB webcams and many CSI cameras (via drivers) appear as V4L2 devices.

```bash
# List available V4L2 devices
v4l2-ctl --list-devices

# List supported formats for a device
v4l2-ctl --device=/dev/video0 --list-formats-ext
```

```bash
# Basic V4L2 pipeline
gst-launch-1.0 \
  v4l2src device=/dev/video0 ! \
  video/x-raw,width=1280,height=720,framerate=30/1 ! \
  videoconvert ! autovideosink
```

### CSI Camera on Jetson (nvarguscamerasrc)

Jetson's CSI cameras use the NVIDIA Argus camera stack, which provides direct access to the ISP.

```bash
gst-launch-1.0 \
  nvarguscamerasrc sensor-id=0 ! \
  video/x-raw(memory:NVMM),width=1920,height=1080,framerate=30/1,format=NV12 ! \
  nvvidconv ! videoconvert ! autovideosink
```

> `memory:NVMM` — data stays in GPU memory for zero-copy hardware pipeline.

### USB Camera Pipeline

```bash
gst-launch-1.0 \
  v4l2src device=/dev/video0 ! \
  image/jpeg,width=1280,height=720,framerate=30/1 ! \
  jpegdec ! videoconvert ! autovideosink
```

> Many USB cameras output MJPEG. Use `jpegdec` instead of `videoconvert` for these.

---

## Choosing the Right Encoder for Your Hardware

| Hardware | Recommended Encoder | Why |
|----------|---------------------|-----|
| Raspberry Pi 4 | `v4l2h264enc` | Built-in hardware H.264, saves CPU |
| Raspberry Pi 5 | `v4l2h264enc` | V4L2 M2M encoder |
| NVIDIA Jetson (any) | `nvv4l2h264enc` | NVENC hardware, zero-copy with NVMM |
| Intel NUC (VAAPI) | `vaapih264enc` | Intel Quick Sync |
| Generic x86 / no HW accel | `x264enc` | Software, universal fallback |
| Low-power ARM (no HW enc) | `openh264enc` | Lower CPU than x264 |

### Decision Flow

```
Is your device a Jetson?
  → Yes: Use nvv4l2h264enc
  → No: Does it have V4L2 hardware encoder? (/dev/video-enc)
      → Yes: Use v4l2h264enc
      → No: Does it have Intel VAAPI?
          → Yes: Use vaapih264enc
          → No: Use x264enc (software fallback)
```

### Checking Available Encoders

```bash
# Check if V4L2 encoder is available
v4l2-ctl --list-devices | grep -i enc

# Check GStreamer plugins
gst-inspect-1.0 | grep -i enc
gst-inspect-1.0 v4l2h264enc
gst-inspect-1.0 nvv4l2h264enc
```

---

## Minimizing End-to-End Latency

End-to-end latency is the time from camera capture to display on GCS. Several stages contribute:

```
[Camera]──►[Capture]──►[Encode]──►[Network]──►[Jitter Buffer]──►[Decode]──►[Display]
   ~5ms       ~5ms      ~10ms      variable      ~50-200ms        ~5ms       ~5ms
```

### Encoder Settings (Biggest Impact)

```bash
# x264enc — lowest latency settings
x264enc \
  tune=zerolatency \         # No B-frames, no lookahead
  speed-preset=ultrafast \   # Fastest encoding
  key-int-max=30 \           # Keyframe every 1 sec
  bitrate=4000

# nvv4l2h264enc — Jetson low latency
nvv4l2h264enc \
  bitrate=4000000 \
  iframeinterval=30 \
  preset-level=1 \           # 1 = UltraFastPreset
  insert-sps-pps=true
```

### Pipeline Tuning

```bash
# Remove sync on sinks
autovideosink sync=false
udpsink sync=false async=false

# Reduce jitter buffer latency (or remove on reliable LAN)
rtpjitterbuffer latency=50

# Use queue with max-size-buffers=1 to drop stale frames
queue max-size-buffers=1 leaky=downstream
```

**`leaky=downstream`** — When the queue is full, the oldest buffer is dropped (always show the latest frame).

### Full Low-Latency Pipeline (Rover)

```bash
gst-launch-1.0 \
  v4l2src device=/dev/video0 ! \
  video/x-raw,width=640,height=480,framerate=30/1 ! \
  videoconvert ! \
  x264enc tune=zerolatency speed-preset=ultrafast bitrate=2000 key-int-max=30 ! \
  h264parse ! \
  rtph264pay config-interval=1 ! \
  queue max-size-buffers=1 leaky=downstream ! \
  udpsink host=192.168.1.50 port=5000 sync=false async=false
```

### Full Low-Latency Pipeline (GCS)

```bash
gst-launch-1.0 \
  udpsrc port=5000 buffer-size=2097152 ! \
  application/x-rtp,encoding-name=H264,payload=96 ! \
  rtpjitterbuffer latency=50 ! \
  rtph264depay ! h264parse ! \
  avdec_h264 ! \
  videoconvert ! \
  autovideosink sync=false
```

---

## Network Bandwidth Estimation and Bitrate Selection

### Estimating Available Bandwidth

```bash
# Test available bandwidth using iperf3
# On GCS (server):
iperf3 -s

# On rover (client):
iperf3 -c 192.168.1.50 -t 10

# Result shows available bandwidth in Mbps
```

### Bitrate Selection Guidelines

| Link Type | Available Bandwidth | Recommended Bitrate | Resolution |
|-----------|---------------------|---------------------|-----------|
| Local 5GHz WiFi | 50+ Mbps | 6–8 Mbps | 1080p30 |
| Local 2.4GHz WiFi | 20+ Mbps | 3–5 Mbps | 720p30 |
| LTE (good signal) | 10–20 Mbps | 2–4 Mbps | 720p30 |
| LTE (weak signal) | 2–5 Mbps | 1–2 Mbps | 480p30 |
| Radio link / mesh | variable | 500 Kbps–2 Mbps | 480p or lower |

> **Rule of thumb:** Use no more than **70% of available bandwidth** for video to leave room for telemetry and control traffic.

### Dynamic Bitrate Adjustment (Python)

```python
# Dynamically change x264enc bitrate at runtime
encoder = pipeline.get_by_name("encoder")
encoder.set_property("bitrate", 2000)  # in Kbps
```

---

## Handling Packet Loss Gracefully

### Strategies

| Strategy | Description | GStreamer Element |
|----------|-------------|-----------------|
| **Jitter buffer** | Reorders late packets, discards too-late | `rtpjitterbuffer` |
| **Leaky queue** | Drop oldest frames when full | `queue leaky=downstream` |
| **Frequent keyframes** | Faster visual recovery after loss | `key-int-max=30` |
| **SRT** | Selective retransmission within latency budget | `srtsink` / `srcsrc` |
| **FEC (Forward Error Correction)** | Pre-correct errors without retransmission | `rtpulpfecenc` |

### FEC (Forward Error Correction) in GStreamer

```bash
# Sender with FEC (adds redundancy packets)
gst-launch-1.0 v4l2src ! videoconvert ! x264enc ! rtph264pay ! \
  rtpulpfecenc percentage=25 pt=122 ! \
  udpsink host=192.168.1.50 port=5000

# Receiver with FEC
gst-launch-1.0 udpsrc port=5000 ! \
  rtpbin ! rtpulpfecdec pt=122 ! \
  rtph264depay ! h264parse ! avdec_h264 ! autovideosink
```

- `percentage=25` — 25% redundancy (1 FEC packet per 4 data packets)

---

## Python GStreamer — gi.repository.Gst

Python bindings for GStreamer use `PyGObject` (`gi.repository`).

### Installation

```bash
# Ubuntu/Debian
sudo apt install python3-gi python3-gi-cairo gir1.2-gst-plugins-base-1.0

# Verify
python3 -c "import gi; gi.require_version('Gst', '1.0'); from gi.repository import Gst; print('OK')"
```

### Basic Pipeline (Parse and Launch)

```python
import gi
gi.require_version('Gst', '1.0')
from gi.repository import Gst, GLib

Gst.init(None)

pipeline = Gst.parse_launch(
    "v4l2src device=/dev/video0 ! "
    "video/x-raw,width=1280,height=720,framerate=30/1 ! "
    "videoconvert ! "
    "x264enc tune=zerolatency speed-preset=ultrafast bitrate=4000 ! "
    "h264parse ! rtph264pay config-interval=1 ! "
    "udpsink host=192.168.1.50 port=5000"
)

pipeline.set_state(Gst.State.PLAYING)

loop = GLib.MainLoop()
try:
    loop.run()
except KeyboardInterrupt:
    pass

pipeline.set_state(Gst.State.NULL)
```

### Programmatic Pipeline Construction

```python
import gi
gi.require_version('Gst', '1.0')
from gi.repository import Gst

Gst.init(None)

# Create elements
pipeline = Gst.Pipeline.new("rover-stream")
src      = Gst.ElementFactory.make("v4l2src", "source")
convert  = Gst.ElementFactory.make("videoconvert", "convert")
encoder  = Gst.ElementFactory.make("x264enc", "encoder")
parse    = Gst.ElementFactory.make("h264parse", "parse")
pay      = Gst.ElementFactory.make("rtph264pay", "pay")
sink     = Gst.ElementFactory.make("udpsink", "sink")

# Set properties
src.set_property("device", "/dev/video0")
encoder.set_property("tune", "zerolatency")
encoder.set_property("speed-preset", "ultrafast")
encoder.set_property("bitrate", 4000)
sink.set_property("host", "192.168.1.50")
sink.set_property("port", 5000)

# Add and link
for el in [src, convert, encoder, parse, pay, sink]:
    pipeline.add(el)

src.link(convert)
convert.link(encoder)
encoder.link(parse)
parse.link(pay)
pay.link(sink)

# Start
pipeline.set_state(Gst.State.PLAYING)
```

### Dynamically Changing Encoder Bitrate

```python
def set_bitrate(pipeline, kbps):
    encoder = pipeline.get_by_name("encoder")
    if encoder:
        encoder.set_property("bitrate", kbps)
        print(f"Bitrate set to {kbps} Kbps")
```

---

## Integrating GStreamer into a GCS Dashboard

### GtkSink — Display in a GTK Window

```python
import gi
gi.require_version('Gst', '1.0')
gi.require_version('Gtk', '3.0')
from gi.repository import Gst, Gtk, GdkX11, GstVideo

Gst.init(None)
Gtk.init(None)

window = Gtk.Window()
video_widget = Gtk.DrawingArea()
window.add(video_widget)
window.show_all()

pipeline = Gst.parse_launch(
    "udpsrc port=5000 ! "
    "application/x-rtp,encoding-name=H264,payload=96 ! "
    "rtph264depay ! h264parse ! avdec_h264 ! "
    "videoconvert ! gtksink name=sink"
)

sink = pipeline.get_by_name("sink")
sink.set_property("widget", video_widget)

pipeline.set_state(Gst.State.PLAYING)
Gtk.main()
```

### AppSink → OpenCV Integration

AppSink pulls video frames out of the pipeline as raw numpy arrays for processing with OpenCV.

```python
import gi
gi.require_version('Gst', '1.0')
from gi.repository import Gst
import numpy as np
import cv2

Gst.init(None)

pipeline = Gst.parse_launch(
    "udpsrc port=5000 ! "
    "application/x-rtp,encoding-name=H264,payload=96 ! "
    "rtpjitterbuffer latency=50 ! "
    "rtph264depay ! h264parse ! avdec_h264 ! "
    "videoconvert ! video/x-raw,format=BGR ! "
    "appsink name=sink emit-signals=true max-buffers=1 drop=true"
)

appsink = pipeline.get_by_name("sink")
pipeline.set_state(Gst.State.PLAYING)

def on_new_sample(sink):
    sample = sink.emit("pull-sample")
    if sample is None:
        return Gst.FlowReturn.ERROR

    buf = sample.get_buffer()
    caps = sample.get_caps()

    width  = caps.get_structure(0).get_value("width")
    height = caps.get_structure(0).get_value("height")

    success, map_info = buf.map(Gst.MapFlags.READ)
    if success:
        frame = np.frombuffer(map_info.data, dtype=np.uint8)
        frame = frame.reshape((height, width, 3))
        cv2.imshow("GCS Feed", frame)
        cv2.waitKey(1)
        buf.unmap(map_info)

    return Gst.FlowReturn.OK

appsink.connect("new-sample", on_new_sample)

import GLib
GLib.MainLoop().run()
```

**Key AppSink properties:**
- `emit-signals=true` — Fires `new-sample` signal on each frame
- `max-buffers=1` — Only keep 1 frame in queue
- `drop=true` — Drop frames if OpenCV is slow (prevents memory buildup)
