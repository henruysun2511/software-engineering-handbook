# Chương 5: Next.js

Next.js là framework React được phát triển bởi Vercel, cung cấp các tính năng sẵn có như routing dựa trên hệ thống file, server-side rendering, static generation, tối ưu hình ảnh, và nhiều hơn nữa. Từ phiên bản 13, Next.js giới thiệu **App Router** — mô hình routing mới hoàn toàn dựa trên React Server Components.

---

## 5.1. App Router

### Cấu trúc thư mục `app/`

App Router sử dụng hệ thống file để định nghĩa route. Mỗi thư mục trong `app/` tương ứng với một segment của URL, và các file đặc biệt trong thư mục đó xác định UI của route.

```
app/
├── layout.tsx          # Layout gốc (bắt buộc)
├── page.tsx            # Trang chủ — route "/"
├── loading.tsx         # UI loading cho route "/"
├── error.tsx           # UI lỗi cho route "/"
├── not-found.tsx       # UI 404
├── globals.css
├── dashboard/
│   ├── layout.tsx      # Layout riêng cho /dashboard
│   ├── page.tsx        # Route "/dashboard"
│   └── settings/
│       └── page.tsx    # Route "/dashboard/settings"
└── api/
    └── users/
        └── route.ts    # API endpoint "/api/users"
```

Một segment được bao trong `[]` là **dynamic route**: `app/blog/[slug]/page.tsx` → `/blog/hello-world`.

---

### layout.tsx

`layout.tsx` định nghĩa UI bao ngoài (shell) được **chia sẻ** giữa nhiều page trong cùng segment. Layout không re-render khi điều hướng giữa các page con — state bên trong layout được giữ nguyên.

```tsx
// app/layout.tsx — Root Layout (bắt buộc có)
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: "My App",
  description: "Mô tả ứng dụng",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="vi">
      <body>
        <Header />
        <main>{children}</main>
        <Footer />
      </body>
    </html>
  );
}
```

```tsx
// app/dashboard/layout.tsx — Nested Layout
export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="dashboard">
      <Sidebar />
      <section>{children}</section>
    </div>
  );
}
```

---

### page.tsx

`page.tsx` là component đại diện cho UI của một route cụ thể. Chỉ `page.tsx` mới làm cho một route có thể truy cập công khai.

```tsx
// app/blog/[slug]/page.tsx
interface PageProps {
  params: Promise<{ slug: string }>;
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>;
}

export default async function BlogPost({ params }: PageProps) {
  const { slug } = await params;
  const post = await getPostBySlug(slug);

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  );
}
```

---

### loading.tsx

`loading.tsx` tự động bọc `page.tsx` và các component con trong một `<Suspense>` boundary. UI trong file này hiển thị ngay lập tức khi đang điều hướng đến route, trước khi dữ liệu của page sẵn sàng.

```tsx
// app/dashboard/loading.tsx
export default function DashboardLoading() {
  return (
    <div className="skeleton">
      <div className="skeleton-header" />
      <div className="skeleton-body" />
    </div>
  );
}
```

---

### error.tsx

`error.tsx` là Error Boundary tự động cho route. Khi có lỗi xảy ra trong `page.tsx`, Next.js hiển thị UI trong file này thay vì crash toàn bộ ứng dụng. File này **bắt buộc là Client Component**.

```tsx
// app/dashboard/error.tsx
"use client";

export default function DashboardError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div>
      <h2>Đã xảy ra lỗi!</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Thử lại</button>
    </div>
  );
}
```

---

### not-found.tsx

Hiển thị khi gọi hàm `notFound()` từ `next/navigation` hoặc khi không tìm thấy route khớp.

```tsx
// app/not-found.tsx
import Link from "next/link";

export default function NotFound() {
  return (
    <div>
      <h2>404 — Không tìm thấy trang</h2>
      <Link href="/">Về trang chủ</Link>
    </div>
  );
}
```

```tsx
// Dùng trong Server Component
import { notFound } from "next/navigation";

async function ProductPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const product = await getProduct(id);

  if (!product) notFound(); // Kích hoạt not-found.tsx

  return <div>{product.name}</div>;
}
```

