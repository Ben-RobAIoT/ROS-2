
# ROS2 + micro-ROS + ESP32 — Keyboard Teleop (Compact Guide)

**Mục tiêu:** Gửi lệnh điều khiển động cơ (↑ ↓ ← →) từ máy tính qua ROS2 tới ESP32 (micro-ROS) qua Wi‑Fi.  
Tài liệu gọn, đầy đủ, thích hợp để push lên GitHub.

---

## Tổng quan hệ thống
- **ESP32 (PlatformIO, Arduino + micro-ROS):** nhận topic `/motor_cmd` (std_msgs/String) và điều khiển motor qua PCF8575 + TB6612.
- **PC (ROS2):** chạy `micro_ros_agent` và một node Python (`motor_teleop`) để đọc phím mũi tên và publish message.
- **Giao tiếp:** UDP (micro-ROS agent) — ESP32 kết nối Wi-Fi tới agent.

---

## 1. ESP32 — code (PlatformIO, `src/main.cpp`)

Giữ code ESP32 của bạn như ví dụ dưới; đảm bảo thêm hàm `turn_left()` và `turn_right()` và xử lý chuỗi nhận vào.

```cpp
#include <Arduino.h>
#include <Wire.h>
#include "PCF8575.h"

#include <micro_ros_platformio.h>
#include <rcl/rcl.h>
#include <rclc/rclc.h>
#include <rclc/executor.h>
#include <std_msgs/msg/string.h>

PCF8575 pcf8575(0x22);

#define AIN1_PCF P0
#define AIN2_PCF P1
#define BIN1_PCF P2
#define BIN2_PCF P3

#define PWMA 27
#define PWMB 14
#define STBY 17
#define LED_PIN 2

IPAddress agent_ip(192, 168, 43, 127);
size_t agent_port = 8888;
char ssid[] = "Beniot_hehee";
char psk[] = "PBT112004";

rcl_publisher_t publisher;
rcl_subscription_t subscriber;
std_msgs__msg__String msg;
std_msgs__msg__String recv_msg;

rclc_executor_t executor;
rcl_node_t node;
rclc_support_t support;
rcl_allocator_t allocator;
rcl_timer_t timer;

void both_forward() {
  pcf8575.digitalWrite(AIN1_PCF, HIGH);
  pcf8575.digitalWrite(AIN2_PCF, LOW);
  pcf8575.digitalWrite(BIN1_PCF, HIGH);
  pcf8575.digitalWrite(BIN2_PCF, LOW);
  analogWrite(PWMA, 255);
  analogWrite(PWMB, 255);
  Serial.println("Cả hai motor chạy tới");
}

void both_stop() {
  analogWrite(PWMA, 0);
  analogWrite(PWMB, 0);
  Serial.println("Cả hai motor dừng");
}

void turn_left() {
  pcf8575.digitalWrite(AIN1_PCF, LOW);
  pcf8575.digitalWrite(AIN2_PCF, HIGH);
  pcf8575.digitalWrite(BIN1_PCF, HIGH);
  pcf8575.digitalWrite(BIN2_PCF, LOW);
  analogWrite(PWMA, 200);
  analogWrite(PWMB, 200);
  Serial.println("🌀 Quay trái");
}

void turn_right() {
  pcf8575.digitalWrite(AIN1_PCF, HIGH);
  pcf8575.digitalWrite(AIN2_PCF, LOW);
  pcf8575.digitalWrite(BIN1_PCF, LOW);
  pcf8575.digitalWrite(BIN2_PCF, HIGH);
  analogWrite(PWMA, 200);
  analogWrite(PWMB, 200);
  Serial.println("🌀 Quay phải");
}

void subscription_callback(const void *msgin) {
  Serial.println("Callback nhận dữ liệu ROS2 được gọi!");
  const std_msgs__msg__String *m = (const std_msgs__msg__String *)msgin;
  String cmd = String(m->data.data);
  Serial.print("Nhận lệnh: ");
  Serial.println(cmd);

  if (cmd == "BOTH_FORWARD") both_forward();
  else if (cmd == "BOTH_STOP") both_stop();
  else if (cmd == "TURN_LEFT") turn_left();
  else if (cmd == "TURN_RIGHT") turn_right();
  else Serial.println("⚠️ Lệnh không hợp lệ!");

  std_msgs__msg__String feedback_msg;
  feedback_msg.data.data = (char*)"Command executed";
  feedback_msg.data.size = strlen(feedback_msg.data.data);
  rcl_publish(&publisher, &feedback_msg, NULL);
}

void timer_callback(rcl_timer_t *timer, int64_t last_call_time) {
  RCLC_UNUSED(last_call_time);
  if (timer != NULL) {
    msg.data.data = (char*)"ESP32 still alive via WiFi";
    msg.data.size = strlen(msg.data.data);
    rcl_publish(&publisher, &msg, NULL);
    digitalWrite(LED_PIN, !digitalRead(LED_PIN));
  }
}

void setup() {
  Serial.begin(115200);
  delay(2000);
  Serial.println("===== ESP32 micro-ROS + PCF8575 Motor Control =====");

  pcf8575.pinMode(AIN1_PCF, OUTPUT);
  pcf8575.pinMode(AIN2_PCF, OUTPUT);
  pcf8575.pinMode(BIN1_PCF, OUTPUT);
  pcf8575.pinMode(BIN2_PCF, OUTPUT);
  pcf8575.begin();

  pinMode(PWMA, OUTPUT);
  pinMode(PWMB, OUTPUT);
  pinMode(STBY, OUTPUT);
  digitalWrite(STBY, HIGH);
  pinMode(LED_PIN, OUTPUT);

  set_microros_wifi_transports(ssid, psk, agent_ip, agent_port);

  allocator = rcl_get_default_allocator();
  rclc_support_init(&support, 0, NULL, &allocator);
  rclc_node_init_default(&node, "esp32_motor_node", "", &support);

  rclc_publisher_init_default(&publisher, &node, ROSIDL_GET_MSG_TYPE_SUPPORT(std_msgs, msg, String), "esp32_wifi_chatter");

  recv_msg.data.data = (char*)malloc(100);
  recv_msg.data.size = 0;
  recv_msg.data.capacity = 100;

  rclc_subscription_init_default(&subscriber, &node, ROSIDL_GET_MSG_TYPE_SUPPORT(std_msgs, msg, String), "motor_cmd");

  const unsigned int timer_timeout = 1000;
  rclc_timer_init_default(&timer, &support, RCL_MS_TO_NS(timer_timeout), timer_callback);

  rclc_executor_init(&executor, &support.context, 2, &allocator);
  rclc_executor_add_timer(&executor, &timer);
  rclc_executor_add_subscription(&executor, &subscriber, &recv_msg, &subscription_callback, ON_NEW_DATA);

  Serial.println("Setup hoàn tất, chờ ROS2 gửi lệnh...");
}

void loop() {
  rclc_executor_spin_some(&executor, RCL_MS_TO_NS(100));
  delay(10);
}
```

