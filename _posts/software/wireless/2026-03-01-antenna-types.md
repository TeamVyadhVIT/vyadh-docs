---
title: "Antenna Types"
date: 2026-03-01
author: mahi
categories: [software, wireless]
tags: [gcs, wireless, antenna]
---

## Overview

The team uses two antenna types — omnidirectional at the rover end and sector at the base station. This page explains what each antenna does, its characteristics, and why the team uses this combination.

For the decision of which antenna to mount where on the RocketM5, see [Omni vs Sector]({{ site.baseurl }}/posts/rocketm5-omni-vs-sector/).

---

## Omnidirectional Antenna

An omnidirectional antenna radiates signal equally in all horizontal directions (360°), with a relatively flat vertical radiation pattern.

**Characteristics:**

| Property | Value |
|---|---|
| Horizontal coverage | 360° |
| Typical gain | 5–9 dBi |
| Directionality | None — equal in all directions |
| Best for | Mobile platforms, unknown heading |

**Used on:** The rover.

The rover's heading relative to the base station changes constantly as it drives. An omnidirectional antenna ensures the link is maintained regardless of which direction the rover is facing or travelling. A directional antenna on the rover would cause the link to drop every time the rover turned away from the base station.

---

## Sector Antenna

A sector antenna covers a defined angular sector — typically 60°, 90°, or 120° — with higher gain than an omni within that sector.

**Characteristics:**

| Property | Value |
|---|---|
| Horizontal coverage | 60°, 90°, or 120° depending on model |
| Typical gain | 13–17 dBi |
| Directionality | High — focused beam |
| Best for | Fixed base station with known operating area |

**Used on:** The base station (GCS side).

The base station is fixed in place and the competition course is in a known direction. A sector antenna aimed at the course provides significantly more gain than an omni, extending effective range and improving link quality. It also reduces interference from directions outside the beam — useful in crowded competition RF environments.

---

## Comparison

| Property | Omnidirectional | Sector |
|---|---|---|
| Coverage | 360° | 60–120° |
| Gain | Low (5–9 dBi) | High (13–17 dBi) |
| Must be aimed | No | Yes |
| Mobility | Suitable | Fixed only |
| Interference rejection | None | Rejects signals outside beam |

---

## Physical Mounting Notes

**Rover omni antenna:**
- Mount vertically and as high on the rover body as possible
- Avoid mounting directly behind metal structures — the rover chassis can block signal in certain directions
- Ensure the antenna has a clear line of sight to the base station in all likely rover orientations

**Base station sector antenna:**
- Mount on a tripod or elevated support
- Aim the centre of the beam at the middle of the expected rover operating area
- Tilt slightly downward if the rover will be operating on terrain significantly below the antenna height
- Use the AirOS alignment tool (live RSSI + audio feedback) to fine-tune aim during setup
