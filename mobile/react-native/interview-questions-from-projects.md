# 🎯 **Câu Hỏi Phỏng Vấn IT Intern - Dựa Trên Projects Trong CV**

> **Phỏng vấn viên:** Đây là danh sách câu hỏi chi tiết về các project trong CV của bạn, kèm câu trả lời mẫu để giúp ứng viên chuẩn bị tốt.

---

## **PROJECT 1: iTEST - Online Examination Platform**

### **Phần 1: Câu Hỏi Về AI & Gemini Integration**

---

### **Q1.1: Bạn đã dùng Gemini API để convert PDF sang structured exams. Giải thích quy trình này?**

**Mục đích câu hỏi:** Kiểm tra hiểu biết về AI integration, API calling, data processing

**Câu trả lời mẫu:**

Quy trình gồm 3 bước chính:

**Bước 1: Upload & Extract PDF**
- User upload file PDF (giáo trình)
- Backend nhận file, đọc nội dung PDF
- Lưu tạm vào server hoặc cloud storage (Supabase)

**Bước 2: Gọi Gemini API**
```
Prompt gửi đi:
"Từ nội dung PDF này, tạo một bộ đề thi với:
- 20 câu hỏi trắc nghiệm
- Mỗi câu có 4 đáp án
- Chỉ có 1 đáp án đúng
- Định dạng JSON"

Response từ Gemini:
{
  "questions": [
    {
      "question": "...",
      "options": ["A", "B", "C", "D"],
      "correct_answer": "A"
    }
  ]
}
```

**Bước 3: Xử lý & Lưu Database**
- Parse JSON response từ Gemini
- Kiểm tra validate (đủ câu hỏi, định dạng đúng)
- Loại bỏ câu hỏi trùng lặp
- Lưu vào database (MongoDB/Supabase)
- Trả về cho frontend để user chỉnh sửa

**Vấn đề đã giải quyết:**
- ✅ Duplicate elimination: So sánh tương tự giữa các câu hỏi
- ✅ Error handling: Nếu Gemini fail, retry hoặc thông báo user
- ✅ Parallel processing: Nếu PDF dài, chia thành chunks gửi song song

---

### **Q1.2: PDF của bạn có thể rất dài (100+ trang). Làm sao bạn xử lý mà không bị timeout?**

**Mục đích câu hỏi:** Kiểm tra khả năng xử lý edge case, scaling, chunking strategy

**Câu trả lời mẫu:**

Nếu PDF rất dài, gọi Gemini 1 lần với toàn bộ content có thể:
- ❌ Timeout (Gemini API có limit thời gian)
- ❌ Lỗi "token quá nhiều"
- ❌ Kết quả không tốt

**Giải pháp: Chunking + Parallel Processing**

```javascript
// Chia PDF thành chunks
const chunks = splitPDFIntoChunks(pdfContent, chunkSize = 10); 
// 100 trang → 10 chunks × 10 trang = 10 API calls

// Gọi song song (không chờ từng cái)
const results = await Promise.all(
  chunks.map(chunk => callGeminiAPI(chunk))
);

// Gộp kết quả từ tất cả chunks
const allQuestions = mergeResults(results);
```

**Cụ thể:**
- Chia PDF thành 10 phần (mỗi phần 10 trang)
- Gọi Gemini 10 lần song song (thay vì 1 lần chờ dài)
- Mỗi chunk chỉ mất 5-10 giây
- Tổng thời gian: ~10 giây (không phải 50 giây)

**Xử lý sau merge:**
```javascript
// Loại bỏ trùng lặp
const uniqueQuestions = removeDuplicates(allQuestions);

// So sánh semantic: "Thủ đô của Pháp là?" vs "Paris là..."
// → Coi như 1 câu
```

---

### **Q1.3: Nếu user gọi generate exam 2 lần cùng lúc, có thể xảy ra vấn đề gì?**

**Mục đích câu hỏi:** Kiểm tra khả năng nhận ra race condition, concurrency issues

**Câu trả lời mẫu:**

**Vấn đề có thể xảy ra:**

```
User A: Click "Generate Exam" lần 1
↓ Start Gemini API call (5 giây)
↓ (2 giây sau)
User A: Click "Generate Exam" lần 2
↓ Start Gemini API call thứ 2 (5 giây)
↓ Cùng lúc có 2 API calls chạy
```

**Hậu quả:**
1. **Cost cao hơn:** 2 API calls = cost 2x
2. **Data confusing:** Kết quả từ 2 calls gộp lại không đúng
3. **Database error:** Lưu 2 bộ đề thay vì 1

**Giải pháp:**

```javascript
// 1. Disable button khi đang generate
<button 
  onClick={handleGenerate} 
  disabled={isGenerating}  // ← Disable trong lúc loading
>
  {isGenerating ? "Generating..." : "Generate"}
</button>

// 2. Backend: Kiểm tra nếu đã có request đang chạy
let generatingTaskId = null;

app.post('/generate-exam', async (req, res) => {
  if (generatingTaskId) {
    return res.status(400).json({ 
      error: "Already generating, please wait..." 
    });
  }
  
  generatingTaskId = generateUUID();
  try {
    const result = await callGemini();
    return res.json(result);
  } finally {
    generatingTaskId = null;  // Reset
  }
});

// 3. Hoặc dùng Queue (BullMQ) - chờ lần lượt
const queue = new Queue('generateExam');
queue.add({ pdfId: '123' });  // Tự động xếp hàng chờ
```

**Best practice:** Dùng kết hợp:
- ✅ Disable button frontend (UX)
- ✅ Validation backend (security)
- ✅ Queue system (scaling)

---

### **Phần 2: Câu Hỏi Về Exam Editor & Hierarchical Structure**

---

### **Q1.4: Bạn đã implement Exam Editor hỗ trợ multiple question types (single choice, multiple choice, essay, v.v). Database structure của bạn như thế nào?**

**Mục đích câu hỏi:** Kiểm tra database design, schema flexibility, ORM usage

**Câu trả lời mẫu:**

**Yêu cầu:**
- 1 exam có nhiều parts (phần)
- 1 part có nhiều questions
- 1 question có nhiều options (nếu multiple choice)
- 1 question có 1 hoặc nhiều đáp án đúng (multiple correct answers)

**Database Schema (MongoDB):**

```javascript
// Collections
db.exams
db.parts
db.questions
db.options
db.question_answers

// Exam
{
  _id: "exam123",
  title: "Thi Tiếng Anh",
  duration: 120,  // phút
  parts: ["part1", "part2"]
}

// Part (phần thi)
{
  _id: "part1",
  examId: "exam123",
  partIndex: 1,
  title: "Reading",
  questions: ["q1", "q2", "q3"]
}

// Question
{
  _id: "q1",
  partId: "part1",
  type: "SINGLE_CHOICE",  // hoặc MULTIPLE_CHOICE, ESSAY
  content: "What is the capital of France?",
  options: ["opt1", "opt2", "opt3", "opt4"]
}

// Option
{
  _id: "opt1",
  questionId: "q1",
  label: "A",
  text: "Paris"
}

// Question Answer (đáp án đúng)
{
  _id: "qa1",
  questionId: "q1",
  correctAnswers: ["opt1"],  // Mảng vì có thể multiple
  points: 1
}
```

**Hoặc SQL (Supabase/PostgreSQL):**

```sql
CREATE TABLE exams (
  id UUID PRIMARY KEY,
  title VARCHAR,
  duration INT
);

CREATE TABLE parts (
  id UUID PRIMARY KEY,
  exam_id UUID REFERENCES exams,
  part_index INT,
  title VARCHAR
);

CREATE TABLE questions (
  id UUID PRIMARY KEY,
  part_id UUID REFERENCES parts,
  type ENUM('SINGLE_CHOICE', 'MULTIPLE_CHOICE', 'ESSAY'),
  content TEXT,
  points INT
);

CREATE TABLE options (
  id UUID PRIMARY KEY,
  question_id UUID REFERENCES questions,
  label CHAR(1),  -- 'A', 'B', 'C', ...
  text VARCHAR
);

CREATE TABLE question_answers (
  id UUID PRIMARY KEY,
  question_id UUID REFERENCES questions UNIQUE,
  correct_answer_ids UUID[],  -- Mảng UUID
  points INT
);
```

