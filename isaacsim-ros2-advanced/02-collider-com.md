# 02. 충돌체(Collider) 및 무게중심(COM) 수정

## 1. 학습 목표

- 정적 오브젝트(거치대)와 동적 오브젝트(로봇 본체)에 각각 알맞은 Physics API를 적용하는 차이를 이해한다.
- Isaac Sim Property 패널의 다중 선택 편집으로, 수백 개 서브메시에 동일한 Collision 설정을 한 번에 적용하는 방법을 익힌다.
- Convex Hull과 Convex Decomposition의 차이, 그리고 작은 스케일 메쉬에서 Decomposition이 실패할 수 있다는 실전 함정을 안다.
- Rigid Body의 Mass를 명시적으로 지정하고, Play 모드로 실제 낙하/안착 거동을 검증한다.

## 2. 핵심 개념

**Static vs Dynamic**: USD Physics에서 `CollisionAPI`만 붙은 프림은 **정적(static)** 충돌체로 취급된다 — 즉, 씬 안에서 움직이지 않는 물체(바닥, 거치대 등)에 적합하다. 반면 `RigidBodyAPI`까지 함께 붙으면 PhysX가 그 프림을 **동적(dynamic)** 강체로 시뮬레이션한다. 로봇 본체처럼 자유롭게 움직여야 하는 오브젝트에만 Rigid Body를 추가하고, 거치대 같은 고정 오브젝트에는 Collision만 추가하는 게 원칙이다.

**Mass는 부모 Rigid Body 프림에 속한다**: 여러 서브메시로 쪼개진 오브젝트라도, PhysX 시뮬레이션 단위는 `RigidBodyAPI`가 붙은 프림 하나다. 그 밑의 서브메시들은 각자 Collision 형태(shape)만 제공할 뿐, 질량은 최상위 Rigid Body 프림에 딸린 `MassAPI`에서 하나의 값으로 관리된다.

**Convex Hull vs Convex Decomposition**: Convex Hull은 메쉬를 감싸는 하나의 볼록 껍질이라 계산이 빠르지만, 오목한 부분(로봇 밑면 흡입구 등)을 그대로 다 메꿔버린다. Convex Decomposition은 메쉬를 여러 볼록 조각으로 쪼개 오목한 형태를 살리지만 계산 비용이 크고, 극단적으로 작은 스케일의 메쉬에서는 voxelization 해상도 문제로 오히려 비정상적으로 부풀려진 헐이 나올 수 있다(3.4절 참고). 이 실습에서는 원본 모델이 이미 143개의 잘게 쪼개진 서브메시로 구성돼 있다는 점을 활용해, 서브메시 각각에 **Convex Hull**을 적용하는 방식으로 우회했다 — 조각 단위가 이미 작다 보니, 조각별 Hull들의 합이 전체적인 오목한 형태를 상당히 잘 근사한다.

**Isaac Sim의 다중 선택 편집**: Blender와 달리, Isaac Sim Property 패널은 여러 프림을 선택한 상태에서 값을 한 번 바꾸면 선택된 전체에 곧바로 적용된다(Blender의 `Copy to Selected` 같은 별도 조작이 필요 없음).

## 3. 실습 단계

### 3.1 거치대에 정적 Collider 추가

1. `ChargingDock` 프림 선택
2. 우클릭 → `Add > Physics > Collider Preset`
3. Property 패널에서 `Approximation`을 `Convex Hull`로 설정
4. 뷰포트 눈 아이콘 → `Show By Type > Colliders`로 시각화 확인

### 3.2 로봇 본체에 Rigid Body + Collider 추가

1. `robot1` 프림 선택
2. 우클릭 → `Add > Physics > Rigid Body with Colliders Preset` — `robot1`에 `RigidBodyAPI`가, 하위 모든 메시에 `CollisionAPI`가 일괄로 붙는다

### 3.3 서브메시 전체에 Convex Hull 일괄 적용

`Approximation`은 `robot1` 같은 Xform 프림이 아니라 실제 지오메트리를 가진 서브메시에만 나타난다.

1. Outliner에서 `robot1`을 펼쳐 하위 서브메시 트리 전체 노출
2. 첫 번째 서브메시 클릭 → 마지막 서브메시 Shift+클릭 (범위 선택, ~140개)
3. Property 패널에 나타난 `Approximation`을 `Convex Hull`로 설정 (한 번의 조작으로 선택된 전체에 적용됨)

### 3.4 Mass 설정

