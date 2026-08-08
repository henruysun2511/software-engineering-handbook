# Chương 4: Reverse Proxy & Nginx

> **Mức độ quan trọng:** ⭐⭐⭐⭐  
> **Đối tượng:** Backend Developer (NestJS/Express), trình độ Intern → Junior  
> **Mục tiêu chương:** Hiểu Reverse Proxy là gì và tại sao cần thiết — cấu hình Nginx hoàn chỉnh cho ứng dụng NestJS bao gồm HTTPS, SSL certificate, rate limiting, load balancing và static file serving.

---

## 4.1. Reverse Proxy

### 4.1.1. Proxy là gì?

**Proxy** (hay Forward Proxy) là máy chủ trung gian đứng giữa **client** và **internet**. Client gửi request đến proxy, proxy chuyển tiếp request đến đích, nhận response và trả về cho client.

```
Client → [Forward Proxy] → Internet → Server
         (ẩn danh client)
```

Ví dụ phổ biến: VPN, proxy trường học để lọc nội dung.

### 4.1.2. Reverse Proxy là gì?

**Reverse Proxy** ngược lại — đứng phía **server**, nhận request từ client và chuyển tiếp đến một hoặc nhiều server backend. Client không biết server backend là gì, chỉ thấy reverse proxy.

```
                          ┌──────────────────────────┐
                          │      Reverse Proxy        │
Client → Request:80/443 → │       (Nginx)             │ → NestJS API :3000
                          │                           │ → Static files
                          │  Xử lý: SSL, routing,     │ → WebSocket
                          │  rate limit, cache...     │
                          └──────────────────────────┘
```

### 4.1.3. Tại sao cần Reverse Proxy?

Trong thực tế, **không bao giờ** để ứng dụng Node.js/NestJS nhận trực tiếp request từ internet. Lý do:

| Vấn đề | Giải pháp của Reverse Proxy |
|---|---|
| Node.js không giỏi xử lý SSL/TLS | Nginx xử lý SSL, backend chỉ nhận HTTP thuần |
| Expose port 3000 ra internet trông nghiệp dư | Nginx lắng nghe port 80/443, backend ẩn sau |
| Một server chỉ có một ứng dụng trên port 80 | Nginx route nhiều domain/path đến nhiều backend |
| Node.js kém hiệu quả hơn Nginx phục vụ file tĩnh | Nginx serve static files cực nhanh |
| Thiếu rate limiting, DDoS protection | Nginx có sẵn các tính năng này |
| Khó scale horizontally | Nginx load balance nhiều instance Node.js |
| CORS phức tạp | Nginx xử lý CORS headers tập trung |

### 4.1.4. Nginx trong kiến trúc thực tế

```
Internet
    │
    │ HTTPS :443 / HTTP :80
    ▼
┌─────────────────────────────────────┐
│              Nginx                  │
│                                     │
│  • Terminate SSL                    │
│  • Rate limiting                    │
│  • Gzip compression                 │
│  • Static files                     │
│  • Access logging                   │
│  • Route theo domain/path           │
└─────────────────────────────────────┘
    │              │              │
    │ HTTP :3000   │ HTTP :3001   │ /static
    ▼              ▼              ▼
NestJS API    NestJS API      File System
(instance 1)  (instance 2)   (uploads, assets)
    │
    │
    ▼
PostgreSQL  Redis
```

---

## 4.2. Nginx

### 4.2.1. Nginx là gì?

**Nginx** (đọc là "engine-x") là phần mềm web server mã nguồn mở, nổi tiếng với hiệu năng cao và khả năng xử lý hàng nghìn kết nối đồng thời với tài nguyên tối thiểu. Ra đời năm 2004 bởi Igor Sysoev để giải quyết bài toán C10K (xử lý 10.000 kết nối đồng thời).

Nginx hiện được dùng bởi hơn 34% website trên toàn thế giới, bao gồm Netflix, GitHub, Cloudflare.

### 4.2.2. Kiến trúc event-driven của Nginx

Nginx dùng kiến trúc **event-driven, non-blocking**, khác với Apache dùng kiến trúc thread-based:

```
Apache (thread-based):          Nginx (event-driven):
  Request 1 → Thread 1            ┌─────────────────────┐
  Request 2 → Thread 2            │   Master Process    │
  Request 3 → Thread 3            │   (quản lý workers) │
  ...                             └─────────────────────┘
  1000 requests = 1000 threads         │         │
  → RAM cạn kiệt                  Worker 1   Worker 2
                                (xử lý hàng nghìn
                                 connections mỗi worker)
```

