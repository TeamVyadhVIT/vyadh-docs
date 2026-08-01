---
title: "RGBD Point Selector — 3D Coordinate Picker from Depth Camera"
date: 2026-03-01
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, ros2, depth-camera]
---

## Overview

This is a ROS2 node that lets an operator click on a live RGB camera feed and instantly get the real-world 3D coordinates of the clicked point in centimetres, relative to the camera. It works by combining the colour image from an RGB-D camera (e.g. Intel RealSense) with the corresponding depth image — the colour image shows you what to click, and the depth image tells you how far away that pixel is.

On each click, the node:

1. Records the pixel coordinates `(x, y)` of the click.
2. On the next synchronised RGB+depth frame, reads the depth value at that pixel.
3. Uses the camera's intrinsic calibration matrix to back-project the 2D pixel and depth into a real-world 3D point `(X, Y, Z)` in the camera coordinate frame.
4. Publishes the result as a `geometry_msgs/Vector3` message to the `target_coordinates` topic.
5. Draws a red dot and the XYZ text overlay on the RGB display window.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                  RGBD POINT SELECTOR (this file)                 │
│                                                                   │
│  Subscribers:                                                     │
│  ├─ /camera/camera/color/image_raw  ──┐                          │
│  │                                    ├─► ApproximateTime        │
│  ├─ /camera/camera/depth/image_rect_raw ┘   Synchronizer         │
│  │                                          └─► image_callback() │
│  └─ /camera/camera/color/camera_info ──► camera_info_callback()  │
│                                                                   │
│  Publisher:                                                       │
│  └─ /target_coordinates  (geometry_msgs/Vector3, cm)             │
│                                                                   │
│  OpenCV Window:                                                   │
│  └─ 'RGB Feed'                                                    │
│     └─ Mouse click ──► on_mouse_click() ──► self.point = (x, y)  │
│                                                                   │
│  On each synchronised frame (if self.point is set):              │
│  1. Read depth_image[y, x]  (raw uint16, millimetres)            │
│  2. depth_meters = raw * 0.001                                    │
│  3. Back-project using K matrix → X, Y, Z in metres              │
│  4. Convert to cm → publish Vector3                               │
│  5. Reset self.point = None                                       │
└─────────────────────────────────────────────────────────────────┘
         ▲                    ▲                      ▼
         │                    │                      │
  /camera/color         /camera/depth        /target_coordinates
  /image_raw            /image_rect_raw       (Vector3, cm)
         │                    │
  ┌──────┴────────────────────┴──────┐
  │     Intel RealSense (or similar) │
  │     RGB-D Camera Driver          │
  └──────────────────────────────────┘
```

---

## Dependencies

| Library | Purpose | Install |
|---|---|---|
| `rclpy` | ROS2 Python client library | Included with ROS2 |
| `sensor_msgs` | `Image`, `CameraInfo` message types | Included with ROS2 |
| `geometry_msgs` | `Vector3` message type for output | Included with ROS2 |
| `cv_bridge` | Converts ROS2 `Image` messages to OpenCV `numpy` arrays | `sudo apt install ros-humble-cv-bridge` |
| `cv2` (OpenCV) | Image display, drawing, mouse callback | `pip install opencv-python` |
| `message_filters` | `ApproximateTimeSynchronizer` — synchronises two image streams | Included with ROS2 |
| `numpy` | Reshaping the camera intrinsics array | `pip install numpy` |

### Full Installation

```bash
# ROS2 packages
sudo apt install \
    ros-humble-cv-bridge \
    ros-humble-message-filters \
    ros-humble-sensor-msgs \
    ros-humble-geometry-msgs

