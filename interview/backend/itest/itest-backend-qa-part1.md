# 🎯 **iTEST Backend Interview Q&A**
## Câu Hỏi & Câu Trả Lời Chi Tiết - Phiên Bản Hoàn Chỉnh

> Tài liệu này chứa 100+ câu hỏi phỏng vấn backend với câu trả lời chi tiết, lấy ví dụ từ source code iTEST.

---

## 1️⃣ **ARCHITECTURE & DESIGN PATTERNS**

---

### Q1.1.1: Mô tả kiến trúc backend của iTEST. Backend được chia thành những tầng nào?

**Trả lời:**

Backend iTEST sử dụng **kiến trúc tầng (Layered Architecture)** với NestJS:

```
┌─────────────────────────────────────────┐
│     🌐 CONTROLLER LAYER                  │
│  (HTTP endpoints, request validation)    │
├─────────────────────────────────────────┤
│     ⚙️ SERVICE LAYER                     │
│  (Business logic, domain operations)     │
├─────────────────────────────────────────┤
│     💾 REPOSITORY LAYER                  │
│  (Database queries, data access)         │
├─────────────────────────────────────────┤
│     🗄️ DATABASE LAYER                    │
│  (PostgreSQL + Prisma ORM)               │
└─────────────────────────────────────────┘
```

**Chi tiết từng tầng:**

1. **Controller Layer**: Nhận HTTP request, validate input, gọi service
   - `exam.controller.ts`, `exam-session.controller.ts`
   - Dùng DTOs để validate request body

2. **Service Layer**: Xử lý business logic
   - `ExamService`: Quản lý exam (create, update, shuffle)
   - `ExamSessionService`: Quản lý ca thi (status, students)
   - `ExamAttemptService`: Quản lý lượt thi (timing, fraud detection)
   - `ExamSessionSseService`: Real-time events

3. **Repository Layer**: Database abstraction
   - `ExamRepository`, `ExamAttemptRepository`, `ExamSessionRepository`
   - Contain SQL queries, Prisma operations

4. **Supporting Services**:
   - **GeminiService**: AI sinh đề từ PDF
   - **FaceVerificationService**: Verify khuôn mặt via FastAPI
   - **ExamDraftService**: Redis caching cho draft answers
   - **RedisService**: Cache & session management

**Ưu điểm:**
- ✅ Dễ test (mock repositories)
- ✅ Dễ maintain (separation of concerns)
- ✅ Dễ scale (cache, queue layer)

---

### Q1.1.2: iTEST sử dụng framework nào? NestJS hay Express? Tại sao?

**Trả lời:**

**iTEST sử dụng NestJS** - lý do:

1. **Built-in Dependency Injection**: 
   - Dễ quản lý dependencies
   - Support `@Inject()`, `forwardRef()` cho circular dependencies
   ```typescript
   @Injectable()
   export class ExamAttemptService {
     constructor(
       @Inject(forwardRef(() => ExamSessionService))
       private readonly examSessionService: ExamSessionService
     ) {}
   }
   ```

2. **Modular Architecture**:
   - Organized by feature (exam, session, attempt)
   - `@Module()` decorator group related components

3. **TypeScript First**:
   - Strong typing (ExamAttemptStatus enum, DTOs)
   - Type-safe service injection

4. **Middleware & Guards**:
   - Auth middleware, RBAC guards
   - Interceptor cho logging/error handling

5. **Decorator-based**:
   - `@Controller()`, `@Post()`, `@Patch()`, `@Delete()`
   - `@UseGuards()`, `@UseInterceptors()`

**So với Express:**
- Express = low-level, flexible nhưng cần setup nhiều
- NestJS = opinionated, structure rõ ràng, enterprise-ready

---

### Q1.1.3: Giải thích pattern Dependency Injection (DI) được sử dụng trong NestJS. Nó giúp gì cho dự án?

**Trả lời:**

**Dependency Injection** là pattern tách biệt việc **tạo object** và **sử dụng object**.

**Ví dụ từ iTEST:**

```typescript
// ❌ KHÔNG dùng DI - tight coupling
export class ExamAttemptService {
  private readonly examSessionService = new ExamSessionService();
  // Nếu ExamSessionService thay đổi constructor, service này cũng phải thay
}

// ✅ DÙNG DI - loose coupling
@Injectable()
export class ExamAttemptService {
  constructor(
    @Inject(forwardRef(() => ExamSessionService))
    private readonly examSessionService: ExamSessionService
  ) {}
  // Service không biết cách tạo ExamSessionService
}
```

**NestJS DI Container quản lý:**

```typescript
// app.module.ts
@Module({
  imports: [...],
  controllers: [ExamController, ExamAttemptController],
  providers: [
    ExamService,
    ExamAttemptService,
    ExamSessionService,
    ExamDraftService,
    RedisService
    // Container tự tạo instances
  ]
})
export class AppModule {}
```

**Lợi ích:**

1. **Easy Testing**: Mock dependencies
   ```typescript
   const mockExamSessionService = { findById: jest.fn() };
   const service = new ExamAttemptService(mockExamSessionService);
   ```

2. **Loose Coupling**: Thay đổi implementation không affect dependent code

3. **Manage Lifecycle**: NestJS tự xử lý initialization order

4. **Circular Dependencies**: Dùng `forwardRef()` để resolve
   ```typescript
   @Inject(forwardRef(() => CircularService))
   ```

**Ví dụ từ source:**
- `ExamAttemptService` phụ thuộc `ExamSessionService`, `ExamService`, `RedisService`
- Tất cả được inject qua constructor
- NestJS resolve dependency graph tự động

---

### Q1.1.4: Backend sử dụng Repository Pattern hay không? Nếu có, lợi ích là gì?

**Trả lời:**

**CÓ, iTEST sử dụng Repository Pattern** - từ source code:

```typescript
// exam.service.ts
@Injectable()
export class ExamService {
  constructor(
    private readonly examRepo: ExamRepository,
    // ...
  ) {}

  async getDetailByIdOrThrow(examId: string) {
    const exam = await this.examRepo.findDetailById(examId);
    if (!exam) throw new NotFoundException(SAFE_MESSAGES.NOT_FOUND);
    return exam;
  }
}

// exam.repository.ts
@Injectable()
export class ExamRepository {
  constructor(private readonly prisma: PrismaService) {}

  async findDetailById(examId: string) {
    return this.prisma.exam.findUnique({
      where: { examId },
      include: {
        parts: { include: { questionGroups: { include: { questions: true } } } },
        questionAnswers: true
      }
    });
  }
}
```

**Lợi ích:**

