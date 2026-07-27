# 서비스 비동기 패턴 — 블로킹 대기 대신 콜백으로 응답 받기

[`00-syllabus.md`](./00-syllabus.md)의 4번째 주제. `basics/` 5번 주제에서 만든 `RequestReturnToBase` 서비스(Python 서버)를 C++ 클라이언트로 호출한다. rclcpp의 관용적인 패턴인 **콜백 기반 비동기 호출**을 구현하고, 이게 왜 "안전한 기본형"인지(5번 주제의 데드락과 대비해서) 이해한다.

---

## 학습 목표

- `async_send_request(request, callback)`으로 응답을 콜백에서 받는 패턴을 구현할 수 있다.
- 이 콜백 패턴이 `rclpy`의 `spin_until_future_complete`(블로킹 대기)와 근본적으로 다른 실행 모델임을 설명할 수 있다.
- 서비스 인터페이스(`RequestReturnToBase`) 역시 Python 서버 ↔ C++ 클라이언트로 상호운용됨을 확인한다.

## 핵심 개념

Python(`basics/` 5번)에서는 클라이언트가 아래처럼 **그 자리에서 멈춰 응답을 기다렸다**:

```python
future = self.client_.call_async(request)
rclpy.spin_until_future_complete(self, future)
return future.result()
```

이 패턴은 독립 실행되는 짧은 CLI 스크립트에서는 문제없지만, **다른 콜백이 이미 실행 중인 노드 안에서** 쓰면 5번 주제에서 다룰 데드락으로 이어진다 — 응답을 처리할 스레드가 지금 그 대기 지점에 스스로 묶여 있기 때문이다.

rclcpp에서는 애초에 이 블로킹 대기를 피하고, **응답이 왔을 때 실행될 콜백을 등록**하는 방식을 기본으로 삼는다:

```cpp
auto future = client_->async_send_request(request, callback);
// 여기서 멈추지 않고 즉시 다음 줄로 진행 — callback은 나중에 executor가 실행
```

이 차이는 사소한 스타일 차이가 아니라, **콜백 안에서 서비스를 또 호출해야 하는 상황**(5번 주제)을 안전하게 다루는 근본적인 설계 차이다.

## 실습 단계

### 1. `CMakeLists.txt`에 실행 파일 추가

```cmake
add_executable(request_return_client_cpp src/request_return_client_cpp.cpp)
ament_target_dependencies(request_return_client_cpp rclcpp ros2_basics_msgs)
```

`install(TARGETS ...)` 목록에도 `request_return_client_cpp`를 추가한다.

### 2. `request_return_client_cpp.cpp` 작성

`~/ros2_ws/src/ros2_basics_cpp/src/request_return_client_cpp.cpp`:

```cpp
#include <chrono>
#include <memory>
#include <string>
#include <rclcpp/rclcpp.hpp>
#include <ros2_basics_msgs/srv/request_return_to_base.hpp>

using namespace std::chrono_literals;
using RequestReturnToBase = ros2_basics_msgs::srv::RequestReturnToBase;

class RequestReturnClientCpp : public rclcpp::Node
{
public:
  RequestReturnClientCpp() : Node("request_return_client_cpp")
  {
    client_ = this->create_client<RequestReturnToBase>("request_return_to_base");
  }

  void send_request(const std::string & robot_id)
  {
    while (!client_->wait_for_service(1s)) {
      RCLCPP_INFO(this->get_logger(), "서비스 대기 중...");
    }

    auto request = std::make_shared<RequestReturnToBase::Request>();
    request->robot_id = robot_id;

    using ServiceResponseFuture = rclcpp::Client<RequestReturnToBase>::SharedFuture;
    auto callback = [this](ServiceResponseFuture future) {
      auto response = future.get();
      RCLCPP_INFO(
        this->get_logger(), "응답: accepted=%s, message=\"%s\"",
        response->accepted ? "true" : "false", response->message.c_str());
      rclcpp::shutdown();
    };
    client_->async_send_request(request, callback);
  }

private:
  rclcpp::Client<RequestReturnToBase>::SharedPtr client_;
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  std::string robot_id = (argc > 1) ? argv[1] : "robot_1";

  auto node = std::make_shared<RequestReturnClientCpp>();
  node->send_request(robot_id);
  rclcpp::spin(node);
  return 0;
}
```

`send_request()`는 요청을 보내고 **즉시 반환**한다 — 응답을 기다리는 코드가 없다. 실제 응답 처리는 `rclcpp::spin(node)`가 돌아가는 동안 executor가 `callback`을 호출해줄 때 일어나고, 그 콜백 안에서 `rclcpp::shutdown()`을 불러 프로그램을 종료한다.

### 3. 빌드

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics_cpp
source install/setup.bash
```

### 4. 실행 (터미널 3개)

터미널 A: `ros2 run ros2_basics battery_publisher` (Python)
터미널 B: `ros2 run ros2_basics battery_watcher` (Python — 서비스 서버 역할)

배터리가 20% 초과일 때:
```bash
ros2 run ros2_basics_cpp request_return_client_cpp robot_1
```

배터리가 20% 이하로 떨어진 뒤 다시:
```bash
ros2 run ros2_basics_cpp request_return_client_cpp robot_1
```

## 예상/실제 결과

`basics/` 5번 주제와 동일하게, 배터리가 20% 초과일 때는 `accepted=false`, 20% 이하일 때는 `accepted=true` 응답을 받는다. 서버는 Python(`battery_watcher`), 클라이언트는 C++(`request_return_client_cpp`)인데도 문제없이 동작했다. 실제 응답 로그:

```
응답: accepted=false, message="거부"
...
응답: accepted=true, message="승인 : robot_1 현재 배터리10% - 충전소로 이동하세요."
```

## 알려진 문제와 해결

실습하며 실제로 만난 문제 없음. 다만 서버(`battery_watcher`)의 배터리 값이 시간에 따라 계속 오르내리므로, 낮은 배터리 상태를 확인하려면 서버 로그에서 실제로 `배터리 부족` 경고가 뜨는 시점에 맞춰 클라이언트를 호출해야 한다(고정된 대기 시간으로는 타이밍을 놓칠 수 있다).

## 체크포인트

- [ ] `async_send_request(request, callback)`으로 콜백 기반 비동기 호출을 구현할 수 있다.
- [ ] 이 패턴이 블로킹 대기와 실행 모델상 어떻게 다른지 설명할 수 있다.
- [ ] Python 서비스 서버와 C++ 클라이언트가 상호운용되는 것을 확인했다.

---
다음: [`05-executor-callback-groups.md`](./05-executor-callback-groups.md)
