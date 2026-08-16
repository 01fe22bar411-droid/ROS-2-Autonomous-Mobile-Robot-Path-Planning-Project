# ROS-2-Autonomous-Mobile-Robot-Path-Planning-Project
A complete, from-scratch ROS 2 (Humble) project: a simulated differential-drive robot in Gazebo, mapped with SLAM, localized with AMCL, and navigated autonomously using the Nav2 stack — all visualized live in RViz.  Stack: ROS 2 Humble · Gazebo Classic (gazebo11) · slam_toolbox · Nav2 · RViz2

***Phase 1 — Environment Setup

Terminal 1

bash
# Check ROS 2 distro
printenv ROS_DISTRO

# Create the workspace
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws
colcon build
source install/setup.bash

# If colcon isn't found:
sudo apt install python3-colcon-common-extensions   ***


# ROS 2 Autonomous Mobile Robot — Path Planning Project

A complete, from-scratch ROS 2 (Humble) project: a simulated differential-drive robot in Gazebo, mapped with SLAM, localized with AMCL, and navigated autonomously using the Nav2 stack — all visualized live in RViz.

**Stack:** ROS 2 Humble · Gazebo Classic (gazebo11) · slam_toolbox · Nav2 · RViz2

---

## System Overview

You will typically run **4 terminals at once**, each with a long-running process:

| Terminal | Purpose | Command |
|---|---|---|
| 1 | Gazebo simulation + robot spawn | `ros2 launch my_robot_description spawn_robot.launch.py` |
| 2 | SLAM / mapping | `ros2 launch slam_toolbox online_async_launch.py` |
| 3 | RViz visualization | `rviz2` |
| 4 | Keyboard teleop (driving) | `ros2 run teleop_twist_keyboard teleop_twist_keyboard` |

Every terminal must first run:
```bash
source ~/ros2_ws/install/setup.bash
```

---

## Phase 1 — Environment Setup

**Terminal 1**

```bash
# Check ROS 2 distro
printenv ROS_DISTRO

# Create the workspace
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws
colcon build
source install/setup.bash

# If colcon isn't found:
sudo apt install python3-colcon-common-extensions
```

**Sanity check — talker/listener demo**

Terminal 1:
```bash
ros2 run demo_nodes_cpp talker
```

Terminal 2:
```bash
source /opt/ros/humble/setup.bash
ros2 run demo_nodes_cpp listener
```
You should see the listener printing messages published by the talker. Stop both with `Ctrl+C`.

---

## Phase 2 — Robot Model & Gazebo Simulation

### Install Gazebo (Humble → Gazebo Classic)

```bash
sudo apt update
sudo apt install ros-humble-gazebo-ros-pkgs ros-humble-gazebo-ros2-control
gazebo --version   # verify install
```

