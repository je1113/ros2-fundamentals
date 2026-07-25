# 03. ROS2 브릿지 & OmniGraph 연결

## 1. 학습 목표

- `isaacsim.ros2.bridge`의 `ROS2 Subscribe Twist`로 `/cmd_vel`을 구독하고, `Differential Controller` + `Articulation Controller` 조합으로 실제 조인트를 구동하는 표준 패턴을 익힌다.
- Twist 메시지의 `vector3` 필드에서 필요한 축 성분만 뽑아 스칼라 입력에 연결하는 방법(`Break Vector3`)을 익힌다.
- OmniGraph 노드의 Property 패널 값 표시를 맹신하지 않고, `og.Controller`/`omni.graph.core`로 실제 저장된 속성값을 직접 검증하는 디버깅 습관을 기른다.
- `Isaac Compute Odometry Node` + `ROS2 Publish Odometry`/`ROS2 Publish Raw Transform Tree`/`ROS2 Publish Clock`으로 시뮬레이션 시간 기준 odom/tf/clock을 퍼블리시하는 표준 패턴을 익힌다.
- RTX Lidar 기반 센서를 ROS2로 퍼블리시할 때 render product(Replicator) 자원이 어떻게 생성/캐싱되는지 이해하고, 이 세션에서 만난 상위 버전(Kit 6.0) 특유의 한계를 기록한다(3-C, 미해결).

## 2. 진행 상태 요약

| 하위 토픽 | 내용 | 상태 |
|---|---|---|
| 3-A | `/cmd_vel` → Differential/Articulation Controller 바퀴 구동 | 완료 |
| 3-B | Odometry/TF/Clock 퍼블리시 | 완료 |
| 3-C | LiDAR(RTX) 퍼블리시 | **미해결 — 다음 세션** |

## 2. 핵심 개념

**Carter가 vacuum 프로젝트보다 훨씬 단순한 이유**: `isaacsim-ros2-advanced` 트랙의 로봇청소기는 실제 바퀴 조인트가 없어서(Topic 6, [[isaacsim_ros2_advanced_curriculum]]) `physics:velocity`/`physics:angularVelocity`를 로봇 바디에 매 프레임 직접 write하는 우회 방식을 써야 했다. Carter는 `chassis_link`↔`left_wheel_link`/`right_wheel_link` 사이에 진짜 `RevoluteJoint`+`DriveAPI`가 있어서(Topic 2), Isaac Sim이 정확히 이 상황을 위해 제공하는 `Differential Controller`(Twist → 좌/우 바퀴 각속도 변환) + `Articulation Controller`(조인트 이름으로 실제 드라이브에 값 write) 조합을 그대로 쓸 수 있다. 이게 훨씬 "정석"에 가까운 워크플로우다.

**타입 불일치와 Break Vector3**: `ROS2 Subscribe Twist`의 `Angular Velocity`/`Linear Velocity` 출력은 Twist 메시지의 `angular`/`linear` 필드 전체(`double3` 벡터)다. 반면 `Differential Controller`의 `Desired Linear/Angular Velocity` 입력은 스칼라 `double`이다. 이 스테이지는 **Z-up**이므로(vacuum 프로젝트의 Y-up과 다름, 3.x절 참고) `angular.z`(요 회전율), `linear.x`(전진 속도) 성분만 있으면 되고, `Break Vector3` 노드로 벡터를 x/y/z로 쪼갠 뒤 필요한 성분만 연결한다.

**Exec 핀이 없는 순수 데이터 노드**: `Differential Controller`는 `Exec In`/`Exec Out`이 없는 순수 데이터 변환 노드다. Exec 체인은 이 노드를 건너뛰어 `ROS2 Subscribe Twist`의 `Exec Out`에서 바로 `Articulation Controller`의 `Exec In`으로 이어야 한다. `Differential Controller`는 exec 체인과 무관하게 매 평가마다 자기 입력 연결로부터 값을 계산해 데이터 핀으로 흘려보낸다.

**조인트 이름 vs 링크 이름**: `Articulation Controller`의 `Joint Name` 배열에는 링크 이름(`left_wheel_link`)이 아니라 **조인트 prim 이름**(`left_wheel`, `right_wheel`)을 넣어야 한다. Topic 2에서 만든 물리 구조 덤프(`carter_physics_dump.txt`)가 정확한 조인트 이름의 출처다.

## 3. 실습 단계

