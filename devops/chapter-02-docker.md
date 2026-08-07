# Chương 2: Docker

> **Mức độ quan trọng:** ⭐⭐⭐⭐⭐  
> **Đối tượng:** Backend Developer (NestJS/Express), trình độ Intern → Junior  
> **Mục tiêu chương:** Hiểu Docker từ nguyên lý đến thực hành — Dockerize ứng dụng NestJS hoàn chỉnh với PostgreSQL và Redis, sử dụng Docker Compose để quản lý toàn bộ stack.

---

## 2.1. Docker Là Gì?

### 2.1.1. Vấn đề trước khi có Docker

Trước khi Docker ra đời, một câu nói rất phổ biến trong giới lập trình là:

> *"It works on my machine."*  
> *(Trên máy tôi chạy được mà.)*

Tình huống điển hình:

```
Developer A viết code trên macOS
  → Node.js 18, PostgreSQL 14, Redis 6

Developer B clone code về Ubuntu
  → Node.js 20, PostgreSQL 15, Redis 7
  → Chạy bị lỗi dependency

Lên staging server (CentOS)
  → Lại một bộ phiên bản khác
  → Lại bị lỗi

Lên production (Ubuntu 20.04)
  → Cấu hình khác, path khác
  → Lại lỗi
```

**Nguyên nhân gốc rễ:** Ứng dụng phụ thuộc vào môi trường — OS, phiên bản runtime, thư viện hệ thống, biến môi trường — và môi trường thì khác nhau trên mỗi máy.

### 2.1.2. Docker là gì?

**Docker** là nền tảng mã nguồn mở cho phép đóng gói ứng dụng cùng toàn bộ môi trường cần thiết (runtime, thư viện, cấu hình) vào một đơn vị độc lập gọi là **container**.

Container hoạt động nhất quán trên bất kỳ máy nào có cài Docker — bất kể đó là laptop của developer hay server production.

```
Không có Docker:                 Có Docker:
┌─────────────────┐             ┌─────────────────────────────┐
│   Your App      │             │  Container                  │
│   + Node 18     │             │  ┌────────────────────────┐ │
│   + npm deps    │             │  │  Your App              │ │
├─────────────────┤             │  │  + Node 18             │ │
│  OS Ubuntu 20   │    ≠  OS    │  │  + npm deps            │ │
│  Libs version X │             │  │  + Libs version X      │ │
│  PATH settings  │             │  └────────────────────────┘ │
└─────────────────┘             ├─────────────────────────────┤
                                │  Docker Engine              │
                                ├─────────────────────────────┤
                                │  Any OS (Linux/Mac/Windows) │
                                └─────────────────────────────┘
```

### 2.1.3. Docker giải quyết những vấn đề gì?

| Vấn đề | Giải pháp của Docker |
|---|---|
| "Works on my machine" | Container chứa toàn bộ môi trường, chạy giống nhau ở mọi nơi |
| Xung đột phiên bản | Mỗi container độc lập, tự có runtime riêng |
| Cài đặt phức tạp | `docker run` — một lệnh là chạy được |
| Khó scale | Nhân bản container trong vài giây |
| Triển khai chậm | Build một lần, deploy mọi nơi |

---

## 2.2. Container vs Virtual Machine

### 2.2.1. Virtual Machine (VM)

VM mô phỏng toàn bộ phần cứng và chạy một OS hoàn chỉnh bên trong:

```
┌──────────────────────────────────────────┐
│              Hypervisor                  │
│  ┌────────────┐  ┌────────────┐          │
│  │   VM 1     │  │   VM 2     │          │
│  │ ┌────────┐ │  │ ┌────────┐ │          │
│  │ │  App A │ │  │ │  App B │ │          │
│  │ ├────────┤ │  │ ├────────┤ │          │
│  │ │Guest OS│ │  │ │Guest OS│ │          │
│  │ │(Ubuntu)│ │  │ │(CentOS)│ │          │
│  └────────────┘  └────────────┘          │
├──────────────────────────────────────────┤
│           Host OS (Windows/Linux)        │
├──────────────────────────────────────────┤
│                Hardware                  │
└──────────────────────────────────────────┘
```

### 2.2.2. Container

Container chia sẻ kernel của Host OS, chỉ đóng gói những gì ứng dụng cần:

```
┌──────────────────────────────────────────┐
│             Docker Engine                │
│  ┌────────────┐  ┌────────────┐          │
│  │Container 1 │  │Container 2 │          │
│  │ ┌────────┐ │  │ ┌────────┐ │          │
│  │ │  App A │ │  │ │  App B │ │          │
│  │ ├────────┤ │  │ ├────────┤ │          │
│  │ │  Libs  │ │  │ │  Libs  │ │          │
│  └────────────┘  └────────────┘          │
├──────────────────────────────────────────┤
│              Host OS Kernel              │
├──────────────────────────────────────────┤
│                Hardware                  │
└──────────────────────────────────────────┘
```

### 2.2.3. So sánh