# Python
pip install opencv-python numpy
```

---

## ROS2 Topics

| Topic | Direction | Message Type | Description |
|---|---|---|---|
| `/camera/camera/color/image_raw` | Subscribe | `sensor_msgs/Image` | RGB colour image from the camera |
| `/camera/camera/depth/image_rect_raw` | Subscribe | `sensor_msgs/Image` | Depth image, rectified, 16-bit unsigned, millimetres |
| `/camera/camera/color/camera_info` | Subscribe | `sensor_msgs/CameraInfo` | Camera intrinsic calibration — focal length, principal point |
| `/target_coordinates` | Publish | `geometry_msgs/Vector3` | 3D coordinates of clicked point in centimetres, camera frame |

> **Topic Names**
>
> The topic names above are the defaults for an Intel RealSense camera using the `realsense2_camera` ROS2 driver. If using a different camera or a different driver configuration, the topic names will differ. Check `ros2 topic list` after launching the camera driver and update the subscriber topic strings accordingly.
{: .prompt-info }

---

## The Core Concept — Pinhole Camera Back-Projection

This is the most important concept in the entire codebase. Understanding this makes everything else obvious.

### What the Camera Intrinsic Matrix K Is

Every camera has a calibration matrix `K` (also called the intrinsic matrix) that describes its optical properties. For a standard pinhole camera model:

```
K = | fx   0   cx |
    |  0  fy   cy |
    |  0   0    1 |
```

| Parameter | Meaning |
|---|---|
| `fx` | Focal length in pixels, X axis |
| `fy` | Focal length in pixels, Y axis |
| `cx` | Principal point X — pixel column where the optical axis hits the sensor (usually near the centre) |
| `cy` | Principal point Y — pixel row where the optical axis hits the sensor |

These values come from camera calibration and are published on the `CameraInfo` topic. The `K` field is a flat 9-element array (row-major), which the code reshapes into a 3×3 matrix:

```python
K = np.array(self.camera_info.k).reshape(3, 3)
fx = K[0, 0]
fy = K[1, 1]
cx = K[0, 2]
cy = K[1, 2]
```

### The Back-Projection Formula

Given:
- A pixel position `(x_pixel, y_pixel)` from a mouse click
- The depth at that pixel in metres (`Z`)
- The camera intrinsics `fx`, `fy`, `cx`, `cy`

The real-world 3D position in the **camera coordinate frame** is:

```
X = (x_pixel - cx) * Z / fx
Y = (y_pixel - cy) * Z / fy
Z = Z  (depth, unchanged)
```

**Intuition:** `(x_pixel - cx)` is how many pixels the clicked point is to the right of the optical centre. Dividing by `fx` converts pixels to radians of angle. Multiplying by `Z` (depth) converts angle to distance. The result is how far in metres the point is to the right of the camera's forward axis.

### Camera Coordinate Frame

The output `(X, Y, Z)` is in the **camera coordinate frame**:

```
     Z (forward, into the scene)
    /
   /
  +────── X (right)
  │
  │
  Y (down)
```

- **Z** — distance straight ahead from the camera (depth)
- **X** — distance to the right (positive) or left (negative)
- **Y** — distance downward (positive) or upward (negative)

This is the standard ROS camera frame convention. If you need these coordinates in the robot's base frame or the map frame, a TF transform is required (not implemented here — see Known Limitations).

---

## How It Works — Step by Step

### 1. Camera Info Callback — `camera_info_callback(msg)`

Called once (or whenever the camera publishes updated calibration). Stores the `CameraInfo` message in `self.camera_info`. The node will not attempt 3D computation until this has been received at least once.

```python
def camera_info_callback(self, msg):
    self.camera_info = msg
```

The `CameraInfo` message contains the full intrinsic matrix, distortion coefficients, and rectification data. This node uses only `K` (the intrinsic matrix) because the depth image topic used is `image_rect_raw` — the `rect` indicates it has already been rectified (lens distortion removed), so distortion coefficients are not needed.

---

### 2. Image Synchronisation — `ApproximateTimeSynchronizer`

The RGB and depth images come from the same physical camera but travel through different ROS2 topics and may have slightly different timestamps due to processing delays. Naively subscribing to both independently and pairing the latest of each would result in mismatched frames — the depth map for frame N paired with the colour image for frame N+1.

`ApproximateTimeSynchronizer` solves this:

```python
self.ts = ApproximateTimeSynchronizer(
    [self.rgb_sub, self.depth_sub],
    queue_size=10,
    slop=0.1
)
self.ts.registerCallback(self.image_callback)
```

| Parameter | Value | Meaning |
|---|---|---|
| `queue_size` | 10 | Buffer up to 10 messages from each topic while waiting for a match |
| `slop` | 0.1 | Maximum allowed timestamp difference in seconds (100 ms) |

The synchroniser holds incoming messages in a queue and calls `image_callback` only when it finds an RGB and a depth message whose timestamps are within 100 ms of each other. This guarantees the colour and depth images correspond to the same moment in time.

---

### 3. Image Callback — `image_callback(rgb_msg, depth_msg)`

Called by the synchroniser with a matched pair. Converts both messages to OpenCV arrays:

```python
self.rgb_image = self.bridge.imgmsg_to_cv2(rgb_msg, desired_encoding='bgr8')
self.depth_image = self.bridge.imgmsg_to_cv2(depth_msg, desired_encoding='16UC1')
```

| Encoding | Type | Description |
|---|---|---|
| `bgr8` | `uint8` 3-channel | Standard OpenCV colour format. Values 0–255 per channel. |
| `16UC1` | `uint16` 1-channel | 16-bit unsigned single-channel depth. Each value is depth in **millimetres**. Maximum representable depth: 65535 mm ≈ 65.5 m. |

After conversion, if `self.point` is set (a click has occurred), the 3D computation runs.

---

### 4. Depth Reading and Validation

```python
depth_raw = self.depth_image[y_pixel, x_pixel]
if depth_raw == 0:
    self.get_logger().warn("Depth at clicked point is zero, ignoring.")
