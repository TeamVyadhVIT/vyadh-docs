---
title: "Rau / SLO Color Thresholds"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, advanced_mode]
---

# Rau / SLO Color Thresholds

## Description

Prevents the depth map from interpolating values across strong visual color boundaries.

## Effect

Prevents point cloud spraying.

## Boundary protection

RAU/SLO color thresholds prevent the depth map from interpolating across strong visual color boundaries. This helps avoid “point cloud spraying,” where smoothing creates implausible depth points across an object edge.
