# 07. 가상 학습 데이터 생성 (Teleop)

> 진행 상태: **착수**

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
- `robot1`이 지금 [[isaacsim_ros2_advanced_curriculum]]에서 만든 `YawPivot`/`ForwardPrismaticJoint` 구조로 cmd_vel에 반응하도록 되어 있음 (바닥 밀착은 아직 안 됨 — 뜬 채로 이동하지만 조종 자체엔 문제없음)

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

(진행 중)

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|

## 6. 체크포인트

- [ ] `teleop_twist_keyboard`로 로봇을 여러 방향으로 조종
- [ ] `ros2 topic hz /scan`으로 주행 중 안정적인 발행 주기 확인

---
다음: Topic 8 (slam_toolbox로 격자 지도 빌드)
