# BullMQ – Tổng hợp kiến thức đầy đủ

---

## 1. BullMQ là gì?

BullMQ là **message queue library** cho Node.js, dựa trên Redis. Nó cho phép xử lý các **background jobs** một cách đáng tin cậy, có thể scale và retry khi thất bại.

**Tại sao dùng BullMQ thay vì Bull?**
- BullMQ là phiên bản viết lại hoàn toàn của Bull (TypeScript-first)
- Hỗ trợ **Job Dependencies** (job chạy sau khi job khác xong)
- Hỗ trợ **Flows** (cây job phức tạp)
- API rõ ràng, type-safe hơn

**Các thành phần cốt lõi:**
```
Producer → Queue → Worker → (Events)
```

---

## 2. Kiến trúc tổng quan

```
┌─────────────┐     add job     ┌──────────────┐     process     ┌──────────────┐
│  Producer   │ ─────────────→  │    Queue     │ ──────────────→ │    Worker    │
│  (Service)  │                 │   (Redis)    │                 │  (Processor) │
└─────────────┘                 └──────────────┘                 └──────────────┘
                                       │                                │
                                       ▼                                ▼
                                ┌──────────────┐              ┌──────────────────┐
                                │  QueueEvents │              │  Events/Hooks    │
                                │  (Monitor)   │              │  completed/failed│
                                └──────────────┘              └──────────────────┘
```

**Vòng đời của một Job:**
```
waiting → active → completed
                ↘ failed → (retry) → waiting
                         → delayed
```

---

## 3. Cài đặt

```bash
npm install bullmq ioredis

# Với NestJS
npm install @nestjs/bullmq bullmq ioredis
```

---

## 4. Queue – Hàng đợi

Queue là nơi **nhận và lưu trữ job** trên Redis.

```typescript
import { Queue } from 'bullmq';

const queue = new Queue('email', {
  connection: { host: 'localhost', port: 6379 },
  defaultJobOptions: {
    attempts: 3,               // retry tối đa 3 lần
    backoff: { type: 'exponential', delay: 1000 },
    removeOnComplete: { count: 100 },  // chỉ giữ 100 job completed
    removeOnFail: { count: 200 },
  },
});
```

### Thêm job vào queue

```typescript
// Job đơn giản
await queue.add('send-welcome', { userId: '123', email: 'a@b.com' });

// Job với options
await queue.add('send-report', { reportId: 'R001' }, {
  delay: 5000,           // chạy sau 5 giây
  priority: 1,           // priority cao hơn (số nhỏ = ưu tiên cao hơn)
  jobId: 'report-R001',  // custom ID (idempotent)
  attempts: 5,
  backoff: { type: 'fixed', delay: 2000 },
});

// Thêm nhiều job cùng lúc (bulk)
await queue.addBulk([
  { name: 'send-email', data: { to: 'a@b.com' } },
  { name: 'send-email', data: { to: 'c@d.com' } },
  { name: 'send-sms',   data: { phone: '09xx' } },
]);
```

---

## 5. Worker – Bộ xử lý

Worker **lắng nghe queue** và xử lý job.

```typescript
import { Worker, Job } from 'bullmq';

const worker = new Worker(
  'email',
  async (job: Job) => {
    // Hàm processor – return value = job result
    switch (job.name) {
      case 'send-welcome':
        await sendWelcomeEmail(job.data);
        break;
      case 'send-report':
        const result = await generateReport(job.data.reportId);
        return result; // lưu vào job.returnvalue
    }
  },
  {
    connection: { host: 'localhost', port: 6379 },
    concurrency: 5,        // xử lý tối đa 5 job song song
    limiter: {
      max: 100,            // tối đa 100 job
      duration: 60_000,    // mỗi 60 giây (rate limit)
    },
  }
);
```

### Worker events

