# Chương 7: Monitoring

> **Mức độ quan trọng:** ⭐⭐⭐⭐  
> **Đối tượng:** Backend Developer (NestJS/Express), trình độ Intern → Junior  
> **Mục tiêu chương:** Hiểu và triển khai đầy đủ hệ thống monitoring cho ứng dụng NestJS — từ structured logging, metrics, health check đến alerting và log rotation — để vận hành production tự tin và phát hiện sự cố trước khi người dùng phản ánh.

---

## 7.1. Tại Sao Monitoring Quan Trọng?

### 7.1.1. Vấn đề khi không có monitoring

```
Không có monitoring:
  User phản ánh app bị lỗi
       ↓
  Developer SSH vào server
       ↓
  Tìm log trong mớ file hỗn độn
       ↓
  Không biết lỗi xảy ra từ khi nào
       ↓
  Không biết tần suất lỗi
       ↓
  Debug mò mẫm → mất hàng giờ

Có monitoring:
  Alert tự động khi error rate tăng
       ↓
  Developer nhận thông báo Slack
       ↓
  Mở dashboard → thấy ngay metrics bất thường
       ↓
  Xem structured log với filter → tìm root cause
       ↓
  Fix và deploy → theo dõi metrics ổn định trở lại
  → Mất 15-30 phút
```

### 7.1.2. Ba trụ cột của Observability

**Observability** (khả năng quan sát) là khả năng hiểu trạng thái bên trong hệ thống từ các output bên ngoài. Ba trụ cột:

```
┌──────────────────────────────────────────────────┐
│                  Observability                   │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │  LOGS    │  │ METRICS  │  │  TRACES  │       │
│  │          │  │          │  │          │       │
│  │ "Chuyện  │  │ "Hệ      │  │ "Request │       │
│  │ gì đã   │  │ thống    │  │ đi qua   │       │
│  │ xảy ra" │  │ đang như │  │ đâu"     │       │
│  │         │  │ thế nào" │  │          │       │
│  └──────────┘  └──────────┘  └──────────┘       │
└──────────────────────────────────────────────────┘
```

| Trụ cột | Trả lời câu hỏi | Ví dụ |
|---|---|---|
| **Logs** | Chuyện gì đã xảy ra? | `ERROR: Cannot connect to database` |
| **Metrics** | Hệ thống đang như thế nào? | `CPU: 85%, Error rate: 2%, p99 latency: 450ms` |
| **Traces** | Request đi qua đâu, chậm ở bước nào? | Request → Auth → DB query (slow) → Response |

---

## 7.2. Logging

### 7.2.1. Logging là gì và tại sao cần?

**Log** là bản ghi sự kiện xảy ra trong ứng dụng theo thứ tự thời gian. Log là công cụ debug quan trọng nhất trong production vì bạn không thể attach debugger vào production server.

### 7.2.2. Log Levels

```
FATAL   → Lỗi khiến app crash hoàn toàn
ERROR   → Lỗi cần xử lý ngay, ảnh hưởng user
WARN    → Cảnh báo, chưa phải lỗi nhưng cần chú ý
INFO    → Thông tin hoạt động bình thường
DEBUG   → Chi tiết để debug (chỉ dùng khi dev)
VERBOSE → Cực kỳ chi tiết (hiếm dùng)

Production: chỉ log WARN trở lên
Development: log từ DEBUG trở lên
```

### 7.2.3. Structured Logging — JSON logs

**Structured logging** là ghi log dưới dạng JSON thay vì text thuần. Điều này cho phép log được index, search và filter hiệu quả.

```
❌ Unstructured log (text thuần):
[2024-01-15 10:30:45] ERROR: User 123 login failed: invalid password from IP 1.2.3.4

✅ Structured log (JSON):
{
  "timestamp": "2024-01-15T10:30:45.123Z",
  "level": "error",
  "message": "Login failed",
  "context": "AuthService",
  "userId": 123,
  "ip": "1.2.3.4",
  "reason": "invalid_password",
  "requestId": "req-abc-123",
  "duration": 45
}
```

Với JSON log, bạn có thể dễ dàng query:
```
Tìm tất cả lỗi của user 123 hôm nay
Tìm tất cả request từ IP 1.2.3.4
Đếm số lần login fail trong 1 giờ qua
```

### 7.2.4. Triển khai Logging trong NestJS với Winston

```bash
npm install nest-winston winston winston-daily-rotate-file
```

