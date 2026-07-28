# Lifecycle Node (Managed Node)

[`00-syllabus.md`](./00-syllabus.md)의 16번째(추가) 주제. 일반 노드는 생성되는 순간 바로 동작을 시작하지만, `LifecycleNode`는 `unconfigured → inactive → active → finalized`라는 명시적 상태를 거치며, 상태 전이가 일어날 때만 리소스를 할당/해제하고 실제 작업(발행 등)을 시작/중단한다. Nav2 같은 대형 스택이 여러 노드의 기동 순서를 통제할 때 쓰는 패턴이다.

---

## 학습 목표

- 일반 `Node`와 `LifecycleNode`의 차이(암묵적 시작 vs 명시적 상태 전이)를 설명할 수 있다.
- `on_configure`/`on_activate`/`on_deactivate`/`on_cleanup`/`on_shutdown` 콜백에서 무엇을 해야 하는지 구분할 수 있다.
- `ros2 lifecycle list`/`get`/`set` CLI로 노드의 상태를 조회하고 전이시킬 수 있다.
- `create_lifecycle_publisher`가 일반 퍼블리셔와 다르게 "활성화 상태에서만 실제로 발행"한다는 점을 실습으로 확인한다.

## 핵심 개념

- **관리 상태(Primary State)**: `unconfigured`(생성 직후) → `inactive`(설정 완료, 대기) → `active`(실제 동작 중) → `finalized`(종료). 이 4개가 노드가 "머무르는" 상태다.
- **전이(Transition)**: 상태 사이를 이동시키는 요청. `configure`(unconfigured→inactive), `activate`(inactive→active), `deactivate`(active→inactive), `cleanup`(inactive→unconfigured), `shutdown`(어느 상태에서든→finalized). 전이 중에는 `configuring`/`activating`처럼 임시 상태를 거친다.
- **콜백과 `TransitionCallbackReturn`**: 각 전이마다 `on_xxx(self, state)` 콜백이 호출되고, `TransitionCallbackReturn.SUCCESS`/`FAILURE`/`ERROR`를 반환해 전이 성공 여부를 알린다. `on_activate`/`on_deactivate`는 부모 클래스 구현이 내부적으로 lifecycle 퍼블리셔들을 켜고 끄므로, 오버라이드할 때 `return super().on_activate(state)`로 마무리하는 게 안전하다.
- **`create_lifecycle_publisher`**: 겉보기엔 일반 퍼블리셔와 같지만, 노드가 `active` 상태가 아니면 `publish()`를 호출해도 실제로 아무것도 내보내지 않는다(내부적으로 활성/비활성 스위치가 걸려 있음).
- **CLI**: `ros2 lifecycle list <node>`(현재 상태에서 갈 수 있는 전이 목록), `ros2 lifecycle get <node>`(현재 상태), `ros2 lifecycle set <node> <transition>`(전이 실행).

## 실습 단계

### 1. 노드 작성

`~/ros2_ws/src/ros2_basics/ros2_basics/charging_dock_lifecycle.py`:

```python
import rclpy
from rclpy.lifecycle import LifecycleNode, LifecycleState, TransitionCallbackReturn
from std_msgs.msg import String


class ChargingDockLifecycle(LifecycleNode):
    def __init__(self):
        super().__init__('charging_dock_lifecycle')
        self._publisher = None
        self._timer = None
        self._max_slots = 0

    def on_configure(self, state: LifecycleState) -> TransitionCallbackReturn:
        self.declare_parameter('max_slots', 4)
        self._max_slots = self.get_parameter('max_slots').value
        self._publisher = self.create_lifecycle_publisher(String, 'dock_status', 10)
        self.get_logger().info(f'설정 완료: max_slots={self._max_slots}')
        return TransitionCallbackReturn.SUCCESS

    def on_activate(self, state: LifecycleState) -> TransitionCallbackReturn:
        self._timer = self.create_timer(1.0, self._publish_status)
        self.get_logger().info('충전독 활성화 — 상태 발행 시작')
        return super().on_activate(state)

    def on_deactivate(self, state: LifecycleState) -> TransitionCallbackReturn:
        self.destroy_timer(self._timer)
        self._timer = None
        self.get_logger().info('충전독 비활성화 — 상태 발행 중단')
        return super().on_deactivate(state)

    def on_cleanup(self, state: LifecycleState) -> TransitionCallbackReturn:
        self.destroy_publisher(self._publisher)
        self._publisher = None
        self.get_logger().info('정리 완료 — 파라미터/퍼블리셔 해제')
        return TransitionCallbackReturn.SUCCESS

    def on_shutdown(self, state: LifecycleState) -> TransitionCallbackReturn:
        self.get_logger().info(f'종료 (이전 상태: {state.label})')
        return TransitionCallbackReturn.SUCCESS

    def _publish_status(self):
        msg = String()
        msg.data = f'dock ready, slots={self._max_slots}'
        self._publisher.publish(msg)


def main():
    rclpy.init()
    node = ChargingDockLifecycle()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

`on_configure`에서는 파라미터를 읽고 퍼블리셔 객체만 만든다(아직 발행 안 함). `on_activate`에서 타이머를 시작해야 실제로 `/dock_status`에 값이 나간다. `on_deactivate`는 타이머만 멈추고 퍼블리셔는 유지, `on_cleanup`에서 퍼블리셔까지 정리해 `configure` 이전 상태로 되돌린다.

### 2. `package.xml`에 의존성 추가

`<depend>sensor_msgs</depend>` 아래에 한 줄 추가:

```xml
  <depend>std_msgs</depend>
