---
title: "GStreamer — Encoding & Transport Protocols"
date: 2026-03-05
author: mahi
categories: [software, gstreamer]
tags: [gcs, gstreamer, video, rtp]
---

## Video Encoding Overview

Before video is transmitted over a network, it must be **compressed (encoded)** to reduce bandwidth. Raw video is far too large to stream in real time.

**Example — Raw 1080p30 video:**
```
1920 × 1080 pixels × 3 bytes (BGR) × 30 fps = ~186 MB/s
With H.264 encoding at medium quality: ~2–5 Mbps (~0.3–0.6 MB/s)
Compression ratio: ~300:1
```

---

## H.264 vs H.265

### H.264 (AVC — Advanced Video Coding)

- The most widely supported video codec
- Excellent hardware support across all platforms
- Good compression with low CPU cost
- Standard for real-time streaming

**Characteristics:**
- Bitrate: 1–10 Mbps for HD video
- Latency: Low (especially with `tune=zerolatency`)
- CPU cost: Low to medium
- Hardware encoders: `v4l2h264enc` (Pi), `nvv4l2h264enc` (Jetson), `nvenc` (NVIDIA PC)

### H.265 (HEVC — High Efficiency Video Coding)

- Successor to H.264 — ~50% better compression at same quality
- Higher CPU cost; not all decoders support it
- Preferred when **bandwidth is critically limited**

**Characteristics:**
- Bitrate: 0.5–5 Mbps for HD video (half of H.264 for same quality)
- Latency: Slightly higher than H.264
- CPU cost: High (software); similar to H.264 if hardware-accelerated
- Hardware encoders: `nvv4l2h265enc` (Jetson), `vaapih265enc` (Intel)

### H.264 vs H.265 Comparison

| Feature | H.264 | H.265 |
|---------|-------|-------|
| Compression Efficiency | Good | ~50% better |
| Bitrate (1080p30) | 2–8 Mbps | 1–4 Mbps |
| CPU Cost (software) | Low–Medium | High |
| Hardware Support | Universal | Limited (newer hardware) |
| Decoding Compatibility | All platforms | Newer platforms |
| Latency | Low | Slightly higher |
| GStreamer Encoder | `x264enc`, `v4l2h264enc` | `x265enc`, `nvv4l2h265enc` |
| **Best for rovers** | ✓ Default choice | Bandwidth-constrained links |

> **Recommendation for rovers:** Start with H.264. Switch to H.265 only if bandwidth is critically limited and your hardware supports it.

---

## Hardware Acceleration

Software encoding (e.g. `x264enc`) uses the CPU. On embedded hardware like Raspberry Pi or Jetson, this competes with navigation and autonomy tasks. **Hardware encoders** offload this to a dedicated chip.

### Common Hardware Encoders

| Hardware | H.264 Encoder | H.265 Encoder | Notes |
|----------|--------------|--------------|-------|
| Raspberry Pi 4 | `v4l2h264enc` | — | Via V4L2 M2M API |
| NVIDIA Jetson | `nvv4l2h264enc` | `nvv4l2h265enc` | NVENC hardware |
| Intel (VAAPI) | `vaapih264enc` | `vaapih265enc` | Intel Quick Sync |
| NVIDIA GPU (PC) | `nvh264enc` | `nvh265enc` | NVENC |
| Generic V4L2 | `v4l2h264enc` | `v4l2h265enc` | SoC encoders |

### Raspberry Pi Pipeline (Hardware)

```bash
gst-launch-1.0 v4l2src device=/dev/video0 ! \
  video/x-raw,width=1280,height=720,framerate=30/1 ! \
  v4l2h264enc extra-controls="controls,repeat_sequence_header=1" ! \
  h264parse ! rtph264pay ! \
  udpsink host=192.168.1.50 port=5000
```

### Jetson Pipeline (Hardware NVENC)

```bash
gst-launch-1.0 nvarguscamerasrc ! \
  video/x-raw(memory:NVMM),width=1280,height=720,framerate=30/1 ! \
  nvv4l2h264enc bitrate=4000000 ! \
  h264parse ! rtph264pay ! \
  udpsink host=192.168.1.50 port=5000
```

