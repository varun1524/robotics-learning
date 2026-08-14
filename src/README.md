# Source Package Boundaries

Runtime code is intentionally absent from the curriculum scaffold. Add packages
only when their week begins, preserving these conceptual boundaries:

| Planned package | Responsibility |
| --- | --- |
| robot_description | URDF/Xacro, meshes, joint limits, and frame conventions |
| robot_sim | Gazebo worlds, controllers, sensors, and simulation launch files |
| canonical_adapter | Week 9 implementation of only `GetState`/`ObserveState`, `Halt`/`Stop`, `NavigateTo`, and `CaptureImage` primitives |
| simulator_adapter | Translation between canonical operations and simulated ROS interfaces |
| safety_supervisor | Command leases, stale-state checks, limits, stop, and watchdogs |
| mission_runner | Owns `ExecuteMission`; composes canonical adapter primitives into local behavior-tree execution, checkpoints, and mission events |
| command_journal | Durable inbound/outbound records, replay, and deduplication |
| telemetry_gateway | Canonical observations, buffering, backpressure, and replay |
| map_service | Immutable `MapVersion` geometry, immutable one-version `MapRelease` compositions, optional release channels, and run pins by release ID plus digest |
| media_gateway | WebRTC session integration and artifact references |
| identity_service | Enrollment, operational identity, renewal, and revocation |

## Dependency rule

Domain and canonical-operation code must not import simulator, vendor SDK, DDS,
or ROS message types. Adapters perform that translation. This makes it possible
to run the same Week 9 contract tests against a simulator, a fake adapter, and
later a physical robot.

The adapter never accepts a mission graph or exposes `ExecuteMission`.
`mission_runner` owns that operation and calls the primitive adapter boundary.
Choose one name from each documented slash pair in a versioned contract; do not
implement `Halt` and `Stop` as different effects.

A release channel is only a mutable selection aid. Resolve it before creating a
run, then persist the selected `MapRelease` ID and digest in the mission bundle;
execution never resolves a channel or pins a bare `MapVersion`.

Motion control, collision avoidance, watchdogs, limits, and stop remain local.
A remote service may request bounded intent but must not become the real-time
motor loop.

## Maps, meshes, and large source assets

Small sanitized source maps and meshes may be tracked under source or test
fixtures, including PGM, PCD, PLY, and STL files. Generated copies belong
under ignored build, work, bags, or evidence paths.

For a necessary asset that is too large for normal Git, commit a small manifest
containing its media type, byte length, SHA-256 digest, license, and a public
fetch or deterministic regeneration command. A clean checkout must verify the
digest before using the asset.

[Repository home](../README.md) · [Test strategy](../tests/README.md)