```typescript
worker.on('completed', (job, result) => {
  console.log(`Job ${job.id} done:`, result);
});

worker.on('failed', (job, error) => {
  console.error(`Job ${job.id} failed:`, error.message);
});

worker.on('progress', (job, progress) => {
  console.log(`Job ${job.id} progress: ${progress}%`);
});

worker.on('stalled', (jobId) => {
  console.warn(`Job ${jobId} stalled`); // worker bị crash giữa chừng
});

// Graceful shutdown
process.on('SIGTERM', async () => {
  await worker.close();
});
```

---

## 6. Job Options chi tiết

```typescript
await queue.add('task', data, {
  // Retry
  attempts: 3,
  backoff: {
    type: 'exponential',  // 'fixed' | 'exponential' | 'custom'
    delay: 1000,          // delay đầu tiên (ms)
    // exponential: 1s, 2s, 4s, 8s...
  },

  // Delay & Scheduling
  delay: 10_000,          // chạy sau 10 giây
  repeat: {               // cron job
    pattern: '0 9 * * MON-FRI',  // 9h sáng các ngày thường
    // hoặc
    every: 60_000,        // mỗi 60 giây
    limit: 10,            // chạy tối đa 10 lần
  },

  // Priority (1 = cao nhất)
  priority: 1,

  // Deduplication
  jobId: 'unique-id',     // nếu đã tồn tại jobId này → skip

  // Cleanup
  removeOnComplete: true,           // xóa ngay khi done
  removeOnComplete: { count: 100 }, // giữ 100 cái gần nhất
  removeOnComplete: { age: 3600 },  // xóa sau 1h
  removeOnFail: false,              // giữ lại job fail để debug

  // Timeout
  timeout: 30_000,        // job bị kill nếu chạy quá 30s (BullMQ Pro)
});
```

---

## 7. Cron Jobs (Repeatable Jobs)

```typescript
// Thêm cron job
await queue.add('daily-report', {}, {
  repeat: { pattern: '0 8 * * *' }, // mỗi ngày lúc 8h
});

// Lấy danh sách repeatable jobs
const repeatableJobs = await queue.getRepeatableJobs();

// Xóa cron job
await queue.removeRepeatable('daily-report', { pattern: '0 8 * * *' });
// hoặc
await queue.removeRepeatableByKey(job.repeatJobKey);
```

---

## 8. Job Progress & Logging

```typescript
// Trong processor – cập nhật progress
const worker = new Worker('upload', async (job) => {
  const files = job.data.files;

  for (let i = 0; i < files.length; i++) {
    await uploadFile(files[i]);

    // Cập nhật % tiến trình
    await job.updateProgress(Math.round((i + 1) / files.length * 100));

    // Hoặc gửi object
    await job.updateProgress({ current: i + 1, total: files.length });

    // Log thêm dữ liệu vào job
    await job.log(`Uploaded: ${files[i].name}`);
  }
});
```

---

## 9. Job Dependencies

Job B chỉ chạy **sau khi Job A hoàn thành**.

```typescript
const parentJob = await queue.add('process-order', { orderId: '001' });

// Job phụ thuộc vào parent
await queue.add('send-confirmation', { orderId: '001' }, {
  parent: {
    id: parentJob.id,
    queue: 'email',     // queue chứa parent (nếu khác queue)
  },
});
```

---

## 10. Flows (Cây job phức tạp)

Flow cho phép tạo **cây job có quan hệ cha-con**.

```typescript
import { FlowProducer } from 'bullmq';

const flowProducer = new FlowProducer({ connection });

// Con chạy trước → Cha chạy sau khi tất cả con done
const flow = await flowProducer.add({
  name: 'send-invoice',
  queueName: 'notification',
  data: { orderId: '001' },
  children: [
    {
      name: 'generate-pdf',
      queueName: 'pdf',
      data: { orderId: '001' },
      children: [
        { name: 'fetch-order-data', queueName: 'data', data: { orderId: '001' } },
        { name: 'fetch-user-data',  queueName: 'data', data: { userId: 'U1' } },
      ],
    },
    {
      name: 'calculate-tax',
      queueName: 'accounting',
      data: { orderId: '001' },
    },
  ],
});
```

