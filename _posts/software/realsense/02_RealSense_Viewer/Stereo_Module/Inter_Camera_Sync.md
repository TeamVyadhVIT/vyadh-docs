---
title: "Inter Camera Sync Mode"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, stereo_module]
---

# Inter Camera Sync Mode

## Description

Synchronizes multiple camera shutters.

Modes:

- 0 = Default
- 1 = Master
- 2 = Slave

## When to Use

Use when mounting multiple cameras on the rover to prevent infrared projector crosstalk.

## Multi-camera setup

Inter-camera synchronization coordinates shutters across cameras. The guide describes `0` as Default, `1` as Master, and `2` as Slave. Use it on a multi-camera rover to reduce projector crosstalk and make captures temporally consistent.
