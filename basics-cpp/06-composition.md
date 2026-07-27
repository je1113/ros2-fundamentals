# 컴포지션 (rclcpp_components) — 여러 노드를 한 프로세스에 플러그인으로 로드하기

[`00-syllabus.md`](./00-syllabus.md)의 6번째(마지막) 주제. 지금까지 이 트랙과 `basics/` 트랙 전체에서 `battery_publisher`/`battery_watcher`를 항상 **터미널을 두 개 띄워 두 개의 OS 프로세스**로 실행했다. `rclcpp_components`로 같은 두 노드를 공유 라이브러리 컴포넌트로 만들어, **하나의 프로세스**에 플러그인처럼 동적으로 로드해본다 — Python(`basics/` 8번)보다 C++에서 훨씬 널리 쓰이는 실전 패턴이다.

---

## 학습 목표

- 노드를 실행 파일이 아니라 **공유 라이브러리 컴포넌트**로 빌드하는 방법(`rclcpp_components_register_node`)을 안다.
- `ros2 run rclcpp_components component_container` + `ros2 component load`로 여러 노드를 한 프로세스에 로드할 수 있다.
- 같은 컴포넌트 코드가 독립 실행 파일로도, 컨테이너에 로드되는 플러그인으로도 동시에 쓰일 수 있음을 확인한다.

## 핵심 개념

지금까지 만든 노드는 모두 `int main() { rclcpp::spin(std::make_shared<X>()); }` 형태의 **독립 실행 파일**이었다. 노드를 여러 개 실행하려면 그만큼 OS 프로세스를 여러 개 띄워야 했다(터미널 A, B, C...).

`rclcpp_components`는 노드를 **`.so` 공유 라이브러리**로 빌드하고, `class_loader`(pluginlib 기반) 메커니즘으로 런타임에 동적으로 로드할 수 있게 해준다. **컨테이너**(`component_container`)라는 빈 프로세스를 하나 띄운 뒤, 그 안에 원하는 컴포넌트들을 `ros2 component load`로 하나씩 얹으면, 여러 노드가 **같은 프로세스, 같은 executor**에서 동작한다. 이 방식의 실전 장점은:

- 프로세스 개수가 줄어 배포/관리가 단순해진다.
- 같은 프로세스 안의 퍼블리셔-구독이면 인트라프로세스 통신(zero-copy에 가까운 최적화)이 가능해진다.
- 어떤 노드를 로드할지를 **실행 파일이 아니라 launch 설정**으로 결정할 수 있다.

## 실습 단계

### 1. `CMakeLists.txt`에 `rclcpp_components` 의존성과 라이브러리 타깃 추가

`package.xml`에 `<depend>rclcpp_components</depend>`를 추가한다.

`CMakeLists.txt`에 아래를 추가한다:

```cmake
find_package(rclcpp_components REQUIRED)

add_library(battery_publisher_component SHARED src/battery_publisher_component.cpp)
ament_target_dependencies(battery_publisher_component rclcpp rclcpp_components sensor_msgs)
rclcpp_components_register_node(battery_publisher_component
  PLUGIN "ros2_basics_cpp::BatteryPublisherComponent"
  EXECUTABLE battery_publisher_component_exe
)

add_library(battery_watcher_component SHARED src/battery_watcher_component.cpp)
ament_target_dependencies(battery_watcher_component rclcpp rclcpp_components sensor_msgs)
rclcpp_components_register_node(battery_watcher_component
  PLUGIN "ros2_basics_cpp::BatteryWatcherComponent"
  EXECUTABLE battery_watcher_component_exe
)

install(TARGETS
  battery_publisher_component
  battery_watcher_component
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
)
```

`rclcpp_components_register_node`는 컴포넌트를 pluginlib 플러그인으로 등록하는 XML을 자동 생성하고, `EXECUTABLE`로 지정한 이름의 **독립 실행 파일도 함께** 만들어준다 — 같은 코드를 두 가지 방식으로 쓸 수 있게 되는 것이다.

### 2. 배터리 퍼블리셔 컴포넌트

`~/ros2_ws/src/ros2_basics_cpp/src/battery_publisher_component.cpp`:

```cpp
#include <chrono>
#include <memory>
#include <rclcpp/rclcpp.hpp>
#include <rclcpp_components/register_node_macro.hpp>
#include <sensor_msgs/msg/battery_state.hpp>

using namespace std::chrono_literals;

namespace ros2_basics_cpp
{

class BatteryPublisherComponent : public rclcpp::Node
{
public:
  explicit BatteryPublisherComponent(const rclcpp::NodeOptions & options)
  : Node("battery_publisher_component", options), percentage_(1.0f)
  {
    auto qos = rclcpp::QoS(10).best_effort();
    publisher_ = this->create_publisher<sensor_msgs::msg::BatteryState>("battery_state", qos);
    timer_ = this->create_wall_timer(
      1s, std::bind(&BatteryPublisherComponent::timer_callback, this));
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
    RCLCPP_INFO(
      this->get_logger(), "배터리 발행(컴포넌트): %.0f%%",
      static_cast<double>(percentage_) * 100.0);
    publisher_->publish(msg);
  }

  rclcpp::Publisher<sensor_msgs::msg::BatteryState>::SharedPtr publisher_;
  rclcpp::TimerBase::SharedPtr timer_;
  float percentage_;
};

}  // namespace ros2_basics_cpp

RCLCPP_COMPONENTS_REGISTER_NODE(ros2_basics_cpp::BatteryPublisherComponent)
```