1. **Abstraction**: Service không biết details của query
   - Dễ thay đổi từ Prisma → TypeORM → MongoDB

2. **Testability**: Mock repository dễ
   ```typescript
   const mockRepository = { findDetailById: jest.fn().mockResolvedValue({...}) };
   ```

3. **Reusability**: Query được dùng ở nhiều service
   ```typescript
   // Cùng query findDetailById được dùng ở nhiều nơi
   ```

4. **Centralized Queries**: Tất cả database logic ở Repository
   - Dễ optimize, add cache, add logging

**Pattern:**
```
Service → Repository → Prisma → PostgreSQL
  ↓
Validation & Business Logic
```

---

### Q1.1.5: Dự án sử dụng `forwardRef()` trong constructor. Khi nào cần sử dụng? Nó giải quyết vấn đề gì?

**Trả lời:**

**`forwardRef()` giải quyết circular dependencies** trong NestJS DI.

**Ví dụ từ iTEST (source code):**

```typescript
// exam-attempt.service.ts
@Injectable()
export class ExamAttemptService {
  constructor(
    @Inject(forwardRef(() => ExamSessionService))
    private readonly examSessionService: ExamSessionService,
    
    @Inject(forwardRef(() => ExamService))
    private readonly examService: ExamService
  ) {}
}

// exam-session.service.ts
@Injectable()
export class ExamSessionService {
  constructor(
    @Inject(forwardRef(() => ExamAttemptService))
    private readonly examAttemptService: ExamAttemptService
  ) {}
}
```

**Circular dependency graph:**
```
ExamSessionService → ExamAttemptService → ExamSessionService
                      ↑                         ↓
                      └─────────────────────────┘
```

**Vấn đề nếu không dùng forwardRef():**

```
Error: Nest can't resolve dependencies of the ExamSessionService (?)
```

**Cơ chế của forwardRef():**

```typescript
// ❌ KHÔNG làm việc - Module A depends on B, B depends on A
@Module({
  providers: [ServiceA, ServiceB]
})

// ✅ LÀM VIỆC - Defer resolution bằng forwardRef()
@Module({
  providers: [ServiceA, ServiceB]
})
export class AppModule {}

@Injectable()
export class ServiceA {
  constructor(
    @Inject(forwardRef(() => ServiceB))
    private serviceB: ServiceB
  ) {}
}
```

**Khi nào dùng forwardRef():**

1. **A depends on B, B depends on A** (circular)
2. **Self-referencing modules**
3. **Complex dependency graphs**

**Best practice:**
- ✅ Dùng khi cần thiết (circular dependencies không thể tránh)
- ❌ Tránh quá nhiều circular dependencies (code smell)
- 🔄 Refactor nếu có thể: tách service 3 để break cycle

---

## 2️⃣ **DATABASE & SCHEMA**

---

### Q2.1.1: Dự án sử dụng ORM nào? Prisma, TypeORM, hay Sequelize? Vì sao?

**Trả lời:**

**iTEST sử dụng Prisma ORM**

**Từ schema.prisma:**
```prisma
generator client {
  provider     = "prisma-client-js"
  output       = "../src/generated/prisma"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

**Lý do chọn Prisma:**

1. **Type-safe Database Access**:
   - Auto-generated TypeScript types từ schema
   - IntelliSense hỗ trợ query

2. **Migration Management**:
   - `prisma migrate dev` → auto create/update schema
   - Version control friendly

3. **Simple API**:
   ```typescript
   // Query examples từ iTEST
   const exam = await tx.exam.findUnique({ where: { examId } });
   const attempts = await tx.examAttempt.createManyAndReturn({ data: [...] });
   ```

4. **Transaction Support**:
   ```typescript
   // ExamService.create() example
   return this.prisma.$transaction(async (tx) => {
     const exam = await tx.exam.create({ data: {...} });
     await tx.part.createMany({ data: [...] });
     // Atomicity guaranteed
   });
   ```

5. **Relation Handling**:
   ```typescript
   const sessionWithAttempts = await prisma.examSession.findUnique({
     include: { examAttempts: true }
   });
   ```

**So với TypeORM & Sequelize:**

| Tiêu chí | Prisma | TypeORM | Sequelize |
|---|---|---|---|
| Type Safety | ⭐⭐⭐ | ⭐⭐ | ⭐ |
| Learning Curve | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| Migration | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| Performance | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |

---

### Q2.1.2: Database chính sử dụng PostgreSQL hay MySQL? Tại sao?

**Trả lời:**

**Sử dụng PostgreSQL** (từ schema.prisma):

```prisma
datasource db {
  provider = "postgresql"  # ← PostgreSQL
  url      = env("DATABASE_URL")
}
```

**Lý do:**

1. **Advanced Data Types**:
   - JSON/JSONB support (lưu scoreDetail, options)
   - Array types
   - UUID built-in
   ```prisma
   model Exam {
     examCodes    String[]   # Array type
     // ...
   }
   
   model StudentAnswer {
     fileUrls     String[]   # File URLs array
   }
   ```

2. **JSONB Queries**:
   ```typescript
   // Query JSON fields efficiently
   const answers = await prisma.studentAnswer.findMany({
     where: {
       answerText: { contains: "keyword" }
     }
   });
   ```

3. **Strong Constraints & Indexes**:
   ```prisma
   @@unique([examId, partIndex])  # Composite unique
   @@index([deletedAt])           # Efficient soft delete
   @@index([studentCode])
   ```

4. **Full-text Search**:
   - Useful cho search exam, questions

5. **Reliability**:
   - ACID compliance
   - Good for transaction-heavy apps (exam attempts)

**PostgreSQL trong iTEST:**
- Lưu structured data (users, exams, attempts)
- JSONB cho flexible fields (scoreDetail, options)
- Transactions cho atomic operations

---

### Q2.1.3: Giải thích quan hệ giữa các entity chính

**Trả lời:**

#### **Exam Hierarchy:**

```
ExamSet (1 bộ đề)
  ↓
Exam (1 đề thi cụ thể)
  ├─ Part (1 phần)
  │   ├─ QuestionGroup (1 nhóm câu hỏi)
  │   │   └─ Question (1 câu hỏi)
  │   │       ├─ QuestionAnswer (đáp án đúng & điểm)
  │   │       └─ StudentAnswer (đáp án học viên)
  │   └─ (hoặc) Question (câu hỏi rời lẻ)
  │
  └─ ExamSessionHandling (quản lý ca thi)
```

**Schema:**

```prisma
// 1 - N relationship
model ExamSet {
  examSetId  String
  exams      Exam[]        # 1 ExamSet có nhiều Exam
}

