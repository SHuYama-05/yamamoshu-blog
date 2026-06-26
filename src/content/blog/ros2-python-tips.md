---
title: 'ROS2 Pythonパッケージ開発のベストプラクティス【colcon対応】'
description: 'ROS2のPythonパッケージを正しく作る方法を解説。colconでのビルド・パッケージ構成・launch fileの書き方まで実例付きで説明します。'
pubDate: '2026-06-08'
tags: ['ROS2', 'Python', 'colcon', 'パッケージ']
---

ROS2のPythonパッケージを初めて作るとき、ディレクトリ構成やsetup.pyの書き方で詰まることが多いです。

## パッケージ作成

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python my_package \
  --dependencies rclpy std_msgs geometry_msgs
```

## ディレクトリ構成

```
my_package/
├── my_package/
│   ├── __init__.py
│   ├── my_node.py
│   └── utils.py
├── launch/
│   └── my_launch.py
├── config/
│   └── params.yaml
├── package.xml
├── setup.py
└── setup.cfg
```

## setup.py

```python
from setuptools import setup
import os
from glob import glob

package_name = 'my_package'

setup(
    name=package_name,
    version='0.1.0',
    packages=[package_name],
    data_files=[
        ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
        (os.path.join('share', package_name, 'launch'),
            glob('launch/*.py')),
        (os.path.join('share', package_name, 'config'),
            glob('config/*.yaml')),
    ],
    install_requires=['setuptools'],
    entry_points={
        'console_scripts': [
            'my_node = my_package.my_node:main',
        ],
    },
)
```

## launch file

```python
from launch import LaunchDescription
from launch_ros.actions import Node
from launch.substitutions import LaunchConfiguration
from launch.actions import DeclareLaunchArgument

def generate_launch_description():
    return LaunchDescription([
        DeclareLaunchArgument('use_sim_time', default_value='false'),
        Node(
            package='my_package',
            executable='my_node',
            name='my_node',
            parameters=[{
                'use_sim_time': LaunchConfiguration('use_sim_time'),
            }],
            output='screen',
        ),
    ])
```

## yamlパラメーターファイル

```yaml
my_node:
  ros__parameters:
    speed: 0.5
    max_range: 10.0
    target_frame: 'map'
```

## ビルドと実行

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select my_package
source install/setup.bash
ros2 launch my_package my_launch.py
```

`--symlink-install` をつけるとPythonファイルを編集するたびにビルドし直す必要がなくなります。
