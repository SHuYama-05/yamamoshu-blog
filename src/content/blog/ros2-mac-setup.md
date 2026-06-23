---
title: 'ROS2 HumbleをMac（Apple Silicon）にインストールする方法【2025年版】'
description: 'M1/M2/M3 MacにROS2 Humbleをインストールする手順を解説。Dockerを使った方法と、Homebrewを使ったネイティブ方法の両方を紹介します。'
pubDate: '2025-06-23'
heroImage: ''
tags: ['ROS2', 'Mac', 'Apple Silicon', '環境構築']
---

ROS2をMac（Apple Silicon）にインストールしようとして詰まった経験がある方は多いはずです。
この記事では実際に動作確認した手順をまとめます。

## 結論：Dockerを使うのが最速

ネイティブインストールは依存関係の問題で詰まりやすいです。
**Dockerを使う方法が一番ストレスなく動きます。**

## 方法1：Docker（推奨）

### 前提
- Docker Desktop for Mac がインストール済み
- Apple Silicon（M1/M2/M3）

### 手順

```bash
# ROS2 Humble の公式イメージを取得
docker pull osrf/ros:humble-desktop

# コンテナを起動（GUIなし）
docker run -it --rm osrf/ros:humble-desktop bash

# 動作確認
source /opt/ros/humble/setup.bash
ros2 --version
```

### GUIツール（RViz2など）を使う場合

```bash
# XQuartz をインストール
brew install --cask xquartz

# XQuartz を起動後、以下を実行
xhost +localhost

docker run -it --rm \
  -e DISPLAY=host.docker.internal:0 \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  osrf/ros:humble-desktop bash
```

## 方法2：Homebrew（ネイティブ）

ROS2の公式はUbuntuのみサポートですが、有志がHomebrewでのインストールを整備しています。

```bash
# ros/ros2 タップを追加
brew tap ros/ros2

# ROS2 Humble をインストール
brew install ros-humble-ros-base
```

ただし**依存パッケージの競合が起きやすい**ので、Dockerの方を推奨します。

## まとめ

| 方法 | メリット | デメリット |
|------|---------|-----------|
| Docker | 安定・環境汚染なし | GUI設定がやや面倒 |
| Homebrew | ネイティブで動く | 依存関係で詰まりやすい |

実際の開発では**Dockerで動かしてコードはホストで書く**スタイルが一番快適でした。

次回はROS2の基本概念（ノード・トピック・サービス）を解説します。
