---
title: "Error Polling / Output Trigger"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, stereo_module]
---

# Error Polling / Output Trigger

## Description

Monitors hardware faults.

Allows external interrupts.

## When to Use

Enable for robust hardware diagnostics in autonomous systems.

## Diagnostics

Error polling monitors hardware faults and can expose them through external triggers. Enable it in autonomous deployments so the system can log camera faults and distinguish a sensor failure from an empty ROS topic or RViz configuration issue.
