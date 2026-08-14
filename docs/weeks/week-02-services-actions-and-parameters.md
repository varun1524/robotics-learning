# Week 2 — Services, actions, and parameters

[Previous: Week 1](week-01-topics-timing-and-qos.md) · [Curriculum index](README.md) · [Repository workflow](../repository-workflow.md) · [Next: Week 3](week-03-urdf-tf-and-rviz.md)

**Time:** 10–14 hours. **Build:** a validated, cancellable distance-motion server with service interlock and progress feedback.

## Outcomes

By the end of this week you can:

- select topics, services, or actions based on interaction semantics;
- define and build a custom ROS 2 action interface;
- accept/reject goals, publish feedback, return results, and honor cancellation;
- use a service as a simple enable interlock and parameters as validated configuration;
- inject invalid configuration, rejected goals, cancellation, and mid-goal disablement and predict each state transition.

## Prerequisites

- Week 1 package builds and compatible QoS delivery works.
- Basic Python asynchronous-programming vocabulary.
- No physical robot is used: “distance” is integrated by the server so state-machine behavior is deterministic. Week 4 connects the same ideas to simulated motors.

## Public readings

1. [Topics, services, and actions: when to use each](https://docs.ros.org/en/jazzy/How-To-Guides/Topics-Services-Actions.html)
2. [Writing an action server and client in Python](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Writing-an-Action-Server-Client/Py.html)
3. [Creating custom interfaces](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Single-Package-Define-And-Use-Interface.html)
4. [Using parameters in a Python class](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Using-Parameters-In-A-Class-Python.html)

## Concepts

- A **topic** is a stream: many samples, no per-sample response, producers and consumers decoupled.
- A **service** is a short request/response transaction. Avoid long work in a service callback because callers cannot observe progress or request cancellation cleanly.
- An **action** is a goal with acceptance, feedback, cancellation, and a terminal result. It fits navigation, manipulation, docking, and this distance exercise.
- A **parameter** configures a node. Validation belongs at the boundary so a bad value is rejected before it corrupts execution.

The action terminal states matter: **succeeded**, **canceled**, and **aborted** are not interchangeable. User-requested cancellation becomes canceled. Disabling the motion interlock while a goal is active is a safety condition and becomes aborted.

## Environment and packages

```bash
source /opt/ros/jazzy/setup.bash
mkdir -p ~/robotics_ws/src ~/robotics_ws/evidence/week02
cd ~/robotics_ws/src
ros2 pkg create --build-type ament_cmake --license Apache-2.0 week02_interfaces
mkdir -p week02_interfaces/action
```

Create `~/robotics_ws/src/week02_interfaces/action/DriveDistance.action`:

```text
float64 distance_m
---
bool success
float64 final_distance_m
string message
---
float64 distance_traveled_m
```

Replace `week02_interfaces/CMakeLists.txt` with:

```cmake
cmake_minimum_required(VERSION 3.8)
project(week02_interfaces)

find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
  "action/DriveDistance.action"
)

ament_export_dependencies(rosidl_default_runtime)
ament_package()
```

Replace `week02_interfaces/package.xml` with:

```xml
<?xml version="1.0"?>
<package format="3">
  <name>week02_interfaces</name>
  <version>0.0.0</version>
  <description>Interfaces for the Week 2 action lab</description>
  <maintainer email="student@example.com">student</maintainer>
  <license>Apache-2.0</license>
  <buildtool_depend>ament_cmake</buildtool_depend>
  <build_depend>rosidl_default_generators</build_depend>
  <exec_depend>rosidl_default_runtime</exec_depend>
  <member_of_group>rosidl_interface_packages</member_of_group>
  <export><build_type>ament_cmake</build_type></export>
</package>
```

Build the interface before importing it from Python:

```bash
cd ~/robotics_ws
colcon build --packages-select week02_interfaces
source install/setup.bash
ros2 interface show week02_interfaces/action/DriveDistance
```

## Lab 2A — Create the motion package and server

```bash
cd ~/robotics_ws/src
ros2 pkg create --build-type ament_python --license Apache-2.0 \
  week02_motion --dependencies rclpy rcl_interfaces std_srvs week02_interfaces
```

Create `week02_motion/week02_motion/motion_server.py`:

```python
import asyncio
import time

import rclpy
from rcl_interfaces.msg import SetParametersResult
from rclpy.action import ActionServer, CancelResponse, GoalResponse
from rclpy.callback_groups import ReentrantCallbackGroup
from rclpy.executors import MultiThreadedExecutor
from rclpy.node import Node
from std_srvs.srv import SetBool

from week02_interfaces.action import DriveDistance


class MotionServer(Node):
    def __init__(self):
        super().__init__('motion_server')
        self.declare_parameter('speed_mps', 0.25)
        self.declare_parameter('control_rate_hz', 20.0)
        self.declare_parameter('max_distance_m', 2.0)
        self.enabled = True
        self.group = ReentrantCallbackGroup()
        self.parameter_callback = self.add_on_set_parameters_callback(
            self.validate_parameters)
        self.enable_service = self.create_service(
            SetBool, '/motion/enable', self.set_enabled,
            callback_group=self.group)
        self.action_server = ActionServer(
            self,
            DriveDistance,
            '/drive_distance',
            execute_callback=self.execute,
            goal_callback=self.accept_goal,
            cancel_callback=self.accept_cancel,
            callback_group=self.group,
        )
        self.get_logger().info('motion enabled; waiting for /drive_distance')

    def validate_parameters(self, parameters):
        for parameter in parameters:
            if parameter.name in ('speed_mps', 'control_rate_hz', 'max_distance_m'):
                if not isinstance(parameter.value, (int, float)) or parameter.value <= 0.0:
                    return SetParametersResult(
                        successful=False,
                        reason=f'{parameter.name} must be a positive number')
        return SetParametersResult(successful=True)

    def set_enabled(self, request, response):
        self.enabled = bool(request.data)
        response.success = True
        response.message = f'motion enabled={self.enabled}'
        self.get_logger().warn(response.message)
        return response

    def accept_goal(self, goal_request):
        maximum = float(self.get_parameter('max_distance_m').value)
        if goal_request.distance_m <= 0.0 or goal_request.distance_m > maximum:
            self.get_logger().warn(
                f'rejecting distance={goal_request.distance_m:.3f}; '
                f'allowed range is (0, {maximum:.3f}]')
            return GoalResponse.REJECT
        if not self.enabled:
            self.get_logger().warn('rejecting goal while motion is disabled')
            return GoalResponse.REJECT
        return GoalResponse.ACCEPT

    def accept_cancel(self, _goal_handle):
        self.get_logger().info('accepting cancellation request')
        return CancelResponse.ACCEPT

    async def execute(self, goal_handle):
        target = float(goal_handle.request.distance_m)
        traveled = 0.0
        previous_time = time.monotonic()
        self.get_logger().info(f'executing target={target:.3f} m')

        while traveled < target:
            if goal_handle.is_cancel_requested:
                goal_handle.canceled()
                return DriveDistance.Result(
                    success=False,
                    final_distance_m=traveled,
                    message='goal canceled by client')

            if not self.enabled:
                goal_handle.abort()
                return DriveDistance.Result(
                    success=False,
                    final_distance_m=traveled,
                    message='goal aborted: motion disabled')

            now = time.monotonic()
            elapsed = now - previous_time
            previous_time = now
            speed = float(self.get_parameter('speed_mps').value)
            rate = float(self.get_parameter('control_rate_hz').value)
            traveled = min(target, traveled + speed * elapsed)
            feedback = DriveDistance.Feedback()
            feedback.distance_traveled_m = traveled
            goal_handle.publish_feedback(feedback)
            await asyncio.sleep(1.0 / rate)

        goal_handle.succeed()
        return DriveDistance.Result(
            success=True,
            final_distance_m=traveled,
            message='target reached')


def main(args=None):
    rclpy.init(args=args)
    node = MotionServer()
    executor = MultiThreadedExecutor(num_threads=4)
    executor.add_node(node)
    try:
        executor.spin()
    finally:
        node.action_server.destroy()
        node.destroy_node()
        rclpy.shutdown()


if __name__ == '__main__':
    main()
```

`ReentrantCallbackGroup` plus a multithreaded executor allows the enable service and parameter callbacks to run while an action is active. The execution coroutine also yields on every control period.

## Lab 2B — Create a measurable client

Create `week02_motion/week02_motion/drive_client.py`:

```python
import time

import rclpy
from action_msgs.msg import GoalStatus
from rclpy.action import ActionClient
from rclpy.node import Node

from week02_interfaces.action import DriveDistance


STATUS_NAMES = {
    GoalStatus.STATUS_SUCCEEDED: 'SUCCEEDED',
    GoalStatus.STATUS_CANCELED: 'CANCELED',
    GoalStatus.STATUS_ABORTED: 'ABORTED',
}


class DriveClient(Node):
    def __init__(self):
        super().__init__('drive_client')
        self.declare_parameter('distance_m', 1.0)
        self.declare_parameter('cancel_after_s', 0.0)
        self.client = ActionClient(self, DriveDistance, '/drive_distance')
        self.start_time = time.monotonic()
        self.cancel_requested_at = None
        self.feedback_count = 0
        self.goal_handle = None
        self.cancel_timer = None

        self.get_logger().info('waiting for action server')
        self.client.wait_for_server()
        goal = DriveDistance.Goal()
        goal.distance_m = float(self.get_parameter('distance_m').value)
        future = self.client.send_goal_async(goal, feedback_callback=self.feedback)
        future.add_done_callback(self.goal_response)

    def goal_response(self, future):
        self.goal_handle = future.result()
        if not self.goal_handle.accepted:
            self.get_logger().error('goal REJECTED')
            rclpy.shutdown()
            return
        self.get_logger().info('goal ACCEPTED')
        cancel_after = float(self.get_parameter('cancel_after_s').value)
        if cancel_after > 0.0:
            self.cancel_timer = self.create_timer(cancel_after, self.cancel)
        result_future = self.goal_handle.get_result_async()
        result_future.add_done_callback(self.result)

    def feedback(self, message):
        self.feedback_count += 1
        if self.feedback_count % 10 == 0:
            self.get_logger().info(
                f'feedback distance={message.feedback.distance_traveled_m:.3f} m')

    def cancel(self):
        self.cancel_timer.cancel()
        self.cancel_requested_at = time.monotonic()
        self.get_logger().warn('requesting cancellation')
        self.goal_handle.cancel_goal_async()

    def result(self, future):
        wrapped = future.result()
        elapsed = time.monotonic() - self.start_time
        feedback_rate = self.feedback_count / max(elapsed, 1e-9)
        cancel_ms = 'n/a'
        if self.cancel_requested_at is not None:
            cancel_ms = f'{(time.monotonic() - self.cancel_requested_at) * 1000.0:.1f}'
        status = STATUS_NAMES.get(wrapped.status, str(wrapped.status))
        result = wrapped.result
        self.get_logger().info(
            f'status={status} success={result.success} '
            f'final_m={result.final_distance_m:.4f} elapsed_s={elapsed:.3f} '
            f'feedback_hz={feedback_rate:.1f} cancel_to_result_ms={cancel_ms} '
            f'message="{result.message}"')
        rclpy.shutdown()


def main(args=None):
    rclpy.init(args=args)
    node = DriveClient()
    try:
        rclpy.spin(node)
    finally:
        node.destroy_node()
        if rclpy.ok():
            rclpy.shutdown()


if __name__ == '__main__':
    main()
```

Replace `week02_motion/setup.py` with:

```python
from setuptools import find_packages, setup

package_name = 'week02_motion'

setup(
    name=package_name,
    version='0.0.0',
    packages=find_packages(exclude=['test']),
    data_files=[
        ('share/ament_index/resource_index/packages', ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
    ],
    install_requires=['setuptools'],
    zip_safe=True,
    maintainer='student',
    maintainer_email='student@example.com',
    description='Cancellable action and service interlock lab',
    license='Apache-2.0',
    entry_points={'console_scripts': [
        'motion_server = week02_motion.motion_server:main',
        'drive_client = week02_motion.drive_client:main',
    ]},
)
```

Build both packages:

```bash
cd ~/robotics_ws
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install --packages-select \
  week02_interfaces week02_motion
source install/setup.bash
```

## Lab 2C — Happy path and introspection

```bash
# Terminal A
source ~/robotics_ws/install/setup.bash
ros2 run week02_motion motion_server
```

```bash
# Terminal B
source ~/robotics_ws/install/setup.bash
ros2 action list -t
ros2 service list -t | grep /motion/enable
ros2 param list /motion_server
ros2 run week02_motion drive_client --ros-args \
  -p distance_m:=1.0 -p cancel_after_s:=0.0 | \
  tee ~/robotics_ws/evidence/week02/success.log
```

At 0.25 m/s, a 1 m goal should take about 4 seconds, give feedback near 20 Hz, and terminate `SUCCEEDED` with final distance 1.0 m.

## Deliberate failure injections

Run each experiment separately and save its terminal output.

### 1. Invalid parameter is rejected

```bash
ros2 param set /motion_server speed_mps -0.1
ros2 param get /motion_server speed_mps
```

The set must fail and the retained value must remain positive.

### 2. Out-of-range goal is rejected

```bash
ros2 action send_goal /drive_distance \
  week02_interfaces/action/DriveDistance \
  "{distance_m: 99.0}" --feedback
```

The goal must be rejected before execution.

### 3. Client cancellation reaches a canceled result

```bash
ros2 run week02_motion drive_client --ros-args \
  -p distance_m:=1.5 -p cancel_after_s:=1.5 | \
  tee ~/robotics_ws/evidence/week02/canceled.log
```

The client must report `CANCELED`, not succeeded or aborted, and print cancellation-to-result latency.

### 4. Disablement aborts an active goal

Start a long goal in Terminal B:

```bash
ros2 run week02_motion drive_client --ros-args \
  -p distance_m:=2.0 -p cancel_after_s:=0.0 | \
  tee ~/robotics_ws/evidence/week02/disabled.log
```

Within one second, call from Terminal C:

```bash
ros2 service call /motion/enable std_srvs/srv/SetBool "{data: false}"
```

The active goal must become `ABORTED`. A new goal while disabled must be rejected. Restore the interlock:

```bash
ros2 service call /motion/enable std_srvs/srv/SetBool "{data: true}"
```

## Assignment

Run the following matrix and complete it from client logs. Use `ros2 param set` between runs; do not restart the server unless testing restart behavior.

| Test | Target m | Speed m/s | Expected terminal state | Elapsed s | Final m | Error mm | Feedback Hz | Cancel/result ms |
|---|---:|---:|---|---:|---:|---:|---:|---:|
| Baseline | 1.0 | 0.25 | SUCCEEDED |  |  |  |  | n/a |
| Faster | 1.0 | 0.50 | SUCCEEDED |  |  |  |  | n/a |
| Cancel | 1.5 | 0.25 | CANCELED |  |  | n/a |  |  |
| Disable | 2.0 | 0.25 | ABORTED |  |  | n/a |  | n/a |
| Too far | 99.0 | 0.25 | REJECTED |  | n/a | n/a | n/a | n/a |

For the successful rows, compute `error_mm = 1000 * abs(target - final)`. Explain why integrated distance is a state-machine stand-in, not wheel odometry or ground truth.

## Evidence and deliverables

- The two source packages and generated interface.
- `success.log`, `canceled.log`, `disabled.log`, and a completed table in `~/robotics_ws/evidence/week02/report.md`.
- Captured output proving invalid parameter rejection and over-limit goal rejection.
- A state diagram with accepted, executing, succeeded, canceled, aborted, and rejected paths.

## Objective exit criteria

- A 1.0 m goal at 0.25 m/s succeeds in 3.7–4.5 s, with final error at most 1 mm and feedback between 15 and 25 Hz.
- Cancellation produces `CANCELED` within 250 ms of the request on an idle VM.
- Service disablement during execution produces `ABORTED` and new goals remain rejected until re-enabled.
- Negative `speed_mps`, `control_rate_hz`, and `max_distance_m` values are rejected without changing the live value.
- You can justify why the enable operation is a service, the motion request is an action, and feedback is not a second service.

## Troubleshooting

| Symptom | Diagnosis | Fix |
|---|---|---|
| `week02_interfaces` import fails | interface was not built/sourced first | Build it, then `source ~/robotics_ws/install/setup.bash` before building/running the Python package. |
| Action exists but service cannot run during a goal | callback group/executor serialized work | Confirm `ReentrantCallbackGroup` and `MultiThreadedExecutor` are used. |
| Goal stays executing forever | speed/rate invalid or coroutine not yielding | Query parameters; confirm validation and `await asyncio.sleep(...)`. |
| Client prints a numeric status | unexpected terminal state | Compare it with `action_msgs/msg/GoalStatus`; inspect server logs before changing code. |
| Goal is rejected unexpectedly | server disabled or target outside range | Call the enable service, then query `max_distance_m`. |
| Parameter set appears successful but behavior is stale | wrong node or overlay | `ros2 node list`; source the current workspace in every terminal. |

## Next step

Week 3 gives the robot a physical structure: links, joints, coordinate frames, URDF/Xacro validation, and an RViz model that can be inspected before simulation dynamics are added.