**Ưu điểm:**
- ✅ Flexible: Hỗ trợ tất cả loại câu hỏi
- ✅ Scalable: Dễ thêm loại câu mới
- ✅ Efficient: Query nhanh (index trên foreign keys)

---

### **Q1.5: Exam Editor của bạn support math formula rendering (KaTeX). Nếu user nhập công thức sai, frontend có render không? Cách xử lý là gì?**

**Mục đích câu hỏi:** Kiểm tra error handling, user experience, library knowledge

**Câu trả lời mẫu:**

**Vấn đề:**
```
User nhập: \frac{1}{0}  (chia cho 0 - công thức invalid)
Nếu KaTeX không xử lý → error, crash
```

**Giải pháp:**

```javascript
import { render } from 'katex';

const renderMathFormula = (latex) => {
  try {
    // Thử render công thức
    const html = render(latex, { throwOnError: false });
    return html;
  } catch (error) {
    // Nếu lỗi, hiển thị thông báo
    console.error('Invalid formula:', error);
    return `<span class="error">Invalid formula: ${latex}</span>`;
  }
};

// Frontend
<div>
  {renderMathFormula(userInput)}
  <span className="hint">Preview formula here ↑</span>
</div>
```

**Cách tốt hơn:**

```javascript
// Validate trước khi lưu database
const isValidFormula = (latex) => {
  try {
    render(latex, { output: 'mathml', throwOnError: true });
    return true;
  } catch {
    return false;
  }
};

// Backend
app.post('/save-question', (req, res) => {
  const { mathFormula } = req.body;
  
  if (!isValidFormula(mathFormula)) {
    return res.status(400).json({ 
      error: 'Invalid math formula. Please check syntax.' 
    });
  }
  
  // Save to database
  saveQuestion(mathFormula);
});
```

**User Experience:**
```
User tạo question
  ↓
Input: \frac{1}{0}
  ↓
Preview hiển thị: "Invalid formula"
  ↓
User nhìn thấy lỗi ngay lập tức
  ↓
Fix công thức → Preview update
  ↓
Click Save (chỉ save nếu valid)
```

---

### **Phần 3: Câu Hỏi Về Real-time Proctoring & MediaPipe**

---

### **Q1.6: Bạn phát hiện 7 categories of cheating behavior qua camera (no face, multiple faces, head turning, tab switching, IP change). Làm sao để monitor tất cả cùng lúc mà không lag?**

**Mục đích câu hỏi:** Kiểm tra performance optimization, multi-task handling, resource management

**Câu trả lời mẫu:**

**Challenge:**
- Camera stream: 30 fps (30 frame/giây)
- Face detection: Nặng CPU
- Nếu check toàn bộ 30 frame/giây → lag

**Giải pháp: Sampling + Throttling**

```javascript
// Frontend: Không check mỗi frame, mà check mỗi 2 giây
let lastCheckTime = Date.now();
const CHECK_INTERVAL = 2000;  // 2 giây

video.onPlay = async () => {
  while (true) {
    const now = Date.now();
    
    if (now - lastCheckTime >= CHECK_INTERVAL) {
      // Chỉ check mỗi 2 giây
      const frame = captureFrame(video);
      const violations = await detectViolations(frame);
      
      if (violations.length > 0) {
        reportToServer(violations);
      }
      
      lastCheckTime = now;
    }
    
    await sleep(100);  // CPU break: check mỗi 100ms là đủ
  }
};

// Phân công cụ thể:
// - Tab switching: Dùng visibilitychange event (gần như miễn phí CPU)
// - IP change: Check mỗi 5 giây (mỗi heartbeat)
// - Face detection: Check mỗi 2 giây
```

**Chi tiết từng loại:**

| Loại Violation | Cách Detect | Tần Suất | CPU % |
|---|---|---|---|
| Tab switching | visibilitychange event | Real-time | 0% |
| IP change | HTTP request header | Mỗi heartbeat (5s) | 1% |
| Multiple faces | MediaPipe FaceMesh | Mỗi 2 giây | 20% |
| No face | MediaPipe FaceMesh | Mỗi 2 giây | 20% |
| Head turning | Face landmarks angle | Mỗi 2 giây | 20% |
| Window blur | onBlur event | Real-time | 0% |

**Kết quả:**
```
Total CPU: ~20% (hợp lý)
Không lag ✅
```

---

### **Q1.7: Nếu 100 students cùng làm bài và tất cả send violation events, server sẽ handle được không? Nếu không thì sao?**

**Mục đích câu hỏi:** Kiểm tra scalability thinking, load handling, queue management

**Câu trả lời mẫu:**

**Vấn đề:**
```
100 students × 1 violation/giây = 100 events/giây
Server xử lý violation → Save database → Send to teacher
Nếu xử lý sync (từng cái một) → lag, timeout
```

**Giải pháp:**

**1. Async Processing (không block request)**
```javascript
// Backend
app.post('/report-violation', async (req, res) => {
  const violation = req.body;
  
  // Không chờ xử lý xong, trả response ngay
  res.json({ success: true });
  
  // Xử lý async (background)
  processViolation(violation).catch(console.error);
});

async function processViolation(violation) {
  // 1. Save to database
  await db.violations.insert(violation);
  
  // 2. Send notification to teacher
  await notifyTeacher(violation);
  
  // 3. Update student status
  await updateStudentStatus(violation);
}
```

**2. Event Queue (BullMQ / RabbitMQ)**
```javascript
// Thay vì xử lý trực tiếp
const queue = new Queue('violations');

// Khi nhận violation
queue.add({
  studentId: '123',
  type: 'FACE_MISMATCH',
  timestamp: new Date()
});

// Queue tự động xử lý từng cái một
queue.process(async (job) => {
  await db.violations.insert(job.data);
  await notifyTeacher(job.data);
});
```

**3. Batch Processing (Gom 100 violations thành 1)**
```javascript
// Thay vì:
100 events → 100 database writes → 100 notifications

// Làm:
Buffer 100 events trong 1 giây
→ 1 database write (bulk insert)
→ 1 notification (1 message chứa 100 violations)
```

**Kết quả:**
```
Mà không batch:
  100 events → 100 DB writes = 100 * 50ms = 5 seconds

Với batch:
  100 events → 1 DB write = 50ms ✅ (100x faster!)
```

---

### **Phần 4: Câu Hỏi Về Redis Caching & Auto-save**

---

### **Q1.8: Bạn implement auto-save drafts to Redis mỗi 10 giây. Nếu Redis down, học sinh mất data không? Làm sao xử lý?**

**Mục đích câu hỏi:** Kiểm tra error handling, fallback strategy, data persistence

**Câu trả lời mẫu:**

**Scenario:**
```
Student làm bài
→ Trả lời 10 câu trong 1 phút
→ Auto-save to Redis every 10 seconds
→ Có 4 lần save thành công
→ Lần thứ 5, Redis bị down
→ Student mất data từ lần save thứ 4 đến giờ?
```

**Giải pháp: Dual Write Strategy**

```javascript
// Frontend
async function autoSave() {
  const answers = getStudentAnswers();
  
  // Lưu 2 nơi: Redis (nhanh) + Database (chậm nhưng safe)
  
  // 1. Lưu Redis (cache) - nhanh, 10 giây lưu 1 lần
  try {
    await saveToRedis(answers);
    console.log('Saved to Redis ✓');
  } catch (error) {
    console.warn('Redis failed, fallback to database');
    // Redis fail → fallback
  }
  
  // 2. Lưu Database - nhanh hơn redis (batched), 30 giây lưu 1 lần
  try {
    await saveToDatabaseAsync(answers);
    console.log('Saved to Database ✓');
  } catch (error) {
    console.error('Database save failed!');
    // Notify user: "Failed to save, check connection"
  }
}

// Backend
app.post('/auto-save', async (req, res) => {
  const { studentId, answers } = req.body;
  
  res.json({ success: true });  // Respond immediately
  
  // Async: Save both places
  Promise.all([
    redis.set(`draft:${studentId}`, answers),  // TTL: 2 hours
    db.drafts.insertOne({ studentId, answers, timestamp: new Date() })
  ]).catch(error => {
    console.error('Auto-save failed:', error);
    // Alert admin: Some students may lose data
  });
});
```

**Data Recovery Strategy:**

