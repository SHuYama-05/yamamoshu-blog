---
title: "ROS2の基本概念をPythonコードで理解する【Node・Topic・Service・Action】"
emoji: "📡"
type: "tech"
topics: ["ROS2", "Python", "ロボット", "ROS"]
published: true
---

ROS2の「ノード」「トピック」「サービス」「アクション」は、ロボットソフトウェアの基本的な通信パターンです。それぞれどう違うのか、実際のコードで確認します。

---

## Node（ノード）

ROS2の実行単位。1プロセス = 1ノードが基本です。

```python
import rclpy
from rclpy.node import Node

class MyNode(Node):
    def __init__(self):
        super().__init__('my_node')  # ノード名
        self.get_logger().info('起動しました')

def main():
    rclpy.init()
    node = MyNode()
    rclpy.spin(node)  # コールバックを待ち続ける
    node.destroy_node()
    rclpy.shutdown()
```

---

## Topic（トピック）

非同期の1対多通信。センサーデータの配信などに使います。

### パブリッシャー（送信側）

```python
from std_msgs.msg import String
from rclpy.node import Node
import rclpy

class TalkerNode(Node):
    def __init__(self):
        super().__init__('talker')
        self.pub = self.create_publisher(String, 'chatter', 10)
        self.timer = self.create_timer(1.0, self.publish)

    def publish(self):
        msg = String()
        msg.data = f'Hello at {self.get_clock().now()}'
        self.pub.publish(msg)
        self.get_logger().info(f'送信: {msg.data}')
```

### サブスクライバー（受信側）

```python
class ListenerNode(Node):
    def __init__(self):
        super().__init__('listener')
        self.sub = self.create_subscription(
            String, 'chatter', self.callback, 10
        )

    def callback(self, msg: String):
        self.get_logger().info(f'受信: {msg.data}')
```

---

## Service（サービス）

同期的なリクエスト・レスポンス通信。

```python
from std_srvs.srv import SetBool

class ServiceServer(Node):
    def __init__(self):
        super().__init__('service_server')
        self.srv = self.create_service(
            SetBool, 'toggle', self.handle
        )

    def handle(self, request, response):
        self.get_logger().info(f'リクエスト: {request.data}')
        response.success = True
        response.message = 'OK'
        return response
```

```python
# クライアント側
class ServiceClient(Node):
    def __init__(self):
        super().__init__('service_client')
        self.client = self.create_client(SetBool, 'toggle')
        self.client.wait_for_service()

    def call(self, value: bool):
        req = SetBool.Request()
        req.data = value
        future = self.client.call_async(req)
        rclpy.spin_until_future_complete(self, future)
        return future.result()
```

---

## Action（アクション）

長時間処理 + フィードバック付きの非同期通信。ナビゲーションなどに使います。

```python
from rclpy.action import ActionServer
from nav2_msgs.action import NavigateToPose

class NavActionServer(Node):
    def __init__(self):
        super().__init__('nav_server')
        self._action_server = ActionServer(
            self,
            NavigateToPose,
            'navigate_to_pose',
            self.execute
        )

    async def execute(self, goal_handle):
        self.get_logger().info('ナビゲーション開始')

        feedback = NavigateToPose.Feedback()
        for i in range(10):
            feedback.current_pose.pose.position.x = float(i) * 0.1
            goal_handle.publish_feedback(feedback)
            await asyncio.sleep(0.5)

        goal_handle.succeed()
        result = NavigateToPose.Result()
        return result
```

---

## 使い分けまとめ

| 通信方式 | 向き | 用途 |
|---------|------|------|
| Topic | 1対多（非同期） | センサーデータ、cmd_vel |
| Service | 1対1（同期） | 設定変更、短い処理 |
| Action | 1対1（非同期+フィードバック） | ナビゲーション、長い処理 |
