---
title: "Emitter Frequency"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, stereo_module]
---

# Emitter Frequency

## Description

Sets the frequency of the infrared pulses.

Default:

- 57 kHz

## When to Use

Leave at the default value unless avoiding interference with other infrared devices.

## Interference consideration

The default IR pulse frequency is 57 kHz. Leave it at the default for a single camera. Adjust it only when another IR device or multiple-camera setup produces visible interference; use inter-camera synchronization as the primary multi-camera control.
