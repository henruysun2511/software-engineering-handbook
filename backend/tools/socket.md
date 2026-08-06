# Socket.IO – Tổng hợp kiến thức (đặc biệt cho NestJS)

---

## 1. Socket.IO là gì?

Socket.IO là thư viện **real-time, bidirectional communication** giữa client và server, xây dựng trên WebSocket với fallback tự động (long-polling). Khác với HTTP (request-response), Socket.IO cho phép server **chủ động push data** xuống client.

**Dùng Socket.IO khi:**
- Chat real-time
- Notification live
- Collaborative editing (Google Docs kiểu)
- Live dashboard / monitoring
- Online gaming
- Real-time tracking (delivery, ride-sharing)

**So sánh WebSocket thuần vs Socket.IO:**

| | WebSocket thuần | Socket.IO |
|---|---|---|
| Reconnect tự động | ❌ | ✅ |
| Fallback (long-polling) | ❌ | ✅ |
| Rooms & Namespaces | ❌ | ✅ |
| Broadcast | Thủ công | ✅ built-in |
| Acknowledgement | ❌ | ✅ |
| Redis Adapter (scale) | ❌ | ✅ |

---

## 2. Kiến trúc tổng quan

```
┌─────────────┐   WebSocket / HTTP polling   ┌─────────────────┐
│   Client    │ ◄──────────────────────────► │     Server      │
│ (Browser/   │                              │  (NestJS +      │
│  Mobile)    │                              │   Socket.IO)    │
└─────────────┘                              └────────┬────────┘
                                                      │
                                            ┌─────────▼────────┐
                                            │  Redis Adapter   │
                                            │ (multi-instance) │
                                            └──────────────────┘
```

**Các khái niệm cốt lõi:**
- **Socket**: kết nối 1-1 giữa 1 client và server
- **Namespace**: phân vùng logic (`/chat`, `/admin`) — như sub-app
- **Room**: nhóm các socket lại để broadcast
- **Event**: tên của message được gửi/nhận
- **Acknowledgement**: callback xác nhận đã nhận message

---

## 3. Cài đặt NestJS

```bash
npm install @nestjs/websockets @nestjs/platform-socket.io socket.io
npm install -D @types/socket.io

# Redis Adapter (scale nhiều instance)
npm install @socket.io/redis-adapter ioredis
```

---

## 4. Gateway – Trái tim của Socket.IO trong NestJS

Gateway là class xử lý WebSocket, tương đương Controller trong HTTP.

```typescript
// chat.gateway.ts
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  MessageBody,
  ConnectedSocket,
  OnGatewayInit,
  OnGatewayConnection,
  OnGatewayDisconnect,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';

@WebSocketGateway({
  namespace: '/chat',       // namespace (mặc định: '/')
  cors: {
    origin: ['http://localhost:3001'],
    credentials: true,
  },
  transports: ['websocket', 'polling'],
})
export class ChatGateway
  implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect
{
  @WebSocketServer()
  server: Server;

  // Chạy sau khi Gateway khởi tạo
  afterInit(server: Server) {
    console.log('WebSocket Gateway initialized');
  }

  // Client kết nối
  handleConnection(client: Socket) {
    console.log(`Client connected: ${client.id}`);
    console.log('Query:', client.handshake.query);
    console.log('Auth:', client.handshake.auth);
  }

  // Client ngắt kết nối
  handleDisconnect(client: Socket) {
    console.log(`Client disconnected: ${client.id}`);
  }

  // Lắng nghe event 'message'
  @SubscribeMessage('message')
  handleMessage(
    @MessageBody() data: { roomId: string; content: string },
    @ConnectedSocket() client: Socket,
  ) {
    // Gửi lại cho tất cả trong room (trừ người gửi)
    client.to(data.roomId).emit('message', {
      from: client.id,
      content: data.content,
      timestamp: new Date(),
    });

    // Trả về acknowledgement cho người gửi
    return { status: 'ok' };
  }
}
```

### Đăng ký Gateway trong Module

```typescript
// chat.module.ts
@Module({
  providers: [ChatGateway, ChatService],
})
export class ChatModule {}

// app.module.ts
@Module({
  imports: [ChatModule],
})
export class AppModule {}
```

