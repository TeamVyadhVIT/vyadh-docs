---
title: "Color Scheme and Histogram Equalization"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, post_processing]
---

# Color Scheme and Histogram Equalization

## Color Scheme

Defines how the 16-bit depth image is mapped to RGB colors.

Example:

- Jet color map

---

## Histogram Equalization

Maximizes contrast for human viewing.

### Recommendation

Turn off Histogram Equalization to save CPU on autonomous rovers.

## Display-only decision

Color scheme maps 16-bit depth to display colors, for example the Jet colormap. Histogram equalization increases human-visible contrast but is not a depth-quality improvement; the guide recommends disabling it when saving CPU on an autonomous rover.
