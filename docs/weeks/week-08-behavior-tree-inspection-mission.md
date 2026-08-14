# Week 8 — Behavior-Tree Inspection Mission

[← Week 7: Localization and Nav2](week-07-localization-and-nav2.md) · [Week index](README.md) · [Repository workflow](../repository-workflow.md) · [Week 9: Edge adapter and telemetry spool →](week-09-edge-adapter-and-telemetry-spool.md)

**Time budget:** 9–12 hours

**Primary result:** a multi-checkpoint inspection mission with an explicit behavior tree, timeout, retry, cancellation, and machine-readable evidence.

## Outcomes

By the end of this week, you can:

- read a BehaviorTree.CPP XML tree used by Nav2;
- explain `Sequence`, fallback/recovery control flow, node status, halting, and blackboard ports;
- distinguish robot navigation recovery from mission-level retry policy;
- execute named inspection checkpoints with Nav2 Simple Commander;
- log accepted, running, succeeded, failed, canceled, and timed-out states; and
- inject obstruction and cancellation failures without losing the mission history.

## Prerequisites

- Week 7 exit criteria met, including a frozen map and stable localization.
- At least three safe poses from `goals.csv`.
- Familiarity with Python functions, JSON, YAML, and ROS 2 actions.
- The Week 7 simulator, map server, AMCL, and Nav2 stack can be launched unchanged.

## Public readings

