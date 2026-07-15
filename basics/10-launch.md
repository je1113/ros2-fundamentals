# launch 파일 — 세 노드를 한 번에, 파라미터 파일도 인자로

[`00-syllabus.md`](./00-syllabus.md)의 10번째 주제. `battery_publisher` + `battery_watcher`(YAML 파라미터 적용) + `fleet_monitor`를 launch 파일 하나로 함께 실행하고, 실행 시점에 파라미터 파일을 바꿔본다.

---

## 학습 목표

- `launch_ros.actions.Node`로 여러 노드를 하나의 launch 파일에 선언할 수 있다.
- `DeclareLaunchArgument` + `LaunchConfiguration`으로 실행 시점에 값을 바꿀 수 있는 인자(substitution)를 만들 수 있다.
- launch에서 노드에 YAML 파라미터 파일을 연결하는 법을 익힌다.

## 핵심 개념

지금까지는 노드를 하나씩 별도 터미널에서 `ros2 run`으로 실행했다. **launch 파일**은 여러 노드를 하나의 명령으로 함께 실행하고, 각 노드에 파라미터/이름 등을 지정할 수 있게 해준다.

- `launch_ros.actions.Node`로 노드를 선언
- `DeclareLaunchArgument` + `LaunchConfiguration`으로 실행 시점에 값을 바꿀 수 있는 **인자(substitution)** 를 만듦
- `parameters=[yaml_경로]`로 9번 주제의 YAML 파라미터 파일을 노드에 바로 연결 가능

### `ros2 run`을 여러 번 치는 대신 launch를 쓰는 이유 — '재현'

지금까지 배터리 시나리오를 실행할 때마다 터미널 3개를 열어 `ros2 run ros2_basics battery_publisher`, `battery_watcher`, `fleet_monitor`를 순서대로 타이핑했다. 이 방식의 문제는, 그날그날 어떤 노드를 켰는지·각 노드에 어떤 파라미터를 줬는지·이름을 어떻게 remap했는지 같은 "실행 조건"이 **사람의 기억과 터미널 히스토리에만** 남는다는 것이다. 시간이 지나거나, 다른 팀원이 같은 환경을 실행하려 하면, 그 조합을 정확히 **재현**하기 어렵다 — 노드 하나를 빼먹거나 파라미터 값을 다르게 주면 결과가 미묘하게 달라지고, 어디가 달랐는지 찾기도 힘들다.

launch 파일은 "어떤 노드를, 어떤 파라미터로, 어떤 이름으로 실행할지"를 **코드로 고정**해서 저장소에 커밋해둔다. 그러면 `ros2 launch ros2_basics bringup.launch.py` 한 번으로, 언제·누가 실행하든 **항상 동일한 조합이 재현**된다. 즉 launch 파일의 핵심 동기는 여러 노드를 편하게 켜는 것 자체보다, **실행 구성을 버전관리 가능하고 재현 가능한 형태로 남기는 것**에 있다.

## 실습 단계

### 1. 두 번째 파라미터 YAML (비교용)

`~/ros2_ws/src/ros2_basics/config/battery_watcher_params_aggressive.yaml`:

```yaml
battery_watcher:
  ros__parameters:
    low_battery_threshold: 0.50
    critical_battery_threshold: 0.20
```

### 2. launch 파일 작성

`~/ros2_ws/src/ros2_basics/launch/bringup.launch.py`:

```python
import os
from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument
from launch.substitutions import LaunchConfiguration
from launch_ros.actions import Node


def generate_launch_description():
    default_params_file = os.path.join(
        get_package_share_directory('ros2_basics'),
        'config', 'battery_watcher_params.yaml')

    params_file_arg = DeclareLaunchArgument(
        'params_file',
        default_value=default_params_file,
        description='battery_watcher에 적용할 파라미터 YAML 경로')

    battery_publisher_node = Node(
        package='ros2_basics',
        executable='battery_publisher',
        name='battery_publisher',
    )

    battery_watcher_node = Node(
        package='ros2_basics',
        executable='battery_watcher',
        name='battery_watcher',
        parameters=[LaunchConfiguration('params_file')],
    )

    fleet_monitor_node = Node(
        package='ros2_basics',
        executable='fleet_monitor',
        name='fleet_monitor',
    )

    return LaunchDescription([
        params_file_arg,
        battery_publisher_node,
        battery_watcher_node,
        fleet_monitor_node,
    ])
```

### 3. `package.xml`에 의존성 추가

`package.xml`의 `<depend>` 목록 마지막 줄 아래에 두 줄을 추가해서, 아래와 같이 되도록 한다:

```xml
  <depend>rclpy</depend>
  <depend>sensor_msgs</depend>
  <depend>ros2_basics_msgs</depend>
  <depend>launch</depend>
  <depend>launch_ros</depend>
```

### 4. `setup.py`에 launch/config 파일 설치 등록

`setup.py`의 `data_files` 리스트를 아래와 같이 바꾼다 (9번 주제에서 추가한 `config` 항목에 파일 하나를 더하고, `launch` 항목을 새로 추가한다):

```python
    data_files=[
        ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
        ('share/' + package_name + '/config', [
            'config/battery_watcher_params.yaml',
            'config/battery_watcher_params_aggressive.yaml',
        ]),
        ('share/' + package_name + '/launch', ['launch/bringup.launch.py']),
    ],
```

### 5. 빌드

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics
source install/setup.bash
```

### 6. 기본 파라미터로 실행

```bash
ros2 launch ros2_basics bringup.launch.py
```

세 노드(publisher/watcher/monitor)가 한 터미널에서 함께 실행되고, 기본 YAML(`low_battery_threshold: 0.30`)대로 30%에서 경고로 전환되는지 확인한다.

### 7. launch 인자로 다른 파라미터 파일 지정

```bash
ros2 launch ros2_basics bringup.launch.py params_file:=install/ros2_basics/share/ros2_basics/config/battery_watcher_params_aggressive.yaml
```

이번엔 50%에서 더 일찍 경고로 전환되는지 확인한다.

## 예상/실제 결과

한 번의 `ros2 launch` 명령으로 세 노드가 함께 실행되며, `params_file` 인자 값에 따라 같은 코드가 다른 임계값으로 동작한다 — 기본 실행은 30%, `_aggressive.yaml` 지정 시 50%에서 경고가 뜬다. 실제로 두 경우 모두 확인했다.

## 알려진 문제와 해결

이번 실습에서는 별도로 발생한 문제 없음.

## 이해 확인 질문

**Q. 여러 노드를 켤 때 `ros2 run`을 여러 번 치는 대신 launch 파일을 쓰는 이유는 무엇인가요? '재현'이라는 말을 넣어 설명하세요.**
`ros2 run`을 여러 번 실행하는 방식은 어떤 노드를 어떤 파라미터/이름으로 켰는지가 사람의 기억과 터미널 히스토리에만 남아, 시간이 지나거나 다른 사람이 실행하면 그 조합을 정확히 재현하기 어렵다. launch 파일은 이 실행 구성을 코드로 고정해두므로, 같은 명령 한 번으로 언제 누가 실행하든 동일한 조합이 재현된다.

## 체크포인트

- [ ] `launch_ros.actions.Node`로 여러 노드를 하나의 launch 파일에 선언할 수 있다.
- [ ] `DeclareLaunchArgument`/`LaunchConfiguration`으로 실행 시점 인자를 만들 수 있다.
- [ ] launch에서 노드에 YAML 파라미터 파일을 연결할 수 있다.
- [ ] `params_file:=` 인자로 실행 시점에 다른 설정을 주입해봤다.

---
다음: [`11-tf2-time.md`](./11-tf2-time.md)
