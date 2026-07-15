# ROS 2 Advanced — Isaac Sim × ROS2 트랙 — 총론

**과정명**: ROS 2 Advanced (Isaac Sim) — 로봇 청소기 시뮬레이션 프로젝트
**대상**: [`../basics/00-syllabus.md`](../basics/00-syllabus.md) 완료 학습자. [`../nav2-advanced/00-syllabus.md`](../nav2-advanced/00-syllabus.md)를 먼저 마치면 Part D(Nav2 기반) 이해가 쉽지만 필수 선행은 아니다.
**성격**: `nav2-advanced`/`moveit2-advanced`와 달리 **Isaac Sim 기반**으로 진행하는 트랙이다 — CAD로 설계한 커스텀 로봇(로봇청소기)을 직접 시뮬레이션에 가져와 물리·센서·자율주행·예외처리까지 엔드투엔드로 구현한다. 표준 로봇(TurtleBot3/Panda) 트랙과 달리 "하드웨어 가상화"부터 시작하는 것이 핵심 차별점.
**실습 환경**: Isaac Sim + `~/ros2_ws` — ROS2 Jazzy 브릿지(`isaacsim.ros2.bridge`), Nav2/slam_toolbox는 apt 바이너리로 설치. Isaac Sim 실행은 `isaacsim-ros2` alias 사용([[ros2_isaac_curriculum]]에서 확립한 rclpy import 문제 회피용).
**대체 관계**: 기존에 미착수 상태로 남아있던 "Part 3 — Nav2+MoveIt2 통합 캡스톤" 계획은 보류하고, 이 트랙으로 방향을 전환한다.

이 문서는 총론이며, 각 주제의 상세 실습 가이드는 `01-*.md` ~ `10-*.md` 파일에 담는다.

---

## 1. 과정 목표

이 과정을 마치면:

1. CAD(STEP/IGES) 파일을 Isaac Sim에서 쓸 수 있는 경량 USD 로봇으로 변환하고, Rigid Body/Collider/Mass를 물리적으로 타당하게 설정할 수 있다.
2. Isaac Sim에 RTX Lidar, RGB-D 카메라, 접촉/근접 센서를 배치하고 OmniGraph(ActionGraph)로 ROS2 토픽으로 퍼블리시할 수 있다.
3. slam_toolbox로 시뮬레이션 공간의 지도를 만들고 저장할 수 있다.
4. Nav2 스택 위에서 커버리지 경로 계획(CPP)을 커스터마이징하고, 범퍼/추락/배터리 등 이벤트 기반 예외 처리 상태 기계를 설계·구현할 수 있다.

## 2. 전체 주제 목록

### Part A — 하드웨어 가상화 & 에셋 최적화

| # | 주제 | 핵심 키워드 |
|---|---|---|
| 1 | 베이스 모델 설정 & 메쉬 다듬기 | STEP/IGES→OBJ/USD, Mesh Converter, Decimation, Stage 계층화 |
| 2 | 충돌체(Collider) 및 무게중심(COM) 수정 | Rigid Body API, Convex Hull/Decomposition, Mass/COM, Collision Mesh Debug Vis |
| 3 | 청소 가동부(브러쉬) 배치 | Revolute Joint, Rigid Body Material(마찰), Drive velocity target |

### Part B — 가상 센서 & 액션그래프

| # | 주제 | 핵심 키워드 |
|---|---|---|
| 4 | 2D LiDAR & RGB-D 카메라 배치 | RTX Lidar(Rotary), ROS 2 OmniGraphs(Lidar/Camera), frameId/토픽 설정 |
| 5 | 특수 센서 배치 (Bumper & 추락방지) | Contact Report API, Range Sensor, ActionGraph Compare 노드 |
| 6 | 마스터 액션그래프 & ROS2 브릿지 활성화 | TF/Odometry Publisher, ROS2 Subscribe Twist, 헤드리스 실행 |

### Part C — 자율주행 & 내비게이션 (SLAM)

| # | 주제 | 핵심 키워드 |
|---|---|---|
| 7 | 가상 학습 데이터 생성 | teleop_twist_keyboard, `ros2 topic hz /scan` |
| 8 | 격자 지도(Grid Map) 빌드 | slam_toolbox(online_async), RViz Transient Local, `map_saver_cli` |

