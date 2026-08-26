# 🎯 **iTEST Backend Interview Q&A - Part 2**
## Tiếp Tục: Exam Session, SSE, Fraud Detection, Face Verification & Hơn Nữa

---

## 6️⃣ **EXAM SESSION & ATTEMPT (Tiếp Tục)**

### Q6.1.2: Session có những thông tin nào? (date, room, duration, isCameraRequired, ...)

**Trả lời:**

**ExamSession fields (từ schema.prisma):**

```prisma
model ExamSession {
  // Identifiers
  examSessionId     String    @id @default(cuid())
  examSessionCode   String    @unique           # Unique code per session
  examSetId         String                      # Which exam set
  
  // Metadata
  date              DateTime                    # Start time
  duration          Int                         # Minutes (120 = 2 hours)
  room              String                      # Location
  capacity          Int                         # Max students
  
  // Settings
  isCameraRequired  Boolean   @default(false)   # Mandatory camera monitoring
  isLocked          Boolean   @default(false)   # Lock session (no more joins)
  
  // Status
  status            ExamSessionStatus           # NOT_STARTED, IN_PROGRESS, PAUSE, FINISHED
  
  // Relationships
  examAttempts      ExamAttempt[]              # Students attempting
  examRegistrations ExamRegistration[]        # Registrations
  teachers          ExamSessionTeacher[]      # Proctors
  
  // Audit
  createdAt         DateTime  @default(now())
  updatedAt         DateTime  @updatedAt
  deletedAt         DateTime?                  # Soft delete
}
```

**Enum statuses:**

```prisma
enum ExamSessionStatus {
  NOT_STARTED        # Session not started yet
  IN_PROGRESS        # Active session (students can join/take exam)
  PAUSE              # Paused (students can't submit)
  FINISHED           # Ended (no more joins allowed)
}
```

**Ví dụ complete:**

```json
{
  "examSessionId": "sess-20240115-001",
  "examSessionCode": "CS101-2024-Spring-T1",
  "examSetId": "examset-cs101",
  "date": "2024-01-15T08:00:00Z",
  "duration": 120,              // 2 hours
  "room": "Hội trường A, Tầng 2",
  "capacity": 100,
  "isCameraRequired": true,     // Bắt webcam
  "isLocked": false,            // Session mở
  "status": "IN_PROGRESS",
  "createdAt": "2024-01-10T10:00:00Z",
  "updatedAt": "2024-01-15T08:10:00Z",
  "examAttempts": [
    { "studentId": "s001", "status": "IN_PROGRESS" },
    { "studentId": "s002", "status": "IN_PROGRESS" },
    { "studentId": "s003", "status": "SUBMITTED" }
  ],
  "examRegistrations": [
    { "studentId": "s001", "status": "REGISTERED" },
    { "studentId": "s002", "status": "REGISTERED" }
  ],
  "teachers": [
    { "teacherId": "t001", "role": "PROCTOR" }
  ]
}
```

**Key features:**

1. **date + duration**: Tính thời gian kết thúc
   ```typescript
   endTime = new Date(session.date).getTime() + session.duration * 60 * 1000
   ```

2. **capacity**: Giám sát số lượng
   ```typescript
   isFull = examAttempts.length >= capacity
   ```

3. **isCameraRequired**: Bắt face verification
   ```typescript
   if (session.isCameraRequired) {
     // Verify face before joining
   }
   ```

4. **isLocked**: Không cho join thêm
   ```typescript
   if (session.isLocked) {
     throw new BadRequestException('Session locked')
   }
   ```

5. **status flow**:
   ```
   NOT_STARTED → IN_PROGRESS → PAUSE ↔ IN_PROGRESS → FINISHED
   ```

---

### Q6.1.3: Trạng thái session có những giá trị nào? Quy trình chuyển đổi là gì?

**Trả lời:**

**Status values & transitions (từ schema):**

```
NOT_STARTED
     ↓
     └─→ IN_PROGRESS (admin click "start session")
              ↓
              ├─→ PAUSE (admin click "pause") ↔ IN_PROGRESS (admin click "resume")
              │
              └─→ FINISHED (time expires or admin click "end session")
```

**Detailed transitions:**

```typescript
// ExamSessionStatus enum
enum ExamSessionStatus {
  NOT_STARTED = 'NOT_STARTED',   // Initial state
  IN_PROGRESS = 'IN_PROGRESS',   // Students can join/submit
  PAUSE = 'PAUSE',               // Exam paused, no submission
  FINISHED = 'FINISHED'          // Exam ended
}

// State machine logic
type StatusTransition = {
  from: ExamSessionStatus;
  to: ExamSessionStatus;
  action: string;
};

const validTransitions: StatusTransition[] = [
  { from: 'NOT_STARTED', to: 'IN_PROGRESS', action: 'START' },
  { from: 'IN_PROGRESS', to: 'PAUSE', action: 'PAUSE' },
  { from: 'PAUSE', to: 'IN_PROGRESS', action: 'RESUME' },
  { from: 'IN_PROGRESS', to: 'FINISHED', action: 'END' },
  { from: 'PAUSE', to: 'FINISHED', action: 'END' },
];
```

**From source code (`exam-session.service.ts`):**

```typescript
async changeExamSessionStatus(sessionId: string, newStatus: ExamSessionStatus) {
  const session = await this.examSessionRepo.findById(sessionId);
  
  // Validate transition
  if (!this.isValidTransition(session.status, newStatus)) {
    throw new BadRequestException('Invalid status transition');
  }

  // Update session
  const updated = await this.examSessionRepo.updateStatus(sessionId, newStatus);

  // Trigger side effects
  if (newStatus === 'IN_PROGRESS') {
    await this.handleSessionStart(sessionId);
  } else if (newStatus === 'PAUSE') {
    await this.handleSessionPause(sessionId);
  } else if (newStatus === 'RESUME') {
    await this.handleSessionResume(sessionId);
  } else if (newStatus === 'FINISHED') {
    await this.handleSessionFinish(sessionId);
  }

  return updated;
}

private isValidTransition(from: ExamSessionStatus, to: ExamSessionStatus): boolean {
  const validTransitions = {
    NOT_STARTED: ['IN_PROGRESS'],
    IN_PROGRESS: ['PAUSE', 'FINISHED'],
    PAUSE: ['IN_PROGRESS', 'FINISHED'],
    FINISHED: []  // Can't transition from FINISHED
  };

  return validTransitions[from]?.includes(to) ?? false;
}
```

**Side effects per status:**

```typescript
private async handleSessionStart(sessionId: string) {
  // 1. Notify all registered students
  this.examSessionSseService.notifySessionStatusChanged(sessionId, 'IN_PROGRESS');
  
  // 2. Start heartbeat monitoring
  // 3. Start timer for auto-finish
}

private async handleSessionPause(sessionId: string) {
  // 1. Pause all student attempts (consumedTime frozen)
  const attempts = await this.examAttemptRepo.findBySessionId(sessionId);
  await Promise.all(
    attempts.map(a => this.examAttemptRepo.update(a.attemptId, { 
      status: 'PAUSED',
      consumedTime: { increment: Date.now() - a.lastResumedAt }
    }))
  );
  
  // 2. Notify students (display "exam paused" screen)
  this.examSessionSseService.notifyAttemptPaused(sessionId, { attempts });
}

private async handleSessionResume(sessionId: string) {
  // 1. Resume all paused attempts
  const attempts = await this.examAttemptRepo.findBySessionId(sessionId);
  await Promise.all(
    attempts.map(a => this.examAttemptRepo.update(a.attemptId, { 
      status: 'IN_PROGRESS',
      lastResumedAt: new Date()
    }))
  );
  
  // 2. Reset timer
  this.examSessionSseService.notifyAttemptResumed(sessionId, { attempts });
}

private async handleSessionFinish(sessionId: string) {
  // 1. Auto-submit all remaining attempts
  const pending = await this.examAttemptRepo.findBySessionId(sessionId, { 
    status: 'IN_PROGRESS' 
  });
  
  await Promise.all(
    pending.map(a => this.forceSubmitAttempt(a.attemptId))
  );
  
  // 2. Close SSE channels
  this.examSessionSseService.closeSession(sessionId);
  this.examSessionSseService.closeAllStudentChannels(sessionId);
  
  // 3. Calculate results
  await this.resultService.calculateSessionResults(sessionId);
}
```

---

### Q6.3.1: `ExamAttempt` đại diện cho cái gì? 1 sinh viên làm 1 ca thi = 1 attempt à?

**Trả lời:**

**ĐÚng! 1 attempt = 1 sinh viên × 1 ca thi**

```prisma
model ExamAttempt {
  examAttemptId     String             @id
  studentId         String             # Student taking exam
  examSessionId      String            # Session (session with multiple students)
  examId            String             # Which exam (randomly selected from ExamSet)
  
  // Unique constraint
  @@unique([studentId, examSessionId])  # 1 sinh viên chỉ làm 1 lần trong session
  
  // Status & timing
  status            ExamAttemptStatus  # IN_PROGRESS, PAUSED, SUBMITTED, COMPLETED
  startTime         DateTime           # When student joined
  submitTime        DateTime?          # When submitted
  duration          Int                # Session duration (minutes)
  consumedTime      Int                # Time used (seconds)
  lastResumedAt     DateTime           # Last resume timestamp
  
  // Fraud tracking
  warningCount      Int    @default(0) # Number of warnings
  fraudLevel        FraudLevel?        # LOW, MEDIUM, HIGH
  
  // Results
  result            Result?            # Grading result
  
  // Relationships
  studentAnswers    StudentAnswer[]    # Answers to questions
  fraudDetails      FraudDetail[]      # Violations detected
  
  // Audit
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
}
```

**Ví dụ:**