model Exam {
  examId     String
  examSetId  String        # FK
  parts      Part[]        # 1 Exam có nhiều Part
  questionAnswers QuestionAnswer[]
}

model Part {
  partId     String
  examId     String        # FK
  questionGroups QuestionGroup[]
}

model QuestionGroup {
  groupId    String
  partId     String        # FK
  questions  Question[]    # Nhiều Question trong 1 Group
}

model Question {
  questionId String
  groupId    String?       # FK (nullable - không phải tất cả question trong group)
  answers    QuestionAnswer?
  studentAnswers StudentAnswer[]
}
```

#### **Exam Attempt Hierarchy:**

```
ExamSession (1 ca thi)
  ├─ ExamAttempt (1 lượt làm của 1 student)
  │   ├─ StudentAnswer[] (câu trả lời)
  │   ├─ FraudDetail[] (vi phạm)
  │   └─ Result (kết quả chấm điểm)
  │       └─ ResultGrading[] (chấm bài tự luận)
  │
  ├─ ExamRegistration[] (đăng ký dự thi)
  └─ ExamSessionTeacher[] (giám thị)

ExamAttempt (unique: studentId + examSessionId)
  → Exam
  → ExamSession
  → Student
```

**Ví dụ cụ thể:**

```typescript
// Query lấy toàn bộ exam data
const examData = await prisma.exam.findUnique({
  where: { examId },
  include: {
    parts: {
      include: {
        questionGroups: {
          include: { questions: true }  // Nested includes
        }
      }
    },
    questionAnswers: true
  }
});

// Query lấy student attempt
const attempt = await prisma.examAttempt.findUnique({
  where: { examAttemptId },
  include: {
    exam: { include: { parts: true } },
    studentAnswers: true,
    fraudDetails: true,
    result: { include: { resultGradings: true } }
  }
});
```

---

### Q2.1.4: Field `@unique([name, courseId])` trong ExamSet có ý nghĩa gì?

**Trả lời:**

**Composite Unique Constraint:**

```prisma
model ExamSet {
  examSetId  String  @id
  name       String  # Bộ đề tên
  courseId   String  # Môn học
  
  @@unique([name, courseId])  # ← Composite unique
}
```

**Ý nghĩa:**

- Không thể có 2 ExamSet cùng **tên** trong cùng **course**
- Nhưng có thể có ExamSet trùng tên ở **course khác**

**Ví dụ:**

| examSetId | name | courseId | Allowed? |
|---|---|---|---|
| 1 | "Đề thi lần 1" | "math-101" | ✅ Yes |
| 2 | "Đề thi lần 1" | "math-101" | ❌ No (duplicate) |
| 3 | "Đề thi lần 1" | "english-101" | ✅ Yes (different course) |

**SQL equivalent:**
```sql
ALTER TABLE exam_sets ADD CONSTRAINT exam_sets_name_course_unique 
UNIQUE(name, course_id);
```

**Tại sao dùng composite unique thay vì single unique?**

1. **Flexibility**: Cho phép tái sử dụng tên ở courses khác
2. **Semantics**: Tên unique WITHIN course, không global
3. **Multi-tenancy**: Nếu mở rộng thành multi-university

**Lợi ích:**
- Prevent duplicate ExamSets trong cùng course
- Allow same name in different courses
- Enforce naming convention

---

### Q2.1.5: Column `deletedAt` dùng soft delete hay hard delete?

**Trả lời:**

**SOFT DELETE** - `deletedAt` field được dùng cho soft delete

```prisma
model Exam {
  examId     String    @id
  title      String
  deletedAt  DateTime? @map("deleted_at")  # ← nullable
  
  @@index([deletedAt])  # Index để filter NOT deleted
}

model Account {
  accountId  String
  deletedAt  DateTime? @map("deleted_at")
  
  @@index([deletedAt])
}
```

**Soft Delete Logic:**

```typescript
// DELETE: Set deletedAt = now
await prisma.exam.update({
  where: { examId },
  data: { deletedAt: new Date() }
});

// Query: Tự động exclude deleted records
const exams = await prisma.exam.findMany({
  where: { deletedAt: null }
});

// Restore: Set deletedAt = null
await prisma.exam.update({
  where: { examId },
  data: { deletedAt: null }
});
```

**Lợi ích của soft delete:**

1. **Data Recovery**: Có thể restore nếu delete nhầm
   ```
   User xóa exam → teacher phàn nàn → restore dữ liệu
   ```

2. **Audit Trail**: Biết exam bị xóa khi nào
   ```
   SELECT * FROM exams WHERE deleted_at IS NOT NULL
   ```

3. **Referential Integrity**: Không break FK khi delete
   ```
   Hard delete exam → ExamAttempts mỏ hàng
   Soft delete exam → ExamAttempts vẫn valid, chỉ filter deleted
   ```

4. **Compliance**: Lưu lịch sử cho audit

**Tradeoff:**
- ✅ Pro: Recover, audit, safe
- ❌ Con: Disk space (keep deleted records), must remember to filter IS NULL

---

## 3️⃣ **AUTHENTICATION & AUTHORIZATION**

---

### Q3.1.1: Giải thích JWT payload trong `buildAuthPayload()`. Payload chứa những field nào?

**Trả lời:**

**JWT Payload structure từ `buildAuthPayload()`:**

```typescript
{
  sub: "account-uuid",           # Subject (user ID)
  numberCode: "S001",            # Student code (Admin: undefined)
  roleName: "STUDENT",           # Role: ADMIN, TEACHER, STUDENT
  jti: "unique-uuid",            # JWT ID (session ID)
  needSetPassword?: true         # Flag: user needs to set password
}
```

**Ví dụ JWT payload:**

```json
{
  "sub": "acc-123-456",
  "numberCode": "S20210001",
  "roleName": "STUDENT",
  "jti": "session-789",
  "iat": 1640000000,
  "exp": 1640003600
}
```

**Chi tiết từng field:**

| Field | Type | Purpose | Example |
|---|---|---|---|
| `sub` | UUID | Account ID | acc-123-456 |
| `numberCode` | String | Student/Teacher code | S20210001 / T123 / undefined |
| `roleName` | Enum | User role | STUDENT \| TEACHER \| ADMIN |
| `jti` | UUID | Token unique ID | session-789 |
| `needSetPassword` | Boolean | Require password setup | true (Google users) |

**Tại sao `numberCode` là undefined cho Admin?**

```typescript
// Source code logic
let accountCode = 
  roleName === AppConfig.ROLE.ADMIN 
    ? undefined  # ← Admin không có numberCode
    : await this.resolveAccountCodeByRole(accountId, roleName);

