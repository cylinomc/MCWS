# MCWS: Multi-Camera Warehouse Safety Dataset

<p align="center"><img src="assets/overview.jpg" width="100%"></p>
<p align="center"><em>One moment of an episode from each task, seen by the four cameras.</em></p>

MCWS is a synthetic video dataset for anticipating safety events in a warehouse. Each episode is a 15 s scene
rendered in a simulated warehouse with Gazebo and recorded by four static cameras whose positions, orientations
and fields of view are known. The dataset has three tasks:

- `collision`: two agents (two mobile robots, or a mobile robot and a person) on crossing paths.
- `intrusion`: a person walks near a hazardous zone.
- `dropped_box`: a box on a high shelf next to the lane of a mobile robot.

Every episode comes with `target_event` (1 if the task's target event occurs in the episode), a scene category,
and per-time labels at 31 decision times that say whether the target event is predicted within the next 4 s. The
target event is a collision between the two agents (`collision`), the person entering the hazardous zone
(`intrusion`), or the box falling onto the robot (`dropped_box`).

In total the dataset has 5000 episodes and 20000 videos.


<p align="center">
  <img src="assets/collision.gif" width="32%"> <img src="assets/intrusion.gif" width="32%"> <img src="assets/dropped_box.gif" width="32%">
</p>
<p align="center"><em>Positive episodes of collision, intrusion and dropped box (one camera each, 7 to 12 s).</em></p>

## Download

The videos (about 14 GB) will be available for download soon. This repository holds the documentation.

## Episodes per task, category and split

### collision

| category | target_event | train | val | test | total |
| --- | --- | ---: | ---: | ---: | ---: |
| `collision` | 1 | 250 | 150 | 100 | 500 |
| `close_pass` | 0 | 125 | 75 | 50 | 250 |
| `wide_pass` | 0 | 125 | 75 | 50 | 250 |
| `pass_stationary` | 0 | 125 | 75 | 50 | 250 |
| **total** |  | 625 | 375 | 250 | 1250 |

### intrusion

| category | target_event | train | val | test | total |
| --- | --- | ---: | ---: | ---: | ---: |
| `enter_zone` | 1 | 250 | 150 | 100 | 500 |
| `divert_or_stop` | 0 | 125 | 75 | 50 | 250 |
| `pass_close` | 0 | 125 | 75 | 50 | 250 |
| `turn_back` | 0 | 125 | 75 | 50 | 250 |
| `stays_clear` | 0 | 250 | 150 | 100 | 500 |
| **total** |  | 875 | 525 | 350 | 1750 |

### dropped_box

| category | target_event | train | val | test | total |
| --- | --- | ---: | ---: | ---: | ---: |
| `fall_on_robot` | 1 | 250 | 150 | 100 | 500 |
| `fall_robot_clear` | 0 | 250 | 150 | 100 | 500 |
| `wobble_no_fall` | 0 | 250 | 150 | 100 | 500 |
| `static_box` | 0 | 250 | 150 | 100 | 500 |
| **total** |  | 1000 | 600 | 400 | 2000 |

## Videos

- Four cameras per episode, `cam0` to `cam3`. Their placement is the same for every episode of a task and is
  given in the task's `cameras.json`.
- H.264 MP4, 1280 x 960 pixels, 20 fps, 300 frames, 15.0 s, no audio.
- The four videos of an episode are synchronized. Frame `i` (counting from 0) has presentation time `i / 20` s
  and shows the scene at that time. All times in this dataset are in these video seconds.
- Each scene also contains background people or robots moving elsewhere in the warehouse. The labels concern only
  the agents described for each task.

## Folder layout

```
MCWS/
  README.md
  LICENSE
  checksums.sha256          SHA-256 of every other file (sha256sum format)
  collision/
    cameras.json
    labels.csv
    videos/<episode_id>/cam0.mp4 ... cam3.mp4
  intrusion/                same layout
  dropped_box/              same layout
  splits/
    train.txt
    val.txt
    test.txt
```

## labels.csv

One row per episode.

| column | meaning |
| --- | --- |
| `episode_id` | for example `collision_0001`; the videos are in `videos/<episode_id>/` |
| `split` | `train`, `val` or `test` |
| `target_event` | 1 if the target event occurs in the episode, 0 otherwise |
| `category` | scene category, see below |
| `5.95` ... `8.95` | per-time label at that decision time: whether the target event is predicted within the next 4 s (1, 0 or -1) |

The 31 decision times run from 5.95 s to 8.95 s in steps of 0.1 s. A per-time label is computed from the scene at
that moment:

- `1`: the target event is predicted within the next 4 s if every agent keeps the velocity it had over the past
  0.2 s.
- `0`: the target event is not predicted, with a safety margin (see the rule of each task).
- `-1`: a boundary case between the two. Ignore these entries in training and evaluation.

A model that decides at time `t` may use the frames with presentation time up to `t`, that is frames `0` to
`round(20 * t)` (for column `5.95`, frames 0 to 119).

Per-time labels describe what can be predicted at that moment, not the final outcome, so they can be 1 in an
episode whose `target_event` is 0. A person walking straight at the zone is labeled 1 until the turn away shows in their
velocity, and a creeping box is labeled 1 while a robot is heading into its fall area, even if the box never falls.

## Tasks, categories and label rules

All coordinates are in metres in the world frame (right-handed, z up, floor at z = 0).

### collision

<img src="assets/collision_categories.jpg" width="100%">

Two agents, either two mobile robots or a mobile robot and a person, are on crossing paths.

**Target event:** a collision between the two agents.

| category | target_event | what happens |
| --- | --- | --- |
| `collision` | 1 | The two agents touch. In half of these episodes one robot starts from rest 1 to 5 s before the contact. |
| `close_pass` | 0 | Both agents move and pass each other with a closest gap of 0.30 to 1.00 m. |
| `wide_pass` | 0 | Both agents move and pass each other with a closest gap of 1.25 to 2.50 m. |
| `pass_stationary` | 0 | A robot stays still a short distance before the crossing point for the whole episode, and the other agent passes it with a closest gap of 0.7 to 2.5 m. |

**Per-time label.** The velocities of the two agents over the past 0.2 s are extrapolated at constant velocity for 4 s in steps of 0.1 s. A robot is a 1.18 m x 0.68 m rectangle aligned with its heading and a person is a disc of radius 0.25 m. The label is 1 if the two footprints touch, 0 if their minimum distance stays at least 0.25 m, and -1 otherwise.

### intrusion

<img src="assets/intrusion_categories.jpg" width="100%">

A person walks near a hazardous zone, the rectangle x in [-3.7, -0.9] m, y in [-6.9, -5.7] m. The zone lies on a corridor along y = -6.3 m that is crossed by an aisle along x = 0.1 m.

**Target event:** the person entering the hazardous zone.

| category | target_event | what happens |
| --- | --- | --- |
| `enter_zone` | 1 | The person walks into the zone, either straight along the corridor or by turning in from the cross aisle. |
| `divert_or_stop` | 0 | The person walks along the corridor toward the zone, then turns into the cross aisle before reaching it, or slows down and stops 1 to 3 m short of it. |
| `pass_close` | 0 | The person walks toward the zone, comes within about 1 m of its edge, then veers away. |
| `turn_back` | 0 | The person walks toward the zone, then turns around and walks back the way they came. |
| `stays_clear` | 0 | The person never heads into the zone: they walk down the cross aisle and stop in the corridor about 1 m from the zone, or wait in the corridor and then walk away from the zone. |

**Per-time label.** The person's velocity over the past 0.2 s is extrapolated at constant velocity for 4 s in steps of 0.1 s. The person is a disc of radius 0.25 m. The label is 1 if the disc touches the zone, 0 if its minimum distance to the zone stays at least 0.25 m, and -1 otherwise.

### dropped_box

<img src="assets/dropped_box_categories.jpg" width="100%">

A box sits on a high shelf at (x, y, z) = (4.5, -2.95, 2.92) m, next to the lane of a mobile robot along y = -1.8 m. When the box falls, it falls toward +y into the lane.

**Target event:** the box falling onto the robot.

| category | target_event | what happens |
| --- | --- | --- |
| `fall_on_robot` | 1 | The box creeps and rocks, then falls into the lane onto the robot, which is driving past or has stopped in front of the shelf. |
| `fall_robot_clear` | 0 | The box creeps, rocks and falls into the lane, but the robot stays out of the fall area: it drives in a lane farther from the shelf, moves over to that lane, or slows down and stops before the area. |
| `wobble_no_fall` | 0 | The box creeps and rocks but stays on the shelf while the robot drives past or stops in front of the shelf. |
| `static_box` | 0 | The box never moves while the robot drives past or stops in front of the shelf. |

**Per-time label.** The box is unstable if it is still on the shelf (height at least 2.0 m), has moved at least 2 cm toward +y since the start of the episode, and has moved at least 3 mm toward +y in the last 0.25 s. It is stable if it has moved at most 1 cm in total or at most 1 mm in the last 0.25 s. The fall area is a rectangle that extends 0.525 m on each side of the box along x, and along y from 0.275 m behind the box to as far in front of it as the box is high above the floor. The robot, a 1.18 m x 0.68 m rectangle aligned with its direction of travel, is extrapolated with its velocity over the past 0.2 s for 4 s in steps of 0.1 s; it enters the fall area if its footprint touches the area, and it stays clear if its minimum distance to the area is at least 0.25 m. The label is 1 if the box is unstable and the robot enters the fall area, 0 if the box is stable or the robot stays clear (with neither the box nor the robot in between these cases), and -1 otherwise.

## cameras.json

One file per task with the four cameras: `id`, `position_m` (x, y, z), `yaw_rad`, `pitch_rad`, `roll_rad`,
`horizontal_fov_deg`, `resolution` (width, height) and `fps`.

The camera looks along its own +x axis, with +y to the left and +z up in the image. Its orientation in the world
is the rotation R = Rz(yaw) Ry(pitch) Rx(roll), so a positive pitch tilts the camera down. The cameras are ideal
pinhole cameras without lens distortion, with square pixels and the principal point at the image centre; the
focal length in pixels is f = (width / 2) / tan(horizontal_fov / 2).

## Splits

`splits/train.txt`, `splits/val.txt` and `splits/test.txt` list episodes as `<task>/<episode_id>`, for example
`collision/collision_0001`. Episodes that share a scene setup (agent speeds, timing and background motion) are
always in the same split, so please use the splits as given.

## Loading example

```python
import csv, json
import cv2  # pip install opencv-python

root, task = "MCWS", "collision"
rows = list(csv.DictReader(open(f"{root}/{task}/labels.csv")))
times = [c for c in rows[0] if c[0].isdigit()]  # decision times in seconds
cameras = json.load(open(f"{root}/{task}/cameras.json"))["cameras"]

ep = rows[0]
cap = cv2.VideoCapture(f"{root}/{task}/videos/{ep['episode_id']}/cam0.mp4")
frames = []
while True:
    ok, frame = cap.read()
    if not ok:
        break
    frames.append(frame)  # 300 frames of 960 x 1280 x 3 (BGR)

for t in times:
    y = int(ep[t])  # 1, 0 or -1 (ignore)
    visible = frames[: round(float(t) * 20) + 1]  # frames a model may use at time t
```

## License

MCWS is released under the Creative Commons Attribution 4.0 International License (CC BY 4.0). See `LICENSE`.

## Acknowledgements and third-party assets

The videos were rendered with Gazebo (gz-sim). The dataset contains rendered videos only; it does not include any
3D model files. The scenes use these third-party models:

- Warehouse shelves, clutter, desk, trash can, pallet jack, walls and roof: AWS RoboMaker Small Warehouse World
  models (https://github.com/aws-robotics/aws-robomaker-small-warehouse-world), MIT-0, obtained via
  https://github.com/belal-ibrahim/dynamic_logistics_warehouse.
- Walking person: Gazebo Fuel model "actor" by Mingfei (https://app.gazebosim.org/Mingfei/fuel/models/actor),
  CC BY 4.0.
- Mobile robot: Gazebo Fuel model "Warehouse Robot" by Open Robotics
  (https://app.gazebosim.org/OpenRobotics/fuel/models/Warehouse%20Robot), CC0 1.0.

## Citation

```bibtex
@misc{mcws,
  title  = {{MCWS}: Multi-Camera Warehouse Safety Dataset},
  author = {TBD},
  year   = {TBD},
  note   = {TBD}
}
```
