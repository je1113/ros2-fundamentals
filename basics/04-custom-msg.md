# 커스텀 msg 정의 — 배터리 경보를 플릿 단위로 모으기

[`00-syllabus.md`](./00-syllabus.md)의 4번째 주제. 표준 메시지로 표현되지 않는 도메인 데이터(로봇 ID + 배터리 상태 + 경보 등급)를 커스텀 메시지로 정의하고, 여러 로봇의 경보를 한 곳에서 모으는 "플릿 모니터링" 시나리오를 실습한다.

---

## 학습 목표

- 커스텀 인터페이스(`.msg`)를 별도의 `ament_cmake` 패키지에 정의하고 빌드할 수 있다.
- 기존 노드가 다른 패키지의 커스텀 메시지를 의존성으로 가져와 쓰는 흐름을 이해한다.
- 하나의 노드가 구독자이자 동시에 발행자(subscribe → 가공 → publish)로 동작하는 파이프라인 패턴을 만들어본다.

## 핵심 개념

표준 메시지(`std_msgs`, `sensor_msgs` 등)로 표현할 수 없는 도메인 데이터가 필요할 때 **커스텀 인터페이스**를 만든다. 메시지 정의(`.msg`)는 파이썬 코드가 아니라 **별도의 `ament_cmake` 패키지**에 두고, `rosidl`이 빌드 시점에 파이썬/C++ 코드를 자동 생성한다 — 그래서 관례상 `xxx_msgs`라는 이름의 인터페이스 전용 패키지를 분리한다.

이번 실습에서는 `BatteryAlert` 메시지(`robot_id`, `percentage`, `level` + `LEVEL_OK`/`LEVEL_LOW`/`LEVEL_CRITICAL` 상수)를 만들어, `battery_watcher`가 `battery_state`를 구독해 경보로 가공한 뒤 `battery_alert` 토픽으로 재발행하고, `fleet_monitor`가 그걸 모아서 보여주는 구조를 만든다.

```
battery_publisher --(battery_state)--> battery_watcher --(battery_alert)--> fleet_monitor
```

> ⚠️ **conda 주의**: `.msg` 빌드는 `ament_cmake`/`rosidl`(CMake 기반)을 쓴다. `(base)` conda 환경이 켜져 있으면 CMake가 `CONDA_PREFIX`를 우선 참조해 시스템 Python 대신 conda Python을 골라 빌드가 실패할 수 있다. 커스텀 메시지 패키지를 빌드하기 전에는 `conda deactivate`를 먼저 실행한다.

## 실습 단계

### 1. conda 비활성화

```bash
conda deactivate
```

### 2. 인터페이스 패키지 생성

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_cmake ros2_basics_msgs
```

### 3. 메시지 정의

`~/ros2_ws/src/ros2_basics_msgs/msg/BatteryAlert.msg`:

```
string robot_id
float32 percentage
uint8 level

uint8 LEVEL_OK=0
uint8 LEVEL_LOW=1
uint8 LEVEL_CRITICAL=2
```

### 4. `CMakeLists.txt` 수정

`ros2 pkg create --build-type ament_cmake`가 만든 `~/ros2_ws/src/ros2_basics_msgs/CMakeLists.txt`에서 `find_package(ament_cmake REQUIRED)` 아래에 두 블록을 추가해서, 파일 앞부분이 다음과 같이 되도록 한다:

```cmake
cmake_minimum_required(VERSION 3.8)
project(ros2_basics_msgs)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# find dependencies
find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/BatteryAlert.msg"
)

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  ...
```

(이하 `if(BUILD_TESTING)` 블록과 `ament_package()`는 자동 생성된 그대로 둔다.)

### 5. `package.xml` 수정 (`ros2_basics_msgs`)

`~/ros2_ws/src/ros2_basics_msgs/package.xml`에서 `<buildtool_depend>ament_cmake</buildtool_depend>` 아래에 세 줄을 추가해서, 파일 전체가 다음과 같이 되도록 한다:

```xml
<?xml version="1.0"?>
<?xml-model href="http://download.ros.org/schema/package_format3.xsd" schematypens="http://www.w3.org/2001/XMLSchema"?>
<package format="3">
  <name>ros2_basics_msgs</name>
  <version>0.0.0</version>
  <description>TODO: Package description</description>
  <maintainer email="본인 이메일">본인 이름</maintainer>
  <license>TODO: License declaration</license>

  <buildtool_depend>ament_cmake</buildtool_depend>
  <build_depend>rosidl_default_generators</build_depend>
  <exec_depend>rosidl_default_runtime</exec_depend>
  <member_of_group>rosidl_interface_packages</member_of_group>
  <test_depend>ament_lint_auto</test_depend>
  <test_depend>ament_lint_common</test_depend>

  <export>
    <build_type>ament_cmake</build_type>
  </export>
</package>
```

### 6. 빌드 & 확인

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics_msgs
source install/setup.bash
ros2 interface show ros2_basics_msgs/msg/BatteryAlert
```

### 7. `ros2_basics`가 이 메시지를 쓰도록 의존성 추가

`~/ros2_ws/src/ros2_basics/package.xml`에서 기존 `<depend>` 목록 마지막 줄 아래에 추가해서, `<depend>` 목록이 다음과 같이 되도록 한다 (파일의 다른 부분은 그대로 둔다):

```xml
  <depend>rclpy</depend>
  <depend>sensor_msgs</depend>
  <depend>ros2_basics_msgs</depend>
```

### 8. `battery_watcher.py` — 구독 + 가공 + 재발행

