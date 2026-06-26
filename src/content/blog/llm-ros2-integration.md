---
title: 'LLMをROS2に統合してロボットを自然言語で制御する'
description: 'ChatGPT/Claude APIをROS2と組み合わせて、自然言語でロボットを操作する仕組みを実装します。LangChainを使った実装例も紹介。'
pubDate: '2026-06-12'
tags: ['ROS2', 'LLM', 'ChatGPT', 'Python', 'LangChain']
---

「右に1メートル進んで」という指示でロボットを動かせたら面白いですよね。LLMとROS2を組み合わせることで実現できます。

## 仕組み

```
ユーザーの自然言語指示
    ↓
LLM（GPT/Claude）が解析
    ↓
ROS2コマンドに変換
    ↓
ロボット実行
```

## 必要なもの

```bash
pip install openai langchain
pip install rclpy  # ROS2環境内で
```

## 基本実装

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
from openai import OpenAI

client = OpenAI()

class LLMRobotNode(Node):
    def __init__(self):
        super().__init__('llm_robot')
        self.cmd_pub = self.create_publisher(Twist, 'cmd_vel', 10)

    def parse_command(self, user_input: str) -> dict:
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {
                    "role": "system",
                    "content": """ロボット制御コマンドに変換してください。
                    JSON形式で返答: {"linear_x": float, "angular_z": float, "duration": float}
                    linear_x: 前進速度(m/s), angular_z: 回転速度(rad/s), duration: 実行時間(秒)"""
                },
                {"role": "user", "content": user_input}
            ],
            response_format={"type": "json_object"}
        )
        import json
        return json.loads(response.choices[0].message.content)

    def execute_command(self, cmd: dict):
        twist = Twist()
        twist.linear.x = cmd["linear_x"]
        twist.angular.z = cmd["angular_z"]

        end_time = self.get_clock().now().nanoseconds + cmd["duration"] * 1e9
        while self.get_clock().now().nanoseconds < end_time:
            self.cmd_pub.publish(twist)

        # 停止
        self.cmd_pub.publish(Twist())
```

## 使い方

```python
def main():
    rclpy.init()
    node = LLMRobotNode()

    while True:
        user_input = input("指示を入力: ")
        cmd = node.parse_command(user_input)
        print(f"実行: {cmd}")
        node.execute_command(cmd)
```

実行例：
```
指示を入力: 右に90度回転して
実行: {"linear_x": 0.0, "angular_z": -0.5, "duration": 3.14}
```

## 注意点

- LLMの出力は確定的ではないので、コマンドのバリデーションが必要
- 安全のため最大速度・最大時間の上限を設ける
- 実機では必ず緊急停止機能を用意する

次回はControl Barrier Function（CBF）を使った安全な自律移動を解説します。
