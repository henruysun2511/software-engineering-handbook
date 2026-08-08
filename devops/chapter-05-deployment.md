# Chương 5: Deployment

> **Mức độ quan trọng:** ⭐⭐⭐⭐⭐  
> **Đối tượng:** Backend Developer (NestJS/Express), trình độ Intern → Junior  
> **Mục tiêu chương:** Nắm vững toàn bộ quy trình triển khai ứng dụng NestJS lên server thực tế — từ cách quản lý môi trường, quy trình build và deploy, đến zero downtime deployment và rollback khi có sự cố.

---

## 5.1. Tổng Quan Về Deployment

### 5.1.1. Deployment là gì?

**Deployment** (triển khai) là quá trình đưa phiên bản mới của ứng dụng lên môi trường chạy thực tế để người dùng có thể sử dụng. Đây là bước cuối cùng trong vòng đời phát triển phần mềm.

Trong thực tế, deployment không chỉ là "copy code lên server rồi chạy". Một quy trình deployment tốt cần đảm bảo:

- **Không có downtime** — người dùng không bị gián đoạn dịch vụ.
- **Có thể rollback** — nếu phiên bản mới có lỗi, có thể quay về phiên bản cũ trong vài phút.
- **Nhất quán** — mọi môi trường (staging, production) đều chạy cùng artifact.
- **Có audit trail** — biết ai deploy gì, lúc nào, từ commit nào.

### 5.1.2. Vòng đời một thay đổi

```
Viết code → PR Review → Merge → CI pass → Deploy staging → QA test → Deploy production
    ↑                                                                        │
    └────────────────────── Lặp lại ────────────────────────────────────────┘
```

---

## 5.2. Ba Môi Trường Chuẩn

### 5.2.1. Development (Môi trường phát triển)

**Development** là môi trường cục bộ trên máy của developer. Mục tiêu duy nhất là giúp developer viết code nhanh và tiện.

**Đặc điểm:**

| Thuộc tính | Giá trị |
|---|---|
| Vị trí | Máy cá nhân của developer |
| Dữ liệu | Seed data / fake data |
| Database | Chạy local hoặc Docker |
| Debug | Bật toàn bộ (verbose log, stack trace) |
| Hot reload | Có (`nest start --watch`) |
| HTTPS | Không bắt buộc |
| Shared | Không — mỗi dev có môi trường riêng |

**Cấu hình `.env.development`:**

```bash
# .env (hoặc .env.development)
NODE_ENV=development
PORT=3000

# Database chạy local hoặc Docker
DB_HOST=localhost
DB_PORT=5432
DB_NAME=myapp_dev
DB_USER=postgres
DB_PASSWORD=postgres

# TypeORM synchronize=true để tự tạo table (KHÔNG dùng ở production)
DB_SYNCHRONIZE=true
DB_LOGGING=true

# Redis local
REDIS_HOST=localhost
REDIS_PORT=6379

# JWT secret đơn giản cho dev
JWT_SECRET=dev-secret-not-secure
JWT_EXPIRES_IN=7d

# Email — dùng Mailtrap (không gửi thật)
MAIL_HOST=sandbox.smtp.mailtrap.io
MAIL_PORT=2525
MAIL_USER=your-mailtrap-user
MAIL_PASS=your-mailtrap-pass

# Log level verbose
LOG_LEVEL=debug
```

**`package.json` scripts cho development:**

```json
{
  "scripts": {
    "start": "nest start",
    "start:dev": "nest start --watch",
    "start:debug": "nest start --debug --watch",
    "build": "nest build",
    "db:seed": "ts-node src/database/seeds/run-seeds.ts",
    "migration:generate": "typeorm migration:generate -d src/database/data-source.ts",
    "migration:run": "typeorm migration:run -d src/database/data-source.ts",
    "migration:revert": "typeorm migration:revert -d src/database/data-source.ts"
  }
}
```

**Chạy development stack với Docker Compose:**

```bash
# Chỉ khởi động infrastructure (DB + Redis), code chạy trực tiếp trên máy
docker compose up -d postgres redis

# Chạy NestJS với hot reload
npm run start:dev
```

### 5.2.2. Staging (Môi trường kiểm thử)

**Staging** là môi trường giống production nhất có thể — chạy trên server thật, dùng cùng Docker image, cùng cấu hình — nhưng với dữ liệu test.

**Mục đích:**
- QA team test tính năng mới trước khi release.
- Test migration database với dữ liệu gần giống production.
- Demo cho stakeholder, product manager.
- Phát hiện lỗi môi trường (environment-specific bugs) trước khi lên production.

**Nguyên tắc vàng:** Nếu không chạy được trên staging, tuyệt đối không deploy lên production.

**Đặc điểm:**

| Thuộc tính | Giá trị |
|---|---|
| Vị trí | Server riêng (VPS nhỏ hơn production) |
| Dữ liệu | Anonymized copy từ production, hoặc seed data thực tế |
| Database | Server riêng, không share với production |
| Debug | Tắt một phần (không có stack trace trong response) |
| Hot reload | Không — chạy production build |
| HTTPS | Có |
| URL | `staging.myapp.com` hoặc `api-staging.myapp.com` |
| Deploy | Tự động khi merge vào nhánh `main` |

**`.env.staging`:**

```bash
NODE_ENV=production       # Quan trọng — staging chạy ở mode production
PORT=3000

DB_HOST=staging-db.myapp.internal
DB_PORT=5432
DB_NAME=myapp_staging
DB_USER=myapp_staging_user
DB_PASSWORD=${DB_PASSWORD}    # Đọc từ secret manager / CI secrets

DB_SYNCHRONIZE=false      # KHÔNG synchronize — chỉ dùng migration
DB_LOGGING=false          # Tắt SQL logging

REDIS_HOST=staging-redis.myapp.internal
REDIS_PORT=6379
REDIS_PASSWORD=${REDIS_PASSWORD}

JWT_SECRET=${JWT_SECRET}
JWT_EXPIRES_IN=1d

MAIL_HOST=smtp.sendgrid.net   # Email thật nhưng gửi đến test account
MAIL_PORT=587

LOG_LEVEL=info            # Không verbose như dev
```

