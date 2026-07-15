# Executor & 콜백 그룹 — 콜백 안에서 서비스를 부르면 생기는 데드락

[`00-syllabus.md`](./00-syllabus.md)의 7번째 주제. 배터리가 부족할 때 자동으로 복귀 서비스를 호출하는 노드를 만들면서, 구독 콜백 안에서 서비스를 동기적으로 호출할 때 생기는 유명한 데드락을 직접 재현하고 해결한다.

---

## 학습 목표

- `rclpy.spin(node)`가 기본적으로 SingleThreadedExecutor를 쓴다는 것을 이해한다.
- 콜백 안에서 다른 서비스를 동기적으로 호출하면 왜 데드락이 나는지 원리를 이해한다.
- `MultiThreadedExecutor` + `ReentrantCallbackGroup`으로 해결하는 법을 익힌다.

## 핵심 개념

### Executor란 무엇이고, 왜 콜백이 "저절로" 실행되는가

2번 주제에서 봤듯, 우리는 `create_subscription`/`create_timer`/`create_service`로 콜백 함수를 **등록**만 할 뿐, 그 함수를 코드에서 직접 호출한 적이 없다. 그런데도 메시지가 오거나 타이머 주기가 되면 해당 콜백이 실행된다 — 이걸 실제로 실행해주는 주체가 **Executor**다.

`rclpy.spin(node)`를 호출하면 내부적으로 Executor(기본은 **SingleThreadedExecutor**)가 노드를 등록하고, 다음을 계속 반복한다: 노드에 등록된 구독/타이머/서비스/액션 콜백 중 **"지금 실행할 준비가 된 것"**이 있는지 감시하다가, 준비된 게 있으면 그 콜백 함수를 대신 호출해준다. 즉 "무엇을 실행할지"는 우리가 코드로 등록하지만, "언제 실행할지 결정하고 실제로 호출하는 일"은 Executor가 담당한다 — 이런 실행 모델을 이벤트 기반(event-driven) 또는 제어의 역전(Inversion of Control)이라 부른다.

### SingleThreadedExecutor의 한계

`SingleThreadedExecutor`는 이름 그대로, 노드에 등록된 모든 콜백을 **스레드 하나로 순차 실행**한다. 예를 들어 한 노드에 다음 세 콜백이 함께 등록돼 있다고 하자.

- 라이다 구독 콜백 (10Hz로 자주 들어옴, 빠르게 처리돼야 함)
- 1초 주기 타이머 콜백
- 처리에 3초가 걸리는 서비스 콜백

이 서비스 콜백이 한 번 호출되면, SingleThreadedExecutor의 유일한 스레드는 그 3초짜리 작업 하나에 통째로 묶여버린다. 그동안 들어오는 라이다 메시지나 타이머 이벤트는 처리되지 못하고 **대기열에 쌓이기만** 한다. 3초 뒤 서비스 콜백이 끝나야 비로소 그 사이 밀린 라이다/타이머 콜백들이 몰아서(뒤늦게) 처리된다 — 그동안 라이다 데이터의 실시간성은 심각하게 훼손된다. 이것이 **"느린 콜백 하나가 같은 노드의 모든 빠른 콜백을 막는다"**는 SingleThreadedExecutor의 핵심 한계다. (이번 실습의 데드락은 이 한계의 극단적인 경우로, 서비스 응답을 "그 자리에서" 기다리다 보니 3초가 아니라 영원히 막혀버리는 상황이다.)

### 문제 재현 — 콜백 안에서 다른 서비스를 동기 호출하면

1. 구독 콜백이 실행 중 → SingleThreadedExecutor의 유일한 스레드를 점유
2. 서비스 응답을 처리하려면 그 스레드가 다시 이벤트 루프를 돌아야 함
3. 근데 그 스레드는 지금 **자기 자신(구독 콜백)이 끝나기를 기다리는 중** → 영원히 진행 불가 (데드락)

### 해결 — 두 가지 조건이 함께 필요하다

긴 콜백(또는 콜백 안에서의 동기 대기)이 다른 빠른 콜백을 막지 않게 하려면, 아래 **두 가지가 함께** 있어야 한다.

1. **`MultiThreadedExecutor`** — 콜백을 처리할 스레드를 여러 개로 늘린다.
2. **콜백 그룹을 나눈다** (`ReentrantCallbackGroup`, 또는 서로 다른 `MutuallyExclusiveCallbackGroup`) — 관련 콜백들이 실제로 다른 스레드에서 동시에 실행되도록 허용한다.

