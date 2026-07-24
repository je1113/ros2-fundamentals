# Isaac Sim × ROS2 — 표준 에셋 기반 Nav2 검증 트랙 — 총론

**과정명**: 표준 Omniverse 에셋으로 다시 세우는 Nav2 파이프라인
**대상**: [`../isaacsim-ros2-advanced/00-syllabus.md`](../isaacsim-ros2-advanced/00-syllabus.md) Part C까지 진행한 학습자. 이 트랙은 그 트랙의 후속이 아니라 **별도 독립 트랙**이다.
**성격**: `isaacsim-ros2-advanced`는 CAD 커스텀 로봇(로봇청소기)을 메쉬 다듬기부터 직접 만들어 올렸다. 그 프로젝트의 Part D(Nav2 기반 커버리지 경로계획)에서 벽/코너 근처 목표 지점으로 이동할 때 로봇이 끼이는(wedging) 문제가 계속 재현되었고, 원인이 로봇 자체(메쉬/조인트/콜라이더)인지 Nav2/설정 쪽인지 격리되지 않은 채 디버깅이 길어졌다. 이 트랙은 **Isaac Sim Asset Browser의 검증된 표준 로봇 에셋**(사전 리깅된 wheel joint, 콜라이더, 센서 포함)으로 새 USD 스테이지를 처음부터 정석대로 구성해서, 같은 Nav2 시나리오를 재현했을 때도 같은 문제가 나오는지 확인한다.
**실습 환경**: Isaac Sim (`isaacsim-ros2` alias) + `~/ros2_ws` — ROS2 Jazzy 브릿지, Nav2/slam_toolbox는 apt 바이너리. [[isaacsim_ros2_advanced_curriculum]]에서 확립한 conda 비활성화, Y-up 확인 등 표준 절차를 그대로 따른다.

이 문서는 총론이며, 각 주제의 상세 실습 가이드는 `01-*.md` ~ `05-*.md` 파일에 담는다.

---

## 1. 과정 목표

이 과정을 마치면:

1. Isaac Sim Asset Browser에서 표준 로봇 에셋을 새 USD 스테이지에 배치하고, 커스텀 CAD 에셋과 무엇이 다른지(사전 리깅된 물리/센서 구조) 설명할 수 있다.
2. 표준 에셋의 기존 wheel joint/콜라이더/센서 구성을 읽고, 커스텀 로봇청소기 프로젝트에서 겪었던 문제(Convex Hull 붕괴, joint localPos 오염 등)와 비교할 수 있다.
3. ROS2 브릿지와 OmniGraph를 표준 에셋 기준으로 다시 연결하고 정상 동작을 검증할 수 있다.
4. slam_toolbox로 지도를 만들고, Nav2로 벽/코너에 가까운 목표 지점까지 안정적으로 주행시킬 수 있다.
5. 벽 끼임 문제가 로봇 하드웨어 이슈였는지 Nav2/설정 이슈였는지에 대한 근거 있는 결론을 내릴 수 있다.

## 2. 전체 주제 목록

| # | 주제 | 핵심 키워드 |
|---|---|---|
| 1 | 에셋 선정 & 새 스테이지 구성 | Asset Browser, 표준 로봇 에셋, 새 USD 스테이지, Y-up/스케일 확인 |
| 2 | 로봇 물리/센서 구조 파악 | 사전 리깅된 Wheel Joint, Collider, LiDAR/카메라 — 커스텀 에셋과 비교 |
| 3 | ROS2 브릿지 & OmniGraph 연결 | Action Graph, ROS2 Bridge 노드, `/tf`·`/odom`·`/scan` 검증 |
| 4 | SLAM 지도 작성 | slam_toolbox(online_async), teleop, `map_saver_cli` |
| 5 | Nav2 목표 주행 검증 | 벽/코너 근접 목표, wedging 재현 여부, vacuum 프로젝트와 결과 비교 |

## 3. 사전 준비물

폴리곤/메쉬 기초, USD Stage 계층 구조, PhysX Rigid Body/Joint 개념, `sensor_msgs`/`nav_msgs`/`tf2_msgs`, OmniGraph 기본기, slam_toolbox/Nav2 기본 개념 — 모두 [`../isaacsim-ros2-advanced/00-syllabus.md`](../isaacsim-ros2-advanced/00-syllabus.md) Part A~C에서 이미 다뤘다.

## 4. 진행 상태

| # | 주제 | 가이드 | 상태 |
|---|---|---|---|
| 1 | 에셋 선정 & 새 스테이지 구성 | [01-asset-selection-new-stage.md](01-asset-selection-new-stage.md) | 완료 |
| 2 | 로봇 물리/센서 구조 파악 | [02-physics-sensor-structure.md](02-physics-sensor-structure.md) | 완료 |
| 3 | ROS2 브릿지 & OmniGraph 연결 | — | 진행 예정 |
| 4 | SLAM 지도 작성 | — | 진행 예정 |
| 5 | Nav2 목표 주행 검증 | — | 진행 예정 |

## 5. 관련 프로젝트와의 관계

- `isaacsim-ros2-advanced`(로봇청소기): 이 트랙에서 wedging 원인이 로봇 하드웨어 문제로 판명되면 그 프로젝트로 돌아가 콜라이더/조인트를 재점검한다. Nav2/설정 문제로 판명되면 그 결론을 그대로 로봇청소기 프로젝트의 `nav2_params.yaml`에 반영한다.
- `nav2-advanced`(TurtleBot3+Gazebo): Isaac Sim이 아닌 순수 ROS2/Gazebo 환경에서의 Nav2 기초 트랙. 개념은 겹치지만 시뮬레이터가 다르다.
