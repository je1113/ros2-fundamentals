# 파라미터 (Parameters) — 재빌드 없이 배터리 임계값 바꾸기

[`00-syllabus.md`](./00-syllabus.md)의 9번째 주제. `battery_watcher`에 박혀 있던 배터리 임계값 상수를 파라미터로 바꿔, 실행 중에 조회·변경하고 YAML로 초기값을 주입해본다.

---

## 학습 목표

- `declare_parameter`/`get_parameter`로 노드에 파라미터를 선언하고 읽을 수 있다.
- `ros2 param list/get/set` CLI로 실행 중인 노드의 파라미터를 조회·변경할 수 있다.
- `add_on_set_parameters_callback`으로 파라미터 값 검증(유효 범위 체크)을 구현할 수 있다.
- YAML 파일 + `--params-file`로 초기 파라미터를 주입할 수 있다.

## 핵심 개념

지금까지 `LOW_BATTERY_THRESHOLD`/`CRITICAL_BATTERY_THRESHOLD`는 코드에 박힌 상수였다. 실무에서는 이런 임계값을 **재빌드 없이 실행 시점에 바꿀 수 있어야** 한다 — 이를 위한 것이 **파라미터**다.

- `declare_parameter(name, default)`로 선언
- `get_parameter(name).value`로 읽기
- `ros2 param set/get/list` CLI로 실행 중에도 조회/변경 가능
- `add_on_set_parameters_callback`으로 값이 바뀔 때 **검증**(예: 0.0~1.0 범위인지)하거나 부수 동작을 넣을 수 있음
- YAML 파일 + `--params-file`로 실행 시점에 여러 파라미터를 한 번에 주입 가능

### 파라미터를 쓰는 근본적인 이유 — 발행 주기(publish rate)를 예로

`battery_publisher.py`의 `self.create_timer(1.0, self.timer_callback)`에서 `1.0`(1초 주기)이 코드에 상수로 박혀 있다고 해보자. 이 로봇을 실제 배포할 때 "테스트 환경에서는 5초에 한 번만 발행해서 네트워크 부하를 줄이고 싶다"거나 "다른 로봇 기종은 0.1초마다 더 촘촘히 발행해야 한다"는 요구가 생기면, 코드를 열어 숫자를 고치고 `colcon build`로 다시 빌드한 뒤 재배포해야 한다 — 같은 로직인데 값 하나 때문에 매번 빌드/배포 파이프라인을 다시 돌려야 하는 것이다.

이 `1.0`을 파라미터(`self.declare_parameter('publish_rate', 1.0)`)로 선언해두면:
- 이미 빌드된 실행 파일을 **그대로 둔 채** `ros2 param set /battery_publisher publish_rate 5.0`으로 실행 중에 즉시 바꾸거나,
- 로봇 기종별로 다른 YAML 파일을 `--params-file`로 넘겨서 같은 실행 파일이 상황에 맞는 값으로 동작하게 할 수 있다.

즉 파라미터의 근본적인 동기는 **"코드(로직)와 설정값(숫자/문자열)을 분리"**해서, 로직은 한 번만 빌드해두고 설정값만 실행 시점/환경에 따라 재사용하기 위한 것이다. 이번 실습의 배터리 임계값도 같은 이유로 파라미터로 바꾼다 — 로봇마다, 혹은 운영 정책이 바뀔 때마다 "몇 %부터 경고할지"가 달라질 수 있기 때문이다.

## 실습 단계

### 1. `battery_watcher.py`를 파라미터 기반으로 수정