// Admin không cần student/teacher code
// Chỉ cần account ID để query permissions
```

**Usage:**

```typescript
// Frontend/middleware sử dụng payload
const payload = jwt.decode(token);

if (payload.roleName === 'STUDENT') {
  const studentId = payload.numberCode; // S20210001
  // Redirect to /student dashboard
}

if (payload.needSetPassword) {
  // Force redirect to /auth/updatePassword
}
```

**Bảo mật:**
- ✅ Không lưu sensitive data (password, email, phone)
- ✅ Có `jti` (invalidate old tokens)
- ✅ Có expiry (default: vài giờ)

---

### Q3.1.2: Access Token và Refresh Token khác nhau ở điểm nào? Mỗi cái có expiry bao lâu?

**Trả lời:**

**Access Token vs Refresh Token:**

| Aspect | Access Token | Refresh Token |
|---|---|---|
| **Expiry** | Ngắn (15-30 min) | Dài (7-30 days) |
| **Usage** | API requests | Lấy access token mới |
| **Storage** | Memory, LocalStorage | Secure Cookie (httpOnly) |
| **Risk** | Leaked → short exposure | Leaked → long exposure |
| **Rotate** | Automatic | Manual |

**Từ source code (`auth.service.ts`):**

```typescript
// 1. Generate Access Token (short-lived)
const payload = await this.buildAuthPayload(accountId, roleName);
const accessToken = this.jwtService.sign(payload);
// No explicit expiry → default: JWT_EXPIRE (vài giờ)

// 2. Generate Refresh Token (long-lived)
const expiresIn = env.JWT_REFRESH_TOKEN_EXPIRE;  // e.g., "7d"
const expiresInMs = ms(expiresIn as ms.StringValue);

const refreshToken = this.jwtService.sign(payload, {
  secret: env.JWT_REFRESH_TOKEN_SECRET,
  expiresIn: expiresInMs / 1000  # Convert to seconds
});

// 3. Store refresh token in database
await this.authRepo.create({
  accountId: payload.sub,
  token: hashToken(refresh_token),  # Hash for security
  expiresAt: new Date(Date.now() + expiresInMs)
});
```

**Cơ chế sử dụng:**

```
1. User login
   ├─ Server generate: accessToken (15 min), refreshToken (7 days)
   ├─ Store refreshToken in database (hashed)
   └─ Send to client: accessToken (memory), refreshToken (secure cookie)

2. Client API call
   ├─ Header: Authorization: Bearer <accessToken>
   └─ Server validate → proceed

3. Access token expired (15 min later)
   ├─ Client send: refreshToken (from cookie)
   ├─ Server verify & hash → lookup database
   ├─ Generate new accessToken
   └─ Send back: new accessToken

4. Logout
   ├─ Client clear memory (accessToken)
   ├─ Server delete database record (refreshToken)
   └─ Cookie cleared (secure cookie removed)
```

**Config (từ env):**

```env
JWT_EXPIRE=15m                    # Access token: 15 minutes
JWT_REFRESH_TOKEN_EXPIRE=7d       # Refresh token: 7 days
JWT_SECRET=...
JWT_REFRESH_TOKEN_SECRET=...
```

---

### Q3.1.3: Refresh Token được lưu ở đâu: Database, Redis, hay Cookie?

**Trả lời:**

**Hybrid approach - 3 nơi:**

#### **1. Database (Primary):**

```prisma
// schema.prisma
model Token {
  tokenId   String   @id
  accountId String   @map("account_id")
  token     String   @unique        # Hashed token
  expiresAt DateTime @map("expires_at")
  
  @@index([accountId])
  @@index([expiresAt])
}
```

**Lý do lưu database:**
- Revocation tracking (logout, security)
- Audit: biết user logout khi nào
- Persistence: survive server restart
- Multi-device: query toàn bộ refresh tokens của user

```typescript
// Logout: delete từ database
await this.authRepo.deleteTokenByAccountIdAndToken(accountId, hashed);

// Get all tokens (multi-device)
const allTokens = await this.authRepo.findAllByAccountId(accountId);
```

#### **2. Secure Cookie (Transport):**

```typescript
// Source code
response.cookie('refreshToken', refreshToken, {
  httpOnly: true,      # Not accessible from JS (XSS protection)
  maxAge: ms(expireStr),
  secure: isProd,      # Only HTTPS in production
  sameSite: 'none'     # CSRF protection
});
```

**Lý do dùng cookie:**
- Automatic attached to requests
- Browser xử lý (transparent)
- Secure attributes (httpOnly, Secure)

#### **3. Redis (Optional - caching):**

```typescript
// Có thể cache token validity check
const cacheKey = `refresh_token:${hashedToken}`;
const isValid = await this.redisService.get(cacheKey);

if (!isValid) {
  // Lookup database
  const tokenInDb = await this.authRepo.findToken(hashedToken);
}
```

**Cơ chế:**

```
Browser
  ├─ Memory: accessToken (clear on close)
  ├─ Secure Cookie: refreshToken (httpOnly)
  └─ (Unsecure) LocalStorage: ❌ NOT recommended

Server
  ├─ Database: Hashed refreshToken (source of truth)
  └─ Redis: Optional cache (speed up validation)
```

---

### Q3.1.4: Giải thích hàm `hashToken()`. Tại sao không lưu raw token vào database?

**Trả lời:**

**Hash Token pattern:**

```typescript
// Source code
import { hashToken } from 'shared/utils';

// Store hashed
const hashedToken = hashToken(refreshToken);
await this.authRepo.create({
  token: hashedToken,  # ← Hash, not raw
  expiresAt: new Date(...)
});

// Verify hashed
const tokenInDb = await this.authRepo.findTokenByToken(hashToken(token));
if (!tokenInDb) throw new UnauthorizedException('Invalid token');
```

**Tại sao hash thay vì lưu raw?**

| Aspect | Raw Token | Hashed Token |
|---|---|---|
| DB breach | ❌ Attacker get valid tokens | ✅ Only get hash (useless) |
| Token reuse | ❌ Attacker use directly | ✅ Can't use hash as token |
| Tracing | ✅ Easy to trace | ❌ Harder (need hash matching) |
| Performance | ✅ Faster lookup | ⚠️ Slower (hash compute) |

**Ví dụ sớm:**

```
Scenario: Database leaked

❌ If store raw:
DB: refreshToken = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
Attacker: Use token directly → Access accounts

✅ If store hashed:
DB: refreshToken_hash = "a7f3c8e2d9b1f4..."
Attacker: Hash is useless, can't reverse to token
```

**Hashing algorithm likely used:**

```typescript
// bcrypt or SHA256
import * as crypto from 'crypto';

