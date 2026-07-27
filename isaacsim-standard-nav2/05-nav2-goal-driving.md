# 05. Nav2 목표 주행 검증 — 벽/코너 wedging 재현 시도

> 진행 상태: **완료** — 4개 코너 모두 wedging 없이 성공. 이 트랙의 원래 목적(vacuum 프로젝트 wedging 버그가 하드웨어 문제인지 Nav2/설정 문제인지 격리)에 대한 결론 도출

## 1. 학습 목표

- 표준 `nav2_bringup`(map_server + amcl + planner/controller/costmap + bt_navigator)을 이 트랙의 실제 TF/토픽 구성에 맞춰 띄운다.
- Nav2 `NavigateToPose` 액션으로 벽/코너에 가까운 목표 지점까지 주행시키고, `isaacsim-ros2-advanced`(vacuum) 프로젝트의 Part D에서 반복 재현됐던 wedging(벽/코너 근처에서 로봇이 끼여 못 움직이는) 증상이 여기서도 나타나는지 확인한다.
- 이 트랙 전체의 결론(하드웨어 문제 vs Nav2/설정 문제)을 근거를 들어 내린다.

## 2. 핵심 개념

**AMCL 초기 포즈와 lifecycle bringup 타이밍 경쟁**: `nav2_bringup`이 `autostart:=true`로 뜨면, `lifecycle_manager_localization`(map_server+amcl)은 즉시 활성화되지만 `lifecycle_manager_navigation`(costmap/planner/controller/bt_navigator)은 `base_link`→`map` TF(AMCL이 초기 포즈를 받아야 발행)가 있어야 코스트맵을 활성화할 수 있다. 이 TF가 일정 시간(수십 초) 안에 나타나지 않으면 `nav2_costmap_2d`가 활성화를 포기하고 **`lifecycle_manager_navigation`이 전체 브링업을 완전히 중단**한다 — 재시도 없이 그대로 실패 상태로 멈춘다. 즉 `/initialpose`는 "언제 줘도 되는" 게 아니라 **브링업 직후 수 초~수십 초 내에** 줘야 하는 시간 제약이 있다.

**MPPI 컨트롤러의 `DiffDrive` 모션 모델**: 기본 `nav2_params.yaml`의 `FollowPath` 플러그인은 `nav2_mppi_controller::MPPIController`, `motion_model: "DiffDrive"`로 설정돼 있다 — Carter가 진짜 차동구동(differential drive) 로봇이므로 이 기본값을 그대로 쓸 수 있었다(vacuum 프로젝트처럼 조인트 없는 로봇을 위한 우회가 필요 없음).

**`robot_radius`와 실제 로봇 크기**: 기본값 `0.22`m는 TurtleBot3 기준. Carter의 실제 시각 바디 bbox(Topic 3-C에서 측정: `x:[-0.385,0.212], y:[-0.292,0.292]`)에 맞춰 `robot_radius: 0.35`로 올려서 두 코스트맵(local/global) 모두에 적용했다.

## 3. 실습 단계

### 3.1 `nav2_params.yaml` 준비

`/opt/ros/jazzy/share/nav2_bringup/params/nav2_params.yaml`을 베이스로 두 가지만 `sed`로 일괄 수정해서 저장:

```bash
mkdir -p ~/isaac_assets/carter_standard/nav2
sed -e 's/base_frame_id: "base_footprint"/base_frame_id: "base_link"/g' \
    -e 's/robot_radius: 0.22/robot_radius: 0.35/g' \
    /opt/ros/jazzy/share/nav2_bringup/params/nav2_params.yaml \
    > ~/isaac_assets/carter_standard/nav2/nav2_params.yaml
```

- 모든 `base_frame_id: "base_footprint"` → `"base_link"` (amcl, velocity_smoother, docking_server 3곳 — 이 트랙엔 `base_footprint` 프레임이 없음, `robot_base_frame`은 이미 기본값이 `base_link`라 안 건드림)
- `robot_radius: 0.22` → `0.35` (local/global costmap 2곳, Carter의 실제 바디 크기에 맞춤)

### 3.2 Nav2 브링업 + 초기 포즈 (타이밍 이슈 해결)

```bash
conda deactivate
source /opt/ros/jazzy/setup.bash
ros2 launch nav2_bringup bringup_launch.py \
  map:=$HOME/isaac_assets/carter_standard/maps/carter_room_map.yaml \
  params_file:=$HOME/isaac_assets/carter_standard/nav2/nav2_params.yaml \
  use_sim_time:=true autostart:=true
```

**처음 두 번은 실패했다** — `/odom`을 확인하고 `sed`로 파라미터 파일을 준비하는 등 다른 작업을 하다 초기 포즈를 너무 늦게(브링업 시작 후 30초 이상 지나서) 보냈더니, `global_costmap`이 `base_link`→`map` transform 대기 타임아웃으로 활성화 실패 → `lifecycle_manager_navigation`이 브링업 자체를 중단했다. **해결**: 브링업을 띄우는 동시에 `ros2 topic pub -r 2 /initialpose ...`를 `timeout 25`로 감싸 백그라운드에서 2Hz로 25초간 반복 발행 — AMCL 구독자가 언제 준비되든 놓치지 않고 몇 초 안에 전달되도록 했다.

