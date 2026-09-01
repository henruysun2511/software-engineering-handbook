# 🚀 Phỏng vấn Frontend: CSR vs SSR vs SSG

---

## 📌 PHẦN 1: CSR (CLIENT-SIDE RENDERING)

### Câu hỏi 1: CSR là gì?

**Trả lời:**

Client-Side Rendering: Browser downloads JavaScript, executes it, renders HTML.

**Flow:**
```
1. Browser requests HTML
2. Server sends empty HTML + JS bundle
3. Browser downloads JS (large)
4. Browser executes JS
5. Browser renders content
6. Content visible to user
```

**HTML skeleton:**
```html
<!DOCTYPE html>
<html>
<head>
  <title>My App</title>
</head>
<body>
  <div id="root"></div>  <!-- Empty! -->
  <script src="/app.js"></script>
</body>
</html>
```

### Câu hỏi 2: CSR - Ưu/nhược điểm?

**Ưu điểm:**
✅ Rich interactivity (infinite scroll, real-time updates)
✅ Smooth transitions (no full page reload)
✅ Offline support (Service Workers)
✅ Fast subsequent navigation (client-side routing)
✅ No server-side logic needed
✅ Easy to scale (just serve static files)
✅ Development flexibility

**Nhược điểm:**
❌ Large JavaScript bundle (slow initial load)
❌ Poor SEO (crawlers see empty HTML)
❌ Slow Time-to-Interactive (TTI)
❌ High power consumption on mobile
❌ Not accessible if JS disabled
❌ Blank page while loading (poor UX)

### Câu hỏi 3: CSR Performance issues?

**Trả lời:**

```javascript
// Typical CSR bundle size
app.js: 500KB (uncompressed) → 150KB (gzipped)

// Load timeline
0ms:    User clicks link
100ms:  Start downloading HTML (10KB)
150ms:  HTML received (empty)
150ms:  Download JS starts
1000ms: JS download complete (150KB gzipped takes time)
1100ms: JS execution starts
2000ms: Content rendered (TTI = 2 seconds!) 
3000ms: Interactive (hydration complete)

// Bad for UX:
- 1-2 second blank screen
- Content suddenly appears (CLS issue)
- User can't interact during JS parsing
```

**Solutions:**

```javascript
// 1. Code splitting (automatically by Next.js)
// Instead of: app.js (500KB)
// Use: layout.js (50KB) + pages/blog.js (100KB) + pages/products.js (120KB)

// 2. Tree shaking (remove unused code)
import { debounce } from 'lodash';  // ❌ Bad: imports all lodash

import { debounce } from 'lodash-es';  // ✅ Good: tree-shakeable

// 3. Lazy loading components
const Comments = lazy(() => import('./Comments'));

<Suspense fallback={<div>Loading...</div>}>
  <Comments />
</Suspense>

// 4. Preloading (prefetch critical resources)
<link rel="preload" href="/app.js" as="script" />
<link rel="prefetch" href="/blog.js" as="script" />

// 5. Image optimization
// CSR images usually loaded after JS
// Can be very slow!
```

---

## 🖥️ PHẦN 2: SSR (SERVER-SIDE RENDERING)

### Câu hỏi 1: SSR là gì?

**Trả lời:**

Server-Side Rendering: Server executes code, sends complete HTML to browser.

**Flow:**
```
1. Browser requests page
2. Server receives request
3. Server executes JavaScript/code
4. Server renders HTML
5. Server sends complete HTML
6. Browser displays content immediately
7. Browser downloads JS (for interactivity)
8. Browser hydrates (attaches event listeners)
```

**HTML with content:**
```html
<!DOCTYPE html>
<html>
<head>
  <title>My App</title>
</head>
<body>
  <div id="root">
    <h1>Welcome to My App</h1>
    <p>This content was rendered on server</p>
    <ul>
      <li>Item 1</li>
      <li>Item 2</li>
    </ul>
  </div>
  <script src="/app.js"></script>
</body>
</html>
```

### Câu hỏi 2: SSR - Ưu/nhược điểm?

