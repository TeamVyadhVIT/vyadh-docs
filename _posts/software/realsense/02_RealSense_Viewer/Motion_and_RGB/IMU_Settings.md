---
title: "Accelerometer and Gyroscope Settings"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, motion_and_rgb]
---

# Accelerometer and Gyroscope Settings

## Output Format

Defines the IMU output format.

Example:

- MOTION_XYZ32F

---

## Frame Rate

Typical frame rates:

- Gyroscope: 200 Hz
- Accelerometer: 100 Hz

ROS 2 node parameters synchronize these streams.

## Enabling the ROS stream

For a unified IMU stream, enable gyro and accelerometer in the driver and choose a supported `unite_imu_method` (the guide cites `1` or `2`). Check the published rate and frame before feeding the data into VIO or robot-localization software.
