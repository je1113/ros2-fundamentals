# 08. 격자 지도(Grid Map) 빌드

> 진행 상태: **완료** — `/clock` 미발행, TF 타임스탬프 0 고정, 씬에 장애물 없음, 라이다 자기충돌(수평 360도) 등 여러 문제를 순서대로 해결하고 실제 지도 저장까지 확인

## 1. 학습 목표

- `slam_toolbox`의 `online_async` 모드로 `/scan` 데이터를 실시간 점유 격자 지도(Occupancy Grid)로 변환한다.
- RViz2로 지도가 실시간으로 채워지는 과정을 시각적으로 확인한다.
- `map_saver_cli`로 완성된 지도를 `.yaml`+`.pgm`으로 저장한다 (Part D의 Nav2 커버리지 경로 계획에서 재사용).

## 2. 핵심 개념

**Occupancy Grid**: 공간을 일정 해상도(여기선 5cm)의 격자로 나누고, 각 셀을 "비어있음/점유됨/미확인"으로 표시한 2D 지도. `slam_toolbox`가 `/scan` 데이터를 스캔 매칭(scan matching)으로 누적하며 이 격자를 계속 업데이트한다.

**왜 `online_async`인가**: `slam_toolbox`는 동기(sync)/비동기(async) 두 모드가 있다. `sync`는 모든 스캔을 순서대로 빠짐없이 처리하지만 처리 속도가 스캔 속도를 못 따라가면 지연이 쌓인다. `async`는 처리 중 새 스캔이 오면 오래된 것을 건너뛰어 실시간성을 우선한다 — teleop로 라이브 주행하며 지도를 만드는 이 실습에 적합.

**TF 프레임 요구사항과 이 프로젝트의 변형**: `slam_toolbox`는 표준적으로 `map`(자신이 발행) → `odom` → `base_frame` → 스캔의 `frame_id` 순으로 이어지는 TF 체인을 요구한다. 이 프로젝트는 [[isaacsim_ros2_advanced_curriculum]]에서 이미 확인했듯 **진짜 바퀴 오도메트리가 없다** — `OdomCompute`가 매 프레임 USD의 그라운드트루스 트랜스폼을 그대로 읽어서 쓰는 구조라, `/odom` 토픽은 있지만 `odom`/`base_link`라는 이름의 TF 프레임은 실제로 발행되지 않는다 (`/tf`에는 `world → robot1/Lidar_2D/Camera/...`만 존재 — 3.1절에서 직접 확인). 그래서 이 실습에서는 `odom_frame`/`base_frame`을 표준 이름 대신 **실제 발행되는 이름**(`world`/`robot1`)으로 그대로 설정한다 — 이름만 다를 뿐 "누적 오차 없는 오도메트리"라는 점에서 오히려 더 정직한 설정.

**RViz Map 디스플레이와 QoS**: `/map` 토픽은 보통 Transient Local(구독 시점 이전에 발행된 마지막 메시지도 받는) durability로 발행된다 — 그래야 RViz를 지도가 다 만들어진 후에 켜도 최신 지도를 바로 받을 수 있다. RViz2의 Map 디스플레이는 기본으로 이 QoS를 맞춰서 구독하므로 보통 수동 설정이 필요 없다.

## 3. 실습 단계

### 3.1 TF 프레임 확인 (완료)

Play 상태에서 `/tf`를 직접 구독해 실제 발행되는 프레임 쌍을 확인:

```bash
conda deactivate
source /opt/ros/jazzy/setup.bash
/usr/bin/python3 -c "
import rclpy
from rclpy.node import Node
from tf2_msgs.msg import TFMessage
import time

pairs = set()

class L(Node):
    def __init__(self):
        super().__init__('tf_lister')
        self.create_subscription(TFMessage, '/tf', self.cb, 10)
    def cb(self, msg):
        for t in msg.transforms:
            pairs.add((t.header.frame_id, t.child_frame_id))

rclpy.init()
n = L()
end = time.time() + 3
while time.time() < end:
    rclpy.spin_once(n, timeout_sec=0.1)
print(pairs)
"
```

결과: `{('world','robot1'), ('world','Lidar_2D'), ('world','Camera'), ('world','bumper_mount'), ('world','cliff_mount')}` — `odom`/`base_link` 없음, 확인대로 2.의 "핵심 개념"에서 설명한 변형이 필요함.

### 3.2 slam_toolbox 파라미터 준비 (완료)

`~/isaac_assets/vacuum_robot/config/slam_toolbox_params.yaml` — 기본 `mapper_params_online_async.yaml`에서 `odom_frame: world`, `base_frame: robot1`로 바꾸고, `min_laser_range`/`max_laser_range`를 `ScanCompute`의 레이캐스트 범위(`0.05`~`12.0`)에 맞춤, `minimum_travel_distance`를 로봇 크기(~0.35m)에 맞춰 `0.5`→`0.2`로 낮춤.

