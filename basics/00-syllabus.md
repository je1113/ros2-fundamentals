# ROS 2 기본기 튜토리얼 — 총론

**과정명**: ROS 2 Basics — 패키지 생성부터 ros2_control 입문까지
**대상**: ROS 2를 처음 접하거나, 개념을 순서대로 다시 다지고 싶은 학습자
**성격**: Isaac Sim/Gazebo 없이 **ROS 2 자체의 핵심 개념 13가지**를 순서대로 실습하고, 마지막에 배운 것을 종합하는 미니 프로젝트(행맨 게임) 1개로 마무리하는 순수 ROS 2 기초 트랙.
**실습 환경**: `~/ros2_ws` (신규 패키지 `ros2_basics`, `ros2_basics_msgs` 사용 — 기존 `demo_pkg`/`demo_pkg_msgs`와 분리)

이 문서는 총론이며, 각 주제의 상세 실습 가이드는 `01-*.md` ~ `15-*.md` 파일에 담는다.

---

## 1. 과정 목표

이 과정을 마치면:

1. ROS 2 워크스페이스와 패키지 구조를 스스로 만들고 관리할 수 있다.
2. 노드, 토픽(pub/sub), 서비스, 액션의 통신 모델과 각각을 언제 써야 하는지 구분할 수 있다.
3. QoS, Executor/콜백 그룹, 컴포지션 등 통신 안정성과 실행 모델에 관련된 개념을 이해하고 디버깅할 수 있다.
4. 커스텀 인터페이스(msg/srv/action)를 정의하고, 파라미터와 launch로 노드를 구성·실행할 수 있다.
5. tf2 좌표계, 시간(Time/`use_sim_time`), rosbag2/디버깅 도구, ros2_control까지 실전에서 필요한 도구 체인을 다룰 수 있다.

## 2. 전체 주제 목록

| # | 주제 | 핵심 키워드 |
|---|---|---|
| 1 | 패키지 생성 | workspace, colcon, `ament_python`, `package.xml` |
| 2 | 노드 & Publisher/Subscriber | `rclpy.node.Node`, topic, pub/sub |
| 3 | QoS (Quality of Service) | reliability, durability, history, depth |
| 4 | 커스텀 msg 정의 | `.msg`, `ament_cmake`/`rosidl`, 인터페이스 빌드 |
| 5 | 서비스 (Service/Client) | `.srv`, request/response, 동기 호출 |
| 6 | 액션 (Action) | `.action`, goal/feedback/result, 장시간 작업 |
| 7 | Executor & 콜백 그룹 | SingleThreaded/MultiThreadedExecutor, 데드락, ReentrantCallbackGroup |
| 8 | 컴포지션 (Component Node) | `rclcpp_components`/`rclpy` composition, 단일 프로세스 다중 노드 |
| 9 | 파라미터 (Parameters) | `declare_parameter`, YAML, 파라미터 콜백 |
| 10 | launch 파일 | `launch_ros`, substitution, 다중 노드 실행 |
| 11 | tf2 & 시간(Time/`use_sim_time`) | static/dynamic broadcaster, listener, 시뮬레이션 시계 |
| 12 | rosbag2 & 디버깅 툴 | `ros2 bag`, `rqt_graph`, `ros2 doctor`, `ros2 topic/node echo` |
| 13 | ros2_control 입문 | controller_manager, hardware interface, 궤적 제어 데모 |
| 14 | 미니 프로젝트: 행맨 게임 | 서비스, 파라미터, 토픽 종합 실습 (신규 개념 없음) |
| 15 | rqt_console (추가 토픽) | `/rosout`, Exclude/Highlight 필터, From Node 필터 — Topic 12 `rqt_graph`의 짝 |
| 16 | Lifecycle Node (추가 토픽) | `LifecycleNode`, managed state(unconfigured/inactive/active/finalized), `on_configure`/`on_activate`, `ros2 lifecycle` |

## 3. 사전 준비물

- ROS 2(Humble/Jazzy) 설치 및 환경 소싱 완료
- `~/ros2_ws`에 기존 `demo_pkg`/`demo_pkg_msgs`가 있어도 무방 — 이번 트랙은 별도 패키지(`ros2_basics`, `ros2_basics_msgs`)를 새로 만들어 진행
- conda 등 가상환경 비활성화 (ROS 2 툴체인과 반복적으로 충돌하는 `CONDA_PREFIX`/CMake 문제 방지)

## 4. 진행 방식

각 주제 문서는 다음 구조를 따른다.

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
| `docs.ros.org` | ROS 2 공식 튜토리얼 — 이 커리큘럼의 주제 선정 기준 |
| `design.ros2.org` | QoS, Executor 등 설계 문서 |
| `control.ros.org` | ros2_control 공식 문서 |

---
다음: [`01-package-creation.md`](./01-package-creation.md)