| Tiêu chí | Virtual Machine | Container |
|---|---|---|
| **Kích thước** | Hàng GB (cả OS) | Hàng MB (chỉ app + libs) |
| **Khởi động** | Vài phút | Vài giây (thậm chí mili-giây) |
| **Cách ly** | Hoàn toàn (riêng kernel) | Chia sẻ kernel Host OS |
| **Hiệu năng** | Thấp hơn (overhead) | Gần như native |
| **Tính di động** | Trung bình | Rất cao |
| **Use case** | Chạy nhiều OS khác nhau | Microservices, CI/CD, deployment |

> 💡 **Trong thực tế:** VM và Container không loại trừ nhau. Nhiều doanh nghiệp chạy Docker containers **bên trong** VM — VM để cách ly hạ tầng, Container để deploy ứng dụng.

---

## 2.3. Docker Image

### 2.3.1. Khái niệm

**Docker Image** là một template read-only chứa toàn bộ những gì cần thiết để chạy một ứng dụng: OS base, runtime, thư viện, code ứng dụng và các lệnh khởi động.

Image được xây dựng theo từng **layer** (lớp). Mỗi lệnh trong `Dockerfile` tạo ra một layer mới. Các layer được cache lại, giúp build nhanh hơn.

```
┌────────────────────────────┐  ← Layer 4: COPY . . (source code)
├────────────────────────────┤  ← Layer 3: RUN npm ci
├────────────────────────────┤  ← Layer 2: COPY package*.json ./
├────────────────────────────┤  ← Layer 1: FROM node:20-alpine
└────────────────────────────┘  ← Base Layer (read-only)
```

### 2.3.2. Các lệnh quản lý Image

```bash
# Tìm kiếm image trên Docker Hub
docker search nginx
docker search node

# Tải image về (pull)
docker pull node:20-alpine           # node phiên bản 20, Alpine Linux
docker pull postgres:16              # PostgreSQL 16
docker pull redis:7-alpine           # Redis 7, Alpine

# Liệt kê image đã có trên máy
docker images
docker image ls

# Xem chi tiết một image
docker image inspect node:20-alpine

# Xóa image
docker rmi node:20-alpine            # Xóa theo tên:tag
docker rmi abc123def456              # Xóa theo Image ID
docker image prune                   # Xóa tất cả image không dùng (dangling)
docker image prune -a                # Xóa tất cả image không có container đang dùng

# Xem lịch sử layers của image
docker history node:20-alpine
```

### 2.3.3. Image Naming Convention

```
[registry/][namespace/]name[:tag]
     ↑           ↑        ↑    ↑
  Registry   Namespace   Tên  Phiên bản

Ví dụ:
node:20-alpine            → Docker Hub, official image, tag 20-alpine
nginx:1.25                → Docker Hub, official image, tag 1.25
mycompany/my-api:1.0.0    → Docker Hub, namespace mycompany, tag 1.0.0
ghcr.io/user/app:latest   → GitHub Container Registry
```

**Các tag phổ biến:**

| Tag | Ý nghĩa |
|---|---|
| `latest` | Phiên bản mới nhất (mặc định nếu không chỉ định tag) |
| `20` | Phiên bản major cụ thể |
| `20.11.0` | Phiên bản exact |
| `20-alpine` | Phiên bản 20, dựa trên Alpine Linux (nhẹ hơn ~5x) |
| `20-slim` | Phiên bản minimal, bỏ bớt công cụ không cần thiết |

> ⚠️ **Best Practice:** Không dùng `latest` trong production. Luôn chỉ định phiên bản cụ thể để tránh breaking changes khi image được cập nhật.

---

## 2.4. Docker Container

### 2.4.1. Khái niệm

**Docker Container** là một **instance đang chạy** của Docker Image. Mối quan hệ giữa Image và Container giống như:

```
Image   →  Container
Class   →  Object (instance)
.iso    →  Máy ảo đang chạy
```

Khi container chạy, Docker thêm một **writable layer** lên trên các read-only layers của image. Mọi thay đổi trong container (tạo file, cài thêm gói) đều ghi vào writable layer này, không ảnh hưởng đến image gốc.

### 2.4.2. Vòng đời của Container

```
Image ──► Created ──► Running ──► Paused
                         │           │
                         ▼           ▼
                       Stopped ◄── Unpaused
                         │
                         ▼
                       Removed
```

### 2.4.3. Các lệnh quản lý Container

