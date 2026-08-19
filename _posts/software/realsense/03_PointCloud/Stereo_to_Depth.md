---
title: "Stereo to Depth"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 03_pointcloud]
---

# Stereo to Depth

## Description

The D455 generates depth using active stereo vision.

Two infrared cameras capture synchronized images.

The onboard Intel RealSense Vision Processor D4 computes the disparity between corresponding pixels.

The disparity is converted into depth using the camera geometry.

The resulting output is a 16-bit depth image that can be converted into a point cloud.

## Triangulation

Depth is computed from stereo disparity using `depth = focal_length × baseline / disparity`. The result is a two-dimensional Z16 matrix, normally interpreted in the depth-unit scale. Low disparity at long range and poor correspondence on textureless or occluded areas are common sources of uncertainty.
