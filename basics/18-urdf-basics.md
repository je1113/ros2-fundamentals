# URDF 기초 — xacro로 2축 팔 로봇 기술하고 RViz로 확인하기

[`00-syllabus.md`](./00-syllabus.md)의 18번째(추가) 주제. 토픽 13(`ros2_control`)에서 최소한의 URDF는 다뤄봤지만, URDF 자체의 구성 요소(visual/collision/inertial, 조인트 타입)와 반복 구조를 매크로화하는 xacro는 다루지 않았다. 이번 토픽은 `ros2_control` 없이 순수하게 로봇 기술(description)과 RViz 시각화에 집중한다.

---

## 학습 목표

- URDF `<link>`의 세 하위 요소(`visual`/`collision`/`inertial`)가 서로 다른 역할을 한다는 것을 설명할 수 있다.
- `<joint>`의 `parent`/`child`/`origin`/`axis`/`limit`으로 링크 간 연결과 가동 범위를 정의할 수 있다.
- xacro의 `property`/`macro`로 반복되는 링크·조인트 구조를 재사용 가능한 형태로 작성할 수 있다.
- `xacro` CLI로 `.urdf.xacro`를 `.urdf`로 변환하고 `check_urdf`로 검증할 수 있다.
- `robot_state_publisher` + `joint_state_publisher_gui` + RViz로 URDF를 실제로 시각화하고 조인트를 움직여볼 수 있다.

## 핵심 개념

- **`<link>`의 3요소**: `visual`(RViz 등에서 보이는 형상), `collision`(충돌 검사에 쓰이는 형상 — 단순화된 도형을 쓰는 게 일반적), `inertial`(질량/관성 텐서 — 물리 시뮬레이션·`ros2_control`에 필요). 이번 토픽에서는 셋을 모두 채워서 역할 차이를 명확히 한다.
- **`<joint>`**: `parent`/`child`로 링크 트리를 만들고, `origin`(부모 프레임 기준 자식 조인트 위치), `axis`(회전/이동 축), `limit`(가동 범위)을 정의한다. 대표 타입: `fixed`(고정), `revolute`(각도 제한 있는 회전), `continuous`(제한 없는 회전), `prismatic`(직선 이동).
- **xacro**: URDF는 순수 XML이라 반복되는 구조(팔 마디, 바퀴 등)를 그대로 복붙해야 하는데, xacro는 `<xacro:property>`(변수), `<xacro:macro>`(매크로, 파라미터·기본값 지원), `${}` 수식을 제공해 이를 재사용 가능하게 만든다.
- **`xacro` CLI → `check_urdf`**: `xacro file.urdf.xacro > file.urdf`로 매크로를 전개한 순수 URDF를 얻고, `check_urdf`로 링크 트리가 의도대로 파싱되는지 검증한다.
- **시각화 파이프라인**: `robot_state_publisher`가 URDF(`robot_description` 파라미터)를 읽어 링크 간 TF를 발행하고, `joint_state_publisher_gui`가 조인트 각도를 손으로 조절해 `/joint_states`를 발행하면, RViz가 `RobotModel` + `TF` 디스플레이로 실제 자세를 그려준다. 이 조합은 실제 하드웨어나 `ros2_control` 없이도 URDF를 눈으로 검증하는 표준적인 방법이다.

## 실습 단계

### 1. 필요 패키지 확인

```bash
ros2 pkg list | grep -E "^xacro$|joint_state_publisher_gui|rviz2"
which check_urdf
```

없으면: `sudo apt install ros-jazzy-xacro ros-jazzy-joint-state-publisher-gui liburdfdom-tools`

### 2. xacro 파일 작성

`~/ros2_ws/src/ros2_basics/urdf/simple_arm_2dof.urdf.xacro`:

```xml
<?xml version="1.0"?>
<robot name="simple_arm_2dof" xmlns:xacro="http://www.ros.org/wiki/xacro">

  <xacro:property name="segment_radius" value="0.025"/>

  <link name="base_link">
    <visual>
      <geometry><cylinder radius="0.05" length="0.1"/></geometry>
    </visual>
    <collision>
      <geometry><cylinder radius="0.05" length="0.1"/></geometry>
    </collision>
    <inertial>
      <mass value="1.0"/>
      <inertia ixx="0.01" ixy="0" ixz="0" iyy="0.01" iyz="0" izz="0.01"/>
    </inertial>
  </link>

  <xacro:macro name="arm_segment" params="name parent length axis_xyz origin_xyz joint_type:=revolute lower:=-1.57 upper:=1.57">
    <link name="${name}_link">
      <visual>
        <origin xyz="0 0 ${length/2}" rpy="0 0 0"/>
        <geometry><cylinder radius="${segment_radius}" length="${length}"/></geometry>
      </visual>
      <collision>
        <origin xyz="0 0 ${length/2}" rpy="0 0 0"/>
        <geometry><cylinder radius="${segment_radius}" length="${length}"/></geometry>
      </collision>
      <inertial>
        <origin xyz="0 0 ${length/2}" rpy="0 0 0"/>
        <mass value="0.3"/>
        <inertia ixx="0.001" ixy="0" ixz="0" iyy="0.001" iyz="0" izz="0.001"/>
      </inertial>
    </link>

    <joint name="${name}_joint" type="${joint_type}">
      <parent link="${parent}"/>
      <child link="${name}_link"/>
      <origin xyz="${origin_xyz}" rpy="0 0 0"/>
      <axis xyz="${axis_xyz}"/>
      <limit lower="${lower}" upper="${upper}" effort="10" velocity="1.0"/>
    </joint>
  </xacro:macro>

  <xacro:arm_segment name="shoulder" parent="base_link" length="0.3" axis_xyz="0 1 0" origin_xyz="0 0 0.05"/>
  <xacro:arm_segment name="elbow" parent="shoulder_link" length="0.25" axis_xyz="0 1 0" origin_xyz="0 0 0.3"/>

</robot>
```

`arm_segment` 매크로를 `shoulder`, `elbow` 두 번 호출해서 팔 두 마디를 만든다 — `<link>`/`<joint>`를 매번 손으로 반복하지 않고 매크로 파라미터(`name`/`length`/`origin_xyz`)와 `${length/2}` 같은 수식으로 재사용하는 게 xacro의 핵심이다.

### 3. `setup.py` / `package.xml` 갱신

`setup.py`의 `urdf` `data_files` 항목에 추가:

```python
        ('share/' + package_name + '/urdf', [
            'urdf/simple_arm.urdf',
            'urdf/simple_arm_2dof.urdf.xacro',
        ]),
```

`package.xml`에 실행 의존성 3줄 추가:

```xml
  <exec_depend>xacro</exec_depend>
  <exec_depend>joint_state_publisher_gui</exec_depend>
  <exec_depend>rviz2</exec_depend>
```

### 4. 빌드 + xacro → URDF 변환 + 검증

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics
source install/setup.bash

xacro ~/ros2_ws/src/ros2_basics/urdf/simple_arm_2dof.urdf.xacro > /tmp/simple_arm_2dof.urdf
check_urdf /tmp/simple_arm_2dof.urdf
```

`check_urdf`가 `base_link → shoulder_link → elbow_link` 트리를 출력하면 성공.

### 5. 시각화 파이프라인 실행 (터미널 A/B/C)

**터미널 A** — `robot_state_publisher` (xacro 결과를 그때그때 파라미터로 넘김):

```bash
ros2 run robot_state_publisher robot_state_publisher --ros-args -p robot_description:="$(xacro ~/ros2_ws/src/ros2_basics/urdf/simple_arm_2dof.urdf.xacro)"
```

**터미널 B** — 조인트 슬라이더:

```bash
ros2 run joint_state_publisher_gui joint_state_publisher_gui
```

**터미널 C** — RViz:

```bash
rviz2
```

### 6. RViz 설정 (GUI에서 직접)

1. **Displays** 패널의 **Fixed Frame**을 `base_link`로 변경
2. **Add** → By display type → **RobotModel** → OK, **Description Topic**을 `/robot_description`으로 설정
3. **Add** → By display type → **TF** → OK

## 예상/실제 결과

- `check_urdf`가 `robot name is: simple_arm_2dof`와 함께 `base_link → shoulder_link → elbow_link` 트리를 정확히 출력. (확인됨)
- RViz에서 RobotModel + TF 디스플레이 추가 후 원기둥 2마디짜리 팔이 정상적으로 보임. (확인됨)
- `joint_state_publisher_gui`의 `shoulder_joint`/`elbow_joint` 슬라이더를 움직이면 RViz의 팔이 실시간으로 따라 움직임. (확인됨)

## 알려진 문제와 해결

이번 실습에서는 (경로 오타 외) 별도로 발생한 문제 없음.

## 체크포인트

- [x] `<link>`의 visual/collision/inertial 역할 차이를 설명할 수 있다.
- [x] `<joint>`의 parent/child/origin/axis/limit으로 조인트를 정의할 수 있다.
- [x] xacro의 property/macro로 반복 구조를 재사용 가능하게 작성했다.
- [x] `xacro` → `check_urdf`로 변환·검증하는 워크플로우를 실행했다.
- [x] `robot_state_publisher` + `joint_state_publisher_gui` + RViz로 URDF를 시각화하고 조인트를 직접 움직여 확인했다.

---
토픽 13(`ros2_control`)에서 이 URDF에 하드웨어 인터페이스를 씌워 실제로 컨트롤러로 제어하는 것까지 이어서 볼 수 있다.
