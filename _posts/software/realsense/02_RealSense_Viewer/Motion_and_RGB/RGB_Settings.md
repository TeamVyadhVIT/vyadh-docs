---
title: "RGB Resolution / FPS / Color Format"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, motion_and_rgb]
---

# RGB Resolution / FPS / Color Format

## RGB Resolution

Typical resolutions:

- 640 × 480
- 1280 × 720

---

## FPS

Typical frame rate:

- 30 FPS

---

## Color Format

Use:

- RGB8

RGB8 is required to colorize the PointCloud2 message in RViz2.

## Recommended stream choices

The guide lists 640 × 480 and 1280 × 720 as typical RGB resolutions at 30 FPS. RGB8 supplies the color values used to texture a PointCloud2 message; ensure depth-to-color alignment is enabled when a colored cloud is required.