---

## 5. Emit – Các cách gửi message

```typescript
// Trong Gateway
@SubscribeMessage('join-room')
handleJoinRoom(@MessageBody() roomId: string, @ConnectedSocket() client: Socket) {

  // 1. Gửi cho đúng 1 client
  client.emit('event', data);

  // 2. Gửi cho tất cả (kể cả người gửi)
  this.server.emit('event', data);

  // 3. Gửi cho tất cả trong room (trừ người gửi)
  client.to(roomId).emit('event', data);

  // 4. Gửi cho tất cả trong room (kể cả người gửi)
  this.server.to(roomId).emit('event', data);

  // 5. Gửi cho nhiều room
  this.server.to('room1').to('room2').emit('event', data);

  // 6. Gửi cho tất cả trừ 1 socket
  client.broadcast.emit('event', data);

  // 7. Gửi cho socket cụ thể (biết socketId)
  this.server.to(socketId).emit('event', data);

  // 8. Gửi cho tất cả trong namespace
  this.server.of('/chat').emit('event', data);
}
```

---

## 6. Rooms

```typescript
@SubscribeMessage('join-room')
async handleJoinRoom(
  @MessageBody() roomId: string,
  @ConnectedSocket() client: Socket,
) {
  // Tham gia room
  await client.join(roomId);

  // Thông báo cho các thành viên trong room
  this.server.to(roomId).emit('user-joined', {
    userId: client.handshake.auth.userId,
    roomId,
  });

  // Lấy danh sách socket trong room
  const sockets = await this.server.in(roomId).fetchSockets();
  return { members: sockets.length };
}

@SubscribeMessage('leave-room')
async handleLeaveRoom(
  @MessageBody() roomId: string,
  @ConnectedSocket() client: Socket,
) {
  await client.leave(roomId);
  client.to(roomId).emit('user-left', { userId: client.handshake.auth.userId });
}

// Lấy tất cả rooms của 1 socket
const rooms = client.rooms; // Set { socketId, 'room1', 'room2' }
```

---

## 7. Namespaces

```typescript
// Namespace /admin
@WebSocketGateway({ namespace: '/admin' })
export class AdminGateway {
  @WebSocketServer() server: Server;

  @SubscribeMessage('broadcast-alert')
  handleAlert(@MessageBody() message: string) {
    this.server.emit('alert', { message, timestamp: new Date() });
  }
}

// Namespace /notifications
@WebSocketGateway({ namespace: '/notifications' })
export class NotificationGateway {
  @WebSocketServer() server: Server;

  async sendToUser(userId: string, notification: object) {
    this.server.to(`user:${userId}`).emit('notification', notification);
  }
}
```

**Client kết nối namespace:**
```javascript
const chatSocket = io('http://localhost:3000/chat');
const adminSocket = io('http://localhost:3000/admin');
```

---

## 8. Authentication & Guards

### JWT Auth khi connect

```typescript
// ws-jwt.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { WsException } from '@nestjs/websockets';
import { Socket } from 'socket.io';

@Injectable()
export class WsJwtGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const client: Socket = context.switchToWs().getClient();
    const token =
      client.handshake.auth?.token ||
      client.handshake.headers?.authorization?.split(' ')[1];

    if (!token) throw new WsException('Unauthorized');

    try {
      const payload = this.jwtService.verify(token);
      client.data.user = payload; // gắn user vào socket
      return true;
    } catch {
      throw new WsException('Invalid token');
    }
  }
}
```

```typescript
// Dùng guard trong Gateway
@UseGuards(WsJwtGuard)
@SubscribeMessage('private-message')
handlePrivate(
  @MessageBody() data: object,
  @ConnectedSocket() client: Socket,
) {
  const user = client.data.user; // user từ JWT payload
  // ...
}
```

### Auth tại thời điểm connect (Middleware)

```typescript
afterInit(server: Server) {
  server.use(async (socket, next) => {
    const token = socket.handshake.auth?.token;
    try {
      const payload = this.jwtService.verify(token);
      socket.data.user = payload;
      next();
    } catch {
      next(new Error('Authentication error'));
    }
  });
}
```

