# 05. 특수 센서 배치 (Bumper & 추락방지)

> 진행 상태: **범퍼(Contact Sensor) 완료, 추락방지(Cliff) 센서는 다음 세션에서 이어감**

## 1. 학습 목표

- Contact Report API 기반의 Contact Sensor로 로봇의 충돌(범퍼) 이벤트를 감지한다.
- PhysX Raycast로 근접/거리 기반 센서(추락방지)를 구현하는 방법을 익힌다.
- Isaac Sim의 "Generic Range Sensor" 같은 레거시 기능과, 현재도 잘 지원되는 PhysX Scene Query 기반 대안을 구분한다.

## 2. 핵심 개념

**Contact Sensor의 실제 동작 원리**: `Create > Isaac > Contact Sensor`로 센서를 만들면, 그 시점에 **선택되어 있던 프림**에 자동으로 `PhysxContactReportAPI`가 붙는다. 센서 자체를 감싸는 빈 Xform(마운트 포인트)을 선택한 채로 만들면 그 빈 Xform에 API가 붙어버려서 아무 효과가 없다 — 실제 충돌 형상을 가진 Rigid Body 프림에 API가 붙어야 한다. 필요하면 나중에 직접 `PhysxSchema.PhysxContactReportAPI.Apply(rigid_body_prim)`로 재적용해야 한다.

**Contact Sensor의 `Radius`**: `-1`이면 강체 전체의 모든 접촉을 다 감지한다. 로봇처럼 몸체가 항상 바닥에 닿아있는 경우, 범퍼 위치 근처의 접촉만 골라내려면 `Radius`를 작은 양수(예: 0.05)로 제한해야 한다.

**Generic Range Sensor는 이 버전에서 신뢰할 수 없다**: `RangeSensorCreateGeneric` 커맨드로 만든 센서에서 `_range_sensor.acquire_generic_sensor_interface().get_linear_depth_data()`로 값을 읽으면 초기화되지 않은 듯한 쓰레기 값(예: `2.28e+37`, 비정상적으로 작은 subnormal float들)이 나왔다 — 이 레거시 API는 이 Isaac Sim 버전에서 제대로 지원되지 않는 것으로 보인다.

**PhysX Raycast가 더 신뢰할 수 있는 대안**: `omni.physx.get_physx_scene_query_interface().raycast_closest(origin, direction, max_distance)`(Python) 또는 OmniGraph의 **`Raycast, Closest`**(`omni.physx.graph.SceneQueryRaycastClosest`) 노드로 원점/방향을 주면 `hit`, `distance`, `collision`(맞은 프림) 등을 바로 준다 — 표준적이고 잘 문서화된 방식이다.

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

### 3.2 추락방지 센서 시도 (진행 중 — 다음 세션에서 계속)

1. `Generic Range Sensor`(`RangeSensorCreateGeneric`)로 시도했으나 3.2절 문제로 폐기
2. PhysX Raycast로 전환:
   - `cliff_mount` Xform을 로봇 밑면 앞쪽에 생성, 광선이 바닥을 향하도록 회전
   - Python으로 원점/방향을 계산해 `raycast_closest` 테스트
3. **막힌 지점**: 원점을 여러 방향/거리로 옮겨봐도 계속 로봇 자기 자신의 콜라이더(`Object_139` — 로봇 전체 바운딩박스와 거의 같은 크기, 35.7×6.8×35.8cm)에 거리 0으로 맞음. 원점을 10cm씩 밀어도 동일하게 재현됨.
4. 동시에 발견한 문제: **Play 시 로봇 일부(전면부)가 바닥에 살짝 파묻힘** — Convex Hull 콜라이더가 시각 메쉬와 완벽히 일치하지 않아 안착 자세가 미세하게 기운 것으로 추정. 이게 "아래" 방향 계산을 계속 어긋나게 한 원인일 가능성이 있음.

## 4. 예상/실제 결과 확인

- 범퍼: 로봇이 장애물에 부딪히면 `Isaac Read Contact Sensor`의 출력값이 변함 — **확인 완료**.
- 추락방지: (다음 세션 목표) 로봇 밑면 바깥에서 시작한 광선이 바닥까지의 정상적인 거리(수 cm)를 반환해야 함. 로봇이 테이블 가장자리 등 낭떠러지에 도달하면 거리가 급격히 커지거나 `hit=False`가 되어야 함.

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| Contact Sensor를 만들고 로봇을 부딪혀도 값이 안 바뀜 | Contact Sensor 생성 시 `PhysxContactReportAPI`가 센서를 담은 빈 Xform(`bumper_mount`)에 붙었음 — 실제 콜라이더가 없는 프림이라 무효 | `PhysxSchema.PhysxContactReportAPI.Apply(robot1)`로 실제 Rigid Body 프림에 직접 재적용 |
| Cube 장애물을 만들면 로봇보다 훨씬 큼 | Isaac Sim의 기본 Cube 프리미티브가 **100m** 크기로 생성됨(다른 DCC 툴의 "1 unit" 관례와 다름) | Scale을 원하는 실제 크기(m) ÷ 100으로 계산 (예: 15cm 원하면 Scale `0.0015`) |
| `_range_sensor.acquire_generic_sensor_interface().get_linear_depth_data()`가 쓰레기 값을 반환 | Generic Range Sensor가 이 Isaac Sim 버전에서 제대로 지원되지 않는 레거시 기능으로 추정 | PhysX Raycast(`get_physx_scene_query_interface().raycast_closest()` 또는 OmniGraph `Raycast, Closest` 노드)로 대체 |
| Raycast 원점을 반복적으로 밀어도 계속 로봇 자기 자신(`Object_139`)에 거리 0으로 맞음 | 원인 미확정 — 유력 후보: (1) 로컬→월드 축 변환 가정이 틀림(이 스테이지의 up-axis를 확실히 검증한 적 없음), (2) 로봇이 안착 시 살짝 기울어져 있어 "아래" 계산 기준이 실제와 다름, (3) `Object_139`의 Convex Hull이 로봇 전체를 감싸는 큰 형상이라 예상보다 넓은 영역에서 계속 걸림 | **미해결.** 다음 세션에서 좌표 계산 대신 뷰포트에서 디버그 광선을 직접 보며 GUI로 위치 조정 예정. 로봇의 안착 자세(파묻힘 여부)도 먼저 점검 필요 |

## 6. 체크포인트

- [x] `bumper_mount` + Contact Sensor 생성, `Radius` 조정
- [x] `PhysxContactReportAPI`를 실제 Rigid Body(`robot1`)에 적용
- [x] 테스트용 장애물(Cube, 올바른 크기로 스케일)로 접촉 확인
- [x] Action Graph로 Contact Sensor 값 읽기 확인
- [ ] `cliff_mount` + Raycast 기반 추락방지 센서 — 원점이 로봇 콜라이더를 벗어나 바닥을 정상적으로 맞히는지 확인 (미완료)
- [ ] 로봇 안착 자세(파묻힘) 원인 점검 및 필요시 Collider 재조정
- [ ] Compare 노드로 "낭떠러지 감지" 불리언 로직 구성 (미착수)

---
다음: 추락방지 센서 이어서 진행 (같은 파일에 이어 작성 예정), 이후 Topic 6 — 마스터 액션그래프 & ROS2 브릿지 활성화
