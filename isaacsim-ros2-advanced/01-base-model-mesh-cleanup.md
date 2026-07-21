# 01. 베이스 모델 설정 & 메쉬 다듬기

## 1. 학습 목표

- CAD/스캔 원본 3D 에셋을 Isaac Sim에서 다루기 적합한 경량 메쉬로 줄이는 전체 파이프라인(Blender Decimation → OBJ/USD export → Isaac Sim import)을 직접 수행한다.
- Blender의 Decimate modifier를 여러 오브젝트에 한 번에 적용하는 방법과, "적용됐다고 보이는 것"과 "실제로 export에 반영되는 것"의 차이를 이해한다.
- Isaac Sim Stage에서 참조(Reference)로 들어온 프림과 로컬(local) 프림의 차이, 그리고 참조를 로컬로 바꾸는 Flatten 작업을 이해한다.
- Stage 프림을 의미 단위(로봇 본체/거치대)로 그룹핑해 이후 Topic(Collider, Joint 등) 작업의 기반을 만든다.

## 2. 핵심 개념

**폴리곤 경량화가 필요한 이유**: 실시간 물리 시뮬레이션(PhysX)과 렌더링은 폴리곤 수에 민감하다. 특히 Convex Decomposition 같은 Collider 생성 연산은 원본 메쉬의 삼각형 수에 거의 선형으로 비용이 늘어나므로, 물리에 쓰일 메쉬는 시각적 디테일을 크게 해치지 않는 선에서 최대한 줄여두는 것이 정석이다. 이 트랙에서는 20,000~60,000 tris를 실습용 목표 범위로 잡았다.

**Decimate modifier vs 실제 적용(bake)**: Blender의 modifier는 기본적으로 "뷰포트에 보여지는 결과"일 뿐, 오브젝트의 실제 지오메트리 데이터(mesh datablock)는 바뀌지 않는다. Export나 이후 작업에서 이 감소된 결과를 그대로 쓰려면 modifier를 지오메트리에 굽는(bake/apply) 과정이 별도로 필요하다 — `Object > Convert > Mesh`가 선택된 모든 오브젝트에 대해 이 작업을 한 번에 해준다.

**USD의 Reference/Payload와 "ancestral prim"**: Isaac Sim이 OBJ/FBX 같은 비-USD 포맷을 import할 때는, 파일을 곧바로 Stage에 지오메트리로 굽는 게 아니라 별도 `.usd` 파일로 변환한 뒤 그 파일을 **참조(Reference)**로 Stage에 연결한다. 참조된 콘텐츠 안의 프림들은 현재 Stage의 관점에서 "ancestral"(상위 레이어 소속) 상태이므로, 지금 Stage에서 직접 이름을 바꾸거나 부모를 바꾸는(reparent) 작업이 막힌다. `Usd.Stage.Flatten()`으로 모든 참조/페이로드를 하나의 로컬 레이어로 합치면 이 제약이 풀린다.

## 3. 실습 단계

### 3.1 에셋 준비

CAD 원본(STEP/IGES)이 없어 Sketchfab에서 CC 라이선스 3D 모델("XIAOMI Robot Vacuum X10+")을 FBX로 대체 사용했다.

```bash
# 원본 FBX 위치
~/isaac_assets/vacuum_robot/source/robovacum.fbx
```

### 3.2 Isaac Sim에서 원본 폴리곤 수 확인

`Window > Script Editor`에서:

```python
from pxr import Usd, UsdGeom

stage = omni.usd.get_context().get_stage()
root_path = "/World/robovacum"
total_tris = 0
for prim in Usd.PrimRange(stage.GetPrimAtPath(root_path)):
    if prim.IsA(UsdGeom.Mesh):
        mesh = UsdGeom.Mesh(prim)
        counts = mesh.GetFaceVertexCountsAttr().Get()
        if counts:
            total_tris += sum(c - 2 for c in counts)
print(f"Total tris: {total_tris}")
```

원본은 **635,509 tris** — 목표 범위(20,000~60,000)의 10배 이상이라 Decimation이 필수였다.

### 3.3 Blender에서 Decimation

```bash
sudo apt install blender   # Ubuntu 저장소 버전, 4.0.2
```

