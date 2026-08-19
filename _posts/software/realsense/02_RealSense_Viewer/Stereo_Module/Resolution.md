---
title: "Resolution"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, stereo_module]
---

# Resolution

## Description

Sets the spatial grid size (e.g., 640x480).

Higher resolution results in higher density points.

## When to Use

For rovers, use:

- 848x480
- 640x480

These balance depth precision with CPU and USB bandwidth.

## Trade-off

Resolution sets the depth image grid and therefore the maximum raw point-cloud density. Higher resolution gives more spatial samples but increases USB bandwidth, CPU load, and downstream point-cloud processing. The guide recommends 848 × 480 or 640 × 480 as practical rover choices.
