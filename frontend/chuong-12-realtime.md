# Chương 12: Real-time Communication

Giao tiếp thời gian thực (real-time) là khả năng server **chủ động đẩy dữ liệu** xuống client ngay khi có thay đổi, thay vì client phải liên tục hỏi server. Điều này cần thiết cho các tính năng như chat, thông báo đẩy, cập nhật giá, trạng thái online, và collaborative editing.

---

## 12.1. Polling & Long Polling

### Short Polling

Client gửi request lên server **theo định kỳ** (mỗi vài giây) để hỏi có dữ liệu mới không. Đây là cách đơn giản nhất nhưng kém hiệu quả nhất.

```tsx
// hooks/usePolling.ts
function usePolling<T>(url: string, interval: number = 5000) {
  const [data, setData] = useState<T | null>(null);

  useEffect(() => {
    async function poll() {
      const res = await fetch(url);
      const json = await res.json();
      setData(json);
    }

    poll(); // Gọi lần đầu ngay
    const timer = setInterval(poll, interval);

    return () => clearInterval(timer); // Cleanup khi unmount
  }, [url, interval]);

  return data;
}

// Dùng
function NotificationBell() {
  const notifications = usePolling<Notification[]>("/api/notifications", 10000);
  return <Bell count={notifications?.length ?? 0} />;
}
```

### Long Polling

Client gửi request, server **giữ kết nối** cho đến khi có dữ liệu mới hoặc timeout. Ngay khi nhận response, client lập tức gửi request mới. Hiệu quả hơn short polling nhưng vẫn tốn overhead HTTP.

```tsx
// hooks/useLongPolling.ts
function useLongPolling<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const controllerRef = useRef<AbortController | null>(null);

  useEffect(() => {
    let active = true;

    async function poll() {
      while (active) {
        controllerRef.current = new AbortController();
        try {
          const res = await fetch(url, {
            signal: controllerRef.current.signal,
          });

          if (!res.ok) throw new Error(`HTTP ${res.status}`);
          const json: T = await res.json();
          if (active) setData(json);
        } catch (err) {
          if ((err as Error).name === "AbortError") break;
          // Backoff khi lỗi — tránh spam server
          await new Promise((r) => setTimeout(r, 3000));
        }
      }
    }

    poll();
    return () => {
      active = false;
      controllerRef.current?.abort();
    };
  }, [url]);

  return data;
}
```

---

## 12.2. SSE — Server-Sent Events

SSE là giao thức **một chiều**: server đẩy dữ liệu xuống client qua một kết nối HTTP duy nhất, duy trì liên tục. Client không gửi dữ liệu qua kết nối này. Trình duyệt hỗ trợ tích hợp qua `EventSource` API và tự động reconnect khi mất kết nối.

### Server (Next.js Route Handler)

```typescript
// app/api/events/route.ts
export async function GET() {
  const encoder = new TextEncoder();

  const stream = new ReadableStream({
    start(controller) {
      // Gửi event ngay khi kết nối
      function send(data: object) {
        const payload = `data: ${JSON.stringify(data)}\n\n`;
        controller.enqueue(encoder.encode(payload));
      }

      send({ type: "connected", timestamp: Date.now() });

      // Giả lập push thông báo mỗi 3 giây
      const interval = setInterval(() => {
        send({
          type: "notification",
          message: "Bạn có tin nhắn mới",
          timestamp: Date.now(),
        });
      }, 3000);

      // Cleanup khi client disconnect
      const cleanup = () => {
        clearInterval(interval);
        controller.close();
      };

      // Timeout sau 30 giây để tránh kết nối lâu quá
      setTimeout(cleanup, 30000);
    },
  });

  return new Response(stream, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache",
      Connection: "keep-alive",
    },
  });
}
```

### Client

```tsx
// hooks/useSSE.ts
function useSSE<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [status, setStatus] = useState<"connecting" | "open" | "closed">(
    "connecting"
  );

  useEffect(() => {
    const eventSource = new EventSource(url);

    eventSource.onopen = () => setStatus("open");

    eventSource.onmessage = (event) => {
      const parsed: T = JSON.parse(event.data);
      setData(parsed);
    };

    eventSource.onerror = () => {
      setStatus("closed");
      eventSource.close(); // Trình duyệt tự reconnect nếu không close
    };

    return () => {
      eventSource.close();
      setStatus("closed");
    };
  }, [url]);

  return { data, status };
}

// Dùng trong component
function LiveFeed() {
  const { data, status } = useSSE<{ message: string; timestamp: number }>(
    "/api/events"
  );

  return (
    <div>
      <span className={`status status--${status}`}>{status}</span>
      {data && (
        <p>
          {data.message} — {new Date(data.timestamp).toLocaleTimeString()}
        </p>
      )}
    </div>
  );
}
```

---

## 12.3. WebSocket & Socket.IO

WebSocket là giao thức **hai chiều**, duy trì kết nối liên tục giữa client và server. Cả hai bên có thể gửi dữ liệu bất cứ lúc nào. Phù hợp cho chat, game, collaborative editing, live dashboard.

**Socket.IO** là thư viện xây dựng trên WebSocket, bổ sung thêm: tự động fallback (nếu WebSocket không khả dụng), reconnection tự động, rooms/namespaces, và event-based API.