```javascript
// Khi student quay lại thi (sau khi dc/lag)
app.get('/resume-exam', async (req, res) => {
  const { studentId } = req.query;
  
  // 1. Cố gắng lấy từ Redis (mới nhất)
  let cachedAnswers = await redis.get(`draft:${studentId}`);
  
  if (cachedAnswers) {
    return res.json({ answers: cachedAnswers, source: 'Redis' });
  }
  
  // 2. Redis không có → lấy từ Database (lâu hơn chút nhưng đầy đủ)
  let dbAnswers = await db.drafts.findOne({ studentId }, { sort: { timestamp: -1 } });
  
  if (dbAnswers) {
    // Restore to Redis for next time
    await redis.set(`draft:${studentId}`, dbAnswers.answers);
    return res.json({ answers: dbAnswers.answers, source: 'Database' });
  }
  
  // 3. Cả Redis và Database không có → trả lại empty
  return res.json({ answers: {}, source: 'None' });
});
```

**Best Practice Checklist:**
- ✅ Never rely on single point of failure
- ✅ Use async/background processing
- ✅ Have fallback mechanism
- ✅ Log all failures for debugging
- ✅ Monitor Redis health

---

### **Q1.9: Redis là in-memory database. Nếu có 1000 students cùng làm bài, Redis memory không bị vượt quá không?**

**Mục đích câu hỏi:** Kiểm tra understanding of memory limits, TTL, eviction policies

**Câu trả lời mẫu:**

**Tính toán:**
```
1 student's answers ~= 10 KB (50 questions × 200 bytes/answer)
1000 students = 10 MB

Redis default: 128 MB (hoặc tùy config)
10 MB << 128 MB ✓ Không vượt
```

**Nhưng nếu cộng cache khác:**
```
Draft answers: 10 MB
User sessions: 5 MB
Real-time notifications: 2 MB
Rate limiting: 1 MB
Total: 18 MB (still okay)
```

**Nếu vượt quá, giải pháp:**

**1. Set TTL (Time To Live)**
```javascript
// Tự động xóa data sau 2 giờ (hết phiên thi)
await redis.setex(
  `draft:${studentId}`,
  2 * 3600,  // 2 hours in seconds
  answers
);

// Redis tự động xóa sau 2 giờ → memory tự giải phóng
```

**2. Eviction Policy**
```javascript
// config.redis
maxmemory: 256mb
maxmemory-policy: allkeys-lru  // Xóa key ít dùng nhất khi full

// Options:
// - allkeys-lru: Xóa key ít dùng nhất
// - allkeys-lfu: Xóa key ít được access nhất
// - volatile-lru: Chỉ xóa key có TTL
```

**3. Monitoring**
```javascript
// Check Redis memory mỗi phút
setInterval(async () => {
  const info = await redis.info('memory');
  console.log(`Used: ${info.used_memory_mb} MB`);
  
  if (info.used_memory_mb > 200) {  // Warning at 80%
    alert('Redis memory high!');
    // Kích hoạt emergency cleanup
  }
}, 60000);
```

**Kết luận:**
- ✅ Set TTL cho tất cả keys
- ✅ Monitor memory usage
- ✅ Configure eviction policy
- ✅ Fallback to database nếu cần

---

## **PROJECT 2: NovaWave - Music Streaming Platform**

### **Phần 1: Câu Hỏi Về WebSocket & Real-time Sync**

---

### **Q2.1: Bạn implement two-way waveform synchronization. Giải thích cách bạn sync progress bar và audio waveform in real time?**

**Mục đích câu hỏi:** Kiểm tra WebSocket understanding, state management, real-time data sync

**Câu trả lời mẫu:**

**Yêu cầu:**
```
User A: Drag progress bar
  ↓
→ Audio seek to new position
→ Waveform position cập nhật
→ Toàn bộ users trong room thấy thay đổi
```

**Kỹ thuật:**

**1. Tính toán vị trí từ progress bar**
```javascript
// Cách 1: Kéo progress bar
const progressBar = document.querySelector('.progress');

progressBar.addEventListener('mousedown', (e) => {
  const rect = progressBar.getBoundingClientRect();
  const x = e.clientX - rect.left;
  const percentage = x / rect.width;  // 0 to 1
  
  const newTime = percentage * audio.duration;  // second
  
  // Update audio
  audio.currentTime = newTime;
  
  // Send to server
  socket.emit('seek', { newTime });
});
```

**2. Cách 2: Drag waveform trực tiếp**
```javascript
// Nếu click vào waveform (Waveform.js library)
waveform.on('interaction', (percentage) => {
  const newTime = percentage * audio.duration;
  audio.currentTime = newTime;
  
  socket.emit('seek', { newTime, room: 'room123' });
});
```

**3. Backend: Broadcast đến room**
```javascript
// Socket.IO
io.on('connection', (socket) => {
  socket.on('seek', ({ newTime, room }) => {
    // Broadcast to everyone in room
    io.to(room).emit('user-seek', {
      userId: socket.id,
      newTime: newTime,
      timestamp: Date.now()
    });
  });
});
```

**4. Frontend: Nhận update từ server**
```javascript
// Người khác trong room
socket.on('user-seek', ({ newTime, userId }) => {
  if (userId !== myUserId) {
    // Update UI (progress bar, waveform)
    audio.currentTime = newTime;
    progressBar.style.left = `${(newTime / audio.duration) * 100}%`;
    waveform.setPosition(newTime);
  }
});
```

**Flow đầy đủ:**
```
User A seek to 1:30
  ↓
Frontend: audio.currentTime = 90
  ↓
Frontend: socket.emit('seek', { newTime: 90 })
  ↓
Backend: Nhận message
  ↓
Backend: io.to(room).emit('user-seek', { newTime: 90 })
  ↓
User B + User C: Nhận message
  ↓
Frontend User B: audio.currentTime = 90
  ↓
Frontend User C: audio.currentTime = 90
  ↓
Tất cả cùng position ✓
```

---

### **Q2.2: Nếu user A seek to 1:30 nhưng connection lag (latency 1 giây), sẽ xảy ra gì?**

**Mục đích câu hỏi:** Kiểm tra handling of network delay, optimistic updates, conflict resolution

**Câu trả lời mẫu:**

**Vấn đề:**
```
User A seek to 1:30 (9:55:00 client time)
  ↓ Send message
  ↓ Network delay 1 giây
  ↓ Backend nhận lúc 9:55:01
  ↓ Quay lại User B/C (delay thêm 1 giây)
  ↓ User B nhận lúc 9:55:02
  ↓ Nhưng audio của User B đã chạy thêm 2 giây
  ↓ Position bị lệch
```

**Giải pháp:**

**1. Optimistic Update (Client-side)**
```javascript
// User A:
// Cập nhật UI ngay lập tức (không chờ server)
audio.currentTime = 90;
progressBar.style.left = '50%';

// Gửi đến server
socket.emit('seek', { newTime: 90 });

// Nếu server reply lệch quá, correct lại
socket.on('seek-failed', ({ correctTime }) => {
  // Server nói sai, fix lại
  audio.currentTime = correctTime;
});
```

**2. Server Verification + Correction**
```javascript
// Backend
socket.on('seek', ({ newTime, timestamp }) => {
  // Kiểm tra: newTime có hợp lệ không?
  if (newTime < 0 || newTime > audio.duration) {
    socket.emit('seek-failed', { correctTime: audio.currentTime });
    return;
  }
  
  // Update server state
  roomState[room].currentTime = newTime;
  roomState[room].lastUpdateTime = Date.now();
  
  // Broadcast đến room
  io.to(room).emit('user-seek', {
    userId: socket.id,
    newTime: newTime,
    serverTime: Date.now()  // ← Timestamp từ server
  });
});
```

**3. Client Reconciliation**
```javascript
// Khi nhận update từ server
socket.on('user-seek', ({ newTime, serverTime }) => {
  const clientNow = Date.now();
  const lagMs = clientNow - serverTime;
  
  // Compensate for lag
  const adjustedTime = newTime + (lagMs / 1000);
  
  audio.currentTime = adjustedTime;
  waveform.setPosition(adjustedTime);
});
```

**Kết quả:**
```
Trước (không có fix):
  User A: 1:30
  User B: 1:28 (lệch 2 giây)

Sau (có compensation):
  User A: 1:30
  User B: 1:30 (sync ✓)
```

