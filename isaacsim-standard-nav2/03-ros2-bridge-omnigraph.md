# 03. ROS2 브릿지 & OmniGraph 연결

## 1. 학습 목표

- `isaacsim.ros2.bridge`의 `ROS2 Subscribe Twist`로 `/cmd_vel`을 구독하고, `Differential Controller` + `Articulation Controller` 조합으로 실제 조인트를 구동하는 표준 패턴을 익힌다.
- Twist 메시지의 `vector3` 필드에서 필요한 축 성분만 뽑아 스칼라 입력에 연결하는 방법(`Break Vector3`)을 익힌다.
- OmniGraph 노드의 Property 패널 값 표시를 맹신하지 않고, `og.Controller`/`omni.graph.core`로 실제 저장된 속성값을 직접 검증하는 디버깅 습관을 기른다.

## 2. 핵심 개념

**Carter가 vacuum 프로젝트보다 훨씬 단순한 이유**: `isaacsim-ros2-advanced` 트랙의 로봇청소기는 실제 바퀴 조인트가 없어서(Topic 6, [[isaacsim_ros2_advanced_curriculum]]) `physics:velocity`/`physics:angularVelocity`를 로봇 바디에 매 프레임 직접 write하는 우회 방식을 써야 했다. Carter는 `chassis_link`↔`left_wheel_link`/`right_wheel_link` 사이에 진짜 `RevoluteJoint`+`DriveAPI`가 있어서(Topic 2), Isaac Sim이 정확히 이 상황을 위해 제공하는 `Differential Controller`(Twist → 좌/우 바퀴 각속도 변환) + `Articulation Controller`(조인트 이름으로 실제 드라이브에 값 write) 조합을 그대로 쓸 수 있다. 이게 훨씬 "정석"에 가까운 워크플로우다.

**타입 불일치와 Break Vector3**: `ROS2 Subscribe Twist`의 `Angular Velocity`/`Linear Velocity` 출력은 Twist 메시지의 `angular`/`linear` 필드 전체(`double3` 벡터)다. 반면 `Differential Controller`의 `Desired Linear/Angular Velocity` 입력은 스칼라 `double`이다. 이 스테이지는 **Z-up**이므로(vacuum 프로젝트의 Y-up과 다름, 3.x절 참고) `angular.z`(요 회전율), `linear.x`(전진 속도) 성분만 있으면 되고, `Break Vector3` 노드로 벡터를 x/y/z로 쪼갠 뒤 필요한 성분만 연결한다.

**Exec 핀이 없는 순수 데이터 노드**: `Differential Controller`는 `Exec In`/`Exec Out`이 없는 순수 데이터 변환 노드다. Exec 체인은 이 노드를 건너뛰어 `ROS2 Subscribe Twist`의 `Exec Out`에서 바로 `Articulation Controller`의 `Exec In`으로 이어야 한다. `Differential Controller`는 exec 체인과 무관하게 매 평가마다 자기 입력 연결로부터 값을 계산해 데이터 핀으로 흘려보낸다.

**조인트 이름 vs 링크 이름**: `Articulation Controller`의 `Joint Name` 배열에는 링크 이름(`left_wheel_link`)이 아니라 **조인트 prim 이름**(`left_wheel`, `right_wheel`)을 넣어야 한다. Topic 2에서 만든 물리 구조 덤프(`carter_physics_dump.txt`)가 정확한 조인트 이름의 출처다.

## 3. 실습 단계

### 3.1 그래프 골격 구성

1. `Window` → `Visual Scripting` → `Action Graph`에서 새 그래프 생성 (`/World/CarterGraph`)
2. `On Playback Tick` 추가
3. `ROS2 Subscribe Twist` 추가, `On Playback Tick.outputs:tick` → `ROS2 Subscribe Twist.inputs:execIn` 연결. Topic Name은 기본값 `cmd_vel` 확인.

### 3.2 Twist → 바퀴 각속도 변환

1. `Differential Controller` 추가
2. `Break Vector3` 두 개 추가
3. `ROS2 Subscribe Twist.outputs:angularVelocity` → Break Vector3 #1 입력 → **z** 출력 → `Differential Controller.inputs:angularVelocity`
4. `ROS2 Subscribe Twist.outputs:linearVelocity` → Break Vector3 #2 입력 → **x** 출력 → `Differential Controller.inputs:linearVelocity`
5. `On Playback Tick`의 `Delta Seconds` 출력 → `Differential Controller.inputs:dt`
6. `Differential Controller`의 `Wheel Radius`/`Wheel Distance`와 `Max Acceleration`/`Max Angular Acceleration`/`Max Linear Speed`/`Max Wheel Speed`/`Max Angular Speed`/`Max Deceleration`을 모두 명시적으로 채운다 (5.1절 참고 — 이 노드는 미입력 시 기본값이 전부 `0`이라 값을 안 넣으면 조용히 안 움직인다).

Wheel Radius/Wheel Distance 실측값(0.24m / 0.6284m)은 GUI에서 손으로 스케일×회전을 계산하지 않고, `UsdGeom.Cylinder`의 `radius`/`axis` 속성과 `ComputeLocalToWorldTransform`으로 로컬 원 위의 점들을 월드로 변환해 중심으로부터의 거리로 직접 측정했다(스크립트: `measure_carter_wheel_radius_2026-07-25.py`). 4개 샘플 포인트 모두 정확히 0.24000으로 일치해 신뢰도를 확보했다.

