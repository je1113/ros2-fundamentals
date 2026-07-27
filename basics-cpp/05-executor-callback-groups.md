# Executor & 콜백 그룹 — 콜백 안에서 같은 노드로 동기 대기하면 C++은 조용히 안 멈춘다

[`00-syllabus.md`](./00-syllabus.md)의 5번째 주제. `basics/` 7번 주제(Python)에서는 구독 콜백 안에서 서비스를 동기 대기하면 **조용히 멈추는 데드락**이 났다. 같은 짓을 C++로 재현해보면, 결과가 다르다 — 그리고 Python의 해결책(`MultiThreadedExecutor` + `ReentrantCallbackGroup`)이 C++에서는 통하지 않는다는 것도 실제로 확인한다.

---

## 학습 목표

- rclcpp에서 **노드 하나는 동시에 하나의 실행기(Executor)에만 속할 수 있다**는 제약을 이해한다.
- 콜백 안에서 같은 노드로 `rclcpp::spin_until_future_complete()`를 부르면 왜 데드락이 아니라 예외(`runtime_error`)가 나는지 설명할 수 있다.
- 이 문제는 `MultiThreadedExecutor`나 콜백 그룹으로 고칠 수 없고, 4번 주제의 콜백 기반 비동기 패턴으로 **구조를 바꿔야** 해결된다는 것을 실제로 확인한다.

## 핵심 개념

### Python과 C++이 "같은 실수"에 다르게 반응하는 이유

`basics/` 7번 주제의 핵심 원인은 "SingleThreadedExecutor의 유일한 스레드가 자기 자신(지금 실행 중인 콜백)이 끝나기를 기다리며 멈춰버린다"는 것이었다. C++에서도 구조는 똑같아 보인다 — 구독 콜백 안에서 서비스 응답을 기다린다.

그런데 rclcpp의 `spin_until_future_complete(node, future)`는 Python과 구현이 다르다: **호출될 때마다 내부적으로 새로운 임시 `SingleThreadedExecutor`를 만들고, 그 실행기에 노드를 추가(`add_node`)하려고 시도**한다. 문제는 rclcpp가 **"한 노드는 동시에 하나의 실행기에만 속할 수 있다"**는 제약을 코드로 강제한다는 점이다 — 우리 노드는 이미 `main()`의 `rclcpp::spin(node)`가 만든 (바깥) 실행기에 속해 있으므로, 콜백 안에서 또 다른 임시 실행기에 같은 노드를 추가하려는 시도는 **그 순간 즉시 `std::runtime_error` 예외**로 거부된다. 즉 C++은 멈추는 게 아니라 "너 지금 그거 하면 안 돼"라고 그 자리에서 알려준다.

이 제약은 **노드 단위**로 걸려 있다 — 어떤 콜백 그룹을 쓰든, 실행기를 `SingleThreadedExecutor`에서 `MultiThreadedExecutor`로 바꾸든 상관없이, "이미 실행기에 속한 노드를 또 다른 실행기에 추가하는 시도" 자체가 막힌다. 그래서 Python에서 통했던 해결책(`MultiThreadedExecutor` + `ReentrantCallbackGroup`)은 C++의 이 문제에는 적용되지 않는다 — 애초에 `spin_until_future_complete`가 "새 실행기를 만들어 노드를 추가"하는 구현 방식 자체가 문제이기 때문에, 실행기 종류나 콜백 그룹 구성을 바꿔봐야 소용없다.

**진짜 해결책**: 콜백 안에서 같은 노드로 블로킹 대기(`spin_until_future_complete`)를 하지 않는 것. 4번 주제에서 만든 콜백 기반 `async_send_request(request, callback)` 패턴을 쓰면, 새 실행기를 만들거나 노드를 어디에 추가하려는 시도 자체가 없으므로 이 제약에 걸리지 않는다.

## 실습 단계

### 1. 문제를 재현하는 노드

`~/ros2_ws/src/ros2_basics_cpp/src/auto_return_manager_cpp.cpp`:

