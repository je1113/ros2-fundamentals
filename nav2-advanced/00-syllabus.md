# ROS 2 Advanced — Nav2 트랙 — 총론

**과정명**: ROS 2 Advanced Part 1 — Nav2 자율주행 내비게이션
**대상**: [`../basics/00-syllabus.md`](../basics/00-syllabus.md) 13개 토픽(패키지~ros2_control)을 마친 학습자
**성격**: `basics` 트랙과 마찬가지로 Isaac Sim 없이 **표준 시뮬레이션 로봇(TurtleBot3) + Gazebo**로 진행하는 순수 ROS 2 트랙. MoveIt2(모션플래닝)는 별도 트랙(`moveit2-advanced/`, 미착수)으로 분리한다.
**실습 환경**: `~/ros2_ws` — Nav2/TurtleBot3는 apt 바이너리 패키지(`ros-jazzy-navigation2`, `ros-jazzy-nav2-bringup`, `ros-jazzy-turtlebot3*`)로 설치, 커스텀 코드(미션 클라이언트, BT 플러그인)는 신규 패키지 `nav2_advanced`에 작성

이 문서는 총론이며, 각 주제의 상세 실습 가이드는 `01-*.md` ~ `07-*.md` 파일에 담는다.

---

## 1. 과정 목표

이 과정을 마치면:

1. Nav2 스택의 아키텍처(lifecycle 노드, BT 기반 orchestration)를 이해하고 직접 실행/디버깅할 수 있다.
2. SLAM으로 지도를 만들고, AMCL로 localization하며, Nav2를 통해 로봇을 자율주행시킬 수 있다.
3. Costmap, Planner, Controller 플러그인의 역할을 이해하고 파라미터를 조정할 수 있다.
4. Behavior Tree로 Nav2 태스크를 조합하고, 커스텀 BT 노드를 작성할 수 있다.
5. `NavigateToPose`, `NavigateThroughPoses`, `FollowWaypoints` 액션 클라이언트로 미션 코드를 작성할 수 있다.
6. 네임스페이스/도메인 ID를 이용한 멀티로봇 내비게이션 구조를 구성할 수 있다.

## 2. 전체 주제 목록

| # | 주제 | 핵심 키워드 |
|---|---|---|
| 1 | Nav2 개요 & 아키텍처 (TurtleBot3+Gazebo 첫 실행 포함, 토픽 2와 통합 진행됨) | lifecycle node, bt_navigator, planner/controller/behavior server, map_server, AMCL, nav2_bringup |
| ~~2~~ | ~~TurtleBot3 + Gazebo로 Nav2 첫 실행~~ (토픽 1에 통합) | — |
| 3 | SLAM으로 지도 생성 | slam_toolbox, `ros2 run nav2_map_server map_saver_cli` |
| 4 | Costmap & Planner/Controller 플러그인 | global/local costmap, NavFn/SmacPlanner, DWB/RPP controller |
| 5 | Behavior Tree로 Nav2 태스크 조합 | BT.CPP, `.xml` BT, 커스텀 BT 노드 |
| 6 | Nav2 액션 클라이언트로 미션 작성 | `NavigateToPose`, `NavigateThroughPoses`, `FollowWaypoints` |
| 7 | 멀티로봇 내비게이션 | namespace, `ROS_DOMAIN_ID`, TF 프리픽스 |

## 3. 사전 준비물

- [`basics`](../basics/00-syllabus.md) 트랙 완료 (특히 06-action, 09-parameters, 10-launch, 11-tf2-time)
- Nav2/TurtleBot3 바이너리 설치: `sudo apt install ros-jazzy-navigation2 ros-jazzy-nav2-bringup ros-jazzy-turtlebot3-gazebo ros-jazzy-slam-toolbox`
- conda 등 가상환경 비활성화 (기존 트랙에서 반복적으로 겪은 툴체인 충돌 방지)

## 4. 진행 방식

`basics` 트랙과 동일한 문서 구조를 따른다.

1. **학습 목표**
2. **핵심 개념 설명**
3. **실습 단계** — 단계별 실행 순서, 실제 커맨드/코드
4. **예상/실제 결과 확인**
5. **알려진 문제와 해결** (실습 중 실제로 만난 에러만 기록)
6. **체크포인트**

실습은 직접 터미널에서 명령을 실행하며 진행한다 — 코드/명령은 가이드로 제공하고, 실제 실행과 결과 확인은 본인이 직접 한다.

## 5. 참고자료

| 자료 | 용도 |
|---|---|
| [docs.nav2.org](https://docs.nav2.org/) | Nav2 공식 문서 — 이 트랙의 주제 선정 기준 |
| [navigation2_tutorials](https://github.com/ros-navigation/navigation2_tutorials) | 공식 튜토리얼 예제 코드 |

## 6. 이후 트랙 (미착수)

- **Part 2 — MoveIt2**: URDF/SRDF, Setup Assistant, MoveGroupInterface, Planning Scene (표준 로봇: Panda)
- **Part 3 — 통합**: Nav2 + MoveIt2 결합 모바일 매니퓰레이터 캡스톤

---
Nav2 Advanced 트랙 완료 (토픽 1, 3~7). 다음은 Part 2 — MoveIt2 트랙(`../moveit2-advanced/`, 미착수).

- [`01-overview-architecture.md`](./01-overview-architecture.md)
- [`03-slam-map-generation.md`](./03-slam-map-generation.md)
- [`04-costmap-planner-controller-plugins.md`](./04-costmap-planner-controller-plugins.md)
- [`05-behavior-tree.md`](./05-behavior-tree.md)
- [`06-action-client-mission.md`](./06-action-client-mission.md)
- [`07-multi-robot-navigation.md`](./07-multi-robot-navigation.md)
