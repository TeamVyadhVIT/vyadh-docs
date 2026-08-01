---
title: "GStreamer — Troubleshooting"
date: 2026-03-05
author: mahi
categories: [software, gstreamer]
tags: [gcs, gstreamer, video, troubleshooting]
---

## General Debugging Approach

When a pipeline fails or behaves unexpectedly, follow this sequence:

```
1. Read the error message carefully
2. Enable GST_DEBUG to get more detail
3. Inspect elements with gst-inspect-1.0
4. Test the pipeline in segments (isolate the failing element)
5. Check caps negotiation
6. Verify hardware and driver availability
```

### Enable Debug Output

```bash
# Basic debug — shows errors and warnings
GST_DEBUG=2 gst-launch-1.0 ...

# Moderate detail — shows state changes and info
GST_DEBUG=4 gst-launch-1.0 ...

# Show caps negotiation specifically
GST_DEBUG=GST_CAPS:4 gst-launch-1.0 ...

# Debug a specific element
GST_DEBUG=v4l2src:5,x264enc:4 gst-launch-1.0 ...

# Save debug output to file
GST_DEBUG=5 GST_DEBUG_FILE=/tmp/gst_debug.log gst-launch-1.0 ...
```

### Visualize the Pipeline

```bash
GST_DEBUG_DUMP_DOT_DIR=/tmp gst-launch-1.0 videotestsrc ! autovideosink
dot -Tpng /tmp/*.dot -o /tmp/pipeline.png
```

---

## gst-inspect-1.0 — Finding and Verifying Plugins

`gst-inspect-1.0` is the primary tool for discovering what plugins are installed and what caps/properties they support.

### Usage

```bash
# List all installed plugins
gst-inspect-1.0

# Inspect a specific element
gst-inspect-1.0 x264enc
gst-inspect-1.0 v4l2src
gst-inspect-1.0 nvv4l2h264enc

# Search for elements matching a keyword
gst-inspect-1.0 | grep -i h264
gst-inspect-1.0 | grep -i nvv4l2

# Check if a plugin package is installed
gst-inspect-1.0 | grep -i "good\|bad\|ugly\|libav"
```

### What to Look for in gst-inspect-1.0 Output

```
Element Properties:
  device          : V4L2 device location
  ...

Pad Templates:
  SRC template: 'src'
    Availability: Always
    Capabilities:
      video/x-raw
                 format: { (string)NV12, (string)I420, ... }
                  width: [ 1, 32768 ]
                 height: [ 1, 32768 ]
              framerate: [ 0/1, 2147483647/1 ]
```

The **Pad Templates → Capabilities** section tells you exactly what formats an element can accept or produce.

### Checking Plugin Installation

```bash
# Check which GStreamer plugin packages are installed
dpkg -l | grep gstreamer

# Install common plugin packages
sudo apt install \
  gstreamer1.0-tools \
  gstreamer1.0-plugins-base \
  gstreamer1.0-plugins-good \
  gstreamer1.0-plugins-bad \
  gstreamer1.0-plugins-ugly \
  gstreamer1.0-libav
```

| Package | Contains |
|---------|---------|
| `gstreamer1.0-plugins-base` | Core elements: `videoconvert`, `audioresample`, `volume` |
| `gstreamer1.0-plugins-good` | `v4l2src`, `udpsrc`, `udpsink`, `rtph264pay`, `rtpjitterbuffer` |
| `gstreamer1.0-plugins-bad` | `nvv4l2h264enc`, `srtsink`, `webrtcbin`, `rtpulpfecenc` |
| `gstreamer1.0-plugins-ugly` | `x264enc` |
| `gstreamer1.0-libav` | `avdec_h264`, `avenc_aac` |

---

## Pipeline Fails to Start — Caps Negotiation Issues

### Symptom

```
ERROR: could not link videoconvert0 to x264enc0
```
or
```
Got error from element "...", source: "negotiation"
```

### What's Happening

Two adjacent elements cannot agree on a common format. The source pad caps and sink pad caps have no intersection.

### Diagnosis

```bash
# See what caps are being negotiated
GST_DEBUG=GST_CAPS:4 gst-launch-1.0 v4l2src ! videoconvert ! x264enc ! fakesink
```

Look for lines like:
```
CAPS             gstpad.c             caps already equal
CAPS             gstpad.c             trying to set caps: video/x-raw, format=(string)YUYV
```

### Common Causes and Fixes

