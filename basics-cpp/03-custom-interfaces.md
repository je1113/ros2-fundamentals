# 커스텀 인터페이스 소비 & 상호운용 — 같은 .msg를 Python과 C++이 함께 쓴다

[`00-syllabus.md`](./00-syllabus.md)의 3번째 주제. `basics/` 4번 주제에서 정의한 `BatteryAlert.msg`를 새로 만들지 않고 그대로, C++에서 생성된 헤더로 소비한다. Python `battery_watcher`가 발행하는 `battery_alert` 토픽을 C++ 노드가 구독해서, **인터페이스 정의(.msg)는 한 번만 하면 모든 언어에서 재사용된다**는 것을 직접 확인한다.

---

## 학습 목표

- 커스텀 인터페이스 패키지(`ros2_basics_msgs`)를 C++ 실행 파일이 `find_package`로 가져와 쓸 수 있다.
- 생성된 헤더의 include 경로(snake_case 파일명)와 C++ 타입 네임스페이스(`ros2_basics_msgs::msg::BatteryAlert`)를 안다.
- `.msg`에 정의된 상수(`LEVEL_OK` 등)를 C++에서 정적 멤버로 접근하고, Python enum 상수와 값이 동일함을 확인한다.

## 핵심 개념

`.msg`/`.srv` 파일은 IDL(인터페이스 정의 언어)이다. `rosidl_generate_interfaces`가 빌드 시점에 언어별 코드를 각각 생성하는데, Python은 `ros2_basics_msgs.msg.BatteryAlert` 클래스로, C++은 `ros2_basics_msgs::msg::BatteryAlert` 구조체로 — **같은 정의에서 언어별 바인딩만 다르게** 나온다. 그래서 이미 `basics/` 4번 주제에서 만든 `BatteryAlert.msg`를 이번 트랙에서는 한 줄도 다시 정의하지 않는다.

| | Python (`rclpy`) | C++ (`rclcpp`) |
|---|---|---|
| import/include | `from ros2_basics_msgs.msg import BatteryAlert` | `#include <ros2_basics_msgs/msg/battery_alert.hpp>` |
| 헤더 파일명 규칙 | (해당 없음) | 메시지 이름의 snake_case (`BatteryAlert` → `battery_alert.hpp`) |
| 필드 접근 | `alert.percentage` (속성) | `alert.percentage` (public 멤버, getter 없음) |
| 상수 접근 | `BatteryAlert.LEVEL_LOW` | `ros2_basics_msgs::msg::BatteryAlert::LEVEL_LOW` |

## 실습 단계

### 1. `CMakeLists.txt`에 실행 파일 추가

`~/ros2_ws/src/ros2_basics_cpp/CMakeLists.txt`에 아래 세 줄을 추가하고, `install(TARGETS ...)` 목록에도 `fleet_monitor_cpp`를 추가한다:

```cmake
add_executable(fleet_monitor_cpp src/fleet_monitor_cpp.cpp)
ament_target_dependencies(fleet_monitor_cpp rclcpp ros2_basics_msgs)
```

### 2. `fleet_monitor_cpp.cpp` 작성

`~/ros2_ws/src/ros2_basics_cpp/src/fleet_monitor_cpp.cpp`:

```cpp
#include <memory>
#include <rclcpp/rclcpp.hpp>
#include <ros2_basics_msgs/msg/battery_alert.hpp>

class FleetMonitorCpp : public rclcpp::Node
{
public:
  FleetMonitorCpp() : Node("fleet_monitor_cpp")
  {
    subscription_ = this->create_subscription<ros2_basics_msgs::msg::BatteryAlert>(
      "battery_alert", 10,
      [this](const ros2_basics_msgs::msg::BatteryAlert::SharedPtr msg) {
        this->alert_callback(msg);
      });
  }

private:
  void alert_callback(const ros2_basics_msgs::msg::BatteryAlert::SharedPtr msg)
  {
    using ros2_basics_msgs::msg::BatteryAlert;

    const char * level_name = "UNKNOWN";
    if (msg->level == BatteryAlert::LEVEL_OK) {
      level_name = "OK";
    } else if (msg->level == BatteryAlert::LEVEL_LOW) {
      level_name = "LOW";
    } else if (msg->level == BatteryAlert::LEVEL_CRITICAL) {
      level_name = "CRITICAL";
    }

    RCLCPP_INFO(
      this->get_logger(), "[%s] %s — %.0f%%",
      msg->robot_id.c_str(), level_name, static_cast<double>(msg->percentage) * 100.0);
  }

  rclcpp::Subscription<ros2_basics_msgs::msg::BatteryAlert>::SharedPtr subscription_;
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<FleetMonitorCpp>());
  rclcpp::shutdown();
  return 0;
}
```

`BatteryAlert::LEVEL_LOW` 같은 상수는 `.msg` 파일의 `uint8 LEVEL_LOW=1` 정의에서 자동 생성된 것이다 — Python 쪽 `BatteryAlert.LEVEL_LOW`와 값(1)이 완전히 같다.

### 3. 빌드

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics_cpp
source install/setup.bash
```

### 4. 상호운용 확인 (터미널 3개, 모두 Python 노드 + C++ 노드 혼합)

터미널 A: `ros2 run ros2_basics battery_publisher` (Python)
터미널 B: `ros2 run ros2_basics battery_watcher` (Python — `battery_alert` 발행)
터미널 C: `ros2 run ros2_basics_cpp fleet_monitor_cpp` (C++ — `battery_alert` 구독)

## 예상/실제 결과

실제로 실행해보니 터미널 C에 아래처럼 찍혔다:

```
[INFO] [fleet_monitor_cpp]: [robot_1] OK — 95%
[INFO] [fleet_monitor_cpp]: [robot_1] OK — 90%
...
[INFO] [fleet_monitor_cpp]: [robot_1] OK — 70%
```

발행자(`battery_watcher`)는 Python, 구독자(`fleet_monitor_cpp`)는 C++인데도 변환 코드 없이 그대로 동작했고, `robot_id`/`percentage`/`level` 필드와 `LEVEL_OK` 상수까지 정확히 읽혔다. `.msg` 정의 하나가 두 언어 모두에서 통한다는 것을 직접 확인했다. (짧게 실행해 배터리가 20% 밑으로 떨어지기 전에 종료해서 `LOW`/`CRITICAL` 분기까지는 이번 실행에서 못 봤지만, 분기 로직 자체는 `basics/` 4번 주제에서 이미 같은 임계값으로 검증된 것과 동일하다.)

## 알려진 문제와 해결

실습하며 실제로 만난 문제 없음.

## 체크포인트

- [ ] `find_package(ros2_basics_msgs REQUIRED)` + `ament_target_dependencies`로 커스텀 인터페이스를 C++에서 쓸 수 있다.
- [ ] 생성된 헤더의 include 경로(snake_case)와 C++ 네임스페이스(PascalCase)를 구분해서 설명할 수 있다.
- [ ] `.msg`의 상수가 언어별로 값이 동일하게 생성된다는 것을 확인했다.
- [ ] Python 퍼블리셔가 낸 커스텀 메시지를 C++ 구독자가 그대로 받는 것을 직접 확인했다.

---
다음: [`04-service-async.md`](./04-service-async.md)
