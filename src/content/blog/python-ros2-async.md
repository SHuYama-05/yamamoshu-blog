---
title: 'ROS2とPython asyncioを組み合わせた非同期処理【LLM API統合の実装例】'
description: 'ROS2のコールバックベースの処理とPythonのasyncioを組み合わせて、LLM APIなど低速な非同期処理を効率的に実装する方法を解説します。スレッドセーフな実装例付き。'
pubDate: '2026-06-01'
tags: ['ROS2', 'Python', 'asyncio', '非同期処理', 'LLM']
---

ROS2のLLM統合を実装しているとき、一番困ったのが「LLM APIの呼び出し（非同期・低速）」と「ロボット制御（同期・高速）」の組み合わせです。

素直に実装しようとするとこんなエラーや問題が出ます。

```python
# NG: コールバック内でAPI呼び出しをブロックすると制御が止まる
def scan_callback(self, msg):
    response = requests.post(...)  # ここでブロック → cmd_velが止まる！
    self.publish_command(response)
```

この記事では、スレッドセーフにasyncioとROS2を組み合わせる方法を実装例付きで解説します。

---

## 問題の整理

ROS2の `spin()` はイベントループを自分で管理していて、Pythonの `asyncio` イベントループとは別物です。

```
ROS2 spin()
├── コールバック1（10Hz）
├── コールバック2（30Hz）
└── タイマー（1Hz）

asyncio event loop（別スレッド）
├── async関数1
└── async関数2
```

これらを直接混ぜることはできませんが、`asyncio.run_coroutine_threadsafe()` でスレッド間通信ができます。

---

## 基本パターン：asyncioループを別スレッドで動かす

```python
import asyncio
import threading
import rclpy
from rclpy.node import Node
from rclpy.executors import MultiThreadedExecutor

class AsyncROS2Node(Node):
    def __init__(self):
        super().__init__('async_node')

        # asyncioループを別スレッドで起動
        self._loop = asyncio.new_event_loop()
        self._thread = threading.Thread(
            target=self._run_loop,
            daemon=True,
            name='asyncio_thread'
        )
        self._thread.start()

    def _run_loop(self):
        asyncio.set_event_loop(self._loop)
        self._loop.run_forever()

    def run_async(self, coro) -> asyncio.Future:
        """コルーチンをasyncioスレッドで実行"""
        return asyncio.run_coroutine_threadsafe(coro, self._loop)

    def destroy_node(self):
        self._loop.call_soon_threadsafe(self._loop.stop)
        self._thread.join(timeout=5.0)
        super().destroy_node()
```

---

## 実装例：LLM APIをROS2コールバックから呼ぶ

```python
import anthropic
from std_msgs.msg import String
from geometry_msgs.msg import Twist
from concurrent.futures import Future

class LLMAsyncNode(AsyncROS2Node):
    def __init__(self):
        super().__init__('llm_async_node')

        self.client = anthropic.AsyncAnthropic()
        self.cmd_pub = self.create_publisher(Twist, 'cmd_vel', 10)
        self.text_sub = self.create_subscription(
            String, 'user_command', self.on_command, 10
        )

        self._pending_future: Future | None = None
        self.get_logger().info('LLM非同期ノード起動')

    def on_command(self, msg: String):
        """ROS2コールバック（同期）"""
        # すでに処理中の場合はスキップ
        if self._pending_future and not self._pending_future.done():
            self.get_logger().warn('前のリクエスト処理中。スキップ')
            return

        self.get_logger().info(f'コマンド受信: {msg.data}')
        # asyncioスレッドで非同期処理を開始
        self._pending_future = self.run_async(
            self.process_command(msg.data)
        )
        # 完了時のコールバックを登録
        self._pending_future.add_done_callback(self.on_llm_done)

    async def process_command(self, text: str) -> dict | None:
        """非同期でLLM APIを呼ぶ"""
        try:
            response = await asyncio.wait_for(
                self.client.messages.create(
                    model="claude-haiku-4-5-20251001",
                    max_tokens=128,
                    messages=[{
                        "role": "user",
                        "content": f"ロボットコマンドをJSONで返してください: {text}"
                    }]
                ),
                timeout=5.0  # 5秒でタイムアウト
            )
            import json
            return json.loads(response.content[0].text)

        except asyncio.TimeoutError:
            self.get_logger().warn('LLM APIタイムアウト')
            return None
        except Exception as e:
            self.get_logger().error(f'LLM APIエラー: {e}')
            return None

    def on_llm_done(self, future: Future):
        """LLM処理完了後のコールバック（asyncioスレッドから呼ばれる）"""
        try:
            result = future.result()
        except Exception as e:
            self.get_logger().error(f'処理エラー: {e}')
            return

        if result is None:
            return

        # ROSのパブリッシュはスレッドセーフではないので注意
        # create_timer で次のspinサイクルに実行する
        self._queued_result = result
        self.create_timer(0.0, self._publish_result)  # 即時実行

    def _publish_result(self):
        if not hasattr(self, '_queued_result'):
            return
        result = self._queued_result
        del self._queued_result

        cmd = Twist()
        cmd.linear.x = float(result.get('linear_x', 0.0))
        cmd.angular.z = float(result.get('angular_z', 0.0))
        self.cmd_pub.publish(cmd)
        self.get_logger().info(f'コマンド実行: {result}')
```

