# 🎵 NOVAWAVE — Luồng Phát Nhạc (Music Player Flow)

> Tài liệu mô tả toàn bộ luồng phát nhạc của NOVAWAVE từ Frontend (Next.js + Zustand) đến Backend (NestJS), bao gồm các trường hợp phát từ bài hát đơn, playlist, album, quảng cáo và sóng âm thanh (WaveSurfer).

---

## 📁 Cấu trúc file liên quan

### Backend — `novawave_be/src/modules/player/`

| File | Vai trò |
|---|---|
| `player.module.ts` | Khai báo module, import SongModule, PlaylistModule, AlbumModule, AdvertisementModule |
| `controllers/player.controller.ts` | REST Controller — 4 endpoints: `start`, `next`, `previous`, `play-from-queue` |
| `services/player.service.ts` | Business logic — quản lý session in-memory, queue, history, ads |
| `dtos/start-playing.dto.ts` | DTO bắt đầu phát: `{ songId, playlistId?, albumId? }` |
| `dtos/get-next-track.dto.ts` | DTO lấy bài tiếp: `{ currentSongId }` |
| `interfaces/player-context.interface.ts` | Interface `IPlayerContext` — cấu trúc session server-side |

### Frontend — `novawave_fe/src/`

| File | Vai trò |
|---|---|
| `stores/usePlayerStore.ts` | Zustand store — global state: `status`, `isPlaying`, `audioRef`, `currentTime` |
| `stores/useSidebarStore.ts` | Zustand store — điều khiển panel phải: `info` / `queue` / `hidden` |
| `services/player.service.ts` | HTTP service — axios calls đến `/player/*` |
| `queries/usePlayerQuery.ts` | React Query mutations: `useStartPlayer`, `useNextSong`, `usePreviousSong`, `usePlayFromQueue` |
| `components/client/Player/song-bar.tsx` | **Player bar** cố định phía dưới — controls, audio element, view counting |
| `components/client/Player/song-info.tsx` | Panel phải — thông tin bài/nghệ sĩ/quảng cáo, follow nghệ sĩ |
| `components/client/Player/song-queue.tsx` | Panel phải — danh sách hàng đợi |
| `components/client/Player/song-queue-card.tsx` | Card bài hát trong queue |
| `components/client/Player/vinyl.tsx` | Hiệu ứng đĩa vinyl xoay |
| `components/client/WavePlayer/wave-player.tsx` | Sóng âm thanh (WaveSurfer.js) — sync với audio element |
| `components/client/Song/song-card.tsx` | Card bài hát — trigger `useStartPlayer` |
| `queries/useSongQuery.ts` | `useIncrementSongView` — tính lượt nghe |

---

## 🏗️ Kiến trúc tổng quan

```
┌───────────────────────────────────────────────────────────────────┐
│                        FRONTEND (Next.js)                         │
│                                                                   │
│  [SongCard / Playlist / Album / Song Detail Page]                 │
│       │  User click play                                          │
│       ▼                                                           │
│  useStartPlayer() mutation  ──────────────── POST /player/start   │
│       │  onSuccess                                                │
│       ▼                                                           │
│  usePlayerStore.setPlayerStatus()                                 │
│  { nowPlayingId, nowPlaying, queue, history, nowPlayingType }     │
│       │                                                           │
│       ├──────────────────────────────────────────────┐           │
│       ▼                                              ▼           │
│  [SongBar] (bottom fixed)              [SongInfo / SongQueue]    │
│  AudioPlayer (react-h5-audio-player)   (right panel sidebar)     │
│       │                                                           │
│       ├─ onPlay/onPause → store.play/pause                        │
│       ├─ onEnded → useNextSong()                                  │
│       ├─ onClickNext → useNextSong()                              │
│       ├─ onClickPrevious → usePreviousSong()                     │
│       ├─ onListen (30s) → useIncrementSongView()                 │
│       └─ audioRef → WavePlayer sync                               │
│                                                                   │
│  [WavePlayer] (WaveSurfer.js)                                     │
│       ├─ Lắng nghe audioRef.timeupdate                            │
│       └─ Cập nhật progress bar sóng âm thanh                     │
└───────────────────────────────────────────────────────────────────┘
                            │ REST HTTP
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│                        BACKEND (NestJS)                           │
│                                                                   │
│  PlayerController (/player)                                       │
│       │                                                           │
│       ▼                                                           │
│  PlayerService                                                    │
│  private sessions = new Map<userId, IPlayerContext>()             │
│  (In-memory, không persist qua restart)                           │
│       │                                                           │
│       ├─ SongService      — lấy detail bài hát, random song      │
│       ├─ PlaylistService  — lấy songIds của playlist             │
│       ├─ AlbumService     — kiểm tra album tồn tại               │
│       └─ AdvertisementService — random quảng cáo                 │
└───────────────────────────────────────────────────────────────────┘
```

