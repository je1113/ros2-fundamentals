# 05. Behavior Tree로 Nav2 태스크 조합

## 1. 학습 목표

- `bt_navigator`가 실행하는 BT.CPP 기반 XML의 구조를 읽고 이해한다.
- BT의 각 노드가 다른 서버(planner/controller/behavior)에 대한 액션 클라이언트라는 것을 이해한다.
- 커스텀 BT XML을 만들어 `bt_navigator`에 연결하고, 트리 구조 변경이 실제 회복(recovery) 동작에 미치는 영향을 관찰한다.

## 2. 핵심 개념

`bt_navigator`는 XML로 정의된 Behavior Tree를 실행하며 전체 내비게이션 흐름을 조율한다. 기본 BT(`navigate_to_pose_w_replanning_and_recovery.xml`)의 구조:

```
RecoveryNode (number_of_retries=6, name=NavigateRecovery)
├── PipelineSequence (NavigateWithReplanning)  ─ 정상 주행 루프
│   ├── RateController(1Hz) → ComputePathToPose (planner_server 액션 클라이언트)
│   └── FollowPath (controller_server 액션 클라이언트)
└── Sequence  ─ 실패 시 회복 로직
    └── ReactiveFallback (RecoveryFallback)
        └── RoundRobin: 코스트맵 클리어 → Spin → Wait → BackUp
```

- **`RecoveryNode`**는 자식이 실패하면 두 번째 자식(회복 로직)을 실행한 뒤 재시도한다. `number_of_retries`가 이 재시도 횟수를 결정한다.
- BT 노드 하나하나가 실제로는 해당 서버에 대한 **액션 클라이언트**다 (`ComputePathToPose` 노드 = `planner_server`의 액션 클라이언트, `FollowPath` 노드 = `controller_server`의 액션 클라이언트).
- BT XML은 `bt_navigator`의 `default_nav_to_pose_bt_xml` 파라미터로 지정하며, 코드 수정 없이 새 XML 파일 경로만 바꾸면 전체 내비게이션 로직을 교체할 수 있다.

## 3. 실습 단계

### 3.1 기본 BT XML 확인

```bash
cat /opt/ros/jazzy/share/nav2_bt_navigator/behavior_trees/navigate_to_pose_w_replanning_and_recovery.xml
```

### 3.2 커스텀 BT XML 작성

```bash
cp /opt/ros/jazzy/share/nav2_bt_navigator/behavior_trees/navigate_to_pose_w_replanning_and_recovery.xml ~/ros2_ws/nav2_config/custom_nav_to_pose_bt.xml
```

`custom_nav_to_pose_bt.xml`의 최상위 `<RecoveryNode number_of_retries="6" name="NavigateRecovery">`를 `number_of_retries="2"`로 수정 — 실패 시 회복을 더 적게 시도하고 빨리 포기하도록 바꾼다.

### 3.3 params 파일에 연결

`nav2_params_smac.yaml`의 `bt_navigator.ros__parameters`에 추가:

```yaml
    default_nav_to_pose_bt_xml: "/home/pw/ros2_ws/nav2_config/custom_nav_to_pose_bt.xml"
```

### 3.4 재실행 후 도달 불가능한 목적지로 테스트

```bash
ros2 launch nav2_bringup tb3_simulation_launch.py map:=$HOME/ros2_ws/maps/tb3_sandbox_map.yaml params_file:=$HOME/ros2_ws/nav2_config/nav2_params_smac.yaml
```

초기 위치 재지정 후, 일부러 도달하기 어려운 지점(벽 너머 등)으로 "Nav2 Goal"을 찍어서 launch 터미널 로그로 회복 시도 횟수와 최종 실패까지의 흐름을 관찰한다.

## 4. 예상/실제 결과 확인

- planner가 `"no valid path found"`로 실패하면 `bt_navigator`가 `Goal failed`를 로그로 남기고 액션을 abort한다.
- `number_of_retries`를 6에서 2로 줄였으므로, 기본 BT보다 훨씬 적은 재시도 후 포기한다 — BT 구조 변경이 실제 동작 시간/끈기에 직접 반영됨을 확인.

## 5. 알려진 문제와 해결

이번 토픽에서는 이전 토픽들에서 다룬 conda 비활성화, 초기 pose 지정 절차를 그대로 따르면 되고, 별도로 새로 마주친 에러는 없었음.

## 6. 체크포인트

- [ ] 기본 `navigate_to_pose_w_replanning_and_recovery.xml` 구조를 읽고 각 노드의 역할 설명 가능
- [ ] 커스텀 BT XML 작성 (`number_of_retries` 변경)
- [ ] `default_nav_to_pose_bt_xml` 파라미터로 커스텀 BT 연결
- [ ] 도달 불가능한 목적지로 회복 시도 횟수 변화 확인

---
다음: [`06-action-client-mission.md`](./06-action-client-mission.md) (미작성)
