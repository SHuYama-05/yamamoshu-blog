---
title: "ROS2 + Gazeboでロボットシミュレーション環境を構築する【URDF・LiDAR・launch】"
emoji: "🌐"
type: "tech"
topics: ["ROS2", "Gazebo", "シミュレーション", "URDF", "ロボット"]
published: true
---

実機でテストする前にシミュレーターで動作確認すると開発が速くなります。ロボットを壊すリスクもゼロ。ROS2 + Gazebo Classicで差動二輪ロボットを動かすまでの手順です。

---

## インストール

```bash
sudo apt install gazebo ros-humble-gazebo-ros-pkgs
sudo apt install ros-humble-robot-state-publisher
```

---

## URDF（差動二輪ロボット）

```xml
<?xml version="1.0"?>
<robot name="diff_robot">

  <link name="base_footprint"/>

  <joint name="base_joint" type="fixed">
    <parent link="base_footprint"/>
    <child link="base_link"/>
    <origin xyz="0 0 0.1"/>
  </joint>

  <link name="base_link">
    <visual>
      <geometry><box size="0.3 0.2 0.1"/></geometry>
    </visual>
    <collision>
      <geometry><box size="0.3 0.2 0.1"/></geometry>
    </collision>
    <inertial>
      <mass value="2.0"/>
      <!-- 慣性テンソルをゼロにすると物理演算が発散する -->
      <inertia ixx="0.0058" ixy="0" ixz="0"
               iyy="0.0108" iyz="0" izz="0.0150"/>
    </inertial>
  </link>

  <link name="left_wheel">
    <visual>
      <geometry><cylinder radius="0.05" length="0.04"/></geometry>
    </visual>
    <inertial>
      <mass value="0.3"/>
      <inertia ixx="0.000188" ixy="0" ixz="0"
               iyy="0.000188" iyz="0" izz="0.000375"/>
    </inertial>
  </link>

  <joint name="left_wheel_joint" type="continuous">
    <parent link="base_link"/>
    <child link="left_wheel"/>
    <origin xyz="0 0.12 -0.05" rpy="-1.5708 0 0"/>
    <axis xyz="0 0 1"/>
  </joint>

  <!-- 右車輪も同様 -->

  <!-- 差動駆動プラグイン -->
  <gazebo>
    <plugin name="diff_drive" filename="libgazebo_ros_diff_drive.so">
      <left_joint>left_wheel_joint</left_joint>
      <right_joint>right_wheel_joint</right_joint>
      <wheel_separation>0.24</wheel_separation>
      <wheel_radius>0.05</wheel_radius>
      <ros>
        <remapping>cmd_vel:=/cmd_vel</remapping>
        <remapping>odom:=/odom</remapping>
      </ros>
      <publish_odom>true</publish_odom>
      <publish_odom_tf>true</publish_odom_tf>
    </plugin>
  </gazebo>

  <!-- LiDARプラグイン -->
  <link name="laser_frame"/>
  <joint name="laser_joint" type="fixed">
    <parent link="base_link"/>
    <child link="laser_frame"/>
    <origin xyz="0.1 0 0.07"/>
  </joint>

  <gazebo reference="laser_frame">
    <sensor name="lidar" type="ray">
      <ray>
        <scan>
          <horizontal>
            <samples>360</samples>
            <min_angle>-3.14159</min_angle>
            <max_angle>3.14159</max_angle>
          </horizontal>
        </scan>
        <range><min>0.12</min><max>10.0</max></range>
      </ray>
      <plugin name="gazebo_ros_laser" filename="libgazebo_ros_ray_sensor.so">
        <ros><remapping>~/out:=scan</remapping></ros>
        <output_type>sensor_msgs/LaserScan</output_type>
        <frame_name>laser_frame</frame_name>
      </plugin>
      <update_rate>10</update_rate>
    </sensor>
  </gazebo>

</robot>
```

---

## launchファイル

```python
from launch import LaunchDescription
from launch.actions import ExecuteProcess, TimerAction
from launch_ros.actions import Node
import os

def generate_launch_description():
    urdf_path = '/path/to/robot.urdf'
    with open(urdf_path) as f:
        robot_desc = f.read()

    spawn_robot = Node(
        package='gazebo_ros',
        executable='spawn_entity.py',
        arguments=['-entity', 'diff_robot', '-topic', 'robot_description',
                   '-x', '0.0', '-y', '0.0', '-z', '0.1']
    )

    return LaunchDescription([
        ExecuteProcess(
            cmd=['gazebo', '--verbose', '-s', 'libgazebo_ros_init.so',
                 '-s', 'libgazebo_ros_factory.so'],
        ),
        Node(
            package='robot_state_publisher',
            executable='robot_state_publisher',
            parameters=[{'robot_description': robot_desc, 'use_sim_time': True}]
        ),
        TimerAction(period=3.0, actions=[spawn_robot]),  # Gazebo起動待ち
    ])
```

---

## よくある詰まりポイント

**ロボットが地面に埋まる** → `<inertial>` の値が0やゼロに近い。現実的な値を設定する

**黒い画面のまま** → `export LIBGL_ALWAYS_SOFTWARE=1` を試す

**cmd_velを送っても動かない** → `ros2 topic list | grep cmd_vel` でトピック名を確認
