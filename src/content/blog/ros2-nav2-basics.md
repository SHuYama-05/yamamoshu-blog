---
title: 'ROS2 Nav2入門：自律移動ロボットのナビゲーション実装'
description: 'ROS2のナビゲーションスタックNav2の基本的な使い方を解説。地図作成からゴール設定まで実装例付きで紹介します。'
pubDate: '2026-06-18'
tags: ['ROS2', 'Nav2', 'ナビゲーション']
---

Nav2はROS2の標準ナビゲーションスタックです。障害物回避・経路計画・自己位置推定をまとめて扱えます。

## Nav2の全体像

```
センサー（LiDAR/カメラ）
    ↓
SLAM（地図作成 + 自己位置推定）
    ↓
経路計画（Global Planner）
    ↓
障害物回避（Local Planner）
    ↓
ロボット制御
```

## インストール

```bash
sudo apt install ros-humble-nav2-bringup \
                 ros-humble-nav2-core \
                 ros-humble-slam-toolbox
```

## 基本的な起動

```bash
# シミュレーター起動（TurtleBot3の例）
export TURTLEBOT3_MODEL=burger
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py

# Nav2起動
ros2 launch nav2_bringup navigation_launch.py \
  use_sim_time:=True \
  map:=/path/to/map.yaml
```

## Pythonからゴールを送る

```python
from nav2_simple_commander.robot_navigator import BasicNavigator
from geometry_msgs.msg import PoseStamped

navigator = BasicNavigator()

goal = PoseStamped()
goal.header.frame_id = 'map'
goal.pose.position.x = 2.0
goal.pose.position.y = 1.0
goal.pose.orientation.w = 1.0

navigator.goToPose(goal)

while not navigator.isTaskComplete():
    feedback = navigator.getFeedback()
    print(f'残り距離: {feedback.distance_remaining:.2f}m')
```

## 詰まりやすいポイント

**座標系の混乱** が一番多いです。`map` フレームと `odom` フレームの違いを理解しておくことが重要です。

- `map`：絶対座標（地図基準）
- `odom`：相対座標（起動時点基準、ドリフトする）
- `base_link`：ロボット本体の座標

次回はSLAMToolboxを使った地図作成を解説します。
