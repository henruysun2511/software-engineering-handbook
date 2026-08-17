# Câu hỏi & Trả lời phỏng vấn — NovaWave
> Dựa trực tiếp vào code thực tế: `usePlayerStore.ts`, `wave-player.tsx`, `song-bar.tsx`, `song-comment.tsx`, `song-info.tsx`, `song-queue.tsx`, `room-detail-left-panel.tsx`, `room-detail-right-panel.tsx`, `room-song-bar.tsx`, `room-visualizer-panel.tsx`, `room-comment-panel.tsx`, `room-member-panel.tsx`, `room-info-panel.tsx`, `room-queue-panel.tsx`, `room-lyrics-panel.tsx`, `room-song-search-panel.tsx`, `room-update-panel.tsx`, `comment-swiper.tsx`

---

## PHẦN 1 — ZUSTAND PLAYER STORE

---

### Câu 1: `usePlayerStore` được thiết kế như thế nào? Tại sao dùng `persist` cho player state?

**Trả lời:**

`usePlayerStore` quản lý toàn bộ trạng thái audio playback global:

```ts
{
  status: {
    nowPlayingId: null,      // ID bài đang phát
    nowPlaying: null,        // Object đầy đủ (song hoặc ad)
    queueIds: [],            // Mảng ID hàng đợi
    queue: [],               // Mảng object đầy đủ
    history: [],             // Lịch sử phát
    nowPlayingType: null,    // 'song' | 'advertisement'
  },
  isPlaying: false,
  currentTime: 0,
  audioRef: null,            // Ref đến HTMLAudioElement thực
}
```

`persist` được dùng để **khôi phục trạng thái sau khi reload trang**. User đang nghe bài đến phút 2:30, F5 → trang load lại → `status` và `currentTime` được restore từ localStorage, player biết cần phát bài gì và từ vị trí nào.

Tuy nhiên `audioRef` **không được persist** vì đây là DOM reference — không thể serialize và không có nghĩa khi restore. `isPlaying` cũng được force về `false` khi rehydrate:

```ts
onRehydrateStorage: () => (state) => {
  if (state) {
    state.isPlaying = false; // Browser không tự play lại sau reload (autoplay policy)
  }
}
```

`partialize` chỉ persist những gì cần thiết:
```ts
partialize: (state) => ({
  status: state.status,
  currentTime: state.currentTime,
  // audioRef, isPlaying bị loại bỏ
})
```

---

### Câu 2: `seekToTime` trong `usePlayerStore` hoạt động như thế nào? Tại sao cần dispatch `timeupdate` event thủ công?

**Trả lời:**

```ts
seekToTime: (time) => {
  const audio = get().audioRef;
  if (audio) {
    audio.currentTime = time;          // 1. Seek audio thực
    get().setCurrentTime(time);        // 2. Sync Zustand store
    
    // 3. Dispatch event để đảm bảo UI cập nhật
    const timeupdate = new Event('timeupdate');
    audio.dispatchEvent(timeupdate);
  }
},
```

`seekToTime` được gọi từ `song-comment.tsx` khi user click vào timestamp của comment:
```ts
<button onClick={() => seekToTime(c.playbackPositionSec)}>
  {formatDuration(c.playbackPositionSec)}
</button>
```

**Tại sao cần dispatch `timeupdate` thủ công:**

Khi set `audio.currentTime = time`, browser tự fire `timeupdate` event. Nhưng nếu audio đang ở trạng thái **paused**, một số browser không fire `timeupdate` sau khi seek. Dispatch thủ công đảm bảo `WavePlayer` và các component lắng nghe `timeupdate` đều nhận được signal để update UI ngay lập tức — kể cả khi đang pause.

---

### Câu 3: `audioRef` trong store — tại sao lưu DOM reference vào Zustand? Có vấn đề gì không?

**Trả lời:**

`audioRef` là `HTMLAudioElement` reference được set từ `SongBar`:
```ts
// song-bar.tsx
useEffect(() => {
  const playerInstance = playerRef.current as any;
  if (playerInstance?.audio?.current) {
    const audioElement = playerInstance.audio.current;
    setAudioRef(audioElement);
    return () => setAudioRef(null);
  }
}, [setAudioRef, audioSource]);
```

Lý do lưu vào Zustand: nhiều component cần truy cập `audioElement` trực tiếp để:
- `WavePlayer` — attach `timeupdate` listener để sync waveform
- `seekToTime` action — set `audio.currentTime`
- `RoomVisualizer` — tạo `MediaElementAudioSourceNode` từ audio element
- `song-bar.tsx` — detect `audioRef` để play khi source thay đổi

Nếu không lưu vào store, cần prop drilling từ `SongBar` → rất nhiều tầng component.

**Vấn đề tiềm ẩn:** DOM reference không thể serialize → không persist được (đã handle bằng `partialize`). Nếu `SongBar` unmount, `audioRef` trở thành stale reference → phải cleanup về `null` trong effect return. Đây là lý do `setAudioRef(null)` được gọi trong cleanup function.

---

### Câu 4: Logic phân biệt `song` và `advertisement` trong `SongBar` — em xử lý như thế nào?

**Trả lời:**

Từ `song-bar.tsx`, type được đọc từ `nowPlaying.type`:

```ts
const nowPlayingType = currentData?.type;
const isCurrentAd = nowPlayingType === PlaySongType.ADVERTISEMENT;

const currentSong = !isCurrentAd ? currentData : null;
const currentAd = isCurrentAd ? currentData : null;
```

Từ đó, toàn bộ UI display được conditional theo `isCurrentAd`:
```ts
const displayName = isCurrentAd ? currentAd?.title : currentSong?.name;
const audioSource = isCurrentAd ? currentAd?.audioUrl : currentSong?.mp3Link;
```

**Khi đang phát quảng cáo, các tính năng bị lock:**
```ts
const handleNext = () => {
  if (isCurrentAd) {
    toast.info("Nghe nhạc free thì chịu nghe quảng cáo đi");
    return;
  }
  // ...
};
```

Skip, previous đều bị block. `handleListen` cũng skip view counting khi là ad:
```ts
if (isCurrentAd) return;
```

Player CSS cũng thêm class riêng:
```tsx
className={`custom-audio-player ${isCurrentAd ? 'ad-mode' : ''}`}
```

---

### Câu 5: Logic `handlePrev` — tại sao phát quá 3 giây thì về đầu bài thay vì về bài trước?

**Trả lời:**

Đây là UX pattern phổ biến của Spotify/Apple Music:

