---
title: '制御バリア関数（CBF）でロボットの安全な自律移動を実現する'
description: '制御バリア関数（Control Barrier Function, CBF）の基本概念と、ROS2を使ったロボット障害物回避への応用方法を解説します。'
pubDate: '2026-06-05'
tags: ['ROS2', 'CBF', '制御理論', '安全', 'Python']
---

自律移動ロボットが人や障害物に衝突しないようにするための制御理論「制御バリア関数（CBF）」を解説します。

## CBFとは

CBF（Control Barrier Function）は、システムが「安全集合」の外に出ないことを数学的に保証する制御手法です。

```
安全集合 C = {x | h(x) ≥ 0}
     例: ロボットと障害物の距離が0.5m以上の状態
```

## 直感的な理解

通常の制御：「目標に向かって進め」
CBF付き制御：「目標に向かって進め。ただし障害物に近づきすぎたら自動でよける」

障害物に近づくほど「安全のための力」が強くなり、衝突を防ぎます。

## Pythonでの実装（2Dロボット）

```python
import numpy as np
from scipy.optimize import minimize

class CBFController:
    def __init__(self, safety_radius=0.5, alpha=1.0):
        self.safety_radius = safety_radius
        self.alpha = alpha  # CBFのゲイン

    def barrier_function(self, robot_pos, obstacle_pos):
        dist = np.linalg.norm(robot_pos - obstacle_pos)
        return dist**2 - self.safety_radius**2

    def safe_control(self, nominal_u, robot_pos, robot_vel, obstacles):
        """
        nominal_u: 目標達成のための通常制御入力
        robot_pos: ロボット位置 [x, y]
        obstacles: 障害物位置リスト [[x1,y1], [x2,y2], ...]
        """
        def objective(u):
            return np.linalg.norm(u - nominal_u)**2

        constraints = []
        for obs in obstacles:
            obs = np.array(obs)
            h = self.barrier_function(robot_pos, obs)
            # grad_h · f(x,u) ≥ -alpha * h(x)
            grad_h = 2 * (robot_pos - obs)

            def cbf_constraint(u, grad_h=grad_h, h=h):
                return grad_h @ u + self.alpha * h

            constraints.append({'type': 'ineq', 'fun': cbf_constraint})

        result = minimize(
            objective,
            nominal_u,
            constraints=constraints,
            method='SLSQP'
        )
        return result.x

# 使用例
controller = CBFController(safety_radius=0.5)
robot_pos = np.array([0.0, 0.0])
robot_vel = np.array([0.5, 0.0])
obstacles = [[1.0, 0.1], [2.0, -0.5]]

nominal_u = np.array([0.5, 0.0])  # 前進したい
safe_u = controller.safe_control(nominal_u, robot_pos, robot_vel, obstacles)
print(f"通常入力: {nominal_u}")
print(f"安全入力: {safe_u}")
```

## LLMとCBFの組み合わせ

私の研究では、LLMが高レベルの行動計画を立て、CBFがその計画を安全に実行する二層構造を採用しています。

```
LLM（ChatGPT/Claude）
  └── 「人を避けながら目的地に行って」を解析
        ↓ 高レベル経路
CBF Controller
  └── 数学的に安全性を保証しながら実行
```

CBFは計算コストが低く、リアルタイム制御に向いているため、LLMの遅い応答を補完するのに適しています。