### 3.3 바퀴 각속도 → 조인트 드라이브

1. `Articulation Controller` 추가
2. `ROS2 Subscribe Twist.outputs:execOut` → `Articulation Controller.inputs:execIn` (Differential Controller를 건너뜀, 2절 참고)
3. `Differential Controller.outputs:velocityCommand` → `Articulation Controller.inputs:velocityCommand`
4. `Articulation Controller.inputs:targetPrim` = `/World/carter_v1`
5. `Articulation Controller.inputs:jointNames` = `["left_wheel", "right_wheel"]` (순서 중요 — Differential Controller 출력이 [left, right] 순서)

### 3.4 검증

Play 후, 별도 터미널(conda 비활성화)에서 `/cmd_vel`을 퍼블리시해 확인한다:

```bash
source ~/anaconda3/etc/profile.d/conda.sh; conda deactivate; conda deactivate; conda deactivate; hash -r
ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.2}, angular: {z: 0.0}}"
```

## 4. 예상/실제 결과 확인

- `linear.x=0.2`로 퍼블리시하면 로봇이 앞으로 이동해야 한다.
- `angular.z=0.5`로 퍼블리시하면 제자리(또는 원호)로 회전해야 한다.
- `ros2 topic info /cmd_vel --verbose`에서 `_World_CarterGraph_ros2_subscribe_twist` 노드가 Subscription으로 보여야 한다.

둘 다 최종적으로 확인됨.

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| `outputs:angularVelocity of type double3 (vector) is not compatible with inputs:angularVelocity of type double` 경고 | `ROS2 Subscribe Twist`는 Twist의 `angular`/`linear` 필드 전체(vector3)를 출력하는데, `Differential Controller`는 스칼라를 받음 | `Break Vector3`로 벡터를 분해해 필요한 성분(이 스테이지는 Z-up이므로 각속도는 `z`, 선속도는 `x`)만 연결 |
| `Differential Controller`에 `Exec Out` 핀 자체가 없어서 `Articulation Controller`로 exec 체인을 이어갈 수 없음 | 이 노드는 exec 없는 순수 데이터 변환 노드로 설계됨 | `ROS2 Subscribe Twist`의 `Exec Out`에서 `Differential Controller`를 건너뛰고 바로 `Articulation Controller`의 `Exec In`으로 연결 |
| `Max Acceleration`/`Max Angular Acceleration`/`Max Linear Speed`/`Max Wheel Speed`/`Max Deceleration`을 기본값(전부 `0`)으로 두면, `Velocity Command` 출력이 목표값(~0.83rad/s)이 아니라 거의 0에 가까운 극소값에 고정되고 시간이 지나도 램프업되지 않음 | 이 노드에서 `0`은 "무제한"이 아니라 문자 그대로 "가/감속·속도 상한 0"으로 해석돼, 사실상 아무 속도 변화도 허용하지 않음 | 모든 Max 계열 필드에 여유 있는 명시적 값(가속도 5.0, 속도 상한 2.0~10.0 등)을 입력 |
| Property 패널에 `Wheel Radius`를 `0.24`로 입력했다고 표시됐는데, `Velocity Command`가 계속 `linearVelocity / 324`에 해당하는 극소값(`0.000617`)으로 나옴 | GUI 숫자 입력 필드가 실제로는 `324.0`을 저장하고 있었음(오타 또는 입력 UI의 알 수 없는 변환) — Property 패널에 보이는 값과 실제 저장된 USD 속성값이 달랐던 사례 | `og.Controller.get/set`으로 해당 attribute path(`/World/CarterGraph/differential_controller.inputs:wheelRadius`)를 직접 읽고 써서 확정 — GUI 표시값을 믿지 말고 스크립트로 재검증 |
| Wheel Radius를 스케일이 걸린 Cylinder에서 "로컬 radius × Scale" 암산으로 구하려다 축(axis)·회전 조합을 헷갈릴 뻔함 | Cylinder에 `axis=Z`, `scale=(0.48, 0.48, 0.09)`처럼 비균등 스케일이 걸려 있어 손으로 계산하면 축을 잘못 짚기 쉬움 | `UsdGeom.Cylinder`의 `radius`/`axis` 속성을 읽고, 로컬 원 둘레의 점들을 `ComputeLocalToWorldTransform`으로 월드 변환해 중심으로부터의 실제 거리를 스크립트로 직접 측정(vacuum 프로젝트의 "블라인드 좌표 암산 대신 실측/시각 확인" 교훈과 동일한 패턴) |

## 6. 체크포인트

- [ ] `/World/CarterGraph` Action Graph 생성
- [ ] `On Playback Tick` → `ROS2 Subscribe Twist` → `Break Vector3`×2 → `Differential Controller` 데이터 흐름 구성
- [ ] `Differential Controller`의 Wheel Radius(0.24)/Wheel Distance(0.6284)와 모든 Max 필드를 명시적으로 채움
- [ ] `ROS2 Subscribe Twist.execOut` → `Articulation Controller.execIn` 직결
- [ ] `Articulation Controller`의 `targetPrim`(`/World/carter_v1`)과 `jointNames`(`["left_wheel","right_wheel"]`) 설정
- [ ] `/cmd_vel` linear.x, angular.z 각각 퍼블리시해 실제 이동/회전 확인

---
다음: Odometry/TF/Clock 퍼블리시, LiDAR 퍼블리시 (진행 중)
