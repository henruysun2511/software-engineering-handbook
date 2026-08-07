# Chương 3: CI/CD

> **Mức độ quan trọng:** ⭐⭐⭐⭐⭐  
> **Đối tượng:** Backend Developer (NestJS/Express), trình độ Intern → Junior  
> **Mục tiêu chương:** Hiểu CI/CD từ nguyên lý đến xây dựng pipeline GitHub Actions hoàn chỉnh — tự động test, build Docker image, push lên Registry và deploy lên VPS khi push code.

---

## 3.1. CI Là Gì? (Continuous Integration)

### 3.1.1. Vấn đề trước khi có CI

Trong quy trình phát triển truyền thống, các developer làm việc độc lập trong thời gian dài — có thể vài ngày, vài tuần — rồi mới merge code vào nhánh chính (main). Hệ quả là:

```
Dev A làm feature A (2 tuần)  ─┐
Dev B làm feature B (2 tuần)  ─┤→ Merge vào main → 💥 Conflict khổng lồ
Dev C làm feature C (2 tuần)  ─┘    Bug xuất hiện không biết từ đâu
                                     Mất ngày debug và fix conflict
```

Vấn đề này được gọi là **"Integration Hell"** (địa ngục tích hợp).

### 3.1.2. CI là gì?

**Continuous Integration (CI)** là thực hành tích hợp code thường xuyên (ít nhất mỗi ngày) vào nhánh chính, kết hợp với quy trình **tự động build và test** mỗi khi có code mới được push lên.

```
Mỗi lần push code
       ↓
CI Server tự động:
  1. Pull code mới nhất
  2. Cài dependencies
  3. Chạy linter (kiểm tra code style)
  4. Chạy unit tests
  5. Chạy integration tests
  6. Build ứng dụng
       ↓
Kết quả: ✅ Pass → Developer biết code không phá vỡ gì
         ❌ Fail → Developer biết ngay, sửa ngay khi code còn "nóng"
```

### 3.1.3. Lợi ích của CI

| Lợi ích | Giải thích |
|---|---|
| **Phát hiện lỗi sớm** | Lỗi được tìm thấy ngay sau khi push, khi code còn mới trong đầu |
| **Giảm Integration Hell** | Merge nhỏ và thường xuyên dễ xử lý hơn merge lớn định kỳ |
| **Code quality tự động** | Linter và test chạy tự động, không ai có thể bỏ qua |
| **Confidence** | Team tự tin hơn khi merge vì có test bảo vệ |
| **Tài liệu sống** | Test cases mô tả cách hệ thống nên hoạt động |

---

## 3.2. CD Là Gì? (Continuous Delivery & Continuous Deployment)

### 3.2.1. Continuous Delivery

**Continuous Delivery** mở rộng CI — sau khi build và test thành công, code sẵn sàng để deploy lên bất kỳ môi trường nào **bằng một thao tác** (thường là bấm nút).

```
Push code → CI pass → Artifact sẵn sàng → [Bấm nút] → Deploy
                                                ↑
                                         Con người quyết định khi nào deploy
```

### 3.2.2. Continuous Deployment

**Continuous Deployment** đi xa hơn một bước — **tự động deploy** lên production ngay khi CI pass, không cần con người can thiệp.

```
Push code → CI pass → Tự động deploy lên production
```

### 3.2.3. So sánh

```
           Continuous            Continuous            Continuous
           Integration           Delivery              Deployment
               │                     │                     │
  Build & Test ✓          Build & Test ✓        Build & Test ✓
                         Deploy-ready ✓         Deploy-ready ✓
                         Manual deploy ✓        Auto deploy  ✓
```

| | Continuous Integration | Continuous Delivery | Continuous Deployment |
|---|---|---|---|
| **Build tự động** | ✅ | ✅ | ✅ |
| **Test tự động** | ✅ | ✅ | ✅ |
| **Deploy tự động** | ❌ | ❌ (manual) | ✅ |
| **Phù hợp** | Mọi dự án | Cần approval trước release | SaaS, web app |

> 💡 **Thực tế tại doanh nghiệp:** Phần lớn startup và sản phẩm SaaS dùng **Continuous Deployment** cho môi trường `staging` và **Continuous Delivery** (yêu cầu approval) cho môi trường `production`.

---

## 3.3. CI/CD Pipeline

### 3.3.1. Khái niệm

**Pipeline** là chuỗi các bước tự động được thực hiện tuần tự (hoặc song song) từ khi push code đến khi deploy. Mỗi bước được gọi là một **stage** hoặc **job**.

### 3.3.2. Cấu trúc Pipeline điển hình

