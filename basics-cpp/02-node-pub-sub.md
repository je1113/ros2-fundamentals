# 노드 & Publisher/Subscriber — rclcpp의 SharedPtr 콜백 패턴

[`00-syllabus.md`](./00-syllabus.md)의 2번째 주제. `basics/` 2·3번 주제에서 만든 배터리 발행/구독을 C++로 다시 작성하면서, `rclpy`와 `rclcpp`의 콜백 서명·라이프타임 관리 차이를 확인한다. 더 나아가 **Python 노드와 C++ 노드가 같은 토픽을 그대로 주고받을 수 있는지** 직접 확인한다.

---

## 학습 목표

- `rclcpp::Node`를 상속해 퍼블리셔/서브스크립션/타이머를 멤버 변수로 관리하는 이유(라이프타임)를 이해한다.
- 구독 콜백의 인자가 `MessageT::SharedPtr`(참조 카운트 스마트 포인터)인 이유를 설명할 수 있다.
- `rclcpp::QoS`로 `basics/` 3번 주제와 동일한 QoS(BEST_EFFORT)를 맞춰, Python 퍼블리셔 ↔ C++ 구독자가 같은 토픽으로 통신되는 것을 확인한다.

## 핵심 개념

### 왜 퍼블리셔/타이머를 멤버 변수로 들고 있어야 하는가

`create_publisher`/`create_subscription`/`create_wall_timer`는 `shared_ptr`를 반환한다. 이 반환값을 지역 변수로만 받고 버리면, 함수가 끝나는 순간 참조 카운트가 0이 되어 **객체가 즉시 소멸**하고 콜백이 더 이상 호출되지 않는다. 그래서 반드시 노드의 멤버 변수(`rclcpp::Publisher<T>::SharedPtr` 등)로 들고 있어야 노드가 살아있는 동안 함께 살아남는다. Python(`rclpy`)에서는 가비지 컬렉터가 `self.publisher_ = self.create_publisher(...)`처럼 `self`에 붙여두기만 하면 되는 것과 원리는 같지만, C++에서는 스마트 포인터 참조 카운트로 명시적으로 관리해야 한다는 점이 다르다.

### 구독 콜백 서명 — `SharedPtr` 인자

```cpp
void battery_callback(const sensor_msgs::msg::BatteryState::SharedPtr msg)
```

메시지를 **복사하지 않고** 참조 카운트 포인터로 넘겨받는다. `rclpy`에서 `def battery_callback(self, msg: BatteryState)`가 파이썬 객체 참조를 받는 것과 비슷한 동기(불필요한 복사 방지)지만, C++에서는 이게 스마트 포인터 타입으로 명시적으로 드러난다.

## 실습 단계

### 1. `CMakeLists.txt`에 `sensor_msgs` 의존성과 실행 파일 추가

`~/ros2_ws/src/ros2_basics_cpp/CMakeLists.txt`를 아래처럼 바꾼다 (1번 주제 버전에 `sensor_msgs` `find_package`와 두 실행 파일이 추가된 것이다):

```cmake
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(ros2_basics_msgs REQUIRED)
find_package(sensor_msgs REQUIRED)

add_executable(hello_node_cpp src/hello_node_cpp.cpp)
ament_target_dependencies(hello_node_cpp rclcpp)

add_executable(battery_publisher_cpp src/battery_publisher_cpp.cpp)
ament_target_dependencies(battery_publisher_cpp rclcpp sensor_msgs)

add_executable(battery_watcher_cpp src/battery_watcher_cpp.cpp)
ament_target_dependencies(battery_watcher_cpp rclcpp sensor_msgs)

install(TARGETS
  hello_node_cpp
  battery_publisher_cpp
  battery_watcher_cpp
  DESTINATION lib/${PROJECT_NAME}
)
```

`package.xml`에도 `<depend>sensor_msgs</depend>`를 `<depend>ros2_basics_msgs</depend>` 아래에 추가한다.

### 2. 배터리 퍼블리셔

`~/ros2_ws/src/ros2_basics_cpp/src/battery_publisher_cpp.cpp`:

```cpp
#include <chrono>
#include <memory>
#include <rclcpp/rclcpp.hpp>
#include <sensor_msgs/msg/battery_state.hpp>

using namespace std::chrono_literals;

class BatteryPublisherCpp : public rclcpp::Node
{
public:
  BatteryPublisherCpp() : Node("battery_publisher_cpp"), percentage_(1.0f)
  {
    auto qos = rclcpp::QoS(10).best_effort();
    publisher_ = this->create_publisher<sensor_msgs::msg::BatteryState>("battery_state", qos);
    timer_ = this->create_wall_timer(1s, std::bind(&BatteryPublisherCpp::timer_callback, this));
  }

private:
  void timer_callback()
  {
    percentage_ -= 0.05f;
    if (percentage_ < 0.0f) {
      percentage_ = 1.0f;
    }
    auto msg = sensor_msgs::msg::BatteryState();
    msg.percentage = percentage_;
    RCLCPP_INFO(this->get_logger(), "배터리 발행(C++): %.0f%%", static_cast<double>(percentage_) * 100.0);
    publisher_->publish(msg);
  }

  rclcpp::Publisher<sensor_msgs::msg::BatteryState>::SharedPtr publisher_;
  rclcpp::TimerBase::SharedPtr timer_;
  float percentage_;
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<BatteryPublisherCpp>());
  rclcpp::shutdown();
  return 0;
}
```

여기서 쓴 `rclcpp::QoS(10).best_effort()`는 `basics/` 3번 주제에서 `battery_watcher`(Python)가 맞춰둔 `BEST_EFFORT`와 동일한 정책이다 — QoS가 언어와 무관하게 와이어 프로토콜 수준에서 맞아야 한다는 걸 보여준다.

### 3. 배터리 구독자

`~/ros2_ws/src/ros2_basics_cpp/src/battery_watcher_cpp.cpp`:

```cpp
#include <memory>
#include <rclcpp/rclcpp.hpp>
#include <sensor_msgs/msg/battery_state.hpp>

class BatteryWatcherCpp : public rclcpp::Node
{
public:
  BatteryWatcherCpp() : Node("battery_watcher_cpp")
  {
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
    double pct = static_cast<double>(msg->percentage);
    if (pct <= 0.20) {
      RCLCPP_WARN(this->get_logger(), "배터리 부족(C++): %.0f%%", pct * 100.0);
    } else {
      RCLCPP_INFO(this->get_logger(), "배터리 정상(C++): %.0f%%", pct * 100.0);
    }
  }

  rclcpp::Subscription<sensor_msgs::msg::BatteryState>::SharedPtr subscription_;
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<BatteryWatcherCpp>());
  rclcpp::shutdown();
  return 0;
}
```

### 4. 빌드

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics_cpp
source install/setup.bash
```

### 5. 같은 언어끼리 확인 (터미널 2개)

터미널 A: `ros2 run ros2_basics_cpp battery_publisher_cpp`
터미널 B: `ros2 run ros2_basics_cpp battery_watcher_cpp`

### 6. 상호운용 확인 — Python 퍼블리셔 + C++ 구독자

터미널 A: `ros2 run ros2_basics battery_publisher` (Python, `basics/` 2번 주제에서 만든 것)
터미널 B: `ros2 run ros2_basics_cpp battery_watcher_cpp` (C++)

```bash
ros2 topic info /battery_state --verbose
```

## 예상/실제 결과

5단계에서는 C++ 퍼블리셔가 1초마다 배터리 값을 발행하고, C++ 구독자가 20% 이하에서 WARN 로그를 찍는다. 6단계에서는 Python 퍼블리셔(`battery_publisher`)가 발행한 값(100%, 95%, 90%...)을 C++ 구독자(`battery_watcher_cpp`)가 그대로 받아 로그를 찍었다. `ros2 topic info /battery_state --verbose`에도 실제로 두 언어의 노드가 함께 나열됐다:

```
Publisher count: 1
Node name: battery_publisher          # Python
Endpoint type: PUBLISHER

Subscription count: 1
Node name: battery_watcher_cpp        # C++
Endpoint type: SUBSCRIPTION
```

인터페이스가 언어 중립적이라는 것을 실제로 확인했다.

## 알려진 문제와 해결

실습하며 실제로 만난 문제 없음. (QoS를 `BEST_EFFORT`로 맞추지 않았다면 Python 3번 주제와 같은 "메시지가 전혀 안 옴" 증상이 재현됐을 것 — [`../basics/03-qos.md`](../basics/03-qos.md) 참고.)

## 체크포인트

- [ ] 퍼블리셔/타이머를 노드 멤버 변수로 들고 있어야 하는 이유(참조 카운트)를 설명할 수 있다.
- [ ] 구독 콜백이 `MessageT::SharedPtr`를 인자로 받는 이유를 설명할 수 있다.
- [ ] Python 노드와 C++ 노드가 QoS만 맞으면 같은 토픽으로 상호운용된다는 것을 직접 확인했다.

---
다음: [`03-custom-interfaces.md`](./03-custom-interfaces.md)
