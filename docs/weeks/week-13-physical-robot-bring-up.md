# Week 13 — Robot Bring-up: Physical or Adversarial Simulation

[← Week 12: WebRTC media and artifacts](week-12-webrtc-media-and-artifacts.md) · [Week index](README.md) · [Repository workflow](../repository-workflow.md) · [Week 14: Calibration and navigation tuning →](week-14-calibration-and-navigation-tuning.md)

**Estimated effort:** 8–12 hours over two sessions

**Lab mode:** choose one graduation path and record it in `evidence/selected-track.txt`:

- **Track A — physical robot:** blocks-first, low-speed bring-up followed by one bounded floor test; or
- **Track B — adversarial-physics simulation:** deterministic Gazebo bring-up under traction, payload, actuator, sensor, process, and timing faults.

Both tracks exercise the same canonical adapter, watchdog, stale-state, restart, measurement, and evidence contracts. Track B completes the course without purchasing hardware, but it is not evidence that any physical robot, battery, cutoff, or mechanical assembly is safe.

> **Track A safety rule:** No command should be able to surprise you. Begin with the driven wheels or legs physically unable to propel the robot, use the lowest practical speed and shortest command lease, keep a physical motor-power cutoff within immediate reach, and have a second person act as spotter for the first floor tests. Track B never weakens or substitutes for these physical requirements.

## Outcomes

By the end of this week you will be able to:

- bring up a robot interface in layers without enabling unconstrained motion;
- prove a local watchdog, command lease, software halt, and either a physical cutoff (Track A) or independently injected simulated actuator inhibit (Track B);
- validate sensor topics, coordinate frames, time, motor direction, and capability reporting;
- expose a physical or simulated platform through the same canonical adapter instead of leaking implementation APIs upward;
- measure stop distance, halt latency, command latency, sensor rate, and stale-state behavior; and
- produce a signed-off bring-up record that another person can reproduce.

## Prerequisites

- Weeks 1–12 complete in simulation, including the Week 9 canonical adapter,
  durable operation IDs, and its fake/simulator contract suite. This week adds
  a new backend to that contract; it does not redesign the boundary.
- ROS 2 Jazzy on Ubuntu 24.04 and the accepted Gazebo model from Week 4.
- A canonical adapter test double capable of recording calls and returning deterministic failures.

Track A additionally requires:

- manufacturer robot, battery, charger, and software manuals for the exact hardware revision;
- a stable workbench, wheel blocks or rated stand, eye protection, and a clear 2 m × 2 m floor area;
- a reachable physical motor-power cutoff or manufacturer E-stop independent of Wi-Fi, ROS, and the application; and
- a second adult spotter for initial floor motion.

Track B additionally requires:

- the Gazebo Harmonic world, robot description, `ros2_control` configuration, and sensor stack accepted in Weeks 4–5;
- permission to create deterministic launch profiles for friction, payload/inertia, actuator effort, sensor rate/latency, and process loss; and
- at least 5 GB of free storage for bags and profile results.

Do not improvise around missing safety hardware. If any Track A prerequisite is absent, select Track B and perform no physical motion. Completing Track B does not waive a future physical bring-up: before later using hardware, complete every Track A safety step and exit criterion.

## Public readings