```
ExamSession (CA THI CS101)
├─ Student 1 (S001)
│  └─ ExamAttempt 1
│     ├─ examId: "exam-001" (Exam A)
│     ├─ status: "IN_PROGRESS"
│     ├─ startTime: 2024-01-15 08:00
│     ├─ consumedTime: 1200 (20 minutes)
│     └─ studentAnswers: [q1→A, q2→B, q3→[A,C], ...]
│
├─ Student 2 (S002)
│  └─ ExamAttempt 2
│     ├─ examId: "exam-002" (Exam B - different from S001)
│     ├─ status: "SUBMITTED"
│     ├─ startTime: 2024-01-15 08:00
│     ├─ submitTime: 2024-01-15 10:05
│     ├─ consumedTime: 7500 (125 minutes)
│     └─ result: { score: 85, status: "NOT_GRADED" }
│
└─ Student 3 (S003)
   └─ ExamAttempt 3
      ├─ examId: "exam-001" (Same as S001)
      ├─ status: "DISCONNECTED"
      ├─ warningCount: 2
      ├─ fraudLevel: "MEDIUM"
      └─ fraudDetails: [
           { fraudType: "NO_FACE_DETECTED", ... },
           { fraudType: "TAB_SWITCHING", ... }
         ]
```

**Constraints:**

```prisma
@@unique([studentId, examSessionId])
```

Meaning: **1 sinh viên chỉ có 1 attempt mỗi session**

```typescript
// ✅ Allowed
await prisma.examAttempt.create({
  data: {
    studentId: 's001',
    examSessionId: 'sess-001',
    examId: 'exam-001'
  }
});

// ❌ Throws error - duplicate unique key
await prisma.examAttempt.create({
  data: {
    studentId: 's001',
    examSessionId: 'sess-001',  // Same session
    examId: 'exam-002'          // Different exam but same student+session
  }
});
```

---

### Q6.3.2: Attempt status có thứ tự chuyển đổi nào?

**Trả lời:**

**Attempt status flow:**

```
┌─────────────────────────────────────────────────────────┐
│ ExamAttemptStatus state machine                         │
└─────────────────────────────────────────────────────────┘

                    IN_PROGRESS (doing exam)
                       ↓  ↑
                    PAUSED (frozen)
                    
                       ↓
                    
      SUBMITTED (submitted by student or timeout)
      ↓
      ├─→ COMPLETED (auto-scored, not graded yet)
      │
      └─→ DISCONNECTED (lost connection > 15s)
               ↓
               └─→ Can reconnect → IN_PROGRESS

Also:
- SUBMITTED → force submitted (teacher or violation)
- COMPLETED → after auto-scoring, before teacher review
```

**From schema:**

```prisma
enum ExamAttemptStatus {
  IN_PROGRESS        # Student actively taking exam
  PAUSED             # Session paused or student timed out temporarily
  SUBMITTED          # Student submitted answers
  COMPLETED          # Auto-scoring done, waiting for essay grading
  DISCONNECTED       # Lost connection (> 15 seconds)
  CANCELLED          # Attempt cancelled by admin
}
```

**Transitions:**

```typescript
type ValidTransition = [from: ExamAttemptStatus, to: ExamAttemptStatus, reason: string];

const validAttemptTransitions: ValidTransition[] = [
  // Normal flow
  ['IN_PROGRESS', 'SUBMITTED', 'student_submit'],
  ['IN_PROGRESS', 'PAUSED', 'session_paused'],
  ['PAUSED', 'IN_PROGRESS', 'session_resumed'],
  ['IN_PROGRESS', 'DISCONNECTED', 'heartbeat_timeout'],
  
  // Recovery
  ['DISCONNECTED', 'IN_PROGRESS', 'student_reconnected'],
  
  // Auto-submit
  ['IN_PROGRESS', 'SUBMITTED', 'time_expired'],
  ['PAUSED', 'SUBMITTED', 'session_ended_while_paused'],
  
  // Completion
  ['SUBMITTED', 'COMPLETED', 'auto_scoring_done'],
  
  // Cancellation
  ['IN_PROGRESS', 'CANCELLED', 'admin_cancelled'],
  ['SUBMITTED', 'CANCELLED', 'admin_cancelled'],
];
```

**Time-based transitions:**

```typescript
// Timer expired
if (Date.now() >= endTime) {
  await this.forceSubmitAttempt(attemptId);  // AUTO-SUBMIT
  // status: IN_PROGRESS → SUBMITTED
}

// Heartbeat timeout
if (Date.now() - lastHeartbeat > 15000) {  // 15 seconds
  await this.markAttemptDisconnected(attemptId);
  // status: IN_PROGRESS → DISCONNECTED
}

// Auto-submit after 30s disconnected (or teacher action)
if (Date.now() - disconnectedAt > 30000) {
  await this.forceSubmitDisconnectedAttempt(attemptId);
  // status: DISCONNECTED → SUBMITTED
}
```

---

## 7️⃣ **REAL-TIME COMMUNICATION (SSE)**

### Q7.1.1: SSE (Server-Sent Events) được implement như thế nào?

**Trả lời:**

**SSE Implementation từ `exam-session-sse.service.ts`:**

```typescript
@Injectable()
export class ExamSessionSseService {
  // In-memory channels
  private sessions = new Map<string, SessionState>();        // Teacher channel
  private studentChannels = new Map<string, SessionState>();  // Student channel

  interface SessionState {
    subject: Subject<ExamSessionEvent>;  // RxJS Subject
    subscriberCount: number;
  }
}
```

**Architecture:**

```
┌──────────────────────────────────────────┐
│         Browser Client (Teacher)         │
│                                          │
│  GET /exam-session/sess-001/subscribe   │
│            ↓                             │
│    [SSE Connection Open]                │
│    (long-lived HTTP)                    │
│                                          │
│  Event 1: STUDENT_JOINED                │
│  Event 2: STUDENT_VIOLATION             │
│  Event 3: STUDENT_SUBMITTED             │
│  ...                                    │
└──────────────────────────────────────────┘
              ↑
              │ [SSE Stream]
              │
┌──────────────────────────────────────────┐
│      Backend (NestJS Controller)         │
│                                          │
│  @Sse('subscribe')                      │
│  subscribeToSession(sessionId): Observable
│    ↓                                     │
│  examSessionSseService.subscribeToSession
│    ↓                                     │
│  Returns: Observable<ExamSessionEvent>  │
└──────────────────────────────────────────┘
              ↑
              │ [Subject.next(event)]
              │
┌──────────────────────────────────────────┐
│     Backend Events (Other Services)      │
│                                          │
│  examAttemptService.handleSubmit()      │
│    → examSessionSseService.emitEvent()   │
│                                          │
│  fraudDetectionService.detectViolation()│
│    → examSessionSseService.emitEvent()   │
└──────────────────────────────────────────┘
```

**2-channel design:**

```typescript
// Channel 1: Teacher channel (broadcast)
private sessions = new Map<
  examSessionId: string,
  SessionState
>();

// Channel 2: Student channel (isolated per student)
private studentChannels = new Map<
  "${examSessionId}:${studentId}": string,
  SessionState
>();
```

**RxJS Observable + Subject:**

```typescript
// Subject = both Observable (data source) + Observer (can emit)
const subject = new Subject<ExamSessionEvent>();

// Emit event
subject.next({ type: 'STUDENT_JOINED', data: {...}, timestamp: new Date() });

// Subscribe to events
const subscription = subject.asObservable().subscribe(
  event => console.log('Received:', event),
  error => console.error('Error:', error),
  () => console.log('Completed')
);

// Unsubscribe
subscription.unsubscribe();
```

**Controller implementation:**

```typescript
// exam-session.controller.ts
@Sse('exam-session/:id/subscribe')
subscribeToSession(
  @Param('id') sessionId: string,
  @Query('role') role: 'TEACHER' | 'STUDENT',
  @Query('studentId') studentId?: string
): Observable<MessageEvent> {
  let eventStream: Observable<ExamSessionEvent>;

  if (role === 'TEACHER') {
    // Teacher subscribed to session
    eventStream = this.examSessionSseService.subscribeToSession(sessionId);
  } else {
    // Student subscribed to own channel
    eventStream = this.examSessionSseService.subscribeStudent(sessionId, studentId);
  }

  // Convert to MessageEvent for browser SSE
  return eventStream.pipe(
    map(event => ({
      id: event.timestamp.getTime().toString(),
      type: event.type,
      data: JSON.stringify(event.data)
    }))
  );
}
```

**Browser side:**

```javascript
// Teacher
const eventSource = new EventSource(
  `/exam-session/sess-001/subscribe?role=TEACHER`
);

eventSource.addEventListener('STUDENT_JOINED', (event) => {
  const data = JSON.parse(event.data);
  console.log(`Student ${data.studentName} joined`);
});

eventSource.addEventListener('STUDENT_VIOLATION', (event) => {
  const violation = JSON.parse(event.data);
  console.log(`Violation detected: ${violation.violationType}`);
});

eventSource.onerror = () => {
  console.error('Connection lost');
  // Browser auto-reconnect every 1s
};
```

---

### Q7.1.2: Có 2 channel riêng: teacher channel và student channel. Mục đích là gì?

**Trả lời:**

**Separation of channels:**

#### **Channel 1: Teacher Channel** (broadcast to all teachers)

```typescript
// exam-session-sse.service.ts
private sessions = new Map<string, SessionState>();

subscribeToSession(examSessionId: string): Observable<ExamSessionEvent> {
  return this.buildSubscription(this.sessions, examSessionId, `session:${examSessionId}`);
}

emitEvent(event: ExamSessionEvent): void {
  const state = this.sessions.get(event.examSessionId);
  if (!state) return;
  state.subject.next(event);  // Broadcast to all teachers
}
```

**Events broadcast:**

