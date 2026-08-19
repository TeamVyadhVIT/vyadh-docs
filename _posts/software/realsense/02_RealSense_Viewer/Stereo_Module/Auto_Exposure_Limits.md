---
title: "Auto Exposure Limit / Auto Gain Limit"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, stereo_module]
---

# Auto Exposure Limit / Auto Gain Limit

## Description

Sets hard mathematical boundaries for the Auto Exposure algorithm.

## When to Use

For rovers:

- Cap exposure to prevent motion blur during turns.
- Cap gain to prevent noisy point clouds.

## Why limits matter

These limits bound the values selected by Auto Exposure. Cap exposure to reduce motion blur while turning, and cap gain to avoid noisy depth and point clouds. This is particularly important on a mobile robot because a visually bright image is not necessarily a geometrically reliable depth input.
