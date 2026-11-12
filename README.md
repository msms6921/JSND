# JSND — 시스템 ROS 2 Jazzy 워크스페이스

기존 코드와 launch 파일을 그대로 사용하며, 시스템 Python과 ROS 2 Jazzy로 빌드합니다.
워크스페이스는 일반적인 구조로 통일했습니다:

```text
~/JSND/
  src/racing_stack/
  build/
  install/
  log/
  .vscode/
```

현재는 표준 `build/install/log`만 사용합니다. 이번 작업에서 만들었던
`build_jazzy_sim`, `install_jazzy_sim` 및 `*_jazzy_system` 디렉터리는 사용하지 않아
백업 위치에서도 삭제했습니다. 설정 원본과 나머지 복구 자료는
`~/JSND_backups/20261112_141844/`에 보존했습니다.
새 실행 스크립트와 별도 의존성 ROS 패키지는 추가하지 않았습니다.

## 실차 실행

사용자가 실행했던 다음 명령을 그대로 사용합니다:

```bash
ros2 launch stack_master race.launch.xml map:=test0
```

이 launch의 기본값은 `sim:=false`, `localization:=pf`입니다.
VESC, 조향 변환기, 실제 LiDAR와 PF/EKF가 필요한 구성입니다.
기존 `vesc_driver not found` 오류는 앞선 설치 결과가 시뮬레이션 패키지만 포함하여
VESC를 빌드하지 않았기 때문입니다. VESC·시리얼·LiDAR·PF를 포함해 기존 패키지
36개를 시스템 Jazzy로 새 `build/install`에 빌드했습니다.

`.bashrc`가 시스템 Jazzy와 `~/JSND/install/local_setup.bash`를 로드합니다.
이미 열려 있던 터미널에서는 표준 설치 결과를 다시 로드하세요:

```bash
source ~/JSND/install/setup.bash
ros2 pkg prefix vesc_driver
# /home/jsnd/JSND/install/vesc_driver
```

터미널이나 VS Code 원격 서버가 이전 환경 변수를 상속한 상태라면 새 로그인 세션에서
정리된 설정을 확인하세요. 실차 launch를 중지할 때는 해당 터미널에서 `Ctrl+C`를 사용합니다.

## .bashrc에 남긴 설정

시뮬레이션 전용 `RAYCASTER_*`, `PYGLET_SHADOW_WINDOW`, 스레드 제한,
`PYTHONNOUSERSITE` 전역 설정은 제거했습니다. 남긴 설정은 시스템 ROS 로딩,
일반 워크스페이스 로딩, 다음 ROS 네트워크 설정입니다:

```bash
export ROS_DOMAIN_ID=173
unset ROS_LOCALHOST_ONLY
export ROS_AUTOMATIC_DISCOVERY_RANGE=SUBNET
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
export CYCLONEDDS_URI="file://$HOME/JSND/src/racing_stack/cyclonedds.xml"
```

노트북도 같은 도메인을 사용해야 합니다. 이전에 Jetson `192.168.50.228`과
Humble 노트북 `192.168.50.139` 사이에서 도메인 173의 양방향 문자열 토픽을 확인했습니다.
노트북 설정 파일은 변경하지 않았습니다. 노트북 RViz/custom 메시지 전체 연동은 미검증입니다.

`cb`, `cbs`, `cbp`, `sb`는 일반 `build/install/log`를 사용하도록 정리했습니다.
`cb`는 `~/JSND`로 이동하여 빌드하므로 `src/` 아래 중복 build/install을 만들지 않습니다.
선택적 패키지까지 포함한 전체 빌드 대신 이번에 확인한 실차 PF 구성은 아래 명령으로
재현할 수 있습니다.

## RViz와 DDS participant 인덱스 오류

별도 GUI 터미널에서 실행합니다. 주행 중에는 경량 표시 설정을 사용합니다:

```bash
source ~/.bashrc
ros2 launch stack_master pitwall.launch.xml race:=true
```

정차 상태에서 전체 표시와 rqt 파라미터 창을 열려면 `race:=true`를 생략합니다.

