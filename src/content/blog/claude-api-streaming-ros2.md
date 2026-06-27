---
title: 'Claude APIのストリーミング応答をROS2で使う【リアルタイム音声合成・UI更新】'
description: 'Anthropic Claude APIのストリーミング機能をROS2ノードで活用する方法を解説。応答が届くたびにROSトピックに流すことで、音声合成やUIのリアルタイム更新に活用できます。'
pubDate: '2026-06-24'
tags: ['ROS2', 'Claude', 'LLM', 'ストリーミング', 'Python']
---

LLMの応答は全部揃ってから届くより、生成されるたびに届く方が体験がいい。Claude APIのストリーミングをROS2と組み合わせて、ロボットの応答をリアルタイムに処理する方法を解説します。

---

## なぜストリーミングが必要か

通常の呼び出し：
```
リクエスト → [3〜10秒待つ] → 全文が届く → 音声合成開始
```

ストリーミング：
```
リクエスト → [0.5秒] → 最初の単語 → 音声合成開始
           → [逐次届く] → テキストをROSトピックに流す
```

ユーザーへの体験が大きく変わります。

---

## 基本的なストリーミング実装

```python
import anthropic

client = anthropic.Anthropic()

with client.messages.stream(
    model="claude-haiku-4-5-20251001",
    max_tokens=512,
    messages=[{"role": "user", "content": "ROS2について教えて"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)

# 最終的なメッセージ全体も取得できる
final_message = stream.get_final_message()
```

---

## ROS2ノードへの統合

文字が届くたびに `/llm_stream` トピックに流します。

```python
import anthropic
import threading
from std_msgs.msg import String
from rclpy.node import Node
import rclpy

class StreamingLLMNode(Node):
    def __init__(self):
        super().__init__('streaming_llm')

        # ストリーミングの各チャンクを流すトピック
        self.chunk_pub = self.create_publisher(String, 'llm_chunk', 10)
        # 文章が完成したタイミングで流すトピック
        self.sentence_pub = self.create_publisher(String, 'llm_sentence', 10)
        # 全文完成後のトピック
        self.complete_pub = self.create_publisher(String, 'llm_complete', 10)

        self.client = anthropic.Anthropic()
        self.sub = self.create_subscription(
            String, 'user_input', self.on_input, 10
        )
        self._processing = False

    def on_input(self, msg: String):
        if self._processing:
            self.get_logger().warn('前の処理中。スキップ')
            return
        self._processing = True
        thread = threading.Thread(
            target=self.stream_response,
            args=(msg.data,),
            daemon=True
        )
        thread.start()

    def stream_response(self, user_text: str):
        """別スレッドでストリーミング実行"""
        try:
            buffer = ''  # 文章バッファ

            with self.client.messages.stream(
                model="claude-haiku-4-5-20251001",
                max_tokens=512,
                messages=[{
                    "role": "user",
                    "content": user_text
                }]
            ) as stream:
                for text_chunk in stream.text_stream:
                    # チャンクを即座にパブリッシュ
                    chunk_msg = String()
                    chunk_msg.data = text_chunk
                    self.chunk_pub.publish(chunk_msg)

                    buffer += text_chunk

                    # 句読点で文章を区切って音声合成に送る
                    for delimiter in ['。', '！', '？', '\n']:
                        if delimiter in buffer:
                            parts = buffer.split(delimiter, 1)
                            sentence = parts[0] + delimiter
                            buffer = parts[1]

                            sentence_msg = String()
                            sentence_msg.data = sentence
                            self.sentence_pub.publish(sentence_msg)
                            self.get_logger().info(f'文章: {sentence}')

            # 残りのバッファを送出
            if buffer.strip():
                sentence_msg = String()
                sentence_msg.data = buffer
                self.sentence_pub.publish(sentence_msg)

            # 全文完了通知
            complete_msg = String()
            complete_msg.data = stream.get_final_text()
            self.complete_pub.publish(complete_msg)

        except Exception as e:
            self.get_logger().error(f'ストリーミングエラー: {e}')
        finally:
            self._processing = False
```

---

## 音声合成ノードとの連携

`/llm_sentence` を音声合成ノードが受け取る構成。

```python
import subprocess

class TTSNode(Node):
    """文章を受け取って音声合成するノード"""
    def __init__(self):
        super().__init__('tts_node')
        self.sub = self.create_subscription(
            String, 'llm_sentence', self.speak, 10
        )

    def speak(self, msg: String):
        text = msg.data
        self.get_logger().info(f'読み上げ: {text}')
        # Open JTalkで音声合成（例）
        subprocess.run([
            'open_jtalk',
            '-x', '/var/lib/mecab/dic/open-jtalk/naist-jdic',
            '-m', '/usr/share/hts-voice/nitech-jp-atr503-m001/nitech_jp_atr503_m001.htsvoice',
            '-ow', '/tmp/speech.wav'
        ], input=text.encode(), check=True)
        subprocess.run(['aplay', '/tmp/speech.wav'])
```

---

## 非同期クライアントを使う場合

asyncio版のストリーミング。

```python
import anthropic
import asyncio

async def stream_async(user_text: str):
    client = anthropic.AsyncAnthropic()

    async with client.messages.stream(
        model="claude-haiku-4-5-20251001",
        max_tokens=512,
        messages=[{"role": "user", "content": user_text}]
    ) as stream:
        async for text in stream.text_stream:
            print(text, end="", flush=True)
            # ここでROSトピックに流す処理を入れる

asyncio.run(stream_async("ROS2の特徴を教えて"))
```

---

## まとめ

- `client.messages.stream()` でストリーミング開始
- チャンクが届くたびにROSトピックにパブリッシュ
- 句読点で区切って音声合成ノードに送ると自然な読み上げになる
- ストリーミング処理は必ず別スレッドで（ROS2 spinをブロックしない）
