# Chương 11: Authentication & Authorization

Bảo mật là yêu cầu bắt buộc của mọi ứng dụng web. Hai khái niệm nền tảng — Authentication và Authorization — thường bị nhầm lẫn nhưng có vai trò hoàn toàn khác nhau và cần được triển khai đúng từ đầu.

---

## 11.1. Authentication (Xác thực)

**Authentication** là quá trình xác minh danh tính: **"Bạn là ai?"**

Hệ thống kiểm tra xem người dùng có thực sự là người mà họ tự nhận hay không, thông qua thông tin xác thực (credentials) như mật khẩu, mã OTP, sinh trắc học, hoặc token từ bên thứ ba.

Các hình thức xác thực phổ biến:

| Hình thức | Mô tả | Ví dụ |
|---|---|---|
| **Password-based** | Tên đăng nhập + mật khẩu | Email/password truyền thống |
| **OTP** | Mã dùng một lần | SMS, Google Authenticator |
| **Social Login** | Xác thực qua bên thứ ba | Google, GitHub, Facebook |
| **Biometric** | Sinh trắc học | Vân tay, Face ID |
| **Magic Link** | Link đăng nhập gửi qua email | Notion, Slack |
| **SSO** | Single Sign-On | Đăng nhập một lần dùng nhiều dịch vụ |

---

## 11.2. Authorization (Phân quyền)

**Authorization** là quá trình xác định quyền hạn: **"Bạn được làm gì?"**

Sau khi đã xác thực xong (biết bạn là ai), hệ thống kiểm tra bạn có quyền thực hiện hành động cụ thể đó không.

```
Người dùng → Authentication → "Đây là An" → Authorization → "An được xem, không được xóa"
```

### So sánh Authentication vs Authorization

| | Authentication | Authorization |
|---|---|---|
| Câu hỏi | Bạn là ai? | Bạn được làm gì? |
| Xảy ra khi | Đăng nhập | Mỗi lần truy cập tài nguyên |
| Thất bại → | `401 Unauthorized` | `403 Forbidden` |
| Dựa trên | Credentials (mật khẩu, token) | Roles, permissions |
| Ví dụ | Đăng nhập bằng email/password | Admin xóa được, user chỉ xem |

> **Lưu ý thực tế:** `401` nghĩa là chưa xác thực — cần đăng nhập lại. `403` nghĩa là đã xác thực nhưng không đủ quyền — không nên redirect về trang login mà hiển thị thông báo lỗi quyền hạn.

---

## 11.3. Session

Session là cơ chế lưu trạng thái đăng nhập phía **server**. Sau khi người dùng đăng nhập thành công, server tạo ra một session record trong database hoặc bộ nhớ (Redis), sau đó gửi **Session ID** về client qua cookie.

```
1. Client gửi: POST /login { email, password }
2. Server xác thực → Tạo session trong DB → Trả về Session ID qua cookie
3. Client gửi cookie kèm mỗi request
4. Server tra cứu Session ID → Biết đây là user nào
```

### Session vs Token (JWT)

| | Session | JWT |
|---|---|---|
| Lưu trữ | Server (DB/Redis) | Client (cookie/localStorage) |
| Kiểm tra | Tra cứu DB mỗi request | Verify chữ ký — không cần DB |
| Revoke | Dễ — xóa session trong DB | Khó — phải dùng blocklist |
| Scale | Cần shared storage | Stateless, dễ scale |
| Kích thước | Session ID nhỏ (~32 bytes) | Token lớn hơn (~200–500 bytes) |
| Dùng khi | App truyền thống, cần revoke ngay | API, microservice, SPA |

---

## 11.4. JWT (JSON Web Token)

JWT là chuẩn mở (RFC 7519) để truyền thông tin an toàn giữa các bên dưới dạng JSON được **ký số (signed)**. Server không cần lưu trữ token — chỉ cần verify chữ ký là biết token hợp lệ hay không.

### Cấu trúc JWT

