# 🎵 NOVAWAVE — Luồng Phòng Nghe Nhạc Chung (Shared Listening Room)

> Tài liệu mô tả toàn bộ luồng hoạt động của tính năng **Phòng nghe nhạc chung** từ Frontend (Next.js) đến Backend (NestJS), bao gồm REST API và WebSocket (Socket.io).

---

## 📁 Cấu trúc file liên quan

### Backend — `novawave_be/src/modules/rooms/`

| File | Vai trò |
|---|---|
| `room.module.ts` | Khai báo module, inject dependencies |
| `controllers/room.controller.ts` | REST API Controller — xử lý HTTP request |
| `gateways/room.gateway.ts` | WebSocket Gateway — xử lý real-time events |
| `services/room.service.ts` | Business logic chính của toàn bộ tính năng |
| `repositories/room.repository.ts` | CRUD trực tiếp với MongoDB collection `rooms` |
| `repositories/room-queue.repository.ts` | CRUD collection `roomqueueitems` |
| `repositories/room-participant.repository.ts` | CRUD collection `roomparticipants` |
| `repositories/room-message.repository.ts` | CRUD collection `roommessages` |
| `schemas/room.schema.ts` | Mongoose schema — document Room |
| `schemas/room-queue-item.schema.ts` | Mongoose schema — document hàng đợi bài hát |
| `schemas/room-participant.schema.ts` | Mongoose schema — document người tham gia |
| `schemas/room-message.schema.ts` | Mongoose schema — document chat message |
| `dtos/create-room.dto.ts` | DTO tạo phòng |
| `dtos/update-room.dto.ts` | DTO cập nhật phòng |
| `dtos/room-control.dto.ts` | DTO điều khiển playback (PLAY/PAUSE/NEXT…) |
| `dtos/sync-room-playback.dto.ts` | DTO đồng bộ trạng thái phát |
| `dtos/add-room-queue-item.dto.ts` | DTO thêm bài hát vào hàng đợi |
| `dtos/update-room-queue-item.dto.ts` | DTO duyệt / từ chối yêu cầu bài hát |
| `dtos/create-room-message.dto.ts` | DTO gửi tin nhắn |
| `dtos/moderate-room-participant.dto.ts` | DTO kick/ban thành viên |
| `dtos/query-room.dto.ts` | DTO query danh sách phòng |

### Frontend — `novawave_fe/src/`

| File | Vai trò |
|---|---|
| `app/(client)/room/page.tsx` | Trang danh sách phòng |
| `app/(client)/room/createRoom/` | Trang tạo phòng mới |
| `app/roomDetail/[id]/page.tsx` | **Trang chính** — container toàn bộ phòng nghe nhạc |
| `app/roomDetail/[id]/room-detail-helpers.ts` | Utility functions: merge state, upsert queue/participant/message |
| `app/roomDetail/[id]/left-panel/room-detail-left-panel.tsx` | Panel trái — player, queue, info |
| `app/roomDetail/[id]/left-panel/room-info-panel.tsx` | Thông tin phòng, host controls |
| `app/roomDetail/[id]/left-panel/room-queue-panel.tsx` | Danh sách hàng đợi bài hát |
| `app/roomDetail/[id]/left-panel/room-song-bar.tsx` | Thanh phát nhạc (audio player) |
| `app/roomDetail/[id]/left-panel/room-visualizer-panel.tsx` | Visualizer âm nhạc |
| `app/roomDetail/[id]/left-panel/room-lyrics-panel.tsx` | Hiển thị lời bài hát |
| `app/roomDetail/[id]/left-panel/room-update-panel.tsx` | Panel chỉnh sửa phòng (host) |
| `app/roomDetail/[id]/right-panel/room-detail-right-panel.tsx` | Panel phải — chat, thành viên, yêu cầu bài |
| `app/roomDetail/[id]/right-panel/room-comment-panel.tsx` | Chat bình luận real-time |
| `app/roomDetail/[id]/right-panel/room-member-panel.tsx` | Danh sách thành viên + moderation |
| `app/roomDetail/[id]/right-panel/room-song-search-panel.tsx` | Tìm kiếm & yêu cầu bài hát |
| `hooks/useRoomSocket.ts` | Custom hook — kết nối socket & đăng ký event listeners |
| `libs/socket.ts` | Socket.io client singleton — `connectSocket`, `getSocket`, `disconnectSocket` |
| `queries/useRoomQuery.ts` | React Query hooks — `useRoomDetail`, `useRoomMessages`, `useAddRoomQueueItem`… |
| `services/room.service.ts` | HTTP service layer — axios calls đến REST API |
| `libs/http.ts` | Axios instance với interceptor (auth token) |

