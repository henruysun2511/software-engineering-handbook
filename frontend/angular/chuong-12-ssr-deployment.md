# Chương 12: SSR & Deployment

## Giới thiệu chương

Chương cuối cùng giải quyết bài toán đưa Angular application lên production: Server-Side Rendering để cải thiện performance và SEO, tối ưu hóa build size, quản lý environment configuration, và các chiến lược deployment thực tế từ static hosting đến Docker container.

---

## 12.1 SSR vs CSR vs SSG — Chọn đúng cho từng loại app

| Chiến lược | Phù hợp | Không phù hợp |
|------------|---------|---------------|
| **CSR** (Client-Side Rendering) | Dashboard nội bộ, app sau login, real-time app | Landing page cần SEO, e-commerce |
| **SSR** (Server-Side Rendering) | E-commerce, blog, app cần SEO | App hoàn toàn private, heavy real-time |
| **SSG** (Static Site Generation) | Blog, docs, marketing page | App có data thay đổi thường xuyên |
| **Hybrid** | App vừa có public page vừa có dashboard | — |

Angular 18 hỗ trợ cả ba chiến lược trong cùng một project thông qua `@angular/ssr`.

---

## 12.2 Cài đặt và Cấu hình @angular/ssr

### Thêm SSR vào project có sẵn

```bash
ng add @angular/ssr
```

Lệnh này tạo ra:
- `server.ts` — Express server để serve SSR
- `app.config.server.ts` — Config riêng cho server-side
- Cập nhật `angular.json` với build targets mới

### Cấu trúc sau khi thêm SSR

```
src/
├── app/
│   ├── app.config.ts          # Shared config (browser + server)
│   └── app.config.server.ts   # Server-only config
├── main.ts                    # Browser bootstrap
└── main.server.ts             # Server bootstrap
server.ts                      # Express server
```

```typescript
// app.config.ts — Shared config
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(
      withFetch(), // Dùng Fetch API — hoạt động cả browser lẫn Node.js
      withInterceptors([authInterceptor, errorInterceptor])
    ),
    provideClientHydration(), // Bật hydration — Angular 17+
    provideAnimationsAsync(),
  ],
};
```

```typescript
// app.config.server.ts — Chỉ chạy trên server
import { mergeApplicationConfig, ApplicationConfig } from '@angular/core';
import { provideServerRendering } from '@angular/platform-server';
import { appConfig } from './app.config';

const serverConfig: ApplicationConfig = {
  providers: [
    provideServerRendering(),
    // Cung cấp API URL cho server-side (không có CORS)
    {
      provide: API_URL,
      useValue: process.env['INTERNAL_API_URL'] ?? 'http://localhost:3000/api',
    },
  ],
};

export const config = mergeApplicationConfig(appConfig, serverConfig);
```

```typescript
// server.ts — Express server
import {
  AngularNodeAppEngine,
  createNodeRequestHandler,
  isMainModule,
  writeResponseToNodeResponse,
} from '@angular/ssr/node';
import express from 'express';
import { dirname, resolve } from 'node:path';
import { fileURLToPath } from 'node:url';

const serverDistFolder = dirname(fileURLToPath(import.meta.url));
const browserDistFolder = resolve(serverDistFolder, '../browser');

const app = express();
const angularApp = new AngularNodeAppEngine();

// Serve static files từ browser dist folder
app.use(
  express.static(browserDistFolder, {
    maxAge: '1y',         // Cache static assets 1 năm
    index: false,         // Không serve index.html cho routes
    redirect: false,
  })
);

// Health check endpoint
app.get('/health', (_req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// Tất cả requests khác → Angular SSR
app.use('/**', (req, res, next) => {
  angularApp
    .handle(req)
    .then((response) => {
      if (response) {
        writeResponseToNodeResponse(response, res);
      } else {
        next();
      }
    })
    .catch(next);
});

if (isMainModule(import.meta.url)) {
  const port = process.env['PORT'] || 4000;
  app.listen(port, () => {
    console.log(`Server running on http://localhost:${port}`);
  });
}