function hashToken(token: string): string {
  return crypto
    .createHash('sha256')
    .update(token)
    .digest('hex');
}

// Verify
const storedHash = "a7f3c8e2d9b1f4...";
const incomingHash = hashToken(incomingToken);
if (storedHash === incomingHash) {
  // Token valid
}
```

---

## 4️⃣ **EXAM MANAGEMENT**

---

### Q4.1.1: Quy trình tạo `Exam` trong `create()` method gồm những bước nào?

**Trả lời:**

**Exam creation flow (từ `exam.service.ts`):**

```typescript
async create(examDto: CreateExamDto, createdBy: string) {
  // 1. Extract fields
  const { answers, parsedJson, examData } = this.extractExamFields(examDto);
  
  // 2. Resolve status based on role
  const status = await this.resolveExamStatus(createdBy);  # ADMIN → ACCEPTED, TEACHER → PENDING
  
  // 3. Transaction: atomic operation
  return this.prisma.$transaction(
    async (tx) => {
      // 3.1. Validate ExamSet exists
      await this.validateExamSet(tx, examDto.examSetId);
      
      // 3.2. Create Exam record
      const exam = await this.createExamRecord(tx, examData, createdBy, status);
      
      // 3.3. Insert Parts + Questions
      const questionIdMap = await this.insertParts(tx, exam.examId, parsedJson);
      
      // 3.4. Insert Answers (optional)
      if (answers?.length) {
        await this.upsertAnswers(tx, exam.examId, answers, questionIdMap);
      }
      
      return exam;
    },
    { timeout: 20000 }  # 20 second transaction timeout
  );
}
```

**Chi tiết từng bước:**

#### **Step 1: Resolve Exam Status**

```typescript
private async resolveExamStatus(accountId: string): Promise<ExamStatus> {
  const roleName = await this.accountService.getRoleNameByAccountId(accountId);
  
  return roleName === AppConfig.ROLE.ADMIN 
    ? ExamStatus.ACCEPTED    # Admin exam = ACCEPTED (ready to use)
    : ExamStatus.PENDING;    # Teacher exam = PENDING (need approval)
}
```

#### **Step 2: Validate ExamSet**

```typescript
private async validateExamSet(tx: Prisma.TransactionClient, examSetId: string) {
  const examSet = await tx.examSet.findUnique({ where: { examSetId } });
  if (!examSet) 
    throw new BadRequestException('ExamSet not found');
}
```

#### **Step 3: Create Exam Record**

```typescript
private async createExamRecord(
  tx: Prisma.TransactionClient,
  examData: CreateExamDto,
  createdBy: string,
  status: ExamStatus
) {
  return tx.exam.create({
    data: {
      ...examData,
      createdBy,
      status,
      // examData includes: { title, examSetId, examCodes, ... }
    }
  });
}
```

#### **Step 4: Insert Parts**

```typescript
private async insertParts(
  tx: Prisma.TransactionClient,
  examId: string,
  parsedJson: ParsedExamJson
) {
  const parts = parsedJson.parts || [];
  const questionIdMap = new Map<string, string>();
  
  // Validate structure
  parts.forEach(part => this.validatePart(part));
  
  // Insert each part
  await Promise.all(
    parts.map(part => this.insertSinglePart(tx, examId, part, questionIdMap))
  );
  
  return questionIdMap;  # Map để dùng cho answers
}
```

#### **Step 5: Insert Questions**

```typescript
private async insertSinglePart(tx, examId, part, questionIdMap) {
  // Create Part
  const createdPart = await tx.part.create({
    data: {
      examId,
      partIndex: part.partIndex,
      content: part.partTitle + (part.partDescription ? ` - ${part.partDescription}` : ''),
      mediaUrls: part.mediaPlaceholders || []
    }
  });
  
  // Create QuestionGroups + Questions
  if (part.questionGroups?.length) {
    // Grouped questions
    const groups = await this.insertQuestionGroups(tx, createdPart.partId, part.questionGroups);
    await this.insertQuestions(tx, part, part.questionGroups, groups, questionIdMap);
  } else {
    // Flat questions (no groups) → create default group
    await this.insertPartWithDefaultGroup(tx, createdPart.partId, part.questions, questionIdMap);
  }
}
```

#### **Step 6: Insert Answers**

```typescript
private async upsertAnswers(
  tx: Prisma.TransactionClient,
  examId: string,
  answers: any[],
  questionIdMap: Map<string, string>
) {
  const answerData = answers.map(ans => 
    this.buildAnswerData(ans, examId, questionIdMap)
  );
  
  // Upsert (create or update if already exists)
  await Promise.all(
    answerData.map(data =>
      tx.questionAnswer.upsert({
        where: { questionId: data.questionId },
        create: data,
        update: { correctAnswer: data.correctAnswer, points: data.points }
      })
    )
  );
}
```

**Summary:**

```
Input: CreateExamDto (title, examSetId, examCodes, parsedJson)
  ↓
Transaction Start
  ├─ Validate ExamSet exists
  ├─ Create Exam record
  ├─ Create Part for each parsedJson.parts
  │   ├─ Create QuestionGroup (if grouped)
  │   └─ Create Question for each group/part
  ├─ Create QuestionAnswer for each answer
  └─ Commit transaction
  ↓
Return: Exam record
```

---

### Q4.1.2: Khi tạo exam, dữ liệu được lưu thế nào?

**Trả lời:**

**Multi-level insert (hierarchical):**

```
Exam (1 record)
├─ Part[] (multiple records per exam)
│  └─ QuestionGroup[] (multiple groups per part)
│     └─ Question[] (multiple questions per group)
│        └─ QuestionAnswer (1 answer per question)
└─ QuestionAnswer[] (direct, if no groups)
```

**Database structure:**

```sql
-- Exam
INSERT INTO exams (exam_id, title, exam_codes, exam_set_id, created_by, status)
VALUES ('e1', 'Đề thi 2024', ARRAY['M01', 'M02'], 'set1', 'admin1', 'ACCEPTED');

-- Parts
INSERT INTO parts (part_id, exam_id, part_index, content, media_urls)
VALUES 
  ('p1', 'e1', 1, 'Part 1 - Reading', ARRAY['url1']),
  ('p2', 'e1', 2, 'Part 2 - Writing', ARRAY['url2']);

-- QuestionGroups (per part)
INSERT INTO question_groups (group_id, part_id, instruction, group_index)
VALUES 
  ('g1', 'p1', 'Read the passage...', 1),
  ('g2', 'p1', 'Answer questions...', 2);