### 3.1 그래프 골격 구성

1. `Window` → `Visual Scripting` → `Action Graph`에서 새 그래프 생성 (`/World/CarterGraph`)
2. `On Playback Tick` 추가
3. `ROS2 Subscribe Twist` 추가, `On Playback Tick.outputs:tick` → `ROS2 Subscribe Twist.inputs:execIn` 연결. Topic Name은 기본값 `cmd_vel` 확인.

### 3.2 Twist → 바퀴 각속도 변환

1. `Differential Controller` 추가
2. `Break Vector3` 두 개 추가
3. `ROS2 Subscribe Twist.outputs:angularVelocity` → Break Vector3 #1 입력 → **z** 출력 → `Differential Controller.inputs:angularVelocity`
4. `ROS2 Subscribe Twist.outputs:linearVelocity` → Break Vector3 #2 입력 → **x** 출력 → `Differential Controller.inputs:linearVelocity`
5. `On Playback Tick`의 `Delta Seconds` 출력 → `Differential Controller.inputs:dt`
6. `Differential Controller`의 `Wheel Radius`/`Wheel Distance`와 `Max Acceleration`/`Max Angular Acceleration`/`Max Linear Speed`/`Max Wheel Speed`/`Max Angular Speed`/`Max Deceleration`을 모두 명시적으로 채운다 (5.1절 참고 — 이 노드는 미입력 시 기본값이 전부 `0`이라 값을 안 넣으면 조용히 안 움직인다).

Wheel Radius/Wheel Distance 실측값(0.24m / 0.6284m)은 GUI에서 손으로 스케일×회전을 계산하지 않고, `UsdGeom.Cylinder`의 `radius`/`axis` 속성과 `ComputeLocalToWorldTransform`으로 로컬 원 위의 점들을 월드로 변환해 중심으로부터의 거리로 직접 측정했다(스크립트: `measure_carter_wheel_radius_2026-07-25.py`). 4개 샘플 포인트 모두 정확히 0.24000으로 일치해 신뢰도를 확보했다.

### 3.3 바퀴 각속도 → 조인트 드라이브

1. `Articulation Controller` 추가
2. `ROS2 Subscribe Twist.outputs:execOut` → `Articulation Controller.inputs:execIn` (Differential Controller를 건너뜀, 2절 참고)
3. `Differential Controller.outputs:velocityCommand` → `Articulation Controller.inputs:velocityCommand`
4. `Articulation Controller.inputs:targetPrim` = `/World/carter_v1`
5. `Articulation Controller.inputs:jointNames` = `["left_wheel", "right_wheel"]` (순서 중요 — Differential Controller 출력이 [left, right] 순서)

### 3.4 검증 (3-A)

Play 후, 별도 터미널(conda 비활성화)에서 `/cmd_vel`을 퍼블리시해 확인한다:

```bash
source ~/anaconda3/etc/profile.d/conda.sh; conda deactivate; conda deactivate; conda deactivate; hash -r
ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.2}, angular: {z: 0.0}}"
```

### 3.5 Odometry 계산 및 퍼블리시 (3-B)

1. `Isaac Read Simulation Time` 추가 — Exec 핀이 없는 순수 데이터 노드. 이후 모든 Publish 노드의 Timestamp 입력에 이 출력을 물린다.
2. `Isaac Compute Odometry Node` 추가. `On Playback Tick`의 Tick에서 **직접** 분기해 이 노드의 Exec In에 연결한다(`ROS2 Subscribe Twist`를 거치지 않음 — tick 소스에서 바로 팬아웃하는 게 안전하다는 vacuum 프로젝트의 교훈을 재사용).
3. `Chassis Prim` 입력에 `/World/carter_v1/chassis_link`(루트 `carter_v1`이 아니라 실제 RigidBody) 지정.

### 3.6 Odometry 퍼블리시

1. `ROS2 Publish Odometry` 추가.
2. `Isaac Compute Odometry Node`의 Exec Out → 이 노드의 Exec In.
3. `Position`/`Orientation`/`Linear Velocity`/`Angular Velocity` 출력 → 같은 이름의 입력들에 연결.
4. `Isaac Read Simulation Time`의 Simulation Time → Timestamp.
5. `Odom Frame Id` = `odom`, `Chassis Frame Id` = `base_link`(실제 prim 이름은 `chassis_link`지만, TF 프레임 이름은 prim 이름과 무관하게 ROS 표준 이름을 그대로 써도 된다).

