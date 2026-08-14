# Week 7 — Localization and Nav2

[← Week 6: SLAM and map creation](week-06-slam-and-map-creation.md) · [Week index](README.md) · [Repository workflow](../repository-workflow.md) · [Week 8: Behavior-tree inspection mission →](week-08-behavior-tree-inspection-mission.md)

**Time budget:** 8–10 hours

**Primary result:** a repeatable autonomous-navigation benchmark on the frozen Week 6 map.

## Outcomes

By the end of this week, you can:

- distinguish odometry, global localization, planning, control, costmaps, and recovery behaviors;
- initialize and evaluate AMCL on a static occupancy map;
- inspect Nav2 lifecycle state, actions, plans, costmaps, and transforms;
- send navigation goals from RViz and the ROS 2 action CLI;
- measure terminal success, elapsed time, final pose error, and recovery behavior; and
- diagnose bad initial pose, stale localization, unreachable goals, and obstacle-induced replanning.

## Prerequisites

- Week 6 exit criteria met.
- The selected `site_v1.yaml` and `site_v1.pgm` are together in `$HOME/robotics-learning-lab/maps/week06/`.
- The simulator and map use the same world geometry and robot model.
- ROS 2 Jazzy, TurtleBot3 simulation, Nav2, and SLAM Toolbox packages from Week 6.

Do not use a map merely because it looks attractive. A blocked doorway, incorrect origin, or wrong scale will become a navigation failure.

## Public readings