```typescript
// src/logger/logger.config.ts
import { utilities as nestWinstonModuleUtilities } from 'nest-winston';
import * as winston from 'winston';
import 'winston-daily-rotate-file';

const { combine, timestamp, errors, json, printf, colorize } = winston.format;

// Format cho development — dễ đọc, có màu sắc
const devFormat = combine(
  colorize({ all: true }),
  timestamp({ format: 'HH:mm:ss' }),
  errors({ stack: true }),
  printf(({ timestamp, level, message, context, stack, ...meta }) => {
    const ctx = context ? `[${context}]` : '';
    const extra = Object.keys(meta).length ? ` ${JSON.stringify(meta)}` : '';
    const err = stack ? `\n${stack}` : '';
    return `${timestamp} ${level} ${ctx} ${message}${extra}${err}`;
  }),
);

// Format cho production — JSON để dễ parse
const prodFormat = combine(
  timestamp(),
  errors({ stack: true }),
  json(),
);

const isDev = process.env.NODE_ENV !== 'production';

export const winstonConfig: winston.LoggerOptions = {
  level: process.env.LOG_LEVEL || (isDev ? 'debug' : 'warn'),
  format: isDev ? devFormat : prodFormat,
  defaultMeta: {
    service: 'myapp-api',         // Tên service — hữu ích khi có nhiều service
    env: process.env.NODE_ENV,
  },
  transports: [
    // Console transport — luôn có
    new winston.transports.Console({
      silent: process.env.NODE_ENV === 'test', // Tắt log khi chạy test
    }),

    // File transport cho production — rotate mỗi ngày
    ...(isDev ? [] : [
      // Chỉ ghi ERROR vào file riêng để dễ theo dõi
      new winston.transports.DailyRotateFile({
        filename: 'logs/error-%DATE%.log',
        datePattern: 'YYYY-MM-DD',
        level: 'error',
        maxSize: '20m',     // Tối đa 20MB mỗi file
        maxFiles: '30d',    // Giữ 30 ngày
        zippedArchive: true, // Nén file cũ
      }),

      // Ghi tất cả log INFO trở lên vào file combined
      new winston.transports.DailyRotateFile({
        filename: 'logs/combined-%DATE%.log',
        datePattern: 'YYYY-MM-DD',
        level: 'info',
        maxSize: '20m',
        maxFiles: '14d',
        zippedArchive: true,
      }),
    ]),
  ],

  // Không crash app khi gặp uncaught exception
  exceptionHandlers: [
    new winston.transports.Console(),
    ...(isDev ? [] : [
      new winston.transports.DailyRotateFile({
        filename: 'logs/exceptions-%DATE%.log',
        datePattern: 'YYYY-MM-DD',
        maxFiles: '30d',
      }),
    ]),
  ],
};
```

```typescript
// src/app.module.ts — đăng ký Winston
import { Module } from '@nestjs/common';
import { WinstonModule } from 'nest-winston';
import { winstonConfig } from './logger/logger.config';

@Module({
  imports: [
    WinstonModule.forRoot(winstonConfig),
    // ... other modules
  ],
})
export class AppModule {}
```

```typescript
// src/main.ts — dùng Winston làm logger mặc định của NestJS
import { NestFactory } from '@nestjs/core';
import { WINSTON_MODULE_NEST_PROVIDER } from 'nest-winston';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    // Tắt logger mặc định của NestJS
    bufferLogs: true,
  });

  // Dùng Winston thay thế
  app.useLogger(app.get(WINSTON_MODULE_NEST_PROVIDER));

  await app.listen(3000);
}
bootstrap();
```

