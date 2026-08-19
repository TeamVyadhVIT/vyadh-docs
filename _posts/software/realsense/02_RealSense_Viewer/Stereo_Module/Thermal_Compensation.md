---
title: "Thermal Compensation"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, stereo_module]
---

# Thermal Compensation

## Description

Adjusts camera calibration dynamically as the ASIC heats up.

## When to Use

Enable to prevent depth degradation during long periods of operation.

## Long-duration runs

Thermal compensation dynamically adjusts calibration as the ASIC warms up. Enable it for long rover runs to reduce depth degradation caused by temperature changes after startup.
