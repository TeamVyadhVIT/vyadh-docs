---
title: "Map Visualizer — ROS2 Interactive Map Viewer"
date: 2026-03-01
author: mahi
categories: [software, gcs-codebase]
tags: [gcs, codebase, ros2, map]
---

## Overview

This is a ROS2 node and desktop GUI application that provides a real-time interactive map viewer for the rover. It runs on the GCS laptop and does the following simultaneously:

1. Subscribes to the **occupancy grid map** (`/map`) published by a SLAM system (e.g. SLAM Toolbox, Cartographer) and renders it as a grayscale image.
2. Subscribes to **odometry** (`/odom`) and draws the rover's live position and heading on the map as a butterfly icon with a directional arrow.
3. Subscribes to the **Nav2 planned path** (`/plan`) and draws it as a blue line on the map.
4. Lets the operator **click on the map** to either place named, colour-coded markers or publish navigation goals directly to Nav2 via `/goal_pose`.
5. Displays everything in a **Tkinter GUI window** that refreshes at 20 Hz.

---

## Architecture Overview

```
┌────────────────────────────────────────────────────────────────┐
│               MAP VISUALIZER                                   │
│                                                                │
│  ┌──────────────────────────────┐  ┌───────────────────────┐   │
│  │   MapVisualizer (ROS2 Node)  │  │   App (Tkinter GUI)   │   │
│  │                              │  │                       │   │
│  │  Subscribers:                │  │  Buttons:             │   │
│  │  ├─ /map → map_callback      │  │  ├─ Color picker      │   │
│  │  ├─ /odom → odom_callback    │  │  ├─ Place Marker      │   │
│  │  └─ /plan → path_callback    │  │  ├─ Clear All Markers │   │
│  │                              │  │  └─ Remove Selected   │   │
│  │  Publishers:                 │  │                       │   │
│  │  └─ /goal_pose               │  │  Click Modes:         │   │
│  │                              │  │  ├─ Marker mode       │   │
│  │  State:                      │  │  └─ Goal mode         │   │
│  │  ├─ map_bgr (OpenCV image)   │  │                       │   │
│  │  ├─ map_info (metadata)      │  │  update_gui() @ 50ms  │   │
│  │  ├─ robot_pose (x,y,yaw)     │◄─┤  └─ spin_once(node)   │   │
│  │  ├─ markers [(x,y,color)]    │  │  └─ render + display  │   │
│  │  └─ path_points [(x,y)]      │  │                       │   │
│  └──────────────────────────────┘  └───────────────────────┘   │
└────────────────────────────────────────────────────────────────┘
         ▲              ▲              ▲              ▼
         │              │              │              │
       /map           /odom          /plan       /goal_pose
         │              │              │              │
   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │  SLAM    │   │  Rover   │   │  Nav2    │   │  Nav2    │
   │ Toolbox  │   │  Driver  │   │ Planner  │   │ BT Nav   │
   └──────────┘   └──────────┘   └──────────┘   └──────────┘
```

---

## Dependencies

| Library | Purpose | Install |
|---|---|---|
| `rclpy` | ROS2 Python client library | Included with ROS2 installation |
| `nav_msgs` | `OccupancyGrid`, `Odometry`, `Path` message types | Included with ROS2 |
| `geometry_msgs` | `PoseStamped` message type for goal publishing | Included with ROS2 |
| `numpy` | Array manipulation for map data | `pip install numpy` |
| `cv2` (OpenCV) | Image processing — resize, rotate, draw, colour convert | `pip install opencv-python` |
| `tkinter` | Desktop GUI framework | `sudo apt install python3-tk` |
| `Pillow` | Convert OpenCV image to format Tkinter can display | `pip install Pillow` |
| `math` | Quaternion-to-yaw conversion, trigonometry | Python stdlib |

### Full Installation

```bash
# ROS2 (assumes ROS2 Humble or later is already installed)
sudo apt install ros-humble-nav2-msgs  # if nav_msgs not already present

# Python deps
pip install numpy opencv-python Pillow

# Tkinter
sudo apt install python3-tk
```

---

## ROS2 Topics