```typescript
// src/users/users.service.ts — Dùng logger trong service
import { Injectable, Logger, NotFoundException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './user.entity';

@Injectable()
export class UsersService {
  // Logger tự động gắn context = 'UsersService'
  private readonly logger = new Logger(UsersService.name);

  constructor(
    @InjectRepository(User)
    private readonly usersRepo: Repository<User>,
  ) {}

  async findOne(id: number): Promise<User> {
    this.logger.debug(`Finding user: ${id}`);

    const user = await this.usersRepo.findOne({ where: { id } });

    if (!user) {
      // Log WARN vì đây không phải lỗi hệ thống, chỉ là không tìm thấy
      this.logger.warn(`User not found`, { userId: id });
      throw new NotFoundException(`User #${id} not found`);
    }

    return user;
  }

  async updateProfile(id: number, dto: UpdateProfileDto): Promise<User> {
    const startTime = Date.now();

    try {
      const user = await this.findOne(id);
      Object.assign(user, dto);
      const updated = await this.usersRepo.save(user);

      this.logger.log('User profile updated', {
        userId: id,
        fields: Object.keys(dto),
        duration: Date.now() - startTime,
      });

      return updated;
    } catch (error) {
      this.logger.error('Failed to update user profile', {
        userId: id,
        error: error.message,
        stack: error.stack,
        duration: Date.now() - startTime,
      });
      throw error;
    }
  }
}
```

### 7.2.5. HTTP Request Logging Middleware

Tự động log mỗi request đến API:

```typescript
// src/common/middleware/logger.middleware.ts
import { Injectable, NestMiddleware, Logger } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { v4 as uuidv4 } from 'uuid';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  private readonly logger = new Logger('HTTP');

  use(req: Request, res: Response, next: NextFunction) {
    const startTime = Date.now();
    const requestId = uuidv4();

    // Gắn requestId vào request để dùng trong toàn bộ request lifecycle
    req['requestId'] = requestId;

    // Log khi request kết thúc
    res.on('finish', () => {
      const duration = Date.now() - startTime;
      const logData = {
        requestId,
        method: req.method,
        url: req.originalUrl,
        statusCode: res.statusCode,
        duration,
        ip: req.ip,
        userAgent: req.headers['user-agent'],
        userId: req.user?.['id'],    // Nếu đã authenticate
        contentLength: res.get('content-length'),
      };

      // Phân loại log level theo status code
      if (res.statusCode >= 500) {
        this.logger.error('Request failed', logData);
      } else if (res.statusCode >= 400) {
        this.logger.warn('Request client error', logData);
      } else if (duration > 3000) {
        // Request quá chậm (> 3 giây)
        this.logger.warn('Slow request detected', logData);
      } else {
        this.logger.log('Request completed', logData);
      }
    });

    next();
  }
}
```

```typescript
// src/app.module.ts — đăng ký middleware
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';

@Module({})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .exclude(
        '/health',        // Không log health check (tránh noise)
        '/health/(.*)',
        '/metrics',       // Không log metrics scraping
      )
      .forRoutes('*');    // Apply cho tất cả routes còn lại
  }
}
```

### 7.2.6. Exception Filter — Log lỗi nhất quán

```typescript
// src/common/filters/all-exceptions.filter.ts
import {
  ExceptionFilter, Catch, ArgumentsHost,
  HttpException, HttpStatus, Logger,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name);

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const req = ctx.getRequest<Request>();
    const res = ctx.getResponse<Response>();

    let status: number;
    let message: string;
    let code: string;

    if (exception instanceof HttpException) {
      status = exception.getStatus();
      const response = exception.getResponse();
      message = typeof response === 'string'
        ? response
        : (response as any).message || exception.message;
      code = (response as any).error || 'HTTP_ERROR';
    } else if (exception instanceof Error) {
      // Lỗi không mong đợi — 500
      status = HttpStatus.INTERNAL_SERVER_ERROR;
      message = 'Internal server error';
      code = 'INTERNAL_ERROR';

      // Log đầy đủ stack trace cho lỗi 500
      this.logger.error('Unhandled exception', {
        requestId: req['requestId'],
        method: req.method,
        url: req.originalUrl,
        userId: req.user?.['id'],
        error: exception.message,
        stack: exception.stack,
      });
    } else {
      status = HttpStatus.INTERNAL_SERVER_ERROR;
      message = 'Unknown error';
      code = 'UNKNOWN_ERROR';
    }

    // Log lỗi 4xx ở mức WARN, 5xx ở mức ERROR
    if (status >= 500) {
      // Đã log ở trên với stack trace
    } else if (status >= 400) {
      this.logger.warn(`Client error: ${code}`, {
        requestId: req['requestId'],
        status,
        message,
        method: req.method,
        url: req.originalUrl,
        userId: req.user?.['id'],
      });
    }

    // Response cho client — KHÔNG trả về stack trace
    res.status(status).json({
      statusCode: status,
      message,
      code,
      timestamp: new Date().toISOString(),
      path: req.url,
      requestId: req['requestId'],    // Để client report lỗi
    });
  }
}
```

```typescript
// src/main.ts — đăng ký filter
app.useGlobalFilters(new AllExceptionsFilter());
```

---

## 7.3. Health Check

Đã được đề cập chi tiết trong **Chương 5.8**. Tóm tắt:

```typescript
// /health/ping  → Liveness: app có chạy không?
// /health       → Readiness: app + deps có healthy không?

@Get('ping')
ping() {
  return { status: 'ok', timestamp: new Date().toISOString() };
}

@Get()
@HealthCheck()
check() {
  return this.health.check([
    () => this.db.pingCheck('database'),
    () => this.redis.isHealthy('redis'),
    () => this.disk.checkStorage('disk', { path: '/', thresholdPercent: 0.9 }),
    () => this.memory.checkHeap('memory_heap', 500 * 1024 * 1024),
  ]);
}
```

---

## 7.4. Metrics

### 7.4.1. Metrics là gì?

**Metrics** là các số đo lường trạng thái và hiệu năng của hệ thống theo thời gian. Khác với log (ghi sự kiện rời rạc), metrics là dữ liệu liên tục cho phép vẽ biểu đồ và phát hiện xu hướng.

### 7.4.2. Các metrics quan trọng cho NestJS API

```
RED Method — 3 metrics cơ bản nhất:
  Rate     → Số request mỗi giây (req/s)
  Errors   → Tỉ lệ request lỗi (%)
  Duration → Thời gian xử lý request (latency)

