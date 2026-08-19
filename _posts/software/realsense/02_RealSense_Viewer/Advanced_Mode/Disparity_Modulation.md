---
title: "Disparity Modulation (A-Factor)"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, advanced_mode]
---

# Disparity Modulation (A-Factor)

## Description

Counteracts staircase quantization noise on flat walls.

## Recommended Value

- 0.08

## Effect

Improves subpixel linearity.

## Subpixel linearity

The A-Factor counters staircase-like disparity quantization on flat walls. The guide gives 0.08 as an example setting for improved subpixel linearity. Evaluate it on a planar test target before applying it to a navigation stack.
