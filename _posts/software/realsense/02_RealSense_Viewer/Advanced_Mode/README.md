---
title: "Advanced Mode"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, advanced_mode]
---

# Advanced Mode

Advanced Mode provides access to the raw mathematical controls used by the Intel RealSense Vision Processor.

## Parameters

- Depth Control
- HDAD (Ignore SAD)
- Rau Support Vector Control
- Rau / SLO Color Thresholds
- Disparity Modulation
- Census Enable Register

These parameters affect stereo matching, confidence, smoothing, edge preservation, and disparity calculations.

## Parameter documentation

- [Depth Control](Depth_Control.md)
- [HDAD (Ignore SAD)](HDAD.md)
- [Rau and SLO Control](Rau_and_SLO_Control.md)
- [Color Thresholds](Color_Thresholds.md)
- [Disparity Modulation](Disparity_Modulation.md)
- [Census Register](Census_Register.md)

## Caution and purpose

Advanced Mode exposes D4 ASIC registers that influence the stereo matching cost function. Change one group at a time and record the preset before/after testing: aggressive settings can trade dense depth for confidence, or vice versa. These controls are best used after core stream and lighting settings are stable.

Return to [RealSense Viewer Parameters](../README.md).