USE Method — cho infrastructure:
  Utilization  → CPU, RAM đang dùng bao nhiêu %
  Saturation   → Hàng chờ, độ backlog
  Errors       → Lỗi phần cứng/network
```

### 7.4.3. Prometheus + Grafana Stack

**Prometheus** là hệ thống thu thập metrics mã nguồn mở. **Grafana** là công cụ visualize metrics thành dashboard.

```
NestJS API          Prometheus        Grafana
  /metrics    ←──── scrape ────→  visualize
  (expose)       (mỗi 15s)         (dashboard)
```

```bash
npm install @willsoto/nestjs-prometheus prom-client
```

```typescript
// src/metrics/metrics.module.ts
import { Module } from '@nestjs/common';
import {
  PrometheusModule,
  makeCounterProvider,
  makeHistogramProvider,
  makeGaugeProvider,
} from '@willsoto/nestjs-prometheus';

@Module({
  imports: [
    PrometheusModule.register({
      path: '/metrics',        // Endpoint Prometheus sẽ scrape
      defaultMetrics: {
        enabled: true,         // Tự động thu thập: CPU, RAM, event loop, GC
      },
    }),
  ],
  providers: [
    // Đếm số request
    makeCounterProvider({
      name: 'http_requests_total',
      help: 'Total number of HTTP requests',
      labelNames: ['method', 'route', 'status_code'],
    }),

    // Đo thời gian xử lý request (histogram có nhiều bucket)
    makeHistogramProvider({
      name: 'http_request_duration_seconds',
      help: 'HTTP request duration in seconds',
      labelNames: ['method', 'route', 'status_code'],
      buckets: [0.01, 0.05, 0.1, 0.3, 0.5, 1, 2, 5],
    }),

    // Đo độ trễ DB queries
    makeHistogramProvider({
      name: 'db_query_duration_seconds',
      help: 'Database query duration in seconds',
      labelNames: ['operation', 'table'],
      buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1],
    }),

    // Đếm user đang active
    makeGaugeProvider({
      name: 'active_users_total',
      help: 'Number of currently active users',
    }),
  ],
  exports: [PrometheusModule],
})
export class MetricsModule {}
```

```typescript
// src/common/interceptors/metrics.interceptor.ts
import {
  Injectable, NestInterceptor, ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';
import { InjectMetric } from '@willsoto/nestjs-prometheus';
import { Counter, Histogram } from 'prom-client';

@Injectable()
export class MetricsInterceptor implements NestInterceptor {
  constructor(
    @InjectMetric('http_requests_total')
    private readonly requestCounter: Counter<string>,

    @InjectMetric('http_request_duration_seconds')
    private readonly requestDuration: Histogram<string>,
  ) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const req = context.switchToHttp().getRequest();
    const startTime = Date.now();

    // Làm sạch route: /users/123 → /users/:id
    // Tránh cardinality explosion (quá nhiều label value)
    const route = req.route?.path || req.url.split('?')[0];

    return next.handle().pipe(
      tap({
        next: () => {
          const res = context.switchToHttp().getResponse();
          const duration = (Date.now() - startTime) / 1000;
          const labels = {
            method: req.method,
            route,
            status_code: String(res.statusCode),
          };

          this.requestCounter.inc(labels);
          this.requestDuration.observe(labels, duration);
        },
        error: (err) => {
          const duration = (Date.now() - startTime) / 1000;
          const statusCode = err.status || 500;
          const labels = {
            method: req.method,
            route,
            status_code: String(statusCode),
          };

          this.requestCounter.inc(labels);
          this.requestDuration.observe(labels, duration);
        },
      }),
    );
  }
}
```

### 7.4.4. Prometheus + Grafana với Docker Compose

```yaml
# Thêm vào docker-compose.yml

  prometheus:
    image: prom/prometheus:latest
    container_name: myapp-prometheus
    restart: unless-stopped
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'  # Giữ data 30 ngày
      - '--web.enable-lifecycle'
    ports:
      - "9090:9090"   # Chỉ mở trong dev; production: không expose ra ngoài
    networks:
      - app-network

  grafana:
    image: grafana/grafana:latest
    container_name: myapp-grafana
    restart: unless-stopped
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin}
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_SERVER_ROOT_URL: "https://grafana.myapp.com"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning:ro
    ports:
      - "3001:3000"   # Truy cập tại http://localhost:3001
    depends_on:
      - prometheus
    networks:
      - app-network

