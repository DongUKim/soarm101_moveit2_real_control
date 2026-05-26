# SO-ARM101 MoveIt2 Real Control

[![ROS2](https://img.shields.io/badge/ROS2-Humble-blue)](https://docs.ros.org/en/humble/)
[![MoveIt2](https://img.shields.io/badge/MoveIt-2-orange)](https://moveit.picknik.ai/)
[![License](https://img.shields.io/badge/license-BSD--3--Clause-green)](#license)

MoveIt2와 PILZ Industrial Motion Planner를 사용해 [LeRobot **SO-ARM101**](https://github.com/huggingface/lerobot) 5-DOF 매니퓰레이터의 경로를 계획하고, 이를 **실제 로봇(Feetech 모터)** 에서 재생·제어하는 ROS 2 워크스페이스입니다.

> 워크플로우: **MoveIt2로 경로 계획 → Joint Trajectory를 YAML로 저장 → 실제 SO-ARM101에서 재생/실시간 제어**

---

## 주요 기능

- **MoveIt2 모션 플래닝** — PILZ Industrial Motion Planner (PTP / LIN) 기반 경로 계획
- **Trajectory 저장/재생** — 계획된 Joint Trajectory를 메타데이터와 함께 YAML로 직렬화
- **실제 로봇 제어** — LeRobot / Feetech 모터 버스를 통해 물리 SO-ARM101 구동
  - YAML 트래젝토리 재생 (가감속 S-curve 프로파일, 관절 부호/오프셋 보정)
  - IK 기반 실시간 제어 (`ikpy`)
  - 키보드 텔레오퍼레이션
- **RViz 시각화** — MoveIt 데모 및 인터랙티브 마커 기반 그리퍼 포즈 제어

---

## 패키지 구성

| 패키지 | 설명 |
|--------|------|
| `dt_arm_description` | SO-ARM101(40mm UP 버전) URDF / 메시 / 로봇 디스크립션 |
| `arm_moveit_config` | SO-ARM101용 MoveIt2 설정 (SRDF, kinematics, PILZ, 컨트롤러) |
| `dt_arm_moveit_config` | 대체 디스크립션(`dt_arm_description`) 기반 MoveIt2 설정 |
| `soarm101_trajectory_planner` | 경로 계획 + YAML 저장/재생 + 실제 로봇 제어 노드/스크립트 |

---

## 요구 사항

- Ubuntu 22.04 + [ROS 2 Humble](https://docs.ros.org/en/humble/)
- MoveIt 2 및 PILZ 플래너
- 실제 로봇 제어용: [LeRobot](https://github.com/huggingface/lerobot) (Feetech 모터 드라이버), `ikpy`, `pyserial`

```bash
# ROS 2 / MoveIt 의존성
sudo apt install ros-humble-moveit ros-humble-moveit-planners-pilz

# 실제 로봇 제어용 Python 의존성
pip install lerobot ikpy pyyaml numpy
```

> ⚠️ 이 저장소에는 MoveIt2 소스 패키지가 포함되어 있지 않습니다. apt 바이너리(`ros-humble-moveit`)를 사용하거나, 소스 빌드가 필요하면 [moveit2](https://github.com/moveit/moveit2)를 워크스페이스에 추가로 clone 하세요.

---

## 설치 및 빌드

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src
git clone -b ver1 https://github.com/DongUKim/soarm101_moveit2_real_control.git

# clone된 src 폴더 내용을 워크스페이스 src로 이동
mv soarm101_moveit2_real_control/src/* .
rm -rf soarm101_moveit2_real_control

cd ~/ros2_ws
colcon build --packages-select \
  dt_arm_description arm_moveit_config dt_arm_moveit_config soarm101_trajectory_planner
source install/setup.bash
```

---

## 사용법

### 1. MoveIt2 실행 (시뮬레이션 / 플래닝)

```bash
ros2 launch arm_moveit_config demo.launch.py
```

### 2. 경로 계획 후 YAML 저장

```bash
# XYZ 좌표 목표
ros2 run soarm101_trajectory_planner plan_trajectory.py --x 0.2 --y 0.1 --z 0.15

# Named target (SRDF 정의 포즈)
ros2 run soarm101_trajectory_planner plan_trajectory.py --target home

# 여러 waypoint 배치 계획
ros2 run soarm101_trajectory_planner batch_planner.py -c waypoints.yaml
```

자세한 옵션과 YAML 포맷은 [`src/soarm101_trajectory_planner/README.md`](src/soarm101_trajectory_planner/README.md) 참고.

### 3. 실제 SO-ARM101에서 재생

```bash
cd ~/ros2_ws/src/soarm101_trajectory_planner/play_so101

# YAML 트래젝토리 재생 (포트는 환경에 맞게 변경)
python play_yaml_trajectory_so101.py --yaml trajectory.yaml --port /dev/ttyACM0

# 실제 모터 구동 없이 값만 확인 (dry-run)
python play_yaml_trajectory_so101.py --yaml trajectory.yaml --dry-run

# 관절 방향이 반대인 경우 부호 보정
python play_yaml_trajectory_so101.py --yaml trajectory.yaml \
    --joint-signs "shoulder_lift:-1,elbow_flex:-1"
```

### 4. 키보드 / IK 실시간 제어

```bash
python keyboard_so101_fixed.py      # 키보드 텔레오퍼레이션
python so101_ik_control.py          # IK 기반 실시간 제어
```

---

## 좌표계 및 Named Targets

```
        Z (위)
        |
        +------ Y (왼쪽)
       /
      X (앞)
```
Base link는 로봇 베이스 중심, end effector(`gripper_link`)의 위치가 목표 좌표입니다.

| Arm target | 설명 | Gripper target | 설명 |
|------------|------|----------------|------|
| `home` | 초기 대기 자세 | `open` | 그리퍼 열림 |
| `zero` | 모든 관절 0도 | `close` | 그리퍼 닫힘 |

---

## 트러블슈팅

- **No valid motion plan found** — 목표가 워크스페이스 내인지, 충돌 없는 경로가 가능한지 확인
- **IK solution not found** — 목표 orientation 도달 가능 여부, kinematics solver timeout 확인
- **MoveIt 연결 실패** — `ros2 node list | grep move_group`으로 move_group 실행 확인
- **시리얼 포트 권한** — `sudo usermod -aG dialout $USER` 후 재로그인, 포트(`/dev/ttyACM*`) 확인

---

## License

- `soarm101_trajectory_planner`, `arm_moveit_config`, `dt_arm_moveit_config`: BSD-3-Clause
- `dt_arm_description`: Apache-2.0
- `play_so101` 내 LeRobot 기반 스크립트: Apache-2.0 (© HuggingFace Inc.)

## Maintainer

deeptree (deeptree00@gmail.com)