```ts
const handlePrev = () => {
  if (isCurrentAd || !nowPlayingId || isSkipLoading) return;

  // Đọc currentTime trực tiếp từ store (không phải từ closure)
  const isPlayedLongEnough = usePlayerStore.getState().currentTime > 3;

  if (isPlayedLongEnough) {
    const audio = usePlayerStore.getState().audioRef;
    if (audio) {
      audio.currentTime = 0;
      setCurrentTime(0);
      return; // ← Không gọi previousMutation
    }
  }

  previousMutation.mutate({ currentSongId: nowPlayingId });
};
```

Nếu đã nghe > 3 giây: reset bài về 0 thay vì nhảy bài trước. Lý do về UX: user có thể vô tình bấm previous khi đang nghe — việc reset về đầu bài ít gây phiền hơn là nhảy sang bài khác.

Điểm kỹ thuật đáng chú ý: dùng `usePlayerStore.getState().currentTime` thay vì đọc từ closure. Nếu đọc từ closure của `usePlayerStore((s) => s.currentTime)`, giá trị có thể bị stale do closure capture tại thời điểm component render. `getState()` luôn trả về giá trị hiện tại.

---

### Câu 6: View counting logic — tại sao count sau 30 giây và dùng `viewCounted` state?

**Trả lời:**

```ts
const [viewCounted, setViewCounted] = useState<string | null>(null);

const handleListen = (e: any) => {
  if (isCurrentAd || !audioSource) return;
  const currentTime = e.target.currentTime;

  if (currentTime >= 30 && viewCounted !== nowPlayingId) {
    incrementView(nowPlayingId ?? "");
    setViewCounted(nowPlayingId); // Đánh dấu đã count cho bài này
  }
};

// Reset khi đổi bài
useEffect(() => {
  if (nowPlayingId) setViewCounted(null);
}, [nowPlayingId]);
```

**30 giây** — ngưỡng phổ biến trong ngành để coi là "lượt nghe hợp lệ" (tương tự YouTube = 30s, Spotify = ~30s). Tránh tính view khi user chỉ preview vài giây rồi skip.

**`viewCounted` state** — guard để đảm bảo chỉ gọi `incrementView` **một lần duy nhất** mỗi bài. `handleListen` được gọi mỗi 250ms (theo `listenInterval`), nếu không có guard sẽ gọi API liên tục sau khi qua mốc 30s. So sánh `viewCounted !== nowPlayingId` thay vì boolean flag để handle đúng khi user skip bài rồi quay lại bài cũ.

---

## PHẦN 2 — WAVESURFER & WAVEFORM

---

### Câu 7: Two-way sync giữa WaveSurfer waveform và audio player — làm sao tránh infinite loop?

**Trả lời:**

Trong `wave-player.tsx`, hai chiều được tách biệt rõ ràng bằng cơ chế khác nhau:

**Chiều 1 — Audio → Waveform (khi đang play):**
```ts
useEffect(() => {
  if (!audioRef || !isPlaying || !isThisSongPlaying) return;

  const handleTimeUpdate = () => {
    const ws = wavesurferRef.current;
    if (!ws || !ws.getDuration() || !audioRef) return;
    const progress = audioRef.currentTime / (audioRef.duration || 1);
    ws.seekTo(progress); // Sync waveform theo audio
  };

  audioRef.addEventListener("timeupdate", handleTimeUpdate);
  return () => audioRef.removeEventListener("timeupdate", handleTimeUpdate);
}, [audioRef, isPlaying, isThisSongPlaying]);
```

**Chiều 2 — User kéo waveform → Audio:**
```ts
const unsubInteraction = ws.on("interaction", (time: number) => {
  onSeekRef.current?.(time); // Gọi callback onSeek truyền từ cha
});
```

Callback `onSeek` → gọi `seekToTime(time)` từ store → set `audio.currentTime` trực tiếp.

**Tại sao không loop:** `ws.seekTo()` không fire event `interaction` — nó là method call một chiều. `interaction` event chỉ fire khi **user tương tác trực tiếp** (click/drag) vào canvas WaveSurfer, không fire khi code gọi `seekTo()`. Hai chiều dùng hai mechanism khác nhau nên không tạo loop.

**Guard thêm** — khi `currentTime` thay đổi từ store, chỉ sync nếu lệch hơn 0.5 giây:
```ts
useEffect(() => {
  const ws = wavesurferRef.current;
  if (!ws || !ws.getDuration() || !isThisSongPlaying) return;
  const diff = Math.abs(ws.getCurrentTime() - currentTime);
  if (diff > 0.5) {
    ws.seekTo(currentTime / ws.getDuration());
  }
}, [currentTime, isThisSongPlaying]);
```
Ngưỡng 0.5 giây tránh sync liên tục khi `currentTime` update mỗi 250ms — chỉ sync khi có sai lệch đáng kể (ví dụ: user seek từ timestamp comment).

---

### Câu 8: `shouldInitWave` và `alwaysShowWave` — WaveSurfer được khởi tạo khi nào?

**Trả lời:**

```ts
const isThisSongPlaying = songId === nowPlayingSongId;
const shouldInitWave = (alwaysShowWave || isThisSongPlaying) && !!url;
```

Có 2 mode:

**Mode 1 — `alwaysShowWave = false` (default, dùng trong `SongBar` area):** WaveSurfer chỉ được khởi tạo khi `songId` của component khớp với bài đang phát. Nếu render 50 card bài hát mà không có flag này, 50 WaveSurfer instances sẽ được tạo đồng thời — rất tốn memory và CPU.

**Mode 2 — `alwaysShowWave = true` (dùng trong trang chi tiết bài hát):** WaveSurfer render ngay cả khi bài chưa được phát, để user thấy waveform trước khi play.

Khi `shouldInitWave = false`, component render placeholder:
```tsx
<div className="wave-wrapper flex h-[90px] items-center justify-center ...">
  <p>Phát bài hát này để xem sóng âm thanh</p>
</div>
```

Khi `songId` thay đổi hoặc bài được play, `shouldInitWave` thay đổi → `useEffect` chạy lại → WaveSurfer được khởi tạo mới. WaveSurfer cũ được cleanup trong return:
```ts
return () => {
  unsubReady();
  unsubInteraction();
  ws.destroy();           // Giải phóng memory
  wavesurferRef.current = null;
  setWaveDuration(0);
};
```

---

### Câu 9: Comment markers (avatar) được render lên waveform như thế nào?

**Trả lời:**

Em không vẽ lên canvas của WaveSurfer mà dùng **HTML overlay** định vị tuyệt đối bên trên:

```tsx
<div className="wave-wrapper relative">
  <div ref={waveformRef} id="waveform" />  {/* WaveSurfer canvas */}

  {waveDuration > 0 && commentMarkers.map((m) => {
    const t = Math.min(Math.max(0, m.timeSec), waveDuration - 0.01);
    const leftPct = (t / waveDuration) * 100;

    return (
      <button
        key={m.id}
        style={{
          position: 'absolute',
          left: `${leftPct}%`,
          bottom: 0,
          transform: 'translateX(-50%)',
          width: 28, height: 28,
        }}
        onClick={(e) => {
          e.stopPropagation();
          onSeekRef.current?.(m.timeSec); // Click avatar → seek đến timestamp
        }}
      >
        <img src={m.avatarUrl} className="rounded-full object-cover" />
      </button>
    );
  })}
</div>
```

`left: (timeSec / waveDuration) * 100%` — map timestamp sang phần trăm chiều ngang của waveform. `waveDuration` được đọc từ `ws.getDuration()` sau khi WaveSurfer fire event `ready`.

`e.stopPropagation()` trên click avatar ngăn event bubble xuống waveform canvas — nếu không, WaveSurfer sẽ nhận click và seek đến vị trí đó, gây double-seek.

Guard `waveDuration > 0`: chỉ render markers sau khi WaveSurfer load xong và biết duration. Nếu render trước, `leftPct` sẽ là `NaN` hoặc `Infinity`.

---

### Câu 10: `onSeekRef` — tại sao dùng ref thay vì truyền `onSeek` trực tiếp vào event handler?

**Trả lời:**

```ts
const onSeekRef = useRef<((time: number) => void) | null>(null);

useEffect(() => {
  onSeekRef.current = onSeek || null;
}, [onSeek]);
```

WaveSurfer event listener (`ws.on("interaction", handler)`) được đăng ký một lần trong `useEffect` khởi tạo. Nếu dùng `onSeek` trực tiếp trong handler, closure sẽ capture giá trị `onSeek` tại thời điểm listener được tạo (có thể là `undefined` hoặc giá trị cũ).

Pattern "ref mirror" này đảm bảo handler luôn gọi `onSeek` hiện tại nhất mà không cần re-register listener. Tương tự pattern `queryClientRef` trong BingeBox — giải quyết cùng vấn đề stale closure nhưng trong context khác.

```ts
// Trong useEffect khởi tạo WaveSurfer:
const unsubInteraction = ws.on("interaction", (time: number) => {
  onSeekRef.current?.(time); // Luôn gọi onSeek mới nhất
});
```

Nếu `onSeek` prop thay đổi (parent re-render), `useEffect` update ref nhưng không cần re-run useEffect khởi tạo WaveSurfer.

---

## PHẦN 3 — TIMESTAMP-LINKED COMMENTS

---

### Câu 11: Timestamp-linked comment system hoạt động như thế nào? Comment được link với timestamp ra sao?

**Trả lời:**

Từ `song-comment.tsx`, khi user tạo comment, `playbackPositionSec` được lấy từ Zustand store:

```ts
const currentTime = usePlayerStore((state) => state.currentTime);

createComment(
  {
    songId,
    content,
    playbackPositionSec: Math.floor(currentTime || 0), // Timestamp lúc gửi
  },
  { onSuccess: () => refetch() }
);
```

Timestamp được lưu vào DB cùng với content. Khi render list comment:

```tsx
{typeof c.playbackPositionSec === "number" && c.playbackPositionSec >= 0 && (
  <button
    onClick={() => seekToTime(c.playbackPositionSec)}
    className="rounded-full border border-green px-2 py-0.5 text-xs text-green"
  >
    {formatDuration(c.playbackPositionSec)} {/* "1:23" */}
  </button>
)}
```

Click vào badge timestamp → gọi `seekToTime` từ store → audio seek đến đúng giây đó.

Timestamp badge chỉ hiển thị nếu `playbackPositionSec >= 0` (comment có timestamp). Comment tạo khi chưa phát bài (currentTime = 0) vẫn có timestamp = 0 — là hợp lệ. Check `typeof === "number"` đảm bảo không render khi field là `null` hoặc `undefined`.

Data flow:
```
User gõ comment lúc 1:23 → currentTime = 83
→ API: { content, playbackPositionSec: 83 }
→ DB lưu 83
→ Render: badge "1:23"
→ Click badge → seekToTime(83) → audio.currentTime = 83
→ WavePlayer nhận timeupdate → seekTo(83/duration)
```

---

### Câu 12: Comment markers trong `WavePlayer` — data được truyền từ đâu xuống?

**Trả lời:**

Trong trang chi tiết bài hát (`song/[id]/page.tsx`), comments được fetch rồi transform thành `commentMarkers`:

```ts
// page.tsx (song detail)
const { data: commentsData } = useCommentList(songId, { page: 1 });

const commentMarkers: WaveCommentMarker[] = (commentsData?.data || [])
  .filter((c: any) => typeof c.playbackPositionSec === "number")
  .map((c: any) => ({
    id: c._id,
    timeSec: c.playbackPositionSec,
    avatarUrl: c.userId?.avatar,
  }));

// Truyền xuống WavePlayer
<WavePlayer
  url={song.mp3Link}
  currentTime={currentTime}
  onSeek={seekToTime}
  songId={song._id}
  alwaysShowWave={true}
  commentMarkers={commentMarkers}
/>
```

`WaveCommentMarker` interface:
```ts
export interface WaveCommentMarker {
  id?: string;
  timeSec: number;
  avatarUrl?: string;
}
```

Chỉ comments có `playbackPositionSec` hợp lệ mới được include. Avatar hiển thị ảnh user, nếu không có thì render dot `·`.

---

## PHẦN 4 — SHARED LISTENING ROOM

---

### Câu 13: Kiến trúc tổng thể của shared listening room — ai control playback? Members sync như thế nào?

**Trả lời:**

Room theo mô hình **host-controlled**: chỉ host mới có quyền play/pause/seek/next, members chỉ nghe theo.

Từ `room-info-panel.tsx`, các control button chỉ render cho host:
```tsx
{isHost && (
  <>
    <Button onClick={() => emitHostControl(
      room.isPlaying ? RoomControlAction.PAUSE : RoomControlAction.PLAY,
      { currentTime: Math.round(audioRef.current?.currentTime ?? playbackSeconds) * 1000 }
    )}>
      {room.isPlaying ? "Tạm dừng" : "Tiếp tục"}
    </Button>
    <Button onClick={() => emitHostControl(RoomControlAction.NEXT)}>Bài tiếp theo</Button>
    <Button onClick={() => emitHostControl(RoomControlAction.END)}>Kết thúc</Button>
  </>
)}
```

