# 📚 **iTEST Backend Interview Guide**
## Complete Study Materials for Backend Developers

> Hướng dẫn hoàn chỉnh ôn tập phỏng vấn backend iTEST - 150+ câu hỏi & câu trả lời chi tiết

---

## 📖 **Tài Liệu Có Sẵn**

Bộ tài liệu này bao gồm **4 file chính**:

### 1. **itest-backend-qa-part1.md** ⭐ START HERE
   - **50+ câu hỏi & trả lời chi tiết**
   - Chủ đề:
     - ✅ Architecture & Design Patterns (5 Q)
     - ✅ Database & Schema (5 Q)
     - ✅ Authentication & Authorization (4 Q)
     - ✅ Exam Management (3 Q)
     - ✅ AI & Gemini Integration (4 Q)
     - ✅ Exam Session & Attempt (1 Q)
   - **Time: 2-3 hours to study**

### 2. **itest-backend-qa-part2.md** ⭐ CONTINUE HERE
   - **50+ câu hỏi & trả lời chi tiết**
   - Chủ đề:
     - ✅ Exam Session & Attempt (Tiếp) (2 Q)
     - ✅ Real-time Communication SSE (3 Q)
     - ✅ Fraud Detection & Proctoring (4 Q)
     - ✅ Face Verification (3 Q)
     - ✅ Scoring & Results (3 Q)
     - ✅ Performance & Optimization (3 Q)
     - ✅ Testing & Error Handling (3 Q)
   - **Time: 2-3 hours to study**

### 3. **itest-backend-code-examples.md**
   - **Live coding problems & scenarios**
   - Chủ đề:
     - ✅ Code Reading & Analysis (4 problems)
     - ✅ Live Coding Problems (4 problems)
     - ✅ Database Query Problems (2 problems)
     - ✅ API Design Problems (2 problems)
     - ✅ Debugging Scenarios (5 problems)
   - **Time: 1-2 hours practice**

### 4. **itest-backend-interview-questions.md**
   - **150+ structured interview questions**
   - Organized by topic with difficulty levels
   - Summary tables & references
   - **Time: 30 min quick reference**

---

## 🎯 **Study Path (Recommended)**

### **Week 1: Fundamentals**

**Day 1-2: Architecture & Core Concepts**
- File: `itest-backend-qa-part1.md`
- Topics:
  - Architecture & Design Patterns
  - Dependency Injection
  - NestJS framework choice
  - Repository pattern
- Exercises: Explain architecture to friend

**Day 3-4: Database**
- File: `itest-backend-qa-part1.md`
- Topics:
  - Prisma ORM
  - PostgreSQL choice
  - Schema relationships
  - Composite unique keys
- Exercises: Draw ER diagram by hand

**Day 5: Authentication**
- File: `itest-backend-qa-part1.md`
- Topics:
  - JWT payload structure
  - Access vs Refresh tokens
  - Token storage & hashing
  - OAuth integration
- Exercises: Implement JWT from scratch

---

### **Week 2: Core Features**

**Day 1-2: Exam Management**
- File: `itest-backend-qa-part1.md`
- Topics:
  - Exam creation flow
  - Transaction handling
  - Question structure
  - Exam shuffling
- Exercises: Write exam creation service

**Day 3-4: AI Integration**
- File: `itest-backend-qa-part1.md` + `itest-backend-code-examples.md`
- Topics:
  - PDF processing
  - Chunking strategy
  - Gemini API calls
  - Error handling
- Exercises: Implement chunk-based PDF processing

**Day 5: Exam Session**
- File: `itest-backend-qa-part2.md`
- Topics:
  - Session lifecycle
  - Attempt status flow
  - Time tracking
- Exercises: Design exam join flow

---

### **Week 3: Real-time & Monitoring**

**Day 1-2: SSE (Server-Sent Events)**
- File: `itest-backend-qa-part2.md`
- Topics:
  - RxJS Observable/Subject
  - Teacher vs Student channels
  - Memory management
  - Event emission
