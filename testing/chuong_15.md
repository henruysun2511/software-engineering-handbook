# GIÁO TRÌNH KIỂM THỬ VÀ ĐẢM BẢO CHẤT LƯỢNG PHẦN MỀM

---

# CHƯƠNG 15: TESTING TRONG CI/CD

---

## 15.1 CI/CD là gì?

**CI (Continuous Integration):** Thực hành developer thường xuyên merge code vào nhánh chính. Mỗi merge kích hoạt tự động build và test.

**CD (Continuous Deployment):** Tự động deploy lên staging hoặc production sau khi CI pass.

**Lợi ích với Tester:**
- Phát hiện lỗi ngay sau commit — feedback nhanh
- Không phải chờ "bản build" cuối sprint
- Test chạy nhất quán, không phụ thuộc môi trường local
- Audit trail đầy đủ — biết commit nào gây lỗi

---

## 15.2 Testing trong Pipeline

### 15.2.1 Testing Pipeline chuẩn

```
Developer push code
         ↓
    [GitHub/GitLab]
         ↓
    Trigger CI Pipeline
         ↓
┌────────────────────────────────────────────────────────┐
│                    CI PIPELINE                          │
│                                                        │
│  Stage 1: Build                                        │
│    - Compile code                                      │
│    - Lint (ESLint, Pylint)                             │
│    - Type check (TypeScript)                           │
│    - Build Docker image                                │
│           ↓                                            │
│  Stage 2: Unit Tests                                   │
│    - Jest / pytest / JUnit                             │
│    - Code coverage check (≥80%)                       │
│    - Mutation testing (nếu cấu hình)                  │
│           ↓                                            │
│  Stage 3: Integration Tests                            │
│    - API tests với test database                       │
│    - Contract tests (Pact)                            │
│           ↓                                            │
│  Stage 4: Security Scan                               │
│    - SAST: Sonar, Snyk (scan code)                    │
│    - Dependency check (npm audit, safety)             │
│           ↓                                            │
│  Stage 5: Deploy to Staging                           │
│           ↓                                            │
│  Stage 6: E2E Tests (Playwright)                      │
│    - Smoke tests                                       │
│    - Critical path E2E                                 │
│           ↓                                            │
│  Stage 7: Performance Tests                           │
│    - k6 load test (nếu cấu hình)                     │
│           ↓                                            │
│  [Quality Gate] ← PASS ALL? → Deploy to Production    │
│                 ← FAIL?     → Notify + Block deploy   │
└────────────────────────────────────────────────────────┘
```

---

## 15.3 GitHub Actions — Triển khai thực tế

