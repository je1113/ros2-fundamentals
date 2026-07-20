# 미니 프로젝트: 행맨 게임 — 서비스 + 파라미터 + 토픽 종합 실습

[`00-syllabus.md`](./00-syllabus.md)의 14번째(마지막) 주제. 새 개념은 없다 — 5번(서비스), 9번(파라미터), 2번(토픽) 주제에서 배운 것만으로 알파벳 맞추기 게임(행맨)을 만들어, 지금까지 배운 통신 모델을 조합해서 쓰는 감각을 익힌다.

---

## 학습 목표

- 서비스(요청-응답)를 게임의 턴제 상호작용(글자 추측 → 결과 응답)에 적용할 수 있다.
- 서버 노드 내부에 상태(정답 단어, 맞춘 글자, 남은 기회)를 유지하면서, 매 요청마다 그 상태를 갱신·판정하는 패턴을 구현할 수 있다.
- 같은 상태 변화를 서비스 응답(요청자 전용)과 토픽 발행(누구나 관전 가능) 두 가지 방식으로 동시에 내보내는 이유를 설명할 수 있다.
- 파라미터로 단어 목록과 최대 기회 횟수를 주입해, 코드를 고치지 않고도 난이도를 바꿀 수 있다.

## 핵심 개념

지금까지 배운 세 가지를 한 게임에 조합한다.

| 배운 개념 | 이번 실습에서의 역할 |
|---|---|
| 서비스 (5번) | 플레이어가 글자 하나를 추측 → 서버가 정답 여부/남은 기회를 즉시 응답 |
| 파라미터 (9번) | 단어 목록(`word_list`)과 최대 오답 허용 횟수(`max_attempts`)를 YAML로 주입 |
| 토픽 (2번) | 서버가 매 추측마다 게임 상태를 `hangman_state`로 발행 — 요청자 외의 관전 노드도 진행 상황을 볼 수 있게 |

서비스와 토픽을 같이 쓰는 이유는 5번 주제(배터리 복귀 요청)와 반대 방향의 조합이다: 5번은 "토픽 구독으로 쌓인 상태를 서비스 응답에 활용"했다면, 이번에는 "서비스 호출로 바뀐 상태를 토픽으로 내보내" 요청을 보낸 클라이언트 외에 다른 노드(관전자, 로깅 노드 등)도 같은 상태를 구독할 수 있게 한다.

```
hangman_client --(guess_letter, 서비스 요청)--> hangman_server --(hangman_state, 토픽)--> (관전 가능한 아무 노드)
```

## 실습 단계

### 1. 서비스 인터페이스 정의

`~/ros2_ws/src/ros2_basics_msgs/srv/GuessLetter.srv`:

```
string letter
---
bool correct
string masked_word
int32 remaining_attempts
bool game_over
bool won
```

### 2. 상태 브로드캐스트용 msg 정의

`~/ros2_ws/src/ros2_basics_msgs/msg/HangmanState.msg`:

```
string masked_word
int32 remaining_attempts
int32 max_attempts
bool game_over
bool won
```

### 3. `CMakeLists.txt`에 추가

`~/ros2_ws/src/ros2_basics_msgs/CMakeLists.txt`의 `rosidl_generate_interfaces(...)` 블록에 두 줄을 추가해서, 아래와 같이 되도록 한다 (기존 항목은 그대로 둔다):

```cmake
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/BatteryAlert.msg"
  "msg/HangmanState.msg"
  "srv/RequestReturnToBase.srv"
  "srv/GuessLetter.srv"
  "action/ReturnToBase.action"
  DEPENDENCIES action_msgs
)
```

### 4. 빌드 & 확인

```bash
conda deactivate
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics_msgs
source install/setup.bash
ros2 interface show ros2_basics_msgs/srv/GuessLetter
ros2 interface show ros2_basics_msgs/msg/HangmanState
```

### 5. 파라미터 YAML 작성

`~/ros2_ws/src/ros2_basics/config/hangman_params.yaml`:

```yaml
hangman_server:
  ros__parameters:
    word_list:
      - NODE
      - TOPIC
      - SERVICE
      - ACTION
      - LAUNCH
      - PARAMETER
      - EXECUTOR
      - COMPOSITION
    max_attempts: 6
```

단어 목록을 지금까지 배운 ROS 2 개념 이름으로 채웠다 — 원하는 단어로 바꿔도 무방하다.

