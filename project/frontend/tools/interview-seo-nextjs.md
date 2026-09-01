# 🔍 Phỏng vấn Frontend: SEO & SEO trong Next.js

---

## 📚 PHẦN 1: SEO CƠ BẢN

### Câu hỏi 1: SEO là gì? Tại sao quan trọng?

**Trả lời:**

SEO (Search Engine Optimization) là tập hợp các kỹ thuật để cải thiện ranking của website trên search engines như Google.

**Tại sao quan trọng:**
- 93% online experiences start with search engine
- Organic traffic không tốn tiền (không như ads)
- High-intent users (people searching for solutions)
- Long-term value (vs ads which stop when you stop paying)
- Builds credibility & trust

**3 pillars của SEO:**

| Pillar | Focus | Example |
|--------|-------|---------|
| **Technical SEO** | Codebase, structure, performance | Page speed, mobile-friendly, structured data |
| **On-page SEO** | Content, keywords, meta tags | Title, description, H1, keyword usage |
| **Off-page SEO** | Backlinks, brand mentions, social | Quality links, mentions, social signals |

---

### Câu hỏi 2: On-page SEO - Làm sao optimize?

**Trả lời:**

On-page SEO tập trung vào content và HTML optimization.

**Key elements:**

```html
<!-- 1. Meta tags -->
<head>
  <!-- ✅ Title: 50-60 characters, keyword-rich -->
  <title>Best Coffee Shops in New York - 2024 Guide</title>

  <!-- ✅ Meta description: 150-160 characters -->
  <meta name="description" content="Discover 15 amazing coffee shops in NYC. Expert reviews, locations, hours. Find your perfect coffee spot today.">

  <!-- ✅ Meta keywords (less important but helps) -->
  <meta name="keywords" content="coffee shops, NYC, cafes, espresso">

  <!-- ✅ Canonical: Prevent duplicate content -->
  <link rel="canonical" href="https://example.com/coffee-shops-nyc">

  <!-- ✅ Viewport: Mobile-friendly -->
  <meta name="viewport" content="width=device-width, initial-scale=1.0">

  <!-- ✅ Open Graph: Social sharing -->
  <meta property="og:title" content="Best Coffee Shops in NYC">
  <meta property="og:description" content="15 amazing coffee shops">
  <meta property="og:image" content="https://example.com/coffee.jpg">
  <meta property="og:url" content="https://example.com/coffee-shops-nyc">
</head>

<body>
  <!-- ✅ H1: Only 1 per page, keyword-rich -->
  <h1>Best Coffee Shops in New York City</h1>

  <!-- ✅ Heading hierarchy: H1 > H2 > H3 (semantic) -->
  <h2>Top Espresso Bars</h2>
  <h3>Gramercy Coffee</h3>

  <!-- ✅ Alt text for images -->
  <img src="coffee.jpg" alt="Espresso being pulled at Gramercy Coffee shop">

  <!-- ✅ Internal links: Help structure, distribute authority -->
  <a href="/best-coffee-machines">Best home espresso machines</a>

  <!-- ✅ Keyword usage: 1-2% keyword density (natural) -->
  <p>Looking for the best coffee shops in New York? Our guide covers 15 top-rated coffee shops across NYC...</p>
</body>
```

**Best practices:**

```
✅ Title tag:
  - Include primary keyword
  - Front-load important words
  - 50-60 characters
  - Example: "Best Coffee Shops in NYC - 2024 Guide"

✅ Meta description:
  - Include keyword naturally
  - Make it compelling (it's a teaser)
  - 150-160 characters
  - Include call-to-action

✅ Content:
  - Write for humans first (not search engines)
  - Use keywords naturally (not stuffing)
  - Minimum 300-500 words (for blog posts)
  - Use semantic HTML (H1, H2, H3, etc)
  - Break content into sections

✅ Images:
  - Descriptive alt text
  - Compress for performance
  - Use webp format
  - Include captions if relevant

✅ Internal linking:
  - Link to related content
  - Use descriptive anchor text
  - Help users navigate
  - Distribute page authority
```

---

### Câu hỏi 3: Technical SEO - Cần làm gì?

**Trả lời:**

Technical SEO là foundation - nếu làm sai, on-page & off-page SEO không giúp được.

**Key factors:**

