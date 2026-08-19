---
title: "Organized vs Unorganized Point Clouds"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 03_pointcloud]
---

# Organized vs Unorganized Point Clouds

## Organized Point Cloud

An organized point cloud preserves the image structure.

Each pixel in the depth image corresponds to one point.

---

## Unorganized Point Cloud

An unorganized point cloud is stored as a list of points without image indexing.

This representation is commonly used after filtering or processing.

## Use cases

Organized clouds preserve the depth-image matrix, including invalid points, and are efficient for image-style operations such as surface-normal estimation. Unorganized clouds contain a one-dimensional set of valid XYZ points; their smaller payload is useful for Nav2 voxel grids and filtered processing.
