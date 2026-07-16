# 05. 특수 센서 배치 (Bumper & 추락방지)

> 진행 상태: **완료** (범퍼, 추락방지 센서 둘 다 확인)

## 1. 학습 목표

- Contact Report API 기반의 Contact Sensor로 로봇의 충돌(범퍼) 이벤트를 감지한다.
- PhysX Raycast로 근접/거리 기반 센서(추락방지)를 구현하는 방법을 익힌다.
- Isaac Sim의 "Generic Range Sensor" 같은 레거시 기능과, 현재도 잘 지원되는 PhysX Scene Query 기반 대안을 구분한다.

## 2. 핵심 개념

**Contact Sensor의 실제 동작 원리**: `Create > Isaac > Contact Sensor`로 센서를 만들면, 그 시점에 **선택되어 있던 프림**에 자동으로 `PhysxContactReportAPI`가 붙는다. 센서 자체를 감싸는 빈 Xform(마운트 포인트)을 선택한 채로 만들면 그 빈 Xform에 API가 붙어버려서 아무 효과가 없다 — 실제 충돌 형상을 가진 Rigid Body 프림에 API가 붙어야 한다. 필요하면 나중에 직접 `PhysxSchema.PhysxContactReportAPI.Apply(rigid_body_prim)`로 재적용해야 한다.

**Contact Sensor의 `Radius`**: `-1`이면 강체 전체의 모든 접촉을 다 감지한다. 로봇처럼 몸체가 항상 바닥에 닿아있는 경우, 범퍼 위치 근처의 접촉만 골라내려면 `Radius`를 작은 양수(예: 0.05)로 제한해야 한다.

**Generic Range Sensor는 이 버전에서 신뢰할 수 없다**: `RangeSensorCreateGeneric` 커맨드로 만든 센서에서 `_range_sensor.acquire_generic_sensor_interface().get_linear_depth_data()`로 값을 읽으면 초기화되지 않은 듯한 쓰레기 값(예: `2.28e+37`, 비정상적으로 작은 subnormal float들)이 나왔다 — 이 레거시 API는 이 Isaac Sim 버전에서 제대로 지원되지 않는 것으로 보인다.

**PhysX Raycast가 더 신뢰할 수 있는 대안**: `omni.physx.get_physx_scene_query_interface().raycast_closest(origin, direction, max_distance)`(Python) 또는 OmniGraph의 **`Raycast, Closest`**(`omni.physx.graph.SceneQueryRaycastClosest`) 노드로 원점/방향을 주면 `hit`, `distance`, `collision`(맞은 프림) 등을 바로 준다 — 표준적이고 잘 문서화된 방식이다.

**이 스테이지의 up-axis는 Y다**: 이번 토픽 전까지 계속 Z축을 수직/위아래로 암묵적으로 가정해왔는데, 실제로는 `GroundPlane`의 `axis` 속성이 `Y`이고, 로봇의 world bbox에서도 Y 범위(약 9cm)가 정확히 로봇 높이와 일치한다. 앞으로 이 스테이지에서 방향 관련 계산을 할 땐 Y를 수직축으로 놓고 시작할 것.

## 3. 실습 단계

### 3.1 범퍼 Contact Sensor (완료)

1. `robot1` 하위에 `bumper_mount` Xform 생성, 로봇 정면 가장자리에 배치
2. `bumper_mount` 선택 상태에서 `Create > Isaac > Contact Sensor` 생성
3. `Radius`를 `0.05`로 설정 (전체 강체 감지 방지)
4. **중요**: 자동으로 붙은 ContactReportAPI가 `bumper_mount`(빈 Xform)에 붙어서 무효였음 — 아래 스크립트로 실제 Rigid Body(`robot1`)에 재적용

```python
import omni.usd
from pxr import PhysxSchema

stage = omni.usd.get_context().get_stage()
robot1 = stage.GetPrimAtPath("/World/robotvacum_decimated/robot1")

contact_report = PhysxSchema.PhysxContactReportAPI.Apply(robot1)
contact_report.CreateThresholdAttr(0.0)
```

5. 테스트용 장애물(작은 Cube, Scale 조정 시 기본 100m 크기이므로 `0.0015` 배율로 15cm급으로 축소, 정적 Collider만 추가) 배치
6. Action Graph에 `Isaac Read Contact Sensor` 노드 추가, `CS Prim`에 Contact Sensor 연결
7. Play 후 로봇을 큐브 쪽으로 밀어서 접촉 시 값이 바뀌는지 확인 — **확인 완료**

### 3.2 추락방지 센서 — 축 혼동 디버깅