### 5.2.3. Production (Môi trường thực tế)

**Production** là môi trường người dùng thật sử dụng. Mọi thay đổi ở đây đều có tác động trực tiếp đến người dùng và doanh nghiệp.

**Nguyên tắc của production:**
- **Bảo mật tối đa** — mọi secret phải được quản lý chặt chẽ.
- **Hiệu năng tối ưu** — không debug log, không stack trace.
- **Sẵn sàng cao** — health check, auto-restart, monitoring.
- **Không thử nghiệm** — chỉ deploy code đã test kỹ trên staging.

**Đặc điểm:**

| Thuộc tính | Giá trị |
|---|---|
| Vị trí | Server mạnh, có thể multi-instance |
| Dữ liệu | Dữ liệu thật của người dùng |
| Database | High availability, backup hàng ngày |
| Debug | Tắt hoàn toàn |
| HTTPS | Bắt buộc |
| URL | `api.myapp.com` |
| Deploy | Có manual approval |
| Monitoring | 24/7 |

**`.env.production` (lưu trong secret manager, KHÔNG commit lên Git):**

```bash
NODE_ENV=production
PORT=3000

DB_HOST=prod-db.myapp.internal
DB_PORT=5432
DB_NAME=myapp_production
DB_USER=myapp_prod_user
DB_PASSWORD=<very-strong-random-password>
DB_SYNCHRONIZE=false
DB_LOGGING=false
DB_SSL=true

REDIS_HOST=prod-redis.myapp.internal
REDIS_PORT=6379
REDIS_PASSWORD=<very-strong-redis-password>

JWT_SECRET=<256-bit-random-secret>
JWT_EXPIRES_IN=1h         # Ngắn hơn dev vì bảo mật

# Email production
MAIL_HOST=smtp.sendgrid.net
MAIL_PORT=587
MAIL_USER=apikey
MAIL_PASS=<sendgrid-api-key>
MAIL_FROM=noreply@myapp.com

LOG_LEVEL=warn            # Chỉ log warning và error
```

---

## 5.3. Environment Variables — Quản Lý Cấu Hình

### 5.3.1. Nguyên tắc 12-Factor App

**The Twelve-Factor App** là tập hợp best practices cho ứng dụng hiện đại. Factor III — Config — quy định: *"Lưu cấu hình trong môi trường, không trong code."*

```
Sai ❌                          Đúng ✅
─────────────────────────────────────────────────────
const dbUrl = "postgresql://   const dbUrl = process.env.DATABASE_URL
  prod-user:secret@            // Giá trị đến từ bên ngoài code
  db.prod.com/myapp"
```

### 5.3.2. Cấu hình NestJS với `@nestjs/config`

```typescript
// src/config/configuration.ts
// Định nghĩa cấu trúc config và giá trị mặc định

export default () => ({
  app: {
    nodeEnv: process.env.NODE_ENV || 'development',
    port: parseInt(process.env.PORT, 10) || 3000,
    logLevel: process.env.LOG_LEVEL || 'debug',
  },

  database: {
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT, 10) || 5432,
    name: process.env.DB_NAME || 'myapp_dev',
    user: process.env.DB_USER || 'postgres',
    password: process.env.DB_PASSWORD || 'postgres',
    synchronize: process.env.DB_SYNCHRONIZE === 'true',
    logging: process.env.DB_LOGGING === 'true',
    ssl: process.env.DB_SSL === 'true',
  },

  redis: {
    host: process.env.REDIS_HOST || 'localhost',
    port: parseInt(process.env.REDIS_PORT, 10) || 6379,
    password: process.env.REDIS_PASSWORD || undefined,
  },

  jwt: {
    secret: process.env.JWT_SECRET || 'dev-secret',
    expiresIn: process.env.JWT_EXPIRES_IN || '7d',
  },
});
```

```typescript
// src/config/env.validation.ts
// Validate biến môi trường khi app khởi động
// Nếu thiếu biến bắt buộc → app từ chối khởi động (fail fast)

import { plainToInstance } from 'class-transformer';
import {
  IsEnum,
  IsNumber,
  IsOptional,
  IsString,
  Min,
  Max,
  validateSync,
} from 'class-validator';

enum Environment {
  Development = 'development',
  Staging = 'staging',
  Production = 'production',
  Test = 'test',
}

class EnvironmentVariables {
  @IsEnum(Environment)
  NODE_ENV: Environment;

  @IsNumber()
  @Min(1)
  @Max(65535)
  PORT: number;

  @IsString()
  DB_HOST: string;

  @IsNumber()
  DB_PORT: number;

  @IsString()
  DB_NAME: string;

  @IsString()
  DB_USER: string;

  @IsString()
  DB_PASSWORD: string;

  @IsString()
  JWT_SECRET: string;

  @IsOptional()
  @IsString()
  REDIS_HOST?: string;
}

export function validate(config: Record<string, unknown>) {
  const validatedConfig = plainToInstance(EnvironmentVariables, config, {
    enableImplicitConversion: true,
  });

  const errors = validateSync(validatedConfig, {
    skipMissingProperties: false,
  });

  if (errors.length > 0) {
    // Hiển thị lỗi rõ ràng — biết ngay thiếu biến nào
    throw new Error(
      `Environment validation failed:\n${errors
        .map((e) => Object.values(e.constraints || {}).join(', '))
        .join('\n')}`,
    );
  }

  return validatedConfig;
}
```

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import configuration from './config/configuration';
import { validate } from './config/env.validation';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      load: [configuration],
      validate,              // Validate ngay khi app khởi động
      cache: true,           // Cache để không đọc process.env liên tục
    }),
  ],
})
export class AppModule {}
```

### 5.3.3. Đọc config trong service

```typescript
// src/users/users.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class UsersService {
  constructor(private readonly config: ConfigService) {}

  someMethod() {
    // Cách 1: Đọc trực tiếp biến môi trường
    const dbHost = this.config.get<string>('DB_HOST');

    // Cách 2: Đọc qua nested config (khuyến nghị)
    const dbHost2 = this.config.get<string>('database.host');
    const port = this.config.get<number>('app.port');

    // Cách 3: Với giá trị mặc định
    const logLevel = this.config.get<string>('app.logLevel', 'info');

    // Cách 4: Bắt buộc phải có (throw nếu không tìm thấy)
    const jwtSecret = this.config.getOrThrow<string>('jwt.secret');
  }
}
```

---

## 5.4. Build Process

### 5.4.1. Quy trình build NestJS

```
Source (TypeScript)
    │
    ▼ npm run build (tsc)
