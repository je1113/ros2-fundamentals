# 노드 & Publisher/Subscriber — 배터리 상태로 배워보는 pub/sub

[`00-syllabus.md`](./00-syllabus.md)의 2번째 주제. `ros2_basics` 패키지에 배터리 상태를 흉내내는 Publisher/Subscriber 노드 한 쌍을 만들어 ROS 2 통신의 기본 단위인 토픽 pub/sub를 실습한다.

---

## 학습 목표

- 토픽(topic) 기반 pub/sub 통신 모델을 이해한다.
- `create_publisher` / `create_subscription`으로 노드 간 메시지를 주고받을 수 있다.
- 표준 메시지 타입(`sensor_msgs/msg/BatteryState`)을 실제 상황(배터리 부족 감시)에 적용해본다.

## 핵심 개념

ROS 2 통신의 기본은 **토픽** 기반 pub/sub이다. **Publisher**는 특정 토픽 이름으로 메시지를 발행하고, **Subscriber**는 같은 토픽을 구독해 콜백으로 메시지를 받는다. 발행자와 구독자는 서로의 존재를 몰라도 되며(N:M 관계 가능), ROS 2 미들웨어(DDS)가 노드 디스커버리와 메시지 전달을 담당한다.

핵심 요소:

- **토픽 이름과 메시지 타입이 일치**해야 통신된다 (예: `sensor_msgs/msg/BatteryState`, 토픽명 `battery_state`).
- `create_publisher(msg_type, topic_name, qos)` / `create_subscription(msg_type, topic_name, callback, qos)`.
- 노드가 콜백을 실제로 실행하려면 `rclpy.spin(node)`로 이벤트 루프를 돌려야 한다.
- `create_timer(period, callback)`로 주기적인 퍼블리시 패턴을 만든다.

