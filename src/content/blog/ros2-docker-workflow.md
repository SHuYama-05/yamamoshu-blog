---
title: 'ROS2開発をDockerで快適にする方法【devcontainer対応】'
description: 'ROS2の開発環境をDockerで構築する方法を解説。VS Code devcontainerと組み合わせてホストを汚さずに開発する手順を紹介します。'
pubDate: '2026-06-15'
tags: ['ROS2', 'Docker', '開発環境', 'VSCode']
---

ROS2の環境構築はOSバージョンに依存するため、Dockerを使うと環境の再現性が上がります。

## Dockerfile

```dockerfile
FROM osrf/ros:humble-desktop

# 開発ツールをインストール
RUN apt-get update && apt-get install -y \
    python3-pip \
    python3-colcon-common-extensions \
    ros-humble-nav2-bringup \
    && rm -rf /var/lib/apt/lists/*

# ワーキングディレクトリ
WORKDIR /ros2_ws

# bashrcにsourceを追加
RUN echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
```

## docker-compose.yml

```yaml
services:
  ros2:
    build: .
    volumes:
      - ./src:/ros2_ws/src
      - /tmp/.X11-unix:/tmp/.X11-unix
    environment:
      - DISPLAY=${DISPLAY}
      - ROS_DOMAIN_ID=0
    network_mode: host
    stdin_open: true
    tty: true
```

## VS Code devcontainer設定

`.devcontainer/devcontainer.json`:

```json
{
  "name": "ROS2 Humble",
  "dockerComposeFile": "../docker-compose.yml",
  "service": "ros2",
  "workspaceFolder": "/ros2_ws",
  "extensions": [
    "ms-python.python",
    "ms-iot.vscode-ros"
  ],
  "postCreateCommand": "colcon build --symlink-install"
}
```

## 使い方

```bash
# コンテナ起動
docker compose up -d

# コンテナに入る
docker compose exec ros2 bash

# ビルド
colcon build --symlink-install
source install/setup.bash
```

VS Codeで「Reopen in Container」を選べば自動でコンテナ内で開発できます。

## Macでの注意点

Apple SiliconではGUIツール（RViz2）を使うためにXQuartzが必要です。

```bash
brew install --cask xquartz
xhost +localhost
```

その後 `docker-compose.yml` の `DISPLAY` を `host.docker.internal:0` にすればRViz2が動きます。