### 6. `hangman_server.py` 구현

`~/ros2_ws/src/ros2_basics/ros2_basics/hangman_server.py`:

```python
import random
import rclpy
from rclpy.node import Node
from ros2_basics_msgs.srv import GuessLetter
from ros2_basics_msgs.msg import HangmanState

DEFAULT_WORD_LIST = [
    'NODE', 'TOPIC', 'SERVICE', 'ACTION',
    'LAUNCH', 'PARAMETER', 'EXECUTOR', 'COMPOSITION',
]


class HangmanServer(Node):
    def __init__(self):
        super().__init__('hangman_server')
        self.declare_parameter('word_list', DEFAULT_WORD_LIST)
        self.declare_parameter('max_attempts', 6)

        word_list = self.get_parameter('word_list').value
        self.max_attempts_ = self.get_parameter('max_attempts').value

        self.word_ = random.choice(word_list).upper()
        self.guessed_letters_ = set()
        self.remaining_attempts_ = self.max_attempts_
        self.game_over_ = False
        self.won_ = False

        self.state_publisher_ = self.create_publisher(HangmanState, 'hangman_state', 10)
        self.create_service(GuessLetter, 'guess_letter', self.handle_guess)

        self.get_logger().info(
            f'행맨 시작 — 단어 길이 {len(self.word_)}자, 남은 기회 {self.remaining_attempts_}회')
        self.publish_state()

    def masked_word(self) -> str:
        return ' '.join(c if c in self.guessed_letters_ else '_' for c in self.word_)

    def publish_state(self):
        msg = HangmanState()
        msg.masked_word = self.masked_word()
        msg.remaining_attempts = self.remaining_attempts_
        msg.max_attempts = self.max_attempts_
        msg.game_over = self.game_over_
        msg.won = self.won_
        self.state_publisher_.publish(msg)

    def handle_guess(self, request, response):
        letter = request.letter.strip().upper()

        if self.game_over_:
            self.get_logger().warn('이미 종료된 게임입니다 — 새 게임은 노드를 재시작하세요')
        elif len(letter) != 1 or not letter.isalpha():
            self.get_logger().warn(f'잘못된 입력: "{request.letter}" — 알파벳 한 글자만 가능')
        elif letter in self.guessed_letters_:
            self.get_logger().info(f'"{letter}"는 이미 시도한 글자입니다')
        else:
            self.guessed_letters_.add(letter)
            if letter in self.word_:
                self.get_logger().info(f'"{letter}" 정답!')
            else:
                self.remaining_attempts_ -= 1
                self.get_logger().info(
                    f'"{letter}" 오답 — 남은 기회 {self.remaining_attempts_}회')

            if all(c in self.guessed_letters_ for c in self.word_):
                self.game_over_ = True
                self.won_ = True
            elif self.remaining_attempts_ <= 0:
                self.game_over_ = True
                self.won_ = False
                self.guessed_letters_.update(self.word_)

        response.correct = len(letter) == 1 and letter in self.word_
        response.masked_word = self.masked_word()
        response.remaining_attempts = self.remaining_attempts_
        response.game_over = self.game_over_
        response.won = self.won_

        self.publish_state()
        if self.game_over_:
            result = '승리' if self.won_ else f'패배 (정답: {self.word_})'
            self.get_logger().info(f'게임 종료 — {result}')

        return response


def main():
    rclpy.init()
    node = HangmanServer()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

이미 시도한 글자를 다시 추측하거나 잘못된 입력(빈 문자열, 두 글자 이상, 숫자 등)을 보내면 기회를 소모하지 않는다 — 서버가 클라이언트의 입력을 신뢰하지 않고 자체적으로 검증하는 부분이다.

### 7. `hangman_client.py` 구현

`~/ros2_ws/src/ros2_basics/ros2_basics/hangman_client.py`:

```python
import rclpy
from rclpy.node import Node
from ros2_basics_msgs.srv import GuessLetter


class HangmanClient(Node):
    def __init__(self):
        super().__init__('hangman_client')
        self.client_ = self.create_client(GuessLetter, 'guess_letter')

    def guess(self, letter: str):
        while not self.client_.wait_for_service(timeout_sec=1.0):
            self.get_logger().info('서비스 대기 중...')
        request = GuessLetter.Request()
        request.letter = letter
        future = self.client_.call_async(request)
        rclpy.spin_until_future_complete(self, future)
        return future.result()


