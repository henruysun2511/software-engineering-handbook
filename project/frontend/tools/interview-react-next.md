# ⚛️ Phỏng vấn Frontend: React, Next.js vs Angular, Vue

---

## 📚 PHẦN 1: REACT CƠ BẢN

### Câu hỏi 1: React là gì? Nó khác jQuery/Vanilla JS như thế nào?

**Trả lời:**

React là một JavaScript library để xây dựng user interfaces bằng cách chia nhỏ giao diện thành các components có thể tái sử dụng.

**So sánh:**

```javascript
// ❌ Vanilla JS (Imperative - chỉ thị máy làm gì)
const button = document.querySelector('button');
let count = 0;

button.addEventListener('click', () => {
  count++;
  button.textContent = `Count: ${count}`;  // Phải cập nhật DOM thủ công
});

// ❌ Vấn đề:
// - Phải quản lý DOM thủ công
// - Nếu có 10 chỗ cần cập nhật → 10 lệnh DOM manipulation
// - Khó track state
// - Khó maintain khi project lớn

// ✅ React (Declarative - mô tả giao diện như thế nào)
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}

// ✅ Lợi ích:
// - Mô tả giao diện (UI = f(state))
// - React tự động cập nhật DOM
// - State quản lý rõ ràng
// - Dễ maintain
```

**Key differences:**

| Khía cạnh | Vanilla JS | jQuery | React |
|----------|-----------|--------|-------|
| **Approach** | Imperative (chỉ thị) | Imperative | Declarative (mô tả) |
| **DOM** | Direct manipulation | Easier manipulation | Virtual DOM (optimal) |
| **State** | Quản lý thủ công | Quản lý thủ công | Centralized (useState) |
| **Reusability** | Functions | jQuery plugins | Components |
| **Learning** | Easy start | Easy start | Steep learning curve |
| **Performance** | Good | Good | Excellent (large apps) |
| **Scalability** | Hard | Hard | Easy |

### Câu hỏi 2: Component là gì?

**Trả lời:**

Component là một phần tái sử dụng của giao diện, có thể nhận input (props) và trả về JSX mô tả giao diện.

```javascript
// ✅ Functional Component (hiện đại)
function Greeting({ name }) {
  return <h1>Hello, {name}!</h1>;
}

// ✅ Với useState hook
function Counter() {
  const [count, setCount] = useState(0);  // state

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </div>
  );
}

// Sử dụng component
export default function App() {
  return (
    <>
      <Greeting name="John" />
      <Counter />
    </>
  );
}

// Ưu điểm:
// - Reusable: Dùng Greeting component ở nhiều chỗ
// - Composable: Ghép nhiều component lại
// - Testable: Mỗi component có thể test riêng
// - Maintainable: Dễ sửa/update
```

### Câu hỏi 3: Props vs State - Sự khác biệt?

**Trả lời:**

```javascript
// Props: Dữ liệu từ parent truyền xuống child (read-only)
function Child({ name, age }) {
  // ❌ Không thể thay đổi props
  // name = "Jane";  // Error!
  
  return <p>{name} is {age} years old</p>;
}

function Parent() {
  return <Child name="John" age={25} />;
}

// State: Dữ liệu component quản lý, có thể thay đổi
function Counter() {
  const [count, setCount] = useState(0);  // state
  
  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}

// Quy tắc:
// - Props: từ parent → child, không thay đổi (immutable)
// - State: quản lý bên trong component, thay đổi được
// - Khi state thay đổi → Component re-render
// - Khi props thay đổi → Component re-render
```

**So sánh table:**

| Khía cạnh | Props | State |
|----------|-------|-------|
| **Source** | Parent component | Component tự quản lý |
| **Mutable** | Read-only | Có thể thay đổi |
| **Re-render** | Khi props thay đổi | Khi setState được gọi |
| **Usage** | Truyền data | Lưu trữ data |
| **Scope** | Cục bộ component | Có thể lift up |

### Câu hỏi 4: React Hooks là gì? useState, useEffect?

**Trả lời:**

Hooks là functions cho phép bạn sử dụng state và lifecycle features trong functional components.

