---
title: "RF Fundamentals"
date: 2026-03-01
author: mahi
categories: [software, wireless]
tags: [gcs, wireless, rf]
---

## Frequency Bands

The two frequency bands used in the team's wireless setup are 2.4 GHz and 5 GHz.

| Property | 2.4 GHz | 5 GHz |
|---|---|---|
| Max Speed | ~50 Mbps | ~1300 Mbps |
| Range | Larger area | Smaller area |
| Obstacle Penetration | High — passes through walls and solid objects | Lower — more easily blocked |
| Interference | High — fewer non-overlapping channels, heavily congested | Lower — more channels available, less congestion |
| Typical Use Cases | IoT devices, long-range low-bandwidth links | HD video streaming, high-throughput links |

The team uses **5 GHz** via the RocketM5 for the primary rover link. The higher throughput supports video streaming, and the less congested spectrum is advantageous in competition environments where multiple teams operate on 2.4 GHz simultaneously.

---

## Channel Width

Channel width defines the amount of frequency spectrum a wireless signal occupies during transmission. A wider channel carries more data but is more susceptible to interference and has shorter effective range.

Think of it as a road — a wider road carries more traffic but is harder to keep clear of congestion.

| Channel Width | Throughput | Range | Stability | Interference Risk |
|---|---|---|---|---|
| 10 MHz | Low | Long | Very High | Very Low |
| 20 MHz | Medium | Medium | High | Medium |
| 40 MHz | High | Shorter | Lower | High |

For competition use, **20 MHz** offers the best balance between throughput and stability. Use 10 MHz if the RF environment is particularly noisy and link stability is the priority.

---

## Channel Selection

A channel is a specific frequency sub-band on which a wireless device transmits and receives. For example, channel 149 corresponds to 5745 MHz in the 5 GHz band.

Selecting the wrong channel causes two types of interference:

- **Co-channel interference** — two devices on the exact same channel collide directly. Signal quality collapses.
- **Adjacent-channel interference** — devices on nearby channels partially overlap, reducing throughput and increasing errors.

Use the AirOS frequency scan tool before deployment to identify which channels are in use. Choose the channel with the lowest occupancy and the most separation from other active channels.

---

## Signal Quality Metrics

### RSSI — Received Signal Strength Indicator

RSSI measures the received signal power in dBm. Higher (less negative) values mean a stronger signal.

| RSSI (dBm) | Quality |
|---|---|
| -50 to -60 | Excellent |
| -60 to -70 | Good |
| -70 to -80 | Fair |
| Below -80 | Poor — link may be unreliable |

### Noise Floor

The noise floor is the level of background radio energy present in the environment, measured in dBm. It is the sum of all unwanted signals — interference from other devices, thermal noise, and environmental RF. A lower noise floor means a cleaner RF environment.

### SNR — Signal-to-Noise Ratio

SNR is the difference between received signal strength and the noise floor, expressed in dB.

```
SNR (dB) = RSSI (dBm) - Noise Floor (dBm)
```

A high SNR indicates a clean, usable link. SNR above 20 dB is generally good for reliable communication.

---

## Link Budget

The link budget is a calculation of the total signal power available at the receiver after accounting for all gains and losses from transmitter to receiver.

```
Received Power (dBm) = TX Power (dBm) + TX Antenna Gain (dBi) + RX Antenna Gain (dBi) - Path Loss (dB)
```

A positive link budget means the received signal is above the receiver's sensitivity threshold — the link will work. A margin of 10–20 dB above the minimum threshold ensures the link holds up under adverse conditions.

---

## Line of Sight (LOS)

Line of sight is a direct, unobstructed path between transmitter and receiver antennas. At 5 GHz, obstacles — trees, terrain, structures — cause significant signal degradation. Competition courses often require careful base station positioning to maintain LOS with the rover throughout the run.

### Fresnel Zone

The Fresnel zone is an elliptical region around the direct LOS path that must remain largely unobstructed for optimal signal propagation. Objects within the first Fresnel zone reduce signal strength even if the direct line appears clear. As a practical rule, keep the first Fresnel zone at least 60% clear.

---

## Antenna Gain

Antenna gain (measured in dBi — decibels relative to an isotropic radiator) describes how effectively an antenna focuses radio energy in a specific direction compared to a theoretical antenna that radiates equally in all directions.

A higher gain antenna concentrates energy into a narrower beam, increasing range in the intended direction while reducing coverage elsewhere. Antenna type selection depends on whether coverage area or link range and directionality is the priority.

See [Antenna Types]({{ site.baseurl }}/posts/antenna-types/) for the specific antennas used by the team.
