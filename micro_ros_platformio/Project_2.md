
# 🚀 Hướng dẫn Kết nối ESP32 với ROS 2 bằng micro-ROS và Điều khiển LED Built-in

## 🧠 Mục tiêu

Bật/tắt LED trên ESP32 thông qua ROS 2 topic `led_toggle`, nhằm hiểu cách ROS 2 giao tiếp với vi điều khiển qua micro-ROS.

---

## 🧩 1. Mô hình tổng quan

Trên **ROS 2 (PC Host)**:
```bash
ros2 topic pub /led_toggle std_msgs/Bool "data: true" -1
```
Trên **ESP32 (micro-ROS)**:
- Node: `micro_ros_led`
- Subscriber: `/led_toggle`
- Khi nhận `true` → bật LED
- Khi nhận `false` → tắt LED

---

## ⚙️ 2. Cấu hình `platformio.ini`

Tạo project mới trong PlatformIO (hoặc sửa project hiện có), sau đó thêm nội dung:

```ini
[env:nodemcu-32s]
platform = espressif32
board = nodemcu-32s
framework = arduino
monitor_speed = 115200

lib_ldf_mode = deep

lib_deps =
    https://github.com/micro-ROS/micro_ros_arduino.git
```

---

## 🧰 3. Code ESP32 (file `src/main.cpp`)

```cpp
#include <Arduino.h>
#include <micro_ros_arduino.h>
#include <rcl/rcl.h>
#include <rclc/rclc.h>
#include <rclc/executor.h>
#include <std_msgs/msg/bool.h>

#define LED_PIN 2 // Built-in LED cho NodeMCU-32S

rcl_subscription_t subscriber;
std_msgs__msg__Bool msg;
rclc_executor_t executor;
rcl_node_t node;
rclc_support_t support;
rcl_allocator_t allocator;

void led_callback(const void * msgin)
{
  const std_msgs__msg__Bool * msg = (const std_msgs__msg__Bool *)msgin;
  digitalWrite(LED_PIN, msg->data ? HIGH : LOW);
}

void setup()
{
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);

  set_microros_serial_transports(Serial);
  Serial.begin(115200);
  delay(2000);

  allocator = rcl_get_default_allocator();
  rclc_support_init(&support, 0, NULL, &allocator);
  rclc_node_init_default(&node, "micro_ros_led", "", &support);

  rclc_subscription_init_default(
    &subscriber,
    &node,
    ROSIDL_GET_MSG_TYPE_SUPPORT(std_msgs, msg, Bool),
    "led_toggle");

  rclc_executor_init(&executor, &support.context, 1, &allocator);
  rclc_executor_add_subscription(&executor, &subscriber, &msg, &led_callback, ON_NEW_DATA);
}

void loop()
{
  rclc_executor_spin_some(&executor, RCL_MS_TO_NS(100));
  delay(10);
}
```

---

## ⚡ 4. Upload chương trình lên ESP32

```bash
pio run -t upload
```

---

## 🧠 5. Chạy micro-ROS Agent

Trên máy tính có ROS 2:
```bash
ros2 run micro_ros_agent micro_ros_agent serial --dev /dev/ttyUSB0 -v6
```

Khi ESP32 khởi động lại, bạn sẽ thấy trong log:
```
create_participant
create_subscriber
create_datareader
```

---

## 🧪 6. Kiểm tra trên ROS 2

Danh sách topic:
```bash
ros2 topic list
```
Kết quả mong muốn:
```
/led_toggle
/parameter_events
/rosout
```

Thử gửi lệnh bật/tắt LED:
```bash
ros2 topic pub /led_toggle std_msgs/Bool "data: true" -1
ros2 topic pub /led_toggle std_msgs/Bool "data: false" -1
```

LED trên ESP32 sẽ bật/tắt theo lệnh ROS 2 🎉

---

## 🩺 7. Debug thường gặp

| Vấn đề | Nguyên nhân | Cách xử lý |
|--------|--------------|------------|
| Không thấy `/led_toggle` | ESP32 chưa nạp code có subscriber | Upload lại code ở phần 3 |
| `micro_ros_arduino.h` not found | Thiếu thư viện | Kiểm tra `lib_deps` trong `platformio.ini` |
| Không thấy log “create_subscriber” | ESP32 chưa kết nối với agent | Kiểm tra cổng `/dev/ttyUSBx` và tốc độ baud |

---

## 📘 8. Mở rộng

- ESP32 có thể publish ngược trạng thái LED hoặc cảm biến về ROS 2.
- Có thể dùng giao thức UDP thay cho Serial (nếu cần tốc độ cao).

---

**Tác giả:** Benoit Phan  
**Dự án:** Kết nối ESP32 ↔ ROS 2 qua micro-ROS để điều khiển LED build-in.  
**Ngày cập nhật:** 2025-10-06  
