---
title: "Multi-Camera Feed GUI System"
date: 2025-09-16
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, video, pyside6]
---

## Project Goal

To design and implement a **real-time, multi-camera GUI system** to stream, monitor, and manage six USB camera feeds from a rover platform. The goal was to ensure live feedback, flexible resolution control, stream restarts, and error handling in field conditions.

---

## Why We Chose a Python-Based GUI

We considered two primary options:

- **Web-based GUI (Kivy or JS based frameworks)**  
- **Python-based Desktop GUI (PyQt5/PySide6)**

After discussion and experimentation, we chose a **Python desktop GUI using PySide6**, because:

- **Gstreamer integration**: Pyside6 has an inbuilt Gstreamer integration through `Gst` and `GstVideo`.
- **Memory Management and latency**: Avoids memory leak problems while streaming multiple video feeds. No browser overhead. 
- **Simpler threading and concurrency control**: With Python’s `asyncio`.
- **Simple launch setup**: Easier deployment on GCS laptop.

---

## System Architecture

- **Frontend**: Python GUI built with PySide6  
- **Streaming**: GStreamer pipelines using `v4l2src → H264 encoding → UDP streaming`  
- **Backend Server**: Python `asyncio` + `websockets`  
- **Transport Protocol**: UDP for video feeds, WebSockets based requests for resolution control.
- **Fault Handling**: Watchdogs for each stream enabling auto-restart in the event of a crash. 
- **Total Cameras Supported**: 6 (can be extended)

---

## What We Built

### GUI Dashboard
- Built using PySide6.
- 6 camera feed windows arranged in a grid.
- Each camera has:
  - **Live video feed**
  - **Resolution dropdown** (`320x240` to `1920x1080`)
  - **Restart button** (client-side pipeline restart)
  - **Stop button** (server-side process termination via WebSocket)

### Resolution Control
- Operator can change resolution from the dropdown.
- A message is sent via **WebSocket** to the backend in JSON format:
  ```json
  {
    "camera_id": 2,
    "width": 800,
    "height": 600
  }
- Server stops the old stream and starts a new one with updated parameters.
- This can also be used to start a stream after using the "Stop" button.

## WebSocket Resilience
- Initial implementation would crash if the connection was dropped.
- We added reconnection logic using exponential backoff:
    - Background thread continuously attempts reconnection.
    - On reconnect, the GUI can resume sending resolution change requests.

## GUI Architecture Overview

- Built using **PySide6** (Qt for Python).
- Main window (`Dashboard`) arranges 6 camera widgets in a **2×3 grid layout**.
- Each `CameraWidget` includes:
  - Video display label (target for GStreamer overlay).
  - Resolution dropdown (`QComboBox`).
  - Expand button to pop out a full-size view.
  - Restart button (restarts the local GStreamer pipeline).
  - Stop button (sends stop signal to backend server).

---

## GStreamer Integration

- Each feed uses UDP streaming with video decoding and rendering pipelines.
- Pipelines are dynamically constructed based on selected resolution.
- Port numbers are fixed per device to handle up to 6 camera streams. `5000,5001,5002..etc`

---

## WebSocket Backend Communication

- The GUI uses a persistent WebSocket connection to communicate with the server.
- Connection is handled in a separate asyncio thread with reconnection logic.
- Messages sent from the GUI include camera ID and selected resolution.
- The server:
  - Validates if the requested resolution is supported by the camera.
  - Stops and restarts the GStreamer pipeline for the camera.
  - Sends confirmation or error messages back to the client.

---

## What Worked

### WebSocket Integration

- A dedicated asyncio loop in a separate thread handled reconnects and message sending.
- A thread-safe signaling mechanism (`ws_connected_event`) ensured safe communication.
- Resolution changes triggered correct actions on the backend.

### Reliable Streaming via GStreamer

- Low-latency and robust UDP-based streaming with multiple cameras worked reliably.
- Pipelines successfully rendered using Qt’s native VideoOverlay integration.

### Overlay Expansion

- Pop-out expansion windows reused the same GStreamer pipeline with a different label as target.

### Device Reconnection

- Server included monitoring logic to detect and automatically restart crashed camera pipelines.
- Stream locks ensured that no two threads accessed the same camera simultaneously.

---

## What Didn’t Work Initially (and Fixes)

### WebSocket Not Reconnecting After Drop

- Issue: The WebSocket connection would not recover after a drop or server restart.
- Fix: Added persistent retry logic with exponential backoff and connection state monitoring using threading events.

### Resolution Changes Had No Effect

- Issue: Resolution changes appeared to have no impact on actual stream output. Some resolutions were not supported by the cameras.
- Fix:
  - Confirmed JSON messages were reaching the server.
  - Implemented resolution validation using `v4l2-ctl`.
  - Used only the **intersection of all supported resolutions** across different USB cameras to avoid unsupported formats.

---

## Panorama Integration Challenge

### Context

- During panoramic stitching, the system does not know in advance which camera will be selected.
- Once selected, the corresponding stream must be stopped immediately to release the device for panorama capture.

### Implemented Solution

- A **Stop** button was added per camera widget, triggering a server-side stream kill.
- Restarting the stream is currently handled by **changing the resolution**, which reinitializes the stream from scratch.

### Future Scope

- Add a **Toggle** button instead of **Stop** button which will check current state before inverting it.

---

## Testing Strategy

- Manual testing with 6 different USB cameras.
- Verified functionality for:
  - Stream start/stop.
  - Resolution switching.
  - GUI scaling and responsiveness.
  - Reconnection behavior of WebSocket.
- Resolution compatibility was verified using `v4l2-ctl`.

---

## Final Notes and Future Improvements

- WebSocket reconnection logic has been made reliable.
- GUI supports flexible, modular camera layout with clean overlays.
- Suggested future enhancements:
  - Show FPS or bitrate indicators.
  - Centralized Restart/Stop All buttons.
  - Dynamic resolution detection per camera.
  - Display of actual resolution after stream begins.

---

## Final Result

A robust, real-time dashboard for managing 6 cameras onboard the rover:
- Live video feeds from all cameras
- Resolution control per stream
- Stream stop and restart control
- Crash recovery and auto-restart
- Clean, extensible GUI built on PySide6

This system will serve as the foundation for further visual analysis of sites, panoramic image stitching during AbEx and teleoperation during RADO and IDMO.
