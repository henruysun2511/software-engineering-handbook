# Study Flow

End-to-end flow: fetch due cards → start session → review cards → end session → view stats.

---

## 1. Session Lifecycle

```
[Study Dashboard] ──click "Học"──→ [Study Page] ──fetch due cards──→ [Done / Review]
                                                                         │
                                                                    [rate card]
                                                                         │
                                                           ┌─────────────┴─────────────┐
                                                           │ not last card             │ last card
                                                           │ → next card               │ → end session
                                                           └───────────────────────────┘
                                                                         │
                                                                   [Completion Screen]
                                                                         │
                                                     ┌───────────────────┴──────────┐
                                                     │ Xem chi tiết                │ Danh sách
                                                     │ → /study/session/:id        │ → /study
                                                     └──────────────────────────────┘
```

### 1.1 Create Session

**Endpoint:** `POST /study-sessions`  
**Body:** `{ deckId?: string }` — deckId optional (null = cross-deck session)  
**Backend** (`StudySessionService.start`):

1. If `deckId` provided:
   - Verify deck exists + user has read access via `DeckService.checkReadAccess`
   - Check for existing **active** session for same user+deck (`findActiveByUserAndDeck`) → resume if found
2. Check `countActiveSessions(userId)` ≤ `MAX_ACTIVE_SESSIONS` (5)
3. Create `StudySession` record (`{ userId, deckId, startedAt: now(), endedAt: null }`)
4. Return `{ id, deckId, startedAt, endedAt }`

**Frontend** (`study/[deckId]/page.tsx`):

```ts
useEffect → load due cards → if cards exist:
  setAllCards(dueCards)
  startSession().then(() => setPhase("studying"))
```

- `startSession` calls `createSession.mutateAsync({ deckId })`
- `sessionIdRef.current = session.id` → used to associate reviews with session
- Phase stays `"loading"` (spinner) until session is created

### 1.2 End Session

**Endpoint:** `PATCH /study-sessions/:id/end`  
**Backend** (`StudySessionService.end`):

1. Find session by ID → fail if not found / not owner / already ended
2. Set `endedAt = new Date()`
3. Return updated `{ id, deckId, startedAt, endedAt }`

**Frontend triggers (both call `endSession.mutateAsync`):**

| Trigger | Condition |
|---|---|
| `handleRating` | `isLastCard` = true |
| `handleSkip` (bury/suspend) | `isLastCard` = true |

```ts
if (isLastCard) {
  if (sessionIdRef.current) {
    await endSession.mutateAsync(sessionIdRef.current)
  }
  setPhase("done")
}
```

---

## 2. Study Page (`/study/[deckId]`)

### 2.1 Card Loading

**Endpoint:** `GET /cards/due?deckId=xxx&page=1&limit=200`  
**Backend** (`CardService.getDueCards`):

