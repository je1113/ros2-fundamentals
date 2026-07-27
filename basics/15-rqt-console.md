# rqt_console — 여러 노드의 로그를 한 화면에서 필터링

`00-syllabus.md`에는 없던 추가 토픽. Topic 2에서 "`get_logger()`로 남긴 로그는 `/rosout` 토픽으로도 발행되어, `rqt_console` 같은 도구로 여러 노드의 로그를 한 화면에 모아 필터링해서 볼 수 있다"고 언급만 하고 실습은 미뤄뒀던 부분이다. Topic 12에서 `rqt_graph`(노드-토픽 **연결 구조**)를 다뤘다면, 이번엔 `rqt_console`로 각 노드가 실제로 **무슨 로그를 남기는지**를 본다 — 구조를 보는 도구와 로그 내용을 보는 도구는 역할이 다르다.

새 노드를 작성하지 않고, 기존 배터리 파이프라인(`battery_publisher`/`battery_watcher`/`fleet_monitor`)을 그대로 재사용한다.

---

## 학습 목표

- `rqt_console`로 `/rosout`에 모이는 여러 노드의 로그를 한 화면에서 확인할 수 있다.
- Exclude/Highlight 필터로 원하는 로그만 남기거나 강조할 수 있다.
- "From Node" 같은 타입별 필터로 특정 노드의 로그만 골라볼 수 있다.

## 핵심 개념

**`rqt_graph`(구조) vs `rqt_console`(내용)**: `rqt_graph`는 "무엇이 무엇에 연결돼 있는가"를 보여주고, `rqt_console`은 "각 노드가 실제로 무슨 말을 하고 있는가"(로그)를 보여준다. 터미널을 여러 개 띄워 각각의 출력을 눈으로 대조하는 대신, `/rosout`을 구독하는 `rqt_console` 하나로 모든 노드의 로그를 한곳에서 볼 수 있다(Topic 2에서 다룬 "`print`는 분산 디버깅이 어렵다"는 문제의 실제 해결책).

**Highlight와 Exclude는 반대 방향의 필터다**: `Exclude`는 조건에 맞는 로그를 화면에서 **아예 지운다**. `Highlight`는 지우지 않고, 조건에 맞는 로그는 그대로 두고 **나머지를 회색으로 dim**시켜 상대적으로 눈에 띄게 만든다 — 글자색을 칠하거나 배경색을 넣는 방식이 아니라, "관심 없는 것들을 흐리게" 만드는 방식이다.

**필터 타입(+버튼)**: 기본 입력란은 로그 메시지 텍스트 전체를 대상으로 하는 자유 텍스트 검색이다. `+` 버튼으로 필터 타입을 추가하면(예: "From Node") 텍스트를 직접 입력하는 대신 **현재 로그에 등장한 노드 이름 목록에서 클릭으로 선택**해 그 노드의 로그만 골라낼 수 있다 — 노드 이름을 정확히 기억/타이핑할 필요가 없다.

**Severity(로그 레벨) 색상은 Exclude/Highlight와 별개**: Info/Warn 등 로그 레벨은 Severity 컬럼 자체에 이미 색으로 구분되어 표시된다 — 이건 필터를 걸지 않아도 항상 보이는, `get_logger()`의 레벨 시스템이 주는 기본 혜택이다(Topic 2에서 언급한 "로그 레벨 필터링이 가능하다"의 시각적 버전).

## 실습 단계

### 1. 배터리 파이프라인 실행

```bash
conda deactivate
cd ~/ros2_ws
source install/setup.bash
ros2 launch ros2_basics bringup.launch.py
```

### 2. rqt_console 실행

새 터미널에서:

```bash
conda deactivate
source ~/ros2_ws/install/setup.bash
rqt_console
```

**`rqt_console: command not found`가 뜨면** 아래 대안을 쓴다 (5. 알려진 문제 참고):

```bash
ros2 run rqt_console rqt_console
```

### 3. 기본 화면 확인

로그가 그리드로 흐르고, Severity 컬럼에 Info/Warn이 색으로 구분되어 보이는지 확인한다.

### 4. Highlight로 특정 로그 강조

화면 하단(또는 상단)의 **Highlight** 입력란에 `부족`을 입력하고 Enter — "부족"이 포함된 로그(배터리 부족 경고)는 그대로, 나머지 로그들은 회색으로 dim되는 것을 확인한다.

### 5. Exclude로 시끄러운 로그 숨기기

**Exclude** 입력란에 `발행`을 입력하고 Enter — `battery_publisher`가 매초 남기는 "배터리 발행: N%" 로그가 화면에서 사라지고, `battery_watcher`/`fleet_monitor`의 로그만 남는 것을 확인한다.

### 6. 노드 이름으로 직접 필터링

Exclude(또는 Highlight) 옆의 **`+`** 버튼을 눌러 필터 타입을 추가하고 **"From Node"**를 선택한다. 텍스트를 입력하는 대신 현재까지 로그를 남긴 노드 목록이 나열되므로, 그중 하나(예: `battery_publisher`)를 클릭해서 그 노드의 로그만 걸러본다.

## 4. 예상/실제 결과 확인

- Highlight("부족"): 일치하는 로그는 검은 글씨 그대로, 나머지가 회색으로 dim됨. (확인됨)
- Exclude("발행"): 일치하는 로그가 화면에서 완전히 사라짐. (확인됨)
- Severity 컬럼: Info/Warn이 서로 다른 색으로 구분되어 표시됨(필터와 무관하게 항상). (확인됨)
- "From Node" 필터: 텍스트 입력 없이 노드 이름 목록에서 클릭으로 선택해 필터링 가능. (확인됨)

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| `/opt/ros/jazzy/setup.bash`를 소싱한 뒤에도 `rqt_console: command not found` | `ros-jazzy-rqt-console` 패키지는 설치돼 있지만(`dpkg -l`로 확인됨) 독립 실행 스크립트가 이 설치본에서는 PATH에 안 잡힘 | `ros2 run rqt_console rqt_console`로 실행 |

## 6. 체크포인트

- [x] `rqt_console`로 여러 노드의 `/rosout` 로그를 한 화면에서 확인했다.
- [x] Highlight(dim 방식)와 Exclude(완전 제거) 필터의 동작 차이를 구분한다.
- [x] "From Node" 타입 필터로 특정 노드의 로그만 골라낼 수 있다.
- [x] Severity 컬럼의 로그 레벨 색상 구분을 확인했다.

---
`rqt_graph`(Topic 12, 구조) + `rqt_console`(이 토픽, 내용)로 ROS 2 GUI 디버깅 도구의 두 축을 모두 다뤘다.
