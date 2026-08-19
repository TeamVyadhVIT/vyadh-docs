---
title: "Global Time Enabled"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, stereo_module]
---

# Global Time Enabled

## Description

Synchronizes hardware timestamps with the host PC clock.

## When to Use

Enable for accurate ROS 2 TF tree alignments.

## Timestamp alignment

Global Time synchronizes hardware timestamps to the host PC clock. Enable it when ROS 2 transforms and sensor messages must be correlated with host-time data; the guide identifies it as important for accurate TF-tree alignment.
