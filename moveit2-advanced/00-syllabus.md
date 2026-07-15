# ROS 2 Advanced — MoveIt2 트랙 — 총론

**과정명**: ROS 2 Advanced Part 2 — MoveIt2 매니퓰레이터 모션플래닝
**대상**: [`../nav2-advanced/00-syllabus.md`](../nav2-advanced/00-syllabus.md) Nav2 트랙을 마친 학습자 (Nav2 지식을 직접 전제하진 않지만, lifecycle 노드/파라미터/액션 클라이언트 등 같은 개념 기반 위에서 진행)
**성격**: `nav2-advanced`와 마찬가지로 Isaac Sim 없이 **표준 매니퓰레이터(Panda) + MoveIt2**로 진행하는 독립 트랙. Nav2 트랙과 별개로 진행 가능하며, 두 트랙을 합치는 통합 캡스톤은 이후 Part 3로 미룬다(미착수).
**실습 환경**: `~/ros2_ws` — MoveIt2/Panda는 apt 바이너리 패키지(`ros-jazzy-moveit`, `ros-jazzy-moveit-resources`)로 설치, 커스텀 코드(모션 스크립트, 커스텀 플래닝 씬)는 신규 패키지 `moveit2_advanced`에 작성

이 문서는 총론이며, 각 주제의 상세 실습 가이드는 `01-*.md` ~ `03-*.md` 파일에 담는다.

---

## 1. 과정 목표

이 과정을 마치면:

1. URDF/SRDF의 역할 차이와, MoveIt Setup Assistant로 SRDF(플래닝 그룹, 충돌 매트릭스 등)를 생성하는 과정을 이해한다.
2. `MoveGroupInterface`(Python `moveit_py`)로 매니퓰레이터에 모션 플래닝 goal을 보내고 실행할 수 있다.
3. Planning Scene에 충돌 오브젝트를 추가/제거하고, 충돌 회피 플래닝이 실제로 동작하는 것을 확인할 수 있다.
4. RViz의 MotionPlanning 플러그인으로 대화형으로 플래닝 결과를 확인하고 디버깅할 수 있다.

## 2. 전체 주제 목록

| # | 주제 | 핵심 키워드 |
|---|---|---|
| 1 | MoveIt2 개요 & URDF/SRDF | URDF vs SRDF, planning group, MoveIt Setup Assistant, `move_group` 노드 |
| 2 | MoveGroupInterface로 모션 플래닝 | `moveit_py`, `MoveGroupInterface`, 조인트/포즈 goal, plan & execute |
| 3 | Planning Scene & 충돌 회피 | `PlanningSceneInterface`, collision object, RViz MotionPlanning 플러그인 |

## 3. 사전 준비물

- [`nav2-advanced`](../nav2-advanced/00-syllabus.md) 또는 최소 [`basics`](../basics/00-syllabus.md) 트랙 완료 (특히 파라미터, launch, 액션 개념)
- MoveIt2/Panda 바이너리 설치: `sudo apt install ros-jazzy-moveit ros-jazzy-moveit-resources ros-jazzy-moveit-visual-tools`
- conda 등 가상환경 비활성화 (기존 트랙에서 반복적으로 겪은 툴체인 충돌 방지 — 이번 트랙에서도 매 터미널마다 확인)

## 4. 진행 방식

`nav2-advanced` 트랙과 동일한 문서 구조를 따른다.

1. **학습 목표**
2. **핵심 개념 설명**
3. **실습 단계** — 단계별 실행 순서, 실제 커맨드/코드
4. **예상/실제 결과 확인**
5. **알려진 문제와 해결** (실습 중 실제로 만난 에러만 기록)
6. **체크포인트**

실습은 직접 터미널에서 명령을 실행하며 진행한다 — 코드/명령은 가이드로 제공하고, 실제 실행과 결과 확인은 본인이 직접 한다. (단, 이 커리큘럼에서는 코드/설정 파일 작성은 필요에 따라 Claude가 직접 작성하고, 빌드·실행·결과 확인은 본인이 직접 하는 방식도 병행한다.)

## 5. 참고자료

| 자료 | 용도 |
|---|---|
| [moveit.picknik.ai](https://moveit.picknik.ai/) | MoveIt2 공식 문서 — 이 트랙의 주제 선정 기준 |
| [moveit2_tutorials](https://github.com/moveit/moveit2_tutorials) | 공식 튜토리얼 예제 코드 |

## 6. 이후 트랙 (미착수)

- **Part 3 — 통합**: [`../nav2-advanced/`](../nav2-advanced/00-syllabus.md) + MoveIt2 결합 모바일 매니퓰레이터 캡스톤 — "이동 후 픽업" 미션

---
다음: `01-overview-urdf-srdf.md` (미작성)