1. `Generic Range Sensor`(`RangeSensorCreateGeneric`)로 시도했으나 쓰레기 값만 나와 폐기 (2절 참고)
2. PhysX Raycast로 전환, `cliff_mount` Xform을 로봇 밑면 앞쪽에 생성
3. **막힌 지점 1**: 원점을 반복적으로 밀어도 계속 로봇 자기 자신의 콜라이더(`Object_139`)에 거리 0으로 맞음. 원인은 이 스테이지의 **up-axis가 Y**라는 걸 검증 안 하고 계속 Z축을 "위/아래"로 가정해서 계산했기 때문 — 실제로는 X/Z 평면과 거의 평행하게 밀고 있었던 것. `robot1`의 world bbox(Y 범위가 로봇 높이 ~9cm와 정확히 일치)와 `GroundPlane`의 `axis` 속성(`Y`)으로 확인.
4. **막힌 지점 2**: up-axis를 Y로 고치고 원점을 로봇 발밑(Y 근처)에 둬도 여전히 `Object_139`에 거리 0으로 맞음 — `Object_139`가 로봇의 곡면 외피(shell) 조각인데, **Convex Hull로 감싸면서 곡면 안쪽 빈 공간까지 통짜로 채워져** 로봇 실루엣 안 어디를 찍어도(높이 상관없이) "내부"로 잡히는 것으로 확인됨 (Topic 2에서 이미 파악한 Convex Hull의 오목한 형태 손실 문제가 극단적으로 나타난 경우).
5. **해결**: 원점을 로봇 자신의 XZ 풋프린트 **바깥**(가장자리 너머)으로 빼고, 방향은 로컬 축 변환 대신 월드 좌표로 직접 `(0, -1, 0)` 고정:

```python
from omni.physx import get_physx_scene_query_interface
from pxr import UsdGeom

bbox_cache = UsdGeom.BBoxCache(0, [UsdGeom.Tokens.default_])
box = bbox_cache.ComputeWorldBound(robot1_prim).ComputeAlignedBox()
center = box.GetMidpoint()
max_z = box.GetMax()[2]

origin = (center[0], 0.02, max_z + 0.02)  # 로봇 가장자리 밖, 바닥 위 2cm
direction = (0, -1, 0)
hit_info = get_physx_scene_query_interface().raycast_closest(origin, direction, 0.5)
# -> collision: '/World/GroundPlane/CollisionPlane', distance: 0.02  (정상)
```

`cliff_mount`를 이 검증된 위치(로봇 중심의 X/Z, 로봇 가장자리보다 살짝 밖, 바닥 위 2cm)로 이동.

### 3.3 Action Graph로 배선 (완료)

노드: `On Playback Tick` → `Get Prim Local to World Transform`(Prim Path = `cliff_mount`) → `Get Translation` → `Raycast, Closest`(`Origin`에 연결, `Direction`은 `Construct Vector3`로 만든 상수 `(0,-1,0)`, `Distance`는 `Construct Double`로 만든 상수 `0.5`) → `Compare`(`A` = Raycast의 `hit`, `B` = 상수 `false`, `Operation` = `==`) → `Result`가 절벽 감지 불리언.

**막힌 지점 3**: `Direction`/`Distance` 입력 필드가 Property 패널에서 "unresolved union" 상태라 직접 값을 못 넣음 — `vectorf[3]/vectord[3]`, `float/double`처럼 여러 타입을 허용하는 입력은 연결이 있어야 타입이 확정됨. `Construct Vector3`/`Construct Double` 같은 상수 노드를 만들어서 연결해야 값을 넣을 수 있었음.

### 3.4 검증 (완료)

Play 도중 `robot1`을 매니퓰레이터로 직접 끌어서 테스트하면 Joint로 연결된 `Cylinder`(브러쉬)가 급격한 위치 변화에 물리적으로 반응해 떨림 — 정상적인 현상(Joint가 갑작스런 원격이동에 저항하는 것)이므로 Stop 상태에서 옮기고 다시 Play하는 방식으로 우회.

- 로봇을 바닥 근처에 둔 상태: `hit=true` → `Result=false`
- 로봇을 1m 이상 들어올린 상태(사거리 0.5m 초과, 무한 GroundPlane이라 실제 낭떠러지 대신 이렇게 시뮬레이션): `hit=false` → `Result=true`

주의: **1cm 정도만 들어올리면 사거리(0.5m) 안에 바닥이 여전히 있어서 `hit`이 안 바뀐다** — 테스트 시 확실히 사거리보다 크게 들어올려야 함.

## 4. 예상/실제 결과 확인