```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import BatteryState
from ros2_basics_msgs.msg import BatteryAlert
from rclpy.qos import QoSProfile, ReliabilityPolicy

LOW_BATTERY_THRESHOLD = 0.20
CRITICAL_BATTERY_THRESHOLD = 0.05


class BatteryWatcher(Node):
    def __init__(self):
        super().__init__('battery_watcher')
        qos = QoSProfile(depth=10, reliability=ReliabilityPolicy.BEST_EFFORT)
        self.subscription_ = self.create_subscription(
            BatteryState, 'battery_state', self.battery_callback, qos)
        self.alert_publisher_ = self.create_publisher(BatteryAlert, 'battery_alert', 10)

    def battery_callback(self, msg: BatteryState):
        pct = msg.percentage
        alert = BatteryAlert()
        alert.robot_id = 'robot_1'
        alert.percentage = pct
        if pct <= CRITICAL_BATTERY_THRESHOLD:
            alert.level = BatteryAlert.LEVEL_CRITICAL
            self.get_logger().error(f'배터리 위험 수준: {pct * 100:.0f}%')
        elif pct <= LOW_BATTERY_THRESHOLD:
            alert.level = BatteryAlert.LEVEL_LOW
            self.get_logger().warn(f'배터리 부족: {pct * 100:.0f}%')
        else:
            alert.level = BatteryAlert.LEVEL_OK
            self.get_logger().info(f'배터리 정상: {pct * 100:.0f}%')
        self.alert_publisher_.publish(alert)


def main():
    rclpy.init()
    node = BatteryWatcher()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

### 9. `fleet_monitor.py` — 여러 로봇의 경보를 모으는 노드

```python
import rclpy
from rclpy.node import Node
from ros2_basics_msgs.msg import BatteryAlert


class FleetMonitor(Node):
    def __init__(self):
        super().__init__('fleet_monitor')
        self.create_subscription(BatteryAlert, 'battery_alert', self.alert_callback, 10)

    def alert_callback(self, msg: BatteryAlert):
        level_str = {0: 'OK', 1: 'LOW', 2: 'CRITICAL'}[msg.level]
        self.get_logger().info(
            f'[Fleet] {msg.robot_id}: {msg.percentage * 100:.0f}% ({level_str})')


def main():
    rclpy.init()
    node = FleetMonitor()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

### 10. `setup.py`에 entry_point 추가

`setup.py`의 `entry_points`에 한 줄을 추가해서, `console_scripts` 리스트가 다음과 같이 되도록 한다:

```python
    entry_points={
        'console_scripts': [
            'hello_node = ros2_basics.hello_node:main',
            'battery_publisher = ros2_basics.battery_publisher:main',
            'battery_watcher = ros2_basics.battery_watcher:main',
            'fleet_monitor = ros2_basics.fleet_monitor:main',
        ],
    },
```

### 11. 전체 빌드 & 실행 (터미널 3개)

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics_msgs ros2_basics
source install/setup.bash
```

터미널 A: `ros2 run ros2_basics battery_publisher`
터미널 B: `ros2 run ros2_basics battery_watcher`
터미널 C: `ros2 run ros2_basics fleet_monitor`

## 예상/실제 결과

`fleet_monitor`에서 `[Fleet] robot_1: XX% (OK/LOW/CRITICAL)` 로그가, `battery_watcher`에서는 기존 정상/부족/위험 로그가 그대로 찍힌다. 실제로 세 노드를 모두 실행해 파이프라인 끝까지(publisher → watcher → fleet_monitor) 메시지가 전달되는 것을 확인했다.

## 알려진 문제와 해결

실습 중 `battery_watcher.py`를 손으로 옮겨 적다가 아래 3가지가 한 번에 겹치는 버그가 발생했다 — 겉보기엔 "메시지가 조용히 안 온다"는 QoS 불일치(3번 주제)와 똑같은 증상이라 헷갈리기 쉬웠다.

| 증상 | 실제 원인 | 구분 포인트 |
|---|---|---|
| `fleet_monitor` 실행 시 `ModuleNotFoundError: No module named 'ros2_basics_msg'` | import 문에서 패키지 이름 오타(`ros2_basics_msg` vs `ros2_basics_msgs`) | **Python 트레이스백이 뜬다** — `rclpy.init()` 전 단계에서 나는 에러라 QoS/토픽 문제와는 무관 |
| `battery_watcher`도 `fleet_monitor`도 로그가 전혀 안 찍힘 (에러 없음) | `battery_watcher`의 구독이 원래 구독해야 할 `BatteryState`/`'battery_state'`가 아니라, 자신이 발행해야 할 `BatteryAlert`/`'battery_alert'`를 구독하고 있었음. 게다가 `alert_publisher_` 생성과 `publish()` 호출 자체가 빠져 있어 애초에 아무도 `'battery_alert'`를 발행하지 않는 상태였음 | 에러가 없는 "조용한 실패"라 QoS 문제로 오인하기 쉽지만, **토픽 이름/타입 자체가 잘못 연결**된 구조적 버그였음. 노드가 정상 실행되고 QoS도 맞는데 콜백만 안 불린다면, 먼저 `ros2 topic info <토픽> --verbose`로 Pub/Sub가 애초에 그 토픽에 연결되어 있는지부터 확인한다 |

## 체크포인트

- [ ] 커스텀 메시지가 별도의 `ament_cmake`/`rosidl` 패키지로 분리되는 이유를 설명할 수 있다.
- [ ] `.msg` 파일 문법(필드, 상수)을 작성할 수 있다.
- [ ] 한 노드가 구독 → 가공 → 재발행하는 파이프라인을 구현할 수 있다.
- [ ] "조용한 실패"의 원인이 QoS 불일치인지, 토픽/타입 연결 오류인지 구분해서 진단할 수 있다.

---
다음: [`05-service.md`](./05-service.md)