```
┌─────────────────────────────────────────────────────────────────┐
│                         CI/CD Pipeline                          │
│                                                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐    │
│  │  Source  │──►│   CI     │──►│  Build   │──►│  Deploy  │    │
│  │          │   │          │   │          │   │          │    │
│  │ Git push │   │ Lint     │   │ Docker   │   │ Staging  │    │
│  │          │   │ Test     │   │ build    │   │    ↓     │    │
│  │          │   │ Security │   │ Push to  │   │ Prod     │    │
│  │          │   │ scan     │   │ Registry │   │          │    │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘    │
│                                                                 │
│  Nếu bất kỳ stage nào fail → Pipeline dừng + Thông báo         │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3.3. Quy trình thực tế tại doanh nghiệp

```
Developer
    │
    │ git push origin feature/user-auth
    ▼
GitHub (nhận code)
    │
    │ trigger GitHub Actions workflow
    ▼
┌─────────────────────────────────────────┐
│         GitHub Actions Runner           │
│                                         │
│  Job 1: CI                             │
│    ├── Checkout code                   │
│    ├── Setup Node.js 20                │
│    ├── npm ci (cài deps)               │
│    ├── npm run lint                    │
│    ├── npm run test                    │
│    └── npm run test:e2e                │
│                                         │
│  Job 2: Build (chỉ chạy khi CI pass)  │
│    ├── Docker build                    │
│    ├── Docker push → GHCR             │
│    └── Tag: ghcr.io/user/api:sha-abc  │
│                                         │
│  Job 3: Deploy Staging (auto)          │
│    ├── SSH vào staging server          │
│    ├── docker pull (image mới)         │
│    ├── docker compose up -d            │
│    └── Health check                    │
│                                         │
│  Job 4: Deploy Production (manual)     │
│    ├── [Chờ approval]                 │
│    ├── SSH vào prod server             │
│    └── Tương tự staging               │
└─────────────────────────────────────────┘
    │
    ▼
Slack/Email notification: ✅ Deploy thành công
```

---

## 3.4. GitHub Actions

### 3.4.1. Khái niệm

**GitHub Actions** là nền tảng CI/CD tích hợp sẵn trong GitHub. Nó cho phép tự động hóa workflow trực tiếp trong repository, không cần server CI riêng.

**Ưu điểm:**
- Miễn phí cho public repo (và một số giới hạn cho private)
- Tích hợp sâu với GitHub (PRs, branches, releases, secrets)
- Hàng nghìn Actions có sẵn trên Marketplace
- Không cần cài đặt — chỉ cần tạo file YAML

### 3.4.2. Các khái niệm cốt lõi

```
Repository
    └── .github/
            └── workflows/
                    ├── ci.yml        ← Workflow 1
                    └── deploy.yml    ← Workflow 2
```

| Khái niệm | Giải thích |
|---|---|
| **Workflow** | File YAML định nghĩa toàn bộ quy trình tự động |
| **Event (trigger)** | Sự kiện kích hoạt workflow (push, PR, schedule...) |
| **Job** | Một nhóm steps chạy trên cùng một runner |
| **Step** | Một tác vụ đơn lẻ trong job (chạy lệnh hoặc dùng Action) |
| **Action** | Component tái sử dụng được (từ Marketplace hoặc tự viết) |
| **Runner** | Máy chủ chạy job (GitHub-hosted hoặc self-hosted) |
| **Secret** | Biến nhạy cảm lưu trong GitHub Settings |
| **Artifact** | File output của job (build files, reports...) |

### 3.4.3. Cú pháp cơ bản

```yaml
# .github/workflows/example.yml
name: Tên workflow

# Sự kiện kích hoạt
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'  # Chạy lúc 2:00 AM mỗi ngày
  workflow_dispatch:       # Cho phép chạy thủ công từ UI

# Biến môi trường dùng chung
env:
  NODE_VERSION: '20'
  REGISTRY: ghcr.io

# Định nghĩa jobs
jobs:
  job-name:
    name: Tên hiển thị
    runs-on: ubuntu-latest     # Runner (máy chủ)
    
    # Chiến lược matrix (test nhiều version)
    strategy:
      matrix:
        node-version: [18, 20]
    
    # Timeout
    timeout-minutes: 15
    
    # Job này chỉ chạy sau khi job khác pass
    needs: [other-job]
    
    # Điều kiện chạy job
    if: github.ref == 'refs/heads/main'
    
    # Permissions cần thiết
    permissions:
      contents: read
      packages: write
    
    # Các bước thực hiện
    steps:
      # Bước 1: Checkout code
      - name: Checkout code
        uses: actions/checkout@v4

      # Bước 2: Setup runtime
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      # Bước 3: Chạy lệnh shell
      - name: Install dependencies
        run: npm ci

      # Bước 4: Dùng secret
      - name: Deploy
        env:
          SSH_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
        run: |
          echo "$SSH_KEY" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh user@server "docker compose up -d"

      # Bước 5: Upload artifact
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()   # Chạy dù step trước fail
        with:
          name: test-results
          path: coverage/
```

---

## 3.5. Build — Tự Động Build Ứng Dụng

### 3.5.1. Build là gì trong context CI/CD?

**Build** là quá trình chuyển đổi source code thành artifact có thể deploy:

```
Source Code (TypeScript)
    ↓ npm run build
