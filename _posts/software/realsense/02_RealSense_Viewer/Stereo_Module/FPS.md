---
title: "FPS (Frame Rate)"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, stereo_module]
---

# FPS (Frame Rate)

## Description

Defines the image capture speed.

Maximum frame rate:

- 90 FPS

## When to Use

Use:

- 15 FPS
- 30 FPS

for Nav2 costmaps to save CPU resources.

## Rover guidance

Frame rate controls how often a new depth image and point cloud can be produced. The D455 can reach 90 FPS in supported modes, but 15 or 30 FPS is normally sufficient for Nav2 costmaps and preserves compute capacity for localization and planning. Select FPS together with resolution because both determine bandwidth.