**왜 하나만으로는 안 되는가:**

- `MultiThreadedExecutor`만 쓰고 콜백 그룹을 나누지 않으면: rclpy에서 콜백을 그룹으로 지정하지 않으면 노드의 기본 콜백 그룹(`MutuallyExclusiveCallbackGroup`)에 모두 속하게 되는데, 이 그룹은 **같은 그룹 안의 콜백끼리는 스레드가 몇 개든 동시 실행을 막는다.** 즉 스레드는 여러 개 있어도 실질적으로는 여전히 하나씩만 실행되어, SingleThreadedExecutor와 결과가 같아진다.
- 콜백 그룹만 나누고 여전히 `SingleThreadedExecutor`를 쓰면: 애초에 물리적으로 스레드가 하나뿐이므로, 그룹을 아무리 나눠도 동시에 실행할 스레드 자체가 없어 여전히 순차 실행될 수밖에 없다.

그래서 이번 실습에서는 `MultiThreadedExecutor(num_threads=4)`와 `ReentrantCallbackGroup()`을 **함께** 적용해서 데드락을 해결한다.

## 실습 단계

### 1. 데드락을 재현하는 노드

`~/ros2_ws/src/ros2_basics/ros2_basics/auto_return_manager.py`:

```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import BatteryState
from ros2_basics_msgs.srv import RequestReturnToBase

LOW_BATTERY_THRESHOLD = 0.20


class AutoReturnManager(Node):
    def __init__(self):
        super().__init__('auto_return_manager')
        self.client_ = self.create_client(RequestReturnToBase, 'request_return_to_base')
        self.create_subscription(BatteryState, 'battery_state', self.battery_callback, 10)

    def battery_callback(self, msg):
        if msg.percentage > LOW_BATTERY_THRESHOLD:
            self.get_logger().info(f'배터리 정상: {msg.percentage * 100:.0f}%')
            return

        self.get_logger().info('배터리 부족 감지 — 복귀 요청 서비스 호출 (동기 대기)')
        request = RequestReturnToBase.Request()
        request.robot_id = 'robot_1'
        future = self.client_.call_async(request)
        rclpy.spin_until_future_complete(self, future)  # <- 데드락 지점
        self.get_logger().info(f'응답: {future.result().message}')


def main():
    rclpy.init()
    node = AutoReturnManager()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

### 2. `setup.py`에 entry_point 추가

`setup.py`의 `entry_points`에 한 줄을 추가해서, `console_scripts` 리스트가 다음과 같이 되도록 한다:

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
        ],
    },
```

### 3. 빌드

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics
source install/setup.bash
```

### 4. 데드락 재현 (터미널 3개)

터미널 A: `ros2 run ros2_basics battery_publisher`
터미널 B: `ros2 run ros2_basics battery_watcher` (서비스 서버 역할)
터미널 C: `ros2 run ros2_basics auto_return_manager`

배터리가 20% 이하로 떨어지면 `배터리 부족 감지 — 복귀 요청 서비스 호출 (동기 대기)` 로그가 찍힌 뒤, 그 이후로 **아무 로그도 더 찍히지 않고 완전히 멈춘다**. `Ctrl+C`로 종료한다.

### 5. 해결 — MultiThreadedExecutor + ReentrantCallbackGroup

```python
from rclpy.executors import MultiThreadedExecutor
from rclpy.callback_groups import ReentrantCallbackGroup
from rclpy.qos import QoSProfile, ReliabilityPolicy
```

```python
    def __init__(self):
        super().__init__('auto_return_manager')
        callback_group = ReentrantCallbackGroup()
        qos = QoSProfile(depth=10, reliability=ReliabilityPolicy.BEST_EFFORT)
        self.client_ = self.create_client(
            RequestReturnToBase, 'request_return_to_base', callback_group=callback_group)
        self.create_subscription(
            BatteryState, 'battery_state', self.battery_callback, qos,
            callback_group=callback_group)
```

```python
def main():
    rclpy.init()
    node = AutoReturnManager()
    executor = MultiThreadedExecutor(num_threads=4)
    executor.add_node(node)
    executor.spin()
    node.destroy_node()
    rclpy.shutdown()
