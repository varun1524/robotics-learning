# Week 0 — Environment and the ROS 2 graph

[Course home](../../README.md) · [Curriculum index](README.md) · [Repository workflow](../repository-workflow.md) · [Next: Week 1](week-01-topics-timing-and-qos.md)

**Time:** 6–10 hours. **Target:** an ARM64 Ubuntu 24.04 VM on Apple silicon, ROS 2 Jazzy, and Gazebo Harmonic.

## Outcomes

By the end of this week you can:

- verify the CPU architecture, OS, ROS distribution, Gazebo version, and graphics path;
- explain nodes, topics, publishers, subscribers, discovery, and the ROS graph from direct observation;
- run and inspect a multi-node graph from the command line and `rqt_graph`;
- keep working when accelerated 3D, the Gazebo GUI, or RViz rendering fails;
- produce a small, reproducible environment report rather than “it works on my machine.”

## Prerequisites

- An Apple-silicon Mac and an Ubuntu **24.04 ARM64** VM with internet access and a desktop session.
- Recommended VM allocation: 4–8 virtual CPUs, 12–16 GB RAM, at least 50 GB free disk, and 3D acceleration enabled if the VM product supports it.
- Comfort opening three terminals and using `sudo`.

Do not install Gazebo Classic. It is end-of-life; Jazzy's recommended simulator pairing is Gazebo Harmonic.

## Public readings

