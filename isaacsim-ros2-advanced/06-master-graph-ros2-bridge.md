# 06. 마스터 액션그래프 & ROS2 브릿지 활성화

> 진행 상태: **완료** — 3.1(cmd_vel 구동)/3.2(Odometry)/3.3(TF 트리) 전부 확인. 헤드리스 실행 테스트만 남음.

## 1. 학습 목표

- 지금까지 만든 개별 센서 그래프(LiDAR, RGB-D 카메라, 범퍼, 추락방지)를 하나의 일관된 마스터 그래프로 통합한다.
- `/cmd_vel`(Twist)을 구독해 로봇을 실제로 움직인다.
- TF 트리와 Odometry를 퍼블리시해 Nav2/RViz 등 하위 스택이 쓸 수 있는 완전한 ROS2 인터페이스를 만든다.

## 2. 핵심 개념

**바퀴 Joint가 없는 로봇의 cmd_vel 처리 — 최종 방식은 Joint Drive**: `isaacsim.robot.wheeled_robots`의 `Differential Controller`/`Holonomic Controller`는 실제 바퀴 Joint가 있는 로봇을 위한 노드라 이 로봇(바퀴를 개별 모델링하지 않고 몸체 하나로 단순화함)에는 안 맞는다.

처음엔 `Write Prim Attribute`로 `physics:velocity`/`physics:angularVelocity`에 매 프레임 직접 쓰는 방식, 그다음 `omni.isaac.dynamic_control`(`set_rigid_body_linear_velocity`/`apply_body_force`)로 PhysX 텐서 API를 직접 호출하는 방식을 시도했지만 **이 Isaac Sim 5.1.0-rc.19 설치본에서는 둘 다 안 먹혔다** — velocity/force를 아무리 정확히 넣어도 읽어보면 값은 맞는데 실제 위치는 절대 안 바뀌는 현상이 반복됐다 (자세한 원인 조사는 5절 참고). 대신 **PhysX Joint Drive**(`UsdPhysics.DriveAPI`, targetVelocity)는 Topic 3의 브러쉬 Angular Drive에서 이미 검증됐던 대로 이번에도 확실히 작동했다.

**최종 구조 — Yaw Pivot + Prismatic 방식**: robot1은 바퀴가 없는 단일 강체라 "제자리 회전 + 그 방향으로 전진"을 자유 강체 velocity로 표현할 수 없었던 것도 문제였다 (world-frame 고정 velocity로는 회전할수록 전진 방향이 안 맞음). 대신 2단 Joint 체인으로 만들었다:
- **`/World/YawPivot`**: 질량만 있고 충돌은 없는 더미 RigidBody. `/World/YawRevoluteJoint`로 **월드 고정 프레임**(body0 비워둠)에 Y축 Revolute Joint + Angular Drive로 연결 — robot1의 회전(yaw)을 담당.
- **`/World/robotvacum_decimated/robot1`**: `/World/ForwardPrismaticJoint`로 YawPivot에 X축 Prismatic Joint + Linear Drive로 연결. 이 Joint의 X축은 YawPivot 로컬 프레임 기준이라 **YawPivot이 돌면 같이 돈다** — 그래서 "지금 향하고 있는 방향으로 전진"이 자동으로 성립.
- **부작용(의도된 것)**: Prismatic Joint가 YawPivot 기준 X축 이동 1자유도만 남기고 나머지(Y/Z 이동, 3축 회전)를 전부 잠그기 때문에, robot1은 **중력의 영향을 받지 않고 YawPivot 높이에 고정된 채 뜬 상태로 이동**한다. 오늘은 이 상태로 "cmd_vel 방향대로 정확히 움직이는지"만 확인했고, 바닥에 붙어서 구르는 느낌을 원하면 나중에 별도로 다듬어야 함(예: Y축에 살짝 여유를 주는 스프링/댐퍼 Joint 추가).

## 3. 계획된 노드 구성

grep으로 정확한 노드 타입/uiName을 미리 확인해뒀다 (`~/isaacsim_env/lib/python3.11/site-packages/isaacsim/exts/isaacsim.ros2.bridge/ogn/docs/*.rst`):

### 3.1 cmd_vel로 로봇 구동 — 완료

