---
title: "Minimum / Maximum Distance"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, post_processing]
---

# Minimum / Maximum Distance

## Description

Bounds the visual rendering window.

Example:

- Minimum Distance: 0.0 m
- Maximum Distance: 6.0 m

## Rendering bounds

These values bound the visual rendering window rather than changing physical scene depth. A 0.0–6.0 m range is given as an example. Set bounds to make the operating region readable in Viewer/RViz2, and use deliberate thresholding when the application truly needs to discard depth by range.
