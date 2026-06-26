---
title: 'ROS2のTF2（座標変換）を完全理解する【よくあるエラー対処法あり】'
description: 'ROS2開発で避けて通れないTF2（Transform）の概念と使い方を徹底解説。座標系の変換・ブロードキャスト・リスニングを実装例で説明し、よくあるエラーの対処法も紹介します。'
pubDate: '2026-06-10'
tags: ['ROS2', 'TF2', '座標変換']
---

ROS2でロボット開発をしていると必ずぶつかるのがTF2（Transform）関連のエラーです。

```
[WARN] Lookup would require extrapolation into the past
[ERROR] "base_link" passed to lookupTransform argument target_frame does not exist
```

こういうエラーで数時間溶かした経験がある人は多いはずです。この記事でTF2を根本から理解します。

---

## TF2とは何か

TF2はROS2の**座標変換ライブラリ**です。ロボットには複数の「座標系（フレーム）」があります。

```
world（世界座標）
  └── map（地図座標）
        └── odom（オドメトリ座標）
              └── base_link（ロボット本体）
                    ├── laser_frame   （LiDAR）
                    ├── camera_frame  （カメラ）
                    └── imu_frame     （IMU）
```

「LiDARが壁を検出した座標」を「地図上の座標」に変換するとき、このツリーをたどって計算します。TF2はそのツリーの管理と計算を自動でやってくれます。

### 各フレームの意味

| フレーム | 意味 | 注意点 |
|---------|------|--------|
| `map` | 地図の原点 | SLAMで決まる絶対座標 |
| `odom` | 起動時点からの累積移動量 | ドリフトする（時間とともにズレる） |
| `base_link` | ロボット本体の中心 | 全センサーの基準点 |
| `base_footprint` | ロボットの接地点 | `base_link` の地面投影 |

---

## TFブロードキャスター（座標を発信）

センサーの取り付け位置など、**変化しない**座標変換は静的ブロードキャスターを使います。

### 静的ブロードキャスター（センサー取り付け位置）

```python
from tf2_ros.static_transform_broadcaster import StaticTransformBroadcaster
from geometry_msgs.msg import TransformStamped

class SensorTFPublisher(Node):
    def __init__(self):
        super().__init__('sensor_tf_publisher')
        self.broadcaster = StaticTransformBroadcaster(self)
        self.publish_static_transforms()

    def publish_static_transforms(self):
        transforms = []

        # LiDARはロボット中心から前方10cm・上方20cmに設置
        lidar_tf = TransformStamped()
        lidar_tf.header.stamp = self.get_clock().now().to_msg()
        lidar_tf.header.frame_id = 'base_link'
        lidar_tf.child_frame_id = 'laser_frame'
        lidar_tf.transform.translation.x = 0.10
        lidar_tf.transform.translation.y = 0.00
        lidar_tf.transform.translation.z = 0.20
        lidar_tf.transform.rotation.w = 1.0  # 回転なし
        transforms.append(lidar_tf)

        # カメラは前方15cm・上方25cm・下向き15度
        import math
        camera_tf = TransformStamped()
        camera_tf.header.stamp = self.get_clock().now().to_msg()
        camera_tf.header.frame_id = 'base_link'
        camera_tf.child_frame_id = 'camera_frame'
        camera_tf.transform.translation.x = 0.15
        camera_tf.transform.translation.z = 0.25
        # Y軸まわりに-15度回転（下向き）
        angle = math.radians(-15)
        camera_tf.transform.rotation.y = math.sin(angle / 2)
        camera_tf.transform.rotation.w = math.cos(angle / 2)
        transforms.append(camera_tf)

        self.broadcaster.sendTransform(transforms)
```

### 動的ブロードキャスター（ロボットの現在位置）

ロボットが移動するたびに更新が必要な座標変換には動的ブロードキャスターを使います。

