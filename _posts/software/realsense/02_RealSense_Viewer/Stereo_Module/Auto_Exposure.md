---
title: "Auto Exposure"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, stereo_module]
---

# Auto Exposure

## Description

Allows the ASIC to automatically adjust:

- Exposure
- Gain

## When to Use

Enable Auto Exposure when operating in dynamic lighting conditions.

## Use in changing light

Auto Exposure lets the ASIC adjust exposure and gain as illumination changes. It is recommended when a rover moves between differently lit areas. Pair it with exposure and gain limits so adaptation does not create long shutter times or noisy high-gain depth.
