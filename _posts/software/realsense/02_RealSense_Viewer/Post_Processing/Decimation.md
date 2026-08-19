---
title: "Decimation"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, post_processing]
---

# Decimation

## Description

Intelligently downsamples the depth image.

Example:

- Magnitude 2 reduces points by 75%.

## Effect

Reduces:

- CPU load
- Bandwidth usage

## Resource impact

Decimation downsamples the depth image before later processing. A magnitude of 2 reduces points by 75%, which can dramatically cut CPU, bandwidth, and voxel-processing work. Keep it off while learning or validating geometry; enable it when the compute budget requires it.