volumes:
  prometheus-data:
  grafana-data:
```

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s      # Scrape mỗi 15 giây
  evaluation_interval: 15s

scrape_configs:
  # Scrape NestJS API
  - job_name: 'nestjs-api'
    static_configs:
      - targets: ['api:3000']   # Tên service Docker
    metrics_path: '/metrics'

  # Scrape Prometheus chính nó (để monitor Prometheus)
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Node Exporter — thu thập system metrics (CPU, RAM, disk)
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']

  # PostgreSQL Exporter
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
```

**Các Grafana Dashboard hữu ích:**

```
Import từ grafana.com/dashboards:
  Node Exporter Full:   ID 1860  (system metrics)
  NestJS:               ID 11159 (app metrics)
  PostgreSQL:           ID 9628  (DB metrics)
  Redis:                ID 11835 (Redis metrics)
  Nginx:                ID 12708 (Nginx metrics)
```

---

## 7.5. Alerting

### 7.5.1. Alerting là gì?

**Alerting** là cơ chế tự động gửi thông báo khi metrics vượt ngưỡng bất thường — trước khi người dùng phàn nàn.

```
Không có Alerting:           Có Alerting:
User báo lỗi                 Prometheus phát hiện error rate tăng
→ Developer biết             → Prometheus gửi alert đến Alertmanager
→ Quá muộn                   → Alertmanager gửi Slack message
                             → Developer biết trước user
                             → Fix ngay khi vấn đề còn nhỏ
```

### 7.5.2. Alerting Rules trong Prometheus

```yaml
# monitoring/alert-rules.yml
groups:
  - name: nestjs-api-alerts
    rules:

      # Alert khi error rate > 5% trong 5 phút
      - alert: HighErrorRate
        expr: |
          rate(http_requests_total{status_code=~"5.."}[5m])
          /
          rate(http_requests_total[5m])
          > 0.05
        for: 2m          # Phải duy trì 2 phút mới fire (tránh false alarm)
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: >
            Error rate is {{ humanizePercentage $value }}
            (> 5%) for 2 minutes.
            Possible causes: database down, code bug.

      # Alert khi p95 latency > 2 giây
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            rate(http_request_duration_seconds_bucket[5m])
          ) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High API latency"
          description: "95th percentile latency is {{ $value }}s (> 2s)"

      # Alert khi API không available
      - alert: APIDown
        expr: up{job="nestjs-api"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "NestJS API is down"
          description: "API has been down for more than 1 minute"

      # Alert khi RAM > 85%
      - alert: HighMemoryUsage
        expr: |
          (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"
          description: "Memory usage is {{ humanizePercentage $value }} (> 85%)"

      # Alert khi disk > 80%
      - alert: DiskSpaceRunningLow
        expr: |
          (1 - node_filesystem_free_bytes / node_filesystem_size_bytes) > 0.80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Disk space running low"
          description: "Disk usage is {{ humanizePercentage $value }} on {{ $labels.mountpoint }}"

      # Alert khi CPU > 90% trong 10 phút
      - alert: HighCPUUsage
        expr: |
          100 - (avg by(instance) (
            rate(node_cpu_seconds_total{mode="idle"}[5m])
          ) * 100) > 90
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage"
          description: "CPU usage is {{ $value }}% (> 90%)"
```

### 7.5.3. Alertmanager — Gửi alert qua Slack

```yaml
# monitoring/alertmanager.yml
global:
  slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'severity']
  group_wait: 30s        # Chờ 30s để group alerts lại
  group_interval: 5m     # Gửi lại nếu còn firing sau 5 phút
  repeat_interval: 12h   # Nhắc lại mỗi 12h nếu chưa resolve
  receiver: 'slack-notifications'
  routes:
    # Critical alerts → gửi ngay, không group
    - match:
        severity: critical
      receiver: 'slack-critical'
      group_wait: 0s

receivers:
  - name: 'slack-notifications'
    slack_configs:
      - channel: '#alerts-prod'
        title: '{{ template "slack.default.title" . }}'
        text: >-
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Severity:* {{ .Labels.severity }}
          *Description:* {{ .Annotations.description }}
          *Fired at:* {{ .StartsAt.Format "2006-01-02 15:04:05" }}
          {{ end }}
        color: >-
          {{ if eq .Status "firing" }}
            {{ if eq (index .Alerts 0).Labels.severity "critical" }}danger{{ else }}warning{{ end }}
          {{ else }}good{{ end }}

  - name: 'slack-critical'
    slack_configs:
      - channel: '#alerts-critical'
        send_resolved: true
```

### 7.5.4. Simple Alerting Không Dùng Prometheus