Member player bị lock bằng CSS:
```tsx
<div className={`${isHost ? "" : "pointer-events-none opacity-80"}`}>
  <AudioPlayer ... />
</div>
```

`pointer-events-none` ngăn member click vào player controls.

**Sync flow:**
1. Host bấm Play → `emitHostControl(PLAY, { currentTime })` → emit WebSocket event lên server
2. Server broadcast đến tất cả member trong room
3. Client mỗi member nhận event → update `room.isPlaying` và `playbackSeconds` → `AudioPlayer` với `autoPlay={room.isPlaying}` tự play

Member indicator trạng thái:
```tsx
{!isHost && (
  <div className="animate-pulse">
    {room.isPlaying ? "Đang đồng bộ cùng chủ phòng" : "Đang chờ chủ phòng..."}
  </div>
)}
```

---

### Câu 14: `RoomVisualizer` — Web Audio API được setup như thế nào? Tại sao dùng singleton pattern?

**Trả lời:**

```ts
// Singleton globals — tồn tại ngoài component
let sharedAudioCtx: AudioContext | null = null;
let sharedAnalyser: AnalyserNode | null = null;
let sharedSource: MediaElementAudioSourceNode | null = null;
```

**Tại sao singleton:** `MediaElementAudioSourceNode` chỉ được tạo **một lần** cho mỗi `HTMLAudioElement`. Nếu gọi `createMediaElementSource(audio)` lần 2 với cùng element, browser throw error `"InvalidStateError: HTMLMediaElement already connected"`. Singleton đảm bảo source chỉ được tạo một lần dù component re-render hoặc user chuyển tab.

```ts
useEffect(() => {
  if (!sharedAudioCtx) {
    sharedAudioCtx = new AudioContext();
    sharedAnalyser = sharedAudioCtx.createAnalyser();
    sharedAnalyser.fftSize = 512; // Độ phân giải frequency
  }

  if (!sharedSource) {
    try {
      sharedSource = sharedAudioCtx.createMediaElementSource(audio);
      sharedSource.connect(sharedAnalyser!);
      sharedAnalyser!.connect(sharedAudioCtx.destination); // Output ra loa
    } catch (e) {
      console.warn("Audio source already connected");
    }
  }
}, [audioRef, songName]);
```

**Pipeline:** `AudioElement` → `MediaElementSourceNode` → `AnalyserNode` → `AudioDestination` (loa). `AnalyserNode` như một "tap" — audio vẫn chạy bình thường, đồng thời cho phép đọc frequency data.

**Visualizer logic — bass-driven animation:**
```ts
let bass = 0;
for (let i = 0; i < 12; i++) bass += dataArray[i]; // 12 frequency bins thấp nhất
bass = bass / 12;
const bassFactor = bass / 255; // Normalize 0-1

// Star speed tăng theo bass
const starSpeed = 1.5 + bassFactor * 25;
// Glow circle radius mở rộng theo bass
const baseRadius = 100 + bassFactor * 30;
```

`requestAnimationFrame` loop render frame → đọc `analyser.getByteFrequencyData(dataArray)` → vẽ canvas → request frame tiếp theo.

---

### Câu 15: `RoomSongBar` — tại sao member không thể control player nhưng vẫn nghe được nhạc?

**Trả lời:**

`AudioPlayer` (react-h5-audio-player) vẫn render và load audio source cho tất cả members, nhưng controls bị vô hiệu hóa bằng CSS:

```tsx
<div className={`${isHost ? "" : "pointer-events-none opacity-80"}`}>
  <AudioPlayer
    src={room.currentSong?.mp3Link}
    autoPlay={room.isPlaying}
    autoPlayAfterSrcChange={room.isPlaying}
    showSkipControls={isHost}           // Controls chỉ hiện với host
    onClickNext={() => emitHostControl(RoomControlAction.NEXT)}
    onListen={(e) => onPlayerListen(e.target.currentTime)}
    onPlay={onPlayerPlay}
    onPause={onPlayerPause}
    onSeeked={(e) => onPlayerSeek(e.target.currentTime)}
    onEnded={onPlayerEnded}
  />
</div>
```

`pointer-events-none` chặn tất cả mouse/touch events → member không thể click play/pause/seek. Nhưng audio element vẫn active và phát nhạc.

`autoPlay={room.isPlaying}` và `autoPlayAfterSrcChange={room.isPlaying}`: khi server broadcast trạng thái room thay đổi → React re-render với `room.isPlaying` mới → `AudioPlayer` nhận prop mới và tự điều chỉnh playback.

`onPlayerListen` được gọi mỗi 250ms khi đang play, cập nhật `playbackSeconds` để:
- `RoomLyricsPanel` sync thanh progress
- `RoomInfoPanel` hiển thị thời gian đúng

---

### Câu 16: Song request queue — member request bài hát như thế nào? Host approve/reject ra sao?

**Trả lời:**

**Member request bài** — từ `SongSearchPanel` (tab "Yêu cầu nhạc"):
```tsx
<Button
  loading={requestingSongId === song._id}
  onClick={() => handleRequestSong(song._id)}
>
  Gửi yêu cầu
</Button>
```

`handleRequestSong` emit WebSocket event với `songId` lên server. Server tạo `RoomQueueItem` với status `PENDING`, broadcast về room.

**Host approve/reject** — từ `QueueItemCard` trong tab "Danh sách yêu cầu":
```tsx
{isHost ? (
  <div className="flex gap-2">
    <Button
      icon={<CheckOutlined />}
      onClick={() => handleResolveRequest(item._id, RoomQueueItemStatus.APPROVED)}
    />
    <Button
      icon={<CloseOutlined />}
      onClick={() => handleResolveRequest(item._id, RoomQueueItemStatus.REJECTED)}
    />
  </div>
) : (
  <RoomQueueStatusChip status={item.status} /> // Member chỉ xem trạng thái
)}
```

`handleResolveRequest` emit event lên server → server update status → broadcast lại → `visibleQueue` và `requestQueue` được cập nhật trên tất cả clients.

**Phân biệt `visibleQueue` và `requestQueue`:**
- `visibleQueue` — các bài đã APPROVED hoặc đang PLAYING, hiển thị cho tất cả
- `requestQueue` — các bài đang PENDING, chỉ host thấy để duyệt/từ chối

---

### Câu 17: `RoomDetailLeftPanel` — `tabItems` được tính bằng `useMemo`. Khi nào list tabs thay đổi?

**Trả lời:**