| Topic | Direction | Message Type | Purpose |
|---|---|---|---|
| `/map` | Subscribe | `nav_msgs/OccupancyGrid` | Occupancy grid from SLAM — the map itself |
| `/odom` | Subscribe | `nav_msgs/Odometry` | Rover position and orientation in the map frame |
| `/plan` | Subscribe | `nav_msgs/Path` | Planned path from Nav2 global planner |
| `/goal_pose` | Publish | `geometry_msgs/PoseStamped` | Navigation goal sent to Nav2 on map click |

> **Topic Name Variants**
>
> If Nav2 is configured differently, the path topic may be `/global_plan` instead of `/plan`. Check the Nav2 planner plugin configuration and update the subscription accordingly. The goal topic may also be `/auto_goal` on some configurations — check the comment in `__init__`.
{: .prompt-info }

---

## How It Works — Component by Component

### 1. Coordinate Systems and the Coordinate Conversion Problem

This is the most important concept to understand before reading the rest of the code. There are three coordinate spaces in play:

**Map frame (world coordinates):** The real-world metric coordinate system used by ROS2. The origin is wherever SLAM decided to put it (usually the rover's starting position). Units are metres. Odometry poses and path waypoints are all in this frame. X is forward, Y is left.

**Image pixel coordinates (raw map image):** The `OccupancyGrid` message stores the map as a flat array of cells. When reshaped into a 2D array, row 0 is the bottom of the map in ROS convention (Y increases upward). OpenCV and screen coordinates have Y increasing downward. The image must be vertically flipped before display.

**Display pixel coordinates (the scaled canvas):** The map image is scaled to fit inside the 800×800 viewport while preserving aspect ratio, and centred on a grey canvas. A click on the canvas must be reverse-transformed back through the scale and offset to get the correct raw pixel, which is then reverse-transformed through the rotation and flip to get the map-frame coordinate.

`convert_map_coords_to_pixel(x, y)` — forward transform: map frame → display pixel  
`pixel_to_map_coords(px, py)` — inverse transform: display pixel → map frame

Both functions account for the map origin offset, resolution scaling, Y-flip, and rotation.

---

### 2. Map Callback — `map_callback(msg)`

Called every time a new `OccupancyGrid` is published. Runs the full map render pipeline:

**Step 1 — Decode the occupancy grid:**

The `OccupancyGrid.data` field is a flat 1D array of `int8` values:

| Value | Meaning | Rendered as |
|---|---|---|
| `-1` | Unknown — not yet mapped | Grey (127) |
| `0` | Free space — traversable | White (255) |
| `100` | Occupied — obstacle | Black (0) |

```python
data = np.array(msg.data, dtype=np.int8).reshape((height, width))
img[data == -1] = 127
img[data == 0]  = 255
img[data == 100] = 0
```

**Step 2 — Flip vertically:**

```python
img = cv2.flip(img, 0)
```

ROS2 map data has row 0 at the bottom (Y-up convention). OpenCV has row 0 at the top. Flip brings them into agreement.

**Step 3 — Rotate to account for map origin orientation:**

The map's origin has an orientation quaternion that describes how the map frame is rotated relative to the world. The yaw component of this quaternion is extracted and the image is rotated by the negative of that angle so north stays up:

```python
q = msg.info.origin.orientation
yaw = quaternion_to_yaw(q.x, q.y, q.z, q.w)
self.map_rotation_degrees = math.degrees(yaw)
img = self.rotate_image(img, -self.map_rotation_degrees)
```

`rotate_image` uses `cv2.warpAffine` with a rotation matrix centred on the image, and expands the output canvas to avoid cropping corners.

**Step 4 — Convert to BGR and store:**

```python
self.map_bgr = cv2.cvtColor(img, cv2.COLOR_GRAY2BGR)
```

Stored as BGR (OpenCV's native colour order) so coloured markers and path lines can be drawn on it.

---

### 3. Odometry Callback — `odom_callback(msg)`

Called at 50 Hz. Extracts the rover's position and heading:

```python
x = msg.pose.pose.position.x
y = msg.pose.pose.position.y
q = msg.pose.pose.orientation
yaw = quaternion_to_yaw(q.x, q.y, q.z, q.w)
self.robot_pose = (x, y, yaw)
```

Heading is stored as yaw in radians (rotation about Z axis, counterclockwise positive). The quaternion-to-yaw conversion uses the standard formula:

```python
siny_cosp = 2.0 * (qw * qz + qx * qy)
cosy_cosp = 1.0 - 2.0 * (qy * qy + qz * qz)
yaw = math.atan2(siny_cosp, cosy_cosp)
```

This extracts only the Z-axis rotation (yaw), discarding roll and pitch — correct for a ground robot operating on a flat plane.

---

### 4. Path Callback — `path_callback(msg)`

Called whenever Nav2 publishes a new global plan. Extracts all waypoint positions:

```python
for pose in msg.poses:
    x = pose.pose.position.x
    y = pose.pose.position.y
    self.path_points.append((x, y))
```

Stored as a list of (x, y) map-frame coordinates. Drawn as a connected blue line in `draw_markers_and_robot`.

---

### 5. Drawing — `draw_markers_and_robot()`

Called after any of the three callbacks update state. Composites all visual elements onto a fresh copy of the base map:

```python
img = self.map_bgr.copy()  # Never draw on the base map directly
```

**Markers** — drawn as filled circles of radius 8 at the stored map coordinates, converted to pixel coordinates:

```python
cv2.circle(img, px_py, 8, color, thickness=-1)
```

**Robot position** — drawn as a butterfly icon (`draw_butterfly`) centred on the rover's pixel position, with an orientation arrow (`draw_orientation_arrow`) pointing in the direction of travel.

**Path** — drawn as a series of connected line segments in blue (BGR `(255, 0, 0)`), thickness 3:

```python
cv2.line(img, prev_px, px_py, (255, 0, 0), 3)
```

The result is stored in `self.display_image` — the Tkinter GUI reads from this every 50 ms.

---

### 6. Robot Visualisation

**Butterfly icon — `draw_butterfly(img, center_px, scale=1.5)`**

The robot is rendered as a pixel-art butterfly using a hardcoded pattern of (dx, dy, BGR_color) offsets from the centre pixel. Each entry draws one pixel:

```python
pattern = [
    (0, -2, (120, 0, 120)),   # body
    (-2, -4, (255, 105, 180)), # upper-left wing
    ...
]
for dx, dy, color in pattern:
    x = px + int(dx * scale)
    y = py + int(dy * scale)
    img[y, x] = color
```

The `scale` parameter multiplies all offsets, making the butterfly larger or smaller. Default scale is 1.5.

**Orientation arrow — `draw_orientation_arrow(img, center_px, yaw, length=25)`**

Draws a `cv2.arrowedLine` from the robot centre in the direction of the yaw angle. The end point calculation accounts for the Y-axis inversion:

```python
end_x = int(px + length * math.cos(yaw))
end_y = int(py - length * math.sin(yaw))  # minus because Y increases downward in image
```

---

### 7. Coordinate Conversion — Forward and Inverse

**Forward — `convert_map_coords_to_pixel(x, y)`**

Converts a ROS2 map-frame coordinate (metres) to a pixel position on the stored `map_bgr` image:

```python
# 1. Subtract map origin offset
dx, dy = x - ox, y - oy

# 2. Scale by resolution (metres per cell → pixels)
px = dx / res
py = dy / res

# 3. Flip Y (ROS Y-up → image Y-down)
py = self.map_bgr.shape[0] - py

# 4. Rotate to match the rotated image
px, py = rotate_point(px, py, cx, cy, -self.map_rotation_degrees)

# 5. Clamp to image bounds
px = int(max(0, min(self.map_bgr.shape[1] - 1, px)))
py = int(max(0, min(self.map_bgr.shape[0] - 1, py)))
```

**Inverse — `pixel_to_map_coords(px, py)`**

Reverses each step in the opposite order:

```python
# 1. Undo rotation (rotate in the opposite direction)
px, py = rotate_point(px, py, cx, cy, +self.map_rotation_degrees)

# 2. Undo Y flip
py = self.map_bgr.shape[0] - py

# 3. Scale back to metres and add origin offset
mx = px * res + ox
my = py * res + oy
```

Note that `pixel_to_map_coords` works on raw `map_bgr` pixel coordinates, not on viewport/canvas coordinates. The `on_map_click` handler in the `App` class must first undo the viewport scaling and offset before calling this function.

---

### 8. Viewport Scaling — `get_scaled_image_fit_window()`

The map image can be any size (depending on how large an area SLAM has mapped). It is scaled to fit inside the 800×800 viewport while preserving its aspect ratio, then centred on a grey canvas:

```python
scale = min(w / iw, h / ih)     # uniform scale, limited by whichever dimension is tighter
resized = cv2.resize(...)
canvas = np.full((h, w, 3), 220, dtype=np.uint8)  # grey background
# paste resized image centred on canvas
canvas[y_off:y_off + resized.shape[0], x_off:x_off + resized.shape[1]] = resized
```

---

### 9. Tkinter GUI — `App` Class

The `App` class owns the Tkinter window and all GUI state. It is not a ROS2 node itself — it holds a reference to the `MapVisualizer` node and calls `rclpy.spin_once(self.node, timeout_sec=0)` inside `update_gui()` every 50 ms to process any pending ROS2 callbacks without blocking the Tkinter event loop.

This is the key integration point between ROS2 and Tkinter — `spin_once` processes one batch of incoming messages, updates the node's state, and returns. Tkinter's `root.after(50, self.update_gui)` schedules the next call, creating a cooperative polling loop.

**`update_gui()`** runs every 50 ms and:
1. Calls `spin_once` to process ROS2 messages
2. Gets the scaled display image
3. Converts BGR → RGB (OpenCV stores BGR, PIL/Tkinter expects RGB)
4. Converts to a `PIL.Image`, then to `ImageTk.PhotoImage`
5. Updates the label widget
6. Refreshes the marker list if the count has changed

> **PhotoImage Reference**
>
> `self.photo` must be stored as an instance variable. If it is a local variable, Python's garbage collector will delete it before Tkinter has finished displaying it, causing the image to go blank. This is a classic Tkinter/PIL gotcha.
{: .prompt-warning }

---

### 10. Click Handling — `on_map_click(event)`

When the user clicks on the map canvas, the click position must be reverse-transformed through the viewport scaling before it can be converted to map coordinates:

```python
# 1. Compute the scale and offsets used during rendering
scale = min(w / iw, h / ih)
x_off = (w - iw * scale) / 2
y_off = (h - ih * scale) / 2

# 2. Undo the scale and offset to get raw image pixel
px = int((event.x - x_off) / scale)
py = int((event.y - y_off) / scale)

# 3. Bounds check — click may be on the grey border outside the map
if px < 0 or py < 0 or px >= iw or py >= ih:
    return

# 4. Convert raw pixel to map-frame coordinates
coords = self.node.pixel_to_map_coords(px, py)
```

Based on `self.click_mode`, the resulting map coordinate either:
- Places a marker (`self.node.place_marker_at_coords(mx, my)`)
- Publishes a Nav2 goal (`self.publish_goal(mx, my)`)

---

### 11. Goal Publishing — `publish_goal(x, y)`

Creates and publishes a `PoseStamped` message to `/goal_pose`. Nav2's behaviour tree node listens to this topic and drives the rover to the specified position:

```python
goal = PoseStamped()
goal.header.frame_id = "map"        # Goal is in the map frame
goal.header.stamp = self.node.get_clock().now().to_msg()
goal.pose.position.x = x
goal.pose.position.y = y
goal.pose.position.z = 0.0
goal.pose.orientation.w = 1.0       # Identity quaternion — no preferred heading
```

The orientation is set to identity (`w=1.0`, all others 0), meaning Nav2 will approach the goal from any direction. To send the rover to a goal with a specific final heading, set the orientation quaternion accordingly.

---

### 12. Marker System

Markers are stored in `self.markers` as a list of `(map_x, map_y, bgr_color)` tuples. They persist in memory for the lifetime of the session.

| Function | Description |
|---|---|
| `place_marker_at_robot()` | Reads current `robot_pose` and appends to markers |
| `place_marker_at_coords(mx, my)` | Appends the given coordinates with the current `marker_color` |
| `clear_markers()` | Empties the markers list |
| `remove_marker(index)` | Removes a specific marker by index |

Available colours (stored in BGR format as used by OpenCV):

| Button Label | BGR Value |
|---|---|
| Red | `(0, 0, 255)` |
| Green | `(0, 255, 0)` |
| Blue | `(255, 0, 0)` |
| Yellow | `(0, 255, 255)` |
| Cyan | `(255, 255, 0)` |
| Magenta | `(255, 0, 255)` |
| Orange | `(0, 165, 255)` |

---

## How to Run

### Prerequisites

- ROS2 (Humble or later) sourced in the terminal
- A running SLAM node publishing to `/map`
- A running odometry source publishing to `/odom`
- (Optional) Nav2 running for path visualisation and goal publishing

### Launch

```bash
# Source ROS2
source /opt/ros/humble/setup.bash

# Run the visualizer
python3 map_visualizer.py
```

The Tkinter window opens immediately. The map will appear as soon as the first `/map` message is received. The rover icon appears as soon as the first `/odom` message is received.

---

## File Location in MkDocs

```
docs/codebase/map-visualizer.md
```

Add to `mkdocs.yml`:

```yaml
- Codebase:
    - Sensor Node (GPS + MQ Sensors): codebase/sensor-node.md
    - GCS Dashboard: codebase/gcs-dashboard.md
    - Map Visualizer: codebase/map-visualizer.md
```

---

## Known Limitations and TODOs

### 1. Markers Are Not Persisted

Markers exist only in memory. Closing the window loses all placed markers. To persist them, serialise `self.markers` to a JSON file on close and reload on startup.

### 2. No Heading Control for Published Goals

Goals are published with identity orientation (`w=1.0`). Nav2 will choose the approach heading automatically. To allow the operator to specify a heading, the click handler would need a second interaction (e.g. click-drag) to set the goal orientation before publishing.

### 3. `spin_once` with `timeout_sec=0` May Miss Messages

`rclpy.spin_once(node, timeout_sec=0)` returns immediately whether or not a message is waiting. At 20 Hz GUI refresh, messages arriving between ticks are buffered by rclpy's executor. At 50 Hz odometry this is fine — messages queue and are processed on the next tick. If a message type arrives at a very high rate, the queue could grow. A safer approach is `spin_once(node, timeout_sec=0.05)` but this would block the GUI for up to 50 ms per tick.

### 4. Map Rotation Assumes Map Origin Quaternion Represents Yaw Only

`quaternion_to_yaw` extracts only the Z-axis rotation. If the SLAM system ever produces a map origin with non-zero roll or pitch (unusual but possible on uneven terrain), the rotation correction will be incorrect.

### 5. `rotate_image` Expands the Canvas

`rotate_image` grows the image canvas to avoid cropping corners after rotation. This means the stored `map_bgr` image is larger than the original grid for any non-zero rotation. Coordinate conversions account for this correctly, but be aware that `map_bgr.shape` does not directly correspond to `map_info.width` and `map_info.height`.

### 6. No Undo for Marker Placement

Clicking the map places a marker immediately with no confirmation and no undo. Misclicks require manually selecting and removing the marker from the list. An undo stack (`collections.deque`) would improve usability.

### 7. Goal Mode Does Not Indicate Success or Failure

`publish_goal` fires and forgets. There is no feedback on whether Nav2 accepted the goal, is executing it, or failed. Integrating with the Nav2 action server (`NavigateToPose` action) rather than the simple topic would provide feedback, but is significantly more complex.

### 8. Butterfly Icon is Pixel-Art — Not Resolution-Independent

The butterfly is drawn as individual pixels with hardcoded offsets. At very small map scales (zoomed far out), it becomes invisible or a single dot. At large scales it remains clear. If the map viewport size or scale changes significantly, the butterfly scale parameter may need tuning.