```yaml
1. Site Speed:
  - Page load < 3 seconds (ideal < 1.5s)
  - Core Web Vitals: LCP, FID, CLS
  - Optimize images, code splitting
  - Use CDN

2. Mobile-Friendly:
  - Responsive design
  - Touch-friendly buttons
  - Readable text (16px+ font)
  - Google Mobile-Friendly Test

3. URL Structure:
  - Descriptive URLs (not /p?id=123)
  - Use hyphens (not underscores)
  - Example: /blog/best-coffee-shops-nyc (Good)
            /p/123 (Bad)

4. XML Sitemap:
  - List all important pages
  - Submit to Google Search Console
  - Update frequently

5. Robots.txt:
  - Control crawling
  - Disallow unnecessary pages
  - Allow important pages

6. SSL/HTTPS:
  - Secure connection
  - Google ranking factor
  - Trust signal for users

7. Structured Data (Schema):
  - Help search engines understand content
  - Show rich snippets in search results
  - JSON-LD format (best)

8. Duplicate Content:
  - Use canonical tags
  - Keep URL structure consistent
  - Avoid parameter variations

9. Crawlability:
  - All links are crawlable (not behind JavaScript)
  - No nofollow on important links
  - Good internal link structure

10. Site Architecture:
  - Logical hierarchy
  - Max 3 clicks to reach any page
  - Clear navigation
```

---

### Câu hỏi 4: Structured Data (Schema markup) - Làm sao dùng?

**Trả lời:**

Structured data giúp search engines hiểu content tốt hơn, enable rich snippets.

```json
// Article schema (for blog posts)
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "Best Coffee Shops in New York City",
  "description": "Our comprehensive guide to 15 top-rated coffee shops in NYC",
  "image": "https://example.com/coffee.jpg",
  "datePublished": "2024-01-15",
  "dateModified": "2024-08-26",
  "author": {
    "@type": "Person",
    "name": "John Smith"
  },
  "mainEntity": {
    "@type": "Article",
    "headline": "Best Coffee Shops in NYC"
  }
}

// Product schema (for ecommerce)
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "Premium Coffee Beans",
  "description": "High-quality arabica beans",
  "image": "https://example.com/coffee-beans.jpg",
  "brand": {
    "@type": "Brand",
    "name": "BestCoffee"
  },
  "offers": {
    "@type": "Offer",
    "price": "29.99",
    "priceCurrency": "USD",
    "availability": "https://schema.org/InStock"
  },
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.5",
    "reviewCount": "512"
  }
}

// LocalBusiness schema (for local SEO)
{
  "@context": "https://schema.org",
  "@type": "LocalBusiness",
  "name": "Gramercy Coffee",
  "image": "https://example.com/gramercy.jpg",
  "address": {
    "@type": "PostalAddress",
    "streetAddress": "123 Main St",
    "addressLocality": "New York",
    "addressRegion": "NY",
    "postalCode": "10003",
    "addressCountry": "US"
  },
  "telephone": "+1-212-555-0123",
  "url": "https://grammercycoffee.com",
  "priceRange": "$$"
}

// FAQ schema (for featured snippets)
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "What are the best coffee shops in NYC?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "The best coffee shops include Gramercy Coffee, Stumptown, and La Colombe..."
      }
    }
  ]
}
```

---

### Câu hỏi 5: Backlinks - Tại sao quan trọng?

**Trả lời:**

Backlinks là "votes of confidence" từ websites khác. Chúng là ranking factor mạnh nhất.

**Quality > Quantity:**
- 1 backlink từ Authority site (DA 70+) > 100 từ spam sites
- Relevant links (related topic) > Random domains
- Natural links > Bought links (Google penalizes)

**Làm sao build backlinks:**

```
1. Create linkable content:
   - Comprehensive guides
   - Original research
   - Infographics
   - Tools/resources

2. Guest posting:
   - Write for reputable blogs
   - Include link in bio
   - Quality > quantity

3. Broken link building:
   - Find broken links on authority sites
   - Create similar content
   - Email webmaster with replacement

4. Resource pages:
   - Find resource lists in your niche
   - Ask to be included
   - Provide value

5. HARO (Help a Reporter Out):
   - Journalists looking for experts
   - Get mentioned in publications
   - Earn backlinks

6. Partnerships:
   - Co-create with other brands
   - Link to partners, they link back

❌ Avoid:
   - Private Blog Networks (PBN)
   - Link buying/selling
   - Comment spam
   - Automated link building
   - Irrelevant directory submissions
```

---

### Câu hỏi 6: Local SEO - Làm sao optimize cho location-specific?

**Trả lời:**

Important cho businesses với physical locations.

