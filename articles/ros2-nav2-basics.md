---
title: "ROS2 Nav2で自律ナビゲーション【セットアップ・ウェイポイント・パラメーター調整】"
emoji: "🗺️"
type: "tech"
topics: ["ROS2", "Nav2", "ナビゲーション", "Python", "ロボット"]
published: true
---

ROS2のナビゲーションスタック「Nav2」を使うと、地図を使った自律移動がすぐに実装できます。ただしパラメーターが多く、最初は何を設定すればいいか迷います。

---

## インストール

```bash
sudo apt install ros-humble-nav2-bringup \
                 ros-humble-navigation2 \
                 ros-humble-slam-toolbox
```

---

## 基本的な起動

```bash
# シミュレーターと合わせて起動する場合
ros2 launch nav2_bringup navigation_launch.py \
  use_sim_time:=true \
  params_file:=./nav2_params.yaml
```

---

## ウェイポイントナビゲーション（Python）

```python
import rclpy
from rclpy.node import Node
from rclpy.action import ActionClient
from nav2_msgs.action import NavigateToPose
from geometry_msgs.msg import PoseStamped
import tf_transformations

class WaypointNavigator(Node):
    def __init__(self):
        super().__init__('waypoint_navigator')
        self._client = ActionClient(self, NavigateToPose, 'navigate_to_pose')

    def navigate_to(self, x: float, y: float, yaw: float = 0.0):
        """指定した座標に移動"""
        self._client.wait_for_server()

        goal = NavigateToPose.Goal()
        goal.pose = PoseStamped()
        goal.pose.header.frame_id = 'map'
        goal.pose.header.stamp = self.get_clock().now().to_msg()
        goal.pose.pose.position.x = x
        goal.pose.pose.position.y = y

        q = tf_transformations.quaternion_from_euler(0, 0, yaw)
        goal.pose.pose.orientation.z = q[2]
        goal.pose.pose.orientation.w = q[3]

        self.get_logger().info(f'目標: ({x}, {y})')
        future = self._client.send_goal_async(
            goal,
            feedback_callback=self.feedback_cb
        )
        future.add_done_callback(self.goal_response_cb)

    def feedback_cb(self, feedback_msg):
        dist = feedback_msg.feedback.distance_remaining
        self.get_logger().info(f'残り距離: {dist:.2f}m')

    def goal_response_cb(self, future):
        goal_handle = future.result()
        if not goal_handle.accepted:
            self.get_logger().warn('ゴール拒否')
            return
        result_future = goal_handle.get_result_async()
        result_future.add_done_callback(self.result_cb)

    def result_cb(self, future):
        self.get_logger().info('到着！')


def main():
    rclpy.init()
    nav = WaypointNavigator()

    # 複数ウェイポイントを順番に巡回
    waypoints = [(1.0, 0.0), (1.0, 1.0), (0.0, 1.0), (0.0, 0.0)]
    for x, y in waypoints:
        nav.navigate_to(x, y)

    rclpy.spin(nav)
    rclpy.shutdown()
```

---

## 重要なパラメーター

```yaml
# nav2_params.yaml（最小設定）
controller_server:
  ros__parameters:
    controller_frequency: 20.0
    FollowPath:
      plugin: "nav2_regulated_pure_pursuit_controller::RegulatedPurePursuitController"
      desired_linear_vel: 0.3      # 目標速度 (m/s)
      max_linear_accel: 0.5        # 最大加速度
      max_lateral_accel: 0.5
      use_velocity_scaled_lookahead_dist: true

planner_server:
  ros__parameters:
    GridBased:
      plugin: "nav2_navfn_planner::NavfnPlanner"
      tolerance: 0.5
      use_astar: false

local_costmap:
  local_costmap:
    ros__parameters:
      width: 3
      height: 3
      resolution: 0.05
      inflation_layer:
        inflation_radius: 0.55    # ロボット半径+余裕
```

---

## よくある問題

### ロボットが壁に近づきすぎる

`inflation_radius` を大きくする（ロボット半径 + 0.1〜0.2m）

### 経路が遠回りになる

`NavfnPlanner` の `use_astar: true` で改善することがある

### ゴール付近でぐるぐる回る

`xy_goal_tolerance` と `yaw_goal_tolerance` を緩くする

```yaml
general_goal_checker:
  xy_goal_tolerance: 0.25   # デフォルト0.25
  yaw_goal_tolerance: 0.25
```
