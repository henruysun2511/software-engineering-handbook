# BullMQ Queue System — Audio Processing for Notes

## Tổng quan

Queue system dùng **BullMQ** (trên Redis) để xử lý bất đồng bộ audio cho các từ vựng khi import batch notes. Nếu user confirm **1 từ** → xử lý inline (đồng bộ). Nếu **≥2 từ** → tạo notes trong DB, đẩy job vào queue, response ngay lập tức.

---

## 1. `src/config/redis.config.ts` — Redis Connection Config

Cấu hình kết nối Redis cho BullMQ (retry strategy, keepAlive, offline queue).

```ts
import { AppConfig } from './app.config';

export const redisConnection = {
    host: AppConfig.REDIS_HOST,
    port: AppConfig.REDIS_PORT,
    password: AppConfig.REDIS_PASSWORD || undefined,
    maxRetriesPerRequest: null,
    retryStrategy(times: number) {
        const delay = Math.min(times * 50, 2000);
        return delay;
    },
    reconnectOnError(err: Error) {
        if (err.message.includes('ECONNRESET')) {
            return true;
        }
        return false;
    },
    keepAlive: 30000,
    enableOfflineQueue: true,
};
```

- `maxRetriesPerRequest: null` — **Bắt buộc với BullMQ** để queue tự quản lý retry, không để ioredis retry mãi.
- `retryStrategy` — Tăng dần delay (tối đa 2s) khi mất kết nối.
- `reconnectOnError` — Tự động reconnect nếu lỗi ECONNRESET.
- `keepAlive + enableOfflineQueue` — Giữ kết nối TCP, xếp hàng job khi Redis offline.

---

## 2. `src/config/queue.config.ts` — Default Queue Options

Cấu hình BullMQ global: prefix theo môi trường, retry 3 lần, backoff exponential 2s.

```ts
import { redisConnection } from './redis.config';
import { AppConfig } from './app.config';

export const defaultQueueOptions = {
    connection: redisConnection,
    prefix: `bull:${AppConfig.NODE_ENV}`,
    defaultJobOptions: {
        attempts: 3,
        backoff: {
            type: 'exponential' as const,
            delay: 2000,
        },
        timeout: 5 * 60 * 1000,
        removeOnComplete: { count: 100 },
        removeOnFail: { age: 7 * 24 * 3600, count: 1000 },
    },
};
```

- `prefix` — `bull:development` / `bull:production` để phân cách môi trường.
- `defaultJobOptions` — Áp dụng cho tất cả job:
  - `attempts: 3` — Retry tối đa 3 lần (kể cả lần đầu = 3 attempts tổng cộng).
  - `backoff: exponential, 2s` — 2s → 4s → 8s giữa các lần retry.
  - `timeout: 5 phút` — Job bị coi là failed nếu chạy quá 5 phút.
  - `removeOnComplete: 100` — Chỉ giữ 100 job completed gần nhất.
  - `removeOnFail: 7 ngày / 1000` — Giữ failed job 1 tuần, tối đa 1000.

---

## 3. `src/infrastructures/queue/queue.constant.ts` — Tên Queue & Job

Định nghĩa tên queue và job name dùng chung cho toàn bộ hệ thống queue.

```ts
export const QueueName = {
    ADD_NOTE: 'add-note',
} as const;

export const JobName = {
    PROCESS_WORD: 'process-word',
} as const;
```

- `QueueName.ADD_NOTE = 'add-note'` — Queue duy nhất hiện tại, xử lý audio cho notes.
- `JobName.PROCESS_WORD = 'process-word'` — Job name cho mỗi từ cần resolve audio.

---

## 4. `src/infrastructures/queue/queue.module.ts` — Queue Module

Module đăng ký queue 'add-note' và export NoteAudioProducer (producer pattern).

```ts
@Module({
    imports: [
        BullModule.registerQueue({ name: QueueName.ADD_NOTE }),
    ],
    providers: [NoteAudioProducer],
    exports: [NoteAudioProducer],
})
export class QueueModule { }
```

