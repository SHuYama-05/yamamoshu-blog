---
title: 'ROS2 Nav2入門：自律移動ロボットのナビゲーション実装【実機・シミュレーター対応】'
description: 'ROS2のナビゲーションスタックNav2の基本的な使い方を徹底解説。地図作成・自己位置推定・ゴール設定まで、実装コード付きで説明します。'
pubDate: '2026-06-18'
tags: ['ROS2', 'Nav2', 'ナビゲーション', 'SLAM']
---

Nav2はROS2の標準ナビゲーションスタックです。「地図を作る」「自分の位置を把握する」「目的地までの経路を計画する」「障害物を避ける」を全部まとめて扱えます。

初めて触ると設定ファイルや起動コマンドが多くて圧倒されます。この記事では最小限の構成から順に説明します。

## Nav2の全体像

```
LiDAR / カメラ（センサー）
        ↓
  SLAM Toolbox
  ├── 地図作成（Mapping）
  └── 自己位置推定（Localization）
        ↓
  Nav2
  ├── Global Planner（全体経路計画）
  │     └── start → goalまでの大まかなルート
  ├── Local Planner（局所障害物回避）
  │     └── 動的障害物・誤差を考慮してリアルタイム修正
  └── Controller（速度指令）
        ↓
  ロボット（/cmd_vel）
```

---

## インストール

```bash
# Nav2 本体
sudo apt install ros-humble-nav2-bringup \
                 ros-humble-nav2-core \
                 ros-humble-nav2-common

# SLAM
sudo apt install ros-humble-slam-toolbox

# TurtleBot3（シミュレーター用）
sudo apt install ros-humble-turtlebot3 \
                 ros-humble-turtlebot3-simulations
```

---

## ステップ1：シミュレーターで試す

まずTurtleBot3のシミュレーターで動かすのが一番手っ取り早いです。

```bash
# 環境変数を設定（.bashrcに書いておくと毎回打たなくていい）
export TURTLEBOT3_MODEL=burger
export GAZEBO_MODEL_PATH=$GAZEBO_MODEL_PATH:/opt/ros/humble/share/turtlebot3_gazebo/models

# Gazebo起動
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py

# 別ターミナル：SLAMで地図作成しながらNav2起動
ros2 launch nav2_bringup tb3_simulation_launch.py slam:=True
```

RViz2が起動したら「2D Goal Pose」ボタンで目的地を指定するとロボットが動き始めます。

---

## ステップ2：地図を保存する

SLAMで作った地図を保存しておけば、次回からローカライゼーションだけで使えます。

```bash
# 地図を保存（~/maps/my_map.yaml と ~/maps/my_map.pgm が生成される）
ros2 run nav2_map_server map_saver_cli -f ~/maps/my_map

# 保存した地図でナビゲーション起動
ros2 launch nav2_bringup tb3_simulation_launch.py \
  slam:=False \
  map:=$HOME/maps/my_map.yaml
```

---

## ステップ3：Pythonからゴールを送る

GUIで指定するだけでなく、Pythonコードからゴールを送ることで自律的なミッションを組めます。

```python
import rclpy
from rclpy.node import Node
from nav2_simple_commander.robot_navigator import BasicNavigator, TaskResult
from geometry_msgs.msg import PoseStamped
import math

def make_pose(navigator, x: float, y: float, yaw: float = 0.0) -> PoseStamped:
    """PoseStampedを生成するヘルパー"""
    pose = PoseStamped()
    pose.header.frame_id = 'map'
    pose.header.stamp = navigator.get_clock().now().to_msg()
    pose.pose.position.x = x
    pose.pose.position.y = y
    # yaw（Z軸回転）をクォータニオンに変換
    pose.pose.orientation.z = math.sin(yaw / 2.0)
    pose.pose.orientation.w = math.cos(yaw / 2.0)
    return pose


def main():
    rclpy.init()
    navigator = BasicNavigator()

    # 初期位置を設定（地図上のどこにいるかを教える）
    initial_pose = make_pose(navigator, x=0.0, y=0.0, yaw=0.0)
    navigator.setInitialPose(initial_pose)

    # Nav2が完全に起動するまで待機
    navigator.waitUntilNav2Active()
    print('Nav2起動完了')

    # ゴールを設定
    goal = make_pose(navigator, x=2.0, y=1.5, yaw=math.pi / 2)
    navigator.goToPose(goal)

    # 完了まで待ちながらフィードバックを表示
    while not navigator.isTaskComplete():
        feedback = navigator.getFeedback()
        if feedback:
            dist = feedback.distance_remaining
            print(f'残り距離: {dist:.2f}m')

    # 結果確認
    result = navigator.getResult()
    if result == TaskResult.SUCCEEDED:
        print('ゴール到達！')
    elif result == TaskResult.CANCELED:
        print('キャンセルされました')
    elif result == TaskResult.FAILED:
        print('ナビゲーション失敗')

    rclpy.shutdown()
```

### 複数のゴールを順番に回る（巡回パターン）

```python
waypoints = [
    make_pose(navigator, 2.0, 0.0),
    make_pose(navigator, 2.0, 2.0),
    make_pose(navigator, 0.0, 2.0),
    make_pose(navigator, 0.0, 0.0),  # スタートに戻る
]

navigator.followWaypoints(waypoints)

while not navigator.isTaskComplete():
    feedback = navigator.getFeedback()
    if feedback:
        idx = feedback.current_waypoint
        print(f'ウェイポイント {idx + 1}/{len(waypoints)} に向かっています')
```

---

## nav2_params.yaml の重要パラメーター

設定ファイルは `/opt/ros/humble/share/nav2_bringup/params/nav2_params.yaml` をベースにカスタマイズします。

```yaml
# ロボットのフットプリント（形状）
local_costmap:
  local_costmap:
    ros__parameters:
      robot_radius: 0.22  # 円形ロボットの半径(m)
      # または多角形の場合
      # footprint: "[[0.15, 0.10], [0.15, -0.10], [-0.15, -0.10], [-0.15, 0.10]]"

# 障害物のインフレーション（どれくらい余裕を持って避けるか）
inflation_layer:
  inflation_radius: 0.55  # この半径内に障害物がないようにする
  cost_scaling_factor: 3.5

# 最大速度
controller_server:
  ros__parameters:
    FollowPath:
      max_vel_x: 0.26       # 前進最大速度 (m/s)
      max_vel_theta: 1.0    # 回転最大速度 (rad/s)
      min_vel_x: -0.26      # 後退最大速度 (m/s)
```

---

## よくある詰まりポイント

### ロボットが動かない

```bash
# Nav2の状態確認
ros2 action list
# /navigate_to_pose が表示されていればNav2は動いている

# コストマップの確認（RViz2でLocal/Global Costmapを表示）
ros2 topic echo /local_costmap/costmap
```

### 地図上の位置がズレる（ローカライゼーション失敗）

AMCL（自己位置推定）がうまくいっていない場合です。

1. RViz2で「2D Pose Estimate」で初期位置を手動で設定
2. その後ゆっくり移動させると収束することが多い

### 「Could not find plugin」エラー

```bash
# プラグインが入っているか確認
ros2 pkg list | grep nav2
```

足りないパッケージを `apt install` で追加してください。

---

## まとめ

Nav2は最初の設定が大変ですが、一度動けばかなり安定しています。

1. まずTurtleBot3シミュレーターで動かしてみる
2. `nav2_simple_commander` でPythonから制御する
3. `nav2_params.yaml` でロボットに合わせてチューニング

次回はSLAM Toolboxで地図を作る手順を詳しく解説します。
