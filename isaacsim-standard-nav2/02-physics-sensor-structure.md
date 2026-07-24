# 02. 로봇 물리/센서 구조 파악

## 1. 학습 목표

- `ArticulationRootAPI` 기반의 정식 관절 로봇(articulation)이 무엇이고, 개별 RigidBody 조합과 무엇이 다른지 이해한다.
- `RevoluteJoint` + `DriveAPI`로 구성된 실제 바퀴 구동 방식을 읽을 수 있다.
- 시각 메쉬와 충돌 콜라이더를 완전히 분리해서 만드는 표준 패턴(프리미티브 콜라이더)을 이해한다.
- Script Editor에서 USD Physics 스키마를 코드로 덤프해 구조를 한 번에 파악하는 방법을 익힌다.
- 지금까지 파악한 내용을 로봇청소기(vacuum) 프로젝트의 구조와 항목별로 비교한다.

## 2. 핵심 개념

**Articulation Root**: `/World/carter_v1`에는 `PhysicsArticulationRootAPI` + `PhysxArticulationAPI`가 붙어 있다. 이 스키마가 있으면 PhysX는 그 프림 아래의 모든 조인트/링크를 하나의 **관절 체인(articulation)**으로 취급해 함께 풀어낸다. 로봇청소기 프로젝트는 이런 구조가 없었다 — `robot1`이라는 단일 RigidBody를 월드에 앵커된 Prismatic/Revolute 조인트(Gantry rig)로 감싸고, 매 프레임 `physics:velocity`/`physics:angularVelocity`를 직접 write하는 "텔레포트에 가까운" 방식으로 구동했다. Carter는 반대로 실제 바퀴가 조인트 축을 중심으로 회전하며 로봇을 움직이는, 교과서적인 articulated 구동이다.

**실제 바퀴 조인트**: `chassis_link` 아래 `left_wheel`/`right_wheel`이라는 두 개의 `PhysicsRevoluteJoint`가 각각 `chassis_link`↔`left_wheel_link`, `chassis_link`↔`right_wheel_link`를 연결한다. 각 조인트에는 `DriveAPI:angular`(`stiffness=0`, `damping≈17453`)가 붙어 있다 — stiffness 0 + damping만 있는 조합은 위치가 아니라 **속도를 목표로 삼는 순수 속도 드라이브**다. `targetVelocity`에 원하는 바퀴 각속도를 넣으면 PhysX가 그 속도를 유지하도록 토크를 계산해준다.

**캐스터(후방 바퀴)의 2단 조인트 구조**: `rear_pivot`(chassis↔rear_pivot_link, 자유 회전, drive 없음) → `rear_axle`(rear_pivot_link↔rear_wheel_link, 자유 회전, drive 없음). 드라이브가 없는 두 개의 자유 회전 조인트를 직렬로 연결해, 실제 캐스터 바퀴처럼 좌우로 자유롭게 방향을 틀면서 굴러가는 거동을 만든다.

**무게중심/센서를 별도 RigidBody + FixedJoint로 표현**: `com_offset`과 `imu`는 둘 다 `chassis_link`에 종속된 단순 Xform이 아니라, 각각 자체 `RigidBodyAPI`+`MassAPI`를 가진 별도 바디이고 `FixedJoint`(`com_joint`, `imu_joint`)로 `chassis_link`에 고정돼 있다. 무게중심 오프셋을 물리적으로 분리된 작은 질량체로 표현하면, 단순히 `MassAPI`의 `centerOfMass` 필드를 조정하는 것보다 관성텐서 계산이 더 직관적으로 맞아떨어진다. 로봇청소기 프로젝트는 이런 패턴을 쓰지 않았다.

**시각 메쉬와 콜라이더의 완전한 분리**: `carter_wheel_left`(Mesh), `chassis_obj/carter_main`(Mesh) 같은 실제 시각 지오메트리에는 물리 API가 전혀 없다 — `MaterialBindingAPI`(재질 연결)만 있다. 대신 `left_wheel_link/cylinder`(Cylinder), `chassis_link/box`(Cube), `rear_pivot_link/box`(Cube)처럼 **눈에 안 보이는 단순 프리미티브**가 별도로 겹쳐져서 콜라이더 역할만 한다. 로봇청소기 프로젝트는 시각 메쉬(140여 개 서브메시) 자체에 `Approximation=Convex Hull`을 직접 적용하는 방식이었고, 이 때문에 작은 스케일에서 Convex Decomposition이 부풀어 오르는 버그([[isaacsim_ros2_advanced_curriculum]] Topic 2)까지 겪었다. Carter처럼 시각 메쉬와 무관한 단순 프리미티브를 콜라이더로 쓰면 그 버그 자체가 애초에 발생할 수 없는 구조다.

