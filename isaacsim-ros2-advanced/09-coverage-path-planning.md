# 09. 커버리지 경로 계획 (CPP)

> 진행 상태: **진행 중** — Nav2 스택 기동, 코스트맵 튜닝, 커스텀 Boustrophedon 플래너 작성까지 완료. 이 세션 최대 성과는 로봇의 회전 응답이 애초에 잘못돼 있던 근본 원인(라디안/도 단위 불일치)을 찾아 고친 것. Nav2 액션(`navigate_to_pose`)이 목표를 거부/무응답하는 문제가 아직 남아 다음 세션으로 이월.

## 1. 학습 목표

- Nav2 코스트맵(costmap)을 이 로봇의 실제 크기(~0.35m)와 방 크기(~6m×5m)에 맞게 튜닝한다 (`inflation_radius`).
- Nav2의 기본 내비게이션 스택(경로 계획+제어)이 이 프로젝트의 `/cmd_vel` 기반 커스텀 구동부(Joint Drive 방식, [[isaacsim_ros2_advanced_curriculum]])와 문제없이 맞물리는지 확인한다.
- Boustrophedon(왕복/지그재그) 패턴으로 방 전체를 훑는 **커스텀 CPP(Coverage Path Planning) 플래너**를 직접 작성한다 — 표준 Nav2엔 "한 지점까지 가기"만 있고 "전체 영역을 빠짐없이 훑기"는 없어서 직접 만들어야 한다.

## 2. 핵심 개념

**Coverage Path Planning vs 일반 내비게이션**: 일반적인 Nav2 내비게이션(`NavigateToPose`)은 "여기서 저기까지 장애물 피해서 가라"가 목표다. 로봇청소기는 다르다 — 목표 지점이 없고, **바닥의 빈 공간을 전부 덮어야** 한다. 이건 Nav2가 기본 제공하지 않는 별도 문제라, 커버리지 경로(웨이포인트 시퀀스)를 직접 계산해서 Nav2에 순서대로 넘겨주는 방식으로 구현한다.

**Boustrophedon 패턴**: "소가 밭을 가는 방향(황소, boustrophedon = 그리스어로 '황소가 도는 방식')"이라는 어원처럼, 평행한 직선을 왕복하며 영역을 빈틈없이 덮는 가장 단순하고 널리 쓰이는 커버리지 패턴. 볼록하고 장애물이 적은 공간(지금 우리 방처럼)에 특히 효과적이다. 알고리즘: (1) 점유 격자 지도에서 빈 공간의 경계 상자를 구한다 (2) 로봇 폭(청소 스와스 너비)만큼 간격을 두고 평행선을 여러 개 긋는다 (3) 각 선의 양 끝점을 지그재그 순서로 이어서 웨이포인트 리스트를 만든다.

**`inflation_radius` 튜닝이 필요한 이유**: Nav2 기본값(`0.7`)은 TurtleBot 같은 30cm급 로봇+넓은 공간 기준. 이 프로젝트는 로봇이 더 작고(~0.35m) 방도 훨씬 작다(~6m×5m) — 기본값을 그대로 쓰면 코스트맵이 벽 주변을 너무 넓게 "위험 지역"으로 부풀려서 방의 상당 부분이 내비게이션 불가능 영역으로 잡힐 수 있다. `robot_radius`(~0.2m)에 맞춰 `inflation_radius`를 `0.25` 정도로 낮춤 (`nav2_params.yaml`).

**이 프로젝트에 필요한 프레임 대응**: 표준 Nav2는 `map`→`odom`→`base_link` TF 체인과 AMCL(파티클 필터 localization)을 기대한다. 이 프로젝트는 [[isaacsim_ros2_advanced_curriculum]]에서 이미 확인했듯 실제 바퀴 오도메트리가 없고, `odom`/`base_link`란 이름의 프레임도 없다. Topic 8에서 이미 실제 프레임 이름(`world`/`robot1`)에 맞춰 slam_toolbox를 붙여놨으므로, **AMCL이나 별도 map_server 없이 Topic 8의 slam_toolbox(mapping 모드)를 그대로 켜둔 채로** `/map`과 `map→world` TF를 공급받는다 — Nav2 쪽 파라미터도 `odom_frame`/`base_frame` 자리에 `world`/`robot1`을 그대로 대입한다 (`nav2_params.yaml`).

## 3. 실습 단계

### 3.1 사전 준비 (완료)