### 3.7 TF 퍼블리시

1. `ROS2 Publish Raw Transform Tree` 추가 (prim 관계가 아니라 translation/rotation 값을 직접 받는 노드라, 실제 prim 이름과 무관하게 원하는 프레임 이름을 쓸 수 있다).
2. `Isaac Compute Odometry Node`의 Exec Out에서 팬아웃 → 이 노드의 Exec In.
3. `Position`/`Orientation` → `Translation`/`Rotation`.
4. `Isaac Read Simulation Time`의 Simulation Time → Timestamp.
5. `Parent Frame Id` = `odom`, `Child Frame Id` = `base_link`.

### 3.8 Clock 퍼블리시

1. `ROS2 Publish Clock` 추가.
2. `On Playback Tick`의 Tick에서 팬아웃 → Exec In.
3. `Isaac Read Simulation Time`의 Simulation Time → Timestamp.

### 3.9 검증 (3-B)

```bash
ros2 topic list   # /clock /cmd_vel /odom /parameter_events /rosout /tf
ros2 topic hz /odom /tf /clock   # 각각 ~50Hz
ros2 topic echo /odom --once     # frame_id: odom, child_frame_id: base_link
```

### 3.10 LiDAR 퍼블리시 시도 (3-C, 미해결)

`chassis_link`에 이미 장착된 `XT_32_10Hz`는 32채널 3D LiDAR(elevation -16°~+15°)라 `type=laser_scan`(2D)에 쓸 수 없다(Topic 2, 2절 참고 — vacuum 프로젝트의 RTX Lidar 2D/3D 프로파일 함정과 동일 계열). 별도 2D LiDAR를 `isaacsim.sensors.rtx.LidarRtx` 클래스로 추가 장착 시도:

1. `config_file_name="Example_Rotary_2D"`로 첫 시도 → **이 설치본(Isaac Sim Kit 6.0)에는 해당 설정 파일이 없어** 알 수 없는 값으로 조용히 대체됨(128채널, 4방향 azimuth 클러스터라는 이상한 형태로 생성됨).
2. 실제 존재하는 2D 프로파일인 `config_file_name="RPLIDAR_S2E"`(SLAMTEC RPLidar S2E, `lidar_configs/SLAMTEC/RPLIDAR_S2E.json`)로 재시도 → 성공했지만, 이 프로파일은 프리미티브 센서 prim 하나가 아니라 `materials`+`mesh`+실제 센서(`RPLidar_S2E`, `OmniLidar` 타입)로 구성된 서브트리를 만든다 — `Isaac Create Render Product`의 `Camera Prim`은 감싸는 Xform이 아니라 **중첩된 실제 `OmniLidar` prim**(`.../Lidar2D/RPLidar_S2E`)을 가리켜야 한다.
3. `Camera Prim`을 바로잡은 뒤에도 `ROS2PublishLaserScan: GMO buffer is empty or invalid. Skipping.`이 매 프레임 반복 — 배선은 정확한데 실제 RTX 레이캐스트 결과 버퍼가 계속 비어있음.
4. Isaac Sim을 완전히 재시작해 고아 리소스를 지운 뒤에도 동일 증상 재현. 게다가 재시작 후에는 **렌더 프로덕트가 세션당 사실상 1개로 캡핑된 것처럼 보임** — `Isaac Create Render Product` 노드를 다른 이름으로 여러 번 새로 만들어도(`XT_32_10Hz`를 가리키도록) 로그에 새 `Registered render product` 항목이 전혀 안 찍히고, 항상 맨 처음 등록된 render product(`RPLidar_S2E`용)만 유지됨.

**미해결로 다음 세션에 넘김.** 3-A/3-B는 완전히 검증됐으므로 SLAM(Topic 4)/Nav2(Topic 5)로 넘어가기 전에 LiDAR를 반드시 먼저 해결해야 한다(slam_toolbox가 `/scan`을 필요로 함).

## 4. 예상/실제 결과 확인

