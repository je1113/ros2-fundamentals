# tf2 & 시간(Time/`use_sim_time`) — 충전소까지 거리 계산과 시뮬레이션 시계

[`00-syllabus.md`](./00-syllabus.md)의 11번째 주제. 고정된 충전소와 이동하는 로봇의 좌표계를 tf2로 관리해 실시간 거리를 계산하고, 가짜 `/clock`을 발행해 `use_sim_time`이 실제로 시간 흐름을 바꾸는 것을 확인한다.

---

## 학습 목표

- static/dynamic tf 브로드캐스터의 차이를 이해하고 구현할 수 있다.
- `tf2_ros.Buffer`/`TransformListener`로 두 좌표계 간 변환을 조회(lookup)하고 거리를 계산할 수 있다.
- `use_sim_time` 파라미터와 `/clock` 토픽이 노드의 시간 흐름을 어떻게 바꾸는지 이해한다.

## 핵심 개념

**tf2**는 로봇 각 부위의 좌표계(frame) 간 변환을 트리로 관리한다. `world` → `base_link`(로봇 위치, 계속 바뀜) → `battery_sensor`(로봇 기준 고정 오프셋)처럼, **고정된 것은 static**, **움직이는 것은 dynamic** 브로드캐스터로 나눠 발행한다.

- `StaticTransformBroadcaster`: `/tf_static`에 한 번만(래치드) 발행 — 안 변하는 관계(센서 장착 위치 등)
- `TransformBroadcaster`: `/tf`에 주기적으로 발행 — 계속 변하는 관계(로봇 위치 등)
- `Buffer` + `TransformListener`: 두 프레임 간 변환을 시간까지 고려해 조회(`lookup_transform`)

**시간**은 ROS 2에서 벽시계(wall clock) 대신 `/clock` 토픽 기반의 **시뮬레이션 시간**을 쓸 수 있다. 노드가 `use_sim_time` 파라미터를 `true`로 설정하면, `self.get_clock().now()`와 타이머가 실제 시간이 아니라 `/clock`이 발행하는 시간을 따라간다 — Isaac Sim 같은 시뮬레이터와 연동할 때 필수적인 개념이다.

## 실습 단계

### Part A: tf2

**1. Static 브로드캐스터** — `~/ros2_ws/src/ros2_basics/ros2_basics/static_frames.py`:

```python
import rclpy
from rclpy.node import Node
from tf2_ros import StaticTransformBroadcaster
from geometry_msgs.msg import TransformStamped


class StaticFrames(Node):
    def __init__(self):
        super().__init__('static_frames')
        self.broadcaster_ = StaticTransformBroadcaster(self)
        self.broadcast_charging_station()
        self.broadcast_battery_sensor()

    def broadcast_charging_station(self):
        t = TransformStamped()
        t.header.stamp = self.get_clock().now().to_msg()
        t.header.frame_id = 'world'
        t.child_frame_id = 'charging_station'
        t.transform.translation.x = 5.0
        t.transform.translation.y = 0.0
        t.transform.translation.z = 0.0
        t.transform.rotation.w = 1.0
        self.broadcaster_.sendTransform(t)

    def broadcast_battery_sensor(self):
        t = TransformStamped()
        t.header.stamp = self.get_clock().now().to_msg()
        t.header.frame_id = 'base_link'
        t.child_frame_id = 'battery_sensor'
        t.transform.translation.x = 0.1
        t.transform.translation.y = 0.0
        t.transform.translation.z = 0.05
        t.transform.rotation.w = 1.0
        self.broadcaster_.sendTransform(t)


def main():
    rclpy.init()
    node = StaticFrames()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

**2. Dynamic 브로드캐스터** — `~/ros2_ws/src/ros2_basics/ros2_basics/robot_mover.py`:

```python
import rclpy
from rclpy.node import Node
from tf2_ros import TransformBroadcaster
from geometry_msgs.msg import TransformStamped