```python
from tf2_ros import TransformBroadcaster
import math

class OdomTFPublisher(Node):
    def __init__(self):
        super().__init__('odom_tf_publisher')
        self.broadcaster = TransformBroadcaster(self)
        self.x = 0.0
        self.y = 0.0
        self.theta = 0.0
        # オドメトリを受け取ってTFを更新
        self.create_subscription(Odometry, 'odom', self.odom_callback, 10)

    def odom_callback(self, msg: Odometry):
        t = TransformStamped()
        t.header.stamp = msg.header.stamp
        t.header.frame_id = 'odom'
        t.child_frame_id = 'base_link'
        t.transform.translation.x = msg.pose.pose.position.x
        t.transform.translation.y = msg.pose.pose.position.y
        t.transform.translation.z = 0.0
        t.transform.rotation = msg.pose.pose.orientation
        self.broadcaster.sendTransform(t)
```

---

## TFリスナー（座標変換を取得）

```python
from tf2_ros import TransformListener, Buffer
from tf2_geometry_msgs import do_transform_point
from geometry_msgs.msg import PointStamped

class TFReader(Node):
    def __init__(self):
        super().__init__('tf_reader')
        self.tf_buffer = Buffer()
        self.tf_listener = TransformListener(self.tf_buffer, self)

    def get_robot_position(self) -> tuple[float, float] | None:
        """mapフレームでのロボット位置を取得"""
        try:
            transform = self.tf_buffer.lookup_transform(
                'map',              # ターゲットフレーム（変換先）
                'base_link',        # ソースフレーム（変換元）
                rclpy.time.Time(),  # 最新の変換を使う
                timeout=rclpy.duration.Duration(seconds=1.0)
            )
            x = transform.transform.translation.x
            y = transform.transform.translation.y
            return x, y

        except Exception as e:
            self.get_logger().warn(f'TF取得失敗: {e}')
            return None

    def transform_point_to_map(self, point_in_laser: PointStamped) -> PointStamped | None:
        """laser_frameの点をmapフレームに変換"""
        try:
            transform = self.tf_buffer.lookup_transform(
                'map',
                point_in_laser.header.frame_id,
                point_in_laser.header.stamp,
                timeout=rclpy.duration.Duration(seconds=0.5)
            )
            return do_transform_point(point_in_laser, transform)
        except Exception as e:
            self.get_logger().warn(f'座標変換失敗: {e}')
            return None
```

---

## デバッグコマンド

```bash
# TFツリーをPDFで可視化（超便利）
ros2 run tf2_tools view_frames
# カレントディレクトリに frames.pdf が生成される

# 特定フレーム間の変換をリアルタイム確認
ros2 run tf2_ros tf2_echo map base_link

# TFが届いているか確認
ros2 topic echo /tf
ros2 topic echo /tf_static

# フレームの一覧
ros2 run tf2_ros tf2_monitor
```

---

## よくあるエラーと対処法

### `LookupException: "base_link" passed to lookupTransform`

**原因**: `base_link` フレームがTFツリーに存在しない

```bash
# TFツリーを確認
ros2 run tf2_tools view_frames
# base_link が表示されているか確認
```

**対処**: `base_link` をブロードキャストするノードが起動しているか確認。`robot_state_publisher` が動いていればURDFから自動生成されます。

### `ExtrapolationException: Lookup would require extrapolation`

**原因**: 要求した時刻のTFデータがバッファに存在しない（タイムスタンプのズレ）

```python
# NG: 特定の時刻を指定している
transform = tf_buffer.lookup_transform('map', 'base_link', msg.header.stamp)

# OK: 最新のTFを使う
transform = tf_buffer.lookup_transform('map', 'base_link', rclpy.time.Time())

# OK: 少し古いデータを許容する
transform = tf_buffer.lookup_transform(
    'map', 'base_link',
    rclpy.time.Time(),
    timeout=rclpy.duration.Duration(seconds=1.0)
)
```

### `ConnectivityException: Could not find a connection`

**原因**: ソースとターゲットのフレームがツリーでつながっていない

**対処**: `view_frames` でTFツリーを確認し、途切れている箇所のブロードキャスターが動いているか確認する。

---

## まとめ

- TF2はロボット内の座標系をツリー構造で管理するライブラリ
- **変わらない**座標変換 → `StaticTransformBroadcaster`
- **変わる**座標変換 → `TransformBroadcaster`（毎フレーム更新）
- `view_frames` でツリーを可視化するとデバッグが格段に楽になる
- エラーが出たらまずタイムスタンプを `rclpy.time.Time()` にして最新を使うようにする