**useState Hook:**

```javascript
// useState: Quản lý state trong functional component
function Counter() {
  // count: giá trị hiện tại
  // setCount: function thay đổi state
  // 0: initial value
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </div>
  );
}

// Multiple states
function Form() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  
  return (
    <>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <input value={email} onChange={(e) => setEmail(e.target.value)} />
    </>
  );
}
```

**useEffect Hook:**

```javascript
// useEffect: Xử lý side effects (API calls, timers, etc)
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Chạy khi component mount hoặc userId thay đổi
    setLoading(true);
    
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        setUser(data);
        setLoading(false);
      });
  }, [userId]);  // Dependency array: khi nào chạy lại

  return loading ? <p>Loading...</p> : <p>{user?.name}</p>;
}

// useEffect lifecycle:
// 1. [] (empty) → chạy 1 lần khi mount
// 2. [dep] → chạy lại khi dep thay đổi
// 3. không [] → chạy mỗi lần render

// Cleanup function (unmount)
useEffect(() => {
  const timer = setInterval(() => {
    console.log('Tick');
  }, 1000);
  
  return () => clearInterval(timer);  // Cleanup
}, []);
```

**Hooks quan trọng khác:**

```javascript
// useContext: Truy cập context (global state)
const theme = useContext(ThemeContext);

// useReducer: Quản lý state phức tạp
const [state, dispatch] = useReducer(reducer, initialState);

// useCallback: Memoize callback function
const handleClick = useCallback(() => {
  // ...
}, [dependencies]);

// useMemo: Memoize computed value
const expensiveValue = useMemo(() => {
  return complexCalculation(data);
}, [data]);

// useRef: Reference đến DOM element
const inputRef = useRef(null);
inputRef.current.focus();
```

### Câu hỏi 5: Rendering và Re-render - Làm sao tối ưu?

**Trả lời:**

**Khi component re-render:**
```javascript
function App() {
  const [count, setCount] = useState(0);

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      <ExpensiveChild />  // ❌ Re-render mỗi lần count thay đổi!
    </>
  );
}

// Vấn đề: ExpensiveChild re-render mỗi lần App render
// Giải pháp: Memoization
```

**Tối ưu 1: React.memo**

```javascript
// ❌ Child re-render khi parent render
function Child({ data }) {
  return <div>{data}</div>;
}

// ✅ Child chỉ re-render khi props thay đổi
const Child = React.memo(function Child({ data }) {
  return <div>{data}</div>;
});

function Parent() {
  const [count, setCount] = useState(0);

  return (
    <>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      <Child data={count} />  // Chỉ re-render khi count thay đổi
    </>
  );
}
```

**Tối ưu 2: useCallback**

```javascript
// ❌ Callback mới mỗi render → Child re-render
function Parent() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    console.log('Clicked');
  };

  return <Child onClick={handleClick} />;
}

// ✅ Callback cùng mỗi render (nếu dependency không đổi)
function Parent() {
  const [count, setCount] = useState(0);

  const handleClick = useCallback(() => {
    console.log('Clicked');
  }, []);  // Dependency array trống

  return <MemoizedChild onClick={handleClick} />;
}
```

**Tối ưu 3: useMemo**

```javascript
// ❌ Tính toán lại mỗi render
function Parent({ data }) {
  const filtered = data.filter(item => item.active);
  
  return <Child filteredData={filtered} />;
}

// ✅ Tính toán chỉ khi data thay đổi
function Parent({ data }) {
  const filtered = useMemo(
    () => data.filter(item => item.active),
    [data]
  );
  
  return <Child filteredData={filtered} />;
}
```

**Tối ưu 4: Code Splitting (Dynamic Import)**

```javascript
// ❌ Import toàn bộ component
import HeavyComponent from './HeavyComponent';

// ✅ Lazy load component
const HeavyComponent = React.lazy(
  () => import('./HeavyComponent')
);

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <HeavyComponent />
    </Suspense>
  );
}
```

---

## 🚀 PHẦN 2: NEXT.JS

### Câu hỏi 1: Next.js là gì? Tại sao dùng?

**Trả lời:**

