# Báo Cáo Thực Hành: Cấu Hình Custom Error Page 404 Với Chỉ Thị `internal` Trên Nginx

- **Môn học:** DevOps Fundamentals (IT-209)
- **Bài tập:** Session 03 - Exercise 2: Custom 404 Error Page & Internal Directive
- **Thư mục nộp bài:** `homework/session_03/ex2/`

---

## 1. Mục Tiêu Bài Tập

1. **Giao diện thân thiện (User Experience):** Thiết kế trang báo lỗi `404.html` tùy biến, phong cách hiện đại (Dark space theme, Glassmorphism, hiệu ứng sao động, vector phi hành gia) thay thế trang báo lỗi mặc định trắng đen đơn điệu của Nginx.
2. **Cấu hình Server Block:** Áp dụng chỉ thị `error_page 404 /404.html;` để tự động điều hướng người dùng tới trang thông báo lỗi khi truy cập sai đường dẫn tĩnh.
3. **Bảo mật với chỉ thị `internal`:** Sử dụng khối `location = /404.html { internal; }` để ngăn người dùng internet truy cập trực tiếp bằng URL `http://<IP_ADDRESS_DROPLET>/404.html`.

---

## 2. Cấu Trúc Thư Mục Nộp Bài

```text
homework/session_03/ex2/
├── 404.html       # Mã nguồn giao diện trang lỗi 404 tùy chỉnh
├── my-web.conf    # Tệp cấu hình Server Block Nginx
└── README.md      # Hướng dẫn triển khai, giải thích cơ chế và kết quả kiểm thử
```

---

## 3. Giải Thích Cơ Chế Hoạt Động Của Nginx

### 3.1. Chỉ thị `error_page`
- **Cú pháp:** `error_page 404 /404.html;`
- **Vai trò:** Khi Nginx gặp mã lỗi HTTP 404 (ví dụ `try_files` không tìm thấy file hoặc URL không khớp location nào), Nginx sẽ thực hiện một **chuyển hướng nội bộ (internal redirect)** đến đường dẫn URI `/404.html`.

### 3.2. Chỉ thị `internal`
- **Cú pháp:**
  ```nginx
  location = /404.html {
      root /var/www/my-web/html;
      internal;
  }
  ```
- **Vai trò:** Chỉ định rằng location này **chỉ được phép xử lý các request nội bộ** do chính Nginx sinh ra (như từ `error_page`, `rewrite`, hay `proxy_pass` nội bộ).
- **Hành vi bảo mật:**
  - Nếu người dùng từ trình duyệt hoặc lệnh `curl` gửi request trực tiếp `GET /404.html`, Nginx sẽ phát hiện đây là request từ bên ngoài (client request) và **từ chối phục vụ**, ngay lập tức phản hồi mã trạng thái `404 Not Found`.
  - Nhờ vậy, người dùng không thể "đoán" hoặc xem trực tiếp file nguồn của trang lỗi như một trang tĩnh thông thường.

### 3.3. Sơ đồ Luồng Xử Lý (Request Flow Comparison)

```mermaid
sequenceDiagram
    autonumber
    actor Client as Người Dùng (Client / curl)
    participant Nginx as Web Server Nginx
    participant Disk as Web Root (/var/www/my-web/html)

    Note over Client,Disk: KỊCH BẢN 1: Truy cập đường dẫn không tồn tại (/invalid-path-demo)
    Client->>Nginx: GET /invalid-path-demo
    Nginx->>Disk: Kiểm tra file tồn tại qua try_files
    Disk-->>Nginx: Không tìm thấy (Triggers 404)
    Nginx->>Nginx: error_page 404 kích hoạt -> Internal redirect sang /404.html
    Nginx->>Disk: Đọc nội dung /var/www/my-web/html/404.html (cho phép qua internal)
    Nginx-->>Client: HTTP 404 Not Found + Nội dung trang HTML 404 tùy biến

    Note over Client,Disk: KỊCH BẢN 2: Truy cập trực tiếp tệp tin lỗi (/404.html)
    Client->>Nginx: GET /404.html (Direct Request)
    Nginx->>Nginx: Khớp location = /404.html có chỉ thị "internal;"
    Nginx->>Nginx: Request đến từ bên ngoài -> BỊ TỪ CHỐI
    Nginx-->>Client: HTTP 404 Not Found (Chặn xem trực tiếp)
```

---

## 4. Các Bước Triển Khai Trên Droplet / Server

### Bước 1: Kết nối SSH vào Droplet
```bash
ssh root@<IP_ADDRESS_DROPLET>
```

### Bước 2: Tạo thư mục Web Root và phân quyền
```bash
# Tạo thư mục web root chuẩn cho website
sudo mkdir -p /var/www/my-web/html

# Phân quyền sở hữu cho user hiện tại và quyền đọc cho Nginx
sudo chown -R $USER:$USER /var/www/my-web/html
sudo chmod -R 755 /var/www/my-web
```