export const reqHandler = createNodeRequestHandler(app);
```

---

## 12.3 Viết Code tương thích SSR

### `isPlatformBrowser` và `isPlatformServer`

Khi SSR render, không có browser APIs (window, document, localStorage). Phải kiểm tra platform trước khi dùng:

```typescript
// core/services/storage.service.ts
@Injectable({ providedIn: 'root' })
export class StorageService {
  private readonly platformId = inject(PLATFORM_ID);
  private readonly isBrowser = isPlatformBrowser(this.platformId);

  get(key: string): string | null {
    if (!this.isBrowser) return null;
    return localStorage.getItem(key);
  }

  set(key: string, value: string): void {
    if (!this.isBrowser) return;
    localStorage.setItem(key, value);
  }

  remove(key: string): void {
    if (!this.isBrowser) return;
    localStorage.removeItem(key);
  }
}
```

```typescript
// Dùng DOCUMENT injection token thay vì document trực tiếp
@Injectable({ providedIn: 'root' })
export class ThemeService {
  private readonly document = inject(DOCUMENT);
  private readonly platformId = inject(PLATFORM_ID);

  applyTheme(isDark: boolean): void {
    if (!isPlatformBrowser(this.platformId)) return;
    this.document.documentElement.classList.toggle('dark-theme', isDark);
  }
}
```

```typescript
// Component tránh dùng window/document trực tiếp
@Component({
  selector: 'app-scroll-to-top',
  standalone: true,
  template: `
    @if (showButton()) {
      <button mat-fab (click)="scrollToTop()">
        <mat-icon>arrow_upward</mat-icon>
      </button>
    }
  `,
})
export class ScrollToTopComponent implements OnInit {
  private readonly platformId = inject(PLATFORM_ID);
  protected readonly showButton = signal(false);

  ngOnInit(): void {
    if (!isPlatformBrowser(this.platformId)) return;

    // Chỉ chạy trong browser
    fromEvent(window, 'scroll')
      .pipe(
        throttleTime(100),
        map(() => window.scrollY > 300),
        distinctUntilChanged()
      )
      .subscribe((show) => this.showButton.set(show));
  }

  protected scrollToTop(): void {
    if (!isPlatformBrowser(this.platformId)) return;
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }
}
```

### TransferState — Tránh double-fetch data

Khi SSR render, Angular fetch data trên server. Sau khi hydrate trên browser, Angular lại fetch lần nữa — lãng phí và làm nháy giao diện. `TransferState` giải quyết vấn đề này:

```typescript
// core/utils/transfer-state.util.ts
import { makeStateKey, TransferState } from '@angular/core';

/**
 * Wrapper tự động handle TransferState pattern.
 * - Trên server: fetch data rồi lưu vào TransferState
 * - Trên browser: đọc từ TransferState, nếu có thì dùng, không thì fetch
 */
export function withTransferState<T>(
  key: string,
  fetchFn: () => Observable<T>
): Observable<T> {
  const transferState = inject(TransferState);
  const platformId = inject(PLATFORM_ID);
  const stateKey = makeStateKey<T>(key);

  if (isPlatformBrowser(platformId)) {
    // Browser: lấy từ TransferState
    if (transferState.hasKey(stateKey)) {
      const cached = transferState.get(stateKey, null as unknown as T);
      transferState.remove(stateKey); // Dùng một lần rồi xóa
      return of(cached);
    }
    return fetchFn();
  }

  // Server: fetch rồi lưu vào TransferState
  return fetchFn().pipe(
    tap((data) => transferState.set(stateKey, data))
  );
}
```

```typescript
// Sử dụng trong service
@Injectable({ providedIn: 'root' })
export class ProductService {
  private readonly http = inject(HttpClient);

