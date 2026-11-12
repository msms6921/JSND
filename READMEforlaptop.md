# 원격 노트북에서 pitwall 실행하기

대상은 **Ubuntu 22.04 / x86_64 노트북**입니다. 시스템에 설치된 ROS 2 Humble은
그대로 두고, 노트북의 별도 Conda 환경에 ROS 2 Jazzy를 설치해 Jetson과 통신합니다.
Jetson은 기존 시스템 Jazzy를 계속 사용합니다. 이 문서의 설치·빌드 명령은
별도 표시가 없는 한 **노트북의 로컬 데스크톱 터미널**에서 실행하세요.
SSH로 Jetson에 접속한 터미널에서 실행하는 절차가 아닙니다.

이 구성은 원격 RViz와 rqt 파라미터 창을 위한 것입니다. 노트북에서 차량의
`race.launch.xml`, `mapping.launch.xml`이나 모터·LiDAR 드라이버를 실행하지 않습니다.

확인한 범위: 프로젝트의 launch/CMake/패키지 선언과 RoboStack의 linux-64 패키지
배포 여부를 확인했습니다. **실제 Ubuntu 22.04 노트북에서 Conda 의존성 해결,
빌드, GUI 및 Jetson 데이터 표시까지 검증한 문서는 아닙니다.** 과거 Humble과
Jetson 사이의 문자열 토픽 통신 확인은 이 Conda 구성의 검증을 대신하지 않습니다.

## 1. 기본 도구 준비

```bash
hostname
uname -m
cat /etc/os-release
```

Ubuntu 22.04와 `x86_64`인지 확인합니다. ARM 노트북의 Conda 패키지 조합은 별도로
확인해야 합니다. GitHub의 `msms6921/MSND`는 비공개이므로 소유자 계정이나
초대를 수락한 collaborator 계정이 필요합니다.

노트북에서 도구가 없을 경우 관리자 권한으로 설치합니다. ROS Humble을 제거하지 않습니다.

```bash
sudo apt-get install --no-install-recommends git curl ca-certificates gh
```

## 2. Humble을 상속하지 않는 셸 열기

일반 터미널에서 다음 명령으로 독립된 셸에 들어갑니다. **새 터미널에서 작업할
때마다 이 단계부터 시작**합니다. `bash --norc`만으로는 이미 상속한 Humble
환경 변수가 없어지지 않으므로 `env -i`도 사용합니다.

```bash
env -i \
  HOME="$HOME" USER="$USER" LOGNAME="$USER" SHELL=/bin/bash \
  TERM="${TERM:-xterm-256color}" LANG="${LANG:-C.UTF-8}" \
  PATH=/usr/local/bin:/usr/bin:/bin \
  DISPLAY="${DISPLAY-}" XAUTHORITY="${XAUTHORITY-}" \
  WAYLAND_DISPLAY="${WAYLAND_DISPLAY-}" \
  XDG_RUNTIME_DIR="${XDG_RUNTIME_DIR-}" \
  DBUS_SESSION_BUS_ADDRESS="${DBUS_SESSION_BUS_ADDRESS-}" \
  /bin/bash --noprofile --norc
```

이후에는 `/opt/ros/humble/setup.bash`, 기존 Humble 워크스페이스의 setup,
`~/.bashrc`를 source하지 않습니다. 셸을 나가면 기존 Humble 터미널로 돌아갑니다.
시스템 `.bashrc`와 기본 ROS 배포판을 수정할 필요가 없습니다.

## 3. Miniforge 설치와 Jazzy 환경 생성 — 최초 1회

이미 `~/miniforge3`가 있으면 설치 부분을 건너뛰고 기존 설치를 사용합니다.
다른 위치에 설치했다면 아래 `conda.sh` 경로를 실제 위치로 바꿉니다.

새로 설치하는 경우, 2번에서 연 깨끗한 셸에서 실행합니다.

