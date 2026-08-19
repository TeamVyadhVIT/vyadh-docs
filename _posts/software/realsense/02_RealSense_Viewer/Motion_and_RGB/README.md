---
title: "Motion Module and RGB Camera"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, motion_and_rgb]
---

# Motion Module and RGB Camera

This section contains the RGB camera and IMU settings.

## RGB Camera

- RGB Resolution
- FPS
- Color Format

## Motion Module

- Accelerometer Output Format
- Gyroscope Output Format
- Accelerometer FPS
- Gyroscope FPS

## Parameter documentation

- [RGB Settings](RGB_Settings.md)
- [IMU Settings](IMU_Settings.md)

## ROS integration

Use RGB8 when publishing a colored point cloud; the guide identifies RGB8 as required for RViz2 colorization. Enable the Motion Module when IMU-based SLAM is planned, and keep the gyro/accelerometer settings compatible with the synchronization behavior of the ROS 2 node.

Continue to [Point Clouds](../../03_PointCloud/README.md) after configuring RGB and IMU streams.
