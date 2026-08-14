# Week 5 — Sensors, rosbag, and replay

[Previous: Week 4](week-04-gazebo-control-and-watchdogs.md) · [Curriculum index](README.md) · [Repository workflow](../repository-workflow.md) · [Next: Week 6](week-06-slam-and-map-creation.md)

**Time:** 10–14 hours. **Build:** an inspect–measure–record–replay workflow for lidar, camera, and IMU data, including a reproducible sensor outage.

## Outcomes

By the end of this week you can:

- interpret the essential fields and frame/timestamp contracts of `LaserScan`, `Image`, `CameraInfo`, and `Imu`;
- discover the QoS actually offered by a sensor endpoint and request a compatible subscription;
- measure sensor rate and bandwidth without relying on a GUI;
- record MCAP bags, inspect metadata, stop acquisition cleanly, and replay without Gazebo;
- inject a sensor outage, preserve it in a bag, and detect the gap during replay.

## Prerequisites

- Week 4 simulation and watchdog pass.
- At least 8 GB free disk. Raw images grow quickly; the required camera capture is intentionally short.
- Gazebo's rendering path from Week 0 for camera and GPU lidar. IMU does not require a rendering sensor.

## Public readings

1. [ROS–Gazebo simulation demos for Jazzy](https://docs.ros.org/en/jazzy/p/ros_gz_sim_demos/)
2. [`sensor_msgs` message definitions](https://docs.ros.org/en/jazzy/p/sensor_msgs/interfaces/msg.html)
3. [Recording and playing back data with rosbag2](https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Recording-And-Playing-Back-Data/Recording-And-Playing-Back-Data.html)
4. [rosbag2 storage and QoS documentation](https://github.com/ros2/rosbag2)
5. [ROS 2 sensor-data QoS profile](https://docs.ros.org/en/jazzy/Concepts/Intermediate/About-Quality-of-Service-Settings.html#qos-profiles)

## Concepts

Every sensor sample needs three contracts:

1. **Schema:** units and meaning of fields. `LaserScan.ranges` is metres; `Imu.angular_velocity` is rad/s; image bytes require `height`, `width`, `step`, and `encoding`.
2. **Space:** `header.frame_id` says which coordinate frame expresses the measurement. Calibration connects that frame to the robot.
3. **Time:** `header.stamp` says when the measurement was valid. Arrival time is later and includes transport/scheduling delay.

High-rate sensors commonly use volatile, best-effort, shallow-history QoS so new data is favored over retransmission. Do not assume: inspect `ros2 topic info -v`, then match the offered profile. A bag is not merely a video—it preserves typed topics and timing so algorithms can be debugged without the original live system. Stop a recorder with Ctrl+C so metadata and indexes are finalized.

## Environment and packages

```bash
source /opt/ros/jazzy/setup.bash
sudo apt update
sudo apt install -y \
  ros-jazzy-ros-gz-sim-demos ros-jazzy-ros-gz-bridge \
  ros-jazzy-rqt-image-view ros-jazzy-rviz2 \
  ros-jazzy-rosbag2 ros-jazzy-rosbag2-storage-mcap
mkdir -p ~/robotics_ws/evidence/week05/bags
```

These labs use maintained public demo worlds so the topic names and message bridges are reproducible on Jazzy/Harmonic. Stop each demo before starting the next; otherwise multiple Gazebo servers and same-named topics make measurements ambiguous.

## Lab 5A — Lidar: geometry, rate, and QoS

Start the official lidar demo:

```bash
# Terminal A
source /opt/ros/jazzy/setup.bash
ros2 launch ros_gz_sim_demos gpu_lidar_bridge.launch.py rviz:=false
```

For a VM where the Gazebo client is unusable, replace that launch with the two-terminal server-only form:

```bash
# Terminal A: locate the installed public demo and run only the server
LIDAR_WORLD="$(find /opt/ros/jazzy/share -name gpu_lidar_sensor.sdf -print -quit)"
test -n "${LIDAR_WORLD}"
LIBGL_ALWAYS_SOFTWARE=1 gz sim -s -r "${LIDAR_WORLD}"
```

```bash
# Terminal B: bridge the same two topics as the official launch file
ros2 run ros_gz_bridge parameter_bridge \
  'lidar@sensor_msgs/msg/LaserScan@gz.msgs.LaserScan' \
  '/lidar/points@sensor_msgs/msg/PointCloud2@gz.msgs.PointCloudPacked'
```

In a separate terminal, inspect before visualizing:

```bash
source /opt/ros/jazzy/setup.bash
ros2 topic list -t | grep lidar
ros2 topic info /lidar -v | tee \
  ~/robotics_ws/evidence/week05/lidar-endpoint.txt
ros2 interface show sensor_msgs/msg/LaserScan
ros2 topic echo /lidar --once --qos-durability volatile
timeout 12 ros2 topic hz /lidar --window 100
timeout 12 ros2 topic bw /lidar
```

Record `header.frame_id`, angular minimum/maximum/increment, range minimum/maximum, number of ranges, measured Hz, offered reliability, and offered durability. Optionally open RViz, set Fixed Frame to the message's frame, and add **LaserScan** `/lidar` and **PointCloud2** `/lidar/points`.

### Deliberate QoS failure — request unavailable durability

The bridge's live sensor publisher is volatile. Requesting transient-local durability asks for stored historical samples the publisher cannot offer:

```bash
timeout 5 ros2 topic echo /lidar \
  --qos-durability transient_local
```

Expected: an incompatible-QoS warning and zero samples. Now change only the request:

```bash
ros2 topic echo /lidar --once --qos-durability volatile
```

Save both outputs. This failure has a topic, matching type, and discovered endpoints; the mismatch is the durability policy.

## Lab 5B — Camera: pixels and calibration

Stop the lidar demo and start the official camera demo:

```bash
# Terminal A
ros2 launch ros_gz_sim_demos camera.launch.py rviz:=false
```

Headless alternative:

```bash
# Terminal A
CAMERA_WORLD="$(find /opt/ros/jazzy/share -name camera_sensor.sdf -print -quit)"
test -n "${CAMERA_WORLD}"
LIBGL_ALWAYS_SOFTWARE=1 gz sim -s -r "${CAMERA_WORLD}"
```

```bash
# Terminal B
ros2 run ros_gz_bridge parameter_bridge \
  '/camera@sensor_msgs/msg/Image@gz.msgs.Image' \
  '/camera_info@sensor_msgs/msg/CameraInfo@gz.msgs.CameraInfo'
```

Inspect the contracts and load:

```bash
ros2 topic info /camera -v | tee \
  ~/robotics_ws/evidence/week05/camera-endpoint.txt
ros2 topic echo /camera --once --no-arr
ros2 topic echo /camera_info --once
timeout 12 ros2 topic hz /camera --window 100
timeout 12 ros2 topic bw /camera
ros2 run rqt_image_view rqt_image_view
```

Select `/camera` in `rqt_image_view`. In your report record image `width`, `height`, `encoding`, `step`, frame ID, measured rate/bandwidth, and the `CameraInfo.k` intrinsic matrix. `--no-arr` suppresses the large byte array but retains metadata.

If software rendering cannot produce camera frames, verify that `glxinfo -B` works, keep `LIBGL_ALWAYS_SOFTWARE=1`, and reduce the VM's other graphics load. A headless server still needs a rendering engine for camera/GPU lidar sensors; “headless” removes the client window, not sensor rendering.

## Lab 5C — IMU: vectors and covariance

Stop the camera demo and start IMU:

```bash
ros2 launch ros_gz_sim_demos imu.launch.py rqt:=false
```

The IMU bridge publishes `/imu`. Inspect it:

```bash
ros2 topic info /imu -v | tee \
  ~/robotics_ws/evidence/week05/imu-endpoint.txt
ros2 interface show sensor_msgs/msg/Imu
ros2 topic echo /imu --once
timeout 12 ros2 topic hz /imu --window 200
timeout 12 ros2 topic bw /imu
```

Record frame ID, orientation quaternion, angular velocity, linear acceleration, all three covariance arrays, rate, and bandwidth. A covariance whose first element is `-1` means that estimate is unavailable; a matrix of zeros means covariance is unknown, not perfect certainty.

For a headless manual launch, locate `sensors.sdf`, run it server-only, and bridge exactly:

```bash
IMU_WORLD="$(find /opt/ros/jazzy/share -name sensors.sdf -print -quit)"
test -n "${IMU_WORLD}"
gz sim -s -r "${IMU_WORLD}"
```

```bash
ros2 run ros_gz_bridge parameter_bridge \
  '/imu@sensor_msgs/msg/Imu@gz.msgs.IMU'
```

## Lab 5D — Record a clean MCAP bag

Return to the lidar demo. With `/lidar` publishing, run:

```bash
ros2 bag record -s mcap \
  -o ~/robotics_ws/evidence/week05/bags/lidar-clean \
  /lidar /lidar/points
```

Let it record for 20 seconds and press Ctrl+C **once**. Inspect the finalized metadata:

```bash
ros2 bag info ~/robotics_ws/evidence/week05/bags/lidar-clean | tee \
  ~/robotics_ws/evidence/week05/lidar-clean-info.txt
du -sh ~/robotics_ws/evidence/week05/bags/lidar-clean
```

Write down duration, message count by topic, serialization format, storage identifier (`mcap`), and disk size. If rosbag reports a QoS incompatibility, do not ignore it: compare recorder subscriptions with the offered endpoints and use a QoS override file as documented by rosbag2.

## Lab 5E — Replay with no simulator

Stop Gazebo and the bridge completely. Confirm the sensor publisher is gone:

```bash
ros2 topic info /lidar -v
```

Start a subscriber in Terminal A and playback in Terminal B:

```bash
# Terminal A
ros2 topic echo /lidar --once --qos-durability volatile
```

```bash
# Terminal B
ros2 bag play ~/robotics_ws/evidence/week05/bags/lidar-clean \
  --clock --rate 1.0
```

The sample must arrive with no Gazebo process. Run playback again while measuring `ros2 topic hz /lidar`; it should approximate the recorded rate. `--clock` publishes playback time for nodes configured with `use_sim_time`.

## Deliberate failure injection — preserve a sensor outage

Start the lidar demo, then start one continuous recording:

```bash
ros2 bag record -s mcap \
  -o ~/robotics_ws/evidence/week05/bags/lidar-outage \
  /lidar
```

Follow this exact timeline and note wall-clock times:

1. Record healthy lidar for 10 seconds.
2. Stop the demo/bridge cleanly with Ctrl+C; leave the recorder running for 10 seconds.
3. Restart the same lidar demo; wait for discovery and record another 10 seconds.
4. Stop the recorder cleanly with Ctrl+C.

Create `/tmp/lidar_gap_detector.py`:

```python
import time

import rclpy
from rclpy.node import Node
from rclpy.qos import qos_profile_sensor_data
from sensor_msgs.msg import LaserScan


class GapDetector(Node):
    def __init__(self):
        super().__init__('lidar_gap_detector')
        self.count = 0
        self.last = None
        self.max_gap = 0.0
        self.create_subscription(
            LaserScan, '/lidar', self.sample, qos_profile_sensor_data)

    def sample(self, _message):
        now = time.monotonic()
        if self.last is not None:
            gap = now - self.last
            self.max_gap = max(self.max_gap, gap)
            if gap > 0.5:
                self.get_logger().warn(f'GAP_SECONDS={gap:.3f}')
        self.last = now
        self.count += 1


rclpy.init()
node = GapDetector()
try:
    rclpy.spin(node)
except KeyboardInterrupt:
    pass
finally:
    print(f'COUNT={node.count} MAX_GAP_SECONDS={node.max_gap:.3f}', flush=True)
    node.destroy_node()
    rclpy.shutdown()
```

With no simulator running, replay the outage bag at 1× while the detector runs:

```bash
# Terminal A
python3 /tmp/lidar_gap_detector.py | tee \
  ~/robotics_ws/evidence/week05/outage-replay.txt
```

```bash
# Terminal B
ros2 bag play ~/robotics_ws/evidence/week05/bags/lidar-outage \
  --clock --rate 1.0
```

After playback finishes, press Ctrl+C in Terminal A. The detector must report a gap of at least 8 seconds. This reproduces the failure without Gazebo and proves why recorded timing is operational evidence.

## Assignment — sensor and evidence report

Complete the table using measured values, not demo defaults:

| Stream | ROS type | Frame | Offered QoS | Rate Hz | Bandwidth | Key validity check |
|---|---|---|---|---:|---:|---|
| `/lidar` | `LaserScan` |  |  |  |  | finite ranges within min/max |
| `/lidar/points` | `PointCloud2` |  |  |  |  | nonzero width and point step |
| `/camera` | `Image` |  |  |  |  | `step × height` agrees with data size |
| `/camera_info` | `CameraInfo` |  |  |  |  | nonzero focal lengths in `k` |
| `/imu` | `Imu` |  |  |  |  | quaternion/covariance interpreted |

Also produce:

- the 20-second clean lidar MCAP bag and its `ros2 bag info` output;
- the outage MCAP bag and replay detector output;
- a 5-second camera bag containing `/camera` and `/camera_info`, with size and message counts;
- a 20-second IMU bag, with measured count versus `duration × measured_rate` and percent difference;
- a paragraph distinguishing measurement timestamp, arrival time, bag record time, and replay time.

Use these exact recorder commands while the corresponding demo is active; stop each with Ctrl+C at the requested duration, then run `ros2 bag info` on its directory:

```bash
ros2 bag record -s mcap \
  -o ~/robotics_ws/evidence/week05/bags/camera-5s \
  /camera /camera_info
```

```bash
ros2 bag record -s mcap \
  -o ~/robotics_ws/evidence/week05/bags/imu-20s \
  /imu
```

## Evidence and deliverables

- Three endpoint files, two lidar bag directories, short camera/IMU bags, and all `ros2 bag info` outputs under `~/robotics_ws/evidence/week05`.
- Completed sensor table and bag-size/count table in `report.md`.
- `outage-replay.txt` showing the preserved gap.
- One RViz lidar screenshot and one camera screenshot, or documented CLI evidence plus the exact graphics blocker and fallback attempted.
- QoS failure output showing transient-local rejection and volatile recovery.

## Objective exit criteria

- Lidar, camera, camera-info, and IMU each publish; their type, frame, offered QoS, measured rate, and bandwidth are documented.
- The transient-local lidar subscriber receives zero samples; changing only durability to volatile restores delivery.
- `ros2 bag info` identifies MCAP, reports nonzero counts for every requested topic, and the clean lidar duration is 18–22 seconds.
- Clean lidar replay produces samples and approximately the recorded rate with Gazebo stopped.
- Outage replay reports a gap of at least 8 seconds, bounded by healthy samples before and after it.
- Camera and IMU bag sizes/counts are reported and all recorders were stopped cleanly.

## Troubleshooting

| Symptom | Diagnosis | Fix |
|---|---|---|
| Sensor topic is absent | demo, Gazebo sensor, or bridge failed | Inspect launch output, then `gz topic -l`; verify exact official launch and bridge type strings. |
| Lidar/camera absent only headless | rendering engine unavailable | Use `LIBGL_ALWAYS_SOFTWARE=1`, verify `glxinfo -B`, reduce load; server-only still needs rendering for these sensors. |
| Topic exists but echo is silent | QoS mismatch | Inspect `ros2 topic info -v`; match offered reliability and durability explicitly. |
| Camera output floods terminal | byte array printed | Use `ros2 topic echo /camera --once --no-arr` or `rqt_image_view`. |
| Bag directory already exists | rosbag refuses overwrite | Choose a new descriptive output name; preserve previous evidence instead of deleting it. |
| Bag has zero messages | recorder subscription was incompatible or started too soon | Verify endpoint and recorder QoS, wait for discovery, then use a new bag name. |
| Bag metadata missing after interruption | recorder was killed, not stopped cleanly | Preserve it, copy before experimenting, then try `ros2 bag reindex <bag-directory>`; repeat the required capture cleanly. |
| Replay shows no data | subscriber started late or QoS mismatch | Start subscriber first, play at 1×, and inspect playback publisher QoS. |
| Outage bag has only the first segment | restarted bridge not rediscovered | Confirm recorder remains alive and `ros2 topic info -v` shows its subscription before the final 10 seconds. |
| Disk fills quickly | raw image volume | Keep the required camera bag to 5 seconds, check `df -h`, and never record all topics blindly. |

## Next step

Week 6 uses the lidar stream, TF tree, odometry, timing discipline, and bag evidence from these labs to build and assess a map with SLAM.