`nav2_bringup`을 실행한 터미널과는 **다른 새 터미널**에서, 브링업을 띄우자마자 바로 이어서 실행한다 (브링업이 foreground를 점유하고 있으므로):

먼저 로봇의 현재 위치를 확인한다 (맵을 만들 때와 같은 `odom` 원점 기준이라, 이 값을 그대로 `map` 프레임 초기 추정치로 써도 충분히 가깝다):

```bash
conda deactivate
source /opt/ros/jazzy/setup.bash
ros2 topic echo /odom --once
```

`pose.pose.position`(`x`,`y`)과 `pose.pose.orientation`(`x`,`y`,`z`,`w`) 값을 그대로 아래 명령의 해당 자리에 넣는다:

```bash
conda deactivate
source /opt/ros/jazzy/setup.bash
timeout 25 ros2 topic pub -r 2 /initialpose geometry_msgs/msg/PoseWithCovarianceStamped \
  "{header: {frame_id: 'map'}, pose: {pose: {position: {x: <위에서 확인한 x>, y: <위에서 확인한 y>, z: 0.0}, orientation: {x: <확인한 x>, y: <확인한 y>, z: <확인한 z>, w: <확인한 w>}}, covariance: [0.25,0,0,0,0,0, 0,0.25,0,0,0,0, 0,0,0,0,0,0, 0,0,0,0,0,0, 0,0,0,0,0,0, 0,0,0,0,0,0.0685]}}"
```

성공 시 로그에 `lifecycle_manager_localization: Managed nodes are active`, `lifecycle_manager_navigation: Managed nodes are active`가 순서대로 뜨고 `/tf`에 `map→odom`, `odom→base_link` 두 쌍이 모두 나타난다.

### 3.3 열린 공간 목표 (동작 확인용)

```bash
conda deactivate
source /opt/ros/jazzy/setup.bash
ros2 action send_goal /navigate_to_pose nav2_msgs/action/NavigateToPose \
  "{pose: {header: {frame_id: 'map'}, pose: {position: {x: 1.0, y: -1.0, z: 0.0}, orientation: {w: 1.0}}}}"
```

`Goal finished with status: SUCCEEDED` — 이후 코너 테스트 전에 스택이 기본적으로 정상 동작함을 먼저 확인.

### 3.4 벽/코너 근접 목표 4개 — wedging 재현 시도

6m×5m 방(벽 안쪽 면: x∈[-2.95,2.95], y∈[-2.45,2.45])의 네 코너 각각에서 안쪽으로 약 0.35~0.55m 들어간 지점(로봇 반지름 `0.35`와 비슷한 수준의 여유)으로 하나씩 순서대로 보낸다. `--feedback`을 붙이면 `number_of_recoveries`(Spin/BackUp 등 복구 행동이 몇 번 트리거됐는지)를 실시간으로 볼 수 있다 — wedging이 재현된다면 여기 값이 계속 올라간다.

```bash
conda deactivate
source /opt/ros/jazzy/setup.bash

# NE 코너
ros2 action send_goal /navigate_to_pose nav2_msgs/action/NavigateToPose \
  "{pose: {header: {frame_id: 'map'}, pose: {position: {x: 2.4, y: 1.9, z: 0.0}, orientation: {w: 1.0}}}}" --feedback

# SW 코너
ros2 action send_goal /navigate_to_pose nav2_msgs/action/NavigateToPose \
  "{pose: {header: {frame_id: 'map'}, pose: {position: {x: -2.6, y: -2.1, z: 0.0}, orientation: {w: 1.0}}}}" --feedback

# NW 코너
ros2 action send_goal /navigate_to_pose nav2_msgs/action/NavigateToPose \
  "{pose: {header: {frame_id: 'map'}, pose: {position: {x: -2.6, y: 1.9, z: 0.0}, orientation: {w: 1.0}}}}" --feedback

# SE 코너
ros2 action send_goal /navigate_to_pose nav2_msgs/action/NavigateToPose \
  "{pose: {header: {frame_id: 'map'}, pose: {position: {x: 2.4, y: -2.1, z: 0.0}, orientation: {w: 1.0}}}}" --feedback
```

| 목표 | 결과 | `number_of_recoveries` |
|---|---|---|
| (2.4, 1.9) — NE | SUCCEEDED | 0 |
| (-2.6, -2.1) — SW | SUCCEEDED | 0 |
| (-2.6, 1.9) — NW | SUCCEEDED | 0 |
| (2.4, -2.1) — SE | SUCCEEDED | 0 |

4개 코너 전부 `number_of_recoveries`가 끝까지 `0`으로 유지됐다 — 어떤 복구 행동도 트리거되지 않았고, 오실레이션이나 정지 없이 매끄럽게 도착해 종료했다.

## 4. 예상/실제 결과 확인