---

### route.ts

`route.ts` định nghĩa **API endpoint** (Route Handler). File này không thể tồn tại cùng `page.tsx` trong cùng thư mục.

```ts
// app/api/users/route.ts
import { NextRequest, NextResponse } from "next/server";

export async function GET(request: NextRequest) {
  const users = await db.user.findMany();
  return NextResponse.json(users);
}

export async function POST(request: NextRequest) {
  const body = await request.json();
  const user = await db.user.create({ data: body });
  return NextResponse.json(user, { status: 201 });
}
```

---

### template.tsx (biết)

`template.tsx` tương tự `layout.tsx` nhưng **tạo instance mới** mỗi khi điều hướng — không giữ state. Dùng khi cần reset state hoặc chạy animation giữa các page.

```tsx
// app/template.tsx
export default function Template({ children }: { children: React.ReactNode }) {
  return <div className="page-transition">{children}</div>;
}
```

### So sánh layout.tsx vs template.tsx

| | `layout.tsx` | `template.tsx` |
|---|---|---|
| Re-render khi navigate | Không | Có |
| Giữ state | Có | Không |
| Dùng khi | Shell cố định (header, sidebar) | Cần animation, reset state |

---

## 5.2. Rendering

Next.js hỗ trợ nhiều chiến lược rendering, mỗi chiến lược phù hợp với loại nội dung khác nhau.

### CSR — Client-Side Rendering

HTML trống được gửi về, JavaScript chạy trên trình duyệt để fetch data và render UI. Đây là cách hoạt động của Single Page Application (SPA) thuần.

```tsx
"use client";
import { useEffect, useState } from "react";

function UserProfile() {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetch("/api/user").then((r) => r.json()).then(setUser);
  }, []);

  if (!user) return <p>Đang tải...</p>;
  return <h1>{user.name}</h1>;
}
```

---

### SSR — Server-Side Rendering

HTML được tạo ra **trên server mỗi khi có request**. Client nhận HTML đầy đủ ngay lập tức. Trong App Router, một Server Component async fetch data là SSR theo mặc định khi dữ liệu là dynamic.

```tsx
// app/dashboard/page.tsx — SSR mỗi request
export const dynamic = "force-dynamic"; // Tắt cache hoàn toàn

export default async function DashboardPage() {
  const data = await fetch("https://api.example.com/stats", {
    cache: "no-store", // Không cache — luôn fetch mới
  }).then((r) => r.json());

  return <StatsPanel data={data} />;
}
```

---

### SSG — Static Site Generation

HTML được tạo ra **một lần tại thời điểm build**. File HTML tĩnh được phục vụ cực nhanh qua CDN. Phù hợp với nội dung không thay đổi thường xuyên.

```tsx
// app/blog/[slug]/page.tsx — SSG
export async function generateStaticParams() {
  const posts = await getAllPosts();
  // Trả về danh sách params để Next.js build trước
  return posts.map((post) => ({ slug: post.slug }));
}

export default async function BlogPost({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  const post = await getPost(slug);
  return <article>{post.content}</article>;
}
```

---

### ISR — Incremental Static Regeneration

Kết hợp SSG và SSR: trang được build tĩnh nhưng **tự động làm mới sau một khoảng thời gian** (`revalidate`). Khi có request sau thời gian revalidate, Next.js trả về trang cũ ngay lập tức, đồng thời rebuild trang mới ở nền.

```tsx
// app/products/page.tsx — ISR, làm mới mỗi 60 giây
export const revalidate = 60;

export default async function ProductsPage() {
  const products = await fetch("https://api.example.com/products", {
    next: { revalidate: 60 },
  }).then((r) => r.json());

  return <ProductGrid products={products} />;
}
```

---

### Streaming

Streaming cho phép server gửi HTML về từng phần (chunk) khi sẵn sàng, thay vì chờ toàn bộ dữ liệu. Kết hợp với `<Suspense>`, từng phần của trang hiển thị ngay khi dữ liệu của nó đến — không cần chờ phần chậm nhất.

