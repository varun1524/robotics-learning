# Week 4 — Gazebo control and command watchdogs

[Previous: Week 3](week-03-urdf-tf-and-rviz.md) · [Curriculum index](README.md) · [Repository workflow](../repository-workflow.md) · [Next: Week 5](week-05-sensors-rosbag-and-replay.md)

**Time:** 12–16 hours. **Build:** a Gazebo Harmonic differential-drive robot controlled through `ros2_control`, with odometry and a measured stale-command stop.

## Outcomes

By the end of this week you can:

- separate Gazebo physics, simulated hardware, controller manager, controllers, and ROS command interfaces;
- expose wheel velocity command/state interfaces through `gz_ros2_control`;
- configure and activate `joint_state_broadcaster` and `diff_drive_controller`;
- command a base with `TwistStamped`, inspect odometry/TF, and quantify straight-line behavior;
- prove that the base stops when command publication disappears and diagnose a command sent to the wrong topic.

## Prerequisites

- Week 3 final model validates and its TF tree has one root.
- Gazebo Harmonic runs by GUI or the Week 0 headless path.
- Never apply this lab's open-loop test commands to physical hardware without an emergency stop, clear test area, and the platform's documented enable procedure.

## Public readings

1. [`gz_ros2_control` Jazzy documentation](https://control.ros.org/jazzy/doc/gz_ros2_control/doc/index.html)
2. [`ros2_control` getting started](https://control.ros.org/jazzy/doc/getting_started/getting_started.html)
3. [`diff_drive_controller` Jazzy interface and parameters](https://control.ros.org/jazzy/doc/ros2_controllers/diff_drive_controller/doc/userdoc.html)
4. [Gazebo simulation with ROS 2](https://gazebosim.org/docs/harmonic/ros2_overview/)
5. [Controller-manager command-line tools](https://control.ros.org/jazzy/doc/ros2_control/ros2controlcli/doc/userdoc.html)

## Concepts

Gazebo advances world physics. `gz_ros2_control` acts as simulated hardware: it exports wheel command and state interfaces to a **controller manager**. The joint-state broadcaster publishes measured joint states. The differential-drive controller converts desired body linear/angular velocity into left/right wheel velocity, then integrates wheel feedback into odometry.

This is a safety boundary: a velocity is not a one-shot destination. It is a lease that must be refreshed. With `cmd_vel_timeout: 0.5`, the controller considers a command stale after 0.5 seconds and commands zero. The timeout remains necessary even if a higher-level planner has its own stop logic.

Odometry is an estimate, not ground truth. Wheel radius and separation calibration directly affect distance and turn angle; slip creates additional error.

## Environment and packages

```bash
source /opt/ros/jazzy/setup.bash
sudo apt update
sudo apt install -y \
  ros-jazzy-gz-ros2-control ros-jazzy-ros2-control \
  ros-jazzy-ros2-controllers ros-jazzy-controller-manager \
  ros-jazzy-ros-gz-sim ros-jazzy-ros-gz-bridge \
  ros-jazzy-xacro ros-jazzy-robot-state-publisher

mkdir -p ~/robotics_ws/src ~/robotics_ws/evidence/week04
cd ~/robotics_ws/src
ros2 pkg create --build-type ament_cmake --license Apache-2.0 week04_sim
mkdir -p week04_sim/urdf week04_sim/config week04_sim/launch
```

Replace `week04_sim/CMakeLists.txt` with:

```cmake
cmake_minimum_required(VERSION 3.8)
project(week04_sim)
find_package(ament_cmake REQUIRED)
install(DIRECTORY config launch urdf DESTINATION share/${PROJECT_NAME})
ament_package()
```

Replace `week04_sim/package.xml` with:

```xml
<?xml version="1.0"?>
<package format="3">
  <name>week04_sim</name>
  <version>0.0.0</version>
  <description>Differential-drive Gazebo control and watchdog lab</description>
  <maintainer email="student@example.com">student</maintainer>
  <license>Apache-2.0</license>
  <buildtool_depend>ament_cmake</buildtool_depend>
  <exec_depend>controller_manager</exec_depend>
  <exec_depend>diff_drive_controller</exec_depend>
  <exec_depend>gz_ros2_control</exec_depend>
  <exec_depend>joint_state_broadcaster</exec_depend>
  <exec_depend>launch_ros</exec_depend>
  <exec_depend>robot_state_publisher</exec_depend>
  <exec_depend>ros_gz_bridge</exec_depend>
  <exec_depend>ros_gz_sim</exec_depend>
  <exec_depend>rviz2</exec_depend>
  <exec_depend>xacro</exec_depend>
  <export><build_type>ament_cmake</build_type></export>
</package>
```

## Lab 4A — Add simulated hardware to the robot

Create `week04_sim/urdf/course_bot_sim.urdf.xacro`:

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro" name="course_bot">
  <xacro:property name="pi" value="3.141592653589793"/>
  <xacro:property name="wheel_radius" value="0.075"/>
  <xacro:property name="wheel_width" value="0.04"/>

  <material name="blue"><color rgba="0.1 0.35 0.8 1"/></material>
  <material name="dark"><color rgba="0.08 0.08 0.08 1"/></material>

  <link name="base_footprint"/>
  <link name="base_link">
    <visual><geometry><box size="0.40 0.30 0.12"/></geometry><material name="blue"/></visual>
    <collision><geometry><box size="0.40 0.30 0.12"/></geometry></collision>
    <inertial>
      <mass value="5.0"/>
      <inertia ixx="0.0435" ixy="0" ixz="0" iyy="0.0727" iyz="0" izz="0.1042"/>
    </inertial>
  </link>
  <joint name="base_footprint_joint" type="fixed">
    <parent link="base_footprint"/><child link="base_link"/>
    <origin xyz="0 0 0.12"/>
  </joint>

  <xacro:macro name="wheel" params="side y">
    <link name="${side}_wheel_link">
      <visual>
        <origin rpy="${pi/2} 0 0"/>
        <geometry><cylinder radius="${wheel_radius}" length="${wheel_width}"/></geometry>
        <material name="dark"/>
      </visual>
      <collision>
        <origin rpy="${pi/2} 0 0"/>
        <geometry><cylinder radius="${wheel_radius}" length="${wheel_width}"/></geometry>
      </collision>
      <inertial>
        <mass value="0.30"/>
        <inertia ixx="0.00045" ixy="0" ixz="0" iyy="0.00085" iyz="0" izz="0.00045"/>
      </inertial>
    </link>
    <joint name="${side}_wheel_joint" type="continuous">
      <parent link="base_link"/><child link="${side}_wheel_link"/>
      <origin xyz="0 ${y} -0.045"/>
      <axis xyz="0 1 0"/>
      <limit effort="10" velocity="15"/>
      <dynamics damping="0.02" friction="0.0"/>
    </joint>
  </xacro:macro>

  <xacro:wheel side="left" y="0.17"/>
  <xacro:wheel side="right" y="-0.17"/>

  <link name="caster_link">
    <visual><geometry><sphere radius="0.04"/></geometry><material name="dark"/></visual>
    <collision><geometry><sphere radius="0.04"/></geometry></collision>
    <inertial><mass value="0.10"/><inertia ixx="0.000064" ixy="0" ixz="0" iyy="0.000064" iyz="0" izz="0.000064"/></inertial>
  </link>
  <joint name="caster_joint" type="fixed">
    <parent link="base_link"/><child link="caster_link"/>
    <origin xyz="-0.15 0 -0.08"/>
  </joint>

  <gazebo reference="left_wheel_link"><mu1>1.0</mu1><mu2>1.0</mu2></gazebo>
  <gazebo reference="right_wheel_link"><mu1>1.0</mu1><mu2>1.0</mu2></gazebo>
  <gazebo reference="caster_link"><mu1>0.0</mu1><mu2>0.0</mu2></gazebo>

  <ros2_control name="GazeboSimSystem" type="system">
    <hardware><plugin>gz_ros2_control/GazeboSimSystem</plugin></hardware>
    <joint name="left_wheel_joint">
      <command_interface name="velocity"/>
      <state_interface name="position"/>
      <state_interface name="velocity"/>
    </joint>
    <joint name="right_wheel_joint">
      <command_interface name="velocity"/>
      <state_interface name="position"/>
      <state_interface name="velocity"/>
    </joint>
  </ros2_control>

  <gazebo>
    <plugin filename="libgz_ros2_control-system.so"
            name="gz_ros2_control::GazeboSimROS2ControlPlugin">
      <parameters>$(find week04_sim)/config/controllers.yaml</parameters>
    </plugin>
  </gazebo>
</robot>
```

Validate the expanded URDF before invoking Gazebo:

```bash
cd ~/robotics_ws
xacro src/week04_sim/urdf/course_bot_sim.urdf.xacro > /tmp/course_bot_sim.urdf
check_urdf /tmp/course_bot_sim.urdf
```

## Lab 4B — Configure controllers and the watchdog

Create `week04_sim/config/controllers.yaml`:

```yaml
controller_manager:
  ros__parameters:
    update_rate: 100
    use_sim_time: true

    joint_state_broadcaster:
      type: joint_state_broadcaster/JointStateBroadcaster

    diff_drive_controller:
      type: diff_drive_controller/DiffDriveController

diff_drive_controller:
  ros__parameters:
    left_wheel_names: [left_wheel_joint]
    right_wheel_names: [right_wheel_joint]
    wheel_separation: 0.34
    wheel_radius: 0.075
    position_feedback: true
    open_loop: false
    odom_frame_id: odom
    base_frame_id: base_footprint
    enable_odom_tf: true
    publish_rate: 50.0
    cmd_vel_timeout: 0.5
    publish_limited_velocity: true
    velocity_rolling_window_size: 10
    linear.x.max_velocity: 0.5
    linear.x.min_velocity: -0.5
    linear.x.max_acceleration: 1.0
    linear.x.max_deceleration: -1.0
    angular.z.max_velocity: 1.5
    angular.z.min_velocity: -1.5
    angular.z.max_acceleration: 3.0
    angular.z.max_deceleration: -3.0
```

The controller's Jazzy input is `geometry_msgs/msg/TwistStamped` on `/diff_drive_controller/cmd_vel`. Its automatic stale-command stop is set to 0.5 seconds.

## Lab 4C — Launch Gazebo, spawn the robot, and activate controllers

Create `week04_sim/launch/sim.launch.py`:

```python
import os

from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument, IncludeLaunchDescription, TimerAction
from launch.conditions import IfCondition
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import Command, LaunchConfiguration
from launch_ros.actions import Node
from launch_ros.parameter_descriptions import ParameterValue


def generate_launch_description():
    share = get_package_share_directory('week04_sim')
    ros_gz_share = get_package_share_directory('ros_gz_sim')
    model = os.path.join(share, 'urdf', 'course_bot_sim.urdf.xacro')
    robot_description = ParameterValue(Command(['xacro ', model]), value_type=str)

    gazebo = IncludeLaunchDescription(
        PythonLaunchDescriptionSource(
            os.path.join(ros_gz_share, 'launch', 'gz_sim.launch.py')),
        launch_arguments={'gz_args': LaunchConfiguration('gz_args')}.items())

    spawn = Node(
        package='ros_gz_sim', executable='create', output='screen',
        arguments=['-topic', 'robot_description', '-name', 'course_bot', '-z', '0.02'])

    joint_broadcaster = Node(
        package='controller_manager', executable='spawner', output='screen',
        arguments=['joint_state_broadcaster', '--controller-manager', '/controller_manager',
                   '--controller-manager-timeout', '60'])
    drive_controller = Node(
        package='controller_manager', executable='spawner', output='screen',
        arguments=['diff_drive_controller', '--controller-manager', '/controller_manager',
                   '--controller-manager-timeout', '60'])

    return LaunchDescription([
        DeclareLaunchArgument('gz_args', default_value='-r empty.sdf'),
        DeclareLaunchArgument('rviz', default_value='false'),
        gazebo,
        Node(
            package='robot_state_publisher', executable='robot_state_publisher',
            parameters=[{'robot_description': robot_description, 'use_sim_time': True}],
            output='screen'),
        Node(
            package='ros_gz_bridge', executable='parameter_bridge',
            arguments=['/clock@rosgraph_msgs/msg/Clock[gz.msgs.Clock'],
            output='screen'),
        spawn,
        TimerAction(period=5.0, actions=[joint_broadcaster, drive_controller]),
        Node(
            package='rviz2', executable='rviz2',
            parameters=[{'use_sim_time': True}],
            condition=IfCondition(LaunchConfiguration('rviz'))),
    ])
```

Build and launch with the Gazebo GUI:

```bash
cd ~/robotics_ws
rosdep install --from-paths src --ignore-src -r -y
colcon build --packages-select week04_sim
source install/setup.bash
ros2 launch week04_sim sim.launch.py
```

Or use the Week 0 headless path:

```bash
ros2 launch week04_sim sim.launch.py gz_args:="-s -r empty.sdf" rviz:=false
```

Wait for both spawners to report active, then inspect in another terminal:

```bash
source ~/robotics_ws/install/setup.bash
ros2 control list_controllers
ros2 control list_hardware_interfaces
ros2 topic info /diff_drive_controller/cmd_vel -v
timeout 8 ros2 topic hz /diff_drive_controller/odom
ros2 topic echo /joint_states --once
```

Both controllers must be `active`; both wheel velocity command interfaces must be claimed by `diff_drive_controller`.

## Lab 4D — Command and observe

In Terminal A watch forward velocity:

```bash
ros2 topic echo /diff_drive_controller/odom \
  --field twist.twist.linear.x
```

In Terminal B publish a command lease at 20 Hz for five seconds:

```bash
timeout 5 ros2 topic pub -r 20 \
  /diff_drive_controller/cmd_vel geometry_msgs/msg/TwistStamped \
  "{twist: {linear: {x: 0.20}, angular: {z: 0.0}}}"
```

Exit code 124 from `timeout` is expected. The robot should drive forward and then stop. Inspect the final estimate:

```bash
ros2 topic echo /diff_drive_controller/odom --once
ros2 topic echo /diff_drive_controller/cmd_vel_out --once
```

## Deliberate failure injection 1 — command the wrong topic

Publish continuously to the common but incorrect root name in Terminal A:

```bash
ros2 topic pub -r 5 /cmd_vel geometry_msgs/msg/TwistStamped \
  "{twist: {linear: {x: 0.20}, angular: {z: 0.0}}}"
```

While it is still running, inspect from Terminal B, then press Ctrl+C in Terminal A:

```bash
ros2 topic info /cmd_vel -v
ros2 topic list | grep cmd_vel
```

The robot must not move and `/cmd_vel` must show zero subscribers. The correct controller-private topic shows a subscriber. This is a namespace defect, not a physics or QoS defect.

## Deliberate failure injection 2 — measure the stale-command stop

Create the temporary measurement probe below as `/tmp/watchdog_probe.py`. It measures from the last nonzero input command to odometry returning below 0.01 m/s:

```python
import time

import rclpy
from geometry_msgs.msg import TwistStamped
from nav_msgs.msg import Odometry

rclpy.init()
node = rclpy.create_node('watchdog_probe')
state = {'last_cmd': None, 'saw_motion': False, 'done': False}


def command(message):
    if abs(message.twist.linear.x) > 0.05:
        state['last_cmd'] = time.monotonic()


def odometry(message):
    speed = abs(message.twist.twist.linear.x)
    if speed > 0.05:
        state['saw_motion'] = True
    if (state['saw_motion'] and state['last_cmd'] is not None and
            speed < 0.01 and not state['done']):
        state['done'] = True
        milliseconds = (time.monotonic() - state['last_cmd']) * 1000.0
        print(f'WATCHDOG_STOP_MS={milliseconds:.1f}', flush=True)
        rclpy.shutdown()


node.create_subscription(
    TwistStamped, '/diff_drive_controller/cmd_vel', command, 10)
node.create_subscription(
    Odometry, '/diff_drive_controller/odom', odometry, 10)
node.create_timer(15.0, lambda: (print('TIMEOUT', flush=True), rclpy.shutdown()))
rclpy.spin(node)
```

Run the probe in Terminal A, then the command in Terminal B:

```bash
# Terminal A
source ~/robotics_ws/install/setup.bash
python3 /tmp/watchdog_probe.py | tee \
  ~/robotics_ws/evidence/week04/watchdog.txt
```

```bash
# Terminal B
timeout 3 ros2 topic pub -r 20 \
  /diff_drive_controller/cmd_vel geometry_msgs/msg/TwistStamped \
  "{twist: {linear: {x: 0.20}, angular: {z: 0.0}}}"
```

Expected stop latency is roughly the 500 ms timeout plus deceleration and scheduling, and must be at most 850 ms on an idle VM. If it says `TIMEOUT`, first confirm `cmd_vel_out` exists and the controller is active.

## Assignment — controlled motion report

Reset by restarting the simulation. Record odometry while executing these segments, always publishing at 20 Hz: forward 0.20 m/s for 5 s; stop for 1 s; rotate +0.50 rad/s for 3.14 s; stop for 1 s. Use a clean Ctrl+C to end the bag:

```bash
mkdir -p ~/robotics_ws/evidence/week04/bags
ros2 bag record -o ~/robotics_ws/evidence/week04/bags/motion \
  /diff_drive_controller/odom /diff_drive_controller/cmd_vel_out /tf
```

Complete this table:

| Measurement | Your result | Target/interpretation |
|---|---:|---|
| Odometry publish rate |  | 45–55 Hz |
| 5 s forward final x |  | 0.90–1.10 m |
| 5 s forward absolute y drift |  | <0.05 m |
| 5 s forward absolute yaw drift |  | <0.05 rad |
| 3.14 s turn yaw change |  | 1.40–1.75 rad |
| Watchdog stop latency |  | ≤850 ms |
| Wrong-topic displacement |  | 0.00 m within odom noise |

Explain how wheel radius affects linear scale, wheel separation affects turn scale, and why simulation odometry can still differ from the commanded integral.

## Evidence and deliverables

- Final URDF/Xacro, controller YAML, and launch file.
- `watchdog.txt`, the motion bag, completed measurement table, and controller/hardware-interface listings.
- GUI screenshot or headless launch log proving the model and both active controllers.
- A failure note showing `/cmd_vel` had zero subscribers and naming the correct topic/type.

## Objective exit criteria

- The robot spawns without URDF/plugin errors; both controllers are active and wheel command interfaces are claimed.
- `/diff_drive_controller/odom` is 45–55 Hz and the forward/turn measurements meet the table bounds.
- Publishing to `/cmd_vel` causes no motion; publishing `TwistStamped` to the controller topic does.
- When the 20 Hz command stream ends, measured speed falls below 0.01 m/s within 850 ms without an explicit zero command.
- You can point to the exact YAML line that enforces stale-command behavior and explain why it remains necessary on real hardware.

## Troubleshooting

| Symptom | Diagnosis | Fix |
|---|---|---|
| `controller_manager` never appears | Gazebo plugin did not load | Inspect launch output for `libgz_ros2_control-system.so`; verify package install and Xacro `<parameters>` path. |
| Controllers remain unconfigured | spawner raced model insertion or YAML invalid | Wait/retry `ros2 run controller_manager spawner ...`; check YAML indentation and controller types. |
| Controller activation fails | joint interfaces/names mismatch | Compare YAML wheel names, URDF joints, and `ros2 control list_hardware_interfaces`. |
| Robot moves backward | wheel axis/sign or geometry orientation | Compare both joint axes and cylinder rotations; do not mask with a negative wheel radius. |
| Robot falls/explodes | invalid collision/inertia or spawn intersection | Re-run `check_urdf`, inspect positive inertia, and increase spawn `-z` slightly. |
| Odometry TF conflicts | two publishers use the same child | Keep controller `base_frame_id: base_footprint`; robot-state publisher owns `base_footprint -> base_link`. |
| No motion from command | wrong topic/type or inactive controller | Use `ros2 topic info -v`, exact `TwistStamped` type, and controller-private topic. |
| Watchdog never stops | timeout disabled or another publisher exists | Query `cmd_vel_timeout`, inspect publisher count, and stop teleop/other command nodes. |
| Headless camera/render issue | no GUI/render context | This week needs physics only; use `gz_args:="-s -r empty.sdf"`. Week 5 gives sensor-specific fallbacks. |

## Next step

Week 5 introduces camera, lidar, and IMU streams, their sensor-data QoS, and rosbag recording/replay so failures can be reproduced without a live simulator.
