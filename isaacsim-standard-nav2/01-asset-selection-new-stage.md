# 01. 에셋 선정 & 새 스테이지 구성

## 1. 학습 목표

- Isaac Sim Asset Browser에서 검증된 표준 로봇 에셋을 찾아 새 스테이지에 배치할 수 있다.
- 표준 에셋을 배치한 직후 Stage 패널만으로 그 안에 이미 갖춰진 구조(링크 이름, 센서, 재질)를 파악할 수 있다.
- Measure/Frame Selected를 이용해 배치된 에셋의 실제 크기가 그럴듯한지 빠르게 검증할 수 있다.

## 2. 핵심 개념

**왜 커스텀 CAD 에셋 대신 표준 에셋인가**: `isaacsim-ros2-advanced` 트랙(로봇청소기)은 Sketchfab에서 받은 CAD 메쉬를 직접 데시메이션·콜라이더·조인트까지 전부 손으로 구성했다. 그 과정에서 나온 여러 버그(Convex Decomposition 부풀림, Reparent 시 월드 좌표 튐, PrismaticJoint localPos 오염 등)가 로봇 하드웨어 자체의 문제인지 Nav2/설정 문제인지 구분하기 어려웠다. Isaac Sim의 Asset Browser에 있는 표준 로봇 에셋은 NVIDIA가 미리 리깅해 둔 것이라, 같은 Nav2 시나리오를 재현했을 때 문제가 사라지면 원인이 로봇 하드웨어 쪽이었다고 추론할 수 있는 대조군 역할을 한다.

**Asset Browser 경로**: `Window > Browsers > Assets`에서 Isaac 계열 표준 로봇은 `Isaac > Robots` 카테고리 아래 있다. 이 실습에서는 그중 `carter_v1.usd`(NVIDIA Carter 로봇, 차동구동+후방 캐스터 구성)를 사용한다.

## 3. 실습 단계

### 3.1 새 스테이지 생성

`File > New From Stage Template`(또는 `Ctrl+N`)으로 빈 스테이지를 연다. 기존 로봇청소기 프로젝트 스테이지와 섞이지 않도록 반드시 새 파일로 시작한다.

### 3.2 Asset Browser에서 표준 로봇 찾기

`Window > Browsers > Assets`를 열고 `Isaac > Robots` 카테고리로 들어가 `carter_v1.usd`를 찾는다.

### 3.3 뷰포트에 배치

`carter_v1.usd`를 뷰포트로 드래그&드롭해 원점 근처에 배치한다.

### 3.4 계층 구조 확인

Stage 패널에서 `/World/carter_v1` 아래 자식 프림들을 펼쳐 확인한다.

### 3.5 크기 검증

배치된 `carter_v1`을 선택하고 `F`(Frame Selected)를 눌러, 뷰포트 그리드(칸 하나 = 1m) 대비 실제 크기가 그럴듯한지 확인한다.

## 4. 예상/실제 결과 확인

배치 직후 `/World/carter_v1` 아래 다음 자식 프림들이 보여야 한다:

```
/World/carter_v1/chassis_link
/World/carter_v1/left_wheel_link
/World/carter_v1/right_wheel_link
/World/carter_v1/rear_pivot_link
/World/carter_v1/rear_wheel_link
/World/carter_v1/com_offset
/World/carter_v1/imu
/World/carter_v1/WheelMaterial
```

`left_wheel_link`/`right_wheel_link`(구동 바퀴)와 `rear_pivot_link`/`rear_wheel_link`(자유 회전 캐스터)의 조합으로 이미 차동구동 로봇의 링크 이름 규칙이 잡혀 있고, `imu`(센서)와 `WheelMaterial`(마찰 physics material)까지 기본 포함돼 있다 — 로봇청소기 프로젝트에서는 이 전부를 Topic 1~5에 걸쳐 손으로 만들어야 했다.

크기는 그리드 칸 절반 정도(~0.4~0.5m급)로 나와야 한다. Carter는 실물 기준 대략 45cm급 실내 로봇이므로 이 정도면 정상이다.

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| `~/isaacsim_assets`와 `~/isaac_assets` 두 디렉토리가 둘 다 존재해서 스크립트/에셋 경로 혼동 | 과거 세션에서 오타로 두 개의 유사한 이름의 디렉토리가 각각 쓰인 채 남음 | 이 트랙(및 로봇청소기 트랙)의 실제 작업 디렉토리는 `~/isaac_assets`(중간에 "sim" 없음)이다. `~/isaacsim_assets`는 레거시이므로 새 파일은 항상 `~/isaac_assets` 쪽에 만든다 |

## 6. 체크포인트

- [ ] 새 스테이지로 시작 (기존 vacuum 프로젝트 스테이지와 분리)
- [ ] Asset Browser `Isaac > Robots`에서 `carter_v1.usd` 확인
- [ ] 뷰포트에 배치, Stage 패널에서 8개 자식 프림(`chassis_link`/`left_wheel_link`/`right_wheel_link`/`rear_pivot_link`/`rear_wheel_link`/`com_offset`/`imu`/`WheelMaterial`) 확인
- [ ] `F`로 프레임, 그리드 대비 ~0.4~0.5m급 크기 확인

---
다음: `02-physics-sensor-structure.md` — 로봇 물리/센서 구조 파악 (표준 에셋의 조인트/콜라이더/센서 구성을 vacuum 프로젝트와 비교)
