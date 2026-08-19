---
title: "Post Processing"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, post_processing]
---

# Post Processing

Post-processing filters modify the depth image before visualization or point cloud generation.

## Parameters

| Parameter | Purpose |
|-----------|---------|
| Visual Presets | High-level ASIC configurations |
| Color Scheme | Maps depth to RGB colors |
| Histogram Equalization | Increases visualization contrast |
| Minimum / Maximum Distance | Defines rendering range |
| Decimation | Reduces depth image density |
| Spatial Filter | Smooths flat surfaces |
| Temporal Filter | Stabilizes depth across frames |
| Hole Filling | Patches zero-depth pixels |

## Filter documentation

- [Visual Presets](Visual_Presets.md)
- [Color Scheme](Color_Scheme.md)
- [Minimum / Maximum Distance](Min_Max_Distance.md)
- [Decimation](Decimation.md)
- [Spatial Filter](Spatial_Filter.md)
- [Temporal Filter](Temporal_Filter.md)
- [Hole Filling](Hole_Filling.md)

## Processing order

Post-processing modifies depth before visualization or point-cloud generation. Begin with the least destructive controls: visual preset and display range, then spatial/temporal filtering, and finally decimation or hole filling. Confirm that a cleaner-looking display still meets measurement and obstacle-detection needs.

See the result in [Point Clouds](../../03_PointCloud/README.md).