JavaScript (dist/)
    ↓ docker build
Docker Image
    ↓ docker push
Registry (GHCR/Docker Hub)
```

### 3.5.2. Tối ưu build với Cache

```yaml
# .github/workflows/build.yml (excerpt)

- name: Setup Node.js với cache
  uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'                    # Cache ~/.npm directory

- name: Cache node_modules
  uses: actions/cache@v4
  with:
    path: node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

- name: Setup Docker Buildx (hỗ trợ cache cho Docker build)
  uses: docker/setup-buildx-action@v3

- name: Cache Docker layers
  uses: actions/cache@v4
  with:
    path: /tmp/.buildx-cache
    key: ${{ runner.os }}-buildx-${{ github.sha }}
    restore-keys: |
      ${{ runner.os }}-buildx-

- name: Build Docker Image
  uses: docker/build-push-action@v5
  with:
    context: .
    push: false
    tags: myapp:latest
    cache-from: type=local,src=/tmp/.buildx-cache
    cache-to: type=local,dest=/tmp/.buildx-cache-new,mode=max

# Workaround: ngăn cache tăng kích thước vô hạn
- name: Move cache
  run: |
    rm -rf /tmp/.buildx-cache
    mv /tmp/.buildx-cache-new /tmp/.buildx-cache
```

---

## 3.6. Test — Tự Động Kiểm Thử

### 3.6.1. Các loại test trong NestJS

| Loại test | Mục đích | Tốc độ | Phạm vi |
|---|---|---|---|
| **Unit Test** | Test từng function/class độc lập | Nhanh | Hẹp |
| **Integration Test** | Test tương tác giữa các module | Trung bình | Trung bình |
| **E2E Test** | Test toàn bộ flow từ API đến DB | Chậm | Rộng |

### 3.6.2. Cấu hình test trong NestJS

```typescript
// src/users/users.service.spec.ts — Unit Test
import { Test, TestingModule } from '@nestjs/testing';
import { UsersService } from './users.service';
import { getRepositoryToken } from '@nestjs/typeorm';
import { User } from './user.entity';

describe('UsersService', () => {
  let service: UsersService;

  // Mock repository
  const mockUsersRepository = {
    findOne: jest.fn(),
    save: jest.fn(),
    create: jest.fn(),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: getRepositoryToken(User),
          useValue: mockUsersRepository,
        },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
  });

  afterEach(() => {
    jest.clearAllMocks();
  });

  describe('findOne', () => {
    it('should return a user when found', async () => {
      const mockUser = { id: 1, email: 'test@example.com' };
      mockUsersRepository.findOne.mockResolvedValue(mockUser);

      const result = await service.findOne(1);

      expect(result).toEqual(mockUser);
      expect(mockUsersRepository.findOne).toHaveBeenCalledWith({
        where: { id: 1 },
      });
    });

    it('should return null when user not found', async () => {
      mockUsersRepository.findOne.mockResolvedValue(null);

      const result = await service.findOne(999);

      expect(result).toBeNull();
    });
  });
});
```

```typescript
// test/users.e2e-spec.ts — E2E Test
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';
import { DataSource } from 'typeorm';

describe('UsersController (e2e)', () => {
  let app: INestApplication;
  let dataSource: DataSource;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();

    dataSource = moduleFixture.get<DataSource>(DataSource);
  });

  afterAll(async () => {
    await dataSource.destroy();
    await app.close();
  });

  beforeEach(async () => {
    // Dọn dẹp dữ liệu test trước mỗi test
    await dataSource.query('TRUNCATE TABLE users RESTART IDENTITY CASCADE');
  });

  describe('POST /api/users', () => {
    it('should create a new user', async () => {
      const createUserDto = {
        name: 'Alice',
        email: 'alice@example.com',
        password: 'Password123!',
      };

      const response = await request(app.getHttpServer())
        .post('/api/users')
        .send(createUserDto)
        .expect(201);

      expect(response.body).toMatchObject({
        id: expect.any(Number),
        name: 'Alice',
        email: 'alice@example.com',
      });
      // Không trả về password
      expect(response.body.password).toBeUndefined();
    });

    it('should return 409 when email already exists', async () => {
      // Tạo user trước
      await request(app.getHttpServer())
        .post('/api/users')
        .send({ name: 'Alice', email: 'alice@example.com', password: 'Password123!' });

      // Thử tạo lại với cùng email
      await request(app.getHttpServer())
        .post('/api/users')
        .send({ name: 'Bob', email: 'alice@example.com', password: 'Password456!' })
        .expect(409);
    });
  });

  describe('GET /api/users/:id', () => {
    it('should return user by id', async () => {
      // Tạo user
      const createRes = await request(app.getHttpServer())
        .post('/api/users')
        .send({ name: 'Alice', email: 'alice@example.com', password: 'Password123!' });

      const userId = createRes.body.id;

      // Lấy user
      const getRes = await request(app.getHttpServer())
        .get(`/api/users/${userId}`)
        .expect(200);

      expect(getRes.body.email).toBe('alice@example.com');
    });

    it('should return 404 when user not found', () => {
      return request(app.getHttpServer())
        .get('/api/users/99999')
        .expect(404);
    });
  });
});
```

```json
// package.json — Các scripts test
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:cov": "jest --coverage",
    "test:e2e": "jest --config ./test/jest-e2e.json",
    "test:ci": "jest --coverage --coverageReporters=lcov --forceExit"
  }
}
```

---

## 3.7. Deploy — Tự Động Triển Khai

### 3.7.1. Chiến lược deploy lên VPS qua SSH

```
GitHub Actions Runner
    │
    │ (kết nối SSH)
    ▼