- `BullModule.registerQueue` — Đăng ký queue `add-note` với NestJS DI.
- Chỉ export `NoteAudioProducer` (không export Queue trực tiếp) — tuân thủ **producer pattern**.

---

## 5. `src/infrastructures/queue/producers/note-audio.producer.ts` — Producer

Producer thêm job xử lý audio vào queue 'add-note' (hỗ trợ đơn lẻ và bulk).

```ts
@Injectable()
export class NoteAudioProducer {
    constructor(@InjectQueue(QueueName.ADD_NOTE) private readonly queue: Queue) { }

    async addJob(noteId: string, word: string, language: string) {
        return this.queue.add(JobName.PROCESS_WORD, { noteId, word, language }, {
            attempts: 3,
            backoff: { type: 'exponential', delay: 2000 },
            removeOnComplete: { count: 1000 },
            removeOnFail: { count: 5000 },
        });
    }

    async addBulk(jobs: { noteId: string; word: string; language: string }[]) {
        return this.queue.addBulk(
            jobs.map((data) => ({
                name: JobName.PROCESS_WORD,
                data,
                opts: {
                    attempts: 3,
                    backoff: { type: 'exponential', delay: 2000 },
                    removeOnComplete: { count: 1000 },
                    removeOnFail: { count: 5000 },
                },
            })),
        );
    }
}
```

- `addJob(noteId, word, language)` — Thêm 1 job vào queue.
- `addBulk(jobs[])` — Thêm nhiều job cùng lúc (dùng `queue.addBulk`, tối ưu hơn gọi `addJob` trong loop).
- `opts` override `defaultJobOptions` của queue config — giữ nhiều job completed/failed hơn (1000/5000) vì mỗi job là 1 từ riêng lẻ.

---

## 6. `src/modules/notes/note.processor.ts` — Worker (Processor)

Worker xử lý audio cho note: AudioCache → Dictionary (en) → TTS, cập nhật audioUrl vào DB.

```ts
@Processor(QueueName.ADD_NOTE)
export class NoteProcessor extends WorkerHost {
    // Inject PrismaService, AudioCacheService, DictionaryService, TtsService

    async process(job: Job<ProcessAudioData>) {
        const { noteId, word, language } = job.data;
        try {
            const audioUrl = await this.resolveAudio(word, language);
            if (audioUrl) {
                await this.prisma.note.update({
                    where: { id: noteId },
                    data: { audioUrl },
                });
            }
        } catch (err) {
            throw err; // BullMQ auto-retry dựa vào attempts/backoff config
        }
    }

    private async resolveAudio(word: string, language: string): Promise<string | null> {
        // 1. Kiểm tra AudioCache → đã có URL
        const cached = await this.audioCache.findByTextLanguage(word, language);
        if (cached?.audioUrl) return cached.audioUrl;

        // 2. Nếu là tiếng Anh → tra Free Dictionary API
        if (language === 'en') {
            const dictUrl = await this.dictionary.fetchAudio(word);
            if (dictUrl) {
                await this.audioCache.save(word, language, dictUrl);
                return dictUrl;
            }
        }

        // 3. Fallback: Google TTS → Cloudinary
        const ttsUrl = await this.tts.generate(word, language);
        if (ttsUrl) {
            await this.audioCache.save(word, language, ttsUrl);
            return ttsUrl;
        }

        return null;
    }
}
```

- `extends WorkerHost` — Abstract class NestJS cung cấp sẵn, implement `process()`.
- **Priority sourcing audio**: AudioCache (DB) → Dictionary API (English only) → Google TTS + Cloudinary.
- **Fail → retry**: Nếu `resolveAudio` throw (VD: TTS API lỗi), job tự động retry 3 lần với backoff exponential.
- `@OnWorkerEvent('completed' | 'failed')` — Log kết quả xử lý từng job.