```
1. Google Business Profile:
   - Complete all information
   - Add photos/videos
   - Respond to reviews
   - Add business posts

2. Local citations:
   - Consistent Name, Address, Phone (NAP)
   - Submit to business directories (Yelp, Yellow Pages)
   - Local schema markup

3. Local content:
   - Location-specific pages
   - Local keywords
   - Local events/news

4. Reviews:
   - Encourage customer reviews
   - Respond to all reviews
   - More reviews = better ranking

5. Location pages:
   - Separate page for each location
   - Local schema markup
   - Local keywords in content
   - Different phone numbers per location

Example: Coffee shop with 3 locations
/locations/new-york
/locations/los-angeles
/locations/chicago
```

---

## 🚀 PHẦN 2: NEXT.JS SEO

### Câu hỏi 1: Next.js SEO advantages?

**Trả lời:**

**Advantages:**
✅ Server-side rendering (SSR) - Search engines can crawl content
✅ Static generation (SSG) - Fast, better for SEO
✅ Built-in Image optimization
✅ Automatic code splitting
✅ API routes (backend integrated)
✅ File-based routing (semantic URLs)

**Challenges:**
❌ Client-side rendering (if not careful)
❌ Dynamic routes need proper handling
❌ JavaScript-heavy (though Next.js helps)

---

### Câu hỏi 2: Meta tags & Head management trong Next.js?

**Trả lời:**

**Cách 1: next/head (Pages Router)**
```jsx
// pages/blog/[slug].js
import Head from 'next/head';

export default function BlogPost({ post }) {
  return (
    <>
      <Head>
        <title>{post.title} | My Blog</title>
        <meta name="description" content={post.excerpt} />
        <meta name="keywords" content={post.keywords} />
        
        {/* Open Graph */}
        <meta property="og:title" content={post.title} />
        <meta property="og:description" content={post.excerpt} />
        <meta property="og:image" content={post.image} />
        <meta property="og:url" content={`https://example.com/blog/${post.slug}`} />
        <meta property="og:type" content="article" />
        
        {/* Twitter Card */}
        <meta name="twitter:card" content="summary_large_image" />
        <meta name="twitter:title" content={post.title} />
        <meta name="twitter:description" content={post.excerpt} />
        <meta name="twitter:image" content={post.image} />
        
        {/* Canonical */}
        <link rel="canonical" href={`https://example.com/blog/${post.slug}`} />
      </Head>
      
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </>
  );
}
```

**Cách 2: next/head (App Router - Metadata API)**
```jsx
// app/blog/[slug]/page.js
export const generateMetadata = async ({ params }) => {
  const post = await getPost(params.slug);
  
  return {
    title: `${post.title} | My Blog`,
    description: post.excerpt,
    keywords: post.keywords,
    
    // Open Graph
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: [{ url: post.image }],
      url: `https://example.com/blog/${post.slug}`,
      type: 'article'
    },
    
    // Twitter
    twitter: {
      card: 'summary_large_image',
      title: post.title,
      description: post.excerpt,
      images: [post.image]
    },
    
    // Canonical
    alternates: {
      canonical: `https://example.com/blog/${post.slug}`
    }
  };
};

export default function BlogPost({ params }) {
  const post = getPost(params.slug);
  
  return (
    <>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </>
  );
}
```

**Best practice:**
```jsx
// Create reusable function
// lib/seo.js
export const generateMetadata = (title, description, image, url) => ({
  title: `${title} | My Site`,
  description,
  openGraph: {
    title,
    description,
    images: [{ url: image }],
    url
  },
  twitter: {
    card: 'summary_large_image',
    title,
    description,
    images: [image]
  },
  alternates: {
    canonical: url
  }
});

// Use in page
import { generateMetadata } from '@/lib/seo';

