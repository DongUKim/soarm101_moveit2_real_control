# soarm101_moveit_driver

MoveIt2의 **Plan & Execute**가 실제 SO-ARM101(Feetech/LeRobot) 모터를 **바로 구동**하게 해주는 직접 구동 브리지입니다.

기존 워크플로우(좌표 확인 → `plan_trajectory.py`로 **YAML 기록** → `play_yaml_trajectory_so101.py`로 **재생**)의 중간 단계를 없애고, RViz에서 목표를 잡고 Execute만 누르면 실제 팔이 움직이도록 단순화합니다.

```
기존:  RViz 플래닝 ──> YAML 저장 ──> 재생 스크립트 ──> 실제 로봇
신규:  RViz 플래닝 ──(Plan & Execute)──> 실제 로봇   (YAML 불필요)
```

## 동작 원리

`moveit_motor_bridge` 노드가 MoveIt의 simple controller manager와 매칭되는 액션 서버를 띄우고, 받은 trajectory를 rad→deg 변환(+부호/오프셋 보정) 후 LeRobot `SO101Follower.send_action()`으로 스트리밍합니다. 동시에 실제 엔코더를 `/joint_states`로 publish 하여 RViz와 `gripper_pose_monitor`가 실제 팔 상태를 그대로 표시합니다.

| 인터페이스 | 타입 | 용도 |
|-----------|------|------|
| `/arm_trajectory_controller/follow_joint_trajectory` | `control_msgs/FollowJointTrajectory` | 팔 5축 실행 |
| `/gripper_action_controller/gripper_cmd` | `control_msgs/GripperCommand` | 그리퍼 실행 |
| `/joint_states` | `sensor_msgs/JointState` | 실제 모터 위치 피드백 |

> 액션 이름은 `arm_moveit_config/config/moveit_controllers.yaml`의 컨트롤러 정의와 일치하므로, move_group이 별도 설정 없이 이 브리지로 명령을 보냅니다. mock `ros2_control`은 사용하지 않습니다.

## 실행

```bash
# 실제 로봇 (포트는 환경에 맞게)
ros2 launch soarm101_moveit_driver real.launch.py port:=/dev/ttyACM0

# 로봇 없이 RViz 파이프라인만 확인 (가상 모터)
ros2 launch soarm101_moveit_driver real.launch.py dry_run:=true
```

RViz가 뜨면 인터랙티브 마커로 목표 자세를 잡고 **Plan & Execute**를 누르면 실제 팔이 그대로 따라 움직입니다. 그리퍼는 Planning 그룹을 `gripper`로 바꿔 open/close를 Execute 하면 됩니다.

별도 터미널에서 좌표 확인도 동시에 가능합니다:

```bash
ros2 run soarm101_trajectory_planner gripper_pose_monitor.py
```

## 파라미터

`config/driver_params.yaml`에서 조정하거나 launch 인자로 덮어쓸 수 있습니다.

| 파라미터 | 기본값 | 설명 |
|----------|--------|------|
| `port` | `/dev/ttyACM0` | Feetech 시리얼 포트 |
| `robot_id` | `my_follower_arm` | LeRobot 캘리브레이션 ID |
| `dry_run` | `false` | true면 실제 모터 없이 RViz 표시만 |
| `control_freq` | `50.0` | trajectory 스트리밍 주파수(Hz) |
| `state_pub_rate` | `30.0` | `/joint_states` publish 주파수(Hz) |
| `signs.<joint>` | play_yaml과 동일 | URDF↔모터 방향 보정 (±1) |
| `offsets_deg.<joint>` | `0.0` | URDF 0점↔모터 0점 오프셋(deg) |

> 부호/오프셋은 기존 `play_yaml_trajectory_so101.py`의 `DEFAULT_JOINT_SIGNS`/`OFFSETS`와 동일한 기본값을 사용합니다. 실제 로봇에서 방향이 반대로 움직이면 해당 관절의 `signs`를 뒤집으세요.

## 의존성

```bash
sudo apt install ros-jazzy-moveit ros-jazzy-pilz-industrial-motion-planner
pip install lerobot
```

## 안전 참고

- 처음에는 반드시 `dry_run:=true`로 RViz 동작과 trajectory를 확인한 뒤 실제 로봇에 연결하세요.
- 실제 실행 전, 워크스페이스에 장애물이 없는지·비상정지(전원 차단) 수단이 가까운지 확인하세요.
- `control_freq`를 너무 높이면 시리얼 버스가 포화될 수 있습니다. 50Hz 전후를 권장합니다.