dist/ (JavaScript)
    │
    ▼ docker build
Docker Image
    │
    ▼ docker push
Container Registry
    │
    ▼ docker pull (trên server)
    │
    ▼ docker compose up -d
Container đang chạy
```

### 5.4.2. `tsconfig.json` cho production build

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "declaration": true,
    "removeComments": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowSyntheticDefaultImports": true,
    "target": "ES2021",
    "sourceMap": false,
    "outDir": "./dist",
    "baseUrl": "./",
    "incremental": true,
    "skipLibCheck": true,
    "strictNullChecks": false,
    "noImplicitAny": false,
    "strictBindCallApply": false,
    "forceConsistentCasingInFileNames": false,
    "noFallthroughCasesInSwitch": false,
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": [
    "node_modules",
    "dist",
    "**/*.spec.ts",
    "**/*.test.ts",
    "test/**/*"
  ]
}
```

### 5.4.3. Kiểm tra artifact trước khi deploy

```bash
# Build locally để verify
npm run build

# Kiểm tra dist/ có đầy đủ không
ls -la dist/
node dist/main.js  # Thử chạy nhanh để xem có lỗi runtime không

# Kiểm tra Docker image
docker build -t myapp:test .
docker run --rm -p 3000:3000 \
  -e NODE_ENV=production \
  -e PORT=3000 \
  -e DB_HOST=localhost \
  myapp:test &

curl http://localhost:3000/health/ping
```

---

## 5.5. Deploy Flow — Quy Trình Deploy Thực Tế

### 5.5.1. Chuẩn bị VPS lần đầu

Kịch bản: Server Ubuntu 22.04 mới tinh, chưa có gì.

```bash
# ─────────────────────────────────────────
# Bước 1: Cập nhật hệ thống
# ─────────────────────────────────────────
sudo apt update && sudo apt upgrade -y

# ─────────────────────────────────────────
# Bước 2: Cài đặt công cụ cơ bản
# ─────────────────────────────────────────
sudo apt install -y \
  curl wget git vim htop \
  ufw fail2ban \
  unzip tar

# ─────────────────────────────────────────
# Bước 3: Cài Docker
# ─────────────────────────────────────────
curl -fsSL https://get.docker.com | sh

# Tạo user deploy (không dùng root)
sudo useradd -m -s /bin/bash deploy
sudo usermod -aG docker deploy
sudo usermod -aG sudo deploy

# Chuyển sang user deploy
sudo su - deploy

# Tạo SSH key cho GitHub Actions
ssh-keygen -t ed25519 -C "github-actions" -f ~/.ssh/github_actions -N ""
cat ~/.ssh/github_actions.pub >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys

# In private key để thêm vào GitHub Secrets
echo "=== Copy nội dung này vào GitHub Secret SSH_PRIVATE_KEY ==="
cat ~/.ssh/github_actions

# ─────────────────────────────────────────
# Bước 4: Cấu hình Firewall
# ─────────────────────────────────────────
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh          # Port 22
sudo ufw allow http         # Port 80
sudo ufw allow https        # Port 443
sudo ufw enable
sudo ufw status

# ─────────────────────────────────────────
# Bước 5: Tạo cấu trúc thư mục ứng dụng
# ─────────────────────────────────────────
sudo mkdir -p /opt/myapp/{data,logs,uploads,nginx/ssl}
sudo chown -R deploy:deploy /opt/myapp

# ─────────────────────────────────────────
# Bước 6: Tạo file .env production
# ─────────────────────────────────────────
cat > /opt/myapp/.env << 'EOF'
NODE_ENV=production
PORT=3000
DB_HOST=postgres
DB_PORT=5432
DB_NAME=myapp_production
DB_USER=myapp_user
DB_PASSWORD=CHANGE_ME_STRONG_PASSWORD
DB_SYNCHRONIZE=false
DB_SSL=false
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=CHANGE_ME_REDIS_PASSWORD
JWT_SECRET=CHANGE_ME_256_BIT_RANDOM_SECRET
JWT_EXPIRES_IN=1h
LOG_LEVEL=warn
EOF

# Bảo vệ file .env
chmod 600 /opt/myapp/.env

# ─────────────────────────────────────────
# Bước 7: Cài Nginx và lấy SSL
# ─────────────────────────────────────────
sudo apt install nginx -y
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# Lấy certificate (domain phải trỏ về server trước)
sudo certbot --nginx -d api.myapp.com --non-interactive \
  --agree-tos --email admin@myapp.com
```

### 5.5.2. Script deploy hoàn chỉnh trên server