---

## 9. Acknowledgements (xác nhận đã nhận)

```typescript
// Server
@SubscribeMessage('send-message')
handleSendMessage(@MessageBody() data: object) {
  // return value → acknowledgement gửi về client
  return { messageId: uuid(), status: 'delivered', timestamp: new Date() };
}

// Hoặc dùng callback
@SubscribeMessage('send-message')
handleSendMessage(
  @MessageBody() data: object,
  @ConnectedSocket() client: Socket,
) {
  // ...process...
  return { ok: true }; // ack
}
```

```javascript
// Client
socket.emit('send-message', { content: 'Hello' }, (ack) => {
  console.log('Server acknowledged:', ack); // { messageId: '...', status: 'delivered' }
});
```

---

## 10. Emit từ Service (ngoài Gateway)

Đây là pattern rất phổ biến: HTTP request trigger → push Socket event.

```typescript
// notification.service.ts
@Injectable()
export class NotificationService {
  constructor(
    @InjectRepository(Notification) private repo: Repository<Notification>,
    private notificationGateway: NotificationGateway,
  ) {}

  async createAndNotify(userId: string, message: string) {
    const notif = await this.repo.save({ userId, message });

    // Push real-time ngay khi tạo notification
    this.notificationGateway.sendToUser(userId, notif);

    return notif;
  }
}

// notification.gateway.ts
@WebSocketGateway({ namespace: '/notifications' })
export class NotificationGateway {
  @WebSocketServer() server: Server;

  sendToUser(userId: string, data: object) {
    this.server.to(`user:${userId}`).emit('new-notification', data);
  }
}
```

**Lưu ý quan trọng:** Tránh circular dependency — Service inject Gateway, Gateway đừng inject lại Service.

---

## 11. User-Socket Mapping (online presence)

```typescript
// Lưu mapping userId ↔ socketId
@WebSocketGateway({ namespace: '/chat' })
export class ChatGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer() server: Server;

  // Map<userId, Set<socketId>> — user có thể dùng nhiều tab
  private userSockets = new Map<string, Set<string>>();

  handleConnection(client: Socket) {
    const userId = client.data.user?.id;
    if (!userId) return;

    // Join room cá nhân để dễ emit sau này
    client.join(`user:${userId}`);

    if (!this.userSockets.has(userId)) {
      this.userSockets.set(userId, new Set());
    }
    this.userSockets.get(userId).add(client.id);
  }

  handleDisconnect(client: Socket) {
    const userId = client.data.user?.id;
    if (!userId) return;

    const sockets = this.userSockets.get(userId);
    sockets?.delete(client.id);
    if (sockets?.size === 0) {
      this.userSockets.delete(userId);
      // Broadcast user offline
      this.server.emit('user-offline', { userId });
    }
  }

  isOnline(userId: string): boolean {
    return (this.userSockets.get(userId)?.size ?? 0) > 0;
  }
}
```

---

## 12. Exception Filters

```typescript
// ws-exception.filter.ts
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseWsExceptionFilter, WsException } from '@nestjs/websockets';

@Catch(WsException)
export class WsExceptionFilter extends BaseWsExceptionFilter {
  catch(exception: WsException, host: ArgumentsHost) {
    const client = host.switchToWs().getClient<Socket>();
    client.emit('error', {
      message: exception.message,
      timestamp: new Date(),
    });
  }
}

// Áp dụng
@UseFilters(WsExceptionFilter)
@WebSocketGateway()
export class ChatGateway { ... }
```

---

## 13. Interceptors & Pipes trong Gateway

```typescript
// Validation pipe cho WebSocket
@WebSocketGateway()
export class ChatGateway {

  @UsePipes(new ValidationPipe())
  @SubscribeMessage('message')
  handleMessage(@MessageBody() dto: SendMessageDto) {
    // dto đã được validate
  }
}

// DTO
import { IsString, IsNotEmpty, MaxLength } from 'class-validator';

export class SendMessageDto {
  @IsString()
  @IsNotEmpty()
  roomId: string;

  @IsString()
  @IsNotEmpty()
  @MaxLength(1000)
  content: string;
}
```

