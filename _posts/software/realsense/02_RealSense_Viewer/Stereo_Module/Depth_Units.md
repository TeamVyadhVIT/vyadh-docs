---
title: "Depth Units"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, stereo_module]
---

# Depth Units

## Description

Defines the metric scale of the Z16 depth image.

Default:

- 0.001000 (1 mm)

## When to Use

Modify to:

- 0.0001 (100 µm)

to improve subpixel linearity at very close ranges.

## Metric scale

Depth Units scale the Z16 depth image into meters. The guide lists 0.001000 m (1 mm) as the default and notes that 0.0001 m (100 µm) may improve subpixel linearity at very close ranges. Any scale change must be interpreted consistently by consumers of the depth data.
