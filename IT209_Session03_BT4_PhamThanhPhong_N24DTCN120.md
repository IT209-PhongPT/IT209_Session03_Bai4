# Báo cáo Bài tập 4: Chạy song song nhiều Cổng dịch vụ (Nginx Port-based Virtual Hosts)

**Thông tin sinh viên:**
- **Họ và tên:** Phạm Thanh Phong
- **Mã sinh viên / Lớp:** N24DTCN120
- **Môn học:** IT209 - Phát triển ứng dụng Web / Cloud Infrastructure
- **Session:** Session 03 - Bài tập 4

---

## 1. Mục tiêu & Bối cảnh bài toán

### 1.1. Mục tiêu
- Triển khai kiến trúc **Port-based Virtual Hosting** (Lưu trữ ảo dựa trên cổng) trên Web Server Nginx nhằm phục vụ đồng thời nhiều ứng dụng web độc lập trên cùng một địa chỉ IP máy chủ duy nhất.
- Phục vụ hai môi trường ứng dụng riêng biệt:
  1. **Beta Application (Ứng dụng thử nghiệm):** Lắng nghe trên cổng **`8080`**.
  2. **Internal Application (Cổng thông tin nội bộ):** Lắng nghe trên cổng **`8090`**.
- Mở và quản lý phân luồng cổng chính xác trên cả 2 tầng tường lửa: **UFW (Host Firewall)** và **DigitalOcean Cloud Firewall (Edge Firewall)**.

### 1.2. Ràng buộc
- Thư mục mã nguồn:
  - Beta App: `/var/www/beta-app/html/index.html`.
  - Internal App: `/var/www/internal-app/html/index.html`.
- Cấu hình Nginx: Sử dụng tệp `/etc/nginx/sites-available/multi-port.conf` chứa 2 khối `server { ... }` riêng biệt lắng nghe các cổng `8080` và `8090`.
- Mở cổng `8080/tcp` và `8090/tcp` trên UFW và Cloud Firewall.
- Kiểm tra truy cập độc lập qua HTTP trả về đúng giao diện của từng ứng dụng.

---

## 2. Kiến trúc Port-based Virtual Hosting trong Nginx

```
                           Incoming HTTP Traffic
                                     │
           ┌─────────────────────────┴─────────────────────────┐
           │                                                   │
  Port 8080 (Beta Traffic)                            Port 8090 (Internal Traffic)
           │                                                   │
           ▼                                                   ▼
┌───────────────────────────────┐                   ┌───────────────────────────────┐
│ Server Block 1 (listen 8080)  │                   │ Server Block 2 (listen 8090)  │
│ Root: /var/www/beta-app/html  │                   │ Root: /var/www/internal-app/  │
│ Log: beta-app.access.log      │                   │ Log: internal-app.access.log  │
└──────────────┬────────────────┘                   └───────────────┬───────────────┘
               │                                                    │
               ▼                                                    ▼
    [Beta App: index.html]                              [Internal App: index.html]
```

---

## 3. Quy trình triển khai chi tiết

### Bước 1: Chuẩn bị thư mục và mã nguồn cho 2 ứng dụng

```bash
# 1. Tạo các thư mục web root độc lập
sudo mkdir -p /var/www/beta-app/html
sudo mkdir -p /var/www/internal-app/html

# 2. Phân quyền sở hữu cho user hiện tại
sudo chown -R $USER:$USER /var/www/beta-app/html
sudo chown -R $USER:$USER /var/www/internal-app/html
sudo chmod -R 755 /var/www/beta-app /var/www/internal-app

# 3. Tạo file index.html cho Beta App
cat << 'EOF' > /var/www/beta-app/html/index.html
<!DOCTYPE html>
<html lang="vi">
<head><title>Beta App</title></head>
<body>
    <h1>Trang Kiểm thử Ứng dụng (Beta App)</h1>
    <p>Lắng nghe trên Cổng: 8080 | Sinh viên: Phạm Thanh Phong (N24DTCN120)</p>
</body>
</html>
EOF

# 4. Tạo file index.html cho Internal App
cat << 'EOF' > /var/www/internal-app/html/index.html
<!DOCTYPE html>
<html lang="vi">
<head><title>Internal App</title></head>
<body>
    <h1>Trang Thông tin Nội bộ (Internal App)</h1>
    <p>Lắng nghe trên Cổng: 8090 | Sinh viên: Phạm Thanh Phong (N24DTCN120)</p>
</body>
</html>
EOF
```

---

### Bước 2: Tạo tệp cấu hình Nginx Multi-Port Virtual Host