```bash
# Chạy container
docker run nginx                              # Chạy nginx, foreground
docker run -d nginx                           # Chạy nền (detached)
docker run -d -p 80:80 nginx                  # Map port host:container
docker run -d -p 3000:3000 --name my-api app  # Đặt tên cho container
docker run -it ubuntu bash                    # Chạy interactive terminal

# Flags quan trọng của docker run:
# -d, --detach     Chạy nền
# -p host:cont     Map port
# --name           Đặt tên
# -e KEY=VALUE     Đặt biến môi trường
# -v host:cont     Mount volume
# --rm             Tự xóa khi dừng
# --network        Gán vào network
# --restart        Chính sách tự restart

docker run -d \
  --name my-nestjs-api \
  -p 3000:3000 \
  -e NODE_ENV=production \
  -e PORT=3000 \
  -e DB_HOST=postgres \
  --restart unless-stopped \
  my-nestjs-api:1.0.0

# Liệt kê container
docker ps                      # Container đang chạy
docker ps -a                   # Tất cả container (kể cả đã dừng)
docker ps -q                   # Chỉ hiển thị ID

# Xem log container
docker logs my-api             # Xem toàn bộ log
docker logs my-api --tail 100  # 100 dòng cuối
docker logs my-api -f          # Follow realtime
docker logs my-api -f --tail 50  # Kết hợp

# Vào trong container (exec)
docker exec -it my-api bash    # Mở bash shell trong container
docker exec -it my-api sh      # Nếu không có bash (Alpine dùng sh)
docker exec my-api ls /app     # Chạy lệnh không cần vào shell

# Dừng và xóa container
docker stop my-api             # Dừng gracefully (SIGTERM, chờ 10s)
docker stop -t 30 my-api       # Chờ 30s trước khi SIGKILL
docker kill my-api             # Dừng ngay (SIGKILL)
docker rm my-api               # Xóa container đã dừng
docker rm -f my-api            # Dừng và xóa luôn
docker rm $(docker ps -aq)     # Xóa tất cả container đã dừng

# Copy file vào/ra container
docker cp my-api:/app/logs ./logs     # Copy từ container ra host
docker cp .env my-api:/app/.env       # Copy từ host vào container

# Xem thông tin chi tiết
docker inspect my-api          # JSON với toàn bộ thông tin
docker stats                   # Monitor CPU/RAM realtime của tất cả container
docker stats my-api            # Monitor một container cụ thể

# Dọn dẹp
docker system prune            # Xóa container đã dừng + image không dùng + network không dùng
docker system prune -a         # Thêm cả image không có container nào dùng
docker system df               # Xem Docker đang chiếm bao nhiêu disk
```

---

## 2.5. Dockerfile

### 2.5.1. Khái niệm

**Dockerfile** là file văn bản chứa tập hợp các lệnh (instructions) để Docker tự động build một image. Dockerfile giống như "công thức" để tạo ra môi trường chạy ứng dụng.

### 2.5.2. Các lệnh cơ bản trong Dockerfile

| Lệnh | Mục đích |
|---|---|
| `FROM` | Chỉ định base image |
| `WORKDIR` | Đặt thư mục làm việc trong container |
| `COPY` | Copy file từ host vào container (theo .dockerignore) |
| `ADD` | Giống COPY nhưng hỗ trợ URL và tự giải nén tar |
| `RUN` | Chạy lệnh trong quá trình build image |
| `ENV` | Đặt biến môi trường |
| `ARG` | Biến chỉ dùng lúc build (không có trong container) |
| `EXPOSE` | Khai báo port container sẽ lắng nghe (tài liệu, không tự mở) |
| `CMD` | Lệnh mặc định khi container chạy (có thể override) |
| `ENTRYPOINT` | Lệnh chính khi container chạy (khó override hơn) |
| `HEALTHCHECK` | Định nghĩa cách kiểm tra sức khỏe container |
| `USER` | Chạy các lệnh tiếp theo với user cụ thể |
| `VOLUME` | Khai báo mount point |

### 2.5.3. Dockerfile cho NestJS — Từ cơ bản đến chuẩn Production

**Phiên bản 1 — Đơn giản (không nên dùng production):**

```dockerfile
# Tên file: Dockerfile.simple
FROM node:20

WORKDIR /app

COPY . .

RUN npm install
RUN npm run build

EXPOSE 3000

CMD ["node", "dist/main.js"]
```

Vấn đề với phiên bản này:
- Image rất to (~1GB) vì dùng `node:20` (full Debian) và copy cả `node_modules`
- Chứa devDependencies không cần thiết
- Chạy bằng user root — không an toàn
- Không có health check
- Không tối ưu layer cache

**Phiên bản 2 — Production-ready với Multi-stage Build:**

```dockerfile
# Tên file: Dockerfile
# ============================================================
# STAGE 1: Build
# Dùng image đầy đủ để build TypeScript sang JavaScript
# ============================================================
FROM node:20-alpine AS builder

# Đặt thư mục làm việc
WORKDIR /app

# Copy package files trước — tận dụng Docker layer cache
# Nếu package.json không đổi, Docker sẽ dùng layer cache
# và bỏ qua bước npm ci (tiết kiệm thời gian build đáng kể)
COPY package*.json ./

# Cài đặt TẤT CẢ dependencies (bao gồm devDependencies cần để build)
# npm ci: nhanh hơn npm install, dùng đúng version trong package-lock.json
RUN npm ci

# Copy toàn bộ source code
COPY . .

# Build TypeScript → JavaScript
RUN npm run build

# ============================================================
# STAGE 2: Production
# Chỉ lấy những gì cần để chạy app — image nhỏ và an toàn hơn
# ============================================================
FROM node:20-alpine AS production

# Cài dumb-init để handle signals đúng cách (PID 1 problem)
# node process sẽ không nhận SIGTERM trực tiếp nếu là PID 1
RUN apk add --no-cache dumb-init

# Tạo user không có quyền root để chạy ứng dụng
# -S: system user, -G: gán vào group
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Đặt NODE_ENV trước khi npm ci để chỉ cài production deps
ENV NODE_ENV=production

# Copy package files
COPY package*.json ./

# Chỉ cài production dependencies (không có devDependencies)
RUN npm ci --only=production && \
    # Xóa npm cache để giảm kích thước image
    npm cache clean --force

# Copy artifact đã build từ stage builder
COPY --from=builder /app/dist ./dist

# Copy các file cần thiết khác (nếu có)
# COPY --from=builder /app/assets ./assets

# Đặt quyền sở hữu cho appuser
RUN chown -R appuser:appgroup /app

# Chuyển sang user không phải root
USER appuser

# Khai báo port (chỉ mang tính tài liệu)
EXPOSE 3000

# Health check — Docker sẽ dùng để kiểm tra container có hoạt động không
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

# Dùng dumb-init làm PID 1 để forward signals đúng cách
ENTRYPOINT ["dumb-init", "--"]

# Lệnh chạy ứng dụng
CMD ["node", "dist/main.js"]
```