**1. Camera outputs a format the encoder won't accept directly**

```bash
# Problem: v4l2src outputs YUYV, x264enc wants I420
# Fix: Add videoconvert to handle the conversion
v4l2src ! videoconvert ! x264enc
```

**2. Enforcing caps the hardware doesn't support**

```bash
# Problem: Requesting format the camera doesn't offer
v4l2src ! video/x-raw,format=BGR ! ...   # Camera may not output BGR

# Fix: Let videoconvert handle it
v4l2src ! videoconvert ! video/x-raw,format=I420 ! x264enc
```

**3. Jetson NVMM pipeline mixed with non-NVMM elements**

```bash
# Problem: nvv4l2h264enc requires memory:NVMM
nvarguscamerasrc ! videoconvert ! nvv4l2h264enc  # WRONG

# Fix: Use nvvidconv for NVMM-to-NVMM conversion
nvarguscamerasrc ! nvvidconv ! nvv4l2h264enc    # CORRECT
```

**4. Missing h264parse between encoder and payloader**

```bash
# Problem: rtph264pay may reject raw encoder output without parsing
x264enc ! rtph264pay                   # May fail

# Fix: Always insert h264parse
x264enc ! h264parse ! rtph264pay       # Always safe
```

### Caps Mismatch Debugging Template

```bash
# Step 1: Test source alone
gst-launch-1.0 v4l2src device=/dev/video0 ! fakesink

# Step 2: Add capsfilter and test
gst-launch-1.0 v4l2src device=/dev/video0 ! \
  video/x-raw,width=1280,height=720 ! fakesink

# Step 3: Add convert and test
gst-launch-1.0 v4l2src device=/dev/video0 ! videoconvert ! fakesink

# Step 4: Add encoder and test
gst-launch-1.0 v4l2src device=/dev/video0 ! videoconvert ! x264enc ! fakesink

# Add one element at a time until failure is isolated
```

---

## High Latency — Encoder / Jitter Buffer Tuning

### Symptom

Video is noticeably delayed (>500ms) between rover camera and GCS display.

### Diagnosis

```bash
# Measure latency — add latency property to pipeline
gst-launch-1.0 \
  v4l2src ! videoconvert ! x264enc ! h264parse ! rtph264pay ! \
  udpsink host=127.0.0.1 port=5000

# On receiver — use GST_TRACERS to measure latency
GST_TRACERS="latency" GST_DEBUG="GST_TRACER:7" \
  gst-launch-1.0 udpsrc port=5000 ! ...
```

### Common Causes and Fixes

**1. Encoder not configured for low latency**

```bash
# Bad — default x264enc settings have lookahead and B-frames
x264enc

# Good — zero latency mode
x264enc tune=zerolatency speed-preset=ultrafast key-int-max=30
```

**2. Jitter buffer latency too high**

```bash
# Default is 200ms — reduce for local network
rtpjitterbuffer latency=50      # 50ms for local WiFi
rtpjitterbuffer latency=0       # Disable (only on stable LAN)
```

**3. Queue buffers building up**

```bash
# Problem: queue fills up, adding latency
queue

# Fix: Limit queue size and make it leaky
queue max-size-buffers=1 leaky=downstream
```

**4. sync=true on sink (default)**

```bash
# Problem: sink waits for clock, adds latency
autovideosink

# Fix: Disable sync
autovideosink sync=false
```

**5. Decoder buffering**

```bash
# Some decoders buffer multiple frames
avdec_h264 max-threads=1    # Reduce threading overhead
```

### Optimized Low-Latency Full Pipeline

```bash
# Rover
gst-launch-1.0 \
  v4l2src ! video/x-raw,width=640,height=480,framerate=30/1 ! \
  videoconvert ! \
  x264enc tune=zerolatency speed-preset=ultrafast bitrate=2000 key-int-max=30 ! \
  h264parse ! rtph264pay config-interval=1 ! \
  queue max-size-buffers=1 leaky=downstream ! \
  udpsink host=192.168.1.50 port=5000 sync=false async=false

# GCS
gst-launch-1.0 \
  udpsrc port=5000 buffer-size=2097152 ! \
  application/x-rtp,encoding-name=H264,payload=96 ! \
  rtpjitterbuffer latency=50 ! \
  rtph264depay ! h264parse ! avdec_h264 ! \
  videoconvert ! autovideosink sync=false
```

---

## Dropped Frames — Bandwidth or CPU Bottleneck

### Symptom

