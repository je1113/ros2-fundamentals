# 03. 청소 가동부(브러쉬) 배치 — Revolute Joint

## 1. 학습 목표

- 소스 3D 에셋에 필요한 부품이 없을 때, Primitive로 대체해 물리 학습 목표는 그대로 달성하는 실전 판단을 익힌다.
- 두 개의 독립된 Rigid Body를 USD Physics Joint(Revolute)로 연결하고, 회전축을 올바르게 설정한다.
- Joint에 Angular Drive(속도 기반 구동)를 추가해 지속 회전을 구현한다.
- Rigid Body Material로 마찰 계수를 지정하고 바닥과의 상호작용을 조정한다.
- USD 프림 reparent 시 월드 트랜스폼이 깨질 수 있다는 함정과 대응법을 익힌다.

## 2. 핵심 개념

**Joint는 계층이 아니라 관계다**: 두 Rigid Body를 물리적으로 연결하는 것은 USD Stage의 부모-자식 Xform 계층이 아니라 별도의 Joint 프림(`physics:body0`/`physics:body1` relationship)이다. Joint로 연결되는 두 Body는 Stage 트리 어디에 있든 상관없다 — 오히려 서로 다른 Rigid Body를 부모-자식으로 중첩시키면 PhysX가 매 프레임 월드/로컬 변환을 이중으로 계산해야 해서 문제가 생길 수 있다 (형제 관계 권장).

**Reparent와 월드 트랜스폼**: Outliner에서 드래그로 프림을 다른 부모 밑으로 옮기면, 로컬 `xformOp` 값은 그대로 유지된 채 부모만 바뀐다. 새 부모의 월드 트랜스폼이 이전 부모와 다르면, 프림의 실제 월드 위치가 옮긴 만큼 어긋난다(로컬 오프셋이 새 부모 기준으로 재해석되기 때문). 이동 전후로 반드시 월드 위치를 확인해야 한다.

**Velocity Drive vs Position Drive**: `UsdPhysics.DriveAPI`는 `stiffness`(위치 추종 강도)와 `damping`(속도 추종 강도) 두 값으로 동작 모드가 결정된다 — `stiffness=0`, `damping>0`으로 두면 순수 속도 제어(Velocity Drive)가 되어 `targetVelocity`를 향해 지속적으로 회전력을 가한다. 브러쉬처럼 계속 도는 부품에 적합하다.

## 3. 실습 단계

### 3.1 브러쉬 지오메트리 부재 확인 및 대체 결정

원본 무료 3D 에셋(Sketchfab)에는 브러쉬 슬롯(구멍)만 모델링되어 있고 실제 롤러 브러쉬 메쉬는 없었다. 좌표 기반으로 후보를 찾는 대신(아래 5절 참고) 직접 뷰포트에서 슬롯 위치를 눈으로 확인해 결론 내렸다. 시각적 완성도보다 Joint/Drive/마찰 학습이 목표이므로, `Create > Mesh > Cylinder`로 대략적인 실물 비율(로봇 전체 지름 대비, 길이 20cm/지름 4cm)의 Primitive를 만들어 대체했다.

### 3.2 브러쉬를 독립 Rigid Body로 설정

1. `Cylinder` 프림 선택
2. `Add > Physics > Rigid Body with Colliders Preset`
3. Move(`W`)/Rotate(`E`) 툴로 슬롯 위치·방향에 맞춰 배치

### 3.3 Revolute Joint 생성 및 축 설정

1. `robot1`과 `Cylinder`를 함께 선택 (Ctrl+클릭)
2. `Create > Physics > Joint > Revolute Joint`
3. 생성된 Joint 선택 → 뷰포트의 축 기즈모가 브러쉬의 실제 회전 방향과 일치하는지 확인, 필요시 Property 패널의 `Axis`를 X/Y/Z로 조정
4. `Enable Limit`은 꺼둔 상태 유지 (연속 회전이므로 각도 제한 없음)

### 3.4 Angular Drive 추가 (Script Editor)

Property 패널의 `Add` 버튼에서 `Angular Drive` UI 항목을 찾기 어려웠다 — Script Editor로 직접 `UsdPhysics.DriveAPI`를 적용하는 게 더 확실했다.

```python
import omni.usd
from pxr import UsdPhysics

stage = omni.usd.get_context().get_stage()
joint_prim = stage.GetPrimAtPath("/World/robotvacum_decimated/robot1/RevoluteJoint")

drive = UsdPhysics.DriveAPI.Apply(joint_prim, "angular")
drive.CreateTypeAttr().Set("force")
drive.CreateTargetVelocityAttr().Set(300.0)
drive.CreateDampingAttr().Set(1000.0)
drive.CreateStiffnessAttr().Set(0.0)
```

### 3.5 Physics Material(마찰) 추가 (Script Editor)

```python
import omni.usd
from pxr import UsdShade, UsdPhysics

stage = omni.usd.get_context().get_stage()

mat_path = "/World/PhysicsMaterials/BrushMaterial"
material = UsdShade.Material.Define(stage, mat_path)
physics_mat_api = UsdPhysics.MaterialAPI.Apply(material.GetPrim())
physics_mat_api.CreateStaticFrictionAttr().Set(0.8)
physics_mat_api.CreateDynamicFrictionAttr().Set(0.8)
physics_mat_api.CreateRestitutionAttr().Set(0.1)

cylinder_prim = stage.GetPrimAtPath("/World/Cylinder")
binding_api = UsdShade.MaterialBindingAPI.Apply(cylinder_prim)
binding_api.Bind(material, bindingStrength=UsdShade.Tokens.strongerThanDescendants, materialPurpose="physics")
```

