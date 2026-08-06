# TỔNG HỢP KIẾN THỨC WINSTON LOGGER TRONG NESTJS

## Mục lục
1. Giới thiệu chung
2. Cài đặt
3. Các khái niệm cốt lõi
4. Tích hợp Winston vào NestJS
5. Cấu trúc log chuẩn cho production
6. Best Practice & Clean Code
7. Các lỗi thường gặp
8. Tổng kết

---

## 1. Giới thiệu chung

**Winston** là thư viện logging phổ biến nhất cho Node.js, cho phép:
- Ghi log ra nhiều nơi cùng lúc (console, file, cloud...) gọi là **transport**.
- Phân loại log theo **mức độ nghiêm trọng** (level).
- Tùy biến **định dạng** log (format): JSON, text, có màu, có timestamp...
- Dễ dàng tích hợp với hệ thống giám sát như ELK, Datadog, CloudWatch.

**Vì sao dùng Winston thay vì `console.log`?**

| Tiêu chí | console.log | Winston |
|---|---|---|
| Phân loại mức độ log | Không | Có (error, warn, info...) |
| Ghi ra file/nhiều nguồn | Không | Có |
| Định dạng JSON cho hệ thống log tập trung | Không | Có |
| Bật/tắt log theo môi trường | Khó | Dễ (cấu hình theo level) |
| Hiệu năng khi log nhiều | Kém | Tối ưu hơn |

---

## 2. Cài đặt

```bash
npm install winston
npm install nest-winston   # package tích hợp Winston chính thức cho NestJS
```

---

## 3. Các khái niệm cốt lõi

### 3.1. Log Level (mức độ log)

Winston mặc định dùng chuẩn **npm levels**, sắp xếp theo độ nghiêm trọng giảm dần:

```
error → warn → info → http → verbose → debug → silly
```

> Quy tắc: khi đặt `level: 'info'`, Winston sẽ ghi log **info trở lên** (info, warn, error), bỏ qua debug/verbose/silly.

### 3.2. Transport (nơi ghi log)

Transport quyết định log được lưu ở đâu:
- `Console` – hiển thị ra terminal (dùng khi dev).
- `File` – ghi vào file `.log`.
- `DailyRotateFile` – tự động chia file log theo ngày (cần thêm `winston-daily-rotate-file`).
- Ngoài ra còn transport cho Elasticsearch, MongoDB, HTTP...

### 3.3. Format (định dạng log)

Winston cho phép **kết hợp nhiều format** bằng `winston.format.combine()`:

```javascript
winston.format.combine(
  winston.format.timestamp(),
  winston.format.errors({ stack: true }),
  winston.format.json(),
)
```

- `timestamp()` – thêm thời gian ghi log.
- `errors({ stack: true })` – log kèm stack trace khi log Error object.
- `json()` – xuất log dạng JSON, phù hợp cho production (dễ hệ thống khác parse).
- `colorize()` + `simple()` – log đẹp, có màu, phù hợp khi dev.

---

## 4. Tích hợp Winston vào NestJS

### 4.1. Cách khuyến nghị: dùng `nest-winston`

**Bước 1 — Tạo cấu hình Winston riêng biệt (clean code: tách config ra file riêng)**

```typescript
// src/config/winston.config.ts
import * as winston from 'winston';
import { utilities as nestWinstonModuleUtilities } from 'nest-winston';

export const winstonConfig = {
  transports: [
    new winston.transports.Console({
      level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.ms(),
        nestWinstonModuleUtilities.format.nestLike('MyApp', {
          colors: true,
          prettyPrint: true,
        }),
      ),
    }),
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json(),
      ),
    }),
    new winston.transports.File({
      filename: 'logs/combined.log',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json(),
      ),
    }),
  ],
};
```

**Bước 2 — Gắn Winston làm logger mặc định của toàn bộ ứng dụng**

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { WinstonModule } from 'nest-winston';
import { AppModule } from './app.module';
import { winstonConfig } from './config/winston.config';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    logger: WinstonModule.createLogger(winstonConfig),
  });
  await app.listen(3000);
}
bootstrap();
```

> Khi làm như trên, Winston sẽ **thay thế hoàn toàn** Logger mặc định của NestJS. Mọi log nội bộ của framework (khởi động module, route...) cũng sẽ đi qua Winston.

### 4.2. Sử dụng trong Service/Controller (Dependency Injection – đúng chuẩn NestJS)

```typescript
// src/user/user.service.ts
import { Injectable, Inject } from '@nestjs/common';
import { WINSTON_MODULE_PROVIDER } from 'nest-winston';
import { Logger } from 'winston';

@Injectable()
export class UserService {
  constructor(
    @Inject(WINSTON_MODULE_PROVIDER) private readonly logger: Logger,
  ) {}