```bash
#!/bin/bash
# File: /opt/myapp/scripts/deploy.sh
# Mục đích: Deploy ứng dụng NestJS với Docker Compose
# Gọi bởi: GitHub Actions hoặc thủ công

set -euo pipefail    # Dừng ngay khi có lỗi (-e), unbound var (-u), pipe fail (-o pipefail)

# ─────────────────────────────────────────
# Cấu hình
# ─────────────────────────────────────────
APP_DIR="/opt/myapp"
COMPOSE_FILE="$APP_DIR/docker-compose.yml"
COMPOSE_PROD_FILE="$APP_DIR/docker-compose.prod.yml"
LOG_FILE="/opt/myapp/logs/deploy.log"
HEALTH_URL="http://localhost:3000/health/ping"
HEALTH_RETRIES=12          # Số lần thử health check
HEALTH_WAIT=5              # Giây chờ giữa mỗi lần thử
IMAGE_TAG="${IMAGE_TAG:-latest}"    # Đọc từ env hoặc dùng latest

# Tạo thư mục log nếu chưa có
mkdir -p "$(dirname $LOG_FILE)"

# Hàm log có timestamp
log() {
  local msg="[$(date '+%Y-%m-%d %H:%M:%S')] $1"
  echo "$msg"
  echo "$msg" >> "$LOG_FILE"
}

# Hàm kiểm tra health
wait_for_health() {
  log "⏳ Waiting for app to be healthy..."
  for i in $(seq 1 $HEALTH_RETRIES); do
    HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" "$HEALTH_URL" 2>/dev/null || echo "000")
    if [ "$HTTP_CODE" = "200" ]; then
      log "✅ App is healthy (HTTP $HTTP_CODE) after $i attempt(s)"
      return 0
    fi
    log "   Attempt $i/$HEALTH_RETRIES: HTTP $HTTP_CODE, retrying in ${HEALTH_WAIT}s..."
    sleep $HEALTH_WAIT
  done
  log "❌ App failed health check after $HEALTH_RETRIES attempts"
  return 1
}

# ─────────────────────────────────────────
# Bắt đầu deploy
# ─────────────────────────────────────────
log "========================================="
log "🚀 Starting deployment"
log "   Image tag : $IMAGE_TAG"
log "   Deploy dir: $APP_DIR"
log "========================================="

cd "$APP_DIR"

# Kiểm tra file cần thiết tồn tại
for f in "$COMPOSE_FILE" "$COMPOSE_PROD_FILE" ".env"; do
  if [ ! -f "$f" ]; then
    log "❌ Required file not found: $f"
    exit 1
  fi
done

# Đăng nhập registry (nếu private)
if [ -n "${REGISTRY_TOKEN:-}" ]; then
  log "🔐 Logging into container registry..."
  echo "$REGISTRY_TOKEN" | docker login ghcr.io \
    -u "$REGISTRY_USER" --password-stdin
fi

# Pull image mới nhất
log "📥 Pulling image: $IMAGE_TAG..."
docker compose \
  -f "$COMPOSE_FILE" \
  -f "$COMPOSE_PROD_FILE" \
  pull api

# Lưu thông tin container đang chạy (để rollback nếu cần)
CURRENT_IMAGE=$(docker inspect myapp-api \
  --format '{{.Config.Image}}' 2>/dev/null || echo "none")
log "📌 Current image: $CURRENT_IMAGE"

# Chạy database migration TRƯỚC khi swap container
# (migration phải tương thích ngược — backward compatible)
log "🗄️  Running database migrations..."
docker compose \
  -f "$COMPOSE_FILE" \
  -f "$COMPOSE_PROD_FILE" \
  run --rm \
  -e RUN_MIGRATIONS=true \
  api node dist/database/migrate.js || {
    log "❌ Migration failed! Aborting deploy."
    exit 1
  }

# Deploy container mới (chỉ restart service api, không restart db/redis)
log "🔄 Deploying new container..."
docker compose \
  -f "$COMPOSE_FILE" \
  -f "$COMPOSE_PROD_FILE" \
  up -d --no-deps --force-recreate api nginx

# Chờ và kiểm tra health
if wait_for_health; then
  log "🎉 Deployment successful!"
  # Dọn dẹp image cũ không dùng
  docker image prune -f >> "$LOG_FILE" 2>&1
  log "🧹 Cleaned up unused images"
else
  log "❌ Deployment failed! Rolling back..."
  # Rollback về image cũ
  if [ "$CURRENT_IMAGE" != "none" ]; then
    docker compose \
      -f "$COMPOSE_FILE" \
      -f "$COMPOSE_PROD_FILE" \
      up -d --no-deps --force-recreate api
    log "⏪ Rolled back to: $CURRENT_IMAGE"
  fi
  exit 1
fi

log "========================================="
log "✅ Deploy completed at $(date)"
log "========================================="
```

```bash
# Cấp quyền thực thi
chmod +x /opt/myapp/scripts/deploy.sh
```

### 5.5.3. `docker-compose.prod.yml` hoàn chỉnh

