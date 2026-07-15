# 03. Planning Scene & 충돌 회피

## 1. 학습 목표

- Planning Scene에 충돌 오브젝트(박스)를 코드로 추가/제거할 수 있다.
- 환경 오브젝트가 실제로 플래닝 성공/실패에 영향을 준다는 것을 직접 확인한다.
- RViz MotionPlanning 패널의 "Scene Objects" 탭으로 씬 상태를 시각적으로 확인할 수 있다.

## 2. 핵심 개념

- `PlanningSceneMonitor`가 로봇 상태 + 환경 오브젝트를 함께 관리하는 "Planning Scene"을 유지한다. `panda.get_planning_scene_monitor()`로 얻는다.
- 씬을 수정할 때는 `with scene_monitor.read_write() as scene:` 컨텍스트로 락을 잡고, `scene.apply_collision_object(collision_object)`를 호출한다. `CollisionObject.operation`이 `ADD`면 추가, `REMOVE`면 제거.
- `CollisionObject`는 `moveit_msgs/msg/CollisionObject` 메시지로, `header.frame_id`(기준 좌표계), `primitives`(`shape_msgs/SolidPrimitive` — 박스/구/실린더 등), `primitive_poses`(그 도형의 위치/자세)로 구성된다.
- 플래닝 시 충돌 체크는 로봇 자기 자신(self-collision)뿐 아니라 이런 환경 오브젝트도 함께 고려한다 — 목표 자세가 오브젝트와 겹치면 플래닝 자체가 실패한다.

## 3. 실습 단계

이전 토픽의 `mission_arm.py`와 동일한 `MoveItConfigsBuilder`/`MoveItPy`/`PlanRequestParameters` 설정(토픽 2에서 정리한 우회 4가지 모두 포함)을 재사용해서 `planning_scene_demo.py`를 작성했다.

1. 장애물 없이 `"ready"` → `"extended"` 플래닝·실행 (기준선 확인)
2. `panda_link0` 기준 `(x=0.4, y=0.0, z=0.4)` 위치에 한 변 0.25m 박스(`obstacle_box`)를 씬에 추가
3. 같은 `"ready"` 목표로 다시 플래닝 시도
4. 박스를 씬에서 제거
5. `"extended"` 목표로 다시 플래닝·실행해 정상 동작 확인

```bash
cd ~/ros2_ws
colcon build --packages-select moveit2_advanced
source install/setup.bash
ros2 run moveit2_advanced planning_scene_demo
```

RViz MotionPlanning 패널의 "Scene Objects" 탭을 열어두면 박스가 추가/제거되는 순간을 시각적으로 확인할 수 있다.

## 4. 예상/실제 결과 확인

- 박스 추가 전: `"ready"`/`"extended"` 둘 다 플래닝 성공 및 실제 이동.
- 박스 추가 후 `"ready"` 재시도: 우연히 박스 위치가 `"ready"` 자세의 손 위치와 겹쳐서, 목표 상태 자체가 충돌 상태가 됨 → 로그에 `Found a contact between 'obstacle_box' (type 'Object') and 'panda_hand'`, 이어서 `Unable to sample any valid states for goal tree`로 플래닝 실패.
- 박스 제거 후 `"extended"`: 다시 정상적으로 플래닝·실행됨.
- RViz "Scene Objects" 탭에서 작은 박스가 실제로 나타났다가 사라지는 것을 육안으로 확인.

## 5. 알려진 문제와 해결

이번 토픽은 토픽 2에서 이미 겪은 `moveit_py` 관련 문제들(tuple→list 변환, `planning_pipelines.pipeline_names`, `PlanRequestParameters` 명시적 설정, `time.sleep`)을 그대로 재사용해서 새로 마주친 문제는 없었음.

## 6. 체크포인트

- [ ] `PlanningSceneMonitor.read_write()` 컨텍스트로 `CollisionObject` 추가
- [ ] 오브젝트가 목표 자세와 충돌할 때 플래닝이 실패하는 것을 로그로 확인
- [ ] `CollisionObject.REMOVE`로 오브젝트 제거 후 플래닝 재성공 확인
- [ ] RViz "Scene Objects" 탭에서 오브젝트 추가/제거를 시각적으로 확인

---
MoveIt2 Advanced 트랙(토픽 1~3) 완료. 다음은 Part 3 — Nav2 + MoveIt2 통합 캡스톤(미착수).