---

## Bitrate, Keyframe Interval, and Latency

### Bitrate

Bitrate controls the **amount of data per second** used to represent the video.

| Bitrate | Quality | Bandwidth Use | Use Case |
|---------|---------|---------------|---------|
| 500 Kbps | Low | Very low | Emergency fallback |
| 1–2 Mbps | Medium | Low | Teleoperation over LTE |
| 4–8 Mbps | High | Medium | Local WiFi streaming |
| 10+ Mbps | Very high | High | Short-range, high-quality |

**Setting bitrate in GStreamer:**
```bash
# x264enc (software)
x264enc bitrate=4000    # 4 Mbps (in Kbps for x264enc)

# nvv4l2h264enc (Jetson hardware)
nvv4l2h264enc bitrate=4000000   # 4 Mbps (in bps)
```

### Keyframe Interval (I-frame Interval)

A **keyframe (I-frame)** is a complete frame — it does not depend on other frames. Non-keyframes (P-frames, B-frames) only store differences from previous frames.

| Keyframe Interval | Effect |
|------------------|--------|
| Low (1–30 frames) | Faster recovery from packet loss, higher bitrate |
| High (60–300 frames) | Better compression, slower recovery |

**For rover streaming:** Set keyframe interval to **30–60 frames** (1–2 seconds at 30fps) — balances bandwidth and recovery speed.

```bash
x264enc key-int-max=30 tune=zerolatency
```

### Latency Tuning

Key encoder settings to minimize end-to-end latency:

```bash
# x264enc low latency settings
x264enc tune=zerolatency speed-preset=ultrafast key-int-max=30

# Explanation:
# tune=zerolatency     — disables b-frames and buffering
# speed-preset=ultrafast — fastest encoding, slightly lower quality
# key-int-max=30       — keyframe every 1 second
```

### Profile and Level

The H.264 **profile** defines which encoding features are used:

| Profile | Description | Use Case |
|---------|-------------|---------|
| `baseline` | Simplest, no B-frames | Low-power devices, streaming |
| `main` | Moderate complexity | Balanced |
| `high` | Full feature set | Recording, high quality |

The **level** defines maximum bitrate and resolution constraints (e.g. `level=3.1` for 720p).

```bash
x264enc tune=zerolatency speed-preset=ultrafast \
  option-string="profile=baseline:level=3.1"
```

---

## Transport Protocols

### RTP — Real-Time Transport Protocol

**RTP** is the standard protocol for **packetizing and transmitting multimedia** over IP networks.

**What RTP does:**
- Wraps encoded video/audio into network packets
- Adds sequence numbers for ordering
- Adds timestamps for synchronization
- Adds payload type identifier (H.264, H.265, etc.)
- Runs over **UDP** (usually)

**RTP does NOT handle:**
- Connection setup
- Stream discovery
- Reliability or retransmission

```
Encoded H.264 data
        ↓
[rtph264pay]  ← Packetizes H.264 into RTP packets
        ↓
[udpsink]     ← Sends RTP packets via UDP
```

**GStreamer RTP elements:**

| Element | Function |
|---------|----------|
| `rtph264pay` | Packetize H.264 into RTP |
| `rtph264depay` | Depacketize RTP into H.264 |
| `rtph265pay` | Packetize H.265 into RTP |
| `rtpjitterbuffer` | Buffer RTP packets, handle reordering |

### RTSP — Real Time Streaming Protocol

**RTSP** is a **session control protocol** — it sits on top of RTP and handles stream negotiation, setup, and control.

Think of RTSP as the "remote control" for an RTP stream:

```
Client (GCS)               Server (Rover)
    │                           │
    │──── RTSP DESCRIBE ───────►│  "What streams do you have?"
    │◄─── 200 OK + SDP ─────────│  "I have H.264 video on port 5000"
    │                           │
    │──── RTSP SETUP ──────────►│  "Set up the stream"
    │◄─── 200 OK ───────────────│
    │                           │
    │──── RTSP PLAY ───────────►│  "Start streaming"
    │◄═══ RTP data (UDP) ═══════│  ← Actual video data
    │                           │
    │──── RTSP TEARDOWN ───────►│  "Stop streaming"
```

