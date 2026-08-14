# Week 14 — Calibration and Navigation Tuning

[← Week 13: Physical robot bring-up](week-13-physical-robot-bring-up.md) · [Week index](README.md) · [Repository workflow](../repository-workflow.md) · [Week 15: Identity, security, and lifecycle →](week-15-identity-security-and-lifecycle.md)

**Estimated effort:** 10–14 hours

**Lab mode:** continue the track selected in Week 13:

- **Track A — physical robot:** measured low-speed calibration and navigation in the clear zone; or
- **Track B — adversarial-physics simulation:** ground-truth calibration, one-variable tuning, and blind seeded Gazebo trials under adverse profiles.

Both tracks produce the same calibration, frame, map, navigation, pinning, and evidence contracts. Track B is a complete graduation path but makes no claim about physical dimensions, mechanics, traction, batteries, cutoffs, or real sensor behavior.

## Outcomes

By the end of this week you will be able to:

- calibrate wheel distance, yaw, IMU bias, and sensor extrinsics with repeatable datasets;
- validate the `map → odom → base_link → sensor` frame chain and timestamps;
- build and publish an immutable map baseline with provenance and a digest;
- configure footprint, costmaps, inflation, velocity/acceleration limits, controller, and collision monitor;
- tune one variable at a time against a fixed course and quantitative scorecard; and
- show that changed maps and calibration values do not silently rewrite a running mission’s pinned inputs.

## Prerequisites

- The common Week 13 exit criteria and every criterion for the selected Week 13 track have passed.
- A planar base model with wheel odometry, IMU, and 2D lidar or depth-derived scan.
- The Week 13 canonical integration profile, limits, conformance tests, and immutable input digests.

Track A additionally requires a flat 3 m × 3 m area, measurement tools, three stable soft obstacles, the manufacturer-approved battery/charger, a physical cutoff, and a spotter for every motion run.

Track B additionally requires the Week 13 Gazebo baseline and adverse profiles, a ground-truth pose topic isolated from navigation inputs, deterministic seeds, and a fault injector for transform, scan, localization, and process failures.

Both tracks remain at the Week 13 command limits until the final acceptance run. Track A does not proceed if any physical prerequisite or safety control is absent.

## Public readings

