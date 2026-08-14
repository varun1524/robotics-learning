# Hands-on Robotics Learning Plan

This is a simulation-first, measurement-driven curriculum for learning modern
mobile robotics and the cloud-edge engineering around it. The path starts with
ROS graph literacy, develops navigation and local mission execution, and ends
with an auditable inspection system that continues safely through a network
outage.

The course assumes six to eight focused hours per week for seventeen weeks.
Physical hardware is optional for the entire course and deliberately delayed
until the simulator, frames, control, sensors, mapping, and navigation
foundations are repeatable. Weeks 13–14 offer either Track A on a physical robot
or Track B in deterministic adversarial-physics simulation; both lead to the
same software contracts and capstone.

## Graduation outcome

By the end, one robot can:

- enroll with a renewable operational identity;
- advertise the primitive `GetState`/`ObserveState`, `Halt`/`Stop`,
  `NavigateTo`, and `CaptureImage` capabilities through a vendor-neutral
  adapter;
- receive authenticated, expiring, idempotent mission intent;
- durably acknowledge receipt without confusing receipt with execution;
- execute a pinned, immutable inspection mission locally;
- navigate against a pinned `MapRelease` ID and digest;
- stop locally when a command lease or controller heartbeat expires;
- buffer observations while disconnected and replay them after reconnect;
- stream live camera video over WebRTC;
- upload checksummed images through an object-artifact path; and
- produce an audit timeline from request through evidence and terminal result.

## System mental model

The remote side governs identity, fleet intent, mission versions, map releases,
observability, evidence metadata, and analysis. The robot side owns real-time
control, localization, obstacle avoidance, motion limits, watchdogs, stop, and
physical interlocks.

| Layer | Responsibility |
| --- | --- |
| Robot controller | Drivers, motor loop, localization, collision avoidance, physical stop |
| Robot agent | Primitive canonical adapter, validation, leases, `mission_runner`, offline buffers |
| Command path | Durable bidirectional messages, replay, deduplication, receipts, results |
| Mission path | Immutable definitions, approval, dispatch, local attempts, audit |
| Map path | Immutable `MapVersion` geometry, immutable approved `MapRelease`, optional channel, run pin |
| Telemetry path | Operational observations, retry spool, analytical retention |
| Media path | Short-lived WebRTC sessions and live tracks |
| Artifact path | Maps, images, bags, models, logs, checksums, and provenance |

### Principles to retain

1. Send bounded high-level intent remotely; keep real-time and safety decisions
   local.
2. Place simulator, ROS, DDS, and vendor details behind adapters.
3. Treat delivery as at least once and make physical effects idempotent.
4. Distinguish transport acknowledgement, durable peer receipt, and application
   result.
5. Let the robot initiate remote connectivity rather than requiring unsolicited
   inbound access.
6. Separate bootstrap enrollment from renewable runtime identity.
7. Make mission versions immutable and model each run as one invocation.
8. Package observed native and canonical geometry as an immutable `MapVersion`;
   package exactly one version plus approved alignment, overlays, policies, and
   routes as an immutable `MapRelease`.
9. Keep commands, operational telemetry, live media, and large artifacts on
   independent paths.
10. Preserve stable IDs, versions, checksums, timestamps, and provenance so a
    result can be explained later.

## Primary development environment

The default host is an ARM64 Ubuntu 24.04 VM running on an Apple-silicon Mac.
Use the Mac for editing and version control; run the ROS graph and simulator
inside Ubuntu.

Core tools:

- ROS 2 Jazzy Jalisco;
- Gazebo Harmonic;
- RViz 2 and tf2;
- ros2_control and gz_ros2_control;
- rosbag2, preferably with MCAP where supported;
- SLAM Toolbox;
- Nav2;
- BehaviorTree.CPP;
- Python for orchestration and C++ where ROS integration or timing benefits;
- pytest, launch testing, colcon, Git, and containers; and
- public local services such as an MQTT broker, SQLite, MinIO, and a WebRTC SFU.

