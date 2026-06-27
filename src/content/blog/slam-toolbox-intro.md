---
title: 'ROS2 + slam_toolboxでSLAM入門【地図生成・ローカライゼーション実装】'
description: 'ROS2でslam_toolboxを使ったSLAM（自己位置推定と地図生成の同時実行）の実装方法を解説。LiDARデータから地図を作り、Nav2で自律ナビゲーションにつなげるまでの手順を説明します。'
pubDate: '2026-06-22'
tags: ['ROS2', 'SLAM', 'slam_toolbox', 'Nav2', 'LiDAR']
---

SLAM（Simultaneous Localization and Mapping）はロボットが「自分の位置を推定しながら地図を作る」技術です。ROS2では `slam_toolbox` が標準的に使われています。

---

## SLAMとは

```
既知の地図なし
    ↓
LiDARで周囲を計測
    ↓
「今どこにいるか」と「地図」を同時に推定
    ↓
走り回るにつれて地図が完成
```

完成した地図を使って次からは自律ナビゲーション（Nav2）ができます。

---

## インストール

```bash
sudo apt install ros-humble-slam-toolbox
```

---

## 基本的な起動

```bash
# オンラインSLAMモード（リアルタイムで地図作成）
ros2 launch slam_toolbox online_async_launch.py \
  use_sim_time:=false \
  params_file:=./slam_params.yaml
```

必要なトピック：
- `/scan` （`sensor_msgs/LaserScan`）← LiDARデータ
- `/odom` （`nav_msgs/Odometry`）← オドメトリ
- TF: `odom → base_link`

---

## slam_toolboxのパラメーター

```yaml
# slam_params.yaml
slam_toolbox:
  ros__parameters:
    # モード設定
    mode: mapping                    # mapping or localization

    # センサー設定
    odom_frame: odom
    map_frame: map
    base_frame: base_link
    scan_topic: /scan

    # 地図解像度（m/pixel）
    resolution: 0.05

    # スキャンマッチング設定
    minimum_travel_distance: 0.5    # 何m動いたら更新するか
    minimum_travel_heading: 0.5     # 何rad回転したら更新するか

    # 地図のサイズ
    map_start_at_dock: true
    debug_logging: false
```

---

## 地図の保存

rviz2でSLAMの様子を確認しながらロボットを動かします。地図ができたら保存。

```bash
# 地図を保存
ros2 run nav2_map_server map_saver_cli -f ~/maps/my_map

# 保存されるファイル
# ~/maps/my_map.yaml  → 地図のメタデータ
# ~/maps/my_map.pgm   → 地図の画像（白=空き、黒=障害物、グレー=不明）
```

---

## 保存した地図でローカライゼーション

地図ができたら、次からはSLAMじゃなくてローカライゼーションモードで起動。

```bash
# 地図サーバーを起動
ros2 run nav2_map_server map_server --ros-args \
  -p yaml_filename:=/home/user/maps/my_map.yaml

# AMCLで自己位置推定（パーティクルフィルター）
ros2 run nav2_amcl amcl

# または slam_toolbox のlocalizationモード
ros2 launch slam_toolbox localization_launch.py \
  map:=/home/user/maps/my_map.yaml
```

---

## Nav2と組み合わせた完全構成

```
slam_toolbox（地図作成 or 自己位置推定）
    ↓ /map トピック
Nav2（経路計画・障害物回避）
    ↓ /cmd_vel
ロボット
```

```bash
# 全部まとめて起動するlaunch file
ros2 launch nav2_bringup bringup_launch.py \
  use_sim_time:=false \
  slam:=True \
  map:=/home/user/maps/my_map.yaml
```

---

## よくある問題

### 地図がズレていく

オドメトリの精度が低いと地図がズレます。エンコーダーのキャリブレーションを確認してください。

```bash
# オドメトリの精度確認：直進して戻ってきたときのズレを測定
ros2 topic echo /odom --once
```

### スキャンが更新されない

LiDARのQoSを確認。slam_toolboxはデフォルトで `BEST_EFFORT` を期待しています。

```python
qos = QoSProfile(
    reliability=ReliabilityPolicy.BEST_EFFORT,
    depth=10
)
```

### 地図が黒くなりすぎる

`minimum_travel_distance` が小さすぎてノイズを地図に書き込んでいる可能性。値を大きくしてみる。

---

## まとめ

1. `slam_toolbox` をインストールしてLiDAR・オドメトリ・TFを揃える
2. `online_async_launch.py` で起動してロボットを動かす
3. rviz2で地図ができたら `map_saver_cli` で保存
4. 次からはローカライゼーションモード + Nav2で自律ナビゲーション
