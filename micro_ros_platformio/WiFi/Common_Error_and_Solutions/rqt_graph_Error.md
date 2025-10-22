# 🧩 Hướng Dẫn Hiển Thị Node micro-ROS ESP32 Trên `rqt_graph`

## 📘 Giới Thiệu

Khi sử dụng **ESP32 kết hợp micro-ROS** để giao tiếp với **ROS 2 (Humble/Foxy)**, `rqt_graph` là công cụ GUI giúp hiển thị các node và topic trong hệ thống. 
Tuy nhiên, nhiều trường hợp node không xuất hiện dù ESP32 vẫn gửi dữ liệu UDP/Serial. 
Tài liệu này tổng hợp **nguyên nhân, cách khắc phục và mẹo xử lý triệt để**.

---

## 🧠 1. Nguyên Nhân Khi Node Không Hiển Thị Trên `rqt_graph`

### 1.1. micro-ROS Agent chưa chạy khi ESP32 kết nối
- Khi ESP32 gửi gói handshake mà Agent chưa sẵn sàng, quá trình khởi tạo node thất bại.
- Sau đó, dù Agent khởi động, ESP32 không tự gửi lại yêu cầu đăng ký.

✅ **Cách khắc phục:**
- Khởi chạy agent trước rồi mới cấp nguồn cho ESP32:
  ```bash
  ros2 run micro_ros_agent micro_ros_agent udp4 --port 8888 -v6
  ```
- Hoặc thêm kiểm tra kết nối trong code ESP32 bằng:
  ```cpp
  while (RMW_RET_OK != rmw_uros_ping_agent(1000, 3)) {
      Serial.println("Agent not available, retrying...");
      delay(1000);
  }
  ```

---

### 1.2. Domain ID giữa Agent và ESP32 khác nhau
- ROS 2 sử dụng `ROS_DOMAIN_ID` để tách mạng DDS.
- Nếu khác domain, ESP32 và `rqt_graph` sẽ không thấy nhau.

✅ **Cách khắc phục:**
- Đảm bảo cả hai cùng domain ID:
  ```bash
  export ROS_DOMAIN_ID=0
  micro-ros-agent udp4 --port 8888 -v6 -d 0
  ```

---

### 1.3. Node ESP32 chưa được khởi tạo đúng
- Code thiếu các bước như `rclc_node_init_default()` hoặc `rclc_executor_init()`.
- Khi đó, Agent vẫn nhận UDP nhưng không tạo entity trong DDS.

✅ **Kiểm tra:** nhật ký Serial nếu có lỗi kiểu `Error creating node: 1`.

Ví dụ mẫu node hoạt động đúng:

```cpp
#include <micro_ros_arduino.h>
#include <rcl/rcl.h>
#include <rclc/executor.h>
#include <rclc/rclc.h>
#include <std_msgs/msg/string.h>

rcl_publisher_t publisher;
std_msgs__msg__String msg;
rclc_executor_t executor;
rclc_support_t support;
rcl_allocator_t allocator;
rcl_node_t node;
rcl_timer_t timer;

void timer_callback(rcl_timer_t * timer, int64_t last_call_time) {
  (void) last_call_time;
  if (timer != NULL) {
    msg.data.data = (char*)"Hello from ESP32 via WiFi!";
    msg.data.size = strlen(msg.data.data);
    msg.data.capacity = msg.data.size + 1;
    rcl_publish(&publisher, &msg, NULL);
  }
}

void setup() {
  set_microros_wifi_transports("SSID", "PASS", "192.168.1.100", 8888);
  allocator = rcl_get_default_allocator();
  rclc_support_init(&support, 0, NULL, &allocator);
  rclc_node_init_default(&node, "esp32_node", "", &support);
  rclc_publisher_init_default(&publisher, &node,
      ROSIDL_GET_MSG_TYPE_SUPPORT(std_msgs, msg, String), "esp32_wifi_chatter");
  rclc_timer_init_default(&timer, &support, RCL_MS_TO_NS(1000), timer_callback);
  rclc_executor_init(&executor, &support.context, 1, &allocator);
  rclc_executor_add_timer(&executor, &timer);
}

void loop() {
  rclc_executor_spin_some(&executor, RCL_MS_TO_NS(100));
  delay(100);
}
```

---

## 🧩 2. Các Lỗi Thường Gặp Và Cách Xử Lý

| Lỗi | Nguyên Nhân | Cách Xử Lý |
|------|---------------|------------|
| Không thấy node trên `rqt_graph` | Agent chưa chạy khi ESP32 gửi handshake | Chạy Agent trước hoặc dùng `rmw_uros_ping_agent()` |
| Phải reset ESP32 mỗi khi restart Agent | Mất session UDP | Viết code tự reconnect hoặc reset tự động |
| `ros2 node list` rỗng | Node chưa được tạo | Kiểm tra code init node |
| `Failed to delete datawriter` | Cảnh báo shutdown DDS | Bỏ qua, không ảnh hưởng |
| Không hiển thị GUI `rqt_graph` | Thiếu Qt binding | Cài `sudo apt install python3-pyqt5 python3-pydot graphviz` |

---

## ⚙️ 3. Lưu Ý Khi Dùng `rqt_graph`

- Kiểm tra domain ID: `echo $ROS_DOMAIN_ID`
- Kiểm tra node thực tế: `ros2 node list`
- Kiểm tra topic: `ros2 topic list`
- Dùng lệnh `micro-ros-agent udp4 --port 8888 -v6` để bật log chi tiết
- Nếu đang chạy trên WSL → bật X11 server (VcXsrv hoặc Xming) và `export DISPLAY=:0`

---

## ✅ 4. Kết Luận

Hiện tượng không thấy node trên `rqt_graph` **không phải lỗi phần cứng** mà do **mất đồng bộ handshake hoặc khác domain**.  
Giải pháp an toàn nhất là:
1. **Khởi chạy Agent trước ESP32**
2. **Thêm đoạn kiểm tra ping agent trong code**
3. **Đảm bảo domain ID đồng nhất**

Với các bước này, node ESP32 sẽ hiển thị ổn định trên `rqt_graph` mà không cần reset thủ công.