Next.js là React framework cung cấp Server-Side Rendering, Static Generation, API routes, và nhiều features khác để xây dựng production-ready apps.

**Ưu điểm so với CRA (Create React App):**

```javascript
// CRA (Client-side only)
// Problem: Slow initial load, poor SEO, no server features
// Bundle: ~100KB JS cho Hello World

// Next.js (Full-stack)
// Advantage: SSR/SSG, API routes, Image optimization, SEO
// Bundle: Tối ưu hơn, code splitting automatic
```

**Key features:**

| Feature | CRA | Next.js |
|---------|-----|---------|
| **SSR** | ❌ | ✅ |
| **SSG** | ❌ | ✅ |
| **API Routes** | ❌ | ✅ |
| **Image Optimization** | ❌ | ✅ (next/image) |
| **Code Splitting** | Manual | Automatic |
| **Performance** | Good | Excellent |
| **SEO** | Bình thường | Excellent |

### Câu hỏi 2: App Router vs Pages Router?

**Trả lời:**

Next.js có 2 cách routing: Pages Router (cũ) và App Router (mới, khuyến khích).

**Pages Router (Cũ):**

```javascript
// pages/blog/[slug].js
export default function Post({ post }) {
  return <h1>{post.title}</h1>;
}

export async function getStaticProps({ params }) {
  const post = await getPost(params.slug);
  return { props: { post }, revalidate: 3600 };
}

export async function getStaticPaths() {
  const posts = await getAllPosts();
  return {
    paths: posts.map(p => ({ params: { slug: p.slug } })),
    fallback: 'blocking'
  };
}
```

**App Router (Mới):**

```javascript
// app/blog/[slug]/page.js
export async function generateMetadata({ params }) {
  const post = await getPost(params.slug);
  return { title: post.title };
}

export default async function PostPage({ params }) {
  const post = await getPost(params.slug);
  return <h1>{post.title}</h1>;
}

export async function generateStaticParams() {
  const posts = await getAllPosts();
  return posts.map(p => ({ slug: p.slug }));
}
```

**So sánh:**

| Khía cạnh | Pages Router | App Router |
|----------|--------------|-----------|
| **Location** | pages/ | app/ |
| **Server components** | Không | Có (default) |
| **API Routes** | pages/api/ | route.js |
| **Metadata** | next/head | generateMetadata |
| **Layouts** | Per page | Nested layouts |
| **Streams** | Không | Có |

**Khuyến khích: Dùng App Router (mới hơn, tốt hơn)**

### Câu hỏi 3: Server Components vs Client Components?

**Trả lời:**

**Server Components (Default):**

```javascript
// app/page.js
import { getAllPosts } from '@/lib/posts';

// ✅ Có thể dùng async/await trực tiếp
export default async function HomePage() {
  const posts = await getAllPosts();

  return (
    <div>
      {posts.map(post => (
        <div key={post.id}>{post.title}</div>
      ))}
    </div>
  );
}

// Ưu điểm:
// - Queries database trực tiếp
// - Không expose API secrets
// - Giảm bundle size
// - Tính toán nặng trên server
```

**Client Components:**

```javascript
// app/components/Counter.js
'use client';  // ✅ Mark as client component

import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}

// Khi dùng:
// - Hooks (useState, useEffect, etc)
// - Event listeners (onClick, onChange)
// - Browser APIs (localStorage, etc)
// - Interactive features
```

**Pattern tốt nhất:**

```javascript
// app/page.js (Server Component)
import Counter from '@/components/Counter';
import Posts from '@/components/Posts';

export default async function HomePage() {
  const posts = await getPosts();  // ✅ Server side

  return (
    <>
      <Posts posts={posts} />
      <Counter />  // ✅ Client component
    </>
  );
}

// app/components/Posts.js (Server Component)
export default function Posts({ posts }) {
  return (
    <div>
      {posts.map(p => <div key={p.id}>{p.title}</div>)}
    </div>
  );
}

// app/components/Counter.js (Client Component)
'use client';
import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>;
}
```

### Câu hỏi 4: Next.js API Routes?

**Trả lời:**

API Routes cho phép tạo backend endpoints trong Next.js project.