class RobotMover(Node):
    def __init__(self):
        super().__init__('robot_mover')
        self.broadcaster_ = TransformBroadcaster(self)
        self.x_ = 0.0
        self.create_timer(0.5, self.timer_callback)

    def timer_callback(self):
        t = TransformStamped()
        t.header.stamp = self.get_clock().now().to_msg()
        t.header.frame_id = 'world'
        t.child_frame_id = 'base_link'
        t.transform.translation.x = self.x_
        t.transform.translation.y = 0.0
        t.transform.translation.z = 0.0
        t.transform.rotation.w = 1.0
        self.broadcaster_.sendTransform(t)
        self.get_logger().info(f'로봇 위치 x={self.x_:.1f}')

        self.x_ += 0.3
        if self.x_ > 6.0:
            self.x_ = 0.0


def main():
    rclpy.init()
    node = RobotMover()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

**3. tf2 리스너 — 거리 계산** — `~/ros2_ws/src/ros2_basics/ros2_basics/distance_to_charger.py`:

```python
import math
import rclpy
from rclpy.node import Node
from rclpy.time import Time
from tf2_ros import Buffer, TransformListener
from tf2_ros import LookupException, ConnectivityException, ExtrapolationException


class DistanceToCharger(Node):
    def __init__(self):
        super().__init__('distance_to_charger')
        self.tf_buffer_ = Buffer()
        self.tf_listener_ = TransformListener(self.tf_buffer_, self)
        self.create_timer(0.5, self.timer_callback)

    def timer_callback(self):
        try:
            t = self.tf_buffer_.lookup_transform('base_link', 'charging_station', Time())
        except (LookupException, ConnectivityException, ExtrapolationException) as e:
            self.get_logger().warn(f'tf lookup 실패: {e}')
            return

        dx = t.transform.translation.x
        dy = t.transform.translation.y
        distance = math.sqrt(dx * dx + dy * dy)
        if distance < 0.3:
            self.get_logger().info(f'충전소 도착! 거리={distance:.2f}m')
        else:
            self.get_logger().info(f'충전소까지 거리: {distance:.2f}m')


def main():
    rclpy.init()
    node = DistanceToCharger()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

**4. `package.xml`/`setup.py` 등록**

`~/ros2_ws/src/ros2_basics/package.xml`에서 `<depend>` 목록 마지막 줄 아래에 두 줄을 추가해서, 아래와 같이 되도록 한다:

```xml
  <depend>rclpy</depend>
  <depend>sensor_msgs</depend>
  <depend>ros2_basics_msgs</depend>
  <depend>launch</depend>
  <depend>launch_ros</depend>
  <depend>tf2_ros</depend>
  <depend>geometry_msgs</depend>
```

`setup.py`의 `entry_points`에 세 줄을 추가해서, `console_scripts` 리스트가 다음과 같이 되도록 한다:

```python
    entry_points={
        'console_scripts': [
            'hello_node = ros2_basics.hello_node:main',
            'battery_publisher = ros2_basics.battery_publisher:main',
            'battery_watcher = ros2_basics.battery_watcher:main',
            'fleet_monitor = ros2_basics.fleet_monitor:main',
            'request_return_client = ros2_basics.request_return_client:main',
            'charging_station_server = ros2_basics.charging_station_server:main',
            'return_to_base_client = ros2_basics.return_to_base_client:main',
            'auto_return_manager = ros2_basics.auto_return_manager:main',
            'combined_bringup = ros2_basics.combined_bringup:main',
            'static_frames = ros2_basics.static_frames:main',
            'robot_mover = ros2_basics.robot_mover:main',
            'distance_to_charger = ros2_basics.distance_to_charger:main',
        ],
    },
```

**5. 빌드 & 실행 (터미널 3개)**

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics
source install/setup.bash
```

터미널 A: `ros2 run ros2_basics static_frames`
터미널 B: `ros2 run ros2_basics robot_mover`
터미널 C: `ros2 run ros2_basics distance_to_charger`

(선택) `ros2 run tf2_ros tf2_echo world base_link`로 실제 tf 값도 확인 가능하다.

### Part B: 시간(Time)/`use_sim_time`

**1. 기본 동작 확인** — `~/ros2_ws/src/ros2_basics/ros2_basics/time_demo_node.py`:

