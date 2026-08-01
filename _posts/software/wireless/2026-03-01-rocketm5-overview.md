---
title: "RocketM5 — Overview"
date: 2026-03-01
author: mahi
categories: [software, wireless]
tags: [gcs, wireless, rocketm5]
---

## What is the RocketM5

The RocketM5 is a 5 GHz 802.11n radio platform manufactured by Ubiquiti Networks. It is **not a self-contained access point with a built-in antenna** — it is a radio module that connects to a separate external antenna via an N-type connector. The antenna is selected separately based on the use case (omnidirectional for the rover, sector for the base station).

The device runs **AirOS**, Ubiquiti's embedded firmware, which exposes a web-based configuration interface accessible over Ethernet.

The team uses two RocketM5 units:

| Unit | Location | Antenna | AirOS Mode |
|---|---|---|---|
| Unit 1 | Base station (GCS side) | Sector antenna | Access Point |
| Unit 2 | Rover | Omnidirectional antenna | Station |

---

## Accessing the AirOS Interface

1. Connect a laptop to the RocketM5 via Ethernet
2. Open a browser and navigate to:

```
http://192.168.1.20
```

Default credentials:

```
Username: ubnt
Password: ubnt
```

> Change the default password after first setup. If the radio is ever connected to a network accessible by others, the default credentials are publicly known.
{: .prompt-warning }

If the IP has been changed and you do not know the current address, use the **Ubiquiti Device Discovery Tool** (available at ui.com) to find it on the network.

---

## AirOS Interface — Key Tabs

| Tab | What it Contains |
|---|---|
| **Main** | Live link status — RSSI, SNR, CCQ (Connection Quality), TX/RX rate |
| **Wireless** | Operating mode, SSID, channel, channel width, TX power, security |
| **Network** | IP address configuration — static or DHCP, bridge or router mode |
| **Advanced** | airMAX enable/disable, distance setting, ACK timeout |
| **Tools** | Frequency scan, antenna alignment tool (live RSSI + audio), ping, traceroute |

During field setup, the **Main** tab and **Tools** tab are the ones used most frequently — the Main tab to monitor link quality, and Tools to run a frequency scan and alignment before the run.

---

## CCQ — Connection Quality

CCQ (Client Connection Quality) is a Ubiquiti-specific metric shown on the Main tab. It represents the efficiency of the wireless link as a percentage.

| CCQ | Interpretation |
|---|---|
| 90–100% | Excellent link |
| 70–89% | Good — minor retransmissions |
| 50–69% | Fair — noticeable degradation |
| Below 50% | Poor — link is unreliable |

Monitor CCQ alongside RSSI and SNR. A high RSSI with low CCQ suggests interference rather than a distance or alignment problem.
