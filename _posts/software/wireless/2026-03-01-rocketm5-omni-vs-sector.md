---
title: "RocketM5 — Omni vs Sector"
date: 2026-03-01
author: mahi
categories: [software, wireless]
tags: [gcs, wireless, rocketm5, antenna]
---

## The Rule

| Location | Antenna | Reason |
|---|---|---|
| Rover | Omnidirectional | Rover heading changes constantly — 360° coverage ensures the link holds regardless of direction |
| Base station (GCS) | Sector | Base station is fixed — higher gain in the direction of the course improves link quality and range |

This is not a preference — it is the only configuration that works reliably in practice. Swapping them (sector on rover, omni at base) will result in the link dropping every time the rover turns away from the base station.

---

## Why Not Omni at Both Ends

An omnidirectional antenna at the base station would work but sacrifices gain. Gain is not free — a sector antenna achieves its higher gain by concentrating energy into a narrower angle. At the base station, that concentrated beam can be permanently aimed at the course. The rover benefits from the extra dB without giving anything up.

An omni at the base radiates energy in all directions equally, including directions where there is no rover and no useful link to maintain. That energy is wasted, and the link in the direction of the rover is weaker than it could be.

---

## Why Not Sector at Both Ends

A sector antenna on the rover would require the rover to always face the base station to maintain the link — impractical for an autonomously navigating or remotely driven rover. The link would drop the moment the rover turned.

---

## Gain Tradeoff at a Glance

Assume:
- Rover omni gain: 6 dBi
- Base sector gain: 15 dBi
- Base omni gain (alternative): 6 dBi

With sector at base: effective gain contribution = 6 + 15 = 21 dBi total antenna system gain

With omni at base: effective gain contribution = 6 + 6 = 12 dBi total antenna system gain

The sector gives 9 dB more system gain — roughly doubling the effective range, or allowing the same range with significantly better link margin.

---

## Sector Antenna Aim

The sector antenna must be aimed at the competition course before the run. Use the AirOS Alignment Tool (Tools tab) for this — it provides a live RSSI readout and audio feedback.

Aim for the centre of the area the rover will operate in. If the course spans a wide area, aim at the middle rather than one end. The 120° beamwidth of a typical sector antenna provides sufficient coverage across most rover competition layouts without needing to pan the antenna during the run.

If the course layout is known in advance, sketch the angular coverage needed and verify the sector antenna's beamwidth is sufficient. If the rover will operate outside the sector's coverage angle, signal will drop sharply at the edges.