---

## 🔑 Session Server-side — `IPlayerContext`

**File:** `interfaces/player-context.interface.ts`

```typescript
interface IPlayerContext {
  userId: string;
  currentTrack: string;    // songId đang phát
  queue: string[];          // danh sách songId sắp phát
  history: string[];        // tối đa 10-15 bài đã nghe (LIFO)
  playlistId?: string;      // nguồn phát (nếu từ playlist)
  albumId?: string;         // nguồn phát (nếu từ album)
  isPremium: boolean;       // phân biệt free/premium → có quảng cáo không
  songPlayedCount: number;  // đếm bài → cứ 3 bài chèn 1 quảng cáo
  nextAds?: string | null;  // advertisementId nếu đang phát ads
  isAdsPlaying: boolean;    // đang phát quảng cáo hay không
}
```

> **Lưu ý:** Session lưu **in-memory** (`Map`) tại BE. Không persist sang DB — khi server restart, session mất và user cần bắt đầu lại.

---

## 🚀 Luồng 1 — Bắt đầu phát nhạc

### 1A. Phát bài đơn (từ SongCard, trang chi tiết bài hát)

```
User click nút Play trên SongCard / Song Detail
        │
        ▼
song-card.tsx → startPlayerMutation({ songId })
        │
        ▼
useStartPlayer().mutate({ songId })
  → PlayerService.start({ songId }) 
  → HTTP: POST /player/start
     Body: { songId: "abc123" }
        │
        ▼
BE: PlayerService.start()
  - Tạo IPlayerContext: { currentTrack: songId, queue: [], history: [] }
  - isPremium từ JWT payload
  - Không có playlistId/albumId → queue = []
  - saveSession(userId, ctx)
  - getFullPlayerStatus(userId)
        │
        ▼
BE: getFullPlayerStatus()
  - Fetch chi tiết nowPlaying từ SongService.getDetail()
  - Fetch chi tiết queue[] (rỗng lúc đầu)
  - Fetch chi tiết history[]
  - Return: { nowPlayingId, nowPlaying: {song + type:'song'}, queue, history }
        │
        ▼
FE: onSuccess → usePlayerStore.setPlayerStatus(response)
  - status.nowPlayingId = songId
  - status.nowPlaying = { ...songDetail, type: 'song' }
  - status.queue = []
  - isPlaying = true
        │
        ▼
SongBar re-render:
  - audioSource = nowPlaying.mp3Link
  - AudioPlayer src thay đổi → tự động phát (autoPlay={isPlaying})
```

### 1B. Phát từ Playlist

```
User click Play trên Playlist Detail
        │
        ▼
startPlayerMutation({ songId, playlistId })
  → HTTP: POST /player/start
     Body: { songId: "bài_đầu", playlistId: "pl123" }
        │
        ▼
BE: PlayerService.start()
  - PlaylistService.getSongIdsOfPlaylistById(playlistId)
  - ctx.queue = playlist.songIds.filter(id ≠ songId)
    (loại bài đang phát ra khỏi queue)
  - saveSession → getFullPlayerStatus()
        │
        ▼
FE: status.queue = [bài2, bài3, ...] (đã có chi tiết)
```