```javascript
// app/api/posts/route.js (App Router)
export async function GET(request) {
  const posts = await db.post.findAll();
  return Response.json(posts);
}

export async function POST(request) {
  const data = await request.json();
  const post = await db.post.create(data);
  return Response.json(post);
}

// app/api/posts/[id]/route.js
export async function GET(request, { params }) {
  const post = await db.post.findOne(params.id);
  return Response.json(post);
}

export async function PUT(request, { params }) {
  const data = await request.json();
  const post = await db.post.update(params.id, data);
  return Response.json(post);
}

export async function DELETE(request, { params }) {
  await db.post.delete(params.id);
  return Response.json({ success: true });
}

// pages/api/posts.js (Pages Router - cũ)
export default async function handler(req, res) {
  if (req.method === 'GET') {
    const posts = await db.post.findAll();
    res.status(200).json(posts);
  } else if (req.method === 'POST') {
    const post = await db.post.create(req.body);
    res.status(201).json(post);
  }
}
```

**Client usage:**

```javascript
'use client';
import { useState, useEffect } from 'react';

export default function PostsList() {
  const [posts, setPosts] = useState([]);

  useEffect(() => {
    fetch('/api/posts')
      .then(r => r.json())
      .then(setPosts);
  }, []);

  const handleCreate = async (title) => {
    const res = await fetch('/api/posts', {
      method: 'POST',
      body: JSON.stringify({ title })
    });
    const newPost = await res.json();
    setPosts([...posts, newPost]);
  };

  return <div>...</div>;
}
```

### Câu hỏi 5: Image Optimization trong Next.js?

**Trả lời:**

```javascript
import Image from 'next/image';

export default function Home() {
  return (
    <>
      {/* ❌ HTML img tag - không optimize */}
      <img src="/photo.jpg" alt="Photo" />

      {/* ✅ Next.js Image component */}
      <Image
        src="/photo.jpg"
        alt="Photo"
        width={600}
        height={400}
        quality={75}           // Compression
        priority={false}        // Lazy load
        placeholder="blur"      // Blur effect while loading
        sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
      />
    </>
  );
}

// Ưu điểm:
// - Automatic optimization (WebP, AVIF)
// - Responsive images (multiple sizes)
// - Lazy loading (default)
// - Blur placeholder
// - Prevents CLS (Cumulative Layout Shift)
```

---

## ⚖️ PHẦN 3: REACT VS ANGULAR VS VUE

### Comparision Table Chi tiết

| Khía cạnh | React | Angular | Vue |
|----------|-------|---------|-----|
| **Type** | Library | Full framework | Progressive framework |
| **Learning Curve** | Trung bình | Dốc (steep) | Dễ |
| **Bundle Size** | ~40KB | ~500KB | ~35KB |
| **Performance** | Excellent | Good | Excellent |
| **Company** | Meta (Facebook) | Google | Community |
| **Language** | JavaScript | TypeScript (required) | JavaScript/TypeScript |
| **HTML Template** | JSX | Template syntax | Template syntax |
| **Data Binding** | One-way | Two-way | Two-way |
| **State Management** | Redux, Zustand, etc | RxJS, NgRx | Vuex, Pinia |
| **Routing** | React Router | Angular Router | Vue Router |
| **Community** | Rất lớn | Lớn | Vừa |
| **Job Market** | Cao nhất | Vừa | Thấp |
| **Mobile** | React Native | NativeScript | NativeUI |

### Ví dụ 1: Rendering Hello World

**React:**
```javascript
import React from 'react';

function HelloWorld() {
  return <h1>Hello World</h1>;
}

export default HelloWorld;
```

**Angular:**
```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-hello-world',
  template: `<h1>Hello World</h1>`,
  styles: []
})
export class HelloWorldComponent {
}
```

**Vue:**
```vue
<template>
  <h1>Hello World</h1>
</template>

<script>
export default {
  name: 'HelloWorld'
}
</script>
```

### Ví dụ 2: Component với State

**React:**
```javascript
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}
```

**Angular:**
```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <button (click)="increment()">
      Count: {{ count }}
    </button>
  `
})
export class CounterComponent {
  count = 0;

