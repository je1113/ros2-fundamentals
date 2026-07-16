# 04. 2D LiDAR & RGB-D 카메라 배치

> 진행 상태: **완료** (LiDAR, RGB-D 카메라 둘 다 ROS2 퍼블리시 확인)

## 1. 학습 목표

- 로봇 본체에 센서 마운트(Xform)를 만들고 RTX Lidar를 배치한다.
- OmniGraph(Action Graph)로 센서 데이터를 ROS2 토픽으로 퍼블리시하는 파이프라인(Render Product → Sensor Helper)을 구성한다.
- RTX Lidar의 2D/3D 프로파일 차이를 이해하고, 로봇청소기 SLAM에 맞는 2D 스캔 설정을 고른다.
- Isaac Sim 5.1.0에서 UI가 불안정하거나 메뉴를 찾기 어려울 때, 설치된 확장 소스를 직접 grep해서 정확한 노드/속성 이름을 확인하고 Script Editor(USD Python API, `omni.graph.core`)로 직접 구성하는 방법을 익힌다.

## 2. 핵심 개념

**Render Product 기반 센서 파이프라인**: RTX 계열 센서(Lidar, Camera)는 실제 렌더링 파이프라인을 통해 데이터를 생성한다. 그래서 항상 `Isaac Create Render Product`(센서 프림을 받아 렌더 프로덕트를 생성) → 센서별 ROS2 Helper 노드(그 렌더 프로덕트에서 데이터를 뽑아 ROS2로 퍼블리시) 순서로 그래프를 구성해야 한다.

**RTX Lidar의 2D vs 3D 프로파일**: RTX Lidar 프림은 `elevationDeg` 등 채널별 설정을 가진 특정 config(JSON 프로파일, 예: `Example_Rotary`, `Example_Rotary_2D`)를 기반으로 만들어진다. `sensor_msgs/LaserScan`(단일 평면 2D 스캔)으로 퍼블리시하려면 반드시 **모든 채널의 elevation이 0인 2D 프로파일**을 써야 한다 — 3D(다중 채널) 프로파일로는 LaserScan 변환 자체가 내부적으로 거부된다.

**Isaac Sim 5.1.0에서 막히면 소스를 직접 확인**: 이번 토픽에서 메뉴 탐색으로 못 찾은 것들(정확한 OmniGraph 노드 이름, Physics Material 위치 등)을 전부 `~/isaacsim_env/lib/python3.11/site-packages/isaacsim/exts/`, `.../extscache/` 밑의 `.ogn`/`.py` 소스를 grep해서 해결했다. 이 패턴이 이 프로젝트에서 이제 반복적으로 유효했다 — UI로 안 될 때는 설치된 확장 소스가 가장 정확한 1차 자료다.

## 3. 실습 단계

### 3.1 센서 마운트 생성

1. `robot1` 하위에 새 Xform 생성, 이름 `lidar_mount`
2. 로봇 상단 중앙으로 위치 이동 (`W`)

### 3.2 RTX Lidar 배치 (첫 시도 — 3D 프로파일로 실패)

`Create > Isaac > Sensors > RTX Lidar > Rotary`로 만들면 기본값이 **`Example_Rotary`(3D, 다중 채널)** 프로파일이라 LaserScan 변환이 실패한다 (5.2절 참고). 최종적으로는 Python API로 2D 프로파일을 직접 지정해 다시 만들었다:

```python
from isaacsim.sensors.rtx import LidarRtx
import numpy as np

lidar = LidarRtx(
    prim_path="/World/robotvacum_decimated/robot1/lidar_mount/Lidar_2D",
    name="lidar_2d",
    translation=np.array([0.0, 0.0, 0.0]),
    config_file_name="Example_Rotary_2D",
)
```

사용 가능한 2D 프로파일은 `isaacsim.sensors.rtx`의 `data/lidar_configs/` 밑에 실제 상용 2D Lidar 모델들(SICK, SLAMTEC RPLIDAR 등)도 있다 — 로봇청소기라면 실제로 `RPLIDAR_S2E` 같은 프로파일이 더 현실적인 대안이 될 수 있다 (다음에 시도해볼 만함).