export const metadata = generateMetadata(
  'Blog Title',
  'Blog description',
  '/image.jpg',
  'https://example.com/blog/post'
);
```

---

### Câu hỏi 3: Sitemap & Robots.txt trong Next.js?

**Trả lời:**

**Robots.txt:**
```javascript
// public/robots.txt
User-agent: *
Allow: /
Disallow: /admin
Disallow: /private
Disallow: /*.json$
Disallow: /*?*sort=
Disallow: /search?

Crawl-delay: 1

Sitemap: https://example.com/sitemap.xml
```

**Dynamic Sitemap (App Router):**
```javascript
// app/sitemap.js
import { getAllBlogPosts } from '@/lib/blog';
import { getAllProducts } from '@/lib/products';

export default async function sitemap() {
  const baseUrl = 'https://example.com';

  // Static pages
  const staticPages = [
    '',
    '/about',
    '/contact',
    '/blog',
    '/products'
  ].map(route => ({
    url: `${baseUrl}${route}`,
    lastModified: new Date(),
    changeFrequency: 'weekly',
    priority: route === '' ? 1.0 : 0.8
  }));

  // Dynamic blog posts
  const posts = await getAllBlogPosts();
  const postPages = posts.map(post => ({
    url: `${baseUrl}/blog/${post.slug}`,
    lastModified: post.updatedAt,
    changeFrequency: 'monthly',
    priority: 0.7
  }));

  // Dynamic products
  const products = await getAllProducts();
  const productPages = products.map(product => ({
    url: `${baseUrl}/products/${product.id}`,
    lastModified: product.updatedAt,
    changeFrequency: 'weekly',
    priority: 0.6
  }));

  return [...staticPages, ...postPages, ...productPages];
}
```

**Dynamic Sitemap (Pages Router):**
```javascript
// pages/sitemap.xml.js
function generateSiteMap(posts, products) {
  return `<?xml version="1.0" encoding="UTF-8"?>
    <urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
      ${posts
        .map(({ slug, updatedAt }) => {
          return `
        <url>
            <loc>${`https://example.com/blog/${slug}`}</loc>
            <lastmod>${updatedAt}</lastmod>
            <priority>0.7</priority>
        </url>
      `;
        })
        .join('')}
      ${products
        .map(({ id, updatedAt }) => {
          return `
        <url>
            <loc>${`https://example.com/products/${id}`}</loc>
            <lastmod>${updatedAt}</lastmod>
            <priority>0.6</priority>
        </url>
      `;
        })
        .join('')}
    </urlset>
 `;
}

function SiteMap() {}

export async function getServerSideProps({ res }) {
  const posts = await getAllBlogPosts();
  const products = await getAllProducts();

  const sitemap = generateSiteMap(posts, products);

  res.setHeader('Content-Type', 'text/xml');
  res.write(sitemap);
  res.end();

  return {
    props: {},
  };
}

export default SiteMap;
```

---

### Câu hỏi 4: Image optimization trong Next.js?

**Trả lời:**

Next.js `next/image` component tự động optimize.

```jsx
import Image from 'next/image';

// ❌ Bad: Standard HTML img
<img src="/blog.jpg" alt="Blog" />
// Problems: No optimization, no lazy loading, causes CLS

// ✅ Good: Next.js Image
<Image
  src="/blog.jpg"
  alt="Blog post thumbnail"
  width={600}
  height={400}
  priority={false}  // true only for LCP images
  quality={80}      // Compression quality (default 75)
  placeholder="blur" // Blurred placeholder while loading
  blurDataURL="data:image/jpeg;..." // Custom blur
/>

// ✅ For dynamic images
<Image
  src={post.coverImage}
  alt={post.title}
  width={600}
  height={400}
  sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
  responsive={true}
/>

// ✅ For background images
<div
  style={{
    backgroundImage: `url(${urlStart})`,
    backgroundSize: 'cover',
  }}
>
  <Image
    alt="Background"
    src={image}
    quality={30}
    fill
    style={{
      objectFit: 'cover',
    }}
  />
</div>
```

**Benefits:**
✅ Automatic resizing (generates multiple sizes)
✅ Format optimization (WebP, AVIF if browser supports)
✅ Lazy loading by default
✅ Prevents Cumulative Layout Shift
✅ Blur effect while loading

---

### Câu hỏi 5: SSG vs SSR - Khi nào dùng?

**Trả lời:**

**SSG (Static Site Generation) - BEST for SEO:**
```javascript
// pages/blog/[slug].js
export default function Post({ post }) {
  return <h1>{post.title}</h1>;
}

export async function getStaticPaths() {
  const posts = await getAllBlogPosts();
  
  return {
    paths: posts.map(post => ({
      params: { slug: post.slug }
    })),
    fallback: 'blocking' // Generate new pages on demand
  };
}

export async function getStaticProps({ params }) {
  const post = await getPostBySlug(params.slug);
  
  return {
    props: { post },
    revalidate: 3600 // Regenerate every hour (ISR)
  };
}

// ✅ Benefits:
// - Pre-rendered at build time
// - Incredibly fast (CDN)
// - Best SEO (search engines see complete content)
// - Low server cost
```

**SSR (Server-Side Rendering) - Use when:**
```javascript
// pages/products/[id].js
export default function Product({ product }) {
  return <h1>{product.name}</h1>;
}

export async function getServerSideProps({ params }) {
  const product = await getProductById(params.id);
  
  return {
    props: { product },
    revalidate: 60
  };
}

// ✅ Use when:
// - Content changes frequently
// - User-specific content
// - Can't pre-generate (millions of paths)
// - Realtime data needed

// ❌ Issues for SEO:
// - Slower TTFB (Time to First Byte)
// - Higher server load
// - Time-dependent crawling
```

**ISR (Incremental Static Regeneration) - BEST OF BOTH:**
```javascript
export async function getStaticProps({ params }) {
  const post = await getPost(params.slug);
  
  return {
    props: { post },
    revalidate: 3600  // Regenerate every hour
    // If someone visits after 1 hour, serve stale
    // while regenerating in background
  };
}
```

---

### Câu hỏi 6: SEO-friendly URL structure dalam Next.js?

**Trả lời:**

```javascript
// ✅ Good URL structure (semantic, descriptive)
/blog                          // Blog home
/blog/seo-tips-2024           // Blog post
/blog/category/nextjs         // Category
/products                      // Products home
/products/coffee-beans        // Product detail
/products/category/accessories // Product category
/docs/getting-started         // Documentation

// ❌ Bad URL structure
/p/123
/article?id=456
/product.php?pid=789
/index.php?page=blog&id=123

// Next.js file structure that produces good URLs:
app/
  ├── page.js                    → /
  ├── about/
  │   └── page.js               → /about
  ├── blog/
  │   ├── page.js               → /blog
  │   ├── [slug]/
  │   │   └── page.js           → /blog/[slug]
  │   └── category/
  │       └── [category]/
  │           └── page.js       → /blog/category/[category]
  ├── products/
  │   ├── page.js               → /products
  │   ├── [id]/
  │   │   └── page.js           → /products/[id]
  │   └── category/
  │       └── [category]/
  │           └── page.js       → /products/category/[category]
```

---

### Câu hỏi 7: Structured data (Schema) trong Next.js?

**Trả lời:**

```jsx
// app/blog/[slug]/page.js
import { generateMetadata } from '@/lib/seo';

export default function BlogPost({ post }) {
  const schema = {
    '@context': 'https://schema.org',
    '@type': 'BlogPosting',
    headline: post.title,
    description: post.excerpt,
    image: post.image,
    datePublished: post.publishedAt,
    dateModified: post.updatedAt,
    author: {
      '@type': 'Person',
      name: 'John Doe'
    },
    publisher: {
      '@type': 'Organization',
      name: 'My Blog',
      logo: {
        '@type': 'ImageObject',
        url: 'https://example.com/logo.png'
      }
    }
  };

  return (
    <>
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(schema) }}
      />
      
      <article>
        <h1>{post.title}</h1>
        <time dateTime={post.publishedAt}>
          {new Date(post.publishedAt).toLocaleDateString()}
        </time>
        <img src={post.image} alt={post.title} />
        <p>{post.content}</p>
      </article>
    </>
  );
}
```

**Reusable helper:**
```javascript
// lib/schema.js
export function generateJsonLd(data) {
  return (
    <script
      type="application/ld+json"
      dangerouslySetInnerHTML={{ __html: JSON.stringify(data) }}
      key={`schema-${data['@type']}`}
    />
  );
}

// Usage
import { generateJsonLd } from '@/lib/schema';

export default function Page() {
  const schema = {
    '@context': 'https://schema.org',
    '@type': 'BlogPosting',
    // ... schema data
  };

  return (
    <>
      {generateJsonLd(schema)}
      <article>...</article>
    </>
  );
}
```

---

### Câu hỏi 8: Core Web Vitals & Performance SEO trong Next.js?

**Trả lời:**

Google cares about user experience metrics.

**3 Core Web Vitals:**

| Metric | Target | Solution |
|--------|--------|----------|
| **LCP** (Largest Contentful Paint) | < 2.5s | Optimize images, code split, reduce JS |
| **FID** (First Input Delay) | < 100ms | Reduce main thread blocking |
| **CLS** (Cumulative Layout Shift) | < 0.1 | Reserve space for dynamic content |

```jsx
// ✅ Optimize LCP (Largest Contentful Paint)
// 1. Prioritize above-the-fold images
<Image
  src={heroImage}
  alt="Hero"
  priority  // ✅ High priority
  width={1200}
  height={600}
/>

// 2. Dynamic import for non-critical components
import dynamic from 'next/dynamic';

const Comments = dynamic(() => import('@/components/Comments'), {
  loading: () => <div>Loading...</div>
});

// 3. Code splitting
// next.js does this automatically

// ✅ Optimize FID (First Input Delay)
// Move heavy computation to Web Worker
// Use useTransition (React 18)
import { useTransition } from 'react';

function SearchPage() {
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  const handleSearch = (query) => {
    startTransition(() => {
      // Heavy filtering/sorting
      const filtered = expensiveFilter(allResults, query);
      setResults(filtered);
    });
  };

  return (
    <>
      <input onChange={(e) => handleSearch(e.target.value)} />
      {isPending && <p>Searching...</p>}
      {results.map(r => <div key={r.id}>{r.name}</div>)}
    </>
  );
}

// ✅ Optimize CLS (Cumulative Layout Shift)
// Reserve space for dynamic content
<div style={{ height: '400px', overflow: 'hidden' }}>
  <Image
    src={image}
    alt="Product"
    width={400}
    height={400}
    placeholder="blur" // ✅ Prevents jump
  />
</div>

// Or use skeleton
function ProductCard() {
  const [isLoaded, setIsLoaded] = useState(false);

  return (
    <div style={{ minHeight: '300px' }}>
      {!isLoaded && <Skeleton height={300} />}
      <Image
        src={product.image}
        alt={product.name}
        width={300}
        height={300}
        onLoadingComplete={() => setIsLoaded(true)}
      />
    </div>
  );
}
```

**Monitor Core Web Vitals:**
```javascript
// app/layout.js
import { SpeedInsights } from "@vercel/speed-insights/next"
import { Analytics } from "@vercel/analytics/react"

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        {children}
        <Analytics />
        <SpeedInsights />
      </body>
    </html>
  )
}
```

---

### Câu hỏi 9: Handle dynamic routes & canonicals trong Next.js?

**Trả lời:**

```jsx
// pages/products/[id].js
import Head from 'next/head';
import { useRouter } from 'next/router';

export default function Product({ product }) {
  const router = useRouter();
  const canonicalUrl = `https://example.com${router.asPath}`;

  return (
    <>
      <Head>
        <title>{product.name}</title>
        <meta name="description" content={product.description} />
        
        {/* ✅ Canonical tag */}
        <link rel="canonical" href={canonicalUrl} />
        
        {/* ✅ Prevent duplicate content with trailing slash */}
        <link rel="canonical" href={canonicalUrl.replace(/\/$/, '')} />
      </Head>
      
      <h1>{product.name}</h1>
    </>
  );
}