USD Physics 준비 (Script Editor, `UsdPhysics`/`PhysxSchema` Python API로 직접 구성):
- `/World/YawPivot`: `RigidBodyAPI` + `MassAPI`(0.1kg), 충돌 없음, robot1 시작 위치에 배치
- `/World/YawRevoluteJoint`: `PhysicsRevoluteJoint`, axis=`Y`, body0 비움(월드 고정) → body1=`YawPivot`, `DriveAPI` instance `"angular"` (`type=force`, `damping=1e5`, `stiffness=0`)
- `/World/ForwardPrismaticJoint`: `PhysicsPrismaticJoint`, axis=`X`, body0=`YawPivot` → body1=`robot1`, `DriveAPI` instance `"linear"` (동일 damping/stiffness)

OmniGraph (`/World/CmdVelGraph`):
- `ROS2 Subscribe Twist` (`isaacsim.ros2.bridge.ROS2SubscribeTwist`) — 토픽 `cmd_vel`. 출력 `Linear Velocity`/`Angular Velocity` (각 `vectord[3]`)
- `Break Vector3` 2개 (`omni.graph.nodes.BreakVector3`)로 각각 x/y/z 스칼라로 분해 — Drive의 `targetVelocity`는 스칼라 float라 벡터를 그대로 못 씀
- `Write Prim Attribute` 2개 (`omni.graph.nodes.WritePrimAttribute`, **ScriptNode 아닌 네이티브 컴파일 노드**):
  - `Prim Path=/World/ForwardPrismaticJoint`, `Attribute Name=drive:linear:physics:targetVelocity` ← `BreakLinear.outputs:x` (ROS `linear.x`)
  - `Prim Path=/World/YawRevoluteJoint`, `Attribute Name=drive:angular:physics:targetVelocity` ← `BreakAngular.outputs:z` (ROS `angular.z` — 이 스테이지는 Y-up이라 world Y회전 축과 정확히 일치, 별도 축 변환 불필요)
- 연결: `On Playback Tick` → `ROS2 Subscribe Twist`(Exec) 및 두 `Write Prim Attribute`(Exec, 매 프레임 — cmd_vel 새 메시지 여부와 무관하게 항상 재적용)

### 3.2 Odometry 퍼블리시 — 완료

- `ROS2 Publish Odometry` (`isaacsim.ros2.bridge.ROS2PublishOdometry`)는 `Position`/`Orientation`을 world-frame으로 직접 입력받는 노드라, 이 스테이지가 Y-up이라는 걸 명시적으로 변환해야 했다. robot1의 회전은 `YawPivot`/`ForwardPrismaticJoint` 체인 덕분에 **Y축 순수 yaw 회전으로만 제한**되어 있어서 축 변환이 단순해졌다: 매 프레임 작은 `ScriptNode`(`OdomCompute`, 순수 수학 계산만 함 — 오늘 겪은 "raw velocity/force가 안 먹히는" 문제와는 무관한 영역)가 `robot1`의 `ComputeLocalToWorldTransform()`을 읽어서
  - 위치: Isaac `(x, y, z)` → ROS `(x, z, y)`
  - 방향: Isaac Y축 쿼터니언에서 `yaw = 2*atan2(quat.imaginary.y, quat.real)`을 뽑아 ROS Z축 쿼터니언 `(0, 0, sin(yaw/2), cos(yaw/2))`으로 재구성
  하고, 이 값을 `og.Controller.set()`으로 `OdomPublish` 노드의 `inputs:position`/`inputs:orientation`에 직접 밀어넣는다 (동적 ScriptNode 출력 핀을 새로 만드는 대신, 이미 있는 다운스트림 노드의 입력에 바로 쓰는 방식 — 오늘 여러 번 써서 검증된 패턴).
- `Linear/Angular Velocity`: 3.1의 `ROS2SubscribeTwist` 출력을 그대로 재사용(연결만 함, 별도 변환 없음). 단 `ROS2PublishOdometry`의 기본 동작(`publishRawVelocities=False`)은 내부적으로 `robotFront`/`robotUp=[0,0,1]`(Z-up 가정)로 world velocity를 로봇 로컬 프레임에 투영하는데, 이 스테이지는 Y-up이라 이 가정이 안 맞는다 → **`publishRawVelocities=True`로 켜서 이 내부 투영을 건너뛰고, 이미 ROS 컨벤션인 cmd_vel 값을 그대로 통과**시키는 쪽을 택함.
- `Chassis Frame Id=base_link`, `Odom Frame Id=odom`, `Topic Name=odom`