else:
    depth_meters = depth_raw * 0.001
```

`depth_raw == 0` means the depth sensor returned no valid measurement at that pixel. This happens at:
- Reflective or transparent surfaces (mirrors, glass, water)
- Pixels beyond the sensor's maximum range
- Pixels too close to the sensor (below minimum range, typically ~15–20 cm for RealSense)
- Edges of objects where the IR projector pattern is disrupted
- Pixels in the shadow of the IR projector

When depth is zero, the node logs a warning and skips the computation. `self.point` is **not** reset, meaning the node will try again on the next frame. If the operator clicks a valid area, it will eventually compute successfully.

The raw depth value is converted from millimetres to metres by multiplying by `0.001`.

---

### 5. 3D Coordinate Computation

```python
K = np.array(self.camera_info.k).reshape(3, 3)
fx, fy = K[0, 0], K[1, 1]
cx, cy = K[0, 2], K[1, 2]

X = (x_pixel - cx) * depth_meters / fx
Y = (y_pixel - cy) * depth_meters / fy
Z = depth_meters

X_cm = X * 100.0
Y_cm = Y * 100.0
Z_cm = Z * 100.0
```

The result is converted to centimetres before publishing. This is a team convention — cm is more intuitive than metres for close-range object distances.

---

### 6. Publish and Display

```python
coord_msg = Vector3()
coord_msg.x = float(X_cm)
coord_msg.y = float(Y_cm)
coord_msg.z = float(Z_cm)
self.coord_pub.publish(coord_msg)
```

A red dot is drawn at the clicked pixel and the XYZ text is overlaid on the display image. `self.point` is then reset to `None`, so the result is published exactly once per click — on the next available valid frame after the click.

---

### 7. Mouse Callback — `on_mouse_click(event, x, y, flags, param)`

```python
def on_mouse_click(self, event, x, y, flags, param):
    if event == cv2.EVENT_LBUTTONDOWN:
        self.point = (x, y)
```

OpenCV's mouse callback is called on the main thread whenever the user interacts with the `RGB Feed` window. Only left-button-down events are handled. The click coordinates are stored and the actual 3D computation happens on the next image callback, not here — this keeps the mouse handler fast and non-blocking.

---

### 8. Quit Handling

```python
key = cv2.waitKey(1) & 0xFF
if key == ord('q'):
    cv2.destroyAllWindows()
    raise KeyboardInterrupt
```

Pressing `q` in the OpenCV window raises `KeyboardInterrupt`, which is caught in `main()` and triggers clean shutdown:

```python
try:
    rclpy.spin(node)
except KeyboardInterrupt:
    pass
finally:
    node.destroy_node()
    cv2.destroyAllWindows()
    rclpy.shutdown()
```

Unlike the Map Visualizer (which uses Tkinter's `root.mainloop()`), this node uses `rclpy.spin()` directly — the OpenCV window is driven by `cv2.waitKey(1)` inside the image callback, called every frame.

---

## Example Output

When the operator clicks on an object 1.5 m in front of the camera, slightly to the right and above centre:

**Terminal log:**
```
[INFO] [rgbd_point_selector]: Published Vector3 coordinates: x=12.3 cm, y=-8.7 cm, z=150.2 cm
```

**On-screen overlay:**
```
XYZ: [12.3, -8.7, 150.2] cm
```

**Published message on `/target_coordinates`:**
```
geometry_msgs/Vector3:
  x: 12.3
  y: -8.7
  z: 150.2
