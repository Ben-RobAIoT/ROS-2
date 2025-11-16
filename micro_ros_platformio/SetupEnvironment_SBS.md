# 🚀 HƯỚNG DẪN CÀI ĐẶT & THIẾT LẬP MÔI TRƯỜNG **ROS2 + micro-ROS** (SIÊU ĐƠN GIẢN)

## 🔗 Link tải cần thiết
| Nội dung | Link |
|----------|------|
| ROS2 Humble | https://docs.ros.org/en/humble/index.html |
| micro-ROS Docs | https://micro.ros.org/docs/tutorials/core/overview/ |
| micro-ROS PlatformIO Repo | https://github.com/micro-ROS/micro_ros_platformio/tree/main?tab=readme-ov-file#micro-ros-for-platformio |

---

## 📌 MỤC LỤC
- [1. Cài đặt ROS2](#1-️-cài-đặt-ros2)
- [2. Cài đặt micro-ROS](#2-️-cài-đặt-micro-ros)
- [3. Cài đặt micro-ROS Agent](#3-️-cài-đặt-micro-ros-agent)
- [4. Cài đặt ROS2 trên Ubuntu (chi tiết)](#4-️-hướng-dẫn-chi-tiết-cài-đặt-ros-trên-ubuntu)
- [5. Cài đặt micro-ROS Desktop build system](#5-️-cài-đặt-microros-build-system)
- [6. micro-ROS với PlatformIO](#6-️-micro-ros--platformio)
- [7. Chạy micro-ROS Agent bằng Docker](#7-️-chạy-micro-ros-agent-bằng-docker)

---

# 1️⃣ Cài đặt ROS2
### Bước 1.1 — Truy cập trang cài đặt  
👉 https://docs.ros.org/en/humble/index.html

### Bước 1.2 — Chọn OS và làm theo hướng dẫn

---

# 2️⃣ Cài đặt micro-ROS
### Bước 2.1 — Truy cập docs  
👉 https://micro.ros.org/docs/tutorials/core/overview/

### Bước 2.2 — Tải ROS Agent & toolchain

---

# 3️⃣ Cài đặt micro-ROS Agent
Sẽ chạy trên máy tính để giao tiếp ESP32 / STM32 / Teensy / Pico

---

# 4️⃣ HƯỚNG DẪN CHI TIẾT CÀI ĐẶT ROS TRÊN UBUNTU

> Áp dụng cho: Ubuntu 22.04 (khuyến nghị)

---

## 🔧 Bước 1 — Set Locale

```bash
locale  # Check UTF-8
sudo apt update && sudo apt install locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8
locale
🔧 Bước 2 — Setup ROS Sources
sudo apt install software-properties-common
sudo add-apt-repository universe
sudo apt update && sudo apt install curl -y

export ROS_APT_SOURCE_VERSION=$(curl -s https://api.github.com/repos/ros-infrastructure/ros-apt-source/releases/latest | grep -F "tag_name" | awk -F\" '{print $4}')
curl -L -o /tmp/ros2-apt-source.deb "https://github.com/ros-infrastructure/ros-apt-source/releases/download/${ROS_APT_SOURCE_VERSION}/ros2-apt-source_${ROS_APT_SOURCE_VERSION}.$(. /etc/os-release && echo ${UBUNTU_CODENAME:-${VERSION_CODENAME}})_all.deb"
sudo dpkg -i /tmp/ros2-apt-source.deb

🔧 Bước 3 — Install ROS2

Cập nhật:

sudo apt update
sudo apt upgrade

Install ROS Desktop (khuyến nghị)
sudo apt install ros-humble-desktop

Hoặc ROS Base (nhẹ hơn)
sudo apt install ros-humble-ros-base

Công cụ lập trình:
sudo apt install ros-dev-tools

🌱 Setup môi trường
source /opt/ros/humble/setup.bash

🧪 Test ROS2: Talker - Listener
Terminal 1:
source /opt/ros/humble/setup.bash
ros2 run demo_nodes_cpp talker

Terminal 2:
source /opt/ros/humble/setup.bash
ros2 run demo_nodes_py listener

5️⃣ CÀI ĐẶT micro-ROS BUILD SYSTEM
source /opt/ros/$ROS_DISTRO/setup.bash
mkdir microros_ws
cd microros_ws
git clone -b $ROS_DISTRO https://github.com/micro-ROS/micro_ros_setup.git src/micro_ros_setup


Cài dependencies:

sudo apt update && rosdep update
rosdep install --from-paths src --ignore-src -y
sudo apt-get install python3-pip


Build:

colcon build
source install/local_setup.bash

📦 Workflow micro-ROS Build System
Step	Chức năng
1. Create	Tải toolchain + code
2. Configure	Chọn app & cấu hình
3. Build	Cross-compile
4. Flash	Nạp vào MCU
🎯 Ví dụ tạo Firmware
ros2 run micro_ros_setup create_firmware_ws.sh freertos olimex-stm32-e407

6️⃣ micro-ROS + PlatformIO

👉 Repo:
https://github.com/micro-ROS/micro_ros_platformio

Thêm vào platformio.ini
lib_deps =
    https://github.com/micro-ROS/micro_ros_platformio

Compile & Upload:
pio lib install
pio run
pio run --target upload

🌍 Chọn transport
Serial
Serial.begin(115200);
set_microros_serial_transports(Serial);

WiFi
set_microros_wifi_transports(ssid, psk, agent_ip, agent_port);

Ethernet
set_microros_ethernet_transports(client_ip, gateway, netmask, agent_ip, agent_port);

7️⃣ CHẠY micro-ROS AGENT BẰNG DOCKER
🔌 Serial Agent
docker run -it --rm -v /dev:/dev -v /dev/shm:/dev/shm --privileged --net=host microros/micro-ros-agent:$ROS_DISTRO serial --dev /dev/ttyUSB0 -v6

🌐 UDP Agent
docker run -it --rm -v /dev:/dev -v /dev/shm:/dev/shm --privileged --net=host microros/micro-ros-agent:$ROS_DISTRO udp4 --port 8888 -v6

🧠 Ghi nhớ

✔ ROS2 chạy trên PC
✔ micro-ROS chạy trên ESP32 / STM32 / Pico
✔ ROS Agent = cầu nối 2 bên
✔ Code micro-ROS build bằng platformIO hoặc microros_ws

📄 License

Apache 2.0 — xem file LICENSE

❤️ Gửi bạn lời chúc may mắn!

Nếu README hữu ích, hãy ⭐ repo nhé!


---

Nếu bạn muốn **thêm hình minh họa**, mình có thể giúp bạn chèn luôn (đã resize + căn giữa).