**Luồng thực thi:**
```
fetch-order-data ─┐
                  ├→ generate-pdf ─┐
fetch-user-data  ─┘                ├→ send-invoice
calculate-tax ─────────────────────┘
```

---

## 11. QueueEvents – Monitor

```typescript
import { QueueEvents } from 'bullmq';

const queueEvents = new QueueEvents('email', { connection });

queueEvents.on('completed', ({ jobId, returnvalue }) => {
  console.log(`Job ${jobId} completed:`, returnvalue);
});

queueEvents.on('failed', ({ jobId, failedReason }) => {
  console.error(`Job ${jobId} failed:`, failedReason);
});

queueEvents.on('delayed', ({ jobId, delay }) => {
  console.log(`Job ${jobId} delayed ${delay}ms`);
});

queueEvents.on('progress', ({ jobId, data }) => {
  console.log(`Job ${jobId} progress:`, data);
});

// Chờ job hoàn thành (dùng trong integration test / sync flow)
const job = await queue.add('task', data);
const result = await job.waitUntilFinished(queueEvents, 30_000); // timeout 30s
```

---

## 12. Sandbox Processor (Child Process)

Chạy processor trong **process riêng biệt** — tránh crash làm chết toàn bộ app.

```typescript
// worker.ts
const worker = new Worker('heavy-task', path.resolve('./processor.js'), {
  connection,
  concurrency: 3,
});

// processor.js (file riêng)
module.exports = async (job) => {
  // chạy trong child process riêng
  return await heavyCpuTask(job.data);
};
```

---

## 13. BullMQ trong NestJS (@nestjs/bullmq)

### Setup

```typescript
// app.module.ts
import { BullModule } from '@nestjs/bullmq';

@Module({
  imports: [
    BullModule.forRoot({
      connection: { host: 'localhost', port: 6379 },
      defaultJobOptions: {
        attempts: 3,
        backoff: { type: 'exponential', delay: 1000 },
        removeOnComplete: { count: 1000 },
        removeOnFail: { count: 5000 },
      },
    }),
    BullModule.registerQueue(
      { name: 'email' },
      { name: 'notification' },
      { name: 'report' },
    ),
  ],
})
export class AppModule {}
```

### Producer (Service)

```typescript
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';

@Injectable()
export class EmailService {
  constructor(@InjectQueue('email') private emailQueue: Queue) {}

  async sendWelcome(userId: string) {
    await this.emailQueue.add('welcome', { userId }, { delay: 0 });
  }

  async scheduleWeeklyDigest() {
    await this.emailQueue.add('weekly-digest', {}, {
      repeat: { pattern: '0 9 * * MON' },
    });
  }

  async getQueueStats() {
    const [waiting, active, completed, failed] = await Promise.all([
      this.emailQueue.getWaitingCount(),
      this.emailQueue.getActiveCount(),
      this.emailQueue.getCompletedCount(),
      this.emailQueue.getFailedCount(),
    ]);
    return { waiting, active, completed, failed };
  }
}
```

### Consumer (Processor)

```typescript
import { Processor, WorkerHost, OnWorkerEvent } from '@nestjs/bullmq';
import { Job } from 'bullmq';

@Processor('email', { concurrency: 10 })
export class EmailProcessor extends WorkerHost {

  async process(job: Job): Promise<any> {
    switch (job.name) {
      case 'welcome':
        return this.handleWelcome(job);
      case 'weekly-digest':
        return this.handleDigest(job);
      default:
        throw new Error(`Unknown job: ${job.name}`);
    }
  }

  private async handleWelcome(job: Job) {
    await job.updateProgress(10);
    const user = await this.userService.findById(job.data.userId);
    await job.updateProgress(50);
    await this.mailer.send({ to: user.email, template: 'welcome' });
    await job.updateProgress(100);
    return { sent: true };
  }

  @OnWorkerEvent('completed')
  onCompleted(job: Job) {
    console.log(`[Email] Job ${job.id} done`);
  }

  @OnWorkerEvent('failed')
  onFailed(job: Job, error: Error) {
    console.error(`[Email] Job ${job.id} failed: ${error.message}`);
  }
}
```

