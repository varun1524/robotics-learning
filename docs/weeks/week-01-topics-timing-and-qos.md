# Week 1 — Topics, timing, and QoS

[Previous: Week 0](week-00-environment-and-ros-graph.md) · [Curriculum index](README.md) · [Repository workflow](../repository-workflow.md) · [Next: Week 2](week-02-services-actions-and-parameters.md)

**Time:** 8–12 hours. **Build:** a timestamped publisher/subscriber pair and a small communications benchmark.

## Outcomes

By the end of this week you can:

- choose a topic for streaming state and identify its message contract;
- measure publish rate, bandwidth, sequence loss, inter-arrival time, and same-host transport latency;
- explain reliability, history, depth, durability, and deadline in operational terms;
- diagnose a graph that has matching names and types but incompatible QoS;
- make a data-backed QoS choice rather than reflexively selecting “reliable.”

## Prerequisites

- Week 0 exit criteria completed.
- `~/robotics_ws` exists and every terminal sources `/opt/ros/jazzy/setup.bash`.
- Python fundamentals: class, callback, list, and formatted string.

## Public readings

1. [ROS 2 publisher/subscriber tutorial (Python)](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Py-Publisher-And-Subscriber.html)
2. [ROS 2 Quality of Service settings](https://docs.ros.org/en/jazzy/Concepts/Intermediate/About-Quality-of-Service-Settings.html)
3. [`ros2 topic` command tutorial](https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Topics/Understanding-ROS2-Topics.html)
4. [`std_msgs/Float64MultiArray` definition](https://docs.ros.org/en/jazzy/p/std_msgs/msg/Float64MultiArray.html)

## Concepts

Topics are for asynchronous streams where a producer should not wait for each consumer. **Rate** is samples per second; **bandwidth** includes serialized message volume; **latency** is send-to-receive delay; **jitter** is variation in timing; **loss** is detected here with monotonically increasing sequence numbers.

QoS is a compatibility contract, not a quality slider. A reliable subscriber cannot connect to a best-effort publisher because the publisher cannot satisfy the request. A best-effort subscriber can accept a reliable publisher. Sensor streams often prefer fresh samples over retransmitting stale ones; commands and low-rate state may justify reliability.

This lab places publisher and subscriber in one VM and uses `time.monotonic()`. The resulting latency is useful for comparisons **within this VM only**. Monotonic clocks from two computers do not share an epoch; distributed latency needs synchronized clocks or round-trip measurement.

## Environment and packages

```bash
source /opt/ros/jazzy/setup.bash
sudo apt update
sudo apt install -y python3-colcon-common-extensions python3-rosdep
mkdir -p ~/robotics_ws/src ~/robotics_ws/evidence/week01
cd ~/robotics_ws/src
ros2 pkg create --build-type ament_python --license Apache-2.0 \
  week01_topics --dependencies rclpy std_msgs
```

The rest of this lab assumes the package did not already exist. If `ros2 pkg create` reports that it does, inspect and reuse it; do not create a duplicate workspace.

## Lab 1A — Implement the timed publisher

Create `~/robotics_ws/src/week01_topics/week01_topics/timed_publisher.py`:

```python
import time

import rclpy
from rclpy.node import Node
from rclpy.qos import HistoryPolicy, QoSProfile, ReliabilityPolicy
from std_msgs.msg import Float64MultiArray


def reliability_policy(name: str):
    policies = {
        'reliable': ReliabilityPolicy.RELIABLE,
        'best_effort': ReliabilityPolicy.BEST_EFFORT,
    }
    if name not in policies:
        raise ValueError("reliability must be 'reliable' or 'best_effort'")
    return policies[name]


class TimedPublisher(Node):
    def __init__(self):
        super().__init__('timed_publisher')
        self.declare_parameter('rate_hz', 50.0)
        self.declare_parameter('reliability', 'reliable')

        rate_hz = float(self.get_parameter('rate_hz').value)
        reliability = str(self.get_parameter('reliability').value)
        if rate_hz <= 0.0:
            raise ValueError('rate_hz must be positive')

        qos = QoSProfile(
            history=HistoryPolicy.KEEP_LAST,
            depth=10,
            reliability=reliability_policy(reliability),
        )
        self.publisher = self.create_publisher(
            Float64MultiArray, '/timed_samples', qos)
        self.sequence = 0
        self.timer = self.create_timer(1.0 / rate_hz, self.publish_sample)
        self.get_logger().info(
            f'rate={rate_hz:.1f} Hz reliability={reliability} depth=10')

    def publish_sample(self):
        message = Float64MultiArray()
        message.data = [float(self.sequence), time.monotonic()]
        self.publisher.publish(message)
        self.sequence += 1


def main(args=None):
    rclpy.init(args=args)
    node = TimedPublisher()
    try:
        rclpy.spin(node)
    finally:
        node.destroy_node()
        rclpy.shutdown()


if __name__ == '__main__':
    main()
```

The two array entries are `[sequence_number, monotonic_send_time_seconds]`. A purpose-built interface is preferable in production; the standard array keeps this first instrumentation lab focused on transport.

## Lab 1B — Implement the instrumented subscriber

Create `~/robotics_ws/src/week01_topics/week01_topics/timed_subscriber.py`:

```python
import time

import rclpy
from rclpy.node import Node
from rclpy.qos import HistoryPolicy, QoSProfile, ReliabilityPolicy
from std_msgs.msg import Float64MultiArray


def reliability_policy(name: str):
    policies = {
        'reliable': ReliabilityPolicy.RELIABLE,
        'best_effort': ReliabilityPolicy.BEST_EFFORT,
    }
    if name not in policies:
        raise ValueError("reliability must be 'reliable' or 'best_effort'")
    return policies[name]


class TimedSubscriber(Node):
    def __init__(self):
        super().__init__('timed_subscriber')
        self.declare_parameter('reliability', 'reliable')
        reliability = str(self.get_parameter('reliability').value)
        qos = QoSProfile(
            history=HistoryPolicy.KEEP_LAST,
            depth=10,
            reliability=reliability_policy(reliability),
        )
        self.subscription = self.create_subscription(
            Float64MultiArray, '/timed_samples', self.receive, qos)
        self.received = 0
        self.missed = 0
        self.reordered_or_reset = 0
        self.last_sequence = None
        self.last_arrival = None
        self.latency_sum_ms = 0.0
        self.interarrival_sum_ms = 0.0
        self.get_logger().info(
            f'reliability={reliability} depth=10; reporting every 50 samples')

    def receive(self, message):
        if len(message.data) != 2:
            self.get_logger().error('expected [sequence, monotonic_send_time]')
            return

        now = time.monotonic()
        sequence = int(message.data[0])
        latency_ms = (now - message.data[1]) * 1000.0

        if self.last_sequence is not None:
            expected = self.last_sequence + 1
            if sequence > expected:
                self.missed += sequence - expected
            elif sequence < expected:
                self.reordered_or_reset += 1
        if self.last_arrival is not None:
            self.interarrival_sum_ms += (now - self.last_arrival) * 1000.0

        self.received += 1
        self.latency_sum_ms += latency_ms
        self.last_sequence = sequence
        self.last_arrival = now

        if self.received % 50 == 0:
            average_latency = self.latency_sum_ms / self.received
            average_interarrival = self.interarrival_sum_ms / max(1, self.received - 1)
            self.get_logger().info(
                f'received={self.received} missed={self.missed} '
                f'reordered_or_reset={self.reordered_or_reset} '
                f'avg_latency_ms={average_latency:.3f} '
                f'avg_interarrival_ms={average_interarrival:.3f}')


def main(args=None):
    rclpy.init(args=args)
    node = TimedSubscriber()
    try:
        rclpy.spin(node)
    finally:
        node.destroy_node()
        rclpy.shutdown()


if __name__ == '__main__':
    main()
```

## Lab 1C — Register, build, and run

Replace `~/robotics_ws/src/week01_topics/setup.py` with:

```python
from setuptools import find_packages, setup

package_name = 'week01_topics'

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
    description='Timed topic and QoS experiments',
    license='Apache-2.0',
    entry_points={
        'console_scripts': [
            'timed_publisher = week01_topics.timed_publisher:main',
            'timed_subscriber = week01_topics.timed_subscriber:main',
        ],
    },
)
```

Build and source the overlay:

```bash
cd ~/robotics_ws
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install --packages-select week01_topics
source install/setup.bash
```

Run the reliable baseline in two terminals:

```bash
# Terminal A
source ~/robotics_ws/install/setup.bash
ros2 run week01_topics timed_publisher --ros-args \
  -p rate_hz:=50.0 -p reliability:=reliable
```

```bash
# Terminal B
source ~/robotics_ws/install/setup.bash
ros2 run week01_topics timed_subscriber --ros-args \
  -p reliability:=reliable | tee ~/robotics_ws/evidence/week01/reliable-50hz.log
```

In Terminal C:

```bash
source ~/robotics_ws/install/setup.bash
ros2 topic info /timed_samples -v
timeout 12 ros2 topic hz /timed_samples --window 200
timeout 12 ros2 topic bw /timed_samples
ros2 topic echo /timed_samples --once
```

Let the pair run for at least 30 seconds before recording the subscriber's latest totals.

## Deliberate failure injection — compatible names, incompatible QoS

Stop both nodes. Start a best-effort publisher and a reliable subscriber:

```bash
# Terminal A
ros2 run week01_topics timed_publisher --ros-args \
  -p rate_hz:=50.0 -p reliability:=best_effort
```

```bash
# Terminal B
ros2 run week01_topics timed_subscriber --ros-args \
  -p reliability:=reliable
```

The subscriber should warn about an incompatible QoS policy and receive no samples. Prove the diagnosis:

```bash
ros2 topic info /timed_samples -v | tee \
  ~/robotics_ws/evidence/week01/qos-incompatible.txt
timeout 5 ros2 topic echo /timed_samples \
  --qos-reliability reliable
```

Do not “fix” the topic name—it is already correct. Stop only the subscriber and request what the publisher can offer:

```bash
ros2 run week01_topics timed_subscriber --ros-args \
  -p reliability:=best_effort
```

Samples should flow immediately. Capture the endpoint QoS again and explain the requested/offered rule in your own words.

## Assignment — communications benchmark

Run every row below for 30 seconds. Restart both nodes between rows so counters begin cleanly. On an otherwise idle VM, collect the subscriber's last report, `ros2 topic hz`, and `ros2 topic bw`.

| Publisher rate | Publisher QoS | Subscriber QoS | Measured Hz | Bandwidth | Avg latency ms | Missed | Expected |
|---:|---|---|---:|---:|---:|---:|---|
| 10 | reliable | reliable |  |  |  |  | delivery |
| 50 | reliable | reliable |  |  |  |  | delivery |
| 100 | reliable | reliable |  |  |  |  | delivery |
| 50 | best effort | best effort |  |  |  |  | delivery |
| 50 | best effort | reliable |  |  | n/a | n/a | no delivery |

Add one stress run: start `gz sim -s -r shapes.sdf` in another terminal, repeat the 100 Hz reliable test, and compare jitter/latency. Do not claim a universal performance result from one VM; explain what changed in this environment.

## Evidence and deliverables

- Source package `week01_topics` builds without warnings from your code.
- `reliable-50hz.log`, `qos-incompatible.txt`, and a completed benchmark table in `~/robotics_ws/evidence/week01/report.md`.
- A screenshot or saved output of endpoint QoS from `ros2 topic info -v`.
- A five-sentence design note: which QoS would you choose for camera frames, velocity commands, and a one-time map, and why?

## Objective exit criteria

- The 10, 50, and 100 Hz compatible runs measure within ±10% of the requested rate on an idle VM.
- The subscriber reports zero sequence loss in the 50 Hz reliable baseline for 30 seconds.
- The intentionally incompatible run receives exactly zero samples and you identify **reliability**, not discovery, type, or naming, as the cause.
- Changing only the subscriber to best effort restores delivery from the best-effort publisher.
- Your report states that the latency numbers are same-host comparative measurements, not synchronized multi-machine latency.

## Troubleshooting

| Symptom | Diagnosis | Fix |
|---|---|---|
| Package imports old code | overlay not sourced | Run `source ~/robotics_ws/install/setup.bash`; `--symlink-install` reflects Python edits. |
| Console executable not found | `setup.py` entry point missing or stale | Rebuild the package, then source the overlay in that terminal. |
| No messages in the baseline | inspect `ros2 topic info -v` | Confirm same type, domain ID, and compatible reliability on both endpoints. |
| Publisher exits at startup | invalid parameter spelling/value | Use exactly `reliable` or `best_effort`, and a positive `rate_hz`. |
| Measured latency is negative | clocks are not comparable or payload was modified | Keep both nodes in the same VM and use `time.monotonic()` at both ends. |
| `timeout` exits with code 124 | expected after measurement window | The command deliberately ended; use the statistics printed before exit. |
| 100 Hz misses badly | VM scheduling or overload | Close GUIs, repeat three times, report median and VM allocation instead of hiding the result. |

## Next step

Week 2 introduces request/response services, long-running cancellable actions, and validated parameters—the control-plane patterns that should not be modeled as continuous topics.
