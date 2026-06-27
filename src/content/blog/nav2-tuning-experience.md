---
title: 'Nav2のパラメーターチューニングで学んだこと【実機調整の失敗談と解決策】'
description: 'ROS2 Nav2を実機ロボットに適用したときのパラメーターチューニング経験をまとめました。ロボットが壁に激突する・ゴールに近づけない・経路がぐるぐるするなどの問題の原因と対処法を解説します。'
pubDate: '2026-06-27'
tags: ['ROS2', 'Nav2', 'ナビゲーション', 'チューニング', 'ロボット']
---

Nav2をシミュレーターで動かすのは比較的簡単ですが、実機に持っていくと全然動かないことがほとんどです。実際にチューニングして学んだことをまとめます。

---

## 問題1: ロボットが壁に激突する

### 症状

Nav2がゴールへの経路を計画・追従しているのに、壁スレスレを通ろうとして激突する。

### 原因

`inflation_radius` が小さすぎる。コストマップ上で壁の膨張が足りず、壁のギリギリを「通れる」と判断している。

```yaml
# NG: デフォルト値（ロボットサイズを考慮していない）
local_costmap:
  local_costmap:
    ros__parameters:
      inflation_layer:
        inflation_radius: 0.25  # 小さすぎ
```

### 解決策

`inflation_radius = ロボットの半径 + 余裕（0.1〜0.2m）` に設定する。

```yaml
# OK
local_costmap:
  local_costmap:
    ros__parameters:
      inflation_layer:
        inflation_radius: 0.55   # ロボット半径0.35m + 余裕0.2m
        cost_scaling_factor: 3.0 # 壁に近いほどコストを急激に上げる
```

**重要**: `global_costmap` と `local_costmap` 両方に設定する。どちらか片方だけだと片方は激突、片方は遠回りという不思議な動きになる。

---

## 問題2: ゴール付近でロボットがぐるぐる回り続ける

### 症状

ゴールの近くまで来たのに、行ったり来たりしてなかなかゴール判定が出ない。

### 原因

`xy_goal_tolerance` が小さく、オドメトリのノイズで常に許容範囲から外れている。

```yaml
# NG: 厳しすぎる許容値
general_goal_checker:
  xy_goal_tolerance: 0.05   # 5cm以内に入れ → オドメトリノイズで常に失敗
  yaw_goal_tolerance: 0.05
```

### 解決策

実機のオドメトリ精度に合わせた許容値にする。

```yaml
# OK
general_goal_checker:
  xy_goal_tolerance: 0.25   # 25cmはゆるい気もするが実機では必要
  yaw_goal_tolerance: 0.3   # 約17度
  stateful: true             # ゴール到達後は再チェックしない
```

`stateful: true` は重要。Falseだとゴール判定後もロボットが動いて再び外れたとき「未到達」に戻ってしまう。

---

## 問題3: 狭い場所で立ち往生する

### 症状

廊下の曲がり角など狭い場所で経路が見つからず停止する。

### 原因

`local_costmap` の解像度が荒く、狭い通路を「通れない」と判断している。

```yaml
# NG
local_costmap:
  local_costmap:
    ros__parameters:
      resolution: 0.1  # 10cmセル → 狭い通路が通れない判定になる
```

### 解決策

解像度を細かくする。ただし計算負荷が上がる。

```yaml
# OK
local_costmap:
  local_costmap:
    ros__parameters:
      resolution: 0.05  # 5cmセル
      width: 3.0        # 3m × 3mの局所地図
      height: 3.0
```

加えて、`RecoveryNode` を有効にしてスタックしたときに自力回復できるようにする。

```yaml
# nav2_params.yaml
recoveries_server:
  ros__parameters:
    costmap_topic: local_costmap/costmap_raw
    footprint_topic: local_costmap/published_footprint
    cycle_frequency: 10.0
    recovery_plugins: ["spin", "back_up", "wait"]
    spin:
      plugin: "nav2_recoveries/Spin"
    back_up:
      plugin: "nav2_recoveries/BackUp"
```

---

## 問題4: 急に経路が変わってロボットが揺れる

### 症状

直進中に突然大きく曲がる動作が入り、ガクガクした動きになる。

### 原因

`controller_frequency` が高すぎて、LiDARノイズのたびに経路を再計算している。

```yaml
# NG
controller_server:
  ros__parameters:
    controller_frequency: 50.0  # 高すぎ
```

### 解決策

適切な頻度に下げ、速度の変化率（加速度）を制限する。

```yaml
# OK
controller_server:
  ros__parameters:
    controller_frequency: 10.0

    FollowPath:
      plugin: "nav2_regulated_pure_pursuit_controller::RegulatedPurePursuitController"
      desired_linear_vel: 0.3
      max_linear_accel: 0.3       # 加速度を制限（急加速を防ぐ）
      max_linear_decel: 0.5
      max_angular_accel: 1.0
      use_velocity_scaled_lookahead_dist: true
      min_lookahead_dist: 0.3
      max_lookahead_dist: 0.9
```

---

## 問題5: シミュレーターでは動くのに実機で動かない

よくある原因チェックリスト：

```bash
# 1. トピック名の確認
ros2 topic list | grep -E "scan|odom|cmd_vel"
# シミュレーターと実機でトピック名が違うことがある

# 2. TFツリーの確認
ros2 run tf2_tools view_frames
# base_link → odom → map が繋がっているか

# 3. オドメトリのフレームIDを確認
ros2 topic echo /odom | grep frame_id
# "odom" になっているか（"base_link" になっているとTFエラー）

# 4. LiDARのQoSを確認
ros2 topic info /scan -v
# "Reliability: BEST_EFFORT" の場合、Nav2の設定も合わせる必要がある
```

### LiDARのQoSをBEST_EFFORTに合わせる

```yaml
# nav2_params.yaml
local_costmap:
  local_costmap:
    ros__parameters:
      obstacle_layer:
        scan:
          topic: /scan
          obstacle_max_range: 5.0
          obstacle_min_range: 0.0
          raytrace_max_range: 6.0
          raytrace_min_range: 0.0
          max_obstacle_height: 2.0
          min_obstacle_height: 0.0
          clearing: true
          marking: true
          data_type: "LaserScan"
          sensor_frame: ""
          inf_is_valid: false
          qos_overrides:
            /scan:
              reliability: best_effort  # これを追加
```

---

## チューニングの順番

1. まず `inflation_radius` を正しく設定（衝突防止の最優先）
2. `xy_goal_tolerance` をオドメトリ精度に合わせる
3. `controller_frequency` と加速度制限で動きを滑らかに
4. 狭い場所でのスタック対策はリカバリー設定

シミュレーターと実機でパラメーターが違うのは当然と思って、実機専用の設定ファイルを用意するのが現実的です。
