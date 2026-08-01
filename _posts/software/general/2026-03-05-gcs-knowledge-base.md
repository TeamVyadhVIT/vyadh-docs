---
title: "GCS Knowledge Base — Index"
date: 2026-03-05
author: mahi
categories: [software, general]
tags: [gcs, index, networking, wireless, mqtt, gstreamer, codebase]
---

Ground Control Station — technical documentation for the rover team.

This is the entry point for the GCS documentation set. It covers the networking and
RF fundamentals the link depends on, the two transport stacks we run over that link
(MQTT for telemetry, GStreamer for video), and a walkthrough of every program in the
GCS and rover codebase.

## Networking

- [IP Addressing]({{ site.baseurl }}/posts/ip-addressing/)
- [TCP and UDP]({{ site.baseurl }}/posts/tcp-and-udp/)
- [OSI Model]({{ site.baseurl }}/posts/osi-model/)
- [Routing and Switching]({{ site.baseurl }}/posts/routing-and-switching/)

## Wireless & RF

- [Wireless Communication]({{ site.baseurl }}/posts/wireless-communication/) — start here
- [RF Fundamentals]({{ site.baseurl }}/posts/rf-fundamentals/)
- [Antenna Types]({{ site.baseurl }}/posts/antenna-types/)
- [RocketM5 — Overview]({{ site.baseurl }}/posts/rocketm5-overview/)
- [RocketM5 — Configuration]({{ site.baseurl }}/posts/rocketm5-configuration/)
- [RocketM5 — Omni vs Sector]({{ site.baseurl }}/posts/rocketm5-omni-vs-sector/)

## MQTT

- [Concepts & Protocol]({{ site.baseurl }}/posts/mqtt-concepts-and-protocol/)
- [Broker Setup (Mosquitto)]({{ site.baseurl }}/posts/mqtt-broker-setup-mosquitto/)
- [Rover Usage]({{ site.baseurl }}/posts/mqtt-rover-usage/)
- [Tools]({{ site.baseurl }}/posts/mqtt-tools/)

## GStreamer

- [Core Concepts]({{ site.baseurl }}/posts/gstreamer-core-concepts/)
- [Encoding & Transport Protocols]({{ site.baseurl }}/posts/gstreamer-encoding-and-transport/)
- [Common Pipelines]({{ site.baseurl }}/posts/gstreamer-common-pipelines/)
- [Rover Usage]({{ site.baseurl }}/posts/gstreamer-rover-usage/)
- [Troubleshooting]({{ site.baseurl }}/posts/gstreamer-troubleshooting/)

## Codebase

### GCS Applications

- [GCS Dashboard — Multi-Server Video Client]({{ site.baseurl }}/posts/gcs-dashboard-multi-server-video-client/)
- [Map Visualizer — ROS2 Interactive Map Viewer]({{ site.baseurl }}/posts/map-visualizer-ros2-interactive-map-viewer/)
- [RGBD Point Selector]({{ site.baseurl }}/posts/rgbd-point-selector/)
- [Multi-Camera Feed GUI System]({{ site.baseurl }}/posts/multi-camera-feed-gui-system/)
- [Backup Client-Side Bash Script]({{ site.baseurl }}/posts/backup-client-side-bash-script/)

### Camera Server

- [Overview]({{ site.baseurl }}/posts/camera-server-overview/)
- [Pi Mode]({{ site.baseurl }}/posts/camera-server-pi-mode/)
- [Laptop Mode]({{ site.baseurl }}/posts/camera-server-laptop-mode/)

### GPS

- [Subsystem Overview]({{ site.baseurl }}/posts/gps-subsystem-overview/)
- [Receiver GUI — GCS Side]({{ site.baseurl }}/posts/gps-receiver-gui-gcs-side/)
- [Serial to UDP Bridge — Rover Side]({{ site.baseurl }}/posts/gps-serial-to-udp-bridge-rover-side/)
- [ESP32 Firmware — GPS + MQ Sensors]({{ site.baseurl }}/posts/esp32-firmware-gps-and-mq-sensors/)

### Sensors

- [Sensor Dashboard — GCS Side]({{ site.baseurl }}/posts/sensor-dashboard-gcs-side/)
- [Serial-to-MQTT Bridge — Rover Side]({{ site.baseurl }}/posts/serial-to-mqtt-bridge-rover-side/)
- [ESP32 Sensor Node]({{ site.baseurl }}/posts/esp32-sensor-node/)
- [ESP32 Sensor Node 1 — BME680 + Soil NPK]({{ site.baseurl }}/posts/esp32-sensor-node-1-bme680-and-soil-npk/)
- [ESP32 Sensor Node 2 — GPS + MQ Gas Sensors]({{ site.baseurl }}/posts/esp32-sensor-node-2-gps-and-mq-gas-sensors/)

### Traversal

- [AstroBIO Rover — System Overview]({{ site.baseurl }}/posts/astrobio-rover-system-overview/)
- [Rover UDP–Serial/ROS Bridge]({{ site.baseurl }}/posts/traversal-rover-udp-serial-bridge/)
- [GCS Joystick UDP Sender]({{ site.baseurl }}/posts/traversal-gcs-joystick-udp-sender/)

### AstroBIO

- [GUI Node]({{ site.baseurl }}/posts/astrobio-gcs-gui-node/)
- [Rover Arduino Firmware]({{ site.baseurl }}/posts/astrobio-rover-arduino-firmware/)

### Logs

- [ROS2 Log Monitor GUI]({{ site.baseurl }}/posts/ros2-log-monitor-gui/)

> The MkDocs nav also listed a **GCS Architecture** section and an **Operations**
> section (field setup SOP, troubleshooting guide). Those pages were never written in
> the source repository, so there was nothing to migrate — they still need authoring.
{: .prompt-info }