### 3.3 Action Graph로 ROS2 퍼블리시 파이프라인 구성

GUI에서 `Isaac Create Render Product` 노드 추가 시 `Accessed invalid null prim` 에러로 반복 실패해서, Script Editor로 전체를 구성했다.

```python
import omni.graph.core as og

keys = og.Controller.Keys

og.Controller.edit(
    {"graph_path": "/World/SensorGraph", "evaluator_name": "execution"},
    {
        keys.CREATE_NODES: [
            ("OnPlaybackTick", "omni.graph.action.OnPlaybackTick"),
            ("RenderProduct", "isaacsim.core.nodes.IsaacCreateRenderProduct"),
            ("LidarHelper", "isaacsim.ros2.bridge.ROS2RtxLidarHelper"),
        ],
        keys.CONNECT: [
            ("OnPlaybackTick.outputs:tick", "RenderProduct.inputs:execIn"),
            ("RenderProduct.outputs:execOut", "LidarHelper.inputs:execIn"),
            ("RenderProduct.outputs:renderProductPath", "LidarHelper.inputs:renderProductPath"),
        ],
        keys.SET_VALUES: [
            ("RenderProduct.inputs:cameraPrim", "/World/robotvacum_decimated/robot1/lidar_mount/Lidar_2D"),
            ("LidarHelper.inputs:topicName", "scan"),
            ("LidarHelper.inputs:frameId", "sim_lidar"),
            ("LidarHelper.inputs:type", "laser_scan"),
        ],
    },
)
```

### 3.4 값만 바꿀 때는 `og.Controller.attribute()`로 직접 접근

이미 만들어진 그래프에 `og.Controller.edit({"graph_path": ...}, {SET_VALUES: [...]})`만 다시 호출하면 `Failed to create ComputeGraph ... A graph already exists at this path` 에러가 났다 (5.3절 참고). 값만 바꿀 때는 그래프 전체를 다시 감싸지 말고 속성에 직접 접근하는 게 안전하다:

```python
import omni.graph.core as og

attr = og.Controller.attribute("/World/SensorGraph/RenderProduct.inputs:cameraPrim")
og.Controller.set(attr, "/World/robotvacum_decimated/robot1/lidar_mount/Lidar_2D")
```

### 3.5 LiDAR 검증

```bash
conda deactivate
source /opt/ros/jazzy/setup.bash
ros2 topic list       # /scan이 보이는지
ros2 topic hz /scan   # 실제 데이터가 흐르는지
```

### 3.6 RGB-D 카메라 배치 (GUI로 성공)

LiDAR 때와 달리 이번엔 Action Graph 편집창에서 노드 추가가 **정상적으로 작동했다** — 3.3절의 `Accessed invalid null prim` 에러는 매번 재현되는 구조적 문제가 아니라 간헐적 글리치였던 것으로 보인다. GUI로 먼저 시도해보고 안 될 때만 스크립트로 전환하는 순서가 맞다.

1. `robot1` 하위에 `camera_mount` Xform 생성, 로봇 정면 근처에 배치
2. `Create > Camera`로 카메라 프림 생성, `camera_mount` 밑으로 이동
3. Action Graph에 노드 추가: `On Playback Tick` → `Isaac Create Render Product`(`Camera Prim` = 방금 만든 Camera) → **`ROS2 Camera Helper` 두 개** (하나는 RGB용, 하나는 Depth용)
4. `Isaac Create Render Product`의 `Exec Out`/`Render Product Path`를 두 Helper 노드 각각에 연결 (하나의 출력에서 여러 입력으로 분기 가능)
5. 각 Helper의 Property 설정:
   - RGB: `Topic Name = camera/rgb`, `Frame Id = sim_camera`, `Type = rgb`
   - Depth: `Topic Name = camera/depth`, `Frame Id = sim_camera`, `Type = depth`

### 3.7 검증

```bash
ros2 topic list | grep camera
ros2 topic hz /camera/rgb
ros2 topic hz /camera/depth
```

## 4. 예상/실제 결과 확인

