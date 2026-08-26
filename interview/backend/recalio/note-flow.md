# Luồng Preview & Confirm Note

## Tổng quan

2 endpoints chính:
- **`POST /notes/preview`** (public) — nhận danh sách từ, detect ngôn ngữ, kiểm tra AudioCache, trả preview
- **`POST /notes/confirm`** (auth) — upsert notes vào DB, xử lý audio (inline nếu 1 từ, queue nếu ≥2 từ)

---

## File structure

```
src/modules/notes/
├── note.controller.ts      # Routes
├── note.service.ts         # Business logic (preview, confirm, resolveAudio)
├── note.repository.ts      # DB operations (createBatch, countByDeck, findById, update)
├── note.dto.ts             # DTOs: PreviewNoteDto, ConfirmNoteDto, ConfirmWordDto, UpdateNoteDto
├── note.constant.ts        # Hằng số, AudioStatus, AudioSource, WordPreviewItem, PreviewSummary
├── note.error.ts           # Exception factory
├── note.module.ts          # Module wiring
└── note.processor.ts       # BullMQ worker: resolve audio async

src/modules/audio/
├── audio.module.ts                   # Module wiring
├── audio-cache.repository.ts         # DB: AudioCache CRUD (findByTextLanguage, mget, save)
└── services/
    ├── audio-cache.service.ts        # Proxy qua repository
    ├── dictionary.service.ts         # Free Dictionary API → Cloudinary (English only)
    └── tts.service.ts                # Google TTS → Cloudinary (fallback)

src/modules/languages/
└── language.service.ts               # detectLanguage(), isSupported()

src/modules/decks/
└── deck.service.ts                   # getOwner(), checkReadAccess()
```

---

## 1. `note.controller.ts` — Routes

```ts
@Controller('notes')
export class NoteController {
    // GET /notes/decks/:deckId — public list notes
    @Get('decks/:deckId')
    @Public()
    async findByDeck(@CurrentUser('id') userId: string | undefined, @Param('deckId') deckId: string, @Query() dto: PaginationDto) {
        return this.service.findByDeck(userId, deckId, dto);
    }

    // POST /notes/preview — public, detect language + check audio cache
    @Post('preview')
    @Public()
    async preview(@Body() dto: PreviewNoteDto) {
        return this.service.preview(dto);
    }

    // POST /notes/confirm — auth, upsert notes + resolve audio
    @Post('confirm')
    @ApiBearerAuth()
    async confirm(@CurrentUser('id') userId: string, @Body() dto: ConfirmNoteDto) {
        return this.service.confirm(dto, userId);
    }

    // PATCH /notes/:id
    @Patch(':id')
    @ApiBearerAuth()
    async update(@CurrentUser('id') userId: string, @Param('id') id: string, @Body() dto: UpdateNoteDto) {
        return this.service.update(userId, id, dto);
    }

    // DELETE /notes/:id
    @Delete(':id')
    @HttpCode(HttpStatus.NO_CONTENT)
    @ApiBearerAuth()
    async delete(@CurrentUser('id') userId: string, @Param('id') id: string) {
        await this.service.delete(userId, id);
    }
}
```

- `preview` và `findByDeck` là `@Public()` — không cần token.
- `confirm`, `update`, `delete` yêu cầu `@ApiBearerAuth()`, lấy `userId` từ `@CurrentUser('id')`.

---

## 2. `note.constant.ts` — Hằng số & Types

```ts
export const NOTE_CONSTANTS = {
    TAG_MAX: 20,
    TAG_MAX_LENGTH: 50,
    NOTES_PER_DECK_MAX: 50,
    SORT_FIELDS: ['createdAt', 'updatedAt', 'word'] as const,
} as const;

export const AudioStatus = {
    UNSUPPORTED: 'UNSUPPORTED',
    USER_PROVIDED: 'USER_PROVIDED',
    CACHE_HIT: 'CACHE_HIT',
    CACHE_MISS: 'CACHE_MISS',
} as const;

export const AudioSource = {
    USER: 'USER',
    DICTIONARY: 'DICTIONARY',
    TTS: 'TTS',
} as const;

export interface WordPreviewItem {
    word: string;
    detectedLanguage: string;
    isSupported: boolean;
    audioStatus: AudioStatus;
    audioUrl: string | null;
    audioSource: AudioSource | null;
    userAudioUrl: string | null;
    willUseCachedAudio: boolean;
}

export interface PreviewSummary {
    total: number;
    cacheHit: number;
    cacheMiss: number;
    userAudioProvided: number;
    unsupportedLanguage: number;
}
```