def main():
    rclpy.init()
    node = HangmanClient()

    print('행맨 게임 — 알파벳을 한 글자씩 입력하세요 (그만하려면 Ctrl+C)')
    game_over = False
    while not game_over:
        letter = input('추측할 글자: ').strip()
        if not letter:
            continue
        result = node.guess(letter)
        mark = 'O' if result.correct else 'X'
        print(f'[{mark}] {result.masked_word}  (남은 기회 {result.remaining_attempts}회)')
        if result.game_over:
            print('정답입니다! 승리!' if result.won else '기회를 모두 소진했습니다. 패배.')
            game_over = True

    node.destroy_node()
    rclpy.shutdown()
```

### 8. `setup.py`에 등록

`entry_points`의 `console_scripts`에 두 줄을 추가한다:

```python
            'hangman_server = ros2_basics.hangman_server:main',
            'hangman_client = ros2_basics.hangman_client:main',
```

`data_files`의 `config` 항목에도 새 YAML을 추가한다:

```python
        ('share/' + package_name + '/config', [
            'config/battery_watcher_params.yaml',
            'config/battery_watcher_params_aggressive.yaml',
            'config/simple_arm_controllers.yaml',
            'config/hangman_params.yaml',
        ]),
```

### 9. 빌드

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics_msgs ros2_basics
source install/setup.bash
```

### 10. 실행 (터미널 3개)

터미널 A — 서버를 파라미터 파일과 함께 실행:
```bash
ros2 run ros2_basics hangman_server --ros-args --params-file install/ros2_basics/share/ros2_basics/config/hangman_params.yaml
```

터미널 B — 클라이언트로 플레이:
```bash
ros2 run ros2_basics hangman_client
```

터미널 C — 관전(클라이언트 없이도 진행 상황 확인):
```bash
ros2 topic echo /hangman_state
```

## 예상/실제 결과

터미널 B에서 글자를 입력할 때마다 정답/오답과 마스킹된 단어(`_ _ P _ _`)가 갱신되고, 동시에 터미널 C에서도 같은 상태가 토픽으로 흘러나온다. 이미 시도한 글자를 다시 넣거나 두 글자 이상을 넣으면 기회가 줄지 않고, 기회를 다 소진하면 서버 로그에 정답 단어가 공개되면서 `game_over=True, won=False`가 응답에 담긴다. 실제로 `ros2 service call`로 빈 문자열/두 글자 입력이 기회를 소모하지 않는 것, 정답 단어(예: `NODE`)를 끝까지 맞혀 `won=True`로 끝나는 것, 단어 목록에 없는 글자들(B/F/G/J/K/Q)만으로 6회 오답을 내 `TOPIC`이 패배로 공개되는 것, 종료 후 추가 요청이 상태를 바꾸지 않고 경고만 남기는 것을 모두 확인했다. `hangman_client.py`도 파이프 입력으로 실행해 `LAUNCH`를 끝내 못 맞히고 패배 메시지가 출력되는 전체 흐름을 확인했다.

```
[INFO] [...] [hangman_server]: "Q" 오답 — 남은 기회 0회
[INFO] [...] [hangman_server]: 게임 종료 — 패배 (정답: TOPIC)
[WARN] [...] [hangman_server]: 이미 종료된 게임입니다 — 새 게임은 노드를 재시작하세요
```

## 알려진 문제와 해결

이번 실습에서는 별도로 발생한 문제 없음. 다만 `hangman_client.py`의 `input()`이 EOF(파이프 입력이 끊긴 경우)를 만나면 예외가 나므로, `except EOFError: break`로 방어 처리를 추가했다.

## 체크포인트

- [ ] 서비스 요청-응답을 게임의 턴제 상호작용으로 구현할 수 있다.
- [ ] 서버 내부 상태(정답 단어, 맞춘 글자, 남은 기회)를 요청마다 갱신·검증하는 패턴을 구현할 수 있다.
- [ ] 같은 상태 변화를 서비스 응답과 토픽 발행으로 동시에 내보내는 이유를 설명할 수 있다.
- [ ] 파라미터 YAML로 단어 목록/최대 기회 횟수를 바꿔 난이도를 조절할 수 있다.

---
이것으로 [`00-syllabus.md`](./00-syllabus.md)의 14개 주제를 모두 마쳤다.
