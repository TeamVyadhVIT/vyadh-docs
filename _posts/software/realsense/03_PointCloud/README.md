---
title: "Point Clouds"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 03_pointcloud]
---

# Point Clouds

This folder explains how the Intel RealSense D455 generates point clouds from stereo images and how they are visualized in ROS 2.

## Topics

- Stereo to Depth
- PointCloud2
- Organized vs Unorganized Point Clouds
- Colored Point Clouds
- Point Cloud Density and Noise
- Point Clouds for SLAM

## Point-cloud documentation

- [Stereo to Depth](Stereo_to_Depth.md)
- [PointCloud2](PointCloud2.md)
- [Organized vs Unorganized Point Clouds](Organized_vs_Unorganized.md)
- [Colored Point Clouds](Colored_PointCloud.md)
- [Point Cloud Density, Noise and SLAM](Noise_and_SLAM.md)
- [Point Cloud Visualization](RViz_2.md)

## Data path

The camera pipeline is: D4 ASIC → Z16 depth image → ROS wrapper alignment → `sensor_msgs/PointCloud2` → RViz2, Nav2, or SLAM. Enable `align_depth.enable:=true` when RGB-colored points are required.

If the cloud or its inputs are missing, use the [Troubleshooting](../04_Troubleshooting/README.md) section.
