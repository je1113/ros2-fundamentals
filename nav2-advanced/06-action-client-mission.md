# 06. Nav2 액션 클라이언트로 미션 작성

## 1. 학습 목표

- RViz의 "Nav2 Goal" 버튼이 내부적으로 액션 클라이언트라는 것을 실제 코드로 재현한다.
- `FollowWaypoints` 액션으로 여러 지점을 순서대로 방문하는 미션 스크립트를 작성한다.
- [`basics/06-action.md`](../basics/06-action.md)에서 다룬 액션 클라이언트 패턴(`send_goal_async` → `add_done_callback` → `get_result_async`)이 Nav2 규모에서도 동일하게 적용됨을 확인한다.

## 2. 핵심 개념

- `FollowWaypoints`는 `geometry_msgs/PoseStamped[]`를 goal로 받아 순서대로 방문하는 액션이다. `waypoint_follower` 서버가 처리한다.
- feedback으로 `current_waypoint`(현재 몇 번째 지점으로 이동 중인지)를 실시간으로 알려준다.
- result의 `missed_waypoints`로 도달 실패한 지점들을 알 수 있다.
- 실제 좌표는 지도 위에서 직접 확인해야 한다 — RViz의 "Publish Point" 툴로 `/clicked_point` 토픽에 찍은 좌표를 읽어서 사용했다.

## 3. 실습 단계

### 3.1 실제 갈 수 있는 좌표 확보

```bash
ros2 topic echo /clicked_point
```

RViz에서 "Publish Point"로 지도 위 열린 공간을 클릭하며 좌표를 수집.

### 3.2 패키지 생성

```
~/ros2_ws/src/nav2_advanced/
├── package.xml
├── setup.py
├── setup.cfg
├── resource/nav2_advanced
└── nav2_advanced/
    ├── __init__.py
    └── mission_client.py
```

`mission_client.py`는 `ActionClient(self, FollowWaypoints, 'follow_waypoints')`로 클라이언트를 만들고, 수집한 좌표들을 `PoseStamped` 리스트로 감싸 goal로 전송한다. 콜백 구조는 [`basics/06-action.md`](../basics/06-action.md)의 `return_to_base_client.py`와 동일한 패턴(`goal_response_callback` → `get_result_callback`)이다.

### 3.3 빌드 및 실행

Nav2가 이미 실행 중이고 로봇이 로컬라이즈된 상태에서:

```bash
cd ~/ros2_ws
colcon build --packages-select nav2_advanced
source install/setup.bash
ros2 run nav2_advanced mission_client
```

## 4. 예상/실제 결과 확인

- 터미널에 `N개 웨이포인트로 미션 전송` → `미션 수락됨` → `현재 N번째 웨이포인트로 이동 중`(웨이포인트마다) → `모든 웨이포인트 방문 완료` 순으로 로그가 찍힘.
- RViz에서 로봇이 3개 지점을 순서대로 방문하는 것을 육안으로 확인.

## 5. 알려진 문제와 해결

이번 토픽에서는 새로 마주친 에러 없이 한 번에 정상 동작함 — 좌표를 실제 지도에서 직접 확인해서 쓴 덕분에 "도달 불가능한 목적지" 문제(토픽 1/5에서 겪음)를 피할 수 있었음.

## 6. 체크포인트

- [ ] `/clicked_point`로 실제 갈 수 있는 좌표 3곳 확보
- [ ] `nav2_advanced` 패키지 생성, `mission_client.py`에 `FollowWaypoints` 액션 클라이언트 작성
- [ ] 빌드 후 실행, feedback 로그와 실제 로봇 이동 경로 확인

---
다음: [`07-multi-robot-navigation.md`](./07-multi-robot-navigation.md) (미작성)