- 열린 공간 목표: 성공해야 한다. (확인됨)
- 코너 근접 목표 4개: **vacuum 프로젝트와 같은 wedging(끼임)이 재현될 것인가가 이 트랙 전체의 핵심 질문이었다.** 결과: **재현되지 않음** — 4개 코너 모두 recovery 없이 깨끗하게 성공.

## 5. 결론 — 이 트랙 전체의 결론

`isaacsim-ros2-advanced`(vacuum) 프로젝트의 Part D에서 겪은 벽/코너 wedging은, 같은 시나리오(6m×5m 방, 코너 근접 목표, 표준 Nav2 MPPI 스택)를 **검증된 표준 로봇 에셋(carter_v1: 사전 리깅된 진짜 wheel joint, primitive 콜라이더)**으로 재현했을 때는 전혀 나타나지 않았다.

이는 vacuum 프로젝트에서 이미 여러 세션에 걸쳐 문서화된 로봇 자체의 문제들 — 특히 [[isaacsim_ros2_advanced_curriculum]] Topic 2/5에서 반복적으로 확인된 **Convex Hull이 곡면 쉘의 오목한 부분을 채워버려 로봇 풋프린트가 실제보다 훨씬 뭉툭해지는 콜라이더 버그**, 그리고 원래 실제 wheel joint가 없어 `physics:velocity`를 직접 write하는 **teleport 방식 드라이브트레인**(가감속/관성이 물리적으로 정직하지 않음) — 이 wedging의 실질적 원인이었을 가능성에 무게를 싣는 결과다. Nav2 자체의 기본 설정(MPPI, 코스트맵, 인플레이션)은 표준 크기의 차동구동 로봇에 대해 코너 근접 목표를 문제없이 처리한다.

**다음 행동**: 이 결론을 vacuum 프로젝트로 가져가, Nav2 파라미터를 더 튜닝하기보다는 **로봇 자신의 콜라이더/드라이브트레인 쪽을 우선 재점검**하는 방향으로 [[isaacsim_ros2_advanced_curriculum]]의 다음 세션을 잡는다.

## 6. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| `ros2 action send_goal /navigate_to_pose ...` → `Goal was rejected` | `lifecycle_manager_navigation`이 코스트맵 활성화 타임아웃으로 브링업 자체를 중단(`bt_navigator`가 inactive 상태) | 초기 포즈를 브링업 직후 몇 초 안에 반드시 전달. `timeout N ros2 topic pub -r 2 /initialpose ...`로 브링업과 동시에 몇 초간 반복 발행해 타이밍 경쟁을 없앰 (3.2절) |
| `global_costmap`가 `Timed out waiting for transform from base_link to map` 반복, 결국 `Failed to activate global_costmap` | `base_link`→`map` TF는 AMCL이 초기 포즈를 받아야만 발행하기 시작하는데, 그 전에 코스트맵의 대기 시간(하드코딩된 값, 파라미터로 못 늘림)이 먼저 끝나버림 | 위와 동일 — 초기 포즈를 최대한 빨리 전달하는 것 외엔 근본 해결책이 없음(재시도 로직이 없어 한 번 실패하면 프로세스를 완전히 재시작해야 함) |
| `ros2 launch ... map:=~/... params_file:=~/...`에서 `[Errno 2] No such file or directory: '~/isaac_assets/...'` — `~`가 문자 그대로 전달됨 | `이름:=값`은 `이름` 부분에 콜론이 섞여 있어 셸이 유효한 변수 대입문(`name=value`)으로 인식하지 못함 — 셸의 "대입문에서 `=` 뒤 `~`는 확장" 규칙이 적용 안 돼서 `~`가 그대로 리터럴 문자로 넘어감 | `~` 대신 `$HOME`(변수 치환이라 어디서든 확장됨)이나 전체 절대경로를 씀. vacuum 프로젝트에서도 이미 겪었던 것과 같은 함정([[isaacsim_ros2_advanced_curriculum]]) |

## 7. 체크포인트

- [x] `nav2_params.yaml`을 이 트랙의 실제 프레임(`base_link`)과 로봇 크기(`robot_radius: 0.35`)에 맞춰 준비
- [x] `nav2_bringup` 완전 활성화(`lifecycle_manager_localization`/`lifecycle_manager_navigation` 둘 다 `Managed nodes are active`) 확인
- [x] 열린 공간 목표 성공 확인
- [x] 4개 코너 근접 목표 전부 성공, `number_of_recoveries: 0` 확인 — **wedging 미재현**
- [x] 결론 도출: wedging은 Nav2/설정 문제가 아니라 vacuum 프로젝트 로봇 자체(콜라이더/드라이브트레인)의 문제일 가능성이 높음

---
이 트랙(`isaacsim-standard-nav2`, 5개 토픽)은 여기서 완료. 다음 행동은 [[isaacsim_ros2_advanced_curriculum]](vacuum 프로젝트) 쪽에서, 이 결론을 바탕으로 콜라이더/드라이브트레인을 재점검하는 것.