---

## 🏗️ Kiến trúc tổng quan

```
┌─────────────────────────────────────────────────────────┐
│                    FRONTEND (Next.js)                   │
│                                                         │
│  [Trang danh sách]  ──→  [Trang chi tiết phòng]        │
│  /room/page.tsx          /roomDetail/[id]/page.tsx      │
│                                    │                    │
│          ┌─────────────────────────┴──────────────┐    │
│          │            page.tsx (container)         │    │
│          │  - React state: room, messages,         │    │
│          │    participants, queue, activityFeed     │    │
│          │  - useRoomSocket() → lắng nghe events   │    │
│          │  - useRoomQuery hooks → REST API         │    │
│          └──────────────┬──────────────────────────┘    │
│                         │                              │
│         ┌───────────────┴──────────────┐               │
│         ▼                              ▼               │
│   LeftPanel                      RightPanel            │
│   (Player, Queue, Info)          (Chat, Members,       │
│                                   Song Request)        │
│                                                         │
│  ┌─────────────────┐  ┌──────────────────────────┐    │
│  │  libs/socket.ts │  │ services/room.service.ts │    │
│  │  (Socket.io)    │  │ (axios HTTP calls)        │    │
│  └────────┬────────┘  └─────────────┬────────────┘    │
└───────────┼──────────────────────────┼─────────────────┘
            │ WebSocket                │ REST HTTP
            ▼                          ▼
┌─────────────────────────────────────────────────────────┐
│                    BACKEND (NestJS)                     │
│                                                         │
│  ┌──────────────────┐    ┌─────────────────────────┐  │
│  │  room.gateway.ts │    │  room.controller.ts     │  │
│  │  (WebSocket)     │    │  (REST /rooms/*)        │  │
│  └────────┬─────────┘    └────────────┬────────────┘  │
│           └──────────────┬────────────┘                │
│                          ▼                             │
│               ┌──────────────────────┐                 │
│               │  room.service.ts     │                 │
│               │  (Business Logic)    │                 │
│               └────────┬─────────────┘                 │
│                        │                               │
│         ┌──────────────┼──────────────────┐            │
│         ▼              ▼                  ▼            │
│   RoomRepository  QueueRepository  ParticipantRepo     │
│         │              │                  │            │
│         └──────────────┴──────────────────┘            │
│                        │                               │
│                   MongoDB Atlas                        │
│         rooms | roomqueueitems | roomparticipants      │
│                   | roommessages                       │
└─────────────────────────────────────────────────────────┘
```

---

## 🔌 Kết nối WebSocket

**File:** `novawave_fe/src/libs/socket.ts`

```typescript
// Kết nối socket với JWT token từ cookie/sessionStorage/localStorage
const socket = io(NEXT_PUBLIC_SOCKET_URL, {
  transports: ["websocket"],
  auth: { token }
});
```

**File:** `novawave_be/src/modules/rooms/gateways/room.gateway.ts`