### 4.2.3. Cài đặt Nginx

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install nginx -y

# Kiểm tra version và trạng thái
nginx -v                          # nginx/1.24.0
sudo systemctl status nginx

# Cấu trúc thư mục sau khi cài
/etc/nginx/
├── nginx.conf                    # File cấu hình chính
├── sites-available/              # Cấu hình các virtual host
│   └── default
├── sites-enabled/                # Symlink đến sites-available (đang active)
│   └── default -> ../sites-available/default
├── conf.d/                       # Cấu hình bổ sung (auto-loaded)
├── snippets/                     # Đoạn cấu hình tái sử dụng
│   ├── fastcgi-php.conf
│   └── snakeoil.conf
└── modules-enabled/              # Module được bật

/var/log/nginx/
├── access.log                    # Log mọi request
└── error.log                     # Log lỗi

/var/www/html/                    # Thư mục web mặc định
```

### 4.2.4. Kiểm tra và reload cấu hình

```bash
# Luôn test cấu hình TRƯỚC khi reload — tránh nginx chết vì config sai
sudo nginx -t
# Output khi OK:
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful

# Reload cấu hình mà không restart (không ngắt kết nối đang có)
sudo nginx -s reload
sudo systemctl reload nginx    # Tương đương

# Restart hoàn toàn (ngắt tất cả kết nối)
sudo systemctl restart nginx

# Xem log realtime
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log
```

---

## 4.3. Cấu Trúc File Cấu Hình Nginx

### 4.3.1. Cú pháp cơ bản

```nginx
# Comment bắt đầu bằng #

# Directive (chỉ thị) — kết thúc bằng ;
worker_processes auto;

# Block (khối) — nhóm các directives liên quan
http {
    # Directives bên trong block
    gzip on;
    
    # Block lồng nhau
    server {
        listen 80;
        
        location / {
            proxy_pass http://localhost:3000;
        }
    }
}
```

### 4.3.2. Cấu trúc phân cấp

```
Main context
├── events { }          # Cấu hình xử lý kết nối
└── http { }            # Cấu hình HTTP
    ├── upstream { }    # Định nghĩa nhóm server backend
    ├── server { }      # Virtual host (một domain/IP)
    │   ├── location { }  # Xử lý theo URL path
    │   └── location { }
    └── server { }
```

### 4.3.3. `nginx.conf` cơ bản — giải thích từng dòng

```nginx
# ──────────────────────────────────────────
# Main context
# ──────────────────────────────────────────

# Số worker processes — auto = bằng số CPU core
worker_processes auto;

# File lưu PID của master process
pid /run/nginx.pid;