ROS 2 Jazzy and Gazebo Harmonic are a mature pairing on Ubuntu 24.04. The
[Gazebo installation guide](https://gazebosim.org/docs/harmonic/install_ubuntu/)
and [ROS 2 Jazzy documentation](https://docs.ros.org/en/jazzy/) are the
authoritative setup sources.

### Simulator gate

Week 0 records exactly one supported course-simulation mode:

1. Gazebo with accelerated rendering;
2. software-rendered Gazebo; or
3. headless Gazebo with RViz visualization.

Webots on the macOS host may be used for optional visual exploration when VM
graphics are poor. It is visualization-only in this curriculum: it is not
connected to the Ubuntu ROS graph, does not replace Gazebo physics, and does not
satisfy a Gazebo or ROS acceptance criterion.

Do not spend multiple weeks debugging VM rendering. Physics, interfaces,
measurements, and fault behavior matter more than visual polish.

## How to work each week

Use this six-to-eight-hour rhythm:

- 60–90 minutes for official reading and a concept note;
- three to four hours for the vertical lab;
- one hour to deliberately break the system;
- thirty minutes to measure and graph the result; and
- thirty minutes to update the evidence manifest and record a short demo.

Every week ends with five durable artifacts:

1. a repeatable command or launch file;
2. an automated check;
3. one quantitative result;
4. one injected failure and observed recovery; and
5. a short explanation of what the evidence does not prove.

Weekly lab directories are disposable. Follow the
[repository and lab workflow](repository-workflow.md): promote reusable source,
tests, decisions, and sanitized evidence into the tracked repository, then pass
the clean-checkout gate before advancing.

## Weekly roadmap

| Week | Assignment | Graduation result |
| ---: | --- | --- |
| 0 | [Environment and ROS graph](weeks/week-00-environment-and-ros-graph.md) | Working graph and selected simulator mode |
| 1 | [Topics, timing, and QoS](weeks/week-01-topics-timing-and-qos.md) | Measured rate, delay, loss, and freshness |
| 2 | [Services, actions, and parameters](weeks/week-02-services-actions-and-parameters.md) | Cancellable and bounded drive-square action |
| 3 | [URDF, TF, and RViz](weeks/week-03-urdf-tf-and-rviz.md) | Coherent robot model and frame tree |
| 4 | [Gazebo control and watchdogs](weeks/week-04-gazebo-control-and-watchdogs.md) | Simulated base stops after command loss |
| 5 | [Sensors, rosbag, and replay](weeks/week-05-sensors-rosbag-and-replay.md) | Reproduced result from recorded data |
| 6 | [SLAM and map creation](weeks/week-06-slam-and-map-creation.md) | Saved, reloaded, and evaluated map |
| 7 | [Localization and Nav2](weeks/week-07-localization-and-nav2.md) | Ten-goal navigation scorecard |
| 8 | [Behavior-tree inspection mission](weeks/week-08-behavior-tree-inspection-mission.md) | Local inspect-and-return mission |
| 9 | [Edge adapter and telemetry spool](weeks/week-09-edge-adapter-and-telemetry-spool.md) | Canonical adapter implementation and offline replay |
| 10 | [Durable command delivery](weeks/week-10-durable-command-delivery.md) | Idempotent commands, receipts, and results |
| 11 | [Map versions and releases](weeks/week-11-map-versions-and-releases.md) | `MapRelease` ID and digest pinned to a run |
| 12 | [WebRTC media and artifacts](weeks/week-12-webrtc-media-and-artifacts.md) | Separate live and object-data paths |
| 13 | [Physical or adversarial-simulation bring-up](weeks/week-13-physical-robot-bring-up.md) | Watchdog-protected selected-track base |
| 14 | [Calibration and navigation tuning](weeks/week-14-calibration-and-navigation-tuning.md) | Selected-track navigation scorecard |
| 15 | [Identity, security, and lifecycle](weeks/week-15-identity-security-and-lifecycle.md) | Renewable identity that fails closed |
| 16 | [Capstone cloud-edge inspection](weeks/week-16-capstone-cloud-edge-inspection.md) | Auditable mission surviving network loss |

## Six production-shaped POCs

The weeks combine into six larger proofs of concept.

### POC 1: Canonical adapter and local safety

In Week 9, implement a small primitive interface for `GetState` or streaming
`ObserveState`, `Halt` (also called `Stop` in some profiles), `NavigateTo`, and
`CaptureImage`. Back it with a simulator adapter and a deliberately simple fake
adapter. Pick one name from each slash pair in the versioned code contract and
do not give the aliases different semantics.

`ExecuteMission` is not an adapter capability. It belongs to `mission_runner`,
which executes the behavior tree above the adapter by composing those
primitives. Cancellation is part of the `NavigateTo` action handle rather than
a fifth robot capability.

The canonical package must not import simulator, vendor, DDS, or ROS message
types. Adapters advertise capabilities and perform translation. The safety
supervisor enforces command freshness, velocity limits, a motion lease,
preemptive stop, and restart-safe behavior.

Pass conditions:

- the same contract tests pass against both adapters;
- an unsupported capability fails before dispatch;
- stale state is explicitly stale;
- a duplicate request causes one physical effect;
- lease expiry produces a measured stop; and
- restarting a process never resumes unfinished motion.

### POC 2: Durable bidirectional commands

Create independent inbound and outbound journals for each robot. A message has
a stable producer ID, robot generation, monotonic endpoint sequence, type
version, creation time, optional expiry, correlation ID, and bounded payload.

Submission succeeds when the local outbound journal commits. A delivery receipt
means the peer durably appended and deduplicated the record. An application
result is a separate correlated message.

Pass conditions:

- reconnect replays every missing record;
- repeated delivery creates one effect;
- the same ID with different content is rejected;
- unknown history or a lower sequence enters explicit recovery;
- an expired journal record is retained but its command is rejected;
- a full journal rejects new work rather than deleting outstanding work; and
- oversized data uses an artifact reference.

### POC 3: Enrollment and operational identity

Use a local certificate authority to model a registration profile, a
pre-enrolled robot, one-use bootstrap grant, locally generated key and
certificate request, renewable operational certificate, revocation, and
recovery.

Pass conditions:

- the private key never leaves the robot agent;
- bootstrap credentials cannot call runtime services;
- the certificate is bound to the intended stable robot identity;
- the wrong scope is rejected despite a well-formed payload;
- rotation uses a new key and bounded overlap;
- revocation denies reconnect and closes an active session; and
- recovery preserves the stable identity.

### POC 4: Immutable mission bundle

Build a deterministic mission with Navigate, CaptureImage, Navigate,
CaptureImage, and Return steps. Validate and sign an immutable version, then
download the complete execution bundle before starting.

The robot runs the behavior tree locally. Any learned or probabilistic node
must expose typed inputs and outputs, confidence, evidence, timeout, and a
deterministic fallback. Safety and actuator interlocks remain deterministic.

Pass conditions:

- a bad signature, wrong robot, unsupported node, or map mismatch blocks motion;
- losing the remote link does not require a call between local nodes;
- retry does not create a second mission run;
- restart resumes only at a declared safe checkpoint;
- cancellation terminates through an authoritative local event; and
- telemetry never invents mission success.

### POC 5: Map version and approved release

Produce native SLAM artifacts, canonical occupancy geometry, a manifest,
checksums, provenance, and a metric frame as one immutable `MapVersion`. Create
a separate immutable `MapRelease` that references exactly one `MapVersion` and
adds approved alignment, overlays, policies, and routes. No-go and speed zones,
docks, and inspection poses live in that release composition.

An optional mutable release channel or alias may point to a `MapRelease`, but a
mission resolves it before assignment and pins the resulting `MapRelease` ID
and digest. Running work never follows the channel.

Pass conditions:

- checksum corruption blocks admission;
- a bad third alignment point exposes excessive residual error;
- overlay changes create a release without rewriting observed geometry;
- new SLAM geometry creates a new map version;
- an active mission remains pinned to its original `MapRelease` ID and digest
  when a channel moves; and
- native localization details remain behind the adapter.

### POC 6: Telemetry, media, and artifacts

Use an MQTT broker and local retry spool for operational observations, WebRTC
for live camera, and S3-compatible storage for images, maps, bags, and other
large evidence.

Pass conditions:

- broker acknowledgement is not reported as durable analytical retention;
- analytical failure does not stop local control or missions;
- replay produces no duplicate logical observations;
- media credentials and packets never enter command messages;
- viewer and publisher permissions are separate and short lived;
- object upload access is short lived and scoped to one artifact; and
- freshness is explicit for last-known state.

## Hardware decision

Do not buy a robot during Weeks 0–3. At the end of Week 4:

1. confirm the simulated square, odometry measurement, and watchdog test pass;
2. check whether a lab or teammate already has compatible hardware;
3. record the required ROS distribution, interfaces, sensors, and test space;
4. decide whether personal hardware materially improves repetition; and
5. preserve budget for shipping, tax, battery storage, and replacement parts.

Buying hardware is never required for graduation. A learner who does not buy or
borrow a robot completes Weeks 13–14 through the adversarial-physics Gazebo
track. That track validates software contracts and failure behavior but does not
authorize later physical motion; complete the physical track in full before
energizing any real base.

A practical sub-500-dollar option is the
[Yahboom MicroROS-Pi5](https://category.yahboom.net/products/microros-pi5)
robot-only kit plus a Raspberry Pi 5, provided the delivered total remains
within budget. It combines an ESP32 microcontroller, wheel encoders, IMU, 2D
lidar, camera, battery, and four-wheel skid-steer base.

Bring up its factory image first and record its topic, action, service,
parameter, frame, device, and firmware contracts. The vendor material currently
uses a ROS 2 Humble environment. Do not casually mix distributions; port the
complete integration deliberately as an advanced assignment. See the
[vendor tutorials](https://www.yahboom.com/study/MicroROS-Pi5) and
[source repository](https://github.com/YahboomTechnology/MicroROS-Car-Pi5).

The [MentorPi M1](https://www.hiwonder.com/products/mentorpi-m1) is an
alternative if its complete delivered price and documentation fit better.
[TurtleBot 3](https://www.robotis.us/turtlebot-3/) remains a valuable
simulation and ecosystem reference but usually exceeds this course budget.

## Physical safety gate

Before the first powered-wheel test:

- place the base securely on blocks;
- verify a reachable physical power cut or emergency stop;
- start below 0.15 metres per second in a clear indoor area;
- enforce a local command timeout and velocity limit;
- verify wheel polarity, encoder direction, IMU axes, and lidar orientation;
- keep people, pets, stairs, glass, cables, and fragile objects outside the
  test space;
- charge and store the battery according to its instructions on a suitable
  nonflammable surface; and
- treat hobby hardware and middleware as non-certified learning equipment.

Never make a remote network connection the only stop mechanism. Any stop-time
goal in this curriculum is an experiment target, not a universal safety
standard or certification.

## Capstone: networked inspection mission

Create a simulated rack, aisle, or room with three inspection poses.

End-to-end flow:

1. enroll and qualify one robot for the required capabilities;
2. publish a checksummed `MapVersion`, then an approved `MapRelease` containing
   that version and a no-go-zone overlay;
3. validate and sign immutable inspection mission version 1;
4. submit one mission run through the remote-side API;
5. append the command durably and return only a submission acceptance;
6. let the robot accept and execute the full bundle locally;
7. stream live camera video independently;
8. upload three checksummed evidence images independently;
9. spool telemetry during an artificial network outage; and
10. reconnect, replay, reconcile, and build a complete audit timeline.

Acceptance matrix:

| Property | Required result |
| --- | --- |
| Mission reliability | At least 8 of 10 clean runs complete |
| Contacts | Zero collisions in all 10 attempts |
| Idempotency | Ten duplicate submissions produce one local run |
| Expiry | Every stale or expired motion intent is rejected |
| Local stop | Meet and report the declared lab target |
| Network outage | Documented local policy and complete reconciliation |
| Map pin | `MapRelease` ID or digest mismatch prevents mission start |
| Media separation | Live frames never enter command or telemetry storage |
| Audit | Reviewer reconstructs request, receipt, attempts, evidence, result |
| Handoff | Predesignated attempts 8–10 are three consecutive successes run by another learner from a clean checkout and runbook |

## Theory learned just in time

| Lab trigger | Theory |
| --- | --- |
| Frames and RViz | Vectors, rotation, homogeneous transforms, SE(2), composition |
| Odometry | Differential and skid-steer kinematics, drift, covariance |
| Sensors | Sampling, noise, bias, field of view, timestamps, calibration |
| Localization | Probability, particles, scan matching, observability |
| Navigation | Costmaps, graph search, local control, recovery, behavior trees |
| Remote edge | Queues, backpressure, idempotency, sequence, epochs |
| Identity | Public-key infrastructure, certificate requests, mTLS, revocation |
| Missions | State machines, immutable versions, cancellation, checkpoints |
| Evidence | Provenance, checksums, access boundaries, retention |

Useful free references include
[Modern Robotics](https://hades.mech.northwestern.edu/index.php/Modern_Robotics),
[Robotics, Vision and Control](https://www.roboticsbook.org/intro.html),
[Nav2 documentation](https://docs.nav2.org/),
[SLAM Toolbox](https://github.com/SteveMacenski/slam_toolbox), and
[BehaviorTree.CPP](https://www.behaviortree.dev/docs/learn-the-basics/BT_basics/).

## Mentoring and review loop

For each checkpoint, bring:

- the source and relevant decision record;
- the exact command used;
- the final relevant log lines;
- the measurement method and result;
- the evidence manifest; and
- your own explanation of the observed failure.

Review should focus on system boundaries, frame and time assumptions,
repeatability, safety, failure behavior, and what the evidence cannot establish.
The first checkpoint is Week 0: graph inspection, a recorded/replayed topic, and
a documented simulator mode.

[Repository home](../README.md) ·
[Repository workflow](repository-workflow.md) ·
[Weekly index](weeks/README.md) ·
[Begin Week 0](weeks/week-00-environment-and-ros-graph.md)
