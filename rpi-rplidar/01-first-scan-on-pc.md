# 01. PC에서 RPLIDAR 첫 연결

> 진행 상태: **완료** — 충전전용 케이블 문제와 apt 저장소 충돌을 해결하고 `/scan` 실데이터 확인

라즈베리파이에 연결할 모니터가 아직 없어서, 오늘은 기존 PC(ROS2 Jazzy 설치되어 있음)에 RPLIDAR A1을 바로 붙여서 먼저 검증한다. 나중에 라즈베리파이로 옮길 때도 같은 절차를 그대로 반복하면 된다.

## 1. 학습 목표

- 실물 USB 시리얼 장치(RPLIDAR)를 리눅스에서 인식시키고, ROS2 노드로 데이터를 받아온다.
- USB 장치가 인식이 안 될 때 `lsusb`/`dmesg`로 원인(케이블/드라이버/권한)을 좁혀나가는 진단 순서를 익힌다.
- 시리얼 포트 접근 권한(`dialout` 그룹)의 필요성과, 그룹 변경이 즉시 반영되지 않는 이유(로그인 세션 단위)를 이해한다.

## 2. 핵심 개념

**충전 전용 케이블 vs 데이터 케이블**: 겉보기엔 똑같은 USB 케이블이라도 내부에 D+/D- 데이터선이 없는 "충전 전용" 케이블이 있다. 이런 케이블로 장치를 꽂으면 `lsusb`/`dmesg` 어디에도 **아무 흔적이 안 남는다** — 에러조차 안 뜨고 완전히 조용하다. `sudo dmesg -w`를 띄워놓고 케이블을 뽑았다 꽂았을 때 정말 아무 반응이 없으면 케이블을 의심해야 한다.

**그룹 변경은 새 로그인 세션부터 적용된다**: `sudo usermod -aG dialout $USER`로 그룹을 추가해도, **이미 열려있는 터미널의 `groups` 출력은 안 바뀐다** — 그 셸은 로그인할 때의 그룹 목록을 그대로 들고 있기 때문이다. `newgrp dialout`으로 그 자리에서 바로 적용하거나, 완전히 로그아웃/재로그인해야 한다.

**`rplidar_ros`의 기본 launch 파일은 이미 A1/A2용이다**: 패키지에 모델별 launch 파일이 일부만 있고(`rplidar_a3.launch.py`, `rplidar_s1.launch.py` 등), A1 전용 파일은 따로 없다 — 대신 기본 `rplidar.launch.py` 자체가 `serial_baudrate: 115200`(A1/A2 값)으로 이미 설정되어 있어서 별도 옵션 없이 그대로 쓰면 된다.

## 3. 실습 단계

### 3.1 사전 문제: apt 저장소 충돌 (ROS2와 무관)

`sudo apt update`가 다음 에러로 실패했다:
```
E: Conflicting values set for option Signed-By regarding source https://packages.microsoft.com/repos/code/ stable: /etc/apt/keyrings/packages.microsoft.gpg != /usr/share/keyrings/microsoft.gpg
```

VS Code 저장소가 옛날 방식(`/etc/apt/sources.list.d/vscode.list`)과 새 방식(`vscode.sources`, VS Code가 자동 관리)으로 중복 등록되어 있었다. 새 방식 쪽을 남기고 옛날 파일만 치웠다:

```bash
sudo mv /etc/apt/sources.list.d/vscode.list /etc/apt/sources.list.d/vscode.list.bak
sudo apt update
```

### 3.2 `rplidar_ros` 설치

```bash
conda deactivate
sudo apt install ros-jazzy-rplidar-ros
```

### 3.3 RPLIDAR 연결 — 첫 시도는 인식 안 됨

```bash
ls /dev/ttyUSB*
# ls: cannot access '/dev/ttyUSB*': No such file or directory
```

`lsusb`에도 라이다 관련 장치가 전혀 안 보였다. 실시간으로 지켜보며 케이블을 뽑았다 꽂아봤다:

```bash
sudo dmesg -w
```

**아무 반응도 없었다** — 케이블 자체가 충전 전용이라는 신호. 데이터 전송이 확실한 다른 케이블(휴대폰 동기화용 등)로 교체하니 즉시 인식됐다:

```
usb 1-4: Product: CP2102 USB to UART Bridge Controller
usb 1-4: Manufacturer: Silicon Labs
cp210x 1-4:1.0: cp210x converter detected
usb 1-4: cp210x converter now attached to ttyUSB0
```

### 3.4 시리얼 포트 권한 설정

```bash
ls -l /dev/ttyUSB0
# crw-rw---- 1 root dialout 188, 0 ... /dev/ttyUSB0
groups | grep dialout   # 안 나옴
sudo usermod -aG dialout $USER
newgrp dialout           # 로그아웃 없이 바로 적용
groups                   # dialout 포함되어 나옴
```

**주의**: `newgrp`으로 연 새 셸에서 conda가 자동으로 `(base)` 활성화되는 경우가 있다 — ROS2 명령 전에 `conda deactivate` 필요.

### 3.5 `rplidar_ros` 실행

```bash
conda deactivate
source /opt/ros/jazzy/setup.bash
ros2 launch rplidar_ros rplidar.launch.py
```

### 3.6 데이터 확인

새 터미널에서:
```bash
ros2 topic hz /scan
ros2 topic echo /scan --once
```

## 4. 예상/실제 결과 확인

- `ros2 topic hz /scan`: 평균 **~6.49Hz**로 안정적으로 발행됨 (RPLIDAR A1 기본 회전속도 기준 정상 범위). (확인됨)
- `ros2 topic echo /scan --once`: `ranges` 배열에 `inf`(무반사/범위밖)뿐 아니라 실제 거리 숫자값들이 섞여서 나옴 — 주변 물체까지의 실측 거리. (확인됨)

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| `sudo apt update`가 `Conflicting values set for option Signed-By` 에러로 실패 | VS Code apt 저장소가 `vscode.list`(구 방식)와 `vscode.sources`(신 방식, 자동관리)로 중복 등록 | 구 방식 파일(`vscode.list`)만 백업 이름으로 이동, 자동관리되는 `vscode.sources`는 그대로 둠 (3.1절) |
| `ls /dev/ttyUSB*`가 아무것도 안 보여줌, `lsusb`에도 라이다가 안 뜸 | USB 케이블이 충전 전용(데이터선 없음) | `sudo dmesg -w`로 지켜보며 뽑았다 꽂아서 반응 없음을 확인 → 데이터 전송이 확실한 다른 케이블로 교체 (3.3절) |
| `sudo dmesg`가 `read kernel buffer failed: Operation not permitted` | 일반 `dmesg`는 커널 로그 접근 권한 제한(`dmesg_restrict`)에 걸림 | `sudo dmesg` 또는 `sudo dmesg -w`로 실행 |
| `sudo usermod -aG dialout $USER` 실행 후에도 `groups`에 `dialout`이 안 보임 | 그룹 변경은 그 시점 이후의 새 로그인 세션부터 적용됨, 이미 열린 셸엔 반영 안 됨 | `newgrp dialout`으로 즉시 적용 (완전한 방법은 로그아웃/재로그인) (3.4절) |
| `newgrp` 이후 새 셸에 `(base)`가 자동으로 뜸 | conda가 셸 시작 시 자동 활성화되도록 설정되어 있음 | ROS2 명령 실행 전 `conda deactivate` (이 프로젝트 전체에서 반복되는 표준 절차) |

## 6. 체크포인트

- [x] apt 저장소 충돌 해결, `sudo apt update` 정상화
- [x] `ros-jazzy-rplidar-ros` 설치
- [x] USB 인식 문제를 `lsusb`/`dmesg`로 직접 진단해서 원인(충전전용 케이블)을 찾음
- [x] `dialout` 그룹 설정, 시리얼 포트 접근 권한 확보
- [x] `rplidar.launch.py` 실행, `/scan` 토픽에서 ~6.5Hz 실데이터 확인

---
다음: `02-raspberry-pi-setup.md` (미작성) — 같은 파이프라인을 헤드리스 라즈베리파이로 이전. RViz 시각화(GLX/Wayland 문제)는 별도로 보류 중, `04-rviz-visualization.md`(미작성)에서 재도전 예정.