Jazzy rmw_cyclonedds의 기본 discovery 설정은 자동 participant 인덱스 최대값을 32로
설정합니다. 주행 노드에 joystick, ROS CLI daemon, RViz, rqt 등이 추가되면
`Failed to find a free participant index`가 발생할 수 있습니다.
이는 rqt 플러그인 로딩 실패보다 앞선 DDS 노드 초기화 실패입니다.
[ROS Jazzy RMW 소스](https://github.com/ros2/rmw_cyclonedds/blob/jazzy/rmw_cyclonedds_cpp/src/rmw_node.cpp)에서 기본값을 확인했습니다.

저장소에 원래 있던 `cyclonedds.xml`은 `ParticipantIndex=none`을 지정합니다.
이 파일을 `.bashrc`의 `CYCLONEDDS_URI`로 읽도록 복구했습니다. 별도 스크립트나
launch 변경, 재빌드 없이 새 프로세스에 적용됩니다. 기존 실행 중인 프로세스의
환경은 바뀌지 않으므로 이후 실행 터미널에서 `.bashrc`를 다시 불러오세요.

기존 설정은 loopback과 노트북 통신용 WiFi를 선택합니다. 다른 PC용 optional interface가
없다는 안내가 출력될 수 있지만 필수 인터페이스 오류는 아닙니다.
호스트의 rmem/wmem 최대값 25 MiB가 XML의 10 MB 최소 버퍼 조건을 충족하는 것도
확인했으며 시스템 설정 변경은 하지 않았습니다.

테스트 도메인 174에서 독립 ROS 프로세스 40개를 생성하여 기본 설정에서는 8개가
인덱스 오류로 실패하고, XML 설정을 적용하면 40개 모두 생성되는 것을 확인했습니다.
테스트 프로세스만 종료했으며 차량 드라이버는 시작하거나 종료하지 않았습니다.
로그는 `~/JSND/log/dds_capacity_20261112_145952/result.json`에 있습니다.

## 재빌드

```bash
cd ~/JSND
CMAKE_BUILD_PARALLEL_LEVEL=1 MAKEFLAGS=-j1 colcon build \
  --base-paths src/racing_stack/{stack_master,controller,state_machine,perception} \
    src/racing_stack/prediction/gp_traj_predictor \
    src/racing_stack/planner/{gb_optimizer,spliner,recovery_spliner,lane_change_planner} \
    src/racing_stack/race_utils/{pitwall,f110_msgs} \
    src/racing_stack/race_utils/unicorn_gym/{f1tenth_gym_ros,virtual_perception} \
    src/racing_stack/race_utils/f110_utils/libs/{frenet_conversion,frenet_conversion_msgs,grid_filter,vel_planner} \
    src/racing_stack/race_utils/f110_utils/nodes/{sector_tuner,overtaking_sector_tuner,frenet_conversion_server,frenet_odom_republisher,lap_analyser} \
    src/racing_stack/sensor/{vesc,transport_drivers,laser_proc,urg_c,urg_node,urg_node_msgs,elrs_joy} \
    src/racing_stack/state_estimation/particle_filter \
  --executor sequential --symlink-install \
  --cmake-args -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=OFF -DPython3_EXECUTABLE=/usr/bin/python3
source ~/JSND/install/local_setup.bash
```

빌드 병렬도는 Jetson 메모리를 고려해 1개로 제한했습니다. Cartographer/KISS/SQP 등
다른 선택적 구성까지 이번 명령이 빌드하는 것은 아닙니다.

## 시스템 의존성

Ubuntu 24.04.5 / aarch64, 시스템 ROS `/opt/ros/jazzy`, `/usr/bin/python3` 3.12.3,
`/usr/bin/colcon`을 사용합니다. OS·JetPack·CUDA·NVIDIA 드라이버는 교체하지 않았습니다.
Conda·가상환경·Docker를 활성화하지 않습니다.

관리자 명령은 사용자가 직접 실행했습니다. 공통 의존성:

```bash
sudo apt-get install --no-upgrade --no-install-recommends \
  ros-jazzy-ackermann-msgs ros-jazzy-xacro ros-jazzy-tf-transformations \
  ros-jazzy-rmw-cyclonedds-cpp python3-transforms3d python3-cloudpickle \
  python3-pybind11 pybind11-dev python3-pynput python3-tqdm \
  python3-sklearn python3-networkx python3-build python3-installer
```

추가 실차 의존성:

```bash
sudo apt-get install --no-upgrade --no-install-recommends \
  libasio-dev ros-jazzy-diagnostic-updater ros-jazzy-nav2-map-server \
  ros-jazzy-nav2-lifecycle-manager ros-jazzy-robot-localization
```

apt에 없는 원본 Python 배포물은 PyPA installer로 표준
`/usr/local/lib/python3.12/dist-packages`에 설치했습니다. sudo pip나
`--break-system-packages`는 사용하지 않았습니다.
버전: Gymnasium 0.29.1, Farama Notifications 0.0.4, pyglet 1.5.20,
trajectory-planning-helpers 0.79, quadprog 0.1.14, FilterPy 1.4.5, CCMA 0.0.1,
f110_gym 0.3.0, Numba 0.65.0, llvmlite 0.47.0, coverage 7.10.6, range_libc 0.3.
range_libc는 저장소 원본 C++ 바인딩을 시스템 컴파일러로 빌드한 것입니다.
PF는 이 라이브러리를 직접 import하므로 임의 PYTHONPATH 추가가 필요하지 않습니다.

기존 NumPy/SciPy/OpenCV 등 apt 과학 연산 라이브러리는 유지했습니다.
trajectory-planning-helpers의 오래된 quadprog 0.1.7 고정 대신 현재 Python/aarch64에서
사용 가능한 0.1.14를 사용하며 QP 연산을 확인했습니다.
Numba와 기존 coverage 7.4.4의 API 충돌은 coverage 7.10.6으로 맞춰 JIT 연산을 확인했습니다.
설치 기록과 해시는 위 백업 폴더에 함께 보존했습니다.

## 검증한 범위

- 표준 디렉터리에서 36개 패키지 빌드 성공.
- VESC, Ackermann 변환기, LiDAR, PF 패키지와 실행 파일이 표준 install에서 검색됨.
- VESC·시리얼·LiDAR 라이브러리/실행 파일 13개의 동적 링크와 심볼 확인.
- PF의 시스템 range_libc import, ROS OccupancyGrid 변환과 pcddt raycasting 확인.
- 사용자 명령 `race.launch.xml map:=test0`의 실제 기본값으로 31개 노드 명령과
  파라미터 파일을 펼치는 사전 검증 통과. 이 검증에서는 프로세스 시작을 차단했습니다.
- 실제 차량 드라이버, 모터/조향 노드는 실행하지 않았습니다. 장치 연결, 실제 센서 데이터,
  실차 TF·위치추정·주행 성능은 아직 검증하지 않았습니다.

로그: `~/JSND/log/standard_layout_build.log`, `real_launch_offline_validation.json`,
`real_binary_link_check.json`, `standard_environment_check.json`.
원래 실행 코드와 launch 파일에는 수정이 없습니다.

## 시뮬레이션이 필요한 경우

시뮬레이션용 옵션은 전역 bashrc에서 빼고 해당 실행에만 지정합니다.
실차 도메인 173과 분리한 174를 사용합니다:

```bash
ROS_DOMAIN_ID=174 \
RAYCASTER_DIR="$HOME/JSND/src/racing_stack/race_utils/raycaster" \
PYGLET_SHADOW_WINDOW=False \
ros2 launch stack_master race.launch.xml sim:=true map:=test0
```

원본 시뮬레이션은 RViz와 키보드 노드를 항상 실행하므로 접근 가능한 데스크톱이 필요합니다.
별도 실행 스크립트는 사용하지 않습니다. `f` 맵은 현재 없으므로 복원 전에는 사용할 수 없습니다.

DDS 수정 후 실제 `ros2 launch stack_master pitwall.launch.xml`도 별도로 실행해
RViz와 rqt_reconfigure 두 ROS 노드, NVIDIA OpenGL 4.6 초기화 및 시작 오류가 없는 것을
확인했습니다. 이 검증에서 생성한 GUI와 확인용 노드는 모두 종료했습니다.
차량 노드나 주행 명령은 실행하지 않았습니다.
로그: `~/JSND/log/pitwall_dds_20261112_150259/result.json` 및 같은 폴더의 `launch.log`.