- `linear.x=0.2`로 퍼블리시하면 로봇이 앞으로 이동해야 한다. (확인됨)
- `angular.z=0.5`로 퍼블리시하면 제자리(또는 원호)로 회전해야 한다. (확인됨)
- `ros2 topic info /cmd_vel --verbose`에서 `_World_CarterGraph_ros2_subscribe_twist` 노드가 Subscription으로 보여야 한다. (확인됨)
- `/odom`, `/tf`, `/clock`이 각각 ~50Hz로 흐르고, `/odom`/`/tf`의 frame_id가 `odom`→`base_link`로 정확히 찍혀야 한다. (확인됨)
- `/scan`이 `sensor_msgs/LaserScan` 타입으로 advertise되고 실제 메시지가 발행돼야 한다. (**미확인 — GMO buffer 문제로 토픽만 뜨고 데이터 없음**)

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| `outputs:angularVelocity of type double3 (vector) is not compatible with inputs:angularVelocity of type double` 경고 | `ROS2 Subscribe Twist`는 Twist의 `angular`/`linear` 필드 전체(vector3)를 출력하는데, `Differential Controller`는 스칼라를 받음 | `Break Vector3`로 벡터를 분해해 필요한 성분(이 스테이지는 Z-up이므로 각속도는 `z`, 선속도는 `x`)만 연결 |
| `Differential Controller`에 `Exec Out` 핀 자체가 없어서 `Articulation Controller`로 exec 체인을 이어갈 수 없음 | 이 노드는 exec 없는 순수 데이터 변환 노드로 설계됨 | `ROS2 Subscribe Twist`의 `Exec Out`에서 `Differential Controller`를 건너뛰고 바로 `Articulation Controller`의 `Exec In`으로 연결 |
| `Max Acceleration`/`Max Angular Acceleration`/`Max Linear Speed`/`Max Wheel Speed`/`Max Deceleration`을 기본값(전부 `0`)으로 두면, `Velocity Command` 출력이 목표값(~0.83rad/s)이 아니라 거의 0에 가까운 극소값에 고정되고 시간이 지나도 램프업되지 않음 | 이 노드에서 `0`은 "무제한"이 아니라 문자 그대로 "가/감속·속도 상한 0"으로 해석돼, 사실상 아무 속도 변화도 허용하지 않음 | 모든 Max 계열 필드에 여유 있는 명시적 값(가속도 5.0, 속도 상한 2.0~10.0 등)을 입력 |
| Property 패널에 `Wheel Radius`를 `0.24`로 입력했다고 표시됐는데, `Velocity Command`가 계속 `linearVelocity / 324`에 해당하는 극소값(`0.000617`)으로 나옴 | GUI 숫자 입력 필드가 실제로는 `324.0`을 저장하고 있었음(오타 또는 입력 UI의 알 수 없는 변환) — Property 패널에 보이는 값과 실제 저장된 USD 속성값이 달랐던 사례 | `og.Controller.get/set`으로 해당 attribute path(`/World/CarterGraph/differential_controller.inputs:wheelRadius`)를 직접 읽고 써서 확정 — GUI 표시값을 믿지 말고 스크립트로 재검증 |
| Wheel Radius를 스케일이 걸린 Cylinder에서 "로컬 radius × Scale" 암산으로 구하려다 축(axis)·회전 조합을 헷갈릴 뻔함 | Cylinder에 `axis=Z`, `scale=(0.48, 0.48, 0.09)`처럼 비균등 스케일이 걸려 있어 손으로 계산하면 축을 잘못 짚기 쉬움 | `UsdGeom.Cylinder`의 `radius`/`axis` 속성을 읽고, 로컬 원 둘레의 점들을 `ComputeLocalToWorldTransform`으로 월드 변환해 중심으로부터의 실제 거리를 스크립트로 직접 측정(vacuum 프로젝트의 "블라인드 좌표 암산 대신 실측/시각 확인" 교훈과 동일한 패턴) |
| (3-C) `Isaac Create Render Product`의 `Camera Prim` 관계를 GUI로 설정했다고 표시됐는데 실제로는 빈 배열(`[]`)로 저장됨 | Property 패널의 relationship 필드 표시와 실제 USD 값이 다시 한번 어긋난 사례(`wheelRadius` 사례와 동일 계열) | `og.Controller.get/set`으로 `inputs:cameraPrim` attribute를 직접 확인/설정 |
| (3-C) `config_file_name="Example_Rotary_2D"`로 만든 2D LiDAR가 이상한 128채널/4-azimuth-클러스터 구성으로 생성됨 | 이 Isaac Sim 설치본(Kit 6.0)에는 그 이름의 설정 파일이 존재하지 않음(`isaacsim.sensors.rtx`가 `extsDeprecated`로 이동, 예제용 `Example_*` 설정 자체가 제거됨) — 존재하지 않는 이름을 줘도 에러 없이 알 수 없는 값으로 조용히 대체됨 | `find ~/isaacsim_env -ipath "*lidar_configs*" -iname "*.json"`로 실제 존재하는 설정 파일 목록을 먼저 확인. 이 설치본엔 `SLAMTEC/RPLIDAR_S2E.json` 같은 실제 2D 스캐너 프로파일이 있음 |
| (3-C) `RPLIDAR_S2E` 설정으로 만든 prim(`Lidar2D`)의 속성을 읽었더니 스키마가 하나도 없는 빈 `Xform`으로 나옴 | 이 프로파일은 `prim_path` 자체를 센서로 만드는 게 아니라, `materials`+`mesh`+실제 `OmniLidar` 센서를 자식으로 갖는 서브트리를 생성함(Hesai 계열 설정과 다른 동작) | `Usd.PrimRange`로 하위 전체를 순회해 실제 `OmniLidar` 타입 prim을 찾음 — 이 경우 `.../Lidar2D/RPLidar_S2E` |
| (3-C, 미해결) `Camera Prim`을 올바른 중첩 prim으로 고쳐도 `ROS2PublishLaserScan: GMO buffer is empty or invalid. Skipping.`가 매 프레임 반복, `/scan` 토픽은 뜨지만 메시지가 전혀 안 옴 | 확실히 규명 못함. Isaac Sim 완전 재시작 후에도 재현되고, 재시작 후에는 렌더 프로덕트가 세션당 1개로 캡핑된 것처럼 보여(다른 이름의 `Isaac Create Render Product` 노드를 새로 만들어도 새 render product가 등록되지 않음) 같은 세션 안에서 XT_32_10Hz로 바꿔 point_cloud로 교차검증하는 것도 막힘 | 미해결. 다음 세션에 재시작을 여러 번 더 시도하거나, Kit 6.0의 render product 개수 제한 관련 설정을 찾아보거나, 아예 다른 2D 프로파일(SICK TIM781 등)을 시도해볼 것 |