### 3.3 TF 트리 퍼블리시 — 완료

- `ROS2 Publish Transform Tree` (`isaacsim.ros2.bridge.ROS2PublishTransformTree`)
- `Target Prims`(다중 입력, `SET_VALUES`에 prim path 문자열 리스트로 전달)에 `robot1`, `lidar_mount/Lidar_2D`, `camera_mount/Camera`, `bumper_mount`, `cliff_mount` 등록
- **frameId 불일치 확인 및 수정**: Topic 4에서 LiDAR/카메라 Helper 노드의 `frameId`가 `sim_lidar`/`sim_camera`라는 임의 문자열이었는데, `ROS2PublishTransformTree`는 타겟 프림의 **실제 프림 이름**을 frame_id로 쓴다(`Lidar_2D`, `Camera`) — 매칭 안 되면 RViz 등에서 센서 데이터가 TF에 안 붙는다. `/World/SensorGraph/LidarHelper.inputs:frameId`를 `Lidar_2D`로, `/World/ActionGraph/RgbHelper.inputs:frameId`와 `DepthHelper.inputs:frameId`를 `Camera`로 각각 `og.Controller.attribute()` + `og.Controller.set()`으로 직접 고쳐서 일치시켰다.

## 4. 예상/실제 결과 확인

`ros2 topic pub /cmd_vel geometry_msgs/msg/Twist ...`로 로봇이 명령대로 방향 전환하며 이동, `ros2 topic echo /odom`과 `ros2 topic echo /tf` 둘 다 값이 정상적으로 갱신되며 퍼블리시되는 것 확인. (`tf2_echo`로 특정 프레임 쌍 조회, RViz 시각 확인은 아직 안 함 — 다음에 필요하면.)

## 5. 알려진 문제와 해결

3.1 검증이 예상보다 훨씬 오래 걸렸다 (하루 전체). 원인이 하나가 아니라 여러 개가 겹쳐 있었어서, 순서대로 기록한다.