- `WordPreviewItem` — kết quả trả về cho mỗi từ trong preview.
- `PreviewSummary` — tổng hợp số liệu cache hit/miss.
- `AudioStatus` — trạng thái audio của từng từ.
- `AudioSource` — nguồn audio (user tự cung cấp / dictionary API / TTS).

---

## 3. `note.dto.ts` — DTOs

### PreviewNoteDto / PreviewWordDto

```ts
export class PreviewWordDto {
    @IsNotEmpty()
    word: string;

    @IsOptional()
    detectedLanguage?: string;        // Gợi ý ngôn ngữ, nếu không có sẽ detect tự động

    @IsOptional()
    userAudioUrl?: string;            // Audio do user tự cung cấp
}

export class PreviewNoteDto {
    @ValidateNested({ each: true })
    @ArrayMinSize(1)
    words: PreviewWordDto[];
}
```

### ConfirmNoteDto / ConfirmWordDto

```ts
export class ConfirmWordDto {
    @IsOptional()
    id?: string;                      // Có → update, không có → tạo mới

    @IsOptional()                     // Bắt buộc nếu là tạo mới (validate trong service)
    templateId?: string;

    @IsNotEmpty()
    languageId: string;

    @IsOptional()
    word?: string;

    @IsOptional()
    meaning?: string;

    // ... ipa, partOfSpeech, example, audioUrl, imageUrl, tags, fields
}

export class ConfirmNoteDto {
    @IsNotEmpty()
    deckId: string;

    @ValidateNested({ each: true })
    @ArrayMinSize(1)
    words: ConfirmWordDto[];
}
```

- `ConfirmWordDto.templateId` là `@IsOptional()` — service tự validate bắt buộc cho creates (có `id` là update, không cần templateId mới).

---

## 4. `note.service.ts` — Business Logic

### 4a. `preview(dto)` — xử lý preview

```ts
async preview(dto: PreviewNoteDto) {
    // 1. Detect ngôn ngữ cho từng từ
    const withLang = dto.words.map((item) => {
        const detectedLanguage = item.detectedLanguage ?? this.languageService.detectLanguage(item.word);
        return { ...item, detectedLanguage, isSupported: this.languageService.isSupported(detectedLanguage) };
    });

    // 2. Chỉ check cache cho từ được support + không có userAudioUrl
    const needCacheCheck = withLang.filter((item) => item.isSupported && !item.userAudioUrl);
    const cacheMap = await this.audioCache.mget(
        needCacheCheck.map(({ word, detectedLanguage }) => ({ text: word, language: detectedLanguage })),
    );

    // 3. Xây dựng response cho từng từ
    const words: WordPreviewItem[] = withLang.map((item) => {
        if (!item.isSupported) {
            return { word: item.word, audioStatus: 'UNSUPPORTED', ... };
        }
        if (item.userAudioUrl) {
            return { word: item.word, audioStatus: 'USER_PROVIDED', ... };
        }
        const hashKey = this.audioCache.buildHashKey(item.word, item.detectedLanguage);
        const cachedUrl = cacheMap.get(hashKey) ?? null;
        return { word: item.word, audioStatus: cachedUrl ? 'CACHE_HIT' : 'CACHE_MISS', ... };
    });

    // 4. Tổng hợp summary
    return { summary: { total, cacheHit, cacheMiss, userAudioProvided, unsupportedLanguage }, words };
}
```

**Logic:**
1. Với mỗi từ:
   - Nếu user gửi `detectedLanguage` → dùng luôn, không thì gọi `languageService.detectLanguage()`
   - Kiểm tra ngôn ngữ có được support không