```typescript
// BE xác thực JWT ngay khi client kết nối
handleConnection(client: Socket) {
  const token = client.handshake.auth?.token;
  const payload = jwtService.verify(token, { secret });
  client.data.userId = payload.sub;          // lưu userId vào socket
  client.join(`host_${payload.sub}`);        // tham gia private channel của user
}
```

**Luồng:**
1. FE gọi `connectSocket()` → Socket.io kết nối đến BE
2. BE verify JWT trong `handleConnection` → lưu `userId` vào `client.data`
3. Nếu token không hợp lệ → `client.disconnect()`

---

## 🚀 Luồng 1 — Tạo phòng

```
FE: /room/createRoom → POST /rooms
        │
        ▼
BE: RoomController.create()
        │
        ▼
BE: RoomService.create()
  1. Kiểm tra user có phòng đang active không (chỉ 1 phòng mỗi lúc)
  2. resolveInitialSource() → lấy danh sách songIds từ:
     - initialSongId (1 bài)
     - playlistId (từ PlaylistService)
     - albumId (từ AlbumService + SongService)
  3. Tạo Room document (status = STREAMING nếu không lên lịch)
  4. Tạo RoomQueueItem[] cho từng bài hát (item[0] = PLAYING)
  5. Tạo RoomParticipant cho host (role = HOST, status = ACTIVE)
  6. Cập nhật room.currentSongId, room.currentQueueItemId
  7. Trả về getDetail() (room + queue + participants + currentSong)
```

**Files liên quan:**
- `dtos/create-room.dto.ts` — `{ name, description, imageUrl, scheduledAt?, initialSongId | playlistId | albumId }`
- `schemas/room.schema.ts` — MongoDB document Room
- `queries/useRoomQuery.ts` — `useCreateRoom()`

---

## 🚀 Luồng 2 — Vào trang phòng & Join

```
FE: Navigate to /roomDetail/[id]
        │
        ├─ 1. HTTP: GET /rooms/:id           → useRoomDetail()
        │         Lấy thông tin phòng, queue, participants, currentSong
        │
        ├─ 2. HTTP: GET /rooms/:id/messages  → useRoomMessages()
        │         Lấy lịch sử chat (page=1, size=30)
        │
        ├─ 3. HTTP: GET /rooms/:id/participants → useRoomParticipants()
        │         Lấy danh sách thành viên active
        │
        └─ 4. WebSocket: useRoomSocket(roomId, handlers)
                  │
                  ├─ emit('JOIN_ROOM', { roomId })
                  │         ↓
                  │   BE: RoomGateway.joinRoom()
                  │         ↓
                  │   BE: RoomService.join()
                  │     - Tạo/cập nhật RoomParticipant (status = ACTIVE)
                  │     - Tăng room.participantCount
                  │     - broadcastUserJoined() → 'USER_JOINED' đến room channel
                  │
                  └─ Đăng ký listeners cho toàn bộ events real-time
```

**Files liên quan:**
- `app/roomDetail/[id]/page.tsx` — container chính
- `hooks/useRoomSocket.ts` — quản lý socket lifecycle
- `services/room.service.ts` — HTTP calls
- `schemas/room-participant.schema.ts`

---

## 🎵 Luồng 3 — Phát nhạc (Audio Sync)

### 3A. Host điều khiển player

