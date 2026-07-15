# rosbag2 & 디버깅 툴 — 기록·재생·시각화·진단

[`00-syllabus.md`](./00-syllabus.md)의 12번째 주제. 새 노드를 작성하지 않고, 지금까지 만든 배터리 파이프라인을 ROS 2 도구들로 기록·재생·시각화·진단해본다.

---

## 학습 목표

- `ros2 bag record`/`play`로 토픽 데이터를 기록하고 재생할 수 있다.
- `rqt_graph`로 노드-토픽 연결 그래프를 시각적으로 확인할 수 있다.
- `ros2 doctor`, `ros2 topic hz/bw`로 ROS 2 환경과 통신 상태를 진단할 수 있다.

## 핵심 개념

- **rosbag2**: 토픽 데이터를 파일로 기록(`ros2 bag record`)하고 나중에 재생(`ros2 bag play`)할 수 있다 — 실제 로봇/센서 없이도 같은 데이터로 반복 테스트/디버깅이 가능하다.
- **rqt_graph**: 현재 실행 중인 노드-토픽 연결 그래프를 시각적으로 보여준다.
- **ros2 doctor**: ROS 2 환경(네트워크, 토픽 연결, QoS 등)의 이상 유무를 점검한다.
- **ros2 topic hz/bw**: 토픽의 발행 주기(Hz)와 대역폭(bandwidth)을 확인한다.

## 실습 단계

### 1. 실행 환경 준비

```bash
cd ~/ros2_ws
source install/setup.bash
ros2 launch ros2_basics bringup.launch.py
```

(10번 주제에서 만든 launch 파일 — publisher/watcher/monitor 3개 노드가 함께 뜬다)

### 2. rosbag2로 기록

```bash
cd ~/ros2_ws
ros2 bag record -o battery_bag /battery_state /battery_alert
```

10~15초 정도 기록한 뒤 `Ctrl+C`로 종료한다.

### 3. 기록된 bag 정보 확인

```bash
ros2 bag info battery_bag
```

### 4. 재생 — 노드 없이도 데이터가 흐르는지 확인

`bringup.launch.py`를 종료한 뒤(발행자 없는 상태):

```bash
ros2 topic echo /battery_state
```

다른 터미널에서:

```bash
ros2 bag play battery_bag
```

### 5. rqt_graph로 노드/토픽 그래프 시각화

`bringup.launch.py`를 다시 실행한 상태에서:

```bash
rqt_graph
```

### 6. ros2 doctor로 환경 점검

```bash
ros2 doctor
```

### 7. 토픽 주기/대역폭 확인

```bash
ros2 topic hz /battery_state
ros2 topic bw /battery_state
```

## 예상/실제 결과

- 3단계: `ros2 bag info`에 토픽별 메시지 개수, 타입, 기록 시간(duration)이 표시됨.
- 4단계: 원본 노드가 꺼진 상태에서도 `ros2 bag play`로 기록된 배터리 메시지가 `echo`에 그대로 재생됨.
- 5단계: `rqt_graph`에 `battery_publisher → battery_watcher → fleet_monitor` 화살표와 `request_return_to_base` 서비스 연결이 시각적으로 표시됨.
- 6단계: `ros2 doctor`에서 별도 경고 없이 정상.
- 7단계: `battery_publisher`가 1초 주기로 발행하므로 `ros2 topic hz`가 약 1Hz로 표시됨.

실제로 2~7단계 모두 확인했다.

## 알려진 문제와 해결

이번 실습에서는 별도로 발생한 문제 없음.

## 체크포인트

- [ ] `ros2 bag record`/`play`로 토픽 데이터를 기록·재생할 수 있다.
- [ ] `rqt_graph`로 노드-토픽 연결 구조를 눈으로 확인할 수 있다.
- [ ] `ros2 doctor`로 환경 이상 유무를 점검할 수 있다.
- [ ] `ros2 topic hz`/`bw`로 토픽의 주기와 대역폭을 확인할 수 있다.

---
다음: [`13-ros2-control.md`](./13-ros2-control.md)