```ts
const tabItems = useMemo(() => {
  const baseItems = [
    { key: "info", ... },
    { key: "queue", ... },
    { key: "lyrics", ... },
    { key: "visualizer", ... },
  ];

  if (isHost) {
    baseItems.push({
      key: "settings",
      label: <SettingOutlined /> + "Cài đặt phòng",
      children: <RoomUpdatePanel room={room} />,
    });
  }

  return baseItems;
}, [room, isHost, playbackSeconds, durationSeconds, visibleQueue, requestQueue,
    selectedQueueItemId, selectedQueueItem, audioRef]);
```

Tab "Cài đặt phòng" chỉ xuất hiện khi `isHost = true`. Member không thấy tab này.

**Tại sao dùng `useMemo`:** `tabItems` là mảng object phức tạp chứa JSX. Nếu không memoize, mỗi render của `RoomDetailLeftPanel` sẽ tạo mảng mới → `Tabs` component nhận props mới → re-render toàn bộ tab content. Với `useMemo`, tabs chỉ được rebuild khi các dependency thực sự thay đổi.

Dependency array bao gồm `playbackSeconds` và `durationSeconds` vì `RoomInfoPanel` và `RoomLyricsPanel` cần giá trị này để hiển thị progress — chúng update thường xuyên (mỗi 250ms khi đang play), nên tabs vẫn re-render theo. Đây là trade-off có thể optimize thêm bằng cách tách `playbackSeconds` ra khỏi dependency và dùng ref.

---

### Câu 18: `RoomDetailRightPanel` — tại sao tab structure khác nhau giữa host và member?

**Trả lời:**

```ts
const items = isHost
  ? [
      { key: "comments", ... },    // Chat
      { key: "members", ... },     // Quản lý thành viên (kick/ban)
      { key: "add-song", ... },    // Thêm bài trực tiếp vào queue
    ]
  : [
      { key: "comments", ... },    // Chat
      { key: "request", ... },     // Gửi yêu cầu bài hát
    ];
```

Host có thêm:
- **Tab "Thành viên"** — xem danh sách participant, kick/ban user vi phạm
- **Tab "Thêm bài"** — thêm bài trực tiếp vào queue với label "Thêm" (không cần approval)

Member có tab "Yêu cầu nhạc" — search và gửi request với label "Gửi yêu cầu" (cần host approve).

Cả host và member đều dùng cùng `SongSearchPanel` component, chỉ khác `actionLabel` và `placeholder`:
```tsx
<SongSearchPanel
  actionLabel={isHost ? "Thêm" : "Gửi yêu cầu"}
  placeholder={isHost ? "Tìm bài hát..." : "Yêu cầu bài hát..."}
  emptyTitle={isHost ? "Nhập tên bài hát để thêm vào hàng đợi." : "Gửi yêu cầu bài hát cho chủ phòng."}
/>
```

---

## PHẦN 5 — PLAYER STORE & SONG QUEUE

---

### Câu 19: `SongBar` — tại sao cần `useEffect` riêng để đảm bảo audio play khi source thay đổi?

**Trả lời:**

```ts
useEffect(() => {
  if (isPlaying && audioRef && audioSource) {
    const playPromise = audioRef.play();
    if (playPromise !== undefined) {
      playPromise.catch((err) => {
        console.warn("Autoplay was prevented or interrupted:", err);
      });
    }
  }
}, [audioSource, isPlaying, audioRef]);
```

Khi user skip sang bài mới, `audioSource` (mp3 URL) thay đổi → `AudioPlayer` load source mới. `autoPlay={isPlaying}` prop của `react-h5-audio-player` đôi khi không đủ để trigger play sau khi source change — do browser autoplay policy hoặc race condition giữa src load và play().

Effect này là safety net: khi `audioSource` thay đổi và `isPlaying = true`, gọi `audioRef.play()` trực tiếp. `.catch()` xử lý trường hợp browser block autoplay (thường xảy ra khi chưa có user gesture trong session).

---

### Câu 20: `SongQueue` — `playFromQueue` hoạt động như thế nào? Tại sao logic ở server thay vì client?

**Trả lời:**

```ts
const { mutate: playFromQueue } = usePlayFromQueue();

const handlePlayFromQueue = (songId: string) => {
  if (isPending) return;
  playFromQueue({ songId }); // Gửi lên server, không tự xử lý ở client
};
```

Thay vì client tự cắt queue và update state, em gửi `songId` lên server và để server xử lý. Server biết toàn bộ queue state, tính toán phần queue còn lại sau bài được chọn, trả về `status` mới (nowPlayingId, queue mới, history).

**Tại sao server-side:** Queue có thể được nhiều device/tab cùng một user access. Nếu xử lý ở client, state sẽ diverge giữa các tab. Server là source of truth duy nhất. Sau khi mutation thành công, server trả về `PlayerStatus` mới → `setNowPlaying(fullStatus)` → Zustand update → `SongBar` nhận bài mới.

---

## PHẦN 6 — TRADE-OFFS & LESSONS LEARNED

---

### Câu 21: WaveSurfer chỉ init khi bài đang phát — có edge case nào không? Em xử lý như thế nào?

**Trả lời:**

Edge case chính: **Khi URL thay đổi nhưng `shouldInitWave` không thay đổi.**

Trong `useEffect` khởi tạo WaveSurfer:
```ts
useEffect(() => {
  if (!shouldInitWave || !waveformRef.current) return;

  if (waveformRef.current) {
    waveformRef.current.innerHTML = ''; // Clear canvas cũ trước khi tạo mới
  }

  const ws = WaveSurfer.create({ container: waveformRef.current, url, ... });
  // ...
  return () => { ws.destroy(); };
}, [url, shouldInitWave]); // url trong dependency
```

`url` trong dependency array đảm bảo WaveSurfer được destroy và tạo lại khi bài mới được phát (URL thay đổi). `waveformRef.current.innerHTML = ''` clear DOM container để tránh WaveSurfer append canvas mới vào container đã có canvas cũ.

Một edge case khác: `waveDuration` có thể là 0 khi WaveSurfer chưa fire `ready` event. Guard `waveDuration > 0` trước khi render comment markers xử lý điều này.

---

### Câu 22: Nếu làm lại NovaWave, em sẽ thay đổi gì trong thiết kế `usePlayerStore`?

**Trả lời:**

Nhìn lại, có 3 điểm em muốn cải thiện:

**1. Không lưu `audioRef` vào Zustand:** DOM reference trong Zustand là anti-pattern. Tốt hơn nên dùng React Context hoặc một module singleton riêng (`audioManager.ts`) để expose audio element. Zustand nên chỉ chứa serializable state.