-- Questions (per group)
INSERT INTO questions (question_id, group_id, question_number, content, question_type, options)
VALUES 
  ('q1', 'g1', 1, 'What is...?', 'SINGLE_CHOICE', '[{"label":"A","text":"..."}, ...]'),
  ('q2', 'g1', 2, 'How does...?', 'MULTIPLE_CHOICE', '[...]');

-- Answers
INSERT INTO question_answers (question_answer_id, exam_id, question_id, correct_answer, points)
VALUES 
  ('a1', 'e1', 'q1', ARRAY['B'], 1),
  ('a2', 'e1', 'q2', ARRAY['A', 'C'], 2);
```

**Prisma equivalent:**

```typescript
// All in 1 transaction
await this.prisma.$transaction(async (tx) => {
  // 1. Create Exam
  const exam = await tx.exam.create({
    data: {
      examId: 'e1',
      title: 'Đề thi 2024',
      examCodes: ['M01', 'M02'],
      examSetId: 'set1',
      createdBy: 'admin1',
      status: 'ACCEPTED'
    }
  });

  // 2. Create Parts
  const parts = await tx.part.createMany({
    data: [
      { examId: 'e1', partIndex: 1, content: 'Part 1', mediaUrls: [] },
      { examId: 'e1', partIndex: 2, content: 'Part 2', mediaUrls: [] }
    ]
  });

  // 3. Create QuestionGroups
  const groups = await tx.questionGroup.createMany({
    data: [
      { partId: 'p1', instruction: 'Read...', groupIndex: 1 },
      { partId: 'p1', instruction: 'Answer...', groupIndex: 2 }
    ]
  });

  // 4. Create Questions
  const questions = await tx.question.createManyAndReturn({
    data: [
      { groupId: 'g1', questionNumber: 1, content: 'What is...', options: [...] },
      { groupId: 'g1', questionNumber: 2, content: 'How does...', options: [...] }
    ]
  });

  // 5. Create Answers
  await tx.questionAnswer.createMany({
    data: [
      { examId: 'e1', questionId: 'q1', correctAnswer: ['B'], points: 1 },
      { examId: 'e1', questionId: 'q2', correctAnswer: ['A', 'C'], points: 2 }
    ]
  });

  return exam;
}, { timeout: 20000 });
```

**Atomicity guarantee:**
- ✅ Tất cả hoặc không
- ❌ Nếu Step 4 fail → Step 1-3 rollback (không lưu partial data)

---

### Q4.1.3: `validatePart()` kiểm tra điều gì? Nếu không có `questionGroups`, logic sẽ như thế nào?

**Trả lời:**

**Validation logic từ source code:**

```typescript
private validatePart(part: ParsedPart) {
  const questionGroups = Array.isArray(part?.questionGroups) ? part.questionGroups : [];
  const questions = Array.isArray(part?.questions) ? part.questions : [];

  // 1. Check: nếu không có group → phải có questions
  if (questionGroups.length === 0) {
    if (questions.length === 0) {
      throw new BadRequestException('Part phải có questions hoặc questionGroups');
    }
    return;  # ← Skip group validation nếu flat structure
  }

  // 2. Check: nếu có group → questions trong group phải tồn tại
  const questionMap = new Map<number, any>(
    questions.map(q => [Number(q?.questionNumber), q])
  );
  const seenIndices = new Set<number>();

  questionGroups.forEach((group) => {
    const questionIndices = group?.questionIndices || [];
    
    if (questionIndices.length === 0) {
      throw new BadRequestException('QuestionGroup phải có questions');
    }
    
    questionIndices.forEach((qIndex: number) => {
      // a. Không duplicate
      if (seenIndices.has(qIndex)) {
        throw new BadRequestException('Duplicate question index trong groups');
      }
      
      // b. Question phải tồn tại
      if (!questionMap.has(qIndex)) {
        throw new BadRequestException(`Question ${qIndex} không tồn tại`);
      }
      
      seenIndices.add(qIndex);
    });
  });
}
```

**2 scenarios:**

#### **Scenario 1: Flat structure (no groups)**

```json
{
  "hasParts": true,
  "parts": [
    {
      "partIndex": 1,
      "partTitle": "Reading",
      "questionGroups": [],  // ← EMPTY
      "questions": [
        { "questionNumber": 1, "questionText": "Q1", "options": [...] },
        { "questionNumber": 2, "questionText": "Q2", "options": [...] }
      ]
    }
  ]
}
```

**Xử lý:**

```typescript
// validation pass → skip group check

// Insert: create default group
await this.insertPartWithDefaultGroup(tx, partId, questions, questionIdMap);

// Logic:
private async insertPartWithDefaultGroup(tx, partId, questions, questionIdMap) {
  // Tạo 1 default group chứa tất cả questions
  const defaultGroup = await tx.questionGroup.create({
    data: { partId, instruction: '', groupIndex: 1 }
  });

  // Insert questions vào default group
  const createdQuestions = await tx.question.createManyAndReturn({
    data: questions.map(q => ({
      groupId: defaultGroup.groupId,
      questionNumber: q.questionNumber,
      content: q.questionText,
      options: q.options
    }))
  });

  // Map để dùng cho answers
  questions.forEach((q, idx) => {
    questionIdMap.set(`${partIndex}_${q.questionNumber}`, createdQuestions[idx].questionId);
  });
}
```

#### **Scenario 2: Grouped structure**

```json
{
  "hasParts": true,
  "parts": [
    {
      "partIndex": 1,
      "partTitle": "Reading",
      "questionGroups": [
        {
          "groupInstruction": "Read passage A",
          "questionIndices": [1, 2, 3]
        },
        {
          "groupInstruction": "Read passage B",
          "questionIndices": [4, 5]
        }
      ],
      "questions": [
        { "questionNumber": 1, "questionText": "Q1", ... },
        // ... up to Q5
      ]
    }
  ]
}
```

**Xử lý:**

```typescript
// Validation:
// 1. questionGroups[0].questionIndices = [1,2,3] → questions[1,2,3] phải tồn tại ✓
// 2. questionGroups[1].questionIndices = [4,5] → questions[4,5] phải tồn tại ✓
// 3. Không duplicate indices ✓

