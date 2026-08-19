---
title: "Stereo Module"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, stereo_module]
---

# Stereo Module

The Stereo Module contains the primary depth camera settings.

## Parameters

- Resolution
- FPS
- Laser Power
- Emitter
- Emitter Frequency
- Exposure and Gain
- Auto Exposure
- Auto Exposure Limit
- Error Polling
- Depth Units
- Stereo Baseline
- Inter Camera Sync Mode
- Global Time Enabled
- Thermal Compensation
- HDR

## Parameter documentation

- [Resolution](Resolution.md)
- [FPS](FPS.md)
- [Laser Power](Laser_Power.md)
- [Emitter](Emitter.md)
- [Emitter Frequency](Emitter_Frequency.md)
- [Exposure and Gain](Exposure_and_Gain.md)
- [Auto Exposure](Auto_Exposure.md)
- [Auto Exposure Limits](Auto_Exposure_Limits.md)
- [Error Polling](Error_Polling.md)
- [Depth Units](Depth_Units.md)
- [Stereo Baseline](Stereo_Baseline.md)
- [Inter-Camera Sync](Inter_Camera_Sync.md)
- [Global Time](Global_Time.md)
- [Thermal Compensation](Thermal_Compensation.md)
- [HDR](HDR.md)

## Recommended rover profile

Start with 848 × 480 depth at 30 FPS, Auto Exposure enabled, and low gain. Indoors, enable the emitter and use laser power appropriate for the scene (the guide suggests 150–360). Enable thermal compensation for long runs and Global Time when host-aligned timestamps matter.

Return to [RealSense Viewer Parameters](../README.md).