### 2.5.4. File `.dockerignore`

`.dockerignore` hoạt động giống `.gitignore` — chỉ định file/thư mục KHÔNG copy vào image khi dùng lệnh `COPY . .`. Thiếu file này, Docker sẽ copy cả `node_modules` (hàng trăm MB) vào image.

```dockerignore
# File: .dockerignore

# Thư mục node_modules — sẽ được cài lại trong container
node_modules/

# Build output — sẽ được build lại trong container
dist/
build/

# File môi trường — tuyệt đối không đưa vào image
.env
.env.*
!.env.example

# Git files
.git/
.gitignore

# Editor configs
.vscode/
.idea/
*.swp
*.swo

# Logs
logs/
*.log
npm-debug.log*

# Test files — không cần trong production image
**/*.spec.ts
**/*.test.ts
**/*.e2e-spec.ts
test/
coverage/

# Docker files
Dockerfile*
docker-compose*.yml

# Documentation
*.md
docs/

# OS files
.DS_Store
Thumbs.db
```

### 2.5.5. Build và chạy image thủ công

```bash
# Build image
docker build -t my-nestjs-api:1.0.0 .
docker build -t my-nestjs-api:1.0.0 -f Dockerfile .  # Chỉ định Dockerfile rõ ràng

# Build với build args
docker build \
  --build-arg NODE_ENV=production \
  -t my-nestjs-api:1.0.0 \
  .

# Xem kích thước image sau khi build
docker images my-nestjs-api

# Chạy thử
docker run -d \
  --name my-api-test \
  -p 3000:3000 \
  -e DB_HOST=host.docker.internal \  # Trỏ đến localhost của host machine
  -e DB_PORT=5432 \
  -e DB_NAME=myapp_dev \
  -e DB_USER=postgres \
  -e DB_PASSWORD=postgres \
  my-nestjs-api:1.0.0

# Kiểm tra
curl http://localhost:3000/health
docker logs my-api-test
```

---

## 2.6. Docker Volume

### 2.6.1. Khái niệm và vấn đề

**Vấn đề:** Mọi dữ liệu trong container chỉ tồn tại trong writable layer của container đó. Khi container bị xóa, **toàn bộ dữ liệu mất**.

```
docker rm postgres-container  →  Mất hết dữ liệu database!
```

**Giải pháp: Docker Volume** — cơ chế lưu trữ dữ liệu bên ngoài container, tồn tại độc lập với vòng đời container.

### 2.6.2. Ba loại Volume

```
┌─────────────────────────────────────────────────────┐
│              Host Machine                           │
│                                                     │
│  /var/lib/docker/volumes/  ←── Named Volume         │
│  /home/alice/data/         ←── Bind Mount           │
│  RAM (tmpfs)               ←── tmpfs Mount          │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │              Container                      │    │
│  │  /data       ← Named Volume mounted here   │    │
│  │  /app        ← Bind Mount mounted here     │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

**1. Named Volume** — Docker quản lý, dữ liệu ở `/var/lib/docker/volumes/`

```bash
# Tạo volume
docker volume create postgres-data
docker volume create redis-data

# Dùng volume khi chạy container
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=secret \
  -v postgres-data:/var/lib/postgresql/data \  # Tên volume:đường dẫn trong container
  postgres:16

# Quản lý volume
docker volume ls                    # Liệt kê tất cả volume
docker volume inspect postgres-data # Xem chi tiết
docker volume rm postgres-data      # Xóa volume (dữ liệu mất!)
docker volume prune                 # Xóa volume không được dùng bởi container nào
```

**2. Bind Mount** — Mount thư mục cụ thể từ host machine

```bash
# Thường dùng trong development để sync code realtime
docker run -d \
  --name my-api-dev \
  -p 3000:3000 \
  -v $(pwd):/app \           # Thư mục hiện tại → /app trong container
  -v /app/node_modules \     # Giữ nguyên node_modules của container
  my-nestjs-api:dev

# Ứng dụng trong container sẽ thấy thay đổi code ngay lập tức
```

**3. tmpfs** — Lưu trong RAM, không ghi ra disk

```bash
docker run -d \
  --name nginx \
  --tmpfs /tmp:rw,noexec,nosuid,size=100m \  # Mount /tmp vào RAM
  nginx