```tsx
// app/dashboard/page.tsx
import { Suspense } from "react";

export default function DashboardPage() {
  return (
    <div>
      <h1>Dashboard</h1>

      {/* Hiển thị ngay — không cần fetch */}
      <QuickStats />

      {/* Streaming — hiển thị khi data của từng phần sẵn sàng */}
      <Suspense fallback={<RevenueChartSkeleton />}>
        <RevenueChart />   {/* fetch chậm */}
      </Suspense>

      <Suspense fallback={<ActivityFeedSkeleton />}>
        <ActivityFeed />   {/* fetch nhanh hơn */}
      </Suspense>
    </div>
  );
}
```

### So sánh các chiến lược Rendering

| | CSR | SSR | SSG | ISR | Streaming |
|---|---|---|---|---|---|
| Render tại | Browser | Server (mỗi request) | Build time | Build + tự động | Server (từng phần) |
| Tốc độ ban đầu | Chậm | Trung bình | Nhanh nhất | Nhanh | Nhanh (progressive) |
| Data mới nhất | Có | Có | Không | Gần đúng | Có |
| SEO | Kém | Tốt | Tốt nhất | Tốt | Tốt |
| Dùng cho | App nội bộ, dashboard | Trang cần data realtime | Blog, docs | Sản phẩm, giá cả | Trang có nhiều phần phụ thuộc |

---

## 5.3. Data Fetching

### fetch() trong Server Component

Next.js mở rộng Web `fetch()` API với các tùy chọn cache tích hợp sẵn. Trong Server Component, có thể gọi `fetch()` trực tiếp, không cần `useEffect`.

```tsx
// Fetch đơn giản trong Server Component
async function getUsers() {
  const res = await fetch("https://api.example.com/users");
  if (!res.ok) throw new Error("Không thể lấy danh sách users");
  return res.json();
}

export default async function UsersPage() {
  const users = await getUsers();
  return <UserList users={users} />;
}
```

---

### Cache

Next.js mặc định cache kết quả của `fetch()`. Có thể kiểm soát hành vi cache qua tùy chọn `cache`:

```tsx
// Cache mãi mãi (SSG behavior) — mặc định trong Next.js 14 trở về trước
const data = await fetch(url, { cache: "force-cache" });

// Không cache, luôn fetch mới (SSR behavior)
const data = await fetch(url, { cache: "no-store" });
```

---

### Revalidate

Revalidate xác định thời gian (giây) sau đó cache sẽ bị làm mới (ISR behavior):

```tsx
// Làm mới cache sau 3600 giây (1 tiếng)
const data = await fetch(url, {
  next: { revalidate: 3600 },
});
```

Revalidate theo tag — làm mới đúng nhóm dữ liệu cần thiết:

```tsx
// Gắn tag khi fetch
const data = await fetch(url, {
  next: { tags: ["products"] },
});

// Làm mới theo tag (thường dùng trong Server Action)
import { revalidateTag } from "next/cache";
revalidateTag("products"); // Xóa cache tất cả fetch có tag "products"
```

---

### Dynamic Rendering

Next.js tự động chọn static hoặc dynamic rendering dựa trên API được dùng trong route:

| API được dùng | Hành vi |
|---|---|
| `fetch` với `cache: "force-cache"` | Static |
| `fetch` với `cache: "no-store"` | Dynamic |
| `cookies()`, `headers()` | Dynamic |
| `searchParams` | Dynamic |
| `export const dynamic = "force-dynamic"` | Luôn dynamic |

```tsx
import { cookies } from "next/headers";

// Route này tự động là dynamic vì dùng cookies()
export default async function Page() {
  const cookieStore = await cookies();
  const token = cookieStore.get("token");
  // ...
}
```

---

### generateStaticParams()

Dùng với dynamic routes để Next.js biết trước danh sách các params cần build tĩnh.

