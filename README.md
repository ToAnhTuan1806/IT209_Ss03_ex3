# Bài 3: Bảo mật tài nguyên bằng HTTP Basic Authentication

Báo cáo thiết lập bảo mật thư mục nội bộ `/admin` trên Web Server Nginx bằng HTTP Basic Authentication với tài khoản `admin_user` và file mật khẩu băm `.htpasswd` lưu an toàn tại `/etc/nginx/`.

---

## 1. Các Bước Thực Hiện

### Bước 1: Cài đặt công cụ apache2-utils
Cài đặt gói công cụ để lấy tiện ích sinh mật khẩu băm `htpasswd`:
```bash
sudo apt update
sudo apt install apache2-utils -y
```

---

### Bước 2: Tạo tệp mật khẩu ẩn `.htpasswd`
Tạo file `.htpasswd` tại thư mục cấu hình `/etc/nginx/` (không để trong `/var/www/` để tránh lộ file qua web) và tạo tài khoản `admin_user`:

```bash
sudo htpasswd -c /etc/nginx/.htpasswd admin_user
```
*(Hệ thống sẽ yêu cầu nhập và xác nhận lại mật khẩu).*

Kiểm tra nội dung tệp mật khẩu đã được băm:
```bash
sudo cat /etc/nginx/.htpasswd
```

**Log Terminal:**
```text
devops@ubuntu:~$ sudo htpasswd -c /etc/nginx/.htpasswd admin_user
New password: 
Re-type new password: 
Adding password for user admin_user

devops@ubuntu:~$ sudo cat /etc/nginx/.htpasswd
admin_user:$apr1$t7K2qM4P$v1Nq8S8yX3pT0R4uV5wY6/
```

Phân quyền an toàn cho tệp `.htpasswd`:
```bash
sudo chown www-data:www-data /etc/nginx/.htpasswd
sudo chmod 640 /etc/nginx/.htpasswd
```

---

### Bước 3: Cấu hình Server Block trong Nginx
Mở file cấu hình virtual host:
```bash
sudo nano /etc/nginx/sites-available/my-web
```

Thêm block `location /admin` yêu cầu xác thực:
```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;

    root /var/www/my-web/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    # Bảo mật khu vực /admin bằng HTTP Basic Auth
    location /admin {
        auth_basic "Restricted Admin Area";
        auth_basic_user_file /etc/nginx/.htpasswd;
        try_files $uri $uri/ =404;
    }
}
```

---

### Bước 4: Tạo tài nguyên cho thư mục `/admin`
Tạo thư mục `admin` và trang `index.html` mẫu để phục vụ khi xác thực thành công:
```bash
sudo mkdir -p /var/www/my-web/html/admin
echo "<h1>Admin Dashboard - Login Successful!</h1>" | sudo tee /var/www/my-web/html/admin/index.html
```

---

### Bước 5: Kiểm tra cú pháp và nạp lại Nginx
```bash
sudo nginx -t
sudo systemctl reload nginx
```

**Log Terminal:**
```text
devops@ubuntu:~$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

devops@ubuntu:~$ sudo systemctl reload nginx
```

---

## 2. Kiểm Tra Kết Quả

### Kiểm tra 1: Truy cập thông thường không có tài khoản (Chặn truy cập)
Chạy lệnh kiểm tra gửi request trực tiếp không có xác thực:
```bash
curl -I http://<IP_ADDRESS_DROPLET>/admin
```

**Log Terminal:**
```text
user@laptop:~$ curl -I http://159.223.45.67/admin
HTTP/1.1 401 Unauthorized
Server: nginx/1.18.0 (Ubuntu)
Date: Thu, 01 Oct 2026 15:50:10 GMT
Content-Type: text/html
Content-Length: 188
Connection: keep-alive
WWW-Authenticate: Basic realm="Restricted Admin Area"
```
👉 **Kết quả:** Trả về mã trạng thái **`HTTP 401 Unauthorized`** kèm tiêu đề `WWW-Authenticate: Basic realm="Restricted Admin Area"`.

---

### Kiểm tra 2: Truy cập với tài khoản và mật khẩu hợp lệ
Chạy lệnh kiểm tra gửi kèm cờ `-u admin_user:<PASSWORD>`:
```bash
curl -u admin_user:AdminSecure@2026 -I http://<IP_ADDRESS_DROPLET>/admin/
```

**Log Terminal:**
```text
user@laptop:~$ curl -u admin_user:AdminSecure@2026 -I http://159.223.45.67/admin/
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Thu, 01 Oct 2026 15:50:25 GMT
Content-Type: text/html
Content-Length: 43
Connection: keep-alive
```
👉 **Kết quả:** Trả về mã trạng thái **`HTTP 200 OK`**, xác thực thành công và cho phép truy cập tài nguyên bên trong thư mục `/admin`.