VPS (Ubuntu)
    ├── docker pull image mới
    ├── docker compose up -d --no-deps --build api
    └── health check
```

### 3.7.2. Chuẩn bị trên VPS

```bash
# Trên VPS — chạy một lần để chuẩn bị
# 1. Cài Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# 2. Tạo user deploy riêng (không dùng root)
sudo useradd -m -s /bin/bash deploy
sudo usermod -aG docker deploy

# 3. Tạo thư mục ứng dụng
sudo mkdir -p /opt/myapp
sudo chown deploy:deploy /opt/myapp

# 4. Chuyển sang user deploy
sudo su - deploy

# 5. Tạo SSH key (cho GitHub Actions dùng để login)
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_actions -N ""

# 6. Thêm public key vào authorized_keys
cat ~/.ssh/github_actions.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# 7. In private key để copy vào GitHub Secrets
cat ~/.ssh/github_actions
# Mang toàn bộ output này → GitHub → Settings → Secrets → SSH_PRIVATE_KEY

# 8. Tạo file .env production trên server
cat > /opt/myapp/.env << 'EOF'
NODE_ENV=production
PORT=3000
DB_HOST=postgres
DB_PORT=5432
DB_NAME=myapp_prod
DB_USER=postgres
DB_PASSWORD=your-secure-password-here
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=your-redis-password-here
JWT_SECRET=your-jwt-secret-here
EOF
chmod 600 /opt/myapp/.env

# 9. Copy docker-compose files lên server
# (Lần đầu copy thủ công, sau đó CI/CD sẽ update)
scp docker-compose.yml docker-compose.prod.yml deploy@your-server:/opt/myapp/
scp -r nginx/ deploy@your-server:/opt/myapp/
```

---

## 3.8. Environment — Quản Lý Môi Trường

### 3.8.1. Ba môi trường chuẩn

```
Development  →  Staging  →  Production
     │              │             │
   Local        Giống prod     Thật sự
   machine       nhất có        chạy
               thể, test      cho user
               trước khi
               lên prod
```

| Môi trường | Mục đích | Dữ liệu | Tự động deploy |
|---|---|---|---|
| **Development** | Viết code hàng ngày | Fake/seed data | Không (local) |
| **Staging** | QA, UAT, demo | Clone từ prod (ẩn danh) | Có (tự động khi merge vào main) |
| **Production** | Phục vụ người dùng | Dữ liệu thật | Có (sau khi approve) |

### 3.8.2. GitHub Environments

GitHub cung cấp tính năng **Environments** để quản lý deploy theo môi trường với các tính năng:

- **Protection rules**: Yêu cầu approval trước khi deploy
- **Environment secrets**: Secret riêng cho từng môi trường
- **Deployment history**: Lịch sử deploy theo môi trường

**Cấu hình trên GitHub:**

```
Repository → Settings → Environments → New environment

Tạo 2 environments:
  1. "staging"    → Không cần approval
  2. "production" → Yêu cầu approval từ: [your-username]
```

---

## 3.9. Secret Management

### 3.9.1. Các loại Secret cần quản lý

```
SSH Keys          → Để GitHub Actions kết nối VPS
Docker Registry   → Để push/pull Docker image  
Database URL      → Credentials database
JWT Secret        → Key ký token
API Keys          → Third-party services
```

### 3.9.2. GitHub Secrets

GitHub cung cấp 3 cấp lưu Secret:

- **Repository secrets**: Chỉ dùng trong một repo cụ thể
- **Environment secrets**: Khác nhau theo môi trường (staging/production)
- **Organization secrets**: Dùng chung cho nhiều repo trong org

**Tạo Secret:**

```
GitHub Repository
  → Settings
  → Secrets and variables
  → Actions
  → New repository secret

Secrets cần tạo:
  SSH_PRIVATE_KEY      ← Private key để SSH vào server
  SSH_HOST_STAGING     ← IP hoặc domain staging server
  SSH_HOST_PRODUCTION  ← IP hoặc domain production server
  SSH_USER             ← Username (vd: deploy)
  GHCR_TOKEN           ← GitHub Personal Access Token (read:packages, write:packages)
  SLACK_WEBHOOK_URL    ← Để gửi thông báo (nếu cần)