### Register Processor trong Module

```typescript
@Module({
  imports: [
    BullModule.registerQueue({ name: 'email' }),
  ],
  providers: [EmailService, EmailProcessor],
})
export class EmailModule {}
```

---

## 14. Bull Board – Dashboard UI

```bash
npm install @bull-board/nestjs @bull-board/express @bull-board/api
```

```typescript
// app.module.ts
import { BullBoardModule } from '@bull-board/nestjs';
import { BullMQAdapter } from '@bull-board/api/bullMQAdapter';
import { ExpressAdapter } from '@bull-board/express';

@Module({
  imports: [
    BullBoardModule.forRoot({ route: '/queues', adapter: ExpressAdapter }),
    BullBoardModule.forFeature({
      name: 'email',
      adapter: BullMQAdapter,
    }),
  ],
})
export class AppModule {}
```

Truy cập: `http://localhost:3000/queues` để xem dashboard.

---

## 15. Patterns & Best Practices

### 15.1. Idempotent Jobs
```typescript
// Dùng jobId cố định để tránh duplicate
await queue.add('sync-user', { userId }, {
  jobId: `sync-user-${userId}`,  // nếu job đang chờ/active → skip
});
```

### 15.2. Dead Letter Queue
```typescript
// Worker fail → chuyển sang queue riêng để xử lý sau
worker.on('failed', async (job, error) => {
  if (job.attemptsMade >= job.opts.attempts) {
    await deadLetterQueue.add('dead', {
      originalQueue: 'email',
      jobName: job.name,
      jobData: job.data,
      error: error.message,
    });
  }
});
```

### 15.3. Graceful Shutdown
```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  app.enableShutdownHooks(); // tự động gọi onModuleDestroy
  await app.listen(3000);
}

// Trong processor
@Processor('email')
export class EmailProcessor extends WorkerHost implements OnModuleDestroy {
  async onModuleDestroy() {
    await this.worker.close(); // đợi job hiện tại xong rồi mới close
  }
}
```

### 15.4. Phân tách Queue theo priority
```typescript
// Tạo nhiều queue thay vì dùng priority
// → dễ scale worker riêng theo từng loại
BullModule.registerQueue(
  { name: 'email-critical' },  // transactional emails
  { name: 'email-bulk' },      // marketing emails
)
```

### 15.5. Không để job quá nặng
```typescript
// ❌ Lưu data lớn trực tiếp vào job
await queue.add('process', { csvData: '...10MB...' });

// ✅ Chỉ lưu reference
await queue.add('process', { fileUrl: 's3://bucket/file.csv' });
```

---

## 16. Xử lý lỗi & Retry strategy

```typescript
// Custom backoff
const worker = new Worker('task', processor, {
  connection,
  settings: {
    backoffStrategy: (attemptsMade, type, err, job) => {
      // Không retry lỗi validation
      if (err instanceof ValidationError) return -1;
      // Retry sau 5s, 10s, 20s...
      return attemptsMade * 5000;
    },
  },
});

// Trong processor – throw để trigger retry
async process(job: Job) {
  try {
    await riskyOperation(job.data);
  } catch (err) {
    if (err.isRetryable) throw err;          // retry
    await job.moveToFailed(err, true);       // fail ngay, không retry
  }
}
```

---

## 17. Checklist khi dùng BullMQ

- [ ] Luôn đặt `attempts` và `backoff` cho job quan trọng
- [ ] Set `removeOnComplete` để tránh Redis đầy
- [ ] Dùng `jobId` cố định cho idempotent jobs
- [ ] Graceful shutdown để tránh mất job
- [ ] Monitor với Bull Board hoặc QueueEvents
- [ ] Tách queue theo loại công việc (email, report, sync...)
- [ ] Chỉ lưu reference (ID, URL) thay vì data lớn trong job
- [ ] Xử lý stalled jobs (worker crash giữa chừng)
- [ ] Log đầy đủ trong processor để debug
- [ ] Test processor độc lập (không cần Redis)
