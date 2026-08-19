---
title: "HDAD (Ignore SAD)"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, advanced_mode]
---

# HDAD (Ignore SAD)

## Description

Forces the ASIC to rely purely on the Census transform instead of the Sum of Absolute Differences (SAD).

## Recommendation

Recommended for outdoor environments.

## Outdoor use

HDAD's Ignore SAD setting can force the ASIC to rely on Census-transform information rather than absolute brightness (Sum of Absolute Differences). The guide recommends this approach for outdoor environments, where brightness variation can make direct intensity comparison less reliable.