```

**Dùng Secret trong workflow:**

```yaml
- name: Deploy to server
  env:
    # Tham chiếu đến secret
    SSH_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
    SERVER: ${{ secrets.SSH_HOST_STAGING }}
    USER: ${{ secrets.SSH_USER }}
  run: |
    # Ghi private key ra file
    mkdir -p ~/.ssh
    echo "$SSH_KEY" > ~/.ssh/id_rsa
    chmod 600 ~/.ssh/id_rsa
    
    # Thêm server vào known_hosts để bỏ qua xác nhận
    ssh-keyscan -H $SERVER >> ~/.ssh/known_hosts
    
    # SSH vào server và deploy
    ssh $USER@$SERVER "cd /opt/myapp && docker compose pull && docker compose up -d"
```

> ⚠️ **Quan trọng:** Secret không bao giờ xuất hiện trong log của GitHub Actions. Nếu bạn cố `echo $SECRET`, GitHub sẽ hiển thị `***`.

---

## 3.10. Workflow Hoàn Chỉnh — NestJS CI/CD Pipeline

Dưới đây là pipeline CI/CD đầy đủ cho dự án NestJS thực tế:

### 3.10.1. Cấu trúc Workflows

```
.github/
└── workflows/
    ├── ci.yml          ← Chạy khi có PR hoặc push vào nhánh bất kỳ
    └── deploy.yml      ← Chạy khi merge vào main/production
```

### 3.10.2. File `ci.yml` — Continuous Integration

```yaml
# Tên file: .github/workflows/ci.yml
# Mục đích: Kiểm tra code quality và chạy test tự động

name: CI — Lint, Test & Build

on:
  push:
    branches-ignore:
      - main          # main sẽ dùng deploy.yml
  pull_request:
    branches:
      - main
      - develop

# Hủy run cũ khi có commit mới trên cùng branch/PR
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

env:
  NODE_VERSION: '20'

jobs:
  # ─────────────────────────────────────────
  # JOB 1: Lint và Type Check
  # ─────────────────────────────────────────
  lint:
    name: Lint & Type Check
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint

      - name: Run TypeScript type check
        run: npx tsc --noEmit

  # ─────────────────────────────────────────
  # JOB 2: Unit Tests
  # ─────────────────────────────────────────
  unit-test:
    name: Unit Tests
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run unit tests với coverage
        run: npm run test:ci

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage-report
          path: coverage/
          retention-days: 7

      # Gửi coverage lên Codecov (nếu dùng)
      - name: Upload to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: false

  # ─────────────────────────────────────────
  # JOB 3: E2E Tests (cần Database + Redis)
  # ─────────────────────────────────────────
  e2e-test:
    name: E2E Tests
    runs-on: ubuntu-latest
    timeout-minutes: 20

    # Khai báo services (container) chạy cùng với job
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: myapp_test
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    env:
      NODE_ENV: test
      DB_HOST: localhost
      DB_PORT: 5432
      DB_NAME: myapp_test
      DB_USER: postgres
      DB_PASSWORD: postgres_test
      REDIS_HOST: localhost
      REDIS_PORT: 6379
      JWT_SECRET: test-jwt-secret-key

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run database migrations
        run: npm run migration:run

      - name: Run E2E tests
        run: npm run test:e2e

  # ─────────────────────────────────────────
  # JOB 4: Build Check (không push image)
  # ─────────────────────────────────────────
  build-check:
    name: Docker Build Check
    runs-on: ubuntu-latest
    timeout-minutes: 15
    needs: [lint, unit-test]  # Chỉ chạy khi lint và unit test pass

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build Docker image (không push)
        uses: docker/build-push-action@v5
        with:
          context: .
          push: false           # Không push, chỉ kiểm tra build thành công
          tags: myapp:test
          cache-from: type=gha  # GitHub Actions cache
          cache-to: type=gha,mode=max
```

### 3.10.3. File `deploy.yml` — Deploy Pipeline

```yaml
# Tên file: .github/workflows/deploy.yml
# Mục đích: Build image và deploy khi merge vào main

name: Deploy — Build & Deploy

on:
  push:
    branches:
      - main          # Trigger khi merge vào main

concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false  # Không hủy deploy đang chạy

env:
  NODE_VERSION: '20'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}  # username/repo-name