# Bao gồm module configs
include /etc/nginx/modules-enabled/*.conf;

# ──────────────────────────────────────────
# Events context — cấu hình xử lý kết nối
# ──────────────────────────────────────────
events {
    # Số kết nối tối đa mỗi worker
    # Tổng max connections = worker_processes × worker_connections
    worker_connections 1024;
    
    # Dùng epoll (Linux) — hiệu quả nhất trên Linux
    use epoll;
    
    # Accept nhiều kết nối cùng lúc
    multi_accept on;
}

# ──────────────────────────────────────────
# HTTP context
# ──────────────────────────────────────────
http {
    # MIME types — để browser biết loại file
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Format log — ghi lại thông tin request
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    # Tối ưu hiệu năng
    sendfile on;              # Dùng kernel sendfile thay vì read+write
    tcp_nopush on;            # Tối ưu TCP packet — dùng với sendfile
    tcp_nodelay on;           # Giảm latency

    # Timeout
    keepalive_timeout 65;     # Giữ kết nối TCP bao lâu (giây)
    client_header_timeout 15;
    client_body_timeout 15;
    send_timeout 15;

    # Bảo mật — ẩn version Nginx
    server_tokens off;

    # Include cấu hình virtual hosts
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

---

## 4.4. Reverse Proxy Configuration

### 4.4.1. Cấu hình cơ bản — Proxy đến NestJS

```nginx
# File: /etc/nginx/sites-available/myapp
# Tạo symlink: sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/

server {
    listen 80;
    server_name api.myapp.com;    # Domain của bạn (hoặc _ để match mọi domain)

    # Giới hạn kích thước request (quan trọng cho file upload)
    client_max_body_size 20M;

    # ─────────────────────────────────────────
    # Proxy tất cả request đến NestJS
    # ─────────────────────────────────────────
    location / {
        # Địa chỉ backend (NestJS đang chạy)
        proxy_pass http://localhost:3000;

        # HTTP version — cần 1.1 để support WebSocket và keep-alive
        proxy_http_version 1.1;

        # Headers bắt buộc phải pass
        proxy_set_header Upgrade $http_upgrade;        # WebSocket
        proxy_set_header Connection 'upgrade';         # WebSocket
        proxy_set_header Host $host;                   # Domain gốc
        proxy_set_header X-Real-IP $remote_addr;       # IP thực của client
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  # Chain of IPs
        proxy_set_header X-Forwarded-Proto $scheme;    # http hoặc https

        # Bỏ qua cache khi có WebSocket upgrade
        proxy_cache_bypass $http_upgrade;

        # Timeout cho proxy
        proxy_connect_timeout 10s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

**Tại sao cần các headers này trong NestJS?**

```typescript
// src/main.ts — NestJS cần trust proxy để lấy IP thật của client
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Trust Nginx reverse proxy
  // Không có dòng này, req.ip sẽ trả về IP của Nginx (127.0.0.1)
  // thay vì IP thật của client
  app.set('trust proxy', 1);

  // Nếu Nginx xử lý SSL, NestJS vẫn biết request là HTTPS
  // nhờ header X-Forwarded-Proto

  await app.listen(3000, '127.0.0.1');  // Chỉ lắng nghe localhost, không ra ngoài
}
bootstrap();
```

### 4.4.2. Upstream — Load Balancing nhiều instance

```nginx
# File: /etc/nginx/sites-available/myapp

# Định nghĩa nhóm backend servers
upstream nestjs_cluster {
    # Thuật toán mặc định: round-robin (luân phiên)
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;

    # Các thuật toán khác:
    # least_conn;     → Gửi đến server ít kết nối nhất
    # ip_hash;        → Client cùng IP luôn đến cùng server (session sticky)

    # Cấu hình từng server
    # server 127.0.0.1:3000 weight=3;   # Server này nhận 3x traffic
    # server 127.0.0.1:3001 backup;     # Chỉ dùng khi servers khác down
    # server 127.0.0.1:3002 max_fails=3 fail_timeout=30s;  # Health check
    
    # Giữ kết nối đến backend (tối ưu hiệu năng)
    keepalive 32;
}

server {
    listen 80;
    server_name api.myapp.com;

    location /api/ {
        proxy_pass http://nestjs_cluster;   # Trỏ đến upstream group
        proxy_http_version 1.1;
        proxy_set_header Connection "";     # Cần cho keepalive với upstream
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 4.4.3. Routing theo Path và Domain

```nginx
server {
    listen 80;
    server_name myapp.com;

    # ─────────────────────────────────────────
    # Route theo Path
    # ─────────────────────────────────────────

    # /api/* → NestJS backend
    location /api/ {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # /health → Health check (không log để tránh noise)
    location /health {
        proxy_pass http://localhost:3000;
        access_log off;              # Không log health check requests
    }

    # /ws → WebSocket
    location /ws {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 3600s;    # WebSocket cần timeout dài
        proxy_send_timeout 3600s;
    }

    # / → Frontend (React/Next.js) hoặc static files
    location / {
        root /var/www/myapp/frontend;
        index index.html;
        # Single Page Application — fallback về index.html
        try_files $uri $uri/ /index.html;
    }
}

# ─────────────────────────────────────────
# Server block riêng cho subdomain
# ─────────────────────────────────────────
server {
    listen 80;
    server_name admin.myapp.com;      # Subdomain riêng cho admin

    location / {
        proxy_pass http://localhost:3001;   # Admin chạy ở port khác
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 4.4.4. Các biến (Variables) quan trọng trong Nginx

| Biến | Giá trị |
|---|---|
| `$host` | Domain trong request (vd: `api.myapp.com`) |
| `$remote_addr` | IP của client (hoặc IP proxy gần nhất) |
| `$http_upgrade` | Giá trị header Upgrade (dùng cho WebSocket) |
| `$scheme` | `http` hoặc `https` |
| `$request_uri` | Full URI kèm query string (`/api/users?page=1`) |
| `$uri` | URI không có query string (`/api/users`) |
| `$args` | Query string (`page=1&limit=10`) |
| `$status` | HTTP status code của response |
| `$body_bytes_sent` | Kích thước response (bytes) |
| `$request_time` | Thời gian xử lý request (giây) |

---

## 4.5. Static File Serving

### 4.5.1. Nginx vs Node.js phục vụ file tĩnh

```
Benchmark: Phục vụ file ảnh 100KB, 1000 req/s

Node.js (Express static):   ~2,000 req/s, CPU 30%
Nginx:                     ~50,000 req/s, CPU 5%
```

Nginx phục vụ file tĩnh nhanh hơn Node.js khoảng **25 lần**. Luôn để Nginx xử lý file tĩnh.

### 4.5.2. Cấu hình phục vụ file tĩnh

```nginx
server {
    listen 80;
    server_name myapp.com;

    # ─────────────────────────────────────────
    # File tĩnh — CSS, JS, Images
    # ─────────────────────────────────────────
    location /static/ {
        alias /var/www/myapp/static/;    # Thư mục chứa files
        
        # Cache trong browser — 1 năm cho file có hash trong tên
        # (vd: main.abc123.js, style.def456.css)
        expires 1y;
        add_header Cache-Control "public, immutable";
        
        # Tắt access log cho static files (giảm I/O)
        access_log off;
        
        # Bật gzip cho text files
        gzip on;
        gzip_types text/css application/javascript application/json image/svg+xml;
    }

    # ─────────────────────────────────────────
    # File upload của user
    # ─────────────────────────────────────────
    location /uploads/ {
        alias /var/www/myapp/uploads/;
        
        # Cache trung bình — file có thể thay đổi
        expires 7d;
        add_header Cache-Control "public";
        
        # Bảo mật — không cho execute file upload
        # (tránh user upload file .php rồi execute)
        location ~* \.(php|asp|aspx|cgi|pl)$ {
            deny all;
        }
    }

    # ─────────────────────────────────────────
    # Frontend SPA (React/Vue/Angular)
    # ─────────────────────────────────────────
    location / {
        root /var/www/myapp/frontend/dist;
        index index.html;
        
        # SPA routing — mọi path đều trả về index.html
        # (để React Router xử lý)
        try_files $uri $uri/ /index.html;
        
        # index.html không cache (để nhận app update ngay)
        location = /index.html {
            expires -1;
            add_header Cache-Control "no-cache, no-store, must-revalidate";
        }
        
        # Assets có hash trong tên → cache dài hạn
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
            access_log off;
        }
    }

    # API route đặt sau static để static được check trước
    location /api/ {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 4.6. HTTPS và SSL Certificate

### 4.6.1. TLS/SSL là gì?

**HTTPS** = HTTP + TLS (Transport Layer Security). TLS mã hóa dữ liệu truyền giữa client và server, đảm bảo:

- **Confidentiality**: Dữ liệu không bị đọc bởi bên thứ ba (kể cả ISP)
- **Integrity**: Dữ liệu không bị sửa đổi trên đường truyền
- **Authentication**: Client xác nhận đang kết nối đúng server (không bị MITM)

### 4.6.2. SSL Handshake — Cách HTTPS hoạt động

```
Client (Browser)                    Server (Nginx)
       │                                  │
       │─── 1. Client Hello ─────────────►│
       │    (TLS version, cipher suites)  │
       │                                  │
       │◄── 2. Server Hello ─────────────│
       │    (chọn cipher suite,           │
       │     gửi SSL Certificate)         │
       │                                  │
       │─── 3. Xác thực Certificate ─────►│
       │    (kiểm tra CA đã ký không)     │
       │                                  │
       │─── 4. Key Exchange ─────────────►│
       │    (trao đổi session key)        │
       │                                  │
       │◄── 5. Finished ─────────────────│
       │    (kết nối đã mã hóa)          │
       │                                  │
       │═══ 6. Encrypted HTTP ═══════════►│
       │    (mọi request từ đây là        │
       │     dữ liệu mã hóa)             │
```

### 4.6.3. Các loại SSL Certificate

| Loại | Xác thực | Thời gian | Giá | Dùng khi nào |
|---|---|---|---|---|
| **DV** (Domain Validation) | Chỉ domain | Phút | Miễn phí (Let's Encrypt) | Hầu hết web/API |
| **OV** (Organization Validation) | Domain + tổ chức | Ngày | Có phí | Doanh nghiệp |
| **EV** (Extended Validation) | Kiểm tra kỹ nhất | Tuần | Đắt nhất | Ngân hàng, tài chính |
| **Wildcard** | `*.domain.com` | Giống DV/OV | Có phí | Nhiều subdomain |

---

## 4.7. Let's Encrypt — SSL Certificate Miễn Phí

### 4.7.1. Let's Encrypt là gì?

**Let's Encrypt** là Certificate Authority (CA) phi lợi nhuận cung cấp SSL certificate miễn phí, tự động và đáng tin cậy. Được sử dụng bởi hơn 300 triệu website.

**Certbot** là công cụ chính thức để tự động lấy và gia hạn certificate từ Let's Encrypt.

### 4.7.2. Cài đặt Certbot

```bash
# Ubuntu/Debian — cài qua snap (khuyến nghị)
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# Hoặc cài qua apt
sudo apt install certbot python3-certbot-nginx -y
```

### 4.7.3. Lấy Certificate

```bash
# Cách 1: Certbot tự động cấu hình Nginx (khuyến nghị cho người mới)
sudo certbot --nginx -d myapp.com -d www.myapp.com

# Cách 2: Chỉ lấy certificate, cấu hình Nginx thủ công
sudo certbot certonly --nginx -d myapp.com -d api.myapp.com

# Cách 3: Dùng webroot (khi Nginx đang chạy)
sudo certbot certonly --webroot \
    -w /var/www/html \
    -d myapp.com \
    -d www.myapp.com

# Vị trí certificate sau khi lấy:
# /etc/letsencrypt/live/myapp.com/
# ├── fullchain.pem  → Certificate đầy đủ (dùng trong Nginx)
# ├── privkey.pem    → Private key (BẢO MẬT TUYỆT ĐỐI)
# ├── cert.pem       → Certificate (không bao gồm chain)
# └── chain.pem      → Intermediate certificates
```

### 4.7.4. Gia hạn Certificate tự động

Certificate của Let's Encrypt có hiệu lực **90 ngày**. Certbot tự động gia hạn qua cron:

```bash
# Kiểm tra auto-renewal có hoạt động không
sudo certbot renew --dry-run

# Xem cron job do certbot tạo
sudo cat /etc/cron.d/certbot
# hoặc
sudo systemctl list-timers | grep certbot

# Gia hạn thủ công (hiếm khi cần)
sudo certbot renew
```

---

## 4.8. Nginx Hoàn Chỉnh với HTTPS

### 4.8.1. Cấu hình production đầy đủ

```nginx
# File: /etc/nginx/sites-available/myapp
# Áp dụng cho: api.myapp.com

# ──────────────────────────────────────────────────────
# Upstream — NestJS backend cluster
# ──────────────────────────────────────────────────────
upstream nestjs_api {
    server 127.0.0.1:3000;
    # Thêm instances khi scale:
    # server 127.0.0.1:3001;
    # server 127.0.0.1:3002;
    keepalive 32;
}

# ──────────────────────────────────────────────────────
# Rate limiting zones
# ──────────────────────────────────────────────────────
# Giới hạn theo IP: 10 request/giây, lưu state 10MB
# 10MB ≈ 160,000 IP addresses
limit_req_zone $binary_remote_addr zone=api_zone:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=auth_zone:10m rate=5r/m;

# ──────────────────────────────────────────────────────
# Server block 1: Redirect HTTP → HTTPS
# ──────────────────────────────────────────────────────
server {
    listen 80;
    listen [::]:80;                    # IPv6
    server_name api.myapp.com;

    # Let's Encrypt renewal challenge
    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    # Tất cả request HTTP → redirect HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}

# ──────────────────────────────────────────────────────
# Server block 2: HTTPS — xử lý chính
# ──────────────────────────────────────────────────────
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;                          # Bật HTTP/2 (nhanh hơn HTTP/1.1)
    server_name api.myapp.com;

    # ─────────────────────────────────────
    # SSL Certificate
    # ─────────────────────────────────────
    ssl_certificate     /etc/letsencrypt/live/api.myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.myapp.com/privkey.pem;

    # Cấu hình TLS hiện đại — chỉ dùng TLS 1.2 và 1.3
    ssl_protocols TLSv1.2 TLSv1.3;

    # Cipher suites an toàn — ưu tiên server's preference
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;     # TLS 1.3 tự chọn, không ép

    # SSL Session Cache — tăng tốc kết nối lặp lại
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # OCSP Stapling — giảm thời gian verify certificate
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/api.myapp.com/chain.pem;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;

    # ─────────────────────────────────────
    # Security Headers
    # ─────────────────────────────────────
    # Ẩn version Nginx
    server_tokens off;

    # HSTS — bắt browser luôn dùng HTTPS (31536000s = 1 năm)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Ngăn clickjacking
    add_header X-Frame-Options "SAMEORIGIN" always;

    # Ngăn MIME type sniffing
    add_header X-Content-Type-Options "nosniff" always;

    # XSS Protection (cũ nhưng vẫn hữu ích)
    add_header X-XSS-Protection "1; mode=block" always;

    # Referrer Policy
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Permissions Policy — tắt tính năng browser không cần
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

    # ─────────────────────────────────────
    # Gzip Compression
    # ─────────────────────────────────────
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;             # Không nén file nhỏ hơn 1KB
    gzip_proxied expired no-cache no-store private auth;
    gzip_types
        text/plain
        text/css
        text/javascript
        application/javascript
        application/json
        application/xml
        image/svg+xml;
    gzip_disable "msie6";             # Không nén cho IE6 (bug)

    # ─────────────────────────────────────
    # Request limits
    # ─────────────────────────────────────
    client_max_body_size 20M;         # Cho phép upload file tối đa 20MB
    client_body_buffer_size 128k;

    # ─────────────────────────────────────
    # Timeouts
    # ─────────────────────────────────────
    proxy_connect_timeout 10s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;

    # ─────────────────────────────────────
    # Locations
    # ─────────────────────────────────────

    # Health check — không rate limit, không log
    location /health {
        proxy_pass http://nestjs_api;
        proxy_set_header Host $host;
        access_log off;
    }

    # Auth endpoints — rate limit nghiêm ngặt (chống brute force)
    location /api/auth/ {
        limit_req zone=auth_zone burst=10 nodelay;
        limit_req_status 429;

        proxy_pass http://nestjs_api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # API chính — rate limit bình thường
    location /api/ {
        limit_req zone=api_zone burst=20 nodelay;
        limit_req_status 429;

        proxy_pass http://nestjs_api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # CORS headers (nếu không xử lý trong NestJS)
        add_header Access-Control-Allow-Origin "https://myapp.com" always;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, PATCH, DELETE, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Authorization, Content-Type" always;
        add_header Access-Control-Max-Age 86400;

        # Handle preflight OPTIONS request
        if ($request_method = OPTIONS) {
            return 204;
        }
    }

    # WebSocket
    location /ws/ {
        proxy_pass http://nestjs_api;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }

    # Upload files — timeout dài hơn
    location /api/upload {
        client_max_body_size 100M;       # Upload lớn hơn bình thường
        proxy_pass http://nestjs_api;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 300s;         # 5 phút cho upload lớn
    }

    # Serve uploaded files trực tiếp từ Nginx
    location /uploads/ {
        alias /var/www/myapp/uploads/;
        expires 7d;
        add_header Cache-Control "public";

        # Bảo mật: không cho phép truy cập file .env, config, script
        location ~* \.(env|conf|php|sh|py)$ {
            deny all;
        }
    }

    # Chặn request đến file nhạy cảm
    location ~ /\. {
        deny all;        # Chặn .env, .git, .htaccess...
    }

    location ~* \.(git|svn|htaccess|htpasswd)$ {
        deny all;
    }

    # Custom error pages
    error_page 429 /errors/429.json;
    error_page 502 503 504 /errors/5xx.json;

    location /errors/ {
        root /var/www/myapp;
        internal;         # Chỉ dùng cho internal redirect
    }
}
```

### 4.8.2. Snippet tái sử dụng

Thay vì copy các đoạn cấu hình lặp lại, tạo snippet:

```nginx
# File: /etc/nginx/snippets/proxy-params.conf
proxy_http_version 1.1;
proxy_set_header Connection "";
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_connect_timeout 10s;
proxy_send_timeout 60s;
proxy_read_timeout 60s;
```

```nginx
# File: /etc/nginx/snippets/ssl-params.conf
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;
```

```nginx
# Dùng snippet trong site config
server {
    listen 443 ssl;
    server_name api.myapp.com;

    ssl_certificate /etc/letsencrypt/live/api.myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.myapp.com/privkey.pem;

    include snippets/ssl-params.conf;      # Include snippet

    location /api/ {
        include snippets/proxy-params.conf; # Include snippet
        proxy_pass http://nestjs_api;
    }
}
```

---

## 4.9. Nginx Trong Docker Compose

### 4.9.1. Khi dùng Docker, Nginx chạy như container

```
Internet :80/:443
       │
  [Nginx Container]   ← proxy đến
       │
  [NestJS Container]  ← qua Docker network
```

**File `nginx/nginx.conf` dùng với Docker Compose:**

```nginx
# nginx/nginx.conf — dành cho môi trường Docker

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Log format với request time
    log_format main '$remote_addr - [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    'rt=$request_time';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    sendfile on;
    keepalive_timeout 65;
    server_tokens off;
    gzip on;
    gzip_types text/plain application/json application/javascript text/css image/svg+xml;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=auth:10m rate=5r/m;

    # Upstream — dùng tên service Docker Compose
    upstream nestjs_api {
        server api:3000;     # "api" = tên service trong docker-compose.yml
        keepalive 16;
    }

    # ─────────────────────────────────────
    # HTTP → HTTPS redirect
    # ─────────────────────────────────────
    server {
        listen 80;
        server_name _;

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        location / {
            return 301 https://$host$request_uri;
        }
    }

    # ─────────────────────────────────────
    # HTTPS
    # ─────────────────────────────────────
    server {
        listen 443 ssl;
        http2 on;
        server_name api.myapp.com;

        ssl_certificate     /etc/nginx/ssl/fullchain.pem;
        ssl_certificate_key /etc/nginx/ssl/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 1d;

        client_max_body_size 20M;

        add_header Strict-Transport-Security "max-age=31536000" always;
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;

        location /health {
            proxy_pass http://nestjs_api;
            proxy_set_header Host $host;
            access_log off;
        }

        location /api/auth/ {
            limit_req zone=auth burst=10 nodelay;
            limit_req_status 429;
            proxy_pass http://nestjs_api;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /api/ {
            limit_req zone=api burst=20 nodelay;
            limit_req_status 429;
            proxy_pass http://nestjs_api;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location ~ /\. {
            deny all;
        }
    }
}
```

**Tích hợp vào `docker-compose.prod.yml`:**

```yaml
# docker-compose.prod.yml (phần nginx)
services:
  nginx:
    image: nginx:1.25-alpine
    container_name: myapp-nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      # Config file (read-only)
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      # SSL certificates từ Certbot
      - /etc/letsencrypt:/etc/nginx/ssl:ro
      # Certbot webroot challenge
      - certbot-webroot:/var/www/certbot:ro
      # Nginx logs (persistent)
      - nginx-logs:/var/log/nginx
    depends_on:
      api:
        condition: service_healthy
    networks:
      - app-network
    # Health check cho Nginx
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Certbot container để gia hạn certificate tự động
  certbot:
    image: certbot/certbot:latest
    container_name: myapp-certbot
    volumes:
      - /etc/letsencrypt:/etc/letsencrypt
      - certbot-webroot:/var/www/certbot
    # Chỉ chạy khi cần gia hạn (dùng với cron hoặc entrypoint)
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"
    profiles:
      - certbot    # Chỉ chạy khi: docker compose --profile certbot up

volumes:
  nginx-logs:
  certbot-webroot:
```

---

## 4.10. Quy Trình Thực Tế

### 4.10.1. Setup Nginx + HTTPS trên VPS mới

```bash
# ─────────────────────────────────────
# Bước 1: DNS trỏ về server trước
# ─────────────────────────────────────
# Tại nhà cung cấp domain, thêm A record:
# api.myapp.com → 203.0.113.10 (IP server)
# Chờ DNS propagate (thường 5-30 phút)

# Verify DNS đã trỏ đúng
dig api.myapp.com
curl http://api.myapp.com    # Phải về server (có thể 502 là OK)

# ─────────────────────────────────────
# Bước 2: Cài Nginx
# ─────────────────────────────────────
sudo apt update
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx

# ─────────────────────────────────────
# Bước 3: Tạo cấu hình tạm (HTTP only) để Certbot verify
# ─────────────────────────────────────
sudo tee /etc/nginx/sites-available/myapp << 'EOF'
server {
    listen 80;
    server_name api.myapp.com;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    location / {
        return 200 'Nginx OK';
        add_header Content-Type text/plain;
    }
}
EOF

sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default   # Xóa config mặc định
sudo nginx -t && sudo systemctl reload nginx

# Kiểm tra HTTP hoạt động
curl http://api.myapp.com

# ─────────────────────────────────────
# Bước 4: Lấy SSL Certificate
# ─────────────────────────────────────
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

sudo certbot --nginx \
    -d api.myapp.com \
    --non-interactive \
    --agree-tos \
    --email admin@myapp.com

# Certbot tự động sửa file nginx config để thêm SSL

# ─────────────────────────────────────
# Bước 5: Thay thế bằng cấu hình production
# ─────────────────────────────────────
sudo tee /etc/nginx/sites-available/myapp << 'NGINX_CONF'
# (Dán cấu hình production đầy đủ từ mục 4.8.1)
NGINX_CONF

sudo nginx -t && sudo systemctl reload nginx

# ─────────────────────────────────────
# Bước 6: Kiểm tra
# ─────────────────────────────────────
# Kiểm tra HTTPS hoạt động
curl https://api.myapp.com/health

# Kiểm tra HTTP redirect HTTPS
curl -I http://api.myapp.com

# Kiểm tra SSL rating tại:
# https://www.ssllabs.com/ssltest/

# ─────────────────────────────────────
# Bước 7: Mở firewall
# ─────────────────────────────────────
sudo ufw allow 'Nginx Full'    # Mở port 80 và 443
sudo ufw allow OpenSSH         # Giữ SSH
sudo ufw enable
sudo ufw status
```

### 4.10.2. Debug các vấn đề thường gặp

```bash
# Lỗi 502 Bad Gateway — NestJS không chạy hoặc sai port
sudo tail -f /var/log/nginx/error.log
curl http://localhost:3000/health   # Thử kết nối trực tiếp NestJS
ss -tlnp | grep :3000              # Kiểm tra port 3000 có mở không

# Lỗi 504 Gateway Timeout — NestJS xử lý quá chậm
# Tăng timeout trong nginx config:
# proxy_read_timeout 120s;

# Lỗi Certificate — SSL handshake failed
sudo certbot certificates           # Xem certificate còn hiệu lực không
sudo nginx -t                       # Kiểm tra path certificate đúng không

# Lỗi "413 Request Entity Too Large" — file upload quá lớn
# Tăng client_max_body_size trong nginx config

# Lỗi "429 Too Many Requests" — bị rate limit
# Tạm thời tắt rate limit để test:
# Comment dòng limit_req... trong nginx config

# Kiểm tra config đang active
sudo nginx -T                       # In toàn bộ cấu hình đã merge
sudo nginx -T | grep -A5 "server_name api.myapp.com"
```

---

## 4.11. Best Practices

### 4.11.1. Bảo mật

```nginx
# ✅ Luôn ẩn version Nginx
server_tokens off;

# ✅ Redirect HTTP → HTTPS
server {
    listen 80;
    return 301 https://$host$request_uri;
}

# ✅ HSTS — bắt buộc HTTPS
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

# ✅ Chặn access đến file ẩn
location ~ /\. {
    deny all;
}

# ✅ Rate limiting cho auth endpoints
location /api/auth/ {
    limit_req zone=auth burst=10 nodelay;
}

# ✅ Chỉ dùng TLS 1.2+ (không dùng TLS 1.0, 1.1 hay SSL)
ssl_protocols TLSv1.2 TLSv1.3;
```

### 4.11.2. Hiệu năng

```nginx
# ✅ Bật HTTP/2
listen 443 ssl;
http2 on;

# ✅ Gzip compression
gzip on;
gzip_types application/json application/javascript text/css;

# ✅ Keepalive đến upstream
upstream nestjs_api {
    server 127.0.0.1:3000;
    keepalive 32;
}

location /api/ {
    proxy_http_version 1.1;
    proxy_set_header Connection "";   # Cần cho keepalive
    proxy_pass http://nestjs_api;
}

# ✅ Cache file tĩnh
location /static/ {
    expires 1y;
    add_header Cache-Control "public, immutable";
    access_log off;
}

# ✅ Sendfile
sendfile on;
tcp_nopush on;
```

### 4.11.3. Quy trình thay đổi config an toàn

```bash
# Luôn theo 3 bước này:

# 1. Backup cấu hình hiện tại
sudo cp /etc/nginx/sites-available/myapp /etc/nginx/sites-available/myapp.bak

# 2. Sửa cấu hình
sudo vim /etc/nginx/sites-available/myapp

# 3. Test TRƯỚC khi reload
sudo nginx -t
# Nếu OK:
sudo systemctl reload nginx
# Nếu fail → restore:
# sudo cp /etc/nginx/sites-available/myapp.bak /etc/nginx/sites-available/myapp
```

---

## Tóm Tắt Chương 4

| Khái niệm | Vai trò |
|---|---|
| **Reverse Proxy** | Trung gian giữa internet và backend, xử lý SSL/routing/security |
| **Nginx** | Web server / reverse proxy hiệu năng cao |
| **`server` block** | Định nghĩa virtual host theo domain/port |
| **`location` block** | Xử lý request theo URL pattern |
| **`upstream`** | Nhóm backend servers cho load balancing |
| **SSL/TLS** | Mã hóa kết nối HTTPS |
| **Let's Encrypt** | CA miễn phí, tự động gia hạn qua Certbot |
| **Rate limiting** | Giới hạn số request/giây theo IP |
| **`proxy_pass`** | Chuyển tiếp request đến backend |
| **Security headers** | HSTS, X-Frame-Options, X-Content-Type-Options |

---

> **Chương tiếp theo:** [Chương 5 — Deployment](./chapter-05-deployment.md)  
> Bạn đã có Nginx làm cổng vào. Tiếp theo, chúng ta sẽ tổng hợp toàn bộ kiến thức để xây dựng quy trình deploy hoàn chỉnh — từ môi trường development đến production — bao gồm zero downtime deployment và rollback.