- Exercises: Implement in-memory channel management

**Day 3-4: Fraud Detection**
- File: `itest-backend-qa-part2.md` + code examples
- Topics:
  - 7 fraud types
  - Detection methods
  - Fraud levels
  - Proctoring actions
- Exercises: Implement state machine for violations

**Day 5: Face Verification**
- File: `itest-backend-qa-part2.md`
- Topics:
  - Two-stage enrollment
  - Periodic verification
  - Queue-based processing
- Exercises: Design face verification flow

---

### **Week 4: Advanced Topics**

**Day 1: Scoring & Results**
- File: `itest-backend-qa-part2.md`
- Topics:
  - Auto-scoring logic
  - Answer comparison
  - Essay grading workflow
  - Score calculation
- Exercises: Implement scoring engine

**Day 2: Performance**
- File: `itest-backend-qa-part2.md` + code examples
- Topics:
  - Database indexing
  - Redis caching
  - Query optimization
  - Memory management
- Exercises: Optimize slow queries

**Day 3: Testing & Debugging**
- File: `itest-backend-qa-part2.md` + code examples
- Topics:
  - Exception handling
  - Input validation
  - Logging
  - Debugging scenarios
- Exercises: Debug 5 scenarios

**Day 4-5: Live Coding Practice**
- File: `itest-backend-code-examples.md`
- Practice all 10+ problems
- Simulate interview (1 hour each)

---

## 💡 **Interview Preparation Tips**

### **1. Before Interview (1 week)**

- [ ] Read Part 1 + Part 2 completely (2x if time allows)
- [ ] Practice 5 live coding problems
- [ ] Create note cards for key concepts
- [ ] Record yourself explaining architecture
- [ ] Do mock interview with friend

**Checklist:**
- Can explain JWT flow in < 3 min?
- Can draw exam creation transaction in < 5 min?
- Can explain 7 fraud types from memory?
- Can implement exam shuffle algorithm?

### **2. During Interview**

**Time allocation (60-min interview):**
- 5 min: Greeting + question about background
- 20 min: Architecture & design questions
- 20 min: Implementation question (live coding)
- 10 min: System design question
- 5 min: Your questions for interviewer

**Tips:**
- ✅ Explain before coding (show thinking process)
- ✅ Ask clarifying questions
- ✅ Test with examples
- ✅ Discuss trade-offs
- ✅ Mention from actual codebase

**Red flags to avoid:**
- ❌ Not familiar with codebase
- ❌ Can't explain architecture choices
- ❌ No knowledge of transaction handling
- ❌ Unfamiliar with JWT/auth
- ❌ No understanding of fraud detection

### **3. Key Topics to Memorize**

**Architecture:**
- 4 layers: Controller → Service → Repository → Database
- NestJS benefits: DI, modules, decorators
- Forwardref for circular dependencies

**Database:**
- Prisma ORM, PostgreSQL
- 5 key entity relationships
- Soft delete with `deletedAt`
- Composite unique keys

**Authentication:**
- JWT payload: { sub, numberCode, roleName, jti, needSetPassword }
- Access token: 15min, Refresh token: 7d
- Hash tokens before storing (security)

**Exam Management:**
- Transaction: Exam → Part → Question → QuestionAnswer (atomic)
- Flat vs grouped questions
- Random exam selection

**AI Integration:**
- Chunk strategy: PDF 60 pages = 6 chunks (10 pages each)
- Parallel limit: 6 concurrent Gemini calls
- Merge & dedup after chunking

**SSE:**
- 2 channels: teacher (broadcast) + student (isolated)
- Subscribe count for memory cleanup
- Memory leak prevention with `finalize()`

**Fraud Detection:**
- 7 types: FACE_MISMATCH, MULTIPLE_FACES, NO_FACE, TAB_SWITCHING, WINDOW_BLUR, IP_CHANGED, NETWORK_DISRUPTION
- Detection latency: instant (tab), 1-2s (face), 15s (heartbeat)
- Auto-suspend at 3 violations