```tsx
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await fetch("https://api.example.com/posts").then((r) =>
    r.json()
  );

  // Trả về mảng object với key là tên param
  return posts.map((post: { slug: string }) => ({
    slug: post.slug,
  }));
}

// Next.js sẽ build sẵn trang cho mỗi slug trong danh sách trên
export default async function BlogPost({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  const post = await getPost(slug);
  return <article>{post.content}</article>;
}
```

---

## 5.4. React Server Component (RSC)

### Khái niệm

React Server Component là component chỉ render **trên server**, không bao giờ gửi JavaScript của component đó xuống client. Kết quả render (HTML/RSC payload) được stream xuống browser.

Trong Next.js App Router, **mọi component đều là Server Component theo mặc định** — trừ khi khai báo `"use client"`.

### Khi nào dùng?

- Khi cần truy cập database, filesystem, hoặc các tài nguyên server.
- Khi cần fetch data mà không muốn expose API key ra client.
- Khi component không cần tương tác (không có event handler, không dùng state hay effect).
- Khi muốn giảm JavaScript bundle gửi về client.

```tsx
// app/products/page.tsx — Server Component
import { db } from "@/lib/db";

export default async function ProductsPage() {
  // Truy cập DB trực tiếp — an toàn vì chạy trên server
  const products = await db.product.findMany({
    orderBy: { createdAt: "desc" },
  });

  return (
    <div>
      {products.map((product) => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}
```

### Ưu và nhược điểm

| | Server Component |
|---|---|
| **Ưu điểm** | Không gửi component JS xuống client → bundle nhỏ hơn |
| | Truy cập trực tiếp DB, filesystem, secrets |
| | Fetch data không qua API route |
| | Render nhanh hơn (không hydration) |
| **Nhược điểm** | Không dùng được `useState`, `useEffect`, hooks |
| | Không dùng được event handler (`onClick`, ...) |
| | Không truy cập được browser APIs (`window`, `document`) |

---

## 5.5. Client Component

### "use client"

`"use client"` là directive đặt ở đầu file, đánh dấu component đó (và tất cả import của nó) là Client Component — tức là sẽ được hydrate và chạy trên browser.

```tsx
"use client";

import { useState } from "react";

export default function Counter() {
  const [count, setCount] = useState(0);
  return (
    <button onClick={() => setCount((c) => c + 1)}>
      Đã click {count} lần
    </button>
  );
}
```

### Khi nào dùng?

- Khi cần `useState`, `useReducer`, `useEffect`, hoặc các hooks khác.
- Khi cần event handlers (`onClick`, `onChange`, `onSubmit`...).
- Khi cần truy cập browser APIs (`window`, `localStorage`, `navigator`...).
- Khi dùng thư viện bên thứ ba chưa hỗ trợ RSC.

### Giới hạn

- Client Component **không thể** là async component.
- Client Component **không thể** import Server Component trực tiếp.
- Client Component **có thể** nhận Server Component làm `children` (composition pattern).

```tsx
// Pattern đúng — truyền Server Component qua children
// app/dashboard/page.tsx (Server Component)
import ClientWrapper from "./ClientWrapper";
import ServerData from "./ServerData"; // Server Component

export default function Page() {
  return (
    <ClientWrapper>
      <ServerData /> {/* Hợp lệ — truyền qua children */}
    </ClientWrapper>
  );
}
```

### So sánh Server Component vs Client Component

| | Server Component | Client Component |
|---|---|---|
| Directive | Không có (mặc định) | `"use client"` |
| Render tại | Server | Server (initial) + Client (hydration) |
| State & Hooks | Không | Có |
| Event Handlers | Không | Có |
| Truy cập DB/FS | Có | Không |
| Browser APIs | Không | Có |
| Bundle size | Không tăng | Tăng theo kích thước component |

---

## 5.6. Server Action

Server Action là hàm async chạy **trên server**, có thể được gọi trực tiếp từ Client Component hoặc form. Đây là cách xử lý form submission và data mutation mà không cần tạo API route riêng.

Khai báo bằng directive `"use server"`.