```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import BatteryState
from ros2_basics_msgs.msg import BatteryAlert
from rclpy.qos import QoSProfile, ReliabilityPolicy
from ros2_basics_msgs.srv import RequestReturnToBase
from rcl_interfaces.msg import SetParametersResult


class BatteryWatcher(Node):
    def __init__(self):
        super().__init__('battery_watcher')
        qos = QoSProfile(depth=10, reliability=ReliabilityPolicy.BEST_EFFORT)
        self.subscription_ = self.create_subscription(
            BatteryState, 'battery_state', self.battery_callback, qos
        )
        self.alert_publisher_ = self.create_publisher(BatteryAlert, 'battery_alert', 10)
        self.last_percentage_ = 1.0
        self.create_service(
            RequestReturnToBase, 'request_return_to_base', self.handle_return_request
        )
        self.declare_parameter('low_battery_threshold', 0.20)
        self.declare_parameter('critical_battery_threshold', 0.05)
        self.add_on_set_parameters_callback(self.parameter_callback)

    def parameter_callback(self, params):
        for param in params:
            if param.name in ('low_battery_threshold', 'critical_battery_threshold'):
                if not 0.0 <= param.value <= 1.0:
                    return SetParametersResult(
                        successful=False, reason='임계값은 0.0~1.0 사이여야 합니다')
        self.get_logger().info('파라미터 변경 적용됨')
        return SetParametersResult(successful=True)

    def battery_callback(self, msg: BatteryState):
        pct = msg.percentage
        self.last_percentage_ = pct
        low = self.get_parameter('low_battery_threshold').value
        critical = self.get_parameter('critical_battery_threshold').value
        alert = BatteryAlert()
        alert.robot_id = 'robot_1'
        alert.percentage = pct
        if pct <= critical:
            alert.level = BatteryAlert.LEVEL_CRITICAL
            self.get_logger().error(f'배터리 위험 수준: {pct * 100:.0f}%')
        elif pct <= low:
            alert.level = BatteryAlert.LEVEL_LOW
            self.get_logger().warn(f'배터리 부족: {pct * 100:.0f}%')
        else:
            alert.level = BatteryAlert.LEVEL_OK
            self.get_logger().info(f'배터리 정상: {pct * 100:.0f}%')
        self.alert_publisher_.publish(alert)

    def handle_return_request(self, request, response):
        low = self.get_parameter('low_battery_threshold').value
        if self.last_percentage_ <= low:
            response.accepted = True
            response.message = (
                f'승인 : {request.robot_id} 현재 배터리'
                f'{self.last_percentage_ * 100:.0f}% - 충전소로 이동하세요.'
            )
        else:
            response.accepted = False
            response.message = '거부'
        return response
```

### 2. YAML 파라미터 파일

`~/ros2_ws/src/ros2_basics/config/battery_watcher_params.yaml`:

```yaml
battery_watcher:
  ros__parameters:
    low_battery_threshold: 0.30
    critical_battery_threshold: 0.10
```

### 3. `setup.py`에 config 파일 설치 등록

`setup.py`의 `data_files` 리스트에 새 항목을 추가해서, 아래와 같이 되도록 한다 (`entry_points`는 그대로 둔다):

```python
    data_files=[
        ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
        ('share/' + package_name + '/config', ['config/battery_watcher_params.yaml']),
    ],
```

### 4. 빌드

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics
source install/setup.bash
```

### 5. 기본값 확인

터미널 A: `ros2 run ros2_basics battery_publisher`
터미널 B: `ros2 run ros2_basics battery_watcher`

터미널 C:
```bash
ros2 param list /battery_watcher
ros2 param get /battery_watcher low_battery_threshold
```

### 6. 실행 중 파라미터 변경

```bash
ros2 param set /battery_watcher low_battery_threshold 0.5
```

### 7. 잘못된 값 검증

```bash
ros2 param set /battery_watcher low_battery_threshold 1.5
```

### 8. YAML로 초기 파라미터 주입

```bash
ros2 run ros2_basics battery_watcher --ros-args --params-file install/ros2_basics/share/ros2_basics/config/battery_watcher_params.yaml
ros2 param get /battery_watcher low_battery_threshold
```

## 예상/실제 결과

- 5단계: `low_battery_threshold` 기본값 `0.2` 조회됨.
- 6단계: 파라미터를 `0.5`로 바꾸자 배터리가 50% 근처로 떨어지는 즉시(기존 20%가 아니라) `WARN` 로그로 전환됨 — 재빌드 없이 동작이 바뀜.
- 7단계: `1.5`(범위 밖)로 설정 시도 시 `parameter_callback`의 검증에 막혀 거부됨.
- 8단계: `--params-file`로 실행하자 `low_battery_threshold`가 YAML에 정의한 `0.3`으로 시작함.

실제로 4단계 전체를 순서대로 확인했다.

## 알려진 문제와 해결

이번 실습에서는 별도로 발생한 문제 없음.

## 이해 확인 질문

**Q. 파라미터를 쓰는 이유(동기)는 무엇인가요? 발행 주기를 예로 설명하세요.**
발행 주기(예: 1초)를 코드 상수로 박아두면 값을 바꿀 때마다 재빌드/재배포가 필요하다. 파라미터로 선언해두면 이미 빌드된 실행 파일을 그대로 둔 채 `ros2 param set`이나 `--params-file`로 실행 시점/환경마다 다른 값을 넣을 수 있다 — 즉 파라미터의 근본 동기는 "코드(로직)와 설정값을 분리"해서 로직은 한 번만 빌드하고 값은 재사용 가능하게 만드는 것이다.

## 체크포인트

- [ ] `declare_parameter`/`get_parameter`로 하드코딩된 상수를 파라미터로 바꿀 수 있다.
- [ ] `ros2 param set/get/list`로 실행 중인 노드의 파라미터를 다룰 수 있다.
- [ ] `add_on_set_parameters_callback`으로 잘못된 값 설정을 막을 수 있다.
- [ ] YAML + `--params-file`로 초기 파라미터를 주입할 수 있다.

---
다음: [`10-launch.md`](./10-launch.md)