**Scoring:**
- All-or-nothing (0 or full points)
- Auto-score all except ESSAY
- Score = correctAnswer match

---

## 🔍 **Common Interview Questions**

### **Q: Explain the architecture in 5 minutes**

**Answer structure:**
```
iTEST backend uses NestJS with 4-layer architecture:
1. Controllers receive HTTP requests
2. Services contain business logic
3. Repositories abstract database queries
4. Database layer (PostgreSQL + Prisma)

Key patterns:
- Dependency Injection (loose coupling)
- Transaction handling (atomic operations)
- SSE for real-time (teacher monitoring)
- Queue for async tasks (face verification)

Tech stack:
- NestJS (framework)
- Prisma (ORM)
- PostgreSQL (database)
- Redis (caching)
- RxJS (reactive)
- Bull (queue)
```

### **Q: How does exam creation work?**

**Answer structure:**
```
1. Receive CreateExamDto with parsed JSON
2. Validate ExamSet exists
3. Start transaction
4. Create Exam record
5. For each part:
   - Create Part
   - Create QuestionGroups (if grouped)
   - Create Questions
6. Create QuestionAnswers
7. Commit transaction
8. Return Exam

Key: Everything in 1 transaction → atomicity guaranteed
```

### **Q: How do you detect cheating?**

**Answer structure:**
```
7 fraud types detected:
- TAB_SWITCHING (< 100ms, visibilitychange event)
- WINDOW_BLUR (< 100ms, blur event)
- NO_FACE (1-2s, MediaPipe)
- MULTIPLE_FACES (1-2s, MediaPipe)
- FACE_MISMATCH (5-10s, periodic verify)
- IP_CHANGED (5-30s, heartbeat check)
- NETWORK_DISRUPTION (15s timeout)

Actions:
- 1st violation: WARNING
- 2nd violation: REPRIMAND
- 3rd+ violation: SUSPENSION (auto-submit)
```

### **Q: Explain real-time monitoring (SSE)**

**Answer structure:**
```
2-channel architecture:

Teacher Channel (broadcast):
- All teachers subscribe to session
- Receive all events: STUDENT_JOINED, VIOLATION, SUBMITTED
- Used for monitoring dashboard

Student Channel (isolated):
- Each student subscribes to own channel
- Receive only own events: WARNING, SUSPENSION, RETAKE_GRANTED
- Privacy: student doesn't see other students' events

Implementation:
- RxJS Subject (both Observable + Observer)
- In-memory Map tracking subscribers
- Clean up when subscriberCount = 0
- finalize() operator for teardown
```

---

## 📊 **Self-Assessment**

### **Before Interview: Rate yourself 1-5**

| Topic | Rating | Confidence Level |
|---|---|---|
| Architecture & Design | ? | Low ☐ Medium ☐ High ☐ |
| Database & ORM | ? | Low ☐ Medium ☐ High ☐ |
| Authentication | ? | Low ☐ Medium ☐ High ☐ |
| Exam Management | ? | Low ☐ Medium ☐ High ☐ |
| AI Integration | ? | Low ☐ Medium ☐ High ☐ |
| SSE & Real-time | ? | Low ☐ Medium ☐ High ☐ |
| Fraud Detection | ? | Low ☐ Medium ☐ High ☐ |
| Performance | ? | Low ☐ Medium ☐ High ☐ |

**Goal:** All "High" before interview

---

## 🛠️ **Practice Resources**

### **Live Coding Exercises**

From `itest-backend-code-examples.md`:

1. **Implement Exam Shuffle Algorithm** (20 min)
   - Difficulty: Medium
   - Topics: Algorithm, Seeded randomness

2. **Design Auto-save Draft Logic** (25 min)
   - Difficulty: Medium
   - Topics: Redis, Timing, State management

3. **Multi-chunk PDF Processing** (30 min)
   - Difficulty: Hard
   - Topics: Parallel processing, Error handling

4. **Fraud Detection State Machine** (25 min)
   - Difficulty: Hard
   - Topics: State transitions, Business logic

