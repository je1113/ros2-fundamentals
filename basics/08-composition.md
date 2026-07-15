# 컴포지션 (Component Node) — 여러 노드를 프로세스 하나로 묶기

[`00-syllabus.md`](./00-syllabus.md)의 8번째 주제. `battery_publisher`, `battery_watcher`, `fleet_monitor` 세 노드를 각각 별도 프로세스로 띄우는 대신 하나의 프로세스로 묶어 실행한다.

---

## 학습 목표

- 컴포지션(여러 노드를 한 프로세스로 묶는 것)의 목적을 이해한다.
- 여러 `Node` 인스턴스를 하나의 Executor에 묶어 함께 `spin`하는 rclpy 패턴을 구현할 수 있다.
- `ros2 node list`로 "프로세스 수와 노드 수는 별개"임을 확인한다.

## 핵심 개념

여러 노드를 각각 별도 프로세스(`ros2 run`을 여러 번)로 띄우는 대신, **하나의 프로세스 안에서 여러 노드를 함께 실행**하는 것이 컴포지션이다. C++(`rclcpp`)에서는 zero-copy in-process 전송까지 최적화되지만, Python(`rclpy`)에서는 주로 **프로세스 수 절감과 배포 단순화**가 이점이다 — 예를 들어 한 로봇에 딸린 배터리 관련 노드들(퍼블리셔/워처/모니터)을 하나의 프로세스로 묶어 배포하면 관리가 쉬워진다.

가장 단순한 rclpy 컴포지션 방법은 여러 `Node` 인스턴스를 만들어 **하나의 Executor**에 묶어 함께 `spin`하는 것이다. 노드 수는 그대로 3개지만(`ros2 node list`에는 3개로 보임), 실행되는 OS 프로세스는 1개뿐이다.

## 실습 단계

### 1. 여러 노드를 묶는 진입점 작성

`~/ros2_ws/src/ros2_basics/ros2_basics/combined_bringup.py`:

```python
import rclpy
from rclpy.executors import MultiThreadedExecutor

from ros2_basics.battery_publisher import BatteryPublisher
from ros2_basics.battery_watcher import BatteryWatcher
from ros2_basics.fleet_monitor import FleetMonitor


def main():
    rclpy.init()
    publisher = BatteryPublisher()
    watcher = BatteryWatcher()
    monitor = FleetMonitor()

    executor = MultiThreadedExecutor(num_threads=4)
    executor.add_node(publisher)
    executor.add_node(watcher)
    executor.add_node(monitor)
    try:
        executor.spin()
    finally:
        publisher.destroy_node()
        watcher.destroy_node()
        monitor.destroy_node()
        rclpy.shutdown()
```

기존 노드들의 `main()` 함수(각자 `rclpy.init`/`spin`/`shutdown`을 호출)는 건드리지 않고, **클래스만 import**해서 새 진입점에서 인스턴스화한다는 점이 핵심이다.

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
            'combined_bringup = ros2_basics.combined_bringup:main',
        ],
    },
```

### 3. 빌드 & 실행

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics
source install/setup.bash
ros2 run ros2_basics combined_bringup
```

### 4. 확인

같은 터미널 로그에 배터리 정상/부족/위험 로그(`battery_watcher` 역할)와 `[Fleet] ...`(`fleet_monitor` 역할) 로그가 섞여서 찍히는지 확인한다. 다른 터미널에서:

```bash
ros2 node list
```

`battery_publisher`, `battery_watcher`, `fleet_monitor` 세 노드가 모두 나오는지 확인한다 — 프로세스는 하나지만 노드는 3개로 보인다.

## 예상/실제 결과

한 프로세스(`combined_bringup`)의 로그에 세 노드의 출력이 함께 찍히고, `ros2 node list`에는 여전히 3개의 개별 노드로 표시된다. 실제로 두 가지 모두 확인했다.

## 알려진 문제와 해결

이번 실습에서는 별도로 발생한 문제 없음.

## 체크포인트

- [ ] 컴포지션의 목적(프로세스 수 절감/배포 단순화)을 설명할 수 있다.
- [ ] 여러 `Node` 인스턴스를 하나의 Executor에 묶어 함께 실행할 수 있다.
- [ ] "프로세스 개수"와 "노드 개수"가 별개 개념임을 `ros2 node list`로 확인했다.

---
다음: [`09-parameters.md`](./09-parameters.md)
