# 02. MoveGroupInterface로 모션 플래닝

## 1. 학습 목표

- `moveit_py`(Python API)로 `MoveItPy`/`PlanningComponent`를 사용해 모션 플래닝 요청을 코드로 보낸다.
- `MoveItConfigsBuilder`로 SRDF/URDF/컨트롤러 설정을 불러와 `MoveItPy`에 넘기는 과정을 이해한다.
- 이름 붙은 자세(`group_state`)로 순서대로 이동하는 미션 스크립트를 작성하고, 실제로 self-collision 때문에 플래닝이 실패하는 경우까지 실제로 확인한다.

## 2. 핵심 개념

- `MoveItPy`는 **실행 중인 `move_group` 노드에 접속하는 방식이 아니다.** `MoveItConfigsBuilder`로 URDF/SRDF/컨트롤러 설정을 직접 불러와, 자기 자신의 내부 rclcpp 컨텍스트에 임베드된 형태로 독자적으로 플래닝한다 (`move_group`과 같은 라이브러리를 공유하지만 별개의 인스턴스).
- `panda_arm = panda.get_planning_component('panda_arm')`으로 SRDF의 planning group에 대응하는 `PlanningComponent`를 얻는다.
- `set_start_state_to_current_state()` / `set_goal_state(configuration_name=...)`로 시작/목표를 지정하고, `plan()` → `execute()` 순서로 실행한다 — RViz의 "Plan"/"Execute" 버튼과 동일한 동작을 코드로 재현하는 것.

## 3. 실습 단계

### 3.1 moveit_py 설치

```bash
sudo apt install ros-jazzy-moveit-py
```

### 3.2 패키지 작성

```
~/ros2_ws/src/moveit2_advanced/
├── package.xml
├── setup.py
├── setup.cfg
├── resource/moveit2_advanced
└── moveit2_advanced/
    ├── __init__.py
    └── mission_arm.py
```

`mission_arm.py`는 `MoveItConfigsBuilder('moveit_resources_panda')`로 `panda.urdf.xacro`/`panda.srdf`/`gripper_moveit_controllers.yaml`을 불러와 `MoveItPy`를 생성하고, `"ready"` → `"extended"` → `"transport"` 순서로 플래닝·실행한다.

### 3.3 실행

`demo.launch.py`(RViz)가 떠 있는 상태에서:

```bash
cd ~/ros2_ws
colcon build --packages-select moveit2_advanced
source install/setup.bash
ros2 run moveit2_advanced mission_arm
```

## 4. 예상/실제 결과 확인

- `"ready"`, `"extended"` 자세는 플래닝 성공 → 실행되어 RViz/Panda 팔이 실제로 이동함.
- `"transport"` 자세는 **플래닝 자체가 실패** — 로그에 `Found a contact between 'panda_link7' and 'panda_link5'`, `Unable to sample any valid states for goal tree`가 찍힘. 이 SRDF/URDF 조합에서는 `transport` 프리셋 자세 자체가 실제로 self-collision 상태라, 애초에 도달 불가능한 목표라는 뜻이다. 이름 붙은 상태(`group_state`)라고 해서 항상 충돌 없는 자세가 보장되는 건 아니라는 것을 실제로 확인.

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| `MoveItPy(config_dict=...)` 생성 시 `RuntimeError: Failed to load planning pipelines from parameter server` | `MoveItConfigsBuilder`의 dict는 리스트 값을 Python **tuple**로 저장한다. 일부 ROS 2 파라미터 전달 경로(`launch_ros`의 dict→파라미터파일 변환 등)가 tuple을 비표준 YAML 태그 `!!python/tuple`로 직렬화하는데, ROS 2의 안전 YAML 로더는 이 태그를 못 읽는다 | `config_dict`를 `MoveItPy`에 넘기기 전, 모든 tuple을 list로 재귀 변환하는 `tuples_to_lists()` 헬퍼를 거친다 |
| tuple을 list로 바꿔도 여전히 같은 에러 | `moveit_configs_utils`가 만드는 `planning_pipelines` 키는 **평평한 리스트**(`planning_pipelines: [ompl]`) 구조인데, 이건 `move_group` 노드 자체의 (오래된) 파라미터 로딩 경로가 기대하는 형식이다. `MoveItCpp`/`moveit_py` 파사드는 이와 달리 `planning_pipelines.pipeline_names` 형태의 **중첩 구조**를 기대한다 (라이브러리 문자열에서 `pipeline_names` 키를 직접 확인) | `config_dict['planning_pipelines'] = {'pipeline_names': ['ompl']}`로 덮어써서 `MoveItCpp`가 기대하는 구조를 맞춰준다 |
| 파이프라인은 로드됐지만 `No planning pipeline available for name ''`로 매 플래닝 실패 | `plan()`이 내부적으로 `plan_request_params.planning_pipeline` 등을 ROS 파라미터에서 읽으려 하는데, 이 네임스페이스를 설정해준 적이 없어서 빈 문자열로 기본값 처리됨 | `PlanRequestParameters(panda, 'plan_request_params')`를 만들고 `planning_pipeline`/`planner_id`/`planning_time` 등을 코드에서 직접 대입한 뒤 `plan(single_plan_parameters=...)`로 전달 |
| `demo.launch.py`를 막 (재)실행한 직후 실행하면 모든 목표가 `panda_hand - panda_link5, panda_link5 - panda_link7` self-collision으로 실패 | `MoveItPy`가 생성되자마자 플래닝을 시도하면 `current_state_monitor`가 아직 실제 `/joint_states`를 한 번도 못 받은 상태라, 기본/미초기화 상태를 기준으로 충돌 체크를 하게 됨 | `MoveItPy` 생성 직후 `time.sleep(2.0)`으로 실제 조인트 상태를 받을 시간을 준다 |
| 스크립트가 모든 목표 처리를 끝낸 뒤(`"transport" 플래닝 실패` 로그 이후) `Segmentation fault`로 종료 | `Deleting MoveItCpp` 로그 이후, `moveit_py` 내부 정리(pybind11/rclcpp 종료) 과정에서 발생하는 것으로 보임. 우리 스크립트의 로직(플래닝·실행·결과 출력)은 이미 다 끝난 뒤에 발생하므로 실습 결과 자체에는 영향 없음 | 무해한 것으로 확인, 별도 조치 없이 진행 (종료 직전 크래시이므로) |

## 6. 체크포인트

- [ ] `ros-jazzy-moveit-py` 설치
- [ ] `moveit2_advanced` 패키지 + `mission_arm.py` 작성 (tuple 변환, `pipeline_names` 중첩 구조, `PlanRequestParameters` 세 가지 우회 모두 반영)
- [ ] `"ready"`/`"extended"` 자세로 플래닝→실행 성공, RViz에서 실제 이동 확인
- [ ] `"transport"` 자세의 self-collision 플래닝 실패를 로그로 확인하고 원인 설명 가능

---
다음: [`03-planning-scene-collision.md`](./03-planning-scene-collision.md) (미작성)