### 3.3 slam_toolbox 실행

Isaac Sim이 Play 상태인 것을 확인한 뒤, 새 터미널에서:

```bash
conda deactivate
source /opt/ros/jazzy/setup.bash
ros2 launch slam_toolbox online_async_launch.py \
  slam_params_file:=/home/pw/isaac_assets/vacuum_robot/config/slam_toolbox_params.yaml
```

### 3.4 RViz2로 시각화

또 다른 터미널에서:

```bash
conda deactivate
source /opt/ros/jazzy/setup.bash
rviz2
```

RViz에서:
1. `Fixed Frame`을 `map`으로 설정
2. `Add` → `By display type` → `Map` 추가, Topic을 `/map`으로 설정
3. `Add` → `By display type` → `LaserScan` 추가, Topic을 `/scan`으로 설정
4. `Add` → `By display type` → `TF` 추가 (프레임 관계 확인용)

### 3.5 teleop로 주행하며 지도 빌드

또 다른 터미널에서 (Topic 7과 동일):

```bash
conda deactivate
source /opt/ros/jazzy/setup.bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

RViz를 보면서 로봇을 이리저리 몰아 빈 공간을 채운다. 같은 구역을 다시 지나가면 루프 클로저로 지도가 보정되는지도 관찰.

### 3.6 지도 저장

원하는 만큼 지도가 만들어지면 (slam_toolbox와 Isaac Sim은 계속 실행한 채로) 새 터미널에서:

```bash
conda deactivate
source /opt/ros/jazzy/setup.bash
ros2 run nav2_map_server map_saver_cli -f ~/isaac_assets/vacuum_robot/maps/robotvacum_map
```

`robotvacum_map.yaml` + `robotvacum_map.pgm`이 생성되면 성공 — Part D의 Nav2 코스트맵/경로계획에서 이 지도를 불러와 쓴다.

### 3.7 `/clock` 미발행 발견 및 수정

slam_toolbox를 처음 실행했을 때 `/scan`/`/tf` 둘 다 정상 발행 중인데도 `/map`/`/map_metadata`가 몇 분이 지나도 전혀 나오지 않았다. `ros2 topic info /clock --verbose`로 확인해보니 **`Publisher count: 0`** — `use_sim_time: true`로 도는 slam_toolbox의 내부 시계가 아예 흐르지 않고 있었다. Isaac Sim의 ROS2 브릿지는 그래프에 명시적으로 `ROS2 Publish Clock` 노드를 넣어야만 `/clock`을 발행한다 (자동으로 켜지지 않음) — 지금까지 이 그래프를 쓰는 어떤 토픽도 `use_sim_time` 동기화가 필요 없었어서 놓치고 있었던 노드.

**해결**: `/World/SensorGraph`에 `ROS2 Publish Clock`(`isaacsim.ros2.bridge.ROS2PublishClock`) 노드를 GUI로 추가 — `On Playback Tick.Tick` → `Exec In`, 기존 `SimTime`(`Isaac Read Simulation Time`) → `Time Stamp`. **처음엔 `Exec In` 연결을 빠뜨려서 여전히 발행이 안 됐다** — Exec 연결 없는 노드는 아예 실행되지 않는다는 걸 다시 확인.

### 3.8 TF 타임스탬프가 항상 0으로 찍히는 문제

`/clock`을 고친 뒤에도 slam_toolbox 로그에 `Detected jump back in time. Clearing TF buffer.` / `Message Filter dropping message ... the timestamp on the message is earlier than all the data in the transform cache`가 계속 떴다. `/tf`를 직접 찍어보니 `world→Lidar_2D`의 `stamp`가 **항상 `sec: 0, nanosec: 0`** — `CmdVelGraph`의 `TfPublish`(`ROS2PublishTransformTree`)와 `OdomPublish` 둘 다 `inputs:timeStamp`가 애초에 아무 데도 연결되어 있지 않았다 (Topic 6에서 만들 때부터 빠져있었음 — `/scan`/`/camera`는 Helper 노드가 렌더 파이프라인에서 자동으로 타임스탬프를 넣어주지만, `TfPublish`/`OdomPublish`는 그런 자동 소스가 없어서 명시적으로 연결해야 했다). Isaac Sim 자체도 `Frame world Timestamp is invalid. Timestamp 0 will be neglected for all published ROS messages` 경고를 띄우고 있었다.

**해결**: `CmdVelGraph`에 `SimTime`(`Isaac Read Simulation Time`) 노드를 추가하고 `OdomPublish.inputs:timeStamp`/`TfPublish.inputs:timeStamp`에 연결. 기존 그래프에 노드 하나 추가하는 부분 수정은 이 프로젝트에서 계속 실패했던 패턴이라([[isaacsim_ros2_advanced_curriculum]]), `CmdVelGraph` 전체를 `og.Controller.edit()` 한 번으로 통째로 재생성.

### 3.9 씬에 라이다가 맞힐 게 없음 — 방(벽) 추가

TF/시간 문제를 다 고쳐도 `/map_metadata`가 `width: 0, height: 0`으로 계속 비어있었다. `/scan`을 직접 찍어보니 **360개 광선 전부 `inf`**(무한대, 감지 안 됨) — 원인은 버그가 아니라 **씬에 라이다가 감지할 벽/장애물이 아예 없었던 것**. 스테이지엔 로봇+충전독+수평 GroundPlane뿐인데, 수평으로 나가는 광선은 수평 바닥면을 절대 못 맞힌다. 로봇청소기 시뮬레이션인데 정작 청소할 방이 없었던 셈.

**해결**: `/World/Room`에 벽 4개(약 6m×5m, 높이 1.2m, `UsdPhysics.CollisionAPI`만 적용한 정적 Cube)를 스크립트로 배치. `CreateMeshPrimWithDefaultXform`로 만든 프림에 `ClearXformOpOrder()` 후 새 xformOp을 추가할 때 **기존 `xformOp:scale`의 정밀도(`double3`)와 안 맞으면 `AddXformOp` 에러** — `AddScaleOp(precision=UsdGeom.XformOp.PrecisionDouble)`로 명시해서 해결 (Topic 6에서 이미 겪은 xformOp precision 불일치 문제의 재발).

### 3.10 라이다 자기충돌 — 360도 전 방향에서 로봇 자신에 막힘

벽을 만든 뒤에도 여전히 `finite hits: 0`. `raycast_closest`로 라이다 위치에서 동서남북 4방향을 직접 쏴보니 **전부 거리 0.0으로 로봇 자신의 `Object_139`(곡면 쉘 서브메시)에 맞았다** — Topic 5의 절벽 센서에서 겪었던 "Convex Hull이 곡면 쉘 내부를 채워서 로봇 풋프린트 전체가 꽉 찬 판때기가 되는" 문제가, 이번엔 아래쪽 한 방향이 아니라 **수평 360도 전 방향**에서 터진 것. `ScanCompute`가 이 자기충돌을 "감지 안 됨"으로 걸러내다 보니, 진짜 벽까지 도달하기도 전에 매번 필터링당해 아무것도 못 맞히고 있었다.

`raycast_closest`는 가장 가까운 히트 하나만 주기 때문에 자기 몸체를 "지나서" 다시 쏠 방법이 없다. **해결**: `get_physx_scene_query_interface().raycast_all(origin, dir, max_distance, report_fn, bothSides=True)`로 전환 — 콜백(`report_fn`)이 광선 경로상의 **모든 히트**를 보고해준다. 매 광선마다 모든 히트를 모은 뒤 `robot1` 하위 경로가 아닌 히트들 중 최솟값을 진짜 거리로 사용. `bothSides=False`(기본값)로는 라이다 원점이 이미 로봇 콜라이더 내부에 있는 경우 히트 자체가 하나도 안 잡혀서 `bothSides=True`가 필수였다.

### 3.11 ScriptNode가 바뀐 스크립트를 실시간으로 안 읽음

`ScanCompute`의 `inputs:script` 속성을 `raycast_all` 버전으로 바꾼 뒤(`prim.GetAttribute("inputs:script").Set(...)`으로 확인상 정상 반영됨, Stop→Play도 해봄) `ros2 topic hz`/직접 구독으로 확인해도 여전히 `finite hits: 0` — 같은 로직을 그래프 밖에서 스탠드얼론으로 돌리면 360/360 정상 히트. 즉 **USD 속성은 바뀌었는데 실행 중인 ScriptNode는 옛날에 컴파일된 `compute()`를 계속 쓰고 있었다** (Stop/Play로도 재컴파일 안 됨). **해결**: `SensorGraph`를 통째로 삭제하고 새 스크립트 내용을 `CREATE_NODES`/`SET_VALUES` 안에 처음부터 박아서 재생성 — 노드가 생성되는 시점에 스크립트가 이미 최종본이면 문제없이 동작. **교훈**: ScriptNode의 `inputs:script`를 이미 만들어진 노드에서 바꾸는 건 신뢰할 수 없다 — 스크립트를 고칠 땐 값만 갱신하지 말고 처음부터 노드를 다시 만들 것.

### 3.12 세션 중 Isaac Sim 종료로 미저장 작업 유실

이 세션 도중 Isaac Sim이 (원인 불명으로) 한 번 종료되면서, 그 시점까지 저장하지 않았던 `CmdVelGraph`의 `SimTime` 연결과 `/World/Room` 벽이 전부 날아갔다 (반면 더 일찍 만든 `SensorGraph`는 어느 시점엔가 저장이 된 상태라 남아있었음). 재시작 후 두 스크립트(`rebuild_cmdvelgraph.py`, `build_room.py`)를 다시 실행해서 복구. **교훈**: OmniGraph를 스크립트로 수정한 뒤에는 다음 단계로 넘어가기 전에 `File > Save`(Ctrl+S)를 습관적으로 눌러둘 것 — Stop/Play는 저장이 아니다.

## 4. 예상/실제 결과 확인

위 3.7~3.11의 문제를 순서대로 해결한 뒤: `/scan`이 360개 광선 전부 유효한 거리(약 0.12m~6.4m, 방 크기와 로봇 위치에 맞음)를 반환, `/clock`이 약 19Hz로 정상 발행, `/tf`의 모든 프레임이 실제 시뮬레이션 시간으로 타임스탬프됨. slam_toolbox 재시작 후 teleop로 방을 돌아다니자 `/map_metadata`가 `width: 122, height: 102`(≈6.1m×5.1m, 5cm 해상도)로 채워짐 — 방 크기와 일치. `map_saver_cli`로 `robotvacum_map.yaml`+`robotvacum_map.pgm`을 `~/isaac_assets/vacuum_robot/maps/`에 저장 완료.

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| slam_toolbox를 기본 파라미터(`odom_frame: odom`, `base_frame: base_footprint`)로 실행하면 지도가 전혀 안 만들어짐 | 이 프로젝트의 `/tf`에는 `odom`/`base_link`/`base_footprint` 프레임이 존재하지 않음 (3.1절) | `odom_frame: world`, `base_frame: robot1`로 실제 발행되는 프레임 이름에 맞춤 (3.2절) |
| `/scan`/`/tf`가 정상인데 `/map`이 몇 분째 안 나옴, 에러도 없음 | `/clock`에 Publisher가 0개 — `use_sim_time` 노드들의 내부 시계가 정지 | `ROS2 Publish Clock` 노드를 그래프에 추가 (Exec In 연결 빠뜨리지 않도록 주의) (3.7절) |
| `/clock` 수정 후에도 `Message Filter dropping message`/`jump back in time` | `TfPublish`/`OdomPublish`의 `inputs:timeStamp`가 애초에 미연결이라 항상 0으로 찍힘 | `CmdVelGraph`에 `SimTime` 노드 추가, 통째로 재생성 (3.8절) |
| TF/시간 다 고쳐도 `/map_metadata`가 계속 `width:0, height:0` | 씬에 수평 라이다가 맞힐 벽/장애물이 없음 (GroundPlane은 수평이라 수평 광선이 못 맞힘) | 정적 Cube 4개로 방 벽 추가 (3.9절) |
| 벽을 만들어도 여전히 `/scan` 전부 `inf` | 라이다 원점이 로봇 자신의 Convex Hull(곡면 쉘이 부풀어 오른 것) 내부에 있어서 360도 전 방향이 자기충돌로 필터링됨 | `raycast_closest` → `raycast_all(bothSides=True)`로 전환, 로봇 아닌 가장 가까운 히트 사용 (3.10절) |
| 스크립트를 새로 `.Set()`해도(Stop/Play 후에도) `/scan`이 안 바뀜 | ScriptNode가 컴파일된 이전 `compute()`를 캐싱하고 있어서 속성 변경이 반영 안 됨 | 노드를 값만 갱신하지 말고 `SensorGraph` 전체를 새 스크립트로 재생성 (3.11절) |
| Isaac Sim이 세션 중 종료되면서 미저장 그래프/씬 수정이 유실 | Stop/Play는 저장이 아님, `File > Save`를 누른 적 없었음 | 스크립트 실행 직후 습관적으로 Ctrl+S (3.12절) |

## 6. 체크포인트

- [x] `/tf` 실제 프레임 구성 확인, slam_toolbox 파라미터를 여기 맞춤
- [x] `/clock`/TF 타임스탬프/씬 장애물/라이다 자기충돌 문제 모두 해결
- [x] slam_toolbox 실행, `/map_metadata`가 방 크기(122×102, 5cm 해상도)로 채워지는 것 확인
- [x] `map_saver_cli`로 `~/isaac_assets/vacuum_robot/maps/robotvacum_map.yaml`+`.pgm` 저장 확인

---
다음: Topic 9 (Part D, 커버리지 경로 계획) — 이번에 저장한 지도를 Nav2 코스트맵으로 불러와서 사용