---

### **Q2.3: Shared listening room của bạn có play/pause sync. Nếu user A pause nhưng user B vẫn play, thì sao?**

**Mục đích câu hỏi:** Kiểm tra conflict resolution, state consistency, user experience

**Câu trả lời mẫu:**

**Vấn đề:**
```
User A (room owner): Click pause
User B: Vẫn đang playing (chưa nhận message)

Kết quả:
- A: Paused
- B: Playing
→ Not in sync ✗
```

**Giải pháp:**

**1. Room Owner Authority**
```javascript
// Backend: Chỉ room owner được control play/pause
socket.on('pause', ({ room }) => {
  if (roomState[room].owner !== socket.id) {
    socket.emit('error', 'Only owner can control playback');
    return;
  }
  
  // Update room state
  roomState[room].isPlaying = false;
  roomState[room].pausedAt = Date.now();
  
  // Broadcast
  io.to(room).emit('playback-changed', { isPlaying: false });
});

socket.on('resume', ({ room }) => {
  if (roomState[room].owner !== socket.id) {
    return;
  }
  
  roomState[room].isPlaying = true;
  io.to(room).emit('playback-changed', { isPlaying: true });
});
```

**2. Frontend: Reflect changes**
```javascript
// Tất cả users (A và B) listen on same event
socket.on('playback-changed', ({ isPlaying }) => {
  if (isPlaying) {
    audio.play();
  } else {
    audio.pause();
  }
});
```

**3. Fallback: Majority Vote (nếu không có owner)**
```javascript
// Nếu bao nhiêu % users pause → mọi người pause
const playingCount = room.users.filter(u => u.isPlaying).length;
const totalCount = room.users.length;

if (playingCount < totalCount / 2) {
  // Majority paused
  broadcastToAll({ action: 'pause' });
} else {
  // Majority playing
  broadcastToAll({ action: 'resume' });
}
```

**Best Practice:**
- ✅ Set 1 owner/leader để control
- ✅ Validate mỗi action từ backend
- ✅ Broadcast state changes
- ✅ Có fallback strategy

---

### **Q2.4: Song request feature. User B request 1 bài hát, nhưng User A không muốn. Xử lý như thế nào?**

**Mục đích câu hỏi:** Kiểm tra feature design, voting system, conflict resolution

**Câu trả lời mẫu:**

**Scenario:**
```
Room: 4 users (A, B, C, D)
- A: Room owner
- B: Request song "X"
- A, C, D: Không thích

Cần handle conflict này
```

**Giải pháp:**

**1. Queue + Voting System**
```javascript
// Database
{
  roomId: 'room123',
  queue: [
    {
      id: 'req1',
      songId: 'song-X',
      requestedBy: 'B',
      votes: { upvote: 1, downvote: 3 },  // A, C, D downvoted
      status: 'pending'  // hoặc 'approved', 'rejected'
    }
  ]
}

// Backend
socket.on('request-song', ({ songId, room }) => {
  const request = {
    id: generateId(),
    songId,
    requestedBy: socket.id,
    votes: { upvote: 1, downvote: 0 },
    createdAt: Date.now()
  };
  
  roomState[room].queue.push(request);
  
  // Broadcast
  io.to(room).emit('new-song-request', request);
});

socket.on('vote-request', ({ requestId, vote, room }) => {
  const request = roomState[room].queue.find(r => r.id === requestId);
  
  if (vote === 'up') {
    request.votes.upvote++;
  } else {
    request.votes.downvote++;
  }
  
  // Check if approved/rejected
  const totalVotes = request.votes.upvote + request.votes.downvote;
  if (request.votes.downvote > totalVotes / 2) {
    request.status = 'rejected';
  } else if (request.votes.upvote > totalVotes * 0.7) {
    request.status = 'approved';
    // Add to actual queue
  }
  
  io.to(room).emit('request-updated', request);
});
```

**2. UI cho User**
```
Song Requested by User B: "Blinding Lights"
[👍 1 upvote] [👎 3 downvotes]

Vote: [👍] [👎]  ← User click để vote

Status: [Rejected] (3 người không thích)
```

**3. Owner Override (Quyền đặc biệt)**
```javascript
// A (owner) có thể force approve/reject
socket.on('owner-override', ({ requestId, action, room }) => {
  if (roomState[room].owner !== socket.id) {
    socket.emit('error', 'Not authorized');
    return;
  }
  
  const request = roomState[room].queue.find(r => r.id === requestId);
  
  if (action === 'approve') {
    request.status = 'approved';
    // Thêm vào queue
  } else if (action === 'reject') {
    request.status = 'rejected';
  }
  
  io.to(room).emit('request-overridden', request);
});
```

**Kết quả:**
```
Democratic (voting):
- Dân chủ, công bằng
- Tốn thời gian, phức tạp

With owner override:
- Owner có quyền tối cuối
- Nhanh, quyết đoán
- Users có thể vote để suggest
```

---

## **PROJECT 3: Recalio - Spaced-Repetition Learning Platform**

### **Phần 1: Câu Hỏi Về Spaced Repetition Algorithm**

---

### **Q3.1: Bạn implement 2 configurable spaced-repetition algorithms (learning steps, leech threshold, retention rate). Khác nhau như thế nào?**

**Mục đích câu hỏi:** Kiểm tra algorithm knowledge, customization, scheduling logic

**Câu trả lời mẫu:**

**Spaced Repetition Concept:**
```
Quên: Exponential decay
  ↓
Cách giải: Repeat trước khi quên
  ↓
Interval: Day 1 → Day 3 → Day 7 → Day 21 → ...
```

**Algorithm 1: Fixed Intervals**
```
Config:
{
  intervals: [1, 3, 7, 21, 45],  // ngày
  easyBonus: 1.3x,  // Nếu easy → nhân 1.3
  hardPenalty: 0.5x  // Nếu hard → nhân 0.5
}

Process:
Card status: New
  ↓
User review & rate: Easy/Good/Hard
  ↓
If Easy:
  Next interval = 1 day × 1.3 = 1.3 days ✓
If Good:
  Next interval = 1 day ✓
If Hard:
  Next interval = 1 day × 0.5 = 0.5 days (reset)

Progression:
Day 1 → 1.3 → 4 → 5 → ... (cộng dồn)
```

**Algorithm 2: SM-2 (Supermemo 2)**
```
Config:
{
  startingEaseFactor: 2.5,
  minEaseFactor: 1.3,
  retentionRate: 0.9  // 90% cards retained
}

Formula:
EF = max(1.3, EF + (0.1 - (5 - q) * 0.08 - 0.02))
  where q = quality (0-5)

newInterval = previousInterval × EF

Process:
Card: "Apple" (New)
EF = 2.5
  ↓
User rate: 4/5 (Good)
EF = 2.5 + (0.1 - (5-4)*0.08 - 0.02) = 2.5 (unchanged)
Next interval = 1 × 2.5 = 2.5 days
  ↓
User rate: 5/5 (Perfect)
EF = 2.5 + (0.1 - 0 - 0.02) = 2.58
Next interval = 2.5 × 2.58 = 6.45 days
  ↓
User rate: 2/5 (Fail)
EF = 2.58 + (0.1 - 3*0.08 - 0.02) = 2.18
Next interval = 1 × 2.18 = 2.18 days (reset)
```

**Comparison:**

| Aspect | Fixed Intervals | SM-2 |
|---|---|---|
| **Complexity** | Đơn giản | Phức tạp |
| **Customization** | Config cứng | Dynamic |
| **Performance** | OK | Excellent |
| **User Control** | Cao | Thấp |

**Code Implementation:**
```javascript
class SpacedRepetition {
  constructor(algorithm) {
    this.algorithm = algorithm;  // 'fixed' hoặc 'sm2'
  }
  
  calculateNextInterval(card, quality) {
    if (this.algorithm === 'fixed') {
      return this.fixedInterval(card, quality);
    } else {
      return this.sm2Algorithm(card, quality);
    }
  }
  
  fixedInterval(card, quality) {
    const baseInterval = card.config.intervals[card.reps];
    
    if (quality === 'easy') {
      return baseInterval * 1.3;
    } else if (quality === 'good') {
      return baseInterval;
    } else {  // hard
      return baseInterval * 0.5;
    }
  }
  
  sm2Algorithm(card, quality) {
    const q = convertQualityToScore(quality);  // 0-5
    
    let ef = card.easeFactor;
    ef = Math.max(1.3, ef + (0.1 - (5 - q) * 0.08 - 0.02));
    
    const newInterval = card.interval * ef;
    
    // Update card
    card.easeFactor = ef;
    card.interval = newInterval;
    card.lastReview = new Date();
    
    return newInterval;
  }
}
```

