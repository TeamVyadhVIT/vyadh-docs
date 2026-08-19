---
title: "Stereo Baseline"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, stereo_module]
---

# Stereo Baseline

## Description

Read-only value indicating the physical distance between the stereo sensors.

For the D455:

- 95.0371 mm

## Usage

Used internally by the triangulation formula.

## Role in triangulation

Stereo Baseline is a read-only physical separation between the depth imagers; the D455 value in the guide is 95.0371 mm. It is used internally with focal length and disparity: `depth = focal_length × baseline / disparity`.