```

### 6. 재빌드 & 재실행

```bash
colcon build --symlink-install --packages-select ros2_basics
source install/setup.bash
ros2 run ros2_basics auto_return_manager
```

## 예상/실제 결과

- 4단계(데드락): `배터리 부족 감지...` 로그 이후 완전히 멈춤 — 응답 로그도, 다음 배터리 값 로그도 없음.
- 6단계(해결 후): 배터리 로그가 계속 이어지고, 20% 이하에서 `응답: ...` 로그까지 정상적으로 찍힘.

실제로 두 시나리오 모두 확인했다.

## 알려진 문제와 해결

| 문제 | 원인 | 해결 |
|---|---|---|
| 배터리 20% 이하에서 노드가 완전히 멈춤 (데드락) | SingleThreadedExecutor에서 구독 콜백이 `rclpy.spin_until_future_complete()`로 서비스 응답을 동기 대기 — 응답을 처리할 스레드가 이미 그 콜백 자신에게 점유되어 있어 영원히 진행 불가 | `MultiThreadedExecutor` + `ReentrantCallbackGroup`으로 구독/서비스 클라이언트 콜백을 다른 스레드가 처리할 수 있게 함 |
| 해결 코드 적용 후에도 `WARN ... incompatible QoS ... RELIABILITY` 경고와 함께 배터리 로그가 안 찍힘 | `auto_return_manager`의 구독이 기본 QoS(`RELIABLE`)를 쓰는데 `battery_publisher`는 3번 주제에서 `BEST_EFFORT`로 맞춰둔 상태라 다시 QoS 불일치 발생 | 구독 QoS를 `QoSProfile(depth=10, reliability=ReliabilityPolicy.BEST_EFFORT)`로 맞춤 — 새 노드를 추가할 때마다 기존에 맞춰둔 QoS를 함께 확인해야 함을 보여주는 사례 |

**교훈**: Executor/콜백 그룹 문제를 고쳐도, 그 아래 QoS가 안 맞으면 여전히 "조용한 실패"가 난다. 새 노드가 기존 토픽을 구독할 때는 항상 그 토픽의 기존 QoS 설정([`03-qos.md`](./03-qos.md) 참고)을 확인해야 한다.

## 이해 확인 질문

**Q1. 우리가 콜백을 코드에서 직접 부르지 않는데도 콜백이 실행되는 이유는 무엇인가요? executor 개념으로 설명하세요.**
`create_subscription`/`create_timer` 등은 콜백을 "등록"만 할 뿐이다. `rclpy.spin(node)`가 만드는 Executor가 등록된 콜백 중 실행 준비가 된 것을 감시하다가 대신 호출해준다 — 무엇을 실행할지는 우리가 정하지만, 언제·누가 실제로 호출하는지는 Executor가 담당하는 이벤트 기반 실행 모델이다.

**Q2. 기본 SingleThreadedExecutor의 한계는 무엇인가요? 한 노드에 라이다 콜백·타이머 콜백·3초 걸리는 서비스 콜백이 함께 있는 상황으로 설명하세요.**
SingleThreadedExecutor는 노드의 모든 콜백을 스레드 하나로 순차 실행한다. 3초짜리 서비스 콜백이 실행되는 동안 그 스레드가 통째로 묶여, 그 사이 들어오는 라이다/타이머 콜백은 처리되지 못하고 대기열에 쌓였다가 3초 후에야 몰아서 처리된다 — 느린 콜백 하나가 빠른 콜백 전부를 막는다.

**Q3. 긴 콜백이 빠른 콜백을 막지 않게 하려면 두 가지 조건이 함께 필요합니다. 무엇과 무엇인가요? 그리고 하나만 했을 때 왜 안 되는지 설명하세요.**
① `MultiThreadedExecutor`(스레드를 여러 개로) + ② 콜백 그룹 분리(`ReentrantCallbackGroup` 등, 관련 콜백이 실제로 동시 실행되게 허용). `MultiThreadedExecutor`만 쓰면 기본 콜백 그룹(`MutuallyExclusiveCallbackGroup`)이 같은 그룹 내 콜백의 동시 실행을 막아 스레드가 여러 개여도 결과는 순차 실행과 같다. 반대로 그룹만 나누고 `SingleThreadedExecutor`를 쓰면 애초에 스레드가 하나뿐이라 동시 실행할 물리적 수단이 없다.

## 체크포인트

- [ ] SingleThreadedExecutor에서 콜백 안 동기 서비스 호출이 왜 데드락으로 이어지는지 설명할 수 있다.
- [ ] `MultiThreadedExecutor`와 `ReentrantCallbackGroup`을 함께 써서 데드락을 해결할 수 있다.
- [ ] 새 노드를 추가할 때 기존 토픽의 QoS를 확인해야 하는 이유를 실제로 겪어봤다.

---
다음: [`08-composition.md`](./08-composition.md)
