---
title: "Census Enable Register"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, advanced_mode]
---

# Census Enable Register

## Description

Defines the pixel patch size used in the stereoscopic search.

Parameters include:

- u-Diameter
- v-Diameter

## Stereo patch size

The Census Enable Register controls the `u` and `v` diameters of the pixel patch used during stereo search. Patch size changes the local image context used to distinguish correspondences, so tune it only with repeatable scene tests.