  increment() {
    this.count++;
  }
}
```

**Vue:**
```vue
<template>
  <button @click="count++">
    Count: {{ count }}
  </button>
</template>

<script>
import { ref } from 'vue';

export default {
  setup() {
    const count = ref(0);
    return { count };
  }
}
</script>
```

### Ví dụ 3: API Call

**React:**
```javascript
import { useState, useEffect } from 'react';

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(setUser);
  }, [userId]);

  return user ? <p>{user.name}</p> : <p>Loading...</p>;
}
```

**Angular:**
```typescript
import { Component, OnInit, Input } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Component({
  selector: 'app-user-profile',
  template: `
    <p *ngIf="user">{{ user.name }}</p>
    <p *ngIf="!user">Loading...</p>
  `
})
export class UserProfileComponent implements OnInit {
  @Input() userId: number;
  user: any;

  constructor(private http: HttpClient) {}

  ngOnInit() {
    this.http.get(`/api/users/${this.userId}`)
      .subscribe(data => this.user = data);
  }
}
```

**Vue:**
```vue
<template>
  <p v-if="user">{{ user.name }}</p>
  <p v-else>Loading...</p>
</template>

<script>
import { ref, onMounted } from 'vue';

export default {
  props: ['userId'],
  setup(props) {
    const user = ref(null);

    onMounted(async () => {
      const res = await fetch(`/api/users/${props.userId}`);
      user.value = await res.json();
    });

    return { user };
  }
}
</script>
```

### Ví dụ 4: List Rendering

**React:**
```javascript
function ItemList({ items }) {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}
```

**Angular:**
```html
<ul>
  <li *ngFor="let item of items">{{ item.name }}</li>
</ul>
```

**Vue:**
```html
<ul>
  <li v-for="item in items" :key="item.id">
    {{ item.name }}
  </li>
</ul>
```

### Ví dụ 5: Two-way Data Binding

**React (One-way):**
```javascript
function Form() {
  const [name, setName] = useState('');

  return (
    <>
      <input 
        value={name} 
        onChange={(e) => setName(e.target.value)}
      />
      <p>Name: {name}</p>
    </>
  );
}
```

**Angular (Two-way):**
```html
<input [(ngModel)]="name" />
<p>Name: {{ name }}</p>
```

**Vue (Two-way):**
```vue
<template>
  <input v-model="name" />
  <p>Name: {{ name }}</p>
</template>

<script>
export default {
  data() {
    return {
      name: ''
    }
  }
}
</script>
```

---

## 🎯 PHẦN 4: KỈ HUỐNG THỰC TẾ

### Scenario 1: Chọn framework cho startup

**Yêu cầu:**
- Team nhỏ (3-5 developers)
- Cần shipping nhanh
- Budget hạn chế
- Không cần large enterprise structure

**Khuyến khích: React + Next.js**

```
Tại sao:
✅ Dễ tìm developers (demand cao)
✅ Có thể build full-stack một mình (API Routes)
✅ Eco-system lớn (libraries có sẵn)
✅ Flexible (tự chọn stack)
✅ Deployment dễ (Vercel hỗ trợ)
✅ Prototyping nhanh
```

### Scenario 2: Large enterprise project

**Yêu cầu:**
- Team lớn (50+ developers)
- Long-term maintenance
- Strict code standards
- TypeScript required

**Khuyến khích: Angular**

```
Tại sao:
✅ Opinionated (có structure rõ ràng)
✅ TypeScript integrated
✅ Large ecosystem (corporate)
✅ Enterprise support (Google)
✅ CLI tooling tốt
✅ Dependency injection, testing built-in
❌ Nhưng: Dốc learning curve, setup phức tạp
```

### Scenario 3: Side project / Portfolio

**Yêu cầu:**
- Học tập, showcase
- Ngắn hạn
- Độc lập

**Khuyến khích: Vue hoặc React**

```
Vue: Dễ bắt đầu, syntax dễ hiểu
React: Tốt cho portfolio (demand cao), skill transferable
```

### Scenario 4: SPA with real-time features

**Yêu cầu:**
- Bidirectional communication
- Real-time updates
- Heavy interactivity
- Complex state management

**Khuyến khích: React**

```
Tại sao:
✅ Ecosystem lớn (TanStack Query, Socket.io integ)
✅ State management options (Redux, Zustand)
✅ Flexible architecture
✅ Performance optimization tools
❌ Angular cũng được, nhưng setup phức tạp hơn
```

---

## 🔧 PHẦN 5: ADVANCED REACT PATTERNS

### Pattern 1: Container/Presentational Components

```javascript
// Presentational Component (dumb)
function UserCard({ user, onEdit, onDelete }) {
  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <button onClick={onEdit}>Edit</button>
      <button onClick={onDelete}>Delete</button>
    </div>
  );
}

