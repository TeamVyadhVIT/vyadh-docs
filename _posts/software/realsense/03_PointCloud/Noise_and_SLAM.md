---
title: "Point Cloud Density, Noise and SLAM"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 03_pointcloud]
---

# Point Cloud Density, Noise and SLAM

## Density

Point cloud density depends on:

- Resolution
- Visual Preset
- Decimation

---

## Noise

Noise can be reduced using:

- Spatial Filter
- Temporal Filter
- Hole Filling

---

## SLAM

Stable and consistent point clouds improve SLAM performance.

## Noise sources and mapping

Raw clouds can have Z-axis fluctuations and stereo occlusion shadows where only one imager observes a point. RTAB-Map-style SLAM uses stable geometric features such as corners and planes to align frames, estimate trajectory, and build an occupancy grid. Clean, consistent geometry matters more than maximum point count.