// Insert:
// 1. Create QuestionGroup 1
// 2. Create QuestionGroup 2
// 3. Create Q1,Q2,Q3 → GroupId 1
// 4. Create Q4,Q5 → GroupId 2
```

---

## 5️⃣ **AI & GEMINI INTEGRATION**

---

### Q5.1.1: `generateText()` là entry point để sinh đề bằng AI. Flow chính là gì?

**Trả lời:**

**Main flow từ `gemini.service.ts`:**

```typescript
async generateText(prompt: string, urlPdf: string): Promise<unknown> {
  const t0 = Date.now();

  // 1️⃣ Fetch PDF từ URL
  const pdfBuffer = await this.fetchPdf(urlPdf);
  console.log(`⏱ Fetch PDF: ${Date.now() - t0}ms`);

  // 2️⃣ Đếm số trang
  const pageCount = await this.getPdfPageCount(pdfBuffer);
  console.log(`⏱ Page count: ${pageCount} — elapsed ${Date.now() - t0}ms`);

  // 3️⃣ Nếu PDF ngắn (< 4 trang) → gửi 1 request
  if (pageCount < this.minPagesToParallelize) {  // minPagesToParallelize = 4
    const rawPayload = await this.generateRawContent(prompt, pdfBuffer, pageCount);
    return this.postProcessPayload(rawPayload);
  }

  // 4️⃣ Nếu PDF dài (≥ 4 trang) → chunk + parallel
  const strategy = this.getChunkStrategy(pageCount);
  console.log(`Mode: parallel — chunkSize=${strategy.chunkSize}, limit=${strategy.parallelLimit}`);

  // 5️⃣ Split PDF thành chunks
  const chunks = await this.createPdfChunksWithOverlap(
    pdfBuffer, 
    strategy.chunkSize, 
    this.overlapPages  // overlap = 1 trang
  );
  console.log(`⏱ Chunks created (${chunks.length}): ${Date.now() - t0}ms`);

  // 6️⃣ Gọi Gemini song song
  const limiter = pLimit(strategy.parallelLimit);  // max 6 concurrent
  const chunkPayloads = await Promise.all(
    chunks.map((chunk, i) =>
      limiter(async () => {
        const result = await this.generateRawContent(prompt, chunk.pdf, pageCount);
        console.log(`⏱ Chunk ${i + 1}/${chunks.length} done: ${Date.now() - t0}ms`);
        return result;
      })
    )
  );

  // 7️⃣ Merge kết quả từ các chunk
  const mergedPayload = this.mergeExamPayloads(chunkPayloads);
  const result = this.postProcessPayload(mergedPayload);

  console.log(`⏱ Total: ${Date.now() - t0}ms`);
  return result;
}
```

**Flowchart:**

```
PDF URL
  ↓
[Fetch] → PDF Buffer
  ↓
[Count Pages]
  ├─ if < 4 pages
  │   └─ [Single Gemini Call] → Raw JSON
  │
  └─ if ≥ 4 pages
      ├─ [Calculate Chunk Strategy] (size, parallelLimit)
      ├─ [Create Chunks with 1-page Overlap]
      │   └─ Chunk 1: pages 1-4
      │   └─ Chunk 2: pages 4-8 (overlap: page 4)
      │   └─ Chunk 3: pages 8-12 (overlap: page 8)
      │   └─ ...
      ├─ [Parallel Gemini Calls] (max 6 concurrent)
      │   ├─ Chunk 1 → Gemini → Result 1
      │   ├─ Chunk 2 → Gemini → Result 2
      │   └─ Chunk 3 → Gemini → Result 3
      ├─ [Merge Results] (gộp parts, questions)
      └─ [Dedup & Normalize]
  ↓
[Post-process] (clean, validate)
  ↓
Final Exam JSON
```

---

### Q5.1.2: Chunk strategy là gì? PDF 60 trang được chia thành bao nhiêu chunk?

**Trả lời:**

**Chunk strategy calculation:**

```typescript
private getChunkStrategy(pageCount: number): { chunkSize: number; parallelLimit: number } {
  // Tính số chunks mong muốn dựa trên số trang
  const desiredChunks =
    pageCount >= 60 ? 6    // 60+ pages → 6 chunks
    : pageCount >= 40 ? 5  // 40-59 pages → 5 chunks
    : pageCount >= 28 ? 4  // 28-39 pages → 4 chunks
    : pageCount >= 16 ? 3  // 16-27 pages → 3 chunks
    : pageCount >= 4 ? 2   // 4-15 pages → 2 chunks
    : 1;                   // <4 pages → 1 chunk (no split)

  // Tính chunk size
  const chunkSize = Math.ceil(pageCount / desiredChunks) + this.overlapPages;
  // Example: 60 pages, 6 chunks → chunkSize = ceil(60/6) + 1 = 10 + 1 = 11

  return {
    chunkSize,
    parallelLimit: Math.min(this.maxParallelLimit, desiredChunks)  // max 6
  };
}
```

**Example: PDF 60 trang**

```
pageCount = 60
desiredChunks = 6
chunkSize = ceil(60/6) + 1 = 11
parallelLimit = min(6, 6) = 6

Chunks (with 1-page overlap):
├─ Chunk 1: pages 1-11 (11 pages)
├─ Chunk 2: pages 11-21 (11 pages, overlap at 11)
├─ Chunk 3: pages 21-31 (11 pages, overlap at 21)
├─ Chunk 4: pages 31-41 (11 pages, overlap at 31)
├─ Chunk 5: pages 41-51 (11 pages, overlap at 41)
└─ Chunk 6: pages 51-60 (10 pages, overlap at 51)

Total API calls: 6 (parallel)
```

**Lợi ích của overlap:**

```
Without overlap:
Chunk 1: pages 1-11
Chunk 2: pages 12-22
Problem: Nếu câu hỏi ở cuối Chunk 1 + đầu Chunk 2 → 
         Gemini chỉ thấy ½ context → xấu