```typescript
notifyStudentJoined(examSessionId, studentData) {
  // All teachers see: Student X joined
  this.emitEvent({
    examSessionId,
    type: 'STUDENT_JOINED',
    data: { studentId: 's001', studentName: 'Nguyen A' },
    timestamp: new Date()
  });
}

notifyViolation(examSessionId, violationData) {
  // All teachers see: Student X has violation
  this.emitEvent({
    examSessionId,
    type: 'STUDENT_VIOLATION',
    data: {
      studentId: 's001',
      violationType: 'FACE_MISMATCH',
      details: { accuracy: 0.65 }
    },
    timestamp: new Date()
  });
}
```

#### **Channel 2: Student Channel** (isolated per student)

```typescript
private studentChannels = new Map<string, SessionState>();  // Key: `${sessionId}:${studentId}`

subscribeStudent(examSessionId: string, studentId: string): Observable<ExamSessionEvent> {
  const key = `${examSessionId}:${studentId}`;
  return this.buildSubscription(this.studentChannels, key, `student:${key}`);
}

emitToStudent(examSessionId: string, studentId: string, event: ExamSessionEvent): void {
  const key = `${examSessionId}:${studentId}`;
  const state = this.studentChannels.get(key);
  if (!state) return;
  state.subject.next(event);  // Only this student sees
}
```

**Events isolated per student:**

```typescript
notifySessionHandlingEvent(examSessionId, studentId, type: ProctoringHandleType) {
  // Only Student S001 sees: WARNING, REPRIMAND, SUSPENSION
  this.emitToStudent(examSessionId, studentId, {
    examSessionId,
    type,  // WARNING | REPRIMAND | SUSPENSION
    data: null,
    timestamp: new Date()
  });
}

notifyRetakePermissionGranted(examSessionId, payload) {
  // Only Student S001 sees: Retake granted
  this.emitToStudent(examSessionId, payload.studentId, {
    examSessionId,
    type: 'RETAKE_GRANTED',
    data: { examAttemptId: '...', examId: '...', status: 'NEW' },
    timestamp: new Date()
  });
}
```

**Why 2 channels?**

| Aspect | Teacher Channel | Student Channel |
|---|---|---|
| **Scope** | Session-wide | Per-student |
| **Events** | STUDENT_JOINED, VIOLATION, SUBMITTED | WARNING, SUSPENSION, RETAKE_GRANTED |
| **Visibility** | All teachers see all events | Student only sees own events |
| **Use case** | Monitoring dashboard | Student notifications |

**Scenario:**

```
Session: CS101 Morning
├─ Teacher 1 (Proctor)
│  └─ Listen to: session:sess-001
│     Receives:
│     - STUDENT_JOINED (S001)
│     - STUDENT_JOINED (S002)
│     - STUDENT_VIOLATION (S001)
│     - STUDENT_SUBMITTED (S002)
│
├─ Student S001
│  └─ Listen to: sess-001:s001
│     Receives:
│     - WARNING (from proctoring)
│     (does NOT receive other students' events)
│
└─ Student S002
   └─ Listen to: sess-001:s002
      Receives:
      - RETAKE_GRANTED (permission granted)
      (does NOT receive other students' events)
```

---

### Q7.2.2: `subscriberCount` dùng để làm gì?

**Trả lời:**

**Memory management & cleanup:**

```typescript
interface SessionState {
  subject: Subject<ExamSessionEvent>;
  subscriberCount: number;  // Track active subscribers
}

private buildSubscription(
  map: Map<string, SessionState>,
  key: string,
  logLabel: string
): Observable<ExamSessionEvent> {
  const state = this.getOrCreateFromMap(map, key);
  state.subscriberCount++;  // ← Increment

  console.log(`[SSE] Connected | ${logLabel} | subscribers: ${state.subscriberCount}`);

  return state.subject.asObservable().pipe(
    finalize(() => {
      state.subscriberCount--;  // ← Decrement on unsubscribe
      console.log(`[SSE] Disconnected | ${logLabel} | remaining: ${state.subscriberCount}`);

      // Cleanup when no subscribers
      if (state.subscriberCount === 0 && state.subject.closed) {
        map.delete(key);  // Remove from memory
        console.log(`[SSE] Removed from memory: ${logLabel}`);
      }
    })
  );
}
```

**Lifecycle:**

```
1. First teacher subscribes to session-001
   ├─ sessions.set('sess-001', { subject, subscriberCount: 1 })
   └─ Log: "Connected | session:sess-001 | subscribers: 1"

2. Second teacher subscribes to same session
   ├─ sessions.get('sess-001').subscriberCount = 2
   └─ Log: "Connected | session:sess-001 | subscribers: 2"

3. First teacher closes connection
   ├─ sessions.get('sess-001').subscriberCount = 1
   └─ Log: "Disconnected | session:sess-001 | remaining: 1"

4. Second teacher closes connection
   ├─ sessions.get('sess-001').subscriberCount = 0
   ├─ sessions.delete('sess-001')  # ← Remove from memory
   └─ Log: "Removed from memory: session:sess-001"
```

**Memory efficiency:**

```typescript
// Bad: Keep all sessions forever
private sessions = new Map<string, SessionState>();
// After 1000 sessions: 1000 * 50KB = 50MB

// Good: Delete empty sessions
if (state.subscriberCount === 0) {
  map.delete(key);  // Free memory
}
// After users leave: only active sessions remain
```

**Monitoring:**

```typescript
getStats(): {
  activeSessions: number;
  activeStudentChannels: number;
  totalSubscribers: number;
} {
  let totalSubscribers = 0;
  this.sessions.forEach(state => totalSubscribers += state.subscriberCount);
  this.studentChannels.forEach(state => totalSubscribers += state.subscriberCount);

  return {
    activeSessions: this.sessions.size,
    activeStudentChannels: this.studentChannels.size,
    totalSubscribers  // Total active connections
  };
}

// Usage
setInterval(() => {
  const stats = this.examSessionSseService.getStats();
  console.log(`Active SSE connections: ${stats.totalSubscribers}`);
  // If too many → alert admin
  if (stats.totalSubscribers > 1000) {
    this.logger.warn('High SSE subscriber count');
  }
}, 60000);
```

---

### Q7.3.1: `emitEvent()` gửi event đến teacher channel. Nó broadcast hay unicast?

**Trả lời:**

**BROADCAST - tất cả teachers trong session đó nhận:**

```typescript
emitEvent(event: ExamSessionEvent): void {
  const state = this.sessions.get(event.examSessionId);
  if (!state) return;

  state.subject.next({...event, timestamp: new Date()});  // Broadcast
}
```

**Ví dụ broadcast:**

```
Session: sess-001
├─ Teacher 1 (connected)
├─ Teacher 2 (connected)
└─ Teacher 3 (connected)

Event: STUDENT_VIOLATION (Student S001 detected no face)

Broadcasting:
├─ Teacher 1 ← receives event
├─ Teacher 2 ← receives event
└─ Teacher 3 ← receives event

All teachers see the violation!
```

**vs Unicast (emitToStudent):**

```typescript
emitToStudent(examSessionId: string, studentId: string, event: ExamSessionEvent): void {
  const key = `${examSessionId}:${studentId}`;
  const state = this.studentChannels.get(key);
  if (!state) return;

  state.subject.next({...event, timestamp: new Date()});  // Unicast to specific student
}

// Only student S001 receives WARNING event
// Student S002, S003 don't receive
```

**Real scenario:**

```
[Exam Session] CS101-Morning
├─ [Teacher Dashboard - Proctor 1]
│  ├─ SSE: subscribeToSession('sess-001')
│  └─ Receives all events
│
├─ [Teacher Dashboard - Proctor 2]
│  ├─ SSE: subscribeToSession('sess-001')
│  └─ Receives all events
│
├─ [Student Exam - S001]
│  ├─ SSE: subscribeStudent('sess-001', 's001')
│  └─ Only receives events for S001
│
└─ [Student Exam - S002]
   ├─ SSE: subscribeStudent('sess-001', 's002')
   └─ Only receives events for S002


Event: STUDENT_VIOLATION (S001)
├─ emitEvent() → Proctor 1 sees
├─ emitEvent() → Proctor 2 sees
├─ emitToStudent('s001') → S001 sees WARNING
└─ emitToStudent('s002') → S002 does NOT see
```

---

### Q7.4.1: STUDENT_JOINED event khi nào được emit? Chi tiết?

**Trả lời:**

**Emitted when: Student joins exam session**

```typescript
// exam-attempt.service.ts
async joinExamSession(sessionId: string, studentId: string) {
  // 1. Validate session
  const session = await this.examSessionService.findById(sessionId);
  if (!session || session.status !== 'IN_PROGRESS') {
    throw new BadRequestException('Session not available');
  }

  // 2. Face verification (if required)
  if (session.isCameraRequired) {
    const faceFile = ... // from request
    await this.faceVerificationService.verifyFace(faceFile, studentEmbedding);
  }

  // 3. Create ExamAttempt
  const examAttempt = await this.examAttemptRepo.create({
    studentId,
    examSessionId: sessionId,
    examId: this.randomExamFromSet(session.examSetId),
    status: 'IN_PROGRESS',
    startTime: new Date(),
    consumedTime: 0,
    lastResumedAt: new Date()
  });

  // 4. Emit STUDENT_JOINED event ← HERE
  const student = await this.studentService.getById(studentId);
  this.examSessionSseService.notifyStudentJoined(sessionId, {
    studentId,
    studentName: student.name || student.email
  });

  // 5. Return exam data + cached submission
  const examData = await this.examService.getExamContent(examAttempt.examAttemptId, examAttempt.examId);
  const cachedSubmission = await this.examDraftService.getDraftAnswers(sessionId, studentId);

  return {
    examAttemptId: examAttempt.examAttemptId,
    exam: examData,
    startTime: examAttempt.startTime,
    duration: session.duration,
    isCameraRequired: session.isCameraRequired,
    cachedSubmission  // Restore previous answers if disconnect
  };
}
```