- Deck setting limits: `newCardsPerDay`, `reviewsPerDay` (counted from today's ReviewLog)
- Query: `state IN [NEW, LEARNING, RELEARNING, REVIEW]`, `due <= now()`, note `deletedAt IS NULL`
- Ordered by `state ASC, due ASC`
- Limit applies separately to new cards vs review cards
- Response includes rendered `frontHtml` / `backHtml` (template substitution from Note fields)

### 2.2 Review Flow

```
[Card front shown]
       │
       ▼  (click / Space)
[Card flipped — back shown]
       │
       ├── [Ẩn] → PATCH /cards/:id/bury → set due = tomorrow 00:00 → skip card
       ├── [Tạm dừng] → PATCH /cards/:id/suspend → toggle state SUSPENDED → skip card
       │
       └── [Again / Hard / Good / Easy]
                  │
                  ▼
           POST /cards/:id/review
           { rating, responseTimeMs, sessionId? }
                  │
                  ▼
           Update card (SM-2) + create ReviewLog + award XP
                  │
                  ▼
           Last card? ─yes──→ PATCH /study-sessions/:id/end → done
              │
              no
              │
              ▼
           Next card
```

### 2.3 State Machine (SM-2)

See [Card Review Engine](#3-card-review-engine-sm-2) below.

### 2.4 Completion Screen

Shows:
- Total cards reviewed
- Accuracy % (`(good + easy) / total * 100`)
- "Xem chi tiết" → `/study/session/{sessionId}`
- "Danh sách" → `/study`

---

## 3. Card Review Engine (SM-2)

**Endpoint:** `POST /cards/:id/review`  
**Body:** `{ rating: AGAIN|HARD|GOOD|EASY, responseTimeMs: number, sessionId?: string }`

### 3.1 State Machine

```
NEW ────GOOD────→ LEARNING ────GOOD (step ≥ last) ────→ REVIEW
  │                    │                                    │
  ├─AGAIN→ NEW         ├─AGAIN→ LEARNING (step=0)           ├─AGAIN→ RELEARNING
  ├─HARD→ LEARNING     ├─HARD→ LEARNING (×2 current step)   ├─HARD→ REVIEW (×1.2, ease-0.15)
  └─EASY→ REVIEW       └─EASY→ REVIEW                       ├─GOOD→ REVIEW (×ease)
                                                              └─EASY→ REVIEW (×ease×1.3, ease+0.15)
```

### 3.2 Constants

| Constant | Default | Description |
|---|---|---|
| `LEARNING_STEPS` | `[1, 10]` | Steps in minutes for LEARNING/RELEARNING |
| `GRADUATING_INTERVAL` | 1 day | Interval when graduating from LEARNING → REVIEW |
| `EASY_INTERVAL` | 4 days | Interval for EASY from NEW state |
| `EASY_BONUS` | 1.3 | Multiplier for EASY rating in REVIEW state |
| `MIN_EASE` | 1.3 | Minimum ease factor |
| `MAX_INTERVAL` | 36500 days | Cap on all intervals |
| `RELEARNING_STEPS` | `[10]` | Steps in minutes after AGAIN in REVIEW |
| `HARD_INTERVAL` | 1.2 | Multiplier for HARD rating in REVIEW state |
| `INTERVAL_MODIFIER` | 1.0 | Global modifier from deck settings |

All constants overridable per-deck via `DeckSetting`.

### 3.3 Side Effects

1. **Card update:**
   - `state`, `interval`, `easeFactor`, `repetitions`, `lapses`, `currentStep`, `due`, `lastReviewAt`

2. **ReviewLog created:**
   - `{ userId, cardId, sessionId?, rating, stateBefore, stateAfter, intervalBefore, intervalAfter, easeBefore, easeAfter, responseTimeMs, reviewedAt }`

3. **Gamification** (`GamificationService.awardReviewXp`):
   - Base XP per review (default 10)
   - Daily goal bonus: `floor(targetReviews × 0.5)` on first meeting target each day
   - Streak tracking (increments consecutive study days via `lastStudyDate`)
   - Achievement checks (streak/reviews/cards thresholds)
   - Level formula: `floor(sqrt(totalXP / LEVEL_BASE_XP)) + 1` (default `LEVEL_BASE_XP = 100`)

---

## 4. Session Detail Page (`/study/session/[id]`)

### 4.1 Data Fetching

| Query | Endpoint | Usage |
|---|---|---|
| Session + stats | `GET /study-sessions/:id` | `{ id, deckId, startedAt, endedAt, stats }` |
| Review logs | `GET /study-sessions/:id/review-logs?page=&limit=` | `{ data: ReviewLog[], meta: pagination }` |
| Deck name | `GET /decks/:id` | Display deck name |

### 4.2 Session Stats (computed)

`StudySessionRepository.getSessionReviewStats(sessionId)`:

```sql
SELECT rating, COUNT(*), SUM(responseTimeMs)
FROM review_logs
WHERE session_id = ?
GROUP BY rating
```

Returns:
```json
{
  "reviewedCards": 12,
  "timeSpentMs": 45000,
  "again": 2,
  "hard": 3,
  "good": 5,
  "easy": 2
}
```

### 4.3 Display

- **Stat cards:** reviewed count, duration (from `startedAt` → `endedAt`), status (Xong/Đang học)
- **Rating distribution:** bar chart with Again/Hard/Good/Easy counts + percentages
- **Review logs:** paginated list with rating badge, word/meaning, response time, state transition

---

## 5. Study Dashboard (`/study`)

### 5.1 Main Sections

```
┌─ Streak + Goal bar ─────────────────────┐
│ 🔥 Streak 7 | 🎯 Mục tiêu: 45/50 ██████ │
└──────────────────────────────────────────┘

┌─ Stats cards ────────────────────────────┐
│ 12 thẻ đến hạn │ 5 chưa học │ ...        │
└──────────────────────────────────────────┘

┌─ Học tất cả ─────────────────────────────┐
│ ▶ Học tất cả — 12 thẻ                    │
└──────────────────────────────────────────┘

┌─ Bộ thẻ của bạn ─────────────────────────┐
│ [Deck 1] 80% ████████░░  Cần học         │
│ [Deck 2] 100% ██████████  Hoàn thành     │
└──────────────────────────────────────────┘

┌─ Lịch sử học ────────────────────────────┐
│ Deck 1 — 10/07 14:30  5 phút             │
│ Deck 2 — 10/07 13:15  Đang học           │
└──────────────────────────────────────────┘
```

### 5.2 Session History

- Fetched via `GET /study-sessions?page=1&limit=10`
- Shows last 5 sessions inline
- Each item: deck name, date/time, duration (if ended) or "Đang học" badge
- Click → `/study/session/{id}`

---

## 6. Key Data Models

### 6.1 StudySession

```
┌─────────────────────────────┐
│ StudySession                │
├─────────────────────────────┤
│ id          String @id      │
│ userId      String          │
│ deckId      String?         │ ← nullable for cross-deck sessions
│ startedAt   DateTime        │
│ endedAt     DateTime?       │ ← null = active
│                             │
│ user        User @relation  │
│ deck        Deck? @relation │
│ reviewLogs  ReviewLog[]     │
└─────────────────────────────┘
```

### 6.2 ReviewLog

```
┌─────────────────────────────┐
│ ReviewLog                   │
├─────────────────────────────┤
│ id             String @id   │
│ userId         String       │
│ cardId         String       │
│ sessionId      String?      │ ← nullable (reviews without session)
│ rating         ReviewRating │
│ stateBefore    CardState    │
│ stateAfter     CardState    │
│ intervalBefore Int          │
│ intervalAfter  Int          │
│ easeBefore     Float        │
│ easeAfter      Float        │
│ responseTimeMs Int          │
│ reviewedAt     DateTime     │
│                             │
│ user    User          @r    │
│ card    Card          @r    │
│ session StudySession? @r    │
└─────────────────────────────┘
```

---

## 7. Key API Endpoints

### Study Sessions

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/study-sessions` | Create/resume session |
| `PATCH` | `/study-sessions/:id/end` | End session |
| `GET` | `/study-sessions` | List session history |
| `GET` | `/study-sessions/:id` | Session detail + stats |
| `GET` | `/study-sessions/:id/review-logs` | Paginated review logs |

### Cards

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/cards/due` | Fetch due cards |
| `POST` | `/cards/:id/review` | Submit review (SM-2) |
| `PATCH` | `/cards/:id/suspend` | Toggle suspend state |
| `PATCH` | `/cards/:id/bury` | Bury until tomorrow |
| `GET` | `/cards/stats` | Card stats by state |

---

## 8. Error Handling

### Common Errors

| Scenario | HTTP | Message |
|---|---|---|
| Session not found | 404 | Phiên học không tồn tại |
| Not session owner | 403 | Bạn không có quyền thao tác với phiên học này |
| Session already ended | 400 | Phiên học đã kết thúc |
| Too many active sessions | 400 | Đã đạt giới hạn phiên học đang mở |
| Card suspended | 400 | Thẻ đang bị tạm dừng |

### Frontend error handling

- All API errors caught in `try/catch` → displayed via `handleError` (toast)
- Session creation failure → `router.push("/study")` (redirect to dashboard)
- Review failure → toast only, user can retry
- Skip (bury/suspend) failure → toast only, user can retry