---

## 7. `src/modules/notes/note.service.ts` (queue-related phần) — Gọi Producer

### Trong `confirm()`:

```ts
// TRƯỜNG HỢP 1: 1 từ mới → xử lý inline
if (dto.words.length === 1 && !dto.words[0].id) {
    const created = await this.repo.createBatch(userId, dto.deckId, [word], cardTemplateMap);
    const note = created[0];
    if (note && !note.audioUrl) {
        const audioUrl = await this.resolveAudio(note.word ?? '', note.languageId);
        if (audioUrl) {
            await this.repo.update(note.id!, { audioUrl });
        }
    }
    return { created: 1, updated: 0, audioJobs: 0 };
}

// TRƯỜNG HỢP 2: ≥2 từ mới → batch + queue
const audioJobs = created.filter((n) => !n.audioUrl);
if (audioJobs.length) {
    await this.noteAudioProducer.addBulk(
        audioJobs.map((n) => ({ noteId: n.id, word: n.word ?? '', language: n.languageId })),
    );
}
```

- **1 từ**: Gọi `this.resolveAudio()` (private method của service) trực tiếp, giống hệt logic trong processor. Response có `audioUrl` ngay.
- **≥2 từ**: `producer.addBulk()`, response ngay không chờ audio (`audioJobs > 0` để FE biết có job đang chạy).

Phương thức `resolveAudio` private trong service copy chính xác logic từ processor để đảm bảo inline và queue cho kết quả giống nhau.

---

## 8. `src/app.module.ts` — Root Config

```ts
@Module({
    imports: [
        SharedModule,
        BullModule.forRoot(defaultQueueOptions),  // ← BullMQ global config
        AuthModule,
        // ... feature modules
    ],
})
```

- `BullModule.forRoot(defaultQueueOptions)` — Cấu hình BullMQ global cho toàn app (Redis connection, prefix, default job options).

---

## 9. `src/modules/notes/note.module.ts` — Wiring

```ts
@Module({
    imports: [
        DeckModule,
        NoteTemplateModule,
        AudioModule,
        LanguageModule,
        QueueModule,
    ],
    providers: [NoteService, NoteRepository, NoteProcessor],
    exports: [NoteService],
})
```

- `QueueModule` được import bởi `NoteModule`.
- `NoteProcessor` là provider của `NoteModule` — nó tự động được NestJS khởi tạo và đăng ký listener cho queue `add-note`.
- Khi app chạy, processor sẽ consume job từ Redis queue ngay lập tức.

---

## Luồng dữ liệu tổng thể

```
User POST /notes/confirm (≥2 từ)
  → NoteService.confirm()
    → NoteRepository.createBatch()   [save to DB]
    → NoteAudioProducer.addBulk()    [push to Redis queue]
  ← response { created: N, audioJobs: M }

[Async] BullMQ Worker:
  → NoteProcessor.process(job)
    → resolveAudio(word, lang)
      → AudioCache?       [DB check]
      → Dictionary? (en)  [Free Dictionary API]
      → TTS (fallback)    [Google TTS → Cloudinary]
    → NoteRepository.update(audioUrl)  [save to DB]
  ← done

User GET notes → audioUrl đã có (sau khi worker hoàn thành)
```

---

## File structure

```
src/
├── config/
│   ├── redis.config.ts          # Redis connection config
│   └── queue.config.ts          # BullModule.forRoot options
├── infrastructures/
│   └── queue/
│       ├── queue.constant.ts    # QueueName, JobName
│       ├── queue.module.ts      # QueueModule (register + export producer)
│       └── producers/
│           └── note-audio.producer.ts  # NoteAudioProducer
└── modules/
    └── notes/
        ├── note.processor.ts    # Worker (NoteProcessor)
        ├── note.service.ts      # Gọi producer khi confirm batch
        └── note.module.ts       # Import QueueModule, register NoteProcessor
```
