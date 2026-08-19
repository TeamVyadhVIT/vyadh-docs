---
title: "RViz Shows Nothing"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 05_troubleshooting]
---

# RViz Shows Nothing

## Possible Causes

- Incorrect Fixed Frame
- Incorrect PointCloud2 topic
- Camera node not running

## Recovery steps

Set RViz2 Fixed Frame to `camera_link` or a valid `camera_depth_optical_frame`. Confirm the selected point-cloud topic exists and the camera node is running. If RViz2 reports a transform error, inspect `/tf` and `/tf_static` before changing visual settings.
