---
title: 'LLMをROS2に統合してロボットを自然言語で制御する【実装例あり】'
description: 'Claude/GPT APIをROS2と組み合わせて、自然言語でロボットを操作する仕組みを実装します。ツール呼び出し（Function Calling）を使った安全な実装例を紹介。'
pubDate: '2026-06-12'
tags: ['ROS2', 'LLM', 'Claude', 'Python']
---

「右に1メートル進んで」「障害物を避けながら部屋の反対側に行って」という自然言語でロボットを動かせたら便利ですよね。LLMとROS2を組み合わせることで実現できます。

研究でこの仕組みを実装したので、実際のコードと詰まったポイントを共有します。

## 全体の仕組み

```
ユーザーの自然言語指示
        ↓
  LLM（APIを呼ぶ）
  └── Function Callingで
      コマンドを構造化
        ↓
  ROS2ノードが受け取る
        ↓
  速度指令をパブリッシュ
        ↓
  ロボットが実行
```

なぜFunction Calling（ツール呼び出し）を使うかというと、LLMに自由にテキストを出力させると、パースがブレて制御が不安定になるからです。ツール呼び出しを使えば常に決まった構造のJSONが返ってきます。

---

## 実装

### 必要なもの

```bash
pip install anthropic  # または openai
```

### ツール定義（Function Calling）

```python
ROBOT_TOOLS = [
    {
        "name": "move_robot",
        "description": "ロボットを指定した速度・時間で移動させる",
        "input_schema": {
            "type": "object",
            "properties": {
                "linear_x": {
                    "type": "number",
                    "description": "前後方向の速度 (m/s)。正が前進、負が後退。最大0.5"
                },
                "angular_z": {
                    "type": "number",
                    "description": "回転速度 (rad/s)。正が左回転、負が右回転。最大1.0"
                },
                "duration": {
                    "type": "number",
                    "description": "実行時間 (秒)。最大10秒"
                },
                "reason": {
                    "type": "string",
                    "description": "この動作を選んだ理由の説明"
                }
            },
            "required": ["linear_x", "angular_z", "duration", "reason"]
        }
    },
    {
        "name": "stop_robot",
        "description": "ロボットを即座に停止させる"
    }
]
```

### ROS2ノード本体

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
import anthropic
import time

# 安全のための上限値
MAX_LINEAR = 0.5   # m/s
MAX_ANGULAR = 1.0  # rad/s
MAX_DURATION = 10.0  # 秒

class LLMRobotNode(Node):
    def __init__(self):
        super().__init__('llm_robot')
        self.cmd_pub = self.create_publisher(Twist, 'cmd_vel', 10)
        self.client = anthropic.Anthropic()
        self.get_logger().info('LLMロボットノード起動')

    def parse_command(self, user_input: str) -> dict | None:
        """LLMで自然言語をロボットコマンドに変換"""
        try:
            response = self.client.messages.create(
                model="claude-haiku-4-5-20251001",  # 速度重視で軽量モデル
                max_tokens=256,
                system="あなたはロボット制御AIです。ユーザーの指示をロボットコマンドに変換してください。安全を最優先にし、危険な動作は拒否してください。",
                messages=[{"role": "user", "content": user_input}],
                tools=ROBOT_TOOLS,
                tool_choice={"type": "any"}
            )

            # ツール呼び出しを取得
            for block in response.content:
                if block.type == "tool_use":
                    return {"tool": block.name, "input": block.input}
            return None

        except Exception as e:
            self.get_logger().error(f'LLM API エラー: {e}')
            return None

    def execute_command(self, cmd: dict):
        """コマンドを実行"""
        if cmd["tool"] == "stop_robot":
            self.stop()
            return

        if cmd["tool"] == "move_robot":
            inp = cmd["input"]
            self.get_logger().info(f'実行理由: {inp.get("reason", "不明")}')

            # 安全上限でクリップ
            linear_x = max(-MAX_LINEAR, min(MAX_LINEAR, inp["linear_x"]))
            angular_z = max(-MAX_ANGULAR, min(MAX_ANGULAR, inp["angular_z"]))
            duration = min(MAX_DURATION, inp["duration"])

            self.move(linear_x, angular_z, duration)

    def move(self, linear_x: float, angular_z: float, duration: float):
        twist = Twist()
        twist.linear.x = linear_x
        twist.angular.z = angular_z

        start = time.time()
        rate = self.create_rate(10)  # 10Hz
        while time.time() - start < duration:
            self.cmd_pub.publish(twist)
            rate.sleep()

        self.stop()

    def stop(self):
        self.cmd_pub.publish(Twist())
        self.get_logger().info('停止')


def main():
    rclpy.init()
    node = LLMRobotNode()

    print('ROS2 LLMロボット起動。指示を入力してください（qで終了）')
    while rclpy.ok():
        user_input = input('\n指示: ').strip()
        if user_input.lower() == 'q':
            break
        if not user_input:
            continue

        print('LLMで解析中...')
        cmd = node.parse_command(user_input)

        if cmd is None:
            print('コマンドを解析できませんでした')
            continue

        print(f'実行: {cmd}')
        node.execute_command(cmd)

    node.destroy_node()
    rclpy.shutdown()
```

---

## 実際の動作例

```
指示: 前に1メートル進んで

LLMで解析中...
実行: {'tool': 'move_robot', 'input': {'linear_x': 0.3, 'angular_z': 0.0, 'duration': 3.3, 'reason': '1m ÷ 0.3m/s ≈ 3.3秒前進'}}
実行理由: 1m ÷ 0.3m/s ≈ 3.3秒前進
停止

指示: 右に90度回転して

実行: {'tool': 'move_robot', 'input': {'linear_x': 0.0, 'angular_z': -0.5, 'duration': 3.14, 'reason': '90度 = π/2 rad, π/2 ÷ 0.5 rad/s ≈ 3.14秒右回転'}}
```

LLMが自分で計算式を考えて時間を算出しているのが面白いところです。

---

## 詰まったポイントと解決策

### 1. LLMの応答が遅くてリアルタイム性が損なわれる

Claude Haiku（軽量モデル）を使うことで応答時間を0.5〜1秒程度に短縮できます。Opus/Sonnetは賢いですが遅いので制御には不向きです。

### 2. 距離・角度の計算が不正確

LLMは「1メートル前進 = 0.3m/sで3.3秒」という計算を自分でしてくれますが、ロボットの実際の速度特性とズレることがあります。キャリブレーションして実測値でプロンプトに補正値を入れると精度が上がります。

### 3. 安全性

上限値のクリッピングは必須です。LLMが「linear_x: 10.0」と出力してもコードで弾きます。実機では**ハードウェアの緊急停止ボタン**も必ず用意してください。

---

## 次のステップ

このままだとLLMは「現在のロボットの状態」を知りません。センサーデータ（LiDAR・カメラ）をコンテキストとして渡すことで、障害物を考慮した判断ができるようになります。次回その実装を解説します。
