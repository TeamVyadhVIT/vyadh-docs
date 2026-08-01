---
title: "Backup Client-Side Bash Script"
date: 2025-09-16
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, video, bash]
---

This script provides a **backup client-side video streaming solution** for monitoring multiple RTP/H.264 streams.
It automatically launches streams, allows the user to position them once, saves the positions, and continuously monitors the windows.
If a stream window crashes, it automatically restarts and repositions it to its previous position to avoid confusion during missions.

---

## Features

* Handles **multiple UDP video streams** (ports and names configurable).
* Uses **GStreamer** (`gst-launch-1.0`) to play RTP/H.264 streams.
* Uses **xdotool** to:

  * Detect and rename stream windows.
  * Store window positions.
  * Reposition windows after crashes.
* Implements a **watchdog loop** to auto-restart crashed streams.

---

## Requirements

Make sure the following dependencies are installed:

* **GStreamer** (`gst-launch-1.0`)
* **Gstreamer plugins (good, bad and ugly)**
* **xdotool**

---

## Configuration

Streams are defined by ports and names:

```bash
ports=(5000 5001 5002 5003 5004 5005)
stream_names=("stream1" "stream2" "stream3" "stream4" "stream5" "stream6")
```

* Each `ports[i]` corresponds to `stream_names[i]`.
* Modify these arrays to match your network stream setup.

---

## How It Works

### 1. Start Streams

Start `server.py` and wait 5-10 seconds for all streams to initialize.

---

### 2. Position Windows

After starting all streams:

* User has **10 seconds** to manually position windows on screen.
* Script then saves each window’s position for recovery later.

---

### 3. Watchdog Monitoring

* Continuously checks if each stream window still exists.
* If a stream crashes:

  * Restarts the GStreamer pipeline.
  * Waits for a new window to appear.
  * Renames and repositions it to its saved coordinates.

---

## Workflow

1. Make the script executable:
   ```bash
   chmod +x backup_client.sh
   ```
2. Run the script:

   ```bash
   ./backup_client.sh
   ```

3. Wait for all stream windows to open.

4. Arrange them on the screen within **10 seconds**.

5. Script saves positions and begins watchdog monitoring.

6. If a stream window crashes, it will be automatically restarted and restored.

---

## Key Variables

* `ports[]` → List of UDP ports to listen for RTP video.
* `stream_names[]` → Stream window names for identification.
* `win_ids[]` → Maps each stream index to its window ID.
* `win_coords[]` → Stores window positions for recovery.

---

## Notes

* Ensure the main GUI streaming client is **not running** before launching this backup client.
* Network firewall must allow UDP traffic on the specified ports.
* If new streams are added, update both `ports[]` and `stream_names[]`.

---

## Limitations

* Requires manual positioning at startup before positions are locked in.

---