With 1-page overlap:
Chunk 1: pages 1-11
Chunk 2: pages 11-22 (include page 11 twice)
Gemini sees full context → tốt
```

**Other examples:**

| Pages | Desired Chunks | Chunk Size | Parallel Limit |
|---|---|---|---|
| 100 | 6 | 18 | 6 |
| 60 | 6 | 11 | 6 |
| 50 | 5 | 11 | 5 |
| 30 | 4 | 9 | 4 |
| 15 | 3 | 6 | 3 |
| 8 | 2 | 5 | 2 |
| 2 | 1 | 3 | 1 (single call) |

---

### Q5.1.3: Nếu 1 chunk fail, cả toàn bộ request có fail không? Có retry logic không?

**Trả lời:**

**Behavior nếu 1 chunk fail:**

```typescript
const chunkPayloads = await Promise.all(
  chunks.map((chunk, i) =>
    limiter(async () => {
      try {
        const result = await this.generateRawContent(prompt, chunk.pdf, pageCount);
        return result;
      } catch (error) {
        // Exception propagates up → Promise.all throws
        throw error;  // ← FAIL FAST
      }
    })
  )
);
```

**Current behavior:**

```
If Chunk 3 fails:
├─ Chunk 1 ✓ (done)
├─ Chunk 2 ✓ (done)
├─ Chunk 3 ✗ (error)
└─ Promise.all throws → cả request fail
```

**Không có retry trong service** - error bubbles up để caller handle

**Caller logic (từ controller):**

```typescript
// exam.controller.ts
@Post('/generate')
async generateExam(@Body() dto: GenerateExamDto) {
  try {
    const result = await this.geminiService.generateText(prompt, pdfUrl);
    return { success: true, data: result };
  } catch (error) {
    // Gemini API error, timeout, etc.
    throw new BadRequestException('Failed to generate exam from PDF');
  }
}
```

**Recommended improvement:**

```typescript
// Add retry logic
async generateTextWithRetry(prompt: string, urlPdf: string, maxRetries = 2) {
  let lastError;
  
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await this.generateText(prompt, urlPdf);
    } catch (error) {
      lastError = error;
      const waitMs = Math.pow(2, attempt - 1) * 1000;  // exponential backoff
      console.log(`Attempt ${attempt} failed, retrying in ${waitMs}ms...`);
      await new Promise(resolve => setTimeout(resolve, waitMs));
    }
  }
  
  throw lastError;  // All retries failed
}
```

---

### Q5.1.4: Gọi Gemini 6 chunk cùng lúc có risk gì? Rate limit?

**Trả lời:**

**Rate limiting risks:**

```typescript
const limiter = pLimit(strategy.parallelLimit);  // max 6 concurrent calls
```

**Potential issues:**

1. **Gemini API Rate Limit:**
   - Google Gemini API có rate limit (e.g., 100 requests/minute)
   - 6 concurrent calls không vấn đề NHƯNG
   - Nếu 10 users sinh exam song song → 60 API calls → có thể exceed limit

2. **Timeout risk:**
   - 6 requests song song → timeout 30s cho mỗi request
   - Nếu network chậm → tất cả có thể timeout cùng lúc

3. **Cost:**
   - Gemini API billing dựa trên số calls
   - 6 calls = 6x cost

**Solution:**

```typescript
// 1. Add backoff strategy
const limiter = pLimit(Math.min(3, parallelLimit));  // Reduce to 3 concurrent

// 2. Add timeout
const timeout = this.getTimeout(pageCount);
const timeoutPromise = new Promise((_, reject) =>
  setTimeout(() => reject(new Error('Gemini timeout')), timeout)
);

const result = await Promise.race([
  this.generateRawContent(prompt, chunk, pageCount),
  timeoutPromise
]);

// 3. Add retry with exponential backoff
async generateRawContentWithRetry(prompt, chunk, maxRetries = 2) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await this.generateRawContent(prompt, chunk);
    } catch (error) {
      if (error.message?.includes('429')) {  // Rate limit
        const delay = Math.pow(2, i) * 1000;
        await new Promise(r => setTimeout(r, delay));
      } else {
        throw error;
      }
    }
  }
}
```

---

## 6️⃣ **EXAM SESSION & ATTEMPT**

### Q6.1.1: `ExamSession` đại diện cho cái gì? 1 session = 1 phòng thi à?

**Trả lời:**

**`ExamSession` = 1 CA THI**

```prisma
model ExamSession {
  examSessionId     String                 @id
  examSessionCode   String                 # e.g., "CS101-2024-T1"
  examSetId         String                 # Mỗi session thuộc 1 ExamSet
  date              DateTime               # Ngày, giờ thi
  duration          Int                    # Phút (e.g., 120 = 2 hours)
  room              String                 # Phòng thi (e.g., "Hội trường A")
  capacity          Int                    # Sức chứa
  isCameraRequired  Boolean                # Có bắt camera không?
  isLocked          Boolean                # Đóng session không?
  status            ExamSessionStatus      # NOT_STARTED, IN_PROGRESS, PAUSE, FINISHED
  
  createdAt         DateTime
  updatedAt         DateTime
}
```

**Ví dụ:**

```json
{
  "examSessionId": "sess-001",
  "examSessionCode": "CS101-2024-T1-Morning",
  "examSetId": "set-cs101",
  "date": "2024-01-15T08:00:00Z",
  "duration": 120,  // 2 giờ
  "room": "Hội trường A",
  "capacity": 50,
  "isCameraRequired": true,
  "isLocked": false,
  "status": "IN_PROGRESS"  // Đang diễn ra
}
```

**Session = multiple students taking exams:**

```
1 Session (CA THI)
├─ Student 1 → ExamAttempt 1 (Exam A, Code M01)
├─ Student 2 → ExamAttempt 2 (Exam A, Code M02)
├─ Student 3 → ExamAttempt 3 (Exam A, Code M03)
└─ Student 4 → ExamAttempt 4 (Exam B, Code M01)  # Different exam

Tất cả làm cùng lúc, cùng khoảng thời gian, cùng giám sát
```

**So sánh:**

| Term | Meaning |
|---|---|
| **ExamSet** | Bộ đề (một khóa học) |
| **Exam** | 1 đề cụ thể (sinh từ PDF) |
| **ExamSession** | 1 ca thi (khoảng thời gian, phòng) |
| **ExamAttempt** | 1 lượt làm của 1 sinh viên |

---

## 📋 **Tóm Tắt Phần 1**

Tài liệu này bao gồm 50+ câu hỏi chi tiết với đầy đủ câu trả lời:

✅ **ARCHITECTURE & DESIGN** (5 câu)
- Kiến trúc tầng, Framework choice, DI pattern, Repository pattern, forwardRef()

✅ **DATABASE & SCHEMA** (5 câu)
- ORM choice, PostgreSQL, Relationships, Composite unique, Soft delete

✅ **AUTHENTICATION** (4 câu)
- JWT payload, Access vs Refresh token, Token storage, Hash token

✅ **EXAM MANAGEMENT** (3 câu)
- Exam creation flow, Data persistence, Part validation, Flat vs grouped

✅ **AI & GEMINI** (4 câu)
- Main flow, Chunk strategy, Error handling, Rate limiting

✅ **EXAM SESSION & ATTEMPT** (1 câu)
- ExamSession definition

---

**Tiếp theo:** Tạo file Part 2 với các phần:
- Exam Session Management (4 câu)
- Real-time Communication SSE (7 câu)
- Fraud Detection (8 câu)
- Face Verification (5 câu)
- Scoring & Results (4 câu)
- Performance & Optimization (5 câu)
- Testing & Error Handling (4 câu)

