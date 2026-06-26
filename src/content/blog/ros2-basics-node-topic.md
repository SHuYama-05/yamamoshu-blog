---
title: 'ROS2の基本概念をわかりやすく解説【ノード・トピック・サービス・アクション】'
description: 'ROS2を始める人が最初に躓くノード・トピック・サービス・アクションの概念を、実例コード付きでわかりやすく解説します。ROS1との違いも説明。'
pubDate: '2026-06-20'
tags: ['ROS2', '入門', 'Python']
---

ROS2を学び始めたとき、最初に混乱するのが「ノード」「トピック」「サービス」「アクション」の違いです。この記事では実際に動くPythonコードを使って丁寧に説明します。

## ノード（Node）

ノードはROS2の最小実行単位です。1つのプロセス＝1つのノードと考えると分かりやすいです。

たとえばカメラを制御するノード、画像処理をするノード、ロボットを動かすノードのように、機能ごとに分けて実装します。

```python
import rclpy
from rclpy.node import Node

class MyNode(Node):
    def __init__(self):
        super().__init__('my_node')  # ノード名（ros2 node listで表示される名前）
        self.get_logger().info('ノード起動！')
        # タイマー：1秒ごとにcallbackを呼ぶ
        self.timer = self.create_timer(1.0, self.timer_callback)

    def timer_callback(self):
        self.get_logger().info('1秒経過')

def main():
    rclpy.init()
    node = MyNode()
    rclpy.spin(node)   # ノードを動かし続ける
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

```bash
# 別ターミナルでノードを確認
ros2 node list
# /my_node
```

---

## トピック（Topic）

トピックは**一方向の非同期通信**です。センサーデータの送受信によく使います。

```
パブリッシャー → [トピック名] → サブスクライバー
（送信側）                        （受信側）
```

### パブリッシャー（送信側）

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String

class TalkerNode(Node):
    def __init__(self):
        super().__init__('talker')
        # トピック名: 'chatter', メッセージ型: String, キューサイズ: 10
        self.publisher = self.create_publisher(String, 'chatter', 10)
        self.timer = self.create_timer(0.5, self.publish_message)
        self.count = 0

    def publish_message(self):
        msg = String()
        msg.data = f'Hello ROS2! count={self.count}'
        self.publisher.publish(msg)
        self.get_logger().info(f'送信: {msg.data}')
        self.count += 1
```

### サブスクライバー（受信側）

```python
class ListenerNode(Node):
    def __init__(self):
        super().__init__('listener')
        self.subscription = self.create_subscription(
            String,            # メッセージ型
            'chatter',         # トピック名（パブリッシャーと一致させる）
            self.on_message,   # コールバック関数
            10                 # キューサイズ
        )

    def on_message(self, msg: String):
        self.get_logger().info(f'受信: {msg.data}')
```

```bash
# トピックの確認
ros2 topic list
ros2 topic echo /chatter        # 流れているデータを表示
ros2 topic hz /chatter          # 周波数を確認
ros2 topic info /chatter        # パブリッシャー・サブスクライバー数
```

よく使うメッセージ型：

| パッケージ | 型 | 用途 |
|-----------|-----|------|
| `std_msgs` | `String`, `Float64`, `Bool` | 基本型 |
| `geometry_msgs` | `Twist`, `Pose`, `PoseStamped` | 位置・速度 |
| `sensor_msgs` | `LaserScan`, `Image`, `Imu` | センサー |
| `nav_msgs` | `OccupancyGrid`, `Path` | ナビゲーション |

---

## サービス（Service）

サービスは**双方向の同期通信**です。「リクエストを送ったらレスポンスが返ってくる」形式です。

```
クライアント → [リクエスト] → サーバー
            ← [レスポンス] ←
```

一時的な処理（パラメーター変更・ファイル保存・計算）に使います。

### サービスサーバー

```python
from rclpy.node import Node
from example_interfaces.srv import AddTwoInts

class AddTwoIntsServer(Node):
    def __init__(self):
        super().__init__('add_two_ints_server')
        self.srv = self.create_service(
            AddTwoInts,      # サービス型
            'add_two_ints',  # サービス名
            self.handle_request
        )
        self.get_logger().info('サービス準備完了')

    def handle_request(self, request, response):
        response.sum = request.a + request.b
        self.get_logger().info(f'{request.a} + {request.b} = {response.sum}')
        return response
```

### サービスクライアント

```python
from example_interfaces.srv import AddTwoInts

class AddTwoIntsClient(Node):
    def __init__(self):
        super().__init__('add_two_ints_client')
        self.client = self.create_client(AddTwoInts, 'add_two_ints')
        # サービスが起動するまで待機
        while not self.client.wait_for_service(timeout_sec=1.0):
            self.get_logger().info('サービス待機中...')

    def send_request(self, a: int, b: int):
        req = AddTwoInts.Request()
        req.a = a
        req.b = b
        future = self.client.call_async(req)
        rclpy.spin_until_future_complete(self, future)
        return future.result()

# 使い方
client_node = AddTwoIntsClient()
result = client_node.send_request(3, 5)
print(f'結果: {result.sum}')  # 8
```

```bash
# コマンドラインからサービスを呼ぶ
ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a: 3, b: 5}"
```

---

## アクション（Action）

アクションは**長時間かかる処理**に使います。途中でフィードバックを受け取ったり、キャンセルしたりできます。

```
クライアント → [ゴール送信] → サーバー
            ← [フィードバック] ←  （処理中に何度も）
            ← [結果] ←            （完了時に1回）
```

ナビゲーション（「目的地まで移動して」）や、アームの動作などに使われます。

```python
from rclpy.action import ActionServer
from action_tutorials_interfaces.action import Fibonacci

class FibonacciActionServer(Node):
    def __init__(self):
        super().__init__('fibonacci_server')
        self._action_server = ActionServer(
            self, Fibonacci, 'fibonacci', self.execute
        )

    async def execute(self, goal_handle):
        self.get_logger().info(f'フィボナッチ数列を{goal_handle.request.order}個計算します')
        feedback_msg = Fibonacci.Feedback()
        feedback_msg.partial_sequence = [0, 1]

        for i in range(1, goal_handle.request.order):
            feedback_msg.partial_sequence.append(
                feedback_msg.partial_sequence[i] + feedback_msg.partial_sequence[i-1]
            )
            goal_handle.publish_feedback(feedback_msg)  # 途中経過を送信
            await asyncio.sleep(0.5)

        goal_handle.succeed()
        result = Fibonacci.Result()
        result.sequence = feedback_msg.partial_sequence
        return result
```

---

## まとめ：どれを使うべきか

| 通信方式 | 方向 | 同期 | キャンセル | 典型的な用途 |
|---------|------|------|-----------|------------|
| トピック | 一方向 | 非同期 | ✗ | センサーデータ・速度指令 |
| サービス | 双方向 | 同期 | ✗ | 設定変更・単純な計算 |
| アクション | 双方向 | 非同期 | ✅ | ナビゲーション・長時間処理 |

迷ったら「センサーデータ → トピック」「一回限りの処理 → サービス」「長い処理 → アクション」で判断するとシンプルです。

次回はROS2 + Nav2を使った自律ナビゲーションの実装を解説します。