**2. Tách `currentTime` ra khỏi persist:** `currentTime` được persist nhưng khi restore sau reload, audio cần seek đến đúng vị trí này. Hiện tại chưa có logic tự động seek sau rehydrate — user phải tự bấm play lại từ đầu bài.

**3. Type strict hơn cho `nowPlaying`:** Hiện tại `nowPlaying` là `any`, chứa cả `Song` lẫn `Advertisement`. Nên dùng discriminated union:
```ts
type NowPlayingState =
  | { type: 'song'; data: Song }
  | { type: 'advertisement'; data: Ad }
  | null;
```
Loại bỏ các check `isCurrentAd` rải rác khắp component, TypeScript có thể narrow type tự động.

---

## PHẦN 7 — WEBSOCKET & REALTIME ROOM

---

### Câu 23: `useRoomSocket` được thiết kế như thế nào? Tách 2 `useEffect` riêng biệt có ý nghĩa gì?

**Trả lời:**

`useRoomSocket` có 2 `useEffect` với trách nhiệm hoàn toàn tách biệt:

**Effect 1 — Join/Leave room (dependency: `roomId`):**
```ts
useEffect(() => {
  if (!roomId) return;
  const socket = getSocket() ?? connectSocket();

  const joinRoom = () => {
    socket.emit("JOIN_ROOM", { roomId });
  };

  if (socket.connected) {
    joinRoom();           // Socket đã connect → join ngay
  } else {
    socket.on("connect", joinRoom); // Chưa connect → đợi connect rồi join
  }

  return () => {
    socket.emit("LEAVE_ROOM", { roomId });
    socket.off("connect", joinRoom);
  };
}, [roomId]);
```

Chạy khi `roomId` thay đổi. Cleanup emit `LEAVE_ROOM` để server biết user rời phòng và cập nhật participant list. `socket.off("connect", joinRoom)` quan trọng để tránh duplicate join nếu effect re-run trước khi socket connect.

**Effect 2 — Register/unregister event handlers (dependency: `handlers`):**
```ts
useEffect(() => {
  const socket = getSocket();
  const handlerEntries = Object.entries(handlers);

  handlerEntries.forEach(([eventName, handler]) => {
    if (handler) socket.on(eventName, handler);
  });

  return () => {
    handlerEntries.forEach(([eventName, handler]) => {
      if (handler) socket.off(eventName, handler);
    });
  };
}, [handlers]);
```

Chạy khi `handlers` object thay đổi — tức là mỗi lần `roomSocketHandlers` được recompute (do `useMemo`). Cleanup off tất cả handlers cũ trước khi on handlers mới.

**Tại sao tách 2 effect:** Join/leave và handler registration là 2 lifecycle khác nhau. Nếu gộp 1 effect với `[roomId, handlers]` làm dependency, mỗi khi `handlers` thay đổi sẽ emit `LEAVE_ROOM` rồi `JOIN_ROOM` lại — gây ra nhấp nháy participant list không cần thiết.

---

### Câu 24: `roomSocketHandlers` được bọc trong `useMemo` với dependency dài. Điều này giải quyết vấn đề gì?

**Trả lời:**

```ts
const roomSocketHandlers = useMemo(
  () => ({
    ROOM_UPDATED: (payload) => { ... },
    ROOM_ENDED: (payload) => { ... },
    QUEUE_UPDATED: (payload) => { ... },
    HOST_CONTROL: (payload) => { ... },
    RECEIVE_MESSAGE: (payload) => { ... },
    USER_JOINED: (payload) => { ... },
    USER_LEFT: (payload) => { ... },
    NEW_REQUEST_NOTIFICATION: (payload) => { ... },
    PARTICIPANT_MODERATED: (payload) => { ... },
    PLAYER_STATE_SYNC: (payload) => { ... },
  }),
  [currentUser?.sub, handleResolveRequest, isHost, notifyApi, roomId, queryClient]
);
```

**Vấn đề nếu không dùng `useMemo`:** Mỗi render của `RoomDetailPage`, một object `handlers` mới được tạo ra (reference khác nhau dù value giống nhau). `useRoomSocket` nhận `handlers` mới → `useEffect` cleanup cũ (off tất cả) → register lại (on tất cả). Điều này xảy ra liên tục khi bất kỳ state nào ở page thay đổi (ví dụ `playbackSeconds` update mỗi 250ms).

Với `useMemo`, handlers object chỉ được tạo lại khi các dependency thực sự thay đổi — `isHost`, `currentUser?.sub`, `roomId`... Giảm drastically số lần `socket.off/on` không cần thiết.

---

### Câu 25: `emitHostControl` hoạt động như thế nào? Tại sao `currentTime` gửi lên server theo đơn vị milliseconds?

**Trả lời:**

```ts
const emitHostControl = useCallback((
  action: RoomControlAction,
  extra?: { currentTime?: number; currentSongId?: string; currentQueueItemId?: string }
) => {
  if (!room || !isHost) return;  // Guard: chỉ host mới được emit
  const socket = getSocket();
  if (!socket) { toast.error("Socket chưa sẵn sàng"); return; }

  socket.emit("HOST_CONTROL", {
    roomId,
    action,
    currentTime: extra?.currentTime,
    currentSongId: extra?.currentSongId,
    currentQueueItemId: extra?.currentQueueItemId,
  });

  queryClient.invalidateQueries({ queryKey: [ROOM_QUERY_KEY, roomId] });
}, [isHost, room, roomId, toast]);
```

**Tại sao milliseconds:** Server lưu `playbackPositionMs` (xem `room.playbackPositionMs` trong sync effect). Milliseconds cho độ chính xác cao hơn khi tính toán compensation latency. Ví dụ host pause lúc giây 1:23.456 → server nhận 83456ms → khi broadcast cho members, server có thể cộng thêm network latency (vài chục ms) để members bắt đầu đúng vị trí.

Cách convert khi gửi: `Math.round(audioRef.current?.currentTime * 1000)` — lấy giây từ audio element, nhân 1000, làm tròn.

`invalidateQueries` sau emit đảm bảo TanStack Query refetch `roomDetail` để sync state với DB sau khi server xử lý control action.

---

### Câu 26: `syncingAudioRef` dùng để làm gì? Tại sao các player handlers đều check flag này?

**Trả lời:**

```ts
const syncingAudioRef = useRef(false);
```

`syncingAudioRef` là flag ngăn **feedback loop** giữa server sync và player events.

