# 🧩 micro-ROS trên ESP32 (PlatformIO) — Tổng hợp lỗi & Hướng khắc phục

> **Mục tiêu:** Kết nối ESP32 với micro-ROS để đọc dữ liệu cảm biến dò line (PCF8575) và publish lên ROS 2 topic.  
> **Board:** NodeMCU-32S (ESP32)  
> **Framework:** Arduino  
> **Build tool:** PlatformIO  
> **Library:** micro_ros_platformio  
> **Cảm biến:** PCF8575 qua giao tiếp I²C  

---

## 📚 Mục lục
1. [Tổng quan & Thuật ngữ](#-1-tổng-quan--thuật-ngữ)
2. [Các lỗi thường gặp & Hướng khắc phục](#-2-các-lỗi-thường-gặp--hướng-khắc-phục)
3. [Cấu trúc dự án mẫu](#-3-cấu-trúc-dự-án-mẫu)
4. [Bài học rút ra](#-4-bài-học-rút-ra)
5. [Hướng phát triển tiếp theo](#-5-hướng-phát-triển-tiếp-theo)

---

## 🧠 1. Tổng quan & Thuật ngữ

| Thuật ngữ | Giải thích |
|------------|------------|
| **micro-ROS** | Phiên bản rút gọn của ROS 2 dành cho vi điều khiển (MCU). |
| **rcl / rclc** | Thư viện lõi của ROS 2 Client Library, cung cấp API cho Node, Publisher, Executor, v.v. |
| **rmw (ROS Middleware)** | Lớp trung gian giữa ROS 2 và DDS/micro-XRCE-DDS. |
| **colcon build** | Hệ thống build ROS 2 / micro-ROS khi biên dịch từ source. |
| **PlatformIO** | Công cụ build và quản lý thư viện cho các vi điều khiển. |
| **Precompiled Libraries** | Thư viện đã biên dịch sẵn, không cần colcon build. |
| **set_microros_transports()** | Hàm khởi tạo phương thức giao tiếp (Serial, UDP, WiFi...) giữa MCU và ROS 2 Agent. |

---

## 💥 2. Các lỗi thường gặp & Hướng khắc phục

### 🔸 Lỗi 1: Không clone được thư viện từ GitHub

**Thông báo lỗi:**
```bash
fatal: unable to access 'https://github.com/micro-ROS/micro_ros_arduino.git/': Couldn't connect to server
VCSBaseException: Could not process command ['git', 'clone', ...]
```

**Nguyên nhân:**
- Kết nối mạng bị chặn hoặc GitHub HTTPS timeout.  
- Sai thư viện (dùng `micro_ros_arduino` thay vì `micro_ros_platformio`).

**Cách khắc phục:**
```ini
lib_deps = https://github.com/micro-ROS/micro_ros_platformio
```
Hoặc tải thủ công:
```bash
cd ~/.platformio/lib
git clone https://github.com/micro-ROS/micro_ros_platformio.git
```

---

### 🔸 Lỗi 2: `undefined reference to rcl_publish`, `rclc_support_init`, ...

**Thông báo lỗi:**
```bash
undefined reference to `rcl_publish'
undefined reference to `rclc_support_init'
undefined reference to `rclc_node_init_default'
```

**Nguyên nhân:**
- Không dùng bản precompiled của micro-ROS.
- Thiếu hàm `set_microros_transports()`.

**Cách khắc phục:**
```ini
lib_deps = micro-ROS/micro_ros_platformio@^2.0.2
build_flags = -DMICRO_ROS_USE_PRECOMPILED_LIBS
```

Trong `main.cpp`:
```cpp
set_microros_transports();
```

Sau đó clean lại dự án:
```bash
pio run -t clean
rm -rf .pio
pio run
```

---

### 🔸 Lỗi 3: `colcon build failed` hoặc `Could not find package rmw`

**Thông báo lỗi:**
```bash
Build dev micro-ROS environment failed:
Could not find a package configuration file provided by "rmw"
```

**Nguyên nhân:**
- PlatformIO cố build môi trường dev (dạng ROS 2 workspace).
- Không có ROS 2 hoặc thiếu `-DMICRO_ROS_USE_PRECOMPILED_LIBS`.

**Khắc phục:**
```ini
build_flags = -DMICRO_ROS_USE_PRECOMPILED_LIBS
```
Không để dòng `extra_scripts` gọi `colcon`.

---

### 🔸 Lỗi 4: Cảnh báo vàng tại `rcl_publish(&publisher, &sensor_msg, NULL);`

**Cảnh báo:**
```bash
warning: ignoring return value of 'rcl_publish(...)'
```

**Nguyên nhân:**
- Cảnh báo compile-time, không ảnh hưởng hoạt động.
- Cần xử lý hoặc bỏ qua giá trị trả về.

**Cách khắc phục:**
```cpp
(void) rcl_publish(&publisher, &sensor_msg, NULL);
```

---

### 🔸 Lỗi 5: `std_msgs__msg_UInt16` không tồn tại

**Thông báo:**
```bash
error: 'std_msgs__msg_UInt16' does not name a type; did you mean 'std_msgs__msg__UInt16'?
```

**Nguyên nhân:** Sai tên kiểu dữ liệu ROS 2 message.

**Khắc phục:**
```cpp
std_msgs__msg__UInt16 sensor_msg;
```

---

## ⚙️ 3. Cấu trúc dự án mẫu

### 📄 `platformio.ini`
```ini
[env:nodemcu-32s]
platform = espressif32
board = nodemcu-32s
framework = arduino
monitor_speed = 115200

lib_ldf_mode = deep
lib_deps = micro-ROS/micro_ros_platformio@^2.0.2
build_flags = -DMICRO_ROS_USE_PRECOMPILED_LIBS
```

---

### 📄 `src/main.cpp`
```cpp
#include <Arduino.h>
#include <Wire.h>
#include <micro_ros_platformio.h>
#include <rcl/rcl.h>
#include <rclc/executor.h>
#include <std_msgs/msg/u_int16.h>

#define PCF8575_ADDR 0x20
#define INT_PIN 16

volatile bool flagInterrupt = false;

rcl_publisher_t publisher;
std_msgs__msg__UInt16 sensor_msg;
rclc_support_t support;
rcl_allocator_t allocator;
rcl_node_t node;
rclc_executor_t executor;

void IRAM_ATTR handleInterrupt() {
  flagInterrupt = true;
}

uint16_t readPCF8575() {
  uint16_t value = 0xFFFF;
  Wire.requestFrom(PCF8575_ADDR, 2);
  if (Wire.available() == 2) {
    uint8_t lowByte = Wire.read();
    uint8_t highByte = Wire.read();
    value = (highByte << 8) | lowByte;
  }
  return value;
}

void setup() {
  Serial.begin(115200);
  delay(2000);

  Wire.begin(21, 22);
  pinMode(INT_PIN, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(INT_PIN), handleInterrupt, FALLING);

  Serial.println("=== micro-ROS PCF8575 Line Sensor Node ===");

  set_microros_transports();
  allocator = rcl_get_default_allocator();

  rclc_support_init(&support, 0, NULL, &allocator);
  rclc_node_init_default(&node, "esp32_line_sensor", "", &support);
  rclc_publisher_init_default(
    &publisher,
    &node,
    ROSIDL_GET_MSG_TYPE_SUPPORT(std_msgs, msg, UInt16),
    "line_sensors"
  );
  rclc_executor_init(&executor, &support.context, 1, &allocator);

  Serial.println("ROS 2 Node ready. Waiting for sensor changes...");
}

void loop() {
  if (flagInterrupt) {
    flagInterrupt = false;
    delay(5);
    uint16_t sensor_data = readPCF8575();
    sensor_msg.data = sensor_data;
    (void) rcl_publish(&publisher, &sensor_msg, NULL);
    Serial.print("Sensor data: ");
    Serial.println(sensor_data, BIN);
  }
  rclc_executor_spin_some(&executor, RCL_MS_TO_NS(10));
}
```

---

## 🧾 4. Bài học rút ra

| Bài học | Giải thích |
|----------|-------------|
| 🧩 **Phân biệt thư viện Arduino và PlatformIO** | `micro_ros_arduino` dùng cho Arduino IDE, còn PlatformIO cần `micro_ros_platformio`. |
| ⚙️ **Không để PlatformIO tự colcon build** | ESP32 không có môi trường ROS 2 đầy đủ → phải dùng precompiled libs. |
| 🧠 **Gọi `set_microros_transports()` đúng lúc** | Nếu thiếu, linker sẽ báo lỗi `undefined reference to rmw_uros_*`. |
| 🧹 **Dọn dẹp cache khi gặp lỗi lạ** | Dùng `pio run -t clean && rm -rf .pio` để rebuild sạch. |
| ✅ **Chú ý tên kiểu dữ liệu chuẩn (`__`)** | micro-ROS dùng struct C-style có hai dấu gạch dưới. |

---

## 🚀 5. Hướng phát triển tiếp theo

1. **Kiểm tra kết nối với micro-ROS Agent:**
   ```bash
   ros2 run micro_ros_agent micro_ros_agent serial --dev /dev/ttyUSB0
   ```

2. **Kiểm tra topic publish:**
   ```bash
   ros2 topic echo /line_sensors
   ```

3. **Mở rộng sang Wi-Fi UDP transport** để truyền dữ liệu không dây.  
4. **Tích hợp AI/ROS2 Node** để nhận dạng vạch đường & điều khiển robot AGV.

---

🧰 **Tác giả:** *Ben Phan – RSPAIoT Project*  
📅 **Phiên bản:** 2025-10-09  
📘 **License:** MIT  

