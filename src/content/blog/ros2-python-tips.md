---
title: 'ROS2 Pythonパッケージ開発のベストプラクティス【colcon・launch・パラメーター】'
description: 'ROS2のPythonパッケージを正しく作る方法を解説。colconでのビルド・パッケージ構成・launch fileの書き方・yamlパラメーター管理まで実例付きで説明します。'
pubDate: '2026-06-08'
tags: ['ROS2', 'Python', 'colcon', 'パッケージ']
---

ROS2のPythonパッケージを初めて作るとき、`setup.py` の書き方・`package.xml` の依存関係・`launch` ファイルの書き方など、覚えることが多くて混乱します。

実際の開発でよく使うパターンをまとめました。

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
│   ├── robot_node.py        # メインノード
│   ├── llm_controller.py    # LLM制御クラス
│   └── utils/
│       ├── __init__.py
│       └── geometry.py      # 座標計算などのユーティリティ
├── launch/
│   ├── robot.launch.py      # 本番起動用
│   └── sim.launch.py        # シミュレーター起動用
├── config/
│   ├── robot_params.yaml    # ロボットパラメーター
│   └── nav2_params.yaml     # ナビゲーションパラメーター
├── test/
│   └── test_geometry.py
├── package.xml
├── setup.py
└── setup.cfg
```

---

## setup.py

`data_files` の書き方を間違えると `launch` ファイルや `config` が見つからないエラーになります。

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
        # ament インデックスへの登録（必須）
        ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
        # launch ファイル
        (os.path.join('share', package_name, 'launch'),
            glob('launch/*.py')),
        # config ファイル
        (os.path.join('share', package_name, 'config'),
            glob('config/*.yaml')),
    ],
    install_requires=['setuptools'],
    zip_safe=True,
    entry_points={
        'console_scripts': [
            # 'コマンド名 = パッケージ.モジュール:関数名'
            'robot_node = my_robot.robot_node:main',
            'llm_controller = my_robot.llm_controller:main',
        ],
    },
)
```

## package.xml

```xml
<?xml version="1.0"?>
<package format="3">
  <name>my_robot</name>
  <version>0.1.0</version>
  <description>My robot package</description>
  <maintainer email="your@email.com">your name</maintainer>
  <license>MIT</license>

  <depend>rclpy</depend>
  <depend>std_msgs</depend>
  <depend>geometry_msgs</depend>
  <depend>nav_msgs</depend>
  <depend>sensor_msgs</depend>
  <depend>tf2_ros</depend>
  <depend>tf2_geometry_msgs</depend>

  <test_depend>ament_copyright</test_depend>
  <test_depend>ament_flake8</test_depend>
  <test_depend>ament_pep257</test_depend>
  <test_depend>pytest</test_depend>

  <export>
    <build_type>ament_python</build_type>
  </export>
</package>
```

---

## ノードの書き方（推奨パターン）

```python
import rclpy
from rclpy.node import Node
from rclpy.parameter import Parameter
from geometry_msgs.msg import Twist
from sensor_msgs.msg import LaserScan

class RobotNode(Node):
    def __init__(self):
        super().__init__('robot_node')

        # パラメーターを宣言（デフォルト値付き）
        self.declare_parameter('max_speed', 0.3)
        self.declare_parameter('safety_radius', 0.5)
        self.declare_parameter('use_sim_time', False)

        # パラメーターを取得
        self.max_speed = self.get_parameter('max_speed').value
        self.safety_radius = self.get_parameter('safety_radius').value

        self.get_logger().info(
            f'起動: max_speed={self.max_speed}, safety_radius={self.safety_radius}'
        )

        # パブリッシャー・サブスクライバー
        self.cmd_pub = self.create_publisher(Twist, 'cmd_vel', 10)
        self.scan_sub = self.create_subscription(
            LaserScan, 'scan', self.scan_callback, 10
        )

        # タイマー（10Hz）
        self.timer = self.create_timer(0.1, self.control_loop)

    def scan_callback(self, msg: LaserScan):
        # LaserScanを処理
        ranges = [r for r in msg.ranges if not (r == float('inf') or r != r)]
        if ranges:
            self.min_distance = min(ranges)

    def control_loop(self):
        cmd = Twist()
        if hasattr(self, 'min_distance') and self.min_distance < self.safety_radius:
            # 障害物が近い → 停止
            self.get_logger().warn(f'障害物検出: {self.min_distance:.2f}m')
        else:
            cmd.linear.x = self.max_speed
        self.cmd_pub.publish(cmd)


def main(args=None):
    rclpy.init(args=args)
    node = RobotNode()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()


if __name__ == '__main__':
    main()
```

---

## launch ファイル

```python
from launch import LaunchDescription
from launch.actions import (
    DeclareLaunchArgument,
    IncludeLaunchDescription,
    LogInfo
)
from launch.conditions import IfCondition
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import LaunchConfiguration, PathJoinSubstitution
from launch_ros.actions import Node
from launch_ros.substitutions import FindPackageShare

def generate_launch_description():
    # 引数の宣言
    use_sim_time_arg = DeclareLaunchArgument(
        'use_sim_time',
        default_value='false',
        description='シミュレーター時刻を使うか'
    )
    use_rviz_arg = DeclareLaunchArgument(
        'use_rviz',
        default_value='true',
        description='RViz2を起動するか'
    )

    use_sim_time = LaunchConfiguration('use_sim_time')
    use_rviz = LaunchConfiguration('use_rviz')

    # configファイルのパスを解決
    config_file = PathJoinSubstitution([
        FindPackageShare('my_robot'),
        'config',
        'robot_params.yaml'
    ])

    # メインノード
    robot_node = Node(
        package='my_robot',
        executable='robot_node',
        name='robot_node',
        output='screen',
        parameters=[
            config_file,
            {'use_sim_time': use_sim_time}
        ],
        remappings=[
            ('cmd_vel', '/cmd_vel'),
            ('scan', '/scan'),
        ]
    )

    # RViz2（条件付き起動）
    rviz_node = Node(
        package='rviz2',
        executable='rviz2',
        condition=IfCondition(use_rviz),
        output='screen'
    )

    return LaunchDescription([
        use_sim_time_arg,
        use_rviz_arg,
        LogInfo(msg='ロボット起動'),
        robot_node,
        rviz_node,
    ])
```

```bash
# 起動
ros2 launch my_robot robot.launch.py

# 引数を渡す
ros2 launch my_robot robot.launch.py use_sim_time:=true use_rviz:=false
```

---

## yaml パラメーターファイル

```yaml
# config/robot_params.yaml
robot_node:
  ros__parameters:
    max_speed: 0.3          # m/s
    safety_radius: 0.5      # m
    use_sim_time: false
    llm_model: "claude-haiku-4-5-20251001"
    llm_timeout: 5.0        # 秒
```

**注意**: `ros__parameters` の `__` はアンダースコア2つです。1つだと読み込まれません（よく間違える）。

---

## ビルドTips

```bash
cd ~/ros2_ws

# 全パッケージをビルド
colcon build --symlink-install

# 特定パッケージだけ
colcon build --symlink-install --packages-select my_robot

# ビルド後は必ずsource
source install/setup.bash

# パッケージが見つからないとき
colcon build --symlink-install && source install/setup.bash
ros2 pkg list | grep my_robot
```

`--symlink-install` をつけるとPythonファイルへのシンボリックリンクが作られ、ファイルを編集するたびに再ビルドしなくて済みます（C++ノードはビルドが必要）。
