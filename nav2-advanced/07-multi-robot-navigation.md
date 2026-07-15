# 07. 멀티로봇 내비게이션

## 1. 학습 목표

- 여러 로봇을 하나의 시뮬레이션에서 네임스페이스로 분리해 동시에 운용하는 구조를 이해한다.
- `nav2_bringup`이 제공하는 공식 멀티로봇 launch 예제를 사용해 두 대의 TurtleBot3를 독립적으로 내비게이션시킨다.
- `autostart:=false` 환경에서 lifecycle을 수동으로 STARTUP시키는 서비스 호출 방식을 익힌다.

## 2. 핵심 개념

- Nav2 멀티로봇은 로봇마다 노드/토픽/TF를 **네임스페이스**(`/robot1`, `/robot2`)로 완전히 분리한다. 각 로봇이 자기만의 `bt_navigator`, `planner_server`, `controller_server` 등 전체 Nav2 스택을 독립적으로 가진다.
- `nav2_bringup`은 두 가지 공식 예제를 제공한다: `unique_multi_tb3_simulation_launch.py`(로봇마다 별도 파라미터 파일)와 `cloned_multi_tb3_simulation_launch.py`(동일 설정 복제). 이번 실습은 전자를 사용했다.
- 이 launch 파일은 `autostart:=false`가 기본값이라, 띄운 뒤 각 로봇의 `lifecycle_manager`에 **직접 STARTUP 명령을 서비스로 호출**해야 active 상태가 된다.

## 3. 실습 단계

### 3.1 실행

```bash
ros2 launch nav2_bringup unique_multi_tb3_simulation_launch.py
```

`robot1`(0.0, 0.5), `robot2`(0.0, -0.5) 두 대가 같은 샌드박스 월드에 스폰되고, 로봇마다 별도의 RViz 창이 뜬다.

### 3.2 네임스페이스 확인

```bash
ros2 node list | grep -E "robot1|robot2"
```

`/robot1/...`, `/robot2/...`로 완전히 분리된 노드 목록을 확인.

### 3.3 lifecycle 수동 STARTUP

```bash
ros2 service call /robot1/lifecycle_manager_localization/manage_nodes nav2_msgs/srv/ManageLifecycleNodes "{command: 0}"
ros2 service call /robot1/lifecycle_manager_navigation/manage_nodes nav2_msgs/srv/ManageLifecycleNodes "{command: 0}"
ros2 service call /robot2/lifecycle_manager_localization/manage_nodes nav2_msgs/srv/ManageLifecycleNodes "{command: 0}"
ros2 service call /robot2/lifecycle_manager_navigation/manage_nodes nav2_msgs/srv/ManageLifecycleNodes "{command: 0}"
```

`command: 0`은 `nav2_msgs/srv/ManageLifecycleNodes`의 `STARTUP`.

### 3.4 초기 위치 부여 + 막힌 서버 수동 activate

각 로봇의 RViz 창에서 "2D Pose Estimate"로 해당 로봇의 실제 위치를 지정. 토픽 1과 같은 이유로 `planner_server`/`bt_navigator`/`behavior_server` 등이 `inactive`에 멈춰 있으면 수동으로 activate:

```bash
ros2 lifecycle set /robot1/planner_server activate
ros2 lifecycle set /robot1/behavior_server activate
ros2 lifecycle set /robot1/bt_navigator activate
# robot2도 동일하게 반복
```

### 3.5 독립 내비게이션 테스트

각 로봇의 RViz에서 "Nav2 Goal"로 서로 다른 목적지를 지정, 두 로봇이 간섭 없이 각자 이동하는지 확인.

## 4. 예상/실제 결과 확인

- `ros2 node list`에서 두 로봇의 노드가 완전히 분리되어 보임.
- 각 로봇이 자신에게 보낸 목적지로만 이동하고, 다른 로봇의 목적지에는 반응하지 않음.

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| `ros2 service call ... "{command:0}"` 실행 시 `Failed to populate field` 에러 | YAML 파싱 문제 — 콜론(`:`) 뒤에 공백이 없으면 `command:0` 전체가 하나의 문자열로 해석됨 | `{command: 0}`처럼 콜론 뒤에 공백을 넣어서 재시도 |
| STARTUP 서비스 호출 후에도 `planner_server`/`bt_navigator`/`behavior_server`가 `inactive`에 멈춤 | 토픽 1과 동일한 원인 — AMCL이 초기 pose를 받기 전엔 `global_costmap`이 `map` TF를 못 구해 activate 실패 | 해당 로봇의 RViz에서 "2D Pose Estimate"로 초기 위치 지정 후, 막힌 노드들을 `ros2 lifecycle set <노드> activate`로 수동 활성화 |
| "Nav2 Goal"로 목적지를 찍어도 두 로봇 다 안 움직이는 것처럼 보임 | 전체 Nav2 스택 2벌 + Gazebo + RViz 2개를 동시에 띄우면서 컴퓨터 자원이 부족해짐. `controller_server` 로그에 `Control loop missed its desired rate of 20.0000 Hz. Current loop rate is 4.9751 Hz.`(목표 대비 1/4 속도) 경고가 찍힘 | RViz 창 하나를 닫아 렌더링 부하를 줄이고, 좀 더 길게(15~20초) 관찰 — 완전히 멈춘 게 아니라 심하게 느려진 것뿐이었음 |

## 6. 체크포인트

- [ ] `unique_multi_tb3_simulation_launch.py`로 두 로봇 동시 실행
- [ ] 노드 네임스페이스 분리 확인
- [ ] 두 로봇 각각 lifecycle STARTUP 서비스 호출 + 막힌 노드 수동 activate
- [ ] 각 로봇 초기 위치 지정
- [ ] 두 로봇에 서로 다른 목적지 전송, 독립적으로 이동하는 것 확인

---
Nav2 Advanced 트랙(토픽 1, 3~7) 완료. 다음은 MoveIt2 트랙(`moveit2-advanced/`, 미착수).