```
Host tương tác với audio element (play/pause/seek/ended)
        │
        ▼
page.tsx: handlePlayerPlay/Pause/Seek/Ended()
        │
        ├─ Kiểm tra isHost === true
        ├─ Kiểm tra syncingAudioRef.current === false
        │
        ▼
emitHostControl(action, { currentTime, currentSongId, currentQueueItemId })
        │
        ▼
socket.emit('HOST_CONTROL', { roomId, action, ... })
        │
        ▼
BE: RoomGateway.hostControl() → RoomService.handleHostControl()
  ├─ PLAY   → syncPlayback({ isPlaying: true, currentTime, startedAt: now })
  ├─ PAUSE  → syncPlayback({ isPlaying: false, currentTime })
  ├─ SEEK   → syncPlayback({ currentTime })
  ├─ NEXT   → playNext()
  │     - Đánh dấu bài hiện tại = PLAYED
  │     - Tìm bài tiếp theo trong queue (APPROVED)
  │     - Cập nhật room.currentSongId, room.currentQueueItemId
  │     - broadcastHostControl() + broadcastQueueUpdated()
  └─ END    → update room status = ENDED → broadcastRoomEnded()
        │
        ▼
BE: syncPlayback() → cập nhật isPlaying, playbackPositionMs, status
  → broadcastPlayerSync() → emit 'PLAYER_STATE_SYNC' đến room channel
        │
        ▼
FE (All): Handler 'PLAYER_STATE_SYNC'
  - setRoom(mergeRoomState(prev, payload))
  - queryClient.setQueryData(...)
        │
        ▼
useEffect([room.isPlaying, room.currentSong.mp3Link, room.playbackPositionMs])
  - Sync audio element: src, currentTime, play/pause
```

### 3B. Listener nhận sync (tự động)

```
FE (Listener): Nhận 'PLAYER_STATE_SYNC' qua socket
        │
        ▼
setRoom(prev => mergeRoomState(prev, payload))
        │
        ▼
useEffect trigger → audioRef.current:
  - Nếu src khác → load bài mới
  - Nếu |currentTime - positionSeconds| > 1.5s → seek
  - isPlaying → audio.play() / audio.pause()
```

**Files liên quan:**
- `app/roomDetail/[id]/page.tsx` — `handlePlayerPlay/Pause/Seek/Ended`, audio sync useEffect
- `left-panel/room-song-bar.tsx` — UI thanh phát nhạc
- `dtos/room-control.dto.ts`
- `dtos/sync-room-playback.dto.ts`

---

## 📋 Luồng 4 — Hàng đợi bài hát (Queue)

### 4A. Host thêm bài (tự động approved)

```
FE: Host chọn bài → handleRequestSong(songId)
        │
        ▼
useAddRoomQueueItem().mutate({ id: roomId, data: { songId } })
  → HTTP: POST /rooms/:id/queue
        │
        ▼
BE: RoomService.addQueueItem()
  - isHost === true → status = APPROVED, approvedBy = userId
  - Tạo RoomQueueItem
  - broadcastQueueUpdated() → 'QUEUE_UPDATED'
```

### 4B. Listener yêu cầu bài hát (cần duyệt)

```
FE: Listener → room-song-search-panel → handleRequestSong(songId)
  → HTTP: POST /rooms/:id/queue
        │
        ▼
BE: RoomService.addQueueItem()
  - Giới hạn: MAX_PENDING_REQUESTS_PER_ROOM = 100
  - Giới hạn: MAX_PENDING_REQUESTS_PER_USER = 5
  - isHost === false → status = PENDING
  - notifyNewRequest() → 'NEW_REQUEST_NOTIFICATION'
        │
        ▼
FE (Host): Handler 'NEW_REQUEST_NOTIFICATION'
  - upsertQueueItem + setActivityFeed
  - Hiện antd notification popup với nút Chấp nhận / Từ chối
```

### 4C. Host duyệt / từ chối yêu cầu

```
FE (Host): Click Chấp nhận/Từ chối
  → useUpdateRoomQueueItem()
  → HTTP: PATCH /rooms/:id/queue/:queueId
        │
        ▼
BE: RoomService.updateQueueItem()
  - assertHost() — chỉ host được duyệt
  - Cập nhật status = APPROVED/REJECTED
  - broadcastRequestResolved() → 'REQUEST_UPDATED'
  - Nếu APPROVED → broadcastQueueUpdated() → 'QUEUE_UPDATED'
```