```cpp
#include <memory>
#include <rclcpp/rclcpp.hpp>
#include <sensor_msgs/msg/battery_state.hpp>
#include <ros2_basics_msgs/srv/request_return_to_base.hpp>

using RequestReturnToBase = ros2_basics_msgs::srv::RequestReturnToBase;

class AutoReturnManagerCpp : public rclcpp::Node
{
public:
  AutoReturnManagerCpp() : Node("auto_return_manager_cpp")
  {
    client_ = this->create_client<RequestReturnToBase>("request_return_to_base");
    auto qos = rclcpp::QoS(10).best_effort();
    subscription_ = this->create_subscription<sensor_msgs::msg::BatteryState>(
      "battery_state", qos,
      [this](const sensor_msgs::msg::BatteryState::SharedPtr msg) {
        this->battery_callback(msg);
      });
  }

private:
  void battery_callback(const sensor_msgs::msg::BatteryState::SharedPtr msg)
  {
    if (msg->percentage > 0.20f) {
      RCLCPP_INFO(this->get_logger(), "배터리 정상(C++): %.0f%%", static_cast<double>(msg->percentage) * 100.0);
      return;
    }

    RCLCPP_INFO(this->get_logger(), "배터리 부족 감지 — 복귀 요청 서비스 호출 (동기 대기, 같은 노드로 중첩)");
    auto request = std::make_shared<RequestReturnToBase::Request>();
    request->robot_id = "robot_1";
    auto future = client_->async_send_request(request);
    // 아래 줄이 문제 지점: 이 노드는 이미 main()의 executor에 추가되어 있는데,
    // spin_until_future_complete가 또 새 executor에 같은 노드를 추가하려 시도한다.
    auto result = rclcpp::spin_until_future_complete(this->get_node_base_interface(), future);
    if (result == rclcpp::FutureReturnCode::SUCCESS) {
      RCLCPP_INFO(this->get_logger(), "응답: %s", future.get()->message.c_str());
    }
  }

  rclcpp::Client<RequestReturnToBase>::SharedPtr client_;
  rclcpp::Subscription<sensor_msgs::msg::BatteryState>::SharedPtr subscription_;
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<AutoReturnManagerCpp>());
  rclcpp::shutdown();
  return 0;
}
```

### 2. `CMakeLists.txt`에 실행 파일 추가

```cmake
add_executable(auto_return_manager_cpp src/auto_return_manager_cpp.cpp)
ament_target_dependencies(auto_return_manager_cpp rclcpp sensor_msgs ros2_basics_msgs)
```

`install(TARGETS ...)` 목록에도 추가한다.

### 3. 빌드

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics_cpp
source install/setup.bash
```

### 4. 문제 재현 (터미널 3개)

터미널 A: `ros2 run ros2_basics battery_publisher` (Python)
터미널 B: `ros2 run ros2_basics battery_watcher` (Python — 서비스 서버 역할)
터미널 C: `ros2 run ros2_basics_cpp auto_return_manager_cpp`

배터리가 20% 이하로 떨어지는 순간 터미널 C를 관찰한다.

### 5. 해결 — 블로킹 대기를 없애고 4번 주제의 비동기 패턴으로 교체

`battery_callback`을 아래처럼 바꾼다 (요청을 보내고 콜백에서 응답을 받도록):

```cpp
  void battery_callback(const sensor_msgs::msg::BatteryState::SharedPtr msg)
  {
    if (msg->percentage > 0.20f) {
      RCLCPP_INFO(this->get_logger(), "배터리 정상(C++): %.0f%%", static_cast<double>(msg->percentage) * 100.0);
      return;
    }

    RCLCPP_INFO(this->get_logger(), "배터리 부족 감지 — 복귀 요청 서비스 호출 (비동기 콜백)");
    auto request = std::make_shared<RequestReturnToBase::Request>();
    request->robot_id = "robot_1";

    using ServiceResponseFuture = rclcpp::Client<RequestReturnToBase>::SharedFuture;
    auto callback = [this](ServiceResponseFuture future) {
      RCLCPP_INFO(this->get_logger(), "응답: %s", future.get()->message.c_str());
    };
    client_->async_send_request(request, callback);
  }