```bash
if [ ! -e "$HOME/miniforge3" ]; then
  msnd_installer_dir="$(mktemp -d /tmp/msnd-miniforge.XXXXXX)"
  msnd_miniforge_version=26.9.4-2
  msnd_miniforge_file="Miniforge3-${msnd_miniforge_version}-Linux-x86_64.sh"
  (
    cd "$msnd_installer_dir" &&
    curl -fLO "https://github.com/conda-forge/miniforge/releases/download/${msnd_miniforge_version}/${msnd_miniforge_file}" &&
    curl -fLO "https://github.com/conda-forge/miniforge/releases/download/${msnd_miniforge_version}/${msnd_miniforge_file}.sha256" &&
    sha256sum -c "${msnd_miniforge_file}.sha256" &&
    bash "$msnd_miniforge_file" -b -p "$HOME/miniforge3"
  )
fi

source "$HOME/miniforge3/etc/profile.d/conda.sh"
```

체크섬이나 설치가 실패하면 다음 단계로 진행하지 말고 출력된 오류를 확인하세요.
`conda init`은 실행하지 않습니다. 아래 환경 생성은 `base`와 기존 `thirdimpact`
환경을 바꾸지 않고 `msnd_pitwall`을 만듭니다. 이미 같은 이름의 환경이 있다면
새로 만들거나 덮어쓰지 말고 먼저 `conda env list`로 확인하세요.

```bash
conda env list

conda create -y -n msnd_pitwall \
  --override-channels -c conda-forge -c robostack-jazzy \
  python=3.12 "cmake<4" "setuptools<80" \
  compilers make pkg-config colcon-common-extensions \
  "qt-main=5.*" \
  ros-jazzy-desktop \
  ros-jazzy-rqt-reconfigure \
  ros-jazzy-launch-xml \
  ros-jazzy-rmw-cyclonedds-cpp \
  ros-jazzy-ament-cmake-python \
  ros-jazzy-rosidl-default-generators

conda activate msnd_pitwall
```

`desktop`은 RViz와 기본 ROS 메시지/개발 패키지를 제공합니다. rqt 재설정 창과
CycloneDDS는 명시적으로 설치합니다. 프로젝트의 pitwall 패널은 Qt5를 사용합니다.
패키지 조합이 해결되지 않으면 pin을 임의로 없애거나 Humble 패키지를 섞지 말고
solver 오류를 확인해야 합니다. 위 명령은 노트북에서 설치 성공을 확인한 lock 파일이 아닙니다.

```bash
echo "$ROS_DISTRO"                # jazzy
command -v python                 # .../envs/msnd_pitwall/bin/python
command -v ros2                   # .../envs/msnd_pitwall/bin/ros2
python -c 'import sys; print(sys.executable)'
ros2 pkg prefix rviz2
ros2 pkg prefix rqt_reconfigure
ros2 pkg prefix rmw_cyclonedds_cpp
```

`ROS_DISTRO`가 `humble`이거나 위 패키지가 `/opt/ros/humble`에서 검색되면 중단하고
2번부터 다시 시작합니다. Conda가 ROS 환경을 설정하므로 시스템 ROS setup은 불러오지 않습니다.

## 4. 비공개 저장소 다운로드 — 최초 1회

같은 깨끗한 셸에서 GitHub 인증을 진행합니다. CLI가 표시하는 일회용 코드를
브라우저의 <https://github.com/login/device>에 입력합니다.

```bash
gh auth login --hostname github.com --git-protocol https --web
gh auth setup-git

mkdir -p "$HOME/MSND_laptop/src"
git clone https://github.com/msms6921/MSND.git \
  "$HOME/MSND_laptop/src/racing_stack"
```

이미 해당 위치에 clone했다면 다시 clone하지 않습니다. 필요한 업데이트는 로컬 수정
상태를 먼저 확인한 뒤 `git pull --ff-only`로 받습니다. Jetson의 `build/install`은
복사하지 않습니다. CPU 아키텍처와 ROS/Python 라이브러리가 다르므로 노트북에서 빌드합니다.

## 5. 화면에 필요한 프로젝트 패키지만 빌드

Conda `msnd_pitwall`이 활성화된 상태에서 실행합니다.

