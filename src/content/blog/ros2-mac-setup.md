---
title: 'ROS2 HumbleをMac（Apple Silicon）にインストールする方法【2025年版】'
description: 'M1/M2/M3 MacにROS2 Humbleをインストールする手順を解説。Dockerを使った方法とHomebrewを使ったネイティブ方法の両方を紹介します。'
pubDate: '2025-06-23'
tags: ['ROS2', 'Mac', 'Apple Silicon', '環境構築']
---

ROS2をMac（Apple Silicon）にインストールしようとして詰まった経験がある方は多いはずです。公式ドキュメントはUbuntu中心で、Mac向けの情報が少ないです。この記事では実際に動作確認した手順をまとめます。

## 結論：Dockerを使うのが最速

ネイティブインストールは依存関係の問題で詰まりやすいです。**Dockerを使う方法が一番ストレスなく動きます。**

方法の比較から先に示します。

| 方法 | メリット | デメリット | おすすめ度 |
|------|---------|-----------|-----------|
| Docker | 安定・環境汚染なし・再現性高い | GUI設定がやや面倒 | ★★★★★ |
| Homebrew | ネイティブで動く | 依存関係で詰まりやすい | ★★★☆☆ |
| Lima + QEMU | Ubuntu環境をまるごと動かせる | 遅い・設定が複雑 | ★★☆☆☆ |

---

## 方法1：Docker（推奨）

### 前提条件

- Docker Desktop for Mac がインストール済み（[公式サイト](https://www.docker.com/products/docker-desktop/)）
- Apple Silicon（M1/M2/M3）または Intel Mac

### ステップ1：ROS2 Humbleイメージの取得

```bash
docker pull osrf/ros:humble-desktop
```

`humble-desktop` はRViz2などのGUIツールも含んだフルイメージです。軽量版は `humble-ros-base`。

### ステップ2：コンテナ起動（CUIのみ）

```bash
docker run -it --rm \
  --name ros2_humble \
  -v ~/ros2_ws:/ros2_ws \
  osrf/ros:humble-desktop \
  bash
```

- `-v ~/ros2_ws:/ros2_ws`：ホストのフォルダをコンテナ内にマウント（コードはホスト側で書ける）
- `--rm`：コンテナ終了時に自動削除

コンテナ内で動作確認：

```bash
source /opt/ros/humble/setup.bash
ros2 --version
# ros2 1.3.x が表示されれば成功
```

### ステップ3：RViz2などGUIを使う場合

まずXQuartzをインストールします。

```bash
brew install --cask xquartz
```

XQuartzを起動してから（Applications > XQuartz）、**環境設定 > セキュリティ > 「ネットワーク・クライアントからの接続を許可」にチェック**を入れます。

その後、ターミナルで：

```bash
xhost +localhost
```

Dockerコンテナ起動時に `DISPLAY` を渡します：

```bash
docker run -it --rm \
  --name ros2_gui \
  -e DISPLAY=host.docker.internal:0 \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v ~/ros2_ws:/ros2_ws \
  osrf/ros:humble-desktop \
  bash
```

コンテナ内でRViz2を起動：

```bash
source /opt/ros/humble/setup.bash
rviz2
```

ウィンドウが表示されれば成功です。

### docker-compose.yml（毎回コマンドを打つのが面倒な場合）

```yaml
services:
  ros2:
    image: osrf/ros:humble-desktop
    environment:
      - DISPLAY=host.docker.internal:0
      - ROS_DOMAIN_ID=0
    volumes:
      - ~/ros2_ws:/ros2_ws
      - /tmp/.X11-unix:/tmp/.X11-unix
    network_mode: host
    stdin_open: true
    tty: true
    command: bash -c "source /opt/ros/humble/setup.bash && bash"
```

```bash
# 起動
docker compose up -d

# コンテナに入る
docker compose exec ros2 bash
```

---

## 方法2：Homebrew（ネイティブ）

有志が管理しているHomebrewのROS2フォーミュラを使います。公式サポートではないため壊れることがありますが、Dockerが使えない場面では有効です。

### インストール

```bash
# Homebrewが入っていない場合
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ros/ros2 タップを追加
brew tap ros/ros2

# ROS2 Humble（ベース）をインストール
brew install ros-humble-ros-base
```

フルインストール（RViz2等含む）は時間がかかります（1〜2時間）：

```bash
brew install ros-humble-desktop
```

### セットアップ

```bash
# ~/.zshrc に追加
echo "source /opt/homebrew/opt/ros-humble/setup.zsh" >> ~/.zshrc
source ~/.zshrc

# 動作確認
ros2 --version
```

### よくある依存関係エラーへの対処

```bash
# Python バージョンの競合
brew install python@3.11
export PATH="/opt/homebrew/opt/python@3.11/bin:$PATH"

# OpenSSL 関連のエラー
brew install openssl
export OPENSSL_ROOT_DIR=$(brew --prefix openssl)
```

---

## 開発ワークフローの推奨構成

個人的には以下の構成が一番快適でした：

```
ホスト（Mac）：VSCodeでコードを書く
     ↕ ボリュームマウント
Dockerコンテナ：ROS2の実行・ビルド・テスト
```

VSCodeの「Dev Containers」拡張機能を使うと、コンテナ内に入った状態でVSCodeが動くのでさらに快適です。

`.devcontainer/devcontainer.json`:

```json
{
  "name": "ROS2 Humble",
  "image": "osrf/ros:humble-desktop",
  "postCreateCommand": "echo 'source /opt/ros/humble/setup.bash' >> ~/.bashrc",
  "extensions": [
    "ms-python.python",
    "ms-iot.vscode-ros"
  ]
}
```

---

## まとめ

- **すぐ試したい** → Dockerで `docker run -it osrf/ros:humble-desktop bash`
- **GUIも使いたい** → XQuartz + Docker
- **ネイティブにこだわる** → Homebrew（詰まる覚悟で）
- **本格的に開発** → docker-compose + VSCode Dev Containers

次回はROS2の基本概念（ノード・トピック・サービス）を解説します。
