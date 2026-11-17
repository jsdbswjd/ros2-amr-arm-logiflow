# ROS 2 LogiFlow | AMR–로봇팔 물류 분류

myAGV 2023(AMR)과 myCobot(Robot Arm)을 연동해 주문 접수부터 선반 앞 정렬, 물품 파지, 지정 구역 운반·배치, 주차 복귀까지 수행한 ROS 2 기반 팀 프로젝트입니다.

**담당 | 김윤정** — AMR 주행 제어, 속도 응답 계측 및 분석, ArUco 정렬, 로봇팔 작업 상태 연동

## 최종 시연

선반 앞에서 AMR이 ArUco 마커를 기준으로 정렬합니다. 정렬이 끝나면 로봇팔이 물품을 집고, AMR은 파지 완료 신호를 받아 지정 구역으로 이동합니다. 배치 완료 이후에는 주차 위치로 복귀합니다.

| 선반 이동 | 선반 앞 정렬 |
| :---: | :---: |
| <img width="280" alt="AMR이 선반으로 이동하는 장면" src="https://github.com/user-attachments/assets/fa2d5468-b461-41a6-89c7-13d32e6b39bc" /> | <img width="280" alt="AMR이 선반 앞에 정렬하는 장면" src="https://github.com/user-attachments/assets/2d1777cf-06d4-42e2-a240-0dfa9897fdad" /> |

| 파지 | 운반 및 배치 |
| :---: | :---: |
| <img width="280" alt="로봇팔이 물품을 집는 장면" src="https://github.com/user-attachments/assets/8608d433-bb55-4143-96ee-e26cda7d13a5" /> | <img width="280" alt="물품을 운반해 지정 위치에 놓는 장면" src="https://github.com/user-attachments/assets/56d3055d-65fc-4151-9385-c467db378688" /> |

- [최종 발표 자료](doc/ROS2_AGV로봇팔_최종발표_final.pdf)

## 본인 기여

| 담당 | 구현 및 검증 | 코드 |
| --- | --- | --- |
| 주행 명령 분석 및 속도 필터 | `/cmd_vel_nav`, `/cmd_vel`, 오도메트리 속도를 비교해 명령과 실제 응답의 차이를 확인했습니다. 속도·가속도 제한, 직진·회전 명령 분리, 저속 회전 데드존 보정을 적용했습니다. | [속도 필터](tracer/tracer/vel_filter_node.py) · [주행 로그](tracer/tracer/nav_debug_logger.py) |
| 경로 실행 | 최종 시연 경로를 직진과 회전 동작으로 나누고, 정해진 순서에 따라 구간을 실행하도록 구성했습니다. | [구간 주행](tracer/tracer/primitive_route_runner.py) |
| 선반 앞 정렬 및 상태 연동 | ArUco 마커의 영상상 위치를 이용해 차체를 보정했습니다. 로봇팔 작업 중에는 AMR 이동을 대기하고, 파지·배치 완료 신호에 따라 다음 경로를 실행했습니다. | [ArUco 정렬](tracer/tracer/aruco_align_node.py) · [AMR 미션 제어](tracer/tracer/mission_brain_node.py) |

## 시스템 구성

| 구성 | 역할 |
| --- | --- |
| myAGV 2023 · Jetson Nano | 주행 구간 실행, 속도 명령 보정, 카메라 기반 차체 정렬 |
| Jetson Orin Nano · RealSense · YOLO | 팀 시스템의 주문·작업 흐름 제어 및 물품 위치 인식 |
| myCobot Pi | 물품 파지 및 배치 |
| ROS 2 Humble · Python | 노드 간 명령과 작업 상태 전달 |

팀 시스템의 [`brain_node`](brain_pkg/brain_pkg/brain_node.py)가 주문과 로봇팔 작업을 조율합니다. AMR 쪽의 [`mission_brain_node`](tracer/tracer/mission_brain_node.py)는 목적지와 로봇팔 작업 완료 상태를 받아 이동·대기·복귀 순서를 제어합니다.

## 측정 결과

초기 필터 적용 전후 주행 로그에서 **명령이 활성화된 구간의 속도 추종 RMS 오차**를 비교했습니다.

| 지표 | 적용 전 | 적용 후 | 감소율 |
| --- | ---: | ---: | ---: |
| 선속도 RMS 오차 | 0.0609 m/s | 0.0247 m/s | 59.4% |
| 각속도 RMS 오차 | 0.2290 rad/s | 0.0655 rad/s | 71.4% |

위 수치는 속도 필터 비교 실험의 결과입니다. 최종 시연의 물류 작업 성공률은 별도 지표입니다.

## 실행 경로

- **최종 시연용 `tracer/`** — [`tracer_auto_mission.launch.py`](tracer/launch/tracer_auto_mission.launch.py)가 AMR 미션, 직진·회전 구간 실행, ArUco 정렬을 시작합니다. 실행 전 [myAGV 구동·위치 추정·속도 필터 구성](vendor/elephantrobotics/myagv_navigation2/launch/navigation2_filter_active.launch.py)을 준비합니다.
- **Nav2 비교 실험용 `tracer_nav2/`** — [Nav2 기반 경로 실행 구성](tracer_nav2/launch/tracer_nav2_auto_mission.launch.py)을 보관합니다.
- **환경 설정과 토픽 명세** — [Jetson 환경 설정, 팀 인터페이스 및 대시보드 실행 안내](https://github.com/jsdbswjd/ros2-logiflow/blob/142d930054b9b62f6a9b57e078df0844cb1daac9/README.md)
- **제조사 패키지** — [Elephant Robotics 원본 및 프로젝트 수정 범위](vendor/elephantrobotics/VENDOR.md)
