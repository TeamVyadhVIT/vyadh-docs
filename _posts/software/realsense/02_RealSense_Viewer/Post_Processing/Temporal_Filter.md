---
title: "Temporal Filter"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, post_processing]
---

# Temporal Filter

## Description

Averages depth values across sequential frames.

## Effect

Creates a highly stable point cloud.

Requires tuning so moving obstacles do not leave ghosting trails.

## Motion caveat

Temporal filtering averages values across frames and can create a much more stable cloud. On moving obstacles or a fast-turning rover it can leave ghost trails, so tune it conservatively; the guide specifically recommends keeping alpha low when the rover turns quickly.