- `~/isaac_assets/vacuum_robot/config/nav2_params.yaml` — `nav2_bringup`의 기본 `nav2_params.yaml`에서 AMCL/docking/route_server/collision_monitor 등 이 프로젝트에 필요 없는 섹션을 걷어내고, `robot_base_frame: robot1`, `local_costmap.global_frame: world`, `global_costmap.global_frame: map`, `robot_radius: 0.2`, `inflation_radius: 0.25`로 맞춤.
- Topic 8의 slam_toolbox(mapping 모드)가 계속 실행 중인지 확인 (`ros2 node list`에 `/slam_toolbox`).

### 3.2 Nav2 내비게이션 스택 실행

```bash
conda deactivate
source /opt/ros/jazzy/setup.bash
ros2 launch nav2_bringup navigation_launch.py \
  use_sim_time:=true \
  params_file:=/home/pw/isaac_assets/vacuum_robot/config/nav2_params.yaml
```

`map_server`/`amcl`은 이 launch 파일에 원래 없음 (별도 `localization_launch.py`에만 있음) — Topic 8의 slam_toolbox가 그 역할을 대신하므로 딱 맞는 구성.

**막힘 → `nav2_bringup`의 기본 launch 대신 커스텀 minimal launch 필요 (3.3절 참고)**: `navigation_launch.py`를 그대로 쓰면 lifecycle bring-up이 `collision_monitor`(이 프로젝트 파라미터 파일에 없는 설정을 요구하며 configure 실패) 하나 때문에 **전체가 중단**됐다 — lifecycle_manager는 관리 노드 중 하나라도 configure 실패하면 전부 abort. `route_server`/`docking_server`/`velocity_smoother`도 이 프로젝트엔 불필요. `~/isaac_assets/vacuum_robot/config/nav2_minimal_launch.py`를 직접 작성해서 실제 필요한 6개(controller/planner/smoother/behavior/bt_navigator/waypoint_follower)만 관리하도록 함.

### 3.3 Nav2 minimal launch (완료)

```bash
conda deactivate
source /opt/ros/jazzy/setup.bash
ros2 launch /home/pw/isaac_assets/vacuum_robot/config/nav2_minimal_launch.py \
  use_sim_time:=true \
  params_file:=/home/pw/isaac_assets/vacuum_robot/config/nav2_params.yaml
```

**두 번째 막힘 → `controller_server`가 lifecycle 활성화 요청에 응답 없음**: `ros2 lifecycle get /controller_server`가 계속 타임아웃, Nav2 로그엔 `local_costmap: Timed out waiting for transform from robot1 to world to become available, tf error: Invalid frame ID "world" ... frame does not exist`가 반복 출력. `/tf`를 직접 찍어보니 **Publisher count: 2, 전부 slam_toolbox** — Isaac Sim(`CmdVelGraph`의 `TfPublish`)가 `/tf`를 전혀 발행하고 있지 않았음. 원인은 단순 — **Isaac Sim이 Stop 상태였음**. Play를 누르자 `TfPublish`가 다시 발행을 시작했고 `controller_server`를 포함해 6개 노드 전부 즉시 `active`로 전환됨.

### 3.4 단일 목표 테스트 → 로봇이 완전히 엉뚱한 방향으로 튐 (심각한 버그 발견)

`ros2 action send_goal /navigate_to_pose ...`로 `(1.0, 0.0)` 목표를 보내자 로봇이 목표와 무관하게 계속 다른 방향으로 이동 — `/odom` 위치가 `-0.02 → -0.53 → 3.68`처럼 튀거나, `/cmd_vel`은 정상적인 값(`lin.x≈0.17, ang.z≈0.03`)을 계속 보내는데 로봇 위치가 몇 초씩 완전히 얼어붙는 등 종잡을 수 없는 증상.

**원인 1 — Topic 3 브러시 조인트가 깨진 채로 다시 활성화됨**: Kit 로그에 `PhysicsUSD: CreateJoint - found a joint with disjointed body transforms ... /World/robotvacum_decimated/robot1/RevoluteJoint`가 매 Play마다 반복. 이 경로를 조사해보니 [[isaacsim_ros2_advanced_curriculum]] Topic 3에서 만든 **브러시 Cylinder의 Angular Drive 조인트**였고, `physics:localPos0`가 `(-1142.7346, -3257.1802, 119.63816)`처럼 물리적으로 말이 안 되는 값으로 손상되어 있었다 (오늘 여러 번의 저장/재시작 과정에서 깨진 것으로 추정, 정확한 시점은 특정 못함). 로봇 본체(`robot1`)에 물린 이 깨진 조인트가 로봇 전체 동역학을 방해하고 있었음. **해결**: `physics:jointEnabled = False`로 비활성화 (브러시 회전은 이 세션에 필요 없음 — Part D에서 다시 필요하면 조인트를 정상 값으로 재생성해야 함, 미해결 채무로 남김).