```yaml
# File: docker-compose.prod.yml
# Dùng override lên docker-compose.yml

version: "3.9"

services:
  api:
    image: ghcr.io/yourusername/myapp-api:${IMAGE_TAG:-latest}
    build:
      target: production
    container_name: myapp-api
    restart: always
    env_file:
      - .env                          # Đọc từ file .env trên server
    ports: []                         # Không expose ra ngoài — chỉ qua Nginx
    volumes:
      - uploads:/app/uploads          # Persist user uploads
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:3000/health/ping || exit 1"]
      interval: 30s
      timeout: 10s
      start_period: 40s
      retries: 3
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          memory: 256M

  postgres:
    image: postgres:16-alpine
    container_name: myapp-postgres
    restart: always
    env_file:
      - .env
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports: []                         # Không expose ra ngoài
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./backups:/backups            # Mount thư mục backup
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
    logging:
      driver: "json-file"
      options:
        max-size: "5m"
        max-file: "3"

  redis:
    image: redis:7-alpine
    container_name: myapp-redis
    restart: always
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
      --save 900 1
      --save 300 10
      --appendonly yes
    ports: []
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  nginx:
    image: nginx:1.25-alpine
    container_name: myapp-nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - /etc/letsencrypt:/etc/nginx/ssl:ro
      - nginx-logs:/var/log/nginx
      - uploads:/var/www/uploads:ro   # Nginx serve uploads trực tiếp
    depends_on:
      api:
        condition: service_healthy
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"

volumes:
  postgres-data:
    driver: local
  redis-data:
    driver: local
  nginx-logs:
    driver: local
  uploads:
    driver: local
```

---

## 5.6. Zero Downtime Deployment

### 5.6.1. Vấn đề downtime khi deploy thông thường

```
Deploy thông thường:

[Container cũ đang chạy] → docker stop → docker rm → docker run [Container mới]
         ↑                      ↑               ↑              ↑
   Đang phục vụ          Dừng lại         Xóa đi       Khởi động (30s)
       user               ← Downtime 30-60 giây →
```

Với ứng dụng production có người dùng, downtime 30-60 giây là không chấp nhận được.

### 5.6.2. Blue/Green Deployment

**Blue/Green** là chiến lược chạy hai môi trường song song — Blue (đang chạy) và Green (phiên bản mới) — sau đó switch traffic từ Blue sang Green.

```
Trước deploy:
Internet → Nginx → [Blue: v1.0] (100% traffic)
                   [Green: empty]

Trong deploy:
Internet → Nginx → [Blue: v1.0] (100% traffic)
                   [Green: v1.1] ← đang warm up

Sau deploy:
Internet → Nginx → [Blue: v1.0] (0% — standby, sẵn sàng rollback)
                   [Green: v1.1] (100% traffic)

Sau khi confirm OK:
Internet → Nginx → [Green: v1.1] (100% traffic)
                   [Blue: cleanup]
```

**Triển khai Blue/Green với Docker Compose:**

```bash
#!/bin/bash
# File: scripts/deploy-blue-green.sh

set -euo pipefail

APP_DIR="/opt/myapp"
HEALTH_URL="http://localhost"
NEW_IMAGE="${1:-ghcr.io/yourusername/myapp-api:latest}"

log() { echo "[$(date '+%H:%M:%S')] $1"; }

# Xác định màu hiện tại
CURRENT_COLOR=$(docker ps --filter "name=myapp-api" \
  --format "{{.Names}}" | grep -oP '(?<=myapp-api-).*' || echo "blue")

if [ "$CURRENT_COLOR" = "blue" ]; then
  NEW_COLOR="green"
  CURRENT_PORT=3000
  NEW_PORT=3001
else
  NEW_COLOR="blue"
  CURRENT_PORT=3001
  NEW_PORT=3000
fi

log "🎨 Current: $CURRENT_COLOR (port $CURRENT_PORT)"
log "🆕 New: $NEW_COLOR (port $NEW_PORT)"

cd "$APP_DIR"

# Khởi động container màu mới
log "🚀 Starting $NEW_COLOR container..."
docker run -d \
  --name "myapp-api-$NEW_COLOR" \
  --network myapp_app-network \
  --env-file .env \
  --health-cmd "wget -qO- http://localhost:3000/health/ping || exit 1" \
  --health-interval 10s \
  --health-retries 5 \
  "$NEW_IMAGE"

# Chờ container mới healthy
log "⏳ Waiting for $NEW_COLOR to be healthy..."
for i in $(seq 1 18); do
  STATUS=$(docker inspect "myapp-api-$NEW_COLOR" \
    --format '{{.State.Health.Status}}' 2>/dev/null || echo "unknown")
  if [ "$STATUS" = "healthy" ]; then
    log "✅ $NEW_COLOR is healthy"
    break
  fi
  if [ "$STATUS" = "unhealthy" ]; then
    log "❌ $NEW_COLOR is unhealthy, rolling back..."
    docker rm -f "myapp-api-$NEW_COLOR"
    exit 1
  fi
  log "   Status: $STATUS (attempt $i/18)..."
  sleep 10
done

# Switch traffic: update Nginx upstream
log "🔀 Switching traffic to $NEW_COLOR..."
sed -i "s/server myapp-api-$CURRENT_COLOR:/server myapp-api-$NEW_COLOR:/" \
  "$APP_DIR/nginx/nginx.conf"
docker exec myapp-nginx nginx -s reload

# Verify traffic đang đến container mới
sleep 5
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" "$HEALTH_URL/health/ping")
if [ "$HTTP_CODE" != "200" ]; then
  log "❌ Traffic switch failed! Rolling back..."
  sed -i "s/server myapp-api-$NEW_COLOR:/server myapp-api-$CURRENT_COLOR:/" \
    "$APP_DIR/nginx/nginx.conf"
  docker exec myapp-nginx nginx -s reload
  docker rm -f "myapp-api-$NEW_COLOR"
  exit 1
fi

# Xóa container cũ
log "🗑️  Removing old $CURRENT_COLOR container..."
docker rm -f "myapp-api-$CURRENT_COLOR"

log "🎉 Blue/Green deployment complete! Running: $NEW_COLOR"
```

### 5.6.3. Rolling Deployment với Docker Compose

Cách đơn giản hơn — phù hợp cho single-server setup:

```bash
#!/bin/bash
# File: scripts/deploy-rolling.sh
# Zero downtime cho single server bằng cách scale

set -euo pipefail

APP_DIR="/opt/myapp"
HEALTH_URL="http://localhost:3000/health/ping"

log() { echo "[$(date '+%H:%M:%S')] $1"; }

cd "$APP_DIR"

log "📥 Pulling new image..."
docker compose -f docker-compose.yml -f docker-compose.prod.yml pull api

log "🔄 Scaling up to 2 instances..."
docker compose \
  -f docker-compose.yml \
  -f docker-compose.prod.yml \
  up -d --no-deps --scale api=2 --no-recreate api

# Chờ instance mới healthy
log "⏳ Waiting for new instance..."
sleep 20

# Kiểm tra health qua Nginx
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" "$HEALTH_URL")
if [ "$HTTP_CODE" != "200" ]; then
  log "❌ New instance unhealthy! Rolling back scale..."
  docker compose \
    -f docker-compose.yml \
    -f docker-compose.prod.yml \
    up -d --no-deps --scale api=1 --no-recreate api
  exit 1
fi

log "✅ New instance healthy. Removing old instance..."
docker compose \
  -f docker-compose.yml \
  -f docker-compose.prod.yml \
  up -d --no-deps --scale api=1 --force-recreate api

log "🧹 Cleanup..."
docker image prune -f

log "🎉 Rolling deployment complete!"
```

### 5.6.4. Database Migration và Zero Downtime

Migration database là phần khó nhất khi deploy không có downtime. Nguyên tắc:

```
Nguyên tắc: Migration phải TƯƠNG THÍCH NGƯỢC (Backward Compatible)

❌ Sai — Rename column trực tiếp:
  ALTER TABLE users RENAME COLUMN name TO full_name;
  → App cũ đang chạy dùng "name" sẽ crash

✅ Đúng — Quy trình 3 bước:
  Bước 1 (Deploy 1): ADD COLUMN full_name
    → App cũ vẫn dùng "name", app mới ghi vào cả hai
  Bước 2 (Deploy 2): App mới chỉ dùng "full_name"
  Bước 3 (Deploy 3): DROP COLUMN name
    → Đã an toàn vì không còn code nào dùng "name"
```

```typescript
// src/database/migrations/1700000001-AddFullNameColumn.ts
import { MigrationInterface, QueryRunner, TableColumn } from 'typeorm';

export class AddFullNameColumn1700000001 implements MigrationInterface {
  name = 'AddFullNameColumn1700000001';

  public async up(queryRunner: QueryRunner): Promise<void> {
    // Thêm cột mới — NOT NULL với DEFAULT để không phá app cũ
    await queryRunner.addColumn(
      'users',
      new TableColumn({
        name: 'full_name',
        type: 'varchar',
        length: '255',
        isNullable: true,        // Nullable để app cũ vẫn insert được
      }),
    );

    // Copy dữ liệu từ cột cũ sang cột mới
    await queryRunner.query(
      `UPDATE users SET full_name = name WHERE full_name IS NULL`,
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropColumn('users', 'full_name');
  }
}
```

---

## 5.7. Rollback

### 5.7.1. Chiến lược rollback

**Khi nào cần rollback?**
- Health check fail sau deploy.
- Error rate tăng đột biến sau deploy.
- Có bug nghiêm trọng được báo cáo ngay sau release.

**Các cấp độ rollback:**

```
Cấp 1: Rollback container (nhanh nhất — dưới 5 phút)
  → docker run image-cũ
  → Không rollback database

Cấp 2: Rollback container + revert migration (phức tạp hơn)
  → docker run image-cũ
  → typeorm migration:revert
  → Chỉ khả thi nếu migration là backward compatible

Cấp 3: Restore toàn bộ từ backup (chậm nhất)
  → Restore database backup
  → Chạy lại image cũ
  → Dùng khi dữ liệu bị corrupt
```

### 5.7.2. Script rollback tự động

```bash
#!/bin/bash
# File: scripts/rollback.sh
# Cách dùng: ./rollback.sh sha-abc1234
#            ./rollback.sh 1.2.3

set -euo pipefail

ROLLBACK_TAG="${1:-}"
APP_DIR="/opt/myapp"
REGISTRY="ghcr.io/yourusername/myapp-api"

log() { echo "[$(date '+%H:%M:%S')] $1"; }

if [ -z "$ROLLBACK_TAG" ]; then
  echo "Usage: $0 <image-tag>"
  echo "Available tags:"
  docker images "$REGISTRY" --format "{{.Tag}}\t{{.CreatedAt}}" | head -10
  exit 1
fi

ROLLBACK_IMAGE="$REGISTRY:$ROLLBACK_TAG"
log "⏪ Rolling back to: $ROLLBACK_IMAGE"

# Kiểm tra image có tồn tại không (local hoặc trên registry)
if ! docker image inspect "$ROLLBACK_IMAGE" &>/dev/null; then
  log "Image not found locally, pulling..."
  docker pull "$ROLLBACK_IMAGE" || {
    log "❌ Cannot find image: $ROLLBACK_IMAGE"
    exit 1
  }
fi

cd "$APP_DIR"

# Lưu image hiện tại để reference
CURRENT_IMAGE=$(docker inspect myapp-api \
  --format '{{.Config.Image}}' 2>/dev/null || echo "unknown")
log "📌 Current image: $CURRENT_IMAGE"

# Ghi override image vào file tạm
export IMAGE_TAG="$ROLLBACK_TAG"

# Restart container với image cũ
log "🔄 Restarting with rollback image..."
IMAGE_TAG="$ROLLBACK_TAG" docker compose \
  -f docker-compose.yml \
  -f docker-compose.prod.yml \
  up -d --no-deps --force-recreate api

# Chờ healthy
log "⏳ Checking health..."
for i in $(seq 1 12); do
  HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
    http://localhost:3000/health/ping 2>/dev/null || echo "000")
  if [ "$HTTP_CODE" = "200" ]; then
    log "✅ Rollback successful! Running: $ROLLBACK_TAG"
    
    # Ghi log sự kiện rollback
    echo "$(date): ROLLBACK from $CURRENT_IMAGE to $ROLLBACK_IMAGE" \
      >> "$APP_DIR/logs/deploy.log"
    exit 0
  fi
  log "   Attempt $i/12: HTTP $HTTP_CODE..."
  sleep 5
done

log "❌ Rollback also failed! Manual intervention required."
log "   Try: docker logs myapp-api"
exit 1
```

