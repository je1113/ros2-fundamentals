# 07. 가상 학습 데이터 생성 (Teleop)

> 진행 상태: **완료** — teleop 구동 확인, `/scan`은 RTX Lidar 대신 PhysX Raycast 기반 자체 구현으로 정상 발행 확인

## 1. 학습 목표

- `teleop_twist_keyboard`로 로봇청소기를 키보드로 직접 조종한다.
- 주행 중 `/scan`(LiDAR) 토픽이 안정적인 주기로 계속 발행되는지 `ros2 topic hz`로 확인한다.
- Part C 나머지(Topic 8, slam_toolbox 지도 빌드)에 쓸 "다양한 경로로 돌아다닌 스캔 데이터"를 만들어본다.

## 2. 핵심 개념

**왜 teleop인가**: SLAM(Topic 8)이 지도를 만들려면 로봇이 여러 각도/위치에서 스캔한 데이터가 필요하다 — 제자리에 가만히 있으면 지도에 새로운 영역이 안 생기고, 루프 클로저(같은 곳을 다시 지나가며 지도를 보정하는 것)도 확인할 수 없다. `ros2 topic pub`으로 고정된 값을 반복 발행하는 것과 달리, `teleop_twist_keyboard`는 사람이 즉흥적으로 방향을 바꿔가며 몰 수 있어서 이런 다양한 데이터를 만들기 적합하다.

**`ros2 topic hz`**: 토픽이 초당 몇 번 발행되는지 실시간으로 보여준다. LiDAR가 설정된 스캔 주기대로 안정적으로 도는지, 주행 중(부하가 걸렸을 때) 끊기지 않는지 확인하는 용도.

## 3. 실습 단계

### 3.1 사전 확인

- Isaac Sim에서 Topic 6까지 만든 스테이지가 열려있고 **Play** 상태인지 확인
- `robot1`이 지금 [[isaacsim_ros2_advanced_curriculum]]에서 만든 `YawPivot`/Prismatic·Revolute Joint 구조로 cmd_vel에 반응하도록 되어 있음 (이 세션에서 Joint를 GUI로 재생성해서 안정화함 — 06번 문서 5절 참고, 바닥 밀착은 아직 안 됨)

### 3.2 teleop로 직접 조종

터미널에서 (conda 비활성화, ROS2 소스 상태로):

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

안내되는 키(`i`/`,`/`j`/`l`/`k` 등)로 Isaac Sim 뷰포트를 보면서 직접 몰아본다. 여러 방향으로 돌아다니며 조종해볼 것.

### 3.3 `/scan` 발행 주기 확인

다른 터미널에서, teleop로 몰고 있는 동안:

```bash
ros2 topic hz /scan
```

## 4. 예상/실제 결과 확인

`teleop_twist_keyboard`로 전/후진 및 회전 정상 확인 (Joint를 GUI로 재생성한 뒤). `/camera/rgb`는 `ros2 topic hz`로 26~33Hz 정상 확인.

`/scan`은 RTX Lidar 경로를 완전히 포기하고 PhysX Raycast 기반 자체 구현으로 교체한 뒤 `ros2 topic hz /scan`에 정상적으로 값이 찍히는 것까지 확인 완료 (아래 3.4, 5절 참고).

### 3.4 `/scan` 미발행 문제 — RTX Lidar 파이프라인 자체 결함으로 결론, Raycast 기반으로 우회

다음 세션에서 `SensorGraph`(`RenderProduct`+`LidarHelper`)를 통째로 삭제 후 재생성했지만 **동일 증상 재현** (Kit 로그: `IsaacComputeRTXLidarFlatScan: ... azimuthBufferSize is 0. Skipping execution.`). 이어서 다음을 체계적으로 검증했으나 전부 정상으로 확인됨:

- USD 레이어/스테이지 자체 (`stage.DefinePrim()`으로 새 프림 생성 테스트 정상)
- 기존 `Lidar_2D` 프림의 속성 — 새로 생성한 라이다와 attribute 단위로 diff했을 때 **완전히 동일**
- 월드 트랜스폼 — 스케일 (1,1,1), xformOp 구성 정상
- `RenderProduct` 프림의 실제 구성 — `camera` relationship이 정확한 라이다를 가리킴, `orderedVars`에 `GenericModelOutput`/`RtxSensorMetadata`가 정확히 붙어있음, compute count도 매 프레임 정상 증가
- 라이다 설정 파일(`Example_Rotary_2D.usda`) 원격 로드 — S3에서 200 OK로 정상 로드됨 (네트워크 문제 아님)
- GPU/확장 — RTX 4070(RT 코어 있음), 관련 확장(`omni.sensors.nv.lidar` 등) 전부 정상 로드, 에러 없음
- `user.config.json` — 라이다를 비활성화하는 설정 없음

