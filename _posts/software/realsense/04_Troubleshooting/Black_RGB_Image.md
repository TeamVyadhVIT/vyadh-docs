---
title: "Black RGB Image"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 05_troubleshooting]
---

# Black RGB Image

## Possible Causes

- RGB stream disabled
- Incorrect RGB topic

## Recovery steps

Confirm you selected `/camera/color/image_raw`, not a depth topic. Improve room illumination, enable Auto Exposure in `realsense-viewer`, or raise exposure/gain manually while avoiding excessive gain noise. A dark physical RGB scene cannot be fixed by the RViz2 color transformer alone.