```

### 3. `setup.py`에 entry point 추가

`entry_points`의 `console_scripts` 목록 마지막에 추가:

```python
            'charging_dock_lifecycle = ros2_basics.charging_dock_lifecycle:main',
```

### 4. 빌드

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics
source install/setup.bash
```

### 5. 노드 실행 (터미널 A)

```bash
ros2 run ros2_basics charging_dock_lifecycle
```

### 6. 상태 조회 (터미널 B)

```bash
source ~/ros2_ws/install/setup.bash
ros2 lifecycle list /charging_dock_lifecycle
ros2 lifecycle get /charging_dock_lifecycle
```

### 7. configure 전이

```bash
ros2 lifecycle set /charging_dock_lifecycle configure
ros2 lifecycle get /charging_dock_lifecycle
ros2 topic list | grep dock_status
```

### 8. activate 전이 & 발행 확인

```bash
ros2 lifecycle set /charging_dock_lifecycle activate
ros2 lifecycle get /charging_dock_lifecycle
ros2 topic echo /dock_status --once
```

### 9. deactivate → cleanup → shutdown

```bash
ros2 lifecycle set /charging_dock_lifecycle deactivate
ros2 lifecycle get /charging_dock_lifecycle

ros2 lifecycle set /charging_dock_lifecycle cleanup
ros2 lifecycle get /charging_dock_lifecycle
ros2 topic list | grep dock_status

ros2 lifecycle set /charging_dock_lifecycle shutdown
ros2 lifecycle get /charging_dock_lifecycle
```

## 예상/실제 결과

- `unconfigured` 상태에서 `ros2 lifecycle list`는 `configure`/`shutdown` 전이만 보여줌. (확인됨)
- `configure` 전이 후 터미널 A에 `설정 완료: max_slots=4` 로그, 상태는 `inactive`로 전이. (확인됨)
- `activate` 전이 후 터미널 A에 `충전독 활성화 — 상태 발행 시작` 로그, `ros2 topic echo /dock_status --once`가 `data: dock ready, slots=4`를 반환. (확인됨)
- `deactivate`/`cleanup`/`shutdown`까지 순서대로 전이되고, `shutdown` 이후 노드 프로세스가 Ctrl+C 없이 스스로 종료됨. (확인됨)

## 알려진 문제와 해결

이번 실습에서는 별도로 발생한 문제 없음.

## 체크포인트

- [x] 일반 `Node`와 `LifecycleNode`의 차이(암묵적 시작 vs 명시적 상태 전이)를 설명할 수 있다.
- [x] `on_configure`/`on_activate`/`on_deactivate`/`on_cleanup`/`on_shutdown` 콜백의 역할을 구분해 구현했다.
- [x] `ros2 lifecycle list`/`get`/`set`으로 노드 상태를 조회·전이시켰다.
- [x] `create_lifecycle_publisher`가 `active` 상태에서만 실제로 발행한다는 점을 `configure` 직후(발행 없음) vs `activate` 이후(발행 시작)로 직접 확인했다.

---
`basics` 트랙의 순수 개념 주제는 이것으로 마무리. Nav2 같은 대형 스택에서 이 패턴이 실제로 어떻게 쓰이는지는 `nav2-advanced` 트랙에서 이어서 볼 수 있다.