// Container Component (smart)
function UserCardContainer({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Fetch data, handle logic
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(data => {
        setUser(data);
        setLoading(false);
      });
  }, [userId]);

  const handleEdit = () => {
    // Edit logic
  };

  const handleDelete = () => {
    // Delete logic
  };

  if (loading) return <div>Loading...</div>;

  return (
    <UserCard 
      user={user} 
      onEdit={handleEdit}
      onDelete={handleDelete}
    />
  );
}

// Ưu điểm:
// - Presentational dễ test, tái sử dụng
// - Logic tập trung ở container
// - Dễ maintain
```

### Pattern 2: Render Props

```javascript
// Render prop component
function withMousePosition(Component) {
  return function RenderProps() {
    const [position, setPosition] = useState({ x: 0, y: 0 });

    const handleMouseMove = (e) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };

    return (
      <div onMouseMove={handleMouseMove}>
        <Component position={position} />
      </div>
    );
  };
}

// Usage
const MouseTracker = withMousePosition(({ position }) => (
  <p>Mouse: {position.x}, {position.y}</p>
));
```

### Pattern 3: Custom Hooks

```javascript
// Reusable hook
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    setLoading(true);
    fetch(url)
      .then(r => r.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [url]);

  return { data, loading, error };
}

// Usage
function UserList() {
  const { data: users, loading } = useFetch('/api/users');
  
  return loading ? <p>Loading...</p> : <ul>{users.map(u => ...)}</ul>;
}
```

---

## 📊 PHẦN 6: PERFORMANCE TIPS

### React Performance

```javascript
// 1. Code Splitting
const HeavyComponent = React.lazy(() => import('./Heavy'));

// 2. Memoization
const MemoChild = React.memo(Child);

// 3. useCallback
const handleClick = useCallback(() => {}, []);

// 4. useMemo
const value = useMemo(() => expensive(), [deps]);

// 5. Key prop (khi render list)
{items.map(item => <Item key={item.id} />)}

// 6. useTransition (non-blocking updates)
const [isPending, startTransition] = useTransition();
startTransition(() => setFilter(value));

// 7. useReducer (complex state)
const [state, dispatch] = useReducer(reducer, initial);
```

### Next.js Performance

```javascript
// 1. SSG (Static)
export async function getStaticProps() { }

// 2. ISR (Incremental)
return { revalidate: 3600 };

// 3. Image optimization
import Image from 'next/image';

// 4. Font optimization
import { Inter } from 'next/font/google';

// 5. Script optimization
import Script from 'next/script';

// 6. Dynamic imports
const Component = dynamic(() => import('./Component'));
```

---

## 🎓 PHẦN 7: INTERVIEW TIPS

### Câu hỏi kỹ nước sâu

**1. Re-render mechanism**
```
Q: Khi nào component re-render?
A: - State thay đổi (setState)
   - Props thay đổi
   - Parent re-render
   - Context thay đổi
   - Key thay đổi trong list
```

**2. React Fiber**
```
Q: React Fiber là gì?
A: Architecture của React cho phép:
   - Pause, reuse, abort work
   - Prioritize different types of work
   - Support new concurrency features
```

**3. Virtual DOM**
```
Q: Virtual DOM có gì tốt?
A: - Efficient diffing algorithm
   - Batch updates
   - Optimal DOM manipulation
   - ❌ Không phải magic, cần optimize
