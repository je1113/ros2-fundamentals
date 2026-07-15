# ROS 2 패키지 생성 — `ros2 pkg create`부터 `ros2 run`까지

[`00-syllabus.md`](./00-syllabus.md)의 1번째 주제. 이 문서만으로 워크스페이스 안에 파이썬 노드 패키지를 만들고, 최소한의 노드를 실행하는 데까지 따라할 수 있다.

---

## 학습 목표

- ROS 2 워크스페이스/패키지의 역할과 디렉토리 구조를 이해한다.
- `ros2 pkg create`로 `ament_python` 패키지를 생성할 수 있다.
- `setup.py`의 `entry_points`(콘솔 스크립트) 등록 방식을 이해하고, `colcon build` → `ros2 run`까지 실행할 수 있다.

## 핵심 개념

ROS 2에서 **워크스페이스**는 여러 패키지를 담는 작업 공간이고, **패키지**는 빌드/배포의 최소 단위다. 워크스페이스의 `src/` 아래에 패키지 소스가 있고, `colcon build`가 이를 빌드해 `build/`(중간 산출물), `install/`(설치 결과), `log/`(빌드 로그)를 만든다.

파이썬 노드를 담는 패키지는 `ament_python` 빌드 타입을 쓴다. `ros2 pkg create`로 생성하면 다음 구조가 나온다.

```
ros2_basics/
├── package.xml       # 패키지 메타데이터 + 의존성 선언 (rclpy 등)
├── setup.py          # 빌드/설치 스크립트, entry_points(콘솔 스크립트) 정의
├── setup.cfg         # setup.py 보조 설정 (설치 스크립트 경로 등)
├── resource/ros2_basics   # ament 리소스 마커 (빈 파일, 패키지 인덱싱용)
├── ros2_basics/       # 실제 파이썬 모듈 (import 대상), __init__.py 위치
└── test/              # flake8/pep257 등 코드 스타일 테스트
```

노드를 터미널에서 `ros2 run <패키지> <실행이름>`으로 실행하려면, `setup.py`의 `entry_points['console_scripts']`에 `실행이름 = 모듈경로:함수` 형태로 등록해야 한다. 이게 빠지면 패키지는 빌드되어도 `ros2 run`으로 실행할 방법이 없다.

## 실습 단계

> 사전 조건: `~/ros2_ws/src`가 존재하는 워크스페이스, ROS 2 환경 소싱 완료.

### 1. 패키지 생성

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python ros2_basics --dependencies rclpy
```

### 2. 생성된 구조 확인

```bash
tree ros2_basics    # tree 없으면: find ros2_basics
cat ros2_basics/package.xml
cat ros2_basics/setup.py
```

### 3. 최소 노드 작성

`~/ros2_ws/src/ros2_basics/ros2_basics/hello_node.py`:

```python
import rclpy
from rclpy.node import Node


class HelloNode(Node):
    def __init__(self):
        super().__init__('hello_node')
        self.get_logger().info('ros2_basics 패키지, hello_node 정상 실행!')


def main():
    rclpy.init()
    node = HelloNode()
    rclpy.spin_once(node, timeout_sec=1.0)
    node.destroy_node()
    rclpy.shutdown()
```

### 4. `entry_points` 등록

`ros2 pkg create`가 만든 `setup.py`에는 `entry_points={'console_scripts': []}`가 빈 리스트로 들어있다. 그 리스트 안에 아래 줄을 추가해서, `setup.py` 전체가 다음과 같이 되도록 한다 (굵게 표시한 부분이 새로 추가하는 줄이다):

```python
from setuptools import find_packages, setup

package_name = 'ros2_basics'

setup(
    name=package_name,
    version='0.0.0',
    packages=find_packages(exclude=['test']),
    data_files=[
        ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
    ],
    install_requires=['setuptools'],
    zip_safe=True,
    maintainer='pw',
    maintainer_email='본인 이메일',
    description='TODO: Package description',
    license='TODO: License declaration',
    extras_require={
        'test': [
            'pytest',
        ],
    },
    entry_points={
        'console_scripts': [
            'hello_node = ros2_basics.hello_node:main',
        ],
    },
)
```

`entry_points`의 `console_scripts` 리스트 안에 `'hello_node = ros2_basics.hello_node:main',` 한 줄이 들어간 것이 이번에 바뀐 부분이다. 이후 주제에서 노드를 추가할 때마다 이 리스트에 같은 형식으로 한 줄씩 추가하게 된다.

### 5. 빌드 & 소스

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros2_basics
source install/setup.bash
```

### 6. 실행

```bash
ros2 run ros2_basics hello_node
```

## 예상/실제 결과

```
Starting >>> ros2_basics
Finished <<< ros2_basics [1.15s]

Summary: 1 package finished [1.28s]
```

```
[INFO] [...] [hello_node]: ros2_basics 패키지, hello_node 정상 실행!
```

실제로 위 로그가 그대로 출력되며 정상 종료됨을 확인했다.

## 알려진 문제와 해결

이번 실습에서는 별도로 발생한 문제 없음.

> **참고**: conda `(base)`가 활성화된 셸에서도 이번 `ament_python` 빌드는 문제없이 진행됐다. 다만 이후 커스텀 메시지(4번 주제, `ament_cmake`/`rosidl` 빌드)에서는 conda의 `CONDA_PREFIX`가 CMake의 Python 탐색에 끼어들어 빌드가 실패할 수 있으니, 그 때는 `conda deactivate` 후 진행한다.

## 체크포인트

- [ ] `ros2 pkg create --build-type ament_python`으로 패키지를 생성할 수 있다.
- [ ] `setup.py`의 `entry_points`가 왜 필요한지 설명할 수 있다.
- [ ] `colcon build --packages-select`로 특정 패키지만 빌드할 수 있다.
- [ ] `ros2 run <패키지> <실행이름>`으로 노드를 실행하고 로그를 확인했다.

---
다음: [`02-node-pub-sub.md`](./02-node-pub-sub.md)