```

### 2.6.3. Khi nào dùng loại nào?

| Loại | Dùng khi nào |
|---|---|
| **Named Volume** | Dữ liệu database, file upload — production |
| **Bind Mount** | Development — sync code, cấu hình |
| **tmpfs** | Cache tạm, session data không cần persist |

---

## 2.7. Docker Network

### 2.7.1. Khái niệm

Theo mặc định, mỗi container chạy trong môi trường network **cách ly**. Hai container không thể giao tiếp với nhau trừ khi cùng nằm trong một **Docker Network**.

### 2.7.2. Các loại Driver Network

| Driver | Mô tả | Dùng khi nào |
|---|---|---|
| `bridge` | Mặc định. Network ảo, container giao tiếp qua tên | Local development, Docker Compose |
| `host` | Container dùng chung network với host | Hiệu năng cao, cần thận trọng |
| `none` | Không có network | Container cần cách ly hoàn toàn |
| `overlay` | Kết nối containers trên nhiều host | Docker Swarm, multi-host |

### 2.7.3. Các lệnh quản lý Network

```bash
# Liệt kê networks
docker network ls

# Tạo network
docker network create my-app-network
docker network create --driver bridge my-app-network  # Tường minh

# Xem chi tiết network
docker network inspect my-app-network

# Kết nối container vào network khi chạy
docker run -d \
  --name my-api \
  --network my-app-network \
  my-nestjs-api:1.0.0

# Kết nối/ngắt container khỏi network (sau khi đã chạy)
docker network connect my-app-network my-api
docker network disconnect my-app-network my-api

# Xóa network
docker network rm my-app-network
docker network prune  # Xóa network không dùng
```

### 2.7.4. Giao tiếp giữa các Container

Khi các container nằm trong cùng một custom bridge network, chúng có thể giao tiếp với nhau qua **tên container** (Docker DNS tự giải tên):

```bash
# Tạo network
docker network create app-network

# Chạy PostgreSQL trong network
docker run -d \
  --name postgres \
  --network app-network \
  -e POSTGRES_DB=myapp \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=secret \
  postgres:16

# Chạy API trong cùng network
docker run -d \
  --name my-api \
  --network app-network \
  -e DB_HOST=postgres \   # Dùng tên container "postgres" thay vì IP
  -e DB_PORT=5432 \
  my-nestjs-api:1.0.0

# Container my-api có thể kết nối đến postgres:5432
# Docker DNS sẽ tự resolve "postgres" → IP của container postgres
```

---

## 2.8. Docker Compose

### 2.8.1. Khái niệm

Trong thực tế, một ứng dụng backend không chạy đơn lẻ. Nó thường bao gồm nhiều service:

```
NestJS API  +  PostgreSQL  +  Redis  +  Nginx
```

Quản lý từng container thủ công bằng `docker run` rất tẻ nhạt và dễ sai. **Docker Compose** giải quyết vấn đề này — nó cho phép định nghĩa và chạy **multi-container applications** trong một file YAML duy nhất.

### 2.8.2. Cài đặt Docker Compose

```bash
# Kiểm tra Docker Compose đã có chưa (Docker Desktop đã tích hợp sẵn)
docker compose version

# Cài standalone (nếu chỉ có Docker Engine)
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### 2.8.3. Cấu trúc file `docker-compose.yml`

```yaml
version: "3.9"            # Phiên bản schema của Docker Compose

services:                 # Định nghĩa các container
  service-name:
    image: ...            # Dùng image có sẵn
    build: ...            # Hoặc build từ Dockerfile
    ports: ...            # Map port
    environment: ...      # Biến môi trường
    volumes: ...          # Mount volume
    networks: ...         # Gán vào network
    depends_on: ...       # Phụ thuộc vào service nào
    restart: ...          # Chính sách tự restart

volumes:                  # Khai báo named volumes
  volume-name:

networks:                 # Khai báo networks
  network-name:
```

### 2.8.4. Docker Compose hoàn chỉnh cho NestJS + PostgreSQL + Redis + Nginx

**Cấu trúc thư mục dự án:**

```
my-nestjs-app/
├── src/
├── nginx/
│   └── nginx.conf
├── docker-compose.yml
├── docker-compose.prod.yml
├── Dockerfile
├── .env
└── .env.example
```

**File `.env`:**

```bash
# Application
NODE_ENV=development
APP_PORT=3000

# PostgreSQL
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_DB=myapp_dev
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres_dev_password

# Redis
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=redis_dev_password
```

**File `docker-compose.yml` (Development):**

