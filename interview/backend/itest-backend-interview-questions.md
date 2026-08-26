# 🎯 **iTEST Backend Interview Questions**
## Danh Sách Câu Hỏi Phỏng Vấn Backend iTEST - Phiên Bản Chi Tiết

> **Dưới đây là danh sách 150+ câu hỏi chi tiết về backend của hệ thống thi trực tuyến iTEST, phân loại theo chủ đề chính, từ cơ bản đến nâng cao.**

---

## 📑 **MỤC LỤC**

1. [Architecture & Design Patterns](#1-architecture--design-patterns)
2. [Database & Schema](#2-database--schema)
3. [Authentication & Authorization](#3-authentication--authorization)
4. [Exam Management](#4-exam-management)
5. [AI & Gemini Integration](#5-ai--gemini-integration)
6. [Exam Session & Attempt](#6-exam-session--attempt)
7. [Real-time Communication (SSE)](#7-real-time-communication-sse)
8. [Fraud Detection & Proctoring](#8-fraud-detection--proctoring)
9. [Face Verification](#9-face-verification)
10. [Scoring & Results](#10-scoring--results)
11. [Performance & Optimization](#11-performance--optimization)
12. [Testing & Error Handling](#12-testing--error-handling)

---

## 1. Architecture & Design Patterns

### 1.1 General Architecture

**Q1.1.1** Mô tả kiến trúc backend của iTEST. Backend được chia thành những tầng nào?

**Q1.1.2** iTEST sử dụng framework nào? NestJS hay Express? Tại sao chọn framework đó?

**Q1.1.3** Giải thích pattern Dependency Injection (DI) được sử dụng trong NestJS. Nó giúp gì cho dự án?

**Q1.1.4** Backend sử dụng Repository Pattern hay không? Nếu có, lợi ích là gì?

**Q1.1.5** Mô-đun nào là core module? Mô-đul nào phụ thuộc vào mô-đul nào (dependency graph)?

**Q1.1.6** Dự án sử dụng `forwardRef()` trong constructor. Khi nào cần sử dụng `forwardRef()`? Nó giải quyết vấn đề gì?

**Q1.1.7** Giải thích sự khác nhau giữa `@Module()`, `@Injectable()` và `@Controller()` decorator.

---

### 1.2 Service & Repository Layer

**Q1.2.1** Phân biệt giữa Service layer và Repository layer. Tại sao cần tách riêng?

**Q1.2.2** `ExamAttemptService` phụ thuộc vào `ExamSessionService`. Có thể tạo circular dependency không? Cách handle?

**Q1.2.3** Giải thích pattern `getOrThrow()` vs `getOrNull()`. Khi nào nên dùng cái nào?

**Q1.2.4** Repository method `findByIdOrThrow()` trong `ExamService` có gì khác so với `findById()`?

---

### 1.3 Middleware & Guard

**Q1.3.1** Backend có middleware xác thực (Auth Middleware) không? Nó hoạt động như thế nào?

**Q1.3.2** RBAC (Role-Based Access Control) được implement ở tầng nào? Guard hay Interceptor?

**Q1.3.3** Giải thích cơ chế JWT trong middleware. Token được lưu ở đâu: Cookie hay Header?

**Q1.3.4** Middleware có thể block request trước khi tới controller không? Có ví dụ không?

---

## 2. Database & Schema

### 2.1 Database Design

**Q2.1.1** Dự án sử dụng ORM nào? Prisma, TypeORM, hay Sequelize? Vì sao?

**Q2.1.2** Database chính sử dụng PostgreSQL hay MySQL? Tại sao chọn?

**Q2.1.3** Giải thích quan hệ giữa các entity:
- `Exam` ↔ `ExamSet` ↔ `Course`
- `Part` ↔ `Question` ↔ `QuestionGroup`
- `ExamAttempt` ↔ `ExamSession` ↔ `StudentAnswer`

**Q2.1.4** Field `@unique([name, courseId])` trong `ExamSet` có ý nghĩa gì? Tại sao không dùng `@unique` đơn giản?

**Q2.1.5** Column `deletedAt` dùng soft delete hay hard delete? Tại sao?

**Q2.1.6** Index được tạo trên những field nào? Tại sao lại index những field đó?

---

### 2.2 Relationships & Constraints

**Q2.2.1** Mối quan hệ 1-N giữa `Exam` và `Part` là gì? Có ràng buộc gì không?

**Q2.2.2** `onDelete: Cascade` có ý nghĩa gì? Khi xóa `ExamSet`, điều gì sẽ xảy ra?

**Q2.2.3** `ExamAttempt` có @@unique([studentId, examSessionId]) là gì? Cho phép cùng 1 sinh viên tham gia 2 lần không?

**Q2.2.4** Giải thích `@unique([resultId, teacherId, role])` trong `ResultGrading`. Tại sao cần unique kết hợp 3 field?

**Q2.2.5** `QuestionAnswer` là 1-1 với `Question` hay 1-N? Tại sao?

---

### 2.3 Enums & Statuses

**Q2.3.1** Liệt kê tất cả `ExamAttemptStatus`. Thứ tự chuyển đổi status là gì?

**Q2.3.2** `FraudLevel` có 3 mức: LOW, MEDIUM, HIGH. Mức nào dẫn tới SUSPENSION?

**Q2.3.3** `FraudType` được liệt kê ra sao? Loại gian lận nào được phát hiện từ camera?

**Q2.3.4** `ExamSessionStatus` có giá trị nào? Quy trình chuyển đổi là gì?

**Q2.3.5** `ResultStatus` chỉ có 2 giá trị: NOT_GRADED, PUBLISHED. Tại sao không có GRADING?

---

### 2.4 Data Integrity

**Q2.4.1** Schema sử dụng transaction để đảm bảo tính nguyên tử (atomicity) trong những trường hợp nào?

**Q2.4.2** Khi tạo `Exam`, part, question, answer được tạo trong 1 transaction hay nhiều transaction?

**Q2.4.3** Có foreign key constraint không? Nếu có, conflict nào có thể xảy ra?

---

## 3. Authentication & Authorization

### 3.1 JWT & Token Management

**Q3.1.1** Giải thích JWT payload trong `buildAuthPayload()`. Payload chứa những field nào?

```typescript
{
  sub: accountId,
  numberCode: studentId_or_teacherId,
  roleName: ADMIN | TEACHER | STUDENT,
  jti: uuid,
  needSetPassword?: boolean
}
```

**Q3.1.2** Access Token và Refresh Token khác nhau ở điểm nào? Mỗi cái có expiry bao lâu?

**Q3.1.3** Refresh Token được lưu ở đâu: Database, Redis, hay Cookie? Tại sao?

**Q3.1.4** Giải thích hàm `hashToken()`. Tại sao không lưu raw token vào database?

**Q3.1.5** Khi user logout, refresh token được xóa hay revoke? Cơ chế là gì?

---

### 3.2 Google OAuth Integration

**Q3.2.1** Flow xác thực Google OAuth trong `validateGoogleUser()` là gì?

**Q3.2.2** Nếu user Google lần đầu đăng nhập (không tìm thấy theo googleId và email), hệ thống làm gì?

**Q3.2.3** Trường `needSetPassword` dùng để làm gì? Khi nào được set thành `true`?

**Q3.2.4** Email domain whitelist được kiểm tra ở đâu? Những domain nào được phép?

---

### 3.3 Role-Based Access Control (RBAC)

**Q3.3.1** Liệt kê 3 roles chính trong hệ thống. Mỗi role có quyền gì?

**Q3.3.2** Giải thích `resolveAccountCodeByRole()`. Tại sao Admin không cần `numberCode`?

**Q3.3.3** Role được quản lý trong table `Role` hay hardcode trong constants?

**Q3.3.4** Middleware xác thực role dựa trên JWT payload hay database query?

**Q3.3.5** Có cách phân quyền chi tiết hơn role không? (e.g., permission-based)

---

## 4. Exam Management

### 4.1 Exam Creation & Structure

**Q4.1.1** Quy trình tạo `Exam` trong `create()` method gồm những bước nào?

**Q4.1.2** Khi tạo exam, dữ liệu được lưu thế nào?
- Chỉ `Exam` record?
- Hay `Exam` + `Part` + `Question` + `QuestionAnswer` đều được tạo?

**Q4.1.3** `validatePart()` kiểm tra điều gì? Nếu không có `questionGroups`, logic sẽ như thế nào?

**Q4.1.4** Field `examCodes` là mảng string. Nó dùng để làm gì? Mỗi code dùng bao nhiêu lần?

**Q4.1.5** Giải thích logic `insertPartWithDefaultGroup()`. Khi nào được sử dụng?

---

### 4.2 Exam Status & Publishing

**Q4.2.1** Trạng thái `PENDING` → `ACCEPTED` → `REJECTED` là quy trình gì?

**Q4.2.2** `resolveExamStatus()` dựa trên `createdBy` role. Admin tạo exam có status gì?

**Q4.2.3** Có thể chỉnh sửa exam sau khi được `ACCEPTED` không?

**Q4.2.4** Exam bị `REJECTED` có thể tạo lại hay phải xóa và create mới?

---

### 4.3 Question & Answer Management

**Q4.3.1** `questionNumber` và `questionIndex` khác gì? Cái nào dùng để sắp xếp hiển thị?

**Q4.3.2** QuestionType có những giá trị nào? Mỗi type có structure `options` khác nhau không?

**Q4.3.3** `correctAnswer` là mảng hay string? Loại câu hỏi nào cho phép đáp án đúng nhiều?

**Q4.3.4** Field `points` là gì? Mỗi câu hỏi có số điểm khác nhau không?

**Q4.3.5** Upsert logic trong `upsertAnswers()` là gì? Khi nào dùng `CREATE` vs `UPDATE`?

---

### 4.4 Exam Shuffling & Randomization

**Q4.4.1** `ExamShuffleHelper.shuffleExam()` làm gì? Câu hỏi được shuffle khi nào?

**Q4.4.2** Shuffle logic dựa trên `examAttemptId`. Tại sao lại dùng `examAttemptId` làm seed?

**Q4.4.3** Có thể disable shuffle hay phải luôn shuffle?

**Q4.4.4** Shuffle part, question group, hoặc individual question?

---

## 5. AI & Gemini Integration

### 5.1 PDF Processing & Chunking

**Q5.1.1** `generateText()` là entry point để sinh đề bằng AI. Flow chính là gì?

**Q5.1.2** PDF được fetch từ URL hay upload? Nếu từ URL, timeout bao lâu?

**Q5.1.3** Số trang PDF < 4 (minPagesToParallelize) được xử lý thế nào? Có khác so với PDF > 4?

**Q5.1.4** Chunk strategy là gì? PDF 60 trang được chia thành bao nhiêu chunk?

**Q5.1.5** `overlapPages = 1` là gì? Tại sao cần overlap?

---

### 5.2 Parallel Processing

**Q5.2.1** `pLimit()` dùng để làm gì? Nó giới hạn bao nhiêu luồng song song?

**Q5.2.2** Nếu 1 chunk fail, cả toàn bộ request có fail không? Có retry logic không?

**Q5.2.3** Gọi Gemini 6 chunk cùng lúc có risk gì? Rate limit?

---

### 5.3 Gemini Model Configuration

**Q5.3.1** Model được sử dụng là `gemini-2.5-flash-lite`. Tại sao lại chọn model này?

**Q5.3.2** `temperature = 0` có ý nghĩa gì? Nếu tăng thành 1, điều gì sẽ khác?

**Q5.3.3** `thinkingBudget` là gì? Nó phụ thuộc vào số trang không?

**Q5.3.4** Prompt được gửi như thế nào? Prompt là hardcode hay được tham số hóa?

---

### 5.4 Post-Processing & Validation

**Q5.4.1** `postProcessPayload()` làm gì? Nó xóa duplicate hay normalize dữ liệu?

**Q5.4.2** `mergeExamPayloads()` gộp kết quả từ nhiều chunk. Logic merge là gì?

**Q5.4.3** JSON response từ Gemini có bao giờ invalid không? Có fallback logic không?

**Q5.4.4** Dữ liệu sinh được từ AI phải validate trước khi lưu database không?

---

### 5.5 Error Handling & Logging

**Q5.5.1** Nếu fetch PDF fail, exception nào được throw?

**Q5.5.2** Logging message có bao gồm elapsed time không? Để làm gì?

**Q5.5.3** Nếu Gemini API error (500), hệ thống retry không?

---

## 6. Exam Session & Attempt

### 6.1 Exam Session Management

**Q6.1.1** `ExamSession` đại diện cho cái gì? 1 session = 1 phòng thi à?

**Q6.1.2** Session có những thông tin nào? (date, room, duration, isCameraRequired, ...)

**Q6.1.3** Trạng thái session có những giá trị nào? (NOT_STARTED, IN_PROGRESS, PAUSE, FINISHED)

**Q6.1.4** `changeExamSessionStatus()` phải trigger những logic nào? (e.g., notify students, pause attempts, ...)

**Q6.1.5** Session được lock hay không? `isLocked` field có ý nghĩa gì?

---

### 6.2 Exam Registration

**Q6.2.1** `ExamRegistration` là gì? Khác với `ExamAttempt` như thế nào?

**Q6.2.2** Student phải register trước khi tham gia session không? Thứ tự là gì?

**Q6.2.3** Registration status có những giá trị nào? (REGISTERED, BLOCKED, BANNED, CANCELLED)

**Q6.2.4** `isAccessGranted` field được set khi nào? Khi face verification pass?

**Q6.2.5** `candidateNumber` là gì? Dùng để làm gì?

---

### 6.3 Exam Attempt Lifecycle

**Q6.3.1** `ExamAttempt` đại diện cho cái gì? 1 sinh viên làm 1 ca thi = 1 attempt à?

**Q6.3.2** Attempt status có thứ tự chuyển đổi nào? IN_PROGRESS → ? → ? → COMPLETED

**Q6.3.3** Khi student vào thi, attempt nào được tạo?

**Q6.3.4** Nếu student mất kết nối, attempt status thay đổi gì? DISCONNECTED hay vẫn IN_PROGRESS?

**Q6.3.5** Có thể allow student reconnect và tiếp tục làm bài không?

---

### 6.4 Time Tracking & Consumed Time

**Q6.4.1** `consumedTime` được tính như thế nào? Là tổng thời gian hay delta?

**Q6.4.2** Khi student tạm dừng làm bài, `consumedTime` có được cập nhật không?

**Q6.4.3** `lastResumedAt` dùng để tính gì? Thời gian còn lại của student?

**Q6.4.4** Công thức tính thời gian còn lại là gì?
```
remainingTime = duration * 60 - consumedTime (in seconds)
```

**Q6.4.5** Nếu server trả về `consumedTime` mới sau khi tạm dừng, frontend cập nhật như thế nào?

---

### 6.5 Join Exam Session

**Q6.5.1** Student join session qua endpoint nào? Cần những parameter gì?

**Q6.5.2** Join flow gồm những bước nào?
1. Kiểm tra registration
2. Face verification (nếu cần)
3. Tạo ExamAttempt
4. Return exam data

**Q6.5.3** Nếu student đã tham gia lần trước, attempt được khôi phục hay tạo mới?

**Q6.5.4** Draft answers được khôi phục từ đâu khi re-join?

---

## 7. Real-time Communication (SSE)

### 7.1 SSE Architecture

**Q7.1.1** SSE (Server-Sent Events) được implement như thế nào? Dùng RxJS Observable?

**Q7.1.2** Có 2 channel riêng: teacher channel và student channel. Mục đích là gì?

**Q7.1.3** Teacher channel broadcast event gì? (STUDENT_JOINED, VIOLATION, SUBMITTED, ...)

**Q7.1.4** Student channel chỉ gửi event nào? (RETAKE_GRANTED, SUSPENSION, WARNING, ...)

**Q7.1.5** SSE connection lâu dài bao lâu? Có timeout hay reconnect?

---

### 7.2 Channel Management

**Q7.2.1** In-memory Map được dùng để lưu trữ channel. Nó scale được không? Nếu 10,000 users?

**Q7.2.2** `subscriberCount` dùng để làm gì? Khi nào clean up channel?

**Q7.2.3** Nếu client disconnect, channel có bị xóa không? Hay chờ cho tới khi không còn subscriber?

**Q7.2.4** Có thể persist SSE state vào Redis không? (e.g., Redis pub/sub)

---

### 7.3 Event Emission

**Q7.3.1** `emitEvent()` gửi event đến teacher channel. Nó broadcast hay unicast?

**Q7.3.2** `emitToStudent()` gửi event đến 1 student. Key format là gì? (examSessionId:studentId)

**Q7.3.3** Event object có structure nào? (type, data, timestamp, examSessionId)

**Q7.3.4** Nếu subscriber mất kết nối (unsubscribe), event vẫn được gửi không?

---

### 7.4 Real-time Events List

**Q7.4.1** STUDENT_JOINED event khi nào được emit?

**Q7.4.2** STUDENT_SUBMITTED event chứa dữ liệu gì?

**Q7.4.3** STUDENT_VIOLATION event được emit khi nào? (fraud detection trigger)

**Q7.4.4** SESSION_STATUS_CHANGED event được emit khi session status thay đổi. Frontend phải handle gì?

**Q7.4.5** WARNING, REPRIMAND, SUSPENSION là proctoring handle events. Điều gì khác?

**Q7.4.6** RETAKE_GRANTED event gửi thông tin gì? (attemptId, examId, status)

**Q7.4.7** ATTEMPT_PAUSED, ATTEMPT_RESUMED, ATTEMPT_DISCONNECTED là gì? Khi nào emit?

---

### 7.5 Error Handling

**Q7.5.1** Nếu emit event fail (subscriber exception), cả SSE có bị down không?

**Q7.5.2** Có exponential backoff retry khi emit fail không?

---

## 8. Fraud Detection & Proctoring

### 8.1 Fraud Types & Detection

**Q8.1.1** Liệt kê 7 loại gian lận được phát hiện. Mỗi cái được phát hiện như thế nào?

| Fraud Type | Detection Method |
|---|---|
| FACE_MISMATCH | Face verification |
| MULTIPLE_FACES | MediaPipe |
| NO_FACE | MediaPipe |
| TAB_SWITCHING | visibilitychange event |
| WINDOW_BLUR | blur event |
| IP_CHANGED | Compare client IP |
| NETWORK_DISRUPTION | Heartbeat timeout |

**Q8.1.2** Bao lâu detect được 1 violation? Là real-time hay có delay?

**Q8.1.3** Một student có thể bị phát hiện violation nhiều lần không? Mỗi violation được log riêng không?

**Q8.1.4** `warningCount` field trong ExamAttempt dùng để làm gì?

---

### 8.2 Fraud Level & Action

**Q8.2.1** Fraud level có 3 mức: LOW, MEDIUM, HIGH. Mỗi violation thuộc mức nào?

**Q8.2.2** Khi fraud level tăng, giám thị sẽ nhận được alert không?

**Q8.2.3** Giám thị có thể manual trigger proctoring action không? Hay chỉ từ violation detection?

---

### 8.3 Proctoring Actions

**Q8.3.1** Giám thị có thể thực hiện 4 actions: WARNING, REPRIMAND, SUSPENSION, STOP_FOR_SESSION_TRANSFER. Mỗi cái effect gì?

**Q8.3.2** WARNING action gửi popup alert đến student không? Có block làm bài không?

**Q8.3.3** SUSPENSION action tự động nộp bài hay student phải submit manually?

**Q8.3.4** STOP_FOR_SESSION_TRANSFER dùng trong trường hợp gì?

**Q8.3.5** Sau khi WARNING, student vẫn có thể tiếp tục làm bài không?

---

### 8.4 Fraud Detail Logging

**Q8.4.1** `FraudDetail` table lưu thông tin gì về mỗi violation?

**Q8.4.2** Mỗi violation có một record trong FraudDetail hay gộp lại?

**Q8.4.3** Violation được report sau bao lâu? Real-time hay batch?

---

## 9. Face Verification

### 9.1 Face Registration & Embedding

**Q9.1.1** User phải upload khuôn mặt khi đăng ký không? Hay khi join exam session?

**Q9.1.2** `faceEmbedding` field lưu trữ gì? Là float array hay base64?

**Q9.1.3** Face embedding được tạo bằng model gì? (e.g., face_recognition, mediapipe, ...)

**Q9.1.4** Có thể update face embedding sau khi đã đăng ký không?

---

### 9.2 Face Verification at Entry

**Q9.2.1** Khi student join exam session, có bắt phải xác thực khuôn mặt không? Hay tùy vào `isCameraRequired`?

**Q9.2.2** Xác thực khuôn mặt gọi API nào? FastAPI Backend?

**Q9.2.3** Nếu xác thực fail, student có thể retry không? Bao nhiêu lần?

**Q9.2.4** Kết quả xác thực được lưu ở đâu? ExamRegistration hay ExamAttempt?

---

### 9.3 Periodic Face Verification During Exam

**Q9.3.1** Trong quá trình làm bài, có tự động capture ảnh khuôn mặt không? Bao lâu 1 lần?

**Q9.3.2** Hình ảnh được gửi đến backend hay xử lý trên client?

**Q9.3.3** Nếu periodic face verification fail, là violation loại nào?

**Q9.3.4** Face verification fail 3 lần → SUSPENSION?

---

### 9.4 Face Verification Service

**Q9.4.1** `verifyFace()` method khi gọi API external (FastAPI). Có retry logic không?

**Q9.4.2** Response từ FastAPI chứa gì? (match, accuracy, distance, tolerance)

**Q9.4.3** `tolerance` parameter dùng để tùy chỉnh độ nhạy. Cao hay thấp sẽ lỏng?

**Q9.4.4** Nếu API external down, exception nào được throw?

---

### 9.5 Queue-based Face Verification

**Q9.5.1** `FaceVerificationProducer` dùng queue nào? Bull, RabbitMQ, hay Redis Streams?

**Q9.5.2** Tại sao dùng async queue để xử lý face verification? Không phải realtime à?

**Q9.5.3** Job data trong queue chứa gì? (examAttemptId, imageBuffer, ...)

**Q9.5.4** Retry attempts là bao nhiêu? Backoff strategy?

**Q9.5.5** Khi job complete, kết quả được lưu ở đâu? Database hay Redis?

---

## 10. Scoring & Results

### 10.1 Automatic Scoring

**Q10.1.1** Multiple choice được auto-score bằng cách so sánh student answer với `correctAnswer`.

**Q10.1.2** Nếu question có `points = 10`, student chọn đúng được 10 điểm, sai được 0 điểm à?

**Q10.1.3** Có partial credit không? (e.g., chọn đúng 2/3 đáp án được 6.67 điểm)

**Q10.1.4** Nếu không chọn câu nào (leave blank), được bao nhiêu điểm?

---

### 10.2 Essay Scoring

**Q10.2.1** Bài tự luận được xử lý như thế nào? Auto-score hay manual?

**Q10.2.2** Teacher có thể score essay hay admin?

**Q10.2.3** `ResultGrading` role có 2 giá trị: REVIEWER, FINAL_APPROVER. Mỗi role làm gì?

**Q10.2.4** Có workflow review để confirm điểm trước khi publish không?

---

### 10.3 Result & Score Detail

**Q10.3.1** `Result` table lưu `scoreDetail`. Nó là JSON object hay text?

**Q10.3.2** scoreDetail chứa score của từng câu hay chỉ tổng điểm?

**Q10.3.3** `ResultStatus` có 2 giá trị: NOT_GRADED, PUBLISHED. Khi nào được publish?

**Q10.3.4** Có thể regrade sau khi đã published không?

---

### 10.4 Grading Workflow

**Q10.4.1** ExamSet có `hasEssay = true`, grading flow là gì?

**Q10.4.2** Làm sao để biết exam có essay hay không? Query database mỗi lần?

**Q10.4.3** Teacher assignment cho việc chấm essay. Có thể assign cùng 1 teacher hay không?

---

## 11. Performance & Optimization

### 11.1 Database Optimization

**Q11.1.1** Index nào là quan trọng nhất? `(examSessionId, studentId)` hay `(studentId, examSessionId)`?

**Q11.1.2** Query lấy student answer cho 100 students có thể slow không? Có batch query không?

**Q11.1.3** Có materialized view hay aggregate table không? (e.g., exam session statistics)

**Q11.1.4** Pagination được implement với limit/offset hay cursor-based?

---

### 11.2 Redis Caching

**Q11.2.1** Redis được dùng để cache gì?
- Draft answers? (getCacheExamDraftKey)
- Exam shuffle seed?
- Student presence?

**Q11.2.2** Redis key format là gì? `exam:draft:{sessionId}:{studentId}`?

**Q11.2.3** Draft TTL bao lâu? Khi session kết thúc có xóa không?

**Q11.2.4** Có thể dùng Redis để cache tài nguyên tĩnh (exam data, part data) không?

---

### 11.3 API Response Optimization

**Q11.3.1** `getExamContent()` trả về toàn bộ exam data (all parts, questions). Có thể pagination không?

**Q11.3.2** Có lazy-load câu hỏi hay load hết một lần?

**Q11.3.3** Response payload to thì có gzip compression không?

---

### 11.4 Concurrency & Rate Limiting

**Q11.4.1** Nếu 100 students submit cùng lúc, database có bottleneck không?

**Q11.4.2** Có rate limiter trên endpoint không? (e.g., 10 submit/minute/student)

**Q11.4.3** `pLimit()` được dùng ở đâu? (e.g., Gemini API calls, exam shuffle)

---

### 11.5 Memory Management

**Q11.5.1** SSE in-memory Map có giới hạn không? 10,000 active sessions có thể hold được không?

**Q11.5.2** `ExamAttemptService.onModuleInit()` tạo Redis subscriber. Có memory leak không?

**Q11.5.3** Khi server scale horizontally (multi-process), SSE channel sync như thế nào?

---

## 12. Testing & Error Handling

### 12.1 Error Handling

**Q12.1.1** Exception nào được throw khi resource not found? `NotFoundException` hay `BadRequestException`?

**Q12.1.2** Khi transaction fail, có rollback automatic không?

**Q12.1.3** Có custom exception class không? (e.g., `InvalidExamStructureException`)

**Q12.1.4** Error message có sensitive information không? (e.g., DB query, file path)

---

### 12.2 Validation & Guards

**Q12.2.1** Input validation ở tầng nào? DTO, Pipe, hay Controller?

**Q12.2.2** ExamAttempt có validate trạng thái trước khi submit không?

**Q12.2.3** Student answer có validate question type không? (e.g., single choice phải là string)

---

### 12.3 Logging & Monitoring

**Q12.3.1** Logger được dùng ở đâu? Error, Warn, hay Info level?

**Q12.3.2** Có structured logging (JSON format) không?

**Q12.3.3** Performance metric có log không? (elapsed time, query count, ...)

---

### 12.4 Testing Strategy

**Q12.4.1** Unit test có cover service không? Hay chỉ integration test?

**Q12.4.2** Có mock database, Redis, Gemini API không?

**Q12.4.3** E2E test flow là gì? (register → join session → make submission → check result)

**Q12.4.4** Performance test có khi nào không? (e.g., load test 1000 concurrent students)

---

## 🎓 **Câu Hỏi Khó / Advanced**

### A1. Complex Scenarios

**A1.1** Student tham gia exam, nửa chừng server bị restart. Draft được recover không?

**A1.2** 2 students cùng 1 lần làm bài, cùng 1 ca thi nhưng bị nhầm exam code. Điều gì có thể xảy ra?

**A1.3** Teacher thay đổi exam status từ PENDING → ACCEPTED khi exam đang được sử dụng. Có conflict không?

**A1.4** Gemini API timeout sau khi xử lý chunk thứ 3 của 6 chunk. Kết quả sẽ như thế nào?

**A1.5** SSE connection bị disconnect, student disconnect event được log. Khi nào được resume?

---

### A2. Design Questions

**A2.1** Nếu scale từ 100 users → 10,000 users, kiến trúc nào cần thay đổi?

**A2.2** Giải pháp để scale SSE (server in-memory channel) thành multi-server setup?

**A2.3** Làm sao để backup/restore exam session state nếu server crash?

**A2.4** Nếu Gemini API rate-limited, làm sao để queue requests?

---

### A3. Real-world Issues

**A3.1** False positive: student bị phát hiện 2 khuôn mặt vì filter cam. Làm sao handle?

**A3.2** Lag phát hiện: TAB_SWITCHING được detect sau 2 giây. Có thể optimize không?

**A3.3** Drift: một số students có internet yếu, heartbeat timeout liên tục. Timeout nên là bao lâu?

---

## 📊 **Summary Table**

| Component | Technology | Purpose |
|---|---|---|
| Framework | NestJS | Backend framework |
| Database | PostgreSQL + Prisma | Data persistence |
| Caching | Redis | Draft, presence, SSE |
| Real-time | RxJS SSE | Teacher & Student events |
| AI | Google Gemini API | Exam generation |
| Queue | Bull/Redis | Face verification async |
| Face Recognition | FastAPI | Face matching |
| Authentication | JWT + Google OAuth | User auth |

---

## 📚 **References**

- Prisma Documentation: https://www.prisma.io/docs/
- NestJS Documentation: https://docs.nestjs.com/
- Google Gemini API: https://ai.google.dev/
- Redis Commands: https://redis.io/commands/
- Server-Sent Events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events

---

**Compiled & Written by:** Backend Engineering Team  
**Last Updated:** August 2026  
**Version:** 1.0

