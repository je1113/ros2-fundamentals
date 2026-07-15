# 04. Costmap & Planner/Controller 플러그인

## 1. 학습 목표

- Global/Local Costmap의 역할 차이와 레이어 구조(Static/Obstacle/Inflation/Voxel)를 이해한다.
- `planner_server`/`controller_server`가 플러그인 인터페이스로 알고리즘을 교체 가능하게 설계된 이유를 이해한다.
- 커스텀 params 파일로 costmap 파라미터를 실시간 튜닝하고, planner/controller 플러그인을 바꿔서 경로/주행 특성 차이를 직접 관찰한다.

## 2. 핵심 개념

| | Global Costmap | Local Costmap |
|---|---|---|
| 범위 | 전체 지도 | 로봇 주변 고정 크기 윈도우(로봇 따라 이동, rolling window) |
| 용도 | 전역 경로 계획 | 실시간 장애물 회피 |
| 기본 레이어 | Static + Obstacle + Inflation | Voxel(3D) + Inflation |

**Planner 플러그인** (`nav2_core::GlobalPlanner` 구현, `planner_server`의 `GridBased` 슬롯):
- `nav2_navfn_planner::NavfnPlanner` — 격자 기반 A*/Dijkstra. 빠르고 단순하지만 로봇의 회전 특성을 고려하지 않아 코너에서 각지게 꺾이는 경향
- `nav2_smac_planner::SmacPlanner2D` — 로봇 움직임을 더 고려해 부드러운 곡선 경로 생성

**Controller 플러그인** (`nav2_core::Controller` 구현, `controller_server`의 `FollowPath` 슬롯):
- `nav2_mppi_controller::MPPIController` (기본값) — 샘플링 기반, 다양한 궤적 후보 중 최적 선택
- `nav2_regulated_pure_pursuit_controller::RegulatedPurePursuitController` (RPP) — 경로를 단순 추종하는 방식, 더 예측 가능한 주행 특성

플러그인 교체는 코드 수정 없이 **params 파일의 문자열 하나만 바꾸면** 된다 — `nav2_core`가 정의한 공통 인터페이스 덕분.

## 3. 실습 단계

### 3.1 저장된 지도로 재실행 + Costmap 시각화

토픽 3에서 저장한 지도로 map+AMCL 모드 실행:

```bash
ros2 launch nav2_bringup tb3_simulation_launch.py map:=$HOME/ros2_ws/maps/tb3_sandbox_map.yaml
```

RViz에서 "2D Pose Estimate"로 초기 위치 부여 후, Add 버튼으로 `Costmap` 디스플레이 2개 추가(Topic: `/global_costmap/costmap`, `/local_costmap/costmap`).

### 3.2 Inflation Radius 실시간 튜닝

```bash
ros2 param set /global_costmap/global_costmap inflation_layer.inflation_radius 1.5
```

RViz Global Costmap에서 벽 주변 팽창 영역이 넓어지는 것을 확인.

### 3.3 Planner 플러그인 교체 (NavFn → SmacPlanner2D)

```bash
mkdir -p ~/ros2_ws/nav2_config
cp /opt/ros/jazzy/share/nav2_bringup/params/nav2_params.yaml ~/ros2_ws/nav2_config/nav2_params_smac.yaml
```

`nav2_params_smac.yaml`의 `planner_server.GridBased.plugin`을 `"nav2_navfn_planner::NavfnPlanner"` → `"nav2_smac_planner::SmacPlanner2D"`로 수정 후:

```bash
ros2 launch nav2_bringup tb3_simulation_launch.py map:=$HOME/ros2_ws/maps/tb3_sandbox_map.yaml params_file:=$HOME/ros2_ws/nav2_config/nav2_params_smac.yaml
```

초기 위치 재지정 후 이전과 비슷한 목적지로 Nav2 Goal을 찍어 경로 모양을 비교.

### 3.4 Controller 플러그인 교체 (MPPI → RPP)

같은 `nav2_params_smac.yaml`에서 `controller_server.FollowPath.plugin`을 `"nav2_mppi_controller::MPPIController"` → `"nav2_regulated_pure_pursuit_controller::RegulatedPurePursuitController"`로 수정 후 재실행, 다시 목적지를 찍어 로봇이 실제로 경로를 **추종하는 움직임**(코너링, 부드러움)을 비교.

## 4. 예상/실제 결과 확인

- `inflation_radius`를 늘리면 costmap의 팽창 영역이 즉시 넓어짐 (재시작 불필요, 실시간 파라미터).
- `SmacPlanner2D`로 바꾼 경로는 `NavfnPlanner`보다 코너에서 더 부드러운 곡선으로 관찰됨.
- `RegulatedPurePursuitController`로 바꾼 뒤 실제 주행이 `MPPIController`보다 부드럽게 도는 것으로 관찰됨 (두 컨트롤러 다 기본 파라미터 기준 — 실제 성능 우열이 아니라 튜닝 성향 차이를 보여주는 게 목적).

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| `ros2 param set ... inflation_layer.inflatation_radius`가 "cannot be set because it was not declared" 에러 | 파라미터명 오타 (`inflatation` → `inflation`) | 정확한 파라미터명(`inflation_radius`)으로 재시도 |
| Controller 플러그인을 RPP로 바꿨는데 `FollowPath` 아래에 MPPI 전용 파라미터(`time_steps`, `model_dt` 등)가 그대로 남아있음 | 각 컨트롤러 플러그인은 자기가 선언한 파라미터만 읽고 나머지는 무시하므로 에러는 안 남 | 정리하려면 지워도 되지만, 실습 목적상 굳이 지우지 않아도 동작에 지장 없음 |

## 6. 체크포인트

- [ ] Global/Local Costmap을 RViz에 시각화
- [ ] `inflation_radius` 실시간 변경 및 costmap 변화 확인
- [ ] custom params 파일로 Planner를 NavFn→SmacPlanner2D 교체, 경로 모양 비교
- [ ] custom params 파일로 Controller를 MPPI→RPP 교체, 주행 움직임 비교

---
다음: [`05-behavior-tree.md`](./05-behavior-tree.md) (미작성)