---

## 14. Redis Adapter – Scale nhiều instance

Khi deploy nhiều server instance, cần Redis Adapter để **đồng bộ event** giữa các instance.

```typescript
// main.ts
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const pubClient = createClient({ url: 'redis://localhost:6379' });
  const subClient = pubClient.duplicate();
  await Promise.all([pubClient.connect(), subClient.connect()]);

  const io = app.get(Server); // lấy Socket.IO server instance
  io.adapter(createAdapter(pubClient, subClient));

  await app.listen(3000);
}
```

```
Instance 1 (port 3000) ─┐
                         ├── Redis Pub/Sub ── đồng bộ events
Instance 2 (port 3001) ─┘
```

---

## 15. Patterns & Best Practices

### 15.1. Room naming convention
```typescript
`user:${userId}`         // room riêng cho user
`room:${roomId}`         // phòng chat
`org:${orgId}`           // tổ chức
`session:${sessionId}`   // phiên làm việc
```

### 15.2. Tách event types thành constants
```typescript
// events.ts
export const EVENTS = {
  // Client → Server
  JOIN_ROOM: 'join-room',
  SEND_MESSAGE: 'send-message',
  TYPING: 'typing',

  // Server → Client
  NEW_MESSAGE: 'new-message',
  USER_JOINED: 'user-joined',
  USER_LEFT: 'user-left',
  ERROR: 'error',
} as const;
```

### 15.3. Không xử lý business logic trong Gateway
```typescript
// ❌ Sai – logic trong Gateway
@SubscribeMessage('message')
async handle(@MessageBody() data) {
  const user = await this.db.findUser(data.userId);
  const msg = await this.db.saveMessage(data);
  // ...
}

// ✅ Đúng – delegate sang Service
@SubscribeMessage('message')
async handle(@MessageBody() data, @ConnectedSocket() client: Socket) {
  const result = await this.chatService.sendMessage(data, client.data.user);
  return result;
}
```

### 15.4. Heartbeat / Ping-Pong
```typescript
@WebSocketGateway({
  pingInterval: 25_000,  // gửi ping mỗi 25s
  pingTimeout: 60_000,   // disconnect nếu không pong trong 60s
})
```

### 15.5. Throttle để chống spam
```typescript
@SubscribeMessage('message')
@UseGuards(WsThrottlerGuard) // custom guard đếm event theo socketId
handleMessage() { ... }
```

---

## 16. Testing Gateway

```typescript
// chat.gateway.spec.ts
import { Test } from '@nestjs/testing';
import { ChatGateway } from './chat.gateway';

describe('ChatGateway', () => {
  let gateway: ChatGateway;
  let mockServer: Partial<Server>;

  beforeEach(async () => {
    mockServer = { to: jest.fn().mockReturnThis(), emit: jest.fn() };

    const module = await Test.createTestingModule({
      providers: [ChatGateway, ChatService],
    }).compile();

    gateway = module.get(ChatGateway);
    gateway.server = mockServer as Server;
  });

  it('should broadcast message to room', async () => {
    const mockClient = {
      id: 'socket-1',
      data: { user: { id: 'user-1' } },
      to: jest.fn().mockReturnThis(),
      emit: jest.fn(),
    } as unknown as Socket;

    await gateway.handleMessage({ roomId: 'room-1', content: 'Hi' }, mockClient);

    expect(mockClient.to).toHaveBeenCalledWith('room-1');
  });
});
```

---

## 17. Checklist trước khi deploy

- [ ] Bật CORS đúng origin (không dùng `*` ở production)
- [ ] Xác thực JWT khi client connect (middleware `afterInit`)
- [ ] Dùng Redis Adapter nếu chạy nhiều instance
- [ ] Đặt `pingInterval` và `pingTimeout` hợp lý
- [ ] Rate limit event để chống spam/DDoS
- [ ] Dùng `client.data` để lưu user info (không dùng biến ngoài)
- [ ] Đặt room naming convention thống nhất
- [ ] Log connection/disconnection để monitor
- [ ] Graceful shutdown: đóng kết nối WebSocket khi server dừng
- [ ] Monitor số lượng connection đang active