### 15.3.1 Workflow CI/CD hoàn chỉnh

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: "20"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ===== JOB 1: Lint và Type Check =====
  lint:
    name: Lint & Type Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint

      - name: Type check
        run: npm run type-check

  # ===== JOB 2: Unit Tests =====
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run unit tests with coverage
        run: npm run test:coverage
        env:
          NODE_ENV: test

      - name: Check coverage threshold
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | \
            node -e "const d=require('fs').readFileSync('/dev/stdin','utf8'); \
            const c=JSON.parse(d).total.lines.pct; \
            console.log(c)")
          echo "Coverage: $COVERAGE%"
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage $COVERAGE% is below threshold 80%"
            exit 1
          fi

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info

  # ===== JOB 3: Integration Tests =====
  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: unit-tests
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      redis:
        image: redis:7
        ports:
          - 6379:6379
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run database migrations
        run: npm run db:migrate
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb

      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
          NODE_ENV: test

  # ===== JOB 4: Security Scan =====
  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4

      - name: Run npm audit
        run: npm audit --audit-level=high
        # Fail nếu có lỗ hổng High hoặc Critical

      - name: Run Snyk security test
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

  # ===== JOB 5: Build và Push Docker Image =====
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests, security-scan]
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix={{branch}}-
            type=ref,event=branch

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ===== JOB 6: Deploy to Staging =====
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    environment: staging
    steps:
      - name: Deploy to staging server
        run: |
          echo "Deploying ${{ needs.build.outputs.image-tag }} to staging..."
          # SSH vào server hoặc gọi deployment API
          # kubectl set image deployment/app app=${{ needs.build.outputs.image-tag }}

  # ===== JOB 7: E2E Tests trên Staging =====
  e2e-tests:
    name: E2E Tests
    runs-on: ubuntu-latest
    needs: deploy-staging
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "npm"

      - name: Install Playwright
        run: |
          npm ci
          npx playwright install --with-deps chromium

      - name: Wait for staging to be ready
        run: |
          npx wait-on https://staging.example.com/api/health --timeout 60000

      - name: Run smoke tests
        run: npx playwright test --project=chromium tests/smoke/
        env:
          BASE_URL: https://staging.example.com

      - name: Run E2E tests
        run: npx playwright test --project=chromium
        env:
          BASE_URL: https://staging.example.com
          TEST_BUYER_EMAIL: ${{ secrets.TEST_BUYER_EMAIL }}
          TEST_BUYER_PASSWORD: ${{ secrets.TEST_BUYER_PASSWORD }}

      - name: Upload E2E test report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7

  # ===== JOB 8: API Tests trên Staging =====
  api-tests:
    name: API Tests (Newman)
    runs-on: ubuntu-latest
    needs: deploy-staging
    steps:
      - uses: actions/checkout@v4

      - name: Install Newman
        run: npm install -g newman newman-reporter-htmlextra

      - name: Run API tests
        run: |
          newman run postman/collection.json \
            --environment postman/staging.json \
            --reporters cli,htmlextra \
            --reporter-htmlextra-export reports/api-report.html \
            --bail  # dừng nếu có test fail

      - name: Upload API test report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: api-test-report
          path: reports/api-report.html

  # ===== JOB 9: Deploy to Production =====
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [e2e-tests, api-tests]
    if: github.ref == 'refs/heads/main'  # chỉ từ nhánh main
    environment:
      name: production
      url: https://example.com
    steps:
      - name: Deploy to production
        run: |
          echo "All tests passed! Deploying to production..."
          # kubectl set image deployment/app app=${{ needs.build.outputs.image-tag }}

      - name: Smoke test production
        run: |
          npx wait-on https://example.com/api/health --timeout 30000
          curl -f https://example.com/api/health | jq '.status == "ok"'

      - name: Notify Slack on success
        uses: rtCamp/action-slack-notify@v2
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
          SLACK_MESSAGE: "✅ Deployed to production successfully!"
          SLACK_COLOR: good
```

---

### 15.3.2 Quality Gate

```yaml
# Quality Gate — chặn deploy nếu không đạt tiêu chuẩn
quality-gate:
  name: Quality Gate Check
  runs-on: ubuntu-latest
  needs: [unit-tests, integration-tests, e2e-tests, api-tests]
  steps:
    - name: Check all jobs passed
      run: |
        echo "Unit tests: ${{ needs.unit-tests.result }}"
        echo "Integration tests: ${{ needs.integration-tests.result }}"
        echo "E2E tests: ${{ needs.e2e-tests.result }}"
        echo "API tests: ${{ needs.api-tests.result }}"

        if [[ "${{ needs.unit-tests.result }}" != "success" ]] || \
           [[ "${{ needs.integration-tests.result }}" != "success" ]] || \
           [[ "${{ needs.e2e-tests.result }}" != "success" ]] || \
           [[ "${{ needs.api-tests.result }}" != "success" ]]; then
          echo "❌ Quality Gate FAILED — deployment blocked"
          exit 1
        fi
        echo "✅ Quality Gate PASSED — ready for deployment"
```

---

## 15.4 Docker — Chạy Test trong Container

### 15.4.1 Tại sao dùng Docker cho Testing?

**Vấn đề:** "Test pass trên máy tôi nhưng fail trên CI" — do môi trường khác nhau (Node version, OS, dependencies).

**Giải pháp:** Container đảm bảo môi trường hoàn toàn giống nhau ở mọi nơi.

### 15.4.2 Dockerfile cho Test Environment

```dockerfile
# Dockerfile.test
FROM node:20-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Cài dependencies bao gồm devDependencies
RUN npm ci