브러시 조인트 비활성화 후 단일 목표 재시도 → **`SUCCEEDED`**, 위치도 목표 근처로 정상 도달. 여기서 일단 "고쳐졌다"고 판단했으나 Boustrophedon 다중 웨이포인트 실행에서 더 심각한 문제가 다시 드러남 (3.5절).

### 3.5 커스텀 Boustrophedon 플래너 작성 (완료, 실제 커버리지 주행은 미완료)

`~/isaac_assets/vacuum_robot/scripts/boustrophedon_cpp.py` — Topic 8에서 저장한 지도(`.pgm`+`.yaml`)를 파싱해서 빈 공간(픽셀값 ≥250, nav2 `map_saver` 컨벤션상 흰색=자유공간) 경계 상자를 구하고, 로봇 지름(~0.36m) 간격으로 왕복 웨이포인트를 생성, `nav2_simple_commander`의 `followWaypoints()`로 실행. 웨이포인트 생성 로직만 독립적으로 테스트해서 올바른 지그재그 패턴이 나오는 것 확인 (첫 시도 22~26개 웨이포인트, X/Y가 번갈아 왕복).

**막힘 → 첫 웨이포인트가 벽 코너에 너무 붙어서 로봇이 물리적으로 끼임**: `MARGIN=0.3`(벽에서 떨어질 최소 거리)으로 계산한 첫 웨이포인트가 서쪽 벽과 북쪽 벽이 만나는 코너에서 각각 0.4m 정도밖에 안 떨어져 있었음(로봇 반경+inflation 여유와 거의 같은 수준) — 로봇이 그 쪽으로 이동하다 코너에 낀 채 완전히 정지(`/odom` 위치가 소수점까지 몇 분간 동일). `MARGIN`을 `0.6`으로 올려서 재생성 (22개 웨이포인트, 벽에서 더 여유있게).

### 3.6 회전 응답이 근본적으로 고장나 있었던 것 발견 및 수정 (이 세션의 핵심 성과)

새 첫 웨이포인트로도 Nav2가 계속 목표와 무관한 방향으로 로봇을 보냄 (`(-0.83, 1.76)` 목표인데 `(1.31, 2.97)`처럼 완전히 다른 곳으로). 3.4절의 "고쳐졌다"는 판단은 **틀렸음** — 브러시 조인트 비활성화로 일부 증상은 나아졌지만 진짜 근본 원인은 따로 있었다.

직접 진단: `cmd_vel`로 `angular.z=0.5`(rad/s)를 3초간 지속 발행해도 `/odom`의 orientation(쿼터니언)이 거의 안 바뀜(오차 수준). Isaac Sim 내부에서 `robot1`/`YawPivot`의 **raw USD 쿼터니언을 직접 읽어도** 동일 — ROS 변환 계산 문제가 아니라 실제 물리 레벨에서 회전이 거의 안 일어나고 있었음.

**원인**: USD Physics의 `PhysicsRevoluteJoint` Drive `targetVelocity`는 **도(degree)/초** 단위인데, `CmdVelGraph`의 `WriteAngularDrive`는 ROS `Twist.angular.z`(**라디안/초**)를 변환 없이 그대로 써넣고 있었다 ([[isaacsim_ros2_advanced_curriculum]] Topic 6에서 처음 만들 때부터 이 변환이 빠져 있었음 — 브러시의 Angular Drive는 애초에 `300 deg/s`처럼 도 단위로 직접 입력했었어서 이 문제를 겪지 않았다). 그 결과 일반적인 `angular.z`(0.2~0.8 rad/s)가 도 단위로는 너무 작아 거의 회전이 안 일어났고, Nav2의 MPPI 컨트롤러는 로봇이 원하는 만큼 안 도는 것을 계속 더 큰 명령으로 "보정"하려 했지만 여전히 부족해 결국 방향을 거의 못 틀고 원래 향하던 쪽으로 계속 표류 — 목표와 무관하게 엉뚱한 곳으로 가는 증상과 정확히 일치.

