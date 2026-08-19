---
title: "Point Cloud Visualization in RViz2"
date: 2026-08-19
author: arham
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, rviz2, pointcloud, ros2]
---

# Point Cloud Visualization Summary

RViz2 is used to inspect the Intel RealSense D455 data published by the ROS 2 driver. It provides a single view for validating the point cloud, RGB and depth images, colorization, and camera coordinate frames.

## Required camera configuration

Launch the camera with point-cloud and depth-alignment support enabled:

```text
pointcloud.enable:=true
align_depth.enable:=true
```

The point cloud is normally published on `/camera/depth/color/points`. RGB and depth streams can be checked independently using:

- `/camera/color/image_raw`
- `/camera/depth/image_rect_raw`

## RViz2 setup workflow

1. Start the RealSense ROS 2 node in a sourced terminal.
2. Open RViz2.
3. Set the Fixed Frame to `camera_link`.
4. Add a PointCloud2 display and select `/camera/depth/color/points`.
5. Add RGB and depth Image displays.
6. Add a TF display to inspect the camera frame tree.
7. Save the working configuration, for example as `rover_perception.rviz`.

The Fixed Frame must belong to the same connected TF tree as the camera data. If `camera_link` is unavailable, inspect the published frames and use a valid camera frame only after confirming the TF tree.

## Point cloud display

The PointCloud2 display renders depth points in three-dimensional space. The Squares style with a size of 3 or 5 gives a readable cloud in the baseline setup. An empty cloud usually indicates that point-cloud publishing or depth-to-color alignment is disabled, rather than a rendering problem in RViz2.

PointCloud2 may contain XYZ coordinates and RGB fields when colorization is configured. This makes the cloud useful for visual inspection, 3D mapping, Nav2 voxel layers, and SLAM inputs.

## Image validation

Inspect the RGB and depth images before diagnosing a point-cloud problem. Image displays help isolate whether the camera is publishing valid color and depth streams. An aligned depth-to-color image is especially useful when checking a colorized point cloud.

## Color transformation

The PointCloud2 color transformer controls how points are rendered. Use RGB8 when RGB fields are available for true camera colors. Intensity or AxisColor can be useful when RGB data is unavailable or the scene is dark.

A transformer setting changes visualization only. It cannot repair missing depth, invalid point data, or broken depth-to-color alignment.

## TF validation

The TF display verifies the spatial relationship between the camera frames. A typical chain is:

```text
base_link -> camera_link -> camera_depth_frame / camera_color_frame -> optical frames
```

Check this chain before treating a tilted, displaced, or missing cloud as a depth-processing failure. Transform errors can result from a missing camera frame, an incorrect Fixed Frame, or disabled TF publication.

## Troubleshooting order

Use this order when RViz2 does not display the expected data:

1. Confirm that the RealSense ROS 2 node is running.
2. Verify that the RGB and depth image topics publish data.
3. Confirm `pointcloud.enable:=true` and `align_depth.enable:=true`.
4. Check that `/camera/depth/color/points` is selected in the PointCloud2 display.
5. Verify the Fixed Frame and inspect the TF display.
6. Change the color transformer only after depth, alignment, and TF are working.