1. [ROS 2 Jazzy installation on Ubuntu](https://docs.ros.org/en/jazzy/Installation/Ubuntu-Install-Debs.html)
2. [ROS 2 beginner CLI tutorials](https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools.html)
3. [ROS graph concepts](https://docs.ros.org/en/jazzy/Concepts/Basic/About-Nodes.html)
4. [Gazebo and ROS installation pairing](https://gazebosim.org/docs/harmonic/ros_installation/)
5. [Webots installation](https://cyberbotics.com/doc/guide/installation-procedure) and [system requirements](https://cyberbotics.com/doc/guide/system-requirements)

## Concepts to understand before typing

A **node** is one process-level participant in a distributed computation. A **topic** is a named, typed stream; publishers offer samples and subscribers request them. Discovery builds the **ROS graph** dynamically—there is no central ROS 1 master. DDS handles discovery and transport. The graph tells you what is connected, but a name match alone is insufficient: message types and Quality of Service (QoS) must also be compatible.

RViz visualizes ROS data. It is not a physics simulator. Gazebo owns simulated world state, physics, and sensors; a bridge or ROS-aware plugin exposes selected data to ROS.

## Environment and packages

### Lab 0A — Install a pinned baseline

First prove that this is the intended guest OS and architecture:

```bash
uname -m
lsb_release -ds
df -h /
```

The first command must print `aarch64`, and the OS must be Ubuntu 24.04. If it prints `x86_64`, stop: you created an emulated Intel VM and will waste time diagnosing the wrong platform.

Install ROS from its official apt source, then the supported Gazebo integration packages:

```bash
sudo apt update
sudo apt install -y software-properties-common curl
sudo add-apt-repository universe -y

export ROS_APT_SOURCE_VERSION="$(curl -s https://api.github.com/repos/ros-infrastructure/ros-apt-source/releases/latest | grep -F 'tag_name' | awk -F'"' '{print $4}')"
curl -L -o /tmp/ros2-apt-source.deb \
  "https://github.com/ros-infrastructure/ros-apt-source/releases/download/${ROS_APT_SOURCE_VERSION}/ros2-apt-source_${ROS_APT_SOURCE_VERSION}.$(. /etc/os-release && echo "${UBUNTU_CODENAME:-${VERSION_CODENAME}}")_all.deb"
sudo dpkg -i /tmp/ros2-apt-source.deb
sudo apt update
sudo apt install -y \
  ros-jazzy-desktop ros-dev-tools ros-jazzy-ros-gz \
  ros-jazzy-demo-nodes-cpp ros-jazzy-demo-nodes-py \
  ros-jazzy-rqt-graph ros-jazzy-turtlesim mesa-utils git
```

Source ROS now and make future Bash terminals source it exactly once:

```bash
source /opt/ros/jazzy/setup.bash
grep -qxF 'source /opt/ros/jazzy/setup.bash' ~/.bashrc || \
  echo 'source /opt/ros/jazzy/setup.bash' >> ~/.bashrc

if [ ! -e /etc/ros/rosdep/sources.list.d/20-default.list ]; then
  sudo rosdep init
fi
rosdep update

mkdir -p ~/robotics_ws/src ~/robotics_ws/evidence/week00
```

Capture the baseline:

```bash
{
  date -Is
  uname -a
  lsb_release -a
  printenv ROS_DISTRO
  ros2 doctor --report
  gz sim --versions
} | tee ~/robotics_ws/evidence/week00/environment.txt
```

Expected: `ROS_DISTRO=jazzy`; the Gazebo list includes an 8.x release, which is Harmonic's `gz-sim` major version.

## Lab 0B — Validate graphics, then choose a fallback deliberately

Inspect the OpenGL path inside the VM:

```bash
echo "session=${XDG_SESSION_TYPE:-unknown} display=${DISPLAY:-unset}"
glxinfo -B | tee ~/robotics_ws/evidence/week00/opengl.txt
```

Look for `direct rendering: Yes` and OpenGL 3.3 or newer. An `llvmpipe` renderer means Mesa software rendering: valid but slow. Now test Gazebo's GUI:

```bash
gz sim -v 4 shapes.sdf
```

Let it run for 20 seconds, interact with the camera, then press Ctrl+C. Save a screenshot if it renders correctly.

If the window is black, crashes, or is unusably slow, try software rendering once:

```bash
LIBGL_ALWAYS_SOFTWARE=1 gz sim -v 4 shapes.sdf
```

If that is still poor, adopt the headless path for the rest of the course. Run each command in its own terminal:

```bash
# Terminal A: physics server only
gz sim -s -r -v 4 shapes.sdf
```

```bash
# Terminal B: prove that the server is alive
gz topic -l | tee ~/robotics_ws/evidence/week00/gz-topics.txt
```

```bash
# Terminal C: RViz through Mesa software rendering
source /opt/ros/jazzy/setup.bash
LIBGL_ALWAYS_SOFTWARE=1 rviz2
```

An empty RViz window is success here; there is not yet any robot data to display. Weeks 3–5 connect RViz to real ROS topics while Gazebo remains headless.

### Optional Webots visualization on Apple silicon

The Webots Linux desktop download is x86-64, so do **not** install its `.deb` inside the ARM64 VM. If Gazebo's GUI is impractical, you may install the universal Webots `.dmg` on the **macOS host** from the [official releases](https://github.com/cyberbotics/webots/releases) and use its built-in tutorials for visual exploration. This is not a supported course simulator mode: host Webots is not connected to the Ubuntu ROS graph, does not replace Gazebo physics, and cannot satisfy later ROS/Gazebo acceptance criteria. Remote-controller networking is a separate integration project.

Record your selected course path in `~/robotics_ws/evidence/week00/environment.txt`: accelerated Gazebo, software-rendered Gazebo, or headless Gazebo plus RViz. Record host Webots separately as an optional visualization supplement if you installed it.

## Lab 0C — Observe a ROS graph

Open three fresh terminals. In Terminal A:

```bash
source /opt/ros/jazzy/setup.bash
ros2 run demo_nodes_cpp talker
```

In Terminal B:

```bash
source /opt/ros/jazzy/setup.bash
ros2 run demo_nodes_py listener
```

In Terminal C, inventory and measure the live system:

```bash
source /opt/ros/jazzy/setup.bash
ros2 node list
ros2 topic list -t
ros2 node info /talker
ros2 node info /listener
ros2 topic info /chatter -v
ros2 topic type /chatter
ros2 interface show std_msgs/msg/String
timeout 8 ros2 topic hz /chatter
ros2 topic echo /chatter --once
rqt_graph
```

Take a screenshot of `rqt_graph` showing `/talker`, `/listener`, and `/chatter`. Then close `rqt_graph`.

## Deliberate failure injection — remove a publisher

In Terminal A press Ctrl+C. Do not restart it yet. In Terminal C run:

```bash
ros2 topic info /chatter -v
timeout 5 ros2 topic echo /chatter
```

The topic may remain visible briefly because discovery is eventually consistent, but publisher count must converge to zero and no new samples arrive. Restart the talker and measure recovery:

```bash
# Terminal A
ros2 run demo_nodes_cpp talker
```

The listener should resume without being restarted. Write down how many seconds discovery and recovery took. This is your first evidence that the graph is dynamic rather than centrally configured.

## Assignment

Create `~/robotics_ws/evidence/week00/report.md` containing:

1. VM product, CPU/RAM/disk allocation, `uname -m`, Ubuntu version, ROS distribution, and Gazebo version.
2. The Gazebo graphics mode you selected and any optional visualization supplement.
3. A hand-drawn or Mermaid diagram of `/talker -> /chatter -> /listener` with node, topic, and message type labels.
4. The output of `ros2 topic info /chatter -v` while healthy and after killing the publisher.
5. A paragraph explaining why RViz and Gazebo are different tools.

### Measurements

| Measurement | Command/method | Your result | Target |
|---|---|---:|---:|
| Talker publish rate | `timeout 8 ros2 topic hz /chatter` |  | 0.9–1.1 Hz |
| Healthy publisher count | `ros2 topic info /chatter -v` |  | 1 |
| Failed publisher count | same, after Ctrl+C |  | 0 |
| Discovery recovery | stopwatch: restart to first received sample |  | under 5 s |
| OpenGL renderer | `glxinfo -B` |  | documented |

## Evidence and deliverables

- `environment.txt`, `opengl.txt`, and `gz-topics.txt` where applicable.
- `report.md` with the completed measurement table.
- One ROS graph screenshot and one simulator/RViz screenshot.
- A short `README` note stating the exact fallback command needed on your VM.

## Objective exit criteria

Move on only when all are true:

- `uname -m` is `aarch64`, `ROS_DISTRO` is `jazzy`, and Gazebo reports an 8.x `gz-sim` version.
- The listener receives samples, `/chatter` measures 0.9–1.1 Hz, and you can identify both endpoints with `ros2 topic info -v`.
- Killing the publisher produces zero new messages; restarting it restores delivery without restarting the listener.
- You have a repeatable Gazebo path: native GUI, software-rendered GUI, or headless Gazebo plus RViz. Optional host Webots use is documented separately and is not counted as ROS/Gazebo evidence.
- Every requested evidence artifact exists.

## Troubleshooting

| Symptom | Check | Fix |
|---|---|---|
| `ros2: command not found` | `echo $ROS_DISTRO` | `source /opt/ros/jazzy/setup.bash`; open a new Bash terminal after editing `.bashrc`. |
| ROS apt package is unavailable | `lsb_release -sc; uname -m` | Confirm Ubuntu Noble and `aarch64`; reinstall the official `ros2-apt-source` package. |
| Nodes cannot discover each other | `echo ${ROS_DOMAIN_ID:-0}` in every terminal | Use the same domain ID and ensure both processes are in the same VM/network namespace. |
| Gazebo reports Ogre/OpenGL errors | `glxinfo -B` | Enable VM 3D acceleration, try `LIBGL_ALWAYS_SOFTWARE=1`, then use `gz sim -s`. |
| RViz is black or crashes | run from a terminal | Try `LIBGL_ALWAYS_SOFTWARE=1 rviz2`; reduce VM display scaling and close Gazebo's GUI. |
| `gz sim` is missing | `apt policy ros-jazzy-ros-gz` | `sudo apt install ros-jazzy-ros-gz`. Do not add a second Gazebo repository unless the official pairing guide requires it. |
| Webots `.deb` rejects the architecture | `dpkg --print-architecture` | Expected on ARM64. Use Gazebo headless plus RViz for the course; install the universal macOS app only for optional visual exploration. |

## Next step

Week 1 replaces the demo string with timestamped numeric samples so you can measure rate, loss, latency, and QoS compatibility rather than merely seeing messages scroll by.