  async createUser(dto: CreateUserDto) {
    this.logger.info(`Creating user with email: ${dto.email}`, {
      context: UserService.name,
    });

    try {
      // logic tạo user...
    } catch (error) {
      this.logger.error('Failed to create user', {
        context: UserService.name,
        stack: error.stack,
      });
      throw error;
    }
  }
}
```

**Vì sao dùng `@Inject` thay vì import trực tiếp winston?**
→ Giữ đúng nguyên tắc **Dependency Injection** của NestJS, giúp dễ test (mock logger) và thay đổi cấu hình tập trung một chỗ.

---

## 5. Cấu trúc log chuẩn cho production

Một log tốt cho hệ thống production nên có:

```json
{
  "timestamp": "2026-08-06T10:23:00.000Z",
  "level": "error",
  "context": "UserService",
  "message": "Failed to create user",
  "requestId": "a1b2-c3d4",
  "stack": "Error: ..."
}
```

**Gợi ý field nên có:**
- `timestamp` – bắt buộc, phục vụ tra cứu theo thời gian.
- `level` – mức độ log.
- `context` – tên class/module phát sinh log (giúp tìm nguồn gốc nhanh).
- `requestId` – mã định danh request (rất quan trọng để **truy vết log xuyên suốt 1 request**, đặc biệt trong microservices).
- `message` – nội dung log.

### Ví dụ: Interceptor gắn `requestId` tự động

```typescript
// src/common/interceptors/logging.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  Inject,
} from '@nestjs/common';
import { WINSTON_MODULE_PROVIDER } from 'nest-winston';
import { Logger } from 'winston';
import { v4 as uuidv4 } from 'uuid';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  constructor(
    @Inject(WINSTON_MODULE_PROVIDER) private readonly logger: Logger,
  ) {}

  intercept(context: ExecutionContext, next: CallHandler) {
    const request = context.switchToHttp().getRequest();
    request.requestId = uuidv4();
    const { method, url, requestId } = request;
    const now = Date.now();

    return next.handle().pipe(
      tap(() =>
        this.logger.info(`${method} ${url} - ${Date.now() - now}ms`, {
          context: 'HTTP',
          requestId,
        }),
      ),
    );
  }
}
```

---

## 6. Best Practice & Clean Code

| # | Nguyên tắc | Giải thích ngắn gọn |
|---|---|---|
| 1 | **Tách cấu hình Winston ra file riêng** | Dễ bảo trì, không lẫn logic nghiệp vụ. |
| 2 | **Không log dữ liệu nhạy cảm** | Tuyệt đối không log mật khẩu, token, số thẻ ngân hàng. |
| 3 | **Luôn kèm `context`** | Biết ngay log phát sinh từ class/module nào. |
| 4 | **Dùng level phù hợp** | `error` cho lỗi hệ thống, `warn` cho tình huống bất thường nhưng chưa gãy flow, `info` cho sự kiện nghiệp vụ quan trọng, `debug` chỉ bật khi dev. |
| 5 | **JSON format cho production** | Dễ tích hợp ELK/Datadog để tìm kiếm, lọc log. |
| 6 | **Dùng `DailyRotateFile`** | Tránh file log phình to vô hạn, tự động xóa log cũ. |
| 7 | **Gắn `requestId`** | Giúp truy vết toàn bộ hành trình 1 request, đặc biệt hữu ích khi debug lỗi production. |
| 8 | **Không log trong vòng lặp lớn** | Tránh làm chậm ứng dụng và làm phình file log. |
| 9 | **Log lỗi kèm stack trace** | Dùng `winston.format.errors({ stack: true })` để không mất thông tin debug. |
| 10 | **Inject logger qua DI**, không `new Logger()` thủ công | Đảm bảo tính nhất quán và dễ test (mock). |

---

## 7. Các lỗi thường gặp

- **Log quá nhiều ở mức `info`/`debug` trong production** → làm chậm hệ thống, tốn dung lượng lưu trữ.
- **Không xoay vòng (rotate) file log** → file log lớn dần theo thời gian, gây đầy ổ đĩa.
- **Log cả object phức tạp (circular reference)** mà không xử lý → gây lỗi runtime khi `JSON.stringify`.
- **Dùng chung 1 file log cho mọi mức độ** → khó lọc log lỗi khi cần xử lý sự cố gấp.
- **Quên gắn context** → khi hệ thống lớn, không biết log đến từ đâu.

---

## 8. Tổng kết

Winston kết hợp với `nest-winston` giúp:
- Chuẩn hóa logging trong toàn bộ ứng dụng NestJS thông qua Dependency Injection.
- Phân loại log theo mức độ và đích ghi (console, file, cloud).
- Dễ dàng mở rộng để tích hợp với hệ thống giám sát tập trung.

**Quy trình tối ưu khi triển khai:**
1. Tách file cấu hình Winston riêng.
2. Gắn Winston làm logger mặc định trong `main.ts`.
3. Inject `Logger` vào Service/Controller cần dùng.
4. Thêm Interceptor gắn `requestId` để truy vết log.
5. Áp dụng rotate file log + giới hạn level theo môi trường (dev/production).
