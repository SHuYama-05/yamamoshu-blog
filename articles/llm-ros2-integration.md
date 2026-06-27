---
title: "LLM（Claude API）をROS2に統合してロボットを自然言語で制御する"
emoji: "🧠"
type: "tech"
topics: ["ROS2", "LLM", "Claude", "Python", "ロボット"]
published: true
---

「前に進んで」「左に曲がって止まれ」といった自然言語でロボットを操作する。Claude APIのFunction Callingを使うと、これが意外と簡単に実装できます。

---

## アーキテクチャ

```
ユーザー発話
    ↓
Claude API（Function Calling）
    ↓ move_robot(linear_x, angular_z)
ROS2 Node
    ↓ /cmd_vel
ロボット
```

---

## Function Calling の定義

```python
TOOLS = [
    {
        "name": "move_robot",
        "description": "ロボットを移動させる",
        "input_schema": {
            "type": "object",
            "properties": {
                "linear_x": {
                    "type": "number",
                    "description": "前後速度 m/s（正=前進、負=後退）"
                },
                "angular_z": {
                    "type": "number",
                    "description": "回転速度 rad/s（正=左回転）"
                },
                "duration": {
                    "type": "number",
                    "description": "動作時間（秒）"
                }
            },
            "required": ["linear_x", "angular_z"]
        }
    },
    {
        "name": "stop_robot",
        "description": "ロボットを停止させる",
        "input_schema": {
            "type": "object",
            "properties": {}
        }
    }
]
```

---

## ROS2ノード実装

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
import anthropic
import json

class LLMRobotController(Node):
    def __init__(self):
        super().__init__('llm_robot_controller')
        self.cmd_pub = self.create_publisher(Twist, 'cmd_vel', 10)
        self.client = anthropic.Anthropic()
        self.get_logger().info('LLMロボットコントローラー起動')

    def process_command(self, user_input: str):
        """自然言語コマンドをロボット動作に変換"""
        response = self.client.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=256,
            tools=TOOLS,
            messages=[{
                "role": "user",
                "content": f"ロボットに指示: {user_input}"
            }]
        )

        for block in response.content:
            if block.type == "tool_use":
                if block.name == "move_robot":
                    self.move(**block.input)
                elif block.name == "stop_robot":
                    self.stop()

    def move(self, linear_x: float, angular_z: float, duration: float = 1.0):
        cmd = Twist()
        cmd.linear.x = float(linear_x)
        cmd.angular.z = float(angular_z)
        self.cmd_pub.publish(cmd)
        self.get_logger().info(f'移動: vx={linear_x}, wz={angular_z}')

        # duration後に停止
        self.create_timer(duration, lambda: self.stop())

    def stop(self):
        self.cmd_pub.publish(Twist())
        self.get_logger().info('停止')


def main():
    rclpy.init()
    node = LLMRobotController()

    # インタラクティブループ
    import threading
    def input_loop():
        while True:
            cmd = input('コマンド: ')
            node.process_command(cmd)

    thread = threading.Thread(target=input_loop, daemon=True)
    thread.start()

    rclpy.spin(node)
    rclpy.shutdown()
```

---

## 実行例

```bash
ros2 run my_robot llm_controller

コマンド: 1メートル前進して
→ move_robot(linear_x=0.2, angular_z=0.0, duration=5.0)

コマンド: 右に90度回転
→ move_robot(linear_x=0.0, angular_z=-1.57, duration=1.0)

コマンド: 止まれ
→ stop_robot()
```

---

## ポイントと注意点

**うまくいった点**
- `claude-haiku` で十分な精度。応答が速くてコスト低い
- Function Callingで余計なパース処理が不要

**注意点**
- LLM呼び出しは1〜3秒かかる → コールバックをブロックしないよう別スレッドで実行
- 安全のため最大速度を制限する
- タイムアウト設定必須（APIが返ってこない場合にロボットが止まらない）