// ✅ Handle dynamic routes properly
export async function getStaticPaths() {
  const products = await getAllProducts();
  
  return {
    paths: products.map(p => ({
      params: { id: p.id }
    })),
    fallback: 'blocking'
  };
}

export async function getStaticProps({ params }) {
  const product = await getProduct(params.id);
  
  if (!product) {
    return { notFound: true }; // ✅ Return 404
  }
  
  return {
    props: { product },
    revalidate: 3600
  };
}
```

---

### Câu hỏi 10: Prevent crawling behind authentication?

**Trả lời:**

```javascript
// middleware.js (Next.js 12+)
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const pathname = request.nextUrl.pathname;

  // Block crawlers from authenticated pages
  if (pathname.startsWith('/dashboard')) {
    const ua = request.headers.get('user-agent') || '';
    
    if (isCrawler(ua)) {
      return new NextResponse('Forbidden', { status: 403 });
    }
  }

  return NextResponse.next();
}

function isCrawler(ua: string) {
  return /bot|crawl|spider|googlebot/i.test(ua);
}

export const config = {
  matcher: ['/dashboard/:path*']
};
```

---

## 🔧 PHẦN 3: SEO TOOLS & MONITORING

### Essential SEO Tools:

```
1. Google Search Console:
   - Monitor indexing
   - Crawl errors
   - Search performance
   - Submit sitemaps

