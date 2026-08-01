---
title: "Wireless Communication"
date: 2026-03-01
author: mahi
categories: [software, wireless]
tags: [gcs, wireless, rf]
---

## Overview

This section covers the wireless link between the rover and the Ground Control Station — from fundamental RF concepts through to the specific hardware and configuration used by the team.

---

## How the Wireless Link Works

The rover and base station communicate over a **5 GHz 802.11n point-to-point wireless link** using two Ubiquiti RocketM5 radios. The base station radio acts as an Access Point; the rover radio connects to it as a Station. All network traffic between the rover and GCS — MQTT telemetry, video stream, commands — travels over this single wireless link.

```
[Rover] ── RocketM5 (Station) ──── 5 GHz ──── RocketM5 (AP) ── [GCS Laptop]
              Omni Antenna                      Sector Antenna
```

---

## Pages in This Section

### RF Fundamentals
Core theory — frequency bands, channel width, RSSI, SNR, link budget, line of sight, and antenna gain. Read this first if you are new to wireless.

→ [RF Fundamentals]({{ site.baseurl }}/posts/rf-fundamentals/)

### Antenna Types
Omnidirectional vs sector antennas — what they are, their characteristics, and physical mounting notes.

→ [Antenna Types]({{ site.baseurl }}/posts/antenna-types/)

### RocketM5 — Overview
Hardware overview, accessing the AirOS interface, and understanding the key tabs and metrics.

→ [RocketM5 Overview]({{ site.baseurl }}/posts/rocketm5-overview/)

### RocketM5 — Configuration
Operating modes, airMAX, distance/ACK timeout, frequency scanning, alignment, field checklist, and troubleshooting.

→ [RocketM5 Configuration]({{ site.baseurl }}/posts/rocketm5-configuration/)

### RocketM5 — Omni vs Sector
Which antenna goes where and why — the gain tradeoff explained.

→ [Omni vs Sector]({{ site.baseurl }}/posts/rocketm5-omni-vs-sector/)
