# QoS (Quality of Service) — 조용히 실패하는 통신 디버깅하기

[`00-syllabus.md`](./00-syllabus.md)의 3번째 주제. `battery_publisher`/`battery_watcher` 노드의 QoS를 일부러 불일치시켜 "코드는 맞는데 메시지가 안 오는" 상황을 재현하고, 원인을 확인한 뒤 해결한다.

> 이 문서는 [`02-node-pub-sub.md`](./02-node-pub-sub.md)에서 만든 `battery_publisher.py`/`battery_watcher.py`를 이어서 수정한다. 아래 코드는 **파일 전체 내용**이므로, 각 파일을 통째로 이 내용으로 덮어써도 된다.

---

## 학습 목표

- ROS 2 QoS의 핵심 정책(Reliability, Durability, History)을 이해한다.
- Publisher/Subscriber QoS 불일치가 어떻게 "에러 없이 조용히" 통신 실패로 이어지는지 직접 겪어본다.
- `ros2 topic info --verbose`로 QoS 불일치를 진단하는 법을 익힌다.

## 핵심 개념

ROS 2는 DDS 기반이라 토픽마다 **QoS 프로파일**을 설정할 수 있다. Publisher와 Subscriber의 QoS가 호환되지 않으면 연결 자체가 성립하지 않아 메시지가 전혀 오지 않는다 — 이것이 입문자가 가장 많이 겪는 "코드는 맞는데 왜 안 되지" 버그의 원인이다.

핵심 QoS 정책:

| 정책 | 옵션 | 의미 |
|---|---|---|
| **Reliability** | `RELIABLE` / `BEST_EFFORT` | RELIABLE: 유실 시 재전송 보장. BEST_EFFORT: 유실 허용(센서 데이터처럼 최신값이 중요할 때) |
| **Durability** | `VOLATILE` / `TRANSIENT_LOCAL` | VOLATILE: 구독 시작 이후 메시지만 받음. TRANSIENT_LOCAL: 늦게 구독해도 마지막 발행분을 받음(래치드 토픽) |
| **History** | `KEEP_LAST(depth)` / `KEEP_ALL` | 큐에 몇 개까지 보관할지 |

**호환 규칙**: Subscriber의 요구 수준이 Publisher의 제공 수준보다 낮거나 같아야 한다. Publisher가 `BEST_EFFORT`인데 Subscriber가 `RELIABLE`을 요구하면 매칭되지 않아 메시지를 전혀 받지 못한다 — **에러 메시지도 없이 조용히 안 오기 때문에 QoS를 모르면 원인을 찾기 매우 어렵다.**

실무 관례: 센서 데이터(카메라, 라이다, IMU 등)는 최신값이 중요하고 약간의 유실은 괜찮으므로 `qos_profile_sensor_data`(BEST_EFFORT, depth 5)를 쓰고, 배터리 상태나 명령처럼 유실되면 안 되는 데이터는 기본값인 `RELIABLE`을 쓴다.

## 실습 단계 — 일부러 QoS 불일치를 재현해보기

### 1. Subscriber에 `RELIABLE` QoS 명시

`~/ros2_ws/src/ros2_basics/ros2_basics/battery_watcher.py` 전체를 아래 내용으로 바꾼다 (2번 주제 버전에서 `rclpy.qos` import와 `qos` 변수만 추가됐다):

```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import BatteryState
from rclpy.qos import QoSProfile, ReliabilityPolicy

LOW_BATTERY_THRESHOLD = 0.20
CRITICAL_BATTERY_THRESHOLD = 0.05


class BatteryWatcher(Node):
    def __init__(self):
        super().__init__('battery_watcher')
        qos = QoSProfile(depth=10, reliability=ReliabilityPolicy.RELIABLE)
        self.subscription_ = self.create_subscription(
            BatteryState, 'battery_state', self.battery_callback, qos)

    def battery_callback(self, msg: BatteryState):
        pct = msg.percentage
        if pct <= CRITICAL_BATTERY_THRESHOLD:
            self.get_logger().error(f'배터리 위험 수준: {pct * 100:.0f}%')
        elif pct <= LOW_BATTERY_THRESHOLD:
            self.get_logger().warn(f'배터리 부족: {pct * 100:.0f}%')
        else:
            self.get_logger().info(f'배터리 정상: {pct * 100:.0f}%')


def main():
    rclpy.init()
    node = BatteryWatcher()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

### 2. Publisher에는 일부러 `BEST_EFFORT` QoS 설정

`~/ros2_ws/src/ros2_basics/ros2_basics/battery_publisher.py` 전체를 아래 내용으로 바꾼다:

```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import BatteryState
from rclpy.qos import QoSProfile, ReliabilityPolicy