```

To inspect live from terminal:
```bash
ros2 topic echo /target_coordinates
```

---

## How to Run

```bash
# Terminal 1 — launch the camera driver
ros2 launch realsense2_camera rs_launch.py depth_module.enable:=true

# Terminal 2 — run the node
source /opt/ros/humble/setup.bash
python3 rgbd_point_selector.py
```

The `RGB Feed` OpenCV window opens. Click any point in the scene to print and publish its 3D coordinates.

---

## File Location in MkDocs

```
docs/codebase/rgbd-point-selector.md
```

Add to `mkdocs.yml`:

```yaml
- Codebase:
    - Sensor Node (GPS + MQ Sensors): codebase/sensor-node.md
    - GCS Dashboard: codebase/gcs-dashboard.md
    - Map Visualizer: codebase/map-visualizer.md
    - RGBD Point Selector: codebase/rgbd-point-selector.md
```

---

## Known Limitations and TODOs

### 1. Output is in Camera Frame — Not Robot Frame or Map Frame

The published `(X, Y, Z)` is relative to the camera's optical centre. To be useful for navigation or manipulation, this needs to be transformed into either:
- The robot's base frame (`base_link`) — requires a static TF transform from the camera's mounting position and orientation
- The map frame — requires base_link → map TF from the nav stack

Use `tf2_ros.TransformListener` and `tf2_geometry_msgs.do_transform_point()` to apply the transform. This is the single most important missing feature.

### 2. `self.point` Is Not Thread-Safe

`self.point` is written by the OpenCV mouse callback (called from the OpenCV event thread via `cv2.waitKey`) and read by `image_callback` (called from the rclpy executor thread). This is a race condition. In practice it is unlikely to cause problems because the write is an atomic Python object assignment, but a `threading.Lock` around reads and writes would be correct.

### 3. Single Click — Single Publish

`self.point` is reset to `None` after one successful 3D computation. This means:
- One click = one published coordinate
- If depth is zero at the clicked pixel, the node retries on every subsequent frame until it gets a valid depth — potentially publishing a stale coordinate if the scene has changed

If continuous publishing is needed (e.g. tracking a moving target), remove the `self.point = None` reset line.

### 4. No Averaging Over Depth Neighbourhood

Depth sensors are noisy at object edges and in areas of low texture. Reading a single pixel's depth is susceptible to outliers. A more robust approach is to read a small neighbourhood (e.g. 5×5 pixels) around the clicked point and take the median:

```python
y1, y2 = max(0, y_pixel - 2), min(depth_image.shape[0], y_pixel + 3)
x1, x2 = max(0, x_pixel - 2), min(depth_image.shape[1], x_pixel + 3)
patch = self.depth_image[y1:y2, x1:x2]
valid = patch[patch > 0]
if len(valid) == 0:
    return  # no valid depth in neighbourhood
depth_raw = np.median(valid)
```

### 5. `Vector3` is the Wrong Message Type for a 3D Point

`geometry_msgs/Vector3` is semantically a vector (direction + magnitude), not a position. The correct message type for a 3D point in space is `geometry_msgs/PointStamped`, which also carries a header with frame ID and timestamp. Using `PointStamped` would make the output compatible with RViz visualisation and TF transformations out of the box.

### 6. Camera Window Blocks on Slow Frames

`cv2.waitKey(1)` only processes one key event. If frames arrive slower than expected (camera not publishing, high CPU load), the window may become unresponsive. This is inherent to driving OpenCV with `waitKey` inside a ROS callback — there is no separate GUI event loop keeping the window alive between frames.

### 7. No Distortion Correction at Click Time

The code uses `image_rect_raw` (rectified depth) and `image_raw` (unrectified colour). The colour image has not had lens distortion removed. For typical camera setups where distortion is small, this introduces negligible error. If the camera has significant distortion (wide-angle lens), use `image_rect_color` for the RGB stream instead of `image_raw`.
