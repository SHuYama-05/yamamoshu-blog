---
title: '制御バリア関数（CBF）でロボットの安全な自律移動を実現する【数式・実装付き】'
description: '制御バリア関数（Control Barrier Function, CBF）の数学的な概念から、ROS2を使ったロボット障害物回避への応用まで実装例付きで解説します。LLMとの組み合わせ方も紹介。'
pubDate: '2026-06-05'
tags: ['ROS2', 'CBF', '制御理論', '安全', 'Python']
---

自律移動ロボットが人や障害物に絶対に衝突しないようにするには、どうすればいいでしょうか。

「障害物を検知したら止まる」だけでは、高速で近づいた場合に間に合わないことがあります。制御バリア関数（CBF）は、**数学的に衝突が起こらないことを証明しながら**制御入力を計算する手法です。

---

## CBFの直感的な理解

普通の制御：「目標に向かって進め」

CBF付き制御：「目標に向かって進め。ただし**安全集合の外に出ようとするとき自動的に入力を修正する**」

```
安全集合 C = {x | h(x) ≥ 0}

例: ロボットと障害物の距離が0.5m以上の状態
  h(x) = ||p_robot - p_obs||² - r²   (r = 0.5m)
  h(x) ≥ 0 ⟺ 距離 ≥ 0.5m（安全）
  h(x) < 0 ⟺ 距離 < 0.5m（危険、ここには入れない）
```

障害物に近づくほど「安全のための修正力」が強くなり、安全集合の境界から押し返されます。

---

## 数学的な背景

CBF条件は次のように定義されます。

```
dh/dt + α(h(x)) ≥ 0

- h(x): バリア関数（安全集合を定義）
- α: クラスK関数（α(0) = 0, 単調増加）
  よく使うのは α(h) = γh（γ > 0 の定数）
```

これを制御入力 u について解くと、「安全性を保ちながら目標に向かう最小限の修正」が求まります。最適化問題（QP: 二次計画問題）として解きます。

```
minimize ||u - u_nominal||²     （元の制御入力からの変化を最小化）
subject to:
  ∇h(x)·f(x, u) + α·h(x) ≥ 0   （CBF条件：安全性の保証）
  u_min ≤ u ≤ u_max              （入力制約）
```

---

## Pythonでの実装

### 基本的なCBFコントローラー（2Dロボット）

```python
import numpy as np
from scipy.optimize import minimize, LinearConstraint
from dataclasses import dataclass

@dataclass
class Obstacle:
    x: float
    y: float
    radius: float  # 障害物の大きさ（ロボットのサイズを含む）

class CBFController:
    def __init__(self, safety_margin: float = 0.5, alpha: float = 1.0):
        """
        safety_margin: 安全距離 (m)
        alpha: CBFゲイン（大きいほど安全集合に近づいたとき強く反発する）
        """
        self.safety_margin = safety_margin
        self.alpha = alpha

    def barrier_function(self, robot_pos: np.ndarray, obs: Obstacle) -> float:
        """h(x) = ||p_robot - p_obs||² - (r + margin)²"""
        dx = robot_pos[0] - obs.x
        dy = robot_pos[1] - obs.y
        dist_sq = dx**2 + dy**2
        threshold = (obs.radius + self.safety_margin)**2
        return dist_sq - threshold

    def barrier_gradient(self, robot_pos: np.ndarray, obs: Obstacle) -> np.ndarray:
        """∇h(x) = 2(p_robot - p_obs)"""
        return 2 * np.array([robot_pos[0] - obs.x, robot_pos[1] - obs.y])

    def safe_control(
        self,
        nominal_u: np.ndarray,
        robot_pos: np.ndarray,
        obstacles: list[Obstacle],
        u_max: float = 0.5
    ) -> np.ndarray:
        """
        nominal_u: 目標達成のための通常制御入力 [vx, vy]
        robot_pos: ロボット位置 [x, y]
        """
        def objective(u):
            # 元の入力からの変化を最小化
            return np.linalg.norm(u - nominal_u)**2

        constraints = []
        for obs in obstacles:
            h = self.barrier_function(robot_pos, obs)
            grad_h = self.barrier_gradient(robot_pos, obs)

            # CBF条件: grad_h · u + alpha * h >= 0
            def cbf_constraint(u, grad_h=grad_h, h=h):
                return float(grad_h @ u + self.alpha * h)

            constraints.append({'type': 'ineq', 'fun': cbf_constraint})

        # 入力制約
        bounds = [(-u_max, u_max), (-u_max, u_max)]

        result = minimize(
            objective,
            nominal_u,
            method='SLSQP',
            bounds=bounds,
            constraints=constraints,
            options={'ftol': 1e-6}
        )

        if result.success:
            return result.x
        else:
            # 最適化失敗時は停止
            return np.zeros(2)
```

