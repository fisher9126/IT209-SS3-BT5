# Bài 5 (Session 03): Thiết lập thư mục Web an toàn tại phân vùng /opt/

## Mục tiêu
- Cô lập mã nguồn web và logs ra khỏi đường dẫn mặc định, đặt tại `/opt/my-app/`.
- Phân quyền phân tách vai trò giữa user quản trị `devops` và tiến trình Nginx `www-data`.

## Môi trường
- VPS: Ubuntu 22.04 (host `fisher-9126`)
- Web server: Nginx 1.18.0
- Cổng phục vụ: `8070`

## Các bước thực hiện

### 1. Tạo cây thư mục trong /opt
```bash
sudo mkdir -p /opt/my-app/html
sudo mkdir -p /opt/my-app/logs
```

### 2. Tạo file index.html
```bash
sudo nano /opt/my-app/html/index.html   # dán nội dung trang web
```

### 3. Phân quyền sở hữu toàn bộ /opt/my-app cho devops:www-data
```bash
sudo chown -R devops:www-data /opt/my-app
```

### 4. Phân quyền chi tiết

**Thư mục web tĩnh — devops ghi, www-data đọc:**
```bash
sudo find /opt/my-app/html -type d -exec chmod 750 {} \;
sudo find /opt/my-app/html -type f -exec chmod 640 {} \;
```

**Thư mục logs — www-data cần quyền GHI để tạo file log:**
```bash
sudo chown -R www-data:www-data /opt/my-app/logs
sudo chmod 775 /opt/my-app/logs
```

### 5. Cấu hình Server Block (file `/etc/nginx/sites-available/my-app.conf`)
```nginx
server {
    listen 8070;
    server_name _;
    root /opt/my-app/html;
    index index.html;

    access_log /opt/my-app/logs/access.log;
    error_log  /opt/my-app/logs/error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### 6. Kích hoạt + mở cổng + reload
```bash
sudo ln -s /etc/nginx/sites-available/my-app.conf /etc/nginx/sites-enabled/
sudo ufw allow 8070/tcp
sudo nginx -t
sudo systemctl reload nginx
```

## Kết quả kiểm tra

### Phân quyền thư mục
```
$ ls -la /opt/my-app/
drwxr-x--- devops   www-data  html
drwxrwxr-x www-data www-data  logs
```
- `html/` thuộc `devops:www-data` (750) — devops ghi, www-data đọc.
- `logs/` thuộc `www-data:www-data` (775) — Nginx ghi log được.

### Ghi file bằng user devops (không sudo)
```
$ su - devops
devops@fisher-9126:~$ echo "Update Test" >> /opt/my-app/html/index.html
devops@fisher-9126:~$ exit
```
→ Ghi thành công, không cần quyền root.

### Log tự động phát sinh sau khi truy cập
```
$ curl http://160.187.229.65:8070
$ tail -n 5 /opt/my-app/logs/access.log
160.187.229.65 - - [02/Oct/2026:...] "GET / HTTP/1.1" 200 ...
```
→ Nginx ghi access.log vào đúng đường dẫn mới trong /opt.

## Giải thích phân quyền

| Thư mục | Owner:Group | Quyền | Lý do |
|---------|-------------|-------|-------|
| `/opt/my-app/html` | devops:www-data | 750 (dir) / 640 (file) | devops sửa code, www-data chỉ đọc để phục vụ web |
| `/opt/my-app/logs` | www-data:www-data | 775 | Nginx (www-data) phải GHI được file log |

Điểm mấu chốt: thư mục web chỉ cho www-data **đọc**, nhưng thư mục logs phải cho www-data **ghi** — vì tiến trình Nginx cần tạo và ghi liên tục vào access.log/error.log.

## Kết luận
Mã nguồn và logs được cô lập hoàn toàn trong `/opt/my-app/`, phân quyền phân tách rõ vai trò devops (ghi code) và www-data (đọc web, ghi log). Nginx hoạt động không lỗi 403 và ghi log đúng vị trí chỉ định.