1. Track A: your robot and battery manufacturer’s current safety, charging, and operating manuals.
2. [ROS 2: lifecycle nodes](https://design.ros2.org/articles/node_lifecycle.html)
3. [ROS 2: topics, services, and actions](https://docs.ros.org/en/jazzy/Concepts/Basic/About-Interfaces.html)
4. [ROS 2: tf2 introduction](https://docs.ros.org/en/jazzy/Concepts/Intermediate/About-Tf2.html)
5. [ros2_control: hardware components](https://control.ros.org/jazzy/doc/ros2_control/hardware_interface/doc/hardware_components_userdoc.html)
6. [Nav2: first-time robot setup](https://docs.nav2.org/setup_guides/index.html)
7. [Gazebo Sim physics parameters](https://gazebosim.org/api/sim/8/physics.html)
8. [Gazebo Sim server configuration](https://gazebosim.org/api/sim/8/server_config.html)

## Concepts

| Term | Practical meaning |
| --- | --- |
| Physical cutoff | Hardware control that removes or inhibits actuator power without relying on the application stack. |
| Simulated actuator inhibit | Track B test hook below the adapter that zeros/disables simulated effort; it tests software layering but is not an E-stop. |
| E-stop | A safety-rated emergency function when supplied by the manufacturer; a software `stop` is not an E-stop. |
| Blocks-first | Initial actuator tests with the robot unable to translate, fall, strike a person, or pull cables. |
| Command lease | Motion authorization that expires unless refreshed. Expiry must stop commanded motion locally. |
| Watchdog | Independent local timer that drives the system to a safe state when updates stop. |
| Software halt | High-priority request to stop application-commanded motion; useful, but not a substitute for the cutoff. |
| Canonical adapter | Week 9's `GetState`/`ObserveState`, `Halt`/`Stop`, `NavigateTo`, and `CaptureImage` primitives mapped to one platform driver. |
| Stale state | A once-valid observation whose age exceeds its declared freshness threshold. |
| Adversarial profile | Versioned, seeded simulator inputs that stress one assumption without changing the acceptance criteria after a run. |

Safety layers should remain independent:

```text
operator decision
  -> canonical policy gate
    -> adapter limits
      -> local motion lease and watchdog
        -> motor controller limits
          -> physical cutoff / manufacturer safety system
```

## Environment and packages

Install the ROS tools used in this lab:

```bash
sudo apt update
sudo apt install -y \
  ros-jazzy-ros-base \
  ros-jazzy-ros2-control \
  ros-jazzy-ros2-controllers \
  ros-jazzy-tf2-tools \
  ros-jazzy-rqt-robot-steering \
  ros-jazzy-teleop-twist-keyboard \
  ros-jazzy-diagnostic-updater \
  ros-jazzy-ros-gz \
  ros-jazzy-gz-ros2-control \
  python3-colcon-common-extensions \
  python3-rosdep \
  python3-yaml
```

Create an evidence directory and record the environment:

```bash
mkdir -p ~/robotics-lab/week13/{track-a-physical,track-b-sim,evidence}
cd ~/robotics-lab/week13
source /opt/ros/jazzy/setup.bash
ros2 doctor --report | tee evidence/ros2-doctor.txt
uname -a | tee evidence/host.txt
printf 'A or B\n' > evidence/selected-track.txt
```

Replace `A or B` with exactly one selected track. Track A also records `lsusb`, the relevant network interface with private addresses redacted, and vendor software versions. If the vendor requires another ROS distribution, use a container or separate host rather than mixing binary packages. Record the exception and adapter boundary.

## Track A lab: physical bring-up from zero motion to a bounded floor test

### 1. Write the safety card before powering anything

Create `evidence/safety-card.md` with:

- robot model, hardware revision, mass, and drive type;
- battery chemistry, nominal voltage, allowed voltage range, and charger model copied from the manufacturer documentation;
- physical cutoff location and exact reset procedure;
- maximum speed for this lab: start at `0.05 m/s` linear and `0.15 rad/s` angular or lower if required by the manufacturer;
- motion lease: `250 ms` for direct velocity tests;
- clear-zone dimensions, floor condition, spotter name, and abort word;
- known pinch, fall, sharp-edge, cable, payload, and tip-over hazards; and
- go/no-go criteria for battery, mechanics, sensors, network, and software.

The numeric speed limits above are conservative lab starting points, not a claim about the hardware’s safe operating envelope. Use the lower of these values and the manufacturer’s limit.

### 2. Inspect the battery and robot unpowered

With the battery disconnected:

1. Photograph all sides and the cutoff location.
2. Check fasteners, wheels/feet, guards, connectors, antennas, sensor mounts, payloads, and loose cables.
3. Inspect the battery for swelling, puncture, heat damage, corrosion, odor, or a damaged lead.
4. Confirm battery model and charger compatibility from their labels and manuals.
5. Confirm the battery is secured and cannot enter a wheel, joint, or sharp edge.
6. Place charging equipment on a noncombustible surface away from exits and combustibles.

Battery rules for the entire course:

- use only the manufacturer-approved charger and settings;
- never charge unattended or inside a sealed robot enclosure unless explicitly designed for it;
- stop and isolate a battery that swells, heats abnormally, hisses, leaks, smells unusual, or is physically damaged;
- never puncture, crush, short, or disassemble a pack;
- transport, store, and dispose of the battery according to its manufacturer and local regulations; and
- do not guess voltage thresholds—copy them into the safety card from authoritative documentation.

If any inspection item fails, mark the run `NO-GO` and do not connect the battery.

### 3. Prove the physical cutoff before software bring-up

Place the robot on a rated stand or blocks so driven parts cannot propel it. Stabilize articulated robots against a fall using manufacturer-approved support; do not suspend them from an unsuitable frame.

1. Put the cutoff in the safe/off state.
2. Connect the battery and power only the non-actuator controller if the hardware permits.
3. Verify that actuators cannot energize while cutoff is safe/off.
4. Enable actuator power using the documented procedure.
5. Immediately operate the cutoff and verify actuator power is removed or motion is inhibited.
6. Reset only after identifying the system state and announcing the reset to the spotter.

Repeat three times. Record activation-to-power-removal time using video at 60 fps or a controller timestamp if available. A software command must not be part of this test.

### 4. Bring up compute and communications with motion disabled

Keep the cutoff safe/off. Start only the compute, network, and sensor layer.

```bash
source /opt/ros/jazzy/setup.bash
source ~/robot_ws/install/setup.bash
export VENDOR_BRINGUP_PACKAGE=replace_with_package_name
export SENSORS_LAUNCH_FILE=replace_with_launch_filename
test "$VENDOR_BRINGUP_PACKAGE" != replace_with_package_name
test "$SENSORS_LAUNCH_FILE" != replace_with_launch_filename
ros2 launch "$VENDOR_BRINGUP_PACKAGE" "$SENSORS_LAUNCH_FILE"
```

Replace placeholders with the documented vendor launch command and save it in `evidence/commands.md`.

Inspect the graph:

```bash
ros2 node list | tee evidence/nodes.txt
ros2 topic list -t | tee evidence/topics.txt
ros2 service list -t | tee evidence/services.txt
ros2 action list -t | tee evidence/actions.txt
ros2 param list | tee evidence/parameters.txt
```

For every sensor, record topic type, expected rate, frame, units, clock source, and stale threshold. Measure rather than assume:

```bash
ros2 topic hz /scan
ros2 topic hz /imu/data
ros2 topic hz /joint_states
ros2 topic echo /battery_state --once
ros2 topic echo /diagnostics --once
```

Unavailable topics are acceptable only when the capability is explicitly absent. A missing required sensor is a no-go for floor motion.

### 5. Validate coordinate frames while stationary

Generate and inspect the frame tree:

```bash
ros2 run tf2_tools view_frames
export LIDAR_FRAME=base_scan
ros2 run tf2_ros tf2_echo base_link "$LIDAR_FRAME"
ros2 run tf2_ros tf2_echo odom base_link
```

Check:

- one connected tree for required frames;
- no duplicate parent for a child frame;
- right-handed axes;
- positive `x` forward, positive `y` left, positive yaw counterclockwise unless your canonical interface documents another convention;
- plausible sensor translation and rotation; and
- transform timestamps that continue updating as expected.

Record a photo of the robot with axes drawn on it and compare physical orientation with reported frames.

### 6. Start the driver as an inactive lifecycle component

Where supported, the hardware node must configure without activating motion:

```bash
ros2 lifecycle list
ros2 lifecycle get /base_driver
ros2 lifecycle set /base_driver configure
ros2 lifecycle get /base_driver
```

Pass if the driver is `inactive`, observations are available as designed, and no velocity command reaches an actuator. Do not auto-activate until your launch configuration has an explicit safety review.

### 7. Run blocks-first direction and polarity tests

Keep the robot supported, clear loose items, and place one person at the physical cutoff. Start rosbag recording before enabling actuators:

```bash
ros2 bag record -o evidence/blocks-first \
  /cmd_vel /odom /joint_states /imu/data /diagnostics /tf /tf_static
```

Enable actuator power. Issue one short lease at the lowest usable value:

```bash
timeout 0.20s ros2 topic pub --rate 20 /cmd_vel geometry_msgs/msg/Twist \
  '{linear: {x: 0.02}, angular: {z: 0.0}}'
```

Then test `-x`, `+yaw`, and `-yaw` individually. After each pulse:

- confirm wheel/joint direction matches the canonical sign;
- confirm odometry sign agrees;
- confirm the command reaches zero when publication stops;
- wait for all motion to stop before the next pulse; and
- inspect diagnostics for faults.

Do not combine linear and angular motion yet. Do not use posture, torque-disable, self-righting, jump, or special-motion commands during bring-up.

### 8. Prove watchdog, software halt, and cutoff separately

With the robot still supported:

1. Publish a lease continuously for two seconds, then kill the publisher. Measure time until commanded velocity becomes zero.
2. Publish again and invoke your high-priority software halt. Measure acknowledgement and physical stop time.
3. Publish again and operate the physical cutoff. Measure stop time independently.
4. Kill the adapter process during a lease. The local robot runtime must stop without a cloud or adapter response.

The tests must show three independent stop mechanisms. If halt waits behind a blocked normal command, fix the control path before continuing.

### 9. Expose the robot through the canonical adapter

Implement or configure the adapter to report:

```yaml
platform_identity:
  model: <model>
  hardware_id: <redacted-id>
  firmware: <version>
  interface_version: <version>
readiness:
  control: true
  sensors: true
  localization: false
  safety: true
capabilities:
  - get_state.v1
  - observe_state.v1
  - halt.v1
  - navigate_to.v1       # not-ready until Week 14 localization passes
  - capture_image.v1     # unsupported if no camera is fitted
limits:
  max_linear_mps: 0.05
  max_angular_radps: 0.15
  command_lease_ms: 250
```

The canonical layer must not import vendor SDK types. Map units, errors, state,
and frames at the adapter boundary. Reject non-finite, out-of-range, stale,
not-ready, or unsupported requests before calling the platform interface. Use
one name from each slash pair in the versioned contract. Do not add `MoveFor` or
`ExecuteMission`: the direct velocity pulses below are driver bring-up tests,
and `mission_runner` remains the sole owner of mission execution.

### 10. First floor motion

Proceed only after all prior steps pass.

1. Remove blocks and place the robot at the center of the clear test area.
2. Put the spotter at the cutoff, not behind or in front of the robot.
3. Establish the abort word and remove trip hazards.
4. Record video and ROS topics.
5. Command `0.05 m/s` or less for `0.25 s`, then stop and inspect.
6. Repeat forward and reverse.
7. Command `0.15 rad/s` or less for `0.25 s` left and right.
8. Query `GetState`/`ObserveState`, then prove that `NavigateTo` returns a
   not-ready result before dispatch while localization readiness is false.
9. End with a deliberate canonical `Halt`/`Stop` and then return the cutoff to
   safe/off.

Do not test autonomous navigation, remote teleoperation, or operation near people this week.

## Track B lab: adversarial-physics simulator bring-up

Track B starts from the same “cannot translate” posture as blocks-first hardware and finishes with bounded motion in an empty simulated test cell. Do not tune away failures by changing acceptance limits between profiles.

Before starting, disconnect or power down every real actuator interface, do not launch a vendor hardware driver, and select an otherwise-unused `ROS_DOMAIN_ID`. Save pre-launch and active node/topic inventories. Stop immediately if a real base driver or unexpected command subscriber appears; a simulator track must never send a test command to physical hardware.

### B1. Freeze the model, seeds, limits, and adverse profiles

Copy the accepted Week 4 robot/world/controller inputs into `track-b-sim/baseline/` and record their hashes:

```bash
cd ~/robotics-lab/week13
sha256sum track-b-sim/baseline/* | sort \
  | tee evidence/simulator-input-digests.txt
printf '1301\n1302\n1303\n1304\n1305\n' \
  > track-b-sim/seeds.txt
```

Create `track-b-sim/profiles.yaml` and implement each value as a launch argument, Xacro parameter, world parameter, controller limit, or deterministic fault-injector input:

```yaml
nominal:
  wheel_friction_scale: 1.0
  payload_mass_kg: 0.0
  payload_x_m: 0.0
  actuator_effort_scale: 1.0
  scan_rate_hz: 10.0
  scan_delay_ms: 0
low_traction:
  wheel_friction_scale: 0.25
  payload_mass_kg: 0.0
  payload_x_m: 0.0
  actuator_effort_scale: 1.0
  scan_rate_hz: 10.0
  scan_delay_ms: 0
payload_offset:
  wheel_friction_scale: 1.0
  payload_mass_kg: 1.0
  payload_x_m: 0.12
  actuator_effort_scale: 0.70
  scan_rate_hz: 10.0
  scan_delay_ms: 0
sensor_degraded:
  wheel_friction_scale: 1.0
  payload_mass_kg: 0.0
  payload_x_m: 0.0
  actuator_effort_scale: 1.0
  scan_rate_hz: 4.0
  scan_delay_ms: 180
combined_holdout:
  wheel_friction_scale: 0.40
  payload_mass_kg: 0.7
  payload_x_m: -0.08
  actuator_effort_scale: 0.75
  scan_rate_hz: 5.0
  scan_delay_ms: 120
```

These values are test inputs, not universal robot limits. Record the exact Gazebo parameters they map to. Verify at startup that the requested profile and seed appear in both diagnostics and the bag metadata. A missing profile application is a failed run.

### B2. Create a virtual blocks-first launch

Add a `supported:=true` launch mode that spawns the base with driven wheels clear of the ground or attaches the chassis to the world with a removable fixed joint. It must start the controller inactive and physics paused.

```bash
ros2 launch <your_sim_package> bringup.launch.py \
  profile:=nominal seed:=1301 supported:=true \
  start_controllers:=false paused:=true
ros2 control list_controllers | tee evidence/sim-controllers-initial.txt
ros2 topic echo /joint_states --once
ros2 topic echo /diagnostics --once
```

Replace `<your_sim_package>` with the Week 4 package and save the final command in `evidence/commands.md`. Pass only if the drive controller is inactive, commanded velocity is zero, and unpausing physics cannot translate the base.

### B3. Validate observations and frames before activation

Unpause physics without activating drive. Inventory nodes, interfaces, rates, frames, units, clock source, and freshness exactly as in Track A steps 4–5. Record `/clock` as simulation time and prove all participating nodes have `use_sim_time:=true`.

```bash
ros2 param get /robot_state_publisher use_sim_time
ros2 topic hz /scan
ros2 topic hz /imu/data
ros2 topic hz /joint_states
ros2 run tf2_tools view_frames
ros2 run tf2_ros tf2_echo odom base_link
```

Pause the scan fault injector for longer than its declared stale threshold. The adapter must mark scan state stale using simulation time; it must not keep updating age from wall time or present the last sample as current.

### B4. Run virtual blocks-first sign and lease tests

Activate the controller while `supported:=true`. Record `/cmd_vel`, `/odom`, wheel states, `/tf`, `/diagnostics`, and the simulator ground-truth pose. Issue the Track A `+x`, `-x`, `+yaw`, and `-yaw` pulses one at a time at the same limits.

Pass when wheel signs and odometry agree with the canonical convention, publication loss reaches zero command within the 250 ms lease, and the ground-truth chassis pose remains fixed by the support. Repeat once with an intentionally inverted left-wheel multiplier and prove the automated conformance test rejects the configuration before removing virtual support.

### B5. Prove three independent simulated stop paths

Implement these paths below the mission layer:

1. local command-lease/watchdog expiry;
2. high-priority canonical `halt`; and
3. a test-only simulated actuator inhibit that disables controller effort without calling the adapter.

With virtual support active, command the wheels and exercise each path separately. Record issue time, zero-command time, zero-wheel-speed time, and outcome. Restarting the controller or adapter after any stop must leave the command at zero and require a new operation ID.

The actuator-inhibit result proves fault-injection wiring and lower-layer independence only. Never label it an E-stop or physical-cutoff test.

### B6. Run bounded ground trials across the profile matrix

Remove virtual support and use an empty 4 m × 4 m world with a visible boundary. For every profile, reset to the same pose and run all five seeds:

```bash
ros2 launch <your_sim_package> bringup.launch.py \
  profile:=<profile> seed:=<seed> supported:=false \
  start_controllers:=true paused:=false
ros2 bag record -o evidence/<profile>-<seed> \
  /clock /cmd_vel /odom /joint_states /imu/data /scan \
  /tf /tf_static /diagnostics /ground_truth/pose
```

For each run, issue driver-level forward and reverse displacement pulses and
left and right turns below the canonical adapter, plus duplicate pulse IDs and
a canonical halt during motion. Keep the Track A command limits. These pulses
exercise platform physics, not a new adapter capability. Pass when the robot
stays in bounds, duplicate IDs create one effect, adverse dynamics change
measured error without changing units or signs, and halt/watchdog behavior
remains bounded.

### B7. Exercise process, timing, and sensor adversity

Using the fake adapter or simulator fault injector:

1. kill the command publisher during `low_traction` motion;
2. kill and restart the adapter during `payload_offset` motion;
3. pause scan delivery under `sensor_degraded` and attempt a new motion operation;
4. pause `/clock`, then resume it without allowing a wall-time watchdog to create an unintended command; and
5. restart the controller after an unfinished operation.

Expected behavior is deterministic: commanded motion reaches zero, the unfinished operation never resumes automatically, required stale state blocks new work, and every recovery requires an explicit new operation ID. Save the state transition and adapter-call log for each fault.

### B8. Run an unrehearsed combined holdout

Have another person choose three seeds not in `seeds.txt`; record them only after freezing code and thresholds. Run `combined_holdout` once per seed. Do not change the profile after seeing results. All three runs must stay within the boundary, preserve correct identity/frames/units, stop within the declared bounds, and produce complete evidence. A motion-completion failure is acceptable only if it terminates safely with the specified fault/result; an unsafe or unreported continuation fails the week.

## Deliberate failure injection

Track A performs these tests first on blocks and then, where explicitly safe, on the clear floor:

1. **Command publisher loss:** terminate the velocity publisher. The local watchdog must stop motion within the declared lease bound.
2. **Adapter crash:** kill the adapter while motion is active. The robot runtime must stop and must not resume when the adapter restarts.
3. **Stale sensor:** pause or disconnect one nonessential sensor. Readiness must degrade and stale data must not be presented as current.
4. **Wrong sign:** in simulation only, invert one wheel or yaw sign. The conformance test must fail before floor use.
5. **Physical cutoff:** operate the cutoff during the lowest-speed floor pulse. The robot must stop without software cooperation.

Never inject a battery fault, short circuit, E-stop wiring fault, fall, collision, or unsafe command as a learning exercise.

Track B performs all seven adversarial cases from B4–B7 plus the combined holdout. At minimum, retain separate results for wrong wheel sign, command loss, adapter crash, controller restart, stale scan, paused simulation time, and simulated actuator inhibit. Inject them only through versioned test hooks; editing result files or database state is not a valid fault.

## Assignment

Produce a versioned `RobotIntegrationProfile` for the selected track and run the
unchanged Week 9 conformance suite against its backend. It must cover platform
identity, readiness, `GetState`/`ObserveState`, `Halt`/`Stop`, `NavigateTo`,
`CaptureImage`, action-handle cancellation, units, frames, limits, stale-state
behavior, duplicate operation IDs, disconnect behavior, and restart behavior.
Until Week 14 localization passes, `NavigateTo` is a before-dispatch not-ready
test; bounded direct-motion trials remain below the canonical boundary.

- Track A adds the manufacturer/battery/cutoff safety record and blocks/floor results.
- Track B adds the simulator input digests, profile-to-physics mapping, seeds, virtual-support proof, profile matrix, and unrehearsed holdout.

Conclude with a limitations statement. Track A must not claim certification or safety beyond the measured configuration. Track B must state that simulated stop, traction, payload, sensing, and actuator results do not validate a physical mechanism.

## Measurements

Both tracks run at least five trials for each stop mechanism and report median and worst case:

| Measurement | Method | Result |
| --- | --- | --- |
| Command-to-motion latency | Command timestamp to first measured velocity | |
| Publisher-loss stop latency | Last lease refresh to zero measured velocity | |
| Software-halt latency | Halt issue to zero measured velocity | |
| Stop distance at 0.05 m/s | Floor marks or motion capture | |
| IMU, odometry, and scan rates | `ros2 topic hz` | |
| Stale-state detection delay | Last observation to stale flag | |
| Adapter-crash stop latency | Process termination to zero measured velocity | |
| Restart behavior | Unintended effects after restart | Must be 0 |
| Duplicate-ID effects | Adapter effect count for one repeated ID | Must be 1 |

Track A also reports:

| Physical measurement | Method | Result |
| --- | --- | --- |
| Physical-cutoff stop latency | Cutoff operation to zero measured velocity | |
| Odometry distance error over 0.25 m | Tape measure versus odometry | |
| Blocks-first wheel-sign trials | Expected versus observed direction | |
| Floor-boundary excursion | Video/floor marks | Must be 0 |

Track B also reports every result by profile and seed:

| Simulator measurement | Method | Result |
| --- | --- | --- |
| Simulated actuator-inhibit latency | Inhibit event to zero wheel speed | |
| Ground-truth distance/yaw error | `/ground_truth/pose` versus odometry | |
| Maximum boundary excursion | Ground-truth pose versus 4 m × 4 m bounds | Must be 0 |
| Stop latency under low traction/payload | Event to zero ground-truth velocity | |
| Profile application coverage | Requested versus diagnostics-observed profile/seed | Must be 100% |
| Holdout safe termination | Three unrehearsed combined-profile seeds | Must be 3/3 |
| Unsupported automatic resume | Adapter and controller restarts | Must be 0 |

## Evidence and deliverables

- `selected-track.txt`, environment report, and exact commands
- environment, node/topic/interface inventory, `frames.pdf`, and diagnostic snapshot
- canonical adapter configuration and integration profile
- automated conformance-test output
- raw per-trial timing measurements rather than aggregates alone
- limitations statement and incident/failure notes, including “none” where applicable

Track A additionally delivers:

- completed safety card and signed go/no-go checklist;
- photographs of blocks/stand, clear zone, axes, battery label, charger label, and cutoff;
- blocks-first and floor bags plus annotated video frame counts; and
- manufacturer-derived battery, charger, operating, and cutoff limits.

Track B additionally delivers:

- baseline model/world/controller files and their digests;
- hardware-isolation record plus pre-launch and active graph inventories;
- `profiles.yaml`, profile-to-Gazebo mapping, training and holdout seeds;
- virtual-support, nominal, each adverse-profile, process-fault, and holdout bags;
- ground-truth-versus-odometry CSV and stop-latency table by profile/seed;
- adapter/state-transition logs proving deduplication, stale-state rejection, and restart behavior; and
- an explicit statement that no physical safety property was tested.

## Objective exit criteria

- [ ] Exactly one track is selected and its immutable inputs, commands, and limits are recorded.
- [ ] Direction signs, frames, units, and odometry signs agree with the canonical contract.
- [ ] Loss of command refresh and adapter process both stop motion locally within the declared bound.
- [ ] Software halt remains available while a normal operation is active.
- [ ] Restart never resumes unfinished motion automatically.
- [ ] Missing or stale required state prevents a new motion operation.
- [ ] The selected backend passes the Week 9 boundary tests and exposes no
      `MoveFor` or `ExecuteMission` capability.
- [ ] A duplicate operation ID creates exactly one adapter effect.
- [ ] Evidence is sufficient for another person to repeat the selected track.

Track A is complete only when all common criteria and these criteria pass:

- [ ] Manufacturer battery and robot safety instructions are available in the lab.
- [ ] The battery passes visual inspection and uses the correct charger.
- [ ] The robot is mechanically secure and cables cannot enter moving parts.
- [ ] A physical cutoff is reachable, independently tested three times, and understood by the spotter.
- [ ] All first actuator tests were completed blocks-first at low speed.
- [ ] The floor test stayed inside the marked clear zone and declared speed limit.
- [ ] The physical limitations statement makes no certification claim.

Track B is complete only when all common criteria and these criteria pass:

- [ ] Virtual support prevents translation while sign, polarity, lease, halt, and inhibit paths are tested.
- [ ] Real actuator interfaces remained isolated and graph evidence shows that only simulation nodes received commands.
- [ ] Every requested adverse profile and seed is confirmed in diagnostics and recorded with input digests.
- [ ] Nominal, low-traction, payload-offset, sensor-degraded, and combined-holdout profiles all preserve canonical identity, units, signs, and bounds.
- [ ] Watchdog, halt, adapter crash, controller restart, and simulated actuator inhibit produce bounded zero-motion outcomes without automatic resume.
- [ ] Stale scan and paused-clock cases reach their declared deterministic state and do not authorize new unsafe work.
- [ ] Five seeds per advertised profile and three unrehearsed holdout seeds stay within the simulated boundary and produce complete results.
- [ ] The evidence explicitly states that Track B proves no physical battery, cutoff, mechanical, traction, or collision-safety property.

## Troubleshooting

| Symptom | Safe response |
| --- | --- |
| Robot moves when the driver configures | Operate cutoff, disconnect actuator power, and remove auto-activation or latched command state. |
| Wheels spin opposite the command | Stop, return to blocks, correct adapter polarity, and rerun all sign tests. |
| Motion continues after publisher exits | Operate cutoff; implement or enable the local watchdog before further tests. |
| Halt is delayed by another call | Separate the halt path or driver worker; never queue halt behind normal work. |
| TF tree is disconnected or jumps | Keep motion disabled; fix static transforms, timestamps, and duplicate publishers. |
| Sensor rate is intermittent | Check power, bandwidth, QoS, USB topology, and vendor diagnostics before moving. |
| Battery heats, swells, smells, or alarms | Power down without handling a hot pack unnecessarily, isolate the area, and follow manufacturer/emergency guidance. |
| Robot oscillates or tips | Operate cutoff, support the robot, reduce limits, and do not tune navigation until Week 14. |
| Requested simulator profile has no measured effect | Stop the run; expose applied values in diagnostics and verify the Xacro/world/controller mapping before collecting evidence. |
| Simulation resumes an old command after reset | Clear latched controller input and operation state; require a new operation ID after every reset. |
| `/clock` pauses but watchdog uses wall time unexpectedly | Declare the intended clock for each timer and test both paused simulation and process liveness explicitly. |

## Next step

Week 14 calibrates odometry and sensor geometry, creates a versioned map baseline, and tunes navigation with repeatable metrics. Continue with the same selected track. Track A keeps the Week 13 physical speed limits until measured testing justifies a change; Track B keeps the same command envelope while adding blind adverse-profile trials.