예제로 `std_msgs/String` 대신 **표준 센서 메시지(`sensor_msgs/BatteryState`)**를 썼다. 실무에서는 토픽을 만들 때 커스텀 메시지보다 먼저 [표준 메시지 패키지](https://docs.ros.org/en/jazzy/p/sensor_msgs/)에 적합한 타입이 있는지 확인하는 것이 관례다 — 다른 패키지/도구(rqt, rviz2, nav2 등)와의 호환성이 좋아진다.

### `rclpy.spin(node)`은 정확히 무엇을 하는가

지금까지 코드에서 `battery_callback`이나 `timer_callback`을 직접 호출하는 부분은 어디에도 없다. 대신 `create_subscription`/`create_timer`로 "이 조건이 되면 이 함수를 실행해줘"라고 **등록**만 했을 뿐이다. 이 등록된 콜백들을 실제로 호출해주는 것이 `rclpy.spin(node)`다.

`rclpy.spin(node)`는 내부적으로 Executor(기본은 `SingleThreadedExecutor`, 7번 주제에서 자세히 다룬다)를 만들어 노드를 등록하고, 다음을 프로세스가 끝날 때까지 반복한다: **① 등록된 구독/타이머/서비스 중 무언가 처리할 준비가 됐는지 확인 → ② 준비된 것이 있으면 해당 콜백 함수를 대신 호출 → ③ 다시 ①로 돌아가 대기**. 즉 콜백을 실제로 부르는 주체는 우리 코드가 아니라 `spin`이 돌리는 Executor다 (이 "누가 콜백을 부르는가"라는 질문은 7번 주제 Executor에서 더 깊게 다룬다).

**`rclpy.spin(node)`이 없으면 어떻게 되는가**: 노드 객체는 생성되고 퍼블리셔/구독/타이머도 등록되지만, 그것을 실행할 이벤트 루프가 없으므로 타이머 콜백도, 구독 콜백도 **단 한 번도 호출되지 않는다**. `main()` 함수는 `spin` 줄이 없으면 노드 생성 직후 바로 다음 줄(`destroy_node`)로 넘어가 프로세스가 즉시 종료돼버린다 — 아무것도 발행되지 않고 아무것도 받지 못한 채 끝난다.

### rclpy 노드 실행의 4단계

지금까지 작성한 모든 `main()` 함수는 예외 없이 같은 4단계 구조를 따른다.

1. **`rclpy.init()`** — ROS 2 컨텍스트와 미들웨어(DDS)를 초기화한다.
2. **`node = MyNode()`** — 노드 객체를 생성한다 (내부에서 `super().__init__('node_name')`이 실행되며 퍼블리셔/구독/타이머 등이 등록된다).
3. **`rclpy.spin(node)`** — 이벤트 루프에 진입해 등록된 콜백들이 실제로 실행되게 한다.
4. **`node.destroy_node()` + `rclpy.shutdown()`** — 노드와 리소스를 정리하고 ROS 2 컨텍스트를 종료한다.

이 순서를 반드시 지켜야 하는 이유 중 하나: **1번(`rclpy.init()`)이 2번(노드 생성)보다 먼저 실행되어야 한다.** 노드는 생성될 때 내부적으로 rclpy의 컨텍스트/미들웨어 핸들을 필요로 하는데, `rclpy.init()`을 호출하기 전에 `Node()`를 만들려고 하면 그 컨텍스트가 아직 존재하지 않아 에러가 난다. 즉 "초기화 → 생성 → 실행 → 정리"라는 순서는 각 단계가 이전 단계의 결과물(컨텍스트, 노드 인스턴스, 살아있는 이벤트 루프)에 의존하기 때문에 임의로 바꿀 수 없다.

### `print` 대신 `self.get_logger()`를 쓰는 이유

지금까지 모든 노드에서 `print` 대신 `self.get_logger().info(...)`/`.warn(...)`/`.error(...)`를 썼다. 여기엔 최소 두 가지 이유가 있다.

1. **로그 레벨 필터링이 가능하다.** `get_logger()`가 남기는 로그는 debug/info/warn/error/fatal 레벨을 갖고 있어서, `ros2 run ros2_basics battery_watcher --ros-args --log-level warn`처럼 실행 시점에 "warn 이상만 보이게" 필터링할 수 있다. `print`는 이런 레벨 구분이 아예 없어서 전부 다 찍히거나 코드를 고쳐서 지우는 수밖에 없다.
2. **노드 이름/시간이 자동으로 붙고, `/rosout` 토픽으로도 발행된다.** `get_logger()`로 남긴 로그는 `[INFO] [1234567.89] [battery_watcher]: ...`처럼 어느 노드에서 언제 찍힌 로그인지 자동으로 포함되고, 동시에 `/rosout` 토픽으로도 퍼블리시된다. 그래서 `rqt_console` 같은 도구로 여러 노드의 로그를 한 화면에 모아 필터링해서 볼 수 있다. `print`는 그 프로세스의 표준출력에만 남기 때문에, 노드가 여러 개로 늘어나면(지금 이 커리큘럼만 해도 이미 10개 넘는 노드가 있다) 어느 터미널의 어느 출력이 어떤 노드 것인지 뒤섞여서 분산 디버깅이 매우 어려워진다.

## 실습 단계

### 1. `package.xml`에 `sensor_msgs` 의존성 추가

```xml
<depend>sensor_msgs</depend>
```

### 2. Publisher 노드 — 배터리 방전 시뮬레이션

`~/ros2_ws/src/ros2_basics/ros2_basics/battery_publisher.py`:

```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import BatteryState


class BatteryPublisher(Node):
    def __init__(self):
        super().__init__('battery_publisher')
        self.publisher_ = self.create_publisher(BatteryState, 'battery_state', 10)
        self.timer_ = self.create_timer(1.0, self.timer_callback)
        self.percentage_ = 1.0  # 100%

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

### 3. Subscriber 노드 — 배터리 부족 감시

`~/ros2_ws/src/ros2_basics/ros2_basics/battery_watcher.py`:

```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import BatteryState

LOW_BATTERY_THRESHOLD = 0.20
CRITICAL_BATTERY_THRESHOLD = 0.05


class BatteryWatcher(Node):
    def __init__(self):
        super().__init__('battery_watcher')
        self.subscription_ = self.create_subscription(
            BatteryState, 'battery_state', self.battery_callback, 10)

    def battery_callback(self, msg: BatteryState):
        pct = msg.percentage
        if pct <= CRITICAL_BATTERY_THRESHOLD:
            self.get_logger().error(f'배터리 위험 수준: {pct * 100:.0f}% — 즉시 복귀 필요!')
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

### 4. `setup.py` entry_points 등록

`setup.py`의 `entry_points`에 두 줄을 추가해서, `console_scripts` 리스트가 다음과 같이 되도록 한다:

```python
    entry_points={
        'console_scripts': [
            'hello_node = ros2_basics.hello_node:main',
            'battery_publisher = ros2_basics.battery_publisher:main',
            'battery_watcher = ros2_basics.battery_watcher:main',
        ],
    },
```

### 5. 빌드

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics
source install/setup.bash
```

### 6. 실행 (터미널 2개)

터미널 A:
```bash
ros2 run ros2_basics battery_publisher
```

터미널 B (새 터미널, `source install/setup.bash` 먼저):
```bash
ros2 run ros2_basics battery_watcher
```

(선택) 다른 터미널에서 실제 메시지 필드 확인:
```bash
ros2 topic echo /battery_state
```

## 예상/실제 결과

Publisher가 1초마다 5%씩 방전시키며 퍼블리시하고, Subscriber가 같은 값을 받아 임계값에 따라 로그 레벨을 다르게 찍는다.

```
[INFO] [battery_watcher]: 배터리 정상: 100%
...
[WARN] [battery_watcher]: 배터리 부족: 20%
[WARN] [battery_watcher]: 배터리 부족: 15%
[ERROR] [battery_watcher]: 배터리 위험 수준: 5% — 즉시 복귀 필요!
```

실제로 두 노드를 각각의 터미널에서 실행해 정상/부족/위험 로그가 순서대로 찍히는 것을 확인했다.

## 알려진 문제와 해결

이번 실습에서는 별도로 발생한 문제 없음.

## 이해 확인 질문

**Q1. 노드 실행에서 `rclpy.spin(node)`이 하는 일은 무엇이며, 이것이 없으면 노드는 어떻게 되나요?**
`spin(node)`는 Executor를 통해 등록된 콜백(구독/타이머/서비스 등)이 실제로 실행되도록 반복 대기·처리하는 이벤트 루프에 노드를 진입시킨다. 이게 없으면 콜백이 등록만 되고 한 번도 호출되지 않으며, `main()`이 바로 다음 줄로 넘어가 프로세스가 즉시 종료된다.

**Q2. rclpy 노드의 실행 4단계를 순서대로 쓰고, 순서를 지켜야 하는 이유를 한 가지 설명하세요.**
① `rclpy.init()` → ② `node = MyNode()` → ③ `rclpy.spin(node)` → ④ `node.destroy_node()` + `rclpy.shutdown()`. `rclpy.init()`이 노드 생성보다 먼저여야 하는 이유: 노드가 생성될 때 필요한 rclpy 컨텍스트/미들웨어 핸들이 `init()` 이전에는 존재하지 않아, 순서를 바꾸면 에러가 난다.

**Q3. ROS2에서 `print` 대신 `self.get_logger()`로 로그를 남기는 것이 왜 더 나은가요? 두 가지 이유를 드세요.**
① 로그 레벨(debug/info/warn/error)이 있어 `--log-level`로 실행 시점에 필터링할 수 있다. ② 노드 이름/시간이 자동으로 붙고 `/rosout` 토픽으로도 발행되어, 여러 노드의 로그를 `rqt_console` 등으로 한곳에 모아 볼 수 있다.

## 체크포인트

- [ ] 토픽 이름과 메시지 타입이 일치해야 pub/sub가 성립함을 설명할 수 있다.
- [ ] `create_timer`로 주기적 퍼블리시를 구현할 수 있다.
- [ ] 표준 메시지(`sensor_msgs`)를 커스텀 메시지보다 우선 검토하는 이유를 설명할 수 있다.
- [ ] Publisher/Subscriber를 각각 다른 터미널에서 실행해 통신을 확인했다.
- [ ] `rclpy.spin`과 rclpy 노드 실행 4단계, 로거를 쓰는 이유를 설명할 수 있다.

---
다음: [`03-qos.md`](./03-qos.md)