Nếu chưa có Prometheus, có thể alert đơn giản từ ứng dụng:

```typescript
// src/common/services/alert.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { HttpService } from '@nestjs/axios';
import { firstValueFrom } from 'rxjs';

@Injectable()
export class AlertService {
  private readonly logger = new Logger(AlertService.name);

  constructor(
    private readonly http: HttpService,
    private readonly config: ConfigService,
  ) {}

  async sendSlackAlert(options: {
    title: string;
    message: string;
    severity: 'info' | 'warning' | 'critical';
    context?: Record<string, unknown>;
  }): Promise<void> {
    const webhookUrl = this.config.get<string>('SLACK_WEBHOOK_URL');
    if (!webhookUrl) return;

    const colorMap = {
      info: '#36a64f',      // Green
      warning: '#ff9900',   // Orange
      critical: '#ff0000',  // Red
    };

    const icon = {
      info: ':information_source:',
      warning: ':warning:',
      critical: ':rotating_light:',
    };

    const payload = {
      attachments: [
        {
          color: colorMap[options.severity],
          title: `${icon[options.severity]} ${options.title}`,
          text: options.message,
          fields: options.context
            ? Object.entries(options.context).map(([key, value]) => ({
                title: key,
                value: String(value),
                short: true,
              }))
            : [],
          footer: `${this.config.get('NODE_ENV')} | ${new Date().toISOString()}`,
        },
      ],
    };

    try {
      await firstValueFrom(this.http.post(webhookUrl, payload));
    } catch (error) {
      // Không throw — alert failure không nên crash app
      this.logger.error('Failed to send Slack alert', { error: error.message });
    }
  }
}
```

```typescript
// Dùng trong exception filter hoặc service
await this.alert.sendSlackAlert({
  title: '🚨 Unhandled Exception in Production',
  message: error.message,
  severity: 'critical',
  context: {
    url: req.url,
    method: req.method,
    userId: req.user?.id,
    requestId: req.requestId,
  },
});
```

---

## 7.6. Log Rotation

### 7.6.1. Vấn đề log không rotate

```
Không rotate log:
  app.log: Jan 1 → 10MB
  app.log: Jan 2 → 25MB
  app.log: Jan 3 → 40MB
  ...
  app.log: Mar 1 → 5GB  ← Server hết disk → App crash!
```

**Log rotation** tự động cắt file log theo kích thước/thời gian và xóa file cũ.

### 7.6.2. Logrotate — Công cụ hệ thống

```bash
# Tạo cấu hình logrotate cho ứng dụng
sudo tee /etc/logrotate.d/myapp << 'EOF'
/opt/myapp/logs/*.log {
    # Rotate mỗi ngày
    daily

    # Giữ lại 30 file cũ
    rotate 30

    # Nén file cũ (giảm ~90% kích thước)
    compress

    # Nén từ file thứ 2 trở đi (giữ file hôm qua chưa nén để đọc nhanh)
    delaycompress

    # Không báo lỗi nếu file không tồn tại
    missingok

    # Không rotate nếu file rỗng
    notifempty

    # Tạo file mới sau khi rotate với permission phù hợp
    create 644 deploy deploy

    # Chạy script sau khi rotate (signal app để mở file log mới)
    postrotate
        # Docker: restart container để mở file log mới
        docker kill --signal="USR1" myapp-api 2>/dev/null || true
    endscript
}
EOF

# Test cấu hình
sudo logrotate --debug /etc/logrotate.d/myapp

# Chạy thủ công (để test)
sudo logrotate --force /etc/logrotate.d/myapp

# Logrotate tự động chạy hàng ngày qua cron:
# /etc/cron.daily/logrotate
```

### 7.6.3. Log Rotation với Winston (application-level)

Đã tích hợp trong cấu hình winston ở mục 7.2.4:

```typescript
new winston.transports.DailyRotateFile({
  filename: 'logs/app-%DATE%.log',
  datePattern: 'YYYY-MM-DD',
  maxSize: '20m',      // Rotate khi file đạt 20MB
  maxFiles: '30d',     // Xóa file cũ hơn 30 ngày
  zippedArchive: true, // Nén file cũ
})
```

### 7.6.4. Docker Log Rotation

Docker container có log driver riêng — cần cấu hình để tránh đầy disk:

```yaml
# docker-compose.yml — cấu hình log rotation cho từng service
services:
  api:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"     # Mỗi file tối đa 10MB
        max-file: "5"       # Giữ tối đa 5 file (= 50MB tổng)

  postgres:
    logging:
      driver: "json-file"
      options:
        max-size: "5m"
        max-file: "3"
```

```bash
# Hoặc cấu hình global trong Docker daemon
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

sudo systemctl restart docker
```