```

`spin_until_future_complete` 호출이 완전히 사라졌다 — 새 실행기를 만들거나 노드를 어디에 추가하려는 시도 자체가 없으므로, 노드-실행기 결합 제약에 걸릴 여지가 없다.

### 6. 재빌드 & 재실행

```bash
colcon build --symlink-install --packages-select ros2_basics_cpp
source install/setup.bash
ros2 run ros2_basics_cpp auto_return_manager_cpp
```

## 예상/실제 결과

4단계(재현) — 실제 `auto_return_manager_cpp`를 빌드해서 돌려보니, 배터리가 20% 이하로 떨어지는 순간 이렇게 찍히고 프로세스가 죽었다:

```
[INFO] [auto_return_manager_cpp]: 배터리 부족 감지 — 복귀 요청 서비스 호출 (동기 대기, 같은 노드로 중첩)
terminate called after throwing an instance of 'std::runtime_error'
  what():  Node '/auto_return_manager_cpp' has already been added to an executor.
[ros2run]: Aborted
```

Python처럼 조용히 멈추는 게 아니라, 그 자리에서 프로세스가 즉시 죽는 것으로 실패가 드러난다.

6단계(해결 후) — 같은 시나리오에서 배터리 로그가 죽지 않고 계속 이어졌고, 20% 이하로 떨어진 뒤 배터리 값이 바뀔 때마다 매번 응답이 정상적으로 찍혔다:

```
[INFO] [auto_return_manager_cpp]: 배터리 부족 감지 — 복귀 요청 서비스 호출 (비동기 콜백)
[INFO] [auto_return_manager_cpp]: 응답: 거부
[INFO] [auto_return_manager_cpp]: 배터리 부족 감지 — 복귀 요청 서비스 호출 (비동기 콜백)
[INFO] [auto_return_manager_cpp]: 응답: 승인 : robot_1 현재 배터리15% - 충전소로 이동하세요.
```

## 알려진 문제와 해결

| 문제 | 원인 | 해결 |
|---|---|---|
| 콜백 안에서 같은 노드로 `spin_until_future_complete` 호출 시 `Node '...' has already been added to an executor.` 예외 | `spin_until_future_complete`가 내부적으로 새 임시 실행기를 만들어 같은 노드를 다시 추가하려 시도 — rclcpp는 노드 하나가 동시에 두 실행기에 속하는 것을 허용하지 않음 | `MultiThreadedExecutor`/콜백 그룹으로는 해결되지 않음(노드 단위 제약이라 무관) — 콜백 안에서 블로킹 대기를 없애고 4번 주제의 `async_send_request(request, callback)` 패턴으로 구조를 바꿔야 함 |

**교훈**: Python 튜토리얼에서 익힌 "콜백 안 블로킹 호출 → `MultiThreadedExecutor` + `ReentrantCallbackGroup`" 해결 공식을 C++에 그대로 옮기면 안 된다. rclcpp는 애초에 "노드 하나 = 실행기 하나"를 강제하기 때문에, 콜백 안에서 같은 노드로 블로킹 대기를 하는 패턴 자체를 피하는 것이 유일한 해법이다.

## 체크포인트

- [ ] rclcpp에서 노드 하나가 동시에 하나의 실행기에만 속할 수 있다는 제약을 설명할 수 있다.
- [ ] 콜백 안에서 같은 노드로 `spin_until_future_complete`를 부르면 왜 예외가 나는지 설명할 수 있다.
- [ ] 이 문제가 `MultiThreadedExecutor`/콜백 그룹으로 해결되지 않는 이유를 설명할 수 있다.
- [ ] 콜백 기반 비동기 패턴으로 구조를 바꿔 문제를 실제로 해결했다.

---
다음: [`06-composition.md`](./06-composition.md)
