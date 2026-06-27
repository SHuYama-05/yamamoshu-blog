---
title: "ROS2開発をDockerで完結させる【docker-compose・devcontainer・Tips】"
emoji: "🐳"
type: "tech"
topics: ["ROS2", "Docker", "devcontainer", "開発環境"]
published: true
---

ROS2の開発環境をDockerで管理するメリットは大きい。チームで環境差異をなくせるし、ホストのLinuxを汚さない。実際の開発で使っている構成を紹介します。

---

## docker-compose.yml

```yaml
version: '3.8'

services:
  ros2:
    build:
      context: .
      dockerfile: .devcontainer/Dockerfile
    volumes:
      - .:/root/ros2_ws
      - /tmp/.X11-unix:/tmp/.X11-unix  # GUI用
    environment:
      - DISPLAY=${DISPLAY}
      - ROS_DOMAIN_ID=42
    network_mode: host  # ROS2のDDS通信に必要
    privileged: true    # デバイスアクセス用
    stdin_open: true
    tty: true
    command: /bin/bash
```

---

## Dockerfile

```dockerfile
FROM ros:humble-ros-base

ARG DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    python3-pip \
    python3-colcon-common-extensions \
    python3-rosdep \
    ros-humble-nav2-bringup \
    ros-humble-navigation2 \
    ros-humble-slam-toolbox \
    ros-humble-gazebo-ros-pkgs \
    vim \
    git \
    && rm -rf /var/lib/apt/lists/*

RUN pip3 install \
    anthropic \
    numpy \
    scipy \
    matplotlib

# rosdep初期化
RUN rosdep init && rosdep update

COPY .devcontainer/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

### entrypoint.sh

```bash
#!/bin/bash
source /opt/ros/humble/setup.bash
if [ -f /root/ros2_ws/install/setup.bash ]; then
    source /root/ros2_ws/install/setup.bash
fi
exec "$@"
```

---

## よく使うコマンド

```bash
# コンテナを起動（バックグラウンド）
docker compose up -d

# コンテナに入る
docker compose exec ros2 bash

# コンテナ内でビルド
cd /root/ros2_ws && colcon build --symlink-install

# 停止
docker compose down
```

---

## VSCode devcontainer設定

`.devcontainer/devcontainer.json`:

```json
{
  "name": "ROS2 Humble",
  "dockerComposeFile": "../docker-compose.yml",
  "service": "ros2",
  "workspaceFolder": "/root/ros2_ws",
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python",
        "ms-iot.vscode-ros",
        "ms-vscode.cpptools",
        "ms-python.black-formatter"
      ],
      "settings": {
        "python.defaultInterpreterPath": "/usr/bin/python3",
        "editor.formatOnSave": true
      }
    }
  },
  "postCreateCommand": "cd /root/ros2_ws && colcon build --symlink-install 2>/dev/null || true"
}
```

---

## Tips

### ROS_DOMAIN_IDの設定

同じネットワーク上に複数のROS2環境があると混線する。プロジェクトごとに異なるIDを設定する。

```bash
# docker-compose.ymlで設定するか
export ROS_DOMAIN_ID=42

# または .bashrcに追加
echo "export ROS_DOMAIN_ID=42" >> ~/.bashrc
```

### イメージサイズを減らす

`ros-humble-desktop` は大きい。必要なパッケージだけ指定する。

```dockerfile
# NG: 全部入り（数GB）
FROM osrf/ros:humble-desktop

# OK: 必要なものだけ
FROM ros:humble-ros-base
RUN apt-get install -y ros-humble-nav2-bringup
```
