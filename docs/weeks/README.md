# Week 0–16 Assignments

Complete the weeks in order. Check a week only after every exit criterion
passes and its evidence manifest explains the measured result.

| Done | Week | Assignment | Main graduation result |
| :---: | ---: | --- | --- |
| [ ] | 0 | [Environment and ROS graph](week-00-environment-and-ros-graph.md) | Working ROS graph and validated simulator path |
| [ ] | 1 | [Topics, timing, and QoS](week-01-topics-timing-and-qos.md) | Measured rate, delay, loss, and stale-data behavior |
| [ ] | 2 | [Services, actions, and parameters](week-02-services-actions-and-parameters.md) | Cancellable, timeout-bounded drive-square action |
| [ ] | 3 | [URDF, TF, and RViz](week-03-urdf-tf-and-rviz.md) | Coherent robot model and frame tree |
| [ ] | 4 | [Gazebo control and watchdogs](week-04-gazebo-control-and-watchdogs.md) | Controlled base that stops after command loss |
| [ ] | 5 | [Sensors, rosbag, and replay](week-05-sensors-rosbag-and-replay.md) | Reproducible sensor-processing result from a bag |
| [ ] | 6 | [SLAM and map creation](week-06-slam-and-map-creation.md) | Saved, reloaded, and evaluated map |
| [ ] | 7 | [Localization and Nav2](week-07-localization-and-nav2.md) | Ten-goal navigation scorecard |
| [ ] | 8 | [Behavior-tree inspection mission](week-08-behavior-tree-inspection-mission.md) | Local inspect-and-return mission with recovery |
| [ ] | 9 | [Edge adapter and telemetry spool](week-09-edge-adapter-and-telemetry-spool.md) | Vendor-neutral state with offline replay |
| [ ] | 10 | [Durable command delivery](week-10-durable-command-delivery.md) | Idempotent commands, durable receipts, and results |
| [ ] | 11 | [Map versions and releases](week-11-map-versions-and-releases.md) | `MapRelease` ID and digest pinned to a mission |
| [ ] | 12 | [WebRTC media and artifacts](week-12-webrtc-media-and-artifacts.md) | Separate live media and object evidence paths |
| [ ] | 13 | [Physical robot bring-up](week-13-physical-robot-bring-up.md) | Blocks-first, low-speed, watchdog-protected base |
| [ ] | 14 | [Calibration and navigation tuning](week-14-calibration-and-navigation-tuning.md) | Calibrated physical navigation scorecard |
| [ ] | 15 | [Identity, security, and lifecycle](week-15-identity-security-and-lifecycle.md) | Renewable identity that fails closed |
| [ ] | 16 | [Capstone cloud-edge inspection](week-16-capstone-cloud-edge-inspection.md) | Auditable inspection surviving a WAN outage |

## Weekly definition of done

Every week contains and requires:

- public reading and an explicit concept check;
- exact lab steps and environment requirements;
- a deliberate failure-injection exercise;
- quantitative measurements;
- sanitized evidence and a repeatable run command;
- objective exit criteria; and
- troubleshooting notes based on observed failures.

Lab scratch directories are not deliverables. Before checking a week, follow
the [repository workflow](../repository-workflow.md) and promote reusable
source, tests, decisions, and sanitized evidence into this repository.

Do not advance merely because a demo worked once. Advance when the acceptance
criteria are repeatable and you can explain what the test does not establish.

[Repository home](../../README.md) ·
[Documentation](../README.md) ·
[Learning plan](../learning-plan.md) ·
[Repository workflow](../repository-workflow.md) ·
[Begin Week 0](week-00-environment-and-ros-graph.md)