**Ưu điểm:**
✅ Good SEO (search engines see complete HTML)
✅ Fast First Contentful Paint (FCP)
✅ Content visible immediately
✅ Works without JavaScript (graceful degradation)
✅ Better perceived performance
✅ Better for social sharing (full content in meta)
✅ No blank page (no FOUC - Flash Of Unstyled Content)

**Nhược điểm:**
❌ Slower Time-to-Interactive (TTFB slower)
❌ Server-side rendering takes time
❌ High server load
❌ Hard to scale (stateful servers)
❌ Can't do offline
❌ Full page reload on navigation (if no client-side routing)
❌ More complex (manage server + client)
❌ Session management needed

### Câu hỏi 3: SSR Performance timeline?

**Trả lời:**

```javascript
// SSR timeline
0ms:    User clicks link
50ms:   Request reaches server
100ms:  Server queries database
300ms:  Database returns data
400ms:  Server renders HTML
450ms:  Server sends HTML (fully formed!)
500ms:  Browser receives HTML
550ms:  Browser displays content (FCP) ✅ Much faster than CSR!
600ms:  Browser downloads JS (150KB)
1100ms: JS execution complete
1150ms: Hydration complete (interactive)

// Good:
- Content visible by 550ms (vs 2000ms in CSR)
- Better FCP
- Better for SEO

// Bad:
- TTFB (Time To First Byte) = 450ms
- Still need JS for interactivity
- Server load higher
```

### Câu hỏi 4: Hydration problem?

**Trả lời:**

Hydration: Process of attaching event listeners to server-rendered HTML.

```javascript
// Server renders:
<button onclick="...">Click me</button>

// Browser receives HTML, renders it
// Browser downloads JS, executes it
// React attaches event listeners to button
// Now button is interactive ✅

// Problem: Hydration mismatch
// Server rendered: <button>1</button>
// Client expects: <button>2</button>
// React error! Component doesn't hydrate properly

// Solution:
export default function Counter() {
  const [count, setCount] = useState(0);
  const [isMounted, setIsMounted] = useState(false);

  useEffect(() => {
    setIsMounted(true);
  }, []);

  // Don't render interactive content on server
  // or wrap in <NoSSR>
  if (!isMounted) return null;

  return <button>{count}</button>;
}
```

---

## 📦 PHẦN 3: SSG (STATIC SITE GENERATION)

### Câu hỏi 1: SSG là gì?

**Trả lời:**

Static Site Generation: Generate HTML at build time, serve pre-rendered pages.

**Flow:**
```
Build time (happens once):
1. Developer runs: npm run build
2. Build system fetches all data
3. Renders all pages to static HTML files
4. Uploads to CDN

Runtime (when user visits):
1. User requests page
2. CDN serves pre-built HTML
3. Browser displays content (INSTANT!)
4. Browser downloads JS
5. Hydration happens
```

**Build output:**
```bash
build/
├── blog/
│   ├── post-1.html     (pre-rendered)
│   ├── post-2.html     (pre-rendered)
│   └── post-3.html     (pre-rendered)
├── about.html          (pre-rendered)
└── index.html          (pre-rendered)

# All these are static files on CDN
# No server needed!
```

### Câu hỏi 2: SSG - Ưu/nhược điểm?

**Ưu điểm:**
✅ Incredibly fast (served from CDN, < 50ms)
✅ Best for SEO (complete HTML)
✅ Cheap hosting (just static files)
✅ Super scalable (no server needed)
✅ Secure (no backend to hack)
✅ Best performance (cached by CDN)
✅ Works offline
✅ No runtime overhead

**Nhược điểm:**
❌ Build time can be slow (10+ minutes for 10k pages)
❌ Can't do truly dynamic content
❌ Hard to update frequently (need rebuild)
❌ Not suitable for user-specific content (personalization)
❌ Limited to pre-buildable data
❌ Large builds use lots of memory

### Câu hỏi 3: SSG + ISR (Incremental Static Regeneration)?

**Trả lời:**

ISR: Rebuild specific pages without full rebuild.