**Files liên quan:**
- `right-panel/room-song-search-panel.tsx` — UI tìm kiếm & request
- `left-panel/room-queue-panel.tsx` — UI hiển thị hàng đợi
- `repositories/room-queue.repository.ts`
- `schemas/room-queue-item.schema.ts`

---

## 💬 Luồng 5 — Chat real-time

```
FE: User gõ bình luận → emitComment()
        │
        ├─ Optimistic update: Tạo tempMessage, setMessages() ngay lập tức
        │   (hiển thị với _id = `temp_${Date.now()}`)
        │
        ▼
HTTP: POST /rooms/:id/messages
        │
        ▼
BE: RoomService.createMessage()
  - Cache check: `room_active_user:{roomId}:{userId}` (TTL=5 phút)
  - Nếu không có cache → ensureActiveParticipant() query DB
  - MongoDB Transaction: lưu RoomMessage
  - broadcastMessage() → 'RECEIVE_MESSAGE'
  - Redis: zincrby('rooms:activity', 1, roomId) — tracking hoạt động
        │
        ▼
FE: Handler 'RECEIVE_MESSAGE'
  - upsertMessage: thay tempMessage bằng message thật từ server
  - Nếu HTTP thất bại → rollback: filter tempMessage ra khỏi state
```

> **Throttle:** BE giới hạn `POST /rooms/:id/messages` — 1 request / 2 giây

**Files liên quan:**
- `right-panel/room-comment-panel.tsx` — UI chat
- `schemas/room-message.schema.ts`
- `repositories/room-message.repository.ts`

---

## 👥 Luồng 6 — Quản lý thành viên (Moderation)

### 6A. Kick / Ban thành viên

```
FE (Host): room-member-panel.tsx
  → handleModerate(participantUserId, KICK | BAN)
  → HTTP: POST /rooms/:id/participants/:userId/moderation
        │
        ▼
BE: RoomService.moderateParticipant()
  - assertHost() — chỉ host
  - Không thể moderate chính host
  - Cập nhật status = KICKED | BANNED, leftAt, moderatedBy
  - Giảm participantCount nếu đang ACTIVE
  - broadcastParticipantModerated() → 'PARTICIPANT_MODERATED'
        │
        ▼
FE (All): Handler 'PARTICIPANT_MODERATED'
  - Xóa participant khỏi danh sách
  - Nếu payload.userId === currentUser.sub:
    → setModerationState(KICKED | BANNED)
    → localStorage.setItem(`room-moderation:${roomId}`, ...)
    → Render màn hình "Truy cập bị từ chối"
```

### 6B. Rời phòng (Listener)

```
FE: Click "Rời phòng" → confirm → router.push('/room')
        │
        ▼
useRoomSocket cleanup (useEffect return):
  socket.emit('LEAVE_ROOM', { roomId })
        │
        ▼
BE: RoomGateway.leaveRoom() → RoomService.leave()
  - Host không thể rời (phải end phòng)
  - Cập nhật status = LEFT, leftAt
  - Giảm participantCount
  - broadcastUserLeft() → 'USER_LEFT'
```

---

## 🔚 Luồng 7 — Kết thúc phòng

```
FE (Host): Bấm "Kết thúc phòng"
        │
        ▼
emitHostControl(RoomControlAction.END)
  → socket.emit('HOST_CONTROL', { roomId, action: 'END' })
        │
        ▼
BE: handleHostControl() → case END
  → update(roomId, { status: ENDED })
  → broadcastRoomEnded() → 'ROOM_ENDED'
        │
        ▼
FE (All): Handler 'ROOM_ENDED'
  - setRoom(mergeRoomState(prev, { status: ENDED, isPlaying: false }))
  - activityFeed: "Phòng đã kết thúc"
  - Audio tự dừng vì isPlaying = false
```

---

## 📡 Bảng tổng hợp WebSocket Events

### Client → Server (emit)