2. Google Analytics 4:
   - Organic traffic
   - User behavior
   - Conversion tracking
   - Audience analysis

3. Google PageSpeed Insights:
   - Core Web Vitals
   - Lighthouse score
   - Performance recommendations

4. Screaming Frog:
   - Crawl website
   - Find broken links
   - Analyze metadata
   - Check crawlability

5. Lighthouse (DevTools):
   - Performance
   - Accessibility
   - Best practices
   - SEO audit

6. SEMrush / Ahrefs:
   - Backlink analysis
   - Keyword research
   - Competitor analysis
   - Rank tracking

7. Yoast SEO / Rankmath:
   - Content optimization
   - Keyword analysis
   - Meta tag templates
   - Focus keyword

8. Schema.org Validator:
   - Validate structured data
   - Preview rich snippets
   - Find errors
```

---

## 📝 PHẦN 4: SEO CHECKLIST

### Before launch:

```
□ Meta tags (title, description, keywords)
□ Favicon
□ Open Graph & Twitter cards
□ Canonical tags
□ Robots.txt
□ Sitemap
□ Structured data (Schema)
□ Mobile responsive
□ Core Web Vitals (LCP, FID, CLS)
□ Alt text for all images
□ Internal linking strategy
□ 404 error page
□ Redirects set up (301)
□ HTTPS enabled
□ Analytics installed
□ Search Console verified
```

### Content optimization:

```
□ Unique title tag (50-60 chars)
□ Compelling meta description (150-160 chars)
□ H1 tag (only 1 per page)
□ Heading hierarchy (H2, H3)
□ Keyword usage (1-2% density)
□ Minimum 300 words (blog posts)
□ Semantic HTML
□ Links to authority sources
□ Internal linking to related content
□ Image optimization
□ Alt text descriptive
□ Content updateness
□ Original, valuable content
```

### Technical optimization:

```
□ Site speed < 3 seconds (LCP < 2.5s)
□ Mobile-friendly
□ Descriptive URLs
□ SSL/HTTPS
□ XML Sitemap
□ Robots.txt configured
□ No duplicate content
□ Proper redirects (301)
□ Crawlable links (not behind JS)
□ Navigation clear
□ No broken links
□ Proper HTTP status codes
□ Server response time < 600ms
```

### Off-page optimization:

```
□ Quality backlinks (DA 40+)
□ Relevant link sources
□ Brand mentions
□ Social sharing
□ Google Business Profile (local)
□ NAP consistency (local)
□ Reviews/ratings (local)
```

---

## 🎯 PHẦN 5: COMMON MISTAKES & SOLUTIONS

### Mistake 1: Not optimizing for mobile

```javascript
// ❌ Bad
<div style={{ display: 'none' }}
   className="desktop-only">
  Desktop menu
