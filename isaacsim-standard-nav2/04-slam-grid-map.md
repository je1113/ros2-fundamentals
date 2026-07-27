# 04. SLAM 격자 지도(Grid Map) 빌드

> 진행 상태: **완료** — 씬에 방(벽)이 없던 문제, `base_link`→`base_scan` 정적 TF 누락 두 가지를 해결하고 실제 지도 저장까지 확인

## 1. 학습 목표

- `slam_toolbox`의 `online_async` 모드로 `/scan` 데이터를 실시간 점유 격자 지도(Occupancy Grid)로 변환한다.
- 이 트랙은 Topic 3에서 이미 진짜 `odom`→`base_link` TF와 실제 오도메트리를 갖추고 있으므로, `isaacsim-ros2-advanced`(vacuum) 프로젝트에서 필요했던 `odom_frame`/`base_frame` 이름 대체 없이 표준 파라미터 그대로 SLAM을 붙인다.
- `map_saver_cli`로 완성된 지도를 `.yaml`+`.pgm`으로 저장한다 (Topic 5의 Nav2 목표 주행 검증에서 재사용).

## 2. 핵심 개념

**이 트랙이 vacuum 프로젝트보다 SLAM 설정이 간단한 이유**: `isaacsim-ros2-advanced`는 진짜 바퀴 오도메트리가 없어서 `odom`/`base_link` TF 프레임 자체가 발행되지 않았고, `odom_frame: world`, `base_frame: robot1`처럼 실제 발행되는 이름으로 대체해야 했다([[isaacsim_ros2_advanced_curriculum]] 08번 문서 참고). Carter는 Topic 3-B에서 진짜 `Isaac Compute Odometry Node` + `ROS2 Publish Raw Transform Tree`로 `odom`→`base_link`를 정상 발행하므로, slam_toolbox 기본 파라미터의 `odom_frame: odom` / `base_frame: base_link`를 그대로 쓸 수 있다.

**그런데 TF 체인이 하나 더 필요하다**: slam_toolbox는 `map → odom → base_frame → (스캔의 frame_id)`까지 이어지는 TF를 요구한다. `base_frame`(`base_link`)과 스캔 메시지의 `frame_id`(`base_scan`)가 다른 이름이면, 그 사이를 잇는 변환이 어딘가에는 있어야 한다. Topic 3-C에서 `ROS2 RTX Lidar Helper`의 `Frame Id`를 `base_scan`으로 정했을 때 "아직 `base_link`→`base_scan` static TF는 없지만 지금은 `/scan` 데이터 흐름 확인이 목적"이라고 미뤄뒀던 게 바로 이 지점이다(3-C, 후속 확인 항목).

**Occupancy Grid PGM 값 읽는 법**: `map_saver_cli`가 저장하는 `.pgm`은 픽셀값 `254`=빈 공간(흰색), `0`=점유/장애물(검은색), `205`=미확인(회색) 3가지로 구성된다. 저장 직후 `PIL`/`numpy`로 `np.unique(arr, return_counts=True)`를 찍어보면 실제로 벽이 잡혔는지 코드로 바로 확인할 수 있다 — RViz를 안 띄워도 되는 빠른 검증 방법.

## 3. 실습 단계

### 3.1 씬에 매핑할 방 만들기

Topic 3-C 검증 때 썼던 테스트 벽(`/World/TestWall`)은 진단 후 삭제했으므로, 씬에는 로봇과 평평한 `GroundPlane`만 남아있었다 — 2D LiDAR가 수평면만 스캔하는 한 다시 지도가 안 만들어질 상황. `/World/Room` 아래 벽 4개(6m×5m 방, 두께 0.1m, 높이 1.2m — 라이다 마운트 높이 `z=0.99`를 여유 있게 덮음)를 스크립트로 배치했다.

```python
def make_wall(path, center, size):
    cube = UsdGeom.Cube.Define(stage, path)
    cube.CreateSizeAttr(1.0)  # 1m 기준 큐브 — scale이 곧 미터 단위가 되어 계산이 직관적
    prim = cube.GetPrim()
    xform_api = UsdGeom.XformCommonAPI(prim)
    xform_api.SetScale(Gf.Vec3f(*size))
    xform_api.SetTranslate(Gf.Vec3d(*center))
    UsdPhysics.CollisionAPI.Apply(prim)
```

**Topic 3-C의 `CreateMeshPrimWithDefaultXform` 큐브 크기 버그를 재발하지 않도록**, 이번엔 `UsdGeom.Cube.Define()` + `CreateSizeAttr(1.0)`로 기준 크기를 직접 명시했다 — `scale`이 그대로 미터 단위가 되므로 커맨드의 알 수 없는 기본값에 의존하지 않는다. 배치 후 `UsdGeom.BBoxCache.ComputeWorldBound()`로 4개 벽의 실제 bbox를 재확인했더니 의도한 좌표와 정확히 일치했다(3.12절에서 배운 "실측 없이 스크립트 배치값을 믿지 말 것"을 그대로 적용).

### 3.2 slam_toolbox 파라미터 준비

`~/isaac_assets/carter_standard/config/slam_toolbox_params.yaml` — `/opt/ros/jazzy/share/slam_toolbox/config/mapper_params_online_async.yaml`을 베이스로:
- `odom_frame: odom`, `base_frame: base_link` (실제 발행되는 이름 그대로, 대체 불필요)
- `scan_topic: /scan`
- `min_laser_range: 0.05`, `max_laser_range: 30.0` (RPLIDAR_S2E의 실제 range와 일치)
- `minimum_travel_distance/heading: 0.3` (기본 `0.5`보다 낮춤 — 6m×5m의 작은 방에서 더 촘촘하게 스캔 매칭)