### Server (Next.js với Socket.IO)

```typescript
// server.ts — Custom server (cần thiết cho Socket.IO)
import { createServer } from "http";
import { Server } from "socket.io";
import next from "next";

const app = next({ dev: process.env.NODE_ENV !== "production" });
const handler = app.getRequestHandler();

app.prepare().then(() => {
  const httpServer = createServer(handler);

  const io = new Server(httpServer, {
    cors: {
      origin: process.env.NEXT_PUBLIC_APP_URL,
      methods: ["GET", "POST"],
    },
  });

  // Namespace cho chat
  const chat = io.of("/chat");

  chat.on("connection", (socket) => {
    console.log(`User connected: ${socket.id}`);

    // User tham gia phòng
    socket.on("join_room", (roomId: string) => {
      socket.join(roomId);
      socket.to(roomId).emit("user_joined", { userId: socket.id });
    });

    // User gửi tin nhắn
    socket.on("send_message", (data: { roomId: string; message: string }) => {
      const messagePayload = {
        id: crypto.randomUUID(),
        userId: socket.id,
        message: data.message,
        timestamp: Date.now(),
      };
      // Gửi đến tất cả trong phòng (kể cả người gửi)
      chat.to(data.roomId).emit("receive_message", messagePayload);
    });

    socket.on("disconnect", () => {
      console.log(`User disconnected: ${socket.id}`);
    });
  });

  httpServer.listen(3000);
});
```

### Client

```tsx
// hooks/useChat.ts
"use client";
import { useEffect, useRef, useState } from "react";
import { io, Socket } from "socket.io-client";

interface Message {
  id: string;
  userId: string;
  message: string;
  timestamp: number;
}

function useChat(roomId: string) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [isConnected, setIsConnected] = useState(false);
  const socketRef = useRef<Socket | null>(null);

  useEffect(() => {
    // Kết nối Socket.IO
    const socket = io(process.env.NEXT_PUBLIC_APP_URL!, {
      namespace: "/chat",
    });

    socketRef.current = socket;

    socket.on("connect", () => {
      setIsConnected(true);
      socket.emit("join_room", roomId); // Tham gia phòng
    });

    socket.on("disconnect", () => setIsConnected(false));

    socket.on("receive_message", (msg: Message) => {
      setMessages((prev) => [...prev, msg]);
    });

    return () => {
      socket.disconnect();
    };
  }, [roomId]);

  function sendMessage(text: string) {
    socketRef.current?.emit("send_message", {
      roomId,
      message: text,
    });
  }

  return { messages, isConnected, sendMessage };
}

// Component Chat
function ChatRoom({ roomId }: { roomId: string }) {
  const { messages, isConnected, sendMessage } = useChat(roomId);
  const [input, setInput] = useState("");

  function handleSend() {
    if (!input.trim()) return;
    sendMessage(input);
    setInput("");
  }

  return (
    <div className="chat">
      <div className="chat__status">
        {isConnected ? "🟢 Đã kết nối" : "🔴 Đang kết nối..."}
      </div>

      <div className="chat__messages">
        {messages.map((msg) => (
          <div key={msg.id} className="chat__message">
            <span className="chat__user">{msg.userId.slice(0, 6)}</span>
            <p>{msg.message}</p>
            <time>{new Date(msg.timestamp).toLocaleTimeString()}</time>
          </div>
        ))}
      </div>

      <div className="chat__input">
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => e.key === "Enter" && handleSend()}
          placeholder="Nhập tin nhắn..."
          disabled={!isConnected}
        />
        <button onClick={handleSend} disabled={!isConnected}>
          Gửi
        </button>
      </div>
    </div>
  );
}
```

---

## So sánh các phương pháp Real-time

| | Short Polling | Long Polling | SSE | WebSocket |
|---|---|---|---|---|
| Giao thức | HTTP | HTTP | HTTP | WS (`ws://`) |
| Chiều truyền | Client → Server | Client → Server | Server → Client | Hai chiều |
| Kết nối | Tạo mới mỗi lần | Giữ đến khi có data | Một kết nối liên tục | Một kết nối liên tục |
| Overhead | Cao (nhiều request) | Trung bình | Thấp | Thấp nhất |
| Reconnect tự động | Không | Thủ công | Có (EventSource) | Không (Socket.IO có) |
| Triển khai | Rất dễ | Dễ | Dễ | Cần server riêng |
| Dùng cho | Thông báo đơn giản | Thông báo, queue | Live feed, notification | Chat, game, collab |
| Next.js App Router | ✅ | ✅ | ✅ Route Handler | ⚠️ Cần custom server |

### Khi nào dùng gì?

- **Short Polling:** Dữ liệu thay đổi chậm (mỗi vài phút), không cần real-time thực sự. Đơn giản, dễ debug.
- **Long Polling:** Cần phản hồi nhanh hơn polling nhưng không thể dùng SSE/WebSocket. Fallback khi các phương pháp khác không khả dụng.
- **SSE:** Thông báo, live feed, log streaming, progress update — server đẩy, client chỉ nhận. Hoạt động tốt với Next.js App Router mà không cần cấu hình thêm.
- **WebSocket / Socket.IO:** Chat, multiplayer game, collaborative editing, trading dashboard — cần giao tiếp hai chiều thực sự.