JWT gồm ba phần phân cách bởi dấu `.`, mỗi phần được encode Base64URL:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9   ← Header
.
eyJzdWIiOiIxMjM0IiwibmFtZSI6IkFuIiwicm9sZSI6InVzZXIiLCJleHAiOjE3MjAwMDAwMDB9  ← Payload
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  ← Signature
```

**Header** — Khai báo thuật toán ký:
```json
{ "alg": "HS256", "typ": "JWT" }
```

**Payload** — Chứa claims (thông tin):
```json
{
  "sub": "1234",
  "name": "An",
  "role": "user",
  "iat": 1720000000,
  "exp": 1720003600
}
```

**Signature** — Chữ ký đảm bảo token không bị giả mạo:
```
HMACSHA256(base64(header) + "." + base64(payload), SECRET_KEY)
```

### Access Token

Access Token là JWT ngắn hạn dùng để xác thực mỗi request. Thường có thời hạn **15 phút đến 1 giờ**.

```tsx
// Gửi Access Token trong Authorization header
const response = await fetch("/api/users", {
  headers: {
    Authorization: `Bearer ${accessToken}`,
  },
});
```

### Payload

Payload chứa các **claims** — thông tin về user và token:

| Claim | Mô tả | Ví dụ |
|---|---|---|
| `sub` | Subject — ID của user | `"user_42"` |
| `iat` | Issued At — thời điểm tạo | Unix timestamp |
| `exp` | Expiration — thời điểm hết hạn | Unix timestamp |
| `role` | Custom claim — vai trò | `"admin"` |
| `email` | Custom claim — email | `"an@example.com"` |

> **Lưu ý bảo mật:** Payload chỉ được **encode** (Base64), không được **mã hóa** — ai cũng đọc được nếu có token. Không đặt thông tin nhạy cảm (mật khẩu, số thẻ ngân hàng) trong payload.

### Expiration (Hết hạn)

Access Token phải có thời hạn ngắn để giới hạn thiệt hại nếu bị đánh cắp:

```typescript
// Kiểm tra token hết hạn phía client
function isTokenExpired(token: string): boolean {
  try {
    const payload = JSON.parse(atob(token.split(".")[1]));
    return Date.now() >= payload.exp * 1000;
  } catch {
    return true;
  }
}
```

---

## 11.5. Refresh Token

Refresh Token là token dài hạn (7–30 ngày) dùng **duy nhất để lấy Access Token mới** khi Access Token hết hạn. Refresh Token phải được lưu trữ an toàn hơn Access Token.

### Refresh Flow

```
1. Login → Server trả về: Access Token (15 phút) + Refresh Token (7 ngày)
2. Client dùng Access Token cho mọi API request
3. Access Token hết hạn → API trả về 401
4. Client gửi Refresh Token → POST /auth/refresh
5. Server verify Refresh Token → Trả về Access Token mới
6. Client dùng Access Token mới, tiếp tục bình thường
7. Refresh Token hết hạn → Bắt buộc đăng nhập lại
```

```tsx
// lib/axios.ts — Tự động refresh trong interceptor
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const original = error.config;

    if (error.response?.status === 401 && !original._retry) {
      original._retry = true;

      try {
        // Gửi refresh token (thường qua cookie HttpOnly)
        const { data } = await axios.post<{ accessToken: string }>(
          "/api/auth/refresh",
          {},
          { withCredentials: true } // Gửi kèm cookie
        );

        // Lưu access token mới
        tokenStore.set(data.accessToken);

        // Gắn token mới vào request ban đầu và thử lại
        original.headers.Authorization = `Bearer ${data.accessToken}`;
        return apiClient(original);
      } catch {
        // Refresh thất bại → đăng xuất
        tokenStore.clear();
        window.location.href = "/login";
        return Promise.reject(error);
      }
    }

    return Promise.reject(error);
  }
);
```

### Token Rotation

Token Rotation là cơ chế bảo mật: mỗi lần dùng Refresh Token để lấy Access Token mới, server sẽ **cấp một Refresh Token mới** và **vô hiệu hóa cái cũ**.

```
Lần 1: Refresh Token A → Access Token mới + Refresh Token B (A bị vô hiệu)
Lần 2: Refresh Token B → Access Token mới + Refresh Token C (B bị vô hiệu)
```

**Lợi ích:** Nếu kẻ tấn công đánh cắp được Refresh Token và dùng nó, server phát hiện token cũ được dùng lại → vô hiệu hóa toàn bộ session của user đó ngay lập tức.

---

## 11.6. Cookie

Cookie là mẫu dữ liệu nhỏ lưu trên trình duyệt, được gửi kèm **tự động** trong mọi HTTP request đến cùng domain. Đây là nơi lý tưởng để lưu Refresh Token nhờ các thuộc tính bảo mật.

### HttpOnly

Cookie có `HttpOnly` flag **không thể truy cập qua JavaScript** (`document.cookie`). Điều này bảo vệ token khỏi tấn công **XSS (Cross-Site Scripting)** — kể cả khi attacker inject được script vào trang, script đó vẫn không đọc được cookie.

```
Set-Cookie: refresh_token=abc123; HttpOnly; ...
```

```typescript
// JavaScript không thể đọc được — document.cookie không thấy cookie HttpOnly
console.log(document.cookie); // "" — refresh_token không xuất hiện
```

### Secure

Cookie có `Secure` flag **chỉ được gửi qua HTTPS**. Ngăn chặn cookie bị lộ qua kết nối HTTP không mã hóa.

```
Set-Cookie: refresh_token=abc123; Secure; HttpOnly; ...
```

### SameSite

`SameSite` kiểm soát khi nào cookie được gửi trong cross-site request, bảo vệ khỏi tấn công **CSRF (Cross-Site Request Forgery)**:

| Giá trị | Mô tả | Dùng khi |
|---|---|---|
| `Strict` | Chỉ gửi khi điều hướng từ cùng site | Bảo mật cao nhất, có thể gây UX xấu |
| `Lax` | Gửi khi điều hướng top-level (click link) | Cân bằng tốt — **mặc định hiện đại** |
| `None` | Luôn gửi (kể cả cross-site) | Phải có `Secure`, dùng cho iframe, widget nhúng |

```
Set-Cookie: refresh_token=abc123; HttpOnly; Secure; SameSite=Lax; Max-Age=604800; Path=/api/auth
```

**Cấu hình cookie an toàn trong Next.js:**

```typescript
// app/api/auth/login/route.ts
import { NextRequest, NextResponse } from "next/server";