```bash
cd "$HOME/MSND_laptop"
CMAKE_BUILD_PARALLEL_LEVEL=2 MAKEFLAGS=-j2 colcon build \
  --base-paths src/racing_stack/stack_master src/racing_stack/race_utils/pitwall \
  --packages-select stack_master pitwall \
  --executor sequential --symlink-install \
  --cmake-args -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=OFF \
    -DPython3_EXECUTABLE="$CONDA_PREFIX/bin/python"

source "$HOME/MSND_laptop/install/local_setup.bash"
ros2 pkg prefix stack_master
ros2 pkg prefix pitwall
```

메모리가 부족하면 병렬도 `2`와 `-j2`를 각각 `1`, `-j1`로 낮춥니다.
이 명령은 `stack_master`의 실행 중 사용하지 않는 실차 패키지를 설치하지 않고,
현재 CMake가 요구하는 의존성과 pitwall 패널만 빌드합니다. `rosdep install`을 전체
저장소에 실행하거나 `--packages-up-to stack_master`로 범위를 넓히면 실차용 의존성까지
요구할 수 있습니다.

화면 표시는 표준 Map/Scan/Odometry/Marker 메시지를 사용합니다. `f110_msgs`를
`ros2 topic echo`로 직접 읽으려면 같은 소스 버전의 메시지 패키지를 추가 빌드합니다.

```bash
cd "$HOME/MSND_laptop"
colcon build --base-paths src/racing_stack/race_utils/f110_msgs \
  --packages-select f110_msgs --executor sequential --symlink-install \
  --cmake-args -DBUILD_TESTING=OFF -DPython3_EXECUTABLE="$CONDA_PREFIX/bin/python"
source "$HOME/MSND_laptop/install/local_setup.bash"
```

보기만 하는 구성에는 Gymnasium, f110_gym, Numba, range_libc, 차량 드라이버가
필요하지 않습니다. `thirdimpact.sh`, `setup_conda_onLaptop.sh`, `requirements.txt`,
전체 `environment.yml`은 시뮬레이션 등 더 넓은 설치를 수행하므로 이 절차에서는 사용하지 않습니다.

## 6. 노트북 DDS 설정 — 최초 1회

Jetson과 노트북을 서로 통신 가능한 네트워크에 연결합니다. 작업 당시 Jetson IP는
`192.168.50.228`, 노트북은 `192.168.50.139`였지만 DHCP에 따라 달라질 수 있습니다.
아래 주소를 먼저 실제 Jetson IP로 확인/수정하세요.

```bash
ip -br address
ip -4 route get 192.168.50.228
ping -c 3 192.168.50.228
```

아래는 해당 Jetson으로 가는 실제 인터페이스를 선택해 **노트북 전용** XML을 만듭니다.
`wlo1` 같은 이름을 추측하거나 Jetson의 네트워크 이름을 복사하지 않습니다.

```bash
msnd_netif="$(ip -4 route get 192.168.50.228 | awk '{for(i=1;i<=NF;i++) if($i=="dev"){print $(i+1);exit}}')"
if [ -n "$msnd_netif" ]; then
  mkdir -p "$HOME/.config/msnd"
  if [ -f "$HOME/.config/msnd/cyclonedds-laptop.xml" ]; then
    cp -p "$HOME/.config/msnd/cyclonedds-laptop.xml" \
      "$HOME/.config/msnd/cyclonedds-laptop.xml.bak.$(date +%Y%m%d_%H%M%S)"
  fi
  cat > "$HOME/.config/msnd/cyclonedds-laptop.xml" <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<CycloneDDS xmlns="https://cdds.io/config">
  <Domain id="any">
    <General>
      <Interfaces>
        <NetworkInterface name="$msnd_netif" multicast="true"/>
      </Interfaces>
      <AllowMulticast>spdp</AllowMulticast>
    </General>
    <Discovery><ParticipantIndex>none</ParticipantIndex></Discovery>
  </Domain>
</CycloneDDS>
EOF
else
  echo 'Jetson으로 가는 네트워크 인터페이스를 확인하지 못했습니다. IP와 연결을 확인하세요.'
fi
```

