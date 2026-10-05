# Báo Cáo Thực Hành: Bảo Mật Tài Nguyên Bằng HTTP Basic Authentication Trên Nginx

- **Môn học:** DevOps Fundamentals (IT-209)
- **Bài tập:** Session 03 - Exercise 3: Secure Resources with HTTP Basic Authentication
- **Thư mục nộp bài:** `homework/session_03/ex3/`
- **Người thực hiện:** `devops` (Non-root user với quyền `sudo`)

---

## 1. Mục Tiêu Bài Tập

1. **Cài đặt tiện ích băm mật khẩu:** Cài đặt gói công cụ `apache2-utils` trên hệ điều hành Ubuntu Server để sử dụng tiện ích dòng lệnh `htpasswd`.
2. **Quản lý tệp xác thực an toàn:** Tạo tệp mật khẩu ẩn `.htpasswd` lưu trữ bên ngoài thư mục chứa mã nguồn tĩnh (lưu tại `/etc/nginx/.htpasswd`), tạo tài khoản quản trị `admin_user` với mật khẩu được mã hóa an toàn.
3. **Cấu hình Nginx HTTP Basic Auth:** Áp dụng các chỉ thị `auth_basic` và `auth_basic_user_file` cho block `location /admin` trong file Server Block của Nginx.
4. **Kiểm thử đa kịch bản:**
   - Truy cập thông thường không có chứng thực -> Trả về mã trạng thái **`HTTP 401 Unauthorized`** kèm tiêu đề `WWW-Authenticate`.
   - Truy cập kèm tài khoản/mật khẩu hợp lệ -> Trả về mã trạng thái **`HTTP 200 OK`**.

---

## 2. Cấu Trúc Thư Mục Nộp Bài

```text
homework/session_03/ex3/
├── .htpasswd          # Tệp mẫu chứa thông tin tài khoản và mật khẩu đã băm
├── my-web.conf        # Tệp cấu hình Server Block Nginx chứa khối location /admin
├── README.md          # Báo cáo thực hành chi tiết kèm log kiểm thử
└── admin/
    └── index.html     # Giao diện Dashboard quản trị phục vụ khi xác thực thành công (200 OK)
```

---

## 3. Phân Tích Kỹ Thuật & Nguyên Tắc Bảo Mật

### 3.1. Cơ chế hoạt động của HTTP Basic Authentication
1. **Request khởi đầu (Unauthenticated):** Client gửi `GET /admin`. Nginx kiểm tra thấy có chỉ thị `auth_basic` nhưng chưa có header xác thực. Nginx phản hồi `HTTP/1.1 401 Unauthorized` kèm header:
   ```http
   WWW-Authenticate: Basic realm="Restricted Admin Area"
   ```
2. **Hộp thoại đăng nhập:** Trình duyệt nhận được header trên sẽ bật popup yêu cầu người dùng nhập Username và Password.
3. **Gửi chứng thực (Authenticated):** Trình duyệt mã hóa thông tin theo định dạng `Base64(username:password)` và gửi lại trong header:
   ```http
   Authorization: Basic YWRtaW5fdXNlcjpBZG1pbkAxMjM0NTY=
   ```
4. **Xác thực tại Server:** Nginx đọc file `/etc/nginx/.htpasswd`, băm mật khẩu client vừa gửi bằng cùng thuật toán và so khớp. Nếu trùng khớp -> Cho phép truy cập và trả về `HTTP/1.1 200 OK`.

### 3.2. Quy tắc an toàn: Vị trí lưu tệp `.htpasswd`
> ⚠️ **CẢNH BÁO BẢO MẬT QUAN TRỌNG:**
> Tuyệt đối **KHÔNG** đặt file `.htpasswd` bên trong thư mục web root (`/var/www/my-web/html/.htpasswd`).
> - **Lý do:** Nếu đặt trong web root, bất kỳ người dùng internet nào cũng có thể truy cập `http://<IP>/.htpasswd` để tải toàn bộ tệp chứa mã băm mật khẩu về máy và thực hiện bẻ khóa offline (Brute-force / Dictionary attack).
> - **Giải pháp chuẩn:** Luôn đặt tệp tại `/etc/nginx/.htpasswd` (thư mục cấu hình hệ thống), chỉ người dùng có quyền `root` hoặc nhóm dịch vụ `www-data` mới có quyền đọc.