**Ghi chú:** điều chỉnh địa chỉ `agent_ip`, SSID, PSK, và các chân PWM/I2C theo phần cứng của bạn.

---

## 2. PlatformIO — `platformio.ini` (gợi ý)

```ini
[env:nodemcu-32s]
platform = espressif32
board = nodemcu-32s
framework = arduino
monitor_speed = 115200
lib_deps =
  micro-ROS/micro_ros_platformio
  robtillaart/PCF8575
```

---

## 3. Trên máy tính (PC) — Tạo ROS2 Python package

### 3.1 Tạo package
Trong ROS2 workspace (ví dụ `~/ros2_ws`):

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python motor_teleop
```

### 3.2 Thêm file `keyboard_motor_teleop.py` vào `~/ros2_ws/src/motor_teleop/motor_teleop/`:

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from std_msgs.msg import String
import sys
import termios
import tty

class KeyboardMotorTeleop(Node):
    def __init__(self):
        super().__init__('keyboard_motor_teleop')
        self.publisher_ = self.create_publisher(String, 'motor_cmd', 10)
        self.get_logger().info("Keyboard teleop started. Use arrow keys (↑↓←→). Press 'q' to quit.")
        self.run()

    def get_key(self):
        fd = sys.stdin.fileno()
        old_settings = termios.tcgetattr(fd)
        try:
            tty.setraw(fd)
            key = sys.stdin.read(3)
        finally:
            termios.tcsetattr(fd, termios.TCSADRAIN, old_settings)
        return key

    def run(self):
        key_cmd_map = {
            '\x1b[A': 'BOTH_FORWARD',
            '\x1b[B': 'BOTH_STOP',
            '\x1b[C': 'TURN_RIGHT',
            '\x1b[D': 'TURN_LEFT',
        }

        while rclpy.ok():
            key = self.get_key()
            if key == 'q':
                print("\nExit program.")
                break
            if key in key_cmd_map:
                msg = String()
                msg.data = key_cmd_map[key]
                self.publisher_.publish(msg)
                self.get_logger().info(f"Sent: {msg.data}")
            else:
                print("Invalid key. Use ↑ ↓ ← → or q.")

def main(args=None):
    rclpy.init(args=args)
    node = KeyboardMotorTeleop()
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

### 3.3 `setup.py` cho package

```python
from setuptools import setup