### 3.3 slam_toolbox 실행

```bash
conda deactivate
source /opt/ros/jazzy/setup.bash
ros2 launch slam_toolbox online_async_launch.py \
  slam_params_file:=/home/pw/isaac_assets/carter_standard/config/slam_toolbox_params.yaml \
  use_sim_time:=true
```

### 3.4 `base_link → base_scan` 정적 TF 발행 (누락 발견 및 해결)

실행 직후 로그에 `Message Filter dropping message: frame 'base_scan' ... discarding message because the queue is full`가 반복됐다 — `base_link`(설정된 `base_frame`)에서 `base_scan`(스캔 메시지의 `frame_id`)까지 이어지는 변환이 TF 트리에 없어서, message_filter가 스캔 메시지를 계속 대기시키다 큐가 차서 버리고 있었다. Topic 3-C에서 `Frame Id`를 `base_scan`으로만 정하고 실제 위치 변환은 미뤄뒀던 부분이다.

**해결**: 라이다의 실제 마운트 오프셋(`chassis_link` 기준 `(-0.06, 0, 0.75)`, Topic 3-C 3.12절에서 확정한 값)을 그대로 정적 변환으로 발행.

```bash
conda deactivate
source /opt/ros/jazzy/setup.bash
ros2 run tf2_ros static_transform_publisher \
  --x -0.06 --y 0 --z 0.75 \
  --frame-id base_link --child-frame-id base_scan
```

발행 직후 `Message Filter dropping` 로그가 멈추고 `/map_metadata`가 방 크기에 맞게 채워지기 시작했다.

### 3.5 로봇을 방 안에서 주행시켜 지도 채우기

`teleop_twist_keyboard`로 직접 몰아도 되지만, 이 세션에서는 `/cmd_vel`에 일정 시간 간격으로 전진/회전 명령을 순서대로 퍼블리시하는 파이썬 스크립트로 방 둘레를 도는 사각형 순찰 경로를 재현했다(전진 5~6초 → 제자리 회전 ~90도 × 4회 반복).

### 3.6 지도 저장

```bash
conda deactivate
source /opt/ros/jazzy/setup.bash
ros2 run nav2_map_server map_saver_cli -f ~/isaac_assets/carter_standard/maps/carter_room_map
```

`carter_room_map.yaml` + `carter_room_map.pgm` (120×100, 5cm 해상도 ≈ 6m×5m, 방 크기와 일치) 저장 확인.

## 4. 예상/실제 결과 확인

- `/map_metadata`가 방을 만들기 전에는 존재하지 않다가, 방 생성 후 `width: 119~120, height: 99~100`(≈6m×5m, 5cm 해상도)로 채워져야 한다. (확인됨)
- 정적 TF(`base_link`→`base_scan`) 발행 전에는 `Message Filter dropping` 경고가 반복되고, 발행 후에는 멈춰야 한다. (확인됨)
- 순찰 주행 전후로 지도 크기(`width`/`height`)가 방 전체 크기에 수렴해야 한다 — 순찰 전 첫 스캔만으로도 이미 거의 전체 크기가 나왔고, 순찰 후 `120×100`으로 안정됨. (확인됨)
- 저장된 `.pgm`을 `numpy`로 픽셀 분포 확인 시 점유(0)/미확인(205)/빈 공간(254) 3가지 값이 실제로 섞여 있어야 한다 — 점유 536, 미확인 178, 빈 공간 11286 픽셀로 확인. (확인됨)

## 5. 알려진 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| slam_toolbox 실행 직후 `Message Filter dropping message: frame 'base_scan' ... queue is full`가 계속 반복, `/map_metadata`가 안 채워짐 | `base_frame`(`base_link`)에서 스캔의 `frame_id`(`base_scan`)로 이어지는 TF가 TF 트리에 없음 — Topic 3-C에서 의도적으로 미뤄둔 부분 | `tf2_ros static_transform_publisher`로 라이다의 실제 마운트 오프셋 `(-0.06, 0, 0.75)`을 `base_link`→`base_scan` 정적 변환으로 발행 (3.4절) |
| 씬에 라이다가 맞힐 벽/장애물이 없어서 지도가 안 만들어짐 | Topic 3-C 검증용 `TestWall`을 삭제한 뒤라 로봇+평평한 `GroundPlane`만 남음 | `UsdGeom.Cube.Define()` + `CreateSizeAttr(1.0)`으로 6m×5m 방(벽 4개)을 스크립트로 배치, `ComputeWorldBound()`로 실제 크기 재확인 (3.1절) |

## 6. 체크포인트

- [x] `/World/Room` 벽 4개 배치, `ComputeWorldBound()`로 실제 크기(6m×5m, 높이 1.2m) 확인
- [x] `slam_toolbox_params.yaml` 작성 (`odom_frame: odom`, `base_frame: base_link`, 표준 이름 그대로)
- [x] `base_link`→`base_scan` 정적 TF 발행, `Message Filter dropping` 경고 해소 확인
- [x] 순찰 주행 후 `/map_metadata`가 방 크기(120×100, 5cm 해상도)로 안정화 확인
- [x] `map_saver_cli`로 `~/isaac_assets/carter_standard/maps/carter_room_map.yaml`+`.pgm` 저장, 픽셀 분포로 실제 벽 인식 확인

---
다음: `05-nav2-goal-driving.md` (미작성) — 이번에 저장한 지도로 Nav2 코스트맵을 띄우고, vacuum 프로젝트의 벽/코너 wedging 버그가 여기서도 재현되는지 확인