- [Nav2 behavior trees](https://docs.nav2.org/behavior_trees/index.html)
- [Nav2 default behavior trees](https://docs.nav2.org/behavior_trees/overview/detailed_behavior_tree_walkthrough.html)
- [Groot2 and Nav2 trees](https://docs.nav2.org/tutorials/docs/groot2.html)
- [BehaviorTree.CPP basics](https://www.behaviortree.dev/docs/learn-the-basics/main_concepts/)
- [Nav2 Simple Commander API](https://docs.nav2.org/commander_api/index.html)
- [ROS 2 actions design](https://design.ros2.org/articles/actions.html)

## Concepts

### Behavior-tree status drives control flow

Every tick returns `SUCCESS`, `FAILURE`, or `RUNNING`. A sequence advances while children succeed. A fallback tries alternatives after failure. A recovery node retries its primary child after a recovery child runs. Halting must stop asynchronous work; it is not merely hiding output.

### Blackboard values are typed dataflow

Ports such as `{goal}`, `{path}`, and error codes connect nodes through the tree blackboard. Treat these names as a contract. A syntactically valid XML file can still fail at runtime when a node plugin or port is unavailable.

### Two retry layers, two purposes

- The Nav2 tree can clear costmaps, spin, wait, back up, replan, and retry navigation.
- The mission runner decides whether a failed checkpoint should be attempted again, skipped, or terminate the mission.

Keep both bounded. Nested unbounded retries create robots that appear busy but never finish.

### Inspection must have a result

Arriving at a pose is not the inspection itself. This lab uses a timed placeholder that emits an inspection record. A later camera, barcode, gauge, or anomaly detector can replace it while preserving the mission contract.

## Environment and packages

```bash
sudo apt update
sudo apt install -y \
  ros-jazzy-nav2-simple-commander \
  ros-jazzy-nav2-behavior-tree \
  libxml2-utils \
  python3-yaml

source /opt/ros/jazzy/setup.bash
export TURTLEBOT3_MODEL=waffle ROS_DOMAIN_ID=30
export LAB_ROOT="$HOME/robotics-learning-lab"
mkdir -p "$LAB_ROOT/week08" "$LAB_ROOT/evidence/week08" "$LAB_ROOT/bags/week08"
```

## Lab: inspect the tree, then run a mission

### 1. Start the Week 7 navigation stack

Launch the same Gazebo world in Terminal 1 and `nav2_bringup bringup_launch.py` with the selected map in Terminal 2. Start RViz in Terminal 3, set the correct initial pose, and prove one short goal succeeds before adding mission code.

### 2. Locate and inspect Nav2's installed tree

```bash
export NAV2_BT_SHARE="$(ros2 pkg prefix --share nav2_bt_navigator)"
find "$NAV2_BT_SHARE/behavior_trees" -maxdepth 1 -type f -name '*.xml' -print | sort
export DEFAULT_BT="$NAV2_BT_SHARE/behavior_trees/navigate_to_pose_w_replanning_and_recovery.xml"
xmllint --noout "$DEFAULT_BT"
xmllint --format "$DEFAULT_BT" | sed -n '1,220p' | tee \
  "$LAB_ROOT/evidence/week08/default-tree.txt"
```

On paper or in `tree-walkthrough.md`, trace:

1. where the goal becomes a path;
2. where the path becomes velocity control;
3. which failures trigger contextual costmap clearing;
4. which system-level recoveries can run; and
5. what bounds the overall number of retries.

If you have Groot2, load the installed Nav2 node palette and this XML. Groot is optional; `xmllint` and the textual walkthrough are sufficient.

### 3. Create a controlled tree variant

Never edit the installed file. Copy it and reduce global replanning frequency from 1 Hz to 0.5 Hz so the experiment has one observable change:

```bash
cp "$DEFAULT_BT" "$LAB_ROOT/week08/inspection_nav.xml"
sed -i 's/<RateController hz="1.0">/<RateController hz="0.5">/' \
  "$LAB_ROOT/week08/inspection_nav.xml"
xmllint --noout "$LAB_ROOT/week08/inspection_nav.xml"
grep -nE 'RecoveryNode|RateController|ComputePath|FollowPath|Clear|Spin|Wait|BackUp' \
  "$LAB_ROOT/week08/inspection_nav.xml" | tee \
  "$LAB_ROOT/evidence/week08/custom-tree-nodes.txt"
```

If the installed tree uses a different frequency or formatting, edit only that `RateController` value manually and record the diff:

```bash
diff -u "$DEFAULT_BT" "$LAB_ROOT/week08/inspection_nav.xml" | tee \
  "$LAB_ROOT/evidence/week08/tree.diff"
```

Changing replanning frequency is an experiment, not an automatic improvement.

### 4. Define named checkpoints

Create `$LAB_ROOT/week08/checkpoints.yaml`, replacing coordinates with three proven free poses from Week 7:

```yaml
frame_id: map
checkpoint_timeout_s: 120
retry_limit: 1
inspection_dwell_s: 3
checkpoints:
  - name: entrance
    x: 0.5
    y: 0.0
    yaw: 0.0
  - name: aisle_end
    x: 1.0
    y: 1.0
    yaw: 1.57
  - name: dock_return
    x: 0.0
    y: 0.0
    yaw: 3.14
```

Validate the values against the RViz map. The sample coordinates are placeholders, not guaranteed safe goals.

### 5. Create the mission runner

Save this as `$LAB_ROOT/week08/run_inspection.py`:

```python
#!/usr/bin/env python3
import json
import hashlib
import math
import os
from pathlib import Path
import signal
import sys
import time
import uuid

import rclpy
from geometry_msgs.msg import PoseStamped
from nav2_simple_commander.robot_navigator import BasicNavigator, TaskResult
import yaml


def quaternion_from_yaw(yaw: float):
    return math.sin(yaw / 2.0), math.cos(yaw / 2.0)


def append_event(path: Path, **event):
    event["recorded_at_unix_s"] = time.time()
    with path.open("a", encoding="utf-8") as stream:
        stream.write(json.dumps(event, sort_keys=True) + "\n")


def pose_for(navigator, frame_id, checkpoint):
    pose = PoseStamped()
    pose.header.frame_id = frame_id
    pose.header.stamp = navigator.get_clock().now().to_msg()
    pose.pose.position.x = float(checkpoint["x"])
    pose.pose.position.y = float(checkpoint["y"])
    z, w = quaternion_from_yaw(float(checkpoint["yaw"]))
    pose.pose.orientation.z = z
    pose.pose.orientation.w = w
    return pose


def main():
    if len(sys.argv) != 3:
        raise SystemExit("usage: run_inspection.py CHECKPOINTS.yaml EVENTS.jsonl")

    config = yaml.safe_load(Path(sys.argv[1]).read_text(encoding="utf-8"))
    events_path = Path(sys.argv[2])
    events_path.parent.mkdir(parents=True, exist_ok=True)
    tree_path = Path(os.environ["BT_XML"]).expanduser().resolve()
    if not tree_path.is_file():
        raise SystemExit(f"behavior tree does not exist: {tree_path}")

    mission_id = str(uuid.uuid4())
    timeout_s = float(config["checkpoint_timeout_s"])
    retry_limit = int(config["retry_limit"])
    dwell_s = float(config["inspection_dwell_s"])

    rclpy.init()
    navigator = BasicNavigator()
    canceled = False

    def request_cancel(_signum, _frame):
        nonlocal canceled
        canceled = True
        navigator.cancelTask()

    signal.signal(signal.SIGINT, request_cancel)
    signal.signal(signal.SIGTERM, request_cancel)

    append_event(events_path, mission_id=mission_id, state="MISSION_STARTED",
                 tree_sha256=hashlib.sha256(tree_path.read_bytes()).hexdigest())
    navigator.waitUntilNav2Active()

    mission_ok = True
    try:
        for checkpoint in config["checkpoints"]:
            checkpoint_ok = False
            for attempt in range(1, retry_limit + 2):
                if canceled:
                    break
                goal = pose_for(navigator, config["frame_id"], checkpoint)
                started = time.monotonic()
                append_event(events_path, mission_id=mission_id,
                             checkpoint=checkpoint["name"], attempt=attempt,
                             state="NAVIGATION_REQUESTED")
                navigator.goToPose(goal, behavior_tree=str(tree_path))

                last_feedback_log = 0.0
                while not navigator.isTaskComplete():
                    elapsed = time.monotonic() - started
                    if canceled or elapsed > timeout_s:
                        navigator.cancelTask()
                    feedback = navigator.getFeedback()
                    if feedback and elapsed - last_feedback_log >= 2.0:
                        remaining = getattr(feedback, "distance_remaining", None)
                        append_event(events_path, mission_id=mission_id,
                                     checkpoint=checkpoint["name"], attempt=attempt,
                                     state="NAVIGATION_RUNNING",
                                     elapsed_s=round(elapsed, 3),
                                     distance_remaining_m=remaining)
                        last_feedback_log = elapsed

                result = navigator.getResult()
                elapsed = time.monotonic() - started
                state = {
                    TaskResult.SUCCEEDED: "NAVIGATION_SUCCEEDED",
                    TaskResult.CANCELED: "NAVIGATION_CANCELED",
                    TaskResult.FAILED: "NAVIGATION_FAILED",
                }.get(result, "NAVIGATION_UNKNOWN")
                if elapsed > timeout_s and result != TaskResult.SUCCEEDED:
                    state = "NAVIGATION_TIMED_OUT"
                append_event(events_path, mission_id=mission_id,
                             checkpoint=checkpoint["name"], attempt=attempt,
                             state=state, elapsed_s=round(elapsed, 3))

                if result == TaskResult.SUCCEEDED:
                    time.sleep(dwell_s)
                    append_event(events_path, mission_id=mission_id,
                                 checkpoint=checkpoint["name"], attempt=attempt,
                                 state="INSPECTION_CAPTURED",
                                 observation={"kind": "training-placeholder",
                                              "status": "CLEAR"})
                    checkpoint_ok = True
                    break

            if canceled or not checkpoint_ok:
                mission_ok = False
                break
    finally:
        final_state = "MISSION_CANCELED" if canceled else (
            "MISSION_SUCCEEDED" if mission_ok else "MISSION_FAILED")
        append_event(events_path, mission_id=mission_id, state=final_state)
        navigator.destroy_node()
        rclpy.shutdown()

    raise SystemExit(0 if mission_ok and not canceled else 2)


if __name__ == "__main__":
    main()
```

Make it executable and syntax-check it:

```bash
chmod +x "$LAB_ROOT/week08/run_inspection.py"
python3 -m py_compile "$LAB_ROOT/week08/run_inspection.py"
```

### 6. Run and observe the mission

Terminal 4:

```bash
export BT_XML="$LAB_ROOT/week08/inspection_nav.xml"
ros2 bag record -o "$LAB_ROOT/bags/week08/inspection" \
  /amcl_pose /odom /tf /tf_static /plan /cmd_vel
```

Terminal 5:

```bash
source /opt/ros/jazzy/setup.bash
export ROS_DOMAIN_ID=30 LAB_ROOT="$HOME/robotics-learning-lab"
export BT_XML="$LAB_ROOT/week08/inspection_nav.xml"
python3 "$LAB_ROOT/week08/run_inspection.py" \
  "$LAB_ROOT/week08/checkpoints.yaml" \
  "$LAB_ROOT/evidence/week08/mission-events.jsonl"
```

Follow the global path, local costmap, and robot in RViz. After completion:

```bash
python3 -m json.tool --json-lines \
  "$LAB_ROOT/evidence/week08/mission-events.jsonl" | less
```

If your Python version lacks `--json-lines`, use:

```bash
while IFS= read -r line; do echo "$line" | python3 -m json.tool; done \
  < "$LAB_ROOT/evidence/week08/mission-events.jsonl"
```

## Assignment

Run the same three checkpoints twice:

1. with the installed default tree;
2. with `inspection_nav.xml` at 0.5 Hz replanning.

Compare terminal success, total duration, each checkpoint duration, estimated replanning frequency from `/plan`, recovery count, and path behavior. Do not claim one tree is better from a single run; state only what the evidence supports.

Then replace the inspection placeholder with one real observable check, such as:

- save one compressed camera image;
- compute the minimum forward laser range during the dwell;
- record a temperature or battery sample; or
- verify a fiducial detection.

The observation record must contain capture time, checkpoint, measurement/unit, and success or failure.

## Failure injection

Complete all three:

1. **Blocked checkpoint:** place an obstacle across a route after navigation starts. Let bounded tree recoveries and the one mission retry finish. Record both layers separately.
2. **Operator cancellation:** press `Ctrl-C` during the second checkpoint. Verify the active Nav2 goal is canceled, velocity returns to zero, a terminal `MISSION_CANCELED` record is written, and no later checkpoint begins.
3. **Bad tree reference:** set `BT_XML` to a missing path. The runner must fail before dispatching motion. Restore the correct path and rerun.

Optional: place one checkpoint in occupied space and confirm that failure does not produce `INSPECTION_CAPTURED`.

## Measurements and deliverables

Store under `$LAB_ROOT/evidence/week08/`:

- formatted default tree and `tree-walkthrough.md`;
- custom tree, XML validation result, and `tree.diff`;
- checkpoint YAML;
- complete JSONL event history for success, blocked, and canceled runs;
- rosbag metadata;
- default-vs-custom comparison table;
- one screenshot showing a planned route and costmap; and
- the real inspection measurement or artifact.

Report:

- mission and checkpoint success rates;
- p50 and maximum checkpoint time across repeated runs;
- retry and recovery counts;
- cancellation latency until `/cmd_vel` becomes zero; and
- whether every requested checkpoint has exactly one terminal navigation state.

## Objective exit criteria

Advance only when:

- the custom XML passes `xmllint` and its diff from the installed tree is documented;
- a normal run reaches all checkpoints and emits one inspection record per checkpoint;
- timeout and retry counts are finite and visible in configuration;
- a blocked route ends in a clear success or failure rather than running forever;
- cancellation stops motion within 2 seconds in simulation and writes a terminal mission event;
- no inspection is recorded for a checkpoint that navigation did not reach; and
- a reviewer can reconstruct mission order and outcomes from JSONL alone.

## Troubleshooting

| Symptom | Diagnostic | Correction |
| --- | --- | --- |
| Tree XML parses but goal fails immediately | Inspect navigator logs for missing node/plugin/port | Start from the installed tree for your exact distribution; change one field at a time. |
| `BasicNavigator` waits forever | Check `/amcl` and `/bt_navigator` lifecycle state | Activate Nav2 and set the initial pose. |
| Goal succeeds but no inspection event | Inspect the runner exception and JSONL tail | Keep inspection after the success branch; make observation failures explicit. |
| Cancel leaves robot moving | Inspect `/cmd_vel` and action state | Ensure the signal handler calls `cancelTask`; stop the simulator if safety is uncertain. |
| Every attempt times out | Validate goal coordinates and timeout units | Test a nearby RViz goal, then inspect planner/controller logs. |
| `sed` creates no diff | `grep -n RateController "$DEFAULT_BT"` | Match the installed formatting and document the actual one-line change. |

## Next step

Week 9 builds a generic edge adapter around this robot-side work. You will normalize source-specific telemetry, persist it before network transfer, survive a broker outage, and replay without losing stable event identity.

[← Week 7: Localization and Nav2](week-07-localization-and-nav2.md) · [Week index](README.md) · [Week 9: Edge adapter and telemetry spool →](week-09-edge-adapter-and-telemetry-spool.md)