package_name = 'motor_teleop'

setup(
    name=package_name,
    version='0.0.1',
    packages=[package_name],
    install_requires=['setuptools'],
    zip_safe=True,
    maintainer='beniot-phan',
    maintainer_email='beniot@example.com',
    description='Keyboard teleop for motor control via micro-ROS',
    license='MIT',
    entry_points={
        'console_scripts': [
            'keyboard_motor_teleop = motor_teleop.keyboard_motor_teleop:main'
        ],
    },
)
```

### 3.4 `package.xml`

```xml
<?xml version="1.0"?>
<package format="3">
  <name>motor_teleop</name>
  <version>0.0.1</version>
  <description>Keyboard teleop for motor control via micro-ROS</description>
  <maintainer email="beniot@example.com">beniot-phan</maintainer>
  <license>MIT</license>

  <buildtool_depend>ament_python</buildtool_depend>
  <exec_depend>rclpy</exec_depend>
  <exec_depend>std_msgs</exec_depend>
</package>
```

---

## 4. Build & chạy

1. Build ROS2 workspace:
```bash
cd ~/ros2_ws
colcon build --symlink-install
source install/setup.bash
```

2. Chạy micro-ROS agent (PC):
```bash
ros2 run micro_ros_agent micro_ros_agent udp4 --port 8888 -v6
```

3. Chạy keyboard teleop:
```bash
source ~/ros2_ws/install/setup.bash
ros2 run motor_teleop keyboard_motor_teleop
```

4. (Optional) Mở Serial Monitor PlatformIO để xem logs ESP32.

---

## 5. Troubleshooting nhanh
- **Không thấy `install/setup.bash`:** Chưa build thành công — chạy `colcon build` và xem lỗi.
- **Package không có trong `ros2 pkg list`:** kiểm tra `package.xml` và `setup.py`, đảm bảo `packages=[package_name]` và có `__init__.py`.
- **ESP32 không kết nối agent:** kiểm tra IP agent, SSID/PSK, firewall.
- **PCF8575 không phản hồi:** kiểm tra địa chỉ I2C (dùng I2C scanner), kiểm tra wiring.

---

## 6. Mở rộng & ghi chú
- Bạn có thể thay `std_msgs/String` bằng `geometry_msgs/Twist` nếu muốn điều khiển bằng vận tốc/omega.
- Có thể thêm chế độ Serial control (w/a/s/d) để debug khi agent mất kết nối.
- Điều chỉnh PWM duty cycle để thay đổi vận tốc.

---

## Tài liệu tham khảo
- micro-ROS: https://micro.ros.org
- ROS 2 documentation: https://docs.ros.org
- PCF8575 library: Rob Tillaart

---

**Tác giả:** Beniot Phan  
**Ngày tạo:** 2025-11-13
