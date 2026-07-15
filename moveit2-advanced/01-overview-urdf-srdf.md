# 01. MoveIt2 개요 & URDF/SRDF

## 1. 학습 목표

- URDF와 SRDF의 역할 차이를 이해한다.
- SRDF의 핵심 요소(planning group, group state, disable_collisions, end_effector)를 실제 파일에서 읽어낼 수 있다.
- `move_group` 노드와 RViz MotionPlanning 플러그인으로 대화형 모션 플래닝을 실행해본다.

## 2. 핵심 개념

- **URDF**는 로봇의 물리적 구조(링크, 조인트, 관성, 시각/충돌 형상)를 정의한다 — Nav2 트랙에서 계속 다뤄온 것과 동일한 개념.
- **SRDF**(Semantic Robot Description Format)는 URDF 위에 MoveIt이 필요로 하는 의미 정보를 추가한다. URDF를 대체하거나 확장하는 게 아니라, URDF와 별개로 존재하며 URDF의 링크/조인트 이름을 참조한다.
  - `<group>`: 플래닝할 조인트/링크 묶음 정의 (체인, 개별 조인트/링크, 서브그룹 조합 등 여러 방식으로 지정 가능)
  - `<group_state>`: 그룹의 이름 붙은 자세 (예: "ready", "extended") — 조인트 값의 프리셋
  - `<disable_collisions>`: 항상/절대 충돌하지 않는 링크 쌍을 미리 제외해서 충돌 체크 비용을 줄임
  - `<end_effector>`: 특정 그룹을 다른 그룹(팔)의 엔드이펙터로 지정
  - `<virtual_joint>`: 로봇 베이스와 외부 좌표계(world) 사이의 가상 조인트
- 이 SRDF를 손으로 짜는 대신 **MoveIt Setup Assistant** GUI로 URDF를 불러와 마법사 형태로 생성할 수 있다.
- **`move_group`** 노드가 실제 모션 플래닝을 담당하는 핵심 노드다. `MoveGroupInterface`(다음 토픽에서 다룸)가 여기에 요청을 보낸다.

## 3. 실습 단계

### 3.1 설치

```bash
sudo apt install ros-jazzy-moveit ros-jazzy-moveit-resources ros-jazzy-moveit-visual-tools
```

### 3.2 Panda 데모 실행

```bash
ros2 launch moveit_resources_panda_moveit_config demo.launch.py
```

RViz에 Panda 로봇 팔과 MotionPlanning 패널이 뜬다.

### 3.3 SRDF 읽어보기

```bash
cat /opt/ros/jazzy/share/moveit_resources_panda_moveit_config/config/panda.srdf
```

`panda_arm` 그룹이 `panda_link0`→`panda_link8` 체인으로 정의된 것, `hand` 그룹이 별도로 정의되고 `end_effector`로 `panda_arm`에 연결된 것, `disable_collisions`로 인접 링크 쌍들이 충돌 체크에서 제외된 것을 확인한다.

### 3.4 RViz에서 대화형 플래닝

1. MotionPlanning 패널에서 팔 끝의 인터랙티브 마커(주황색 공)를 드래그해 새 자세로 이동
2. "Planning" 탭 → **Plan** 클릭 (초록 고스트 팔로 경로 미리보기)
3. **Execute** 클릭 → 실제 로봇 팔이 그 경로대로 움직이는지 확인

## 4. 예상/실제 결과 확인

- RViz에 Panda 팔과 MotionPlanning 패널이 정상적으로 뜸.
- 인터랙티브 마커로 지정한 목표 자세로 Plan→Execute 시 실제 팔이 이동함.

## 5. 알려진 문제와 해결

이번 토픽에서는 설치 후 한 번에 정상 동작했고, 새로 마주친 에러는 없었음.

## 6. 체크포인트

- [ ] MoveIt2/Panda 바이너리 설치
- [ ] `demo.launch.py`로 Panda RViz 데모 실행
- [ ] SRDF 파일에서 group/group_state/disable_collisions/end_effector 요소 확인
- [ ] RViz 인터랙티브 마커로 Plan→Execute 실행, 실제 팔 이동 확인

---
다음: [`02-movegroup-interface.md`](./02-movegroup-interface.md) (미작성)
