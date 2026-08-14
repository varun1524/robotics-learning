# Week 3 — URDF, TF, and RViz

[Previous: Week 2](week-02-services-actions-and-parameters.md) · [Curriculum index](README.md) · [Repository workflow](../repository-workflow.md) · [Next: Week 4](week-04-gazebo-control-and-watchdogs.md)

**Time:** 10–14 hours. **Build:** a validated differential-drive robot description and a measurable TF tree.

## Outcomes

By the end of this week you can:

- model a robot as links connected by fixed and movable joints in Xacro/URDF;
- distinguish visual, collision, and inertial geometry;
- apply ROS coordinate-frame conventions and inspect transforms numerically;
- publish `robot_description`, joint states, `/tf`, and `/tf_static` and render them in RViz;
- break a kinematic tree deliberately, use validation output to locate the defect, and restore it.

## Prerequisites

- Weeks 0–2 exit criteria completed and `~/robotics_ws` sourced.
- Familiarity with Cartesian axes, translation, roll/pitch/yaw, and parent/child trees.
- A working RViz path from Week 0; headless learners can complete every numerical check without opening RViz and capture the GUI later.

## Public readings

1. [Building a visual robot model from scratch](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/URDF/Building-a-Visual-Robot-Model-with-URDF-from-Scratch.html)
2. [Using Xacro to clean up a URDF](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/URDF/Using-Xacro-to-Clean-Up-a-URDF-File.html)
3. [Introducing TF2](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Tf2/Introduction-To-Tf2.html)
4. [REP 103 coordinate conventions](https://www.ros.org/reps/rep-0103.html) and [REP 105 mobile-robot frames](https://www.ros.org/reps/rep-0105.html)
5. [RViz user guide](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/RViz/RViz-User-Guide/RViz-User-Guide.html)

## Concepts

URDF is a tree. A **link** is a rigid body; a **joint** defines the transform and allowed motion from one parent link to one child. A non-root link must have exactly one parent. **Visual** geometry is what humans see, **collision** geometry is what physics tests, and **inertial** data tells dynamics how mass is distributed. They can differ, but omitting collision or inertia will matter in Week 4.

TF answers “where is frame B relative to frame A at time t?” Fixed joints belong on `/tf_static`; movable joints are computed from `/joint_states` and published on `/tf`. Use SI units: metres, radians, kilograms. REP 103 robot body axes are x forward, y left, z up. Camera optical frames use z forward, x right, y down.

## Environment and packages

```bash
source /opt/ros/jazzy/setup.bash
sudo apt update
sudo apt install -y \
  ros-jazzy-xacro ros-jazzy-robot-state-publisher \
  ros-jazzy-joint-state-publisher ros-jazzy-joint-state-publisher-gui \
  ros-jazzy-tf2-tools ros-jazzy-rviz2 liburdfdom-tools

mkdir -p ~/robotics_ws/src ~/robotics_ws/evidence/week03
cd ~/robotics_ws/src
ros2 pkg create --build-type ament_cmake --license Apache-2.0 week03_description
mkdir -p week03_description/urdf week03_description/launch
```

Replace `week03_description/CMakeLists.txt` with:

```cmake
cmake_minimum_required(VERSION 3.8)
project(week03_description)

find_package(ament_cmake REQUIRED)

install(DIRECTORY launch urdf
  DESTINATION share/${PROJECT_NAME}
)

ament_package()
```

Replace `week03_description/package.xml` with:

```xml
<?xml version="1.0"?>
<package format="3">
  <name>week03_description</name>
  <version>0.0.0</version>
  <description>Course robot description and TF lab</description>
  <maintainer email="student@example.com">student</maintainer>
  <license>Apache-2.0</license>
  <buildtool_depend>ament_cmake</buildtool_depend>
  <exec_depend>joint_state_publisher</exec_depend>
  <exec_depend>joint_state_publisher_gui</exec_depend>
  <exec_depend>launch_ros</exec_depend>
  <exec_depend>robot_state_publisher</exec_depend>
  <exec_depend>rviz2</exec_depend>
  <exec_depend>xacro</exec_depend>
  <export><build_type>ament_cmake</build_type></export>
</package>
```

## Lab 3A — Build the Xacro model

Create `week03_description/urdf/course_bot.urdf.xacro`:

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro" name="course_bot">
  <xacro:property name="pi" value="3.141592653589793"/>
  <xacro:property name="wheel_radius" value="0.075"/>
  <xacro:property name="wheel_width" value="0.04"/>

  <material name="blue"><color rgba="0.1 0.35 0.8 1"/></material>
  <material name="dark"><color rgba="0.08 0.08 0.08 1"/></material>
  <material name="sensor"><color rgba="0.9 0.15 0.1 1"/></material>

  <xacro:macro name="box_inertial" params="mass x y z">
    <inertial>
      <mass value="${mass}"/>
      <inertia
        ixx="${mass * (y*y + z*z) / 12.0}" ixy="0" ixz="0"
        iyy="${mass * (x*x + z*z) / 12.0}" iyz="0"
        izz="${mass * (x*x + y*y) / 12.0}"/>
    </inertial>
  </xacro:macro>

  <xacro:macro name="wheel" params="side sign">
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
        <inertia ixx="0.00045" ixy="0" ixz="0"
                 iyy="0.00085" iyz="0" izz="0.00045"/>
      </inertial>
    </link>
    <joint name="${side}_wheel_joint" type="continuous">
      <parent link="base_link"/>
      <child link="${side}_wheel_link"/>
      <origin xyz="0 ${sign * 0.17} -0.045" rpy="0 0 0"/>
      <axis xyz="0 1 0"/>
    </joint>
  </xacro:macro>

  <link name="base_footprint"/>

  <link name="base_link">
    <visual>
      <geometry><box size="0.40 0.30 0.12"/></geometry>
      <material name="blue"/>
    </visual>
    <collision><geometry><box size="0.40 0.30 0.12"/></geometry></collision>
    <xacro:box_inertial mass="5.0" x="0.40" y="0.30" z="0.12"/>
  </link>
  <joint name="base_footprint_joint" type="fixed">
    <parent link="base_footprint"/>
    <child link="base_link"/>
    <origin xyz="0 0 0.12"/>
  </joint>

  <xacro:wheel side="left" sign="1"/>
  <xacro:wheel side="right" sign="-1"/>

  <link name="caster_link">
    <visual><geometry><sphere radius="0.04"/></geometry><material name="dark"/></visual>
    <collision><geometry><sphere radius="0.04"/></geometry></collision>
    <inertial><mass value="0.10"/><inertia ixx="0.000064" ixy="0" ixz="0" iyy="0.000064" iyz="0" izz="0.000064"/></inertial>
  </link>
  <joint name="caster_joint" type="fixed">
    <parent link="base_link"/><child link="caster_link"/>
    <origin xyz="-0.15 0 -0.08"/>
  </joint>

  <link name="laser_frame">
    <visual><geometry><cylinder radius="0.04" length="0.04"/></geometry><material name="sensor"/></visual>
  </link>
  <joint name="laser_joint" type="fixed">
    <parent link="base_link"/><child link="laser_frame"/>
    <origin xyz="0.12 0 0.12"/>
  </joint>

  <link name="camera_link">
    <visual><geometry><box size="0.05 0.08 0.04"/></geometry><material name="sensor"/></visual>
  </link>
  <joint name="camera_joint" type="fixed">
    <parent link="base_link"/><child link="camera_link"/>
    <origin xyz="0.10 0 0.18"/>
  </joint>

  <link name="camera_optical_frame"/>
  <joint name="camera_optical_joint" type="fixed">
    <parent link="camera_link"/><child link="camera_optical_frame"/>
    <origin rpy="-${pi/2} 0 -${pi/2}"/>
  </joint>
</robot>
```

Expand and validate before launching anything:

```bash
cd ~/robotics_ws
xacro src/week03_description/urdf/course_bot.urdf.xacro \
  > /tmp/course_bot.urdf
check_urdf /tmp/course_bot.urdf | tee evidence/week03/check-urdf.txt
```

Expected root: `base_footprint`; every other link appears below it, with no parser error.

## Lab 3B — Launch state publishers and RViz

Create `week03_description/launch/display.launch.py`:

```python
import os

from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument
from launch.conditions import IfCondition, UnlessCondition
from launch.substitutions import Command, LaunchConfiguration
from launch_ros.actions import Node
from launch_ros.parameter_descriptions import ParameterValue


def generate_launch_description():
    share = get_package_share_directory('week03_description')
    model = os.path.join(share, 'urdf', 'course_bot.urdf.xacro')
    use_gui = LaunchConfiguration('use_gui')
    use_rviz = LaunchConfiguration('rviz')
    robot_description = ParameterValue(
        Command(['xacro ', model]), value_type=str)

    return LaunchDescription([
        DeclareLaunchArgument('use_gui', default_value='true'),
        DeclareLaunchArgument('rviz', default_value='true'),
        Node(
            package='robot_state_publisher',
            executable='robot_state_publisher',
            parameters=[{'robot_description': robot_description}],
            output='screen'),
        Node(
            package='joint_state_publisher_gui',
            executable='joint_state_publisher_gui',
            condition=IfCondition(use_gui)),
        Node(
            package='joint_state_publisher',
            executable='joint_state_publisher',
            condition=UnlessCondition(use_gui)),
        Node(
            package='rviz2', executable='rviz2',
            condition=IfCondition(use_rviz), output='screen'),
    ])
```

Build, source, and launch:

```bash
cd ~/robotics_ws
colcon build --packages-select week03_description
source install/setup.bash
ros2 launch week03_description display.launch.py
```

In RViz set **Fixed Frame** to `base_footprint`, add **RobotModel**, and select `/robot_description` if prompted. Add a **TF** display. Move each wheel slider in the joint-state GUI and confirm the wheel rotates without moving the chassis.

For a VM without usable GUIs:

```bash
ros2 launch week03_description display.launch.py use_gui:=false rviz:=false
```

## Lab 3C — Measure the transform tree

With the launch running, use another sourced terminal:

```bash
ros2 node info /robot_state_publisher
ros2 topic info /joint_states -v
ros2 topic echo /tf_static --once
timeout 8 ros2 topic hz /tf
timeout 8 ros2 run tf2_ros tf2_echo base_link laser_frame
timeout 8 ros2 run tf2_ros tf2_echo camera_link camera_optical_frame
cd ~/robotics_ws/evidence/week03
ros2 run tf2_tools view_frames
```

The base-to-laser translation must be approximately `(0.12, 0.0, 0.12)`. `view_frames` writes `frames.pdf`; inspect it for a single connected tree rooted at `base_footprint`.

## Deliberate failure injection — orphan the tree

Never debug only through RViz's red status text. Create a broken temporary copy and ask the parser directly:

```bash
sed '0,/base_link/s//base_lnik/' /tmp/course_bot.urdf \
  > /tmp/course_bot-broken.urdf
set +e
check_urdf /tmp/course_bot-broken.urdf 2>&1 | tee \
  ~/robotics_ws/evidence/week03/broken-urdf.txt
BROKEN_STATUS=${PIPESTATUS[0]}
set -e
echo "broken validator exit=${BROKEN_STATUS}"
test "${BROKEN_STATUS}" -ne 0
```

The nonzero exit is the expected result. Explain which links still reference `base_link` and why renaming a link without updating its joints orphans the model. Confirm the original still passes:

```bash
check_urdf /tmp/course_bot.urdf
```

Second failure: stop only the joint-state publisher while leaving `robot_state_publisher` alive. In a sourced terminal, resolve the exact process before terminating it:

```bash
pgrep -af '[/]joint_state_publisher(_gui)?([[:space:]]|$)'
JSP_PID="$(pgrep -f '[/]joint_state_publisher(_gui)?([[:space:]]|$)' | head -n 1)"
test -n "${JSP_PID}"
kill -TERM "${JSP_PID}"
ros2 node list
```

`/robot_state_publisher` must remain listed. Open a **new** terminal and run:

```bash
timeout 5 ros2 run tf2_ros tf2_echo base_link left_wheel_link
```

The dynamic wheel transform is unavailable to the new listener, while fixed transforms remain available from `/tf_static`. Stop and restart the launch to recover.

## Assignment — add a bumper frame

Add a fixed `front_bumper_link` and joint to the Xacro. Requirements:

- box visual and collision geometry, `0.04 × 0.28 × 0.06 m`;
- center located `0.22 m` forward of `base_link` and no lower than the ground plane;
- unique material and a physically plausible positive mass/inertia;
- x forward, y left, z up; no compensating rotations just to “look right.”

Rebuild and produce the measurements below.

| Measurement | Method | Your result | Target |
|---|---|---:|---:|
| Connected components | `frames.pdf` inspection |  | 1 |
| Base → laser xyz m | `tf2_echo` |  | `(0.12, 0, 0.12)` ±1 mm |
| Base → bumper x m | `tf2_echo` |  | `0.22` ±1 mm |
| Dynamic TF rate | `ros2 topic hz /tf` |  | 8–15 Hz default |
| Broken-model validator exit | shell `$?` |  | nonzero |
| Original-model validator exit | shell `$?` |  | 0 |

## Evidence and deliverables

- `course_bot.urdf.xacro` and `display.launch.py`, including your bumper.
- `check-urdf.txt`, `broken-urdf.txt`, `frames.pdf`, completed measurement table, and RViz screenshot.
- A short explanation of `base_footprint`, `base_link`, `camera_link`, and `camera_optical_frame` roles.
- A list of which joints publish through `/tf_static` and which require `/joint_states`.

## Objective exit criteria

- Xacro expansion and `check_urdf` exit zero for the final model; the deliberate broken copy exits nonzero.
- `frames.pdf` has one connected tree, one root, and no repeated child frames.
- Numeric base-to-laser and base-to-bumper transforms meet the table tolerances.
- Moving either wheel joint updates its transform in RViz/TF without changing fixed sensor transforms.
- The robot renders in RViz or, on a headless system, all CLI/TF checks pass and a later GUI-capture plan is documented.

## Troubleshooting

| Symptom | Diagnosis | Fix |
|---|---|---|
| Xacro says an expression is invalid | malformed `${...}` or missing namespace | Keep the `xmlns:xacro` declaration and check braces/quotes. |
| RViz shows `No transform` | wrong Fixed Frame or missing joint states | Set `base_footprint`; inspect `/joint_states`, `/tf`, and `/tf_static`. |
| Robot is white/invisible | material or alpha/scale issue | Use RGBA alpha 1, metre-scale dimensions, and add RobotModel from `/robot_description`. |
| Wheel spins around wrong axis | joint axis and cylinder orientation disagree | Cylinder axis is rotated onto y; keep joint axis `0 1 0`. |
| `view_frames` has disconnected branches | typo in link/joint names | Run `check_urdf`, then compare every child with exactly one defined link. |
| Launch uses source-tree file after edits | package not rebuilt | Rebuild because CMake installs `urdf/` and `launch/`, then source the overlay. |
| GUI fails in VM | graphics path | Use `use_gui:=false rviz:=false`; later try `LIBGL_ALWAYS_SOFTWARE=1 rviz2`. |

## Next step

Week 4 adds Gazebo physics and `ros2_control`, then verifies the most important mobile-base safety behavior: commands expire and the robot stops when command delivery disappears.
