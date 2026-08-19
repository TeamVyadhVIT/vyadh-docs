---
title: "RGB Camera and IMU"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 01_realsense]
---

# RGB Camera and IMU

The D455 uses matched global shutter sensors for both RGB and depth.

Global shutter sensors prevent rolling shutter artifacts (image tearing) when the rover turns rapidly.

The camera also features a Bosch BMI055 Inertial Measurement Unit (IMU).

The IMU provides:

- High-frequency angular velocity (gyroscope)
- Linear acceleration data

## RGB camera

The D455 uses a global-shutter RGB sensor. Global shutter reduces image tearing during rapid rover turns compared with a rolling-shutter capture, which helps visual debugging and colored point-cloud alignment.

## IMU

The onboard Bosch BMI055 IMU publishes angular velocity from the gyroscope and linear acceleration from the accelerometer. The guide uses typical rates of 200 Hz for gyro and 100 Hz for accelerometer. These signals are useful for VIO and robot-localization pipelines when their timestamps and frames are handled consistently.

Related topics: [Coordinate Systems](Coordinate_Systems.md), [RGB Settings](../02_RealSense_Viewer/Motion_and_RGB/RGB_Settings.md), [IMU Settings](../02_RealSense_Viewer/Motion_and_RGB/IMU_Settings.md), and [Point Clouds](../03_PointCloud/README.md).
