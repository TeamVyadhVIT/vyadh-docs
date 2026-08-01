---
title: "GStreamer — Core Concepts"
date: 2026-03-05
author: mahi
categories: [software, gstreamer]
tags: [gcs, gstreamer, video]
---

## What is GStreamer?

GStreamer is an open-source, **pipeline-based multimedia framework** written in C. It enables the construction of graphs of media-handling components (elements) to build complex media processing workflows.

**Key characteristics:**
- Cross-platform (Linux, Windows, macOS, embedded)
- Plugin-based architecture — functionality is loaded as plugins
- Supports audio, video, and arbitrary data streams
- Used heavily in robotics, IoT, broadcast, and embedded systems

**Why GStreamer for rovers:**
- Low-latency video streaming from camera to GCS
- Hardware-accelerated encoding on Jetson / Raspberry Pi
- Flexible pipeline construction for multi-camera setups
- Python bindings for programmatic control

---

## Pipeline Model

A GStreamer **pipeline** is a directed graph of elements through which media data flows from a source to a sink.

```
[Source] → [Filter/Transform] → [Filter/Transform] → [Sink]

Example:
[v4l2src] → [videoconvert] → [x264enc] → [rtph264pay] → [udpsink]
  Camera       Color conv      Encode       RTP pack       Send UDP
```

Data flows in one direction: **left to right**, from source to sink. Each element processes the data and passes it downstream.

---

## Elements

An **element** is the fundamental building block of a GStreamer pipeline. Every element performs a specific task.

### Element Types

| Type | Description | Examples |
|------|-------------|---------|
| **Source** | Produces data, no input | `v4l2src`, `filesrc`, `videotestsrc` |
| **Filter / Transform** | Processes data, has input and output | `videoconvert`, `x264enc`, `capsfilter` |
| **Sink** | Consumes data, no output | `udpsink`, `autovideosink`, `filesink` |
| **Mux / Demux** | Combines or splits streams | `matroskamux`, `qtdemux` |
| **Parser** | Parses stream format | `h264parse`, `mpegvideoparse` |

### Element States

Every element transitions through states:

```
NULL → READY → PAUSED → PLAYING
```

| State | Description |
|-------|-------------|
| `NULL` | Initial state, no resources allocated |
| `READY` | Resources allocated, not yet streaming |
| `PAUSED` | Pipeline ready, data flows but clock is paused |
| `PLAYING` | Fully active, data is flowing |

---

## Pads

**Pads** are the connection points on elements — they are how elements link to each other.

### Pad Types

| Type | Description |
|------|-------------|
| **Sink Pad** | Input pad — receives data from upstream |
| **Source Pad** | Output pad — sends data downstream |

```
         Element
┌──────────────────────┐
│  sink pad ──► logic ──► src pad  │
└──────────────────────┘
   (receives)              (sends)
```

### Pad Availability

| Availability | Description |
|-------------|-------------|
| **Always** | Pad always exists (most common) |
| **Sometimes** | Pad created dynamically (e.g. demuxers after probing) |
| **On Request** | Pad created when explicitly requested (e.g. tee, mixer) |

### Linking Pads

```bash
# In gst-launch, elements are linked with !
gst-launch-1.0 videotestsrc ! videoconvert ! autovideosink
#                           ^             ^
#                         link          link
```

Under the hood, GStreamer calls `gst_element_link()` to connect the src pad of one element to the sink pad of the next.

---

## Caps (Capabilities)

**Caps** (Capabilities) define the **type and format of data** that can flow between two pads. Think of caps as a contract between elements.

### What Caps Describe

- Media type (`video/x-raw`, `video/x-h264`, `audio/x-raw`)
- Width and height
- Framerate
- Pixel format (`I420`, `NV12`, `BGR`)
- Encoding profile

### Caps Negotiation

Before data flows, GStreamer performs **caps negotiation** — upstream and downstream elements agree on a compatible format.

```
[v4l2src]                [videoconvert]           [autovideosink]
  src caps:                sink caps:               sink caps:
  video/x-raw,             video/x-raw              video/x-raw,
  format=YUYV,             (any format)              format=BGRx
  width=1280,                 ↕                         ↕
  height=720        ←── negotiate ───────────────── negotiate ──►
  framerate=30/1
```

If caps are incompatible, the pipeline **fails to start** with a negotiation error.

### capsfilter Element

Used to **explicitly enforce** caps at a point in the pipeline:

```bash
gst-launch-1.0 v4l2src ! \
  video/x-raw,width=1280,height=720,framerate=30/1 ! \
  videoconvert ! autovideosink
```

### Inspecting Caps

```bash
# View supported caps for an element
gst-inspect-1.0 v4l2src

# View caps negotiated during a running pipeline
GST_DEBUG=GST_CAPS:4 gst-launch-1.0 ...
```

---

## Pipeline Syntax — gst-launch-1.0

