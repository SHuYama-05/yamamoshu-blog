---
title: 'ROS2の基本概念をわかりやすく解説【ノード・トピック・サービス】'
description: 'ROS2を始める人が最初に躓くノード・トピック・サービス・アクションの概念を、実例コード付きでわかりやすく解説します。'
pubDate: '2026-06-20'
tags: ['ROS2', '入門', 'Python']
---

ROS2を学び始めたとき、最初に混乱するのが「ノード」「トピック」「サービス」の違いです。この記事では実際に動くコードを使って説明します。

## ノード（Node）

ノードはROS2の最小実行単位です。1つのプログラム＝1つのノードと考えると分かりやすいです。

```python
import rclpy
from rclpy.node import Node

class MyNode(Node):
    def __init__(self):
        super().__init__('my_node')  # ノード名
        self.get_logger().info('ノード起動！')

def main():
    rclpy.init()
    node = MyNode()
    rclpy.spin(node)
    rclpy.shutdown()
```

## トピック（Topic）

トピックは**一方向の非同期通信**です。センサーデータの送受信に使います。

```python
# パブリッシャー（送信側）
from std_msgs.msg import String

class Publisher(Node):
    def __init__(self):
        super().__init__('publisher')
        self.pub = self.create_publisher(String, 'chatter', 10)
        self.timer = self.create_timer(1.0, self.callback)

    def callback(self):
        msg = String()
        msg.data = 'Hello ROS2!'
        self.pub.publish(msg)
```

```python
# サブスクライバー（受信側）
class Subscriber(Node):
    def __init__(self):
        super().__init__('subscriber')
        self.sub = self.create_subscription(
            String, 'chatter', self.callback, 10
        )

    def callback(self, msg):
        self.get_logger().info(f'受信: {msg.data}')
```

## サービス（Service）

サービスは**双方向の同期通信**です。リクエスト→レスポンスの形で使います。

```python
from example_interfaces.srv import AddTwoInts

class AddServer(Node):
    def __init__(self):
        super().__init__('add_server')
        self.srv = self.create_service(
            AddTwoInts, 'add', self.callback
        )

    def callback(self, request, response):
        response.sum = request.a + request.b
        return response
```

## まとめ

| 通信方式 | 方向 | 同期 | 用途 |
|---------|------|------|------|
| トピック | 一方向 | 非同期 | センサーデータ |
| サービス | 双方向 | 同期 | 一時的な処理 |
| アクション | 双方向 | 非同期 | 長時間の処理 |

次回はNav2を使ったナビゲーションについて解説します。