export async function POST(request: NextRequest) {
  const { email, password } = await request.json();
  const { accessToken, refreshToken } = await loginUser(email, password);

  const response = NextResponse.json({ accessToken });

  // Refresh Token lưu trong cookie HttpOnly — client không đọc được
  response.cookies.set("refresh_token", refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "lax",
    maxAge: 60 * 60 * 24 * 7,  // 7 ngày
    path: "/api/auth",          // Chỉ gửi đến /api/auth — giới hạn phạm vi
  });

  return response;
}
```

### So sánh nơi lưu token

| | HttpOnly Cookie | localStorage | Memory (JS variable) |
|---|---|---|---|
| XSS | ✅ An toàn | ❌ Dễ bị đánh cắp | ✅ An toàn |
| CSRF | ❌ Cần SameSite | ✅ Không tự động gửi | ✅ Không tự động gửi |
| Tồn tại sau refresh | ✅ | ✅ | ❌ Mất khi reload |
| Dùng cho | Refresh Token | ❌ Không nên | Access Token ngắn hạn |

> **Best practice:** Lưu **Refresh Token** trong `HttpOnly cookie`. Lưu **Access Token** trong bộ nhớ JavaScript (biến, Zustand, Redux) — không persist, tự động mất khi reload, và lấy lại bằng Refresh Token.

---

## 11.7. OAuth2 & Google Login

### OAuth2 là gì?

OAuth2 là giao thức ủy quyền (authorization protocol) cho phép ứng dụng truy cập tài nguyên của người dùng trên một dịch vụ khác **mà không cần biết mật khẩu của người dùng**. Trong ngữ cảnh "Login with Google", OAuth2 được dùng để xác thực danh tính thông qua Google.

### Luồng Authorization Code (phổ biến nhất)

```
1. User click "Đăng nhập với Google"
2. App redirect → Google Authorization Server
   https://accounts.google.com/o/oauth2/auth?
     client_id=...&redirect_uri=...&scope=email profile&response_type=code

3. User đồng ý cấp quyền trên Google
4. Google redirect về app kèm Authorization Code:
   https://yourapp.com/callback?code=4/ABC123

5. Backend đổi code lấy tokens:
   POST https://oauth2.googleapis.com/token
   { code, client_id, client_secret, redirect_uri }

6. Google trả về: { access_token, id_token, refresh_token }

