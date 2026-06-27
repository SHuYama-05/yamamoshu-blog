---
title: "制御バリア関数（CBF）でロボットの安全な自律移動を実現する【数式・実装付き】"
emoji: "🛡️"
type: "tech"
topics: ["ROS2", "制御理論", "Python", "ロボット", "安全"]
published: true
---

自律移動ロボットが人や障害物に絶対に衝突しないようにするには？「障害物を検知したら止まる」だけでは高速接近に間に合わない。制御バリア関数（CBF）は**数学的に衝突が起こらないことを証明しながら**制御入力を計算する手法です。

---

## CBFの直感的な理解

普通の制御：「目標に向かって進め」

CBF付き制御：「目標に向かって進め。ただし**安全集合の外に出ようとするとき自動的に入力を修正する**」

```
安全集合 C = {x | h(x) ≥ 0}

例: ロボットと障害物の距離が0.5m以上の状態
  h(x) = ||p_robot - p_obs||² - r²   (r = 0.5m)
  h(x) ≥ 0 ⟺ 安全
  h(x) < 0 ⟺ 危険（ここには入れない）
```

---

## 数学的背景

CBF条件：

```
dh/dt + α(h(x)) ≥ 0

α(h) = γh（γ > 0）とすると：
```

これをQP（二次計画問題）として解く：

```
minimize ||u - u_nominal||²
subject to:
  ∇h(x)·u + α·h(x) ≥ 0   （CBF条件）
  u_min ≤ u ≤ u_max
```

元の制御入力からの変化を最小化しながら、安全性を保つ最小限の修正を求める。

---

## Python実装

```python
import numpy as np
from scipy.optimize import minimize
from dataclasses import dataclass

@dataclass
class Obstacle:
    x: float
    y: float
    radius: float

class CBFController:
    def __init__(self, safety_margin: float = 0.5, alpha: float = 1.0):
        self.safety_margin = safety_margin
        self.alpha = alpha

    def barrier_function(self, robot_pos, obs):
        dx = robot_pos[0] - obs.x
        dy = robot_pos[1] - obs.y
        dist_sq = dx**2 + dy**2
        threshold = (obs.radius + self.safety_margin)**2
        return dist_sq - threshold

    def barrier_gradient(self, robot_pos, obs):
        return 2 * np.array([robot_pos[0] - obs.x, robot_pos[1] - obs.y])

    def safe_control(self, nominal_u, robot_pos, obstacles, u_max=0.5):
        def objective(u):
            return np.linalg.norm(u - nominal_u)**2

        constraints = []
        for obs in obstacles:
            h = self.barrier_function(robot_pos, obs)
            grad_h = self.barrier_gradient(robot_pos, obs)

            def cbf_constraint(u, grad_h=grad_h, h=h):
                return float(grad_h @ u + self.alpha * h)

            constraints.append({'type': 'ineq', 'fun': cbf_constraint})

        result = minimize(
            objective, nominal_u, method='SLSQP',
            bounds=[(-u_max, u_max), (-u_max, u_max)],
            constraints=constraints
        )

        return result.x if result.success else np.zeros(2)
```

---

## ROS2への統合

```python
class CBFRobotNode(Node):
    def __init__(self):
        super().__init__('cbf_robot')
        self.cbf = CBFController(safety_margin=0.5, alpha=2.0)
        self.robot_pos = np.zeros(2)
        self.obstacles = []
        self.goal = np.array([2.0, 0.0])

        self.cmd_pub = self.create_publisher(Twist, 'cmd_vel', 10)
        self.create_subscription(Odometry, 'odom', self.odom_cb, 10)
        self.create_subscription(LaserScan, 'scan', self.scan_cb, 10)
        self.create_timer(0.1, self.control_loop)

    def scan_cb(self, msg):
        self.obstacles = []
        angle = msg.angle_min
        for r in msg.ranges:
            if msg.range_min < r < msg.range_max:
                ox = self.robot_pos[0] + r * np.cos(angle)
                oy = self.robot_pos[1] + r * np.sin(angle)
                self.obstacles.append(Obstacle(ox, oy, 0.1))
            angle += msg.angle_increment

    def control_loop(self):
        error = self.goal - self.robot_pos
        nominal_u = 0.5 * error / (np.linalg.norm(error) + 1e-6)
        safe_u = self.cbf.safe_control(nominal_u, self.robot_pos, self.obstacles)

        cmd = Twist()
        cmd.linear.x = float(safe_u[0])
        cmd.linear.y = float(safe_u[1])
        self.cmd_pub.publish(cmd)
```

---

## LLMとCBFの二層構造

研究で実装している構成：

```
LLM（高レベル計画）      → 0.1Hz、ウェイポイント生成
        ↓
CBFコントローラー（低レベル安全保証） → 10〜50Hz、衝突防止
```

LLMは賢いが遅い。CBFは単純だが速くて安全。この組み合わせがうまく機能しています。