jobs:
  # ─────────────────────────────────────────
  # JOB 1: CI (chạy lại để đảm bảo)
  # ─────────────────────────────────────────
  ci:
    name: CI Checks
    runs-on: ubuntu-latest
    timeout-minutes: 20

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: myapp_test
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    env:
      NODE_ENV: test
      DB_HOST: localhost
      DB_PORT: 5432
      DB_NAME: myapp_test
      DB_USER: postgres
      DB_PASSWORD: postgres_test
      REDIS_HOST: localhost
      REDIS_PORT: 6379
      JWT_SECRET: test-jwt-secret-key

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - run: npm ci
      - run: npm run lint
      - run: npm run migration:run
      - run: npm run test:ci
      - run: npm run test:e2e

  # ─────────────────────────────────────────
  # JOB 2: Build và Push Docker Image
  # ─────────────────────────────────────────
  build-and-push:
    name: Build & Push Image
    runs-on: ubuntu-latest
    timeout-minutes: 20
    needs: [ci]    # Chỉ build khi CI pass

    permissions:
      contents: read
      packages: write   # Cần để push lên GHCR

    outputs:
      # Truyền image tag sang jobs tiếp theo
      image-tag: ${{ steps.meta.outputs.version }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Đăng nhập vào GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}  # Token tự động, không cần tạo

      - name: Tạo metadata (tags và labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            # Khi push vào main → tag là "latest"
            type=raw,value=latest,enable={{is_default_branch}}
            # Tag theo Git SHA rút gọn — để rollback
            type=sha,prefix=sha-,format=short
            # Tag theo ngày
            type=raw,value={{date 'YYYY.MM.DD'}},enable={{is_default_branch}}

      - name: Build và Push Docker Image
        uses: docker/build-push-action@v5
        with:
          context: .
          target: production      # Dùng stage production từ multi-stage Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ─────────────────────────────────────────
  # JOB 3: Deploy lên Staging
  # ─────────────────────────────────────────
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    timeout-minutes: 15
    needs: [build-and-push]
    environment:
      name: staging
      url: https://staging.myapp.com  # URL hiển thị trên GitHub

    steps:
      - name: Checkout code (để lấy docker-compose files)
        uses: actions/checkout@v4

      - name: Setup SSH key
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Add server to known hosts
        run: ssh-keyscan -H ${{ secrets.SSH_HOST_STAGING }} >> ~/.ssh/known_hosts

      - name: Copy docker-compose files lên server
        run: |
          scp docker-compose.yml \
              docker-compose.prod.yml \
              ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST_STAGING }}:/opt/myapp/

      - name: Deploy lên Staging
        env:
          IMAGE_TAG: ${{ needs.build-and-push.outputs.image-tag }}
        run: |
          ssh ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST_STAGING }} << EOF
            set -e
            cd /opt/myapp
            
            echo "🔄 Pulling new image..."
            # Đăng nhập GHCR trên server
            echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io \
              -u ${{ github.actor }} \
              --password-stdin
            
            # Pull image mới
            docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            
            echo "🚀 Deploying..."
            # Chỉ restart service api, không restart db/redis
            docker compose \
              -f docker-compose.yml \
              -f docker-compose.prod.yml \
              up -d --no-deps api nginx
            
            echo "⏳ Waiting for health check..."
            sleep 10
            
            echo "🏥 Health check..."
            for i in {1..10}; do
              STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/health/ping)
              if [ "$STATUS" = "200" ]; then
                echo "✅ Health check passed!"
                break
              fi
              echo "Attempt $i failed (status: $STATUS), retrying..."
              sleep 5
            done
            
            if [ "$STATUS" != "200" ]; then
              echo "❌ Health check failed after 10 attempts"
              exit 1
            fi
            
            # Dọn dẹp image cũ
            docker image prune -f
          EOF

      - name: Notify Slack — Staging deployed
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "${{ job.status == 'success' && '✅' || '❌' }} Staging deploy: ${{ job.status }}\nCommit: ${{ github.sha }}\nAuthor: ${{ github.actor }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

  # ─────────────────────────────────────────
  # JOB 4: Deploy lên Production (cần approve)
  # ─────────────────────────────────────────
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    timeout-minutes: 20
    needs: [deploy-staging]
    environment:
      name: production          # Environment này yêu cầu manual approval
      url: https://myapp.com

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup SSH
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY_PROD }}  # Key riêng cho prod

      - name: Add production server to known hosts
        run: ssh-keyscan -H ${{ secrets.SSH_HOST_PRODUCTION }} >> ~/.ssh/known_hosts

      - name: Copy files lên production
        run: |
          scp docker-compose.yml \
              docker-compose.prod.yml \
              ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST_PRODUCTION }}:/opt/myapp/

      - name: Deploy lên Production với Zero Downtime
        run: |
          ssh ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST_PRODUCTION }} << 'EOF'
            set -e
            cd /opt/myapp
            
            echo "🔄 Pulling new image..."
            echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io \
              -u ${{ github.actor }} \
              --password-stdin
            
            docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            
            echo "🚀 Zero-downtime deploy..."
            # Chạy container mới song song trước khi dừng cái cũ
            docker compose \
              -f docker-compose.yml \
              -f docker-compose.prod.yml \
              up -d --no-deps --scale api=2 api
            
            sleep 10
            
            # Health check container mới
            STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/health/ping)
            if [ "$STATUS" != "200" ]; then
              echo "❌ New container unhealthy, rolling back..."
              docker compose \
                -f docker-compose.yml \
                -f docker-compose.prod.yml \
                up -d --no-deps --scale api=1 api
              exit 1
            fi
            
            echo "✅ New container healthy, scaling down old..."
            docker compose \
              -f docker-compose.yml \
              -f docker-compose.prod.yml \
              up -d --no-deps --scale api=1 api
            
            # Dọn dẹp
            docker image prune -f
            
            echo "🎉 Production deploy completed!"
          EOF

      - name: Notify Slack — Production deployed
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "${{ job.status == 'success' && '🎉' || '🚨' }} Production deploy: ${{ job.status }}\nVersion: ${{ github.sha }}\nDeployed by: ${{ github.actor }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### 3.10.4. File `rollback.yml` — Rollback Khẩn Cấp

