---
title: "ROS2 Pythonパッケージ開発のベストプラクティス【colcon・launch・パラメーター】"
emoji: "🐍"
type: "tech"
topics: ["ROS2", "Python", "colcon", "ロボット"]
published: true
---

ROS2のPythonパッケージを作るとき、`setup.py` の書き方や `launch` ファイルの構成で詰まることが多い。実際の開発でよく使うパターンをまとめました。

---

## パッケージ作成

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python my_robot \
  --dependencies rclpy std_msgs geometry_msgs nav_msgs sensor_msgs
```

---

## ディレクトリ構成

```
my_robot/
├── my_robot/
│   ├── __init__.py
│   ├── robot_node.py
│   └── utils/
│       ├── __init__.py
│       └── geometry.py
├── launch/
│   ├── robot.launch.py
│   └── sim.launch.py
├── config/
│   └── robot_params.yaml
├── package.xml
├── setup.py
└── setup.cfg
```

---

## setup.py（よく間違えるポイント）

```python
from setuptools import setup
import os
from glob import glob

package_name = 'my_robot'

setup(
    name=package_name,
    version='0.1.0',
    packages=[package_name, f'{package_name}.utils'],  # サブパッケージも忘れずに
    data_files=[
        ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
        # launchとconfigを忘れると実行時にファイルが見つからない
        (os.path.join('share', package_name, 'launch'),
            glob('launch/*.py')),
        (os.path.join('share', package_name, 'config'),
            glob('config/*.yaml')),
    ],
    install_requires=['setuptools'],
    entry_points={
        'console_scripts': [
            'robot_node = my_robot.robot_node:main',
        ],
    },
)
```

---

## yamlパラメーター

```yaml
# config/robot_params.yaml
robot_node:
  ros__parameters:          # ← アンダースコア2つ！1つだと読まれない
    max_speed: 0.3
    safety_radius: 0.5
    llm_model: "claude-haiku-4-5-20251001"
```

```python
class RobotNode(Node):
    def __init__(self):
        super().__init__('robot_node')
        self.declare_parameter('max_speed', 0.3)
        self.declare_parameter('safety_radius', 0.5)
        self.max_speed = self.get_parameter('max_speed').value
```

---

## launch file

```python
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument
from launch.substitutions import LaunchConfiguration, PathJoinSubstitution
from launch_ros.actions import Node
from launch_ros.substitutions import FindPackageShare

def generate_launch_description():
    config_file = PathJoinSubstitution([
        FindPackageShare('my_robot'), 'config', 'robot_params.yaml'
    ])

    robot_node = Node(
        package='my_robot',
        executable='robot_node',
        output='screen',
        parameters=[config_file],
        remappings=[('cmd_vel', '/cmd_vel')]
    )

    return LaunchDescription([
        DeclareLaunchArgument('use_sim_time', default_value='false'),
        robot_node,
    ])
```

```bash
ros2 launch my_robot robot.launch.py
ros2 launch my_robot robot.launch.py use_sim_time:=true
```

---

## ビルドTips

```bash
# --symlink-installでPythonファイルの編集を即反映
colcon build --symlink-install

# 特定パッケージだけビルド
colcon build --symlink-install --packages-select my_robot

# ビルド後は必ずsource
source install/setup.bash
```

`--symlink-install` をつけるとPythonファイルへのシンボリックリンクが作られ、ファイル編集のたびに再ビルド不要になります（C++は再ビルド必要）。
