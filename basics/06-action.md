# 액션 (Action) — 충전소 복귀를 진행률과 함께, 취소 가능하게

[`00-syllabus.md`](./00-syllabus.md)의 6번째 주제. 5번 주제의 서비스는 "지금 복귀해도 되는지"를 즉시 답했지만, 실제 "충전소까지 이동"은 시간이 걸리고 중간에 취소할 수도 있어야 한다 — 이 요구사항을 액션으로 구현한다.

---

## 학습 목표

- 액션이 서비스와 무엇이 다른지(Goal/Feedback/Result, 취소 가능성) 이해한다.
- `.action` 인터페이스를 정의하고, 액션 서버/클라이언트를 구현할 수 있다.
- `ros2 action send_goal --feedback` CLI로 액션의 진행률 스트리밍과 취소를 확인할 수 있다.

## 핵심 개념

**액션**은 장시간 걸리는 작업에 쓰는 통신 모델로, 서비스와 달리 3가지 요소를 가진다.

- **Goal**: 클라이언트가 보내는 목표 (예: "충전소로 복귀해줘")
- **Feedback**: 서버가 작업 도중 주기적으로 보내는 중간 진행 상황 (예: 진행률 %)
- **Result**: 작업이 끝났을 때(성공/실패/취소) 최종 결과

| | 서비스 | 액션 |
|---|---|---|
| 소요 시간 | 짧음(즉시) | 김(수 초~수 분) |
| 중간 진행 상황 | 없음 | Feedback으로 스트리밍 |
| 취소 | 불가 | 가능 |
| 예시 | "지금 복귀 가능?" | "충전소로 실제로 이동해" |

## 실습 단계

### 1. 액션 인터페이스 정의

`~/ros2_ws/src/ros2_basics_msgs/action/ReturnToBase.action`:

```
# goal
string robot_id
---
# result
bool success
string message
---
# feedback
float32 progress
```

### 2. `CMakeLists.txt`에 action 추가

```cmake
find_package(action_msgs REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/BatteryAlert.msg"
  "srv/RequestReturnToBase.srv"
  "action/ReturnToBase.action"
  DEPENDENCIES action_msgs
)
```

### 3. `package.xml`(`ros2_basics_msgs`)에 추가

`~/ros2_ws/src/ros2_basics_msgs/package.xml`에서 `<buildtool_depend>` 아래에 한 줄을 추가해서, 파일 전체가 다음과 같이 되도록 한다:

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
  <depend>action_msgs</depend>
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

### 4. 빌드 & 확인

```bash
conda deactivate
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics_msgs
source install/setup.bash
ros2 interface show ros2_basics_msgs/action/ReturnToBase
```

### 5. 액션 서버 노드

`~/ros2_ws/src/ros2_basics/ros2_basics/charging_station_server.py`:

```python
import time
import rclpy
from rclpy.action import ActionServer, CancelResponse, GoalResponse
from rclpy.node import Node
from ros2_basics_msgs.action import ReturnToBase


class ChargingStationServer(Node):
    def __init__(self):
        super().__init__('charging_station_server')
        self._action_server = ActionServer(
            self,
            ReturnToBase,
            'return_to_base',
            execute_callback=self.execute_callback,
            goal_callback=self.goal_callback,
            cancel_callback=self.cancel_callback)

    def goal_callback(self, goal_request):
        self.get_logger().info(f'목표 수신: {goal_request.robot_id} 복귀 요청')
        return GoalResponse.ACCEPT

    def cancel_callback(self, goal_handle):
        self.get_logger().info('취소 요청 수신')
        return CancelResponse.ACCEPT

    def execute_callback(self, goal_handle):
        feedback_msg = ReturnToBase.Feedback()
        for i in range(1, 11):
            if goal_handle.is_cancel_requested:
                goal_handle.canceled()
                result = ReturnToBase.Result()
                result.success = False
                result.message = '복귀 취소됨'
                return result
            feedback_msg.progress = i * 10.0
            goal_handle.publish_feedback(feedback_msg)
            self.get_logger().info(f'복귀 진행률: {feedback_msg.progress:.0f}%')
            time.sleep(0.5)

        goal_handle.succeed()
        result = ReturnToBase.Result()
        result.success = True
        result.message = f'{goal_handle.request.robot_id} 충전소 도착 완료'
        return result


def main():
    rclpy.init()
    node = ChargingStationServer()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

### 6. 액션 클라이언트 노드

`~/ros2_ws/src/ros2_basics/ros2_basics/return_to_base_client.py`:

```python
import sys
import rclpy
from rclpy.action import ActionClient
from rclpy.node import Node
from ros2_basics_msgs.action import ReturnToBase