```yaml
# Tên file: .github/workflows/rollback.yml
# Mục đích: Rollback về version trước đó khi production bị lỗi

name: Rollback Production

on:
  workflow_dispatch:        # Chỉ chạy thủ công
    inputs:
      image_tag:
        description: 'Image tag để rollback (vd: sha-abc1234)'
        required: true
        type: string
      environment:
        description: 'Môi trường cần rollback'
        required: true
        type: choice
        options:
          - staging
          - production

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  rollback:
    name: Rollback to ${{ inputs.image_tag }}
    runs-on: ubuntu-latest
    timeout-minutes: 10
    environment: ${{ inputs.environment }}

    steps:
      - name: Setup SSH
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Rollback
        run: |
          # Chọn server dựa trên environment
          if [ "${{ inputs.environment }}" = "production" ]; then
            SERVER="${{ secrets.SSH_HOST_PRODUCTION }}"
          else
            SERVER="${{ secrets.SSH_HOST_STAGING }}"
          fi
          
          ssh-keyscan -H $SERVER >> ~/.ssh/known_hosts
          
          ssh ${{ secrets.SSH_USER }}@$SERVER << EOF
            set -e
            cd /opt/myapp
            
            ROLLBACK_TAG="${{ inputs.image_tag }}"
            FULL_IMAGE="${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:$ROLLBACK_TAG"
            
            echo "⏪ Rolling back to: $ROLLBACK_TAG"
            
            # Pull image cũ
            docker pull $FULL_IMAGE
            
            # Update file .env để dùng image cụ thể
            # (Hoặc dùng IMAGE_TAG env var trong docker-compose)
            export API_IMAGE=$FULL_IMAGE
            
            docker compose \
              -f docker-compose.yml \
              -f docker-compose.prod.yml \
              up -d --no-deps api
            
            sleep 10
            
            STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/health/ping)
            if [ "$STATUS" = "200" ]; then
              echo "✅ Rollback successful!"
            else
              echo "❌ Rollback failed! Status: $STATUS"
              exit 1
            fi
          EOF

      - name: Notify Slack
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "⏪ Rollback ${{ inputs.environment }} to ${{ inputs.image_tag }}: ${{ job.status }}\nTriggered by: ${{ github.actor }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 3.11. Artifact và Cache

### 3.11.1. Artifacts

**Artifacts** là file được tạo ra trong quá trình chạy workflow và cần giữ lại sau khi workflow kết thúc.

```yaml
# Lưu artifact
- name: Build application
  run: npm run build

- name: Upload build artifact
  uses: actions/upload-artifact@v4
  with:
    name: dist-files
    path: dist/
    retention-days: 5      # Giữ 5 ngày

# Download artifact trong job khác
- name: Download build artifact
  uses: actions/download-artifact@v4
  with:
    name: dist-files
    path: dist/
```

**Dùng artifact để chia sẻ giữa các jobs:**

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/

  test-e2e:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/
      - run: npm run test:e2e
```

### 3.11.2. Cache

**Cache** giúp tái sử dụng dữ liệu từ run trước, tránh download lại mỗi lần.

```yaml
# Cache node_modules
- name: Cache node_modules
  uses: actions/cache@v4
  id: npm-cache
  with:
    path: |
      node_modules
      ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

# Chỉ npm ci nếu cache miss
- name: Install dependencies
  if: steps.npm-cache.outputs.cache-hit != 'true'
  run: npm ci
```

---

## 3.12. Best Practices

### 3.12.1. Workflow Design

```yaml
# ✅ Đặt timeout để tránh job chạy mãi
timeout-minutes: 15

# ✅ Dùng concurrency để hủy run cũ
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

# ✅ Pin actions theo tag cụ thể (không dùng @main hoặc @latest)
uses: actions/checkout@v4         # ✅
uses: actions/checkout@main       # ❌ Có thể bị breaking change

# ✅ Chạy jobs song song khi có thể
jobs:
  lint:     # Chạy song song với unit-test
    ...
  unit-test:
    ...
  e2e-test:
    needs: [lint, unit-test]      # Chờ cả hai xong mới chạy
```

### 3.12.2. Security