2. Nếu được support và không có `userAudioUrl` → batch query `AudioCache.mget()` bằng text + language
3. Xếp loại `AudioStatus`:
   - `UNSUPPORTED` — ngôn ngữ không được support
   - `USER_PROVIDED` — user tự cung cấp audio URL
   - `CACHE_HIT` — có sẵn trong AudioCache
   - `CACHE_MISS` — chưa có, cần resolve sau

### 4b. `confirm(dto, userId)` — upsert notes

```ts
async confirm(dto: ConfirmNoteDto, userId: string) {
    // 1. Kiểm tra quyền sở hữu deck
    const ownerId = await this.deckService.getOwner(dto.deckId);
    if (!ownerId) throw NoteError.deckNotFound();
    if (ownerId !== userId) throw NoteError.notOwner();

    // 2. Validate ngôn ngữ được support
    const invalidWords = dto.words.filter((w) => !this.languageService.isSupported(w.languageId));
    if (invalidWords.length > 0) throw new BadRequestException(...);

    // 3. Phân loại: update (có id) vs create (không id)
    const toUpdate = dto.words.filter((w) => w.id);
    const toCreate = dto.words.filter((w) => !w.id);

    // 4. Validate templateId bắt buộc cho creates
    const createWithoutTemplate = toCreate.filter((w) => !w.templateId);
    if (createWithoutTemplate.length) throw new BadRequestException('templateId là bắt buộc cho từ mới');

    // 5. Kiểm tra notes tồn tại cho updates
    for (const item of toUpdate) {
        const note = await this.repo.findById(item.id!);
        if (!note || note.deckId !== dto.deckId) throw NoteError.notFound();
    }

    // 6. Kiểm tra giới hạn 50 notes/deck
    if (toCreate.length) {
        const current = await this.repo.countByDeck(dto.deckId);
        if (current + toCreate.length > NOTE_CONSTANTS.NOTES_PER_DECK_MAX) throw NoteError.limitExceeded();
    }

    // 7. Lấy cardTemplateIds chỉ cho creates
    const cardTemplateMap: Record<string, string[]> = {};
    const createTemplateIds = [...new Set(toCreate.map((w) => w.templateId).filter((tid): tid is string => !!tid))];
    await Promise.all(createTemplateIds.map(async (tid) => {
        cardTemplateMap[tid] = await this.noteTemplateService.getCardTemplateIds(tid);
    }));

    // 8. Update: chỉ gửi field có giá trị
    for (const item of toUpdate) {
        const updateData: Record<string, unknown> = {};
        if (item.templateId !== undefined) updateData.templateId = item.templateId;
        if (item.languageId !== undefined) updateData.languageId = item.languageId;
        if (item.word !== undefined) updateData.word = item.word;
        // ... các field khác
        await this.repo.update(item.id!, updateData as any);
    }

    // 9. TRƯỜNG HỢP 1: 1 từ mới → inline resolve audio
    if (dto.words.length === 1 && !dto.words[0].id) {
        const created = await this.repo.createBatch(userId, dto.deckId, [word], cardTemplateMap);
        const note = created[0];
        if (note && !note.audioUrl) {
            const audioUrl = await this.resolveAudio(note.word ?? '', note.languageId);
            if (audioUrl) await this.repo.update(note.id!, { audioUrl });
        }
        return { created: 1, updated: 0, audioJobs: 0 };
    }

    // 10. TRƯỜNG HỢP 2: ≥2 từ hoặc có update → batch + queue
    const created = toCreate.length ? await this.repo.createBatch(userId, dto.deckId, toCreate, cardTemplateMap) : [];
    const audioJobs = created.filter((n) => !n.audioUrl);
    if (audioJobs.length) {
        await this.noteAudioProducer.addBulk(audioJobs.map((n) => ({ noteId: n.id, word: n.word ?? '', language: n.languageId })));
    }

    return { created: created.length, updated: toUpdate.length, audioJobs: audioJobs.length };
}
```

### 4c. `resolveAudio()` — private method