7. Backend verify id_token → lấy thông tin user (email, name, picture)
8. Tạo hoặc tìm user trong DB → Tạo session/JWT của hệ thống
9. Trả token về client
```

### Triển khai với NextAuth.js (Next.js)

NextAuth.js là thư viện xử lý toàn bộ luồng OAuth2 cho Next.js — không cần tự implement:

```typescript
// app/api/auth/[...nextauth]/route.ts
import NextAuth from "next-auth";
import GoogleProvider from "next-auth/providers/google";

const handler = NextAuth({
  providers: [
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
  ],
  callbacks: {
    async jwt({ token, account, user }) {
      // Lần đầu đăng nhập — gắn thêm thông tin vào token
      if (account && user) {
        token.role = await getUserRole(user.email!);
      }
      return token;
    },
    async session({ session, token }) {
      // Gắn thông tin từ token vào session (có thể dùng trong component)
      session.user.role = token.role as string;
      return session;
    },
  },
});

export { handler as GET, handler as POST };
```

```tsx
// Dùng trong Client Component
"use client";
import { signIn, signOut, useSession } from "next-auth/react";

function AuthButton() {
  const { data: session, status } = useSession();

  if (status === "loading") return <p>Đang tải...</p>;

  if (session) {
    return (
      <div>
        <p>Xin chào, {session.user?.name}</p>
        <button onClick={() => signOut()}>Đăng xuất</button>
      </div>
    );
  }

  return (
    <button onClick={() => signIn("google")}>
      Đăng nhập với Google
    </button>
  );
}
```

---

## 11.8. RBAC (Role-Based Access Control)

RBAC là mô hình phân quyền dựa trên **vai trò (role)** của người dùng. Thay vì gán quyền trực tiếp cho từng user, quyền được gán cho role, sau đó user được gán vào role.

```
User → Role → Permissions

An     → admin  → [read, create, update, delete]
Bình   → editor → [read, create, update]
Châu   → viewer → [read]
```

### Định nghĩa Role và Permission

```typescript
// types/auth.ts
type Role = "admin" | "editor" | "viewer";

type Permission =
  | "users:read"
  | "users:create"
  | "users:update"
  | "users:delete"
  | "posts:read"
  | "posts:create"
  | "posts:update"
  | "posts:delete";

// Ma trận phân quyền
const ROLE_PERMISSIONS: Record<Role, Permission[]> = {
  admin: [
    "users:read", "users:create", "users:update", "users:delete",
    "posts:read", "posts:create", "posts:update", "posts:delete",
  ],
  editor: [
    "users:read",
    "posts:read", "posts:create", "posts:update",
  ],
  viewer: [
    "users:read",
    "posts:read",
  ],
};

function hasPermission(role: Role, permission: Permission): boolean {
  return ROLE_PERMISSIONS[role].includes(permission);
}
```

### Custom Hook kiểm tra quyền

```tsx
// hooks/usePermission.ts
import { useAuth } from "@/providers/AuthProvider";

function usePermission(permission: Permission): boolean {
  const { user } = useAuth();
  if (!user) return false;
  return hasPermission(user.role as Role, permission);
}

// Dùng trong component
function UserManagementPage() {
  const canCreate = usePermission("users:create");
  const canDelete = usePermission("users:delete");

  return (
    <div>
      <h1>Quản lý người dùng</h1>
      {canCreate && <Button onClick={handleCreate}>Thêm user</Button>}
      <UserTable
        users={users}
        showDeleteButton={canDelete}
      />
    </div>
  );
}
```

### RBAC trên server (Next.js Route Handler)

```typescript
// lib/auth.ts — Utility kiểm tra quyền phía server
import { getServerSession } from "next-auth";

export async function requirePermission(permission: Permission): Promise<void> {
  const session = await getServerSession();

  if (!session) {
    throw new Response("Chưa xác thực", { status: 401 });
  }

  const role = session.user.role as Role;
  if (!hasPermission(role, permission)) {
    throw new Response("Không có quyền", { status: 403 });
  }
}

// app/api/users/route.ts
export async function DELETE(request: NextRequest) {
  await requirePermission("users:delete"); // Throw nếu không đủ quyền
  // ... xử lý xóa
}
```

---

## 11.9. Route Protection

Route Protection là cơ chế ngăn người dùng chưa xác thực hoặc không đủ quyền truy cập các trang nhất định.

### Middleware (Next.js) — Bảo vệ ở tầng edge

Middleware là lớp bảo vệ đầu tiên, chạy trước khi request đến page. Đây là cách hiệu quả nhất để bảo vệ route trong Next.js vì không cần load page rồi mới redirect.

```typescript
// middleware.ts
import { NextRequest, NextResponse } from "next/server";
import { getToken } from "next-auth/jwt";

