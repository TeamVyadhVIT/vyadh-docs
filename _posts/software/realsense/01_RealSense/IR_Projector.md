---
title: "IR Projector"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 01_realsense]
---

# IR Projector

To assist the stereo matching algorithm in featureless environments (such as blank indoor walls), the D455 uses an active infrared projector.

The projector emits a semi-random dot pattern at **850 nm**.

## Projected texture

The IR projector emits a semi-random 850 nm dot pattern. It supplies visual texture to the stereo matcher where a scene lacks natural features, such as plain indoor walls or floors.

## Practical use

Use the laser emitter indoors for active stereo. Sunlight can wash out infrared texture, so outdoor or highly reflective scenes may work better with the emitter disabled. Increasing laser power raises dot-pattern density, but it does not replace correct exposure or good camera placement.

Related topics: [Camera Architecture](Camera_Architecture.md), [Emitter](../02_RealSense_Viewer/Stereo_Module/Emitter.md), and [Laser Power](../02_RealSense_Viewer/Stereo_Module/Laser_Power.md).
