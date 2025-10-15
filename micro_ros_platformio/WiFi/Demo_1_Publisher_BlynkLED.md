Tài liệu tham khảo chính [QUAN TRỌNG]: https://github.com/micro-ROS/micro_ros_platformio
# 🚀 Hướng Dẫn Cấu Hình ESP32 Giao Tiếp micro-ROS Qua WiFi (UDP)

Tài liệu này hướng dẫn chi tiết cách cấu hình **ESP32** để giao tiếp với **ROS 2 (Humble)** thông qua **micro-ROS** qua **UDP (WiFi)** thay vì Serial.  
Phù hợp cho các dự án IoT, robot hoặc hệ thống bãi giữ xe thông minh.

---

## 🧩 1. Chuẩn Bị Môi Trường

### ⚙️ Phần mềm cần thiết:
- Ubuntu 22.04 hoặc 24.04 (có ROS 2 Humble)
- PlatformIO (VS Code)
- Docker (nếu muốn chạy micro-ROS Agent bằng container)

### ⚙️ Cài micro-ROS Agent trên máy ROS 2:

Nếu không dùng Docker:

```bash
sudo apt update
sudo apt install ros-humble-micro-ros-agent
```

Nếu apt không tìm thấy gói trên (lỗi “Unable to locate package”), thì chạy Agent bằng Docker:

```bash
sudo apt install docker.io
sudo systemctl enable docker
sudo systemctl start docker
```

---

## 🧠 2. Cấu Hình ESP32 Gửi Dữ Liệu Qua UDP

Tạo project PlatformIO mới, chọn **board ESP32 (NodeMCU-32S)**.  
Trong file `src/main.cpp`, chép toàn bộ nội dung sau:

```cpp
#include <Arduino.h>
#include <micro_ros_platformio.h>
#include <rcl/rcl.h>
#include <rclc/rclc.h>
#include <rclc/executor.h>
#include <std_msgs/msg/string.h>

rcl_publisher_t publisher;
std_msgs__msg__String msg;
rclc_executor_t executor;
rcl_node_t node;
rclc_support_t support;
rcl_allocator_t allocator;
rcl_timer_t timer;

IPAddress agent_ip(192, 168, 43, 127);   // 🖥️ IP máy tính chạy ROS 2
size_t agent_port = 8888;                // Cổng micro-ROS Agent
char ssid[] = "Beniot_hehee";            // 📶 WiFi SSID
char psk[]  = "PBT112004";               // 🔑 Mật khẩu WiFi

#define LED_PIN 2

void timer_callback(rcl_timer_t *timer, int64_t last_call_time) {
  RCLC_UNUSED(last_call_time);
  if (timer != NULL) {
    msg.data.data = (char*)"Hello from ESP32 via WiFi!";
    msg.data.size = strlen(msg.data.data);
    rcl_publish(&publisher, &msg, NULL);
    digitalWrite(LED_PIN, !digitalRead(LED_PIN));
  }
}

void setup() {
  pinMode(LED_PIN, OUTPUT);
  delay(2000);

  Serial.begin(115200);
  set_microros_wifi_transports(ssid, psk, agent_ip, agent_port);

  WiFi.begin(ssid, psk);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("Connected to WiFi!");
  Serial.println(WiFi.localIP());

  allocator = rcl_get_default_allocator();

  // Initialize micro-ROS
  rclc_support_init(&support, 0, NULL, &allocator);

  // Create node
  rclc_node_init_default(&node, "esp32_wifi_node", "", &support);

  // Create publisher
  rclc_publisher_init_default(
    &publisher,
    &node,
    ROSIDL_GET_MSG_TYPE_SUPPORT(std_msgs, msg, String),
    "esp32_wifi_chatter");

  // Create timer
  const unsigned int timer_timeout = 1000; // 1 giây
  rclc_timer_init_default(&timer, &support, RCL_MS_TO_NS(timer_timeout), timer_callback);

  // Create executor
  rclc_executor_init(&executor, &support.context, 1, &allocator);
  rclc_executor_add_timer(&executor, &timer);

  msg.data.data = (char*)"";
  msg.data.size = 0;
  msg.data.capacity = 100;
}

void loop() {
  rclc_executor_spin_some(&executor, RCL_MS_TO_NS(100));
  delay(100);
}
```

---

## ⚙️ 3. File `platformio.ini`

```ini
[env:nodemcu-32s]
platform = espressif32
board = nodemcu-32s
framework = arduino
lib_deps =
    https://github.com/micro-ROS/micro_ros_platformio
monitor_speed = 115200
```

---

## 🌐 4. Kết Nối Mạng

ESP32 và máy tính ROS **phải cùng mạng WiFi**.

Xem IP của máy tính ROS:

```bash
ip a | grep inet
```

Ví dụ IP là `192.168.43.127`, thì giữ nguyên trong code ESP32:
```cpp
IPAddress agent_ip(192, 168, 43, 127);
```

---

## 🛰️ 5. Khởi Động micro-ROS Agent (UDP)

### ✅ Nếu cài bằng apt:
```bash
ros2 run micro_ros_agent micro_ros_agent udp4 --port 8888 -v6
```

### ✅ Nếu dùng Docker:
```bash
docker run -it --rm   --net=host   microros/micro-ros-agent:humble udp4 --port 8888 -v6
```

> ⚠️ Không dùng lệnh `serial` vì ta đang truyền qua WiFi UDP.

---

## 🧪 6. Kiểm Tra Kết Nối

1. Kiểm tra danh sách topic:
   ```bash
   ros2 topic list
   ```

2. Xem nội dung message:
   ```bash
   ros2 topic echo /esp32_wifi_chatter
   ```

Kết quả:
```
data: 'Hello from ESP32 via WiFi!'
---
```

LED trên ESP32 sẽ nhấp nháy mỗi lần gửi message.

---

## ⚡ 7. Lỗi Thường Gặp

| Lỗi | Nguyên nhân | Cách khắc phục |
|------|--------------|----------------|
| `Unable to locate package ros-humble-micro-ros-agent` | Gói chưa có trong repo | Dùng Docker |
| `ping registry-1.docker.io` 100% packet loss | Không truy cập Docker Hub | Đổi DNS sang 8.8.8.8 |
| Không thấy `/dev/ttyUSB0` | ESP32 chưa nhận | Kiểm tra `lsusb` |
| `topic does not appear to be published` | ESP32 chưa gửi dữ liệu | Kiểm tra IP, SSID, agent port |

---

## 🧠 8. Tóm Tắt Luồng Kết Nối

```
ESP32 ───WiFi UDP───> micro-ROS Agent ───> ROS 2 (rcl, rclc, std_msgs)
```

| Thành phần | Vai trò |
|-------------|----------|
| ESP32 | Node micro-ROS, gửi dữ liệu sensor |
| micro-ROS Agent | Gateway ROS 2 <-> ESP32 |
| ROS 2 | Thu thập, hiển thị và xử lý dữ liệu |

---

## 📚 9. Tài Liệu Tham Khảo

- [micro-ROS Documentation](https://micro.ros.org/docs/)
- [micro_ros_platformio](https://github.com/micro-ROS/micro_ros_platformio)
- [ROS 2 Humble](https://docs.ros.org/en/humble/index.html)

---

**Tác giả:** Beniot Phan  
**Dự án:** ESP32 + micro-ROS over WiFi (UDP)  
**Ngày:** 2025-10-15