```python
import rclpy
from rclpy.node import Node


class TimeDemoNode(Node):
    def __init__(self):
        super().__init__('time_demo_node')
        self.create_timer(1.0, self.tick)

    def tick(self):
        now = self.get_clock().now()
        self.get_logger().info(f'현재 시간: {now.nanoseconds / 1e9:.2f}s')


def main():
    rclpy.init()
    node = TimeDemoNode()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

```bash
ros2 run ros2_basics time_demo_node
```

**2. 가짜 시뮬레이션 시계(2배속)** — `~/ros2_ws/src/ros2_basics/ros2_basics/fake_clock_publisher.py`:

```python
import rclpy
from rclpy.node import Node
from rclpy.time import Time
from rosgraph_msgs.msg import Clock


class FakeClockPublisher(Node):
    def __init__(self):
        super().__init__('fake_clock_publisher')
        self.publisher_ = self.create_publisher(Clock, '/clock', 10)
        self.sim_time_ = 0.0
        self.create_timer(0.05, self.timer_callback)

    def timer_callback(self):
        self.sim_time_ += 0.1  # 실제 0.05s마다 시뮬레이션 0.1s 진행 (2배속)
        msg = Clock()
        msg.clock = Time(seconds=self.sim_time_).to_msg()
        self.publisher_.publish(msg)


def main():
    rclpy.init()
    node = FakeClockPublisher()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

**3. `package.xml`/`setup.py` 등록**

`package.xml`에서 `<depend>tf2_ros</depend>` 아래에 한 줄을 추가해서, 아래와 같이 되도록 한다:

```xml
  <depend>tf2_ros</depend>
  <depend>geometry_msgs</depend>
  <depend>rosgraph_msgs</depend>
```

`setup.py`의 `entry_points`에 두 줄을 추가해서, `console_scripts` 리스트가 다음과 같이 되도록 한다:

```python
    entry_points={
        'console_scripts': [
            'hello_node = ros2_basics.hello_node:main',
            'battery_publisher = ros2_basics.battery_publisher:main',
            'battery_watcher = ros2_basics.battery_watcher:main',
            'fleet_monitor = ros2_basics.fleet_monitor:main',
            'request_return_client = ros2_basics.request_return_client:main',
            'charging_station_server = ros2_basics.charging_station_server:main',
            'return_to_base_client = ros2_basics.return_to_base_client:main',
            'auto_return_manager = ros2_basics.auto_return_manager:main',
            'combined_bringup = ros2_basics.combined_bringup:main',
            'static_frames = ros2_basics.static_frames:main',
            'robot_mover = ros2_basics.robot_mover:main',
            'distance_to_charger = ros2_basics.distance_to_charger:main',
            'time_demo_node = ros2_basics.time_demo_node:main',
            'fake_clock_publisher = ros2_basics.fake_clock_publisher:main',
        ],
    },
```

**4. 빌드 & 실행 (터미널 2개)**

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics
source install/setup.bash
```

터미널 A: `ros2 run ros2_basics fake_clock_publisher`
터미널 B:
```bash
ros2 run ros2_basics time_demo_node --ros-args -p use_sim_time:=true
```

## 예상/실제 결과

- Part A: `distance_to_charger`의 로그에서 거리가 점점 줄어들다가 `충전소 도착!`이 찍히고, 로봇 위치가 리셋되면서 다시 거리가 멀어짐.
- Part B: `use_sim_time` 없이 실행하면 로그 값이 실제 경과 시간과 비슷하게 증가하지만, `fake_clock_publisher`가 발행하는 `/clock`을 따라가도록 `use_sim_time:=true`로 실행하면 로그 값이 실제 경과 시간보다 약 2배 빠르게 증가함.

실제로 두 파트 모두 확인했다.

## 알려진 문제와 해결

이번 실습에서는 별도로 발생한 문제 없음.

## 체크포인트

- [ ] static/dynamic tf 브로드캐스터를 구분해서 구현할 수 있다.
- [ ] `tf2_ros.Buffer`/`TransformListener`로 두 프레임 간 변환을 조회하고 활용할 수 있다.
- [ ] `use_sim_time`과 `/clock`이 노드의 시간 흐름에 미치는 영향을 직접 확인했다.

---
다음: [`12-rosbag-debugging.md`](./12-rosbag-debugging.md)