**결정적 재현**: `/World` 루트에 계층 구조 없이 완전히 새로 만든 2D 라이다(`LidarRtx(config_file_name="Example_Rotary_2D")`)로 `RenderProduct`의 `cameraPrim`을 바꿔봐도 **똑같이 실패**. 이건 이 씬의 특정 프림 문제가 아니라 **이 Isaac Sim 5.1.0-rc.19 설치 세션 전체에서 RTX Lidar 렌더링 파이프라인이 죽어있다**는 뜻. Topic 4에서는 동일한 구성으로 분명히 작동했었기 때문에(commit `056eaaf`), Part A/B 세션 어딘가(GPU Collision Stack Size 조정, 조인트 재구성 등)에서 뭔가 영구적으로 깨진 것으로 추정되지만 정확한 지점은 특정하지 못함. [[isaacsim_ros2_advanced_curriculum]]에서 이미 확인된 "이 RC 빌드의 물리 관련 버그"(Topic 6, raw velocity/force 주입 실패)와 같은 계열의 **미해결 RC 빌드 버그**로 결론.

**우회책**: RTX Lidar를 완전히 버리고, Topic 5에서 검증된 PhysX Raycast(`get_physx_scene_query_interface().raycast_closest()`)를 360도, 1도 간격으로 부채꼴로 쏴서 2D 스캔을 직접 합성. `SensorGraph`를 다시 삭제 후 재구성:

- `On Playback Tick` → `ScanCompute`(`omni.graph.scriptnode.ScriptNode`, 매 프레임 raycast 360발을 쏴서 결과를 계산)
- `ScanCompute`는 계산한 `ranges`/`intensities` 배열을 동적 출력 핀 대신 `og.Controller.set()`으로 `ScanPublish` 노드의 입력에 직접 기록 (Topic 6 `OdomCompute`→`OdomPublish`와 동일 패턴)
- `ScanPublish`(`isaacsim.ros2.bridge.ROS2PublishLaserScan`, 네이티브 컴파일 노드) — `Lidar_2D` 프림의 월드 트랜스폼을 원점/기준 방향으로 사용, `frameId="Lidar_2D"`(TF 트리와의 일관성 유지), `topicName="scan"`, `horizontalFov=360`, `horizontalResolution=1.0`(deg), `azimuthRange=[-180,180]`, `depthRange=[0.05, 12.0]`
- `IsaacReadSimulationTime` → `ScanPublish.inputs:timeStamp` (execIn 없는 순수 compute 노드)

라이다 자체 프림(`Lidar_2D`)은 삭제하지 않고 유지 — 더 이상 렌더링에 쓰이진 않지만 raycast 원점/방향 기준과 TF frame으로 계속 사용됨.

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| teleop로 몰면 로봇이 멈추지 않고 계속 밀리는 것처럼 보임 | 알고 보니 실제로는 `/World/YawRevoluteJoint`/`ForwardPrismaticJoint`(Script Editor로 만든 것)가 Y축을 제대로 안 잠가서 **중력으로 자유낙하**하고 있었던 것 — "미끄러진다"는 착각이었음. `Target Velocity` Property는 정상적으로 0이었음 | 06번 문서 5절 참고 — Joint를 GUI(`Create > Physics > Joint`)로 재생성해서 해결. `Stop`을 여러 번 거치며 `ros2 topic pub -r 10`으로 띄웠던 프로세스가 `Ctrl+Z`로 정지된 채(`T` 상태) 안 죽고 남아있던 것도 발견 — `ps aux \| grep "topic pub"`으로 확인 후 `kill -9`로 정리 |
| `/scan`이 `ros2 topic list`엔 있는데 `ros2 topic hz /scan`에 아무 데이터도 안 잡힘. Kit 로그엔 `IsaacComputeRTXLidarFlatScan: ... azimuthBufferSize is 0. Skipping execution.` 반복. 완전히 새로 만든 원점의 라이다로도 재현되어 씬 문제가 아니라 이 설치 세션 전체의 RTX Lidar 파이프라인 결함으로 결론 | 원인 미확정 (RC 빌드 버그로 추정, 3.4절 참고) | RTX Lidar를 버리고 PhysX Raycast 기반 자체 스캔 생성으로 전환 (3.4절) |
| `ROS2PublishLaserScan`에 `linearDepthData`만 채우고 `intensitiesData`를 비워두면 `Linear Depth data and Intensities data sizes do not match` 에러 | 두 배열이 같은 길이(`numCols * numRows`)여야 함 | `ScanCompute`에서 매 레이마다 intensities도 같이 채움 (히트 시 200, 미스 시 0) |
| Raycast fan이 로봇 자기 몸체(`robot1`)에 맞아 모든 방향에서 거리 0에 가깝게 나올 위험 | Topic 5에서 이미 겪은 것과 같은 문제 — Convex Hull이 곡면 쉘의 내부를 채워서 로봇 부피 상당 부분이 "안쪽"으로 판정됨 | raycast 결과의 hit 프림 경로가 `robot1` 하위면 무시하고 `inf`(감지 안 됨) 처리 |

## 6. 체크포인트

- [x] `teleop_twist_keyboard`로 로봇을 여러 방향으로 조종
- [x] `ros2 topic hz /scan`으로 주행 중 안정적인 발행 주기 확인 — RTX Lidar 대신 PhysX Raycast 기반 자체 구현으로 확인 완료

---
다음: Topic 8 (slam_toolbox로 격자 지도 빌드)
