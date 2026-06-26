---
title: 'ROS2 + Gazeboでロボットシミュレーション環境を構築する'
description: 'ROS2とGazebo Classicを使ったロボットシミュレーション環境の構築方法を解説。URDFでロボットモデルを作成し、物理シミュレーションで動かします。'
pubDate: '2026-06-03'
tags: ['ROS2', 'Gazebo', 'シミュレーション', 'URDF']
---

実機でテストする前にシミュレーターで動作確認できると開発がスムーズになります。ROS2 + Gazeboの環境構築を解説します。

## インストール

```bash
# Gazebo Classic (Gazebo11)
sudo apt install gazebo ros-humble-gazebo-ros-pkgs

# または Gazebo Ignition (新しい方)
sudo apt install ros-humble-ros-gz
```

## 簡単なURDFモデル

```xml
<?xml version="1.0"?>
<robot name="simple_robot">
  <!-- ベースリンク -->
  <link name="base_link">
    <visual>
      <geometry>
        <box size="0.3 0.2 0.1"/>
      </geometry>
      <material name="blue">
        <color rgba="0 0 1 1"/>
      </material>
    </visual>
    <collision>
      <geometry>
        <box size="0.3 0.2 0.1"/>
      </geometry>
    </collision>
    <inertial>
      <mass value="1.0"/>
      <inertia ixx="0.01" ixy="0" ixz="0" iyy="0.01" iyz="0" izz="0.01"/>
    </inertial>
  </link>

  <!-- 左車輪 -->
  <link name="left_wheel">
    <visual>
      <geometry>
        <cylinder radius="0.05" length="0.02"/>
      </geometry>
    </visual>
  </link>

  <joint name="left_wheel_joint" type="continuous">
    <parent link="base_link"/>
    <child link="left_wheel"/>
    <origin xyz="0.1 0.11 0" rpy="-1.5707 0 0"/>
    <axis xyz="0 0 1"/>
  </joint>
</robot>
```

## launch fileでGazebo起動

```python
from launch import LaunchDescription
from launch.actions import ExecuteProcess, IncludeLaunchDescription
from launch_ros.actions import Node
from ament_index_python.packages import get_package_share_directory
import os

def generate_launch_description():
    pkg = get_package_share_directory('my_robot')
    urdf = os.path.join(pkg, 'urdf', 'robot.urdf')

    return LaunchDescription([
        # Gazebo起動
        ExecuteProcess(
            cmd=['gazebo', '--verbose', '-s', 'libgazebo_ros_factory.so'],
            output='screen'
        ),
        # ロボットモデルをGazeboにスポーン
        Node(
            package='gazebo_ros',
            executable='spawn_entity.py',
            arguments=['-file', urdf, '-entity', 'my_robot'],
            output='screen'
        ),
        # robot_state_publisher
        Node(
            package='robot_state_publisher',
            executable='robot_state_publisher',
            parameters=[{'robot_description': open(urdf).read()}]
        ),
    ])
```

## よくある詰まりポイント

**Gazeboが起動しない**
```bash
# GPUドライバーの問題の場合
export LIBGL_ALWAYS_SOFTWARE=1
gazebo
```

**ロボットが地面に埋まる**
→ URDFのz座標を確認。`spawn_entity.py` の `-z` オプションで高さを指定する

**物理演算がおかしい**
→ `<inertial>` タグの値を見直す。ゼロや極端に小さい値は不安定になる