</div>

// ✅ Good: Responsive design
<nav className="hidden md:flex">
  Desktop menu
</nav>
```

### Mistake 2: Ignoring Core Web Vitals

```javascript
// ❌ Bad: Causes layout shift
<Image
  src={image}
  alt="Product"
  // No dimensions specified
/>

// ✅ Good: Reserve space
<Image
  src={image}
  alt="Product"
  width={300}
  height={300}
  placeholder="blur"
/>
```

### Mistake 3: Not using structured data

```jsx
// ❌ Bad: Search engines can't understand
<h1>Coffee Shops in NYC</h1>
<p>Rating: 4.5/5</p>

// ✅ Good: Structured data
{generateJsonLd({
  '@context': 'https://schema.org',
  '@type': 'LocalBusiness',
  name: 'Gramercy Coffee',
  aggregateRating: {
    '@type': 'AggregateRating',
    ratingValue: '4.5'
  }
})}
```

### Mistake 4: Duplicate content

```javascript
// ❌ Bad: Same content on multiple URLs
/blog/coffee
/blog/coffee/
/blog?id=coffee

// ✅ Good: Canonical tag
<link rel="canonical" 
      href="https://example.com/blog/coffee" />
```

### Mistake 5: Slow page speed

```javascript
// ❌ Bad: Heavy images
<img src="/hero.jpg" /> {/* 5MB unoptimized */}

// ✅ Good: Optimized images
<Image
  src="/hero.jpg"
  alt="Hero"
  width={1200}
  height={600}
  quality={80}
  priority
/>
// Auto generates WebP, multiple sizes, lazy loads
```

---

## 💡 PHẦN 6: NEXTJS SEO TEMPLATE

```jsx
// app/layout.js (Root layout with SEO)
import type { Metadata } from 'next';

export const metadata: Metadata = {
  metadataBase: new URL('https://example.com'),
  title: {
    default: 'My Site',
    template: '%s | My Site',
  },
  description: 'Default site description',
  robots: {
    index: true,
    follow: true,
  },
  openGraph: {
    type: 'website',
    locale: 'en_US',
    url: 'https://example.com',
    siteName: 'My Site',
  },
  twitter: {
    card: 'summary_large_image',
    creator: '@yourhandle',
  },
};

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <head>
        <link rel="icon" href="/favicon.ico" />
        <link rel="apple-touch-icon" href="/apple-touch-icon.png" />
        <link rel="canonical" href="https://example.com" />
      </head>
      <body>{children}</body>
    </html>
  );
}

