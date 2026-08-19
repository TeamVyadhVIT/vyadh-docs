---
title: "Hole Filling"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, post_processing]
---

# Hole Filling

## Description

Uses nearest-neighbor heuristics to patch zero-depth pixels.

## Measurement caveat

Hole filling uses nearest-neighbor heuristics to patch zero-depth pixels. It is useful for visualization and may make a cloud more continuous, but inserted values are estimates. The guide recommends it for visualization and leaves it optional for measurement-sensitive work.
