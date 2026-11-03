# MCWS annotation rules

This page gives the exact rules behind the per-time labels in `labels.csv`. The episode flag `target_event` in
`episodes.csv` says whether the target event occurs in the episode; the per-time label says whether it is
predicted at a decision frame.

## Coordinates and decision frames

All coordinates are in metres in the world frame (right-handed, z up, floor at z = 0). Frame `i` of a video is
shown at `i / 20` s. Each episode has 31 decision frames, 119, 121, ..., 179 (5.95 s to 8.95 s in steps of 0.1 s).
The label of a decision frame is computed from the scene shown in that frame and from the motion before it.

## Label values

- `1`: the target event is predicted within the next 4 s.
- `0`: the target event is not predicted, with the safety margin given below.
- `-1`: a boundary case between the two. Ignore these rows in training and evaluation.

## collision

The velocities of the two agents over the past 0.2 s are extrapolated at constant velocity for 4 s in steps of 0.1 s. A robot is a 1.18 m x 0.68 m rectangle aligned with its heading and a person is a disc of radius 0.25 m. The label is 1 if the two footprints touch, 0 if their minimum distance stays at least 0.25 m, and -1 otherwise.

## intrusion

The hazardous zone is the rectangle x in [-3.7, -0.9] m, y in [-6.9, -5.7] m. The zone lies on a corridor along y = -6.3 m that is crossed by an aisle along x = 0.1 m.

The person's velocity over the past 0.2 s is extrapolated at constant velocity for 4 s in steps of 0.1 s. The person is a disc of radius 0.25 m. The label is 1 if the disc touches the zone, 0 if its minimum distance to the zone stays at least 0.25 m, and -1 otherwise.

## dropped_box

The box starts on a high shelf at (x, y, z) = (4.5, -2.95, 2.92) m, next to the lane of the robot along y = -1.8 m. When the box falls, it falls toward +y into the lane.

The box is unstable if it is still on the shelf (height at least 2.0 m), has moved at least 2 cm toward +y since the start of the episode, and has moved at least 3 mm toward +y in the last 0.25 s. It is stable if it has moved at most 1 cm in total or at most 1 mm in the last 0.25 s. The fall area is a rectangle that extends 0.525 m on each side of the box along x, and along y from 0.275 m behind the box to as far in front of it as the box is high above the floor. The robot, a 1.18 m x 0.68 m rectangle aligned with its direction of travel, is extrapolated with its velocity over the past 0.2 s for 4 s in steps of 0.1 s; it enters the fall area if its footprint touches the area, and it stays clear if its minimum distance to the area is at least 0.25 m. The label is 1 if the box is unstable and the robot enters the fall area, 0 if the box is stable or the robot stays clear (with neither the box nor the robot in between these cases), and -1 otherwise.
