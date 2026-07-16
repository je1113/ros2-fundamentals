# 06. 마스터 액션그래프 & ROS2 브릿지 활성화

> 진행 상태: **착수 — 노드 구성 계획 확정, 배선/검증은 다음 세션**. 오늘 세션이 매우 길어져 Isaac Sim이 반복적으로 멈추기 시작해 여기서 중단.

## 1. 학습 목표

- 지금까지 만든 개별 센서 그래프(LiDAR, RGB-D 카메라, 범퍼, 추락방지)를 하나의 일관된 마스터 그래프로 통합한다.
- `/cmd_vel`(Twist)을 구독해 로봇을 실제로 움직인다.
- TF 트리와 Odometry를 퍼블리시해 Nav2/RViz 등 하위 스택이 쓸 수 있는 완전한 ROS2 인터페이스를 만든다.

## 2. 핵심 개념

**바퀴 Joint가 없는 로봇의 cmd_vel 처리**: `isaacsim.robot.wheeled_robots`의 `Differential Controller`/`Holonomic Controller`는 실제 바퀴 Joint가 있는 로봇을 위한 노드라 이 로봇(바퀴를 개별 모델링하지 않고 몸체 하나로 단순화함)에는 안 맞는다. 대신 `Write Prim Attribute`로 로봇 Rigid Body의 `physics:velocity`/`physics:angularVelocity` USD 속성에 매 프레임 직접 값을 써서 움직이는 방식을 쓴다.

## 3. 계획된 노드 구성 (다음 세션에 배선/검증)

grep으로 정확한 노드 타입/uiName을 미리 확인해뒀다 (`~/isaacsim_env/lib/python3.11/site-packages/isaacsim/exts/isaacsim.ros2.bridge/ogn/docs/*.rst`):

### 3.1 cmd_vel로 로봇 구동

- `ROS2 Subscribe Twist` (`isaacsim.ros2.bridge.ROS2SubscribeTwist`) — 기본 토픽 `cmd_vel`. 출력: `Linear Velocity`/`Angular Velocity` (각 `vectord[3]`, m/s 및 rad/s)
- `Write Prim Attribute` 2개 (`omni.graph.nodes... OgnWritePrimAttribute`):
  - `Use Path=true`, `Prim Path=/World/robotvacum_decimated/robot1`, `Attribute Name=physics:velocity`, `Value`← Linear Velocity
  - `Use Path=true`, `Prim Path=/World/robotvacum_decimated/robot1`, `Attribute Name=physics:angularVelocity`, `Value`← Angular Velocity
- 연결: `On Playback Tick` → `ROS2 Subscribe Twist`(Exec) → 두 `Write Prim Attribute`(Exec 각각)

### 3.2 Odometry 퍼블리시

- `ROS2 Publish Odometry` (`isaacsim.ros2.bridge.ROS2PublishOdometry`)
- 입력: `Position`/`Orientation`(월드 기준 robot1의 위치/회전 — `Get Prim Local to World Transform` + `Get Translation`/회전 추출 노드 필요, Topic 5의 `cliff_mount` 위치 추적과 같은 패턴), `Linear/Angular Velocity`(3.1의 Twist 구독값을 그대로 재사용해도 되고, 실제 물리 속도를 읽고 싶으면 별도 조사 필요), `Chassis Frame Id=base_link`, `Odom Frame Id=odom`, `Topic Name=odom`

### 3.3 TF 트리 퍼블리시

- `ROS2 Publish Transform Tree` (`isaacsim.ros2.bridge.ROS2PublishTransformTree`)
- `Target Prims`(다중 입력 가능)에 `robot1`, `lidar_mount/Lidar_2D`, `camera_mount/Camera`, `bumper_mount`, `cliff_mount` 등록
- **주의(다음 세션에 확인 필요)**: 이 노드가 퍼블리시하는 frame_id가 프림 이름 기반인지 확인 필요 — Topic 4에서 LiDAR/카메라 Helper 노드의 `frameId`를 `sim_lidar`/`sim_camera`로 임의 문자열로 설정해뒀는데, TF 트리의 frame_id와 안 맞으면 RViz 등에서 센서 데이터가 TF에 제대로 안 붙는다. 필요하면 Helper 노드들의 `frameId`를 실제 프림 이름과 일치시켜야 함.

## 4. 예상/실제 결과 확인

(다음 세션 목표) `ros2 topic pub /cmd_vel geometry_msgs/msg/Twist ...`로 로봇이 실제로 움직이는지, `ros2 topic echo /odom`과 `ros2 run tf2_ros tf2_echo` 등으로 TF/Odometry가 정상 퍼블리시되는지 확인.

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| 세션 후반부에 Isaac Sim이 반복적으로 멈춤(freeze) | 하루 종일(여러 시간) 켜둔 채 그래프 편집/물리 시뮬레이션/센서 디버그 시각화를 계속 누적한 것으로 추정. 시스템 자체 리소스(메모리 20GB 여유, GPU 사용률 9%)는 문제 없었음 — OS 레벨 자원 부족이 아니라 앱 내부 상태(렌더링 캐시, 그래프 누적 편집 이력 등)가 무거워진 것으로 보임 | 재시작으로 완화됨. **다음 세션 시작 시 참고**: 새 토픽 시작할 때 Isaac Sim을 한 번 재시작하고 시작하면 이런 문제를 예방할 수 있을 것 같음. 특히 여러 시간 연속 작업 후에는 중간에 한 번씩 재시작 고려 |

## 6. 체크포인트

- [x] 필요한 ROS2 브릿지 노드 이름/타입 문자열 전부 확인 (`ROS2SubscribeTwist`, `ROS2PublishOdometry`, `ROS2PublishTransformTree`, `WritePrimAttribute`)
- [ ] cmd_vel → `Write Prim Attribute` 배선, `ros2 topic pub /cmd_vel`로 로봇 구동 확인
- [ ] Odometry 퍼블리시 배선 및 `/odom` 토픽 확인
- [ ] TF 트리 배선 및 frame_id 일관성 확인 (LiDAR/카메라 Helper의 `frameId`와 매칭)
- [ ] 헤드리스 실행 테스트 (미착수)

---
다음 세션: 3.1(cmd_vel 구동)부터 이어서 배선 및 검증
