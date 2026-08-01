---
title: "ESP32 Sensor Node"
date: 2026-03-01
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, esp32, sensors]
---

## Overview

This is the firmware running on the rover's environmental sensor node. It is written in Arduino-flavoured C++ and runs on an **ESP32 microcontroller**. The code does three things every 2 seconds:

1. Reads raw NMEA data from a **Neo GPS module** over a hardware UART, parses it using TinyGPSPlus, and prints latitude/longitude to Serial when a valid GPS fix is received.
2. Reads raw ADC values from three **MQ-series gas sensors** (MQ-4, MQ-8, MQ-135) via analog input pins.
3. Prints all sensor data to the Serial monitor at 115200 baud.

At the time of writing, this code outputs data only to Serial. It does **not** publish to MQTT or transmit over WiFi. Integration with the GCS data pipeline is a known TODO — see the [Known Limitations](#known-limitations-and-todos) section.

---

## Hardware

### Microcontroller — ESP32

The ESP32 is a dual-core 32-bit microcontroller with built-in WiFi and Bluetooth. It is used here because:

- It has multiple hardware UART peripherals (needed to run GPS on a separate serial port while keeping USB Serial free for debugging)
- Its ADC supports 12-bit resolution (0–4095), giving better granularity than Arduino Uno's 10-bit ADC
- It has sufficient processing power and memory to run GPS parsing, sensor reading, and eventually WiFi/MQTT simultaneously

### GPS Module — Neo GPS (NEO-6M / NEO-7M / NEO-8M compatible)

The GPS module communicates over **UART (serial)** using the NMEA 0183 standard — a text-based protocol where the module continuously streams sentences like `$GPGGA`, `$GPRMC`, etc., each containing different pieces of location and time data. The TinyGPSPlus library handles parsing this stream.

Default baud rate from the module: **9600**.

### Gas Sensors

All three MQ sensors follow the same basic operating principle: a heating element warms a metal oxide sensing element whose electrical resistance changes in the presence of target gases. This resistance change is converted to a voltage by a voltage divider circuit on the breakout board, which is then read by the ESP32's ADC.

| Sensor | Target Gas | Primary Use on Rover |
|---|---|---|
| MQ-4 | Methane (CH4), natural gas | Detection of combustible gases, cave/underground environments |
| MQ-8 | Hydrogen (H2) | Hydrogen gas detection |
| MQ-135 | CO2, ammonia, benzene, smoke — general air quality | Broad air quality index |

All three sensors require a **warm-up time of approximately 20–30 seconds** after power-on before readings stabilise. Do not trust readings taken immediately after boot.

---

## Pin Map

| Signal | ESP32 GPIO | Notes |
|---|---|---|
| GPS UART RX (ESP32 receives) | GPIO 22 | Connected to TX pin of the GPS module |
| GPS UART TX (ESP32 transmits) | GPIO 23 | Connected to RX pin of the GPS module |
| MQ-4 Analog Output | GPIO 33 | ADC1 channel — safe to use with WiFi |
| MQ-8 Analog Output | GPIO 34 | ADC1 channel — input only pin |
| MQ-135 Analog Output | GPIO 35 | ADC1 channel — input only pin |

> **ADC2 and WiFi Conflict**
>
> On the ESP32, **ADC2 pins (GPIO 0, 2, 4, 12–15, 25–27) cannot be used for analogRead() while WiFi is active.** The pins chosen here (33, 34, 35) all belong to ADC1 and are safe for use alongside WiFi. This was a deliberate choice and must be preserved when adding WiFi/MQTT functionality.
{: .prompt-warning }

> **GPIO 34 and 35 are Input-Only**
>
> GPIO 34 and 35 do not have internal pull-up or pull-down resistors and cannot be configured as outputs. This is fine since they are used only for analog input here.
{: .prompt-info }

---

## Software Dependencies

Install all libraries through the Arduino IDE Library Manager or PlatformIO before compiling.

| Library | Version Tested | Purpose | Source |
|---|---|---|---|
| `Wire.h` | Built-in | I2C communication (reserved for BME680 — see Known Limitations) | Arduino core |
| `Adafruit_Sensor.h` | 1.1.x | Unified sensor abstraction layer used by Adafruit drivers | Arduino Library Manager: "Adafruit Unified Sensor" |
| `TinyGPSPlus.h` | 1.0.3 | Parsing NMEA sentences from the GPS module | Arduino Library Manager: "TinyGPSPlus" by Mikal Hart |

---

## Code Walkthrough

### Pin and Object Definitions

```cpp
#define RX 22
#define TX 23

#define MQ4_PIN   33
#define MQ8_PIN   34
#define MQ135_PIN 35

HardwareSerial neogps(1);
TinyGPSPlus gps;
```

`HardwareSerial neogps(1)` creates a serial port object using the ESP32's **UART1** peripheral. The ESP32 has three hardware UARTs:

- UART0 — used by `Serial` (USB/debug)
- UART1 — used here for GPS (`neogps`)
- UART2 — available for future use

Using a hardware UART (as opposed to SoftwareSerial) is important because it offloads serial reception to dedicated hardware, meaning GPS characters are buffered even while the CPU is doing other work.

`TinyGPSPlus gps` creates the parser object that will process the NMEA character stream.

---

### setup()

```cpp
void setup() {
  Serial.begin(115200);
  while (!Serial);
  Serial.println("BME680 + MQ Sensors + GPS");
  neogps.begin(9600, SERIAL_8N1, RX, TX);
}
```

`Serial.begin(115200)` initialises the USB serial port for debug output. 115200 baud must be matched in your Serial monitor.

`while (!Serial)` waits until the Serial port is ready. On some ESP32 boards this resolves immediately; on others (particularly when powered via USB with a slow host) it can take a moment. **If the ESP32 is deployed without a USB connection (e.g. powered by LiPo on the rover), this line will block forever and the code will never proceed.** This is a known bug — see Known Limitations.

`neogps.begin(9600, SERIAL_8N1, RX, TX)` initialises UART1 at 9600 baud, 8 data bits, no parity, 1 stop bit, on GPIO 22 (RX) and GPIO 23 (TX). `SERIAL_8N1` is the standard UART framing used by virtually all GPS modules.

---

### loop() — GPS Section

```cpp
while (neogps.available()) {
    char c = neogps.read();
    Serial.write(c);   // print raw NMEA data
    gps.encode(c);
}

if (gps.location.isUpdated()) {
    Serial.println();
    Serial.print("LAT: ");
    Serial.println(gps.location.lat(), 6);
    Serial.print("LNG: ");
    Serial.println(gps.location.lng(), 6);
}
```

The GPS module continuously streams NMEA sentences over serial. The `while (neogps.available())` loop drains the hardware receive buffer one character at a time.

Each character is:
1. Echoed raw to the USB Serial monitor via `Serial.write(c)` — this lets you see the raw NMEA stream and verify the GPS module is communicating.
2. Fed into `gps.encode(c)` — TinyGPSPlus accumulates characters internally, detects sentence boundaries (`$` start, `\r\n` end), validates checksums, and updates its internal state fields.

`gps.location.isUpdated()` returns `true` when TinyGPSPlus has successfully parsed a new valid fix from a `$GPGGA` or `$GPRMC` sentence. It returns `false` if:
- No sentence has been received yet
- The last sentence had an invalid checksum
- The GPS module does not yet have a satellite fix (common for 30–90 seconds after cold power-on outdoors)

`gps.location.lat()` and `gps.location.lng()` return `double` values in decimal degrees. The `, 6` argument to `println` specifies 6 decimal places, which gives approximately 10 cm precision — appropriate for GPS.

**GPS Cold Start Behaviour:** After power-on, the GPS module will stream sentences with empty or invalid location fields until it acquires a satellite fix. During this time `isUpdated()` will not trigger. This is normal. Outdoors with clear sky view, fix acquisition typically takes 30–90 seconds. Indoors, a fix may never be acquired.

---

### loop() — MQ Sensor Section

```cpp
int mq4_value   = analogRead(MQ4_PIN);
int mq8_value   = analogRead(MQ8_PIN);
int mq135_value = analogRead(MQ135_PIN);

Serial.print("MQ-4 (CH4): ");
Serial.println(mq4_value);
Serial.print("MQ-8 (H2): ");
Serial.println(mq8_value);
Serial.print("MQ-135 (Air): ");
Serial.println(mq135_value);
Serial.println("----------------------");

delay(2000);
```

`analogRead()` on the ESP32 returns a 12-bit integer between 0 and 4095, proportional to the voltage on the pin (0V = 0, 3.3V = 4095).

The MQ sensor breakout boards include a load resistor that forms a voltage divider with the sensing element. As gas concentration increases, the sensor resistance decreases, and the output voltage (and therefore ADC count) changes accordingly.

**These values are raw ADC counts. They are not in ppm (parts per million).** Converting to ppm requires sensor calibration — see Known Limitations.

`delay(2000)` pauses the entire loop for 2 seconds. During this time, incoming GPS characters accumulate in the hardware UART buffer (which is 256 bytes by default). If the buffer fills before the loop resumes, older NMEA characters will be silently dropped. At 9600 baud and 2 seconds, approximately 240 bytes arrive — close to the buffer limit. This is another known issue.

---

## Expected Serial Output

When working correctly with a GPS fix acquired, the Serial monitor (115200 baud) should show:

```
BME680 + MQ Sensors + GPS
$GPGGA,123519.00,1234.56789,N,07654.32100,E,1,08,0.9,545.4,M,46.9,M,,*4F
$GPGSA,A,3,04,05,09,12,24,36,,,,,,,1.8,0.9,1.5*33
$GPRMC,123519.00,A,1234.56789,N,07654.32100,E,0.00,0.00,130525,,,A*6A

LAT: 12.576480
LNG: 76.905350
MQ-4 (CH4): 1023
MQ-8 (H2): 654
MQ-135 (Air): 2187
----------------------
$GPGGA,...
```

If GPS has no fix yet, you will see NMEA sentences but no LAT/LNG lines. If no NMEA sentences appear at all, the GPS wiring or baud rate is incorrect.

---

## How to Flash

1. Open the `.ino` file in Arduino IDE.
2. Go to **Tools → Board** and select your ESP32 board (e.g. "ESP32 Dev Module").
3. Go to **Tools → Port** and select the correct COM/tty port.
4. Ensure all libraries listed in [Software Dependencies](#software-dependencies) are installed.
5. Click **Upload**.
6. Open **Tools → Serial Monitor**, set baud rate to **115200**.

---

## Known Limitations and TODOs

### 1. MQ Sensors Output Raw ADC — Not Calibrated to ppm

The biggest limitation. `analogRead()` returns a number between 0 and 4095. To convert this to a meaningful gas concentration in ppm, you need to:

- Measure the sensor's resistance in clean air (R0)
- Apply the sensor's characteristic curve from the datasheet (Rs/R0 ratio vs ppm, log-log scale)
- Implement the ppm conversion formula

Until this is done, the values can only be used for relative comparison (e.g. "value increased suddenly") rather than absolute measurement. The MQ sensor datasheets are the starting point — search for the Rs/R0 curve graphs in each datasheet.

### 2. BME680 is Imported but Not Used

The include statements reference `Adafruit_Sensor.h` and the Serial print says "BME680 + MQ Sensors + GPS", but there is no BME680 initialisation or reading in the code. The BME680 is a temperature, humidity, pressure, and VOC sensor. Either it was planned and not yet implemented, or it was previously included and later removed. The hardware may or may not be physically connected. Before adding BME680 support, verify whether the sensor is on the board and which I2C address it is configured to use (0x76 or 0x77).

### 3. `while (!Serial)` Blocks When No USB is Connected

If the ESP32 is powered without a USB serial connection (e.g. from a battery on the rover), `while (!Serial)` will block indefinitely and the code will never run. Fix by adding a timeout:

```cpp
// Replace:
while (!Serial);

// With:
unsigned long t = millis();
while (!Serial && millis() - t < 3000);  // Wait max 3 seconds
```

### 4. No MQTT Publishing

All data goes to Serial only. For integration with the GCS, the sensor readings need to be published over WiFi to the MQTT broker. The intended topic structure is:

```
rover/sensors/gps/lat
rover/sensors/gps/lng
rover/sensors/mq4
rover/sensors/mq8
rover/sensors/mq135
```

Adding WiFi and MQTT requires including `WiFi.h` and `PubSubClient.h` (or `AsyncMqttClient`), initialising WiFi in `setup()`, and replacing or supplementing the `Serial.print` calls with `client.publish()` calls. Note that WiFi initialisation may affect ADC2 pins — the current pin selection (ADC1 only) was made with this in mind.

### 5. `delay(2000)` Blocks the Loop — GPS Buffer Risk

The 2-second blocking delay means GPS characters are not read for 2 seconds each cycle. The ESP32 UART1 hardware buffer is 256 bytes. At 9600 baud, approximately 240 bytes arrive in 2 seconds — this is close to overflowing the buffer. If NMEA sentences are being dropped, replace `delay(2000)` with a non-blocking timing approach:

```cpp
// At the top of loop():
static unsigned long lastPrint = 0;
if (millis() - lastPrint >= 2000) {
    lastPrint = millis();
    // --- print sensor values here ---
}
// GPS reading runs every loop iteration without blocking
```

### 6. No Error Handling for GPS Timeout

If the GPS module stops responding (cable fault, power issue), the code silently continues printing MQ values with no indication that GPS data is stale. A `gps.charsProcessed()` check can detect this:

```cpp
if (millis() > 10000 && gps.charsProcessed() < 10) {
    Serial.println("WARNING: No GPS data received. Check wiring.");
}
```

---

## Sensor Datasheet References

| Sensor | Datasheet |
|---|---|
| MQ-4 | [Hanwei MQ-4 Datasheet](https://www.sparkfun.com/datasheets/Sensors/Biometric/MQ-4.pdf) |
| MQ-8 | [Hanwei MQ-8 Datasheet](https://cdn.sparkfun.com/datasheets/Sensors/Biometric/MQ-8.pdf) |
| MQ-135 | [Hanwei MQ-135 Datasheet](https://www.olimex.com/Products/Components/Sensors/SNS-MQ135/resources/SNS-MQ135.pdf) |
| TinyGPSPlus | [GitHub — mikalhart/TinyGPSPlus](https://github.com/mikalhart/TinyGPSPlus) |
| ESP32 Datasheet | [Espressif ESP32 Datasheet](https://www.espressif.com/sites/default/files/documentation/esp32_datasheet_en.pdf) |
