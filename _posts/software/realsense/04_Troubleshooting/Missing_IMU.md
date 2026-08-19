---
title: "Missing IMU"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 05_troubleshooting]
---

# Missing IMU

## Possible Causes

- Motion Module disabled
- IMU topic not enabled

## Driver settings

Enable `enable_gyro:=true` and `enable_accel:=true`. For a unified IMU topic, set `unite_imu_method:=1` or `2` as supported by the driver. Then inspect `/camera/imu` and confirm that its frame and rate match the requirements of the consumer.