1. `File > Import > FBX`로 원본 FBX 임포트
2. 뷰포트 `A`로 전체 오브젝트 선택
3. 렌치 아이콘(Modifier Properties)이 안 보이면 → 메시 오브젝트가 선택 안 된 상태다. Outliner에서 메시를 직접 클릭해서 선택할 것
4. 활성 오브젝트에 `Add Modifier > Generate > Decimate` 추가, `Collapse` 모드, Ratio `0.08~0.1`
5. 전체 선택 유지한 채 `Ctrl+L` (Link/Transfer Data) → **`Copy Modifiers`**로 나머지 오브젝트에도 동일 modifier 적용
6. 전체 선택 유지한 채 `Object > Convert > Mesh`로 modifier를 실제 지오메트리에 bake
7. `Overlays > Statistics`로 최종 Faces 수 확인 — **48,261**로 목표 범위 안착

### 3.4 OBJ export 및 Isaac Sim 재-import

```
File > Export > Wavefront (.obj)
저장 경로: ~/isaac_assets/vacuum_robot/decimated/robotvacum_decimated.obj
```

Isaac Sim에서 `File > Import`로 이 OBJ를 불러온다. Import 직후 아무것도 안 보이면 root 프림을 선택하고 뷰포트에서 `F`(Frame Selected)를 눌러본다 — 대부분 카메라 프레이밍 문제일 뿐 스케일 문제가 아니다.

### 3.5 Stage 정리: Flatten + 그룹핑

OBJ import는 참조 기반이라 프림 이름 변경/그룹핑이 바로 막힌다(3.6절 참고). Script Editor에서:

```python
import omni.usd

stage = omni.usd.get_context().get_stage()
flattened_layer = stage.Flatten()
result = flattened_layer.Export("/path/to/robotvacum_flattened.usd")
print("Export result:", result)
```

생성된 파일을 `File > Open`으로 **직접** 연다(참조로 다시 불러오면 안 된다 — 그러면 ancestral 문제가 재발한다). 이제 Outliner에서:

1. 로봇 본체에 해당하는 프림들을 다중 선택 → `Ctrl+G` (Group Selected) → 이름을 의미 있게 변경
2. 거치대(도킹 스테이션) 프림들도 같은 방식으로 그룹핑
3. 소스 에셋에 로봇이 두 벌 들어있는 경우(스탠드얼론 버전 + 거치대에 얹힌 "도킹 포즈" 장식용 복제본) 흔하다 — 눈 아이콘으로 하나씩 꺼보며 중복 여부를 확인하고, 장식용 복제본은 삭제한다. 실제 도킹 동작은 나중에 Nav2로 로봇을 거치대 위치까지 이동시켜 구현할 것이므로 미리 박제된 메쉬는 불필요하다.
4. **그룹핑 직후, 그룹의 원점이 실제 메쉬 중심과 일치하는지 바로 확인한다** (아래 5절 표 참고 — `Ctrl+G`는 원점을 지오메트릭 중심이 아니라 그룹 생성 시점의 임의 기준으로 잡을 수 있다). 이 확인을 지금 하지 않으면, 훨씬 나중에 회전 조인트를 붙이는 Topic에서야 "제자리 회전인데 왜 큰 원을 그리지?" 형태로 드러나 원인 추적이 훨씬 어려워진다.
5. `Ctrl+S`로 저장

## 4. 예상/실제 결과 확인