```ts
private async resolveAudio(word: string, language: string): Promise<string | null> {
    // 1. AudioCache → đã có URL
    const cached = await this.audioCache.findByTextLanguage(word, language);
    if (cached?.audioUrl) return cached.audioUrl;

    // 2. English → Free Dictionary API
    if (language === 'en') {
        const dictUrl = await this.dictionary.fetchAudio(word);
        if (dictUrl) { await this.audioCache.save(word, language, dictUrl); return dictUrl; }
    }

    // 3. Fallback: Google TTS → Cloudinary
    const ttsUrl = await this.tts.generate(word, language);
    if (ttsUrl) { await this.audioCache.save(word, language, ttsUrl); return ttsUrl; }

    return null;
}
```

**Priority:** AudioCache → Dictionary (en) → TTS. Kết quả luôn được lưu vào AudioCache để lần sau không cần gọi API lại.

---

## 5. `note.repository.ts` — DB Operations

### `createBatch()`

```ts
async createBatch(userId, deckId, items, cardTemplateMap) {
    return this.prisma.$transaction(async (tx) => {
        // 1. Tạo notes với crypto.randomUUID()
        const notes = items.map((w) => ({
            id: crypto.randomUUID(),
            userId, deckId, templateId: w.templateId, languageId: w.languageId,
            word: w.word, meaning: w.meaning, ...audioUrl, ...tags, ...fields
        }));
        await tx.note.createMany({ data: notes as any });

        // 2. Tạo cards từ cardTemplateMap
        const cardData = notes.flatMap((n) => {
            const ctIds = cardTemplateMap[n.templateId] ?? [];
            return ctIds.map((ctId) => ({ id: crypto.randomUUID(), userId, noteId: n.id, cardTemplateId: ctId, deckId }));
        });
        if (cardData.length) await tx.card.createMany({ data: cardData });

        // 3. Return { id, word, languageId, audioUrl }
        return tx.note.findMany({ where: { id: { in: notes.map((n) => n.id) } }, select: { id: true, word: true, languageId: true, audioUrl: true } });
    });
}
```

- Dùng `$transaction` để đảm bảo notes + cards được tạo atomic.
- `crypto.randomUUID()` sinh ID client-side (do Prisma `createMany` không support `create` + `select`).
- Chỉ return các field cần thiết cho audio processing (`id`, `word`, `languageId`, `audioUrl`).

### `noteSelect` — response selector

```ts
const noteSelect = {
    id: true, deckId: true, templateId: true, languageId: true,
    word: true, meaning: true, ipa: true, partOfSpeech: true,
    example: true, audioUrl: true, imageUrl: true,
    tags: true, fields: true, createdAt: true, updatedAt: true,
} satisfies Prisma.NoteSelect;
```

Dùng cho mọi query — không leak `userId`, `deletedAt`, `isBanned`, `parentId`.

### Other methods

| Method | Mô tả |
|---|---|
| `countByDeck(deckId)` | Đếm số note chưa xoá trong deck |
| `findByDeck(deckId, dto)` | List notes với pagination, orderBy createdAt DESC |
| `findById(id)` | Tìm note theo id |
| `update(id, dto)` | Update note, xử lý fields json |
| `softDelete(id)` | Set `deletedAt = new Date()` |

---

## 6. `note.processor.ts` — BullMQ Worker

```ts
@Processor(QueueName.ADD_NOTE)
export class NoteProcessor extends WorkerHost {
    async process(job: Job<ProcessAudioData>) {
        const { noteId, word, language } = job.data;
        const audioUrl = await this.resolveAudio(word, language);
        if (audioUrl) {
            await this.prisma.note.update({ where: { id: noteId }, data: { audioUrl } });
        }
    }

    // resolveAudio() logic giống hệt service
}
```

Khi confirm ≥2 từ, service tạo notes trong DB rồi đẩy job vào queue. Worker chạy async (có thể sau vài giây), resolve audio và update `audioUrl` vào DB. FE/App gọi `GET /notes/decks/:deckId` sau đó sẽ thấy `audioUrl` đã có.

---

## 7. Audio Services

### `AudioCacheRepository` — DB cache

