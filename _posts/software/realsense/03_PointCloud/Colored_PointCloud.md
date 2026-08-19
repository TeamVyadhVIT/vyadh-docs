---
title: "Colored Point Clouds"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 03_pointcloud]
---

# Colored Point Clouds

## Description

The RGB camera image can be aligned with the depth image.

Each 3D point is assigned its corresponding RGB value.

This produces a colored PointCloud2 for visualization in RViz2.

## Alignment requirement

Colored points require depth to be mapped into the RGB camera perspective. Launch the ROS wrapper with `align_depth.enable:=true`, use RGB8, and verify the aligned depth-to-color image before diagnosing an incorrect point-cloud color transformer.
