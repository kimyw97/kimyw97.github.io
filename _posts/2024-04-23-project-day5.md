---
layout: post
title: "2025.04.23 - 스텝 모터 제어(A4988 + STM32) & Raspberry Pi ROS2 자동 설치"
date: 2025-04-23 21:00:00 +0900
categories: stm32 motor-control ros2 raspberrypi automation
---

## 📌 오늘 한 일

### [1] A4988 + STM32를 활용한 스텝 모터 제어 실습
- 6핀 NEMA17 스텝 모터 구조 및 작동 원리 학습
- A4988 드라이버의 작동 원리 및 핀 구성 정리
- CNC Shield V3 보드 구조 파악
- STM32와의 연결 방식 및 펄스 제어 원리 정리

### [2] Raspberry Pi 자동화 스크립트로 ROS 2 환경 구축
- ROS 2 Humble 자동 설치 스크립트 작성
- GitHub에서 프로젝트 클론 및 워크스페이스 빌드 자동화
- `.bashrc` 등록, rosdep 의존성 설치 포함

---

## ⚙️ NEMA17 (6핀) 스텝 모터 핀아웃

6핀 모터에는 2개의 코일과 중심탭이 존재한다:

```
[1] A+  
[2] A center tap  
[3] A−  
[4] B+  
[5] B center tap  
[6] B−
```

> A4988 같은 bipolar 드라이버에서는 center tap([2], [5])을 연결하지 않으며, [1]-[3], [4]-[6]만 연결한다.

---

## 🔌 A4988 드라이버 핀 구성

| 핀 번호 | 핀 이름 | 설명 |
|---------|----------|------|
| 1       | VMOT     | 모터 전원 입력 (8~35V) |
| 2       | GND      | 모터 GND |
| 3       | 2B       | 스텝 모터 B코일 음극 |
| 4       | 2A       | 스텝 모터 B코일 양극 |
| 5       | 1A       | 스텝 모터 A코일 양극 |
| 6       | 1B       | 스텝 모터 A코일 음극 |
| 7       | VDD      | 로직 전원 입력 (3~5V) |
| 8       | GND      | 로직 GND |
| 9       | STEP     | 스텝 펄스 입력 (HIGH 펄스 시 1스텝 이동) |
| 10      | DIR      | 회전 방향 설정 |
| 11      | ENABLE   | LOW일 때 활성화 |
| 12~14   | MS1~MS3  | 마이크로스텝 설정 |
| 15      | RESET    | LOW이면 리셋 |
| 16      | SLEEP    | LOW이면 슬립모드 진입 |

---

## 🧱 CNC Shield V3 구조

- Arduino UNO 호환 보드로 A4988 드라이버 4개(X/Y/Z/A) 장착 가능
- 각 축은 STEP/DIR 핀으로 제어되며 STM32 사용 시 수동 배선 필요
- ENABLE 핀은 공통으로 묶여 있음

---

## 🛠️ STM32와 A4988 연결 예시

| A4988 핀 | STM32 핀 예시  | 설명           |
|----------|----------------|----------------|
| STEP     | PA0            | 타이머 출력 핀 |
| DIR      | PA1            | GPIO 출력 핀   |
| ENABLE   | GND 또는 GPIO  | LOW로 활성화   |
| 1A~2B    | 스텝모터 연결  | 코일 매칭 필요 |
| VMOT     | 외부 전원 (12V)| 모터 구동용    |
| VDD      | 3.3V 또는 5V   | 로직 구동용    |

---

## 🔄 A4988 제어 원리 요약

- DIR 핀으로 방향 설정 (`HIGH` 또는 `LOW`)
- STEP 핀에 최소 1μs 이상의 펄스를 인가 → 1스텝 이동
- 마이크로스텝은 MS1~MS3 핀 설정으로 조정
- ENABLE 핀은 LOW일 때 동작
---

## 💻 Raspberry Pi ROS2 자동 설치 스크립트

### ✅ 기능 요약

- ROS 2 Humble 설치 자동화
- rosdep 초기화 및 의존성 설치
- GitHub 저장소 클론 및 `ros2_ws` 빌드
- `.bashrc` 등록 및 빌드 성공 플래그 파일 생성

### 🔧 스크립트

```bash
#!/bin/bash
REPO_URL="https://github.com/yourusername/yourrepo.git"
CLONE_DIR="$HOME/temp_cow_repo"
TARGET_WS="$HOME/ros2_ws"

sudo apt update && sudo apt upgrade -y

# ROS 2 설치
if ! dpkg -s ros-humble-desktop > /dev/null 2>&1; then
  sudo apt install -y locales
  sudo locale-gen en_US en_US.UTF-8
  sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
  export LANG=en_US.UTF-8

  sudo apt install -y software-properties-common
  sudo add-apt-repository universe
  sudo apt update
  sudo apt install -y curl
  sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.asc | sudo tee /usr/share/keyrings/ros-archive-keyring.gpg > /dev/null
  echo "deb [signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
  sudo apt update
  sudo apt install -y ros-humble-desktop
fi

# 개발 도구 설치
sudo apt install -y python3-colcon-common-extensions python3-rosdep python3-vcstool \
  python3-argcomplete python3-pip build-essential cmake git

# rosdep 설정
if [ ! -f "/etc/ros/rosdep/sources.list.d/20-default.list" ]; then
  sudo rosdep init
fi

rosdep_retry=0
until rosdep update || [ $rosdep_retry -eq 3 ]; do
  echo "⚠️ rosdep update failed, retrying..."
  sleep 2
  rosdep_retry=$((rosdep_retry + 1))
done

# 저장소 클론 및 복사
mkdir -p "$CLONE_DIR"
git clone "$REPO_URL" "$CLONE_DIR" --depth 1
mkdir -p "$TARGET_WS"
cp -r "$CLONE_DIR/cow/raspberrypi/ros2_ws/"* "$TARGET_WS"

# 빌드
cd "$TARGET_WS"
source /opt/ros/humble/setup.bash
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install --cmake-clean-cache

if [ $? -eq 0 ]; then
  touch "$TARGET_WS/.ros2_build_success"
  echo "✅ Build success"
else
  echo "❌ Build failed"
  exit 1
fi

# .bashrc 등록
if ! grep -Fxq "source ~/ros2_ws/install/setup.bash" ~/.bashrc; then
  echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc
fi
```

---

## 🚀 실행 방법

```bash
# 1. 라즈베리파이에 접속
ssh username@raspberrypi.local

# 2. 스크립트 생성
vi setup-kongji-patji.sh  # 내용 붙여넣기

# 3. 실행 권한 부여
chmod +x setup-kongji-patji.sh

# 4. 실행
./setup-kongji-patji.sh
```

---

## ✅ 다음 목표
- STM32 타이머를 활용한 STEP 펄스 생성 코드 작성
- 마이크로스텝 설정 실험 (FULL, HALF, 1/4 등)