**Event data structure:**

```typescript
// exam-session-sse.service.ts
notifyStudentJoined(examSessionId: string, studentData: {
  studentId: string;
  studentName: string;
}): void {
  this.emitEvent({
    examSessionId,
    type: 'STUDENT_JOINED',  # Event type
    data: studentData,       # { studentId, studentName }
    timestamp: new Date()
  });
}
```

**Event sent:**

```json
{
  "type": "STUDENT_JOINED",
  "data": {
    "studentId": "s001",
    "studentName": "Nguyen Tran A"
  },
  "timestamp": "2024-01-15T08:05:30Z"
}
```

**Teacher sees:**

```
[Exam Monitoring Dashboard]

⏰ Session: CS101-Morning (08:00 - 10:00)
📊 Students joined: 25

Timeline:
├─ 08:00:05 - Student "Nguyen A" joined
├─ 08:00:12 - Student "Tran B" joined
├─ 08:00:18 - Student "Le C" joined
└─ ...
```

---

## 8️⃣ **FRAUD DETECTION & PROCTORING**

### Q8.1.1: Liệt kê 7 loại gian lận được phát hiện. Mỗi cái được phát hiện như thế nào?

**Trả lời:**

**7 Fraud Types từ schema:**

```prisma
enum FraudType {
  FACE_MISMATCH              # Khuôn mặt không khớp
  MULTIPLE_FACES_DETECTED    # Phát hiện 2+ khuôn mặt
  NO_FACE_DETECTED           # Không thấy khuôn mặt
  TAB_SWITCHING              # Chuyển sang tab khác
  WINDOW_BLUR                # Rời cửa sổ browser
  IP_CHANGED                 # Thay đổi IP
  NETWORK_DISRUPTION        # Mất kết nối
}
```

**Chi tiết phát hiện từng loại:**

#### **1. FACE_MISMATCH - Khuôn mặt không khớp**

```typescript
// Detection: Compare current face with registered embedding
async verifyFaceMatch(studentId: string, currentPhoto: Buffer) {
  const student = await this.studentService.getById(studentId);
  const storedEmbedding = student.faceEmbedding;  // Registered

  const result = await this.faceVerificationService.verifyFace(
    currentPhoto,
    storedEmbedding
  );

  if (!result.match || result.accuracy < 0.85) {
    // FRAUD DETECTED
    this.fraudDetectionService.recordViolation({
      examAttemptId,
      fraudType: FraudType.FACE_MISMATCH,
      details: { accuracy: result.accuracy, threshold: 0.85 }
    });
  }
}
```

**When**: 
- Entry: Face verification at session join
- Periodic: Every 5-10 seconds during exam
- Via: FastAPI face recognition service

---

#### **2. MULTIPLE_FACES_DETECTED - 2+ khuôn mặt**

```typescript
// Detection: MediaPipe detects face count
async detectFaceCount(frameBuffer: Buffer) {
  const detection = await this.mediaπipe.detectFaces(frameBuffer);
  
  if (detection.faces.length > 1) {
    // FRAUD DETECTED - Multiple people
    this.fraudDetectionService.recordViolation({
      examAttemptId,
      fraudType: FraudType.MULTIPLE_FACES_DETECTED,
      details: { 
        faceCount: detection.faces.length,
        confidence: detection.confidence
      }
    });
  }
}
```

**When**: Continuous during exam via WebRTC camera stream
**Via**: MediaPipe FaceMesh (client-side or server)

---

#### **3. NO_FACE_DETECTED - Không thấy khuôn mặt**

```typescript
// Detection: Timeout without face detection
async monitorFacePresence(examAttemptId: string) {
  const stream = webcam.getStream();
  let lastFaceTime = Date.now();

  stream.on('frame', async (frame) => {
    const faces = await this.detectFaces(frame);
    
    if (faces.length > 0) {
      lastFaceTime = Date.now();
    }
  });

  // Check every 1 second
  setInterval(() => {
    if (Date.now() - lastFaceTime > 1000) {
      // No face for 1+ second
      this.fraudDetectionService.recordViolation({
        examAttemptId,
        fraudType: FraudType.NO_FACE_DETECTED,
        details: { missingDurationMs: Date.now() - lastFaceTime }
      });
    }
  }, 1000);
}
```

**When**: 1 second threshold during exam
**Cause**: Student looking away, head turned >30°, camera blocked

---

#### **4. TAB_SWITCHING - Chuyển sang tab khác**

```typescript
// Detection: visibilitychange event
window.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    // TAB SWITCHED
    fetch(`/api/exam-attempt/${attemptId}/violation`, {
      method: 'POST',
      body: JSON.stringify({
        fraudType: FraudType.TAB_SWITCHING,
        occurredAt: new Date()
      })
    });

    // Notify server
    this.examSessionSseService.notifyViolation(...);
  }
});
```

**When**: Immediately when switching tabs
**Detection**: Browser visibilitychange event

---

#### **5. WINDOW_BLUR - Rời cửa sổ browser**

```typescript
// Detection: blur/focus events
window.addEventListener('blur', () => {
  // Focus left browser window
  fetch(`/api/exam-attempt/${attemptId}/violation`, {
    method: 'POST',
    body: JSON.stringify({
      fraudType: FraudType.WINDOW_BLUR,
      occurredAt: new Date()
    })
  });
});

window.addEventListener('focus', () => {
  // Focus returned
});
```

**When**: Immediately when focus lost
**Detection**: Window blur event

---

#### **6. IP_CHANGED - Thay đổi IP**

```typescript
// Detection: Compare IPs at heartbeat
async heartbeat(attemptId: string, currentIp: string) {
  const attempt = await this.examAttemptRepo.getWithIpHistory(attemptId);
  
  if (attempt.startIp && attempt.startIp !== currentIp) {
    // IP changed
    this.fraudDetectionService.recordViolation({
      examAttemptId: attemptId,
      fraudType: FraudType.IP_CHANGED,
      details: { 
        previousIp: attempt.startIp,
        currentIp,
        timestamp: new Date()
      }
    });
  }
}
```

**When**: First heartbeat shows different IP
**Detection**: Compare with initial IP from join

---

#### **7. NETWORK_DISRUPTION - Mất kết nối**

```typescript
// Detection: Heartbeat timeout
async handleHeartbeat(attemptId: string, clientIp: string) {
  const attempt = await this.examAttemptRepo.findById(attemptId);
  
  if (!attempt) {
    // Student never registered heartbeat
    throw new NotFoundException();
  }

  // Update lastHeartbeat
  await this.examAttemptRepo.update(attemptId, {
    lastHeartbeat: new Date()
  });
}

// Monitor disconnection (server-side)
setInterval(async () => {
  const allAttempts = await this.examAttemptRepo.findByStatus('IN_PROGRESS');
  
  allAttempts.forEach(attempt => {
    const timeSinceLastHeartbeat = Date.now() - attempt.lastHeartbeat.getTime();
    
    if (timeSinceLastHeartbeat > 15000) {  // 15 seconds
      // DISCONNECTED
      this.fraudDetectionService.recordViolation({
        examAttemptId: attempt.examAttemptId,
        fraudType: FraudType.NETWORK_DISRUPTION,
        details: { 
          missingHeartbeats: Math.floor(timeSinceLastHeartbeat / 5000)
        }
      });

      // Auto-submit after 30 seconds
      if (timeSinceLastHeartbeat > 30000) {
        await this.forceSubmitAttempt(attempt.examAttemptId);
      }
    }
  });
}, 5000);  // Check every 5 seconds
```

**When**: After 15 seconds without heartbeat
**Auto-action**: Force submit after 30 seconds

---

### Q8.1.2: Bao lâu detect được 1 violation? Là real-time hay có delay?

**Trả lời:**

**Detection latency per fraud type:**

| Fraud Type | Latency | Method | Real-time? |
|---|---|---|---|
| TAB_SWITCHING | < 100ms | visibilitychange event | ✅ Yes |
| WINDOW_BLUR | < 100ms | blur/focus event | ✅ Yes |
| NO_FACE_DETECTED | 1-2s | Face detection (every 1s) | ⚠️ Near |
| MULTIPLE_FACES | 1-2s | Face detection (every 1s) | ⚠️ Near |
| FACE_MISMATCH | 5-10s | Periodic verify (every 5s) | ⚠️ Delayed |
| IP_CHANGED | 5-30s | Heartbeat check | ⚠️ Delayed |
| NETWORK_DISRUPTION | 15-30s | Heartbeat timeout (15s threshold) | ❌ No |

**Real-time (< 1s):**

```typescript
// TAB_SWITCHING
window.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    // Detected within 100ms
  }
});
```

**Near real-time (1-2s):**

```typescript
// FACE detection
setInterval(() => {
  const faces = detectFaces(currentFrame);
  if (faces.length > 1) {
    // Detected within 1-2 seconds
  }
}, 1000);  // Check every 1 second
```

**Delayed (5-30s):**

```typescript
// Heartbeat-based
setInterval(() => {
  sendHeartbeat();  // Every 5 seconds
  checkIpChange();  // Every 5 seconds
  checkDisconnection();  // Every 5 seconds with 15s threshold
}, 5000);
```

**Trade-off:**

```
Faster detection → More CPU/battery usage
Slower detection → Less resources but more false negatives

Current iTEST strategy:
- Client-side events (tab switch, blur) → Instant
- Camera processing (face) → 1-2s
- Server-side checks (heartbeat, IP) → 5-30s
```

---

### Q8.2.1: Fraud level có 3 mức: LOW, MEDIUM, HIGH. Mỗi violation thuộc mức nào?

**Trả lời:**

**Fraud level mapping:**

