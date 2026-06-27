---
title: "ROS2 TF2完全ガイド【座標変換・よくあるエラー・実装例】"
emoji: "📐"
type: "tech"
topics: ["ROS2", "TF2", "Python", "ロボット"]
published: true
---

ROS2でロボット開発をしていると必ず出てくるのがTF2（座標変換ライブラリ）。「`lookupTransform` でエラーが出て進めない」という経験は誰しもあると思います。

---

## TF2とは

ロボットの各フレーム（base_link、laser_frame、map、odomなど）の間の座標変換を管理するライブラリ。

```
map
 └── odom
      └── base_link
           ├── laser_frame
           └── camera_frame
```

---

## 基本的な使い方

### 変換のブロードキャスト

```python
from tf2_ros import TransformBroadcaster
from geometry_msgs.msg import TransformStamped
import tf_transformations

class TFBroadcasterNode(Node):
    def __init__(self):
        super().__init__('tf_broadcaster')
        self.br = TransformBroadcaster(self)
        self.create_timer(0.1, self.broadcast)

    def broadcast(self):
        t = TransformStamped()
        t.header.stamp = self.get_clock().now().to_msg()
        t.header.frame_id = 'odom'
        t.child_frame_id = 'base_link'

        # 位置
        t.transform.translation.x = 1.0
        t.transform.translation.y = 0.5
        t.transform.translation.z = 0.0

        # 姿勢（クォータニオン）
        q = tf_transformations.quaternion_from_euler(0, 0, 0.785)  # 45度
        t.transform.rotation.x = q[0]
        t.transform.rotation.y = q[1]
        t.transform.rotation.z = q[2]
        t.transform.rotation.w = q[3]

        self.br.sendTransform(t)
```

### 変換のルックアップ

```python
from tf2_ros import Buffer, TransformListener
import tf2_geometry_msgs

class TFListenerNode(Node):
    def __init__(self):
        super().__init__('tf_listener')
        self.tf_buffer = Buffer()
        self.tf_listener = TransformListener(self.tf_buffer, self)
        self.create_timer(1.0, self.lookup)

    def lookup(self):
        try:
            transform = self.tf_buffer.lookup_transform(
                'map',        # 変換先フレーム
                'base_link',  # 変換元フレーム
                rclpy.time.Time()  # 最新の変換
            )
            pos = transform.transform.translation
            self.get_logger().info(f'ロボット位置: x={pos.x:.2f}, y={pos.y:.2f}')

        except Exception as e:
            self.get_logger().warn(f'TFエラー: {e}')
```

---

## よくあるエラーと解決策

### `Could not find a connection between 'map' and 'base_link'`

TFツリーが繋がっていない。

```bash
# TFツリーを確認
ros2 run tf2_tools view_frames

# ブロードキャストされているフレームを確認
ros2 topic echo /tf
```

### `Lookup would require extrapolation into the future`

タイムスタンプが未来になっている。シミュレーターでは `use_sim_time: true` を忘れずに。

```python
# 最新のTFを使う（タイムスタンプ指定なし）
transform = self.tf_buffer.lookup_transform(
    'map', 'base_link',
    rclpy.time.Time()  # Time()で最新を取得
)
```

### `Transform timed out`

TFがタイムアウト。待機時間を設定する。

```python
transform = self.tf_buffer.lookup_transform(
    'map', 'base_link',
    rclpy.time.Time(),
    timeout=rclpy.duration.Duration(seconds=1.0)  # 1秒待つ
)
```

---

## PointStampedの座標変換

```python
from geometry_msgs.msg import PointStamped
import tf2_geometry_msgs  # 必須

def transform_point(self, point_in_laser: PointStamped) -> PointStamped:
    try:
        point_in_map = self.tf_buffer.transform(
            point_in_laser, 'map'
        )
        return point_in_map
    except Exception as e:
        self.get_logger().error(f'変換失敗: {e}')
        return None
```

`tf2_geometry_msgs` のインポートを忘れると `TypeException: None is not registered` というエラーになります。
