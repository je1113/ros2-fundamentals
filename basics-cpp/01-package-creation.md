# 패키지 생성 & 빌드 구조 — ament_cmake는 Python 패키지와 무엇이 다른가

[`00-syllabus.md`](./00-syllabus.md)의 1번째 주제. `basics/` 1번 주제에서 만든 `ament_python` 패키지(`ros2_basics`)와 나란히, C++용 `ament_cmake` 패키지 `ros2_basics_cpp`를 만들면서 빌드 구조의 차이를 확인한다.

---

## 학습 목표

- `ament_python`과 `ament_cmake` 패키지의 디렉토리/설정 파일 구조 차이를 설명할 수 있다.
- `CMakeLists.txt`에 실행 파일을 등록하고 빌드할 수 있다.
- `--symlink-install`이 Python 패키지에서는 코드 즉시 반영을, C++ 패키지에서는 "설치 경로 심볼릭 링크"만 의미할 뿐 컴파일은 매번 다시 필요하다는 것을 직접 확인한다.

## 핵심 개념

| | `ament_python` (`ros2_basics`) | `ament_cmake` (`ros2_basics_cpp`) |
|---|---|---|
| 빌드 설정 | `setup.py` + `setup.cfg` | `CMakeLists.txt` |
| 실행 단위 | `.py` 스크립트 (인터프리터가 즉시 실행) | 컴파일된 바이너리 |
| 실행 파일 등록 | `setup.py`의 `entry_points` | `CMakeLists.txt`의 `add_executable` + `install(TARGETS ...)` |
| `--symlink-install`의 효과 | `install/` 아래가 소스 `.py`로 심볼릭 링크 → **코드 수정 시 재빌드 불필요** | `install/` 아래 실행 파일 경로만 심볼릭 링크 → 코드 수정 시 **`colcon build`로 재컴파일 필수** |

즉 `--symlink-install`은 Python에서는 "재빌드 스킵"처럼 느껴지지만, C++에서는 그런 의미가 전혀 아니다 — 심볼릭 링크가 가리키는 대상(컴파일된 바이너리) 자체가 소스 코드와 별개이기 때문이다. 이 차이를 모르면 "분명 코드를 고쳤는데 왜 실행 결과가 그대로냐"는 흔한 삽질을 하게 된다.

## 실습 단계

### 1. 패키지 생성

```bash
conda deactivate
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_cmake --dependencies rclcpp ros2_basics_msgs ros2_basics_cpp
```

### 2. `package.xml` 확인/수정

`~/ros2_ws/src/ros2_basics_cpp/package.xml`이 아래와 같이 되도록 한다 (`ros2 pkg create`가 대부분 만들어주며, `<depend>` 두 줄만 있는지 확인):

```xml
<?xml version="1.0"?>
<?xml-model href="http://download.ros.org/schema/package_format3.xsd" schematypens="http://www.w3.org/2001/XMLSchema"?>
<package format="3">
  <name>ros2_basics_cpp</name>
  <version>0.0.0</version>
  <description>ROS 2 C++ (rclcpp) basics track</description>
  <maintainer email="jje320594@gmail.com">pw</maintainer>
  <license>TODO: License declaration</license>

  <buildtool_depend>ament_cmake</buildtool_depend>

  <depend>rclcpp</depend>
  <depend>ros2_basics_msgs</depend>

  <test_depend>ament_lint_auto</test_depend>
  <test_depend>ament_lint_common</test_depend>

  <export>
    <build_type>ament_cmake</build_type>
  </export>
</package>
```

### 3. `CMakeLists.txt` 작성

`~/ros2_ws/src/ros2_basics_cpp/CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.8)
project(ros2_basics_cpp)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(ros2_basics_msgs REQUIRED)

add_executable(hello_node_cpp src/hello_node_cpp.cpp)
ament_target_dependencies(hello_node_cpp rclcpp)

install(TARGETS
  hello_node_cpp
  DESTINATION lib/${PROJECT_NAME}
)

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  set(ament_cmake_copyright_FOUND TRUE)
  set(ament_cmake_cpplint_FOUND TRUE)
  ament_lint_auto_find_test_dependencies()
endif()

ament_package()
```

### 4. 첫 노드 작성

`~/ros2_ws/src/ros2_basics_cpp/src/hello_node_cpp.cpp`:

```cpp
#include <memory>
#include <rclcpp/rclcpp.hpp>

class HelloNode : public rclcpp::Node
{
public:
  HelloNode() : Node("hello_node_cpp")
  {
    RCLCPP_INFO(this->get_logger(), "Hello from C++ node!");
  }
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<HelloNode>());
  rclcpp::shutdown();
  return 0;
}
```

### 5. 빌드 & 실행

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics_cpp
source install/setup.bash
ros2 run ros2_basics_cpp hello_node_cpp
```

### 6. `--symlink-install`의 한계 직접 확인

`hello_node_cpp.cpp`에서 로그 메시지를 아무 문자열로 바꾼 뒤, **빌드 없이** 바로 재실행해본다:

```bash
ros2 run ros2_basics_cpp hello_node_cpp
```

여전히 이전 메시지가 나오는지 확인한 뒤, 다시 빌드하고서야 바뀐 메시지가 나오는지 확인한다:

```bash
colcon build --symlink-install --packages-select ros2_basics_cpp
source install/setup.bash
ros2 run ros2_basics_cpp hello_node_cpp
```

## 예상/실제 결과

5단계에서는 `[INFO] [hello_node_cpp]: Hello from C++ node!`가 출력된다. 6단계에서 실제로 로그 문자열을 바꾼 뒤 재빌드 없이 실행하니 **이전 메시지("Hello from C++ node!")가 그대로** 나왔고, `colcon build --symlink-install`을 다시 돌린 뒤에야 바뀐 메시지("Hello from C++ node! (수정된 버전)")가 나오는 것을 직접 확인했다 — Python 패키지였다면 `--symlink-install` 덕분에 빌드 없이도 바로 바뀐 메시지가 나왔을 상황이다.

## 알려진 문제와 해결

첫 시도에서 `colcon build`가 `ModuleNotFoundError: No module named 'catkin_pkg'`로 실패했다 — `conda deactivate`만으로는 이 환경의 anaconda `python3`가 `PATH`에서 완전히 빠지지 않아, ROS 2용 `catkin_pkg`/`ament_package`가 설치된 시스템 `python3`(`/usr/bin/python3`) 대신 anaconda의 `python3`가 잡혔기 때문이다. `PATH`에서 `anaconda3` 항목을 제거하고 `CONDA_PREFIX` 등 관련 환경 변수를 `unset`한 뒤, `/opt/ros/jazzy/setup.bash`를 다시 소싱해 `PYTHONPATH`를 복구하고서야 정상 빌드됐다.

## 체크포인트

- [ ] `ament_python`과 `ament_cmake` 패키지의 구조 차이를 설명할 수 있다.
- [ ] `CMakeLists.txt`에 `add_executable`/`ament_target_dependencies`/`install(TARGETS ...)`로 실행 파일을 등록할 수 있다.
- [ ] C++ 패키지는 `--symlink-install`을 써도 코드 변경 시 재빌드가 필요하다는 것을 직접 확인했다.

---
다음: [`02-node-pub-sub.md`](./02-node-pub-sub.md)