- `ros2 topic list`에 `/scan`, `/camera/rgb`, `/camera/depth`가 모두 나타난다.
- 세 토픽 전부 `ros2 topic hz`로 정상적인 Hz 값이 출력된다 (LiDAR/카메라 둘 다 본 세션에서 최종 확인 완료).

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| Action Graph 편집창에서 `Isaac Create Render Product` 노드 추가 시 `Accessed invalid null prim` 에러, 노드가 실제로 추가되지 않음 | 간헐적 UI 글리치로 추정 — 카메라 노드 추가 때는 동일한 GUI 조작이 정상 작동했음 (재현성 없음) | 이번엔 `og.Controller.edit()`로 우회했지만, 다음에 같은 에러가 나면 먼저 재시도(같은 조작 다시 시도)해보고 그래도 안 되면 스크립트로 전환할 것 — 무조건 스크립트 우선일 필요는 없음 |
| `ros2 topic hz /scan`이 아무 출력 없음 (토픽은 `ros2 topic list`에 있지만 데이터가 안 흐름). 로그에 `IsaacComputeRTXLidarFlatScan: Failed to initialize correctly. Skipping execution.`가 매 프레임 반복 | Lidar 프림이 `Example_Rotary`(3D, `elevationDeg`에 -15도 등 0이 아닌 값 포함) 프로파일이라 LaserScan(FlatScan) 변환이 내부적으로 거부됨 — 로그에 `Lidar prim is not a 2D Lidar, and node will not execute`로 명시됨 | `isaacsim.sensors.rtx`의 `LidarRtx` 클래스로 `config_file_name="Example_Rotary_2D"`(모든 채널 elevation 0)를 사용해 Lidar를 재생성 |
| 기존 그래프에 `og.Controller.edit({"graph_path": "/World/SensorGraph"}, {SET_VALUES: [...]})`만 호출(CREATE_NODES 없이)하면 `Failed to create ComputeGraph "/World/SensorGraph". A graph already exists at this path` 에러 | `og.Controller.edit()`가 `evaluator_name` 등 원래 생성 시 키가 없으면 기존 그래프를 "감싸기"(wrap) 대신 새로 만들려고 시도하는 것으로 추정 (버전 특이 동작) | 그래프를 통째로 다시 `edit()`하지 말고, `og.Controller.attribute("<node_path>.inputs:<name>")` + `og.Controller.set(attr, value)`로 개별 속성에 직접 접근 |
| 정확한 OmniGraph 노드 타입 문자열(`isaacsim.core.nodes.IsaacCreateRenderProduct`, `isaacsim.ros2.bridge.ROS2RtxLidarHelper` 등)을 문서/기억만으로 확신할 수 없었음 | Isaac Sim 5.1.0 관련 최신 문서가 부족 | 설치된 확장의 `.ogn`/`*Database.py` 파일을 직접 grep해서 `get_node_type()`이 반환하는 정확한 문자열 확인 (`find ~/isaacsim_env -iname "*.ogn"` 또는 `*Database.py`로 검색) |

## 6. 체크포인트

- [x] `robot1` 하위에 `lidar_mount` Xform 생성, 로봇 상단에 배치
- [x] RTX Lidar를 **2D 프로파일**(`Example_Rotary_2D` 등)로 생성 — 3D 기본 프로파일 주의
- [x] Action Graph(`On Playback Tick` → `Isaac Create Render Product` → `ROS2 RTX Lidar Helper`) 구성
- [x] `topicName=scan`, `frameId=sim_lidar`, `type=laser_scan` 설정
- [x] Play 후 `ros2 topic list`에 `/scan` 확인, `ros2 topic hz /scan`으로 데이터 흐름 확인 완료
- [x] `robot1` 하위에 `camera_mount` Xform 생성, 로봇 정면에 배치, Camera 프림 생성
- [x] Action Graph에 RGB/Depth `ROS2 Camera Helper` 두 개 구성
- [x] `/camera/rgb`, `/camera/depth` 퍼블리시 확인 완료

---
Part B Topic 4 완료. 다음: Topic 5 — 특수 센서 배치 (Bumper & 추락방지, `05-bumper-cliff-sensors.md`, 미작성)