### 1C. Phát từ Album

```
User click Play trên Album Detail
        │
        ▼
startPlayerMutation({ songId, albumId })
  → HTTP: POST /player/start
     Body: { songId: "bài_X", albumId: "al456" }
        │
        ▼
BE: PlayerService.start()
  - SongService.findSongIdsByAlbumId(albumId)
  - Tìm vị trí songId trong list
  - ctx.queue = list.slice(index + 1)
    (chỉ lấy từ bài sau trở đi)
  - saveSession → getFullPlayerStatus()
```

**Files liên quan:**
- `components/client/Song/song-card.tsx` — trigger từ bài hát đơn
- `app/(client)/playlist/[id]/page.tsx` — trigger từ playlist
- `app/(client)/album/[id]/page.tsx` — trigger từ album
- `app/(client)/song/[id]/page.tsx` — trigger từ trang chi tiết bài hát
- `dtos/start-playing.dto.ts`

---

## ⏭️ Luồng 2 — Chuyển bài tiếp (Next)

```
User click nút Next  hoặc  bài hát kết thúc (onEnded)
        │
        ▼
SongBar:
  handleNext() / handleEnded()
  → nextMutation.mutate({ currentSongId: nowPlayingId })
  → HTTP: GET /player/next?currentSongId=abc123
        │
        ▼
BE: PlayerService.getNextTrack(currentSongId, user)
        │
        ├─ [Nếu vừa phát xong quảng cáo]:
        │   ctx.isAdsPlaying = false, ctx.nextAds = null
        │   → Tiếp tục logic tìm bài hát tiếp (không return)
        │
        ├─ Lưu currentSongId vào history (nếu không phải ads):
        │   ctx.history.unshift(currentSongId)
        │   ctx.history = ctx.history.slice(0, 10)  // giữ 10 bài
        │
        ├─ [FREE user]: Kiểm tra chèn quảng cáo
        │   ctx.songPlayedCount++
        │   Nếu songPlayedCount >= 3:
        │     songPlayedCount = 0
        │     ads = AdvertisementService.getRandomAds()
        │     Nếu có ads:
        │       ctx.isAdsPlaying = true
        │       ctx.nextAds = ads._id
        │       return getFullPlayerStatus()  ← trả về quảng cáo
        │
        ├─ [Queue còn bài]:
        │   next = ctx.queue.shift()   ← lấy bài đầu queue
        │   ctx.currentTrack = next
        │   return getFullPlayerStatus()
        │
        └─ [Queue hết]:
            randomTrack = SongService.genRandomSong()
            ctx.currentTrack = randomTrack._id
            return getFullPlayerStatus()
        │
        ▼
FE: onSuccess → usePlayerStore.setNowPlaying(response)
  - status.nowPlayingId = bài mới / adsId
  - status.nowPlaying = { ...detail, type: 'song' | 'advertisement' }
  - isPlaying = true
        │
        ▼
SongBar:
  - isCurrentAd = nowPlayingType === 'advertisement'
  - audioSource = isCurrentAd ? ads.audioUrl : song.mp3Link
  - AudioPlayer src thay đổi → phát bài/quảng cáo mới
```

> **Chặn Skip khi đang phát quảng cáo:**
> ```typescript
> handleNext() {
>   if (isCurrentAd) {
>     toast.info("Nghe nhạc free thì chịu nghe quảng cáo đi");
>     return; // Không cho skip ads
>   }
>   nextMutation.mutate({ currentSongId: nowPlayingId });
> }
> ```

---

## ⏮️ Luồng 3 — Quay lại bài trước (Previous)