**Vấn đề không có flag:** Khi server broadcast `HOST_CONTROL { action: PLAY }`, client nhận → sync effect chạy → `audio.play()` được gọi → AudioPlayer fire event `onPlay` → `handlePlayerPlay` được gọi → emit `HOST_CONTROL PLAY` lên server lại → vòng lặp vô hạn.

**Giải pháp:**
```ts
// Trong sync effect — set flag trước khi programmatically control audio
syncingAudioRef.current = true;
audio.play()
  .catch(() => undefined)
  .finally(() => {
    window.setTimeout(() => {
      syncingAudioRef.current = false; // Reset sau khi play xong
    }, 0);  // setTimeout(0) đảm bảo reset sau khi các event handlers đã chạy
  });

// Trong player handlers — skip nếu đang sync
const handlePlayerPlay = useCallback(() => {
  if (!isHost || syncingAudioRef.current || !room) return; // ← Guard
  emitHostControl(RoomControlAction.PLAY, { ... });
}, [...]);
```

`setTimeout(0)` reset flag sau khi call stack hiện tại xử lý xong — đảm bảo `onPlay` event của AudioPlayer (do `audio.play()`) đã được xử lý và bị guard trước khi flag được reset.

---

### Câu 27: Sync audio khi nhận `HOST_CONTROL` hoặc room state thay đổi — logic cụ thể như thế nào?

**Trả lời:**

Effect sync chính trong `page.tsx`:

```ts
useEffect(() => {
  if (!room?.currentSong?.mp3Link || !audioRef.current) return;
  const audio = audioRef.current;
  const positionSeconds = Math.max(0, (room.playbackPositionMs ?? 0) / 1000);

  syncingAudioRef.current = true;

  // 1. Đổi source nếu bài mới
  if (audio.src !== room.currentSong.mp3Link) {
    audio.src = room.currentSong.mp3Link;
    audio.load();
  }

  // 2. Seek nếu lệch hơn 1.5 giây (tránh seek liên tục)
  if (Math.abs(audio.currentTime - positionSeconds) > 1.5) {
    audio.currentTime = positionSeconds;
  }

  setPlaybackSeconds(positionSeconds);
  setDurationSeconds(room.currentSong.duration ?? 0);

  // 3. Play hoặc pause theo room.isPlaying
  if (room.isPlaying) {
    void audio.play().catch(() => undefined).finally(releaseSyncFlag);
  } else {
    audio.pause();
    releaseSyncFlag();
  }
}, [room?.currentSong?.duration, room?.currentSong?.mp3Link,
    room?.isPlaying, room?.playbackPositionMs]);
```

**Ngưỡng 1.5 giây:** Chỉ seek khi lệch hơn 1.5s để tránh seek liên tục khi `playbackPositionMs` update nhỏ. Nếu member vừa join, họ có thể lệch vài giây → seek. Nếu chỉ lệch 0.2s do latency nhỏ → không seek, tránh giật.

**`room.playbackPositionMs` được update từ đâu:** Khi server broadcast `HOST_CONTROL` event, handler trong `roomSocketHandlers` gọi `mergeRoomState(prev, payload.room)` — `payload.room` chứa `playbackPositionMs` mới. Effect này re-run → audio sync.

---

### Câu 28: `HOST_CONTROL` và `PLAYER_STATE_SYNC` — hai event này khác nhau gì?

**Trả lời:**

Từ `roomSocketHandlers`:

**`HOST_CONTROL`** — broadcast ngay khi host thực hiện action (PLAY/PAUSE/SEEK/NEXT):
```ts
HOST_CONTROL: (payload: { action: RoomControlAction; room: Partial<RoomDetail>; queueItem?: RoomQueueItem | null }) => {
  setRoom((prev) => {
    const nextQueue = payload.queueItem
      ? upsertQueueItem(prev.queue, payload.queueItem)
      : prev.queue;
    return mergeRoomState(prev, { ...payload.room, queue: nextQueue });
  });
  setActivityFeed((prev) => [
    createRealtimeMessage("playerSync", `Chủ phòng vừa thực hiện lệnh ${payload.action}.`),
    ...prev,
  ].slice(0, 30));
},
```
Chứa full `room` state mới + `queueItem` nếu có thay đổi. Cập nhật ngay lập tức để members thấy trạng thái mới.

**`PLAYER_STATE_SYNC`** — server chủ động broadcast định kỳ để đảm bảo các member vừa join hoặc bị lag được sync:
```ts
PLAYER_STATE_SYNC: (payload: Partial<RoomDetail>) => {
  setRoom((prev) => mergeRoomState(prev, payload));
  queryClient.setQueryData([ROOM_QUERY_KEY, roomId], (old: any) => ({
    ...old, ...payload,
  }));
},
```
Nhẹ hơn `HOST_CONTROL` — chỉ cập nhật `currentSongId`, `isPlaying`, `playbackPositionMs`. Cũng update TanStack Query cache trực tiếp để các component query `roomDetail` nhận được state mới ngay mà không cần refetch.

**Tóm lại:** `HOST_CONTROL` = triggered by action, `PLAYER_STATE_SYNC` = periodic heartbeat từ server.

---

### Câu 29: `emitComment` có optimistic update. Tại sao có file code cũ bị comment out? Sự khác biệt giữa 2 approach là gì?

**Trả lời:**

File có cả 2 approach được comment rõ ràng:

**Approach cũ (bị comment):** Gửi API → chờ server trả về → update UI:
```ts
// const emitComment = async () => {
//   const response = await createMessage({ id: roomId, data: { content } });
//   const payload = response.data?.data as RoomMessage;
//   if (payload?._id) {
//     setMessages((prev) => upsertMessage(prev, payload).slice(0, 50));
//   }
//   setCommentInput("");
// };
```
User bấm gửi → input bị lock → chờ network → message xuất hiện. Nếu mạng chậm, cảm giác laggy.

**Approach mới (optimistic):**
```ts
const emitComment = async () => {
  const content = commentInput.trim();

  // 1. Hiển thị ngay với temp ID
  const tempMessage: RoomMessage = {
    _id: `temp_${Date.now()}`,
    content,
    userId: currentUser?.sub ?? "",
    roomId,
    createdAt: new Date().toISOString(),
    updatedAt: new Date().toISOString(),
  };
  setMessages((prev) => upsertMessage(prev, tempMessage).slice(0, 50));
  setCommentInput(""); // Clear input ngay

  try {
    const response = await createMessage({ id: roomId, data: { content } });
    const payload = response.data?.data as RoomMessage;
    // 2. Replace temp message bằng message thật (có _id thật từ DB)
    if (payload?._id) {
      setMessages((prev) =>
        prev.map(m => m._id === tempMessage._id ? payload : m)
      );
    }
  } catch (error) {
    // 3. Rollback nếu fail
    setMessages((prev) => prev.filter(m => m._id !== tempMessage._id));
    toast.error("Không thể gửi bình luận");
    setCommentInput(content); // Restore input
  }
};
```

