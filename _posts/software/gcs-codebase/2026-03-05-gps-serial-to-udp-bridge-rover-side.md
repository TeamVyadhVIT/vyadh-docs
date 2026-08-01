---
title: "GPS Serial to UDP Bridge — Rover Side"
date: 2026-03-05
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, gps, udp]
---

## Overview

This script is the link between the ESP32 and the GCS network. It runs on the rover's onboard computer (e.g. a Raspberry Pi or laptop connected to the ESP32 via USB), reads the ESP32's Serial output line by line, and whenever it has collected both a latitude and a longitude value it sends them as a single UDP datagram to the GCS.

It is a deliberate pass-through — it does not process or validate the GPS data, it only repackages it from Serial to UDP.

---

## Where It Fits

```
ESP32 (USB Serial, 115200 baud)
    │
    │  "LAT: 12.576480"
    │  "LNG: 76.905350"
    │  (and other lines it ignores)
    ▼
serial_udp_bridge.py  ←── runs here, on rover computer
    │
    │  UDP datagram: "12.576480,76.905350"
    │  destination: 192.168.1.26:5005
    ▼
GPS Receiver GUI (GCS laptop)
```

---

## Configuration

These four constants at the top of the file must be correct before running:

```python
SERIAL_PORT = "/dev/ttyUSB0"   # USB port the ESP32 is on
BAUDRATE    = 115200            # Must match Serial.begin() in ESP32 firmware
UDP_IP      = "192.168.1.26"   # GCS laptop's IP address on the field network
UDP_PORT    = 5005              # Must match the port the GCS receiver listens on
```

### Finding the Correct Serial Port

On Linux (Raspberry Pi or Ubuntu laptop):
```bash
ls /dev/ttyUSB*     # ESP32 usually appears as ttyUSB0 or ttyUSB1
ls /dev/ttyACM*     # Some ESP32 boards use ttyACM0
dmesg | tail -20    # Check kernel messages after plugging in
```

On Windows, check Device Manager under "Ports (COM & LPT)".

---

## Dependencies

```bash
pip install pyserial
```

`socket` is Python stdlib — no install needed.

---

## Code Walkthrough

### Serial and Socket Setup

```python
ser  = serial.Serial(SERIAL_PORT, BAUDRATE, timeout=1)
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
```

`timeout=1` means `ser.readline()` will wait up to 1 second for a complete line before returning an empty string. Without a timeout, `readline()` blocks indefinitely if the ESP32 stops sending — this would freeze the script. With timeout=1, the loop keeps running even during brief Serial gaps.

`socket.SOCK_DGRAM` creates a UDP socket. UDP is connectionless — `sock.sendto()` fires the packet at the target address without any prior handshake or connection state. If the GCS receiver is not running, the packet is silently dropped.

---

### Main Loop

```python
lat = None
lng = None

while True:
    line = ser.readline().decode(errors='ignore').strip()

    if line.startswith("LAT:"):
        lat = line.split(":")[1].strip()
    elif line.startswith("LNG:"):
        lng = line.split(":")[1].strip()

    if lat and lng:
        msg = f"{lat},{lng}"
        sock.sendto(msg.encode(), (UDP_IP, UDP_PORT))
        print("Sent:", msg)
        lat, lng = None, None
```

**Line parsing:**

`ser.readline()` returns a bytes object terminated by `\n`. `.decode(errors='ignore')` converts it to a string, silently dropping any bytes that are not valid UTF-8 (this prevents crashes from NMEA binary sequences or garbage on the line at startup). `.strip()` removes `\r\n` and any trailing whitespace.

The script only looks for two line prefixes. Everything else — raw NMEA sentences (`$GPGGA`, `$GPRMC`), MQ sensor lines, separator lines — is read and discarded. This is intentional: the bridge is GPS-only.

**Pairing logic:**

`lat` and `lng` are only sent together. The ESP32 firmware always prints LAT before LNG on consecutive lines, so in normal operation this pairs correctly every cycle. The `None, None` reset after sending ensures a stale lat is never paired with a new lng from the next cycle.

> **Pairing Assumption**
>
> The pairing logic assumes LAT always arrives before LNG and that they always arrive in the same 2-second cycle with no intervening loss. If a Serial line is garbled or dropped (e.g. at startup when the ESP32 is booting), a `lat` value from one cycle could sit in memory and pair with a `lng` from a later cycle, producing a mismatched coordinate. This is unlikely in normal operation but possible. A timestamp or sequence number in the Serial protocol would make pairing robust.
{: .prompt-warning }

**UDP send:**

```python
msg = f"{lat},{lng}"                           # e.g. "12.576480,76.905350"
sock.sendto(msg.encode(), (UDP_IP, UDP_PORT))  # send to GCS
```

The payload is a plain comma-separated string. The GCS receiver splits on `,` to recover lat and lng. This format is intentionally minimal — no headers, no framing, no error detection.

---

## How to Run

```bash
# On the rover's onboard computer (Raspberry Pi, etc.)
python3 serial_udp_bridge.py
```

Expected terminal output when working:
```
Reading serial and sending UDP...
Sent: 12.576480,76.905350
Sent: 12.576481,76.905351
Sent: 12.576479,76.905349
```

If no "Sent:" lines appear but the script is running, the GPS module has not acquired a fix yet. Check the raw NMEA output by running `cat /dev/ttyUSB0` in another terminal to confirm the ESP32 is transmitting.

---

## Known Limitations

| Issue | Detail |
|---|---|
| No error detection on the UDP payload | If the GCS receiver gets a malformed packet, `split(",")` will fail silently (the receiver has a bare `except: pass`). No data validation at either end. |
| Hardcoded GCS IP | If the GCS laptop's IP changes (different network, DHCP reassignment), `UDP_IP` must be updated and the script restarted. Consider reading the IP from a config file or argument. |
| No reconnection if Serial port disappears | If the USB cable is unplugged, `ser.readline()` raises `SerialException` and the script crashes. Wrap the loop in a try/except with reconnect logic for reliable field use. |
| MQ sensor data is not forwarded | The bridge discards all non-GPS lines. MQ sensor data goes nowhere — it is printed to Serial but not forwarded to the GCS. A separate MQTT publisher handles sensor data (or should, once implemented). |
| Single destination only | Data is sent to one IP. If multiple GCS machines need the GPS feed, the bridge would need to send to multiple addresses or use UDP multicast/broadcast. |
