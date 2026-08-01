---
title: "GPS Receiver GUI — GCS Side"
date: 2026-03-05
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, gps]
---

## Overview

This is a minimal PyQt6 desktop application that runs on the GCS laptop and displays live GPS coordinates received from the rover over UDP. It listens on port 5005, parses the incoming `lat,lng` payload sent by the [Serial-to-UDP Bridge]({{ site.baseurl }}/posts/gps-serial-to-udp-bridge-rover-side/), and updates two on-screen labels in real time.

---

## Where It Fits

```
Serial-to-UDP Bridge (rover)
    │
    │  UDP datagram: "12.576480,76.905350"
    │  port 5005
    ▼
gps_receiver_gui.py  ←── runs here, on GCS laptop
    │
    ▼
┌─────────────────┐
│  GPS Receiver   │
│                 │
│  LAT: 12.576480 │
│  LNG: 76.905350 │
└─────────────────┘
```

---

## Dependencies

```bash
pip install PyQt6
```

`socket` is Python stdlib — no install needed.

---

## Code Walkthrough

### Threading Model

Qt applications must never block the main thread — doing so freezes the UI. Receiving UDP data is a blocking operation (`sock.recvfrom` waits indefinitely for the next packet). To keep the UI responsive while waiting for UDP data, the socket is run in a background thread using `QThread`.

```
Main Thread (Qt event loop)
    └─ GPSWindow (QWidget)
         └─ lat_label, lng_label

Background Thread (UDPListener QThread)
    └─ sock.recvfrom(1024)  ← blocks here until packet arrives
    └─ emits data_received signal → Qt delivers to main thread
```

Qt's signal/slot mechanism handles the thread boundary safely — the signal is emitted from the background thread, but Qt queues the delivery and calls the connected slot on the main thread. This is the correct pattern for updating UI from a background thread in Qt.

---

### UDPListener

```python
class UDPListener(QThread):
    data_received = pyqtSignal(str, str)

    def run(self):
        sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        sock.bind((UDP_IP, UDP_PORT))

        while True:
            data, _ = sock.recvfrom(1024)
            try:
                lat, lng = data.decode().split(",")
                self.data_received.emit(lat, lng)
            except:
                pass
```

`sock.bind(("0.0.0.0", 5005))` listens on all network interfaces on the machine. Any UDP packet arriving at port 5005 from any source will be received — not just from the rover. In a field environment with only the rover sending to this port this is fine.

`recvfrom(1024)` blocks until a packet arrives, then returns up to 1024 bytes. The expected payload (`"12.576480,76.905350"`) is far shorter than 1024 bytes.

The payload is decoded from bytes to a string and split on `,` to extract lat and lng. If the packet is malformed (missing comma, extra fields, encoding error), the bare `except: pass` silently discards it and the loop continues waiting for the next packet. The UI simply does not update on bad packets — it holds the last valid value.

`data_received.emit(lat, lng)` fires the Qt signal with both values as strings. Qt routes this to `GPSWindow.update_data` on the main thread.

---

### GPSWindow

```python
class GPSWindow(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("GPS Receiver")
        self.setGeometry(300, 200, 300, 150)

        self.lat_label = QLabel("LAT: ---")
        self.lng_label = QLabel("LNG: ---")
        self.lat_label.setStyleSheet("font-size: 18px;")
        self.lng_label.setStyleSheet("font-size: 18px;")

        layout = QVBoxLayout()
        layout.addWidget(self.lat_label)
        layout.addWidget(self.lng_label)
        self.setLayout(layout)

        self.listener = UDPListener()
        self.listener.data_received.connect(self.update_data)
        self.listener.start()
```

`QLabel("LAT: ---")` sets the initial text shown before any data arrives. `---` makes it obvious the display is waiting for a signal rather than showing a zero or blank.

`self.listener.start()` launches `UDPListener.run()` in a background thread. From this point the background thread is blocking on `recvfrom` independently of anything the Qt event loop does.

---

### update_data

```python
def update_data(self, lat, lng):
    self.lat_label.setText(f"LAT: {lat}")
    self.lng_label.setText(f"LNG: {lng}")
```

Called on the main thread by Qt's signal delivery mechanism every time a valid packet arrives. Simply updates both labels. No parsing or validation — the values are displayed as received from the bridge. If the bridge sends `12.576480,76.905350`, the labels show `LAT: 12.576480` and `LNG: 76.905350`.

---

### Entry Point

```python
app = QApplication(sys.argv)
window = GPSWindow()
window.show()
sys.exit(app.exec())
```

Standard PyQt6 entry point. `app.exec()` starts the Qt event loop and blocks until the window is closed. `sys.exit()` ensures the process returns the correct exit code to the OS.

Note that the entry point is not inside a `if __name__ == "__main__":` guard — this means the application would also launch if this file were ever imported as a module. This is not an issue for a standalone script but is worth fixing if the codebase grows.

---

## How to Run

```bash
# On the GCS laptop
python3 gps_receiver_gui.py
```

The window opens immediately showing `LAT: ---` and `LNG: ---`. Values update as soon as UDP packets arrive from the rover. No connection is required before launch — the socket listens passively.

---

## Expected Appearance

```
┌─────────────────────────┐
│  GPS Receiver           │
│                         │
│  LAT: 12.576480         │
│  LNG: 76.905350         │
│                         │
└─────────────────────────┘
```

---

## Known Limitations

| Issue | Detail |
|---|---|
| No timestamp on received data | The display shows the last received coordinate with no indication of when it was received. If the rover stops transmitting, stale coordinates remain on screen indefinitely with no visual warning. Add a `QTimer` that turns the labels red if no update is received within 5 seconds. |
| Bare `except: pass` swallows all errors | Any exception during parsing is silently ignored. If the bridge changes its payload format, the display will simply stop updating with no error message. At minimum log the exception. |
| No entry point guard | The application launches on import. Wrap the bottom three lines in `if __name__ == "__main__":`. |
| Displays raw string, no formatting validation | Lat/lng are displayed exactly as received. If the bridge sends a malformed value (e.g. an empty string after a partial Serial read), it appears on screen as-is. |
| Port conflict gives no clear error | If port 5005 is already in use by another process, `sock.bind()` raises `OSError` inside the background thread, which silently kills the thread. The window stays open but never updates. Add a try/except around `sock.bind()` with an error message. |