class BatteryPublisher(Node):
    def __init__(self):
        super().__init__('battery_publisher')
        qos = QoSProfile(depth=10, reliability=ReliabilityPolicy.BEST_EFFORT)
        self.publisher_ = self.create_publisher(BatteryState, 'battery_state', qos)
        self.timer_ = self.create_timer(1.0, self.timer_callback)
        self.percentage_ = 1.0

    def timer_callback(self):
        msg = BatteryState()
        msg.header.stamp = self.get_clock().now().to_msg()
        msg.voltage = 12.6 * self.percentage_ + 9.0
        msg.percentage = self.percentage_
        msg.present = True
        msg.power_supply_status = BatteryState.POWER_SUPPLY_STATUS_DISCHARGING

        self.publisher_.publish(msg)
        self.get_logger().info(
            f'Battery: {self.percentage_ * 100:.0f}% ({msg.voltage:.2f}V)')

        self.percentage_ = max(0.0, self.percentage_ - 0.05)
        if self.percentage_ <= 0.0:
            self.percentage_ = 1.0


def main():
    rclpy.init()
    node = BatteryPublisher()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

### 3. 빌드 후 각각 실행 — 불일치 재현

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics
source install/setup.bash
```

터미널 A: `ros2 run ros2_basics battery_publisher`
터미널 B: `ros2 run ros2_basics battery_watcher`

### 4. `ros2 topic info`로 불일치 확인

```bash
ros2 topic info /battery_state --verbose
```

Publisher와 Subscriber 항목 각각에 `Reliability` 값이 다르게(`BEST_EFFORT` vs `RELIABLE`) 표시된다.

### 5. 해결 — Subscriber를 Publisher와 동일하게 맞춤

`battery_watcher.py`에서 **`qos = QoSProfile(...)` 한 줄만** 아래처럼 바꾼다 (나머지 코드는 1단계와 동일하다):

```python
        qos = QoSProfile(depth=10, reliability=ReliabilityPolicy.BEST_EFFORT)
```

```bash
colcon build --symlink-install --packages-select ros2_basics
source install/setup.bash
ros2 run ros2_basics battery_watcher   # publisher는 계속 켜둔 채로
```

## 예상/실제 결과

- 3단계(불일치 상태): Publisher는 로그를 계속 찍지만 **Subscriber는 아무 로그도 찍지 않음** — 에러 없이 조용히 실패.
- 4단계: `ros2 topic info --verbose`에서 두 엔드포인트의 Reliability 값이 서로 다르게 표시됨을 확인.
- 5단계(해결 후): Subscriber QoS를 Publisher와 동일하게(`BEST_EFFORT`) 맞추자 정상적으로 로그가 다시 찍힘.

실제로 위 순서 그대로 재현했고, QoS를 맞춘 뒤 정상 동작을 확인했다.

## 알려진 문제와 해결

| 문제 | 원인 | 해결 |
|---|---|---|
| Subscriber에서 메시지를 전혀 못 받음 (에러 없음) | Publisher(`BEST_EFFORT`)와 Subscriber(`RELIABLE`)의 Reliability QoS 불일치 | 둘의 QoS를 동일한 정책으로 맞춤 (여기서는 `BEST_EFFORT`로 통일) |

## 체크포인트

- [ ] Reliability/Durability/History 각각의 의미를 설명할 수 있다.
- [ ] QoS 불일치가 에러 없이 "조용한 실패"로 나타난다는 것을 직접 겪어봤다.
- [ ] `ros2 topic info --verbose`로 QoS 불일치를 진단할 수 있다.
- [ ] 센서 데이터와 신뢰성이 중요한 데이터에 각각 어떤 QoS를 쓰는지 설명할 수 있다.

---
다음: [`04-custom-msg.md`](./04-custom-msg.md)