---

## 7.7. Quy Trình Debug Production

Khi có sự cố trên production, đây là quy trình từng bước:

```
Nhận báo cáo / Alert tự động
         │
         ▼
Bước 1: Đánh giá tức thì (5 phút)
  • Mức độ ảnh hưởng: 10% hay 100% user?
  • Loại lỗi: 5xx hay business logic?
  • Khi nào bắt đầu: sau deploy hay tự nhiên?
         │
         ▼
Bước 2: Thu thập thông tin (10 phút)
  • Xem Grafana dashboard
  • Xem structured log với filter
  • Kiểm tra resource (CPU, RAM, disk)
         │
         ▼
Bước 3: Xác định nguyên nhân
  • Code bug? → Hotfix hoặc rollback
  • DB chậm? → Xem slow query log
  • Hết resource? → Scale up hoặc dọn dẹp
  • External dependency? → Check status page
         │
         ▼
Bước 4: Hành động
  • Rollback nếu do deploy gần nhất
  • Hotfix và deploy nhanh
  • Restart service nếu memory leak
  • Thông báo người dùng nếu downtime kéo dài
         │
         ▼
Bước 5: Post-mortem (sau khi resolve)
  • Nguyên nhân gốc rễ là gì?
  • Tại sao không phát hiện trước?
  • Thêm test/monitoring gì để tránh lần sau?
```

**Lệnh thực tế khi debug:**

```bash
# Xem log realtime
docker logs myapp-api -f --tail=200

# Tìm lỗi trong 1 giờ qua
docker logs myapp-api --since 1h 2>&1 | grep -E "ERROR|FATAL"

# Đếm số lỗi theo loại
docker logs myapp-api --since 1h 2>&1 \
  | grep "ERROR" \
  | jq -r '.message' \
  | sort | uniq -c | sort -rn | head -10

# Xem slow requests (> 3s)
docker logs myapp-api --since 1h 2>&1 \
  | jq -r 'select(.duration > 3000) | [.timestamp, .method, .url, .duration] | @tsv'

# Kiểm tra resource đang dùng
docker stats myapp-api myapp-postgres myapp-redis --no-stream

# Vào container để debug trực tiếp
docker exec -it myapp-api sh

# Kiểm tra kết nối database từ container
docker exec myapp-api node -e "
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
pool.query('SELECT NOW()').then(r => console.log('DB OK:', r.rows[0])).catch(e => console.error('DB ERROR:', e.message));
"
```

---

## 7.8. Stack Monitoring Đơn Giản Cho Người Mới

Nếu bạn mới bắt đầu và chưa muốn setup Prometheus/Grafana, đây là stack nhẹ hơn:

### 7.8.1. Uptimerobot — Free uptime monitoring

```
Uptimerobot.com (miễn phí):
  • Monitor HTTP endpoint mỗi 5 phút
  • Alert qua email/Slack khi down
  • Status page công khai cho user
  
Thêm monitor:
  URL: https://api.myapp.com/health/ping
  Type: HTTP(S)
  Interval: 5 minutes
  Alert: Email + Slack
```

### 7.8.2. Better Stack (Logtail) — Log management đơn giản

```bash
# Gửi log từ Docker lên Better Stack
docker run -d \
  --name log-forwarder \
  -v /var/lib/docker/containers:/var/lib/docker/containers:ro \
  -e LOGTAIL_TOKEN=your-token \
  betterstack/docker-logs-forwarder
```

### 7.8.3. Tóm tắt stack theo giai đoạn

```
Giai đoạn 1 — MVP (miễn phí):
  • Uptime monitoring: UptimeRobot (free)
  • Log: docker logs + grep thủ công
  • Alert: Email từ UptimeRobot

Giai đoạn 2 — Growth ($20/tháng):
  • Uptime: UptimeRobot Pro
  • Log: Better Stack / Logtail ($25/tháng)
  • Alert: Slack integration

Giai đoạn 3 — Scale (self-hosted):
  • Metrics: Prometheus + Grafana (miễn phí, tốn công setup)
  • Log: Loki + Grafana hoặc ELK Stack
  • Alert: Alertmanager → Slack/PagerDuty
  • Tracing: Jaeger / Tempo (OpenTelemetry)
```

---

## 7.9. Best Practices

### 7.9.1. Logging

