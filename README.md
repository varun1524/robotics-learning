# Robotics Learning

A hands-on, simulation-first path from ROS 2 fundamentals to a dependable
cloud-edge inspection robot.

The curriculum targets an ARM64 Ubuntu 24.04 virtual machine on an
Apple-silicon Mac, using ROS 2 Jazzy, Gazebo Harmonic, RViz, ros2_control,
SLAM Toolbox, Nav2, behavior trees, and public open-source infrastructure.
Physical hardware is optional for the entire course. Weeks 13–14 provide a
physical-robot track and an adversarial-physics Gazebo track with equivalent
software-contract, failure, measurement, and evidence requirements.

## Start here

1. Read the [complete learning plan](docs/learning-plan.md).
2. Review the
   [prerequisites and primary environment](docs/learning-plan.md#primary-development-environment).
3. Prepare the machine with
   [Week 0: Environment and ROS graph](docs/weeks/week-00-environment-and-ros-graph.md).
4. Track progress in the [Week 0–16 assignment index](docs/weeks/README.md).
5. Use the [documentation index](docs/README.md) when you need architectural,
   testing, or artifact guidance.
6. Follow the [repository and lab workflow](docs/repository-workflow.md) so
   every completed assignment is reproducible from a clean checkout.

## What you will build

Over 17 weekly assignments, one simulated mobile robot evolves into a small
but production-shaped system:

- a vendor-neutral primitive robot adapter, with local watchdogs, below a
  separate mission runner;
- mapping, localization, navigation, and behavior-tree missions;
- immutable `MapVersion` geometry and approved `MapRelease` compositions, with
  each mission pinned by release ID and digest;
- durable, replayable cloud-to-robot commands;
- offline telemetry buffering and reconciliation;
- immutable, checksummed map releases pinned to missions;
- independent WebRTC media and object-artifact paths;
- renewable robot identity and fail-closed authorization; and
- a measured inspection capstone that tolerates a WAN outage.

The cloud side sends authenticated, bounded intent. Localization, obstacle
avoidance, motion control, and safety remain local to the robot.

## Primary simulator path

The [default simulator path](docs/learning-plan.md#simulator-gate) is Gazebo
Harmonic with ROS 2 Jazzy. Week 0 includes a graphics gate for the ARM64 VM:

1. use normal Gazebo rendering when it is stable; or
2. run Gazebo headless and visualize its ROS state through RViz.

Webots on the macOS host is an optional visualization-only supplement for Week
0. It is not connected to the course ROS graph, does not replace Gazebo physics,
and cannot satisfy later simulator acceptance criteria by itself.

Do not silently switch tools: record the selected simulator in an architecture
decision and keep the assignment measurements comparable.

## Repository map

| Path | Purpose |
| --- | --- |
| [docs](docs/README.md) | Learning plan, weekly labs, and navigation |
| [docs/weeks](docs/weeks/README.md) | Week 0–16 lessons and assignments |
| [repository workflow](docs/repository-workflow.md) | Tracked source, scratch labs, and clean-checkout gates |
| [src](src/README.md) | Planned ROS packages and adapter boundaries |
| [proto](proto/README.md) | Planned versioned command/event contracts |
| [tests](tests/README.md) | Contract, integration, simulation, and fault tests |
| [decisions](decisions/README.md) | Architecture decision records |
| [bags](bags/README.md) | Local rosbag/MCAP recordings and small manifests |
| [evidence](evidence/README.md) | Measurements, screenshots, and demo manifests |

This initial branch contains curriculum documentation and scaffolding only.
Runtime packages and wire schemas are intentionally deferred to the weekly
assignments.

## Working agreement

Each week ends with:

- a repeatable command or launch file;
- an automated check;
- a deliberate failure;
- a measured result;
- a small evidence manifest; and
- a short explanation of what the result does and does not prove.

Follow the safety checklist before energizing physical motors. Hobby hardware
and these exercises are not safety-certified systems.

## Weekly assignments

| Week | Assignment | Week | Assignment |
| ---: | --- | ---: | --- |
| 0 | [Environment and ROS graph](docs/weeks/week-00-environment-and-ros-graph.md) | 9 | [Edge adapter and telemetry spool](docs/weeks/week-09-edge-adapter-and-telemetry-spool.md) |
| 1 | [Topics, timing, and QoS](docs/weeks/week-01-topics-timing-and-qos.md) | 10 | [Durable command delivery](docs/weeks/week-10-durable-command-delivery.md) |
| 2 | [Services, actions, and parameters](docs/weeks/week-02-services-actions-and-parameters.md) | 11 | [Map versions and releases](docs/weeks/week-11-map-versions-and-releases.md) |
| 3 | [URDF, TF, and RViz](docs/weeks/week-03-urdf-tf-and-rviz.md) | 12 | [WebRTC media and artifacts](docs/weeks/week-12-webrtc-media-and-artifacts.md) |
| 4 | [Gazebo control and watchdogs](docs/weeks/week-04-gazebo-control-and-watchdogs.md) | 13 | [Physical or adversarial-simulation bring-up](docs/weeks/week-13-physical-robot-bring-up.md) |
| 5 | [Sensors, rosbag, and replay](docs/weeks/week-05-sensors-rosbag-and-replay.md) | 14 | [Calibration and navigation tuning](docs/weeks/week-14-calibration-and-navigation-tuning.md) |
| 6 | [SLAM and map creation](docs/weeks/week-06-slam-and-map-creation.md) | 15 | [Identity, security, and lifecycle](docs/weeks/week-15-identity-security-and-lifecycle.md) |
| 7 | [Localization and Nav2](docs/weeks/week-07-localization-and-nav2.md) | 16 | [Capstone cloud-edge inspection](docs/weeks/week-16-capstone-cloud-edge-inspection.md) |
| 8 | [Behavior-tree inspection mission](docs/weeks/week-08-behavior-tree-inspection-mission.md) |  |  |

## Course navigation

[Learning plan](docs/learning-plan.md) ·
[Weekly assignments](docs/weeks/README.md) ·
[Week 0](docs/weeks/week-00-environment-and-ros-graph.md)