- 범퍼: 로봇이 장애물에 부딪히면 `Isaac Read Contact Sensor`의 출력값이 변함 — **확인 완료**.
- 추락방지: 정상 바닥 위에서는 `Compare.Result = false`, 사거리를 초과해 바닥이 없으면 `Result = true` — **확인 완료**.

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| Contact Sensor를 만들고 로봇을 부딪혀도 값이 안 바뀜 | Contact Sensor 생성 시 `PhysxContactReportAPI`가 센서를 담은 빈 Xform(`bumper_mount`)에 붙었음 — 실제 콜라이더가 없는 프림이라 무효 | `PhysxSchema.PhysxContactReportAPI.Apply(robot1)`로 실제 Rigid Body 프림에 직접 재적용 |
| Cube 장애물을 만들면 로봇보다 훨씬 큼 | Isaac Sim의 기본 Cube 프리미티브가 **100m** 크기로 생성됨(다른 DCC 툴의 "1 unit" 관례와 다름) | Scale을 원하는 실제 크기(m) ÷ 100으로 계산 (예: 15cm 원하면 Scale `0.0015`) |
| `_range_sensor.acquire_generic_sensor_interface().get_linear_depth_data()`가 쓰레기 값을 반환 | Generic Range Sensor가 이 Isaac Sim 버전에서 제대로 지원되지 않는 레거시 기능으로 추정 | PhysX Raycast(`get_physx_scene_query_interface().raycast_closest()` 또는 OmniGraph `Raycast, Closest` 노드)로 대체 |
| Raycast 원점을 반복적으로 밀어도 계속 로봇 자기 자신(`Object_139`)에 거리 0으로 맞음 (1차) | 이 스테이지의 up-axis가 Y인데 계속 Z축을 "아래"로 가정해서 계산 — 실제로는 수평 방향으로 밀고 있었음 | `robot1`의 world bbox나 `GroundPlane`의 `axis` 속성으로 up-axis를 먼저 검증. 이후 방향은 로컬 회전 변환 대신 월드 좌표 `(0,-1,0)`을 직접 사용 |
| up-axis를 고치고 발밑에 원점을 둬도 여전히 `Object_139`에 거리 0으로 맞음 (2차) | `Object_139`가 곡면 외피 조각인데 Convex Hull이 곡면 안쪽까지 통짜로 채워서, 로봇 실루엣 안이면 높이 상관없이 "내부"로 잡힘 | 원점을 로봇 자신의 XZ 풋프린트 바깥(가장자리 너머)으로 이동 |
| `Raycast, Closest`의 `Direction`/`Distance` 입력이 Property 패널에서 값 입력이 안 되고 "unresolved union"이라고 뜸 | `vectorf[3]/vectord[3]`, `float/double`처럼 여러 타입을 허용하는 입력(union)은 연결이 있어야 타입이 확정되어 값 편집이 가능해짐 | `Construct Vector3`/`Construct Double` 같은 상수 노드를 만들어 연결 |
| Play 중 `robot1`을 매니퓰레이터로 직접 끌면 Joint로 연결된 `Cylinder`(브러쉬)가 격렬하게 떨림 | Joint가 붙은 두 Rigid Body 중 하나를 시뮬레이션 도중 순간이동시키면 PhysX 제약 솔버가 급격한 위치 차이를 무리하게 보정하려 함 — 정상 동작 | Stop 상태에서 위치를 옮긴 뒤 다시 Play. (일반적으로도 Play 중 강체를 직접 끄는 테스트는 되도록 피하는 게 좋음) |
| 로봇을 1cm만 들어올려도 `hit`이 안 바뀜 | Raycast `Distance`(사거리)를 0.5m로 설정했는데, 1cm는 그 범위 안이라 바닥이 여전히 정상적으로 잡힘 | 사거리보다 확실히 크게(예: 1m) 들어올려서 테스트 |

## 6. 체크포인트

- [x] `bumper_mount` + Contact Sensor 생성, `Radius` 조정
- [x] `PhysxContactReportAPI`를 실제 Rigid Body(`robot1`)에 적용
- [x] 테스트용 장애물(Cube, 올바른 크기로 스케일)로 접촉 확인
- [x] Action Graph로 Contact Sensor 값 읽기 확인
- [x] 스테이지 up-axis(Y) 검증, `cliff_mount`를 로봇 풋프린트 바깥의 검증된 위치로 이동
- [x] Action Graph: `Get Prim Local to World Transform` → `Get Translation` → `Raycast, Closest` → `Compare` 배선
- [x] 정상 바닥 위 `Result=false`, 사거리 초과(낭떠러지 시뮬레이션) `Result=true` 확인 완료

---
Part B Topic 5 완료. 다음: Topic 6 — 마스터 액션그래프 & ROS2 브릿지 활성화 (TF/Odometry Publisher, ROS2 Subscribe Twist, 헤드리스 실행)