**RTSP default port:** `554`  
**RTSP URL format:** `rtsp://192.168.1.20:8554/stream`

**GStreamer RTSP elements:**

| Element | Function |
|---------|----------|
| `rtspsrc` | Receive RTSP stream (client) |
| `GstRTSPServer` | Serve RTSP stream (server, via library) |

```bash
# Receive RTSP stream
gst-launch-1.0 rtspsrc location=rtsp://192.168.1.20:8554/stream ! \
  rtph264depay ! h264parse ! avdec_h264 ! videoconvert ! autovideosink
```

---

## UDP vs TCP for Video Transport

| Feature | UDP | TCP |
|---------|-----|-----|
| Latency | Low | Higher |
| Packet Loss Handling | None (drop and continue) | Retransmit (causes delay) |
| Connection Setup | None | Handshake required |
| Buffering | Minimal | Large TCP buffer |
| Suitability for video | ✓ Preferred | ✗ Causes lag spikes |

**Why UDP is preferred for video:**

In real-time video, a **retransmitted frame arrives too late to be useful**. It only adds delay and congestion. UDP drops the packet and the next frame is displayed, causing a brief visual glitch rather than a freeze.

```
TCP on packet loss:    frame 1 → [WAIT] → retransmit → frame 2 → frame 3
                                  ↑ visible freeze / lag spike

UDP on packet loss:    frame 1 → [DROP] → frame 3 → frame 4
                                  ↑ brief visual artifact, stream continues
```

---

## SRT — Secure Reliable Transport

**SRT** is a modern low-latency streaming protocol designed for unreliable networks (internet, LTE, radio links). It combines the speed of UDP with selective retransmission.

### SRT Features

| Feature | Description |
|---------|-------------|
| **Low latency** | Configurable latency buffer (20ms–8000ms) |
| **Selective ARQ** | Retransmits only lost packets, within latency budget |
| **Encryption** | AES-128/256 built-in |
| **Bandwidth estimation** | Adapts to network conditions |
| **Works over internet** | Handles NAT, firewalls, variable bandwidth |

### SRT vs RTP over UDP

| Feature | RTP/UDP | SRT |
|---------|---------|-----|
| Packet loss recovery | None | Selective retransmission |
| Encryption | None (add DTLS separately) | Built-in AES |
| Network traversal | Manual (port forward) | Built-in NAT traversal |
| Latency | Very low (~5ms) | Low (configurable, ~100ms+) |
| Complexity | Simple | Moderate |
| Best for | Local WiFi (reliable) | LTE / internet (unreliable) |

### SRT in GStreamer

```bash
# Sender (rover)
gst-launch-1.0 v4l2src ! videoconvert ! x264enc tune=zerolatency ! \
  h264parse ! mpegtsmux ! srtsink uri="srt://:5000"

# Receiver (GCS)
gst-launch-1.0 srtsrc uri="srt://192.168.1.20:5000" ! \
  tsdemux ! h264parse ! avdec_h264 ! videoconvert ! autovideosink
```

---

## Multicast vs Unicast

### Unicast

- One sender → one receiver
- Separate stream per receiver
- Standard for rover ↔ single GCS

```
Rover ──► GCS (one stream)
```

### Multicast

- One sender → multiple receivers
- Only one stream transmitted; network replicates it
- All receivers use the same multicast IP (range: `224.0.0.0 – 239.255.255.255`)

```
Rover ──► Switch ──► GCS 1
                 ──► GCS 2
                 ──► GCS 3
(one stream from rover, replicated by switch/router)
```

**When to use multicast:** Multiple GCS operators or display stations need the same stream simultaneously on a local network.

```bash
# Multicast sender
gst-launch-1.0 v4l2src ! videoconvert ! x264enc ! rtph264pay ! \
  udpsink host=224.1.1.1 port=5000 auto-multicast=true

# Multicast receiver
gst-launch-1.0 udpsrc address=224.1.1.1 port=5000 auto-multicast=true ! \
  application/x-rtp,media=video,encoding-name=H264 ! \
  rtph264depay ! h264parse ! avdec_h264 ! videoconvert ! autovideosink
```