```yaml
# Tên file: docker-compose.yml
# Mục đích: môi trường development

version: "3.9"

services:
  # ─────────────────────────────────────────
  # NestJS API
  # ─────────────────────────────────────────
  api:
    build:
      context: .                  # Thư mục chứa Dockerfile
      dockerfile: Dockerfile
      target: builder             # Dùng stage "builder" cho dev (có devDeps)
    container_name: myapp-api-dev
    restart: unless-stopped
    ports:
      - "${APP_PORT:-3000}:3000"  # Dùng biến từ .env, mặc định 3000
    environment:
      - NODE_ENV=development
      - PORT=3000
      - DB_HOST=postgres          # Dùng tên service, không phải localhost
      - DB_PORT=5432
      - DB_NAME=${POSTGRES_DB}
      - DB_USER=${POSTGRES_USER}
      - DB_PASSWORD=${POSTGRES_PASSWORD}
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_PASSWORD=${REDIS_PASSWORD}
    volumes:
      - .:/app                    # Bind mount toàn bộ project để hot reload
      - /app/node_modules         # Giữ nguyên node_modules của container
    depends_on:
      postgres:
        condition: service_healthy  # Chờ postgres healthy mới start
      redis:
        condition: service_healthy
    networks:
      - app-network

  # ─────────────────────────────────────────
  # PostgreSQL
  # ─────────────────────────────────────────
  postgres:
    image: postgres:16-alpine
    container_name: myapp-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-myapp_dev}
      POSTGRES_USER: ${POSTGRES_USER:-postgres}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-postgres}
    ports:
      - "5432:5432"               # Expose để dùng GUI như pgAdmin, DBeaver
    volumes:
      - postgres-data:/var/lib/postgresql/data  # Lưu dữ liệu
      - ./db/init:/docker-entrypoint-initdb.d   # Script khởi tạo DB (nếu có)
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres} -d ${POSTGRES_DB:-myapp_dev}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    networks:
      - app-network

  # ─────────────────────────────────────────
  # Redis
  # ─────────────────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: myapp-redis
    restart: unless-stopped
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD:-redis_password}
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"               # Expose để dùng RedisInsight, redis-cli
    volumes:
      - redis-data:/data          # Lưu dữ liệu
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD:-redis_password}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  # ─────────────────────────────────────────
  # pgAdmin (chỉ cho development)
  # ─────────────────────────────────────────
  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: myapp-pgadmin
    restart: unless-stopped
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@local.dev
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"                 # Truy cập tại http://localhost:5050
    depends_on:
      - postgres
    networks:
      - app-network
    profiles:
      - tools                     # Chỉ chạy khi: docker compose --profile tools up

# ─────────────────────────────────────────
# Named Volumes
# ─────────────────────────────────────────
volumes:
  postgres-data:
    driver: local
  redis-data:
    driver: local

# ─────────────────────────────────────────
# Networks
# ─────────────────────────────────────────
networks:
  app-network:
    driver: bridge
```

**File `docker-compose.prod.yml` (Production — override):**

```yaml
# Tên file: docker-compose.prod.yml
# Dùng cùng với docker-compose.yml:
# docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

version: "3.9"

services:
  api:
    build:
      target: production          # Dùng stage production (nhỏ, không có devDeps)
    restart: always
    ports: []                     # Xóa port expose ra ngoài (chỉ qua Nginx)
    volumes: []                   # Không bind mount code

  postgres:
    ports: []                     # Không expose PostgreSQL ra ngoài
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}  # Bắt buộc đặt trong .env.prod

  redis:
    ports: []                     # Không expose Redis ra ngoài

  nginx:
    image: nginx:1.25-alpine
    container_name: myapp-nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - nginx-logs:/var/log/nginx
    depends_on:
      - api
    networks:
      - app-network

volumes:
  nginx-logs:
```

**File `nginx/nginx.conf`:**

```nginx
# Tên file: nginx/nginx.conf

events {
    worker_connections 1024;
}

http {
    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_types text/plain application/json application/javascript text/css;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

    upstream nestjs_api {
        server api:3000;  # "api" là tên service trong Docker Compose
    }

    server {
        listen 80;
        server_name _;

        # Giới hạn request size (cho file upload)
        client_max_body_size 10M;

        location /api/ {
            limit_req zone=api burst=20 nodelay;

            proxy_pass http://nestjs_api;
            proxy_http_version 1.1;

            # Headers cần thiết
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_cache_bypass $http_upgrade;
        }

        # Health check endpoint không rate limit
        location /health {
            proxy_pass http://nestjs_api;
        }
    }
}
```

### 2.8.5. Các lệnh Docker Compose thường dùng

```bash
# Khởi động tất cả services
docker compose up                  # Foreground (xem log trực tiếp)
docker compose up -d               # Background (detached)
docker compose up --build          # Build lại image trước khi start
docker compose up -d --build       # Build + chạy nền

# Chỉ khởi động một số service
docker compose up -d postgres redis

# Xem trạng thái
docker compose ps                  # Danh sách services và trạng thái
docker compose top                 # Processes đang chạy trong mỗi container

# Xem log
docker compose logs                # Log của tất cả services
docker compose logs api            # Log của service api
docker compose logs -f api         # Follow log của api
docker compose logs -f --tail=50 api postgres  # Follow log của nhiều service

# Dừng services
docker compose stop                # Dừng containers (không xóa)
docker compose down                # Dừng và xóa containers + network
docker compose down -v             # + Xóa cả volumes (MẤT DỮ LIỆU!)
docker compose down --rmi all      # + Xóa cả images

# Restart
docker compose restart             # Restart tất cả
docker compose restart api         # Restart một service

# Vào container
docker compose exec api bash       # Mở shell trong container api
docker compose exec postgres psql -U postgres -d myapp_dev

# Scale (chạy nhiều instance)
docker compose up -d --scale api=3  # Chạy 3 instance của api

# Chạy lệnh một lần (không cần container đang chạy)
docker compose run --rm api npm run migration:run

# Build lại image
docker compose build               # Build tất cả
docker compose build api           # Build chỉ service api
docker compose build --no-cache api  # Build không dùng cache

# Production (dùng nhiều file compose)
docker compose \
  -f docker-compose.yml \
  -f docker-compose.prod.yml \
  up -d --build
```

