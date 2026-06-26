---
title: 'ROS2開発をDockerで快適にする方法【VSCode devcontainer完全対応】'
description: 'ROS2の開発環境をDockerで構築する方法を解説。VS Code devcontainerと組み合わせてホストを汚さずに開発する手順と、Macでのハマりポイントも紹介します。'
pubDate: '2026-06-15'
tags: ['ROS2', 'Docker', '開発環境', 'VSCode']
---

ROS2の環境構築はOSバージョンに強く依存します。Ubuntu 22.04にはHumble、24.04にはJazzyというように、ディストリビューションが固定されています。

Macや Windows で開発する場合、またはチームで同じ環境を共有したい場合は **Docker** を使うのがベストです。

この記事では「Dockerでの基本的な使い方」から「VSCode devcontainerで本格的に開発する方法」まで順に説明します。

---

## Dockerfile

```dockerfile
FROM osrf/ros:humble-desktop

# タイムゾーン設定（対話的なプロンプトを避ける）
ENV DEBIAN_FRONTEND=noninteractive
ENV TZ=Asia/Tokyo

# 開発ツールを追加インストール
RUN apt-get update && apt-get install -y \
    python3-pip \
    python3-colcon-common-extensions \
    python3-rosdep \
    ros-humble-nav2-bringup \
    ros-humble-slam-toolbox \
    ros-humble-tf2-tools \
    vim \
    git \
    && rm -rf /var/lib/apt/lists/*

# Python パッケージ
RUN pip3 install \
    anthropic \
    numpy \
    scipy

# ワーキングディレクトリ
WORKDIR /ros2_ws

# ROS2の環境を自動で読み込む
RUN echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc && \
    echo "source /ros2_ws/install/setup.bash 2>/dev/null || true" >> ~/.bashrc

# デフォルトでbashを起動
CMD ["/bin/bash"]
```

---

## docker-compose.yml

```yaml
services:
  ros2:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: ros2_dev
    volumes:
      # ソースコードをマウント（ホスト側で編集したものがコンテナに即反映）
      - ./src:/ros2_ws/src
      # ビルドキャッシュを保持（毎回フルビルドしなくて済む）
      - ros2_build:/ros2_ws/build
      - ros2_install:/ros2_ws/install
      - ros2_log:/ros2_ws/log
      # GUI用（Linux/Mac）
      - /tmp/.X11-unix:/tmp/.X11-unix
    environment:
      - DISPLAY=${DISPLAY:-host.docker.internal:0}
      - ROS_DOMAIN_ID=0
      - TURTLEBOT3_MODEL=burger
    network_mode: host
    stdin_open: true
    tty: true
    # privileged はシリアルポート（実機）にアクセスする場合に必要
    # privileged: true

volumes:
  ros2_build:
  ros2_install:
  ros2_log:
```

### よく使うコマンド

```bash
# コンテナをビルドして起動
docker compose up -d --build

# コンテナに入る
docker compose exec ros2 bash

# 停止
docker compose down

# ボリュームごと削除（ビルドキャッシュをリセットしたいとき）
docker compose down -v
```

---

## VSCode devcontainerで開発する

「コンテナ内でVSCodeが動く」状態にすることで、ホストのVSCodeから直接ROS2コードを書いて実行できます。

### 必要なVSCode拡張機能

- **Dev Containers**（必須）
- **ROS**（ms-iot.vscode-ros）
- **Python**（ms-python.python）
- **C/C++**（実機開発する場合）

### .devcontainer/devcontainer.json

プロジェクトルートに `.devcontainer/` フォルダを作って設定ファイルを置きます。

```json
{
  "name": "ROS2 Humble Dev",
  "dockerComposeFile": "../docker-compose.yml",
  "service": "ros2",
  "workspaceFolder": "/ros2_ws",

  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python",
        "ms-python.pylance",
        "ms-iot.vscode-ros",
        "ms-vscode.cpptools",
        "eamodio.gitlens"
      ],
      "settings": {
        "python.defaultInterpreterPath": "/usr/bin/python3",
        "ros.distro": "humble",
        "editor.formatOnSave": true,
        "python.formatting.provider": "black"
      }
    }
  },

  "postCreateCommand": "cd /ros2_ws && colcon build --symlink-install 2>/dev/null || true",
  "remoteUser": "root"
}
```

### 使い方

1. VSCodeでプロジェクトフォルダを開く
2. 左下の「><」をクリック → 「Reopen in Container」
3. 初回はDockerイメージのビルドが走る（数分かかる）
4. 完了するとコンテナ内でVSCodeが動いている状態になる

コンテナ内のターミナルで：
```bash
# パッケージをビルド
colcon build --symlink-install

# ROS2コマンドが使える
source install/setup.bash
ros2 run my_package my_node
```

---

## MacでGUI（RViz2）を動かす

MacはLinuxと違い、Docker内のGUIをそのまま表示できません。XQuartzを使います。

### XQuartzのインストールと設定

```bash
# インストール
brew install --cask xquartz

# 再起動後、XQuartzを起動してから設定を変更
# XQuartz > 環境設定 > セキュリティ
# 「ネットワーク・クライアントからの接続を許可」にチェック
```

### 毎回必要な作業

```bash
# XQuartzを起動した後
xhost +localhost

# docker-compose.yml の environment に追加
# DISPLAY: host.docker.internal:0
```

コンテナ内で：

```bash
source /opt/ros/humble/setup.bash
rviz2  # ウィンドウが表示されれば成功
```

### Macでよくあるエラー

**`Authorization required, but no authorization protocol specified`**
```bash
xhost +localhost  # を実行してから再度試す
```

**`libGL error: No matching fbConfigs or visuals found`**
```bash
export LIBGL_ALWAYS_SOFTWARE=1
rviz2
```

---

## ビルドを速くするTips

```bash
# 特定のパッケージだけビルド
colcon build --symlink-install --packages-select my_package

# 並列ビルド数を指定（CPUコア数に合わせる）
colcon build --symlink-install --parallel-workers 4

# デバッグビルド（エラーが詳しくなる）
colcon build --symlink-install --cmake-args -DCMAKE_BUILD_TYPE=Debug

# ビルドキャッシュを使いつつ変更のあったパッケージだけ再ビルド
colcon build --symlink-install --packages-above my_package
```

---

## プロジェクト全体のディレクトリ構成

```
project/
├── .devcontainer/
│   └── devcontainer.json
├── src/
│   ├── my_robot/
│   │   ├── my_robot/
│   │   │   ├── __init__.py
│   │   │   └── node.py
│   │   ├── launch/
│   │   ├── config/
│   │   ├── package.xml
│   │   └── setup.py
│   └── my_interfaces/   # カスタムメッセージ定義
├── docker-compose.yml
└── Dockerfile
```

この構成にしておくと、`git clone` してから `docker compose up -d` と「Reopen in Container」だけで環境が整います。チームへの共有も簡単です。