| Event | Handler BE | Mô tả |
|---|---|---|
| `JOIN_ROOM` | `RoomGateway.joinRoom()` | Tham gia phòng, join socket room channel |
| `LEAVE_ROOM` | `RoomGateway.leaveRoom()` | Rời phòng, leave socket room channel |
| `SEND_MESSAGE` | `RoomGateway.sendMessage()` | Gửi tin nhắn chat qua socket |
| `HOST_CONTROL` | `RoomGateway.hostControl()` | Điều khiển playback (chỉ host) |

### Server → Client (broadcast to `room_{roomId}`)

| Event | Trigger bởi | Payload | Handler FE |
|---|---|---|---|
| `ROOM_UPDATED` | `update()` (không phải ENDED) | `Partial<Room>` | Merge room state |
| `ROOM_ENDED` | `remove()` / `update(ENDED)` | `Partial<Room>` | Set ENDED, dừng audio |
| `QUEUE_UPDATED` | Thêm/duyệt/xóa bài hát | `RoomQueueItem` | Upsert vào queue |
| `REQUEST_UPDATED` | Duyệt/từ chối yêu cầu | `RoomQueueItem` | Upsert, hiện activityFeed |
| `NEW_REQUEST_NOTIFICATION` | Listener gửi yêu cầu | `RoomQueueItem` | Popup notification (host) |
| `RECEIVE_MESSAGE` | Có tin nhắn mới | `RoomMessage` | Upsert vào messages list |
| `USER_JOINED` | User join phòng | `RoomParticipant` | Upsert participants |
| `USER_LEFT` | User rời phòng | `{ roomId, userId }` | Filter khỏi participants |
| `PARTICIPANT_MODERATED` | Host kick/ban | `{ userId, action, reason }` | Filter participants, check bản thân |
| `PLAYER_STATE_SYNC` | `syncPlayback()` | `Partial<Room>` | Merge state, sync audio |
| `HOST_CONTROL` | `playNext()` | `{ action, room, queueItem }` | Merge room + queue state |

---

## 🗄️ Database Schema

### Collection: `rooms` — `schemas/room.schema.ts`

| Field | Type | Mô tả |
|---|---|---|
| `name` | String | Tên phòng |
| `description` | String? | Mô tả phòng |
| `imageUrl` | String | Ảnh bìa |
| `hostId` | ObjectId → User | Chủ phòng |
| `scheduledAt` | Date? | Thời gian lên lịch |
| `startedAt` | Date? | Thời điểm bắt đầu |
| `endedAt` | Date? | Thời điểm kết thúc |
| `status` | Enum | `WAITING` / `STREAMING` / `PAUSED` / `ENDED` |
| `sourceType` | Enum | `SONG` / `PLAYLIST` / `ALBUM` |
| `sourceId` | ObjectId | ID nguồn phát ban đầu |
| `currentSongId` | ObjectId → Song | Bài đang phát |
| `currentQueueItemId` | ObjectId | Queue item đang phát |
| `isPlaying` | Boolean | Trạng thái phát |
| `playbackPositionMs` | Number | Vị trí phát (ms) |
| `playbackStartedAt` | Date? | Thời điểm bắt đầu bài hiện tại |
| `participantCount` | Number | Số người trong phòng |

### Collection: `roomqueueitems` — `schemas/room-queue-item.schema.ts`

| Field | Type | Mô tả |
|---|---|---|
| `roomId` | ObjectId | Thuộc phòng nào |
| `songId` | ObjectId → Song | Bài hát |
| `requestedBy` | ObjectId → User | Người yêu cầu |
| `approvedBy` | ObjectId → User? | Người duyệt (host) |
| `order` | Number? | Thứ tự trong queue |
| `status` | Enum | `PENDING` / `APPROVED` / `PLAYING` / `PLAYED` / `REJECTED` / `REMOVED` |

### Collection: `roomparticipants` — `schemas/room-participant.schema.ts`

