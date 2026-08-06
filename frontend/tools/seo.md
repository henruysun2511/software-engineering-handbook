# TỔNG HỢP KIẾN THỨC SEO TRONG NEXT.JS

> **Chủ đề:** Search Engine Optimization với Next.js – Kỹ thuật, API và Chiến lược  
> **Phiên bản tham chiếu:** Next.js 13+ (App Router) & Next.js 12 (Pages Router)

---

## MỤC LỤC

1. [Tổng quan về SEO và Next.js](#1-tổng-quan-về-seo-và-nextjs)
2. [Metadata API (App Router)](#2-metadata-api-app-router)
3. [Head Component (Pages Router)](#3-head-component-pages-router)
4. [Rendering Strategy và SEO](#4-rendering-strategy-và-seo)
5. [Open Graph và Social Media Tags](#5-open-graph-và-social-media-tags)
6. [Sitemap](#6-sitemap)
7. [robots.txt](#7-robotstxt)
8. [Canonical URL](#8-canonical-url)
9. [Structured Data – JSON-LD](#9-structured-data--json-ld)
10. [Tối ưu hình ảnh với next/image](#10-tối-ưu-hình-ảnh-với-nextimage)
11. [Tối ưu Font với next/font](#11-tối-ưu-font-với-nextfont)
12. [Core Web Vitals](#12-core-web-vitals)
13. [Dynamic SEO theo trang](#13-dynamic-seo-theo-trang)
14. [Bảng tổng hợp và Checklist SEO](#14-bảng-tổng-hợp-và-checklist-seo)

---

## 1. Tổng quan về SEO và Next.js

### 1.1 SEO là gì?

**SEO (Search Engine Optimization)** là tập hợp các kỹ thuật tối ưu hóa website nhằm tăng thứ hạng hiển thị trên các công cụ tìm kiếm (Google, Bing, ...) một cách tự nhiên (organic), không trả phí.

Các yếu tố SEO quan trọng bao gồm:
- **On-page SEO:** Nội dung, thẻ meta, tiêu đề, URL chuẩn
- **Technical SEO:** Tốc độ tải trang, khả năng crawl, cấu trúc dữ liệu
- **Off-page SEO:** Backlinks, mạng xã hội, uy tín domain

### 1.2 Tại sao Next.js phù hợp cho SEO?

Next.js giải quyết vấn đề cốt lõi mà các SPA (Single Page Application) như React thuần gặp phải: **công cụ tìm kiếm khó đọc nội dung được render bởi JavaScript**.

| Framework | Cách render | Bot có đọc được? | SEO |
|-----------|-------------|-----------------|-----|
| React (CRA) | Client-side (CSR) | ⚠️ Phụ thuộc JS | Yếu |
| Next.js (SSR) | Server-side | ✅ HTML đầy đủ | Mạnh |
| Next.js (SSG) | Build-time | ✅ HTML tĩnh | Rất mạnh |
| Next.js (ISR) | Incremental | ✅ HTML + cập nhật | Mạnh + linh hoạt |

### 1.3 Kiến trúc Next.js và SEO

```
Request của Bot/User
        ↓
   Next.js Server
        ↓
  [SSR / SSG / ISR]
        ↓
   HTML đầy đủ ← Bot đọc được ngay
        ↓
  Hydration (JS)
        ↓
   Interactive App
```

Next.js trả về **HTML đã có nội dung** từ server, giúp bot lập chỉ mục (index) chính xác và nhanh hơn.

---

## 2. Metadata API (App Router)

### 2.1 Định nghĩa

Từ Next.js 13 với **App Router**, Next.js cung cấp **Metadata API** – một hệ thống khai báo metadata (thẻ meta, tiêu đề, mô tả, ...) trực tiếp trong file `layout.tsx` hoặc `page.tsx` mà không cần thao tác với `<head>` thủ công.

Có hai cách khai báo: **Static Metadata** và **Dynamic Metadata**.

### 2.2 Static Metadata

Dùng khi thông tin SEO cố định, không phụ thuộc dữ liệu bên ngoài.

```tsx
// app/layout.tsx hoặc app/about/page.tsx
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'Tên Website | Slogan ngắn gọn',
  description: 'Mô tả trang web tối đa 160 ký tự, tóm tắt nội dung chính để hiển thị trên kết quả tìm kiếm.',
  keywords: ['next.js', 'seo', 'react', 'web development'],
  authors: [{ name: 'Tên tác giả', url: 'https://example.com' }],
  creator: 'Tên công ty',
  robots: {
    index: true,
    follow: true,
  },
};

export default function Layout({ children }) {
  return <html><body>{children}</body></html>;
}
```

**HTML được tạo ra:**
```html
<title>Tên Website | Slogan ngắn gọn</title>
<meta name="description" content="Mô tả trang web tối đa 160 ký tự...">
<meta name="keywords" content="next.js, seo, react, web development">
<meta name="robots" content="index, follow">
```

### 2.3 Dynamic Metadata

Dùng khi thông tin SEO phụ thuộc vào dữ liệu động (ví dụ: trang sản phẩm, bài viết blog).

```tsx
// app/products/[id]/page.tsx
import type { Metadata } from 'next';

// Nhận params từ URL động
type Props = {
  params: { id: string };
};

// Hàm generateMetadata – Next.js gọi tự động khi build/request
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  // Gọi API để lấy thông tin sản phẩm
  const product = await fetch(`https://api.example.com/products/${params.id}`)
    .then(res => res.json());

  return {
    title: `${product.name} | Shop`,
    description: product.description.slice(0, 160),
    openGraph: {
      title: product.name,
      description: product.description,
      images: [{ url: product.imageUrl }],
    },
  };
}

export default async function ProductPage({ params }: Props) {
  const product = await fetch(`https://api.example.com/products/${params.id}`)
    .then(res => res.json());

  return <div><h1>{product.name}</h1></div>;
}
```

### 2.4 Title Template

Cho phép định nghĩa cấu trúc tiêu đề toàn cục và các trang con chỉ cần khai báo phần riêng.

```tsx
// app/layout.tsx – định nghĩa template ở root
export const metadata: Metadata = {
  title: {
    template: '%s | Tên Website',  // %s = tiêu đề của trang con
    default: 'Tên Website',         // Dùng khi trang con không có title
  },
};

// app/about/page.tsx – chỉ cần khai báo phần riêng
export const metadata: Metadata = {
  title: 'Giới thiệu',  // Kết quả: "Giới thiệu | Tên Website"
};

// app/contact/page.tsx
export const metadata: Metadata = {
  title: 'Liên hệ',  // Kết quả: "Liên hệ | Tên Website"
};
```

---

## 3. Head Component (Pages Router)

### 3.1 Định nghĩa

Với **Pages Router** (Next.js 12 trở về trước hoặc thư mục `pages/`), Next.js cung cấp component `<Head>` từ package `next/head` để chèn các thẻ vào phần `<head>` của HTML.

### 3.2 Cú pháp cơ bản

```tsx
// pages/index.tsx
import Head from 'next/head';

export default function HomePage() {
  return (
    <>
      <Head>
        <title>Trang chủ | Website của tôi</title>
        <meta name="description" content="Mô tả trang chủ, tối đa 160 ký tự." />
        <meta name="viewport" content="width=device-width, initial-scale=1" />
        <meta name="robots" content="index, follow" />
        <link rel="canonical" href="https://example.com/" />
        <link rel="icon" href="/favicon.ico" />
      </Head>
      <main>
        <h1>Chào mừng đến website</h1>
      </main>
    </>
  );
}
```

### 3.3 SEO động với getServerSideProps / getStaticProps

```tsx
// pages/blog/[slug].tsx
import Head from 'next/head';
import { GetStaticProps, GetStaticPaths } from 'next';

type Post = {
  title: string;
  description: string;
  slug: string;
  coverImage: string;
};

export default function BlogPost({ post }: { post: Post }) {
  return (
    <>
      <Head>
        <title>{post.title} | Blog</title>
        <meta name="description" content={post.description} />
        <meta property="og:title" content={post.title} />
        <meta property="og:image" content={post.coverImage} />
        <link rel="canonical" href={`https://example.com/blog/${post.slug}`} />
      </Head>
      <article>
        <h1>{post.title}</h1>
      </article>
    </>
  );
}

// Lấy dữ liệu khi build → HTML tĩnh → SEO tốt nhất
export const getStaticProps: GetStaticProps = async ({ params }) => {
  const post = await fetchPost(params?.slug as string);
  return { props: { post } };
};

export const getStaticPaths: GetStaticPaths = async () => {
  const slugs = await fetchAllSlugs();
  return {
    paths: slugs.map(slug => ({ params: { slug } })),
    fallback: 'blocking', // Các slug chưa có → SSR lần đầu
  };
};
```

---

## 4. Rendering Strategy và SEO

### 4.1 Tổng quan các chiến lược render

Next.js hỗ trợ nhiều chiến lược render, mỗi loại có mức độ ảnh hưởng SEO khác nhau.

### 4.2 SSG – Static Site Generation

**Định nghĩa:** HTML được tạo sẵn tại thời điểm build. Mỗi request trả về file HTML tĩnh từ CDN.

```tsx
// App Router
// app/blog/page.tsx
async function BlogPage() {
  // fetch() mặc định là cache: 'force-cache' → SSG
  const posts = await fetch('https://api.example.com/posts').then(r => r.json());

  return (
    <ul>
      {posts.map(post => <li key={post.id}>{post.title}</li>)}
    </ul>
  );
}
```

**Ưu điểm SEO:** Tốc độ cực nhanh (CDN), Google bot đọc được ngay, tốt nhất cho nội dung ít thay đổi.

### 4.3 SSR – Server-Side Rendering

**Định nghĩa:** HTML được tạo trên server **mỗi khi có request**. Phù hợp cho nội dung thay đổi thường xuyên hoặc phụ thuộc phiên người dùng.

```tsx
// App Router
// app/dashboard/page.tsx
async function DashboardPage() {
  // noStore() hoặc cache: 'no-store' → SSR
  const data = await fetch('https://api.example.com/live-data', {
    cache: 'no-store',
  }).then(r => r.json());

  return <div>Dữ liệu mới nhất: {data.value}</div>;
}
```

**Ưu điểm SEO:** Nội dung luôn mới nhất, bot vẫn đọc được HTML đầy đủ.

### 4.4 ISR – Incremental Static Regeneration

**Định nghĩa:** Kết hợp SSG và SSR. Trang được render tĩnh nhưng **tự động tái tạo** sau một khoảng thời gian (revalidate) hoặc theo yêu cầu.

```tsx
// App Router
// app/news/page.tsx
async function NewsPage() {
  const news = await fetch('https://api.example.com/news', {
    next: { revalidate: 60 }, // Tái tạo sau 60 giây
  }).then(r => r.json());

  return (
    <ul>
      {news.map(item => <li key={item.id}>{item.title}</li>)}
    </ul>
  );
}
```

**Ưu điểm SEO:** Cân bằng giữa tốc độ (CDN) và nội dung cập nhật, rất phù hợp cho blog, tin tức.

### 4.5 So sánh tổng hợp

| Chiến lược | Thời điểm tạo HTML | Tốc độ | Nội dung | Phù hợp |
|-----------|-------------------|--------|----------|---------|
| **SSG** | Lúc build | ⚡⚡⚡ Rất nhanh | Tĩnh | Landing page, docs |
| **SSR** | Mỗi request | ⚡⚡ Nhanh | Luôn mới | Dashboard, giỏ hàng |
| **ISR** | Build + tái tạo | ⚡⚡⚡ Rất nhanh | Mới định kỳ | Blog, tin tức |
| **CSR** | Trình duyệt | ⚡ Chậm | Động | Trang không cần SEO |

---

## 5. Open Graph và Social Media Tags

### 5.1 Định nghĩa

**Open Graph (OG)** là giao thức do Facebook tạo ra, cho phép kiểm soát cách URL hiển thị khi được chia sẻ trên mạng xã hội (Facebook, Twitter/X, LinkedIn, Zalo, ...). Khi thiếu OG tags, mạng xã hội tự chọn ảnh và mô tả ngẫu nhiên, thường không như ý.

### 5.2 Triển khai với App Router

```tsx
// app/layout.tsx
import type { Metadata } from 'next';

export const metadata: Metadata = {
  // Open Graph – cho Facebook, LinkedIn, Zalo
  openGraph: {
    type: 'website',
    url: 'https://example.com',
    title: 'Tiêu đề khi chia sẻ lên mạng xã hội',
    description: 'Mô tả hiển thị khi chia sẻ, nên từ 100–200 ký tự.',
    siteName: 'Tên Website',
    images: [
      {
        url: 'https://example.com/og-image.jpg', // Kích thước khuyến nghị: 1200x630px
        width: 1200,
        height: 630,
        alt: 'Mô tả hình ảnh cho người dùng khiếm thị',
      },
    ],
    locale: 'vi_VN',
  },

  // Twitter Card – cho Twitter/X
  twitter: {
    card: 'summary_large_image', // Loại card: summary | summary_large_image | app | player
    site: '@ten_twitter',         // Twitter handle của website
    creator: '@ten_tac_gia',      // Twitter handle của tác giả
    title: 'Tiêu đề trên Twitter',
    description: 'Mô tả trên Twitter, tối đa 200 ký tự.',
    images: ['https://example.com/twitter-image.jpg'],
  },
};
```

### 5.3 Open Graph cho bài viết (Article)

```tsx
// app/blog/[slug]/page.tsx
export async function generateMetadata({ params }): Promise<Metadata> {
  const post = await getPost(params.slug);

  return {
    openGraph: {
      type: 'article',                           // Loại: article (quan trọng cho blog)
      title: post.title,
      description: post.excerpt,
      url: `https://example.com/blog/${post.slug}`,
      publishedTime: post.publishedAt,           // ISO 8601
      modifiedTime: post.updatedAt,
      authors: ['https://example.com/authors/nguyen-van-a'],
      section: 'Công nghệ',
      tags: post.tags,
      images: [{ url: post.coverImage, width: 1200, height: 630 }],
    },
  };
}
```

### 5.4 HTML output tương ứng

```html
<!-- Open Graph -->
<meta property="og:type" content="website" />
<meta property="og:url" content="https://example.com" />
<meta property="og:title" content="Tiêu đề khi chia sẻ" />
<meta property="og:description" content="Mô tả hiển thị..." />
<meta property="og:image" content="https://example.com/og-image.jpg" />

<!-- Twitter Card -->
<meta name="twitter:card" content="summary_large_image" />
<meta name="twitter:title" content="Tiêu đề trên Twitter" />
<meta name="twitter:image" content="https://example.com/twitter-image.jpg" />
```

---

## 6. Sitemap

### 6.1 Định nghĩa

**Sitemap** là file XML liệt kê tất cả các URL của website, giúp công cụ tìm kiếm **khám phá (crawl) và lập chỉ mục (index)** hiệu quả hơn, đặc biệt với website lớn hoặc các trang mới.

### 6.2 Sitemap tĩnh (App Router)

```tsx
// app/sitemap.ts
import { MetadataRoute } from 'next';

export default function sitemap(): MetadataRoute.Sitemap {
  return [
    {
      url: 'https://example.com',
      lastModified: new Date(),
      changeFrequency: 'yearly',   // Tần suất thay đổi: always|hourly|daily|weekly|monthly|yearly|never
      priority: 1,                  // Mức độ ưu tiên: 0.0 → 1.0
    },
    {
      url: 'https://example.com/about',
      lastModified: new Date(),
      changeFrequency: 'monthly',
      priority: 0.8,
    },
    {
      url: 'https://example.com/contact',
      changeFrequency: 'yearly',
      priority: 0.5,
    },
  ];
}
```

### 6.3 Sitemap động – tích hợp dữ liệu từ API/DB

```tsx
// app/sitemap.ts
import { MetadataRoute } from 'next';

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  // Lấy tất cả bài viết từ API
  const posts = await fetch('https://api.example.com/posts').then(r => r.json());

  // Trang tĩnh cố định
  const staticPages: MetadataRoute.Sitemap = [
    { url: 'https://example.com', priority: 1, changeFrequency: 'daily' },
    { url: 'https://example.com/about', priority: 0.8, changeFrequency: 'monthly' },
  ];

  // Trang động từ dữ liệu
  const postPages: MetadataRoute.Sitemap = posts.map(post => ({
    url: `https://example.com/blog/${post.slug}`,
    lastModified: new Date(post.updatedAt),
    changeFrequency: 'weekly',
    priority: 0.7,
  }));

  return [...staticPages, ...postPages];
}
```

**Output XML tự động tại `/sitemap.xml`:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://example.com</loc>
    <lastmod>2024-01-01</lastmod>
    <changefreq>daily</changefreq>
    <priority>1</priority>
  </url>
  <url>
    <loc>https://example.com/blog/bai-viet-1</loc>
    <lastmod>2024-06-15</lastmod>
    <changefreq>weekly</changefreq>
    <priority>0.7</priority>
  </url>
</urlset>
```

---

## 7. robots.txt

### 7.1 Định nghĩa

**robots.txt** là file văn bản đặt tại thư mục gốc của website, chứa các quy tắc hướng dẫn bot tìm kiếm biết trang nào được phép và không được phép crawl.

### 7.2 Tạo robots.txt với App Router

```tsx
// app/robots.ts
import { MetadataRoute } from 'next';

export default function robots(): MetadataRoute.Robots {
  return {
    rules: [
      {
        userAgent: '*',             // Áp dụng cho tất cả bot
        allow: '/',                 // Cho phép crawl toàn bộ
        disallow: [
          '/admin/',               // Không cho crawl trang admin
          '/api/',                 // Không cho crawl API routes
          '/private/',             // Không cho crawl trang riêng tư
          '/_next/',               // Không cho crawl thư mục Next.js
        ],
      },
      {
        userAgent: 'Googlebot',    // Quy tắc riêng cho Googlebot
        allow: '/',
        crawlDelay: 2,             // Thời gian chờ giữa các request (giây)
      },
    ],
    sitemap: 'https://example.com/sitemap.xml',  // Đường dẫn sitemap
    host: 'https://example.com',                  // Host chính
  };
}
```

**Output tại `/robots.txt`:**
```
User-agent: *
Allow: /
Disallow: /admin/
Disallow: /api/
Disallow: /private/

User-agent: Googlebot
Allow: /
Crawl-delay: 2

Sitemap: https://example.com/sitemap.xml
Host: https://example.com
```

---

## 8. Canonical URL

### 8.1 Định nghĩa

**Canonical URL** là thẻ HTML chỉ định URL **"chính thức"** của một trang khi cùng nội dung có thể truy cập qua nhiều địa chỉ khác nhau. Giúp tránh **duplicate content** (nội dung trùng lặp) gây phân tán điểm SEO.

**Ví dụ các URL trùng nội dung:**
```
https://example.com/san-pham?ref=home
https://example.com/san-pham?utm_source=google
https://example.com/san-pham
```
→ Ba URL trên có cùng nội dung. Canonical xác định cái nào là "gốc".

### 8.2 Khai báo Canonical với App Router

```tsx
// app/products/[id]/page.tsx
export async function generateMetadata({ params }): Promise<Metadata> {
  return {
    alternates: {
      // URL canonical – luôn dùng URL đầy đủ, không có tham số
      canonical: `https://example.com/products/${params.id}`,

      // Phiên bản ngôn ngữ khác (hreflang – SEO đa ngôn ngữ)
      languages: {
        'vi-VN': `https://example.com/vi/products/${params.id}`,
        'en-US': `https://example.com/en/products/${params.id}`,
      },
    },
  };
}
```

**HTML output:**
```html
<link rel="canonical" href="https://example.com/products/123" />
<link rel="alternate" hreflang="vi-VN" href="https://example.com/vi/products/123" />
<link rel="alternate" hreflang="en-US" href="https://example.com/en/products/123" />
```

---

## 9. Structured Data – JSON-LD

### 9.1 Định nghĩa

**Structured Data (Dữ liệu có cấu trúc)** là định dạng thông tin chuẩn hóa giúp công cụ tìm kiếm **hiểu ngữ nghĩa** của nội dung trang web. **JSON-LD** (JavaScript Object Notation for Linked Data) là định dạng được Google khuyến nghị, được nhúng trong thẻ `<script type="application/ld+json">`.

Kết quả: xuất hiện các **Rich Results** (kết quả nổi bật) trên Google như đánh giá sao, giá sản phẩm, câu hỏi thường gặp, ...

### 9.2 Ví dụ: Trang sản phẩm (Product Schema)

```tsx
// app/products/[id]/page.tsx
export default async function ProductPage({ params }) {
  const product = await getProduct(params.id);

  // JSON-LD cho sản phẩm
  const jsonLd = {
    '@context': 'https://schema.org',
    '@type': 'Product',
    name: product.name,
    description: product.description,
    image: product.imageUrl,
    sku: product.sku,
    brand: {
      '@type': 'Brand',
      name: product.brandName,
    },
    offers: {
      '@type': 'Offer',
      price: product.price,
      priceCurrency: 'VND',
      availability: product.inStock
        ? 'https://schema.org/InStock'
        : 'https://schema.org/OutOfStock',
      url: `https://example.com/products/${params.id}`,
    },
    aggregateRating: {
      '@type': 'AggregateRating',
      ratingValue: product.rating,
      reviewCount: product.reviewCount,
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

### 9.3 Ví dụ: Bài viết Blog (Article Schema)

```tsx
const articleJsonLd = {
  '@context': 'https://schema.org',
  '@type': 'Article',
  headline: post.title,
  description: post.excerpt,
  image: post.coverImage,
  datePublished: post.publishedAt,
  dateModified: post.updatedAt,
  author: {
    '@type': 'Person',
    name: post.authorName,
    url: `https://example.com/authors/${post.authorSlug}`,
  },
  publisher: {
    '@type': 'Organization',
    name: 'Tên Website',
    logo: {
      '@type': 'ImageObject',
      url: 'https://example.com/logo.png',
    },
  },
};
```

### 9.4 Ví dụ: FAQ Schema

```tsx
const faqJsonLd = {
  '@context': 'https://schema.org',
  '@type': 'FAQPage',
  mainEntity: [
    {
      '@type': 'Question',
      name: 'Next.js là gì?',
      acceptedAnswer: {
        '@type': 'Answer',
        text: 'Next.js là framework React cho phép server-side rendering và static generation.',
      },
    },
    {
      '@type': 'Question',
      name: 'Tại sao Next.js tốt cho SEO?',
      acceptedAnswer: {
        '@type': 'Answer',
        text: 'Vì Next.js render HTML ở server, giúp bot tìm kiếm đọc nội dung ngay lập tức.',
      },
    },
  ],
};
```

---

## 10. Tối ưu hình ảnh với next/image

### 10.1 Định nghĩa

`next/image` là component tối ưu hình ảnh tích hợp sẵn trong Next.js, tự động thực hiện nhiều kỹ thuật giúp cải thiện **Core Web Vitals** và SEO, bao gồm: lazy loading, responsive sizes, chuyển đổi sang định dạng hiện đại (WebP/AVIF), và tránh **Cumulative Layout Shift (CLS)**.

### 10.2 Cú pháp cơ bản

```tsx
import Image from 'next/image';

// Ảnh với kích thước cố định
function Avatar() {
  return (
    <Image
      src="/avatar.jpg"
      alt="Ảnh đại diện người dùng"  // Bắt buộc – quan trọng cho SEO & accessibility
      width={200}
      height={200}
      quality={85}                    // Chất lượng nén (mặc định 75)
    />
  );
}
```

### 10.3 Ảnh responsive với fill

```tsx
function HeroBanner() {
  return (
    <div style={{ position: 'relative', width: '100%', height: '400px' }}>
      <Image
        src="/hero.jpg"
        alt="Banner trang chủ"
        fill                          // Lấp đầy container
        style={{ objectFit: 'cover' }}
        sizes="(max-width: 768px) 100vw, 50vw"  // Gợi ý kích thước để tối ưu tải
        priority                      // Tải ngay (không lazy) – dùng cho ảnh above-the-fold
      />
    </div>
  );
}
```

### 10.4 Lợi ích SEO của next/image

| Tính năng | Mô tả | Ảnh hưởng SEO |
|-----------|-------|--------------|
| **Lazy loading** | Chỉ tải ảnh khi vào viewport | Tăng LCP, giảm bandwidth |
| **Responsive** | Tự tạo nhiều kích thước | Tải đúng size → nhanh hơn |
| **WebP/AVIF** | Tự chuyển đổi định dạng | File nhỏ hơn 30–50% |
| **No layout shift** | Giữ chỗ trước khi ảnh tải | Cải thiện CLS |
| **priority prop** | Tải ảnh LCP trước | Cải thiện LCP score |

---

## 11. Tối ưu Font với next/font

### 11.1 Định nghĩa

`next/font` tự động tối ưu font chữ bằng cách **tải font về và lưu trữ cùng với ứng dụng**, tránh request đến server bên ngoài (ví dụ: Google Fonts), đảm bảo **zero layout shift** và cải thiện tốc độ tải trang.

### 11.2 Sử dụng Google Fonts

```tsx
// app/layout.tsx
import { Inter, Roboto } from 'next/font/google';

const inter = Inter({
  subsets: ['latin', 'vietnamese'],  // Chỉ tải ký tự cần thiết
  weight: ['400', '600', '700'],
  display: 'swap',                   // Font fallback trong khi tải → tránh invisible text
  variable: '--font-inter',          // CSS variable để dùng trong Tailwind
});

const roboto = Roboto({
  subsets: ['latin'],
  weight: '400',
  display: 'swap',
});

export default function RootLayout({ children }) {
  return (
    <html lang="vi" className={inter.variable}>
      <body className={inter.className}>
        {children}
      </body>
    </html>
  );
}
```

### 11.3 Sử dụng Font cục bộ

```tsx
import localFont from 'next/font/local';

const myFont = localFont({
  src: [
    { path: './fonts/MyFont-Regular.woff2', weight: '400' },
    { path: './fonts/MyFont-Bold.woff2', weight: '700' },
  ],
  display: 'swap',
  variable: '--font-custom',
});
```

**Lợi ích SEO:** Cải thiện **FCP (First Contentful Paint)** và **CLS**, hai chỉ số quan trọng trong Core Web Vitals.

---

## 12. Core Web Vitals

### 12.1 Định nghĩa

**Core Web Vitals** là bộ ba chỉ số hiệu năng do Google xác định, được sử dụng trực tiếp làm **yếu tố xếp hạng tìm kiếm** từ năm 2021. Next.js được thiết kế để tối ưu cả ba chỉ số này.

### 12.2 Ba chỉ số Core Web Vitals

| Chỉ số | Viết tắt | Đo lường | Tốt | Cần cải thiện | Kém |
|--------|----------|---------|-----|--------------|-----|
| **Largest Contentful Paint** | LCP | Tốc độ tải phần tử lớn nhất | ≤ 2.5s | ≤ 4s | > 4s |
| **Interaction to Next Paint** | INP | Thời gian phản hồi tương tác | ≤ 200ms | ≤ 500ms | > 500ms |
| **Cumulative Layout Shift** | CLS | Mức độ dịch chuyển layout | ≤ 0.1 | ≤ 0.25 | > 0.25 |

### 12.3 Cách Next.js cải thiện từng chỉ số

**LCP (Largest Contentful Paint):**
```tsx
// Sử dụng priority để tải ảnh LCP trước
<Image src="/hero.jpg" alt="Hero" fill priority />

// Preload font tránh FOIT (Flash of Invisible Text)
const font = Inter({ display: 'swap' });

// ISR/SSG → HTML sẵn sàng ngay, không chờ JS
export const revalidate = 3600;
```

**CLS (Cumulative Layout Shift):**
```tsx
// next/image tự động giữ chỗ (placeholder) tránh dịch chuyển
<Image
  src="/product.jpg"
  width={400}
  height={300}
  alt="Sản phẩm"
  placeholder="blur"        // Hiện ảnh mờ trong khi tải
  blurDataURL="data:..."    // Ảnh base64 nhỏ làm placeholder
/>
```

**INP (Interaction to Next Paint):**
```tsx
// Server Components giảm JavaScript gửi về client
// Chỉ thêm 'use client' khi thực sự cần interactivity
'use client'; // Chỉ dùng khi cần useState, useEffect, event handlers

// Dynamic import cho component nặng
import dynamic from 'next/dynamic';
const HeavyChart = dynamic(() => import('./HeavyChart'), {
  loading: () => <p>Đang tải biểu đồ...</p>,
  ssr: false,  // Không render ở server → giảm JS bundle
});
```

### 12.4 Đo lường Core Web Vitals trong Next.js

```tsx
// pages/_app.tsx (Pages Router) – Báo cáo Core Web Vitals
export function reportWebVitals(metric) {
  console.log(metric);
  // Gửi về analytics
  if (metric.label === 'web-vital') {
    sendToAnalytics({
      name: metric.name,   // LCP, INP, CLS, FCP, TTFB
      value: metric.value,
    });
  }
}
```

---

## 13. Dynamic SEO theo trang

### 13.1 Tình huống: Website thương mại điện tử

Đây là ví dụ tổng hợp triển khai SEO đầy đủ cho một trang sản phẩm động.

```tsx
// app/products/[slug]/page.tsx
import { Metadata } from 'next';
import Image from 'next/image';

type Product = {
  name: string;
  slug: string;
  description: string;
  price: number;
  image: string;
  rating: number;
  reviewCount: number;
};

async function getProduct(slug: string): Promise<Product> {
  return fetch(`https://api.example.com/products/${slug}`, {
    next: { revalidate: 3600 }, // ISR: tái tạo sau 1 giờ
  }).then(r => r.json());
}

// ── 1. Dynamic Metadata ──────────────────────────────────────────
export async function generateMetadata({ params }): Promise<Metadata> {
  const product = await getProduct(params.slug);
  const url = `https://example.com/products/${product.slug}`;

  return {
    title: `${product.name} | Shop`,
    description: product.description.slice(0, 160),
    alternates: {
      canonical: url,
    },
    openGraph: {
      type: 'website',
      url,
      title: product.name,
      description: product.description,
      images: [{ url: product.image, width: 1200, height: 630 }],
    },
    twitter: {
      card: 'summary_large_image',
      title: product.name,
      images: [product.image],
    },
  };
}

// ── 2. Tạo đường dẫn tĩnh cho SSG ───────────────────────────────
export async function generateStaticParams() {
  const products = await fetch('https://api.example.com/products').then(r => r.json());
  return products.map(p => ({ slug: p.slug }));
}

// ── 3. Page Component với JSON-LD ────────────────────────────────
export default async function ProductPage({ params }) {
  const product = await getProduct(params.slug);

  const jsonLd = {
    '@context': 'https://schema.org',
    '@type': 'Product',
    name: product.name,
    description: product.description,
    image: product.image,
    offers: {
      '@type': 'Offer',
      price: product.price,
      priceCurrency: 'VND',
      availability: 'https://schema.org/InStock',
    },
    aggregateRating: {
      '@type': 'AggregateRating',
      ratingValue: product.rating,
      reviewCount: product.reviewCount,
    },
  };

  return (
    <>
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }}
      />
      <main>
        <Image
          src={product.image}
          alt={product.name}
          width={800}
          height={600}
          priority              // Ảnh chính → tải ưu tiên → cải thiện LCP
        />
        <h1>{product.name}</h1>
        <p>{product.description}</p>
        <p>{product.price.toLocaleString()} đ</p>
      </main>
    </>
  );
}
```

---

## 14. Bảng tổng hợp và Checklist SEO

### 14.1 Bảng tổng hợp kỹ thuật SEO trong Next.js

| Kỹ thuật | API / File | Tác dụng SEO |
|----------|-----------|-------------|
| **Metadata tĩnh** | `export const metadata` | Title, description, robots |
| **Metadata động** | `generateMetadata()` | SEO theo nội dung thực tế |
| **Title Template** | `title.template` | Chuẩn hóa tiêu đề toàn site |
| **Open Graph** | `metadata.openGraph` | Hiển thị đẹp khi chia sẻ MXH |
| **Twitter Card** | `metadata.twitter` | Hiển thị đẹp trên Twitter/X |
| **Sitemap** | `app/sitemap.ts` | Bot khám phá URL dễ hơn |
| **robots.txt** | `app/robots.ts` | Kiểm soát quyền crawl |
| **Canonical URL** | `alternates.canonical` | Tránh duplicate content |
| **Hreflang** | `alternates.languages` | SEO đa ngôn ngữ |
| **JSON-LD** | `<script>` trong JSX | Rich Results trên Google |
| **SSG / ISR** | `fetch` + `revalidate` | HTML sẵn sàng cho bot |
| **next/image** | `<Image priority>` | Cải thiện LCP, CLS |
| **next/font** | `Inter({ display:'swap' })` | Cải thiện CLS, FCP |
| **Dynamic Import** | `dynamic(() => import())` | Giảm JS bundle → INP |

### 14.2 Checklist SEO cho dự án Next.js

```
METADATA
☐ Khai báo title và description cho tất cả trang quan trọng
☐ Title template nhất quán toàn site
☐ robots: index, follow (hoặc noindex cho trang không cần SEO)
☐ Canonical URL đúng cho mọi trang

SOCIAL MEDIA
☐ Open Graph: title, description, image (1200x630px), type
☐ Twitter Card: card type, title, image
☐ OG image có kích thước và alt text đúng

CRAWLING & INDEXING
☐ /sitemap.xml hoạt động và đầy đủ URL
☐ /robots.txt kiểm soát đúng quyền crawl
☐ Không disallow trang quan trọng trong robots.txt

STRUCTURED DATA
☐ JSON-LD phù hợp loại trang (Article, Product, FAQ, ...)
☐ Kiểm tra bằng Google Rich Results Test

PERFORMANCE (Core Web Vitals)
☐ LCP ≤ 2.5s: dùng SSG/ISR, priority image
☐ CLS ≤ 0.1: dùng next/image (width/height hoặc fill)
☐ INP ≤ 200ms: Server Components, dynamic import
☐ next/font với display: 'swap'

RENDERING
☐ Các trang SEO quan trọng dùng SSG hoặc ISR (không dùng CSR)
☐ Nội dung chính có trong HTML trả về (không render chỉ bằng JS)

URL
☐ URL ngắn gọn, có từ khóa, không dấu tiếng Việt
☐ Không có tham số không cần thiết (?ref=, ?utm_)
☐ Phân cấp thư mục logic (example.com/danh-muc/san-pham)
```

---

## Tài liệu tham khảo

- Next.js Official Docs – Metadata: https://nextjs.org/docs/app/building-your-application/optimizing/metadata
- Next.js Official Docs – Image Optimization: https://nextjs.org/docs/app/building-your-application/optimizing/images
- Google Search Central – Core Web Vitals: https://developers.google.com/search/docs/appearance/core-web-vitals
- Google Search Central – Structured Data: https://developers.google.com/search/docs/appearance/structured-data
- Schema.org – Vocabulary: https://schema.org/docs/full.html
- Open Graph Protocol: https://ogp.me

---

*Tài liệu được biên soạn phục vụ mục đích học tập và báo cáo tiểu luận.*
