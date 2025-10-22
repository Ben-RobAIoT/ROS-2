
# 🔌 Kiểm tra và Nhận dạng Thiết bị USB trong Linux (Dành cho ROS)

Khi làm việc với **ROS** và các thiết bị ngoại vi như **LiDAR, Arduino, IMU, Camera**, việc xác định đúng **cổng USB** là rất quan trọng để tránh lỗi kết nối.

Tài liệu này hướng dẫn các **cách kiểm tra thiết bị USB trong Linux** ngoài lệnh quen thuộc:

```bash
ls /dev/tty*
```

---

## 🧩 1. Xem log kernel khi cắm USB (`dmesg`)

Ngay sau khi cắm thiết bị, chạy lệnh:
```bash
dmesg | tail -n 20
```
Hoặc theo dõi theo thời gian thực:
```bash
dmesg -w
```

Ví dụ:
```
[12345.678901] usb 1-1: FTDI USB Serial Device converter now attached to ttyUSB0
```
➡️ Thiết bị của bạn được gán thành `/dev/ttyUSB0`.

---

## ⚙️ 2. Xem danh sách thiết bị USB (`lsusb`)

```bash
lsusb
```

Ví dụ:
```
Bus 001 Device 007: ID 0403:6001 Future Technology Devices International, Ltd FT232 Serial (UART) IC
```

Dòng này cho biết loại chip giao tiếp (FTDI, CH340, CP2102, ...).  
Điều này giúp ROS xác định đúng driver.

---

## 🔍 3. Truy vết chi tiết thiết bị (`udevadm info`)

Nếu biết cổng là `/dev/ttyUSB0`, xem thông tin chi tiết:
```bash
udevadm info -a -n /dev/ttyUSB0
```

Hiển thị nhiều thông tin như:
```
ATTRS{idVendor}=="0403"
ATTRS{idProduct}=="6001"
ATTRS{serial}=="A9XYZ123"
```

👉 Dựa vào đó, bạn có thể tạo **udev rule** để gán tên cố định (ví dụ `/dev/lidar`).

---

## 🧾 4. Đường dẫn ổn định (ROS nên dùng)

Để tránh thay đổi cổng khi khởi động lại, dùng:
```bash
ls -l /dev/serial/by-id/
```

Ví dụ kết quả:
```
usb-FTDI_FT232_USB_UART_A9XYZ123-if00-port0 -> ../../ttyUSB0
```

➡️ Trong ROS, bạn nên sử dụng đường dẫn này:
```xml
<param name="port" value="/dev/serial/by-id/usb-FTDI_FT232_USB_UART_A9XYZ123-if00-port0"/>
```

---

## 🧰 5. Giám sát sự kiện cắm/rút USB (`udevadm monitor`)

Nếu bạn muốn xem thay đổi USB theo thời gian thực:
```bash
udevadm monitor
```

Hiển thị các sự kiện khi thiết bị được cắm hoặc rút ra — rất hữu ích để debug khi ROS node mất kết nối.

---

## 🧠 6. Kiểm tra cổng trực tiếp trong ROS Launch File

Ví dụ với `rosserial_python`:
```xml
<node pkg="rosserial_python" type="serial_node.py" name="serial_node" output="screen">
  <param name="port" value="/dev/serial/by-id/usb-Arduino_Uno_1234-if00"/>
  <param name="baud" value="57600"/>
</node>
```

Nếu cổng không tồn tại, ROS sẽ báo lỗi:
```
[ERROR] [serial.SerialException]: could not open port /dev/ttyUSB0
```

---

## 🧪 7. Kiểm tra dữ liệu cổng thủ công (`minicom` hoặc `screen`)

Cài và sử dụng:
```bash
sudo apt install minicom
minicom -D /dev/ttyUSB0 -b 115200
```

Nếu có dữ liệu phản hồi → thiết bị hoạt động bình thường.

---

## 🧩 Tóm tắt nhanh

| Mục đích | Lệnh |
|-----------|------|
| Liệt kê toàn bộ cổng serial | `ls /dev/tty*` |
| Xem log khi cắm USB | `dmesg | tail -n 20` |
| Xem danh sách thiết bị USB | `lsusb` |
| Xem thông tin chi tiết thiết bị | `udevadm info -a -n /dev/ttyUSB0` |
| Đường dẫn ổn định (ROS nên dùng) | `ls -l /dev/serial/by-id/` |
| Giám sát cắm/rút USB | `udevadm monitor` |
| Kiểm tra dữ liệu port | `minicom -D /dev/ttyUSB0 -b 115200` |

---

## 💡 Gợi ý thêm: Tạo udev rule để đặt tên cố định

Tạo file rule:
```bash
sudo nano /etc/udev/rules.d/99-usb-lidar.rules
```

Thêm nội dung (thay giá trị vendor/product/serial của bạn):
```
SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", ATTRS{serial}=="A9XYZ123", SYMLINK+="lidar"
```

Lưu lại và reload rule:
```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Giờ thiết bị sẽ luôn có cổng:
```
/dev/lidar
```

Dùng trong ROS:
```xml
<param name="port" value="/dev/lidar"/>
```

---

## 🧭 Tổng kết

Khi làm việc với ROS, việc nhận dạng và cố định cổng USB là **bắt buộc** để tránh lỗi "port not found".  
Sử dụng `/dev/serial/by-id` hoặc **udev rule** là cách an toàn, ổn định và chuyên nghiệp nhất.

---

📘 **Tác giả:** Tín Phan  
💡 **Chủ đề:** Kiểm tra & quản lý thiết bị USB khi làm việc với ROS trên Linux  
🧠 **Mục tiêu:** Giúp đảm bảo thiết bị được nhận diện đúng và ổn định khi triển khai hệ thống ROS thực tế.