### 3.6 Play로 검증

Play(▶) 클릭 후 `Cylinder`가 목표 속도로 회전하고, `robot1`은 안정적으로 유지되는지 확인.

## 4. 예상/실제 결과 확인

- Play 중 `Cylinder`가 설정한 `Target Velocity`(300 deg/s)로 지속 회전한다.
- `robot1`은 브러쉬 회전의 반작용으로 튕기거나 흔들리지 않고 안정적으로 유지된다.
- 콘솔에 physics 관련 에러가 없다.

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| 좌표 기반으로 "긴 원통형" 후보를 찾아도 실제 브러쉬 롤러가 안 나옴 (찾은 후보들은 전부 몸체 패널이나 작은 하드웨어 부품이었음) | 무료 소스 에셋 자체에 브러쉬 롤러 메쉬가 없고 슬롯(구멍)만 모델링됨 | 형태 대신 직접 뷰포트로 슬롯을 눈으로 확인 후, Primitive Cylinder를 새로 만들어 대체 (시각 충실도보다 물리 학습이 목적이므로 실용적 판단) |
| `brush` Xform을 `robot1` 밖으로 드래그해서 형제로 옮겼더니, 이후 위치 기반 탐색 스크립트에서 나머지 서브메시들과의 거리가 전부 2m 이상으로 나옴 | Outliner 드래그 reparent가 로컬 `xformOp`는 그대로 두고 부모만 바꿔서, `robot1`이 갖고 있던 위치 오프셋만큼 실제 월드 위치가 어긋남 | Script Editor에서 `UsdGeom.Xformable(...).ComputeLocalToWorldTransform(0)`으로 두 프림의 월드 위치 delta를 계산 → 그 delta를 옮겨진 프림의 로컬 translate에 더해 원래 자리로 복귀 |
| Joint를 지우고 다시 만들었다고 생각했는데, Property 패널/Simulation Data Visualizer가 계속 `Accessed invalid attribute` / `expired 'PhysicsRevoluteJoint'` 에러를 반복 출력. Isaac Sim을 완전히 재시작해도 재발 | (1) 실제로는 예전 Joint(`robot1`↔`brush`)가 삭제되지 않고 그대로 남아있었음 — `brush`에서 Rigid Body를 제거하면서 Joint의 body 관계가 깨졌지만 Joint 프림 자체는 안 지워짐. (2) `user.config.json`의 `visualizationSimulationDataVisualizer` 설정 때문에 재시작할 때마다 Simulation Data Visualizer 패널이 자동으로 다시 열리며 죽은 Joint를 다시 참조 | (1) Script Editor로 `stage.Traverse()`하며 실제 `UsdPhysics.RevoluteJoint` 프림과 그 `body0`/`body1`을 직접 조회해 어떤 Joint가 진짜인지 확인 후, 잘못된 걸 명시적으로 삭제. (2) `~/.local/share/ov/data/Kit/Isaac-Sim Full/5.1/user.config.json`에서 `visualizationSimulationDataVisualizer`를 `false`로 직접 수정 |
| Property 패널의 `Add` 버튼에서 `Angular Drive`, 마찰 관련 `Physics Material` 항목을 못 찾음 | Isaac Sim 5.1.0 UI 배치가 이전 버전 튜토리얼과 달라 정확한 위치 확인이 어려움 (`Physics Material`은 Property `Add`가 아니라 상단 `Create > Physics` 메뉴에 있음) | 설치된 확장 소스(`omni.physx.ui`, `omni.kit.property.physx`)를 직접 grep해서 정확한 스키마/메뉴 이름 확인. UI로 못 찾을 땐 `UsdPhysics.DriveAPI.Apply(prim, "angular")`, `UsdShade.MaterialBindingAPI` 등 USD Python API로 직접 적용 |

## 6. 체크포인트

- [ ] 원본 에셋에 브러쉬 지오메트리가 없음을 확인, Cylinder Primitive로 대체
- [ ] Cylinder에 Rigid Body + Collider(Convex Hull 계열) 추가
- [ ] `robot1` ↔ `Cylinder` 사이에 Revolute Joint 생성, 축을 실제 롤러 방향에 맞게 설정
- [ ] Angular Drive 추가 (`stiffness=0`, `damping>0`, `targetVelocity` 설정)로 속도 기반 지속 회전 구현
- [ ] Physics Material(마찰 0.8)을 생성해 Cylinder 콜라이더에 바인딩
- [ ] Play로 브러쉬가 목표 속도로 회전하고 로봇 본체는 안정적인지 확인
- [ ] 죽은 Joint 참조로 인한 반복 콘솔 에러가 없는 깨끗한 상태로 저장

---
Part A(하드웨어 가상화) 완료. 다음: Part B Topic 4 — 2D LiDAR & RGB-D 카메라 배치 (`04-lidar-rgbd-camera.md`, 미작성)
