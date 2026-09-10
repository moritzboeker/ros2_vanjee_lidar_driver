# ros2_vanjee_lidar_driver

ROS2 driver for Vanjee WLR-series lidars, based on Vanjee's official `vanjee_lidar_sdk` v2.0.7 and ported to build with colcon out of the box.

Tested with a **WLR-719C** (4-layered version of WLR-719) on **ROS2 Jazzy** (Ubuntu 24.04, x86_64 and Raspberry Pi 4B/arm64). The SDK also ships configs for the 716mini, 718h, 719, 720, 721, 722, 733, 750 and 760 — see `vanjee_lidar_sdk/config/`.

## Packages

- `vanjee_lidar_sdk` — the driver node (`vanjee_lidar_sdk_node`), publishes [`sensor_msgs/msg/LaserScan`](https://docs.ros.org/en/jazzy/p/sensor_msgs/msg/LaserScan.html) or [`sensor_msgs/msg/PointCloud2`](https://docs.ros.org/en/jazzy/p/sensor_msgs/msg/PointCloud2.html) depending on the lidar model
- `vanjee_lidar_msg` — message definitions used by the SDK

## Changes compared to the official SDK

The packages were downloaded from [vanjee.net](http://www.vanjee.net/download_files/1571790315333623808.html) (v2.0.7, © 2023 Vanjee, BSD-3-Clause — see `LICENSE`) and modified:

- switched the build from ROS1/catkin to ROS2/colcon (`package_ros2.xml` → `package.xml`, `COMPILE_METHOD COLCON`)
- compile with C++17 on all ROS2 distros (required by rclcpp since Humble; the vendor only enabled it for Humble)
- added a `config_path` ROS2 parameter to `vanjee_lidar_sdk_node`, so the lidar config can live in your bringup package instead of being hardcoded at compile time

## Installation

Add the packages to your ROS2 workspace, either as a git submodule or by copying:

```bash
cd ~/your_ws/src
git submodule add https://github.com/moritzboeker/ros2_vanjee_lidar_driver.git
# or: git clone https://github.com/moritzboeker/ros2_vanjee_lidar_driver.git
```

Install dependencies and build:

```bash
cd ~/your_ws
rosdep install --from-paths src --ignore-src -y
colcon build --packages-up-to vanjee_lidar_sdk
```

## Configuration

Copy the config for your lidar model — e.g. `vanjee_lidar_sdk/config/config_719c.yaml` — into your bringup package (e.g. `my_robot_bringup/config/vanjee_wlr719c_cfg.yaml`) and adjust it. The most relevant fields are:

| Field | Meaning | Default |
|---|---|---|
| `lidar_type` | protocol variant, e.g. `vanjee_719c` | per config file |
| `lidar_address` / `lidar_msop_port` | the lidar's IP and command port | `192.168.0.2` / `6050` |
| `host_address` / `host_msop_port` | the IP the lidar streams to (your machine) and local port | `192.168.0.64` / `3002` |
| `ros_frame_id` | frame id stamped on the point cloud | `vanjee_lidar` |
| `ros_send_point_cloud_topic` | output topic | `/vanjee_points719c` |

> **Note for WLR-719 owners:** There are two different models lidar models both labled as WLR-719 - the single-layered WLR-719 and the four-layer WLR-719C. If you try running the `lidar_type: vanjee_719` (without 'c') on the WLR-719C, the driver will logs `ERRCODE_MSOPTIMEOUT` and no data will arrive. Therefore, use `config_719c.yaml` with `lidar_type: vanjee_719c` instead for the four-layer WLR-719C. Additionally, the `lidar_type: vanjee_719c` is not able to generate laser scans but only point clouds (`send_laser_scan_ros: true` does not change this). If you need a laser scan (e.g. for slam_toolbox), add a [`pointcloud_to_laserscan`](https://github.com/ros-perception/pointcloud_to_laserscan) node to convert the point cloud into a laser scan.

## Network setup

By default, the lidar has a vendor-shipped IP of `192.168.0.2` and streams only to the configured `host_address` of `192.168.0.64`. Those can be changed in Windows with a tool provided by Vanjee. But if you want to go with the defaults, your ethernet interface must own that address. On systems using NetworkManager, this sets it up persistently without touching your other networks (the `/32` + host route leave a LAN that also uses `192.168.0.x` alone):

```bash
sudo nmcli con add type ethernet ifname eth0 con-name lidar \
    ipv4.method manual ipv4.addresses 192.168.0.64/32 \
    ipv4.routes 192.168.0.2/32 ipv6.method disabled
```

Replace `eth0` with your interface (`nmcli device status`). Verify with `ping 192.168.0.2`.

## Launch

Add the driver to your bringup launch file, passing your config via the `config_path` parameter, plus a static transform from your robot to the lidar. The transform's child frame must match `ros_frame_id` in the above mentioned relevant config fields:

```python
from launch import LaunchDescription
from launch_ros.actions import Node
from launch_ros.substitutions import FindPackageShare
from launch.substitutions import PathJoinSubstitution

vanjee_wlr719c_cfg = PathJoinSubstitution([FindPackageShare('my_robot_bringup'), 'config', 'vanjee_wlr719c_cfg.yaml'])

def generate_launch_description():
    vanjee_lidar_node = Node(
        package='vanjee_lidar_sdk',
        executable='vanjee_lidar_sdk_node',
        name='vanjee_lidar_sdk_node',
        output='screen',
        emulate_tty=True,
        parameters=[{'config_path': vanjee_wlr719c_cfg}],
    )

    vanjee_lidar_tf2_node = Node(
        package='tf2_ros',
        executable='static_transform_publisher',
        name='static_tf_pub_laser',
        arguments=['0.123', '0.123', '0.123', '0', '0', '0', '1', 'base_link', 'vanjee_lidar'],
    )

    return LaunchDescription([
        vanjee_lidar_node,
        vanjee_lidar_tf2_node,
    ])
```

Then:

```bash
colcon build --packages-up-to my_robot_bringup
ros2 launch my_robot_bringup bringup.py
```

## License

BSD-3-Clause, © 2023 Vanjee — see `LICENSE` (copied verbatim from `vanjee_lidar_sdk/LICENSE`).

This project is not affiliated with, sponsored by, or endorsed by Vanjee; the name `ros2_vanjee_lidar_driver` is used solely to describe compatibility with Vanjee WLR-series lidars.
