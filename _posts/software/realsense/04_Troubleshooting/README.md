---
title: "Troubleshooting"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 05_troubleshooting]
---

# Troubleshooting

Common issues encountered while using Intel RealSense D455 with RViz2.

## Topics

- RViz shows nothing
- Empty PointCloud2
- Tilted Point Cloud
- Black RGB Image
- Missing IMU
- TF Errors

## Troubleshooting guides

- [RViz Shows Nothing](RViz_No_Display.md)
- [Empty PointCloud2](Empty_PointCloud.md)
- [Tilted Point Cloud](Tilted_PointCloud.md)
- [Black RGB Image](Black_RGB_Image.md)
- [Missing IMU](Missing_IMU.md)
- [TF Errors](TF_Errors.md)

## Triage order

First confirm USB 3.2 detection with `rs-enumerate-devices`, then verify the ROS packages with `ros2 pkg list | grep realsense`. Next inspect camera topics and TF, then RViz2 settings. This order separates hardware, driver, data, and visualization failures.

For configuration context, return to [RealSense Viewer Parameters](../02_RealSense_Viewer/README.md) or [Point Clouds](../03_PointCloud/README.md).