디버깅 중 `damping`(1000 → 20000 → 500000)과 `physics:diagonalInertia`(YawPivot은 충돌체가 없어 자동 관성 계산이 안 되어 기본값이 `(0,0,0)`인 것도 같이 의심하고 직접 계산해 넣어봄, `(2/5)mr²` 소구체 근사)도 여러 차례 조정했으나 **이 중 어느 것도 근본 원인이 아니었다** — 나중에 단위를 고치고 나니 `damping`은 오히려 낮은 편(수천 단위)이 적당했다.

**해결**: `CmdVelGraph`에 `AngularDegConvert`(`omni.graph.scriptnode.ScriptNode`) 노드를 추가 — 매 프레임 `CmdVelSubscriber.outputs:angularVelocity.z`(rad/s)를 읽어 `× 180/π`로 변환한 뒤 `WriteAngularDrive.inputs:value`에 `og.Controller.set()`으로 직접 기록 (`BreakAngular.outputs:z`의 직접 연결은 제거 — `OdomCompute`와 동일한 "값만 바꿀 땐 다운스트림 입력에 직접 쓰기" 패턴). `damping`은 `3000`으로 재설정. 직접 `cmd_vel` 테스트로 검증: `angular.z=0.3`(rad/s) 2초 명령 시 기존엔 약 1°만 돌았는데, 수정 후 약 71° 회전(기대치 34°의 오차 범위 내, 자릿수 완전히 일치) — 확실한 개선.

### 3.7 물리 상태 손상 사고 2건 (교훈으로 기록)

디버깅 과정에서 극단적인 테스트 값을 쓰다가 두 차례 물리 상태가 심각하게 깨짐:
1. `targetVelocity=50`(도/초가 아니라 실수로 매우 큰 값)으로 직접 테스트 → Play 누르자마자 로봇이 격렬하게 회전하며 **벽을 뚫고 튕겨나감** (얇은 벽(0.1m)과 빠른 회전 속도 조합으로 한 타임스텝 안에 충돌 처리가 안 된 것으로 추정 — 터널링).
2. 이후 재테스트 중 로봇이 **바닥을 뚫고 2km 넘게 자유낙하**하는 상태까지 감 (`/odom.z`가 Isaac Y좌표로 `-2169` — Prismatic Joint의 수직 고정이 격렬한 충격으로 깨진 것으로 추정).

**해결**: Isaac Sim에서 저장 전 상태로 되돌림(재시작 후 이전 저장 시점 스테이지 재로드) — 이 과정에서 이미 적용했던 `diagonalInertia`/`damping` 조정과 GUI로 새로 만든 `RevoluteJoint`도 같이 날아가서, 브러시 조인트 비활성화와 `RevoluteJoint` GUI 재생성을 다시 수행해야 했다. **교훈**: 조인트 드라이브 테스트는 절대 큰 값(수십 단위)으로 먼저 하지 말 것 — 작은 값(0.3 이하)으로 시작해서 점진적으로 올릴 것. 이번처럼 물리가 깨지면 스크립트로 되돌리려 하지 말고 **저장 시점으로 되돌리는 게 가장 빠르고 안전**하다 (단, 그 이후의 미저장 수정은 전부 다시 해야 함 — Ctrl+S 습관의 중요성이 다시 확인됨, [[isaacsim_ros2_advanced_curriculum]]).

### 3.8 미해결: Nav2 액션이 목표를 거부하거나 응답 없음

회전 버그를 고친 뒤 `navigate_to_pose`/`compute_path_to_pose` 액션에 새 목표를 보내면 `Goal was rejected` 또는 응답 없이 타임아웃되는 증상이 남아있다. 로봇이 매핑된 자유공간 밖(예: 북쪽 벽 근처)에 있을 때 확실히 거부되는 것은 확인했지만, 방 중앙의 안전한 위치에서도 거부/무응답이 재현되어 원인 미확정 — 코스트맵이 유효한 데이터를 못 받고 있거나, 2시간 반 넘게 켜둔 세션의 시스템 부하 문제일 가능성. `nav2_simple_commander`의 `BasicNavigator.waitUntilNav2Active(localizer="slam_toolbox")`도 응답 없이 멈추는 것을 별도로 확인 — CLI(`ros2 action send_goal`)로 우회 가능하다는 것만 확인.

## 4. 예상/실제 결과 확인

