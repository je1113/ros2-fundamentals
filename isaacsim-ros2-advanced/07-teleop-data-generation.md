# 07. 가상 학습 데이터 생성 (Teleop)

> 진행 상태: **진행 중 — teleop 구동 확인 완료, `/scan` 미발행 이슈로 중단**

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

**`/scan`은 토픽 자체는 `ros2 topic list`에 보이지만(광고는 됨) `ros2 topic hz /scan`에 아무 값도 안 찍힘 — 다음 세션에서 이어서 조사할 것.**

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| teleop로 몰면 로봇이 멈추지 않고 계속 밀리는 것처럼 보임 | 알고 보니 실제로는 `/World/YawRevoluteJoint`/`ForwardPrismaticJoint`(Script Editor로 만든 것)가 Y축을 제대로 안 잠가서 **중력으로 자유낙하**하고 있었던 것 — "미끄러진다"는 착각이었음. `Target Velocity` Property는 정상적으로 0이었음 | 06번 문서 5절 참고 — Joint를 GUI(`Create > Physics > Joint`)로 재생성해서 해결. `Stop`을 여러 번 거치며 `ros2 topic pub -r 10`으로 띄웠던 프로세스가 `Ctrl+Z`로 정지된 채(`T` 상태) 안 죽고 남아있던 것도 발견 — `ps aux \| grep "topic pub"`으로 확인 후 `kill -9`로 정리 |
| `/scan`이 `ros2 topic list`엔 있는데 `ros2 topic hz /scan`에 아무 데이터도 안 잡힘. Kit 로그엔 `IsaacComputeRTXLidarFlatScan: ... azimuthBufferSize is 0. Skipping execution.` 반복 | 원인 미확정. 확인해본 것: `RenderProduct`/`LidarHelper` OmniGraph 노드는 매 프레임 정상 compute됨(컴퓨트 카운트 증가), `cameraPrim`도 정확한 경로를 가리킴, 렌더러는 `RaytracedLighting`/`rtx`로 정상, `/camera/rgb`는 같은 세션에서 정상 발행됨(RTX 렌더링 자체는 살아있음), `Lidar_2D` 프림을 `LidarRtx(config_file_name="Example_Rotary_2D")`로 완전히 재생성해도 동일 증상, Isaac Sim 전체 재시작해도 동일 증상. `omni:sensor:Core:rangeCount = 0` 등 attribute가 비어있는 걸 의심했지만 재생성 전후로 값이 똑같아서 애초에 이 attribute들은 이 값이 정상(다른 곳에서 실제 설정을 읽는 듯)으로 보임 — 아직 결론 못 냄 | **다음 세션 숙제**: SensorGraph(`RenderProduct`+`LidarHelper`) 자체를 통째로 삭제하고 처음부터 다시 만들어보기 (오늘 CmdVelGraph가 부분 수정으로는 안 되고 통째 재생성해야 했던 것과 같은 패턴일 가능성) |

## 6. 체크포인트

- [x] `teleop_twist_keyboard`로 로봇을 여러 방향으로 조종
- [ ] `ros2 topic hz /scan`으로 주행 중 안정적인 발행 주기 확인 — **미해결, 다음 세션**

---
다음: `/scan` 미발행 이슈 조사 이어서 → Topic 8 (slam_toolbox로 격자 지도 빌드)