- [Nav2 getting started](https://docs.nav2.org/getting_started/index.html)
- [Nav2 mapping and localization setup](https://docs.nav2.org/setup_guides/sensors/mapping_localization.html)
- [AMCL configuration guide](https://docs.nav2.org/configuration/packages/configuring-amcl.html)
- [Nav2 concepts](https://docs.nav2.org/concepts/index.html)
- [Nav2 Simple Commander API](https://docs.nav2.org/commander_api/index.html)
- [ROS 2 actions tutorial](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Writing-an-Action-Server-Client/Py.html)

## Concepts

### Three pose relationships

- `odom → base_footprint` is locally continuous and may drift.
- `map → odom` corrects that drift using global localization.
- `map → base_footprint` is the best current global pose and may jump when localization is corrected.

AMCL estimates a probability distribution, not a magic exact pose. Its particle cloud should converge around the robot after an initial-pose estimate and informative motion.

### Global and local planning

The global planner finds a route through the global costmap. The controller continuously turns that route into velocity commands using the local costmap. Inflation creates a safety buffer around obstacles. A valid geometric goal can still be unreachable because of footprint, inflation, unknown-space, or controller constraints.

### Lifecycle and action semantics

Nav2 servers are managed lifecycle nodes. A process can be running while its node is inactive. `NavigateToPose` is an action: it has goal acceptance, feedback, cancellation, and a terminal result. Treat goal acceptance and successful arrival as different events.

## Environment and packages

Install any missing packages:

```bash
sudo apt update
sudo apt install -y \
  ros-jazzy-navigation2 \
  ros-jazzy-nav2-bringup \
  ros-jazzy-nav2-simple-commander \
  ros-jazzy-turtlebot3-gazebo \
  ros-jazzy-tf2-tools
```

Prepare each terminal:

```bash
source /opt/ros/jazzy/setup.bash
export TURTLEBOT3_MODEL=waffle
export ROS_DOMAIN_ID=30
export LAB_ROOT="$HOME/robotics-learning-lab"
export MAP_YAML="$LAB_ROOT/maps/week06/site_v1.yaml"
mkdir -p "$LAB_ROOT/bags/week07" "$LAB_ROOT/evidence/week07"
test -r "$MAP_YAML" && echo "Using $MAP_YAML"
```

## Lab: localize, plan, navigate, and measure

### 1. Start the same simulated world

Terminal 1:

```bash
source /opt/ros/jazzy/setup.bash
export TURTLEBOT3_MODEL=waffle ROS_DOMAIN_ID=30
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py
```

Do not start SLAM Toolbox. This week uses a fixed map.

### 2. Launch localization and navigation

Terminal 2:

```bash
source /opt/ros/jazzy/setup.bash
export TURTLEBOT3_MODEL=waffle ROS_DOMAIN_ID=30
export MAP_YAML="$HOME/robotics-learning-lab/maps/week06/site_v1.yaml"
ros2 launch nav2_bringup bringup_launch.py \
  use_sim_time:=true \
  map:="$MAP_YAML" \
  autostart:=true
```

Terminal 3:

```bash
source /opt/ros/jazzy/setup.bash
export ROS_DOMAIN_ID=30
rviz2
```

Set RViz **Fixed Frame** to `map`. Add **Map**, **RobotModel**, **LaserScan**, **ParticleCloud**, **Path**, and both costmaps if they are not already present.

### 3. Prove that the stack is active

Terminal 4:

```bash
source /opt/ros/jazzy/setup.bash
export ROS_DOMAIN_ID=30
ros2 lifecycle get /map_server
ros2 lifecycle get /amcl
ros2 lifecycle get /planner_server
ros2 lifecycle get /controller_server
ros2 lifecycle get /bt_navigator
ros2 action list -t | sort
ros2 action info /navigate_to_pose
ros2 run tf2_ros tf2_echo map base_footprint
```

All listed lifecycle nodes should be `active`, and `/navigate_to_pose` should have a server.

### 4. Set the initial pose and observe convergence

In RViz select **2D Pose Estimate**, click the robot's simulated location on the map, and drag the arrow to its heading. Then:

```bash
ros2 topic echo /amcl_pose --once
ros2 topic hz /particle_cloud
```

Rotate the robot slowly with teleoperation if the particle cloud is broad or multimodal:

```bash
ros2 run turtlebot3_teleop turtlebot3_teleop_key
```

Stop after the particle cloud clusters around one plausible pose. Capture a screenshot.

### 5. Select and record ten safe goals

Use RViz's map grid or **Publish Point** to choose ten free-space poses:

- one nearby goal;
- one goal requiring a turn through a doorway;
- one long route;
- one return goal near the start; and
- one repeat of a prior goal approached from another direction;
- two goals in different rooms or map regions;
- one goal close to, but safely outside, an inflated obstacle;
- one goal with a demanding final heading; and
- one second long route after the stack has been running for several minutes.

Record each as `(x, y, yaw)` in `$LAB_ROOT/evidence/week07/goals.csv`:

```text
name,x_m,y_m,yaw_rad
near,...,...,...
doorway,...,...,...
long,...,...,...
return,...,...,...
repeat,...,...,...
region_a,...,...,...
region_b,...,...,...
near_inflation,...,...,...
heading,...,...,...
long_repeat,...,...,...
```

Do not place a goal in an inflated obstacle or unknown cell.

### 6. Send one goal through the action CLI

Replace the example coordinates with a free pose from your map:

```bash
export GOAL_X=1.0 GOAL_Y=0.5
ros2 action send_goal --feedback \
  /navigate_to_pose nav2_msgs/action/NavigateToPose \
  "{pose: {header: {frame_id: map}, pose: {position: {x: $GOAL_X, y: $GOAL_Y, z: 0.0}, orientation: {x: 0.0, y: 0.0, z: 0.0, w: 1.0}}}}"
```

While it runs, inspect:

```bash
ros2 topic echo /plan --once
ros2 topic hz /cmd_vel
ros2 topic echo /amcl_pose --once
```

Save the terminal result. A successful action result is the evidence of completion; motion alone is not.

### 7. Record a ten-goal benchmark

Start the bag before the first benchmark goal:

```bash
ros2 bag record -o "$LAB_ROOT/bags/week07/nav-benchmark" \
  /amcl_pose /particle_cloud /odom /tf /tf_static \
  /plan /cmd_vel /goal_pose
```

Send each of the ten goals with RViz **Nav2 Goal** or the action CLI. For each trial record:

```text
goal,accepted,terminal_state,elapsed_s,final_x,final_y,position_error_m,recoveries,notes
```

Position error is:

```text
sqrt((final_x - goal_x)^2 + (final_y - goal_y)^2)
```

Use the final `/amcl_pose` sample after the action terminates. Stop the bag after all ten trials.

### 8. Inspect the benchmark data

```bash
ros2 bag info "$LAB_ROOT/bags/week07/nav-benchmark" | tee \
  "$LAB_ROOT/evidence/week07/bag-info.txt"
ros2 topic echo /amcl_pose --once | tee \
  "$LAB_ROOT/evidence/week07/final-amcl-pose.txt"
```

Compute and report:

- success count out of ten;
- median and maximum elapsed time;
- median and maximum final position error;
- number of recoveries or replans observed; and
- any goal whose result disagreed with your visual impression.

## Assignment

Create `navigation-benchmark.md` containing:

1. a diagram showing map server, AMCL, global costmap/planner, local costmap/controller, behavior-tree navigator, and robot interfaces;
2. the ten-goal table and summary statistics;
3. one screenshot containing map, particle cloud, global path, local costmap, and robot;
4. one paragraph explaining why `map → odom` may change while `odom → base_footprint` remains continuous;
5. one parameter hypothesis—for example inflation radius, AMCL particle count, or controller tolerance—and a before/after measurement from changing only that parameter.

Keep a copy of any changed Nav2 parameter file. Never edit files under `/opt/ros/jazzy`.

## Failure injection

Complete all three:

### A. Wrong initial pose

Use **2D Pose Estimate** to place the robot two or more metres from its true simulated location, with a wrong heading. Observe the particle cloud, laser/map mismatch, plans, and motion. Cancel the goal if motion becomes unsafe. Recover by supplying a correct pose and performing a slow rotation.

### B. Unreachable goal

Send a goal inside a wall or outside known map space. Record whether rejection occurs during planning or after retries. Do not label an accepted action as successful until its terminal result arrives.

### C. New obstacle

Move a Gazebo object into a previously clear local path after planning begins. Observe local costmap marking, controller behavior, replanning, and any recovery. Remove the obstruction only after recording the response.

For each injection record the expected response, actual response, safety action, and one useful topic/action diagnostic.

## Measurements and deliverables

Place under `$LAB_ROOT/evidence/week07/`:

- `goals.csv` and the completed result table;
- `navigation-benchmark.md`;
- bag metadata and the benchmark rosbag;
- initial and converged particle-cloud screenshots;
- the parameter file used;
- an action result for at least one success and one failure; and
- three failure-injection reports.

## Objective exit criteria

Advance only when:

- all core Nav2 lifecycle nodes are active;
- AMCL converges and the laser scan visually aligns with the frozen map;
- at least nine of ten valid goals end in `SUCCEEDED`;
- median final position error is at most 0.35 m and no valid success exceeds 0.60 m;
- the robot stops or replans without collision in the obstacle injection;
- unreachable goals terminate without manual process restarts; and
- all claims are tied to action results, topics, bag data, or screenshots.

If the simulator makes these thresholds too easy, tighten position error to 0.20 m or add a narrow-doorway goal.

## Troubleshooting

| Symptom | Diagnostic | Correction |
| --- | --- | --- |
| Map is blank | `ros2 lifecycle get /map_server`; inspect YAML image path | Fix the map pair and relaunch; do not resave blindly. |
| Robot appears away from scan | Inspect `map → base_footprint`, scan frame, and initial pose | Set a correct initial pose; verify no duplicate transform publisher. |
| Particle cloud never converges | Check scan/map overlap and motion | Start nearer the true pose and rotate slowly through feature-rich views. |
| Planner reports no path | Inspect global costmap and goal cell | Move goal into known free space; check footprint and inflation. |
| Robot oscillates near goal | Inspect controller feedback and tolerances | Check localization stability before changing controller parameters. |
| Goal remains pending | `ros2 action info /navigate_to_pose`; lifecycle states | Activate the stack or fix the missing action server. |
| Robot uses stale map | Print `MAP_YAML` and map checksum | Stop all stacks and relaunch exactly one map server with the selected artifact. |

## Next step

Week 8 replaces isolated point-and-click goals with a measurable inspection mission. You will inspect Nav2's behavior tree, run named checkpoints, add retry and timeout policy, and prove that cancellation and failures produce explicit mission results.

[← Week 6: SLAM and map creation](week-06-slam-and-map-creation.md) · [Week index](README.md) · [Week 8: Behavior-tree inspection mission →](week-08-behavior-tree-inspection-mission.md)
