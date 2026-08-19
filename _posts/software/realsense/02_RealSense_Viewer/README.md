---
title: "RealSense Viewer Parameters"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer]
---

# RealSense Viewer Parameters

This folder contains the parameters available in the Intel RealSense Viewer.

The parameters are grouped into:

- Stereo Module
- Advanced Mode
- Post Processing
- Motion Module and RGB Camera

Each parameter is explained in its own markdown file.

## Parameter groups

- [Stereo Module](Stereo_Module/README.md): depth stream, exposure, emitter, timing, and thermal settings
- [Advanced Mode](Advanced_Mode/README.md): D4 ASIC stereo-matching controls
- [Post Processing](Post_Processing/README.md): depth filtering and visualization controls
- [Motion Module and RGB Camera](Motion_and_RGB/README.md): RGB and IMU settings

## Configuration strategy

Tune the stream resolution/FPS first, then emitter and exposure, and finally post-processing. This order makes it easier to distinguish weak stereo input from filtering side effects. For a rover, 848 × 480 depth at 30 FPS is a useful starting point; lower resolution or decimation can reduce CPU and USB load.
