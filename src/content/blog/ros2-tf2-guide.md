---
title: 'ROS2のTF2（座標変換）を完全理解する'
description: 'ROS2開発で避けて通れないTF2（Transform）の概念と使い方を解説。座標系の変換・ブロードキャスト・リスニングを実装例で説明します。'
pubDate: '2026-06-10'
tags: ['ROS2', 'TF2', '座標変換']
---

ROS2でロボット開発をしていると必ず出てくるTF2。「座標が合わない」「フレームが見つからない」というエラーで詰まった経験がある人は多いはずです。

## TF2とは

TF2はROS2の座標変換ライブラリです。ロボットの各部位（センサー・車輪・アーム等）の位置関係を管理します。

```
world
  └── map
        └── odom
              └── base_link
                    ├── laser_frame  (LiDAR)
                    ├── camera_frame (カメラ)
                    └── imu_frame    (IMU)
```

## TFブロードキャスター（座標を発信）

```python
import rclpy
from rclpy.node import Node
from tf2_ros import TransformBroadcaster
from geometry_msgs.msg import TransformStamped
import math

class TFPublisher(Node):
    def __init__(self):
        super().__init__('tf_publisher')
        self.br = TransformBroadcaster(self)
        self.timer = self.create_timer(0.1, self.broadcast)

    def broadcast(self):
        t = TransformStamped()
        t.header.stamp = self.get_clock().now().to_msg()
        t.header.frame_id = 'base_link'
        t.child_frame_id = 'laser_frame'

        # LiDARはロボット中心から前方0.1m上方0.2mに設置
        t.transform.translation.x = 0.1
        t.transform.translation.y = 0.0
        t.transform.translation.z = 0.2
        t.transform.rotation.w = 1.0  # 回転なし

        self.br.sendTransform(t)
```

## TFリスナー（座標を受け取る）

```python
from tf2_ros import TransformListener, Buffer

class TFReader(Node):
    def __init__(self):
        super().__init__('tf_reader')
        self.tf_buffer = Buffer()
        self.tf_listener = TransformListener(self.tf_buffer, self)
        self.timer = self.create_timer(1.0, self.get_transform)

    def get_transform(self):
        try:
            t = self.tf_buffer.lookup_transform(
                'map',        # ターゲットフレーム
                'base_link',  # ソースフレーム
                rclpy.time.Time()  # 最新の変換を取得
            )
            x = t.transform.translation.x
            y = t.transform.translation.y
            self.get_logger().info(f'ロボット位置: ({x:.2f}, {y:.2f})')
        except Exception as e:
            self.get_logger().warn(f'TF取得失敗: {e}')
```

## よくあるエラーと対処法

**`LookupException: "base_link" passed to lookupTransform`**
→ TFが発信されていない。`ros2 run tf2_tools view_frames` でTFツリーを確認

**`ExtrapolationException: Lookup would require extrapolation`**
→ タイムスタンプのズレ。`rclpy.time.Time()` で最新を取得するか、時刻同期を確認

## デバッグコマンド

```bash
# TFツリーの可視化
ros2 run tf2_tools view_frames

# 特定のフレーム間の変換を確認
ros2 run tf2_ros tf2_echo map base_link
```