- Nav2 minimal launch: 6개 노드(controller/planner/smoother/behavior/bt_navigator/waypoint_follower) 전부 `active` 확인.
- 회전 버그 수정 전/후 직접 비교: 수정 전 `angular.z=0.5`(rad/s) 3초 명령에도 orientation 쿼터니언 변화 없음(측정 불가 수준) / 수정 후 `angular.z=0.3` 2초에 약 71° 회전 (기대치 34°와 자릿수 일치).
- 단일 목표(`(0,0)`, 회전 거의 불필요)는 브러시 조인트 비활성화만으로 `SUCCEEDED` 했었으나, 회전이 크게 필요한 목표들은 회전 버그를 고치기 전까진 전부 실패.
- 커스텀 Boustrophedon 웨이포인트 생성 로직은 지도 데이터 기반으로 정상 동작 확인 (22개 웨이포인트, 올바른 지그재그 좌표).
- 전체 26/22 웨이포인트 커버리지 주행은 위 3.8절 문제로 아직 끝까지 실행 못함.

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| `navigation_launch.py` 기동 시 lifecycle bring-up이 중간에 abort | `collision_monitor`가 파라미터 파일에 없는 설정(`observation_sources`)을 요구하며 configure 실패, lifecycle_manager는 하나라도 실패하면 전체 중단 | 필요한 6개 노드만 관리하는 커스텀 `nav2_minimal_launch.py` 작성 (3.3절) |
| `controller_server`가 lifecycle 질의에 응답 없음, `local_costmap: ... frame "world" does not exist` 반복 | Isaac Sim이 Stop 상태라 `TfPublish`가 `/tf`를 전혀 발행하지 않음 | Play (3.3절) |
| 목표를 보내면 로봇이 무관한 방향으로 이동/제자리에서 얼어붙음 | (표면적 원인) Topic 3 브러시 조인트가 손상된 값(`localPos0`가 -1000단위)으로 재활성화되어 로봇 동역학 방해 | `jointEnabled=False`로 비활성화 (3.4절) — **근본 원인은 아니었음, 3.6절 참고** |
| 브러시 조인트를 꺼도 회전이 필요한 목표는 계속 실패, `cmd_vel angular.z`를 줘도 orientation이 거의 안 바뀜 | `WriteAngularDrive`가 라디안(ROS)을 도(USD Physics RevoluteJoint 단위) 변환 없이 그대로 write | `AngularDegConvert` ScriptNode 추가, `×180/π` 변환 (3.6절) — **이게 진짜 근본 원인** |
| 첫 Boustrophedon 웨이포인트에서 로봇이 완전히 멈춰 몇 분간 위치 불변 | 웨이포인트가 벽 코너에서 ~0.4m밖에 안 떨어져 로봇이 물리적으로 낌 | `MARGIN`을 `0.3`→`0.6`으로 확대 (3.5절) |
| `targetVelocity`를 극단적으로 큰 값(50)으로 테스트하다 로봇이 벽을 뚫고 나가고, 이후 바닥까지 뚫고 2km 자유낙하 | 얇은 벽(0.1m)과 큰 회전 속도로 인한 물리 터널링, 이후 충격이 Prismatic Joint의 수직 고정까지 깨뜨림 | 저장 시점으로 스테이지 되돌림 (되돌리며 미저장 수정 전부 소실, 다시 적용해야 했음) — 앞으로 조인트 드라이브는 항상 작은 값(≤1)으로 먼저 테스트 (3.7절) |
| 회전 버그 수정 후에도 `navigate_to_pose`/`compute_path_to_pose`가 목표를 거부하거나 무응답 | 미확정 — 코스트맵 데이터 문제 또는 장시간 세션의 시스템 부하로 추정 | **다음 세션 숙제** (3.8절) |

## 6. 체크포인트

- [x] Nav2 내비게이션 스택 정상 기동 (lifecycle activate 확인, 커스텀 minimal launch로)
- [x] 코스트맵 `inflation_radius` 등 로봇/방 크기에 맞게 튜닝
- [x] 회전 응답 근본 버그(라디안/도 단위 불일치) 발견 및 수정 — 직접 cmd_vel 테스트로 검증
- [x] 커스텀 Boustrophedon 플래너 작성, 웨이포인트 생성 로직 검증
- [ ] RViz/CLI에서 단일 목표 지정 시 로봇이 안정적으로 도달 — **회전 버그 수정 후 재검증 필요, Nav2 액션 거부 문제로 미완료**
- [ ] 웨이포인트 시퀀스를 Nav2에 넘겨 방 전체를 커버리지 주행

---
다음: Nav2 액션 거부/무응답 원인 조사 (3.8절) → 회전 버그 수정된 상태로 단일 목표 재검증 → Boustrophedon 전체 실행
