# Hướng Dẫn ESP32 Kết Nối micro-ROS Và Điều Khiển Động Cơ Qua PCF8575

## 1. Giới thiệu

Dự án này hướng dẫn cách tích hợp **ESP32** với **micro-ROS**, cho phép nhận lệnh điều khiển động cơ qua **ROS 2** thông qua giao tiếp Wi-Fi, đồng thời xuất tín hiệu điều khiển ra **IC mở rộng I/O PCF8575**.

Ứng dụng phù hợp cho các hệ thống robot, xe tự hành hoặc các ứng dụng IoT có điều khiển động cơ từ xa.

---

## 2. Phần cứng cần thiết

- **ESP32 DevKit (NodeMCU-32S hoặc tương đương)**
- **Module PCF8575 (I2C I/O Expander)**
- **Driver điều khiển motor TB6612**
- **Hai motor DC**
- **Nguồn cấp 5–12V**
- **Dây nối & Breadboard**

---

## 3. Sơ đồ kết nối cơ bản

| Thành phần | Chân ESP32 | Chân PCF8575 | Ghi chú |
|-------------|-------------|--------------|----------|
| SDA         | GPIO21      | SDA          | Giao tiếp I2C |
| SCL         | GPIO22      | SCL          | Giao tiếp I2C |
| PWMA        | GPIO27      | —            | PWM motor A |
| PWMB        | GPIO14      | —            | PWM motor B |
| STBY        | GPIO17      | —            | Chân kích hoạt driver |
| LED báo     | GPIO2       | —            | LED hoạt động |

---

## 4. Phần mềm và môi trường

### 4.1. Cài đặt PlatformIO

1. Cài Visual Studio Code  
2. Cài extension **PlatformIO IDE**
3. Tạo dự án mới cho ESP32

### 4.2. Cấu hình `platformio.ini`

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

## 5. Mã nguồn chính (`main.cpp`)

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

// Các hàm điều khiển motor
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

void subscription_callback(const void *msgin) {
  Serial.println("Callback nhận dữ liệu ROS2 được gọi!");
  const std_msgs__msg__String *msg = (const std_msgs__msg__String *)msgin;
  String cmd = String(msg->data.data);
  Serial.print("Nhận lệnh: ");
  Serial.println(cmd);

  if (cmd == "BOTH_FORWARD") both_forward();
  else if (cmd == "BOTH_STOP") both_stop();
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

---

## 6. Kiểm thử

1. Chạy agent trên máy tính:  
   ```bash
   ros2 run micro_ros_agent micro_ros_agent udp4 --port 8888 -v6
   ```
2. Nạp code ESP32, mở Serial Monitor (115200)
3. Gửi lệnh từ ROS2:
   ```bash
   ros2 topic pub /motor_cmd std_msgs/String "data: 'BOTH_FORWARD'"
   ```

ESP32 sẽ phản hồi trong terminal:  
```
Callback nhận dữ liệu ROS2 được gọi!
Nhận lệnh: BOTH_FORWARD
Cả hai motor chạy tới
```

---

## 7. Ghi chú và lỗi thường gặp

| Vấn đề | Nguyên nhân | Cách khắc phục |
|--------|--------------|----------------|
| Không nhận được lệnh | Thiếu cấp bộ nhớ cho recv_msg | Thêm dòng: `recv_msg.data.data = (char*)malloc(100);` |
| ESP32 không kết nối agent | IP sai hoặc mạng Wi-Fi khác | Kiểm tra lại IP agent và SSID |
| PCF8575 không phản hồi | Sai địa chỉ I2C | Dùng scanner kiểm tra, thường là `0x20–0x27` |

---

## 8. Tài liệu tham khảo

- [micro-ROS for ESP32 (PlatformIO)](https://github.com/micro-ROS/micro_ros_platformio)
- [PCF8575 Library - Rob Tillaart](https://github.com/RobTillaart/PCF8575)
- [ROS2 Documentation](https://docs.ros.org/en/foxy/)

---

**Tác giả:** Beniot Phan  
**Ngày tạo:** 2025-10-28  
**Phiên bản:** 1.0