```prisma
enum FraudLevel {
  LOW      # 1 violation
  MEDIUM   # 2 violations
  HIGH     # 3+ violations
}
```

**Logic từ source:**

```typescript
private determineFraudLevel(warningCount: number): FraudLevel | null {
  if (warningCount === 0) return null;
  if (warningCount <= 1) return FraudLevel.LOW;      // 1 violation
  if (warningCount <= 2) return FraudLevel.MEDIUM;   // 2 violations
  return FraudLevel.HIGH;                             // 3+ violations
}
```

**Actions per level:**

```typescript
async handleViolation(examAttemptId: string, violationType: FraudType) {
  // ... log violation ...
  const updatedAttempt = await this.examAttemptRepo.update(examAttemptId, {
    warningCount: { increment: 1 }
  });

  const fraudLevel = this.determineFraudLevel(updatedAttempt.warningCount);

  if (updatedAttempt.warningCount === 1) {
    // ⚠️ LOW - WARNING
    this.examSessionSseService.notifySessionHandlingEvent(
      attempt.examSessionId,
      attempt.studentId,
      'WARNING'
    );
    // Display warning popup on student screen
    
  } else if (updatedAttempt.warningCount === 2) {
    // 🔴 MEDIUM - REPRIMAND
    this.examSessionSseService.notifySessionHandlingEvent(
      attempt.examSessionId,
      attempt.studentId,
      'REPRIMAND'
    );
    // Stronger warning to student
    
  } else if (updatedAttempt.warningCount >= 3) {
    // 🛑 HIGH - SUSPENSION
    await this.forceSubmitExamAttempt(examAttemptId);
    this.examSessionSseService.notifySessionHandlingEvent(
      attempt.examSessionId,
      attempt.studentId,
      'SUSPENSION'
    );
    // Auto-submit attempt
  }

  // Update fraudLevel in database
  await this.examAttemptRepo.update(examAttemptId, { fraudLevel });
}
```

**Visualization:**

```
warningCount: 0
├─ fraudLevel: null
└─ Action: None

warningCount: 1
├─ fraudLevel: LOW
└─ Action: WARNING (popup on screen, visible in monitoring)

warningCount: 2
├─ fraudLevel: MEDIUM
└─ Action: REPRIMAND (stronger warning)

warningCount: 3+
├─ fraudLevel: HIGH
└─ Action: SUSPENSION (auto-submit, exam ends)
```

---

### Q8.3.1: Giám thị có thể thực hiện 4 actions: WARNING, REPRIMAND, SUSPENSION, STOP_FOR_SESSION_TRANSFER

**Trả lời:**

**Proctoring actions:**

```typescript
enum ProctoringHandleType {
  WARNING              # Cảnh cáo nhẹ
  REPRIMAND           # Khiển trách
  SUSPENSION          # Đình chỉ (auto-submit)
  STOP_FOR_SESSION_TRANSFER  # Dừng để chuyển ca
}
```

**Each action effect:**

#### **1. WARNING**

```typescript
async sendWarningToStudent(examSessionId: string, studentId: string) {
  // Emit event to student channel
  this.examSessionSseService.notifySessionHandlingEvent(
    examSessionId,
    studentId,
    ProctoringHandleType.WARNING
  );

  // Student receives popup
  // Frontend shows: "⚠️ You've been warned. Follow exam rules."
  // Student can continue exam
}
```

**Effect:**
- ✅ Student continues exam
- 📢 Popup warning
- 📊 Logged in records

#### **2. REPRIMAND**

```typescript
async sendReprimandToStudent(examSessionId: string, studentId: string) {
  this.examSessionSseService.notifySessionHandlingEvent(
    examSessionId,
    studentId,
    ProctoringHandleType.REPRIMAND
  );

  // Student receives stronger warning
  // Frontend shows: "🔴 Serious warning. Continued violation will result in suspension."
  // Student can continue exam
}
```

**Effect:**
- ✅ Student continues exam
- 📢 Stronger warning
- 📊 Logged

#### **3. SUSPENSION**

```typescript
async suspendStudent(examSessionId: string, studentId: string) {
  // 1. Auto-submit exam attempt
  const attempt = await this.examAttemptRepo.findByStudentAndSession(
    studentId,
    examSessionId
  );
  
  await this.forceSubmitExamAttempt(attempt.examAttemptId);
  
  // 2. Emit SUSPENSION event
  this.examSessionSseService.notifySessionHandlingEvent(
    examSessionId,
    studentId,
    ProctoringHandleType.SUSPENSION
  );

  // Student screen locks: "Your exam has been submitted due to violation"
}
```

**Effect:**
- ❌ Student cannot continue
- 🔒 Exam auto-submitted
- ⏹️ All answers saved
- 📊 Flagged in result

#### **4. STOP_FOR_SESSION_TRANSFER**

```typescript
async stopForSessionTransfer(examSessionId: string, studentId: string) {
  // 1. Pause student's attempt (don't submit yet)
  const attempt = await this.examAttemptRepo.findByStudentAndSession(
    studentId,
    examSessionId
  );
  
  await this.examAttemptRepo.update(attempt.examAttemptId, {
    status: 'PAUSED'
  });
  
  // 2. Emit STOP event
  this.examSessionSseService.notifySessionHandlingEvent(
    examSessionId,
    studentId,
    'STOP_FOR_SESSION_TRANSFER'
  );

  // Student screen shows: "Stop exam - transfer to different session"
  // Answers are paused (not submitted)
  // Student can resume in different session
}
```

**Effect:**
- ⏸️ Exam paused
- 💾 Answers saved in Redis
- ↔️ Can transfer to different session
- 📊 Not yet submitted

---

### Q8.3.2: Giám thị có thể manual trigger proctoring action hay chỉ từ violation detection?

**Trả lời:**

**BOTH - Manual + Automatic**

#### **Automatic (from violation detection):**

```typescript
// fraud-detection.service.ts
async handleViolation(examAttemptId: string, violationType: FraudType) {
  const attempt = await this.examAttemptRepo.findById(examAttemptId);
  
  // Increment warning
  const updated = await this.examAttemptRepo.update(examAttemptId, {
    warningCount: { increment: 1 }
  });

  // Auto action based on warningCount
  if (updated.warningCount === 1) {
    // Auto-trigger WARNING
    this.examSessionSseService.notifySessionHandlingEvent(
      attempt.examSessionId,
      attempt.studentId,
      ProctoringHandleType.WARNING
    );
  } else if (updated.warningCount >= 3) {
    // Auto-trigger SUSPENSION
    await this.forceSubmitExamAttempt(examAttemptId);
    this.examSessionSseService.notifySessionHandlingEvent(
      attempt.examSessionId,
      attempt.studentId,
      ProctoringHandleType.SUSPENSION
    );
  }
}
```

#### **Manual (from teacher UI):**

```typescript
// exam-session.controller.ts
@Post('exam-session/:id/students/:studentId/action')
async takeAction(
  @Param('id') sessionId: string,
  @Param('studentId') studentId: string,
  @Body() body: { action: ProctoringHandleType }
) {
  const { action } = body;

  // Validate teacher permission
  const teacher = await this.getCurrentTeacher(request);
  const isTeacherOfSession = await this.examSessionService.isTeacherOf(teacher.id, sessionId);
  if (!isTeacherOfSession) {
    throw new UnauthorizedException();
  }

  // Apply action
  switch (action) {
    case ProctoringHandleType.WARNING:
      await this.proctoringService.sendWarning(sessionId, studentId);
      break;
    case ProctoringHandleType.REPRIMAND:
      await this.proctoringService.sendReprimand(sessionId, studentId);
      break;
    case ProctoringHandleType.SUSPENSION:
      await this.proctoringService.suspendStudent(sessionId, studentId);
      break;
    case 'STOP_FOR_SESSION_TRANSFER':
      await this.proctoringService.stopForTransfer(sessionId, studentId);
      break;
  }

  return { success: true, action };
}
```

**Teacher UI (monitoring dashboard):**

```html
<!-- Monitoring Dashboard -->
<div class="student-card">
  <h3>Student: Nguyen A (s001)</h3>
  <p>Status: IN_PROGRESS</p>
  <p>Warnings: 1</p>
  
  <!-- Action buttons (teacher can click anytime) -->
  <button @click="sendWarning('s001')">
    ⚠️ Warning
  </button>
  <button @click="sendReprimand('s001')">
    🔴 Reprimand
  </button>
  <button @click="suspend('s001')">
    🛑 Suspend
  </button>
  <button @click="stopForTransfer('s001')">
    ↔️ Stop for Transfer
  </button>
</div>
```

**Summary:**

| Source | Trigger | Auto? | Manual? |
|---|---|---|---|
| Violation → 1st | TAB_SWITCHING | ✅ Auto WARNING | - |
| Violation → 2nd | MULTIPLE_FACES | ✅ Auto REPRIMAND | - |
| Violation → 3rd+ | NO_FACE | ✅ Auto SUSPENSION | - |
| Teacher action | - | - | ✅ Manual (any action) |

---

## 9️⃣ **FACE VERIFICATION**

### Q9.1.1: User phải upload khuôn mặt khi đăng ký không? Hay khi join exam session?

**Trả lời:**

**TWO-STAGE face enrollment:**

#### **Stage 1: Registration time (optional)**

```typescript
// auth/registration.controller.ts
@Post('register')
async register(@Body() dto: RegisterDto, @UploadedFile() facePhoto: Express.Multer.File) {
  // 1. Create account
  const account = await this.authService.createAccount({
    email: dto.email,
    password: dto.password,
    role: dto.role
  });

  // 2. Create student/teacher profile
  let profile;
  if (dto.role === 'STUDENT') {
    profile = await this.studentService.create({
      accountId: account.accountId,
      studentCode: dto.studentCode,
      name: dto.name
    });
  }

  // 3. Upload face photo (OPTIONAL)
  if (facePhoto) {
    const embedding = await this.faceRecognitionService.extractEmbedding(facePhoto);
    await this.studentService.updateFaceEmbedding(profile.id, embedding);
    // Store: profile.faceEmbedding = float[]
  }

  return { success: true, accountId: account.accountId };
}
```