**Fix plugin path issues** (needed if `/cmd_vel` and `/odom` don't appear later):
```bash
echo 'export GAZEBO_PLUGIN_PATH=/opt/ros/humble/lib:$GAZEBO_PLUGIN_PATH' >> ~/.bashrc
echo 'export GAZEBO_MODEL_PATH=/opt/ros/humble/share:$GAZEBO_MODEL_PATH' >> ~/.bashrc
source ~/.bashrc
```

### Create the robot description package

```bash
cd ~/ros2_ws/src
ros2 pkg create my_robot_description --build-type ament_cmake
cd my_robot_description
mkdir urdf launch worlds
```

### `urdf/my_robot.urdf`

A differential-drive robot: box chassis, 2 driven wheels, 1 caster, plus a Gazebo `diff_drive` plugin.

```xml
<?xml version="1.0"?>
<robot name="my_robot">

  <link name="base_link">
    <visual>
      <geometry><box size="0.4 0.3 0.15"/></geometry>
      <origin xyz="0 0 0.075"/>
      <material name="blue"><color rgba="0.2 0.2 0.8 1"/></material>
    </visual>
    <collision>
      <geometry><box size="0.4 0.3 0.15"/></geometry>
      <origin xyz="0 0 0.075"/>
    </collision>
    <inertial>
      <mass value="5.0"/>
      <origin xyz="0 0 0.075"/>
      <inertia ixx="0.05" ixy="0" ixz="0" iyy="0.06" iyz="0" izz="0.08"/>
    </inertial>
  </link>

  <link name="left_wheel">
    <visual>
      <geometry><cylinder radius="0.05" length="0.04"/></geometry>
      <material name="black"><color rgba="0.1 0.1 0.1 1"/></material>
    </visual>
    <collision>
      <geometry><cylinder radius="0.05" length="0.04"/></geometry>
    </collision>
    <inertial>
      <mass value="0.5"/>
      <inertia ixx="0.001" ixy="0" ixz="0" iyy="0.001" iyz="0" izz="0.001"/>
    </inertial>
  </link>

  <link name="right_wheel">
    <visual>
      <geometry><cylinder radius="0.05" length="0.04"/></geometry>
      <material name="black"/>
    </visual>
    <collision>
      <geometry><cylinder radius="0.05" length="0.04"/></geometry>
    </collision>
    <inertial>
      <mass value="0.5"/>
      <inertia ixx="0.001" ixy="0" ixz="0" iyy="0.001" iyz="0" izz="0.001"/>
    </inertial>
  </link>

  <link name="caster_wheel">
    <visual>
      <geometry><sphere radius="0.04"/></geometry>
      <material name="black"/>
    </visual>
    <collision>
      <geometry><sphere radius="0.04"/></geometry>
    </collision>
    <inertial>
      <mass value="0.2"/>
      <inertia ixx="0.0001" ixy="0" ixz="0" iyy="0.0001" iyz="0" izz="0.0001"/>
    </inertial>
  </link>

  <joint name="left_wheel_joint" type="continuous">
    <parent link="base_link"/>
    <child link="left_wheel"/>
    <origin xyz="0 0.175 0.05" rpy="-1.5708 0 0"/>
    <axis xyz="0 0 1"/>
  </joint>

  <joint name="right_wheel_joint" type="continuous">
    <parent link="base_link"/>
    <child link="right_wheel"/>
    <origin xyz="0 -0.175 0.05" rpy="-1.5708 0 0"/>
    <axis xyz="0 0 1"/>
  </joint>

  <joint name="caster_wheel_joint" type="fixed">
    <parent link="base_link"/>
    <child link="caster_wheel"/>
    <origin xyz="0.15 0 0.025"/>
  </joint>

  <!-- Differential drive plugin -->
  <gazebo>
    <plugin name="diff_drive" filename="libgazebo_ros_diff_drive.so">
      <left_joint>left_wheel_joint</left_joint>
      <right_joint>right_wheel_joint</right_joint>
      <wheel_separation>0.35</wheel_separation>
      <wheel_diameter>0.1</wheel_diameter>
      <max_wheel_torque>20</max_wheel_torque>
      <max_wheel_acceleration>1.0</max_wheel_acceleration>
      <odom_publish_frequency>30</odom_publish_frequency>
      <publish_odom>true</publish_odom>
      <publish_odom_tf>true</publish_odom_tf>
      <publish_wheel_tf>true</publish_wheel_tf>
      <odometry_frame>odom</odometry_frame>
      <robot_base_frame>base_link</robot_base_frame>
    </plugin>
  </gazebo>

  <gazebo reference="left_wheel">
    <mu1>1.0</mu1><mu2>1.0</mu2>
  </gazebo>
  <gazebo reference="right_wheel">
    <mu1>1.0</mu1><mu2>1.0</mu2>
  </gazebo>

  <!-- LIDAR -->
  <link name="lidar_link">
    <visual>
      <geometry><cylinder radius="0.04" length="0.05"/></geometry>
      <material name="black"/>
    </visual>
    <collision>
      <geometry><cylinder radius="0.04" length="0.05"/></geometry>
    </collision>
    <inertial>
      <mass value="0.1"/>
      <inertia ixx="0.0001" ixy="0" ixz="0" iyy="0.0001" iyz="0" izz="0.0001"/>
    </inertial>
  </link>

  <joint name="lidar_joint" type="fixed">
    <parent link="base_link"/>
    <child link="lidar_link"/>
    <origin xyz="0 0 0.15" rpy="0 0 0"/>
  </joint>

  <gazebo reference="lidar_link">
    <sensor type="ray" name="lidar_sensor">
      <pose>0 0 0 0 0 0</pose>
      <visualize>true</visualize>
      <update_rate>10</update_rate>
      <ray>
        <scan>
          <horizontal>
            <samples>360</samples>
            <resolution>1</resolution>
            <min_angle>-3.14159</min_angle>
            <max_angle>3.14159</max_angle>
          </horizontal>
        </scan>
        <range>
          <min>0.12</min>
          <max>10.0</max>
          <resolution>0.015</resolution>
        </range>
      </ray>
      <plugin name="lidar_plugin" filename="libgazebo_ros_ray_sensor.so">
        <ros>
          <namespace>/</namespace>
          <remapping>~/out:=scan</remapping>
        </ros>
        <output_type>sensor_msgs/LaserScan</output_type>
        <frame_name>lidar_link</frame_name>
      </plugin>
    </sensor>
  </gazebo>

</robot>
```

### `worlds/simple_room.world`

A rectangular room (walls) with two box obstacles, used for mapping.

```xml
<?xml version="1.0"?>
<sdf version="1.6">
  <world name="default">
    <include><uri>model://sun</uri></include>
    <include><uri>model://ground_plane</uri></include>

    <model name="wall_north">
      <static>true</static>
      <pose>0 5 0.5 0 0 0</pose>
      <link name="link">
        <collision name="collision"><geometry><box><size>10 0.2 1</size></box></geometry></collision>
        <visual name="visual"><geometry><box><size>10 0.2 1</size></box></geometry></visual>
      </link>
    </model>

    <model name="wall_south">
      <static>true</static>
      <pose>0 -5 0.5 0 0 0</pose>
      <link name="link">
        <collision name="collision"><geometry><box><size>10 0.2 1</size></box></geometry></collision>
        <visual name="visual"><geometry><box><size>10 0.2 1</size></box></geometry></visual>
      </link>
    </model>

    <model name="wall_east">
      <static>true</static>
      <pose>5 0 0.5 0 0 1.5708</pose>
      <link name="link">
        <collision name="collision"><geometry><box><size>10 0.2 1</size></box></geometry></collision>
        <visual name="visual"><geometry><box><size>10 0.2 1</size></box></geometry></visual>
      </link>
    </model>

    <model name="wall_west">
      <static>true</static>
      <pose>-5 0 0.5 0 0 1.5708</pose>
      <link name="link">
        <collision name="collision"><geometry><box><size>10 0.2 1</size></box></geometry></collision>
        <visual name="visual"><geometry><box><size>10 0.2 1</size></box></geometry></visual>
      </link>
    </model>

    <model name="obstacle_1">
      <static>true</static>
      <pose>2 1 0.5 0 0 0</pose>
      <link name="link">
        <collision name="collision"><geometry><box><size>0.5 0.5 1</size></box></geometry></collision>
        <visual name="visual"><geometry><box><size>0.5 0.5 1</size></box></geometry></visual>
      </link>
    </model>

    <model name="obstacle_2">
      <static>true</static>
      <pose>-2 -1.5 0.5 0 0 0</pose>
      <link name="link">
        <collision name="collision"><geometry><box><size>0.5 0.5 1</size></box></geometry></collision>
        <visual name="visual"><geometry><box><size>0.5 0.5 1</size></box></geometry></visual>
      </link>
    </model>

  </world>
</sdf>
```

### `launch/spawn_robot.launch.py`

```python
import os
from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import ExecuteProcess
from launch_ros.actions import Node

def generate_launch_description():
    pkg_path = get_package_share_directory('my_robot_description')
    urdf_path = os.path.join(pkg_path, 'urdf', 'my_robot.urdf')
    world_path = os.path.join(pkg_path, 'worlds', 'simple_room.world')

    with open(urdf_path, 'r') as f:
        robot_desc = f.read()

    return LaunchDescription([
        ExecuteProcess(
            cmd=[
                'gazebo', '--verbose', world_path,
                '-s', 'libgazebo_ros_init.so',
                '-s', 'libgazebo_ros_factory.so'
            ],
            output='screen'
        ),
        Node(
            package='robot_state_publisher',
            executable='robot_state_publisher',
            output='screen',
            parameters=[{'robot_description': robot_desc}]
        ),
        Node(
            package='gazebo_ros',
            executable='spawn_entity.py',
            arguments=['-topic', 'robot_description', '-entity', 'my_robot'],
            output='screen'
        ),
    ])
```

### `CMakeLists.txt` — add before `ament_package()`

```cmake
install(DIRECTORY urdf launch worlds DESTINATION share/${PROJECT_NAME})
```

### Build & launch

```bash
cd ~/ros2_ws
colcon build --packages-select my_robot_description
source install/setup.bash
ros2 launch my_robot_description spawn_robot.launch.py
```

Gazebo should open showing the walled room with the robot spawned inside.

### Verify + drive manually

**Terminal 2**
```bash
source ~/ros2_ws/install/setup.bash
ros2 topic list          # confirm /cmd_vel and /odom exist

sudo apt install ros-humble-teleop-twist-keyboard
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```
Keys: `i` forward · `,` backward · `j` left · `l` right · `k` stop.

> **Common issue:** if `/cmd_vel` / `/odom` don't appear, it's almost always the `GAZEBO_PLUGIN_PATH` env var missing (see Phase 2 install step above).

---

## Phase 3 — LIDAR + RViz Visualization

The LIDAR was already added to the URDF above (`lidar_link` + `libgazebo_ros_ray_sensor.so` plugin, publishing on `/scan`).

**Terminal 1** — rebuild & relaunch:
```bash
cd ~/ros2_ws
colcon build --packages-select my_robot_description
source install/setup.bash
pkill -9 gzserver; pkill -9 gzclient; pkill -9 gazebo
ros2 launch my_robot_description spawn_robot.launch.py
```

**Terminal 2** — verify the scan:
```bash
source ~/ros2_ws/install/setup.bash
ros2 topic list | grep scan
ros2 topic hz /scan       # should show ~10 Hz
```

**Terminal 3** — RViz:
```bash
rviz2
```
In RViz:
- **Global Options → Fixed Frame** → `map` (or `odom` if map isn't published yet)
- **Add → By topic → `/map` → Map**
- **Add → By topic → `/scan` → LaserScan**
- **Add → By display type → RobotModel** → set **Description Topic** = `/robot_description`

---

## Phase 4 — SLAM Mapping

### Install slam_toolbox
```bash
sudo apt install ros-humble-slam-toolbox
```

### Fix the base frame mismatch (important!)

By default, slam_toolbox's config expects `base_frame: base_footprint`. This robot only has `base_link` — the mismatch causes repeated `Failed to compute odom pose` warnings.

```bash
sudo nano /opt/ros/humble/share/slam_toolbox/config/mapper_params_online_async.yaml
```
Change:
```yaml
base_frame: base_footprint
```
to:
```yaml
base_frame: base_link
```
Save (`Ctrl+O`, `Enter`) and exit (`Ctrl+X`).

### Launch SLAM

**Terminal 2** (Gazebo already running in Terminal 1):
```bash
source ~/ros2_ws/install/setup.bash
ros2 launch slam_toolbox online_async_launch.py
```

### Drive to build the map

**Terminal 4**:
```bash
source ~/ros2_ws/install/setup.bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

> **Tip:** Drive in short taps, not smooth continuous motion — tap `i` for under half a second, tap `k` to stop, pause ~1 second, repeat. Fast/long movements cause scan-matching drift, producing a scattered/noisy map instead of clean wall lines. Loop slowly near all 4 walls and both obstacles.

Watch RViz's Map display — walls should appear as clean, straight gray lines.

### Save the map

```bash
mkdir -p ~/ros2_ws/maps
ros2 run nav2_map_server map_saver_cli -f ~/ros2_ws/maps/my_map
```
Produces `my_map.yaml` + `my_map.pgm`.

---

## Phase 5 — Localization (AMCL)

*(Next phase — load the saved map, run AMCL, confirm the estimated pose tracks the robot in RViz.)*

```bash
sudo apt install ros-humble-nav2-amcl ros-humble-nav2-map-server ros-humble-nav2-lifecycle-manager

ros2 run nav2_map_server map_server --ros-args -p yaml_filename:=/home/$USER/ros2_ws/maps/my_map.yaml
ros2 run nav2_amcl amcl
ros2 run nav2_lifecycle_manager lifecycle_manager --ros-args -p autostart:=true -p node_names:=['map_server','amcl']
```
In RViz, set an initial pose estimate with **2D Pose Estimate**, then confirm the estimated pose (particle cloud) tracks the robot as you drive.

---

## Phase 6 — Nav2 Path Planning

*(Load the full Nav2 stack, send goals from RViz's "2D Goal Pose" tool, watch global/local paths plan and execute.)*

```bash
sudo apt install ros-humble-navigation2 ros-humble-nav2-bringup

ros2 launch nav2_bringup navigation_launch.py map:=/home/$USER/ros2_ws/maps/my_map.yaml
```
In RViz, click **2D Goal Pose**, click a point in the mapped room — the robot should plan a path and drive there autonomously.

---

## Phase 7 — Tuning & Extending

- Tune costmap inflation radius, planner/controller parameters (DWB or MPPI)
- Test in a more complex world with more obstacles
- Add multi-goal waypoint following
- Add dynamic obstacle avoidance

---

## Troubleshooting Log (issues hit during this build)

| Symptom | Cause | Fix |
|---|---|---|
| `/cmd_vel`, `/odom` missing from `ros2 topic list` | `GAZEBO_PLUGIN_PATH` not set | Export `GAZEBO_PLUGIN_PATH`/`GAZEBO_MODEL_PATH`, add to `~/.bashrc` |
| `Exception sending a multicast message: Network is unreachable` | Gazebo's multicast discovery, harmless | Ignore, or `export GAZEBO_IP=127.0.0.1` |
| Launch file `SyntaxError: closing parenthesis...` | Manual nano edit broke bracket matching | Rewrite the whole `.py` file cleanly instead of patching |
| `slam_toolbox`: `Failed to compute odom pose` (repeating) | `base_frame: base_footprint` in slam_toolbox config, robot only has `base_link` | Edit `mapper_params_online_async.yaml`, set `base_frame: base_link` |
| Map looks scattered/noisy, walls not straight | Drove too fast/far between scans, scan-matching drift | Restart SLAM, drive in short taps with pauses, not continuous motion |
| RViz "Map" display shows red `Status: Error` | Topic field left unset | Remove display, re-add via **Add → By topic → /map → Map** |

---