```
User click nút Previous
        │
        ▼
SongBar: handlePrev()
  │
  ├─ Nếu isCurrentAd → toast.info, return (không cho quay lại)
  │
  ├─ Nếu currentTime > 3 giây:
  │   audio.currentTime = 0   ← reset về đầu bài hiện tại
  │   return (không gọi API)
  │
  └─ Nếu currentTime ≤ 3 giây:
      previousMutation.mutate()
      → HTTP: GET /player/previous
              │
              ▼
      BE: PlayerService.getPrevious(user)
        - prev = ctx.history.shift()   ← lấy bài đầu lịch sử
        - ctx.queue.unshift(ctx.currentTrack)  ← đẩy bài hiện tại về queue
        - ctx.currentTrack = prev
        - saveSession → getFullPlayerStatus()
              │
              ▼
      FE: setNowPlaying(response)
```

---

## 📋 Luồng 4 — Phát từ hàng đợi (Play from Queue)

```
User click vào bài hát trong SongQueue panel
        │
        ▼
song-queue.tsx: handlePlayFromQueue(songId)
  → usePlayFromQueue().mutate({ songId })
  → HTTP: POST /player/play-from-queue
     Body: { songId: "xyz789" }
        │
        ▼
BE: PlayerService.playFromQueue(songId, user)
  - index = ctx.queue.findIndex(id === songId)
  - skippedFromQueue = ctx.queue.splice(0, index + 1)
    (lấy tất cả bài từ đầu đến bài được chọn)
  - chosenSongId = skippedFromQueue.pop()
    (bài cuối là bài được chọn)
  - ctx.history.unshift(ctx.currentTrack, ...skippedFromQueue)
    (bài hiện tại + bài bị vượt qua → lưu vào history)
  - ctx.history = ctx.history.slice(0, 15)
  - ctx.currentTrack = chosenSongId
  - saveSession → getFullPlayerStatus()
        │
        ▼
FE: setPlayerStatus(response)
  - Queue được cắt ngắn (chỉ còn từ bài kế tiếp trở đi)
  - History bao gồm bài cũ + các bài bị nhảy qua
```

---

## 👀 Luồng 5 — Tính lượt nghe (View Count)

```
SongBar: handleListen(e) được gọi liên tục (onListen event)
        │
        ├─ Nếu đang phát quảng cáo → return (không tính view)
        │
        ├─ Nếu currentTime >= 30 giây
        │  VÀ viewCounted !== nowPlayingId:
        │
        ▼
useIncrementSongView().mutate(nowPlayingId)
  → HTTP: PATCH /songs/:id/view   (hoặc tương đương)
        │
        ▼
setViewCounted(nowPlayingId)  ← đánh dấu đã tính, không tính thêm lần nữa

Reset:
useEffect([nowPlayingId]):
  setViewCounted(null)  ← khi đổi bài mới, reset để có thể tính lại
```

**Logic đảm bảo mỗi bài chỉ được tính view 1 lần / lần phát:**
```typescript
const [viewCounted, setViewCounted] = useState<string | null>(null);
// viewCounted lưu ID bài đã tính view
// Chỉ increment khi: currentTime >= 30s && viewCounted !== nowPlayingId
```

---

## 〰️ Luồng 6 — Sóng âm thanh (WavePlayer + WaveSurfer.js)

**File:** `components/client/WavePlayer/wave-player.tsx`

```
Điều kiện hiển thị sóng:
  shouldInitWave = (alwaysShowWave || isThisSongPlaying) && !!url
  - alwaysShowWave=true: trang chi tiết bài hát (luôn hiện sóng)
  - isThisSongPlaying: bài hát đang được phát trong player
```

### 6A. Khởi tạo WaveSurfer

```
[useEffect] khi url thay đổi hoặc shouldInitWave = true:
        │
        ▼
WaveSurfer.create({
  container: waveformRef.current,
  url,          ← mp3Link của bài hát
  waveColor: "rgba(255,255,255,0.45)",
  progressColor: "#1DB954",  ← màu xanh Spotify
  barWidth: 2, barGap: 1.5, barRadius: 2,
  height: 90,
  normalize: true,
  dragToSeek: true,
})
        │
        ▼
ws.on("ready") → setWaveDuration(ws.getDuration())
ws.on("interaction", time) → onSeek(time)
  → seekToTime(time) trong usePlayerStore
  → audioRef.currentTime = time
```

