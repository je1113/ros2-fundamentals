# 서비스 (Service/Client) — 복귀 요청을 동기적으로 승인/거부하기

[`00-syllabus.md`](./00-syllabus.md)의 5번째 주제. 배터리 상태를 계속 흘려보내는 토픽(pub/sub)과 달리, "지금 복귀해도 되는지"를 그 자리에서 즉시 확인해야 하는 요청-응답 상황을 서비스로 구현한다.

---

## 학습 목표

- 서비스가 토픽과 어떻게 다른지(1:1 요청-응답 vs N:M 스트림) 구분할 수 있다.
- `.srv` 인터페이스를 정의하고, 서비스 서버/클라이언트를 구현할 수 있다.
- 서버가 토픽 구독으로 쌓인 상태(최근 배터리 값)를 서비스 응답에 활용하는 패턴을 이해한다.

## 핵심 개념

**서비스**는 토픽과 달리 **요청(request) → 응답(response)** 1:1 동기 호출 모델이다. 클라이언트가 요청을 보내면 서버가 처리 후 응답을 돌려줄 때까지 기다린다(또는 비동기로 콜백/future 등록).

| | 토픽(pub/sub) | 서비스 |
|---|---|---|
| 통신 방향 | 단방향, N:M | 요청-응답, 1:1 |
| 쓰는 경우 | 센서 스트림, 상태 브로드캐스트 | "지금 이 값 알려줘", "이 동작 지금 해도 돼?" |
| 응답 대기 | 없음 | 있음(동기/비동기) |

서비스 인터페이스는 `.srv` 파일로 정의하며, `---` 위가 요청, 아래가 응답이다.

이번 실습에서는 `battery_watcher`가 `battery_state` 토픽을 구독하면서 쌓아온 **최근 배터리 값**을, `RequestReturnToBase` 서비스 요청이 들어왔을 때 판단 기준으로 사용한다 — 토픽으로 상태를 유지하고, 서비스로 그 상태에 대한 즉각적인 판단을 요청하는 조합이다.

```
battery_publisher --(battery_state, 토픽)--> battery_watcher <--(request_return_to_base, 서비스)-- request_return_client
```

## 실습 단계

### 1. 서비스 인터페이스 정의

`~/ros2_ws/src/ros2_basics_msgs/srv/RequestReturnToBase.srv`:

```
string robot_id
---
bool accepted
string message
```

### 2. `CMakeLists.txt`에 srv 추가

`~/ros2_ws/src/ros2_basics_msgs/CMakeLists.txt`의 `rosidl_generate_interfaces(...)` 블록에 한 줄을 추가해서, 아래와 같이 되도록 한다 (파일의 다른 부분은 4번 주제와 동일하게 둔다):

```cmake
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/BatteryAlert.msg"
  "srv/RequestReturnToBase.srv"
)
```

### 3. 빌드 & 확인

```bash
conda deactivate
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics_msgs
source install/setup.bash
ros2 interface show ros2_basics_msgs/srv/RequestReturnToBase
```

### 4. `battery_watcher.py`에 서비스 서버 추가

`~/ros2_ws/src/ros2_basics/ros2_basics/battery_watcher.py` 전체를 아래 내용으로 바꾼다 (4번 주제 버전에 `RequestReturnToBase` import, `last_percentage_` 저장, `create_service` 등록, `handle_return_request` 메서드가 추가된 것이다):