### 3. 배터리 구독자 컴포넌트

`~/ros2_ws/src/ros2_basics_cpp/src/battery_watcher_component.cpp`:

```cpp
#include <memory>
#include <rclcpp/rclcpp.hpp>
#include <rclcpp_components/register_node_macro.hpp>
#include <sensor_msgs/msg/battery_state.hpp>

namespace ros2_basics_cpp
{

class BatteryWatcherComponent : public rclcpp::Node
{
public:
  explicit BatteryWatcherComponent(const rclcpp::NodeOptions & options)
  : Node("battery_watcher_component", options)
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
      RCLCPP_WARN(this->get_logger(), "배터리 부족(컴포넌트): %.0f%%", pct * 100.0);
    } else {
      RCLCPP_INFO(this->get_logger(), "배터리 정상(컴포넌트): %.0f%%", pct * 100.0);
    }
  }

  rclcpp::Subscription<sensor_msgs::msg::BatteryState>::SharedPtr subscription_;
};

}  // namespace ros2_basics_cpp

RCLCPP_COMPONENTS_REGISTER_NODE(ros2_basics_cpp::BatteryWatcherComponent)
```

### 4. 빌드

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics_cpp
source install/setup.bash
```

### 5. 독립 실행 파일로도 동작하는지 확인 (같은 코드, 다른 실행 방식)

```bash
ros2 run ros2_basics_cpp battery_publisher_component_exe
```

`Ctrl+C`로 종료한다 — 컨테이너 없이도 평범한 노드처럼 동작한다는 것만 확인한다.

### 6. 컨테이너에 두 컴포넌트 함께 로드 (터미널 3개)

터미널 A — 빈 컨테이너 프로세스 시작:
```bash
ros2 run rclcpp_components component_container
```

터미널 B — 컨테이너에 두 컴포넌트를 로드:
```bash
ros2 component load /ComponentManager ros2_basics_cpp ros2_basics_cpp::BatteryPublisherComponent
ros2 component load /ComponentManager ros2_basics_cpp ros2_basics_cpp::BatteryWatcherComponent
```

터미널 C — 확인:
```bash
ros2 component list
ros2 node list
ps -o pid,cmd -C component_container
```

## 예상/실제 결과

실제로 로드해보니 터미널 A(컨테이너)의 로그에 배터리 발행/경고가 **한 프로세스 안에서** 함께 찍혔다:

```
[INFO] [battery_publisher_component]: 배터리 발행(컴포넌트): 95%
[INFO] [battery_watcher_component]: 배터리 정상(컴포넌트): 95%
[INFO] [battery_publisher_component]: 배터리 발행(컴포넌트): 90%
[INFO] [battery_watcher_component]: 배터리 정상(컴포넌트): 90%
```

`ros2 component list`에는 `/ComponentManager` 아래 두 컴포넌트가, `ros2 node list`에는 `/battery_publisher_component`/`/battery_watcher_component` 두 노드가 나열됐지만, `pgrep -af 'lib/rclcpp_components/component_container$'`로 확인한 실제 OS 프로세스는 **단 하나(PID 하나)** 뿐이었다 — 지금까지 이 트랙과 `basics/` 트랙 전체에서 두 노드를 실행할 때마다 터미널(프로세스)을 두 개 썼던 것과 명확히 대조된다.

## 알려진 문제와 해결

실습하며 실제로 만난 문제 없음.

## 체크포인트

- [x] `rclcpp_components_register_node`로 노드를 공유 라이브러리 컴포넌트로 빌드할 수 있다.
- [x] `component_container` + `ros2 component load`로 여러 컴포넌트를 한 프로세스에 로드할 수 있다.
- [x] 같은 컴포넌트 코드가 독립 실행 파일과 플러그인 두 가지 방식으로 쓰일 수 있음을 확인했다.
- [x] 여러 노드가 한 OS 프로세스에서 동작하는 것을 `ps`로 직접 확인했다.

**2026-07-27 실행 검증 완료**: `colcon build` 성공, 독립 실행 파일 동작 확인, 컨테이너에 두 컴포넌트 로드 후 `ros2 component list`/`ros2 node list`/`ps -o pid,cmd -C component_container`로 위 예상 결과와 정확히 일치함을 직접 확인. 터미널 A(컨테이너)에 두 컴포넌트의 로그가 번갈아 찍히는 것도 확인.

---
이것으로 [`00-syllabus.md`](./00-syllabus.md)의 6개 주제를 모두 마쳤다.