### ROS2ノードへの統合

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
from nav_msgs.msg import Odometry
from sensor_msgs.msg import LaserScan
import numpy as np

class CBFRobotNode(Node):
    def __init__(self):
        super().__init__('cbf_robot')

        self.cbf = CBFController(safety_margin=0.5, alpha=2.0)
        self.robot_pos = np.zeros(2)
        self.obstacles = []

        self.cmd_pub = self.create_publisher(Twist, 'cmd_vel', 10)
        self.odom_sub = self.create_subscription(
            Odometry, 'odom', self.odom_callback, 10
        )
        self.scan_sub = self.create_subscription(
            LaserScan, 'scan', self.scan_callback, 10
        )

        # 目標位置（例: 2m前）
        self.goal = np.array([2.0, 0.0])
        self.kp = 0.5  # 比例ゲイン

        self.create_timer(0.1, self.control_loop)

    def odom_callback(self, msg: Odometry):
        self.robot_pos = np.array([
            msg.pose.pose.position.x,
            msg.pose.pose.position.y
        ])

    def scan_callback(self, msg: LaserScan):
        """LiDARデータから障害物を検出"""
        self.obstacles = []
        angle = msg.angle_min
        for r in msg.ranges:
            if msg.range_min < r < msg.range_max:
                # 極座標 → 直交座標（ロボット座標系）
                ox = self.robot_pos[0] + r * np.cos(angle)
                oy = self.robot_pos[1] + r * np.sin(angle)
                self.obstacles.append(Obstacle(x=ox, y=oy, radius=0.1))
            angle += msg.angle_increment

    def control_loop(self):
        # 目標への比例制御（nominal入力）
        error = self.goal - self.robot_pos
        dist_to_goal = np.linalg.norm(error)

        if dist_to_goal < 0.1:
            self.get_logger().info('ゴール到達！')
            self.cmd_pub.publish(Twist())
            return

        nominal_u = self.kp * error / dist_to_goal  # 単位ベクトル × ゲイン

        # CBFで安全な入力に修正
        safe_u = self.cbf.safe_control(nominal_u, self.robot_pos, self.obstacles)

        # 修正された入力でCBFが介入したかログ出力
        if np.linalg.norm(safe_u - nominal_u) > 0.05:
            self.get_logger().info(
                f'CBF介入: nominal={nominal_u} → safe={safe_u}'
            )

        # Twistに変換して送信
        cmd = Twist()
        cmd.linear.x = float(safe_u[0])
        cmd.linear.y = float(safe_u[1])
        self.cmd_pub.publish(cmd)
```

---

## LLMとCBFの二層構造

研究で実装している構成です。

```
LLM（高レベル計画）
├── 「人を避けながら目的地に向かって」を解析
├── ウェイポイント列を生成
└── 応答に数秒かかっても問題ない

       ↓ ウェイポイントを渡す

CBFコントローラー（低レベル安全保証）
├── 10〜50Hzでリアルタイム動作
├── LiDARデータで動的障害物を検出
└── 数学的に衝突を防ぐことを保証
```

LLMは賢いが遅い、CBFは単純だが速い・安全。この組み合わせがうまく機能しています。

```python
class LLMCBFController(Node):
    def __init__(self):
        super().__init__('llm_cbf_controller')
        self.cbf = CBFController()
        self.llm_waypoints = []  # LLMが生成したウェイポイント
        self.current_waypoint_idx = 0

        # 低頻度でLLMを呼ぶ（0.1Hz = 10秒ごと）
        self.create_timer(10.0, self.update_plan_with_llm)
        # 高頻度でCBFを適用（10Hz）
        self.create_timer(0.1, self.safe_control_loop)

    def update_plan_with_llm(self):
        # LLMで経路を更新（別スレッドで実行）
        ...

    def safe_control_loop(self):
        # CBFでリアルタイムに安全性を保証
        ...
```

---

## まとめ

CBFは「安全集合から外に出ない」ことを数学的に保証できる強力な手法です。

- **バリア関数 h(x)** で安全集合を定義
- **CBF条件**を制約としてQP問題を解く
- **元の制御入力からの変化を最小化**しながら安全性を保つ
- LLMなど高レベル計画と組み合わせると特に強力
