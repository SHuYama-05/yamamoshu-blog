---
title: 'ROS2とPython asyncioを組み合わせた非同期処理の実装'
description: 'ROS2のコールバックベースの処理とPythonのasyncioを組み合わせて、効率的な非同期処理を実装する方法を解説します。'
pubDate: '2026-06-01'
tags: ['ROS2', 'Python', 'asyncio', '非同期処理']
---

ROS2のspinとPythonのasyncioは相性が難しいです。LLM APIの呼び出し（非同期・低速）とロボット制御（同期・高速）を組み合わせるときに詰まりました。

## 問題

```python
# これは動かない
async def llm_callback():
    response = await openai.chat.completions.create(...)  # 非同期
    publish_to_ros(response)

# ROS2のspinはasync関数を直接呼べない
rclpy.spin(node)
```

## 解決策：executor + asyncio

```python
import asyncio
import rclpy
from rclpy.node import Node
from rclpy.executors import MultiThreadedExecutor
import threading

class AsyncROS2Node(Node):
    def __init__(self):
        super().__init__('async_node')
        self.loop = asyncio.new_event_loop()
        self.thread = threading.Thread(
            target=self.loop.run_forever, daemon=True
        )
        self.thread.start()

    def call_async(self, coro):
        return asyncio.run_coroutine_threadsafe(coro, self.loop)

    def on_message_received(self, msg):
        # コールバック内から非同期処理を起動
        future = self.call_async(self.process_with_llm(msg.data))
        future.add_done_callback(self.on_llm_done)

    async def process_with_llm(self, text):
        from openai import AsyncOpenAI
        client = AsyncOpenAI()
        response = await client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": text}]
        )
        return response.choices[0].message.content

    def on_llm_done(self, future):
        result = future.result()
        self.get_logger().info(f'LLM応答: {result}')
        # ここでROS2トピックにパブリッシュ


def main():
    rclpy.init()
    node = AsyncROS2Node()
    executor = MultiThreadedExecutor()
    executor.add_node(node)
    executor.spin()
```

## タイムアウト処理

LLM APIは遅いので、タイムアウトを設定することが重要です。

```python
async def process_with_llm_timeout(self, text, timeout=5.0):
    try:
        return await asyncio.wait_for(
            self.process_with_llm(text),
            timeout=timeout
        )
    except asyncio.TimeoutError:
        self.get_logger().warn('LLM APIタイムアウト。デフォルト動作を実行')
        return None
```

## まとめ

- ROS2の `spin` とasyncioは別スレッドで動かす
- `asyncio.run_coroutine_threadsafe` でスレッド間通信
- LLM APIには必ずタイムアウトを設定する
- `MultiThreadedExecutor` でコールバックの並列実行を有効化
