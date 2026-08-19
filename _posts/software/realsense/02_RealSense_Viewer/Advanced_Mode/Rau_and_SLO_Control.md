---
title: "Rau Support Vector Control & SLO Penalty Control"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, advanced_mode]
---

# Rau Support Vector Control & SLO Penalty Control

## Description

Defines:

- Spatial bounding boxes
    - Min West
    - Min East
    - Min North
    - Min South

- Algorithmic penalties
    - K1 Penalty
    - K2 Penalty

## Effect

Used for:

- Smoothing flat surfaces
- Preserving sharp edges

## Surface and edge control

RAU support-vector bounds use minimum west/east/north/south values to define spatial regions. SLO K1/K2 penalties govern smoothing behavior. Together these controls balance smoothing of broad flat surfaces against preservation of sharp scene edges.