### 6B. Sync sóng với audio element

```
[useEffect] khi audioRef, isPlaying, isThisSongPlaying thay đổi:
        │
        ▼
audioRef.addEventListener("timeupdate", handleTimeUpdate)
        │
        ▼
handleTimeUpdate():
  progress = audioRef.currentTime / audioRef.duration
  wavesurferRef.current.seekTo(progress)  ← cập nhật vị trí thanh sóng
```

### 6C. Seek từ sóng âm về player

```
User kéo / click trên sóng WaveSurfer
        │
        ▼
ws.on("interaction", time) → onSeekRef.current(time)
        │
        ▼
Component cha (Song Detail Page):
  onSeek={(time) => {
    usePlayerStore.getState().seekToTime(time);
  }}
        │
        ▼
usePlayerStore.seekToTime(time):
  audioRef.currentTime = time
  setCurrentTime(time)
  audioRef.dispatchEvent(new Event('timeupdate'))
```

### 6D. Comment markers trên sóng

```
commentMarkers = [{ timeSec, avatarUrl, id }]
        │
        ▼
WavePlayer render:
  Mỗi marker → button tuyệt đối tại left = (timeSec / duration) * 100%
  Hiển thị avatar hoặc dấu chấm
  Click marker → onSeek(timeSec) → seek đến đúng vị trí comment
```

---

## 📢 Luồng 7 — Quảng cáo (Free Users)

```
Điều kiện: user.isPremium === false
        │
        ▼
BE: Mỗi lần getNextTrack():
  ctx.songPlayedCount++  (chỉ tính bài hát, không tính ads)

  Nếu songPlayedCount >= 3:
    songPlayedCount = 0
    ads = AdvertisementService.getRandomAds()
    Nếu tìm được ads:
      ctx.isAdsPlaying = true
      ctx.nextAds = ads._id
      return getFullPlayerStatus()  ← nowPlaying = advertisement
        │
        ▼
FE: nowPlayingType = 'advertisement'
  isCurrentAd = true
        │
        ├─ audioSource = currentAd.audioUrl
        ├─ displayName = currentAd.title
        ├─ displaySubText = "Được tài trợ bởi {currentAd.partner}"
        ├─ displayImageUrl = currentAd.bannerUrl
        │
        ├─ handleNext() → toast.info("Nghe nhạc free...") → return
        ├─ handlePrev() → toast.info("Nghe nhạc free...") → return
        └─ WavePlayer: không hiển thị sóng cho quảng cáo
        │
        ▼
Khi quảng cáo kết thúc (onEnded):
  nextMutation.mutate({ currentSongId: nowPlayingId })
  → HTTP: GET /player/next
        │
        ▼
BE: getNextTrack()
  ctx.isAdsPlaying === true:
    ctx.isAdsPlaying = false, ctx.nextAds = null
    → Không return sớm, tiếp tục logic tìm bài hát tiếp theo
    → Lấy bài từ queue (hoặc random nếu queue trống)
```

---

## 📡 REST API — Bảng tổng hợp

| Method | Endpoint | Auth | Mô tả |
|---|---|---|---|
| `POST` | `/player/start` | ✅ | Bắt đầu phát — tạo session, dựng queue |
| `GET` | `/player/next?currentSongId=` | ✅ | Lấy bài tiếp theo (có logic ads cho free user) |
| `GET` | `/player/previous` | ✅ | Quay về bài trước (từ history) |
| `POST` | `/player/play-from-queue` | ✅ | Phát bài cụ thể trong queue |

### Response structure (`getFullPlayerStatus`)

