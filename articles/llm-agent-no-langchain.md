---
title: "LangChainなしでLLMエージェントを作る【Claude API + Python実装】"
emoji: "🤖"
type: "tech"
topics: ["LLM", "Claude", "Python", "エージェント"]
published: true
---
LangChainは便利ですが、ブラックボックスになりがち。Claude APIのFunction Callingを使えば、エージェントの仕組みを理解しながら自分でゼロから実装できます。

---

## エージェントの基本構造

```
ユーザー入力
    ↓
LLM（何をすべきか判断）
    ↓ ツール呼び出しが必要
ツール実行（Python関数）
    ↓ 結果をLLMに返す
LLM（結果を見て次の行動を決める）
    ↓ ... ループ
最終回答
```

これを「ReAct（Reasoning + Acting）」と呼びます。

---

## ツールの定義

```python
import anthropic
import json
from typing import Any

client = anthropic.Anthropic()

# ツール定義
TOOLS = [
    {
        "name": "calculate",
        "description": "数式を計算する",
        "input_schema": {
            "type": "object",
            "properties": {
                "expression": {
                    "type": "string",
                    "description": "計算する数式（例: '2 + 3 * 4'）"
                }
            },
            "required": ["expression"]
        }
    },
    {
        "name": "search_web",
        "description": "キーワードでWeb検索する",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "検索クエリ"
                }
            },
            "required": ["query"]
        }
    },
    {
        "name": "get_current_time",
        "description": "現在の日時を取得する",
        "input_schema": {
            "type": "object",
            "properties": {}
        }
    }
]
```

---

## ツール実行関数

```python
import math
from datetime import datetime

def execute_tool(name: str, inputs: dict) -> str:
    """ツールを実行して結果を文字列で返す"""

    if name == "calculate":
        expression = inputs["expression"]
        try:
            # evalは危険なので本番では使わない（ここはデモ）
            result = eval(expression, {"__builtins__": {}}, {
                "math": math, "sqrt": math.sqrt, "pi": math.pi
            })
            return f"計算結果: {expression} = {result}"
        except Exception as e:
            return f"計算エラー: {e}"

    elif name == "search_web":
        query = inputs["query"]
        # 実際にはserpapi等を使う
        return f"検索結果（ダミー）: '{query}' に関する情報が見つかりました。"

    elif name == "get_current_time":
        now = datetime.now()
        return f"現在時刻: {now.strftime('%Y年%m月%d日 %H:%M:%S')}"

    else:
        return f"未知のツール: {name}"
```

---

## エージェントのメインループ

```python
def run_agent(user_message: str, max_iterations: int = 10) -> str:
    """
    エージェントのメインループ
    ツール呼び出しが不要になるまで繰り返す
    """
    messages = [{"role": "user", "content": user_message}]

    for iteration in range(max_iterations):
        print(f"\n--- イテレーション {iteration + 1} ---")

        response = client.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=1024,
            tools=TOOLS,
            messages=messages
        )

        print(f"stop_reason: {response.stop_reason}")

        # ツール呼び出しがなければ終了
        if response.stop_reason == "end_turn":
            final_text = ""
            for block in response.content:
                if hasattr(block, "text"):
                    final_text += block.text
            return final_text

        # ツール呼び出しを処理
        if response.stop_reason == "tool_use":
            # アシスタントの応答をメッセージ履歴に追加
            messages.append({
                "role": "assistant",
                "content": response.content
            })

            # 各ツールを実行してresultを集める
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    print(f"ツール呼び出し: {block.name}({block.input})")
                    result = execute_tool(block.name, block.input)
                    print(f"結果: {result}")

                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result
                    })

            # ツール結果をメッセージ履歴に追加
            messages.append({
                "role": "user",
                "content": tool_results
            })

    return "最大イテレーション数に達しました"
```

---

## 実行例

```python
# シンプルな計算
result = run_agent("√144 + 2^10 はいくつですか？")
print(result)
# → 計算結果: √144 = 12、2^10 = 1024、合計 = 1036

# 複数ツールを組み合わせる
result = run_agent("今日は何日か教えてください。それと 365 × 24 × 60 を計算してください。")
print(result)
# → 2026年06月27日 15:30:00
# → 365 × 24 × 60 = 525600（1年間の分数）
```

---

## 会話メモリを追加する

```python
class AgentWithMemory:
    def __init__(self):
        self.client = anthropic.Anthropic()
        self.conversation_history = []

    def chat(self, user_message: str) -> str:
        self.conversation_history.append({
            "role": "user",
            "content": user_message
        })

        response = self.client.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=1024,
            system="あなたは親切なアシスタントです。",
            tools=TOOLS,
            messages=self.conversation_history
        )

        # ... ツール処理 ...

        # 会話履歴に追加（次のターンで参照される）
        self.conversation_history.append({
            "role": "assistant",
            "content": response.content
        })

        return final_answer

    def clear_memory(self):
        """会話履歴をリセット"""
        self.conversation_history = []
```

---

## LangChainと比べて

| | 自前実装 | LangChain |
|---|---|---|
| 学習コスト | 低い（APIを直接理解） | 高い（抽象化を学ぶ） |
| デバッグ | 簡単（全部見える） | 難しい（ブラックボックス） |
| カスタマイズ | 自由 | 制約あり |
| コード量 | 多い | 少ない |

研究・学習目的なら自前実装で仕組みを理解する方がおすすめです。