### 5.7.3. Đặt version tag để rollback dễ dàng

```bash
# Trong CI/CD — luôn tag image với Git SHA
GIT_SHA=$(git rev-parse --short HEAD)
VERSION="$(date +%Y.%m.%d)-$GIT_SHA"

docker build -t myapp:$VERSION -t myapp:latest .
docker push myapp:$VERSION
docker push myapp:latest

# Lưu lịch sử deploy trong file (trên server)
echo "$(date) | version=$VERSION | commit=$GIT_SHA | user=$(whoami)" \
  >> /opt/myapp/logs/deploy-history.log

# Xem lịch sử để biết tag cần rollback
cat /opt/myapp/logs/deploy-history.log
# 2024-01-15 10:30:00 | version=2024.01.15-abc1234 | commit=abc1234 | user=deploy
# 2024-01-16 14:00:00 | version=2024.01.16-def5678 | commit=def5678 | user=deploy ← có bug
# → Rollback về: 2024.01.15-abc1234
```

---

## 5.8. Health Check trong NestJS

### 5.8.1. Tại sao cần Health Check?

Health check endpoint cho phép:
- **Docker** biết container có healthy không (để auto-restart).
- **Nginx** biết backend có sẵn sàng nhận request không.
- **CI/CD** verify deploy thành công hay thất bại.
- **Monitoring** phát hiện sự cố sớm.

### 5.8.2. Các loại Health Check

```
/health/ping  → Liveness check: App có đang chạy không? (đơn giản nhất)
/health/ready → Readiness check: App có sẵn sàng nhận traffic không?
               (kết nối DB, Redis... OK chưa?)
/health       → Tổng hợp: trả về chi tiết từng dependency
```

### 5.8.3. Triển khai đầy đủ với `@nestjs/terminus`

```bash
npm install @nestjs/terminus @nestjs/axios axios
```

```typescript
// src/health/health.module.ts
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { TypeOrmModule } from '@nestjs/typeorm';
import { HttpModule } from '@nestjs/axios';
import { HealthController } from './health.controller';
import { RedisHealthIndicator } from './redis.health';

@Module({
  imports: [
    TerminusModule,
    HttpModule,
  ],
  controllers: [HealthController],
  providers: [RedisHealthIndicator],
})
export class HealthModule {}
```

```typescript
// src/health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import {
  HealthCheck,
  HealthCheckService,
  TypeOrmHealthIndicator,
  DiskHealthIndicator,
  MemoryHealthIndicator,
  HttpHealthIndicator,
} from '@nestjs/terminus';
import { RedisHealthIndicator } from './redis.health';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
    private disk: DiskHealthIndicator,
    private memory: MemoryHealthIndicator,
    private redis: RedisHealthIndicator,
  ) {}

  // Liveness probe — app có đang chạy không?
  // Nhẹ nhất, không kiểm tra dependencies
  @Get('ping')
  ping() {
    return {
      status: 'ok',
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
    };
  }

  // Readiness probe — app có sẵn sàng nhận traffic không?
  // Kiểm tra tất cả dependencies
  @Get()
  @HealthCheck()
  async check() {
    return this.health.check([
      // Kiểm tra database
      () => this.db.pingCheck('database', { timeout: 3000 }),

      // Kiểm tra Redis
      () => this.redis.isHealthy('redis'),

      // Kiểm tra disk không bị full (>90% = unhealthy)
      () => this.disk.checkStorage('disk', {
        path: '/',
        thresholdPercent: 0.9,
      }),

      // Kiểm tra RAM heap không quá 500MB
      () => this.memory.checkHeap('memory_heap', 500 * 1024 * 1024),

      // Kiểm tra RSS không quá 1GB
      () => this.memory.checkRSS('memory_rss', 1024 * 1024 * 1024),
    ]);
  }
}
```

```typescript
// src/health/redis.health.ts
import { Injectable } from '@nestjs/common';
import {
  HealthIndicator,
  HealthIndicatorResult,
  HealthCheckError,
} from '@nestjs/terminus';
import { InjectRedis } from '@nestjs-modules/ioredis';
import Redis from 'ioredis';

@Injectable()
export class RedisHealthIndicator extends HealthIndicator {
  constructor(@InjectRedis() private readonly redis: Redis) {
    super();
  }

  async isHealthy(key: string): Promise<HealthIndicatorResult> {
    try {
      const result = await this.redis.ping();
      const isHealthy = result === 'PONG';

      const data = this.getStatus(key, isHealthy, {
        message: isHealthy ? 'Redis is up' : 'Redis ping failed',
      });

      if (!isHealthy) {
        throw new HealthCheckError('Redis check failed', data);
      }

      return data;
    } catch (error) {
      const data = this.getStatus(key, false, {
        message: error.message,
      });
      throw new HealthCheckError('Redis check failed', data);
    }
  }
}
```

**Response của `/health` khi OK:**