# Copy source code
COPY . .

# Cài Playwright browsers
RUN npx playwright install --with-deps chromium

# Entrypoint
CMD ["npm", "run", "test"]
```

```yaml
# docker-compose.test.yml
version: "3.8"

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      DATABASE_URL: postgresql://testuser:testpass@postgres:5432/testdb
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started

  test:
    build:
      context: .
      dockerfile: Dockerfile.test
    environment:
      BASE_URL: http://app:3000
      DATABASE_URL: postgresql://testuser:testpass@postgres:5432/testdb
    depends_on:
      - app
    command: >
      sh -c "
        npx wait-on http://app:3000/api/health &&
        npm run test:e2e
      "
    volumes:
      - ./playwright-report:/app/playwright-report

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U testuser -d testdb"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
```

```bash
# Chạy toàn bộ test stack với Docker Compose
docker-compose -f docker-compose.test.yml up --build --abort-on-container-exit

# Lấy exit code
docker-compose -f docker-compose.test.yml ps -q test | xargs docker inspect -f '{{.State.ExitCode}}'
```

---

## 15.5 Test Report và Notification

### 15.5.1 Allure Report

**Allure** là framework tạo báo cáo đẹp cho nhiều loại test (Pytest, Jest, Playwright).

```bash
# Pytest + Allure
pip install allure-pytest

pytest tests/ --alluredir=allure-results

allure generate allure-results --clean -o allure-report
allure open allure-report  # mở trong browser
```

```typescript
// Playwright + Allure
npm install -D allure-playwright

// playwright.config.ts
reporter: [["allure-playwright"]],

// Trong test — thêm metadata
import { allure } from "allure-playwright";

test("Checkout thành công", async ({ page }) => {
    allure.label("feature", "Checkout");
    allure.label("severity", "critical");
    allure.description("Kiểm thử luồng checkout hoàn chỉnh với COD");

    await allure.step("Vào trang checkout", async () => {
        await page.goto("/checkout");
    });

    await allure.step("Điền thông tin giao hàng", async () => {
        await page.getByLabel("Địa chỉ").fill("123 Test Street");
    });
    // ...
});
```

### 15.5.2 Notification khi Test Fail

```yaml
# GitHub Actions — Slack notification khi fail
- name: Notify Slack on failure
  if: failure()
  uses: rtCamp/action-slack-notify@v2
  env:
    SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
    SLACK_COLOR: danger
    SLACK_TITLE: "❌ CI/CD Pipeline Failed"
    SLACK_MESSAGE: |
      Pipeline failed on branch: ${{ github.ref_name }}
      Triggered by: ${{ github.actor }}
      Commit: ${{ github.sha }}
      
      Failed job: ${{ github.job }}
      
      View details: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
```

---

## 15.6 Best Practices CI/CD Testing

```
✅ Test sắp xếp theo tốc độ:
   Unit (nhanh nhất) → Integration → E2E (chậm nhất)
   Fail fast: nếu unit test fail → không chạy E2E (tiết kiệm thời gian)

✅ Parallel jobs:
   Unit tests, Security scan, Lint chạy song song
   Giảm tổng thời gian pipeline từ 20 phút xuống 8 phút

✅ Cache dependencies:
   GitHub Actions cache node_modules
   Docker layer caching
   Giảm 60-70% thời gian install

✅ Test environment isolation:
   Mỗi PR có test environment riêng
   Không share database giữa các PR

✅ Fail fast và clear error messages:
   Khi test fail, message rõ ràng: "Login test failed: Expected 200, got 401"
   Link đến log đầy đủ

✅ Artifacts:
   Lưu test report, screenshots, videos khi fail
   Retention 7 ngày (đủ để debug, không tốn storage)

✅ Secrets management:
   Dùng GitHub Secrets cho credentials
   Không hardcode trong code hay config
   Different secrets cho staging vs production

✅ Branch protection:
   Bắt buộc CI pass mới được merge PR
   Required reviewers: 1-2 người
   Dismiss stale reviews khi có push mới
```
