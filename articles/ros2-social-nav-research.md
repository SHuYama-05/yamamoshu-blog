---
title: "社会的自律移動ロボットの研究実装メモ【LLM + CBF + ROS2の実際】"
emoji: "🧭"
type: "tech"
topics: ["ROS2", "LLM", "CBF", "ロボット", "研究"]
published: true
---
研究室で取り組んでいる「LLMとCBFを組み合わせた社会的自律移動ロボット」の実装について、詰まったところや設計判断をメモとして書き残します。

---

## 研究の概要

「社会的自律移動」とは、単なる障害物回避だけでなく、**人の意図や行動文脈を理解して動く**ロボット制御のことです。

例えば：
- 廊下で人が向かってくるとき、右側に避ける（交通ルール）
- エレベーターを待っている人の横を素通りせず、少し迂回する
- 「急いでいる人」と「立ち話している人」で回避の仕方を変える

これをLLMとCBFで実現しようとしています。

---

## システム構成

```
センサー入力（LiDAR + カメラ）
    ↓
人物検出（YOLO or OpenPose）
    ↓ 人の位置・姿勢・状態
LLM（Claude API）
    ├── 状況の解析（「人が立ち話中」「急いでいる」）
    └── ウェイポイント列の生成（0.1Hz）
    ↓
CBFコントローラー
    ├── 人との安全距離を保つ
    └── LLMのウェイポイントを安全に追従（10Hz）
    ↓
cmd_vel → ロボット
```

---

## 実装で難しかったこと

### 1. LLMへの状況説明をどう構造化するか

最初は「廊下に人がいます。どう動けばいいですか？」という曖昧なプロンプトを使っていましたが、返答が毎回違って安定しませんでした。

今は状況を構造化してプロンプトに入れています。

```python
def build_prompt(self, humans: list[HumanState], goal: Point) -> str:
    human_descriptions = []
    for h in humans:
        desc = (
            f"人{h.id}: 位置=({h.x:.1f}, {h.y:.1f})m, "
            f"速度={h.speed:.1f}m/s, "
            f"方向={h.direction_deg:.0f}度, "
            f"状態={h.state}"  # 'walking', 'standing', 'running'
        )
        human_descriptions.append(desc)

    return f"""あなたはロボットのナビゲーションシステムです。
現在位置: (0, 0)
目標位置: ({goal.x:.1f}, {goal.y:.1f})

周囲の人:
{chr(10).join(human_descriptions)}

社会的なルールを守りながらゴールに向かうウェイポイントを3〜5点生成してください。
JSON形式で返してください: [{{"x": float, "y": float}}, ...]"""
```

### 2. 10Hzで動くCBFに0.1HzのLLMをどう繋ぐか

LLMが「右から回れ」とウェイポイントを生成しても、その間に人が動いてしまうことがあります。

現在の実装：

```python
class LLMCBFController(Node):
    def __init__(self):
        super().__init__('llm_cbf_controller')
        self.waypoints = []          # LLMが生成したウェイポイント列
        self.waypoint_idx = 0
        self.human_positions = []    # リアルタイムの人位置

        # LLMによる計画更新（10秒ごと）
        self.create_timer(10.0, self.update_plan)
        # CBFによる安全制御（0.1秒 = 10Hz）
        self.create_timer(0.1, self.safe_control_step)

    def safe_control_step(self):
        if not self.waypoints:
            return

        # 現在のウェイポイントに向かう nominal 入力
        target = self.waypoints[self.waypoint_idx]
        error = np.array([target.x, target.y]) - self.robot_pos
        nominal_u = 0.4 * error / (np.linalg.norm(error) + 1e-6)

        # リアルタイムの人位置でCBFをかける
        obstacles = [
            Obstacle(h.x, h.y, radius=0.8)  # 人は0.8mの安全距離
            for h in self.human_positions
        ]
        safe_u = self.cbf.safe_control(nominal_u, self.robot_pos, obstacles)

        # ウェイポイント到達判定
        if np.linalg.norm(error) < 0.3:
            self.waypoint_idx = min(
                self.waypoint_idx + 1, len(self.waypoints) - 1
            )

        cmd = Twist()
        cmd.linear.x = float(safe_u[0])
        cmd.linear.y = float(safe_u[1])
        self.cmd_pub.publish(cmd)
```

### 3. 人の状態推定がノイズだらけ

カメラで人の「急いでいるか・立ち止まっているか」を判定しようとしたが、単純な速度閾値では判定が不安定でした。

現状は単純な移動平均で平滑化しています。

```python
from collections import deque

class HumanTracker:
    def __init__(self, history_len=10):
        self.speed_history = deque(maxlen=history_len)

    def update(self, speed: float) -> str:
        self.speed_history.append(speed)
        avg_speed = sum(self.speed_history) / len(self.speed_history)

        if avg_speed < 0.1:
            return 'standing'
        elif avg_speed < 0.8:
            return 'walking'
        else:
            return 'running'
```

カメラベースのより精度の高い推定は今後の課題です。

---

## 現時点での課題

**解決済み**
- CBFのQP失敗（スラック変数で対応）
- LLM応答のJSON不安定（Function Callingに移行）
- asyncioのデッドロック（スレッド分離）

**未解決**
- 複数人が動いているとき、LLMの計画が追いつかない
- 人の意図（「横を通っていい？」のサイン）の検出
- ゴールまでの経路が複数ある場合のLLMの選択肢の評価

---

## 気づいたこと

LLMは「状況の意味理解」は得意ですが、「リアルタイム制御」は苦手。CBFは「数学的安全保証」は得意ですが、「何が社会的に適切か」はわからない。

この2つを役割分担させることで、それぞれの苦手を補える。この組み合わせは思った以上に相性がいいです。

---

*この記事は研究進行中のメモです。実装が進んだら随時更新します。*
