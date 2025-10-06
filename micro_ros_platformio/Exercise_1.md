# 🚀 Hướng Dẫn Kết Nối ROS 2 Và ESP32 Qua Serial (micro-ROS Publisher)

---

## 🧩 Bước 1: Cài đặt các thư viện và công cụ cần thiết

Trên **máy tính (Ubuntu)** bạn cần cài:

```bash
sudo apt update
sudo apt install git cmake python3-pip
```

Cài đặt ROS 2 (ví dụ bản Humble Hawksbill):

```bash
sudo apt install ros-humble-desktop
source /opt/ros/humble/setup.bash
```

Cài micro-ROS build system:

```bash
mkdir -p ~/microros_ws/src
cd ~/microros_ws/src
git clone -b humble https://github.com/micro-ROS/micro_ros_setup.git
cd ~/microros_ws
rosdep install --from-paths src --ignore-src -y
colcon build
source install/local_setup.bash
```

---

## ⚙️ Bước 2: Tạo PlatformIO Workspace

1. Mở **VSCode**
2. Cài plugin **PlatformIO IDE**
3. Chọn **New Project → ESP32 Dev Module → Framework: Arduino**
4. Chờ PlatformIO tạo xong project (thư mục `src/`, `platformio.ini`…)

---

## 🧾 Bước 3: Cấu hình trong `platformio.ini`

Thêm dòng sau vào file `platformio.ini` (ở thư mục gốc của project):

```ini
lib_deps =
    https://github.com/micro-ROS/micro_ros_platformio
board_microros_distro = humble
board_microros_transport = serial
```

> 💡 **Lưu ý:**  
> - Nhấn **Ctrl+S** để PlatformIO tự động tải và cài các thư viện cần thiết.  
> - Quá trình này có thể mất vài phút (chỉ làm lần đầu tiên).

---

## 🧠 Bước 4: Nạp chương trình lên ESP32

Copy toàn bộ đoạn mã sau vào `src/main.cpp`:

```cpp
#include <Arduino.h>
#include <micro_ros_platformio.h>

#include <rcl/rcl.h>
#include <rclc/rclc.h>
#include <rclc/executor.h>
#include <std_msgs/msg/int32.h>

#if !defined(MICRO_ROS_TRANSPORT_ARDUINO_SERIAL)
#error This example is only avaliable for Arduino framework with serial transport.
#endif

rcl_publisher_t publisher;
std_msgs__msg__Int32 msg;

rclc_executor_t executor;
rclc_support_t support;
rcl_allocator_t allocator;
rcl_node_t node;
rcl_timer_t timer;

#define RCCHECK(fn) { rcl_ret_t temp_rc = fn; if((temp_rc != RCL_RET_OK)){error_loop();}}
#define RCSOFTCHECK(fn) { rcl_ret_t temp_rc = fn; if((temp_rc != RCL_RET_OK)){}}

void error_loop() {
  while(1) { delay(100); }
}

void timer_callback(rcl_timer_t * timer, int64_t last_call_time) {
  RCLC_UNUSED(last_call_time);
  if (timer != NULL) {
    RCSOFTCHECK(rcl_publish(&publisher, &msg, NULL));
    msg.data++;
  }
}

void setup() {
  Serial.begin(115200);
  set_microros_serial_transports(Serial);
  delay(2000);

  allocator = rcl_get_default_allocator();

  RCCHECK(rclc_support_init(&support, 0, NULL, &allocator));
  RCCHECK(rclc_node_init_default(&node, "micro_ros_platformio_node", "", &support));

  RCCHECK(rclc_publisher_init_default(
    &publisher,
    &node,
    ROSIDL_GET_MSG_TYPE_SUPPORT(std_msgs, msg, Int32),
    "micro_ros_platformio_node_publisher"));

  const unsigned int timer_timeout = 1000;
  RCCHECK(rclc_timer_init_default(
    &timer,
    &support,
    RCL_MS_TO_NS(timer_timeout),
    timer_callback));

  RCCHECK(rclc_executor_init(&executor, &support.context, 1, &allocator));
  RCCHECK(rclc_executor_add_timer(&executor, &timer));

  msg.data = 0;
}

void loop() {
  delay(100);
  RCSOFTCHECK(rclc_executor_spin_some(&executor, RCL_MS_TO_NS(100)));
}
```

> ⚡ **Thao tác:**  
> - Nhấn **Build** để kiểm tra lỗi biên dịch.  
> - Sau đó nhấn **Upload** để nạp chương trình vào ESP32.

---

## 🖧 Bước 5: Kết nối và chạy micro-ROS Agent

1. **Trở lại máy tính Ubuntu:**

```bash
cd ~/microros_ws
ros2 run micro_ros_setup create_agent_ws.sh
ros2 run micro_ros_setup build_agent.sh
source install/local_setup.bash
```

2. **Chạy micro-ROS Agent qua Serial:**

```bash
ros2 run micro_ros_agent micro_ros_agent serial --dev /dev/ttyUSB0
```

> 🔁 Nếu không thấy ESP32 phản hồi, hãy **nhấn nút reset** hoặc **rút ra – cắm lại cổng USB**.  
> Dòng `/dev/ttyUSB0` có thể khác (kiểm tra bằng: `ls /dev/ttyUSB*`).

---

## 🧩 Bước 6: Kiểm tra kết quả trên ROS 2

1. Mở **terminal mới**, nạp môi trường ROS:

```bash
source /opt/ros/humble/setup.bash
```

2. Kiểm tra xem node micro-ROS đã hoạt động chưa:

```bash
ros2 topic list
```

Kết quả mong đợi:

```
/micro_ros_platformio_node_publisher
/parameter_events
/rosout
```

3. Hiển thị dữ liệu mà ESP32 gửi:

```bash
ros2 topic echo /micro_ros_platformio_node_publisher
```

> 🟢 Bạn sẽ thấy dữ liệu tăng dần như:

```
data: 0
---
data: 1
---
data: 2
---
```

---

## 🖼️ Bước 7: (Tuỳ chọn) Mở rộng

Bạn có thể thêm:
- **Subscriber** để nhận lệnh từ ROS 2 → bật/tắt LED trên ESP32  
- **WiFi transport** thay vì Serial (dùng `set_microros_wifi_transports()`)  
- **Custom messages** để gửi nhiều loại dữ liệu hơn  

---

## ✅ Tóm tắt nhanh luồng thực thi

| Thành phần | Vai trò | Công cụ |
|-------------|----------|----------|
| **ESP32** | micro-ROS Client (Publisher) | PlatformIO + Arduino |
| **ROS 2 (PC)** | micro-ROS Agent + ROS Master | Ubuntu + ROS 2 Humble |
| **Giao tiếp** | Serial UART (USB) | `/dev/ttyUSB0` |

---

📘 *Tác giả: Tín Phan*  
💡 *Phiên bản: ROS 2 Humble + ESP32 (micro-ROS PlatformIO)*

