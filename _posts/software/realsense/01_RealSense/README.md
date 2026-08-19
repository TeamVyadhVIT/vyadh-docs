---
title: "Intel RealSense D455"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 01_realsense]
---

# Intel RealSense D455

This folder introduces the Intel RealSense D455 hardware.

## Topics

- Camera Architecture
- RGB Camera and IMU
- IR Projector
- Coordinate Systems

Open a topic:

- [Camera Architecture](Camera_Architecture.md)
- [RGB Camera and IMU](RGB_and_IMU.md)
- [IR Projector](IR_Projector.md)
- [Coordinate Systems](Coordinate_Systems.md)

## Files

| File | Description |
|------|-------------|
| Camera_Architecture.md | Stereo depth camera architecture |
| RGB_and_IMU.md | RGB camera and IMU |
| IR_Projector.md | Infrared projector |
| Coordinate_Systems.md | Intrinsics, Extrinsics and Coordinate System |

## D455 operating model

The D455 is an active stereo camera: the two infrared imagers observe the same scene while the projector adds texture when natural features are insufficient. The D4 vision processor computes disparity and derives depth. RGB and IMU data complement that depth stream for colored point clouds, visual debugging, and motion estimation.

The D455's approximately 95 mm stereo baseline improves depth accuracy; the guide quotes less than 2% error at 4 m. Depth quality still depends on scene texture, lighting, camera motion, range, and filter configuration.

Next: [RealSense Viewer Parameters](../02_RealSense_Viewer/README.md) for stream, stereo, and processing controls.