  getFeaturedProducts(): Observable<Product[]> {
    return withTransferState('featured-products', () =>
      this.http.get<Product[]>('/api/products/featured')
    );
  }
}
```

### Meta Tags cho SEO

```typescript
// core/services/seo.service.ts
@Injectable({ providedIn: 'root' })
export class SeoService {
  private readonly meta = inject(Meta);
  private readonly title = inject(Title);

  setPageMeta(config: {
    title: string;
    description?: string;
    image?: string;
    url?: string;
    type?: string;
  }): void {
    const fullTitle = `${config.title} | My App`;

    // Title
    this.title.setTitle(fullTitle);

    // Standard meta
    this.meta.updateTag({ name: 'description', content: config.description ?? '' });

    // Open Graph
    this.meta.updateTag({ property: 'og:title', content: fullTitle });
    this.meta.updateTag({ property: 'og:description', content: config.description ?? '' });
    this.meta.updateTag({ property: 'og:type', content: config.type ?? 'website' });

    if (config.image) {
      this.meta.updateTag({ property: 'og:image', content: config.image });
    }
    if (config.url) {
      this.meta.updateTag({ property: 'og:url', content: config.url });
    }

    // Twitter Card
    this.meta.updateTag({ name: 'twitter:card', content: 'summary_large_image' });
    this.meta.updateTag({ name: 'twitter:title', content: fullTitle });
    if (config.description) {
      this.meta.updateTag({ name: 'twitter:description', content: config.description });
    }
  }
}

// Sử dụng trong component
@Component({ ... })
export class ProductDetailComponent implements OnInit {
  private readonly seoService = inject(SeoService);

  ngOnInit(): void {
    const product = this.product(); // từ resolver hoặc route
    if (product) {
      this.seoService.setPageMeta({
        title: product.name,
        description: product.description,
        image: product.imageUrl,
        type: 'product',
      });
    }
  }
}
```

---

## 12.4 Build và Tối ưu

### Build Production

```bash
# Build production — tạo ra thư mục dist/
ng build --configuration production

# Phân tích bundle size
ng build --configuration production --stats-json
npx webpack-bundle-analyzer dist/my-app/browser/stats.json

# Build SSR
ng build && ng run my-app:server
```

### angular.json — Cấu hình Build

```json
{
  "configurations": {
    "production": {
      "budgets": [
        {
          "type": "initial",
          "maximumWarning": "500kb",
          "maximumError": "1mb"
        },
        {
          "type": "anyComponentStyle",
          "maximumWarning": "4kb",
          "maximumError": "8kb"
        }
      ],
      "outputHashing": "all",
      "optimization": {
        "scripts": true,
        "styles": {
          "minify": true,
          "inlineCritical": true
        },
        "fonts": {
          "inline": true
        }
      }
    }
  }
}
```

### Tối ưu Lazy Loading và Bundle

```typescript
// Tách route-level code splitting tối đa
// app.routes.ts — mọi feature đều lazy load
export const routes: Routes = [
  {
    path: 'dashboard',
    loadComponent: () =>
      import('./features/dashboard/dashboard.component').then(
        (m) => m.DashboardComponent
      ),
  },

  // Heavy dependencies — lazy load khi cần
  {
    path: 'charts',
    loadComponent: () =>
      import('./features/charts/charts.component').then(
        (m) => m.ChartsComponent
      ),
    // ChartsComponent import D3.js hoặc Chart.js chỉ khi user vào route này
  },
];
```

```typescript
// Tránh import toàn bộ thư viện
// ❌ Import cả lodash — 70KB
import _ from 'lodash';

// ✓ Import chỉ function cần dùng — vài KB
import debounce from 'lodash/debounce';
import groupBy from 'lodash/groupBy';

