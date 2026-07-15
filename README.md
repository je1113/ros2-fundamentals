# ROS 2 Fundamentals — 기초부터 Nav2/MoveIt2까지

ROS 2를 패키지 생성부터 자율주행(Nav2)·모션플래닝(MoveIt2)까지 순서대로 실습하는 튜토리얼 기록이다. Isaac Sim 없이, 표준 도구(Gazebo, TurtleBot3, Panda 등)로만 진행한다.

## 대상 독자

- ROS 2를 처음 접하거나, 개념을 순서대로 다시 다지고 싶은 학습자
- Isaac Sim 지식은 필요 없음 — 순수 ROS 2/Gazebo 트랙

## 구조

```
basics/           Part 0 — ROS 2 기초 13개 토픽 (완료)
nav2-advanced/    Part 1 — Nav2 자율주행 7개 토픽 (완료)
moveit2-advanced/ Part 2 — MoveIt2 모션플래닝 3개 토픽 (syllabus만 작성됨)
```

각 트랙 폴더의 `00-syllabus.md`가 총론이고, `NN-topic.md`가 토픽별 실습 가이드다. 실습에 사용한 ROS 2 워크스페이스 코드는 트랙별로 별도 관리한다.

## Part 0 — ROS 2 Basics

| # | 주제 | 가이드 |
|---|---|---|
| 1 | 패키지 생성 | [basics/01-package-creation.md](basics/01-package-creation.md) |
| 2 | 노드 & Publisher/Subscriber | [basics/02-node-pub-sub.md](basics/02-node-pub-sub.md) |
| 3 | QoS | [basics/03-qos.md](basics/03-qos.md) |
| 4 | 커스텀 msg | [basics/04-custom-msg.md](basics/04-custom-msg.md) |
| 5 | 서비스 | [basics/05-service.md](basics/05-service.md) |
| 6 | 액션 | [basics/06-action.md](basics/06-action.md) |
| 7 | Executor & 콜백 그룹 | [basics/07-executor.md](basics/07-executor.md) |
| 8 | 컴포지션 | [basics/08-composition.md](basics/08-composition.md) |
| 9 | 파라미터 | [basics/09-parameters.md](basics/09-parameters.md) |
| 10 | launch 파일 | [basics/10-launch.md](basics/10-launch.md) |
| 11 | tf2 & 시간 | [basics/11-tf2-time.md](basics/11-tf2-time.md) |
| 12 | rosbag2 & 디버깅 툴 | [basics/12-rosbag-debugging.md](basics/12-rosbag-debugging.md) |
| 13 | ros2_control 입문 | [basics/13-ros2-control.md](basics/13-ros2-control.md) |

전체 목표: [basics/00-syllabus.md](basics/00-syllabus.md)

## Part 1 — Nav2 Advanced

| # | 주제 | 가이드 | 상태 |
|---|---|---|---|
| 1 | Nav2 개요 & 아키텍처 (TurtleBot3+Gazebo 첫 실행 포함) | [nav2-advanced/01-overview-architecture.md](nav2-advanced/01-overview-architecture.md) | 완료 |
| 2 | ~~TurtleBot3 + Gazebo로 Nav2 첫 실행~~ (토픽 1에 통합) | — | — |
| 3 | SLAM으로 지도 생성 | [nav2-advanced/03-slam-map-generation.md](nav2-advanced/03-slam-map-generation.md) | 완료 |
| 4 | Costmap & Planner/Controller 플러그인 | [nav2-advanced/04-costmap-planner-controller-plugins.md](nav2-advanced/04-costmap-planner-controller-plugins.md) | 완료 |
| 5 | Behavior Tree로 Nav2 태스크 조합 | [nav2-advanced/05-behavior-tree.md](nav2-advanced/05-behavior-tree.md) | 완료 |
| 6 | Nav2 액션 클라이언트로 미션 작성 | [nav2-advanced/06-action-client-mission.md](nav2-advanced/06-action-client-mission.md) | 완료 |
| 7 | 멀티로봇 내비게이션 | [nav2-advanced/07-multi-robot-navigation.md](nav2-advanced/07-multi-robot-navigation.md) | 완료 |

전체 목표: [nav2-advanced/00-syllabus.md](nav2-advanced/00-syllabus.md)

## Part 2 — MoveIt2 Advanced

| # | 주제 | 가이드 | 상태 |
|---|---|---|---|
| 1 | MoveIt2 개요 & URDF/SRDF | [moveit2-advanced/01-overview-urdf-srdf.md](moveit2-advanced/01-overview-urdf-srdf.md) | 완료 |
| 2 | MoveGroupInterface로 모션 플래닝 | | 예정 |
| 3 | Planning Scene & 충돌 회피 | | 예정 |

전체 목표: [moveit2-advanced/00-syllabus.md](moveit2-advanced/00-syllabus.md)

## 이후 트랙 (미착수)

- **Part 3 — 통합**: Nav2 + MoveIt2 결합 모바일 매니퓰레이터 캡스톤

## 라이선스 / 참고

개인 학습 기록이며, ROS 2 및 각 패키지(Nav2, MoveIt2 등)의 공식 문서 정책을 따른다.
