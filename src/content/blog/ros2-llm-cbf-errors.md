---
title: 'ROS2 + LLM + CBF実装で詰まったエラー集【実際の研究で遭遇したもの】'
description: 'LLM（Claude API）とControl Barrier Function（CBF）をROS2に統合する研究で実際に遭遇したエラーと解決策をまとめました。QP最適化の失敗・スレッド競合・TFタイムアウトなど再現性の高い問題を解説します。'
pubDate: '2026-06-27'
tags: ['ROS2', 'LLM', 'CBF', 'デバッグ', 'Python']
---

LLMとCBFを組み合わせたロボット制御の実装を進めるなかで、「なぜ動かないのか全然わからない」という状態に何度も陥りました。同じ経験をする人のために、実際に遭遇したエラーと解決策をまとめます。

---

## 1. CBFのQP（二次計画問題）が突然失敗する

### 症状

```
[WARN] [cbf_node]: QP optimization failed: Positive directional derivative for linesearch
ロボットが突然停止する
```

### 原因

障害物が複数あるとき、CBF制約が矛盾して実行可能解がなくなることがある。

```python
# 問題のある状況
# 障害物Aは「右に行け」と制約
# 障害物Bは「左に行け」と制約
# → 両方満たす解が存在しない → QP失敗
```

### 解決策

ソフト制約（スラック変数）を追加して、制約が矛盾したときでも最善解を返すようにする。

```python
from scipy.optimize import minimize

def safe_control_with_slack(self, nominal_u, robot_pos, obstacles, u_max=0.5):
    """スラック変数付きCBF（QP失敗を防ぐ）"""
    n_obs = len(obstacles)
    # u = [vx, vy, slack_1, slack_2, ..., slack_n]
    n_vars = 2 + n_obs

    def objective(x):
        u = x[:2]
        slacks = x[2:]
        # 元の入力からの偏差 + スラック変数にペナルティ
        return np.linalg.norm(u - nominal_u)**2 + 1000.0 * np.sum(slacks**2)

    constraints = []
    for i, obs in enumerate(obstacles):
        h = self.barrier_function(robot_pos, obs)
        grad_h = self.barrier_gradient(robot_pos, obs)

        def cbf_constraint(x, grad_h=grad_h, h=h, i=i):
            u = x[:2]
            slack = x[2 + i]
            return float(grad_h @ u + self.alpha * h + slack)

        constraints.append({'type': 'ineq', 'fun': cbf_constraint})

    # スラックは非負
    bounds = [(-u_max, u_max), (-u_max, u_max)] + [(0, None)] * n_obs
    x0 = np.concatenate([nominal_u, np.zeros(n_obs)])

    result = minimize(objective, x0, method='SLSQP',
                      bounds=bounds, constraints=constraints)

    if result.success:
        return result.x[:2]
    else:
        self.get_logger().error('QP失敗。停止します')
        return np.zeros(2)
```

---

## 2. LLM APIのレスポンスがJSONじゃない

### 症状

```python
json.JSONDecodeError: Expecting value: line 1 column 1 (char 0)
```

LLMに「JSONで返して」と指示しても、説明文が混じることがある。

### 原因

```
# LLMが返してきたもの（例）
「以下のJSONを返します：
```json
{"linear_x": 0.2, "angular_z": 0.0}
```
ご確認ください。」
```

### 解決策1: プロンプトで強制する

```python
messages=[{
    "role": "user",
    "content": (
        f"ロボットコマンドをJSONのみで返してください。説明不要。\n"
        f"フォーマット: {{\"linear_x\": float, \"angular_z\": float}}\n"
        f"指示: {user_input}"
    )
}]
```

### 解決策2: JSONを抽出するパーサーを書く

```python
import re
import json

def extract_json(text: str) -> dict | None:
    """テキストからJSONを抽出する"""
    # コードブロックの中のJSONを探す
    code_block = re.search(r'```(?:json)?\s*(\{.*?\})\s*```', text, re.DOTALL)
    if code_block:
        try:
            return json.loads(code_block.group(1))
        except json.JSONDecodeError:
            pass

    # 生のJSONを探す
    json_pattern = re.search(r'\{[^{}]*\}', text)
    if json_pattern:
        try:
            return json.loads(json_pattern.group())
        except json.JSONDecodeError:
            pass

    return None
```

### 解決策3: Function Callingを使う（最も確実）

