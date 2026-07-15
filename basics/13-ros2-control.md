# ros2_control 입문 — 가짜 하드웨어로 1축 팔 움직이기

[`00-syllabus.md`](./00-syllabus.md)의 13번째(마지막) 주제. URDF에 `ros2_control` 태그를 추가하고, 실제 장치 없이 `mock_components`로 1축 팔(`simple_arm`)을 위치 제어해본다.

---

## 학습 목표

- URDF의 `<ros2_control>` 태그로 하드웨어 인터페이스와 명령/상태 인터페이스를 선언할 수 있다.
- `controller_manager` + 컨트롤러 YAML로 `joint_state_broadcaster`와 위치 컨트롤러를 구성할 수 있다.
- `ros2 control list_controllers`/`list_hardware_interfaces`로 상태를 확인하고, 토픽으로 관절을 움직일 수 있다.

## 핵심 개념

- **URDF의 `<ros2_control>` 태그**: 어떤 관절을 어떤 하드웨어 플러그인으로 제어할지 선언한다. 여기서는 실제 장치 없이 테스트용 `mock_components/GenericSystem`을 사용한다.
- **`controller_manager`(`ros2_control_node`)**: 하드웨어 인터페이스와 컨트롤러들을 로드/관리하는 중앙 노드.
- **컨트롤러**: `joint_state_broadcaster`(현재 관절 상태를 `/joint_states`로 발행), `position_controllers/JointGroupPositionController`(위치 명령을 받아 관절에 적용).
- **spawner**: 컨트롤러를 controller_manager에 등록(활성화)하는 유틸리티.

## 실습 단계

### 1. URDF 작성

`~/ros2_ws/src/ros2_basics/urdf/simple_arm.urdf`:

```xml
<?xml version="1.0"?>
<robot name="simple_arm">

  <link name="base_link">
    <visual>
      <geometry><cylinder radius="0.05" length="0.1"/></geometry>
    </visual>
  </link>

  <link name="arm_link">
    <visual>
      <origin xyz="0 0 0.2" rpy="0 0 0"/>
      <geometry><box size="0.05 0.05 0.4"/></geometry>
    </visual>
  </link>

  <joint name="arm_joint" type="revolute">
    <parent link="base_link"/>
    <child link="arm_link"/>
    <origin xyz="0 0 0.05" rpy="0 0 0"/>
    <axis xyz="0 1 0"/>
    <limit lower="-1.57" upper="1.57" effort="10" velocity="1.0"/>
  </joint>

  <ros2_control name="SimpleArmSystem" type="system">
    <hardware>
      <plugin>mock_components/GenericSystem</plugin>
    </hardware>
    <joint name="arm_joint">
      <command_interface name="position"/>
      <state_interface name="position"/>
      <state_interface name="velocity"/>
    </joint>
  </ros2_control>

</robot>
```

### 2. 컨트롤러 설정 YAML

`~/ros2_ws/src/ros2_basics/config/simple_arm_controllers.yaml`:

```yaml
controller_manager:
  ros__parameters:
    update_rate: 50

    joint_state_broadcaster:
      type: joint_state_broadcaster/JointStateBroadcaster

    arm_position_controller:
      type: position_controllers/JointGroupPositionController

arm_position_controller:
  ros__parameters:
    joints:
      - arm_joint
```

### 3. launch 파일

`~/ros2_ws/src/ros2_basics/launch/simple_arm.launch.py`:

```python
import os
from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description():
    pkg_share = get_package_share_directory('ros2_basics')
    urdf_path = os.path.join(pkg_share, 'urdf', 'simple_arm.urdf')
    controllers_yaml = os.path.join(pkg_share, 'config', 'simple_arm_controllers.yaml')

    with open(urdf_path, 'r') as f:
        robot_description = f.read()

    robot_state_publisher_node = Node(
        package='robot_state_publisher',
        executable='robot_state_publisher',
        parameters=[{'robot_description': robot_description}],
    )

    controller_manager_node = Node(
        package='controller_manager',
        executable='ros2_control_node',
        parameters=[{'robot_description': robot_description}, controllers_yaml],
        output='screen',
    )

    joint_state_broadcaster_spawner = Node(
        package='controller_manager',
        executable='spawner',
        arguments=['joint_state_broadcaster'],
    )

    arm_position_controller_spawner = Node(
        package='controller_manager',
        executable='spawner',
        arguments=['arm_position_controller'],
    )

    return LaunchDescription([
        robot_state_publisher_node,
        controller_manager_node,
        joint_state_broadcaster_spawner,
        arm_position_controller_spawner,
    ])
```

### 4. `package.xml`에 실행 의존성 추가

`package.xml`에서 `<depend>rosgraph_msgs</depend>` 아래에 네 줄을 추가해서, 아래와 같이 되도록 한다:

```xml
  <depend>rosgraph_msgs</depend>
  <exec_depend>robot_state_publisher</exec_depend>
  <exec_depend>controller_manager</exec_depend>
  <exec_depend>ros2_controllers</exec_depend>
  <exec_depend>joint_state_broadcaster</exec_depend>
```

### 5. `setup.py`에 urdf/config/launch 파일 설치 등록

`setup.py`의 `data_files` 리스트를 아래와 같이 바꾼다 (10번 주제의 `config`/`launch` 항목에 파일을 하나씩 더하고, `urdf` 항목을 새로 추가한다):

```python
    data_files=[
        ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
        ('share/' + package_name + '/config', [
            'config/battery_watcher_params.yaml',
            'config/battery_watcher_params_aggressive.yaml',
            'config/simple_arm_controllers.yaml',
        ]),
        ('share/' + package_name + '/launch', [
            'launch/bringup.launch.py',
            'launch/simple_arm.launch.py',
        ]),
        ('share/' + package_name + '/urdf', ['urdf/simple_arm.urdf']),
    ],
```

### 6. 빌드 & 실행

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics
source install/setup.bash
ros2 launch ros2_basics simple_arm.launch.py
```

### 7. 컨트롤러 상태 확인 (새 터미널)

```bash
ros2 control list_hardware_interfaces
ros2 control list_controllers
```

### 8. 위치 명령 전송

```bash
ros2 topic pub --once /arm_position_controller/commands std_msgs/msg/Float64MultiArray "{data: [0.8]}"
```

### 9. 관절 상태 확인

```bash
ros2 topic echo /joint_states --once
```

## 예상/실제 결과

`ros2 control list_controllers`에서 `joint_state_broadcaster`와 `arm_position_controller`가 `active` 상태로 표시되고, `0.8` 위치 명령을 보낸 뒤 `/joint_states`의 `arm_joint` position 값이 `0.8` 근처로 바뀐다. 실제로 전체 단계를 확인했다.

## 알려진 문제와 해결

이번 실습에서는 별도로 발생한 문제 없음.

## 체크포인트

- [ ] URDF에 `<ros2_control>` 태그로 하드웨어/명령·상태 인터페이스를 선언할 수 있다.
- [ ] `controller_manager` + YAML로 `joint_state_broadcaster`와 위치 컨트롤러를 구성할 수 있다.
- [ ] `ros2 control list_controllers`/`list_hardware_interfaces`로 상태를 확인할 수 있다.
- [ ] 토픽 명령으로 관절을 움직이고 `/joint_states`에서 결과를 확인했다.

---
이것으로 [`00-syllabus.md`](./00-syllabus.md)의 13개 주제를 모두 마쳤다.
