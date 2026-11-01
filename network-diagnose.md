# Báo Cáo Chẩn Đoán Mạng & Xử Lý Cổng (Session 01 - Exercise 05)

## 1. Tình huống 1: Giải quyết xung đột cổng 8080

### Hiện tượng
Khi khởi chạy service, ứng dụng báo lỗi cổng `8080` đã bị chiếm dụng bởi một tiến trình khác (`Address already in use: bind`).

### Các bước xử lý
1. **Kiểm tra tiến trình đang chiếm cổng:**
   ```bash
   sudo ss -tulpn | grep :8080
   ```
   *Mục đích:* Dùng lệnh `ss` với các cờ `t` (TCP), `u` (UDP), `l` (listening), `p` (processes), `n` (numeric) để lọc cổng 8080 và trích xuất PID của tiến trình gây xung đột (ví dụ: PID `14238`).

2. **Ngắt tiến trình chiếm dụng:**
   ```bash
   sudo kill -9 14238
   ```
   *Mục đích:* Gửi tín hiệu `SIGKILL` buộc tiến trình PID 14238 dừng ngay lập tức và giải phóng socket port 8080.

3. **Xác nhận giải phóng cổng:**
   ```bash
   sudo ss -tulpn | grep :8080
   ```
   *Kết quả:* Không có tiến trình nào trả về, cổng 8080 đã sẵn sàng để deploy service mới.

---

## 2. Tình huống 2: Chẩn đoán lỗi kết nối Database PostgreSQL (IP 10.0.1.20)

### Hiện tượng
Ứng dụng báo lỗi không thể kết nối tới máy chủ Database PostgreSQL tại địa chỉ `10.0.1.20:5432`.

### Các bước chẩn đoán theo tầng mạng (OSI)
1. **Kiểm tra đường truyền vật lý / kết nối mạng (Layer 3 - ICMP):**
   ```bash
   ping -c 4 10.0.1.20
   ```
   *Mục đích:* Gửi 4 gói tin ICMP echo request đến `10.0.1.20`.
   - Nếu `0% packet loss`: Đường truyền mạng thông suốt, routing IP chính xác.
   - Nếu `100% packet loss` hoặc `Request timeout`: Máy chủ Database bị tắt, card mạng hỏng hoặc Security Group / Firewall chặn gói tin ICMP.

2. **Kiểm tra trạng thái mở cổng kết nối Database (Layer 4 - Transport TCP):**
   ```bash
   nc -zv 10.0.1.20 5432
   ```
   *Hoặc kiểm tra chi tiết bằng curl:*
   ```bash
   curl -v telnet://10.0.1.20:5432
   ```
   *Mục đích:* Thăm dò xem cổng 5432 tại `10.0.1.20` có đang ở trạng thái `LISTEN` hay không.
   - Kết quả `Connection to 10.0.1.20 5432 port [tcp/postgresql] succeeded!`: Cổng mở, kết nối TCP bắt tay thành công. Lỗi có thể nằm ở credentials (sai username/password) hoặc `pg_hba.conf` không cho phép IP máy chủ gọi vào.
   - Kết quả `Connection refused`: Máy chủ nhận được request nhưng không có tiến trình nào lắng nghe tại port 5432 (PostgreSQL chưa start hoặc cấu hình `listen_addresses = 'localhost'` thay vì `'*'`).
   - Kết quả `Connection timed out`: Firewall (iptables / cloud security group) đang drop các gói tin TCP hướng tới port 5432.