```python
# JSONパースが不要になる
response = client.messages.create(
    model="claude-haiku-4-5-20251001",
    tools=[{
        "name": "move_robot",
        "input_schema": {
            "type": "object",
            "properties": {
                "linear_x": {"type": "number"},
                "angular_z": {"type": "number"}
            }
        }
    }],
    ...
)
# block.inputはすでにdictなのでJSONパース不要
```

---

## 3. ROS2コールバックとasyncioのデッドロック

### 症状

```
# ノードが固まって何も出力されなくなる
# Ctrl+Cでも止まらない
```

### 原因

asyncioループをROS2のspinスレッドと同じスレッドで動かしていた。

```python
# NG: asyncio.run()をコールバック内で呼ぶ → デッドロック
def on_command(self, msg):
    result = asyncio.run(self.call_llm(msg.data))  # ここで固まる
```

### 解決策

asyncioループは別スレッドで動かす。

```python
import threading

class SafeAsyncNode(Node):
    def __init__(self):
        super().__init__('safe_async_node')
        self._loop = asyncio.new_event_loop()
        threading.Thread(
            target=self._loop.run_forever,
            daemon=True
        ).start()

    def on_command(self, msg):
        # asyncioスレッドにコルーチンを投げるだけ（ブロックしない）
        asyncio.run_coroutine_threadsafe(
            self.call_llm(msg.data), self._loop
        )
```

---

## 4. LiDARの障害物検出でロボットが自分自身を検出する

### 症状

```
CBF介入: nominal=[0.2, 0.0] → safe=[0.0, 0.0]
# ロボットが全然動かない
```

### 原因

LiDARのスキャンに自分のフレーム（車輪・ボディ）が映り込んでいる。

```python
# scan_cbでobs.radius=0.1にすると自分自身も障害物になる
for r in msg.ranges:
    if msg.range_min < r < msg.range_max:  # range_minが小さすぎる
        self.obstacles.append(Obstacle(ox, oy, 0.1))
```

### 解決策

近すぎる点を除外するか、フィルタリングを追加する。

```python
MIN_RANGE = 0.3  # ロボット自身のサイズより大きい値

for r in msg.ranges:
    if MIN_RANGE < r < msg.range_max:  # range_minを自分で設定
        ox = self.robot_pos[0] + r * np.cos(angle)
        oy = self.robot_pos[1] + r * np.sin(angle)
        self.obstacles.append(Obstacle(ox, oy, 0.1))
    angle += msg.angle_increment
```

---

## 5. TFのタイムスタンプずれでSLAM地図が壊れる

### 症状

```
[WARN] [slam_toolbox]: Failed to compute odom pose, Lookup would require extrapolation
地図に歪みが生じる
```

### 原因

LLM APIの呼び出し中にスレッドがビジーになり、TFのブロードキャストが遅延する。

```python
# LLM呼び出しがメインスレッドをブロックしてTFが遅れる
def timer_callback(self):
    response = requests.post(...)  # ここでブロック → TF遅延
    self.broadcast_tf()            # 遅れて送信される
```

### 解決策

TFブロードキャストは専用タイマーで高頻度に維持する。LLM処理とは分離。

```python
class SeparatedTimerNode(Node):
    def __init__(self):
        super().__init__('separated_node')
        # TFブロードキャストは高頻度（50Hz）で維持
        self.create_timer(0.02, self.broadcast_tf)
        # LLM呼び出しは低頻度（0.1Hz）で別スレッド
        self.create_timer(10.0, self.schedule_llm_call)

    def broadcast_tf(self):
        # 軽い処理のみ。絶対にブロックしない
        t = TransformStamped()
        t.header.stamp = self.get_clock().now().to_msg()
        # ...
        self.br.sendTransform(t)

    def schedule_llm_call(self):
        # 別スレッドに投げてすぐ返る
        threading.Thread(target=self._call_llm, daemon=True).start()
```

---

## まとめ

| エラー | 根本原因 | 解決策 |
|--------|---------|--------|
| QP最適化失敗 | 制約が矛盾 | スラック変数を追加 |
| JSONパースエラー | LLMが説明文を混ぜる | Function Callingを使う |
| デッドロック | asyncioをROS spinスレッドで呼ぶ | 別スレッドで動かす |
| 自分自身を障害物検出 | LiDARのrange_minが小さい | MIN_RANGEを自分で設定 |
| TFタイムアウト | LLM処理がTFをブロック | TFと処理スレッドを分離 |

どれも「なぜ？」がわかるまでに時間がかかりました。同じ箇所で詰まっている人の参考になれば。