---

## 2.9. Docker Registry

### 2.9.1. Khái niệm

**Docker Registry** là kho lưu trữ Docker Images. Sau khi build image trên máy local hoặc CI server, bạn cần **push** lên Registry để các server khác có thể **pull** về và chạy.

```
Developer/CI Server              Registry              Production Server
       ↓                            ↓                        ↓
  docker build          →      docker push        →      docker pull
  (tạo image)               (đẩy lên registry)         (kéo về và chạy)
```

### 2.9.2. Các Registry phổ biến

| Registry | Mô tả | Phù hợp |
|---|---|---|
| **Docker Hub** | Registry mặc định, public miễn phí | Open source, cá nhân |
| **GitHub Container Registry (GHCR)** | Tích hợp với GitHub Actions | Dự án dùng GitHub |
| **AWS ECR** | Amazon Elastic Container Registry | Dự án trên AWS |
| **GCP Artifact Registry** | Google Cloud | Dự án trên GCP |
| **Azure Container Registry** | Microsoft Azure | Dự án trên Azure |
| **Self-hosted (Registry 2)** | Tự host, toàn quyền kiểm soát | Doanh nghiệp cần bảo mật cao |

### 2.9.3. Làm việc với Docker Hub

```bash
# Đăng nhập
docker login
docker login -u username -p password  # Không khuyến khích (password lộ trong history)
echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin  # Cách an toàn

# Tag image trước khi push (image phải có tên đúng format: username/repo:tag)
docker tag my-nestjs-api:1.0.0 myusername/my-nestjs-api:1.0.0
docker tag my-nestjs-api:1.0.0 myusername/my-nestjs-api:latest

# Push lên Docker Hub
docker push myusername/my-nestjs-api:1.0.0
docker push myusername/my-nestjs-api:latest

# Pull về (trên server khác)
docker pull myusername/my-nestjs-api:1.0.0
```

### 2.9.4. Làm việc với GitHub Container Registry (GHCR)

```bash
# Đăng nhập với GitHub Personal Access Token
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Tag image
docker tag my-nestjs-api:1.0.0 ghcr.io/username/my-nestjs-api:1.0.0

# Push
docker push ghcr.io/username/my-nestjs-api:1.0.0

# Pull
docker pull ghcr.io/username/my-nestjs-api:1.0.0
```

---

## 2.10. Multi-stage Build

### 2.10.1. Khái niệm

**Multi-stage Build** là kỹ thuật dùng nhiều `FROM` trong một Dockerfile. Mỗi `FROM` bắt đầu một stage mới. Chỉ những file được copy từ stage trước sang stage sau mới có trong image cuối cùng.

**Lợi ích:**
- Image production nhỏ hơn nhiều (chỉ chứa runtime, không có build tools)
- Một Dockerfile duy nhất cho cả build và production
- Build tools (TypeScript compiler, webpack...) không có trong image cuối

### 2.10.2. So sánh kích thước

```
node:20 (full Debian)       ~1.1 GB
node:20-alpine              ~170 MB
node:20-alpine (optimized)  ~85 MB   ← Multi-stage + chỉ copy dist + prod deps
```

### 2.10.3. Ví dụ đầy đủ đã có ở mục 2.5.3

Xem lại phần **Dockerfile — Phiên bản 2** trong mục 2.5.3. Đó chính là ví dụ Multi-stage Build hoàn chỉnh.

---

## 2.11. Ví Dụ Thực Tế: Toàn Bộ Stack NestJS

### 2.11.1. Cấu trúc dự án

```
my-nestjs-app/
├── src/
│   ├── main.ts
│   ├── app.module.ts
│   ├── health/
│   │   └── health.controller.ts
│   └── ...
├── nginx/
│   └── nginx.conf
├── db/
│   └── init/
│       └── 01-init.sql         # Script khởi tạo DB tự động
├── Dockerfile
├── .dockerignore
├── docker-compose.yml
├── docker-compose.prod.yml
├── .env.example
└── package.json
```

### 2.11.2. Health Check endpoint trong NestJS

```typescript
// src/health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import {
  HealthCheck,
  HealthCheckService,
  TypeOrmHealthIndicator,
} from '@nestjs/terminus';
import { InjectConnection } from '@nestjs/typeorm';
import { Connection } from 'typeorm';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      // Kiểm tra kết nối database
      () => this.db.pingCheck('database'),
    ]);
  }

  // Endpoint đơn giản hơn — dùng cho Docker HEALTHCHECK
  @Get('ping')
  ping() {
    return { status: 'ok', timestamp: new Date().toISOString() };
  }
}
```

### 2.11.3. Kết nối Database và Redis trong NestJS

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { CacheModule } from '@nestjs/cache-manager';
import * as redisStore from 'cache-manager-redis-store';