1. 선택 해제 후 `robot1` 단독 선택
2. Property 패널 하단 `Add` 버튼 → `Physics > Mass` — `robot1`에 `MassAPI`가 붙어야 `Mass`/`Center of Mass` 필드가 보인다 (자동 계산 상태에서는 필드 자체가 안 보임)
3. `Mass` 필드에 실제 로봇청소기 근사 무게(3.5~4kg) 입력

### 3.5 Play로 물리 거동 검증

1. `Create > Physics > Ground Plane`으로 바닥 추가
2. `robot1`을 바닥 위 살짝 뜬 위치로 이동
3. Play(▶) 클릭 → 낙하 후 안정적으로 착지하는지 관찰

## 4. 예상/실제 결과 확인

- Play 중 GPU/PhysX 에러 없이 시뮬레이션이 진행되어야 한다.
- 로봇이 바닥을 뚫지 않고, 옆으로 쓰러지지 않고 안정적으로 안착해야 한다.
- Collider 시각화가 실제 메쉬 실루엣과 비슷한 크기/형태여야 한다 (10배 이상 부풀려진 헐이 보이면 3.4/알려진 문제 참고).
- 거치대는 Rigid Body가 없으므로 시뮬레이션 중에도 제자리에서 움직이지 않아야 한다.

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| Play 시 `PxGpuDynamicsMemoryConfig::collisionStackSize buffer overflow` 에러, contact가 드롭됨 | 서브메시 140개에 걸쳐 collider 조각 수가 많아 GPU 충돌 계산 버퍼가 기본값으로 부족함 | Script Editor에서 `PhysxSchema.PhysxSceneAPI`를 스테이지의 모든 `UsdPhysics.Scene` 프림에 적용하고 `CreateGpuCollisionStackSizeAttr()`를 에러 메시지에 명시된 최소값보다 크게(예: 200,000,000) 설정 |
| Ground Plane 생성 후 `PhysicsScene` 프림이 두 개 존재 (`/World/PhysicsScene`와 `/PhysicsScene_<임의문자열>`) | Ground Plane 생성 시 기본 Physics Scene이 자동으로 같이 만들어지면서 기존 것과 중복된 것으로 추정 | 시뮬레이션 결과에 이상이 없으면 당장은 무시 가능. 위 GPU 버퍼 설정 스크립트는 스테이지의 모든 PhysicsScene에 일괄 적용되도록 작성해 두 씬 모두 커버함 |
| 서브메시들의 Approximation을 `Convex Decomposition`으로 설정하면(속성만 바꾸든, Collision을 지우고 새로 추가하든 동일하게 재현됨) collider 시각화가 실제 메쉬보다 10배 이상 커짐 | 작은 스케일(로봇 전체 ~0.35m)의 메쉬에서 PhysX의 SDF 기반 Convex Decomposition이 voxelization 해상도 문제로 비정상적으로 큰 헐을 생성하는 것으로 추정 | Decomposition 대신, 이미 143개로 잘게 쪼개진 서브메시 각각에 `Convex Hull`을 적용 — 조각 단위가 작아 개별 Hull들의 합이 전체 오목 형태를 충분히 근사함 |
| `robot1`(부모 Xform)만 선택한 상태에서 Property 패널에 `Approximation`이 안 보임 | Collision/Approximation 속성은 실제 지오메트리(Mesh)가 있는 프림에만 붙는 속성이라, Xform 컨테이너 자체에는 나타나지 않음 | 하위 서브메시들을 직접(범위) 선택해서 편집 |
| `robot1`만 선택한 상태에서 `Mass` 필드가 안 보임 | `MassAPI`가 아직 추가되지 않아 자동(밀도 기반) 계산 상태 — 명시적 필드가 없음 | Property 패널 `Add` 버튼 → `Physics > Mass`로 명시적으로 추가 |

## 6. 체크포인트

- [ ] `ChargingDock`에 정적 Collider 추가 (Rigid Body 없음, Convex Hull)
- [ ] `robot1`에 `Rigid Body with Colliders Preset` 적용
- [ ] 하위 서브메시 전체를 범위 선택해 `Approximation`을 `Convex Hull`로 일괄 설정 (Convex Decomposition은 소형 메쉬에서 부풀림 버그로 회피)
- [ ] `robot1`에 `MassAPI` 추가, 명시적 Mass 값(3.5~4kg) 입력
- [ ] GPU Collision Stack Size를 스테이지의 모든 PhysicsScene에 상향 설정
- [ ] Ground Plane 추가 후 Play로 낙하/안착 테스트, 에러 없이 안정적으로 착지 확인

---
다음: `03-brush-joint.md` (미작성) — 청소 가동부(브러쉬) 배치, Revolute Joint