```tsx
// app/actions/user.ts
"use server";

import { revalidateTag } from "next/cache";
import { db } from "@/lib/db";

export async function createUser(formData: FormData) {
  const name = formData.get("name") as string;
  const email = formData.get("email") as string;

  await db.user.create({ data: { name, email } });

  revalidateTag("users"); // Làm mới cache sau khi tạo
}
```

```tsx
// app/components/CreateUserForm.tsx
"use client";

import { createUser } from "@/app/actions/user";

export default function CreateUserForm() {
  return (
    <form action={createUser}>
      <input name="name" placeholder="Tên" required />
      <input name="email" type="email" placeholder="Email" required />
      <button type="submit">Tạo user</button>
    </form>
  );
}
```

Server Action với `useActionState` (React 19 / Next.js 15):

```tsx
"use client";

import { useActionState } from "react";
import { createUser } from "@/app/actions/user";

export default function CreateUserForm() {
  const [state, action, isPending] = useActionState(createUser, null);

  return (
    <form action={action}>
      <input name="name" required />
      {state?.error && <p className="error">{state.error}</p>}
      <button type="submit" disabled={isPending}>
        {isPending ? "Đang tạo..." : "Tạo user"}
      </button>
    </form>
  );
}
```

---

## 5.7. Middleware

Middleware chạy **trước khi request đến route handler hay page**, cho phép kiểm tra, redirect, rewrite, hoặc thêm header. File `middleware.ts` đặt ở root của project (cùng cấp với `app/`).

```ts
// middleware.ts
import { NextRequest, NextResponse } from "next/server";

export function middleware(request: NextRequest) {
  const token = request.cookies.get("token")?.value;
  const isAuthPage = request.nextUrl.pathname.startsWith("/login");
  const isProtected = request.nextUrl.pathname.startsWith("/dashboard");

  // Chưa đăng nhập, truy cập trang bảo vệ → redirect về login
  if (isProtected && !token) {
    return NextResponse.redirect(new URL("/login", request.url));
  }

  // Đã đăng nhập, vào trang login → redirect về dashboard
  if (isAuthPage && token) {
    return NextResponse.redirect(new URL("/dashboard", request.url));
  }

  return NextResponse.next();
}

// Chỉ chạy middleware cho các path này
export const config = {
  matcher: ["/dashboard/:path*", "/login"],
};
```

---

## 5.8. Route Handler

Route Handler (file `route.ts`) là API endpoint trong App Router, tương đương `pages/api/` trong Pages Router. Hỗ trợ đầy đủ các HTTP methods: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`.

```ts
// app/api/products/[id]/route.ts
import { NextRequest, NextResponse } from "next/server";

interface RouteContext {
  params: Promise<{ id: string }>;
}

export async function GET(request: NextRequest, { params }: RouteContext) {
  const { id } = await params;
  const product = await db.product.findUnique({ where: { id } });

  if (!product) {
    return NextResponse.json({ error: "Không tìm thấy" }, { status: 404 });
  }

  return NextResponse.json(product);
}

export async function PATCH(request: NextRequest, { params }: RouteContext) {
  const { id } = await params;
  const body = await request.json();
  const updated = await db.product.update({ where: { id }, data: body });
  return NextResponse.json(updated);
}

export async function DELETE(_request: NextRequest, { params }: RouteContext) {
  const { id } = await params;
  await db.product.delete({ where: { id } });
  return new NextResponse(null, { status: 204 });
}
```

Route Handler cũng có thể đọc query params và headers:

```ts
export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const page = searchParams.get("page") ?? "1";
  const authorization = request.headers.get("authorization");
  // ...
}
```

---

## 5.9. Metadata & SEO

Next.js cung cấp **Metadata API** tích hợp sẵn để quản lý `<head>` của trang, hỗ trợ SEO mà không cần thư viện ngoài.

### Metadata API (Static)

Export object `metadata` từ `layout.tsx` hoặc `page.tsx`:

```tsx
// app/about/page.tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: "Về chúng tôi | My App",
  description: "Tìm hiểu về đội ngũ và sứ mệnh của chúng tôi.",
  keywords: ["about", "team", "mission"],
};

