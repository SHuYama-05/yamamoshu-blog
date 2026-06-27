---
title: 'matplotlibでROS2トピックをリアルタイム可視化する【センサーデータグラフ】'
description: 'ROS2のトピックデータをmatplotlibでリアルタイムにグラフ表示する方法を解説。cmd_velやLiDARデータを動的にプロットして、ロボットの動作をデバッグ・分析します。'
pubDate: '2026-06-26'
tags: ['ROS2', 'Python', 'matplotlib', '可視化', 'デバッグ']
---

rqt_plotよりも自由度が高く、rviz2ほど重くない。matplotlibでROS2トピックをリアルタイムに可視化する方法をまとめます。

---

## 基本的なリアルタイムプロット

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
import matplotlib.pyplot as plt
import matplotlib.animation as animation
from collections import deque
import threading

class RealtimePlotter(Node):
    def __init__(self):
        super().__init__('realtime_plotter')

        # データバッファ（最新100点を保持）
        self.max_points = 100
        self.times = deque(maxlen=self.max_points)
        self.linear_x = deque(maxlen=self.max_points)
        self.angular_z = deque(maxlen=self.max_points)
        self.start_time = self.get_clock().now()
        self._lock = threading.Lock()

        self.sub = self.create_subscription(
            Twist, 'cmd_vel', self.callback, 10
        )

    def callback(self, msg: Twist):
        now = self.get_clock().now()
        elapsed = (now - self.start_time).nanoseconds / 1e9  # 秒

        with self._lock:
            self.times.append(elapsed)
            self.linear_x.append(msg.linear.x)
            self.angular_z.append(msg.angular.z)


def main():
    rclpy.init()
    node = RealtimePlotter()

    # ROS2をバックグラウンドスレッドで回す
    ros_thread = threading.Thread(
        target=rclpy.spin, args=(node,), daemon=True
    )
    ros_thread.start()

    # matplotlibのアニメーション
    fig, (ax1, ax2) = plt.subplots(2, 1, figsize=(10, 6))
    fig.suptitle('cmd_vel リアルタイムモニター')

    def update(frame):
        with node._lock:
            times = list(node.times)
            linear_x = list(node.linear_x)
            angular_z = list(node.angular_z)

        if not times:
            return

        ax1.clear()
        ax1.plot(times, linear_x, 'b-', linewidth=2)
        ax1.set_ylabel('linear.x (m/s)')
        ax1.set_ylim(-0.6, 0.6)
        ax1.axhline(y=0, color='k', linestyle='--', alpha=0.3)
        ax1.grid(True, alpha=0.3)

        ax2.clear()
        ax2.plot(times, angular_z, 'r-', linewidth=2)
        ax2.set_ylabel('angular.z (rad/s)')
        ax2.set_xlabel('時間 (s)')
        ax2.set_ylim(-2.0, 2.0)
        ax2.axhline(y=0, color='k', linestyle='--', alpha=0.3)
        ax2.grid(True, alpha=0.3)

    ani = animation.FuncAnimation(
        fig, update, interval=100, cache_frame_data=False
    )
    plt.tight_layout()
    plt.show()

    node.destroy_node()
    rclpy.shutdown()
```

---

## LiDARデータを極座標でリアルタイム表示

```python
from sensor_msgs.msg import LaserScan
import numpy as np

class LiDARPlotter(Node):
    def __init__(self):
        super().__init__('lidar_plotter')
        self.ranges = []
        self.angle_min = 0.0
        self.angle_increment = 0.0
        self._lock = threading.Lock()

        self.sub = self.create_subscription(
            LaserScan, 'scan', self.callback, 10
        )

    def callback(self, msg: LaserScan):
        with self._lock:
            self.ranges = list(msg.ranges)
            self.angle_min = msg.angle_min
            self.angle_increment = msg.angle_increment


def plot_lidar():
    rclpy.init()
    node = LiDARPlotter()

    ros_thread = threading.Thread(
        target=rclpy.spin, args=(node,), daemon=True
    )
    ros_thread.start()

    fig, ax = plt.subplots(subplot_kw={'projection': 'polar'}, figsize=(8, 8))
    ax.set_title('LiDAR リアルタイム表示')
    scatter = ax.scatter([], [], s=2, c='blue', alpha=0.5)

    def update(frame):
        with node._lock:
            ranges = list(node.ranges)
            angle_min = node.angle_min
            angle_inc = node.angle_increment

        if not ranges:
            return scatter,

        n = len(ranges)
        angles = [angle_min + i * angle_inc for i in range(n)]

        # inf を除外
        valid = [(a, r) for a, r in zip(angles, ranges)
                 if r != float('inf') and r == r]  # NaN除外

        if valid:
            angles_v, ranges_v = zip(*valid)
            scatter.set_offsets(np.c_[angles_v, ranges_v])
            ax.set_rmax(max(ranges_v) * 1.1)

        return scatter,

    ani = animation.FuncAnimation(
        fig, update, interval=100, blit=False, cache_frame_data=False
    )
    plt.show()
```

---

## 複数トピックを同時プロット

```python
from nav_msgs.msg import Odometry

class MultiTopicPlotter(Node):
    def __init__(self):
        super().__init__('multi_plotter')
        self.max_points = 200
        self._lock = threading.Lock()

        # 各トピックのバッファ
        self.odom_x = deque(maxlen=self.max_points)
        self.odom_y = deque(maxlen=self.max_points)
        self.cmd_vx = deque(maxlen=self.max_points)

        self.create_subscription(Odometry, 'odom', self.odom_cb, 10)
        self.create_subscription(Twist, 'cmd_vel', self.cmd_cb, 10)

    def odom_cb(self, msg):
        with self._lock:
            self.odom_x.append(msg.pose.pose.position.x)
            self.odom_y.append(msg.pose.pose.position.y)

    def cmd_cb(self, msg):
        with self._lock:
            self.cmd_vx.append(msg.linear.x)


def plot_trajectory():
    rclpy.init()
    node = MultiTopicPlotter()
    ros_thread = threading.Thread(target=rclpy.spin, args=(node,), daemon=True)
    ros_thread.start()

    fig, (ax_traj, ax_vel) = plt.subplots(1, 2, figsize=(12, 5))

    def update(frame):
        with node._lock:
            x = list(node.odom_x)
            y = list(node.odom_y)
            vx = list(node.cmd_vx)

        # 軌跡プロット
        ax_traj.clear()
        if x:
            ax_traj.plot(x, y, 'b-', linewidth=1.5)
            ax_traj.plot(x[-1], y[-1], 'ro', markersize=10)  # 現在位置
        ax_traj.set_aspect('equal')
        ax_traj.set_title('走行軌跡')
        ax_traj.grid(True)

        # 速度プロット
        ax_vel.clear()
        if vx:
            ax_vel.plot(range(len(vx)), vx, 'g-')
        ax_vel.set_title('速度 (m/s)')
        ax_vel.set_ylim(-0.6, 0.6)
        ax_vel.grid(True)

    ani = animation.FuncAnimation(fig, update, interval=100, cache_frame_data=False)
    plt.tight_layout()
    plt.show()
```

---

## Tips

- `threading.Lock()` でデータ競合を防ぐ
- `deque(maxlen=N)` で自動的に古いデータを破棄
- `cache_frame_data=False` をつけないとメモリが増え続ける
- `interval=100`（100ms）でだいたい10FPSのプロット