```javascript
// pages/blog/[slug].js (Next.js)
export default function BlogPost({ post }) {
  return <h1>{post.title}</h1>;
}

export async function getStaticPaths() {
  const posts = await getAllBlogPosts();

  return {
    paths: posts.map(p => ({ params: { slug: p.slug } })),
    fallback: 'blocking'  // Generate new pages on demand
  };
}

export async function getStaticProps({ params }) {
  const post = await getBlogPost(params.slug);

  return {
    props: { post },
    revalidate: 3600  // ✅ Regenerate every hour!
  };
}

// Timeline:
// Build time: Generate top 100 popular posts
// Runtime: 
//   - First visit to post 101: Render on demand, cache it
//   - After 1 hour: Regenerate in background
//   - During regeneration: Serve stale version (fast!)
//   - After regeneration: Serve new version
```

**On-Demand ISR (Newer):**
```javascript
// pages/api/revalidate.js (Pages Router)
export default async function handler(req, res) {
  if (req.query.secret !== process.env.SECRET_TOKEN) {
    return res.status(401).json({ message: 'Invalid token' });
  }

  try {
    // Regenerate specific page
    await res.revalidate(`/blog/${req.query.slug}`);
    
    return res.json({ revalidated: true });
  } catch (err) {
    return res.status(500).json({ message: 'Error revalidating' });
  }
}

// Trigger from CMS:
// POST /api/revalidate?secret=TOKEN&slug=new-post
```

---

## ⚖️ PHẦN 4: CSR vs SSR vs SSG - SO SÁNH CHI TIẾT

### Performance Comparison Table

| Metric | CSR | SSR | SSG |
|--------|-----|-----|-----|
| **FCP** (First Contentful Paint) | 3-5s | 1-2s | 0.3-0.5s |
| **TTFB** (Time To First Byte) | 100ms | 500ms | 10ms |
| **TTI** (Time To Interactive) | 3-5s | 2-3s | 0.5-1s |
| **LCP** (Largest Contentful Paint) | 4-6s | 2-3s | 0.5-1s |
| **CLS** (Cumulative Layout Shift) | High | Medium | Low |
| **Server Cost** | Low | High | Very Low |
| **Build Time** | N/A | N/A | Slow (for many pages) |
| **Hosting Cost** | $5-10/mo | $50-500/mo | $10-50/mo (CDN) |
| **Scalability** | Excellent | Medium | Excellent |

### Feature Comparison Table

| Feature | CSR | SSR | SSG |
|---------|-----|-----|-----|
| **SEO** | Bad | Good | Excellent |
| **Initial Load** | Slow | Medium | Very Fast |
| **Dynamic Content** | Excellent | Good | Poor |
| **Real-time Updates** | Excellent | Good | Requires ISR |
| **User-specific Content** | Excellent | Good | Not suitable |
| **Offline Support** | Possible | Not possible | Possible |
| **Server Load** | None | High | None |
| **Development Complexity** | Simple | Medium | Simple |

### Use Case Comparison

| Use Case | Best Choice | Why |
|----------|-------------|-----|
| **Blog/Docs** | SSG | Content rarely changes, perfect for static |
| **News Site** | SSR + ISR | Content changes, need fresh data |
| **SaaS App** | CSR | User-specific, real-time, lots of interactivity |
| **Ecommerce** | SSG + ISR | Product pages static, update with ISR |
| **Real-time Dashboard** | CSR | Live updates, user-specific |
| **Personal Portfolio** | SSG | Static content, fast load |
| **Chat App** | CSR | Real-time, bi-directional |
| **Documentation** | SSG | Static content, searchable |

---

## 🎯 PHẦN 5: KỈ HUỐNG THỰC TẾ

### Scenario 1: Blog with 10,000 posts

**Requirements:**
- Posts rarely change
- Need fast load
- Good SEO important
- Limited budget

**Solution: SSG + ISR**