```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import BatteryState
from ros2_basics_msgs.msg import BatteryAlert
from rclpy.qos import QoSProfile, ReliabilityPolicy
from ros2_basics_msgs.srv import RequestReturnToBase

LOW_BATTERY_THRESHOLD = 0.20
CRITICAL_BATTERY_THRESHOLD = 0.05


class BatteryWatcher(Node):
    def __init__(self):
        super().__init__('battery_watcher')
        qos = QoSProfile(depth=10, reliability=ReliabilityPolicy.BEST_EFFORT)
        self.subscription_ = self.create_subscription(
            BatteryState, 'battery_state', self.battery_callback, qos)
        self.alert_publisher_ = self.create_publisher(BatteryAlert, 'battery_alert', 10)
        self.last_percentage_ = 1.0
        self.create_service(
            RequestReturnToBase, 'request_return_to_base', self.handle_return_request)

    def battery_callback(self, msg: BatteryState):
        pct = msg.percentage
        self.last_percentage_ = pct
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

    def handle_return_request(self, request, response):
        if self.last_percentage_ <= LOW_BATTERY_THRESHOLD:
            response.accepted = True
            response.message = (
                f'승인: {request.robot_id} 현재 배터리 '
                f'{self.last_percentage_ * 100:.0f}% — 충전소로 이동하세요')
        else:
            response.accepted = False
            response.message = (
                f'거부: {request.robot_id} 현재 배터리 '
                f'{self.last_percentage_ * 100:.0f}% — 아직 복귀 불필요')
        return response


def main():
    rclpy.init()
    node = BatteryWatcher()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

### 5. 클라이언트 노드

`~/ros2_ws/src/ros2_basics/ros2_basics/request_return_client.py`:

```python
import sys
import rclpy
from rclpy.node import Node
from ros2_basics_msgs.srv import RequestReturnToBase


class RequestReturnClient(Node):
    def __init__(self):
        super().__init__('request_return_client')
        self.client_ = self.create_client(RequestReturnToBase, 'request_return_to_base')

    def send_request(self, robot_id: str):
        while not self.client_.wait_for_service(timeout_sec=1.0):
            self.get_logger().info('서비스 대기 중...')
        request = RequestReturnToBase.Request()
        request.robot_id = robot_id
        future = self.client_.call_async(request)
        rclpy.spin_until_future_complete(self, future)
        return future.result()


def main():
    rclpy.init()
    robot_id = sys.argv[1] if len(sys.argv) > 1 else 'robot_1'
    node = RequestReturnClient()
    result = node.send_request(robot_id)
    node.get_logger().info(f'응답: accepted={result.accepted}, message="{result.message}"')
    node.destroy_node()
    rclpy.shutdown()
```

### 6. `setup.py`에 entry_point 추가

`setup.py`의 `entry_points`에 한 줄을 추가해서, `console_scripts` 리스트가 다음과 같이 되도록 한다:

```python
    entry_points={
        'console_scripts': [
            'hello_node = ros2_basics.hello_node:main',
            'battery_publisher = ros2_basics.battery_publisher:main',
            'battery_watcher = ros2_basics.battery_watcher:main',
            'fleet_monitor = ros2_basics.fleet_monitor:main',
            'request_return_client = ros2_basics.request_return_client:main',
        ],
    },
```

### 7. 빌드

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics_msgs ros2_basics
source install/setup.bash
```

### 8. 실행 (터미널 3개)

터미널 A: `ros2 run ros2_basics battery_publisher`
터미널 B: `ros2 run ros2_basics battery_watcher`

배터리가 아직 20% 초과일 때:
```bash
ros2 run ros2_basics request_return_client robot_1
```

배터리가 20% 이하로 떨어진 뒤 다시:
```bash
ros2 run ros2_basics request_return_client robot_1
```

## 예상/실제 결과

배터리가 20% 초과일 때는 `accepted=False`(거부), 20% 이하로 떨어진 뒤에는 `accepted=True`(승인) 응답을 받는다. 실제로 같은 클라이언트 명령을 시점만 다르게 두 번 실행해 거부 → 승인이 정확히 배터리 잔량 임계값 기준으로 갈리는 것을 확인했다.

```
응답: accepted=False, message="거부: robot_1 현재 배터리 XX% — 아직 복귀 불필요"
응답: accepted=True, message="승인: robot_1 현재 배터리 15% — 충전소로 이동하세요"
```

## 알려진 문제와 해결

이번 실습에서는 별도로 발생한 문제 없음.

## 체크포인트

- [ ] 토픽과 서비스를 각각 언제 써야 하는지 구분해서 설명할 수 있다.
- [ ] `.srv` 파일의 요청/응답 구조(`---` 구분자)를 작성할 수 있다.
- [ ] `create_service`/`create_client` + `call_async`/`spin_until_future_complete` 패턴으로 동기 호출을 구현할 수 있다.
- [ ] 서버가 토픽 구독으로 쌓은 상태를 서비스 응답 판단에 활용하는 구조를 이해했다.

---
다음: [`06-action.md`](./06-action.md)