```ts
findByTextLanguage(text, language)  // → { audioUrl } | null (lowercase text)
mget(keys)                           // → Map<hashKey, audioUrl> (batch, lowercase)
save(text, language, audioUrl)       // → upsert (lowercase text)
buildHashKey(text, language)         // → "text::language" (lowercase)
```

- Tất cả text được lowercase trước khi query/save để tránh case-sensitive mismatch.

### `DictionaryService` — Free Dictionary API

```ts
async fetchAudio(word) {
    // 1. GET https://api.dictionaryapi.dev/api/v2/entries/en/{word}
    // 2. Parse response, tìm phonetic.audio đầu tiên
    // 3. Download audio buffer → upload lên Cloudinary (folder: recalio/audio/dictionary)
    // 4. Trả về Cloudinary URL
}
```

- Chỉ hoạt động với tiếng Anh.
- Upload lên Cloudinary để có URL permanent, tránh dependency vào API bên thứ ba.

### `TtsService` — Google TTS

```ts
async generate(text, language) {
    const tl = LANG_MAP[language];  // Map languageId → Google TLS code
    if (!tl) return null;

    // GET https://translate.google.com/translate_tts?ie=UTF-8&client=tw-ob&tl={tl}&q={text}
    // Download audio buffer → upload lên Cloudinary (folder: recalio/audio/tts)
    // Trả về Cloudinary URL
}
```

- Fallback khi Dictionary API không có (hoặc không phải English).
- Support 25+ ngôn ngữ (en, ja, ko, zh, vi, fr, de, ...).

---

## 8. LanguageService

```ts
detectLanguage(text) {
    // 1. Char heuristics: Japanese (Hiragana/Katakana), Korean, Chinese, Thai, Arabic, Vietnamese
    // 2. Fallback → franc (n-gram based)
    // 3. Map ISO 639-3 → ISO 639-1
}

isSupported(languageId) {
    return this.supportedSet.has(languageId);  // cached set, refresh on startup + language CRUD
}
```

---

## 9. `note.error.ts` — Exception Factory

```ts
export class NoteError {
    static notFound()           → 404 'Note không tồn tại'
    static deckNotFound()       → 404 'Deck không tồn tại'
    static notOwner()           → 403 'Bạn không có quyền thao tác với note này'
    static deckNotAccessible()  → 404 'Deck không tồn tại hoặc không thể truy cập'
    static limitExceeded()      → 400 'Deck đã đạt giới hạn 50 notes'
}
```

---

## 10. `note.module.ts` — Module Wiring

```ts
@Module({
    imports: [DeckModule, NoteTemplateModule, AudioModule, LanguageModule, QueueModule],
    controllers: [NoteController],
    providers: [NoteService, NoteRepository, NoteProcessor],
    exports: [NoteService],
})
export class NoteModule { }
```

- `AudioModule` cung cấp `AudioCacheService`, `DictionaryService`, `TtsService`
- `LanguageModule` cung cấp `LanguageService`
- `QueueModule` cung cấp `NoteAudioProducer`
- `NoteProcessor` là provider — NestJS tự động đăng ký worker khi app start

---

## 11. Luồng tổng thể

```
User → POST /notes/preview { words: [{ word: "hello" }, { word: "bonjour" }] }
  → NoteService.preview()
    → detectLanguage() cho từng từ
    → audioCache.mget() batch cache check
    ← response: { summary, words[] với audioStatus }

User → POST /notes/confirm { deckId, words: [{ word: "hello", templateId, languageId }] }
  → NoteService.confirm()
    → Kiểm tra ownership, language support, limit 50
    → Split: toUpdate (có id) vs toCreate (không id)
    → Update: chỉ gửi field có giá trị
    → [[1 từ mới]] → createBatch + resolveAudio inline → response ngay
    → [[≥2 từ]]    → createBatch + addBulk queue → response ngay (audioJobs > 0)

[Async] NoteProcessor.process(job)
  → resolveAudio(word, language)
    → AudioCache → Dictionary (en) → TTS
    → update note.audioUrl trong DB

User → GET /notes/decks/:deckId (sau vài giây)
  → audioUrl đã có trong response
```
