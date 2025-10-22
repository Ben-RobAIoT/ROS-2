
# 📡 Các Lỗi Cơ Bản Khi Làm Việc Với micro-ROS Qua Wi-Fi (ESP32)

Khi phát triển ứng dụng **micro-ROS** trên các vi điều khiển như **ESP32**, việc kết nối qua **Wi-Fi** là một bước phổ biến nhưng cũng dễ phát sinh lỗi.  
Tài liệu này tổng hợp **các lỗi cơ bản, nguyên nhân và cách khắc phục**, giúp bạn dễ dàng debug khi kết nối **micro-ROS Agent** và **Client (ESP32)**.

---

## ⚠️ 1. Không thể kết nối Wi-Fi (Lỗi thường gặp nhất)

### 🧠 Nguyên nhân:
- ESP32 **chỉ hỗ trợ Wi-Fi 2.4GHz**, **không hỗ trợ 5GHz**.
- SSID chứa **khoảng trắng hoặc ký tự đặc biệt**.
- Mật khẩu Wi-Fi dài hơn 63 ký tự hoặc sai định dạng.
- Router chặn thiết bị IoT (MAC Filtering).

### 🧰 Cách khắc phục:
- Kiểm tra lại router, đảm bảo đang phát sóng **2.4GHz**.
- Đặt SSID ngắn, không có ký tự lạ.
- Kiểm tra mật khẩu, không copy thêm ký tự khoảng trắng.
- Thử tạo **Hotspot từ điện thoại** để test.

### 🔍 Ví dụ log lỗi:
```
[WiFi]: Connecting to SSID MyHome5G ...
[WiFi]: Connection failed
```
➡️ Đổi lại sang Wi-Fi 2.4GHz (`MyHome_2.4G`).

---

## 🌐 2. Không kết nối được với micro-ROS Agent (Timeout)

### 🧠 Nguyên nhân:
- ESP32 chưa kết nối mạng thành công.
- IP hoặc port của micro-ROS Agent sai.
- Agent chưa được khởi động, hoặc bị chặn bởi tường lửa.
- ESP32 bị lỗi DNS (nếu sử dụng hostname thay vì IP).

### 🧰 Cách khắc phục:
- Kiểm tra IP của máy tính chạy micro-ROS Agent:
  ```bash
  ifconfig  # hoặc ip addr show
  ```
- Đảm bảo Agent đang chạy:
  ```bash
  ros2 run micro_ros_agent micro_ros_agent udp4 --port 8888
  ```
- Ping thử IP từ ESP32 (hoặc mô phỏng trong code).

### 🔍 Ví dụ log:
```
[ERROR] [RMW_UXRCE_TRANSPORT_ERROR] Connection timed out
```
➡️ Kiểm tra IP, port và trạng thái Wi-Fi.

---

## 🚧 3. Sai cấu hình TRANSPORT (UDP/TCP)

### 🧠 Nguyên nhân:
- micro-ROS Agent đang chạy ở **UDP**, nhưng code ESP32 lại cấu hình **TCP**, hoặc ngược lại.
- Port không khớp giữa 2 bên.

### 🧰 Cách khắc phục:
- Kiểm tra file cấu hình trong ESP32:
  ```cpp
  set_microros_wifi_transports("MySSID", "password", "192.168.1.10", 8888);
  ```
- Kiểm tra Agent:
  ```bash
  ros2 run micro_ros_agent micro_ros_agent udp4 --port 8888
  ```
- Đảm bảo cả hai cùng dùng **UDP hoặc TCP**, và **port trùng nhau**.

---

## 🔌 4. ESP32 không có đủ RAM hoặc stack

### 🧠 Nguyên nhân:
- Một số board ESP32 đời thấp (ESP32-WROOM, WROVER 4MB) có bộ nhớ RAM hạn chế.
- Khi micro-ROS tạo nhiều publisher/subscriber, bộ nhớ bị đầy.

### 🧰 Cách khắc phục:
- Giảm số lượng topic, node hoặc độ lớn message.
- Dùng ESP32-WROVER với PSRAM hoặc ESP32-S3.
- Kiểm tra log debug:
  ```
  [uxr_client] Error: Not enough memory to allocate entities
  ```

---

## 🧱 5. Lỗi MTU / gói tin bị cắt khi gửi qua Wi-Fi