// ✓ Hoặc dùng lodash-es với tree-shaking
import { debounce, groupBy } from 'lodash-es';
```

---

## 12.5 Environment Configuration

### Runtime Configuration — Không hardcode vào build

Thay vì bake config vào bundle lúc build (như `environment.ts`), load config từ server lúc runtime — cho phép thay đổi config mà không cần rebuild:

```typescript
// core/models/app-config.model.ts
export interface AppConfig {
  apiUrl: string;
  wsUrl: string;
  features: {
    newDashboard: boolean;
    advancedReports: boolean;
  };
  analytics: {
    enabled: boolean;
    trackingId?: string;
  };
}
```

```typescript
// core/services/app-config.service.ts
@Injectable({ providedIn: 'root' })
export class AppConfigService {
  private readonly http = inject(HttpClient);
  private readonly config = signal<AppConfig | null>(null);

  readonly isReady = computed(() => this.config() !== null);

  load(): Observable<void> {
    return this.http.get<AppConfig>('/assets/config/app-config.json').pipe(
      tap((config) => this.config.set(config)),
      map(() => void 0),
      catchError(() => {
        // Fallback config nếu không load được
        this.config.set({
          apiUrl: '/api',
          wsUrl: '',
          features: { newDashboard: false, advancedReports: false },
          analytics: { enabled: false },
        });
        return of(void 0);
      })
    );
  }

  get<K extends keyof AppConfig>(key: K): AppConfig[K] {
    const config = this.config();
    if (!config) throw new Error('AppConfig not loaded');
    return config[key];
  }

  isFeatureEnabled(feature: keyof AppConfig['features']): boolean {
    return this.config()?.features[feature] ?? false;
  }
}

// app.config.ts — load config trước khi app khởi động
{
  provide: APP_INITIALIZER,
  useFactory: () => {
    const configService = inject(AppConfigService);
    return () => configService.load();
  },
  multi: true,
},
{
  provide: API_URL,
  useFactory: () => inject(AppConfigService).get('apiUrl'),
},
```

```json
// public/assets/config/app-config.json — có thể thay đổi không cần rebuild
{
  "apiUrl": "https://api.example.com",
  "wsUrl": "wss://api.example.com",
  "features": {
    "newDashboard": true,
    "advancedReports": false
  },
  "analytics": {
    "enabled": true,
    "trackingId": "GA-XXXXX"
  }
}
```

---

## 12.6 Deployment

### Deploy Static (CSR) lên Vercel / Netlify

```bash
# Build
ng build --configuration production

# Vercel — tự detect Angular
vercel --prod