Tạo file cấu hình tại `/etc/nginx/sites-available/multi-port.conf`:

```nginx
# ==========================================
# 1. Virtual Host 1: Beta App (Port 8080)
# ==========================================
server {
    listen 8080;
    listen [::]:8080;

    server_name _;

    root /var/www/beta-app/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/beta-app.access.log;
    error_log /var/log/nginx/beta-app.error.log;
}

# ==========================================
# 2. Virtual Host 2: Internal App (Port 8090)
# ==========================================
server {
    listen 8090;
    listen [::]:8090;

    server_name _;

    root /var/www/internal-app/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/internal-app.access.log;
    error_log /var/log/nginx/internal-app.error.log;
}
```

---

### Bước 3: Kích hoạt Virtual Hosts & Tải lại Nginx

```bash
# 1. Kích hoạt Server Block bằng symbolic link
sudo ln -sf /etc/nginx/sites-available/multi-port.conf /etc/nginx/sites-enabled/

# 2. Kiểm tra tính hợp lệ của cú pháp
sudo nginx -t

# 3. Tải lại cấu hình Nginx
sudo systemctl reload nginx
```

---

### Bước 4: Mở cổng trên Tường lửa UFW & Cloud Firewall

```bash
# 1. Mở cổng 8080 và 8090 trên UFW
sudo ufw allow 8080/tcp comment 'Beta App Web Port'
sudo ufw allow 8090/tcp comment 'Internal App Web Port'

# 2. Kiểm tra trạng thái UFW
sudo ufw status verbose
```

*Đồng thời trên DigitalOcean Console:* Thêm Inbound Rules trong Cloud Firewall cho `Custom TCP Port 8080` và `Custom TCP Port 8090`.

---

## 4. Nhật ký kiểm tra thực tế (Verification Logs)

### 4.1. Kiểm tra trạng thái lắng nghe cổng trên máy chủ Droplet
```bash
devops@it209-ubuntu-sgp-dev01:~$ sudo ss -tulpn | grep nginx
tcp   LISTEN 0      511          0.0.0.0:8080      0.0.0.0:*    users:(("nginx",pid=2345,fd=6))
tcp   LISTEN 0      511          0.0.0.0:8090      0.0.0.0:*    users:(("nginx",pid=2345,fd=7))
tcp   LISTEN 0      511          0.0.0.0:80        0.0.0.0:*    users:(("nginx",pid=2345,fd=8))
```

### 4.2. Kiểm tra truy cập qua cổng 8080 (Beta App)
```bash
PS C:\Users\ACER> curl -i http://159.223.45.188:8080
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Mon, 05 Oct 2026 09:03:00 GMT
Content-Type: text/html
Content-Length: 420
Connection: keep-alive

<!DOCTYPE html>
<html lang="vi">
...
<h1>Trang Kiểm thử Ứng dụng (Beta App)</h1>
<p>Lắng nghe trên Cổng: 8080 | Sinh viên: Phạm Thanh Phong (N24DTCN120)</p>
...
```

### 4.3. Kiểm tra truy cập qua cổng 8090 (Internal App)
```bash
PS C:\Users\ACER> curl -i http://159.223.45.188:8090
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Mon, 05 Oct 2026 09:03:15 GMT
Content-Type: text/html
Content-Length: 432
Connection: keep-alive

<!DOCTYPE html>
<html lang="vi">
...
<h1>Trang Thông tin Nội bộ (Internal App)</h1>
<p>Lắng nghe trên Cổng: 8090 | Sinh viên: Phạm Thanh Phong (N24DTCN120)</p>
...
```

---

## 5. Tổng kết & So sánh các mô hình Virtual Hosting

| Tiêu chí | Port-based Virtual Hosting | Name-based Virtual Hosting | IP-based Virtual Hosting |
| :--- | :--- | :--- | :--- |
| **Phân biệt bằng** | Cổng mạng (Port: 8080, 8090) | Tên miền (Domain: `beta.ptit.edu.vn`) | Địa chỉ IP riêng biệt |
| **Yêu cầu Domain** | Không cần domain (Truy cập trực tiếp qua IP:Port) | Bắt buộc có DNS Domain | Bắt buộc gán thêm IP cho máy chủ |
| **Ứng dụng thực tế** | Môi trường Dev/Test/Staging, dịch vụ nội bộ | Môi trường Production công khai | Dịch vụ đặc thù yêu cầu dedicated IP |
| **Chi phí hạ tầng** | Tối ưu, tận dụng tối đa 1 máy chủ rẻ tiền | Tối ưu, chạy nhiều site trên 1 IP | Tốn thêm chi phí thuê IP |