**Database:**
```javascript
{
  _id: 'card123',
  deckId: 'deck1',
  front: 'Apple',
  back: 'Táo (quả mọc từ cây táo)',
  algorithm: 'sm2',  // hoặc 'fixed'
  
  // SM-2 specific
  easeFactor: 2.5,
  interval: 1,
  reps: 0,
  
  // Config
  config: {
    intervals: [1, 3, 7, 21],
    easyBonus: 1.3
  },
  
  // State
  lastReview: '2024-01-10',
  nextReview: '2024-01-13',
  dueDate: '2024-01-13'
}
```

---

### **Q3.2: Leech threshold là gì? Nếu user học 1 card mà fail liên tục 5 lần, bạn làm gì?**

**Mục đích câu hỏi:** Kiểm tra feature understanding, user support, problem detection

**Câu trả lời mẫu:**

**Leech = "vẫn quên" card**

```
Leech card: Card quá khó, user fail liên tục
Leech threshold: Số fail tối đa (e.g., 5 times)

Purpose:
- Detect problematic cards
- Alert user: "This card is too hard"
- Suggest: "Delete it? Or improve it?"
```

**Implementation:**

```javascript
{
  card: {
    _id: 'card123',
    front: 'Phát âm tiếng Anh của từ này?',
    back: 'pronunciation: [...]',
    
    // Track failures
    failCount: 5,  // Fail 5 times
    leechStatus: 'suspended',  // or 'normal'
    
    config: {
      leechThreshold: 5  // Suspend sau 5 fails
    }
  }
}

// When user rates 'hard'
socket.on('card-rated', ({ cardId, quality }) => {
  const card = await Card.findById(cardId);
  
  if (quality === 'hard') {
    card.failCount++;
    
    // Check if leech
    if (card.failCount >= card.config.leechThreshold) {
      card.leechStatus = 'suspended';
      
      // Notify user
      notifyUser(user, {
        type: 'leech-detected',
        message: `"${card.front}" is a leech card. Consider deleting or improving it.`,
        action: 'View suggestions'
      });
      
      // Remove from review queue
      await updateSchedule(user, card, 'suspended');
    }
  } else if (quality === 'easy') {
    // Reset fail count on success
    card.failCount = 0;
  }
  
  await card.save();
});
```

**UI for User:**
```
Card: "Phát âm của..."
Rating: [Again] [Hard] [Good] [Easy]

Fail count: ⚠️ 5/5 (LEECH!)

⚠️ This card is a leech
→ You've failed it 5 times
→ Options:
  [Delete Card]
  [Edit Card]
  [Hide for now]
```

**User Actions:**
```
Option 1: Delete
- Remove card entirely
- Move on

Option 2: Edit
- Improve question/answer
- Make easier
- Reset fail count

Option 3: Hide
- Suspend for 30 days
- Review later
```

---

### **Q3.3: 4 flashcard types (Basic, Reversed, Cloze, Image Occlusion). Nếu user tạo card Cloze, sau đó thay đổi thành Basic, dữ liệu cũ có bị mất không?**

**Mục đích câu hỏi:** Kiểm tra data migration, type safety, backward compatibility

**Câu trả lời mẫu:**

**4 Types:**

```javascript
// Type 1: Basic
{
  type: 'basic',
  front: 'Apple',
  back: 'Táo'
}

// Type 2: Basic Reversed
{
  type: 'basic-reversed',
  front: 'Táo',
  back: 'Apple'
}

// Type 3: Cloze
{
  type: 'cloze',
  content: 'The capital of France is {{Paris}}',
  // Blank part: {{...}}
}

// Type 4: Image Occlusion
{
  type: 'image-occlusion',
  imageUrl: 'anatomy-heart.jpg',
  occlusions: [
    { x: 100, y: 150, width: 50, height: 60, label: 'Ventricle' },
    { x: 200, y: 200, width: 40, height: 50, label: 'Atrium' }
  ]
}
```

**Problem: Type Conversion**

```
User tạo card Cloze:
{
  type: 'cloze',
  content: 'The capital of France is {{Paris}}'
}

Sau 1 tháng, user muốn đổi thành Basic
→ Thay type: 'basic'
→ Nhưng front/back fields trống!
→ Dữ liệu bị mất ✗
```

**Giải pháp:**