```typescript
{
  nowPlayingId: string,
  nowPlaying: {
    // Song hoặc Advertisement object
    _id, name, mp3Link, imageUrl, artistId, ...
    type: 'song' | 'advertisement'  // thêm vào khi trả về
  },
  queueIds: string[],               // chỉ IDs
  queue: Song[],                    // đã populate chi tiết
  history: Song[]                   // đã populate chi tiết
}
```

---

## 🧩 State Management (Zustand — Frontend)

### `usePlayerStore` — `stores/usePlayerStore.ts`

```typescript
status: {
  nowPlayingId: string | null,       // ID bài/ads đang phát
  nowPlaying: Song | Ad | null,      // chi tiết bài/ads
  queueIds: string[],                // chỉ IDs (để check trùng)
  queue: Song[],                     // chi tiết bài trong hàng đợi
  history: Song[],                   // lịch sử (UI hiển thị)
  nowPlayingType: 'song' | 'advertisement' | null
}
isPlaying: boolean                   // đang phát hay dừng
currentTime: number                  // vị trí phát hiện tại (giây)
audioRef: HTMLAudioElement | null    // DOM audio element

// Actions
setPlayerStatus(status)  → cập nhật status + set isPlaying=true
setNowPlaying(status)    → alias của setPlayerStatus
play()                   → isPlaying = true
pause()                  → isPlaying = false
setAudioRef(ref)         → lưu DOM audio element vào store
setCurrentTime(time)     → cập nhật currentTime
seekToTime(time)         → audioRef.currentTime = time + dispatch event
```

**Persist:** Zustand `persist` middleware lưu `status` và `currentTime` vào `localStorage` với key `music-player-storage`.
- Khi reload: `status` được khôi phục nhưng `isPlaying` bị reset về `false` (onRehydrateStorage).

### `useSidebarStore` — `stores/useSidebarStore.ts`

```typescript
rightPanelMode: 'info' | 'queue' | 'hidden'  // panel phải hiển thị gì
middleSize: string                            // '70%' | '85%' (width content chính)

showInfo()    → mode='info', middleSize='70%'
showQueue()   → mode='queue', middleSize='70%'
hideRightPanel() → mode='hidden', middleSize='85%'
```

---

## 🔗 Các điểm trigger phát nhạc

| Nơi trigger | Component | Context |
|---|---|---|
| Trang chủ — card bài hát | `song-card.tsx` | Phát bài đơn |
| Top chart | `top-song-2.tsx`, `top-song-3.tsx` | Phát bài đơn |
| Trang chi tiết bài hát | `app/(client)/song/[id]/page.tsx` | Phát bài đơn + WavePlayer |
| Trang playlist | `app/(client)/playlist/[id]/page.tsx` | Phát từ playlist |
| Trang album | `app/(client)/album/[id]/page.tsx` | Phát từ album |
| Hàng đợi trong player | `song-queue-card.tsx` | Phát từ queue (play-from-queue) |

---

## ⚙️ Đặc điểm kỹ thuật quan trọng

### `syncingAudioRef` flag
Không tồn tại trong player cá nhân này (chỉ có trong phòng nghe nhạc chung). Player cá nhân hoạt động hoàn toàn local.

### Audio element lifecycle
```
SongBar mount:
  playerRef (react-h5-audio-player) → ref
  useEffect → setAudioRef(playerInstance.audio.current)
  → audioRef được lưu vào Zustand store

Cleanup (SongBar unmount):
  setAudioRef(null)
```

### WaveSurfer không điều khiển audio trực tiếp
WaveSurfer chỉ **đọc** tiến trình từ `audioRef` (qua event `timeupdate`) và **seek** gián tiếp qua `usePlayerStore.seekToTime()`. Audio element thật là `react-h5-audio-player`, không phải WaveSurfer internal audio.

### Quảng cáo không lưu vào history
```typescript
// Chỉ lưu bài hát vào lịch sử (không lưu quảng cáo)
if (!wasAdsPlaying) {
  ctx.history.unshift(currentSongId);
}
```