| 증상 | 원인 | 해결 |
|---|---|---|
| 세션 후반부에 Isaac Sim이 반복적으로 멈춤(freeze) | 하루 종일(여러 시간) 켜둔 채 그래프 편집/물리 시뮬레이션/센서 디버그 시각화를 계속 누적한 것으로 추정 | 재시작으로 완화됨. 새 토픽 시작할 때 Isaac Sim을 한 번 재시작하고 시작할 것. 이번 세션엔 재시작을 여러 번 했는데도 아래의 다른 원인들 때문에 계속 막혔음 — 재시작만으로는 해결 안 되는 문제도 섞여 있었다는 뜻 |
| `isaacsim.core.prims.SingleRigidPrim.set_linear_velocity()`를 호출해도 로봇이 안 움직임, 읽어보면 값은 맞는데 실제 시뮬레이션엔 반영 안 됨 | `SingleRigidPrim.initialize()`가 내부적으로 `SimulationManager.get_physics_sim_view()`를 쓰는데, 이 값은 `World()` 객체를 통한 초기화가 있어야 채워진다. 순수 Isaac Sim + Script Editor 세션(우리 작업 방식)에선 그 초기화가 한 번도 안 일어나서 `get_physics_sim_view()`가 계속 `None`이었고, 우리 스크립트의 `if is None: return` 가드에 매번 조용히 걸려 velocity 설정 자체가 스킵되고 있었음. 별개로: `setup(db)`는 ScriptNode가 처음 로드될 때 딱 한 번만 실행되고 Stop→Play를 반복해도 재실행되지 않아서, 캐시해둔 rigid body 핸들이 이전 Play 세션의 죽은 physics view를 참조하는 문제도 겹쳐 있었음 | 더 저수준인 `omni.isaac.dynamic_control`(`_dynamic_control.acquire_dynamic_control_interface()` → `get_rigid_body(path)` → `set_rigid_body_linear_velocity(handle, ...)`)로 전환. `SimulationManager` 의존성이 없음. (그런데 이것도 결국 이 세션에선 위치 변화가 없었다 — 아래 타임라인 버그가 진짜 원인이었음) |
| 정확한 velocity가 읽히는데도(readback이 명령값과 일치) 실제 위치가 몇 분(초 단위 아님)이 지나도 소수점까지 전혀 안 바뀜 | **`stage.GetEndTimeCode()`가 `0.0`으로 되어 있었고(`startTimeCode`도 `0.0`), 타임라인이 `is_looping=True`인 채 길이 0에 가까운 구간(`0` ~ `1/24초`)을 매 프레임 계속 루프**하고 있었음 — `timeline.get_current_time()`이 `0.0`에 고정된 채 절대 진행되지 않음. Δt가 사실상 0이라 velocity를 아무리 정확히 걸어도 위치 적분이 전혀 안 일어남. `omni.timeline.get_timeline_interface().is_playing()`은 `True`로 보고되고 컴퓨트 카운트도 계속 올라가서, 겉으로는 다 정상처럼 보이는 게 이 버그를 찾기 어려웠던 이유 | `stage.SetEndTimeCode(1000000.0)` + `timeline.set_end_time(...)` + `timeline.set_looping(False)`로 수정하면 `current_time`이 정상적으로 흐르기 시작함. **주의: 이 수정이 Stop→Play 사이클마다 다시 풀리는 경우가 있었다** — Stop 후 Play 하기 전에 매번 다시 적용해야 안전. 진단할 땐 `timeline.get_current_time()`을 반드시 두 번 이상 시차를 두고 찍어서 실제로 흐르는지 확인할 것 — `is_playing`만 보고 판단하면 이 버그를 놓친다 |
| 타임라인을 고치고 `dynamic_control`로 velocity/force(`apply_body_force`/`apply_body_torque`)를 걸어도 여전히 위치가 안 바뀜. 순수 중력(스크립트 없이 RigidBodyAPI만)은 정상적으로 낙하함 | 이 Isaac Sim 5.1.0-rc.19(release candidate) 빌드에서 **외부에서 주입하는 rigid body velocity/force가 실제 시뮬레이션에 반영되지 않는 것으로 보임** — PhysX가 내부적으로 매 스텝 적용하는 힘(중력)은 정상 작동하지만, `dynamic_control`이나 `physics:velocity` USD 속성을 통한 외부 주입은 (readback은 정상으로 보여도) 실제로 반영되지 않았다. 씬 레벨 velocity clamp(`physxRigidBody:maxLinearVelocity` 등)도 확인했지만 원인 아니었음 | **PhysX Joint Drive**(`UsdPhysics.DriveAPI`, targetVelocity)로 전환 — Topic 3의 브러쉬 Angular Drive가 이미 이 설치본에서 검증된 메커니즘이었다는 걸 뒤늦게 떠올림. Free rigid body에 velocity/force를 직접 주입하는 대신, Joint로 연결하고 Joint Drive로 미는 방식은 확실히 작동했다. **이 프로젝트에서 앞으로 로봇을 움직일 일이 있으면 raw velocity/force API보다 Joint Drive를 먼저 시도할 것** |
| Script Editor에서 파일을 고쳐서 다시 `File > Open`으로 열어 실행해도 예전 코드가 그대로 실행됨 (프린트 결과가 수정 전과 동일) | 같은 파일 경로의 탭이 이미 열려있으면 Script Editor가 디스크에서 다시 안 읽고 기존 탭 내용을 그대로 재실행하는 것으로 보임 (여러 번 재현됨) | 파일을 고칠 때마다 새 파일명(`_v2`, `_v3`...)으로 저장해서 여는 게 가장 확실함. 탭을 직접 닫고 다시 여는 것도 될 수 있지만 새 파일명이 더 안전 |
| `UsdGeom.XformCommonAPI.SetTranslate()`/`SetRotate()`를 호출해도 조용히 아무 효과 없음, 콘솔에 `Could not determine xform ops for incompatible xformable` 경고만 뜸 | FBX/OBJ import + flatten을 거친 프림들의 xformOp 구성이 XformCommonAPI가 기대하는 표준 순서(translate/pivot/rotateXYZ/scale)와 안 맞아서 API가 조용히 무시함 | `UsdGeom.Xformable(prim).ClearXformOpOrder()`로 지우고 `AddTranslateOp()`/`AddRotateXYZOp()`/`AddScaleOp()`로 깨끗하게 재생성. 단, 기존에 같은 이름의 op 속성이 남아있으면 타입(float3 vs double3)이 안 맞을 수 있어 `AddXformOp` 호출 시 `ErrorException`(비치명적이지만 스크립트 실행을 중단시킴)이 남 → 먼저 `prim.GetAttribute("xformOp:xxx").GetTypeName()`으로 기존 타입을 확인하고 그 정밀도(`PrecisionFloat`/`PrecisionDouble`)에 맞춰서 호출해야 함 |
| `og.Controller.edit({"graph_path": "<이미 존재하는 그래프>", ...}, {...})`를 그래프 생성 직후가 아니라 나중에 다시 호출하면 `Failed to wrap graph in node` 또는 `A graph already exists at this path` 에러 | dict 형태의 `graph_id`를 넘기면 `og.Controller.edit()`는 그래프가 이미 있어도 항상 `CreateGraphAsNode`를 다시 시도한다 (04번 문서 3.4/5.3절에서 이미 겪은 문제와 동일 계열, 이번엔 에러 메시지만 다름) | 기존 그래프에 새 노드를 추가해야 하면, 그 그래프를 통째로 지우고(`omni.kit.commands.execute("DeletePrims", paths=[graph_path])`) 한 번의 `edit()` 호출로 전체를 다시 만드는 게 제일 안전.가장 작은 단위(같은 그래프를 여러 세션에 걸쳐 부분부분 늘리는 것)로 나눠서 하려던 시도는 이번에도 전부 실패했음 |
| 로봇이 갑자기 바닥을 뚫고 아래로 사라지거나(Y가 음수로 튐), Stop→Play 해도 계속 같은 이상한 좌표로 돌아옴 | (1) 큰 velocity를 순간적으로 주면(수동 테스트용으로 1~2m/s급을 줬을 때) 씬의 얇은 `Plane` 콜라이더(GroundPlane)를 한 프레임에 관통해버리는 터널링이 여러 번 재현됨. (2) 한 번은 그 상태로 자동저장/Stop이 걸려서 저장 파일 자체에 나쁜 위치가 박혀버림 | Stop 상태에서 `stage.SetEditTarget(Usd.EditTarget(stage.GetRootLayer()))` 후 알고 있는 정상 좌표로 직접 되돌리는 스크립트로 복구. 얇은 Plane 대신 두꺼운 Box 콜라이더로 바꿔보기도 했지만 (이번 세션의 진짜 원인은 타임라인 버그였어서) 근본 해결은 아니었음 — 그래도 터널링 자체를 줄이는 덴 유효한 방법이니 기억해둘 것 |

