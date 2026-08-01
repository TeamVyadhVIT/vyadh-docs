---
title: "ESP32 Firmware — GPS + MQ Sensors"
date: 2026-03-05
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, esp32, gps, firmware]
---

## Overview

This is the firmware running on the rover's ESP32. It runs a continuous loop every 2 seconds that:

1. Drains the GPS module's UART buffer, feeds raw NMEA characters into TinyGPSPlus, and prints latitude/longitude to USB Serial when a valid fix is available.
2. Reads raw ADC values from three MQ gas sensors and prints them to USB Serial.

The Serial output from this firmware is read by the [Serial-to-UDP Bridge]({{ site.baseurl }}/posts/gps-serial-to-udp-bridge-rover-side/) running on the rover's onboard computer, which extracts the GPS lines and forwards them to the GCS.

---

## Hardware

| Component | Role |
|---|---|
| ESP32 | Microcontroller — runs this firmware |
| NEO GPS Module (NEO-6M / 7M / 8M) | Streams NMEA sentences over UART at 9600 baud |
| MQ-4 | Methane / natural gas detection |
| MQ-8 | Hydrogen gas detection |
| MQ-135 | General air quality (CO2, ammonia, smoke) |

---

## Pin Map

| Signal | ESP32 GPIO | Notes |
|---|---|---|
| GPS UART RX (ESP32 receives) | GPIO 22 | Connect to TX pin of GPS module |
| GPS UART TX (ESP32 transmits) | GPIO 23 | Connect to RX pin of GPS module |
| MQ-4 Analog Output | GPIO 33 | ADC1 — safe with WiFi |
| MQ-8 Analog Output | GPIO 34 | ADC1 — input only pin |
| MQ-135 Analog Output | GPIO 35 | ADC1 — input only pin |

> **ADC1 vs ADC2**
>
> Pins 33, 34, 35 are all on ADC1. If WiFi is added to this firmware later, do **not** move sensors to ADC2 pins (GPIO 0, 2, 4, 12–15, 25–27) — ADC2 is disabled when WiFi is active on the ESP32.
{: .prompt-warning }

---

## Dependencies

| Library | Install via |
|---|---|
| `Wire.h` | Arduino built-in |
| `Adafruit_Sensor.h` | Arduino Library Manager → "Adafruit Unified Sensor" |
| `TinyGPSPlus.h` | Arduino Library Manager → "TinyGPSPlus" by Mikal Hart |

---

## Code Walkthrough

### Definitions and Objects

```cpp
#define RX 22
#define TX 23
#define MQ4_PIN   33
#define MQ8_PIN   34
#define MQ135_PIN 35

HardwareSerial neogps(1);
TinyGPSPlus gps;
```

`HardwareSerial neogps(1)` opens UART1 — one of the ESP32's three hardware serial peripherals. UART0 is reserved for USB/debug (`Serial`). Using a hardware UART ensures GPS characters are buffered in hardware even while the CPU is executing other code.

`TinyGPSPlus gps` creates the parser object that accumulates NMEA characters and updates its internal state when a complete, valid sentence is received.

---

### setup()

```cpp
Serial.begin(115200);    // USB serial — for debug output and bridge reading
while (!Serial);         // Wait for Serial to be ready
neogps.begin(9600, SERIAL_8N1, RX, TX);  // GPS UART at 9600 baud
```

> **Blocking `while (!Serial)` in Standalone Deployment**
>
> `while (!Serial)` blocks indefinitely if no USB host is connected. When the ESP32 is powered from a battery on the rover without a laptop attached, the code will never proceed past this line. Fix:
> ```cpp
> unsigned long t = millis();
> while (!Serial && millis() - t < 3000);  // Wait max 3 seconds
> ```
{: .prompt-warning }

---

### loop() — GPS Section

```cpp
while (neogps.available()) {
    char c = neogps.read();
    Serial.write(c);     // Echo raw NMEA to USB Serial
    gps.encode(c);
}

if (gps.location.isUpdated()) {
    Serial.print("LAT: ");
    Serial.println(gps.location.lat(), 6);
    Serial.print("LNG: ");
    Serial.println(gps.location.lng(), 6);
}
```

Every character from the GPS module is:
- Written raw to USB Serial (the bridge can see the raw NMEA stream, useful for debugging)
- Fed into `gps.encode()` — TinyGPSPlus accumulates characters, validates checksums, and updates its fields when a complete sentence is received

`gps.location.isUpdated()` returns `true` only when a new valid fix has been parsed. On cold start this takes 30–90 seconds outdoors. Indoors it may never trigger.

The `Serial.println(gps.location.lat(), 6)` output — for example `LAT: 12.576480` — is the exact format the Serial-to-UDP Bridge looks for when parsing output lines.

---

### loop() — MQ Sensor Section

```cpp
int mq4_value   = analogRead(MQ4_PIN);
int mq8_value   = analogRead(MQ8_PIN);
int mq135_value = analogRead(MQ135_PIN);

Serial.print("MQ-4 (CH4): ");   Serial.println(mq4_value);
Serial.print("MQ-8 (H2): ");    Serial.println(mq8_value);
Serial.print("MQ-135 (Air): "); Serial.println(mq135_value);
Serial.println("----------------------");

delay(2000);
```

`analogRead()` returns a 12-bit value (0–4095) proportional to the sensor output voltage. These are **raw ADC counts, not ppm**. Calibration against a known gas concentration is required to convert to ppm — see Known Limitations.

---

## Expected Serial Output

```
$GPGGA,123519.00,1234.56789,N,07654.32100,E,1,08,0.9,545.4,M,...
$GPRMC,123519.00,A,1234.56789,N,07654.32100,E,...

LAT: 12.576480
LNG: 76.905350
MQ-4 (CH4): 1023
MQ-8 (H2): 654
MQ-135 (Air): 2187
----------------------
```

If NMEA sentences appear but no LAT/LNG lines, the GPS module has not yet acquired a satellite fix.
If no NMEA sentences appear at all, check wiring and baud rate.

---

## How to Flash

1. Open the `.ino` file in Arduino IDE
2. **Tools → Board** → Select your ESP32 board (e.g. "ESP32 Dev Module")
3. **Tools → Port** → Select the correct COM / tty port
4. Install all libraries listed above
5. Click **Upload**
6. Open **Tools → Serial Monitor** at **115200 baud**

---

## Known Limitations

| Issue | Detail |
|---|---|
| MQ sensors output raw ADC, not ppm | Calibration (R0 measurement + datasheet Rs/R0 curve) is required for meaningful readings |
| BME680 referenced but not implemented | The include and Serial print mention BME680 but no BME680 code exists. Verify if hardware is present before implementing. |
| `while (!Serial)` blocks in standalone | See fix in setup() walkthrough above |
| `delay(2000)` risks GPS buffer overflow | At 9600 baud, ~240 bytes arrive during the 2s delay; hardware buffer is 256 bytes. Replace with `millis()`-based non-blocking timing for robustness. |
| No MQTT publishing | Output is Serial only. WiFi + MQTT integration is a future TODO. |