const PUBLIC_ROUTES = ["/", "/login", "/register", "/about"];
const AUTH_ROUTES = ["/login", "/register"];

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;
  const token = await getToken({ req: request });
  const isAuthenticated = !!token;

  // Route công khai — cho qua
  if (PUBLIC_ROUTES.includes(pathname)) {
    // Đã đăng nhập, vào trang login → về dashboard
    if (isAuthenticated && AUTH_ROUTES.includes(pathname)) {
      return NextResponse.redirect(new URL("/dashboard", request.url));
    }
    return NextResponse.next();
  }

  // Route cần xác thực — chưa đăng nhập → về login
  if (!isAuthenticated) {
    const loginUrl = new URL("/login", request.url);
    loginUrl.searchParams.set("callbackUrl", pathname); // Redirect về sau khi login
    return NextResponse.redirect(loginUrl);
  }

  // Route admin — kiểm tra role
  if (pathname.startsWith("/admin") && token.role !== "admin") {
    return NextResponse.redirect(new URL("/403", request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: [
    "/((?!api|_next/static|_next/image|favicon.ico).*)",
  ],
};
```

### Guard Component (Client-side)

Client-side guard là lớp bảo vệ thứ hai, dùng để bảo vệ các phần UI trong cùng một page:

```tsx
// components/AuthGuard.tsx
"use client";
import { useSession } from "next-auth/react";
import { useRouter } from "next/navigation";
import { useEffect } from "react";

interface AuthGuardProps {
  children: React.ReactNode;
  requiredRole?: Role;
  fallback?: React.ReactNode;
}

function AuthGuard({ children, requiredRole, fallback }: AuthGuardProps) {
  const { data: session, status } = useSession();
  const router = useRouter();

  useEffect(() => {
    if (status === "unauthenticated") {
      router.push("/login");
    }
  }, [status, router]);

  if (status === "loading") return <LoadingSpinner />;
  if (status === "unauthenticated") return null;

  // Kiểm tra role nếu có yêu cầu
  if (requiredRole && session?.user?.role !== requiredRole) {
    return fallback ?? <p>Bạn không có quyền xem nội dung này.</p>;
  }

  return <>{children}</>;
}

// Dùng trong page
function DashboardPage() {
  return (
    <AuthGuard>
      <h1>Dashboard — chỉ user đã đăng nhập</h1>
    </AuthGuard>
  );
}

function AdminSection() {
  return (
    <AuthGuard
      requiredRole="admin"
      fallback={<p>Chỉ admin mới xem được phần này.</p>}
    >
      <AdminPanel />
    </AuthGuard>
  );
}
```

### Server Component Guard

Trong Next.js App Router, kiểm tra auth trực tiếp trong Server Component:

```tsx
// app/dashboard/page.tsx
import { getServerSession } from "next-auth";
import { redirect } from "next/navigation";
import { authOptions } from "@/app/api/auth/[...nextauth]/route";

export default async function DashboardPage() {
  const session = await getServerSession(authOptions);

  if (!session) {
    redirect("/login");
  }

  if (session.user.role !== "admin") {
    redirect("/403");
  }

  // Session đảm bảo có data — không cần loading state
  return <Dashboard user={session.user} />;
}
```

### Tổng kết các tầng bảo vệ

```
Request
  ↓
Middleware (Edge)          ← Tầng 1: Redirect nhanh nhất, trước khi render
  ↓
Server Component Guard     ← Tầng 2: Kiểm tra trên server, không gửi HTML nếu sai
  ↓
Client Guard (AuthGuard)   ← Tầng 3: Bảo vệ UI sau khi hydration
  ↓
API Route Guard            ← Tầng 4: Bảo vệ dữ liệu — quan trọng nhất
```

> **Nguyên tắc vàng:** Không bao giờ chỉ bảo vệ route ở phía client. Dù ẩn button "Xóa" với user thường, API endpoint `DELETE /api/users/:id` phải luôn kiểm tra quyền độc lập trên server — vì attacker có thể gọi API trực tiếp bỏ qua UI.