```typescript
// ✅ Log đủ context — không log chung chung
this.logger.error('Login failed', {   // Có context
  userId,
  ip,
  reason: 'invalid_password',
  requestId,
});
// ❌ Quá chung chung — không biết lỗi ở đâu
this.logger.error('Error occurred');

// ✅ Không log dữ liệu nhạy cảm
this.logger.log('User authenticated', {
  userId: user.id,
  email: user.email,
  // password: dto.password  ← TUYỆT ĐỐI KHÔNG LOG
});

// ✅ Dùng log level đúng
this.logger.debug('SQL query executed', { sql });   // Chỉ dev cần
this.logger.log('User registered', { userId });     // Sự kiện bình thường
this.logger.warn('Retry attempt 3/5', { service }); // Cần chú ý
this.logger.error('Payment failed', { error });     // Cần xử lý ngay

// ✅ Dùng requestId để trace request xuyên suốt
// Đã setup trong LoggerMiddleware ở mục 7.2.5
```

### 7.9.2. Metrics

```
✅ Chọn metrics có ý nghĩa với business
  Không chỉ monitor technical metrics —
  monitor cả business metrics:
  • Số đơn hàng/phút
  • Số user đăng ký mới/ngày
  • Tỉ lệ thanh toán thành công

✅ Đặt ngưỡng alert dựa trên baseline
  Không alert khi CPU > 80% nếu bình thường vẫn 70%
  Alert khi tăng đột biến so với baseline

✅ Tránh alert fatigue (cảnh báo quá nhiều)
  Mỗi alert phải actionable — phải biết phải làm gì
  Alert critical phải được xử lý ngay
  Alert warning có thể xem trong giờ làm việc
```

### 7.9.3. Runbook — Tài liệu xử lý sự cố

```markdown
# Runbook: High Error Rate Alert

## Triệu chứng
- Alert: HighErrorRate fired
- Error rate > 5%

## Kiểm tra ngay
1. `docker logs myapp-api --tail 100 | grep ERROR`
2. Kiểm tra database: `docker exec myapp-postgres pg_isready`
3. Kiểm tra Redis: `docker exec myapp-redis redis-cli ping`
4. Kiểm tra disk: `df -h`

## Hành động

### Nếu DB down
```bash
docker restart myapp-postgres
# Nếu không recover sau 2 phút → restore từ backup
```

### Nếu do deploy gần nhất
```bash
./scripts/rollback.sh <previous-sha>
```

### Nếu hết disk
```bash
docker image prune -f
find /var/log -mtime +7 -name "*.log" -delete
```

## Liên hệ
- On-call: @team-backend trên Slack
- Escalate: CTO nếu > 15 phút không resolve
```

---

## Tóm Tắt Chương 7

| Khái niệm | Mô tả | Công cụ |
|---|---|---|
| **Structured Logging** | Log dạng JSON, có context đầy đủ | Winston + DailyRotateFile |
| **Log Levels** | FATAL/ERROR/WARN/INFO/DEBUG | Cấu hình theo môi trường |
| **HTTP Logging** | Tự động log mọi request/response | LoggerMiddleware |
| **Exception Filter** | Bắt và log tất cả lỗi nhất quán | AllExceptionsFilter |
| **Health Check** | Kiểm tra app + dependencies | @nestjs/terminus |
| **Metrics** | Đo lường hiệu năng theo thời gian | Prometheus + prom-client |
| **Visualization** | Dashboard metrics trực quan | Grafana |
| **Alerting** | Tự động thông báo khi bất thường | Alertmanager → Slack |
| **Log Rotation** | Tự động xóa/nén log cũ | Logrotate + Winston |
| **Uptime Monitoring** | Kiểm tra endpoint mỗi 5 phút | UptimeRobot (free) |

---

## Kết Luận Toàn Bộ Cuốn Sách

Bạn đã đi qua một hành trình hoàn chỉnh từ nền tảng đến production:

```
Chương 1 — Linux
  Nền tảng vận hành server: file system, process, network, script
        ↓
Chương 2 — Docker
  Đóng gói ứng dụng thành container: image, compose, volume, network
        ↓
Chương 3 — CI/CD
  Tự động hóa: test → build → deploy với GitHub Actions
        ↓
Chương 4 — Reverse Proxy
  Nginx làm cổng vào: HTTPS, SSL, rate limiting, load balancing
        ↓
Chương 5 — Deployment
  Quy trình deploy chuyên nghiệp: zero downtime, rollback, health check
        ↓
Chương 6 — Cloud
  Hạ tầng: VPS, Object Storage, CDN, DNS, Managed Database
        ↓
Chương 7 — Monitoring
  Vận hành: logging, metrics, alerting, log rotation
```

**Bước tiếp theo để phát triển:**

```
✅ Thực hành — Deploy một dự án NestJS thật lên VPS
✅ Kubernetes — Orchestration cho scale lớn hơn
✅ Terraform — Infrastructure as Code
✅ OpenTelemetry — Distributed Tracing
✅ GitOps — ArgoCD, Flux
✅ Security — OWASP, Penetration Testing
✅ AWS/GCP/Azure — Cloud-native services
```