### 🧠 Nguyên nhân:
- Một số router hoặc AP cũ không hỗ trợ gói UDP lớn.
- micro-ROS gửi message lớn (ví dụ: array hoặc image).

### 🧰 Cách khắc phục:
- Giảm kích thước message ROS2.
- Dùng **QoS = Best Effort** thay vì **Reliable**.
- Nếu cần truyền dữ liệu lớn → dùng **micro-ROS + Ethernet (W5500)**.

---

## 🧩 6. Agent và Client không cùng subnet

### 🧠 Nguyên nhân:
- ESP32 và PC Agent không cùng mạng (ví dụ: ESP32 nối qua hotspot khác).
- Dải IP khác nhau (VD: 192.168.1.x và 10.0.0.x).

### 🧰 Cách khắc phục:
- Đảm bảo cùng dải mạng.
- Kiểm tra IP:
  ```bash
  ping 192.168.1.101
  ```
- Nếu không ping được → kiểm tra lại cấu hình router hoặc AP.

---

## 📶 7. micro-ROS Agent bị crash hoặc mất kết nối sau thời gian chạy

### 🧠 Nguyên nhân:
- Agent hoặc network bị gián đoạn.
- ESP32 reconnect Wi-Fi mà không reset micro-ROS client.
- Quên gọi `rmw_uros_ping_agent()` để kiểm tra trạng thái.

### 🧰 Cách khắc phục:
- Trong vòng lặp chính của ESP32, thêm kiểm tra Agent:
  ```cpp
  if (rmw_uros_ping_agent(100, 1) != RMW_RET_OK) {
      // reconnect Wi-Fi + reinit micro-ROS
  }
  ```
- Dùng watchdog để tự khởi động lại thiết bị khi mất kết nối lâu.

---

## 🧪 8. Không gửi/nhận được message ROS2

### 🧠 Nguyên nhân:
- QoS không khớp giữa Agent và Client.
- Topic name sai hoặc bị thiếu prefix `/`.
- Cấu hình message không khớp với file `.msg` của ROS2.

### 🧰 Cách khắc phục:
- Kiểm tra QoS của publisher/subscriber (BestEffort/Reliable).
- Kiểm tra tên topic khớp nhau:
  ```cpp
  rcl_publisher_init(&publisher, &node, type_support, "esp32/status", &pub_options);
  ```
- Sử dụng `ros2 topic list` để kiểm tra trên PC Agent.

---

## 🧾 9. micro-ROS build lỗi hoặc không flash được

### 🧠 Nguyên nhân:
- Thư viện `micro_ros_espidf_component` không đúng phiên bản.
- Cấu hình menuconfig sai.
- Không đủ flash hoặc heap.

### 🧰 Cách khắc phục:
```bash
idf.py fullclean
idf.py build
idf.py flash monitor
```
- Kiểm tra log lỗi cụ thể.
- Nâng cấp ESP-IDF lên phiên bản 5.x để hỗ trợ tốt hơn.

---

## 🧭 Tổng kết

| Nhóm lỗi | Nguyên nhân chính | Giải pháp |
|-----------|------------------|------------|
| Không kết nối Wi-Fi | Dùng Wi-Fi 5GHz, SSID sai | Dùng 2.4GHz, kiểm tra SSID |
| Không kết nối Agent | Sai IP/Port, chưa chạy Agent | Kiểm tra IP, chạy lại Agent |
| Sai transport | UDP/TCP không khớp | Đảm bảo cùng giao thức |
| Thiếu RAM | Board nhỏ, code nặng | Giảm topic, chọn ESP32 có PSRAM |
| Gói tin lỗi | MTU hoặc QoS không phù hợp | Dùng BestEffort hoặc giảm payload |
| Sai mạng | Không cùng subnet | Kiểm tra dải IP |
| Mất kết nối | Không kiểm tra Agent | Dùng `rmw_uros_ping_agent()` |

---

📘 **Tác giả:** Tín Phan  
💡 **Chủ đề:** Các lỗi cơ bản khi sử dụng micro-ROS qua Wi-Fi với ESP32  
🧠 **Mục tiêu:** Giúp lập trình viên debug nhanh khi triển khai micro-ROS trong môi trường IoT hoặc Robot nhỏ gọn.
