# 01. Nav2 개요 & 아키텍처

## 1. 학습 목표

- Nav2가 여러 lifecycle 노드로 쪼개진 이유와 각 서버의 역할을 이해한다.
- lifecycle 상태 머신(unconfigured → inactive → active)을 CLI로 직접 확인하고 조작할 수 있다.
- TurtleBot3 + Gazebo 환경에서 Nav2를 띄우고, RViz로 초기 위치 부여 후 실제 내비게이션 goal을 성공시킨다.
- `NavigateToPose` 액션이 BT를 통해 어떻게 각 서버의 액션 호출로 이어지는지 감을 잡는다.

## 2. 핵심 개념

Nav2는 하나의 거대한 노드가 아니라, **역할별로 쪼개진 lifecycle 노드들의 협업**으로 동작한다.

| 노드 | 역할 |
|---|---|
| `bt_navigator` | Behavior Tree를 실행하며 전체 미션 흐름 조율 |
| `planner_server` | 전역 경로 계획 (Costmap 위에서 global path 생성) |
| `controller_server` | 지역 경로 추종 (실시간 속도 명령 생성) |
| `behavior_server` | 회복 행동 (spin, backup, wait 등) |
| `smoother_server` | 생성된 경로를 부드럽게 다듬기 |
| `waypoint_follower` | 여러 지점을 순회하는 미션 관리 |
| `velocity_smoother` | 최종 속도 명령을 부드럽게 필터링 |
| `lifecycle_manager_navigation` | 위 내비게이션 서버들의 configure→activate를 일괄 관리 |
| `lifecycle_manager_localization` | `map_server`, `amcl`의 configure→activate를 관리 (별도 매니저) |

**BT는 액션 클라이언트의 집합이다.** `bt_navigator`의 BT 노드 하나하나가 다른 서버에 대한 액션 클라이언트다 — 예를 들어 `ComputePathToPose` BT 노드는 `planner_server`가 제공하는 `ComputePathToPose` 액션의 클라이언트다. RViz의 "Nav2 Goal" 버튼을 클릭하는 순간, 내부적으로는 `bt_navigator`에게 `NavigateToPose` goal이 전달되고, BT가 그 안에서 `planner_server`/`controller_server`/`behavior_server`에 각각 액션 goal을 보내는 구조다.

## 3. 실습 단계

### 3.1 설치 확인

```bash
ros2 pkg list | grep -E "^nav2|^slam_toolbox|^turtlebot3"
```

### 3.2 TurtleBot3 + Gazebo로 Nav2 실행

```bash
ros2 launch nav2_bringup tb3_simulation_launch.py
```

Gazebo와 RViz가 뜨는지 확인한다. `TURTLEBOT3_MODEL` 환경변수는 `.bashrc`에서 관리하며(이 실습에서는 `burger` 사용), 실습 중 셸 설정 파일을 직접 건드리지 않는다 — 필요한 값은 실행 전에 본인이 직접 export/확인한다.

### 3.3 lifecycle 상태 확인

전체 노드 목록을 먼저 보고, 관심 있는 노드의 상태를 확인한다.

```bash
ros2 node list
ros2 lifecycle get /planner_server
ros2 lifecycle get /controller_server
ros2 lifecycle get /bt_navigator
```

> `ros2 lifecycle list`는 전체 노드 목록이 아니라 **특정 노드**의 가능한 상태 전이를 보여주는 명령이라 `node_name` 인자가 필수다. 전체 노드는 `ros2 node list`로 본다.

### 3.4 초기 위치 부여

RViz 상단 툴바의 **"2D Pose Estimate"**를 클릭한 뒤, Gazebo상 로봇의 실제 위치/방향에 맞춰 지도 위에 클릭+드래그로 초기 pose를 찍는다. AMCL은 lifecycle상 `active`여도 초기 pose를 받기 전까지는 `map→odom` TF를 발행하지 않는다.

### 3.5 내비게이션 goal 전송

RViz 상단 툴바의 **"Nav2 Goal"**을 클릭하고, 지도 위에서 목적지를 클릭+드래그(방향 포함)로 지정한다.

## 4. 예상/실제 결과 확인

- `ros2 lifecycle get` 결과가 관련 서버 전부 `active [3]`이어야 goal을 받을 준비가 된 것이다.
- goal 전송 후 컨테이너 로그에 `[bt_navigator]: Goal succeeded`가 찍히고 Gazebo에서 로봇이 실제로 이동하면 성공.

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| `ros2 lifecycle list` 실행 시 `node_name` 필수 에러 | 이 명령은 전체 노드 목록이 아니라 특정 노드의 상태 전이 목록을 보여줌 | `ros2 node list`로 전체 노드를 먼저 확인 |
| `controller_server`만 `active`, `planner_server`/`bt_navigator`/`smoother_server`/`behavior_server`는 `inactive`로 멈춤 | `lifecycle_manager_navigation`의 순차 활성화(`controller_server → smoother_server → planner_server → behavior_server → bt_navigator → ...`)가 중간에 멈춤. 이 세션에서는 conda `(base)` 환경이 활성화된 터미널에서 명령을 실행하고 있었던 것이 원인으로 의심됨(과거 커리큘럼에서도 conda 활성화가 ROS 2 툴체인/DDS 통신을 반복적으로 깨뜨린 전례가 있음) | `conda deactivate` 후, 멈춰 있는 각 노드를 수동으로 activate: `ros2 lifecycle set /planner_server activate` (이미 `inactive`면 configure는 끝난 상태라 activate만 하면 됨). 각 서버를 순서대로 수동 activate하면 정상화됨 |
| `global_costmap` activate 실패, 로그에 `Invalid frame ID "map" passed to canTransform` | AMCL이 아직 초기 pose를 못 받아서 `map→odom` TF를 발행하지 않음 (lifecycle상 `active`인 것과 실제로 위치를 추정한 것은 별개) | RViz에서 "2D Pose Estimate"로 초기 위치를 직접 지정 |
| `planner_server`: `GridBased plugin failed to plan ... Failed to create plan with tolerance` → `bt_navigator`: `Goal failed` | 초기 pose가 부정확해 로컬라이제이션이 어긋났거나, 목적지가 벽 안쪽 등 도달 불가능한 위치 | RViz에서 LaserScan(빨간 점)이 지도 벽선과 잘 겹치는지 확인 — 어긋나 있으면 "2D Pose Estimate"를 더 정확히 재지정. 처음엔 로봇 바로 앞의 확실히 열린 가까운 지점으로 goal을 짧게 테스트 |

## 6. 체크포인트

- [ ] Nav2/TurtleBot3/slam_toolbox 설치 확인
- [ ] `tb3_simulation_launch.py`로 Gazebo+RViz 실행
- [ ] `ros2 node list` / `ros2 lifecycle get`으로 서버 구성과 상태 확인
- [ ] 모든 내비게이션 서버를 `active`로 전환 (필요시 수동 activate)
- [ ] RViz "2D Pose Estimate"로 초기 위치 부여
- [ ] RViz "Nav2 Goal"로 목적지 전송, `Goal succeeded` 로그 확인 및 실제 이동 관찰

---
다음: `02-tb3-gazebo-first-run.md` (미작성) — 이번 토픽에서 이미 첫 실행을 다뤘으므로, SLAM 지도 생성(토픽 3)으로 바로 넘어가도 됨