`gst-launch-1.0` is the command-line tool to build and run GStreamer pipelines.

### Basic Syntax

```bash
gst-launch-1.0 [element] [property=value] ! [element] [property=value] ! ...
```

### Common Patterns

```bash
# Test pipeline — generate test video and display
gst-launch-1.0 videotestsrc ! autovideosink

# Capture from camera and display
gst-launch-1.0 v4l2src device=/dev/video0 ! videoconvert ! autovideosink

# Capture, encode, and send over UDP
gst-launch-1.0 v4l2src ! videoconvert ! x264enc ! rtph264pay ! \
  udpsink host=192.168.1.50 port=5000

# Set element properties
gst-launch-1.0 v4l2src device=/dev/video0 num-buffers=100 ! \
  video/x-raw,width=1280,height=720 ! videoconvert ! autovideosink

# Branch a pipeline with tee
gst-launch-1.0 v4l2src ! tee name=t \
  t. ! queue ! autovideosink \
  t. ! queue ! x264enc ! filesink location=output.mp4
```

### Named Elements

Elements can be named for referencing in branches:

```bash
gst-launch-1.0 v4l2src ! tee name=split \
  split. ! queue ! videoconvert ! autovideosink \
  split. ! queue ! x264enc ! rtph264pay ! udpsink host=192.168.1.50 port=5000
```

---

## Bins and Sub-Pipelines

A **Bin** is a container element that holds other elements. It allows grouping elements into a logical unit.

### Pipeline (Top-level Bin)

A **Pipeline** is a special top-level bin. There is exactly one pipeline per application.

```
Pipeline
├── Bin A (capture + encode)
│   ├── v4l2src
│   ├── videoconvert
│   └── x264enc
└── Bin B (transport)
    ├── rtph264pay
    └── udpsink
```

### Why Use Bins?

- Encapsulate complex sub-graphs into reusable units
- Independent state management
- Cleaner programmatic pipeline construction in Python/C

### Example in Python

```python
import gi
gi.require_version('Gst', '1.0')
from gi.repository import Gst

Gst.init(None)

pipeline = Gst.parse_launch(
    "v4l2src ! videoconvert ! x264enc ! rtph264pay ! udpsink host=192.168.1.50 port=5000"
)
pipeline.set_state(Gst.State.PLAYING)
```

---

## GStreamer Bus and Message System

The **GStreamer Bus** is a message-passing mechanism that allows elements to send messages to the application.

### Message Types

| Message | Description |
|---------|-------------|
| `GST_MESSAGE_EOS` | End of stream reached |
| `GST_MESSAGE_ERROR` | Fatal error occurred |
| `GST_MESSAGE_WARNING` | Non-fatal warning |
| `GST_MESSAGE_STATE_CHANGED` | Element state transition |
| `GST_MESSAGE_BUFFERING` | Buffer fill level changed |
| `GST_MESSAGE_QOS` | Quality-of-service message (dropped frames) |

### Listening to the Bus (Python)

```python
bus = pipeline.get_bus()
bus.add_signal_watch()

def on_message(bus, message):
    t = message.type
    if t == Gst.MessageType.EOS:
        print("Stream ended")
        pipeline.set_state(Gst.State.NULL)
    elif t == Gst.MessageType.ERROR:
        err, debug = message.parse_error()
        print(f"Error: {err}, {debug}")

bus.connect("message", on_message)
```

---

## Debugging with GST_DEBUG

GStreamer has a built-in logging system controlled via the `GST_DEBUG` environment variable.

### Debug Levels

| Level | Name | Description |
|-------|------|-------------|
| 0 | NONE | No output |
| 1 | ERROR | Fatal errors only |
| 2 | WARNING | Non-fatal warnings |
| 3 | FIXME | Fixme messages |
| 4 | INFO | Informational messages |
| 5 | DEBUG | Debug messages |
| 6 | LOG | Verbose log messages |
| 7 | TRACE | Very verbose tracing |

### Usage

```bash
# Set global debug level
GST_DEBUG=3 gst-launch-1.0 videotestsrc ! autovideosink

# Debug specific category
GST_DEBUG=v4l2src:5 gst-launch-1.0 v4l2src ! autovideosink

# Debug caps negotiation
GST_DEBUG=GST_CAPS:4 gst-launch-1.0 v4l2src ! autovideosink

# Debug multiple categories
GST_DEBUG=v4l2src:5,x264enc:4,udpsink:3 gst-launch-1.0 ...

# Output to file
GST_DEBUG=5 GST_DEBUG_FILE=/tmp/gst.log gst-launch-1.0 ...
```

### Generating Pipeline Graphs

Visualize the pipeline as a dot graph:

```bash
GST_DEBUG_DUMP_DOT_DIR=/tmp gst-launch-1.0 videotestsrc ! autovideosink

# Convert to PNG
dot -Tpng /tmp/*.dot -o pipeline.png
```