```

**4. Uncontrolled vs Controlled components**
```javascript
// Controlled (recommended)
<input value={value} onChange={e => setValue(e.target.value)} />

// Uncontrolled (avoid)
<input defaultValue="initial" ref={ref} />
```

**5. Key prop importance**
```javascript
// ❌ Bad: index as key
{items.map((item, i) => <Item key={i} />)}

// ✅ Good: unique ID
{items.map(item => <Item key={item.id} />)}

// Why: Khi list reorder, index key gây bug
```

### Red flags trong answers

❌ "React is MVC"
✅ "React is a library for building UIs"

❌ "Virtual DOM is always faster"
✅ "Virtual DOM optimizes DOM updates strategically"

❌ "Never use index as key"
✅ "Use unique IDs as key, avoid index (except stable lists)"

❌ "props are immutable"
✅ "props should be treated as immutable"

---

## 📝 PHẦN 8: CHECKLIST PHỎNG VẤN

### React Core

```
□ Components (functional, class)
□ Props & State
□ Hooks (useState, useEffect, useContext, useReducer)
□ Rendering lifecycle
□ Re-render optimization
□ Event handling
□ Forms & controlled components
□ Lists & Keys
□ Conditional rendering
□ Error boundaries
```

### Advanced React

```
□ Custom hooks
□ Context API
□ Ref & useRef
□ Portals
□ Higher-order components
□ Render props
□ Code splitting & lazy loading
□ Suspense & concurrent features
□ Server components (React 18+)
```

### Next.js

```
□ App Router vs Pages Router
□ Server components vs Client components
□ getStaticProps, getServerSideProps, generateStaticParams
□ API Routes
□ Image optimization
□ Dynamic imports
□ Middleware
□ CSS/Styling solutions
□ Deployment
```

### Performance

```
□ React.memo
□ useCallback, useMemo
□ Code splitting
□ Bundle size analysis
□ Lighthouse audit
□ Core Web Vitals
□ Image optimization
□ Lazy loading
```

### Real-world

```
□ State management (Redux, Zustand, etc)
□ Data fetching (TanStack Query, SWR, etc)
□ Routing (React Router, Next.js)
□ Form handling (React Hook Form, Formik)
□ Testing (Jest, React Testing Library)
□ CSS solutions (Tailwind, CSS-in-JS, etc)
```

---

## 🚀 PHẦN 9: FINAL COMPARISON

### Một lần nhìn tổng quan

**React:**
- ✅ Tốt nhất cho jobs & learning
- ✅ Flexible & powerful
- ✅ Large ecosystem
- ❌ Cần tìm solutions riêng
- ❌ Khó optimize nếu không biết

**Angular:**
- ✅ Full framework (all-in-one)
- ✅ Enterprise support
- ✅ TypeScript built-in
- ❌ Dốc learning curve
- ❌ Overkill cho small projects

**Vue:**
- ✅ Dễ học nhất
- ✅ Great documentation
- ✅ Reactive by default
- ❌ Ecosystem nhỏ hơn
- ❌ Fewer job opportunities

### Recommendation Matrix

```
PROJECT TYPE          RECOMMENDATION
────────────────────────────────────
Startup               React + Next.js
Side project          Vue or React
Enterprise            Angular
SPA Dashboard         React
Static website        Next.js (SSG)
Real-time app         React + Socket.io
Learning              Vue (easiest) → React (most jobs)
Microservices         Angular or React
Mobile                React Native
```

---

## 📚 KEY TAKEAWAYS

```
1. React:
   - Declarative, component-based
   - Hooks for state & effects
   - Flexible ecosystem
   - Best job market

2. Next.js:
   - Framework for React
   - SSR, SSG, API routes
   - Image & performance optimization
   - Production-ready

3. Angular:
   - Full framework
   - TypeScript-first
   - Large enterprise projects
   - Steeper learning curve

4. Vue:
   - Easiest to learn
   - Progressive framework
   - Smaller ecosystem
   - Good for medium projects

5. Choose based on:
   - Team expertise
   - Project requirements
   - Job market
   - Long-term maintenance
   - Budget & timeline
```

---

**Good luck with your interview! 🎉**