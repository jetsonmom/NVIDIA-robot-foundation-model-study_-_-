# ROBOTIS OMX 설치 가이드 (Jetson Orin Nano / Ubuntu 22.04)

> 작성 환경: Jetson Orin Nano, Ubuntu 22.04, ROS 2 Jazzy

---

## 사전 준비

- OMX-L 리더 로봇 (USB 연결)
- OMX-F 팔로워 로봇 (USB 연결)
- USB 캠 2개 (카메라1: 팔로워용, 카메라2: 리더용)

---

## 1단계 - 기존 Docker 컨테이너/이미지 삭제 (재설치 시)

```bash
docker stop open_manipulator physical_ai_server physical_ai_manager
docker rm open_manipulator physical_ai_server physical_ai_manager
docker container prune -f
docker rmi robotis/open-manipulator:latest robotis/physical-ai-server:latest robotis/physical-ai-manager:latest
```

> ✅ 결과: `No such container/image` 에러가 나와도 정상 (이미 없는 경우)

---

## 2단계 - 기존 폴더 삭제 (재설치 시)

```bash
sudo rm -rf ~/open_manipulator
sudo rm -rf ~/physical_ai_tools
```

---

## 3단계 - Docker 완전 삭제 (재설치 시)

```bash
sudo apt purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
sudo rm -rf /etc/docker
```

확인:
```bash
docker --version
# bash: /usr/bin/docker: No such file or directory 나오면 완료
```

---

## 4단계 - Docker 재설치

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
newgrp docker
```

확인:
```bash
docker run hello-world
# "Hello from Docker!" 나오면 완료 ✅
```

---

## 5단계 - 깃 클론

```bash
cd ~
git clone https://github.com/ROBOTIS-GIT/open_manipulator
git clone --recurse-submodules https://github.com/ROBOTIS-GIT/physical_ai_tools.git
```

> ⚠️ `physical_ai_tools`는 서브모듈(lerobot) 포함으로 시간이 걸립니다 (~240MB)

---

## 6단계 - 포트 설정 (Docker 밖에서)

### 시리얼 ID 확인 방법
USB를 연결한 상태에서:
```bash
ls -al /dev/serial/by-id/
```

### 리더 포트 설정
```bash
gedit ~/open_manipulator/open_manipulator_bringup/launch/omx_l_leader_ai.launch.py
```

`port_name` 항목의 `default_value` 수정:
```python
default_value='/dev/serial/by-id/usb-ROBOTIS_OpenRB-150_8AA0EFB3503059384C2E3120FF093220-if00',
```

### 팔로워 포트 설정
```bash
gedit ~/open_manipulator/open_manipulator_bringup/launch/omx_f_follower_ai.launch.py
```

`port_name` 항목의 `default_value` 수정:
```python
default_value='/dev/serial/by-id/usb-ROBOTIS_OpenRB-150_A093CCDB503059384C2E3120FF08162B-if00',
```

> ⚠️ `/dev/ttyACM0` 형식이 아닌 시리얼 ID 전체 경로로 설정해야 USB 재연결 시에도 안정적으로 작동합니다.

---

## 7단계 - 카메라 설정 (Docker 밖에서)

### 카메라 번호 확인
```bash
ls /dev/video*
# /dev/video0  /dev/video1  /dev/video2  /dev/video3
# video0 = 팔로워 카메라, video2 = 리더 카메라
```

### camera_usb_cam.launch.py 수정
```bash
gedit ~/open_manipulator/open_manipulator_bringup/launch/camera_usb_cam.launch.py
```

`camera_nodes` 부분을 아래와 같이 수정 (카메라 2개):
```python
camera_nodes = [
    Node(
        package='usb_cam',
        executable='usb_cam_node_exe',
        parameters=[
            camera_config,
            {
                'video_device': '/dev/video0',  # 팔로워 카메라
            },
        ],
        output='both',
        remappings=[
            ('image_raw', ['camera1', '/image_raw']),
            ('image_raw/compressed', ['camera1', '/image_raw/compressed']),
            ('image_raw/compressedDepth', ['camera1', '/image_raw/compressedDepth']),
            ('image_raw/theora', ['camera1', '/image_raw/theora']),
            ('camera_info', ['camera1', '/camera_info']),
        ]
    ),
    Node(
        package='usb_cam',
        executable='usb_cam_node_exe',
        parameters=[
            camera_config,
            {
                'video_device': '/dev/video2',  # 리더 카메라
            },
        ],
        output='both',
        remappings=[
            ('image_raw', ['camera2', '/image_raw']),
            ('image_raw/compressed', ['camera2', '/image_raw/compressed']),
            ('image_raw/compressedDepth', ['camera2', '/image_raw/compressedDepth']),
            ('image_raw/theora', ['camera2', '/image_raw/theora']),
            ('camera_info', ['camera2', '/camera_info']),
        ]
    ),
]
```

### omx_f_config.yaml 수정
```bash
gedit ~/physical_ai_tools/physical_ai_server/config/omx_f_config.yaml
```

`camera2` 주석 해제:
```yaml
physical_ai_server:
  ros__parameters:
    omx_f:
      observation_list:
        - camera1
        - camera2
        - state
      camera_topic_list:
        - camera1:/camera1/image_raw/compressed
        - camera2:/camera2/image_raw/compressed
      joint_topic_list:
        - follower:/joint_states
        - leader:/leader/joint_trajectory
      joint_list:
        - leader
      joint_order:
        leader:
          - joint1
          - joint2
          - joint3
          - joint4
          - joint5
          - gripper_joint_1
