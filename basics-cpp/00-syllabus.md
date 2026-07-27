# ROS 2 C++ (rclcpp) 핵심 차이 튜토리얼 — 총론

**과정명**: ROS 2 Basics — C++ (rclcpp) 트랙
**대상**: [`../basics/`](../basics/00-syllabus.md) (Python/rclpy) 트랙을 마쳤고, 토픽/서비스/파라미터 등 ROS 2 통신 개념은 이미 이해한 학습자
**성격**: 통신 개념 자체는 다시 설명하지 않는다. **rclcpp가 rclpy와 실제로 다르게 동작하거나 다르게 작성해야 하는 지점 6가지**만 짚는 압축 트랙.
**실습 환경**: `~/ros2_ws`에 신규 패키지 `ros2_basics_cpp`(ament_cmake) 추가. 인터페이스는 새로 만들지 않고 `basics/` 트랙에서 만든 `ros2_basics_msgs`를 그대로 재사용한다 — .msg/.srv는 언어 중립적이라는 걸 직접 확인하는 것도 이 트랙의 목적 중 하나다.

이 문서는 총론이며, 각 주제의 상세 실습 가이드는 `01-*.md` ~ `06-*.md` 파일에 담는다.

---

## 1. 과정 목표

이 과정을 마치면:

1. `ament_python`과 `ament_cmake` 빌드 구조의 차이를 이해하고, C++에서는 `--symlink-install`을 써도 코드가 바뀌면 재빌드(재컴파일)가 필요하다는 것을 안다.
2. rclcpp 노드에서 `shared_ptr` 라이프사이클과 람다/`std::bind` 콜백 등록 패턴을 구현할 수 있다.
3. `basics/`에서 만든 커스텀 인터페이스를 C++ 생성 헤더로 소비하고, 같은 토픽/서비스를 Python 노드와 C++ 노드가 함께 쓸 수 있음을 직접 확인한다.
4. rclcpp 서비스 클라이언트의 콜백 기반 비동기 호출 패턴을 구현할 수 있다.
5. 콜백 안에서 서비스를 동기 대기하면 어떤 문제가 나는지(Python 7번 주제와 같은 계열의 문제) C++에서 재현하고, 왜 `MultiThreadedExecutor`/콜백 그룹으로는 고쳐지지 않는지, 그리고 실제 해결책이 무엇인지 설명할 수 있다.
6. `rclcpp_components`로 노드를 공유 라이브러리 컴포넌트로 만들고, 하나의 컨테이너 프로세스에 여러 컴포넌트를 로드할 수 있다.

## 2. 전체 주제 목록

| # | 주제 | 핵심 키워드 | rclpy 대비 다른 점 |
|---|---|---|---|
| 1 | 패키지 생성 & 빌드 구조 | `ament_cmake`, `CMakeLists.txt`, 컴파일 | `--symlink-install`이 소스 변경엔 재빌드 필요 (Python은 즉시 반영) |
| 2 | 노드 & Pub/Sub | `rclcpp::Node`, `SharedPtr`, 람다 콜백 | 콜백 인자가 `MessageT::SharedPtr`, 퍼블리셔/타이머를 멤버로 들고 있어야 함(라이프타임) |
| 3 | 커스텀 인터페이스 소비 & 상호운용 | 생성된 헤더, `ros2_basics_msgs::msg::BatteryAlert` | 헤더 include 경로(snake_case)와 네임스페이스(PascalCase), getter 없이 public 멤버 직접 접근 |
| 4 | 서비스 비동기 패턴 | `async_send_request` + 콜백 | 블로킹 대기 없이 콜백으로 응답을 받는 게 rclcpp의 기본 패턴 |
| 5 | Executor & 콜백 그룹 | `MultiThreadedExecutor`, `CallbackGroup`, 노드-실행기 결합 제약 | 같은 노드를 실행기에 중첩으로 추가하면 조용히 멈추는 게 아니라 즉시 예외(`already been added to an executor`)가 나고, 콜백 그룹으로도 못 고침 — 해결책은 4번의 비동기 패턴으로 구조를 바꾸는 것 |
| 6 | 컴포지션 (`rclcpp_components`) | 공유 라이브러리 플러그인, `component_container` | Python 8번보다 실제 운영에서 훨씬 널리 쓰이는, C++의 강점이 드러나는 패턴 |

## 3. 사전 준비물

- [`../basics/`](../basics/00-syllabus.md) (Python) 트랙 완료 — 특히 3번(QoS), 4번(커스텀 msg), 5번(서비스), 7번(Executor/콜백 그룹) 주제의 배경 지식을 그대로 가져다 쓴다.
- `~/ros2_ws`에 `ros2_basics_msgs`, `ros2_basics`(Python 패키지)가 이미 빌드되어 있어야 함 — 3번/4번 주제에서 Python 노드와 함께 실행해 상호운용을 확인한다.
- C++14/17 기초: 스마트 포인터(`std::shared_ptr`), 람다, `std::bind`.

## 4. 진행 방식

각 주제 문서는 `basics/` 트랙과 동일한 구조를 따른다.

1. **학습 목표**
2. **핵심 개념 설명**
3. **실습 단계** — 단계별 실행 순서, 실제 코드/커맨드
4. **예상/실제 결과 확인**
5. **알려진 문제와 해결** (실습 중 실제로 만난 에러만 기록)
6. **체크포인트**

## 5. 참고자료

| 자료 | 용도 |
|---|---|
| `docs.ros.org` rclcpp 튜토리얼 | 이 트랙의 코드 패턴 기준 |
| `design.ros2.org` | Executor/콜백 그룹, 컴포지션 설계 문서 |

---
다음: [`01-package-creation.md`](./01-package-creation.md)
