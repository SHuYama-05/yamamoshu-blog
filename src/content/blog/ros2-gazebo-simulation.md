---
title: 'ROS2 + Gazeboでロボットシミュレーション環境を構築する【URDF・launch・センサー】'
description: 'ROS2とGazebo Classicを使ったロボットシミュレーション環境の構築方法を解説。URDFでロボットモデルを作成し、LiDARカメラを追加して物理シミュレーションで動かす手順を紹介します。'
pubDate: '2026-06-03'
tags: ['ROS2', 'Gazebo', 'シミュレーション', 'URDF']
---

実機でテストする前にシミュレーターで動作確認できると、開発サイクルが大幅に速くなります。ロボットを壊すリスクもゼロです。

この記事ではROS2 + Gazebo Classicでシミュレーション環境を構築する手順を、実際に動くURDFと設定ファイルを使って説明します。

---

## インストール

```bash
# Gazebo Classic（Gazebo 11）
sudo apt install gazebo
sudo apt install ros-humble-gazebo-ros-pkgs \
                 ros-humble-gazebo-ros2-control

# RobotStatePublisher（URDFを読んでTFを配信）
sudo apt install ros-humble-robot-state-publisher \
                 ros-humble-joint-state-publisher
```

---

## URDFでロボットモデルを作る

### 最小構成のロボット（差動二輪）

```xml
<?xml version="1.0"?>
<robot name="diff_robot" xmlns:xacro="http://www.ros.org/wiki/xacro">

  <!-- マテリアル定義 -->
  <material name="blue">
    <color rgba="0.2 0.4 0.8 1.0"/>
  </material>
  <material name="gray">
    <color rgba="0.5 0.5 0.5 1.0"/>
  </material>

  <!-- ===== ベース ===== -->
  <link name="base_footprint"/>

  <joint name="base_joint" type="fixed">
    <parent link="base_footprint"/>
    <child link="base_link"/>
    <origin xyz="0 0 0.1"/>  <!-- 地面から10cm上に基準点 -->
  </joint>

  <link name="base_link">
    <visual>
      <geometry>
        <box size="0.3 0.2 0.1"/>
      </geometry>
      <material name="blue"/>
    </visual>
    <collision>
      <geometry>
        <box size="0.3 0.2 0.1"/>
      </geometry>
    </collision>
    <inertial>
      <mass value="2.0"/>
      <!-- 直方体の慣性テンソル: I = m/12 * (h²+d²) など -->
      <inertia ixx="0.0058" ixy="0" ixz="0"
               iyy="0.0108" iyz="0"
               izz="0.0150"/>
    </inertial>
  </link>

  <!-- ===== 左車輪 ===== -->
  <link name="left_wheel">
    <visual>
      <geometry>
        <cylinder radius="0.05" length="0.04"/>
      </geometry>
      <material name="gray"/>
    </visual>
    <collision>
      <geometry>
        <cylinder radius="0.05" length="0.04"/>
      </geometry>
    </collision>
    <inertial>
      <mass value="0.3"/>
      <inertia ixx="0.000188" ixy="0" ixz="0"
               iyy="0.000188" iyz="0"
               izz="0.000375"/>
    </inertial>
  </link>

  <joint name="left_wheel_joint" type="continuous">
    <parent link="base_link"/>
    <child link="left_wheel"/>
    <origin xyz="0 0.12 -0.05" rpy="-1.5708 0 0"/>
    <axis xyz="0 0 1"/>
  </joint>

  <!-- ===== 右車輪 ===== -->
  <link name="right_wheel">
    <visual>
      <geometry>
        <cylinder radius="0.05" length="0.04"/>
      </geometry>
      <material name="gray"/>
    </visual>
    <collision>
      <geometry>
        <cylinder radius="0.05" length="0.04"/>
      </geometry>
    </collision>
    <inertial>
      <mass value="0.3"/>
      <inertia ixx="0.000188" ixy="0" ixz="0"
               iyy="0.000188" iyz="0"
               izz="0.000375"/>
    </inertial>
  </link>

  <joint name="right_wheel_joint" type="continuous">
    <parent link="base_link"/>
    <child link="right_wheel"/>
    <origin xyz="0 -0.12 -0.05" rpy="-1.5708 0 0"/>
    <axis xyz="0 0 1"/>
  </joint>

  <!-- ===== LiDAR ===== -->
  <link name="laser_frame">
    <visual>
      <geometry>
        <cylinder radius="0.03" length="0.04"/>
      </geometry>
    </visual>
  </link>

  <joint name="laser_joint" type="fixed">
    <parent link="base_link"/>
    <child link="laser_frame"/>
    <origin xyz="0.1 0 0.07"/>
  </joint>

  <!-- Gazeboプラグイン：差動駆動 -->
  <gazebo>
    <plugin name="diff_drive" filename="libgazebo_ros_diff_drive.so">
      <left_joint>left_wheel_joint</left_joint>
      <right_joint>right_wheel_joint</right_joint>
      <wheel_separation>0.24</wheel_separation>
      <wheel_radius>0.05</wheel_radius>
      <max_wheel_torque>20</max_wheel_torque>
      <ros>
        <remapping>cmd_vel:=/cmd_vel</remapping>
        <remapping>odom:=/odom</remapping>
      </ros>
      <publish_odom>true</publish_odom>
      <publish_odom_tf>true</publish_odom_tf>
    </plugin>
  </gazebo>

  <!-- Gazeboプラグイン：LiDAR -->
  <gazebo reference="laser_frame">
    <sensor name="lidar" type="ray">
      <pose>0 0 0 0 0 0</pose>
      <ray>
        <scan>
          <horizontal>
            <samples>360</samples>
            <min_angle>-3.14159</min_angle>
            <max_angle>3.14159</max_angle>
          </horizontal>
        </scan>
        <range>
          <min>0.12</min>
          <max>10.0</max>
        </range>
      </ray>
      <plugin name="gazebo_ros_laser" filename="libgazebo_ros_ray_sensor.so">
        <ros>
          <remapping>~/out:=scan</remapping>
        </ros>
        <output_type>sensor_msgs/LaserScan</output_type>
        <frame_name>laser_frame</frame_name>
      </plugin>
      <always_on>true</always_on>
      <update_rate>10</update_rate>
    </sensor>
  </gazebo>

</robot>
```