#### **Stage 2: Exam session join (if required)**

```typescript
// exam-session.controller.ts
@Post('exam-session/:id/join')
async joinExam(
  @Param('id') sessionId: string,
  @Body() dto: JoinExamDto,
  @UploadedFile() facePhoto?: Express.Multer.File
) {
  const session = await this.examSessionService.findById(sessionId);
  const student = await this.studentService.getById(dto.studentId);

  // 1. Check if camera required
  if (session.isCameraRequired) {
    if (!facePhoto) {
      throw new BadRequestException('Face photo required for this session');
    }

    // 2. Verify face
    if (!student.faceEmbedding) {
      // Student never uploaded face at registration
      // Option A: Extract embedding from current photo (store as registration)
      const embedding = await this.faceRecognitionService.extractEmbedding(facePhoto);
      await this.studentService.updateFaceEmbedding(student.id, embedding);
      
      // Option B: Reject - must register face first
      throw new BadRequestException('Please register your face first');
    }

    // 3. Verify current face matches stored
    const result = await this.faceVerificationService.verifyFace(
      facePhoto,
      student.faceEmbedding
    );

    if (!result.match) {
      throw new BadRequestException('Face verification failed');
    }
  }

  // 4. Create attempt + return exam data
  const attempt = await this.examAttemptService.create({
    studentId: dto.studentId,
    examSessionId: sessionId
  });

  return { examAttemptId: attempt.examAttemptId, exam: {...} };
}
```

**Workflow:**

```
┌─ Registration Time ────────────────────┐
│ 1. User creates account                │
│ 2. Optional: Upload face photo         │
│    - Extract embedding                 │
│    - Store in database                 │
│ 3. Complete registration               │
└────────────────────────────────────────┘
           ↓ (Later)
┌─ Exam Session Join ────────────────────┐
│ 1. Check session.isCameraRequired      │
│ 2. If YES:                             │
│    - Request face photo from camera    │
│    - Verify against registered photo   │
│    - If not match → reject             │
│    - If match → allow join             │
│ 3. Create ExamAttempt + return exam    │
└────────────────────────────────────────┘
```

**Best practice:**

```
✅ RECOMMENDED:
1. Registration: Encourage (not force) face upload
2. Session join: Mandatory if session.isCameraRequired
   - If student never uploaded → ask to upload at join time
   - If uploaded → verify current face

❌ NOT RECOMMENDED:
1. Force face upload at registration (privacy)
2. Use different faces each time (fail verification)
```

---

### Q9.2.1: Khi student join exam session, có bắt phải xác thực khuôn mặt không?

**Trả lời:**

**YES, if session.isCameraRequired = true**

```prisma
model ExamSession {
  isCameraRequired  Boolean  @default(false)
}
```

**Logic:**

```typescript
async joinExamSession(sessionId: string, studentId: string, facePhoto?: Buffer) {
  const session = await this.examSessionService.findById(sessionId);

  // Check if camera required
  if (session.isCameraRequired) {
    if (!facePhoto) {
      throw new BadRequestException('Face verification required for this session');
    }

    // Verify face
    const student = await this.studentService.getById(studentId);
    if (!student.faceEmbedding) {
      throw new BadRequestException('Face not registered. Please register first.');
    }

    const result = await this.faceVerificationService.verifyFace(
      facePhoto,
      student.faceEmbedding
    );

    if (!result.match || result.accuracy < 0.85) {
      // Auto-log violation
      await this.fraudDetectionService.recordViolation({
        studentId,
        fraudType: FraudType.FACE_MISMATCH,
        details: { accuracy: result.accuracy }
      });
      
      throw new BadRequestException('Face verification failed. Retry or contact admin.');
    }
  }

  // If passed or not required → proceed
  return this.createExamAttempt(studentId, sessionId);
}
```

**Two scenarios:**

#### **Scenario 1: isCameraRequired = false**

```json
{
  "sessionId": "sess-001",
  "isCameraRequired": false
}
```

```
User join exam:
├─ No face photo required
├─ Can join immediately
└─ SSE monitoring started (but no face verification)
```

#### **Scenario 2: isCameraRequired = true**

```json
{
  "sessionId": "sess-002",
  "isCameraRequired": true
}
```

```
User join exam:
├─ Must provide face photo
├─ Server verifies: currentFace == registeredFace
├─ If match → allow join
└─ If not match → reject + log FACE_MISMATCH violation
```

---

### Q9.3.1: Trong quá trình làm bài, có tự động capture ảnh khuôn mặt không?

**Trả lời:**

**YES - Periodic face verification during exam**

```typescript
// exam-attempt.service.ts (client-side via WebRTC)

class ExamAttemptComponent implements OnInit {
  private webcam: Webcam;
  private faceCheckInterval: NodeJS.Timeout;

  ngOnInit() {
    // 1. Start webcam
    this.webcam.start();

    // 2. Periodic face check (every 5-10 seconds)
    this.faceCheckInterval = setInterval(async () => {
      const frame = this.webcam.captureFrame();
      
      // 3. Send to backend async queue
      await this.examService.submitFaceForVerification({
        examAttemptId: this.attemptId,
        imageBuffer: frame.toBuffer(),
        occurredAt: new Date()
      });
    }, 5000);  // Every 5 seconds
  }

  ngOnDestroy() {
    this.webcam.stop();
    clearInterval(this.faceCheckInterval);
  }
}
```

**Backend async processing:**

```typescript
// face-verification.producer.ts
@Injectable()
export class FaceVerificationProducer {
  constructor(@InjectQueue(QueueName.FACE_VERIFICATION_QUEUE) 
    private readonly queue: Queue) {}

  async submitFaceForVerification(data: IFaceVerificationJobData) {
    // Add to Bull queue (async processing)
    await this.queue.add(
      QueueName.FACE_VERIFICATION_QUEUE,
      {
        examAttemptId: data.examAttemptId,
        accountId: data.accountId,
        occurmedAt: data.occurredAt,
        imageBuffer: data.imageBuffer
      },
      {
        attempts: 2,  // Retry 2 times
        backoff: { type: 'fixed', delay: 3000 },
        removeOnComplete: true,
        removeOnFail: 50
      }
    );
  }
}

// face-verification.consumer.ts
@Process(QueueName.FACE_VERIFICATION_QUEUE)
async handleFaceVerification(job: Job<IFaceVerificationJobData>) {
  const { examAttemptId, imageBuffer, occurredAt } = job.data;

  try {
    // 1. Get student's registered face embedding
    const attempt = await this.examAttemptRepo.findById(examAttemptId);
    const student = await this.studentService.getById(attempt.studentId);

    // 2. Verify face
    const result = await this.faceVerificationService.verifyFace(
      imageBuffer,
      student.faceEmbedding
    );

    // 3. If fail → record violation
    if (!result.match) {
      await this.fraudDetectionService.recordViolation({
        examAttemptId,
        fraudType: FraudType.FACE_MISMATCH,
        occurredAt,
        details: { accuracy: result.accuracy }
      });

      // 4. Notify teacher via SSE
      this.examSessionSseService.notifyViolation(
        attempt.examSessionId,
        {
          studentId: attempt.studentId,
          violationType: FraudType.FACE_MISMATCH,
          details: { accuracy: result.accuracy }
        }
      );
    }
  } catch (error) {
    console.error('Face verification job failed', error);
    throw error;  // Retry (attempts: 2)
  }
}
```

**Timeline:**

```
Exam started (08:00)
  ├─ 08:00:05 - Capture face 1 → Queue → Verify ✓
  ├─ 08:00:10 - Capture face 2 → Queue → Verify ✓
  ├─ 08:00:15 - Capture face 3 → Queue → Verify ✗ (FACE_MISMATCH)
  │   └─ Log violation → Notify teacher
  ├─ 08:00:20 - Capture face 4 → Queue → Verify ✓
  └─ 08:10:00 - Exam submitted
```

**Interval:** Every 5-10 seconds during exam

---

## 🔟 **SCORING & RESULTS**

### Q10.1.1: Multiple choice được auto-score bằng cách so sánh student answer với `correctAnswer`

**Trả lời:**

**Auto-scoring logic:**

```typescript
// exam-submission.service.ts

async scoreExamAttempt(examAttemptId: string) {
  // 1. Get attempt + answers
  const attempt = await this.examAttemptRepo.findById(examAttemptId);
  const studentAnswers = await this.studentAnswerRepo.findByAttempt(examAttemptId);
  const exam = await this.examService.getDetailById(attempt.examId);

  // 2. Score each question
  let totalScore = 0;
  const scoreDetail: any[] = [];

  for (const question of exam.parts.flatMap(p => p.questions)) {
    const studentAnswer = studentAnswers.find(a => a.questionId === question.id);
    const correctAnswer = question.questionAnswer;

    // 3. Skip essay questions (manual grading)
    if (question.questionType === QuestionType.ESSAY) {
      scoreDetail.push({
        questionId: question.id,
        studentAnswer: studentAnswer?.answer || null,
        correctAnswer: null,
        score: 0,
        isCorrect: null,
        type: 'ESSAY'
      });
      continue;
    }

    // 4. Auto-score (all other types)
    const isCorrect = this.compareAnswers(
      studentAnswer?.answer,
      correctAnswer?.correctAnswer,
      question.questionType
    );

    const questionScore = isCorrect ? correctAnswer.points : 0;
    totalScore += questionScore;

    scoreDetail.push({
      questionId: question.id,
      questionNumber: question.questionNumber,
      studentAnswer: studentAnswer?.answer || null,
      correctAnswer: correctAnswer?.correctAnswer,
      score: questionScore,
      isCorrect,
      type: question.questionType,
      points: correctAnswer.points
    });
  }

  // 5. Create Result record
  const result = await this.resultRepo.create({
    examAttemptId,
    score: totalScore,
    scoreDetail,
    status: ResultStatus.NOT_GRADED  // Wait for essay grading
  });

  return result;
}
```

