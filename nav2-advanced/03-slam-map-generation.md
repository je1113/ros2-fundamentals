# 03. SLAM으로 지도 생성

## 1. 학습 목표

- 지도가 없는 상태에서 `slam_toolbox`로 실시간 지도를 만드는 흐름을 이해한다.
- 토픽 1의 map+AMCL 방식과 SLAM 방식의 차이(누가 `map→odom` TF를 발행하는가)를 구분한다.
- teleop으로 로봇을 직접 조종해 지도를 채우고, `map_saver_cli`로 지도를 파일로 저장한다.

## 2. 핵심 개념

- 토픽 1에서는 **이미 있는 지도(`map_server`) + `amcl`**로 로봇 위치를 추정했다. SLAM은 반대로 **지도 자체가 없는 상태에서 라이다 스캔을 누적해 지도를 만들면서, 동시에 자기 위치도 추정**한다.
- `slam_toolbox`가 `map_server`+`amcl` 두 역할을 겸한다 — 지도를 갱신하면서 `map→odom` TF도 직접 발행한다.
- `slam:=True`로 실행하면 `lifecycle_manager_localization`이 `map_server`/`amcl` 대신 `slam_toolbox`를 관리한다.
- 지도가 완성되면 `nav2_map_server`의 `map_saver_cli`로 `.pgm`(점유격자 이미지) + `.yaml`(해상도/원점/임계값 메타데이터)로 저장해서, 이후 토픽 1처럼 정적 지도로 재사용할 수 있다.

## 3. 실습 단계

### 3.1 SLAM 모드로 실행

토픽 1에서 띄운 map+AMCL 방식 launch를 먼저 종료(`Ctrl+C`)한 뒤:

```bash
ros2 launch nav2_bringup tb3_simulation_launch.py slam:=True
```

`ros2 node list | grep -E "slam_toolbox|map_server|amcl"`로 `map_server`/`amcl` 대신 `slam_toolbox`가 떠 있는지 확인한다.

### 3.2 teleop으로 로봇 조종하며 지도 채우기

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

RViz의 Map 디스플레이가 로봇이 이동하는 대로 채워지는지 확인하면서, 직진↔회전을 번갈아 눈에 보이는 구역을 한 바퀴 닫힌 루프로 돈다. 전체 월드를 다 덮을 필요는 없다 — 워크플로우를 익히는 게 목적이므로 방/복도 한 구역 정도면 충분하다.

### 3.3 지도 저장

```bash
mkdir -p ~/ros2_ws/maps
ros2 run nav2_map_server map_saver_cli -f ~/ros2_ws/maps/tb3_sandbox_map
```

## 4. 예상/실제 결과 확인

- `~/ros2_ws/maps/`에 `tb3_sandbox_map.pgm`, `tb3_sandbox_map.yaml`이 생성됨.
- `.yaml`에 `resolution`(기본 0.05m/px), `origin`(지도 좌하단의 world 좌표), `occupied_thresh`/`free_thresh` 등이 담겨 있다.

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| `teleop_keyboard`(turtlebot3_teleop)로 조종해도 로봇이 전혀 안 움직임 (Nav2 Goal로는 잘 움직임) | `turtlebot3_teleop`의 `teleop_keyboard.py`가 `geometry_msgs/msg/TwistStamped`를 발행하도록 하드코딩돼 있는데, 실제 로봇을 구동하는 `ros_gz_bridge`는 `geometry_msgs/msg/Twist`만 구독함. DDS는 같은 토픽이라도 타입이 다르면 매칭 자체가 안 돼서 메시지가 조용히 무시됨. `ros2 topic info /cmd_vel -v`로 publisher 타입이 `TwistStamped`인 것과 subscriber(`ros_gz_bridge`)가 `Twist`인 것을 직접 대조해서 확인 | `teleop_twist_keyboard` 패키지(`ros2 run teleop_twist_keyboard teleop_twist_keyboard`)를 대신 사용 — 이건 기본적으로 `Twist`(unstamped)를 발행해서 브릿지와 타입이 맞음 |
| `a`/`d`(회전)를 여러 번 눌렀더니 로봇이 앞으로 안 가고 제자리에서 빙빙 돎 | `turtlebot3_teleop`은 키를 누를 때마다 속도가 계속 누적됨. linear/angular가 둘 다 최대치(예: burger 기준 0.22 / 2.84)에 걸리면 회전반경이 `v/ω ≈ 8cm`로 매우 작아져 사실상 제자리 회전이 됨 | `space`나 `s`로 완전히 정지시켜 속도를 0으로 리셋한 뒤, 전진은 `w`(또는 `teleop_twist_keyboard`의 `i`)만 연타하고 방향 전환 시에만 회전 키를 짧게 눌러 "직진→살짝 회전→직진"을 반복 |
| RViz 지도가 아무리 돌아다녀도 안 끝나는 것처럼 보임 | `nav2_bringup`의 기본 TurtleBot3 시뮬레이션 월드가 예전의 작은 `turtlebot3_world`가 아니라 `tb3_sandbox.sdf.xacro`라는 훨씬 큰 샌드박스 맵으로 바뀜 (실제로 저장된 지도의 `origin`이 `[-17.971, -23.952]`로 origin에서 멀리 떨어져 있는 것으로 확인) | 전체 월드를 다 매핑할 필요 없음 — 눈에 보이는 한 구역만 닫힌 루프로 돌고 저장하면 이번 실습 목적(SLAM→저장 워크플로우)엔 충분 |

## 6. 체크포인트

- [ ] `slam:=True`로 Nav2 실행, `slam_toolbox` 노드 확인
- [ ] `teleop_twist_keyboard`로 로봇을 조종하며 지도가 실시간으로 채워지는 것 확인
- [ ] 한 구역을 닫힌 루프로 주행
- [ ] `map_saver_cli`로 `.pgm`/`.yaml` 저장 확인

---
다음: `04-costmap-planner-controller-plugins.md` (미작성)