class ReturnToBaseClient(Node):
    def __init__(self):
        super().__init__('return_to_base_client')
        self._action_client = ActionClient(self, ReturnToBase, 'return_to_base')

    def send_goal(self, robot_id):
        self._action_client.wait_for_server()
        goal_msg = ReturnToBase.Goal()
        goal_msg.robot_id = robot_id
        self._send_goal_future = self._action_client.send_goal_async(
            goal_msg, feedback_callback=self.feedback_callback)
        self._send_goal_future.add_done_callback(self.goal_response_callback)

    def feedback_callback(self, feedback_msg):
        self.get_logger().info(f'진행률: {feedback_msg.feedback.progress:.0f}%')

    def goal_response_callback(self, future):
        goal_handle = future.result()
        if not goal_handle.accepted:
            self.get_logger().info('목표 거부됨')
            return
        self.get_logger().info('목표 수락됨')
        self._get_result_future = goal_handle.get_result_async()
        self._get_result_future.add_done_callback(self.get_result_callback)

    def get_result_callback(self, future):
        result = future.result().result
        self.get_logger().info(f'결과: success={result.success}, message="{result.message}"')
        rclpy.shutdown()


def main():
    rclpy.init()
    robot_id = sys.argv[1] if len(sys.argv) > 1 else 'robot_1'
    node = ReturnToBaseClient()
    node.send_goal(robot_id)
    rclpy.spin(node)
```

### 7. `setup.py`에 entry_point 추가

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
        ],
    },
```

### 8. 빌드

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics_msgs ros2_basics
source install/setup.bash
```

### 9. 정상 완료 시나리오 (터미널 2개)

터미널 A: `ros2 run ros2_basics charging_station_server`
터미널 B: `ros2 run ros2_basics return_to_base_client robot_1`

### 10. 취소 시나리오 — 액션의 서비스 대비 차별점

서버(터미널 A)는 켜둔 채로, ROS 2 CLI로 직접 목표를 보내고 중간에 취소한다.

```bash
ros2 action send_goal /return_to_base ros2_basics_msgs/action/ReturnToBase "{robot_id: 'robot_1'}" --feedback
```

진행률이 40~50% 정도 찍혔을 때 `Ctrl+C`로 중단한다.

## 예상/실제 결과

정상 완료 시 클라이언트에 진행률(10%, 20%, ... 100%) 로그가 순서대로 찍히고 마지막에 `success=True`가 뜬다. 취소 시나리오에서는 서버 로그에 `취소 요청 수신` → `복귀 취소됨`이 찍힌다. 실제로 두 시나리오 모두 정상 확인했다.

## 알려진 문제와 해결

실습 코드를 손으로 옮겨 적는 과정에서 5개의 오타/구현 누락 버그가 겹쳐서 발생했다. 액션은 콜백이 여러 개(`goal_callback`/`cancel_callback`/`execute_callback`/`feedback_callback`/`goal_response_callback`/`get_result_callback`)라 오타가 있어도 **해당 콜백이 실제로 호출되는 시점까지는 에러가 나지 않는다** — 이 점이 디버깅을 어렵게 만들었다.

| 파일 | 문제 | 증상이 나타나는 시점 |
|---|---|---|
| `charging_station_server.py` | `GoalResponse.ACEEPT` 오타 (`ACCEPT`가 맞음) | 첫 목표 수신 시 `AttributeError` |
| `charging_station_server.py` | `self.get_logger().ingo(...)` 오타 (`.info`가 맞음) | 취소 요청이 들어오는 시점에만 `AttributeError` — 정상 완료 시나리오만 테스트하면 발견 못 함 |
| `return_to_base_client.py` | `self.action_client`(언더스코어 누락, `self._action_client`가 맞음) | `send_goal()` 호출 즉시 `AttributeError` |
| `return_to_base_client.py` | 목표 거부 분기에 `return` 누락 | 거부됐을 때만 다음 줄로 잘못 진행 — 정상 수락 케이스만 테스트하면 발견 못 함 |
| `return_to_base_client.py` | `add_done_callback(self._get_result_callback)`과 실제 메서드명 `get_result_callback` 불일치 | 결과(Result)가 도착하는 시점에 `AttributeError` |
| `setup.py` | entry_point 이름이 `charging_station_Server`(대문자)로 등록됨 | `ros2 run`에서 소문자로 실행 시 "실행 파일 없음" |

**교훈**: 액션처럼 콜백이 여러 갈래로 나뉜 코드는, 각 콜백이 실제로 트리거되는 시나리오(정상 완료뿐 아니라 거부·취소까지)를 모두 실행해봐야 오타/버그가 드러난다.

## 체크포인트

- [ ] 액션의 Goal/Feedback/Result 3요소를 설명할 수 있다.
- [ ] `.action` 파일 문법(`---` 2개로 구분된 3구획)을 작성할 수 있다.
- [ ] `ActionServer`/`ActionClient`로 장시간 작업과 진행률 스트리밍을 구현할 수 있다.
- [ ] `ros2 action send_goal --feedback`으로 액션을 호출하고, `Ctrl+C`로 취소가 서버까지 전달되는 것을 확인했다.

---
다음: [`07-executor.md`](./07-executor.md)
