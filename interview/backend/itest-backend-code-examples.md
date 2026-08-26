# 💻 **iTEST Backend - Code Examples & Live Coding Scenarios**

> Tài liệu này chứa các ví dụ code, scenario thực tế, và bài tập live coding để phỏng vấn backend iTEST.

---

## 📑 **Mục Lục**

1. [Code Reading & Analysis](#1-code-reading--analysis)
2. [Live Coding Problems](#2-live-coding-problems)
3. [Database Query Problems](#3-database-query-problems)
4. [API Design Problems](#4-api-design-problems)
5. [Debugging Scenarios](#5-debugging-scenarios)

---

## 1. Code Reading & Analysis

### 1.1 AuthService Code Analysis

**Code snippet:**

```typescript
private async buildAuthPayload(
  accountId: string,
  roleName: string
): Promise<{
  sub: string;
  numberCode: string | undefined;
  roleName: string;
  jti: string;
  needSetPassword?: boolean;
}> {
  let accountCode =
    roleName === AppConfig.ROLE.ADMIN 
      ? undefined 
      : await this.resolveAccountCodeByRole(accountId, roleName);
  if (accountCode === null) accountCode = undefined;

  return {
    sub: accountId,
    numberCode: accountCode,
    roleName,
    jti: uuidv4()
  };
}
```

**Questions:**

**Q1.1.1** Tại sao cần kiểm tra `accountCode === null` và set thành `undefined`? Không thể dùng nullish coalescing (`accountCode ?? undefined`) không?

**Q1.1.2** Nếu `resolveAccountCodeByRole()` throw exception, payload building sẽ fail. Có thể wrap try-catch không? Hay để exception propagate?

**Q1.1.3** `jti` (JWT ID) được tạo bằng uuid. Tại sao cần jti? Nó có unique constraint trong database không?

**Q1.1.4** Nếu cùng 1 user login 2 lần cùng lúc, 2 JWT có jti khác nhau. Cái nào được dùng?

---

### 1.2 ExamService.create() - Transaction Flow

**Code snippet:**

```typescript
return this.prisma.$transaction(
  async (tx) => {
    await this.validateExamSet(tx, examDto.examSetId);

    const exam = await this.createExamRecord(tx, examData, createdBy, status);
    const questionIdMap = await this.insertParts(tx, exam.examId, parsedJson);

    if (answers?.length) {
      await this.upsertAnswers(tx, exam.examId, answers, questionIdMap);
    }

    return exam;
  },
  { timeout: 20000 }
);
```

**Questions:**

**Q1.2.1** Tại sao cần `questionIdMap`? Giải thích flow của nó.

Ans: Sau khi insert part, question, và questionGroup, cần map từ `{partIndex}_{questionNumber}` → `questionId` để có thể insert QuestionAnswer.

**Q1.2.2** Timeout 20000ms (20s) có đủ không? Nếu PDF có 200 trang sẽ slow không?

**Q1.2.3** Nếu transaction fail ở step 3 (insertParts), examSet validation đã done. Có redundant check không?

**Q1.2.4** Làm sao để optimize transaction này? Có thể parallel insert parts không?

---

### 1.3 ExamDraftService - Redis Hash vs String

**Code snippet:**

```typescript
private buildDraftHashFields(changes: Array<Record<string, unknown>>): Record<string, string> {
  const fields: Record<string, string> = {};

  for (const change of changes ?? []) {
    if (!change || typeof change !== 'object') continue;

    const key = this.resolveDraftFieldKey(change);
    if (!key) continue;

    try {
      fields[key] = JSON.stringify(change);
    } catch {
      // Bỏ qua nếu không serialize được
    }
  }

  return fields;
}
```

**Questions:**

**Q1.3.1** Tại sao dùng Redis Hash (HSET) thay vì String (SET)? Lợi ích gì?

Ans: 
- Hash cho phép update từng field mà không overwrite toàn bộ
- Tiết kiệm memory nếu draft có nhiều câu hỏi
- Dễ dàng query draft của 1 câu hỏi cụ thể

**Q1.3.2** Field key format `qid:{questionId}`. Tại sao không dùng `{examSessionId}:{studentId}:{questionId}`?

**Q1.3.3** Nếu 1 question answer là object to (500 bytes), 100 questions = 50KB. Có vấn đề gì không?

**Q1.3.4** Làm sao để denormalize draft từ Redis về client efficiently? Có thể HGETALL fail nếu hash to không?

---

### 1.4 Face Verification Async Queue

**Code snippet:**

```typescript
async faceVerification(data: IFaceVerificationJobData) {
  await this.faceVerificationQueue.add(
    QueueName.FACE_VERIFICATION_QUEUE,
    {
      examAttemptId: data.examAttemptId,
      accountId: data.accountId,
      occurredAt: data.occurredAt,
      imageBuffer: data.imageBuffer
    },
    {
      attempts: 2,
      backoff: { type: 'fixed', delay: 3000 },
      removeOnComplete: true,
      removeOnFail: 50
    }
  );
}
```

**Questions:**

**Q1.4.1** Tại sao dùng async queue thay vì sync API call?

Ans: 
- Không block HTTP response
- Retry automatic
- Load balancing

**Q1.4.2** Retry 2 times + fixed delay 3000ms. Nếu fail, có alert không? Học viên có biết face verification fail không?

**Q1.4.3** `removeOnComplete: true` có nghĩa gì? Job sẽ được xóa khỏi queue sau khi complete?

**Q1.4.4** `removeOnFail: 50` là gì? Lưu 50 failed jobs gần nhất?

---

## 2. Live Coding Problems

### 2.1 Problem: Implement Exam Shuffle Algorithm

**Problem:**

Implement hàm `shuffleExam()` để shuffle exam data. Requirements:

1. Shuffle các Part (không shuffle)
2. Shuffle các Question trong mỗi Part (shuffle)
3. Shuffle `options` của mỗi Question (shuffle)
4. Seed dựa trên `examAttemptId` (deterministic)
5. Shuffle của 2 attempts khác nhau phải khác nhau

**Input:**
```typescript
interface Part {
  partId: string;
  partIndex: number;
  questions: {
    questionId: string;
    questionNumber: number;
    options: { label: string; text: string }[];
  }[];
}

// examAttemptId = "uuid-123"
// parts = [Part1, Part2, Part3]
```

**Output:**
```typescript
// Trả về ExamData với parts shuffled
{
  parts: [
    {
      ...Part1_shuffled,
      questions: [q3, q1, q2], // Shuffled
      questions[0].options: [{label: "C", text: "..."}, {label: "A", text: "..."}, ...] // Shuffled
    },
    ...
  ]
}
```

**Hint:** Dùng seeded random generator (e.g., `seedrandom` package)

**Solution Outline:**

```typescript
import seedrandom from 'seedrandom';

export class ExamShuffleHelper {
  static shuffleExam(parts: Part[], examAttemptId: string): Part[] {
    const rng = seedrandom(examAttemptId); // Deterministic RNG
    
    return parts.map(part => ({
      ...part,
      questions: this.shuffleArray(part.questions, rng).map(q => ({
        ...q,
        options: this.shuffleArray(q.options, rng)
      }))
    }));
  }

  private static shuffleArray<T>(array: T[], rng: () => number): T[] {
    const shuffled = [...array];
    for (let i = shuffled.length - 1; i > 0; i--) {
      const j = Math.floor(rng() * (i + 1));
      [shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]];
    }
    return shuffled;
  }
}
```

---

### 2.2 Problem: Implement Draft Auto-save Logic

**Problem:**

Implement auto-save logic cho draft answers. Requirements:

1. Client gửi changes mỗi 10 giây
2. Chỉ gửi nếu có thay đổi (diff-based)
3. Server lưu vào Redis Hash với TTL = session duration
4. Nếu client mất kết nối > 15s, mark as DISCONNECTED
5. Khi client reconnect, khôi phục draft từ Redis

**Input (từ client):**
```typescript
interface SaveDraftRequest {
  examSessionId: string;
  studentId: string;
  changes: Array<{
    questionId: string;
    answer: string | string[]; // Phụ thuộc loại câu hỏi
    updatedAtTs: number; // timestamp
  }>;
}
```

**Backend Implementation:**

```typescript
async saveDraft(req: SaveDraftRequest) {
  const { examSessionId, studentId, changes } = req;

  // 1. Validate session is in progress
  const session = await this.examSessionService.findById(examSessionId);
  if (session.status !== 'IN_PROGRESS') {
    throw new BadRequestException('Session không diễn ra');
  }

  // 2. Calculate TTL = session endTime - now
  const endTime = new Date(session.date).getTime() + session.duration * 60 * 1000;
  const ttlSeconds = Math.ceil((endTime - Date.now()) / 1000);

  // 3. Save to Redis Hash
  await this.examDraftService.saveDraft(
    examSessionId,
    studentId,
    changes,
    ttlSeconds
  );

  // 4. Update presence key (heartbeat)
  const presenceKey = getPresenceKey(examSessionId, studentId);
  await this.redisService.setex(presenceKey, 15, '1'); // 15s timeout

  return { success: true, savedAt: new Date() };
}
```

**Key Points:**

- TTL tính động từ session end time
- Presence key dùng để detect disconnect
- Redis expiry event trigger disconnect handler

---

### 2.3 Problem: Implement Multi-chunk PDF Processing

**Problem:**

Gemini API có limit, không thể xử lý PDF > 50 trang một lần. Implement chunk-based processing:

1. Split PDF thành chunks (4 trang/chunk, 1 trang overlap)
2. Gọi Gemini song song (max 3 concurrent)
3. Merge kết quả từ các chunk
4. Dedup questions
5. Normalize data

**Input:**
```typescript
const pdf = Buffer.from(...); // 100 pages
const prompt = "Generate exam from this PDF...";
```

**Expected Flow:**

```
PDF (100 pages)
  ↓
Chunks: [pages 1-4, pages 4-8, pages 8-12, ..., pages 96-100]
  ↓
Parallel calls (3 concurrent):
  - Chunk 1 → Gemini → result1
  - Chunk 2 → Gemini → result2
  - Chunk 3 → Gemini → result3
  ↓
Merge results → { parts, questions, ... }
  ↓
Dedup & normalize → Final exam data
```

**Implementation Skeleton:**

```typescript
async generateExamFromPdf(pdf: Buffer, prompt: string) {
  // 1. Count pages
  const pageCount = await this.getPdfPageCount(pdf);

  // 2. If small, process single request
  if (pageCount < 4) {
    return this.processWithGemini(pdf, prompt);
  }

  // 3. Create chunks
  const chunks = await this.createChunks(pdf, chunkSize: 4, overlap: 1);

  // 4. Process parallel with rate limit
  const limiter = pLimit(3);
  const results = await Promise.all(
    chunks.map(chunk => limiter(() => this.processWithGemini(chunk, prompt)))
  );

  // 5. Merge & dedup
  return this.mergeResults(results);
}
```

---

### 2.4 Problem: Implement Fraud Detection State Machine

**Problem:**

Implement fraud detection logic:

1. Phát hiện violation → log FraudDetail
2. Accumulate violations → update fraudLevel
3. Automatic actions based on level:
   - 1 violation → WARNING
   - 3 violations → SUSPENSION (auto submit)
   - Khi nào auto emit event đến teacher?

**State Machine:**

```
CLEAN (warningCount=0)
  ↓ violation 1
WARNED (warningCount=1) → emit WARNING event
  ↓ violation 2
WARNED (warningCount=2)
  ↓ violation 3
SUSPENDED (warningCount=3) → emit SUSPENSION event → auto submit
```

**Implementation:**

```typescript
async handleViolation(examAttemptId: string, violationType: FraudType) {
  const attempt = await this.examAttemptRepo.findById(examAttemptId);

  // 1. Log violation
  await this.fraudDetailService.create({
    examAttemptId,
    fraudType: violationType,
    occurredAt: new Date()
  });

  // 2. Increment warningCount
  const updatedAttempt = await this.examAttemptRepo.update(examAttemptId, {
    warningCount: { increment: 1 }
  });

  // 3. Determine action based on warningCount
  if (updatedAttempt.warningCount === 1) {
    // WARNING
    this.examSessionSseService.notifySessionHandlingEvent(
      attempt.examSessionId,
      attempt.studentId,
      'WARNING'
    );
  } else if (updatedAttempt.warningCount >= 3) {
    // SUSPENSION → auto submit
    await this.forceSubmitExamAttempt(examAttemptId);
  }

  // 4. Update fraudLevel
  const fraudLevel = this.determineFraudLevel(updatedAttempt.warningCount);
  await this.examAttemptRepo.update(examAttemptId, { fraudLevel });

  // 5. Notify teacher
  this.examSessionSseService.notifyViolation(attempt.examSessionId, {
    studentId: attempt.studentId,
    violationType,
    details: { warningCount: updatedAttempt.warningCount }
  });
}

private determineFraudLevel(warningCount: number): FraudLevel {
  if (warningCount === 0) return null;
  if (warningCount <= 1) return FraudLevel.LOW;
  if (warningCount <= 2) return FraudLevel.MEDIUM;
  return FraudLevel.HIGH;
}
```

---

## 3. Database Query Problems

### 3.1 Problem: Get Exam Results with Statistics

**Problem:**

Query để lấy exam results của 1 ca thi với thống kê:

1. Tổng sinh viên tham gia
2. Sinh viên hoàn thành
3. Điểm trung bình
4. Điểm cao nhất, thấp nhất
5. Phân bố điểm (A/B/C/D)

**SQL Query:**

```sql
SELECT
  es.exam_session_id,
  es.exam_session_code,
  COUNT(DISTINCT ea.student_id) as total_students,
  COUNT(DISTINCT CASE WHEN ea.status = 'COMPLETED' THEN ea.student_id END) as completed,
  AVG(CAST(r.score_detail::json->>'totalScore' AS FLOAT)) as avg_score,
  MAX(CAST(r.score_detail::json->>'totalScore' AS FLOAT)) as max_score,
  MIN(CAST(r.score_detail::json->>'totalScore' AS FLOAT)) as min_score,
  COUNT(CASE WHEN CAST(r.score_detail::json->>'totalScore' AS FLOAT) >= 80 THEN 1 END) as grade_a,
  COUNT(CASE WHEN CAST(r.score_detail::json->>'totalScore' AS FLOAT) >= 60 AND CAST(r.score_detail::json->>'totalScore' AS FLOAT) < 80 THEN 1 END) as grade_b
FROM exam_sessions es
LEFT JOIN exam_attempts ea ON es.exam_session_id = ea.exam_session_id
LEFT JOIN results r ON ea.exam_attempt_id = r.exam_attempt_id
WHERE es.exam_session_id = $1
GROUP BY es.exam_session_id, es.exam_session_code;
```

**Prisma Query:**

```typescript
async getExamSessionStatistics(examSessionId: string) {
  const results = await this.prisma.examSession.findUnique({
    where: { examSessionId },
    include: {
      examAttempts: {
        include: {
          result: true
        }
      }
    }
  });

  // Process in-memory
  const stats = {
    totalStudents: results.examAttempts.length,
    completed: results.examAttempts.filter(a => a.status === 'COMPLETED').length,
    avgScore: this.calculateAverage(results.examAttempts),
    // ...
  };

  return stats;
}
```

**Problem:** Query sau sẽ load toàn bộ attempts. Nếu 1000 students, sẽ load 1000 records. Optimize bằng cách dùng aggregation trực tiếp trong SQL.

---

### 3.2 Problem: Get Student Draft Answers Efficiently

**Problem:**

Lấy draft answers của student khi join exam. Student trước đó đã làm được 50 questions, muốn khôi phục tất cả.

**Redis Hash Structure:**
```
exam:draft:{sessionId}:{studentId}
  qid:q1 → {"questionId": "q1", "answer": "A", "updatedAtTs": 1234567}
  qid:q2 → {"questionId": "q2", "answer": ["A", "C"], "updatedAtTs": 1234567}
  ...
```

**Implementation:**

```typescript
async getDraftAnswers(sessionId: string, studentId: string) {
  const key = getCacheExamDraftKey(sessionId, studentId);
  
  // Option 1: HGETALL (one command)
  const hashData = await this.redisService.hgetall(key);
  const answers = Object.values(hashData).map(item => JSON.parse(item));

  // Option 2: HSCAN (streaming, for large hash)
  const cursor = "0";
  const answers = [];
  do {
    const [newCursor, items] = await this.redisService.hscan(key, cursor, "COUNT", 100);
    answers.push(...items.map(JSON.parse));
    cursor = newCursor;
  } while (cursor !== "0");

  return answers;
}
```

**Question:** Nếu draft có 1000 fields, HGETALL có load hết memory không?

---

## 4. API Design Problems

### 4.1 Problem: Design POST /exam-session/:id/join Endpoint

**Problem:**

Design endpoint cho student join exam session. Requirements:

1. Validate student registration
2. Face verification (if required)
3. Random exam code
4. Create ExamAttempt
5. Return exam data
6. Notify teacher via SSE

**Request/Response Design:**

```typescript
// POST /exam-session/:id/join

// Request body
{
  studentId: "uuid",
  faceImage?: Base64 // If isCameraRequired
}

// Response
{
  success: true,
  data: {
    examAttemptId: "uuid",
    examId: "uuid",
    examCode: "M01",
    duration: 120, // minutes
    parts: [...],
    questions: [...],
    startTime: "2024-01-01T10:00:00Z",
    isCameraRequired: true,
    cachedSubmission: { // Khôi phục draft
      answers: [...]
    }
  },
  errors?: []
}
```

**Implementation Points:**

1. Transaction: registration check → face verify → create attempt
2. Async: emit SSE notification sau khi transaction success
3. Cache: return exam data từ cache nếu possible
4. Error handling: face verification fail → 400 "Face verification failed"

---

### 4.2 Problem: Design SSE Event Stream

**Problem:**

Design SSE events cho teacher dashboard monitoring exam session.

**Event Types:**

```typescript
interface SessionEvent {
  examSessionId: string;
  type: 
    | 'STUDENT_JOINED'
    | 'STUDENT_SUBMITTED'
    | 'STUDENT_VIOLATION'
    | 'SESSION_STATUS_CHANGED'
    | 'ATTEMPT_PAUSED'
    | 'ATTEMPT_RESUMED'
    | 'ATTEMPT_DISCONNECTED';
  data: any;
  timestamp: Date;
}
```

**Event Examples:**

```typescript
// STUDENT_JOINED
{
  type: 'STUDENT_JOINED',
  data: { studentId: '...', studentName: '...' }
}

// STUDENT_VIOLATION
{
  type: 'STUDENT_VIOLATION',
  data: { 
    studentId: '...', 
    violationType: 'FACE_MISMATCH',
    details: { accuracy: 0.75 }
  }
}

// ATTEMPT_DISCONNECTED
{
  type: 'ATTEMPT_DISCONNECTED',
  data: {
    studentId: '...',
    disconnectedAt: '2024-01-01T10:05:00Z',
    previousStatus: 'IN_PROGRESS'
  }
}
```

**Implementation:**

```typescript
@Sse('exam-session/:id/subscribe')
subscribeToSession(@Param('id') sessionId: string): Observable<MessageEvent> {
  return this.examSessionSseService.subscribeToSession(sessionId).pipe(
    map(event => ({
      data: event // Browser SSE format
    }))
  );
}
```

---

## 5. Debugging Scenarios

### 5.1 Scenario: Exam Shuffle Inconsistency

**Scenario:**

Student 1 và Student 2 cùng làm exam, cùng exam code. Nhưng question order khác nhau.

**Debug Steps:**

1. Check seed: `examAttemptId` đúng không?
2. Check random generator: `seedrandom(examAttemptId)` có deterministic không?
3. Check array mutation: có modify original array không?

**Suspect Code:**

```typescript
// ❌ WRONG - modifies original array
static shuffleExam(parts: Part[], examAttemptId: string): Part[] {
  const rng = seedrandom(examAttemptId);
  for (let i = parts.length - 1; i > 0; i--) {
    const j = Math.floor(rng() * (i + 1));
    [parts[i], parts[j]] = [parts[j], parts[i]]; // Mutate!
  }
  return parts;
}

// ✅ CORRECT - create copy first
static shuffleExam(parts: Part[], examAttemptId: string): Part[] {
  const rng = seedrandom(examAttemptId);
  const shuffled = [...parts]; // Copy!
  for (let i = shuffled.length - 1; i > 0; i--) {
    const j = Math.floor(rng() * (i + 1));
    [shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]];
  }
  return shuffled;
}
```

---

### 5.2 Scenario: Draft Not Being Recovered

**Scenario:**

Student 1 disconnect → reconnect, nhưng draft answers không được khôi phục.

**Debug Checklist:**

1. Redis TTL: key còn tồn tại không?
   ```
   PTTL exam:draft:{sessionId}:{studentId}
   ```

2. Data format: Hash values có phải JSON valid không?
   ```
   HGETALL exam:draft:{sessionId}:{studentId}
   ```

3. Application logic: `getDraftAnswers()` có query Redis không?

4. Race condition: auto-save happening sau khi student join?

**Common Issues:**

- Draft expired vì TTL quá ngắn
- Redis connection lost, silent fail
- Draft key naming mismatch (studentId format)

---

### 5.3 Scenario: Fraud Detection False Positive

**Scenario:**

Student bị phát hiện 2 khuôn mặt liên tục, bị suspension.

**Root Causes:**

1. **Background movement**: Background di chuyển, detected as 2 faces
   - Fix: Threshold tùy chỉnh, require stable 2 faces > 1 second

2. **Reflection**: Mirror/glasses reflect face, detected as 2
   - Fix: Filter by face confidence score

3. **Poor lighting**: Face partially visible, library incorrectly detects
   - Fix: Add manual override, teacher can dismiss violation

**Debug Query:**

```sql
SELECT * FROM fraud_details 
WHERE exam_attempt_id = 'X' 
AND fraud_type = 'MULTIPLE_FACES_DETECTED'
ORDER BY occurred_at DESC
LIMIT 10;
```

---

### 5.4 Scenario: Gemini API Timeout

**Scenario:**

Sinh PDF 80 trang, gọi Gemini 6 chunk. Chunk 4 timeout sau 30s.

**Debug:**

1. Retry logic: job auto-retry 2 times?
2. Backoff: delay 3s giữa retries có đủ không?
3. Timeout config: 30s có quá ngắn không?

**Optimization:**

```typescript
// Increase timeout for large PDFs
const getTimeout = (pageCount: number) => {
  if (pageCount > 100) return 60000; // 60s
  if (pageCount > 50) return 45000;  // 45s
  return 30000; // 30s
};

// Exponential backoff
{
  attempts: 3,
  backoff: {
    type: 'exponential',
    delay: 1000
  }
}
```

---

### 5.5 Scenario: SSE Memory Leak

**Scenario:**

Production server memory tăng từ 500MB → 4GB trong 1 ngày, SSE channels không được cleanup.

**Root Cause:**

```typescript
// ❌ Subscriber không unsubscribe properly
const subscription = this.examSessionSseService.subscribeToSession(sessionId);
subscription.subscribe(
  event => {
    console.log(event);
    // Missing unsubscribe on component destroy!
  }
);
```

**Fix:**

```typescript
// ✅ Unsubscribe on component destroy
export class SessionMonitorComponent implements OnInit, OnDestroy {
  private subscription: Subscription;

  ngOnInit() {
    this.subscription = this.examSessionSseService.subscribeToSession(sessionId)
      .subscribe(event => { /* ... */ });
  }

  ngOnDestroy() {
    this.subscription.unsubscribe(); // Clean up!
  }
}
```

**Backend Monitoring:**

```typescript
// Monitor SSE stats
setInterval(() => {
  const stats = this.examSessionSseService.getStats();
  console.log(`Active sessions: ${stats.activeSessions}, Subscribers: ${stats.totalSubscribers}`);
}, 60000);
```

---

## 🎯 **Summary: Top 10 Interview Challenges**

| # | Challenge | Difficulty | Time |
|---|---|---|---|
| 1 | Implement Exam Shuffle Algorithm | Medium | 20 min |
| 2 | Design Auto-save Draft Logic | Medium | 25 min |
| 3 | Multi-chunk PDF Processing | Hard | 30 min |
| 4 | Fraud Detection State Machine | Hard | 25 min |
| 5 | Optimize Database Queries | Medium | 20 min |
| 6 | SSE Event Stream Design | Medium | 20 min |
| 7 | Debug Memory Leak in SSE | Medium | 25 min |
| 8 | JWT & Token Rotation Logic | Medium | 20 min |
| 9 | Transaction Handling in Exam Creation | Hard | 30 min |
| 10 | Real-time Monitoring Dashboard | Hard | 35 min |

---

**All Solutions & Code Available in Backend Repository**  
**Last Updated:** August 2026