Message xuất hiện ngay lập tức với `_id: "temp_1234"`. Sau khi server confirm, temp message được replace bằng message thật có `_id` từ DB. Nếu fail → remove temp message + restore input content để user không mất nội dung đã gõ.

---

### Câu 30: `NEW_REQUEST_NOTIFICATION` — tại sao host nhận notification dạng Ant Design `notifyApi` thay vì toast thông thường?

**Trả lời:**

```ts
NEW_REQUEST_NOTIFICATION: (payload: RoomQueueItem) => {
  // 1. Update queue state cho tất cả
  setRoom((prev) => ({ ...prev, queue: upsertQueueItem(prev.queue, payload) }));

  // 2. Chỉ host mới thấy popup notification với action buttons
  if (isHost) {
    notifyApi.info({
      key: payload._id,  // Key = queueId để destroy đúng notification
      duration: 10,       // Tự dismiss sau 10 giây
      description: (
        <div>
          {/* Song info card */}
          <Button onClick={() => {
            handleResolveRequest(payload._id, RoomQueueItemStatus.APPROVED);
            notifyApi.destroy(payload._id); // Dismiss sau khi action
          }}>✓ Chấp nhận</Button>
          <Button onClick={() => {
            handleResolveRequest(payload._id, RoomQueueItemStatus.REJECTED);
            notifyApi.destroy(payload._id);
          }}>✕ Từ chối</Button>
        </div>
      ),
    });
  }
},
```

**Lý do dùng `notifyApi` thay vì toast:** Notification chứa **interactive buttons** — host có thể approve/reject ngay từ popup mà không cần mở tab "Danh sách yêu cầu". Toast thông thường chỉ hiển thị text, không chứa được action buttons phức tạp.

`key: payload._id` — nếu member gửi nhiều request, mỗi request có notification riêng. Sau khi host action, `notifyApi.destroy(payload._id)` dismiss đúng notification đó. `duration: 10` — tự dismiss sau 10 giây nếu host không action.

---

### Câu 31: `PARTICIPANT_MODERATED` — khi bị kick/ban, user được xử lý như thế nào ở phía client?

**Trả lời:**

```ts
PARTICIPANT_MODERATED: (payload: { userId: string; action: RoomModerationAction; reason?: string }) => {
  // 1. Remove khỏi participants list (mọi client đều nhận)
  setParticipants((prev) => prev.filter(
    (item) => getUserId(item.userId) !== payload.userId
  ));

  // 2. Nếu MÌNH bị moderate → hiện màn hình block
  if (payload.userId === currentUser?.sub) {
    const nextStatus = payload.action === RoomModerationAction.BAN
      ? RoomParticipantStatus.BANNED
      : RoomParticipantStatus.KICKED;

    setModerationState(nextStatus);
    setModerationReason(payload.reason || "");

    // Persist vào localStorage để block ngay cả khi reload
    window.localStorage.setItem(
      `${moderationStoragePrefix}${roomId}`,
      JSON.stringify({ status: nextStatus, reason: payload.reason || "" })
    );
  }
},
```

Khi `moderationState` được set, page render màn hình block thay vì room:
```tsx
if (moderationState === RoomParticipantStatus.KICKED ||
    moderationState === RoomParticipantStatus.BANNED) {
  return <BlockedScreen reason={moderationReason} />;
}
```

**localStorage persistence:** Khi user bị ban rồi reload trang, `useEffect` đọc lại từ localStorage:
```ts
useEffect(() => {
  const stored = window.localStorage.getItem(`${moderationStoragePrefix}${roomId}`);
  if (stored) {
    const parsed = JSON.parse(stored);
    setModerationState(parsed.status);
    setModerationReason(parsed.reason || "");
  }
}, [roomId]);
```
User không thể bypass ban bằng cách F5. Key theo `roomId` để ban ở phòng này không ảnh hưởng phòng khác.

---

### Câu 32: `upsertMessage`, `upsertParticipant`, `upsertQueueItem` — tại sao cần các helper này thay vì push thẳng?

**Trả lời:**

Đây là **upsert pattern** — insert nếu chưa có, update nếu đã có — giải quyết vấn đề duplicate khi cùng event đến nhiều lần.

Ví dụ với `upsertMessage`:
```ts
// room-detail-helpers.ts (inferred từ usage)
export const upsertMessage = (list: RoomMessage[], newItem: RoomMessage) => {
  const exists = list.find(m => m._id === newItem._id);
  if (exists) {
    return list.map(m => m._id === newItem._id ? newItem : m);
  }
  return [...list, newItem];
};
```

**Tại sao cần:** Socket events có thể đến duplicate do:
- Network retry khi connection bị ngắt
- React Strict Mode chạy effect 2 lần
- Race condition giữa initial HTTP fetch và socket event

Nếu dùng `setMessages(prev => [...prev, payload])` thẳng, message sẽ bị duplicate trong list. Upsert đảm bảo list luôn unique theo `_id`.

Cụ thể với `RECEIVE_MESSAGE`:
```ts
RECEIVE_MESSAGE: (payload: RoomMessage) => {
  setMessages((prev) => upsertMessage(prev, payload).slice(0, 50));
}
```
`.slice(0, 50)` giới hạn memory: không giữ quá 50 messages trong state, tránh memory leak cho session dài.

Tương tự, `upsertQueueItem` cần thiết khi `QUEUE_UPDATED` và `REQUEST_UPDATED` có thể broadcast cùng một queue item với status khác nhau — cần update tại đúng vị trí thay vì append.

---

*Tổng cộng: 32 câu hỏi & trả lời cho NovaWave*
*File nguồn: `usePlayerStore.ts`, `wave-player.tsx`, `song-bar.tsx`, `song-comment.tsx`, `song-info.tsx`, `song-queue.tsx`, `room-detail-left-panel.tsx`, `room-detail-right-panel.tsx`, `room-song-bar.tsx`, `room-visualizer-panel.tsx`, `room-comment-panel.tsx`, `room-member-panel.tsx`, `room-info-panel.tsx`, `room-queue-panel.tsx`, `room-lyrics-panel.tsx`, `room-song-search-panel.tsx`, `room-update-panel.tsx`, `comment-swiper.tsx`, `page.tsx` (room detail), `useRoomSocket.ts`*