### Bước 3: Tạo trang chủ mẫu `index.html` (Nếu chưa có)
```bash
cat << 'EOF' > /var/www/my-web/html/index.html
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <title>Trang Chủ - My Web Server</title>
</head>
<body style="font-family: sans-serif; text-align: center; padding: 50px;">
    <h1>Chào mừng đến với Web Server Nginx!</h1>
    <p>Trang web đang hoạt động bình thường trên Droplet.</p>
</body>
</html>
EOF
```

### Bước 4: Tải tệp `404.html` tùy biến vào Web Root
Tạo hoặc dán nội dung tệp `404.html` vào thư mục `/var/www/my-web/html/`:
```bash
sudo nano /var/www/my-web/html/404.html
# (Dán toàn bộ mã HTML trong tệp 404.html vào đây, nhấn Ctrl+O -> Enter -> Ctrl+X để lưu)
```

Kiểm tra tệp tin đã nằm đúng vị trí:
```bash
ls -la /var/www/my-web/html/404.html
```

### Bước 5: Cấu hình Nginx Server Block
Tạo tệp cấu hình mới trong `sites-available`:
```bash
sudo nano /etc/nginx/sites-available/my-web
```

Dán toàn bộ nội dung cấu hình sau vào:
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

    # Chỉ thị kích hoạt trang lỗi 404 tùy biến
    error_page 404 /404.html;

    # Bảo vệ tệp tin lỗi bằng chỉ thị internal
    location = /404.html {
        root     /var/www/my-web/html;
        internal;
    }
}
```

### Bước 6: Kích hoạt Server Block (Tạo Symlink)
```bash
# Tạo liên kết tượng trưng (symlink) sang thư mục sites-enabled
sudo ln -sf /etc/nginx/sites-available/my-web /etc/nginx/sites-enabled/my-web

# Nếu có cấu hình default gây xung đột port 80, tắt default:
sudo rm -f /etc/nginx/sites-enabled/default
```

### Bước 7: Kiểm tra cú pháp và Reload Nginx
```bash
# Kiểm tra cú pháp cấu hình Nginx
sudo nginx -t
```
*Kết quả hợp lệ:*
```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Sau khi cú pháp chuẩn, reload dịch vụ Nginx để áp dụng ngay lập tức:
```bash
sudo systemctl reload nginx
```

---

## 5. Kiểm Tra & Đánh Giá Kết Quả Thực Nghiệm

### Test 1: Truy cập một đường dẫn không tồn tại
Chạy lệnh kiểm tra header:
```bash
curl -I http://<IP_ADDRESS_DROPLET>/invalid-path-demo
```
**Kết quả mong đợi:**
```http
HTTP/1.1 404 Not Found
Server: nginx/...
Date: ...
Content-Type: text/html
Content-Length: ...
Connection: keep-alive
```

Kiểm tra nội dung HTML trả về có phải là trang tùy biến:
```bash
curl -s http://<IP_ADDRESS_DROPLET>/invalid-path-demo | grep -i "Trang Không Tồn Tại"
```
-> Kết quả trả về dòng tiêu đề trong tệp `404.html` tùy biến.

---

### Test 2: Truy cập trực tiếp tệp tin lỗi `/404.html`
Chạy lệnh kiểm tra:
```bash
curl -I http://<IP_ADDRESS_DROPLET>/404.html
```
**Kết quả mong đợi:**
```http
HTTP/1.1 404 Not Found
Server: nginx/...
Date: ...
Content-Type: text/html
Connection: keep-alive
```
-> **Nhận xét:** Nginx chặn truy cập trực tiếp vào `/404.html` do cơ chế `internal;`, phản hồi ngay mã trạng thái `404 Not Found`.

---

## 6. Checklist Hoàn Thành Bài Tập

- [x] Tạo tệp `404.html` giao diện hiện đại, chuẩn UI/UX, hỗ trợ tiếng Việt UTF-8.
- [x] Đặt tệp `404.html` tại thư mục web root `/var/www/my-web/html/`.
- [x] Cấu hình chỉ thị `error_page 404 /404.html;` trong Server Block.
- [x] Cấu hình khối `location = /404.html` với `root /var/www/my-web/html;` và chỉ thị `internal;`.
- [x] Chạy kiểm tra cú pháp `sudo nginx -t` thành công.
- [x] Reload Nginx bằng `sudo systemctl reload nginx`.
- [x] Kiểm thử `curl -I` truy cập đường dẫn sai -> HTTP 404 + nội dung trang tùy chỉnh.
- [x] Kiểm thử `curl -I` truy cập trực tiếp `/404.html` -> HTTP 404 (bị chặn bởi `internal`).
- [x] Đẩy đủ tệp `404.html`, `my-web.conf` và `README.md` lên GitHub tại `homework/session_03/ex2/`.
