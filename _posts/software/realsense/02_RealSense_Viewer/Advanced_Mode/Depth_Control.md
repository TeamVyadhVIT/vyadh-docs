---
title: "Depth Control"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, advanced_mode]
---

# Depth Control

## Description

Depth Control adjusts the mathematical penalties for left-right image mismatches.

Parameters include:

- Minus Decrement
- Score Threshold A
- Score Threshold B

## Effect

- High thresholds produce sparse, highly confident point clouds.
- Low thresholds produce dense, noisy point clouds.

## Confidence trade-off

Minus Decrement and Score Threshold A/B adjust penalties for left-right image mismatches. Higher thresholds yield a sparser cloud with more confident points; lower thresholds can produce a denser but noisier cloud. Use this trade-off deliberately for obstacle avoidance versus measurement-quality tasks.