export default function AboutPage() {
  return <h1>Về chúng tôi</h1>;
}
```

Dùng `title.template` trong Root Layout để tránh lặp tên app:

```tsx
// app/layout.tsx
export const metadata: Metadata = {
  title: {
    default: "My App",
    template: "%s | My App", // "Về chúng tôi | My App"
  },
  description: "...",
};
```

---

### Dynamic Metadata

Dùng hàm `generateMetadata()` khi metadata phụ thuộc vào dữ liệu động (như tên sản phẩm, tiêu đề bài viết):

```tsx
// app/products/[id]/page.tsx
import type { Metadata } from "next";

interface Props {
  params: Promise<{ id: string }>;
}

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { id } = await params;
  const product = await getProduct(id);

  return {
    title: product.name,
    description: product.description,
    openGraph: {
      title: product.name,
      description: product.description,
      images: [{ url: product.imageUrl }],
    },
  };
}
```

---

### Open Graph

Open Graph cho phép kiểm soát cách trang hiển thị khi chia sẻ trên mạng xã hội (Facebook, Twitter, LinkedIn):

```tsx
export const metadata: Metadata = {
  openGraph: {
    title: "Tên trang",
    description: "Mô tả hiển thị khi share",
    url: "https://example.com",
    siteName: "My App",
    images: [
      {
        url: "https://example.com/og-image.png",
        width: 1200,
        height: 630,
        alt: "Mô tả ảnh",
      },
    ],
    type: "website",
  },
  twitter: {
    card: "summary_large_image",
    title: "Tên trang",
    description: "Mô tả",
    images: ["https://example.com/og-image.png"],
  },
};
```

---

### robots.txt

Hướng dẫn công cụ tìm kiếm nào được phép và không được phép crawl trang:

```ts
// app/robots.ts
import type { MetadataRoute } from "next";

export default function robots(): MetadataRoute.Robots {
  return {
    rules: [
      {
        userAgent: "*",
        allow: "/",
        disallow: ["/admin/", "/api/", "/private/"],
      },
    ],
    sitemap: "https://example.com/sitemap.xml",
  };
}
```

---

### sitemap.xml

Sitemap giúp công cụ tìm kiếm khám phá tất cả các trang của website:

```ts
// app/sitemap.ts
import type { MetadataRoute } from "next";

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const posts = await getAllPosts();

  const postUrls = posts.map((post) => ({
    url: `https://example.com/blog/${post.slug}`,
    lastModified: post.updatedAt,
    changeFrequency: "weekly" as const,
    priority: 0.8,
  }));

  return [
    {
      url: "https://example.com",
      lastModified: new Date(),
      changeFrequency: "yearly",
      priority: 1,
    },
    ...postUrls,
  ];
}
```

---

### Structured Data (JSON-LD)

JSON-LD là định dạng schema giúp công cụ tìm kiếm hiểu ngữ nghĩa nội dung, hiển thị rich results (rich snippets) trên Google:

```tsx
// app/products/[id]/page.tsx
export default async function ProductPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  const product = await getProduct(id);

  const jsonLd = {
    "@context": "https://schema.org",
    "@type": "Product",
    name: product.name,
    description: product.description,
    image: product.imageUrl,
    offers: {
      "@type": "Offer",
      price: product.price,
      priceCurrency: "VND",
      availability: "https://schema.org/InStock",
    },
  };

  return (
    <>
      {/* Nhúng JSON-LD vào <head> */}
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }}
      />
      <h1>{product.name}</h1>
    </>
  );
}
```

### Tổng quan SEO trong Next.js

| Kỹ thuật | Mục đích | Cách triển khai |
|---|---|---|
| Metadata API | Title, description cho tab và SERP | `export const metadata` |
| Dynamic Metadata | Metadata theo nội dung động | `generateMetadata()` |
| Open Graph | Preview khi share mạng xã hội | Trong `metadata.openGraph` |
| robots.txt | Hướng dẫn crawler | `app/robots.ts` |
| sitemap.xml | Giúp crawler khám phá trang | `app/sitemap.ts` |
| JSON-LD | Rich results trên Google | `<script type="application/ld+json">` |