// app/blog/[slug]/page.js (Dynamic page with SEO)
import { generateMetadata } from '@/lib/seo';
import { generateJsonLd } from '@/lib/schema';
import Image from 'next/image';

export async function generateMetadata({ params }) {
  const post = await getPost(params.slug);

  return generateMetadata(
    post.title,
    post.excerpt,
    post.image,
    `https://example.com/blog/${post.slug}`
  );
}

export default function BlogPost({ params }) {
  const post = getPost(params.slug);

  const schema = {
    '@context': 'https://schema.org',
    '@type': 'BlogPosting',
    headline: post.title,
    description: post.excerpt,
    image: post.image,
    datePublished: post.publishedAt,
    author: { '@type': 'Person', name: 'John Doe' },
  };

  return (
    <>
      {generateJsonLd(schema)}

      <article>
        <h1>{post.title}</h1>
        <time dateTime={post.publishedAt}>
          {new Date(post.publishedAt).toLocaleDateString()}
        </time>

        <Image
          src={post.image}
          alt={post.title}
          width={800}
          height={400}
          priority
          placeholder="blur"
        />

        <p>{post.content}</p>
      </article>
    </>
  );
}
```

---

## 🚀 PHẦN 7: ADVANCED SEO

### SSR for dynamic content:

```javascript
// pages/products/[id].js
export async function getServerSideProps({ params }) {
  // Fetch fresh data every request
  const product = await fetchProduct(params.id);

  if (!product) {
    return { notFound: true };
  }

  return {
    props: { product },
    revalidate: 60 // Cache for 60 seconds
  };
}
```

### Prerendering with fallback:

```javascript
export async function getStaticPaths() {
  const products = await getAllProducts();

  return {
    // Prerender top 100 products
    paths: products.slice(0, 100).map(p => ({
      params: { id: p.id }
    })),
    
    // Generate others on demand
    fallback: 'blocking'
  };
}
```

### International SEO (i18n):

```javascript
// next-i18next.config.js
const path = require('path');

module.exports = {
  i18n: {
    defaultLocale: 'en',
    locales: ['en', 'es', 'fr', 'de']
  },
  localePath: path.resolve('./public/locales')
};

// pages/[lang]/blog/[slug].js
export async function getStaticPaths() {
  const langs = ['en', 'es', 'fr'];
  const posts = await getAllPosts();

  const paths = langs.flatMap(lang =>
    posts.map(post => ({
      params: { lang, slug: post.slug }
    }))
  );

  return { paths, fallback: 'blocking' };
}

// ✅ Add alternate links
<link rel="alternate" href="https://example.com/en/blog/post" hrefLang="en" />
<link rel="alternate" href="https://example.com/es/blog/post" hrefLang="es" />
<link rel="alternate" href="https://example.com/x-default" hrefLang="x-default" />
```

---

## 📊 PHẦN 8: MEASURE SEO SUCCESS

```javascript
// Metrics to track:

1. Organic Traffic:
   - Sessions from organic search
   - Month-over-month growth
   - Target: 10-20% growth/month

2. Keyword Rankings:
   - Track position of target keywords
   - Focus on top 3 positions
   - Identify opportunities (4-10 positions)

3. Bounce Rate:
   - Percentage of single-page sessions
   - Target: < 50%
   - Indicates content relevance

4. Time on Page:
   - Average session duration
   - Target: > 2 minutes (blog posts)
   - Indicates engagement

5. Click-through Rate (CTR):
   - Clicks from search results / impressions
   - Target: 3-5% for position 3
   - Optimize titles & descriptions

6. Conversion Rate:
   - Visitors → Customers/Leads
   - Track organic only
   - Compare to paid channels

7. Core Web Vitals:
   - LCP, FID, CLS scores
   - Google ranking factor
   - Monitor in Search Console

8. Crawl Efficiency:
   - Pages crawled per day
   - Crawl errors
   - Monitor in Search Console
```

---

## 🎓 INTERVIEW TIPS

**Good answers show:**
✅ Understanding of all 3 SEO pillars (technical, on-page, off-page)
✅ Knowledge of Next.js specific features
✅ Real-world considerations (Core Web Vitals, mobile)
✅ Practical examples & code
✅ Trade-offs (SSR vs SSG)
✅ Tools & monitoring

**Avoid:**
❌ "Just use Yoast plugin"
❌ "SEO is only about keywords"
❌ Black-hat techniques
❌ No mention of mobile/performance
❌ Ignoring user experience

---

**Good luck with your interview! 🚀**