**Answer comparison logic:**

```typescript
private compareAnswers(
  studentAnswer: unknown,
  correctAnswer: string[],
  questionType: QuestionType
): boolean {
  // Handle null/undefined
  if (!correctAnswer || correctAnswer.length === 0) return false;
  if (studentAnswer === null || studentAnswer === undefined) return false;

  const sanitizeAnswer = (a: any) => {
    if (typeof a === 'string') return a.trim().toUpperCase();
    return String(a).trim().toUpperCase();
  };

  switch (questionType) {
    case QuestionType.SINGLE_CHOICE:
      // Single choice: "A" == "A"?
      return sanitizeAnswer(studentAnswer) === sanitizeAnswer(correctAnswer[0]);

    case QuestionType.MULTIPLE_CHOICE:
      // Multiple choice: ["A", "C"] == ["A", "C"]?
      const studentAnswers = Array.isArray(studentAnswer)
        ? studentAnswer.map(sanitizeAnswer).sort()
        : [sanitizeAnswer(studentAnswer)];
      
      const correct = correctAnswer.map(sanitizeAnswer).sort();
      
      return (
        studentAnswers.length === correct.length &&
        studentAnswers.every((a, i) => a === correct[i])
      );

    case QuestionType.TRUE_FALSE:
      // TRUE_FALSE: "TRUE" == "TRUE"?
      const studentBool = sanitizeAnswer(studentAnswer) === 'TRUE';
      const correctBool = sanitizeAnswer(correctAnswer[0]) === 'TRUE';
      return studentBool === correctBool;

    case QuestionType.FILL_IN_THE_BLANK:
      // Fuzzy match (case-insensitive, trim whitespace)
      return sanitizeAnswer(studentAnswer) === sanitizeAnswer(correctAnswer[0]);

    case QuestionType.ESSAY:
      // Manual grading (skip)
      return null;

    default:
      return false;
  }
}
```

**Score calculation:**

```json
{
  "scoreDetail": [
    {
      "questionId": "q1",
      "questionType": "SINGLE_CHOICE",
      "studentAnswer": "B",
      "correctAnswer": ["A"],
      "points": 1,
      "score": 0,  // Wrong
      "isCorrect": false
    },
    {
      "questionId": "q2",
      "questionType": "MULTIPLE_CHOICE",
      "studentAnswer": ["A", "C"],
      "correctAnswer": ["A", "C"],
      "points": 2,
      "score": 2,  // Correct
      "isCorrect": true
    },
    {
      "questionId": "q3",
      "questionType": "ESSAY",
      "studentAnswer": "...",
      "correctAnswer": null,
      "points": 5,
      "score": 0,  // Wait for grading
      "isCorrect": null,
      "type": "ESSAY"
    }
  ],
  "totalScore": 2  // Sum of scores (excluding essay)
}
```

---

### Q10.1.2: Nếu question có `points = 10`, student chọn đúng được 10 điểm, sai được 0 điểm à?

**Trả lời:**

**ĐÚng - All-or-nothing scoring (không có partial credit)**

```typescript
const questionScore = isCorrect ? correctAnswer.points : 0;
```

**Example:**

```json
{
  "question": "What is the capital of France?",
  "questionType": "SINGLE_CHOICE",
  "options": [
    { "label": "A", "text": "London" },
    { "label": "B", "text": "Paris" },
    { "label": "C", "text": "Berlin" },
    { "label": "D", "text": "Madrid" }
  ],
  "correctAnswer": ["B"],
  "points": 10
}
```

**Scenarios:**

| Student Answer | Is Correct? | Points | Reasoning |
|---|---|---|---|
| B (Paris) | ✅ Yes | 10 | Match correct answer |
| A (London) | ❌ No | 0 | Wrong choice |
| C (Berlin) | ❌ No | 0 | Wrong choice |
| null (blank) | ❌ No | 0 | No answer |

**For MULTIPLE_CHOICE:**

```json
{
  "question": "Which are capitals? (select all)",
  "questionType": "MULTIPLE_CHOICE",
  "correctAnswer": ["A", "C"],  // Paris, Berlin
  "points": 10
}
```

| Student Answer | Correct? | Points |
|---|---|---|
| ["A", "C"] | ✅ Yes | 10 |
| ["A"] | ❌ No | 0 |
| ["A", "B"] | ❌ No | 0 |
| [] (none) | ❌ No | 0 |

**Current implementation:**
- ✅ All-or-nothing (standard for standardized exams)
- ❌ No partial credit
- ❌ No weighted scoring

**Possible enhancement:**

```typescript
// Partial credit (not implemented)
private calculatePartialCredit(
  studentAnswers: string[],
  correctAnswers: string[],
  totalPoints: number
): number {
  const correctCount = studentAnswers.filter(a => correctAnswers.includes(a)).length;
  const pointsPerAnswer = totalPoints / correctAnswers.length;
  return correctCount * pointsPerAnswer;
}
```

---

### Q10.1.3: Nếu không chọn câu nào (leave blank), được bao nhiêu điểm?

**Trả lời:**

**0 điểm - Same as wrong answer**

```typescript
// No answer = wrong answer
if (studentAnswer === null || studentAnswer === undefined) {
  return false;  // Not correct
}

const questionScore = isCorrect ? correctAnswer.points : 0;  // 0 points
```

**Example:**

```
Question: "What is 2+2?"
Points: 5

Scenario 1: Student answers "4"
├─ Correct? Yes
└─ Score: 5

Scenario 2: Student answers "5"
├─ Correct? No
└─ Score: 0

Scenario 3: Student leaves blank (no answer)
├─ Correct? No (null == false)
└─ Score: 0
```

**In scoreDetail:**

```json
{
  "questionId": "q1",
  "studentAnswer": null,  // Blank
  "correctAnswer": ["4"],
  "points": 5,
  "score": 0,
  "isCorrect": false
}
```

---

## 🔟 **PERFORMANCE & OPTIMIZATION**

### Q11.1.1: Index nào là quan trọng nhất?

**Trả lời:**

**Critical indexes (từ schema.prisma):**

```prisma
// High-traffic queries
@@index([examSessionId])                    # Queries by session
@@index([studentId])                        # Queries by student
@@index([status])                           # Filter by status
@@index([createdAt])                        # Time-range queries
@@unique([studentId, examSessionId])        # Lookup by student+session

// Soft delete
@@index([deletedAt])                        # Filter NOT deleted
```

**Performance impact per index:**

```
Query: Find all attempts in session 001 with 10,000 attempts

WITHOUT index:
  examAttempts.find({ examSessionId: 'sess-001' })
  → Full table scan: O(n) = 10,000 rows scanned ❌ SLOW

WITH index on examSessionId:
  examAttempts.find({ examSessionId: 'sess-001' })
  → B-tree lookup: O(log n) = ~14 rows scanned ✅ FAST
```

**Top 5 critical indexes:**

| Index | Query | Impact |
|---|---|---|
| `[examSessionId]` | Find attempts in session | High - most frequent |
| `[studentId, examSessionId]` | Check unique constraint | High - join/retake |
| `[status]` | Find IN_PROGRESS attempts | High - polling |
| `[examSetId]` | Get exams by set | Medium - admin |
| `[deletedAt]` | Filter soft-deleted | Medium - data integrity |

**Recommendation:**

```prisma
model ExamAttempt {
  @@index([examSessionId])
  @@index([studentId])
  @@index([status])
  @@unique([studentId, examSessionId])  # Already creates index
}

model Exam {
  @@index([examSetId])
  @@index([status])
  @@index([createdBy])
}

model StudentAnswer {
  @@index([examAttemptId])  // Queries by attempt
  @@index([questionId])      // Verify correct answers
}
```

---

### Q11.2.1: Redis được dùng để cache gì?

**Trả lời:**

**Redis use cases in iTEST:**

#### **1. Draft Answers Cache**

```typescript
// Key format: exam:draft:{sessionId}:{studentId}
// Type: Hash
// TTL: Session duration + buffer

const draftKey = getCacheExamDraftKey(sessionId, studentId);

// Save draft
await redisService.hmset(draftKey, {
  'qid:q1': JSON.stringify({ questionId: 'q1', answer: 'A', updatedAtTs: 1234567 }),
  'qid:q2': JSON.stringify({ questionId: 'q2', answer: ['A', 'C'], updatedAtTs: 1234568 })
});

await redisService.expire(draftKey, ttlSeconds);  // Auto-delete after TTL

// Retrieve draft
const draftData = await redisService.hgetall(draftKey);
```

**Why Redis?**
- ✅ Fast read/write (< 1ms)
- ✅ Auto-expire (TTL)
- ✅ Easy to delete (per-student draft)

#### **2. Student Presence/Heartbeat**

```typescript
// Key format: exam:presence:{sessionId}:{studentId}
// Type: String (simple existence check)
// TTL: 15 seconds (heartbeat timeout)

// On heartbeat
await redisService.setex(
  `exam:presence:${sessionId}:${studentId}`,
  15,  // 15 second TTL
  '1'
);

// Check disconnect
const isPresent = await redisService.get(`exam:presence:${sessionId}:${studentId}`);
if (!isPresent) {
  // Student disconnected > 15 seconds
  await handleDisconnect(studentId);
}
```

#### **3. Exam Shuffle Seed** (optional)

