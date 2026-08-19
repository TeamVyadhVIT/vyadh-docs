---
title: "Empty PointCloud2"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 05_troubleshooting]
---

# Empty PointCloud2

## Possible Causes

- Depth stream disabled
- Incorrect topic selection
- No valid depth data

## Launch check

For `/camera/depth/color/points`, pass both `pointcloud.enable:=true` and `align_depth.enable:=true` to the launch file. Also verify that depth data is valid and that the topic selection matches the active camera namespace.