노트북 파일에는 Jetson용 XML의 강제 10MB 소켓 버퍼 조건을 넣지 않았습니다.
노트북/공유기가 바뀌면 인터페이스와 경로를 다시 확인합니다. 이 설정은 멀티캐스트
discovery를 사용하므로 AP의 client isolation이나 multicast 차단이 있으면 통신되지 않을 수 있습니다.

## 7. 매번 실행하는 순서

노트북 데스크톱의 새 터미널에서 **2번 명령으로 깨끗한 셸**을 연 뒤 다음을 실행합니다.

```bash
source "$HOME/miniforge3/etc/profile.d/conda.sh"
conda activate msnd_pitwall
source "$HOME/MSND_laptop/install/local_setup.bash"

export ROS_DOMAIN_ID=173
unset ROS_LOCALHOST_ONLY
export ROS_AUTOMATIC_DISCOVERY_RANGE=SUBNET
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
export CYCLONEDDS_URI="file://$HOME/.config/msnd/cyclonedds-laptop.xml"

echo "$ROS_DISTRO / $ROS_DOMAIN_ID / $RMW_IMPLEMENTATION"
echo "$DISPLAY"
ros2 launch stack_master pitwall.launch.xml race:=true
```

Jetson 측도 동일한 도메인 `173`, SUBNET discovery와 통신 가능한 DDS 설정을
사용해야 합니다. 기존 Jetson `.bashrc`에는 이 설정이 있습니다. 이미 실행 중인
노드의 환경은 새 터미널의 export로 바뀌지 않으며, 이 문서를 위해 실차 프로세스를
임의로 재시작하지 않습니다.

`race:=true`는 주행용 경량 표시 구성입니다. 정차 상태에서 전체 화면이 필요하면
`race:=true`를 생략합니다. GUI는 노트북에 표시되어야 합니다. `$DISPLAY`가 비어 있고
사용 가능한 GUI 세션도 없다면 로컬 데스크톱 터미널에서 다시 시작하세요.
원격 GUI 전송이나 OpenGL 설정은 이 절차에서 검증하지 않았습니다.

RViz와 rqt의 종료는 launch를 실행한 터미널에서 `Ctrl+C`입니다. 다른 터미널에서
시작한 차량 노드는 이 명령으로 종료되지 않습니다. 환경 작업을 마치려면
`conda deactivate`, `exit`로 원래 Humble 셸에 돌아갑니다.

## 8. 통신과 표시 확인

추가 노트북 터미널도 2번 및 7번의 환경 설정을 동일하게 적용합니다. 마지막
pitwall launch 명령은 중복 실행하지 않습니다. 먼저 노트북의 CLI daemon만 갱신합니다.

```bash
ros2 daemon stop
ros2 node list
ros2 topic list
```

차량 launch가 실행 중일 때 코드에서 사용하는 토픽을 확인합니다.

```bash
ros2 topic info /map --verbose
ros2 topic info /car_state/odom --verbose
ros2 topic echo /car_state/odom nav_msgs/msg/Odometry --once
ros2 run tf2_ros tf2_echo map base_link
```

맵핑 중에는 `/map_carto`를 확인합니다. 실제 가동 중인 launch에 따라 존재하는
토픽이 다릅니다. 목록에 보이는 것과 데이터가 실제 수신되는 것은 별도로 확인해야 합니다.
RViz의 Fixed Frame은 `map`이며, TF 및 Map/Scan/Odometry 표시 상태를 확인합니다.
기존 실차와 시뮬레이터는 벽시계를 사용하므로 이 구성에서 `use_sim_time`을
임의로 켜지 않습니다. 정상적인 실차 pitwall에 `/clock`은 필요하지 않습니다.

차량을 구동하지 않고 네트워크만 확인하려면 별도 문자열 토픽을 사용합니다.
Jetson 터미널에서 시스템 Jazzy와 기존 환경을 로드한 뒤 먼저 수신을 시작합니다.

```bash
# Jetson: 도메인 173인 환경에서 실행
ros2 topic echo /msnd_pitwall_probe std_msgs/msg/String --once
```

그다음 노트북의 7번 환경에서 반복 발행합니다. 약 15초 뒤 timeout 종료는 정상입니다.