**1. Keep all fields (don't delete)**
```javascript
{
  _id: 'card123',
  type: 'cloze',  // Current type
  
  // All type fields (preserved)
  basic: {
    front: null,
    back: null
  },
  cloze: {
    content: 'The capital of France is {{Paris}}'
  },
  imageOcclusion: {
    imageUrl: null,
    occlusions: []
  },
  
  // Conversion history
  typeHistory: [
    { type: 'cloze', changedAt: '2024-01-10' }
  ]
}

// When converting
socket.on('change-card-type', ({ cardId, newType }) => {
  const card = await Card.findById(cardId);
  
  // Save old type data
  card.typeHistory.push({
    type: card.type,
    changedAt: new Date()
  });
  
  // Change type
  card.type = newType;
  
  // If new type has no data, initialize with old data
  if (newType === 'basic' && !card.basic.front) {
    card.basic.front = extractFromCloze(card.cloze.content);
    card.basic.back = 'Edit me';
  }
  
  await card.save();
});
```

**2. Atau: Backup previous version**
```javascript
{
  _id: 'card123',
  type: 'basic',
  front: 'Apple',
  back: 'Táo',
  
  // Backup old version
  backup: {
    version: 1,
    type: 'cloze',
    content: 'The capital of France is {{Paris}}',
    savedAt: '2024-01-10'
  },
  
  // Restore button
  canRestore: true
}

// Restore old version
socket.on('restore-card-version', ({ cardId, version }) => {
  const card = await Card.findById(cardId);
  
  const oldData = card.backup[version];
  
  // Swap
  card.backup.push({
    version: version + 1,
    type: card.type,
    front: card.front,
    back: card.back
  });
  
  card.type = oldData.type;
  Object.assign(card, oldData);
  
  await card.save();
});
```

**3. UI Confirmation**
```
⚠️ Warning: Changing card type

Current: Cloze
New: Basic

Data will be:
- Cloze content: [SAVED as backup]
- Basic front: "The capital of France is {{Paris}}"
- Basic back: [EMPTY - please edit]

[Cancel] [Change Type]
```

**Best Practice:**
- ✅ Never delete old data
- ✅ Provide backup/restore
- ✅ Show warning to user
- ✅ Auto-convert if possible

---

### **Q3.4: 3-tier text-to-speech pipeline (cache → Dictionary API → Google TTS fallback). Tại sao cần 3 tier?**

**Mục đích câu hỏi:** Kiểm tra system design, fallback strategy, cost optimization

**Câu trả lời mẫu:**

**Why 3 tiers?**

```
Tier 1 (Cache):
- Nhanh nhất (< 1ms)
- Miễn phí
- Nếu có sẵn → dùng

Tier 2 (Dictionary API):
- Trung bình (~500ms)
- Dành riêng cho pronunciation
- Chi phí thấp

Tier 3 (Google TTS):
- Chậm nhất (~2s)
- Chất lượng cao
- Chi phí cao

Strategy: Use cheapest, fallback to expensive
```

**Implementation:**

```javascript
async function getAudioForWord(word) {
  // TIER 1: Database cache
  const cached = await AudioCache.findOne({ word });
  if (cached) {
    console.log('Found in cache ✓');
    return cached.audioUrl;  // Cloudinary URL
  }
  
  // TIER 2: Dictionary API (free tier)
  try {
    const audio = await callDictionaryAPI(word);
    if (audio) {
      console.log('Found in Dictionary API ✓');
      // Save to cache for next time
      await AudioCache.create({ word, audioUrl: audio });
      return audio;
    }
  } catch (error) {
    console.warn('Dictionary API failed:', error);
    // Fall back to Tier 3
  }
  
  // TIER 3: Google TTS (premium fallback)
  try {
    console.log('Calling Google TTS...');
    const audioBuffer = await googleTTS.synthesize(word);
    const audioUrl = await uploadToCloudinary(audioBuffer);
    
    // Save to cache
    await AudioCache.create({ word, audioUrl });
    
    return audioUrl;
  } catch (error) {
    console.error('All TTS methods failed:', error);
    throw new Error('Could not generate audio');
  }
}
```

**Database Schema:**
```javascript
{
  collection: 'audio_cache',
  documents: [
    {
      _id: ObjectId(),
      word: 'apple',
      audioUrl: 'https://cloudinary.com/apple.mp3',
      source: 'dictionary-api',  // Track which tier
      cachedAt: Date(),
      ttl: 30 * 24 * 60 * 60  // 30 days
    }
  ]
}
```

**Cost Optimization:**

```
Without tiers:
100 words × 0.05$ (Google TTS) = $5

With 3 tiers:
- 60 words hit cache: $0
- 30 words from Dictionary API: $0.003
- 10 words from Google TTS: $0.50
Total: $0.50 (10x cheaper) ✓
```

**Monitoring:**

```javascript
// Track hit rates
const stats = {
  cacheHits: 1230,
  dictionaryHits: 456,
  ttsFallback: 89,
  total: 1775
};

const cacheHitRate = (1230 / 1775) * 100 = 69% ✓

// If cache hit rate drops → performance issue
if (cacheHitRate < 50%) {
  alert('High TTS fallback rate!');
}
```

**UI Indication:**
```
Word: "Apple"

Audio: 
[🔊 Pronunciation]
├─ Source: Cache ✓ (instant)
└─ Last updated: 2 days ago

[Refresh Audio]  ← Force regenerate
```

---

## **PROJECT 4: BingeBox - Cinema Ticket Booking**

### **Phần 1: Câu Hỏi Về Real-time Seat Selection & Concurrency**

---

### **Q4.1: Bạn implement real-time seat-selection với Socket.IO. Nếu user A và B cùng click seat số 5 trong cùng 1 giây, thì sao?**

**Mục đích câu hỏi:** Kiểm tra race condition, conflict resolution, locking mechanism

**Câu trả lời mẫu:**

**Race Condition:**

```
Time 0.0s: User A clicks seat 5
           → Emit: 'select-seat', { seatId: 5 }

Time 0.1s: User B clicks seat 5
           → Emit: 'select-seat', { seatId: 5 }

Server receive both messages casi cùng lúc
→ Cả A và B đều think họ booked seat 5?
```

**Solution 1: Pessimistic Locking (Server-side lock)**

```javascript
// Backend
const seatLocks = new Map();  // In-memory lock manager

socket.on('select-seat', async ({ seatId, showId }) => {
  const key = `${showId}:${seatId}`;
  
  // Check if locked
  if (seatLocks.has(key)) {
    socket.emit('seat-unavailable', { seatId });
    return;
  }
  
  // Lock seat (5 minutes)
  seatLocks.set(key, {
    userId: socket.id,
    lockedAt: Date.now(),
    expiryTime: Date.now() + 5 * 60 * 1000
  });
  
  socket.emit('seat-selected', { seatId, success: true });
  
  // Broadcast to others
  socket.broadcast.emit('seat-locked', { seatId, userId: socket.id });
  
  // Auto-unlock after 5 minutes (session timeout)
  setTimeout(() => {
    if (seatLocks.get(key)?.userId === socket.id) {
      seatLocks.delete(key);
      socket.broadcast.emit('seat-unlocked', { seatId });
    }
  }, 5 * 60 * 1000);
});

// When booking confirmed
socket.on('confirm-booking', async ({ selectedSeats, showId }) => {
  try {
    // Save to database
    await Booking.create({
      userId: socket.id,
      showId,
      seats: selectedSeats,
      status: 'confirmed'
    });
    
    // Release locks
    selectedSeats.forEach(seatId => {
      seatLocks.delete(`${showId}:${seatId}`);
    });
    
    socket.emit('booking-confirmed');
  } catch (error) {
    socket.emit('booking-failed', { error });
  }
});
```

**Timeline:**

```
Time 0.0s:
  A: Click seat 5
  → Server lock seat 5 for A
  → Send: "seat-selected" to A ✓
  → Broadcast: "seat-locked" to others (include B)

Time 0.1s:
  B: Click seat 5
  → Server check lock: Seat 5 locked by A ✗
  → Send: "seat-unavailable" to B ✗
  → B sees: "This seat is being selected" ⏳

Time 2s:
  A: Click confirm booking
  → Save to database
  → Release lock
  → Broadcast: "seat-unlocked"
  → B now can select seat 5 ✓
```

**Solution 2: Optimistic Locking (Database-level)**

```javascript
// Database
{
  _id: 'seat-5-show-123',
  seatId: 5,
  showId: 'show-123',
  status: 'available',  // or 'locked', 'booked'
  version: 1,
  
  // Booking info
  bookingId: null,
  bookedBy: null,
  bookedAt: null
}

// Backend: Use transaction
async function selectSeat(userId, seatId, showId) {
  const session = await db.startSession();
  
  try {
    await session.withTransaction(async () => {
      // Read seat with lock
      const seat = await Seat.findOne(
        { seatId, showId },
        { session }
      ).exec();
      
      // Check if available
      if (seat.status !== 'available') {
        throw new Error('Seat not available');
      }
      
      // Update seat
      const result = await Seat.findOneAndUpdate(
        { seatId, showId, status: 'available' },  // ← Only if still available
        { status: 'locked', bookedBy: userId, version: seat.version + 1 },
        { session, new: true }
      ).exec();
      
      if (!result) {
        throw new Error('Seat was booked by someone else');
      }
      
      return result;
    });
    
    return { success: true };
  } catch (error) {
    return { success: false, error: error.message };
  } finally {
    await session.endSession();
  }
}
```

**Best Practice:**

```
For real-time (Socket.IO):
✅ Use Pessimistic locking (in-memory Map)
  - Simple, fast
  - Good for short hold time

For traditional booking:
✅ Use Optimistic locking (database transaction)
  - Atomic, reliable
  - Good for long-running operations

For multi-server setup:
✅ Use Redis lock
  - Distributed
  - Cross-server safe
```

---

### **Q4.2: Seat hold time là bao lâu (5 phút) trước khi unlock? Nếu user quên cancel, seat bị lãng phí không?**

**Mục đích câu hỏi:** Kiểm tra timeout strategy, UX consideration, waste prevention

**Câu trả lời mẫu:**

**Problem:**
```
User A: Select seats → Hold for 5 minutes
User A: Go AFK (forgot to booking)
→ Seat locked for 5 min × 100 users = 500 minutes = 8 hours
→ Revenue lost
```

**Solution: Multi-stage timeout**

```javascript
// Stage 1: Hold timeout (5 minutes)
const HOLD_TIME = 5 * 60 * 1000;

socket.on('select-seat', ({ seatId }) => {
  const lock = {
    userId: socket.id,
    status: 'held',
    heldAt: Date.now(),
    warnings: 0
  };
  
  seatLocks.set(seatId, lock);
  
  // Set timeout for release
  setTimeout(() => {
    releaseSeat(seatId, 'timeout');
  }, HOLD_TIME);
});

// Stage 2: Warning before timeout (1 minute before)
setTimeout(() => {
  socket.emit('seat-hold-warning', {
    message: 'Your seat will be released in 1 minute',
    seatId,
    action: 'Click to extend hold'
  });
}, (HOLD_TIME - 60000));  // Warn at 4 minutes
```

**UI for User:**

```
Selected Seats: [5] [6]

⏱️ Hold expires in: 4:35
[Extend Hold] [Cancel Selection]

Booking Summary:
- Seats: 5, 6
- Price: $30
[Proceed to Payment]
```

**Smart Release:**

```javascript
// Don't just release after 5 min
// Release gradually based on peak times

function releaseStrategy(seatId, time) {
  const now = new Date();
  const hour = now.getHours();
  
  // Peak time (18:00 - 22:00): Release after 3 min
  if (hour >= 18 && hour <= 22) {
    return 3 * 60 * 1000;
  }
  
  // Normal time: Release after 5 min
  return 5 * 60 * 1000;
}
```

**Analytics & Monitoring:**

```javascript
// Track seat waste
const stats = {
  seatsHeld: 150,
  seatsReleased: 120,  // Timeout
  seatsBooked: 30,
  releaseRate: (120 / 150) * 100 = 80%  // 80% wasted
};

// If too high → reduce hold time
if (releaseRate > 70%) {
  HOLD_TIME = 3 * 60 * 1000;  // Reduce to 3 min
}
```

**Best Practice:**
- ✅ Set reasonable timeout (3-5 min)
- ✅ Warn before release
- ✅ Allow extend
- ✅ Monitor waste rate
- ✅ Adjust dynamically

---

### **Q4.3: Bạn implement multi-factor ticket pricing (seat type, room format, showtime, day, age). Nếu user là senior (60+) + book 2D movie + weekend, logic tính giá như thế nào?**

**Mục đích câu hỏi:** Kiểm tra complex business logic, discount combination, calculation accuracy

**Câu trả lời mẫu:**

**Pricing Factors:**

```
Base price: $10

Seat type multiplier:
- Regular: 1.0x
- VIP: 1.5x
- Couple: 1.4x

Format multiplier:
- 2D: 1.0x
- 3D: 1.3x
- IMAX: 1.5x

Showtime multiplier:
- Morning (before 12): 0.8x
- Afternoon (12-17): 1.0x
- Evening (17-21): 1.3x
- Late (after 21): 1.1x

Day multiplier:
- Weekday (Mon-Fri): 0.9x
- Weekend (Sat-Sun): 1.2x

Age discount:
- Senior (60+): 0.7x (30% discount)
- Adult (18-59): 1.0x
- Student (with ID): 0.85x
- Child (5-12): 0.6x
```

**Calculation Example:**

```
Senior (60+), 2D movie, Weekend, Evening slot
Seat: Regular

Price = Base × Seat × Format × Showtime × Day × Age
       = 10 × 1.0 × 1.0 × 1.3 × 1.2 × 0.7
       = 10.92$

Breakdown:
- Base: $10.00
- 2D format: ×1.0 = $10.00
- Weekend: ×1.2 = $12.00
- Evening: ×1.3 = $15.60
- Senior discount: ×0.7 = $10.92 ✓
```

**Code Implementation:**

```javascript
class TicketPricing {
  constructor() {
    this.basePrice = 10;
    this.factors = {
      seatType: {
        regular: 1.0,
        vip: 1.5,
        couple: 1.4
      },
      format: {
        '2d': 1.0,
        '3d': 1.3,
        'imax': 1.5
      },
      showtime: {
        morning: 0.8,    // 00:00-12:00
        afternoon: 1.0,  // 12:00-17:00
        evening: 1.3,    // 17:00-21:00
        late: 1.1        // 21:00-00:00
      },
      day: {
        weekday: 0.9,
        weekend: 1.2
      },
      age: {
        senior: 0.7,     // 60+
        adult: 1.0,      // 18-59
        student: 0.85,   // with ID
        child: 0.6       // 5-12
      }
    };
  }
  
  calculatePrice(booking) {
    const {
      seatType,
      format,
      showtime,
      date,
      customerAge
    } = booking;
    
    // Get multipliers
    const seatFactor = this.factors.seatType[seatType];
    const formatFactor = this.factors.format[format];
    const timeFactor = this.factors.showtime[this.getShowtimeType(showtime)];
    const dayFactor = this.factors.day[this.getDayType(date)];
    const ageFactor = this.factors.age[this.getAgeGroup(customerAge)];
    
    // Calculate
    const finalPrice = 
      this.basePrice × seatFactor × formatFactor × timeFactor × dayFactor × ageFactor;
    
    return {
      basePrice: this.basePrice,
      multipliers: {
        seat: seatFactor,
        format: formatFactor,
        time: timeFactor,
        day: dayFactor,
        age: ageFactor
      },
      finalPrice: Math.round(finalPrice * 100) / 100,  // Round to 2 decimals
      breakdown: this.generateBreakdown(booking, finalPrice)
    };
  }
  
  getShowtimeType(hours) {
    if (hours < 12) return 'morning';
    if (hours < 17) return 'afternoon';
    if (hours < 21) return 'evening';
    return 'late';
  }
  
  getDayType(date) {
    const day = date.getDay();
    return (day === 0 || day === 6) ? 'weekend' : 'weekday';
  }
  
  getAgeGroup(age) {
    if (age >= 60) return 'senior';
    if (age >= 18) return 'adult';
    if (age >= 13) return 'student';
    if (age >= 5) return 'child';
    return null;  // Too young
  }
  
  generateBreakdown(booking, finalPrice) {
    return {
      seatType: `${booking.seatType} seat`,
      format: booking.format,
      showtime: booking.showtime,
      date: booking.date,
      ageGroup: this.getAgeGroup(booking.customerAge),
      totalPrice: finalPrice
    };
  }
}

// Usage
const pricing = new TicketPricing();

const booking = {
  seatType: 'regular',
  format: '2d',
  showtime: 19,  // 7 PM
  date: new Date('2024-01-20'),  // Saturday
  customerAge: 65  // Senior
};

const result = pricing.calculatePrice(booking);
console.log(result);
// Output:
// {
//   basePrice: 10,
//   multipliers: { seat: 1.0, format: 1.0, time: 1.3, day: 1.2, age: 0.7 },
//   finalPrice: 10.92,
//   breakdown: { seatType: 'regular seat', ... }
// }
```

**Database:**

```javascript
{
  ticketId: 'ticket-123',
  showId: 'show-456',
  seatId: 5,
  
  // Pricing info
  basePrice: 10,
  discount: {
    ageDiscount: 'senior',
    discountPercent: 30
  },
  
  // Final
  totalPrice: 10.92,
  
  // Audit
  pricingVersion: 'v2.1',  // In case we change pricing
  calculatedAt: Date(),
  
  // Payment
  paymentStatus: 'pending',
  paymentMethod: 'credit_card'
}
```

---

### **Q4.4: Bạn integrate SePay payment + HMAC-SHA256 webhook verification. Nếu webhook từ SePay bị delay 10 phút, booking status sẽ là gì?**

**Mục đích câu hỏi:** Kiểm tra payment flow, webhook handling, eventual consistency

**Câu trả lời mẫu:**

**Payment Flow:**

```
Time 0s:
  User click "Pay"
  → Frontend gọi Backend API: POST /checkout
  → Backend create order & redirect to SePay

Time 5s:
  User hoàn tất payment on SePay
  → SePay return redirect (success) to our site
  → Frontend update UI: "Payment successful"
  → But backend status vẫn "pending" ✗

Time 10-15 min:
  SePay webhook bị delay
  → Gọi: POST /webhook/sepay
  → Backend verify signature & update status to "completed"
  → Database status: "pending" → "completed"
```

**Problem:**

```
User: "Payment successful on SePay"
Database: Status = "pending"
Result: Booking incomplete ✗
```

**Solution 1: Polling (Ask SePay)**

```javascript
// Frontend
async function checkPaymentStatus(orderId, attempts = 0) {
  const response = await fetch(`/api/payment/status/${orderId}`);
  const { status } = await response.json();
  
  if (status === 'completed') {
    showBookingConfirmation();
  } else if (attempts < 12) {
    // Retry every 5 seconds, max 12 times (60 seconds)
    setTimeout(() => checkPaymentStatus(orderId, attempts + 1), 5000);
  } else {
    // Timeout - ask user to check later
    showMessage('Payment status unclear. Please check later.');
  }
}
```

**Solution 2: Webhook Retry (SePay side)**

```javascript
// Backend: Receive webhook
app.post('/webhook/sepay', (req, res) => {
  const { orderId, transactionId, status, signature } = req.body;
  
  // Verify HMAC-SHA256
  const expectedSignature = generateHMAC({
    orderId,
    transactionId,
    status
  });
  
  if (signature !== expectedSignature) {
    res.status(401).json({ error: 'Invalid signature' });
    return;
  }
  
  // Update order status
  Order.findByIdAndUpdate(
    orderId,
    { paymentStatus: status }
  );
  
  // Respond immediately (so SePay doesn't retry)
  res.json({ received: true });
  
  // Do async work after response
  notifyUser(orderId, status);
  updateInventory(orderId);
});
```

**Solution 3: Dual-check (Both polling + webhook)**

```javascript
// Frontend: Aggressive polling initially
async function checkPaymentStatus(orderId) {
  const checkPayment = async () => {
    const response = await fetch(`/api/payment/status/${orderId}`);
    const { status, verifiedByWebhook } = await response.json();
    
    if (status === 'completed' && verifiedByWebhook) {
      // Confirmed by webhook ✓
      showBookingConfirmation();
      clearInterval(polling);
    } else if (status === 'completed' && !verifiedByWebhook) {
      // Only from redirect, not webhook yet
      showMessage('Payment detected, finalizing...');
    }
  };
  
  // Poll every 5 sec
  const polling = setInterval(checkPayment, 5000);
  
  // Stop after 2 minutes
  setTimeout(() => clearInterval(polling), 120000);
}
```

**Database Status:**

```javascript
{
  orderId: 'order-123',
  
  // Payment tracking
  paymentStatus: 'pending',  // pending | completed | failed
  
  // Verification
  verification: {
    redirectConfirmed: true,   // User returned from SePay
    webhookConfirmed: false,   // Webhook received
    transactionId: 'txn-456',
    
    // Timestamps
    redirectTime: '2024-01-20 19:00:05',
    webhookTime: null,  // Pending webhook
    
    // Reconciliation
    needsReconciliation: false
  },
  
  // Booking status
  bookingStatus: 'reserved',  // reserved | confirmed | cancelled
  // "reserved" = payment likely ok, waiting webhook confirmation
}
```

**HMAC Verification Code:**

```javascript
const crypto = require('crypto');

function generateHMAC(data) {
  const secretKey = process.env.SEPAY_SECRET;
  const dataString = JSON.stringify(data);
  
  return crypto
    .createHmac('sha256', secretKey)
    .update(dataString)
    .digest('hex');
}

// Verify incoming webhook
function verifyWebhookSignature(req) {
  const { orderId, transactionId, status, signature } = req.body;
  
  const expectedSig = generateHMAC({
    orderId,
    transactionId,
    status
  });
  
  // Constant-time comparison (prevent timing attacks)
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expectedSig)
  );
}
```

**Best Practice:**
- ✅ Verify HMAC signature (security)
- ✅ Implement polling as fallback
- ✅ Set timeout for async operations
- ✅ Log all payment events
- ✅ Reconciliation job để handle delayed webhooks

---

### **Q4.5: Redis caching for movie listings, showtimes, dashboard stats (2-15 min TTL). Nếu admin update movie info, cache cũ vẫn hiển thị. Cách invalidate cache là gì?**

**Mục đích câu hỏi:** Kiểm tra cache invalidation strategy, consistency, admin operations

**Câu trả lời mẫu:**

**Problem:**

```
Time 0s: Admin update movie: "Moana" → Rating 9.8 ⭐

Backend:
1. Update database (✓)
2. Redis still has old data (Rating 9.5) ✗

Time 5s: User sees:
- Website (DB): Rating 9.8
- Mobile app (cache): Rating 9.5 ✗
```

**Solution: Cache Invalidation Strategies**

**1. TTL (Time To Live) - Passive**

```javascript
// Automatic expire after TTL
const CACHE_TTL = {
  movies: 5 * 60,      // 5 min
  showtimes: 2 * 60,   // 2 min
  stats: 15 * 60       // 15 min
};

// Get movies
async function getMovies() {
  const cached = await redis.get('movies:list');
  if (cached) {
    return JSON.parse(cached);
  }
  
  const movies = await db.movies.find();
  // Auto expire after 5 minutes
  await redis.setex('movies:list', CACHE_TTL.movies, JSON.stringify(movies));
  return movies;
}
```

**2. Manual Invalidation - Active (on update)**

```javascript
// When admin update movie
app.put('/admin/movie/:id', async (req, res) => {
  const { id } = req.params;
  const update = req.body;
  
  // 1. Update database
  const movie = await db.movies.findByIdAndUpdate(id, update);
  
  // 2. Invalidate related caches
  await redis.delete(`movie:${id}`);         // Single movie
  await redis.delete('movies:list');         // All movies list
  await redis.delete('dashboard:stats');     // Stats affected
  await redis.delete('featured:movies');     // Featured list
  
  // 3. Notify clients
  socket.io.emit('movie-updated', { movieId: id, changes: update });
  
  res.json({ success: true });
});
```

**3. Event-driven Invalidation (Pub/Sub)**

```javascript
// Using Redis Pub/Sub
const redis = require('redis');
const publisher = redis.createClient();
const subscriber = redis.createClient();

// When updating
app.put('/admin/movie/:id', async (req, res) => {
  const { id } = req.params;
  const movie = await db.movies.findByIdAndUpdate(id, req.body);
  
  // Publish event
  publisher.publish('movie-updates', JSON.stringify({
    eventType: 'MOVIE_UPDATED',
    movieId: id
  }));
  
  res.json({ success: true });
});

// Subscribe & invalidate
subscriber.subscribe('movie-updates', (message) => {
  const event = JSON.parse(message);
  
  if (event.eventType === 'MOVIE_UPDATED') {
    // Invalidate all related caches
    redis.delete(`movie:${event.movieId}`);
    redis.delete('movies:list');
    redis.delete('dashboard:stats');
  }
});
```

**4. Versioning Strategy**

```javascript
// Track cache version
{
  _id: 'cache-version',
  moviesVersion: 1,
  showtimesVersion: 3,
  statsVersion: 5
}

// When updating
app.put('/admin/movie/:id', async (req, res) => {
  const movie = await db.movies.findByIdAndUpdate(id, req.body);
  
  // Increment version
  await db.cacheVersion.updateOne(
    { _id: 'cache-version' },
    { $inc: { moviesVersion: 1 } }  // Increment
  );
  
  // Publish
  publisher.publish('cache-version', 'MOVIES_CHANGED');
  
  res.json({ success: true });
});

// Frontend
async function getMovies() {
  const version = await getMoviesCacheVersion();  // Current: v1
  
  const cached = await redis.get(`movies:list:v${version}`);
  if (cached) return cached;
  
  // Fetch & cache with version
  const movies = await db.movies.find();
  await redis.setex(`movies:list:v${version}`, TTL, JSON.stringify(movies));
  return movies;
}
```

**Best Practice:** Combine multiple strategies

```javascript
class CacheManager {
  async invalidateMovie(movieId) {
    // 1. Delete specific movie
    await redis.delete(`movie:${movieId}`);
    
    // 2. Delete collections
    await redis.delete('movies:list');
    
    // 3. Publish event (for distributed systems)
    this.publish('movie-updated', { movieId });
    
    // 4. Increment version (for versioning)
    await this.incrementVersion('movies');
    
    // 5. Notify clients WebSocket
    socket.io.emit('cache-invalidated', { type: 'movies' });
  }
  
  // Add to watch list
  watchMovie(movieId) {
    // If edited in next 30s, invalidate
    this.watchedMovies.add(movieId);
    setTimeout(() => this.watchedMovies.delete(movieId), 30000);
  }
}

// Usage
cacheManager.invalidateMovie('movie-123');
// → Immediately invalidate all related caches
// → All clients get fresh data
```

**Monitoring:**

```javascript
// Track cache hit/miss rate
const stats = {
  hits: 1250,
  misses: 450,
  invalidations: 12
};

const hitRate = (1250 / (1250 + 450)) * 100 = 73%

if (hitRate < 50%) {
  alert('Cache hit rate low! Check invalidation strategy.');
}
```

---

## **INTERVIEW TIP**

✅ **Khi trả lời:**
1. Giải thích vấn đề trước
2. Đề xuất giải pháp
3. Thảo luận trade-offs
4. Mention monitoring/alerts
5. Ask clarifying questions

✅ **Red flags to avoid:**
- ❌ Chỉ nói một giải pháp
- ❌ Không xem xét edge cases
- ❌ Không mention failure scenarios
- ❌ Không có monitoring/logging

✅ **Show your thinking:**
- "So the issue is..."
- "One approach could be..."
- "But that has a tradeoff..."
- "A better solution might be..."

---

**Good luck với phỏng vấn! 🎯**

