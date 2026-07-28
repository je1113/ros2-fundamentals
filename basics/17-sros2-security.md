# SROS2 (ROS 2 Security)

[`00-syllabus.md`](./00-syllabus.md)의 17번째(추가) 주제. 지금까지 만든 모든 노드는 같은 네트워크에 있으면 누구나 토픽을 엿듣거나 끼어들 수 있는 상태였다. SROS2는 DDS-Security 플러그인(인증/접근 제어/암호화)을 이용해 "허가된 노드끼리만" 통신하도록 강제한다. 새 노드 코드 없이, 토픽 2의 `battery_publisher`/`battery_watcher`에 보안 계층만 씌워본다.

---

## 학습 목표

- SROS2의 keystore/enclave 구조(CA, 노드별 신원 인증서, 권한 정책)를 설명할 수 있다.
- `ros2 security create_keystore`/`create_enclave`로 노드별 보안 자격을 발급할 수 있다.
- `ROS_SECURITY_*` 환경변수로 노드를 보안 모드로 실행하고, 정당한 노드끼리는 정상 통신되는 것을 확인한다.
- 보안 자격이 없는 노드는 토픽 디스커버리 자체가 안 되는 것(=차단)을 직접 확인한다.

## 핵심 개념

- **Keystore**: 루트 CA 역할. `identity_ca`(신원 인증용)와 `permissions_ca`(권한 정책 서명용) 두 개의 CA 키/인증서 쌍을 담는다.
- **Enclave**: 하나의 컨텍스트(보통 노드 하나 또는 노드 그룹)에 대응하는 보안 자격 묶음. `create_enclave`로 발급하면 `cert.pem`/`key.pem`(신원), `governance.p7s`(도메인 전체 보안 규칙, CA 서명됨), `permissions.p7s`(이 enclave가 뭘 발행/구독할 수 있는지 정책, CA 서명됨)이 만들어진다.
- **DDS-Security 3대 플러그인**: Authentication(상대가 같은 CA로 서명된 진짜 참가자인지 핸드셰이크로 확인), Access Control(governance/permissions로 어떤 토픽/서비스를 쓸 수 있는지 제한), Cryptography(옵션: 페이로드 암호화). 이번 실습은 기본 설정(암호화 없이 인증+접근제어)만 확인한다.
- **`ROS_SECURITY_*` 환경변수**:
  - `ROS_SECURITY_KEYSTORE`: keystore 경로
  - `ROS_SECURITY_ENABLE=true`: 보안 활성화
  - `ROS_SECURITY_STRATEGY=Enforce`: 보안 자격이 없거나 틀리면 노드 생성 자체를 실패시킴(`Permissive`는 없으면 그냥 비보안으로 폴백)
  - `ROS_SECURITY_ENCLAVE_OVERRIDE`: 이 노드가 어떤 enclave를 쓸지 명시. **자동 추론에 의존하지 말고 항상 명시하는 게 안전**(아래 알려진 문제 참고).

## 실습 단계

### 1. `sros2` 설치 확인

```bash
ros2 pkg list | grep sros2
ros2 security -h
```

없으면 설치: `sudo apt install ros-jazzy-sros2`

### 2. Keystore 생성 + 노드별 enclave 발급

```bash
ros2 security create_keystore ~/sros2_keystore
ros2 security create_enclave ~/sros2_keystore /battery_publisher
ros2 security create_enclave ~/sros2_keystore /battery_watcher
ros2 security list_enclaves ~/sros2_keystore
```

`~/sros2_keystore/enclaves/battery_publisher/`, `.../battery_watcher/` 아래에 각각 `cert.pem`, `key.pem`, `identity_ca.cert.pem`, `permissions_ca.cert.pem`, `governance.p7s`, `permissions.p7s`, `permissions.xml`이 생긴다.

### 3. 보안 모드로 정상 통신 확인 (터미널 A, B)

**터미널 A** — `battery_publisher`:

```bash
export ROS_SECURITY_KEYSTORE=~/sros2_keystore
export ROS_SECURITY_ENABLE=true
export ROS_SECURITY_STRATEGY=Enforce
export ROS_SECURITY_ENCLAVE_OVERRIDE=/battery_publisher
ros2 run ros2_basics battery_publisher
```

**터미널 B** — `battery_watcher`:

```bash
export ROS_SECURITY_KEYSTORE=~/sros2_keystore
export ROS_SECURITY_ENABLE=true
export ROS_SECURITY_STRATEGY=Enforce
export ROS_SECURITY_ENCLAVE_OVERRIDE=/battery_watcher
ros2 run ros2_basics battery_watcher
```

두 노드 모두 로그에 `Found security directory: .../sros2_keystore/enclaves/<노드이름>`이 찍히고, `battery_watcher`가 평소처럼 배터리 상태를 정상 로그로 찍으면 성공.

### 4. 권한 없는 리스너 차단 확인 (터미널 C)

터미널 A, B는 그대로 둔 채, **`ROS_SECURITY_*`를 export하지 않은** 새 터미널에서:

```bash
source ~/ros2_ws/install/setup.bash
ros2 topic echo /battery_status
```

## 예상/실제 결과

- Keystore/enclave 생성 후 `ls -R ~/sros2_keystore`로 본 디렉터리 구조가 예상대로(각 enclave에 신원+권한 파일 전부) 나옴. (확인됨)
- `ROS_SECURITY_ENCLAVE_OVERRIDE` 없이 실행하면 `rclpy._rclpy_pybind11.RCLError: ... couldn't find all security files!`로 노드 생성 자체가 실패. (확인됨, 아래 알려진 문제 참고)
- `ROS_SECURITY_ENCLAVE_OVERRIDE`를 각 노드 이름으로 명시하면 에러 없이 정상 기동, `battery_watcher`가 배터리 상태 로그를 정상적으로 찍음. (확인됨)
- 보안 env 없이 켠 `ros2 topic echo /battery_status`는 `WARNING: topic ... does not appear to be published yet` / `Could not determine the type for the passed topic`만 뜨고 아무 데이터도 못 받음 — 인증 안 된 참가자는 디스커버리 단계에서부터 보안 노드를 아예 보지 못한다는 것을 직접 확인. (확인됨)

## 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| `ROS_SECURITY_KEYSTORE`/`ENABLE`/`STRATEGY`만 export하고 실행하면 `RCLError: ... couldn't find all security files!` (로그에는 `Found security directory: .../sros2_keystore/enclaves`까지만 찍히고 노드 이름 하위 경로가 안 붙음) | enclave 자동 추론이 기대대로 노드의 FQN(`/battery_watcher`)으로 안 되고 keystore 최상위에서 멈춤 | `ROS_SECURITY_ENCLAVE_OVERRIDE=/<노드이름>`을 노드마다 명시적으로 export — 자동 추론에 의존하지 않는다 |

## 체크포인트

- [x] keystore(CA)와 enclave(노드별 신원+권한)의 관계를 설명할 수 있다.
- [x] `create_keystore`/`create_enclave`로 두 노드의 보안 자격을 발급했다.
- [x] `ROS_SECURITY_*` 환경변수로 보안 모드를 켜고, 정당한 노드끼리 정상 통신되는 것을 확인했다.
- [x] `ROS_SECURITY_ENCLAVE_OVERRIDE`를 명시해야 하는 이유(자동 추론 실패 사례)를 직접 겪고 이해했다.
- [x] 보안 자격 없는 리스너가 토픽 디스커버리 자체에서 차단되는 것을 확인했다.

---
`basics` 트랙 통신/실행 모델(QoS, Executor, Lifecycle)에 이어 "누가 통신할 자격이 있는가"까지 다뤘다. 실제 배포 환경에서는 여기에 `ros2 security generate_policy`로 실제 그래프에서 정책을 자동 생성하는 워크플로우가 이어진다.