### Part D — AI 기반 경로 계획 & 예외 처리

| # | 주제 | 핵심 키워드 |
|---|---|---|
| 9 | 커버리지 경로 계획 (CPP) | inflation_radius 튜닝, Boustrophedon 패턴, 커스텀 CPP 플래너 |
| 10 | 기능 구현 & 예외 처리 | 상태 기계(NORMAL/REVERSE/TURN/RETURN_TO_DOCK), 범퍼/추락/배터리 이벤트 |

## 3. 사전 준비물 (Part별 필요 개념)

**Part A**: 폴리곤/버텍스/페이스, 메쉬 경량화·리토폴로지, CAD→시뮬레이션 변환(단위 mm↔m), 강체 물리(질량/관성텐서/COM), Collider 종류별 연산 비용, PhysX USD 스키마, 정지/운동 마찰.

**Part B**: `sensor_msgs/LaserScan,PointCloud2,Image`, `nav_msgs/Odometry`, `tf2_msgs/TFMessage`, TF 트리(base_link/base_scan/odom/map, `/tf` vs `/tf_static`), OmniGraph(Execution Pulse vs Data Pulse), RTX Lidar 원리, RGB-D depth/캘리브레이션, Contact vs Range 센서, `isaacsim.ros2.bridge` 구조, QoS(Reliable/Best Effort).

**Part C**: SLAM(선후계란 문제), Occupancy Grid, 스캔 매칭, 루프 클로저, slam_toolbox(Sync/Async, resolution/max_laser_range), `.yaml`+`.pgm` 지도 포맷, AMCL vs SLAM, teleop_twist_keyboard 원리.

**Part D**: 전역/지역 플래너 차이, CPP(Boustrophedon/나선형/Spanning Tree), Costmap/inflation_radius, 상태 기계 설계(범퍼→후진/회전, 추락→정지, 배터리부족→도킹), `/cmd_vel` 제어 체계, 배터리·도킹 로직.

- ROS2 소스: Isaac Sim 실행 전 `source /opt/ros/jazzy/setup.bash` 필요, 실행은 `isaacsim-ros2` alias 사용
- 필요 시 설치: `sudo apt install ros-jazzy-teleop-twist-keyboard ros-jazzy-slam-toolbox ros-jazzy-navigation2 ros-jazzy-nav2-bringup`
- conda 등 가상환경 비활성화 필수 (이 환경에서 반복적으로 겪은 툴체인 충돌 — [[ros2_isaac_curriculum]], [[ros2_advanced_curriculum]] 모두에서 확인됨)
- CAD 원본 파일(STEP/IGES 등) 준비 — 없으면 대체 소스(무료 3D 모델 등)로 진행 가능

## 4. 진행 방식

기존 트랙과 동일한 문서 구조를 따른다.

1. **학습 목표**
2. **핵심 개념 설명**
3. **실습 단계** — 단계별 실행 순서, 실제 커맨드/코드 (GUI 조작이 핵심인 단계는 스크립트로 대체하지 않고 직접 조작하며 진행)
4. **예상/실제 결과 확인**
5. **알려진 문제와 해결** (실습 중 실제로 만난 에러만 기록)
6. **체크포인트**

GUI로 설계하도록 되어 있는 단계(메쉬 정리, Collider 설정, 센서 배치 등)는 직접 조작하며 진행하고, 코드 합성이 필요한 부분(액션그래프 결과를 감싸는 스크립트, 상태 기계 노드 등)은 함께 작성한다.

## 5. 참고자료

| 자료 | 용도 |
|---|---|
| [Isaac Sim 공식 문서](https://docs.isaacsim.omniverse.nvidia.com/) | Mesh Converter, PhysX, ROS2 Bridge, OmniGraph 레퍼런스 |
| [docs.nav2.org](https://docs.nav2.org/) | Costmap/Planner/Controller, CPP 커스터마이징 기준 |
| [Slam Toolbox](https://github.com/SteveMacenski/slam_toolbox) | SLAM 파라미터 레퍼런스 |

## 6. 이후 트랙

- (보류) Nav2 + MoveIt2 통합 캡스톤 — 이 트랙 완료 후 재검토

---
Part A 1단계부터 순차 진행 예정. 완료된 토픽 없음.
