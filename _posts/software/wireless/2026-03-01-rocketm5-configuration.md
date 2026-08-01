---
title: "RocketM5 — Configuration"
date: 2026-03-01
author: mahi
categories: [software, wireless]
tags: [gcs, wireless, rocketm5]
---

## Operating Modes

The RocketM5 supports several operating modes set under the **Wireless** tab.

| Mode | Description | Team Usage |
|---|---|---|
| Access Point (AP) | Creates a wireless network. Other devices connect to it. | Base station — GCS side |
| Station | Connects to an existing wireless network as a client. | Rover side |
| AP-WDS | Access Point with Wireless Distribution System — transparent bridge | Not currently used |
| Station-WDS | Station with WDS — transparent bridge to a WDS AP | Not currently used |

The base station RocketM5 is always **Access Point**. The rover RocketM5 is always **Station**, connecting to the base station's SSID.

---

## airMAX

airMAX is Ubiquiti's proprietary TDMA (Time Division Multiple Access) MAC layer protocol. Standard WiFi uses CSMA/CA (collision avoidance), where every device listens before transmitting and backs off randomly on collision. In a busy RF environment this is inefficient.

airMAX replaces this with scheduled time slots assigned by the AP to each client, eliminating collisions entirely.

**Benefits for rover use:**
- Significantly reduced and more consistent latency
- Higher throughput in environments with other 2.4/5 GHz traffic
- Better performance with a single client (rover) connecting to the base AP

**Requirement:** airMAX must be enabled on **both** the AP and the Station. If only one side has it enabled, the link will either not form or perform poorly.

Enable airMAX under the **Advanced** tab on both units.

---

## Distance and ACK Timeout

The 802.11 standard uses an ACK (acknowledgement) timeout that assumes very short distances — typically indoor use. For links over a few hundred metres, the default timeout is too short. The AP sends a data frame, the Station's ACK does not arrive in time, the AP assumes the frame was lost and retransmits. This causes significant throughput degradation at distance.

AirOS handles this automatically — set the **Distance** parameter in the Advanced tab to match the expected maximum operating distance (in metres or kilometres). AirOS calculates and sets the correct ACK timeout.

If the rover will operate up to 300 m from the base station, set distance to 300 m.

---

## Frequency Scan

Before every deployment, run a frequency scan to identify the RF environment:

1. Go to **Tools → Frequency Scan**
2. Click **Start**
3. Wait for the scan to complete across all 5 GHz channels
4. Identify the channel with the lowest signal occupancy and noise
5. Set that channel under **Wireless → Channel**

Do not skip this step at competitions. Other teams will be on air and channel selection directly affects link reliability.

---

## Antenna Alignment Tool

The alignment tool provides a live RSSI readout and audio feedback tone that rises in pitch as RSSI improves. Use it to physically aim the sector antenna at the course during setup:

1. Go to **Tools → Alignment**
2. Have one person watch the RSSI value or listen to the audio tone
3. Have another person physically rotate and tilt the antenna mount
4. Stop when RSSI is maximised and CCQ is above 90%

---

## Field Configuration Checklist

Complete this in order before every competition run.

- [ ] Assign static IP addresses to both RocketM5 units — document in the team IP map
- [ ] Set base unit to **Access Point** mode, rover unit to **Station** mode
- [ ] Match SSID and password on both units
- [ ] Enable **airMAX** on both units (Advanced tab)
- [ ] Run **Frequency Scan** — select the cleanest channel (Tools tab)
- [ ] Set **Channel Width** to 20 MHz (Wireless tab)
- [ ] Set **Distance** to expected maximum range (Advanced tab)
- [ ] Set TX power within legal and team-agreed limits (Wireless tab)
- [ ] Use **Alignment Tool** to aim the sector antenna (Tools tab)
- [ ] Verify link on **Main** tab — confirm RSSI, SNR, and CCQ are in acceptable range
- [ ] Ping from GCS to rover IP to confirm end-to-end network connectivity

---

## Troubleshooting

| Symptom | Likely Cause | Action |
|---|---|---|
| No link established | SSID or password mismatch, mode mismatch, airMAX mismatch | Verify Wireless tab settings on both units |
| Low RSSI | Antenna misalignment, obstruction, distance | Re-aim sector antenna, check LOS |
| High latency or packet loss | Co-channel interference, low SNR | Run frequency scan, switch to a cleaner channel |
| Link unstable when rover moves | Rover omni antenna blocked by rover chassis | Check antenna mounting position and orientation |
| Cannot access AirOS at 192.168.1.20 | IP has been changed | Use Ubiquiti Device Discovery Tool to find current IP |
| AirOS login fails | Password changed from default | Try team-standard password; factory reset as last resort (hold reset button 10 s) |
| Good RSSI but low CCQ | RF interference from other devices on same channel | Run frequency scan and change channel |