- Blender Statistics와 Isaac Sim Script Editor 양쪽에서 측정한 삼각형 수가 **48,261**로 정확히 일치해야 한다.
- Stage 최상위에 로봇 본체 Xform과 거치대 Xform 두 개만 남아있어야 한다 (중복 로봇 메쉬 삭제 완료 상태).
- 프림 이름을 더블클릭해서 rename이 에러 없이 되어야 한다 (ancestral prim 문제 해소 확인).

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| `Copy to Selected`(값 필드 우클릭)가 일부 오브젝트에서 비활성화됨 | 이 방식은 대상 오브젝트에 *이미 같은 종류의 modifier가 있어야만* 동작 — 없는 오브젝트엔 회색 처리 | `Ctrl+L` (Link/Transfer Data) → `Copy Modifiers` 사용. 대상에 modifier가 없어도 통째로 복사됨 |
| Blender 뷰포트 Statistics에는 목표치(수천 tris)로 보였는데 실제 export한 OBJ는 원본과 큰 차이 없는 삼각형 수(수십만)로 나옴 | Decimate modifier가 일부 오브젝트에만 실제로 전파/적용됐고, modifier가 지오메트리에 bake되지 않은 채 export됨(exporter의 "Apply Modifiers"에만 의존하면 안 됨) | 전체 선택 후 `Object > Convert > Mesh`로 명시적으로 bake한 뒤 export |
| Export한 OBJ에 `o` 태그가 원래 오브젝트 수(16개)가 아니라 143개로 쪼개짐 (`Object Groups`/`Material Groups` 옵션을 조정해도 동일) | 원본 FBX 자체가 143개의 개별 서브메시로 구성되어 있음 — Isaac Sim이 최초 FBX import 때 보여준 그룹 이름은 Blender 오브젝트 이름이 아니라 Isaac Sim 자체의 표시상 그룹핑이라, Blender에 애초에 보존할 원본 이름 정보가 없었음 | 이름 정리는 Blender에서 포기하고 Isaac Sim Stage에서 직접 진행 (프림 선택 → `Ctrl+G`로 의미 단위 재그룹핑) |
| apt로 설치한 Blender 4.0.2에서 USD export 메뉴 항목 자체가 안 보임, Python에서 `bpy.ops.wm.usd_export()` 호출 시 `AttributeError: ... could not be found` | Ubuntu 저장소의 Blender 패키지가 USD 라이브러리 없이 빌드됨 (오퍼레이터는 등록만 되어 있고 실제 기능 없음) | OBJ 경유 방식으로 진행. USD 직접 export가 꼭 필요하면 apt 대신 Snap(`snap install blender --classic`) 또는 blender.org 공식 tarball 설치 필요 |
| Isaac Sim에서 import한 프림을 `Group Selected`/rename 시도 시 `Cannot move/rename ancestral prim ...` 경고와 함께 실패 | OBJ import가 지오메트리를 Stage에 직접 굽지 않고, 자동 변환된 `.usd`를 **참조(Reference)**로만 연결함 — 참조 내부 프림은 현재 Stage 기준 "ancestral" 상태라 직접 편집 불가 | Script Editor에서 `stage.Flatten()` 후 `.Export()`로 새 파일 저장 → 그 파일을 `File > Open`으로 **직접** 열기(참조 아님) → 이후 rename/그룹핑 정상 동작 |
| 그룹핑(`Ctrl+G`) 후 그룹 Xform의 원점이 실제 메쉬 지오메트릭 중심과 크게 어긋남 — 본 트랙에서는 이 시점에 발견되지 않고 한참 뒤 Topic 9(회전 조인트 작업)에서야 "제자리 회전인데 로봇 몸체가 지름 1m 가까운 원을 그림"으로 드러났다 (`robot1` 원점은 실제로 정지해 있었는데, 시각적 메쉬가 원점에서 약 0.65m 떨어져 있었던 것) | `Ctrl+G`가 그룹 Xform의 원점을 메쉬의 지오메트릭 중심이 아니라 그룹 생성 시점의 임의 기준(월드 원점 등)으로 잡음 | `UsdGeom.BBoxCache`로 그룹의 world bbox 중심과 원점(`ComputeLocalToWorldTransform` 위치)을 비교해 어긋남을 확인. 고칠 때는 **그룹 자신의 translate는 절대 건드리지 말 것** — 이미 조인트가 붙어 있다면 물리 바디를 스크립트로 순간이동시키는 셈이라 위험하다(실제로 물리 시뮬레이션이 붕괴할 수 있다). 대신 자식 메쉬의 `points`(vertex) 데이터와 자식 Xform들의 `translate`를 원점 기준 반대 방향으로 shift해서, 그룹의 좌표계 원점 자체는 그대로 두고 지오메트리만 그 원점 위로 재배치한다 |

## 6. 체크포인트

- [ ] Isaac Sim Script Editor로 원본 메쉬 삼각형 수 측정 (635,509 tris)
- [ ] Blender에서 전체 오브젝트에 Decimate modifier 적용 (`Ctrl+L > Copy Modifiers`)
- [ ] `Object > Convert > Mesh`로 modifier를 실제 지오메트리에 bake
- [ ] OBJ export 후 Isaac Sim에 재-import, 삼각형 수가 목표 범위(20k~60k) 안에서 Blender와 정확히 일치하는지 확인
- [ ] `stage.Flatten()`으로 참조를 로컬 프림으로 변환
- [ ] 로봇 본체 / 거치대 프림을 의미 단위로 그룹핑, 중복 메쉬 정리
- [ ] 그룹핑 후 world bbox 중심과 그룹 원점이 (거의) 일치하는지 확인 — 나중에 회전 조인트를 붙였을 때 안정성에 직결된다
- [ ] 최종 Stage 저장

---
다음: `02-collider-com.md` (미작성) — 충돌체(Collider) 및 무게중심(COM) 수정
