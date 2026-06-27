---
title: "ROS2とPython asyncioを組み合わせた非同期処理【LLM API統合の実装例】"
emoji: "⚡"
type: "tech"
topics: ["ROS2", "Python", "asyncio", "非同期処理", "LLM"]
published: true
---

ROS2でLLM APIを呼ぶとき、「APIの呼び出し（非同期・低速）」と「ロボット制御（同期・高速）」をどう組み合わせるかが問題になります。コールバック内でAPIをブロック呼び出しすると制御が止まります。

---

## 問題

```python
# NG: コールバック内でAPI呼び出しをブロックすると制御が止まる
def scan_callback(self, msg):
    response = requests.post(...)  # ここでブロック → cmd_velが止まる
    self.publish_command(response)
```

ROS2の `spin()` とPythonの `asyncio` イベントループは別物なので、直接混ぜることはできません。

---

## 解決策1: asyncioを別スレッドで動かす

```python
import asyncio
import threading

class AsyncROS2Node(Node):
    def __init__(self):
        super().__init__('async_node')
        self._loop = asyncio.new_event_loop()
        self._thread = threading.Thread(
            target=self._run_loop, daemon=True
        )
        self._thread.start()

    def _run_loop(self):
        asyncio.set_event_loop(self._loop)
        self._loop.run_forever()

    def run_async(self, coro):
        return asyncio.run_coroutine_threadsafe(coro, self._loop)
```

```python
class LLMNode(AsyncROS2Node):
    def __init__(self):
        super().__init__('llm_node')
        self.client = anthropic.AsyncAnthropic()
        self.sub = self.create_subscription(
            String, 'command', self.on_command, 10
        )

    def on_command(self, msg):
        # asyncioスレッドで非同期処理を開始（ブロックしない）
        future = self.run_async(self.call_llm(msg.data))
        future.add_done_callback(self.on_done)

    async def call_llm(self, text):
        try:
            response = await asyncio.wait_for(
                self.client.messages.create(
                    model="claude-haiku-4-5-20251001",
                    max_tokens=128,
                    messages=[{"role": "user", "content": text}]
                ),
                timeout=5.0
            )
            return response.content[0].text
        except asyncio.TimeoutError:
            return None

    def on_done(self, future):
        result = future.result()
        if result:
            # ROSパブリッシュはROS spinスレッドから実行
            self._queued = result
            self.create_timer(0.0, self._publish)

    def _publish(self):
        if hasattr(self, '_queued'):
            msg = String(data=self._queued)
            del self._queued
            self.pub.publish(msg)
```

---

## 解決策2: ThreadPoolExecutor（シンプル）

asyncioより単純。LLM APIのような低頻度呼び出しにはこちらで十分。

```python
from concurrent.futures import ThreadPoolExecutor

class ThreadedLLMNode(Node):
    def __init__(self):
        super().__init__('threaded_llm')
        self.pool = ThreadPoolExecutor(max_workers=2)
        self.client = anthropic.Anthropic()  # 同期クライアント
        self._processing = False

    def on_command(self, msg):
        if self._processing:
            return
        self._processing = True
        future = self.pool.submit(self.call_llm, msg.data)
        future.add_done_callback(self.on_done)

    def call_llm(self, text):
        try:
            resp = self.client.messages.create(
                model="claude-haiku-4-5-20251001",
                max_tokens=128,
                messages=[{"role": "user", "content": text}]
            )
            return resp.content[0].text
        except Exception as e:
            self.get_logger().error(str(e))
            return None

    def on_done(self, future):
        self._processing = False
        # 処理続行...
```

---

## どちらを使う？

| | asyncio | ThreadPoolExecutor |
|---|---|---|
| コード量 | 多い | 少ない |
| I/O多重化 | 得意 | 普通 |
| 向いている場面 | 同時多数リクエスト | 低頻度のAPI呼び出し |

LLM APIは1〜10秒に1回程度なので、**ThreadPoolExecutorで十分**なことが多い。

---

## 注意事項

1. **ROSのパブリッシュはROS spinスレッドから**（別スレッドから呼ぶと競合の危険）
2. **タイムアウト必須**（APIが返ってこなくてもロボット制御が止まらないように）
3. **MultiThreadedExecutorを使う**（SingleThreadedExecutorだとコールバックが並列実行されない）