@Module({
  imports: [
    // Config Module — đọc biến môi trường từ .env
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: '.env',
    }),

    // TypeORM — kết nối PostgreSQL
    TypeOrmModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        type: 'postgres',
        host: config.get<string>('DB_HOST'),
        port: config.get<number>('DB_PORT', 5432),
        database: config.get<string>('DB_NAME'),
        username: config.get<string>('DB_USER'),
        password: config.get<string>('DB_PASSWORD'),
        entities: [__dirname + '/**/*.entity{.ts,.js}'],
        synchronize: config.get<string>('NODE_ENV') !== 'production',
        logging: config.get<string>('NODE_ENV') === 'development',
        ssl: config.get<string>('NODE_ENV') === 'production'
          ? { rejectUnauthorized: false }
          : false,
      }),
    }),

    // Cache Module — kết nối Redis
    CacheModule.registerAsync({
      isGlobal: true,
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        store: redisStore,
        host: config.get<string>('REDIS_HOST', 'localhost'),
        port: config.get<number>('REDIS_PORT', 6379),
        password: config.get<string>('REDIS_PASSWORD'),
        ttl: 300, // 5 phút mặc định
      }),
    }),
  ],
})
export class AppModule {}
```

### 2.11.4. Chạy toàn bộ stack

```bash
# 1. Clone project
git clone https://github.com/yourname/my-nestjs-app.git
cd my-nestjs-app

# 2. Tạo file .env từ template
cp .env.example .env
# Chỉnh sửa .env với giá trị thực

# 3. Khởi động tất cả services
docker compose up -d --build

# 4. Kiểm tra tất cả đã chạy
docker compose ps

# 5. Xem log
docker compose logs -f api

# 6. Chạy database migration
docker compose exec api npm run migration:run

# 7. Kiểm tra health
curl http://localhost:3000/health/ping

# 8. Dừng khi xong
docker compose down
```

---

## 2.12. Best Practices

### 2.12.1. Bảo mật

```dockerfile
# ✅ Không chạy bằng root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# ✅ Không đưa secret vào image — dùng ENV khi chạy container
# ❌ KHÔNG làm thế này:
ENV DB_PASSWORD=secret123  # Bị lộ trong image history!

# ✅ Scan vulnerabilities
docker scout quickview my-api:latest
docker scan my-api:latest  # Phiên bản cũ
```

### 2.12.2. Tối ưu kích thước image

```dockerfile
# ✅ Dùng Alpine base image
FROM node:20-alpine

# ✅ Gộp RUN commands để giảm layer
RUN apk add --no-cache curl wget && \
    npm ci --only=production && \
    npm cache clean --force

# ✅ Dùng .dockerignore
# (xem mục 2.5.4)

# ✅ Multi-stage Build
# (xem mục 2.10)
```

### 2.12.3. Tận dụng Layer Cache

```dockerfile
# ✅ Copy package.json TRƯỚC khi copy source code
# Nếu source code thay đổi nhưng package.json không đổi,
# Docker sẽ dùng cache cho bước npm install
COPY package*.json ./
RUN npm ci
COPY . .  # Chỉ layer này bị invalidate khi code thay đổi

# ❌ Tệ — npm install luôn chạy lại dù deps không đổi
COPY . .
RUN npm install
```

### 2.12.4. Docker Compose

```yaml
# ✅ Luôn đặt healthcheck và depends_on với condition
depends_on:
  postgres:
    condition: service_healthy  # Không phải service_started

# ✅ Đặt restart policy
restart: unless-stopped  # Production
restart: "no"            # Development (tắt là tắt)

# ✅ Dùng biến môi trường từ .env
environment:
  - DB_PASSWORD=${DB_PASSWORD}  # Không hard-code

# ✅ Không expose port database ra ngoài trong production
ports: []  # Override trong docker-compose.prod.yml
```

### 2.12.5. Đặt tên tag hợp lý

```bash
# ✅ Semantic versioning
docker tag app:build-123 myrepo/app:1.2.3
docker tag app:build-123 myrepo/app:latest

# ✅ Dùng Git commit SHA để traceability
GIT_SHA=$(git rev-parse --short HEAD)
docker tag app myrepo/app:${GIT_SHA}

# ❌ Không dùng "latest" trong production — không biết đang chạy version nào
```

---

## Tóm Tắt Chương 2

| Khái niệm | Vai trò |
|---|---|
| **Image** | Template read-only, bản thiết kế của container |
| **Container** | Instance đang chạy từ image |
| **Dockerfile** | Công thức để build image |
| **Volume** | Lưu dữ liệu bền vững ngoài container |
| **Network** | Cho phép container giao tiếp với nhau |
| **Docker Compose** | Quản lý multi-container application |
| **Registry** | Kho lưu trữ và phân phối image |
| **Multi-stage Build** | Tối ưu kích thước image production |

---

> **Chương tiếp theo:** [Chương 3 — CI/CD](./chapter-03-cicd.md)  
> Bạn đã biết cách đóng gói ứng dụng vào Docker. Tiếp theo, chúng ta sẽ học cách **tự động hóa** toàn bộ quy trình từ push code đến deploy bằng GitHub Actions.
