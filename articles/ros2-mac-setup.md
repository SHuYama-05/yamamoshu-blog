---
title: "Apple SiliconのMacでROS2を動かす【Docker + devcontainer完全ガイド】"
emoji: "🤖"
type: "tech"
topics: ["ROS2", "Docker", "Mac", "AppleSilicon", "devcontainer"]
published: true
---

ROS2はLinuxネイティブですが、Apple Silicon（M1/M2/M3）のMacでもDockerを使えばしっかり動きます。この記事では、開発体験を損なわないdevcontainer環境の構築方法を解説します。

---

## 必要なもの

- Apple Silicon Mac（M1以降）
- Docker Desktop for Mac（Apple Silicon版）
- VSCode + Dev Containers拡張

---

## Dockerイメージの選び方

Apple Silicon（arm64）に対応しているROS2イメージを使います。

```bash
# 公式イメージ（arm64対応）
docker pull ros:humble-ros-base
docker pull osrf/ros:humble-desktop
```

`osrf/ros:humble-desktop` はRViz2などGUIツールが含まれますが、GUIをMacで表示するにはXQuartz設定が必要です。

---

## devcontainer構成

### ディレクトリ構成

```
ros2_ws/
├── .devcontainer/
│   ├── devcontainer.json
│   └── Dockerfile
├── src/
│   └── my_robot/
└── docker-compose.yml
```

### Dockerfile

```dockerfile
FROM ros:humble-ros-base

RUN apt-get update && apt-get install -y \
    python3-pip \
    python3-colcon-common-extensions \
    ros-humble-nav2-bringup \
    ros-humble-navigation2 \
    && rm -rf /var/lib/apt/lists/*

# Python依存パッケージ
RUN pip3 install anthropic numpy scipy

# ROS2環境の自動読み込み
RUN echo "source /opt/ros/humble/setup.bash" >> /root/.bashrc
RUN echo "source ~/ros2_ws/install/setup.bash 2>/dev/null || true" >> /root/.bashrc

WORKDIR /root/ros2_ws
```

### devcontainer.json

```json
{
  "name": "ROS2 Humble",
  "build": {
    "dockerfile": "Dockerfile"
  },
  "mounts": [
    "source=${localWorkspaceFolder},target=/root/ros2_ws,type=bind"
  ],
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python",
        "ms-iot.vscode-ros",
        "ms-vscode.cpptools"
      ]
    }
  },
  "postCreateCommand": "cd /root/ros2_ws && colcon build --symlink-install 2>/dev/null || true"
}
```

---

## よくある問題

### `exec format error`

arm64イメージを使っているか確認。

```bash
docker inspect ros:humble-ros-base | grep Architecture
```

### GUIが表示されない

XQuartzをインストールしてネットワーク接続を許可。

```bash
brew install --cask xquartz
# XQuartxの環境設定 → セキュリティ → ネットワーク接続を許可
export DISPLAY=host.docker.internal:0
```

### ビルドが遅い

`colcon build` はarm64上でエミュレーションが入るとさらに遅くなります。`--symlink-install` でPythonパッケージは再ビルド不要にできます。

---

## まとめ

MacでのROS2開発はDockerが最も安定します。devcontainerを使うとVSCodeからワンクリックで環境を立ち上げられて快適です。