| Field | Type | Mô tả |
|---|---|---|
| `roomId` | ObjectId | Thuộc phòng nào |
| `userId` | ObjectId → User | Người tham gia |
| `role` | Enum | `HOST` / `LISTENER` |
| `status` | Enum | `ACTIVE` / `LEFT` / `KICKED` / `BANNED` |
| `joinedAt` | Date? | Lần join gần nhất |
| `leftAt` | Date? | Lần rời gần nhất |
| `lastSeenAt` | Date? | Lần hoạt động gần nhất |
| `moderatedBy` | ObjectId → User? | Người đã kick/ban |
| `moderationReason` | String? | Lý do |

### Collection: `roommessages` — `schemas/room-message.schema.ts`

| Field | Type | Mô tả |
|---|---|---|
| `roomId` | ObjectId | Thuộc phòng nào |
| `userId` | ObjectId → User | Người gửi |
| `content` | String | Nội dung tin nhắn |

---

## 🔒 Phân quyền

| Hành động | Host | Listener (Active) | Khách (chưa join) |
|---|---|---|---|
| Xem danh sách phòng | ✅ | ✅ | ✅ |
| Xem chi tiết phòng | ✅ | ✅ | ✅ (Public) |
| Join phòng | ✅ | ✅ | ✅ (cần account) |
| Chat trong phòng | ✅ | ✅ | ❌ |
| Yêu cầu bài hát | ✅ (auto) | ✅ (cần duyệt) | ❌ |
| Điều khiển playback | ✅ | ❌ | ❌ |
| Duyệt/từ chối request | ✅ | ❌ | ❌ |
| Kick/ban thành viên | ✅ | ❌ | ❌ |
| Kết thúc phòng | ✅ | ❌ | ❌ |
| Chỉnh sửa thông tin phòng | ✅ | ❌ | ❌ |

---

## 🧩 State Management (Frontend)

Toàn bộ state tập trung tại `app/roomDetail/[id]/page.tsx`:

```typescript
const [room, setRoom] = useState<RoomDetail | null>(null);
// Thông tin phòng + queue (cập nhật từ HTTP + real-time events)

const [messages, setMessages] = useState<RoomMessage[]>([]);
// Lịch sử chat (giữ 50 message gần nhất)

const [participants, setParticipants] = useState<RoomParticipant[]>([]);
// Danh sách thành viên active

const [activityFeed, setActivityFeed] = useState<RoomRealtimeNotification[]>([]);
// Feed sự kiện real-time (join/leave/request/control…), giữ 30 events

const audioRef = useRef<HTMLAudioElement | null>(null);
// DOM audio element — được sync bởi useEffect

const syncingAudioRef = useRef(false);
// Flag ngăn vòng lặp: host nhận sync từ server → play/pause → emit lại HOST_CONTROL
```

**Utility functions — `room-detail-helpers.ts`:**
- `mergeRoomState()` — Merge payload từ server vào state, giải quyết conflict `currentSong`
- `upsertQueueItem()` — Cập nhật/thêm queue item, giữ thứ tự `order`
- `upsertParticipant()` — Cập nhật/thêm participant, preserve populated `userId` object
- `upsertMessage()` — Cập nhật/thêm message (replace tempMessage bằng server message)
- `getHostId()` — Lấy hostId từ room (xử lý cả string ID và populated object)

---

## 🔄 Caching Strategy

| Tầng | Công nghệ | Mục đích |
|---|---|---|
| FE React Query | TanStack Query (stale-while-revalidate) | Cache API responses, invalidate sau mutation |
| BE In-Memory | `@nestjs/cache-manager` (TTL=5 phút) | Cache `room_active_user:{roomId}:{userId}` tránh query DB mỗi lần gửi chat |
| BE Redis | `RedisService.zincrby` | Tracking activity score phòng (`rooms:activity` sorted set) |
| BE MongoDB | Compound indexes trên `status`, `hostId`, `roomId` | Query performance |
