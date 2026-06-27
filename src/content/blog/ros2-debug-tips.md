---
title: 'ROS2デバッグ完全ガイド【ros2 bag・rqt・rviz2・よくあるエラー集】'
description: 'ROS2開発でのデバッグ手法を網羅。ros2 bagでのデータ記録・再生、rqtでのGUIデバッグ、rviz2での可視化、よくあるエラーの原因と解決策を実例付きで解説します。'
pubDate: '2026-06-20'
tags: ['ROS2', 'デバッグ', 'ros2bag', 'rqt', 'rviz2']
---

ROS2開発でハマったとき、何から調べればいいか迷うことがあります。この記事では実際によく使うデバッグ手法をまとめました。

---

## コマンドラインで状態確認

まず動いているかどうかの確認から。

```bash
# 起動中のノード一覧
ros2 node list

# トピック一覧
ros2 topic list

# トピックの型と詳細
ros2 topic info /cmd_vel

# トピックをリアルタイムで確認
ros2 topic echo /cmd_vel

# 配信レートを確認（Hzが期待通りか）
ros2 topic hz /scan

# サービス一覧
ros2 service list

# パラメーター確認
ros2 param list /robot_node
ros2 param get /robot_node max_speed
```

---

## ros2 bag でデータを記録・再生

センサーデータやトピックを記録して後で再生できます。実機でのテスト結果を繰り返し再生して開発できるので非常に便利。

```bash
# 全トピックを記録
ros2 bag record -a

# 特定トピックだけ記録
ros2 bag record /scan /odom /cmd_vel -o my_recording

# 記録内容を確認
ros2 bag info my_recording/

# 再生（実時間）
ros2 bag play my_recording/

# 再生速度を変える（0.5倍速）
ros2 bag play my_recording/ --rate 0.5

# ループ再生
ros2 bag play my_recording/ --loop
```

### 活用例：LiDARデータのデバッグ

```bash
# 実機でデータ収集
ros2 bag record /scan /odom -o lidar_test

# デスクで再生しながらアルゴリズムをデバッグ
ros2 bag play lidar_test/ --loop &
ros2 run my_robot obstacle_detector
```

---

## rqt でGUIデバッグ

```bash
# rqt起動
rqt

# よく使うプラグイン
# Plugins → Topics → Topic Monitor（トピックをGUIで監視）
# Plugins → Visualization → Plot（数値をグラフで表示）
# Plugins → Robot Tools → TF Tree（座標変換ツリー確認）
# Plugins → Logging → Console（ログをフィルタリング）
```

### rqt_plotでセンサーデータを可視化

```bash
# cmd_velのlinear.xをリアルタイムプロット
rqt_plot /cmd_vel/linear/x /cmd_vel/angular/z
```

---

## rviz2 で空間的なデバッグ

```bash
rviz2
```

**よく使う表示**
- `LaserScan` → LiDARの点を表示
- `Path` → ナビゲーションの経路
- `TF` → 座標フレームを矢印で表示
- `MarkerArray` → 障害物などを3D表示

### カスタムMarkerを表示

```python
from visualization_msgs.msg import Marker

class DebugVisualizer(Node):
    def __init__(self):
        super().__init__('debug_viz')
        self.marker_pub = self.create_publisher(Marker, 'debug_marker', 10)

    def publish_sphere(self, x: float, y: float, label: str = ''):
        marker = Marker()
        marker.header.frame_id = 'map'
        marker.header.stamp = self.get_clock().now().to_msg()
        marker.type = Marker.SPHERE
        marker.action = Marker.ADD
        marker.pose.position.x = x
        marker.pose.position.y = y
        marker.pose.position.z = 0.0
        marker.scale.x = 0.2
        marker.scale.y = 0.2
        marker.scale.z = 0.2
        marker.color.r = 1.0
        marker.color.a = 1.0  # alpha必須
        self.marker_pub.publish(marker)
```

---

## よくあるエラーと解決策

### `[ERROR] [rcl]: Error in rcl_wait` でクラッシュ

マルチスレッドのコールバック競合が多い。`MultiThreadedExecutor` を使っているか確認。

```python
# NG
rclpy.spin(node)

# OK（コールバックが並列実行される）
from rclpy.executors import MultiThreadedExecutor
executor = MultiThreadedExecutor()
executor.add_node(node)
executor.spin()
```

### `Warning: TF_OLD_DATA`

古いTFデータが来ている。`use_sim_time` の設定ミスが多い。

```bash
# シミュレーターを使っているなら
ros2 param set /robot_node use_sim_time true
```

### ノードは起動しているのにトピックがない

`QoS`（通信品質設定）の不一致。パブリッシャーとサブスクライバーのQoSが合っていないと通信しない。

```python
from rclpy.qos import QoSProfile, ReliabilityPolicy

# センサーデータはBEST_EFFORTが多い
qos = QoSProfile(
    reliability=ReliabilityPolicy.BEST_EFFORT,
    depth=10
)
self.sub = self.create_subscription(LaserScan, 'scan', self.cb, qos)
```

### `colcon build` が通らない

```bash
# 依存パッケージが足りない場合
rosdep install --from-paths src --ignore-src -r -y

# キャッシュが壊れているとき
rm -rf build/ install/ log/
colcon build --symlink-install
```

---

## ログレベルを活用する

```python
self.get_logger().debug('詳細なデバッグ情報')   # 通常は非表示
self.get_logger().info('通常の情報')
self.get_logger().warn('警告（処理は続く）')
self.get_logger().error('エラー（処理に問題）')
self.get_logger().fatal('致命的なエラー')
```

```bash
# デバッグログを表示する
ros2 run my_robot robot_node --ros-args --log-level debug
```

---

## まとめ

1. `ros2 topic echo` でまず通信確認
2. `ros2 bag` で実機データを記録してデスクでデバッグ
3. `rqt_plot` で数値の時系列を確認
4. `rviz2` で空間的な問題を可視化
5. エラーログのキーワードでQoS・TF・スレッド競合を疑う
