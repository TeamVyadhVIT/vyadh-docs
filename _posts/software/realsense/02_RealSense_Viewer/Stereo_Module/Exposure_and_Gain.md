---
title: "Exposure and Gain"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, stereo_module]
---

# Exposure and Gain

## Exposure

Defines how long the shutter remains open.

Measured in microseconds.

## Gain

Digital amplification of the sensor signal.

## Recommendation

Keep gain as low as possible (e.g., 16) to prevent point cloud spatial noise.

## Noise versus motion blur

Exposure is the shutter-open time in microseconds; gain digitally amplifies the sensor signal. Long exposure can blur a moving rover scene, while excessive gain increases spatial noise in the point cloud. Keep gain as low as practical—the guide gives 16 as an example—and use Auto Exposure for changing illumination.
