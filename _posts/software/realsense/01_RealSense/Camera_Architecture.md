---
title: "Camera Architecture"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 01_realsense]
---

# Camera Architecture

The Intel RealSense D455 is an active stereoscopic depth camera.

It calculates depth by comparing the simultaneously captured left and right video images to determine the pixel shift (disparity) between object points.

The onboard Intel RealSense Vision Processor D4 (ASIC) executes a projected texture stereo matching algorithm based on the Census transform and Hamming distance.

The D455 has an extended baseline of approximately **95 millimeters**, improving depth accuracy to **less than 2% error at 4 meters**.

## Depth pipeline

The D4 ASIC matches corresponding pixels from the synchronized left and right infrared images. Its projected-texture stereo matching uses a Census transform and Hamming-distance comparison. The resulting disparity is converted to depth through the calibrated camera geometry.

The extended D455 baseline is approximately 95 mm. A longer baseline improves the measurable disparity for a given scene depth, which is why it helps depth accuracy at rover-relevant ranges.

## Rover implication

Depth is strongest when the cameras see stable, textured surfaces. Configure resolution, frame rate, emitter power, exposure, and filtering together: each affects either the input image quality or the amount of depth data that reaches downstream ROS nodes.

Related topics: [IR Projector](IR_Projector.md), [Coordinate Systems](Coordinate_Systems.md), and [RGB Camera and IMU](RGB_and_IMU.md).
