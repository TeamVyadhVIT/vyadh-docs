---
title: "Spatial Filter"
date: 2026-08-19
author: team-vyadh
categories: [software, realsense-viewer]
tags: [intel-realsense, d455, 02_realsense_viewer, post_processing]
---

# Spatial Filter

## Description

Applies an edge-preserving domain transform.

## Effect

Smooths flat surfaces such as:

- Floors
- Walls

## Intended result

The spatial filter uses an edge-preserving domain transform. It smooths flat floors and walls while attempting to retain object boundaries, reducing local depth noise without simply blurring the whole scene.