```json
{
  "status": "ok",
  "info": {
    "database": { "status": "up" },
    "redis": { "status": "up", "message": "Redis is up" },
    "disk": { "status": "up" },
    "memory_heap": { "status": "up" },
    "memory_rss": { "status": "up" }
  },
  "error": {},
  "details": {
    "database": { "status": "up" },
    "redis": { "status": "up" },
    "disk": { "status": "up" },
    "memory_heap": { "status": "up" },
    "memory_rss": { "status": "up" }
  }
}
```

**Response khi database down:**

```json
{
  "status": "error",
  "info": {
    "redis": { "status": "up" }
  },
  "error": {
    "database": {
      "status": "down",
      "message": "Connection refused"
    }
  }
}
```

---

## 5.9. Backup Database

### 5.9.1. Script backup PostgreSQL tự động

```bash
#!/bin/bash
# File: /opt/myapp/scripts/backup-db.sh
# Cron: 0 2 * * * /opt/myapp/scripts/backup-db.sh

set -euo pipefail

# ─────────────────────────────────────────
# Cấu hình
# ─────────────────────────────────────────
BACKUP_DIR="/opt/myapp/backups"
RETENTION_DAYS=30
DATE=$(date +%Y%m%d_%H%M%S)

# Đọc từ .env
source /opt/myapp/.env
DB_CONTAINER="myapp-postgres"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"; }

mkdir -p "$BACKUP_DIR"

log "📦 Starting database backup..."

# Dump database từ container đang chạy
BACKUP_FILE="$BACKUP_DIR/myapp_${DATE}.sql.gz"

docker exec "$DB_CONTAINER" pg_dump \
  -U "$DB_USER" \
  -d "$DB_NAME" \
  --no-password \
  --clean \
  --if-exists \
  | gzip > "$BACKUP_FILE"

BACKUP_SIZE=$(du -sh "$BACKUP_FILE" | cut -f1)
log "✅ Backup created: $BACKUP_FILE ($BACKUP_SIZE)"

# Xóa backup cũ hơn RETENTION_DAYS ngày
log "🗑️  Removing backups older than $RETENTION_DAYS days..."
find "$BACKUP_DIR" -name "*.sql.gz" -mtime +"$RETENTION_DAYS" -delete

# Kiểm tra còn bao nhiêu backup
BACKUP_COUNT=$(ls "$BACKUP_DIR"/*.sql.gz 2>/dev/null | wc -l)
TOTAL_SIZE=$(du -sh "$BACKUP_DIR" | cut -f1)
log "📊 Total backups: $BACKUP_COUNT files ($TOTAL_SIZE)"

# Thêm vào crontab:
# crontab -e
# 0 2 * * * /opt/myapp/scripts/backup-db.sh >> /opt/myapp/logs/backup.log 2>&1
```

---

## 5.10. Best Practices

### 5.10.1. Nguyên tắc chung

```
✅ Mỗi deploy phải có thể rollback
✅ Deploy staging trước, production sau
✅ Test migration trên staging với dữ liệu gần giống production
✅ Backup trước mỗi migration quan trọng
✅ Đặt tag version rõ ràng (không chỉ dùng "latest")
✅ Monitor sau mỗi deploy ít nhất 15 phút
✅ Có runbook: tài liệu từng bước khi có sự cố
```

### 5.10.2. Checklist trước khi deploy production

```markdown
## Pre-deploy Checklist

### Code
- [ ] PR đã được review và approved
- [ ] CI pipeline pass (lint + test + build)
- [ ] Staging đã deploy và QA verify

### Database
- [ ] Migration đã test trên staging
- [ ] Migration backward compatible
- [ ] Backup database production vừa xong

### Environment
- [ ] Biến môi trường mới đã được thêm vào production
- [ ] Secret mới đã được tạo trong secret manager

### Deploy
- [ ] Đã thông báo team về thời điểm deploy
- [ ] Tránh deploy vào: giờ cao điểm, cuối ngày thứ Sáu, trước holiday
- [ ] Người deploy phải có mặt trong 30 phút sau deploy

### Post-deploy
- [ ] Health check pass
- [ ] Kiểm tra error rate trong monitoring
- [ ] Smoke test tính năng vừa deploy
- [ ] Thông báo team deploy thành công
```

### 5.10.3. Không deploy vào giờ cao điểm

```
✅ Thời điểm tốt để deploy:
   Sáng sớm (6-8h): traffic thấp, còn cả ngày để fix nếu có vấn đề
   Trưa (12-13h): tương đối thấp
   
❌ Thời điểm không nên deploy:
   Tối thứ Sáu: không ai theo dõi cuối tuần
   Giờ cao điểm (8-11h, 19-22h): nhiều user bị ảnh hưởng nếu có lỗi
   Trước ngày lễ lớn
```

---

## Tóm Tắt Chương 5

| Khái niệm | Mô tả |
|---|---|
| **Development** | Môi trường local, hot reload, dữ liệu fake |
| **Staging** | Giống production, test trước khi release |
| **Production** | Môi trường thật, bảo mật tối đa |
| **Env vars** | Cấu hình qua biến môi trường, không hard-code |
| **Build process** | TypeScript → JS → Docker Image → Registry |
| **Deploy flow** | Pull image → Migrate DB → Swap container → Health check |
| **Zero downtime** | Blue/Green hoặc Rolling deployment |
| **Rollback** | Chạy lại image cũ khi phiên bản mới có lỗi |
| **Health check** | Endpoint `/health` để xác nhận app hoạt động |
| **Backup** | Backup DB tự động hàng ngày, giữ 30 ngày |

---

> **Chương tiếp theo:** [Chương 6 — Cloud](./chapter-06-cloud.md)  
> Bạn đã biết cách deploy lên VPS. Tiếp theo, chúng ta sẽ tìm hiểu về hạ tầng cloud — VPS, Object Storage, CDN, DNS — những thành phần cơ bản mà mọi backend developer cần nắm khi làm việc với môi trường production thực tế.