```yaml
# ✅ Đặt permissions tối thiểu
permissions:
  contents: read         # Chỉ cho phép đọc
  packages: write        # Chỉ thêm khi cần push Docker image

# ✅ Không in secret ra log
run: |
  # ❌ Sai — secret bị in ra log (dù GitHub ẩn nhưng vẫn rủi ro)
  echo "Connecting to ${{ secrets.DB_URL }}"
  
  # ✅ Đúng — dùng env var
  echo "Connecting to database..."  # Không lộ thông tin

# ✅ Dùng GITHUB_TOKEN thay vì Personal Access Token khi có thể
password: ${{ secrets.GITHUB_TOKEN }}  # Tự hết hạn sau mỗi run
```

### 3.12.3. Tối ưu tốc độ

```yaml
# ✅ Cache dependencies
- uses: actions/setup-node@v4
  with:
    cache: 'npm'   # Setup Node + cache npm cùng một bước

# ✅ Chạy test song song với matrix
strategy:
  matrix:
    shard: [1, 2, 3, 4]   # Chia test thành 4 phần chạy song song
steps:
  - run: npm test -- --shard=${{ matrix.shard }}/4

# ✅ Chỉ chạy jobs cần thiết dựa trên file thay đổi
- uses: dorny/paths-filter@v3
  id: filter
  with:
    filters: |
      backend:
        - 'src/**'
        - 'package.json'
      docker:
        - 'Dockerfile'
        - 'docker-compose*.yml'

- name: Run backend tests
  if: steps.filter.outputs.backend == 'true'
  run: npm test
```

### 3.12.4. Observability trong Pipeline

```yaml
# ✅ Annotate thất bại để dễ debug
- name: Run tests
  run: npm run test:ci 2>&1 | tee test-output.txt
  continue-on-error: true

- name: Parse test results
  uses: dorny/test-reporter@v1
  if: always()
  with:
    name: Jest Tests
    path: 'junit-report.xml'
    reporter: jest-junit

# ✅ Luôn upload log khi fail
- name: Upload logs on failure
  uses: actions/upload-artifact@v4
  if: failure()
  with:
    name: failure-logs-${{ github.run_id }}
    path: |
      logs/
      test-output.txt
```

---

## 3.13. Quy Trình Thực Tế Đầy Đủ

Tổng kết lại toàn bộ quy trình CI/CD cho một dự án NestJS:

```
Developer gõ code trên máy local
         │
         │ git add . && git commit -m "feat: add user auth"
         │ git push origin feature/user-auth
         ▼
GitHub nhận push → trigger ci.yml
         │
         ├── Job: lint           → ESLint + TypeScript check
         ├── Job: unit-test      → Jest unit tests + coverage
         ├── Job: e2e-test       → Supertest với PostgreSQL + Redis thật
         └── Job: build-check    → Docker build (không push)
         │
         │ CI pass ✅
         │
         │ Developer tạo Pull Request
         │ Team review code
         │ Merge vào main
         ▼
GitHub trigger deploy.yml
         │
         ├── Job: ci             → Chạy lại tất cả tests
         │
         ├── Job: build-and-push
         │       ├── Build Docker image (multi-stage)
         │       └── Push lên GHCR với tags: latest, sha-abc1234, 2024.01.15
         │
         ├── Job: deploy-staging (tự động)
         │       ├── SSH vào staging server
         │       ├── docker pull image mới
         │       ├── docker compose up -d
         │       ├── Health check (retry 10 lần)
         │       └── Notify Slack
         │
         │ QA team test trên staging
         │ Approve deploy production
         │
         └── Job: deploy-production (sau khi approve)
                 ├── SSH vào production server
                 ├── Zero-downtime deploy (scale=2 → health check → scale=1)
                 ├── Health check
                 └── Notify Slack 🎉

Nếu có lỗi sau deploy:
         │
         │ Developer chạy rollback.yml thủ công
         │ Nhập image tag cần rollback (vd: sha-abc1234)
         ▼
         Rollback về version trước trong 5 phút
```

---

## Tóm Tắt Chương 3

| Khái niệm | Vai trò |
|---|---|
| **CI** | Tự động build + test mỗi khi push code |
| **CD** | Tự động hoặc bán tự động deploy sau CI |
| **Pipeline** | Chuỗi bước từ code đến production |
| **GitHub Actions** | Nền tảng CI/CD tích hợp trong GitHub |
| **Workflow** | File YAML định nghĩa pipeline |
| **Job** | Nhóm steps chạy trên cùng runner |
| **Secret** | Biến nhạy cảm lưu trong GitHub Settings |
| **Environment** | Môi trường deploy với protection rules |
| **Artifact** | File output được lưu giữa các jobs/runs |
| **Cache** | Tái sử dụng dữ liệu để tăng tốc pipeline |

---

> **Chương tiếp theo:** [Chương 4 — Reverse Proxy & Nginx](./chapter-04-reverse-proxy.md)  
> Ứng dụng NestJS của bạn đã được đóng gói và deploy tự động. Tiếp theo, chúng ta sẽ tìm hiểu cách đặt Nginx phía trước ứng dụng để xử lý HTTPS, routing và bảo mật.