5. **Database Query Optimization** (20 min)
   - Difficulty: Medium
   - Topics: SQL, Indexing, Performance

---

## 📞 **Interview Day Checklist**

- [ ] Sleep well (8 hours minimum)
- [ ] Eat breakfast
- [ ] Review key concepts (30 min)
- [ ] Test internet connection
- [ ] Have water nearby
- [ ] Calm down (breathing exercises)
- [ ] Have notepad ready
- [ ] Write down clarifying questions

**During interview:**
- [ ] Introduce yourself briefly
- [ ] Listen carefully to questions
- [ ] Think before answering (5-10 seconds is OK)
- [ ] Ask for clarification if needed
- [ ] Show code examples when relevant
- [ ] Discuss trade-offs
- [ ] Ask about next steps

---

## ✅ **Expected Outcomes**

After studying this material, you should be able to:

✅ **Explain:**
- iTEST architecture in detail
- JWT authentication flow
- Exam creation transaction
- SSE real-time communication
- Fraud detection mechanisms

✅ **Implement:**
- Exam shuffling algorithm
- Auto-save draft logic
- Fraud detection state machine
- Database query optimization
- SSE channel management

✅ **Discuss:**
- Trade-offs in design choices
- Performance optimization strategies
- Security considerations
- Scaling approaches
- Error handling patterns

✅ **Answer:**
- Technical deep-dive questions
- Live coding problems
- System design questions
- Debugging scenarios
- Follow-up questions

---

## 📈 **Progress Tracking**

| Week | Topics | Status | Notes |
|---|---|---|---|
| Week 1 | Architecture, Database, Auth | ☐ Not Started ☐ In Progress ☐ Done | |
| Week 2 | Exam Mgmt, AI, Session | ☐ Not Started ☐ In Progress ☐ Done | |
| Week 3 | SSE, Fraud, Face Verification | ☐ Not Started ☐ In Progress ☐ Done | |
| Week 4 | Scoring, Performance, Testing | ☐ Not Started ☐ In Progress ☐ Done | |
| Practice | Live Coding (10 problems) | ☐ Not Started ☐ In Progress ☐ Done | |
| Mock | Mock Interview (1-2 rounds) | ☐ Not Started ☐ In Progress ☐ Done | |

---

## 🎓 **Learning Resources**

### **Official Documentation**
- [NestJS Docs](https://docs.nestjs.com/)
- [Prisma Docs](https://www.prisma.io/docs/)
- [PostgreSQL Docs](https://www.postgresql.org/docs/)
- [RxJS Docs](https://rxjs.dev/)
- [Google Gemini API](https://ai.google.dev/)

### **Recommended Reading**
- "Clean Code" by Robert C. Martin (architecture)
- "Designing Data-Intensive Applications" by Martin Kleppmann (database)
- "Web Security Academy" by PortSwigger (authentication)

### **Video References**
- NestJS tutorial (YouTube)
- PostgreSQL optimization (YouTube)
- Real-time web applications (SSE vs WebSocket)

---

## ❓ **FAQ**

**Q: How long should I study?**
- A: Minimum 2-3 weeks, 2-3 hours/day

**Q: Do I need to memorize all questions?**
- A: No, understand concepts and be able to explain

**Q: What if I don't know an answer?**
- A: Say "I'm not sure, but here's what I think..." and reason through it

**Q: Should I code during interview?**
- A: Only if asked, otherwise explain first then code

**Q: What if they ask something not in this guide?**
- A: Use the patterns you learned to reason through it

---

## 🚀 **Good Luck!**

You're well-prepared. Go ace that interview! 💪

**Remember:**
- Explain your thinking process
- Ask clarifying questions
- Show enthusiasm for the project
- Be honest about what you don't know
- Discuss trade-offs and alternatives

---

**Last Updated:** August 2026  
**Version:** 1.0  
**Total Material:** 150+ Q&As + 10 coding problems + debugging scenarios

Good luck! 🎯