```typescript
// Key format: exam:shuffle:{attemptId}
// Type: String (pre-computed shuffle)
// TTL: Exam duration

const shuffleKey = `exam:shuffle:${examAttemptId}`;

// Cache shuffled exam on first access
const shuffled = await redisService.get(shuffleKey);
if (!shuffled) {
  const shuffledExam = ExamShuffleHelper.shuffleExam(exam, examAttemptId);
  await redisService.setex(shuffleKey, sessionDurationSeconds, JSON.stringify(shuffledExam));
}

return JSON.parse(shuffled);
```

**Memory estimate:**

```
Assuming 100 active sessions, 50 students each:

Draft answers:
  100 sessions × 50 students × 20 questions × 100 bytes = 10 MB

Presence:
  100 × 50 × 10 bytes = 50 KB

Shuffle cache:
  100 × 50 × 10 KB = 50 MB

Total: ~60 MB (reasonable for Redis)
```

---

### Q11.3.1: `getExamContent()` trả về toàn bộ exam data (all parts, questions). Có thể pagination không?

**Trả lời:**

**Current: No pagination - returns full exam**

```typescript
// exam.service.ts
async getExamContent(examAttemptId: string, examId: string) {
  const examData = await this.examRepo.findDetailById(examId);
  
  if (!examData) throw new NotFoundException('Exam not found');
  
  // Shuffle & return ALL parts + questions
  const shuffledParts = ExamShuffleHelper.shuffleExam(examData.parts, examAttemptId);

  return {
    ...examData,
    parts: shuffledParts  // All parts, no pagination
  };
}
```

**Challenge:**

```
Exam with 100 questions = ~500 KB JSON
└─ Network bandwidth
└─ Browser rendering (100+ question boxes)
└─ Memory usage
```

**Potential improvement: Client-side pagination**

```typescript
// Option 1: Lazy-load questions
async getExamContent(examId: string, partIndex?: number) {
  const exam = await this.examRepo.findById(examId);
  
  if (partIndex !== undefined) {
    // Return only 1 part + its questions
    return {
      parts: [exam.parts[partIndex]]
    };
  }
  
  // Return parts metadata only (no questions)
  return {
    parts: exam.parts.map(p => ({
      partId: p.partId,
      partTitle: p.partTitle,
      questionCount: p.questions.length
      // NO questions array
    }))
  };
}

// Then lazy-load questions when needed
async getPartQuestions(partId: string, pageSize = 10, page = 0) {
  const part = await this.examRepo.findPartById(partId, {
    skip: page * pageSize,
    take: pageSize
  });
  
  return { questions: part.questions };
}
```

**Option 2: Stream responses**

```typescript
@Sse('exam/:id/stream')
streamExamContent(@Param('id') examId: string): Observable<MessageEvent> {
  const exam = await this.examService.getDetailById(examId);
  
  return from(exam.parts).pipe(
    mergeMap(part => 
      from(part.questions).pipe(
        map(q => ({
          type: 'QUESTION',
          data: q
        }))
      ),
      5  // Max 5 concurrent
    ),
    map(event => ({ data: JSON.stringify(event) }))
  );
}
```

**Best practice:**
- ✅ Return full exam once (simpler client code)
- ❌ Paginate only if exam > 500 questions
- ⚠️ Consider client-side rendering optimization instead

---

## ⏳ **Testing & Error Handling**

### Q12.1.1: Exception nào được throw khi resource not found?

**Trả lời:**

**Standard NestJS exceptions:**

```typescript
import { NotFoundException, BadRequestException, UnauthorizedException } from '@nestjs/common';

// exam.service.ts
async getDetailByIdOrThrow(examId: string) {
  const exam = await this.examRepo.findDetailById(examId);

  if (!exam) {
    throw new NotFoundException(SAFE_MESSAGES.NOT_FOUND);
    // HTTP 404 + "Resource not found" message
  }

  return exam;
}

// exam-session.service.ts
async findById(sessionId: string) {
  const session = await this.examSessionRepo.findById(sessionId);
  
  if (!session) {
    throw new NotFoundException(`Session ${sessionId} not found`);
  }
  
  return session;
}
```

**Exception hierarchy:**

```
HttpException (base)
├─ 400 BadRequestException
│   └─ Validation error, invalid request
├─ 401 UnauthorizedException
│   └─ Auth failure
├─ 403 ForbiddenException
│   └─ Permission denied
├─ 404 NotFoundException
│   └─ Resource not found
└─ 500 InternalServerErrorException
    └─ Server error
```

**Usage per scenario:**

```typescript
// 404 - Resource not found
if (!exam) throw new NotFoundException('Exam not found');

// 400 - Bad request
if (!examDto.title) throw new BadRequestException('Title required');

// 401 - Unauthorized
if (!user) throw new UnauthorizedException('Login required');

// 403 - Forbidden
if (user.role !== 'ADMIN') throw new ForbiddenException('Admin only');

// 409 - Conflict
if (examExists) throw new ConflictException('Exam already exists');
```

**Response to client:**

```json
// 404
{
  "statusCode": 404,
  "message": "Exam not found",
  "error": "Not Found"
}

// 400
{
  "statusCode": 400,
  "message": "Title required",
  "error": "Bad Request"
}
```

---

### Q12.2.1: Input validation ở tầng nào? DTO, Pipe, hay Controller?

**Trả lời:**

**Validation layers (từ high to low level):**

#### **Level 1: DTO + Class Validator**

```typescript
// exam.dto.ts
export class CreateExamDto {
  @IsString()
  @IsNotEmpty()
  title: string;  // ← Validation metadata

  @IsUUID()
  @IsNotEmpty()
  examSetId: string;

  @IsArray()
  @IsString({ each: true })
  examCodes: string[];

  @IsJSON()
  parsedJson: string;  // ← Must be valid JSON

  @IsOptional()
  @IsArray()
  answers?: any[];
}
```

#### **Level 2: Global Validation Pipe**

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Auto-validate all requests against DTOs
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,      // Strip unknown properties
      forbidNonWhitelisted: true,  // Reject unknown properties
      transform: true       // Auto-convert types (string '123' → number 123)
    })
  );

  await app.listen(3000);
}

bootstrap();
```

#### **Level 3: Controller**

```typescript
// exam.controller.ts
@Controller('exams')
export class ExamController {
  @Post()
  async create(@Body() examDto: CreateExamDto) {
    // DTO already validated by ValidationPipe
    // examDto is guaranteed to have correct types & values
    
    return this.examService.create(examDto);
  }
}
```

#### **Level 4: Service**

```typescript
// exam.service.ts
async create(examDto: CreateExamDto, createdBy: string) {
  // Additional business validation (beyond DTO)
  
  if (!examDto.parsedJson || examDto.parsedJson.length === 0) {
    throw new BadRequestException('Exam must have at least 1 question');
  }

  const examSet = await this.examSetService.findById(examDto.examSetId);
  if (!examSet) {
    throw new BadRequestException('ExamSet not found');
  }

  // Proceed with creation
  return this.examRepo.create(examDto);
}
```

**Validation flow:**

```
HTTP Request
  ↓
[Controller receives @Body() → Mapped to DTO]
  ↓
[ValidationPipe validates DTO]
  ├─ Missing required? → 400
  ├─ Wrong type? → 400
  ├─ Invalid UUID? → 400
  └─ OK? → Pass to controller
  ↓
[Controller method executes]
  ↓
[Service receives validated DTO]
  ↓
[Service performs business validation]
  ├─ Resource not found? → 404
  ├─ Invalid state? → 400
  └─ OK? → Execute logic
  ↓
Response to client
```

---

### Q12.3.1: Logger được dùng ở đâu? Error, Warn, hay Info level?

**Trả lời:**

**Logging levels & usage:**

```typescript
import { Logger } from '@nestjs/common';

@Injectable()
export class ExamService {
  private logger = new Logger(ExamService.name);

  async create(examDto: CreateExamDto, createdBy: string) {
    this.logger.log(`Creating exam: ${examDto.title}`);  // INFO
    
    try {
      const examSet = await this.examSetService.findById(examDto.examSetId);
      if (!examSet) {
        this.logger.warn(`ExamSet not found: ${examDto.examSetId}`);  // WARN
        throw new NotFoundException();
      }

      const exam = await this.prisma.exam.create({ data: {...} });
      this.logger.log(`Exam created: ${exam.examId}`);  // INFO

      return exam;
    } catch (error) {
      this.logger.error(`Failed to create exam: ${error.message}`, error.stack);  // ERROR
      throw error;
    }
  }
}
```

**Logging levels:**

| Level | Use Case | Example |
|---|---|---|
| **DEBUG** | Development tracing | Variable values, function entry/exit |
| **INFO** | Notable events | Exam created, exam started, result published |
| **WARN** | Unexpected but recoverable | ExamSet not found, retry attempt |
| **ERROR** | Errors requiring attention | Database connection failed, API timeout |
| **FATAL** | System failures | Out of memory, database crash |

**Best practices:**

```typescript
// ✅ GOOD
this.logger.log('Exam created', exam.examId);
this.logger.warn('Student disconnected after 10s', { studentId });
this.logger.error('Database query failed', error.stack);

// ❌ BAD
this.logger.log('Exam created: ' + JSON.stringify(exam));  // Too verbose
this.logger.error('Error');  // No context
console.log('Debug');  // Don't use console
```

---

**Ende of Part 2**

**Summary:**

✅ Exam Session & Attempt (3 câu)
✅ Real-time Communication SSE (3 câu)
✅ Fraud Detection & Proctoring (4 câu)
✅ Face Verification (3 câu)
✅ Scoring & Results (3 câu)
✅ Performance & Optimization (3 câu)
✅ Testing & Error Handling (3 câu)

**Total: 50+ detailed Q&As in Part 2**

Tài liệu Part 1 + Part 2 = **100+ comprehensive backend interview questions** với full answers & code examples từ iTEST source!