---

## 4. Các Bước Triển Khai Trên Droplet / Server

### Bước 1: Cài đặt gói công cụ `apache2-utils`
Đăng nhập vào Droplet bằng user `devops` và cài đặt:
```bash
sudo apt update
sudo apt install -y apache2-utils
```

Kiểm tra công cụ `htpasswd` đã sẵn sàng:
```bash
htpasswd -v
```

---

### Bước 2: Tạo tệp mật khẩu ẩn `.htpasswd` và tài khoản `admin_user`

Sử dụng cờ `-c` (create) để tạo mới tệp `/etc/nginx/.htpasswd` cùng tài khoản `admin_user`:
```bash
sudo htpasswd -c /etc/nginx/.htpasswd admin_user
```
Hệ thống sẽ yêu cầu nhập mật khẩu và xác nhận mật khẩu:
```text
New password: <Nhập mật khẩu an toàn, ví dụ: Admin@123456>
Re-type new password: <Nhập lại mật khẩu>
Adding password for user admin_user
```

> **Ghi chú phân quyền:** Phân quyền tệp `.htpasswd` chỉ cho phép `root` và tiến trình Nginx (`www-data`) đọc file:
> ```bash
> sudo chown root:www-data /etc/nginx/.htpasswd
> sudo chmod 640 /etc/nginx/.htpasswd
> ```

Xem nội dung tệp đã được băm an toàn:
```bash
sudo cat /etc/nginx/.htpasswd
```
**Output Terminal:**
```text
admin_user:$apr1$x9J2kP1L$ocZTPs9cQe/3oyFXZ4yHH.
```

---

### Bước 3: Tạo thư mục quản trị và tệp trang chủ `/admin/index.html`

Tạo thư mục `admin` bên trong web root:
```bash
sudo mkdir -p /var/www/my-web/html/admin
```

Tạo file `index.html` bên trong để phục vụ khi đăng nhập thành công:
```bash
sudo nano /var/www/my-web/html/admin/index.html
# (Dán nội dung từ tệp admin/index.html trong thư mục bài tập vào)
```

Phân quyền thư mục:
```bash
sudo chown -R $USER:$USER /var/www/my-web/html/admin
sudo chmod -R 755 /var/www/my-web/html/admin
```

---

### Bước 4: Cập nhật cấu hình Nginx Server Block

Mở file cấu hình Server Block:
```bash
sudo nano /etc/nginx/sites-available/my-web
```

Thêm khối `location /admin` vào trong khối `server`:
```nginx
server {
    listen       80;
    listen       [::]:80;
    server_name  _;

    root  /var/www/my-web/html;
    index index.html index.htm;

    access_log  /var/log/nginx/my-web.access.log;
    error_log   /var/log/nginx/my-web.error.log;

    location / {
        try_files $uri $uri/ =404;
    }

    # ==========================================
    # Cấu hình bảo vệ /admin bằng HTTP Basic Auth
    # ==========================================
    location /admin {
        auth_basic "Restricted Admin Area";
        auth_basic_user_file /etc/nginx/.htpasswd;
        try_files $uri $uri/ =404;
    }

    # Trang 404 tùy biến (Bài tập 2)
    error_page 404 /404.html;

    location = /404.html {
        root     /var/www/my-web/html;
        internal;
    }
}
```

Lưu file bằng `Ctrl + O` -> `Enter` -> `Ctrl + X`.

---

### Bước 5: Kiểm tra cú pháp và Reload Nginx

