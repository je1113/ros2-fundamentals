# ROS 2 Fundamentals — 기초부터 Nav2/MoveIt2, Isaac Sim까지

ROS 2를 패키지 생성부터 자율주행(Nav2)·모션플래닝(MoveIt2)까지 순서대로 실습하는 튜토리얼 기록이다. `basics`/`nav2-advanced`/`moveit2-advanced`는 Isaac Sim 없이 표준 도구(Gazebo, TurtleBot3, Panda 등)로만 진행하고, `isaacsim-ros2-advanced`는 Isaac Sim 기반으로 커스텀 로봇(로봇청소기)을 하드웨어 가상화부터 자율주행까지 엔드투엔드로 다룬다. `basics-cpp`는 `basics`(Python/rclpy)를 마친 뒤, 통신 개념을 다시 설명하지 않고 rclcpp가 실제로 다르게 동작하는 지점만 짚는 압축 트랙이다.

## 대상 독자

- ROS 2를 처음 접하거나, 개념을 순서대로 다시 다지고 싶은 학습자
- 순수 ROS 2/Gazebo 트랙(`basics`~`moveit2-advanced`)은 Isaac Sim 지식이 필요 없음

## 구조

```
basics/                  Part 0 — ROS 2 기초 13개 토픽 + 미니 프로젝트 1개 (완료)
basics-cpp/              Part 0-CPP — rclcpp 핵심 차이 6개 토픽 (완료, 2026-07-27 실행 검증 — 6번 컴포지션 직접 확인)
nav2-advanced/           Part 1 — Nav2 자율주행 7개 토픽 (완료)
moveit2-advanced/        Part 2 — MoveIt2 모션플래닝 3개 토픽 (완료)
isaacsim-ros2-advanced/  Isaac Sim × ROS2 — 로봇청소기 시뮬레이션 프로젝트 (Part A~D, 10개 토픽, 진행 중)
isaacsim-standard-nav2/  Isaac Sim × ROS2 — 표준 에셋 기반 Nav2 검증 트랙 (5개 토픽, 완료 — wedging 미재현, 결론: vacuum 프로젝트 하드웨어 문제로 판단)
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
| 14 | 미니 프로젝트: 행맨 게임 | [basics/14-hangman-game.md](basics/14-hangman-game.md) |

전체 목표: [basics/00-syllabus.md](basics/00-syllabus.md)

## Part 0-CPP — ROS 2 Basics (C++/rclcpp)

`basics/`를 마친 뒤 이어서 보는 트랙. 통신 개념은 다시 설명하지 않고, rclcpp가 rclpy와 실제로 다르게 동작/작성되는 지점만 짚는다. 인터페이스는 새로 만들지 않고 `ros2_basics_msgs`를 재사용하며, 여러 토픽에서 Python 노드와 C++ 노드를 함께 실행해 상호운용을 확인한다.

| # | 주제 | 가이드 |
|---|---|---|
| 1 | 패키지 생성 & 빌드 구조 | [basics-cpp/01-package-creation.md](basics-cpp/01-package-creation.md) |
| 2 | 노드 & Publisher/Subscriber | [basics-cpp/02-node-pub-sub.md](basics-cpp/02-node-pub-sub.md) |
| 3 | 커스텀 인터페이스 소비 & 상호운용 | [basics-cpp/03-custom-interfaces.md](basics-cpp/03-custom-interfaces.md) |
| 4 | 서비스 비동기 패턴 | [basics-cpp/04-service-async.md](basics-cpp/04-service-async.md) |
| 5 | Executor & 콜백 그룹 | [basics-cpp/05-executor-callback-groups.md](basics-cpp/05-executor-callback-groups.md) |
| 6 | 컴포지션 (rclcpp_components) | [basics-cpp/06-composition.md](basics-cpp/06-composition.md) |

전체 목표: [basics-cpp/00-syllabus.md](basics-cpp/00-syllabus.md)

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
| 2 | MoveGroupInterface로 모션 플래닝 | [moveit2-advanced/02-movegroup-interface.md](moveit2-advanced/02-movegroup-interface.md) | 완료 |
| 3 | Planning Scene & 충돌 회피 | [moveit2-advanced/03-planning-scene-collision.md](moveit2-advanced/03-planning-scene-collision.md) | 완료 |

전체 목표: [moveit2-advanced/00-syllabus.md](moveit2-advanced/00-syllabus.md)

## Isaac Sim × ROS2 — 로봇청소기 시뮬레이션 프로젝트

Nav2+MoveIt2 통합 캡스톤(Part 3) 계획은 보류하고, Isaac Sim 기반 커스텀 로봇 프로젝트로 방향을 전환했다.

| # | 주제 | 가이드 | 상태 |
|---|---|---|---|
| 1 | 베이스 모델 설정 & 메쉬 다듬기 | — | 진행 예정 |
| 2 | 충돌체 및 무게중심 수정 | — | 진행 예정 |
| 3 | 청소 가동부(브러쉬) 배치 | — | 진행 예정 |
| 4 | 2D LiDAR & RGB-D 카메라 배치 | — | 진행 예정 |
| 5 | 특수 센서 배치 (Bumper & 추락방지) | — | 진행 예정 |
| 6 | 마스터 액션그래프 & ROS2 브릿지 활성화 | — | 진행 예정 |
| 7 | 가상 학습 데이터 생성 | — | 진행 예정 |
| 8 | 격자 지도 빌드 | — | 진행 예정 |
| 9 | 커버리지 경로 계획 (CPP) | — | 진행 예정 |
| 10 | 기능 구현 & 예외 처리 | — | 진행 예정 |

전체 목표: [isaacsim-ros2-advanced/00-syllabus.md](isaacsim-ros2-advanced/00-syllabus.md)

## Isaac Sim × ROS2 — 표준 에셋 기반 Nav2 검증 트랙

로봇청소기 프로젝트(Part D)에서 벽/코너 근처 목표로 이동할 때 로봇이 끼이는(wedging) 문제의 원인이 로봇 하드웨어(메쉬/조인트/콜라이더)인지 Nav2/설정인지 격리되지 않았다. 검증된 Isaac Sim 표준 로봇 에셋으로 새 USD 스테이지를 정석대로 구성해 같은 시나리오를 재현, 원인을 좁히는 독립 트랙.

| # | 주제 | 가이드 | 상태 |
|---|---|---|---|
| 1 | 에셋 선정 & 새 스테이지 구성 | [isaacsim-standard-nav2/01-asset-selection-new-stage.md](isaacsim-standard-nav2/01-asset-selection-new-stage.md) | 완료 |
| 2 | 로봇 물리/센서 구조 파악 | [isaacsim-standard-nav2/02-physics-sensor-structure.md](isaacsim-standard-nav2/02-physics-sensor-structure.md) | 완료 |
| 3 | ROS2 브릿지 & OmniGraph 연결 | [isaacsim-standard-nav2/03-ros2-bridge-omnigraph.md](isaacsim-standard-nav2/03-ros2-bridge-omnigraph.md) | 완료 |
| 4 | SLAM 지도 작성 | [isaacsim-standard-nav2/04-slam-grid-map.md](isaacsim-standard-nav2/04-slam-grid-map.md) | 완료 |
| 5 | Nav2 목표 주행 검증 | [isaacsim-standard-nav2/05-nav2-goal-driving.md](isaacsim-standard-nav2/05-nav2-goal-driving.md) | 완료 |

전체 목표: [isaacsim-standard-nav2/00-syllabus.md](isaacsim-standard-nav2/00-syllabus.md)

## 라이선스 / 참고

개인 학습 기록이며, ROS 2 및 각 패키지(Nav2, MoveIt2 등)의 공식 문서 정책을 따른다.
