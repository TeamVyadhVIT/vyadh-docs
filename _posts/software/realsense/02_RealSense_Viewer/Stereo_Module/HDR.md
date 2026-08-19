---
title: "HDR (Sequence ID / Size)"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, stereo_module]
---

# HDR (Sequence ID / Size)

## Description

Multiplexes exposure between short and long frames.

## When to Use

Enable when the rover moves between dark and brightly lit areas.

## High-contrast scenes

HDR sequences multiplex short and long exposures. This is useful when the rover transitions between dark and bright regions. Validate the chosen sequence against motion: alternating exposures can improve dynamic range but must still produce stable depth for the intended workload.