```

---

## 8단계 - 컨테이너 시작 (이미지 자동 다운로드)

## 8단계 - 컨테이너 시작 (이미지 자동 다운로드)

> ⚠️ Docker를 새로 설치한 경우 nvidia 런타임을 먼저 설치해야 합니다.

curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker


```bash
cd ~/open_manipulator/docker && ./container.sh start
```

완료 후:
```bash
cd ~/physical_ai_tools/docker && ./container.sh start
```

> ⚠️ 이미지 크기가 크므로 시간이 걸립니다 (open_manipulator ~4GB, physical_ai_server ~17GB)

---

## 9단계 - 빌드 (open_manipulator 컨테이너)

```bash
cd ~/open_manipulator/docker && ./container.sh enter
cd ~/ros2_ws && colcon build --symlink-install --packages-skip open_manipulator_gui
```

> ✅ 결과: `Summary: 21 packages finished`

> ⚠️ `open_manipulator_gui`는 Jetson Orin Nano에서 메모리 부족으로 빌드 실패하므로 스킵합니다.

---

## 10단계 - 빌드 (physical_ai_tools 컨테이너)

```bash
cd ~/physical_ai_tools/docker && ./container.sh enter
cd ~/ros2_ws && colcon build --symlink-install
```

> ✅ 결과: `Summary: 4 packages finished`

---

## 11단계 - 실행

### USB 포트 권한 설정 (매번 재부팅 후 필요)
```bash
sudo chmod 777 /dev/ttyACM0
sudo chmod 777 /dev/ttyACM1
```

### 터미널 1 - 리더 실행
```bash
cd ~/open_manipulator/docker && ./container.sh enter
ros2 launch open_manipulator_bringup omx_l_leader_ai.launch.py
```

### 터미널 2 - 팔로워 실행
```bash
cd ~/open_manipulator/docker && ./container.sh enter
ros2 launch open_manipulator_bringup omx_f_follower_ai.launch.py
```

---

## 트러블슈팅

### USB 포트 인식 안 될 때
```bash
ls -al /dev/serial/by-id/
# 시리얼 ID 확인 후 launch 파일과 일치하는지 확인
```

### Docker 컨테이너 안에서 gedit 안 될 때
컨테이너 밖에서 파일을 수정해도 볼륨 마운트로 자동 반영됩니다.

### 빌드 후 변경사항이 반영 안 될 때
xacro 파일 수정 후 반드시 `colcon build` 를 다시 실행해야 합니다.