```
WARNING: A lot of buffers are being dropped
```
or video stutters and freezes periodically.

### Diagnosis

```bash
# Enable QoS messages — shows dropped frame stats
GST_DEBUG=GST_QOS:5 gst-launch-1.0 ...

# Monitor CPU usage
htop
# Look for: gst-launch near 100% on one core

# Monitor network bandwidth
iftop -i wlan0
# or
nload wlan0
```

### Common Causes and Fixes

**1. CPU overload (software encoder on slow hardware)**

```bash
# Problem: x264enc using 100% CPU on Pi
# Fix option A: Lower resolution
video/x-raw,width=640,height=480,framerate=15/1

# Fix option B: Use hardware encoder
v4l2h264enc    # Raspberry Pi
nvv4l2h264enc  # Jetson

# Fix option C: Reduce encoding quality
x264enc speed-preset=ultrafast     # Already fastest
x264enc speed-preset=ultrafast bitrate=1000  # Lower bitrate = less work
```

**2. Network bandwidth exceeded**

```bash
# Problem: Bitrate set too high for available WiFi
# Fix: Reduce encoder bitrate
x264enc bitrate=1000    # Drop to 1 Mbps

# Check actual network usage while streaming
iftop
```

**3. Receiver processing too slow (Python/OpenCV)**

```bash
# Problem: OpenCV frame processing can't keep up with 30fps
# Fix: Use leaky appsink — drop frames if processing is slow
appsink max-buffers=1 drop=true emit-signals=true

# Fix: Lower framerate at capture
video/x-raw,framerate=15/1
```

**4. UDP receive buffer too small**

```bash
# Problem: OS dropping packets at socket level
# Diagnosis: check socket drops
netstat -su | grep -i "receive buffer errors"

# Fix: Increase buffer
sudo sysctl -w net.core.rmem_max=26214400
udpsrc buffer-size=26214400 port=5000
```

**5. Pipeline clock issue (buffer starvation)**

```bash
# Add queue elements between threaded stages
... ! queue ! x264enc ! queue ! rtph264pay ! ...
```

### Frame Drop QoS Diagnostic

```python
# In Python — listen for QoS messages on the bus
def on_message(bus, message):
    if message.type == Gst.MessageType.QOS:
        live, running_time, stream_time, timestamp, duration = message.parse_qos()
        stats = message.parse_qos_stats()
        values = message.parse_qos_values()
        print(f"QOS: dropped={stats[2]}, processed={stats[1]}")
```

---

## Common Error Reference

| Error Message | Likely Cause | Fix |
|--------------|-------------|-----|
| `could not link [A] to [B]` | Caps mismatch | Add `videoconvert` or `capsfilter` between elements |
| `device /dev/video0 not found` | Camera not connected or wrong path | Check with `v4l2-ctl --list-devices` |
| `no element "x264enc"` | Plugin not installed | `sudo apt install gstreamer1.0-plugins-ugly` |
| `no element "nvv4l2h264enc"` | Not on Jetson or plugin missing | Install NVIDIA GStreamer plugins |
| `Internal data stream error` | Element crashed mid-stream | Enable `GST_DEBUG=5`, check upstream element |
| `Received EOS on all sinks` | Source ran out of data | Normal for filesrc; unexpected for live source |
| `streaming stopped, reason not-linked` | Pad not connected | Check pipeline link order |
| `A lot of buffers are being dropped` | CPU/bandwidth bottleneck | Reduce bitrate, use hardware encoder, add leaky queue |
| `clock not running` | Pipeline clock issue | Set `pipeline.use_clock(None)` or check sync settings |
| `rtph264pay: no caps on pad` | Missing h264parse before pay | Insert `h264parse` between encoder and `rtph264pay` |

---

## Quick Diagnostic Checklist

```
□ Does gst-inspect-1.0 [element] show the element exists?
□ Does the camera appear in v4l2-ctl --list-devices?
□ Does a simple test pipeline work? (videotestsrc ! autovideosink)
□ Does the camera source work alone? (v4l2src ! fakesink)
□ Is videoconvert inserted before the encoder?
□ Is h264parse inserted before rtph264pay?
□ Is the destination IP/port correct in udpsink?
□ Is the firewall blocking the UDP port?
□ Is the UDP receive buffer large enough?
□ Are sync=false and async=false set on sinks?
□ Is the encoder set to tune=zerolatency for low latency?
□ Is the jitter buffer latency set appropriately for the network?
```
