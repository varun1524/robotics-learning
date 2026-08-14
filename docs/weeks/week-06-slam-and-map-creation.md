# Week 6 — SLAM and Map Creation

[← Week 5: Sensors, rosbag, and replay](week-05-sensors-rosbag-and-replay.md) · [Week index](README.md) · [Repository workflow](../repository-workflow.md) · [Week 7: Localization and Nav2 →](week-07-localization-and-nav2.md)

**Time budget:** 8–10 hours

**Primary result:** a reviewed 2D occupancy map and a saved SLAM pose graph made from a repeatable simulated survey.

## Outcomes

By the end of this week, you can:

- explain the roles of odometry, laser scans, scan matching, loop closure, and pose-graph optimization;
- verify the required `map → odom → base_footprint → laser` transform chain;
- create a map with SLAM Toolbox while driving a TurtleBot3 simulation;
- save both a portable occupancy grid (`.yaml` plus `.pgm`) and the richer serialized pose graph;
- quantify map completeness and repeatability instead of judging only by appearance; and
- diagnose time, transform, scan, and motion-quality failures.

## Prerequisites

- Weeks 1–5 completed, especially ROS 2 topics, TF, laser scans, and rosbag recording.
- Ubuntu 24.04 with ROS 2 Jazzy Desktop installed.
- A machine capable of running Gazebo and RViz, ideally with 8 GB RAM or more.
- Comfortable opening four terminals and sourcing ROS in each one.

If you use a physical differential-drive robot, substitute its bringup and teleoperation commands. Keep the frame names and safety limits explicit; complete the lab in simulation first.

## Public readings

Read these before the lab:

- [Nav2: Navigating while Mapping](https://docs.nav2.org/tutorials/docs/navigation2_with_slam.html)
- [SLAM Toolbox documentation](https://docs.ros.org/en/jazzy/p/slam_toolbox/)
- [ROS 2 tf2 introduction](https://docs.ros.org/en/jazzy/Concepts/Intermediate/About-Tf2.html)
- [TurtleBot3 simulation guide](https://emanual.robotis.com/docs/en/platform/turtlebot3/simulation/)
- [Nav2 map saver CLI](https://docs.nav2.org/configuration/packages/map_server/configuring-map-saver.html)

## Concepts to learn before touching the controls

### State estimates are not ground truth

Wheel odometry is locally smooth but drifts. A laser scan constrains the robot relative to nearby surfaces. SLAM aligns scans, adds robot poses and constraints to a graph, detects revisited places, and optimizes the graph. A loop closure can therefore move previously drawn walls. That correction is expected, not a display bug.

### The transform contract

This lab expects:

```text
map --SLAM correction--> odom --wheel odometry--> base_footprint --static--> base_scan
```

SLAM Toolbox publishes `map → odom`; the robot or simulator publishes the rest. Do not add a second publisher for any transform merely to silence an error.

### Occupancy-grid values

A map cell is free, occupied, or unknown. The map YAML records resolution, origin, thresholds, and image interpretation. The image alone is insufficient: always preserve the YAML and image together.

### Mapping motion matters

Fast turns, featureless corridors, glass, moving people, stale transforms, and wheel slip all degrade scan matching. A good survey uses slow motion, overlapping observations, deliberate revisits, and a final loop back to the start.

## Environment and packages

Install once:

```bash
sudo apt update
sudo apt install -y \
  ros-jazzy-navigation2 \
  ros-jazzy-nav2-bringup \
  ros-jazzy-slam-toolbox \
  ros-jazzy-turtlebot3 \
  ros-jazzy-turtlebot3-gazebo \
  ros-jazzy-turtlebot3-teleop \
  ros-jazzy-tf2-tools
```

Prepare every terminal:

```bash
source /opt/ros/jazzy/setup.bash
export TURTLEBOT3_MODEL=waffle
export ROS_DOMAIN_ID=30
export LAB_ROOT="$HOME/robotics-learning-lab"
mkdir -p "$LAB_ROOT/maps/week06" "$LAB_ROOT/bags/week06" "$LAB_ROOT/evidence/week06"
```

Confirm package discovery:

```bash
ros2 pkg executables slam_toolbox
ros2 pkg executables nav2_map_server
ros2 pkg prefix turtlebot3_gazebo
```

If an executable is missing, fix package installation before continuing.

## Lab: survey, close the loop, and save the map

### 1. Launch the simulated robot

Terminal 1:

```bash
source /opt/ros/jazzy/setup.bash
export TURTLEBOT3_MODEL=waffle ROS_DOMAIN_ID=30
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py
```

Wait until simulation time advances and the robot is stationary.

### 2. Verify the raw inputs before starting SLAM

Terminal 2:

```bash
source /opt/ros/jazzy/setup.bash
export ROS_DOMAIN_ID=30
ros2 topic list | sort
ros2 topic hz /scan
ros2 topic hz /odom
ros2 topic echo /scan --once
ros2 run tf2_ros tf2_echo odom base_footprint
```

Pass conditions:

- `/scan` and `/odom` update continuously;
- `header.frame_id` on the scan names the laser frame;
- the `odom → base_footprint` transform updates; and
- timestamps advance with simulation time.

Stop `tf2_echo` with `Ctrl-C` after collecting ten seconds of output.

### 3. Launch asynchronous SLAM and RViz

Terminal 2:

```bash
source /opt/ros/jazzy/setup.bash
export ROS_DOMAIN_ID=30
ros2 launch slam_toolbox online_async_launch.py use_sim_time:=true
```

Terminal 3:

```bash
source /opt/ros/jazzy/setup.bash
export ROS_DOMAIN_ID=30
rviz2
```

In RViz:

1. Set **Fixed Frame** to `map`.
2. Add displays for **Map** (`/map`), **LaserScan** (`/scan`), **RobotModel**, and **TF**.
3. Confirm that the scan overlays nearby walls and the robot does not jump while stationary.

From Terminal 3, also verify the completed transform chain:

```bash
ros2 run tf2_ros tf2_echo map base_footprint
```

### 4. Record the mapping evidence

Terminal 4:

```bash
source /opt/ros/jazzy/setup.bash
export ROS_DOMAIN_ID=30
export LAB_ROOT="$HOME/robotics-learning-lab"
ros2 bag record -o "$LAB_ROOT/bags/week06/survey" \
  /scan /odom /map /tf /tf_static /cmd_vel
```

Leave this running until the survey is complete.

### 5. Drive a deliberate survey

Terminal 5:

```bash
source /opt/ros/jazzy/setup.bash
export TURTLEBOT3_MODEL=waffle ROS_DOMAIN_ID=30
ros2 run turtlebot3_teleop turtlebot3_teleop_key
```

Use this survey pattern:

1. Rotate slowly through about 360 degrees at the start.
2. Trace the outer boundary, maintaining laser overlap with the prior view.
3. Enter each opening and alcove rather than mapping only the corridor centerline.
4. Avoid rapid alternating turns.
5. Revisit one distinctive intersection from a different direction.
6. Return close to the starting pose and pause for 10 seconds.

Watch for a loop-closure correction in RViz. The walls should align more consistently after optimization. Stop the robot, then stop rosbag with `Ctrl-C`.

### 6. Save both map representations

Terminal 4:

```bash
source /opt/ros/jazzy/setup.bash
export ROS_DOMAIN_ID=30
export LAB_ROOT="$HOME/robotics-learning-lab"

ros2 run nav2_map_server map_saver_cli \
  -f "$LAB_ROOT/maps/week06/site_v1"

ros2 service call /slam_toolbox/serialize_map \
  slam_toolbox/srv/SerializePoseGraph \
  "{filename: '$LAB_ROOT/maps/week06/site_v1_posegraph'}"

ls -lh "$LAB_ROOT/maps/week06"
sed -n '1,20p' "$LAB_ROOT/maps/week06/site_v1.yaml"
sha256sum "$LAB_ROOT/maps/week06"/* | tee \
  "$LAB_ROOT/evidence/week06/map-sha256.txt"
```

Expected occupancy-map files are `site_v1.yaml` and `site_v1.pgm`. SLAM Toolbox normally creates pose-graph `.posegraph` and `.data` files using the requested base name.

### 7. Reload the saved occupancy map

Stop SLAM Toolbox so it no longer publishes `/map`. In Terminal 2, launch only the saved map:

```bash
source /opt/ros/jazzy/setup.bash
export ROS_DOMAIN_ID=30
export LAB_ROOT="$HOME/robotics-learning-lab"
ros2 run nav2_map_server map_server --ros-args \
  -p yaml_filename:="$LAB_ROOT/maps/week06/site_v1.yaml"
```

In another terminal, transition the lifecycle node and inspect one message:

```bash
source /opt/ros/jazzy/setup.bash
export ROS_DOMAIN_ID=30
ros2 lifecycle set /map_server configure
ros2 lifecycle set /map_server activate
ros2 topic echo /map --once
```

Display `/map` in RViz and compare it with the saved screenshot. This proves the artifact can be consumed independently of the mapping process. Stop the standalone map server before continuing.

### 8. Inspect map dimensions and coverage

Install Pillow if needed and compute reproducible metrics:

```bash
sudo apt install -y python3-pil
python3 - "$LAB_ROOT/maps/week06/site_v1.pgm" <<'PY' | tee \
  "$LAB_ROOT/evidence/week06/map-metrics.txt"
from pathlib import Path
import sys
from PIL import Image

path = Path(sys.argv[1])
image = Image.open(path).convert("L")
hist = image.histogram()
pixels = image.width * image.height
unknown = hist[205]
occupied = sum(hist[:65])
free = sum(hist[250:])
known = pixels - unknown
print(f"file={path}")
print(f"dimensions_px={image.width}x{image.height}")
print(f"pixels={pixels}")
print(f"known_fraction={known / pixels:.4f}")
print(f"free_fraction={free / pixels:.4f}")
print(f"occupied_fraction={occupied / pixels:.4f}")
print(f"unknown_fraction={unknown / pixels:.4f}")
PY
```

The exact gray value for unknown is controlled by the saver, so confirm it by opening the PGM if the counts look implausible. The useful metric is repeatability under the same save settings, not a universal target.

### 9. Replay the survey without driving

Stop Gazebo and SLAM, then run:

```bash
source /opt/ros/jazzy/setup.bash
export ROS_DOMAIN_ID=30
ros2 bag info "$LAB_ROOT/bags/week06/survey"
ros2 bag play "$LAB_ROOT/bags/week06/survey" --clock
```

In another terminal launch SLAM with `use_sim_time:=true`. Confirm that `/map` is produced. A replayed map need not be byte-identical because startup timing and processing order can vary; record whether its major geometry and loop closure agree.

## Assignment

Create a second map, `site_v2`, from a fresh simulation run. Change one survey variable—maximum driving speed, route order, or loop-closure revisit—and keep everything else fixed.

Submit a short comparison containing:

- the variable changed and your prediction;
- `site_v1` and `site_v2` screenshots with the same RViz view;
- checksums and coverage metrics for both maps;
- at least three visible differences or similarities; and
- a recommendation of which map should proceed to navigation testing, with evidence.

## Failure injection

Perform at least two experiments and recover without adding fake transforms:

1. **Missing scan:** stop the simulator or temporarily remap SLAM to an absent topic:

   ```bash
   ros2 launch slam_toolbox online_async_launch.py \
     use_sim_time:=true scan_topic:=/missing_scan
   ```

   Observe that no useful map develops. Use `ros2 topic info /scan -v` and restore the correct topic.

2. **No simulation clock:** launch SLAM with `use_sim_time:=false` while Gazebo publishes simulated timestamps. Capture the transform/message-filter warning, then restart with `true`.

3. **Poor motion:** remap a small area using rapid spins. Compare wall thickness, duplicated edges, and loop-closure behavior with the controlled survey.

Record the symptom, one discriminating diagnostic, the root cause, and the recovery for each injection.

## Measurements and evidence

Keep these under `$LAB_ROOT/evidence/week06/`:

- `map-sha256.txt` and `map-metrics.txt`;
- one screenshot before and one after loop closure;
- `ros2 bag info` output with topic counts and duration;
- a text capture of `ros2 topic hz /scan` and `/odom`;
- your two failure-injection notes; and
- the map-comparison report.

Measure:

- survey duration;
- known-cell fraction;
- map dimensions and resolution;
- number of obvious gaps, doubled walls, and unmapped reachable areas;
- whether the final revisit produced a visible correction; and
- replay success or the reason replay differed.

## Objective exit criteria

Advance only when all are true:

- `/scan`, `/odom`, and the full `map → base_footprint` transform are healthy;
- the chosen map covers every reachable main area of the world;
- there are no unexplained duplicated walls or broken passages on the planned route;
- the occupancy YAML and image load as a pair and their checksums are recorded;
- the saved pair reloads through `nav2_map_server` without SLAM running;
- the serialized pose graph is present;
- the survey bag can be inspected and replayed; and
- two injected failures are diagnosed from evidence rather than trial-and-error changes.

## Troubleshooting

| Symptom | Checks | Likely correction |
| --- | --- | --- |
| `/map` never appears | `ros2 topic hz /scan`; `ros2 run tf2_ros tf2_echo odom base_footprint` | Restore scans or the robot TF chain; use simulation time consistently. |
| `Message Filter dropping message` | Compare scan timestamps, `/clock`, and TF availability | Set every simulation node to `use_sim_time:=true`; reduce CPU load; do not publish duplicate TF. |
| Map rotates or slides while driving | Inspect odometry direction and scan overlay | Correct wheel/laser frames and signs before tuning SLAM. |
| Walls appear twice | Review driving speed and return path | Slow down, increase overlap, and revisit a distinctive area for loop closure. |
| Saved YAML cannot find image | Inspect the YAML `image:` path | Keep YAML and PGM together or update the relative image path. |
| Gazebo is too slow | Check real-time factor and CPU use | Close extra visualizations, lower rendering quality, or run Gazebo headless. |

## Next step

Week 7 freezes the selected occupancy map, starts AMCL against it, and measures autonomous Nav2 goal performance. Mapping and localization are separate operating modes: do not continue editing the map while trying to benchmark localization.

[← Week 5: Sensors, rosbag, and replay](week-05-sensors-rosbag-and-replay.md) · [Week index](README.md) · [Week 7: Localization and Nav2 →](week-07-localization-and-nav2.md)
