---
title: "Intel RealSense D455 Notes"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, documentation]
---

# Intel RealSense D455 Notes

This repository contains organized notes from the Intel RealSense D455 parameter guide.

## Folder Structure

```
01_RealSense/
02_RealSense_Viewer/
03_PointCloud/
04_RViz2/
05_Troubleshooting/
```

## Contents

### 01_RealSense
Introduction to the Intel RealSense D455 camera architecture, RGB camera, IMU, IR projector and coordinate systems.

- [Open 01_RealSense](01_RealSense/README.md)

### 02_RealSense_Viewer
Explanation of every RealSense Viewer parameter grouped according to their functionality.

- [Open 02_RealSense_Viewer](02_RealSense_Viewer/README.md)

### 03_PointCloud
Concepts related to stereo disparity, depth images, PointCloud2, organized and unorganized point clouds.

- [Open 03_PointCloud](03_PointCloud/README.md)

### 04_Troubleshooting
Common RealSense and RViz2 issues with their causes.

- [Open 04_Troubleshooting](04_Troubleshooting/README.md)

## Recommended reading path

1. [Camera Architecture](01_RealSense/Camera_Architecture.md)
2. [RealSense Viewer Parameters](02_RealSense_Viewer/README.md)
3. [Point Clouds](03_PointCloud/README.md)
4. [Point Cloud Visualization](03_PointCloud/RViz_2.md)
5. [Troubleshooting](04_Troubleshooting/README.md)

## Source scope

These notes are based on the **RealSense D455 Rover Parameter Guide**. They document a practical ROS 2 rover workflow: configure the D455, publish depth/RGB/IMU data, inspect it in RViz2, and diagnose common failures.

## Recommended baseline

- Depth: 848 × 480 at 30 FPS
- RGB: 640 × 480 or 1280 × 720 at 30 FPS
- Auto Exposure and Auto White Balance: enabled
- Indoors: emitter enabled and laser power up to 360
- Spatial and Temporal filters: enabled; keep temporal filtering conservative on a fast-turning rover

For IMU-based SLAM, enable the Motion Module. Enable decimation only when CPU or bandwidth becomes a constraint.
