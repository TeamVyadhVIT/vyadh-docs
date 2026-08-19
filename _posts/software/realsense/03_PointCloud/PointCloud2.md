---
title: "PointCloud2"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 03_pointcloud]
---

# PointCloud2

## Description

PointCloud2 is the ROS 2 message used to represent three-dimensional point cloud data.

Each point contains:

- X
- Y
- Z

Optionally, color information from the RGB camera can also be included.

The depth image is projected into 3D space before publishing as a PointCloud2 message.

## ROS topic and rate

The guide identifies `/camera/depth/color/points` as a `sensor_msgs/PointCloud2` stream at roughly 15–30 Hz. It carries XYZ and, when alignment/colorization are configured, RGB. It is suitable for RViz2 display, Nav2 voxel layers, and 3D SLAM inputs.