---

## より簡単な方法：MultiThreadedExecutor + concurrent.futures

asyncioを使わず、`concurrent.futures.ThreadPoolExecutor` でシンプルに書く方法もあります。

```python
from concurrent.futures import ThreadPoolExecutor
import anthropic

class ThreadedLLMNode(Node):
    def __init__(self):
        super().__init__('threaded_llm_node')
        self.executor_pool = ThreadPoolExecutor(max_workers=2)
        self.client = anthropic.Anthropic()  # 同期クライアント
        self.cmd_pub = self.create_publisher(Twist, 'cmd_vel', 10)
        self.sub = self.create_subscription(
            String, 'user_command', self.on_command, 10
        )
        self._processing = False

    def on_command(self, msg: String):
        if self._processing:
            return
        self._processing = True
        # スレッドプールで実行
        future = self.executor_pool.submit(self.call_llm, msg.data)
        future.add_done_callback(self.on_done)

    def call_llm(self, text: str) -> dict | None:
        """別スレッドで同期的にAPIを呼ぶ"""
        try:
            response = self.client.messages.create(
                model="claude-haiku-4-5-20251001",
                max_tokens=128,
                messages=[{"role": "user", "content": text}],
                timeout=5.0
            )
            import json
            return json.loads(response.content[0].text)
        except Exception as e:
            self.get_logger().error(f'エラー: {e}')
            return None

    def on_done(self, future):
        self._processing = False
        result = future.result()
        if result:
            # ROSのパブリッシュは ROS spinスレッドから呼ぶ方が安全
            # ここでは簡単のため直接呼ぶ
            cmd = Twist()
            cmd.linear.x = float(result.get('linear_x', 0.0))
            self.cmd_pub.publish(cmd)


def main():
    rclpy.init()
    node = ThreadedLLMNode()
    # MultiThreadedExecutorを使う（コールバックの並列実行を許可）
    executor = MultiThreadedExecutor(num_threads=4)
    executor.add_node(node)
    executor.spin()
    node.executor_pool.shutdown()
    node.destroy_node()
    rclpy.shutdown()
```

---

## どちらを使うべきか

| | asyncio | ThreadPoolExecutor |
|-|---------|-------------------|
| コード複雑さ | 高い | 低い |
| パフォーマンス | 高い（I/O多重化） | 普通 |
| スレッド数 | 少なくて済む | API呼び出しごとにスレッド |
| 向いている場面 | 同時に多数のAPIリクエスト | 呼び出し頻度が低い場合 |

LLM APIの呼び出しは秒単位で遅く、頻度も低い（1〜10秒に1回程度）ので、**ThreadPoolExecutorの方がシンプルで十分**なことが多いです。

---

## 注意点まとめ

1. **ROS2のパブリッシュはROS spinスレッドから呼ぶのが安全**（他スレッドから呼ぶと競合することがある）
2. **タイムアウトを必ず設定**（APIが返ってこなくてもロボット制御が止まらないように）
3. **前のリクエスト処理中のガード**（連続してリクエストが来ても1つだけ処理）
4. **MultiThreadedExecutorを使う**（SingleThreadedExecutorだとコールバックが並列実行されない）
