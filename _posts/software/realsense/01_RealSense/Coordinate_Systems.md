---
title: "Coordinate Systems"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 01_realsense]
---

# Coordinate Systems

## Intrinsics

Internal camera parameters including:

- Focal length
- Principal point
- Lens distortion

---

## Extrinsics

The physical rigid-body transformations (rotation and translation) between different sensors on the device.

Example:

- Left depth imager
- RGB imager

---

## Standard CV Coordinate System

- **Z** points forward (depth)
- **X** points right
- **Y** points down

## Intrinsics and extrinsics

Intrinsics describe one imager's focal length, principal point, and lens distortion. They are used to project a depth pixel into a 3D coordinate. Extrinsics are the fixed rotation and translation between device sensors, such as the depth and RGB imagers.

## Optical-frame convention

The standard computer-vision optical convention used by the guide is:

- X: right
- Y: down
- Z: forward, along depth

Use the optical frames for image-derived data. ROS TF then relates those frames to `camera_link` and, through the robot transform chain, to `base_link`.

Related topics: [Camera Architecture](Camera_Architecture.md), [RGB Camera and IMU](RGB_and_IMU.md), and [PointCloud2](../03_PointCloud/PointCloud2.md).