```javascript
// pages/blog/[slug].js
export async function getStaticPaths() {
  // Pre-build top 1000 popular posts
  const popularPosts = await getPopularPosts(1000);

  return {
    paths: popularPosts.map(p => ({
      params: { slug: p.slug }
    })),
    fallback: 'blocking'  // On-demand for less popular
  };
}

export async function getStaticProps({ params }) {
  const post = await getBlogPost(params.slug);

  if (!post) {
    return { notFound: true };
  }

  return {
    props: { post },
    revalidate: 86400  // Regenerate daily
  };
}

// Result:
// - Top 1000: Pre-built, instant load
// - 1000-10000: On-demand, cached in background
// - All pages: Fresh content daily
// - Cost: ~$30/month (CDN only)
```

---

### Scenario 2: Ecommerce with 100,000 products

**Requirements:**
- Many products (can't pre-build all)
- Products change frequently
- Price changes real-time
- Good SEO needed

**Solution: Hybrid SSG + ISR + Client-side**

```javascript
// pages/products/[id].js
export async function getStaticPaths() {
  // Pre-build best sellers
  const topProducts = await getTopProducts(5000);

  return {
    paths: topProducts.map(p => ({
      params: { id: p.id }
    })),
    fallback: 'blocking'
  };
}

export async function getStaticProps({ params }) {
  const product = await getProduct(params.id);
  const reviews = await getProductReviews(params.id);

  return {
    props: { product, reviews },
    revalidate: 3600  // Regenerate every hour
  };
}

// pages/products/[id].js (Client component)
export default function ProductPage({ product: initialProduct, reviews: initialReviews }) {
  const [product, setProduct] = useState(initialProduct);
  const [reviews, setReviews] = useState(initialReviews);
  const [loading, setLoading] = useState(false);

  // Refresh real-time data
  useEffect(() => {
    const interval = setInterval(async () => {
      setLoading(true);
      const freshProduct = await fetch(`/api/products/${product.id}`).then(r => r.json());
      setProduct(freshProduct);
      setLoading(false);
    }, 30000);  // Every 30 seconds

    return () => clearInterval(interval);
  }, [product.id]);

  return (
    <div>
      <h1>{product.name}</h1>
      <p>Price: ${product.price} {loading && <span>(updating...)</span>}</p>
      <p>Stock: {product.stock}</p>
      <Reviews reviews={reviews} />
    </div>
  );
}

// Result:
// - Page loads instantly from CDN
// - Data refreshed every hour
// - Real-time price/stock via client fetch
// - Good SEO (pre-rendered HTML)
// - Scalable (no server rendering)
```

---

### Scenario 3: SaaS Dashboard (User-specific)

**Requirements:**
- User-specific content
- Real-time updates
- No pre-built content
- High interactivity

**Solution: Pure CSR**

```javascript
// app/dashboard.js (Next.js App Router)
'use client';

import { useEffect, useState } from 'react';
import { useAuth } from '@/hooks/useAuth';

export default function Dashboard() {
  const { user } = useAuth();
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (!user) return;

    const fetchData = async () => {
      setLoading(true);
      try {
        const response = await fetch(`/api/dashboard/${user.id}`);
        const dashboardData = await response.json();
        setData(dashboardData);
      } finally {
        setLoading(false);
      }
    };

    fetchData();

    // Real-time updates
    const interval = setInterval(fetchData, 30000);

    return () => clearInterval(interval);
  }, [user?.id]);

  if (loading) return <div>Loading...</div>;

  return (
    <div>
      <h1>Dashboard for {user.name}</h1>
      <Stats data={data} />
      <RecentActivity data={data} />
    </div>
  );
}

// Result:
// - Fast navigation (client-side routing)
// - Real-time updates
// - User-specific content
// - High interactivity
// - Trade-off: First load slower (but worth it for SaaS)
```

---

### Scenario 4: News Site with breaking news

**Requirements:**
- News changes multiple times per day
- Need fresh content
- Good SEO important
- Mobile optimization
- Global CDN needed

**Solution: SSR + ISR**

```javascript
// pages/articles/[slug].js
export async function getServerSideProps({ params, res }) {
  const article = await getArticle(params.slug);

  if (!article) {
    return { notFound: true };
  }

  // Cache on CDN for 60 seconds
  res.setHeader(
    'Cache-Control',
    'public, s-maxage=60, stale-while-revalidate=120'
  );

  return {
    props: { article },
  };
}

export default function ArticlePage({ article }) {
  return (
    <article>
      <h1>{article.title}</h1>
      <time>{article.publishedAt}</time>
      <img src={article.image} alt={article.title} />
      <p>{article.content}</p>
    </article>
  );
}

// Result:
// - Server renders fresh content
// - CDN caches for 60 seconds
// - Breaking news visible within 60 seconds
// - Good SEO
// - Reasonable server load
```

---

## 🛠️ PHẦN 6: NEXTJS IMPLEMENTATION

### Setup CSR (App Router)

```javascript
// app/dashboard/page.js
'use client';  // ✅ Client component

import { useEffect, useState } from 'react';

export default function Dashboard() {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch('/api/data').then(r => r.json()).then(setData);
  }, []);

  return <div>{data?.message}</div>;
}

// Result: Empty HTML sent, JS renders content
```

### Setup SSR (Pages Router)

```javascript
// pages/ssr-page.js
export default function SSRPage({ data }) {
  return <h1>{data.message}</h1>;
}

export async function getServerSideProps() {
  const res = await fetch('https://api.example.com/data');
  const data = await res.json();

  return {
    props: { data },
    revalidate: 60  // Cache for 60 seconds
  };
}

// Result: Complete HTML sent from server
```

### Setup SSG (Pages Router)

```javascript
// pages/ssg-page.js
export default function SSGPage({ data, timestamp }) {
  return (
    <div>
      <h1>{data.message}</h1>
      <p>Generated at: {timestamp}</p>
    </div>
  );
}

export async function getStaticProps() {
  const data = await fetch('https://api.example.com/data').then(r => r.json());

  return {
    props: { 
      data,
      timestamp: new Date().toISOString()
    },
    revalidate: 3600  // Regenerate every hour (ISR)
  };
}

// Result: HTML built at build-time, cached forever (or until revalidate)
```

### Setup SSG with dynamic routes

```javascript
// pages/blog/[slug].js
export default function BlogPost({ post, fallback }) {
  if (fallback) {
    return <div>Loading...</div>;
  }

  return <h1>{post.title}</h1>;
}

export async function getStaticPaths() {
  // Pre-build popular posts
  const posts = await getAllPosts();
  const popular = posts.slice(0, 10);

  return {
    paths: popular.map(p => ({
      params: { slug: p.slug }
    })),
    fallback: 'blocking'  // On-demand for others
  };
}

export async function getStaticProps({ params }) {
  const post = await getPost(params.slug);

  if (!post) {
    return { notFound: true };
  }

  return {
    props: { post, fallback: false },
    revalidate: 3600
  };
}

// Timeline:
// 1. Popular posts: Pre-built (instant)
// 2. Other posts: On-demand (first visit slower)
// 3. All posts: Regenerated hourly
```

---

## 📊 PHẦN 7: PERFORMANCE BENCHMARKS

### Real-world measurements (Chrome DevTools)

**Blog post (static content):**

```
CSR approach:
- HTML: 15KB (empty)
- JS: 150KB (gzipped)
- CSS: 30KB (gzipped)
- Total: 195KB
- FCP: 2.8s (Time to see something)
- TTI: 3.5s (Time to interact)
- LCP: 3.2s (Main content visible)
- Total time: 3.5 seconds

SSR approach:
- HTML: 150KB (includes content)
- JS: 150KB (same JS)
- CSS: 30KB
- Total: 330KB (bigger!)
- FCP: 0.8s ✅ Much faster!
- TTI: 2.5s (JS needed for interactivity)
- LCP: 0.9s
- Total time: 2.5 seconds

SSG approach:
- HTML: 150KB (includes content)
- JS: 150KB (same JS)
- CSS: 30KB
- Total: 330KB
- FCP: 0.1s ✅ Instant (from CDN!)
- TTI: 0.8s
- LCP: 0.2s
- Total time: 0.8 seconds ✅ Winner!
```

**Mobile network (4G):**

```
CSR:
- Download JS: 3s
- Parse + render: 1s
- TTI: 4s ❌ Bad

SSR:
- Download HTML + CSS: 2s
- Render: 0.5s
- Download JS: 3s
- Hydrate: 0.5s
- TTI: 3.5s 

SSG:
- Download from CDN: 0.5s ✅
- Parse HTML: 0.1s
- Render: 0.2s
- Download JS: 3s
- Hydrate: 0.5s
- TTI: 1.3s ✅ Best!
```

### Bundle size impact

```
A 100KB gzipped JS file adds:
- CSR: 2 seconds FCP (must download + parse)
- SSR: 0.5 seconds (already HTML visible)
- SSG: 0.1 seconds (HTML instant)

CSR is 20x slower for initial load!
```

---

## 💡 PHẦN 8: WHEN TO USE WHAT

### Decision Tree

```
Is content static or changes rarely?
├─ YES
│   └─ SSG (best for SEO, performance, cost)
│       └─ Use ISR if need occasional updates
│
└─ NO, content changes frequently
    ├─ Can be pre-built? (know URLs beforehand)
    │   ├─ YES → SSG + ISR on-demand
    │   │   └─ Example: Ecommerce product pages
    │   │
    │   └─ NO → SSR
    │       └─ Example: News article from URL param
    │
    └─ Is user-specific?
        ├─ YES → CSR
        │   └─ Example: Dashboard, personalized content
        │
        └─ NO → SSR
            └─ Example: Dynamic content rendered from data
```

### Quick Reference

```
Blog / Documentation:
→ SSG (build once, serve forever)
→ ISR if updates needed

Ecommerce (products):
→ SSG top products + ISR on-demand
→ Fallback to CSR for filters/cart

News / Real-time:
→ SSR + CDN cache

SaaS Dashboard:
→ CSR (user-specific content)

Marketing Site:
→ SSG (static content, best performance)

Mobile App Webview:
→ CSR (already has JS context)
```

---

## 🔴 PHẦN 9: COMMON MISTAKES

### Mistake 1: Using CSR for everything

```javascript
// ❌ Bad: CSR for static blog
export default function BlogPost() {
  const [post, setPost] = useState(null);

  useEffect(() => {
    fetch(`/api/blog/${slug}`).then(r => r.json()).then(setPost);
  }, [slug]);

  // Bad for SEO, slow FCP
}

// ✅ Good: SSG for static blog
export async function getStaticProps({ params }) {
  const post = await getPost(params.slug);
  return { props: { post }, revalidate: 3600 };
}
```

### Mistake 2: SSG with too many pages

```javascript
// ❌ Bad: Building 100,000 pages at build time
// Causes: 2+ hour builds, server runs out of memory

export async function getStaticPaths() {
  const paths = await getAllProducts();  // 100,000 items!
  return {
    paths: paths.map(p => ({ params: { id: p.id } })),
    fallback: false  // ❌ No fallback = 404 for new products
  };
}

// ✅ Good: Pre-build popular, on-demand for rest
export async function getStaticPaths() {
  const popular = await getTopProducts(5000);
  return {
    paths: popular.map(p => ({ params: { id: p.id } })),
    fallback: 'blocking'  // ✅ On-demand for 5000+
  };
}
```

### Mistake 3: Not using ISR

```javascript
// ❌ Bad: No ISR = stale content
export async function getStaticProps() {
  const data = await fetchData();
  return { props: { data } };
  // No revalidate = data never updates!
}

// ✅ Good: ISR regenerates regularly
export async function getStaticProps() {
  const data = await fetchData();
  return { 
    props: { data },
    revalidate: 3600  // Regenerate every hour
  };
}
```

### Mistake 4: SSR when SSG would work

```javascript
// ❌ Bad: SSR for static content
export async function getServerSideProps() {
  // This runs for EVERY request
  // High server cost, slow response
  const data = await staticData();
  return { props: { data } };
}

// ✅ Good: SSG for static content
export async function getStaticProps() {
  // This runs at build-time once
  // CDN caches forever
  const data = await staticData();
  return { 
    props: { data },
    revalidate: 86400  // ISR daily
  };
}
```

### Mistake 5: Hydration mismatch (SSR)

```javascript
// ❌ Bad: Different content on server vs client
export default function Clock() {
  return <div>{new Date().toLocaleString()}</div>;
}
// Server: "2024-01-15 10:00:00"
// Client: "2024-01-15 10:00:05"
// Mismatch error!

// ✅ Good: useEffect for client-only content
export default function Clock() {
  const [time, setTime] = useState(null);

  useEffect(() => {
    setTime(new Date().toLocaleString());
  }, []);

  return <div>{time}</div>;
}
```

### Mistake 6: CSR images (poor LCP)

```javascript
// ❌ Bad: Images load after JS
export default function Hero() {
  const [image, setImage] = useState(null);

  useEffect(() => {
    fetch('/api/hero-image').then(setImage);
  }, []);

  return <img src={image} />;
  // Image loads AFTER JS execution
  // Bad LCP!
}

// ✅ Good: Image in HTML (SSG/SSR)
export default function Hero({ image }) {
  return <img src={image} alt="Hero" />;
}

export async function getStaticProps() {
  const image = await getHeroImage();
  return { props: { image }, revalidate: 3600 };
}
```

---

## ⚡ PHẦN 10: OPTIMIZATION STRATEGIES

### Hybrid approach (Best of all)

```javascript
// Combine SSG, SSR, CSR strategically

// 1. SSG: Static page structure
export async function getStaticProps() {
  return {
    props: { /* static data */ },
    revalidate: 3600
  };
}

// 2. SSR: Fresh data (e.g., price)
export async function getServerSideProps() {
  const price = await getCurrentPrice();
  return { props: { price } };
}

// 3. CSR: Interactive elements
'use client';
function Cart() {
  const [items, setItems] = useState([]);
  // Client-side cart logic
}

// Result: Best performance + freshness + interactivity
```

### Progressive Enhancement

```javascript
// 1. HTML content first (SSG)
// 2. CSS styling (included)
// 3. JavaScript enhancements (hydration)

// Works even if JS fails!

// ✅ Button works without JS
<form method="POST" action="/api/submit">
  <input name="email" />
  <button type="submit">Subscribe</button>
</form>

// ✅ JS enhances UX
<form onSubmit={handleSubmitWithJS}>
  <input onChange={validateEmail} />
  <button>Subscribe</button>
</form>
```

---

## 📋 PHẦN 11: INTERVIEW CHECKLIST

### Knowledge areas

```
□ Understand CSR, SSR, SSG definitions
□ Know advantages/disadvantages of each
□ Performance metrics (FCP, TTI, LCP)
□ SEO implications
□ Cost/scalability comparison
□ When to use which
□ Real-world scenarios
□ Next.js implementation details
□ ISR and on-demand revalidation
□ Hydration issues
□ Common mistakes
```

### Code knowledge

```
□ Implement SSG with getStaticProps
□ Implement SSR with getServerSideProps
□ Implement CSR with 'use client'
□ Handle dynamic routes
□ ISR implementation
□ On-demand revalidation
□ Error handling (notFound)
□ Fallback strategies
```

### Decision making

```
□ Can explain why choose X over Y
□ Trade-offs between options
□ Real-world impact on users
□ Business considerations
□ Cost optimization
□ Performance optimization
□ SEO optimization
```

---

## 📊 PHẦN 12: COMPARISON SUMMARY

### Quick comparison

```javascript
// The same blog post built with 3 approaches:

// 1. CSR
export default function Blog({ slug }) {
  const [post, setPost] = useState(null);
  useEffect(() => {
    fetch(`/api/blog/${slug}`)
      .then(r => r.json())
      .then(setPost);
  }, [slug]);
  return <h1>{post?.title}</h1>;
}
// Pros: Simple, flexible
// Cons: Slow FCP, bad SEO

// 2. SSR
export async function getServerSideProps({ params }) {
  const post = await db.post.find({ slug: params.slug });
  return { props: { post } };
}
export default function Blog({ post }) {
  return <h1>{post.title}</h1>;
}
// Pros: Good SEO, fresh data
// Cons: Server load, slower TTFB

// 3. SSG + ISR
export async function getStaticPaths() {
  const posts = await db.post.findAll();
  return {
    paths: posts.map(p => ({ params: { slug: p.slug } })),
    fallback: 'blocking'
  };
}
export async function getStaticProps({ params }) {
  const post = await db.post.find({ slug: params.slug });
  return {
    props: { post },
    revalidate: 3600
  };
}
export default function Blog({ post }) {
  return <h1>{post.title}</h1>;
}
// Pros: Super fast, good SEO, cheap
// Cons: Build time, can't be truly real-time
```

---

## 🎯 DECISION TREE (Final)

```
Content can be prerendered at build time?
├─ YES → SSG
│   ├─ Does content change?
│   │   ├─ Rarely → SSG only
│   │   └─ Regularly → SSG + ISR
│   │
│   └─ Known URLs? (can pre-build paths)
│       └─ YES → Perfect for SSG
│       └─ NO → Use fallback: 'blocking'
│
└─ NO
    ├─ Content is user-specific?
    │   └─ YES → CSR
    │       └─ Example: Dashboard, settings
    │
    └─ Content is dynamic but public?
        └─ SSR
            └─ Example: News, real-time data
```

---

## 💼 INTERVIEWER CHECKLIST

**Good answers show:**
✅ Deep understanding of trade-offs
✅ Real-world scenario handling
✅ Performance considerations
✅ SEO implications
✅ Cost/scalability thinking
✅ Code examples
✅ Decision-making framework

**Red flags:**
❌ "Use SSG for everything"
❌ "CSR is best for interactivity"
❌ No mention of ISR
❌ Don't know hydration
❌ Can't explain TTFB vs FCP
❌ No real-world examples

---

## 📝 PHẦN 13: PRACTICAL TIPS

### Local development

```bash
# Test SSG build
npm run build
npm run start

# Test SSR performance
curl -w "Time to first byte: %{time_starttransfer}s\n" http://localhost:3000/page

# Test CSR performance
# Use DevTools Lighthouse
# Check JavaScript execution time
```

### Monitoring

```javascript
// Track performance in production
import { getCLS, getFCP, getFID, getLCP, getTTFB } from 'web-vitals';

getCLS(console.log);
getFCP(console.log);
getFID(console.log);
getLCP(console.log);
getTTFB(console.log);
```

### Optimization checklist

```
For SSG:
□ Pre-build all common pages
□ Use ISR for frequently updated
□ Compress images
□ Code split JS
□ Use CDN

For SSR:
□ Cache responses
□ Use Redis for DB queries
□ Minimize database calls
□ Optimize queries
□ Use load balancer

For CSR:
□ Code split aggressively
□ Lazy load components
□ Preload critical JS
□ Minimize initial bundle
□ Defer non-critical JS
```

---

## 🎓 KEY TAKEAWAYS

```
CSR (Client-Side Rendering):
├─ Rendering happens in browser
├─ Empty initial HTML
├─ JavaScript executes to render
├─ Best for: User-specific, interactive content
├─ Worst for: SEO, initial performance
└─ Example: SaaS apps, dashboards

SSR (Server-Side Rendering):
├─ Rendering happens on server
├─ Complete HTML sent to browser
├─ Server-side computation time
├─ Best for: Dynamic content, good SEO
├─ Worst for: Server costs, complexity
└─ Example: News sites, real-time content

SSG (Static Site Generation):
├─ Rendering happens at build time
├─ Pre-built HTML files
├─ Served from CDN (instant)
├─ Best for: Performance, costs, SEO
├─ ISR updates pages after deploy
└─ Example: Blogs, docs, static sites

Choose based on:
1. Content freshness needs
2. User experience requirements
3. SEO importance
4. Budget constraints
5. Scalability needs
```

---

**Good luck with your interview! 🚀**