```bash
# 노트북
timeout 15s ros2 topic pub -r 2 /msnd_pitwall_probe std_msgs/msg/String \
  "{data: 'laptop-conda-jazzy'}"
```

반대 방향도 확인하려면 노트북에서 같은 echo를 먼저 실행하고, Jetson에서 같은
pub 명령을 실행하되 내용을 `jetson-system-jazzy`로 바꿉니다. 이 명령은 모터나
조향 제어 토픽에 발행하지 않습니다. 두 방향 모두 수신되는지 확인하세요.

## 9. 오류 확인

| 증상 | 확인/조치 |
|---|---|
| `Repository not found` | 비공개 저장소 권한, collaborator 초대 수락, `gh auth status`의 계정을 확인합니다. |
| Conda solver 실패/패키지 없음 | 인터넷, linux-64 지원, 지정 채널과 Python/Qt 버전 충돌을 확인합니다. 이 문서의 조합은 실제 노트북에서 미검증입니다. |
| `package 'stack_master' not found` | 현재 셸에서 Conda 활성화 후 노트북 `install/local_setup.bash`를 로드했는지 확인합니다. |
| `pitwall/PitwallPanel` 로딩 실패 | 노트북에서 `pitwall` 빌드 성공 여부, 사용 중인 RViz가 Conda Jazzy인지, Qt5 및 공유 라이브러리 오류를 확인합니다. Jetson의 `.so`를 복사하지 않습니다. |
| `Failed to find a free participant index` | XML 파일의 존재와 `CYCLONEDDS_URI`, `ParticipantIndex=none` 적용 여부를 확인합니다. 새 설정을 적용한 노트북 pitwall만 재실행합니다. |
| `does not match an available interface` | `ip -br address`와 XML의 인터페이스 이름을 비교하고 6번을 다시 적용합니다. |
| 소켓 버퍼 최소 크기 오류 | 저장소의 Jetson용 XML을 잘못 로드했는지 먼저 확인합니다. 필요 시 `sysctl net.core.rmem_max net.core.wmem_max` 값을 확인해 오류가 요구하는 크기보다 작은 경우에만 노트북 관리자와 조정합니다. |
| 토픽이 없거나 수신되지 않음 | 양쪽 도메인/RMW/discovery, IP 연결, VPN/다중 NIC, 방화벽 및 공유기 client isolation/multicast 정책을 확인합니다. 방화벽 전체를 끄지 않습니다. |
| `cannot connect to display`, Qt xcb/OpenGL 오류 | 노트북 로컬 GUI 세션인지, `$DISPLAY`, 그래픽 드라이버 및 누락된 Qt 라이브러리를 확인합니다. Humble Qt/라이브러리 경로를 Conda에 추가하지 않습니다. |
| RViz `No transform` | Jetson의 실제 TF와 Fixed Frame `map`, `/tf`·`/tf_static` 수신 상태를 확인합니다. |

launch가 출력하는 `~/.ros/log/...`의 로그와 RViz Displays의 Status를 확인합니다.
패키지 설치/빌드 성공, 노드 발견, 실제 데이터 수신, 화면 표시를 각각 구분해서
기록하세요. 차량 제어 파라미터 변경이나 주행 명령은 이 문서의 확인 절차에 포함하지 않습니다.

설치가 성공한 노트북에서 버전을 보관하려면 다음을 실행합니다. GitHub 인증 정보는
기록하지 않습니다. explicit 파일은 해당 OS/CPU용 설치 기록입니다.

```bash
conda list --explicit > "$HOME/MSND_laptop/conda-linux-64-explicit.txt"
```

## 참고 자료

- [Miniforge 공식 설치 안내](https://github.com/conda-forge/miniforge#install)
- [RoboStack Conda 안내 — 시스템 ROS와 분리](https://robostack.github.io/conda.html)
- [RoboStack Jazzy 패키지 목록](https://robostack.github.io/jazzy.html)
- 프로젝트의 `stack_master/launch/pitwall.launch.xml`, `race_utils/pitwall/package.xml`,
  `race_utils/pitwall/CMakeLists.txt`를 기준으로 빌드 범위를 정했습니다.