## 6. 체크포인트

- [x] `/World/CarterGraph` Action Graph 생성
- [x] `On Playback Tick` → `ROS2 Subscribe Twist` → `Break Vector3`×2 → `Differential Controller` 데이터 흐름 구성
- [x] `Differential Controller`의 Wheel Radius(0.24)/Wheel Distance(0.6284)와 모든 Max 필드를 명시적으로 채움
- [x] `ROS2 Subscribe Twist.execOut` → `Articulation Controller.execIn` 직결
- [x] `Articulation Controller`의 `targetPrim`(`/World/carter_v1`)과 `jointNames`(`["left_wheel","right_wheel"]`) 설정
- [x] `/cmd_vel` linear.x, angular.z 각각 퍼블리시해 실제 이동/회전 확인
- [x] Odometry/TF/Clock 그래프 구성, `/odom`·`/tf`·`/clock` ~50Hz 확인
- [ ] LiDAR(`/scan`) 실제 데이터 발행 확인 — **미해결, 다음 세션**

## 7. 이번 세션에서 새로 알게 된 환경 사실

- 이 Isaac Sim 설치본은 Kit **6.0**이다(`~/.nvidia-omniverse/logs/Kit/Isaac-Sim Full/6.0/...`로 확인). [[isaacsim_version_upgrade]] 메모리에는 "5.1.0-rc.19 → 6.x 업그레이드, 아직 시작 안 함"으로 남아있었는데, 실제로는 이미 6.0으로 올라와 있었다 — 이번 세션에서 겪은 `Example_Rotary_2D` 부재, `extsDeprecated` 네임스페이스, render product 캡핑처럼 vacuum 프로젝트(5.1.0 기준)와 다르게 동작하는 지점들의 원인일 가능성이 높다.
- 클립보드 복사가 이 세션 환경에서 아예 동작하지 않아, 모든 스크립트를 파일로 저장하고 Script Editor의 `File > Open`으로 로드하는 방식을 기본 작업 패턴으로 썼다([[isaacsim_standard_nav2_curriculum]] 참고).

---
다음: `04-slam-grid-map.md` (미작성) — 단, 착수 전에 3-C(LiDAR)를 먼저 해결해야 함 (slam_toolbox가 `/scan` 필요)