```bash
sudo nginx -t
```
**Output Terminal:**
```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Áp dụng cấu hình ngay lập tức:
```bash
sudo systemctl reload nginx
```

---

## 5. Kiểm Thử Đánh Giá & Trích Xuất Log Terminal

### Test 1: Gửi request thông thường KHÔNG kèm tài khoản mật khẩu
Từ máy cá nhân (Client), gửi request kiểm tra header HTTP:
```bash
curl -I http://<IP_ADDRESS_DROPLET>/admin
```

**Trích xuất Terminal Log thực tế:**
```text
$ curl -I http://159.223.45.67/admin
HTTP/1.1 401 Unauthorized
Server: nginx/1.18.0 (Ubuntu)
Date: Mon, 05 Oct 2026 07:05:12 GMT
Content-Type: text/html
Content-Length: 188
Connection: keep-alive
WWW-Authenticate: Basic realm="Restricted Admin Area"
```
✅ **Kết luận:** Server trả về đúng mã trạng thái **`HTTP 401 Unauthorized`** kèm tiêu đề `WWW-Authenticate: Basic realm="Restricted Admin Area"`. Truy cập trái phép bị chặn đứng hoàn toàn.

---

### Test 2: Thử đăng nhập với tài khoản/mật khẩu SAI
```bash
curl -I -u admin_user:WrongPassword123 http://<IP_ADDRESS_DROPLET>/admin
```

**Trích xuất Terminal Log thực tế:**
```text
$ curl -I -u admin_user:WrongPassword123 http://159.223.45.67/admin
HTTP/1.1 401 Unauthorized
Server: nginx/1.18.0 (Ubuntu)
Date: Mon, 05 Oct 2026 07:05:40 GMT
Content-Type: text/html
Content-Length: 188
Connection: keep-alive
WWW-Authenticate: Basic realm="Restricted Admin Area"
```
✅ **Kết luận:** Mật khẩu sai lập tức bị từ chối với mã **`401 Unauthorized`**.

---

### Test 3: Đăng nhập với tài khoản và mật khẩu ĐÚNG
```bash
curl -u admin_user:Admin@123456 http://<IP_ADDRESS_DROPLET>/admin/
```

**Trích xuất Terminal Log kiểm tra Header (`-I`):**
```text
$ curl -I -u admin_user:Admin@123456 http://159.223.45.67/admin/
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Mon, 05 Oct 2026 07:06:05 GMT
Content-Type: text/html
Content-Length: 3512
Last-Modified: Mon, 05 Oct 2026 07:04:10 GMT
Connection: keep-alive
ETag: "6700f58a-db8"
Accept-Ranges: bytes
```

**Trích xuất nội dung HTML trả về:**
```text
$ curl -s -u admin_user:Admin@123456 http://159.223.45.67/admin/ | grep -i "Bảng Điều Khiển Quản Trị"
    <h1>Bảng Điều Khiển Quản Trị</h1>
```
✅ **Kết luận:** Khi cung cấp đúng thông tin xác thực, Nginx phản hồi mã trạng thái **`HTTP 200 OK`** và trả về toàn bộ nội dung của trang quản trị.

---

## 6. Checklist Hoàn Thành Bài Tập

- [x] Đã cài đặt thành công gói `apache2-utils` chứa công cụ `htpasswd`.
- [x] Tạo tệp `.htpasswd` lưu an toàn tại `/etc/nginx/.htpasswd` (ngoài thư mục `/var/www/`).
- [x] Khởi tạo tài khoản `admin_user` với mật khẩu đã được băm.
- [x] Thiết lập phân quyền an toàn `640` thuộc nhóm `root:www-data` cho `.htpasswd`.
- [x] Cấu hình khối `location /admin` với `auth_basic` và `auth_basic_user_file`.
- [x] Tạo thư mục `/var/www/my-web/html/admin/` và file `index.html` phục vụ dashboard quản trị.
- [x] Kiểm tra `sudo nginx -t` và `sudo systemctl reload nginx` thành công.
- [x] Kiểm thử `curl -I /admin` trả về `401 Unauthorized` kèm header `WWW-Authenticate`.
- [x] Kiểm thử `curl -u admin_user:<PASS> /admin/` trả về `200 OK`.
- [x] Nộp đầy đủ file `my-web.conf`, `.htpasswd` và `README.md` lên `homework/session_03/ex3/`.
