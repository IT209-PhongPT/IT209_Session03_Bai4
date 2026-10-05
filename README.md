# Bài tập 4: Chạy song song nhiều cổng dịch vụ (Nginx Virtual Hosts)

## Thông tin sinh viên
- **Họ và tên:** Phạm Thanh Phong
- **MSSV:** N24DTCN120
- **Môn học:** IT209 - Phát triển ứng dụng Web / Cloud Infrastructure
- **Session:** Session 03 - Bài tập 4

---

## Tệp tin đính kèm
- [multi-port.conf](./multi-port.conf): Cấu hình Nginx Virtual Hosts lắng nghe Port 8080 và 8090.
- [beta-app/index.html](./beta-app/index.html): Trang ứng dụng thử nghiệm (Port 8080).
- [internal-app/index.html](./internal-app/index.html): Trang cổng thông tin nội bộ (Port 8090).
- [IT209_Session03_BT4_PhamThanhPhong_N24DTCN120.md](./IT209_Session03_BT4_PhamThanhPhong_N24DTCN120.md): Báo cáo chi tiết và log kiểm tra.

---

## Hướng dẫn triển khai nhanh
```bash
# 1. Tạo thư mục web root
sudo mkdir -p /var/www/beta-app/html /var/www/internal-app/html

# 2. Copy mã nguồn
sudo cp beta-app/index.html /var/www/beta-app/html/
sudo cp internal-app/index.html /var/www/internal-app/html/

# 3. Kích hoạt Virtual Hosts
sudo cp multi-port.conf /etc/nginx/sites-available/
sudo ln -sf /etc/nginx/sites-available/multi-port.conf /etc/nginx/sites-enabled/

# 4. Mở tường lửa UFW
sudo ufw allow 8080/tcp
sudo ufw allow 8090/tcp

# 5. Kiểm tra & tải lại Nginx
sudo nginx -t
sudo systemctl reload nginx
```
Kiểm tra:
- `curl http://<IP>:8080` (Beta App)
- `curl http://<IP>:8090` (Internal App)