# Netlify
netlify deploy --prod --dir=dist/my-app/browser
```

```toml
# netlify.toml — cần thiết cho Angular SPA routing
[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

```json
// vercel.json
{
  "rewrites": [
    { "source": "/((?!api/.*).*)", "destination": "/index.html" }
  ],
  "headers": [
    {
      "source": "/assets/(.*)",
      "headers": [
        { "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }
      ]
    }
  ]
}
```

### Deploy SSR lên Node.js Server

```bash
# Build cả browser và server
ng build
ng run my-app:server:production

# Chạy server
node dist/my-app/server/server.mjs
```

### Nginx Config cho Angular SPA

```nginx
# /etc/nginx/sites-available/my-angular-app
server {
    listen 80;
    server_name example.com www.example.com;

    # Redirect HTTP → HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    root /var/www/my-app/browser;
    index index.html;

    # Gzip compression
    gzip on;
    gzip_types text/plain application/javascript text/css application/json;
    gzip_min_length 1000;

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2?)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Cache config.json ngắn hơn (runtime config có thể thay đổi)
    location /assets/config/ {
        expires 5m;
        add_header Cache-Control "public";
    }

    # API proxy — tránh CORS
    location /api/ {
        proxy_pass http://localhost:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Angular routing — tất cả routes về index.html
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### Docker — Production Container

```dockerfile
# Dockerfile — Multi-stage build
# Stage 1: Build Angular app
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files trước — Docker layer cache
COPY package*.json ./
RUN npm ci --legacy-peer-deps

COPY . .
RUN npm run build -- --configuration production

# Stage 2: Production server (SSR)
FROM node:20-alpine AS server

WORKDIR /app

# Chỉ copy những gì cần thiết
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./

# Install chỉ production dependencies
RUN npm ci --omit=dev --legacy-peer-deps

# Security: chạy với user không phải root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 4000

ENV NODE_ENV=production
ENV PORT=4000

CMD ["node", "dist/my-app/server/server.mjs"]
```

```dockerfile
# Dockerfile.nginx — Nếu chỉ deploy CSR (không SSR)
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --legacy-peer-deps
COPY . .
RUN npm run build -- --configuration production

FROM nginx:alpine AS production

# Copy Angular build output
COPY --from=builder /app/dist/my-app/browser /usr/share/nginx/html

# Copy nginx config
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

```yaml
# docker-compose.yml — Local development và staging
version: '3.9'

services:
  angular-app:
    build:
      context: .
      dockerfile: Dockerfile
      target: server
    ports:
      - '4000:4000'
    environment:
      - NODE_ENV=production
      - PORT=4000
      - INTERNAL_API_URL=http://api:3000
    depends_on:
      - api
    restart: unless-stopped

  api:
    image: my-api:latest
    ports:
      - '3000:3000'
    environment:
      - DATABASE_URL=postgresql://user:password@postgres:5432/mydb
    depends_on:
      - postgres

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### GitHub Actions — CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linting
        run: npm run lint

      - name: Run unit tests
        run: npm run test -- --coverage --watchAll=false

      - name: Upload coverage
        uses: codecov/codecov-action@v3

  build:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build production
        run: npm run build -- --configuration production

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
          retention-days: 7

  deploy:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Download artifacts
        uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            docker pull ghcr.io/${{ github.repository }}:latest
            docker stop angular-app || true
            docker rm angular-app || true
            docker run -d \
              --name angular-app \
              --restart unless-stopped \
              -p 4000:4000 \
              -e NODE_ENV=production \
              ghcr.io/${{ github.repository }}:latest
```

---

## 12.7 Performance Monitoring Production

### Thiết lập Web Vitals Tracking

```typescript
// core/services/performance.service.ts
@Injectable({ providedIn: 'root' })
export class PerformanceService {
  private readonly platformId = inject(PLATFORM_ID);

  init(): void {
    if (!isPlatformBrowser(this.platformId)) return;

    // Đo Core Web Vitals
    this.measureWebVitals();
  }

  private measureWebVitals(): void {
    // LCP, FID, CLS — sử dụng thư viện web-vitals
    import('web-vitals').then(({ onCLS, onFCP, onLCP, onTTFB }) => {
      onCLS(this.reportMetric.bind(this));
      onFCP(this.reportMetric.bind(this));
      onLCP(this.reportMetric.bind(this));
      onTTFB(this.reportMetric.bind(this));
    });
  }

  private reportMetric(metric: { name: string; value: number }): void {
    // Gửi lên analytics
    console.log(`[Web Vital] ${metric.name}: ${metric.value.toFixed(2)}`);

    // Gửi lên Google Analytics
    // gtag('event', metric.name, { value: Math.round(metric.value) });
  }
}

// Khởi động trong app.component.ts
@Component({ ... })
export class AppComponent implements OnInit {
  private readonly performanceService = inject(PerformanceService);

  ngOnInit(): void {
    this.performanceService.init();
  }
}
```

---

## 12.8 Checklist trước khi Deploy Production

```typescript
// Checklist dưới dạng code comment — đọc trước mỗi lần deploy

/**
 * PRE-DEPLOYMENT CHECKLIST
 *
 * BUILD
 * [ ] ng build --configuration production chạy không lỗi
 * [ ] Bundle size trong giới hạn budget (angular.json)
 * [ ] Không có console.log() còn sót trong production code
 *
 * TESTING
 * [ ] Unit tests pass: npm run test
 * [ ] Coverage >= threshold (80%)
 * [ ] E2E tests pass trên staging
 *
 * SECURITY
 * [ ] Không có API keys, credentials trong source code
 * [ ] HTTPS đã bật
 * [ ] CSP (Content Security Policy) headers đã cấu hình
 * [ ] HttpOnly cookies cho sensitive tokens
 * [ ] Input validation ở cả client và server
 *
 * PERFORMANCE
 * [ ] Lazy loading cho tất cả feature routes
 * [ ] Images đã optimize (WebP, đúng kích thước)
 * [ ] Angular Material tree-shaking (không import all-component-themes)
 * [ ] trackBy trong tất cả @for loops
 * [ ] OnPush trong tất cả components
 *
 * SEO (nếu có SSR)
 * [ ] Meta tags đầy đủ (title, description, og:image)
 * [ ] robots.txt và sitemap.xml
 * [ ] Canonical URLs
 *
 * MONITORING
 * [ ] Error tracking setup (Sentry hoặc tương đương)
 * [ ] Performance monitoring setup
 * [ ] Health check endpoint
 * [ ] Logging đầy đủ
 *
 * DEPLOYMENT
 * [ ] Environment variables đã set đúng trên server
 * [ ] Database migrations đã chạy (nếu có)
 * [ ] Rollback plan sẵn sàng
 * [ ] Team đã được thông báo
 */
```

---

## Tổng kết chương và Toàn bộ Giáo trình

Chương này hoàn thiện hành trình từ Angular cơ bản đến production-ready application. Những điểm cốt lõi về SSR và Deployment:

1. **`@angular/ssr`** tích hợp sẵn, dễ thêm vào project có sẵn. `withFetch()` là bắt buộc để HttpClient hoạt động cả browser lẫn Node.js.

2. **`isPlatformBrowser()`** trước khi dùng bất kỳ browser API nào — window, document, localStorage. `DOCUMENT` injection token thay cho `document` trực tiếp.

3. **TransferState** tránh double-fetch: server fetch và lưu data, browser đọc từ state thay vì fetch lại — tránh nháy giao diện khi hydrate.

4. **Runtime configuration** (`/assets/config/app-config.json`) linh hoạt hơn build-time environment files — thay đổi config không cần rebuild Docker image.

5. **Multi-stage Docker build** giảm image size đáng kể — stage build dùng Node + full dependencies, stage production chỉ cần dist files và production deps.

6. **CI/CD pipeline** đảm bảo mọi code vào `main` đều qua test → build → deploy tự động — giảm thiểu human error.

---

## Tổng kết toàn bộ giáo trình

Sau 12 chương, bạn đã có đủ kiến thức để xây dựng Angular enterprise application hoàn chỉnh với stack:

| Tầng | Công nghệ | Chương |
|------|-----------|--------|
| Framework | Angular 18, Standalone Components | 1 |
| DI & Services | Dependency Injection, Injectable | 2 |
| Reactivity | Signals, computed, effect | 3 |
| HTTP & Async | RxJS, HttpClient, Zod | 4 |
| Navigation | Angular Router, Guards, Resolvers | 5 |
| Forms | Reactive Forms, Validators, Zod | 6 |
| State | NgRx Signals Store, withEntities | 7 |
| UI | Angular Material, SCSS, CDK | 8 |
| Architecture | Feature-based, Smart/Dumb, Patterns | 9 |
| Testing | Jest, TestBed, Playwright | 10 |
| Auth | JWT, Interceptors, Guards | 11 |
| Deploy | SSR, Docker, CI/CD | 12 |

Bước tiếp theo: **xây dựng một project thực tế** áp dụng tất cả những gì đã học. Không có gì thay thế được việc đối mặt với các vấn đề thực tế và tự giải quyết chúng.