## 6. 체크포인트

- [x] 필요한 ROS2 브릿지 노드 이름/타입 문자열 전부 확인 (`ROS2SubscribeTwist`, `ROS2PublishOdometry`, `ROS2PublishTransformTree`, `WritePrimAttribute`, `BreakVector3`)
- [x] cmd_vel → Joint Drive(`YawRevoluteJoint`/`ForwardPrismaticJoint`) 배선, `ros2 topic pub /cmd_vel`로 방향 전환 포함 정상 구동 확인 (단, robot1이 중력 영향 없이 YawPivot 높이에 뜬 채로 이동 — 바닥 밀착은 다음에 다듬을 것)
- [x] Odometry 퍼블리시 배선 및 `/odom` 토픽 확인 (Y-up→ROS 변환 포함)
- [x] TF 트리 배선 및 frame_id 일관성 확인 (LiDAR/카메라 Helper의 `frameId`를 `Lidar_2D`/`Camera`로 수정해 매칭)
- [ ] 헤드리스 실행 테스트 (미착수)
- [ ] (선택) robot1이 바닥에 붙어서 구르도록 YawPivot/Prismatic 구조에 수직 방향 여유(스프링/댐퍼) 추가

---
Part B (마스터 그래프/ROS2 브릿지) 완료. 다음: Part C(SLAM, teleop 데이터 생성 → slam_toolbox 지도 빌드) 또는 남은 헤드리스 실행 테스트