**LiDAR 사전 장착**: `chassis_link/XT_32_10Hz`는 이미 배치·설정된 `OmniLidar` 프림이다(Hesai XT32 계열 프로파일, 10Hz로 추정). 로봇청소기 프로젝트에서는 RTX Lidar를 손으로 만들고 2D/3D 프로파일 버그([[isaacsim_ros2_advanced_curriculum]] Topic 4)까지 겪었지만, 표준 에셋은 센서가 이미 완제품으로 포함돼 있다.

## 3. 실습 단계

### 3.1 물리 구조 덤프 스크립트 작성

`Usd.PrimRange`로 `/World/carter_v1` 전체를 순회하며 Joint 타입, 적용된 Physics 스키마, Drive 값, ArticulationRoot 위치, 콜라이더 Approximation을 텍스트 파일로 저장하는 스크립트를 준비한다. (`~/isaac_assets/carter_standard/scripts/dump_carter_physics_2026-07-25.py`)

### 3.2 Script Editor에서 실행

`Window > Script Editor`를 열고 `File > Open`으로 위 스크립트 경로를 직접 로드한다(클립보드 복사가 막혀 있을 때는 붙여넣기 대신 이 방법을 쓴다 — 5절 참고). `Run`으로 실행한다.

### 3.3 결과 확인

콘솔에 `Written to ~/isaac_assets/carter_physics_dump.txt`가 뜨면 그 파일을 열어 조인트/드라이브/콜라이더 목록을 확인한다.

## 4. 예상/실제 결과 확인

덤프 결과를 로봇청소기(vacuum) 프로젝트와 항목별로 비교하면:

| 항목 | Carter (표준 에셋) | 로봇청소기 (커스텀 CAD) |
|---|---|---|
| 최상위 구조 | `ArticulationRootAPI` — 진짜 관절 체인 | 단일 `robot1` RigidBody + 월드 앵커 Gantry Prismatic/Revolute 조인트 |
| 구동 방식 | `RevoluteJoint`+`DriveAPI:angular`(속도 드라이브)로 바퀴가 실제로 회전 | 매 프레임 `physics:velocity`/`physics:angularVelocity`를 바디에 직접 write |
| 후방 지지 | 2단 자유 회전 조인트(pivot+axle) 캐스터 | 없음(단일 바디) |
| 콜라이더 | 시각 메쉬와 분리된 단순 프리미티브(Cube/Cylinder) | 시각 메쉬(~140 서브메시)에 직접 Convex Hull 적용 |
| LiDAR | 사전 장착(`XT_32_10Hz`) | RTX Lidar를 수동 생성, 2D/3D 프로파일 버그 직접 해결 |
| 무게중심 표현 | 별도 RigidBody(`com_offset`) + FixedJoint | `MassAPI`의 `centerOfMass` 필드 값으로만 표현 |

이 표에서 가장 중요한 차이는 **구동 방식**이다 — vacuum 프로젝트의 wedging 버그는 벽 근처에서 발생했는데, `physics:velocity` 직접 write 방식은 실제 바퀴-바닥 마찰/접촉 반응이 구동 로직에 전혀 반영되지 않는 반면, Carter의 `RevoluteJoint` 기반 구동은 PhysX 접촉 솔버가 자연스럽게 개입한다. Topic 5에서 같은 벽 근접 시나리오를 재현했을 때 이 차이가 wedging 재현 여부에 영향을 주는지 확인한다.

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| Script Editor에 스크립트를 붙여넣으려 해도 클립보드 복사가 안 됨 | 이 세션의 터미널/브라우저 환경에서 클립보드 복사 기능 자체가 동작하지 않음 | Claude가 스크립트를 디스크 파일로 직접 저장하고, Script Editor의 `File > Open`으로 경로를 입력해 파일을 바로 로드 — 복사/붙여넣기 과정을 아예 건너뛴다. 결과도 `print` 대신 텍스트 파일로 저장해 Claude가 파일시스템에서 직접 읽는다 |

## 6. 체크포인트

- [ ] `/World/carter_v1`에 `ArticulationRootAPI`가 붙어있음을 확인
- [ ] `left_wheel`/`right_wheel` 조인트가 `RevoluteJoint`+`DriveAPI:angular`(속도 드라이브)로 구성됨을 확인
- [ ] `rear_pivot`/`rear_axle` 2단 자유 회전 조인트로 캐스터가 구성됨을 확인
- [ ] 시각 메쉬(`carter_wheel_left` 등)와 콜라이더(`cylinder`, `box`)가 서로 다른 프림으로 분리돼 있음을 확인
- [ ] `XT_32_10Hz` LiDAR가 이미 장착돼 있음을 확인
- [ ] vacuum 프로젝트와의 비교표를 근거로, wedging 재현 여부를 확인할 다음 실습(Topic 5) 가설 수립

---
다음: `03-ros2-bridge-omnigraph.md` (미작성) — ROS2 브릿지 & OmniGraph 연결