---

## launch ファイル

```python
import os
from launch import LaunchDescription
from launch.actions import ExecuteProcess, IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch_ros.actions import Node
from ament_index_python.packages import get_package_share_directory

def generate_launch_description():
    pkg_path = get_package_share_directory('my_robot')
    urdf_path = os.path.join(pkg_path, 'urdf', 'robot.urdf')
    world_path = os.path.join(pkg_path, 'worlds', 'simple_world.world')

    with open(urdf_path, 'r') as f:
        robot_description = f.read()

    return LaunchDescription([
        # Gazebo起動
        ExecuteProcess(
            cmd=['gazebo', '--verbose', world_path,
                 '-s', 'libgazebo_ros_init.so',
                 '-s', 'libgazebo_ros_factory.so'],
            output='screen'
        ),

        # robot_state_publisher（URDFからTFを配信）
        Node(
            package='robot_state_publisher',
            executable='robot_state_publisher',
            parameters=[{'robot_description': robot_description,
                         'use_sim_time': True}],
            output='screen'
        ),

        # ロボットをGazeboにスポーン
        Node(
            package='gazebo_ros',
            executable='spawn_entity.py',
            arguments=[
                '-entity', 'diff_robot',
                '-topic', 'robot_description',
                '-x', '0.0', '-y', '0.0', '-z', '0.1'
            ],
            output='screen'
        ),
    ])
```

---

## よくある詰まりポイント

### ロボットが地面に埋まる・飛んでいく

URDFの `<inertial>` タグの値が問題です。

```xml
<!-- NG: inertiaがゼロや極小値 → 物理演算が発散する -->
<inertia ixx="0" ixy="0" ixz="0" iyy="0" iyz="0" izz="0"/>

<!-- OK: 現実的な値を設定 -->
<!-- 直方体(lxm, lym, lzm, mass=m)の場合 -->
<!-- ixx = m/12 * (ly² + lz²) -->
<inertia ixx="0.0058" ixy="0" ixz="0"
         iyy="0.0108" iyz="0"
         izz="0.0150"/>
```

### Gazeboが起動しない（黒い画面のまま）

```bash
# GPUレンダリングの問題
export LIBGL_ALWAYS_SOFTWARE=1
gazebo

# または環境変数を確認
echo $DISPLAY  # 空の場合は :0 を設定
export DISPLAY=:0
```

### `[spawn_entity.py] process has died`

Gazeboが完全に起動する前にspawnしようとしているケースが多いです。

```python
# launch fileに待機を追加
from launch.actions import TimerAction

TimerAction(
    period=3.0,  # 3秒待ってからspawn
    actions=[spawn_entity_node]
)
```

### cmd_velを送っても動かない

差動駆動プラグインのトピック名を確認してください。

```bash
ros2 topic list | grep cmd_vel
ros2 topic info /cmd_vel

# テスト送信
ros2 topic pub /cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.1}, angular: {z: 0.0}}" --once
```

---

## まとめ

Gazeboのシミュレーションは設定ファイルが多くて最初は大変ですが、一度動けばほぼ実機と同じコードでテストできます。

1. URDFで形状・物理パラメーター・センサーを定義
2. Gazeboプラグインで実際の動作をシミュレート
3. launch fileでGazebo・robot_state_publisher・spawnをまとめて起動

次回はNav2とこのシミュレーターを組み合わせた自律ナビゲーションを解説します。