1. [ROS 2 tf2 tutorials](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Tf2/Tf2-Main.html)
2. [Nav2 first-time robot setup](https://docs.nav2.org/setup_guides/index.html)
3. [Nav2 tuning guide](https://docs.nav2.org/tuning/index.html)
4. [Nav2 costmap configuration](https://docs.nav2.org/configuration/packages/configuring-costmaps.html)
5. [Nav2 Collision Monitor](https://docs.nav2.org/tutorials/docs/using_collision_monitor.html)
6. [ROS 2 bag recording and playback](https://docs.ros.org/en/jazzy/Tutorials/Advanced/Recording-A-Bag-From-Your-Own-Node-Py.html)
7. [REP 105: coordinate frames for mobile platforms](https://www.ros.org/reps/rep-0105.html)

## Concepts

| Concept | Meaning |
| --- | --- |
| Intrinsic calibration | Parameters internal to a sensor or drive model, such as wheel scale or camera intrinsics. |
| Extrinsic calibration | Rigid transform between sensor and robot frames. |
| `odom` | Locally continuous frame that may drift over time. |
| `map` | Globally corrected frame that may jump when localization corrects. |
| Footprint | Robot body polygon used for collision checking; it should include fixed protrusions. |
| Inflation | Cost gradient around obstacles that encourages clearance rather than wall hugging. |
| Pinned map input | Exact map version, files, parameters, and digests used by one run. |
| Baseline | Repeatable course and fixed dataset used to compare one tuning change at a time. |
| Ground truth | Simulator-only reference used to score calibration; it must not feed localization, planning, or control. |
| Training profile | Declared dynamics/sensor condition available while tuning. |
| Blind holdout | Seeded adverse condition hidden until parameters and thresholds are frozen. |

## Environment and packages

```bash
sudo apt update
sudo apt install -y \
  ros-jazzy-navigation2 \
  ros-jazzy-nav2-bringup \
  ros-jazzy-nav2-map-server \
  ros-jazzy-nav2-amcl \
  ros-jazzy-nav2-collision-monitor \
  ros-jazzy-slam-toolbox \
  ros-jazzy-robot-localization \
  ros-jazzy-tf2-tools \
  ros-jazzy-rviz2 \
  ros-jazzy-ros-gz \
  python3-numpy python3-pandas python3-matplotlib

mkdir -p ~/robotics-lab/week14/{track-a-physical,track-b-sim,bags,calibration,maps,params,evidence}
cd ~/robotics-lab/week14
source /opt/ros/jazzy/setup.bash
source ~/robot_ws/install/setup.bash
```

Copy your Week 13 robot integration profile and the current navigation parameter file into `params/`. Copy the selected-track record into `evidence/selected-track.txt`; it must remain A or B for the entire week. Never tune the only copy of a working configuration.

## Track A lab: physical calibration and navigation tuning

### 1. Freeze the baseline and safety envelope

Create `evidence/baseline.md` containing:

- robot hardware/firmware and adapter versions;
- wheel diameter and track width from the current configuration;
- sensor model and mounting measurements;
- ROS distribution and navigation package versions;
- maximum linear/angular speeds, accelerations, lease, and stale thresholds from Week 13;
- footprint polygon;
- test-area dimensions and obstacle locations; and
- battery state at the start of each run.

Record package versions:

```bash
ros2 pkg xml nav2_bt_navigator | grep '<version>' | tee evidence/nav2-version.txt
ros2 pkg xml slam_toolbox | grep '<version>' | tee evidence/slam-version.txt
sha256sum params/* | tee evidence/input-digests-before.txt
```

Draw a fixed course using painter’s tape:

1. a 2.00 m straight line;
2. a marked start heading;
3. a 1.00 m square or rectangle for four 90-degree turns;
4. one 0.90 m-wide passage, only if that leaves safe clearance for your footprint; and
5. three fixed obstacles whose positions will not change during tuning.

### 2. Record stationary bias data

Place the robot level and motionless for at least 120 seconds:

```bash
ros2 bag record -o bags/stationary \
  /imu/data /joint_states /odom /tf /tf_static /scan /diagnostics
```

Compute and record:

- mean and standard deviation of angular velocity on all axes;
- linear acceleration magnitude and variance;
- odometry drift while stationary;
- lidar rate and scan-time variance; and
- transform update rate and maximum age.

Do not hide a large bias with a filter. First check mounting, vibration, units, temperature, and driver configuration.

### 3. Calibrate straight-line distance

Run five low-speed 2.00 m trials in each direction. Use a bounded distance operation rather than open-ended teleoperation.

For each trial, record measured tape distance `d_measured` and reported odometry distance `d_odom`. Compute:

```text
distance_scale = d_measured / d_odom
new_wheel_scale = old_wheel_scale * median(distance_scale)
```

Use the median of all valid forward/reverse trials. Apply the correction once, restart the driver, and repeat the dataset. Reject runs with wheel slip or human intervention; do not edit them into compliance.

Save:

```text
calibration/distance-trials.csv
calibration/distance-before.png
calibration/distance-after.png
```

### 4. Calibrate yaw and track geometry

Place heading marks at `0°`, `90°`, `180°`, and `270°`. Run five slow in-place `360°` turns in each direction, stopping between trials.

Measure actual yaw with floor marks, an overhead camera, or an external reference. Compare with odometry and IMU-integrated yaw. For a differential drive, a first correction is:

```text
yaw_scale = reported_yaw / measured_yaw
new_track_width = old_track_width * yaw_scale
```

Confirm the parameter’s convention in the driver before applying this formula; some drivers expose a multiplier with the inverse convention. Change only one yaw-related parameter, rerun both turn directions, and retain raw evidence.

Pass this step only when clockwise and counterclockwise residuals do not indicate a sign error or severe asymmetry.

### 5. Validate sensor extrinsics against geometry

Place the robot square to a long wall at three measured distances. In RViz:

- display `RobotModel`, `TF`, `LaserScan`, and odometry;
- verify the scan lies on the wall when the robot rotates in place;
- verify a fixed obstacle does not “orbit” the robot due to a wrong sensor transform; and
- compare the configured sensor translation/rotation with physical measurements.

Generate a frame report:

```bash
ros2 run tf2_tools view_frames
mv frames.pdf evidence/frames-calibrated.pdf
ros2 run tf2_ros tf2_monitor map base_link | tee evidence/tf-monitor.txt
```

Adjust only the static extrinsic transform. Do not compensate for a bad transform with costmap inflation or map edits.

### 6. Tune state estimation and freshness

Configure your estimator so that:

- wheel odometry contributes planar velocity/pose terms appropriate to the platform;
- IMU yaw rate is in radians per second with correct covariance;
- timestamps use one clock domain;
- measurements older than the configured timeout are rejected or mark state stale; and
- `odom → base_link` has exactly one publisher.

Test by pausing one sensor publisher. Record the time from the last valid message to a stale/degraded indication. Restoring the sensor should require fresh data, not replay of an old latched observation.

### 7. Build an immutable map baseline

Run SLAM at the Week 13 speed limits. Cover each accessible area in both directions and finish with loop closure near the start.

```bash
ros2 launch slam_toolbox online_async_launch.py use_sim_time:=false
ros2 bag record -o bags/mapping \
  /scan /odom /imu/data /tf /tf_static /map /map_metadata
```

Save the map:

```bash
ros2 run nav2_map_server map_saver_cli \
  -f maps/lab-map-v1 --ros-args -p save_map_timeout:=10000.0
sha256sum maps/lab-map-v1.yaml maps/lab-map-v1.pgm \
  | tee maps/lab-map-v1.sha256
```

Create `maps/lab-map-v1.manifest.json` with:

```json
{
  "mapId": "lab-map",
  "mapVersion": 1,
  "frame": "map",
  "units": "meters",
  "sourceBag": "mapping",
  "robotProfile": "<profile-and-version>",
  "slamConfigurationDigest": "sha256:<digest>",
  "artifacts": [
    {"path": "lab-map-v1.yaml", "sha256": "<digest>"},
    {"path": "lab-map-v1.pgm", "sha256": "<digest>"}
  ]
}
```

Copy the map into a read-only release directory. Any wall, occupancy, resolution, origin, or traversability correction produces version 2; do not overwrite version 1. A changed no-go Zone or route can be a new release over the same map version if observed geometry did not change.

### 8. Configure footprint, costmaps, and collision monitor

Measure the robot at its widest fixed configuration. Prefer a polygon footprint:

```yaml
footprint: "[[0.20, 0.16], [0.20, -0.16], [-0.20, -0.16], [-0.20, 0.16]]"
```

Replace the example with measured values. Include fixed bumpers and sensor protrusions. Then configure:

- obstacle source and clearing/marking ranges supported by the sensor;
- global and local costmap resolution;
- robot footprint padding;
- inflation radius larger than the circumscribed body radius;
- transform tolerance based on measured latency, not guesswork;
- velocity and acceleration limits no higher than Week 13; and
- Collision Monitor stop/slow polygons below the navigation controller.

Before enabling autonomous navigation, push a box or cardboard obstacle into the slow and stop polygons while commanding low-speed motion on blocks or in simulation. Verify the filtered velocity reaches zero in the stop zone.

### 9. Establish a navigation baseline

Start localization and navigation with the immutable map:

```bash
ros2 launch nav2_bringup bringup_launch.py \
  map:=$HOME/robotics-lab/week14/maps/lab-map-v1.yaml \
  params_file:=$HOME/robotics-lab/week14/params/nav2-baseline.yaml
```

Set the initial pose in RViz and verify scan-to-map alignment. Send a single goal:

```bash
ros2 action send_goal /navigate_to_pose nav2_msgs/action/NavigateToPose \
  "{pose: {header: {frame_id: map}, pose: {position: {x: 1.0, y: 0.0, z: 0.0}, orientation: {z: 0.0, w: 1.0}}}}" \
  --feedback
```

Record the navigation topics:

```bash
ros2 bag record -o bags/nav-baseline \
  /cmd_vel /cmd_vel_smoothed /odom /amcl_pose /scan \
  /plan /local_plan /global_costmap/costmap /local_costmap/costmap \
  /behavior_tree_log /diagnostics /tf /tf_static
```

Run the same five-goal course ten times. Reset to the same start pose and battery range for each trial.

### 10. Tune one variable at a time

Choose one observed failure, form one hypothesis, change one parameter family, and rerun all ten trials. Use this order:

1. footprint and transforms;
2. sensor freshness and state estimation;
3. costmap obstacle sources and ranges;
4. inflation radius/cost scaling;
5. maximum velocities and accelerations;
6. controller-specific lookahead or critic weights;
7. planner choice/parameters; and
8. recovery behavior.

For every candidate configuration:

```bash
cp params/nav2-candidate.yaml params/archive/nav2-candidate-<n>.yaml
sha256sum params/archive/nav2-candidate-<n>.yaml \
  >> evidence/navigation-config-digests.txt
```

Reject a change that improves average time but worsens collision margin, oscillation, stop behavior, or worst-case failure rate.

### 11. Pin a mission to the map release

Create a run manifest containing:

```json
{
  "runId": "navigation-acceptance-01",
  "mapId": "lab-map",
  "mapVersion": 1,
  "mapRelease": "lab-map-release-1",
  "mapManifestSha256": "<digest>",
  "navigationParametersSha256": "<digest>",
  "robotProfile": "<profile-and-version>",
  "goals": ["A", "B", "C", "D", "HOME"]
}
```

While that run is active, create a new Zone or map release. Confirm the run continues with its pinned release and digest. New runs may select the newer approved release; history must not be rewritten.

## Track B lab: adversarial-physics calibration and navigation tuning

Track B uses simulator ground truth for scoring, never as a localization or control input. Its purpose is to expose model, timing, observability, tuning, and failure-handling mistakes across repeatable adverse conditions—not to imitate every physical effect.

### B1. Freeze the course, profiles, seeds, and acceptance targets

Copy Week 13 `profiles.yaml`, simulator input digests, and the canonical integration profile into `track-b-sim/`. Create an immutable 4 m × 4 m world with the same straight, square, passage, obstacle, and five-goal geometry as Track A. Hash every world/model/controller/profile input:

```bash
cd ~/robotics-lab/week14
sha256sum track-b-sim/worlds/* track-b-sim/models/* \
  track-b-sim/profiles.yaml params/nav2-baseline.yaml \
  | sort | tee evidence/track-b-input-digests.txt
printf '1401\n1402\n1403\n1404\n1405\n' > track-b-sim/training-seeds.txt
```

Before running, write numeric distance, yaw, final-pose, clearance, stale-detection, collision-stop, and reliability targets in `evidence/track-b-acceptance.md`. Reserve ten holdout seeds that the tuner cannot see until B9. Changing a target, world, seed, or profile creates a new acceptance revision and restarts all final trials.

### B2. Isolate and verify simulator ground truth

Publish simulator pose and twist on `/ground_truth/pose` and `/ground_truth/twist` through a scoring-only node. Add an automated dependency test proving no localization, costmap, planner, controller, behavior-tree, or canonical-adapter process subscribes to either topic:

```bash
ros2 topic info /ground_truth/pose -v \
  | tee evidence/ground-truth-endpoints.txt
ros2 topic info /ground_truth/twist -v \
  | tee -a evidence/ground-truth-endpoints.txt
```

Spawn at three known poses and compare the scoring output with the Gazebo entity pose. Position and yaw encoding must agree before calibration begins. Record all tests under simulation time and log the selected profile and seed with every sample.

### B3. Recover known stationary bias without reading truth during estimation

Add deterministic IMU bias/noise fields to the training profiles. Have one person select the injected values and hide them until after the calculation. Record 120 seconds at rest for each of the five training seeds:

```bash
ros2 bag record -o bags/sim-stationary-<seed> \
  /clock /imu/data /joint_states /odom /scan /tf /tf_static \
  /diagnostics /ground_truth/pose
```

Estimate mean bias, variance, odometry drift, scan timing, and transform age from sensor topics only. Then reveal the injected values and report estimate error and uncertainty. Reject a calibration script that directly reads simulator parameters or ground truth while estimating.

### B4. Calibrate distance and yaw against scoring-only pose

Under the nominal profile, run five forward and five reverse 2.00 m operations plus five clockwise and five counterclockwise 360-degree turns. Use the formulas from Track A, replacing tape/heading marks with the scoring-only ground-truth pose. Apply one wheel-scale change, then one track-width/multiplier change, rerunning the complete dataset after each.

Repeat the final calibration under `low_traction` and `payload_offset` without retuning. Report error by direction, profile, and seed. Calibration must correct systematic nominal model error; it must not conceal slip or profile-specific dynamics by using a different scale per condition.

### B5. Recover a deliberately perturbed lidar extrinsic

Create a training copy of the robot model with a hidden, fixed lidar translation/yaw perturbation inside predeclared bounds. Place the base at three poses beside two perpendicular walls. Estimate the static transform using scan geometry, freeze it, then reveal the injected transform.

Report translation and yaw residuals, scan-to-wall residuals, and whether a rotating scan makes fixed walls appear to orbit. Apply the recovered transform through the normal static-TF configuration. Ground truth may score the result but may not supply the answer to the calibration script.

### B6. Validate freshness and immutable map creation

Run `sensor_degraded` across the training seeds. Pause scan, delay IMU, and pause `/clock` separately. State must become stale according to the explicitly chosen clock, navigation must stop or fail safely, and old samples must not regain freshness after restoration.

Build a map from the nominal seeded world using `use_sim_time:=true`, save the source bag and map files, create the same manifest fields as Track A, and hash every artifact. Reload it in a fresh process and score occupied-wall alignment against known world geometry. The world definition and ground truth remain provenance/scoring inputs, not runtime navigation inputs.

### B7. Configure footprint, costmaps, and independent local stopping

Derive footprint dimensions from the versioned simulated collision geometry, not the visual mesh. Configure obstacle sources, inflation, controller limits, and Collision Monitor exactly as Track A. Verify the footprint covers every fixed collision shape.

For all five training seeds, insert a soft simulated obstacle into slow and stop polygons while low-speed motion is commanded. Collision Monitor must command zero without waiting for global replanning. Also invoke the Week 13 simulated actuator inhibit below the adapter; preserve the distinction between local collision filtering, software halt, watchdog, and test-only inhibit.

### B8. Tune one variable at a time on declared training profiles

Use `nominal`, `low_traction`, `payload_offset`, and `sensor_degraded` as the visible training matrix. Run the fixed five-goal course for every profile/seed before tuning. Choose one observed failure, write one hypothesis, change one parameter family in the Track A order, archive/digest it, and rerun the entire matrix.

Do not create per-profile navigation configurations. Retain one final parameter set that preserves zero contacts, stop/freshness bounds, and no false goal success across the matrix. Reject gains in mean time that worsen p95 clearance, oscillation, recovery loops, or safe termination.

### B9. Run ten blind combined-profile holdouts

Freeze the final navigation configuration, map pin, adapter profile, thresholds, and code commit. Reveal the ten reserved seeds and run `combined_holdout` once per seed from the same start pose. Record:

```bash
ros2 bag record -o bags/sim-holdout-<seed> \
  /clock /cmd_vel /cmd_vel_smoothed /odom /amcl_pose /scan \
  /plan /local_plan /behavior_tree_log /diagnostics /tf /tf_static \
  /ground_truth/pose /ground_truth/twist
```

At least 8 of 10 runs must complete all five goals. All 10 must have zero contact, remain inside the boundary, meet local stop/freshness requirements, and report failure rather than false success when a goal cannot be completed. Any code, threshold, map, profile, or parameter change invalidates the holdout and requires ten new unseen seeds.

### B10. Prove pinning and repeatability

Create the Track A run manifest with `robotProfile` identifying Track B plus the simulator profile/seed set. While one run is active, publish a changed map release and a changed calibration profile. The active run must retain its original map, navigation, calibration, and simulator-input digests.

Give the runbook and three unused seeds to another learner. They must launch from stopped processes and complete three consecutive `combined_holdout` runs without undocumented commands. These three handoff runs are additional repeatability evidence; they do not replace the ten-run B9 reliability set.

## Deliberate failure injection

Track A performs every case in simulation first and repeats only the explicitly safe cases at low speed with a spotter:

1. **Bad lidar yaw:** add a temporary `5°` extrinsic error. Pass if wall alignment and mapping metrics degrade detectably; restore the calibrated transform before physical navigation.
2. **Stale scan:** pause scan publication. Pass if navigation stops or fails safely and state reports the sensor stale.
3. **Unexpected obstacle:** introduce a soft obstacle into the Collision Monitor stop polygon. Pass if the local filter commands zero without waiting for replanning.
4. **Changed map during run:** publish map release 2 while a run is pinned to release 1. Pass if the active run’s map/digest is unchanged.
5. **Localization loss:** move the simulated pose or occlude enough scan features to violate localization confidence. Pass if navigation does not claim goal success from odometry alone.

Track B performs all five cases across deterministic seeds, plus low traction, payload/effort limitation, delayed IMU, paused simulation time, scoring-topic isolation, adapter restart, and controller restart. The combined holdout contains at least three simultaneous adverse factors but never changes the frozen acceptance thresholds. A controlled navigation failure is valid only when motion stops locally and the terminal result identifies the fault without false success.

## Assignment

Produce a calibration package and navigation acceptance report for the selected track. The common package contains raw bags, scripts/notebooks, calibration values with uncertainty, immutable map artifacts and manifest, every candidate parameter file/digest, and a final reproducible launch command. The report justifies each retained tuning change with measurements.

Track A includes the physical measurement method, safety envelope, cutoff/spotter record, and ten clean-course trials. Track B includes simulator/world/profile digests, ground-truth isolation test, injected-versus-estimated calibration results, complete training matrix, ten unseen holdouts, and three handoff runs. Neither track may use the scoring source as a navigation input or alter thresholds after final testing starts.

## Measurements

Report median, p95, worst case, and sample count where applicable:

| Metric | Baseline | Final | Acceptance target |
| --- | ---: | ---: | --- |
| 2 m absolute distance error | | | ≤ 2% or predeclared hardware limit |
| 360° absolute yaw error | | | ≤ 3° or predeclared hardware limit |
| Stationary odometry drift over 120 s | | | Declared and repeatable |
| Goal success rate | | | 10/10 on fixed course |
| Final position error | | | ≤ 0.10 m |
| Final yaw error | | | ≤ 5° |
| Minimum obstacle clearance | | | ≥ declared footprint safety margin |
| Time to goal | | | No safety regression for speed gain |
| Recovery count | | | Explained; no oscillatory loop |
| Collision-stop latency | | | Within measured local bound |
| Scan/TF stale detection | | | Within configured timeout |

If your platform cannot meet a suggested numeric target, declare a different target before the final run, explain the physical limitation, and use it consistently. Do not move the goalposts after seeing results.

Track A additionally reports battery range, cutoff availability, spotter, surface, physical measurement uncertainty, and any intervention for each trial.

Track B additionally reports:

| Adversarial-simulation metric | Baseline | Final | Acceptance target |
| --- | ---: | ---: | --- |
| Injected-versus-estimated IMU bias error | | | Predeclared before reveal |
| Injected-versus-estimated lidar translation/yaw error | | | Predeclared before reveal |
| Ground-truth distance/yaw error by profile | | | Same declared calibration bounds |
| Holdout full-course success | | | At least 8/10 |
| Holdout contacts/out-of-bounds | | | 0/10 |
| Holdout false-success reports | | | 0/10 |
| Holdout safe terminal outcomes | | | 10/10 |
| Handoff consecutive successes | | | 3/3 |
| Ground-truth runtime subscribers | | | Scoring process only |

## Evidence and deliverables

- baseline and safety envelope
- stationary, straight-line, turn, mapping, and navigation bags
- calibration CSVs, calculation script/notebook, and before/after plots
- calibrated frame tree and `tf2_monitor` output
- immutable map files, manifest, digest, and release record
- footprint/costmap/collision-monitor configuration
- baseline plus every candidate navigation parameter file and digest
- ten-run raw result table, plots, and final acceptance report
- failure-injection evidence and active-run map-pinning proof

Track A additionally delivers the safety card, physical course measurements/photos, cutoff/spotter record, and ten physical final-run records.

Track B additionally delivers:

- selected-track record and versioned world/model/profile/seed inputs with digests;
- ground-truth publisher validation and subscriber-isolation test;
- hidden injected values, pre-reveal estimates, post-reveal residuals, and uncertainty;
- nominal and adverse training-matrix raw results;
- ten blind holdout bags/rows plus contact, boundary, safe-terminal, and false-success scoring;
- three independent handoff-run records and exact clean-start runbook; and
- a limitations statement that no physical calibration or safety property was established.

## Objective exit criteria

- [ ] Exactly one Week 13 track is carried forward without silently changing its identity or limits.
- [ ] Distance and yaw calibration meet predeclared targets in both directions.
- [ ] Required TF frames form one tree with one publisher per dynamic transform.
- [ ] Sensor extrinsics agree with the selected track’s independent measurement and fixed-world observations.
- [ ] Stale required sensor data cannot be treated as fresh.
- [ ] Map artifacts are immutable, checksummed, and traceable to source bag/configuration.
- [ ] Footprint represents every fixed collision shape in the selected track.
- [ ] Collision Monitor stops a low-speed command independently of the planner.
- [ ] Position, yaw, clearance, and stop-latency targets pass.
- [ ] Only measured, one-at-a-time tuning changes were retained.
- [ ] An active run remains pinned to its original map release and parameter digests.

Track A is complete only when all common criteria and these criteria pass:

- [ ] Calibration values agree with physical measurements and include measurement uncertainty.
- [ ] Every motion run retained the physical cutoff, spotter, Week 13 limits, and clear-zone controls.
- [ ] Final configuration completes 10/10 fixed-course trials without contact, cutoff use, or intervention.
- [ ] Evidence is sufficient for another person with the same hardware revision to repeat the bounded test safely.

Track B is complete only when all common criteria and these criteria pass:

- [ ] Ground truth has no runtime subscriber outside scoring/evidence and never feeds localization, planning, control, or the adapter.
- [ ] Hidden IMU and lidar perturbations are estimated and scored against predeclared error targets only after estimates are frozen.
- [ ] One final parameter set is tested across nominal, low-traction, payload-offset, sensor-degraded, and combined-holdout profiles.
- [ ] At least 8/10 unseen combined-profile runs complete all five goals; all 10 terminate safely with zero contact, zero boundary excursion, and zero false success.
- [ ] Stale scan, localization loss, paused clock, adapter restart, and controller restart reach their declared safe outcomes.
- [ ] Another learner completes three consecutive unused-seed runs from the clean-start runbook.
- [ ] Evidence explicitly states that simulated calibration, collision, traction, stop, and sensor results do not validate physical hardware.

## Troubleshooting

| Symptom | Investigate first |
| --- | --- |
| Map bends or duplicates walls | Timestamp alignment, wheel scale, lidar extrinsics, slip, and loop closure. |
| Robot oscillates near goal | Footprint, minimum velocity, controller tolerances, acceleration limits, and localization noise. |
| Robot hugs walls | Footprint correctness, inflation radius, cost scaling, and global planner behavior. |
| Obstacles remain after removal | Clearing source, raytrace range, sensor marking/clearing settings, and timestamps. |
| Plans through the robot body | Footprint topic/configuration mismatch or wrong costmap namespace. |
| Localization jumps | Scan frame, map quality, initial pose, AMCL parameters, and reflective/dynamic scene features. |
| Good average but occasional unsafe run | Inspect p95/worst case, not only mean; lower speed and find the nondeterministic input. |
| Map save changes between identical runs | Check YAML origin/resolution, image encoding, timestamps in generated metadata, and canonicalization. |
| Track B looks perfect under every profile | Verify profile values actually reach Gazebo and appear in diagnostics; compare ground-truth dynamics before trusting the result. |
| Ground truth improves navigation | Treat as test contamination; remove the subscription and rerun every affected baseline, tuning, and holdout trial. |
| Holdout fails after tuning | Preserve the failure, return to training with a new hypothesis, then allocate ten new unseen seeds after freezing again. |

## Next step

Week 15 adds a durable robot identity, mTLS, least-privilege authorization, renewal, revocation, and recovery. The selected Track A physical robot or Track B simulated robot and its immutable map become protected resources rather than anonymous endpoints.
