# Chương 1: Nền tảng Angular

## Giới thiệu chương

Angular là một framework frontend được phát triển và duy trì bởi Google, ra đời từ năm 2016 như một bản viết lại hoàn toàn của AngularJS. Khác với React — một thư viện tập trung vào việc render giao diện — Angular là một **full framework**: nó cung cấp sẵn giải pháp cho mọi vấn đề phổ biến trong phát triển ứng dụng web, từ routing, HTTP client, form handling, cho đến dependency injection và testing utilities.

Chương này đặt nền tảng để hiểu Angular đúng bản chất: không phải học một công cụ mới theo kiểu "React nhưng khác cú pháp", mà là tiếp cận một triết lý thiết kế phần mềm khác — một triết lý đặt tính nhất quán, khả năng mở rộng và kiến trúc rõ ràng lên hàng đầu.

---

## 1.1 Triết lý và kiến trúc Angular

### Framework vs Library — sự khác biệt cốt lõi

React là một **library**: nó giải quyết một vấn đề cụ thể (rendering UI theo state), còn lại bạn tự quyết định — chọn router nào, state management nào, HTTP client nào, form library nào. Điều này mang lại sự linh hoạt cao, nhưng cũng có nghĩa là mỗi team React có thể có kiến trúc hoàn toàn khác nhau.

Angular là một **opinionated framework**: nó đưa ra câu trả lời mặc định cho hầu hết các vấn đề phổ biến. Router có sẵn. HTTP client có sẵn. Form system có sẵn. Dependency injection có sẵn. Tính "opinionated" này thường bị nhìn nhận là hạn chế, nhưng trong thực tế enterprise development, đây là một lợi thế lớn:

- Developer mới vào team có thể đọc hiểu code ngay vì cấu trúc tuân theo một quy ước chung
- Không mất thời gian tranh luận "dùng thư viện nào"
- Cập nhật framework thường kéo theo cập nhật đồng bộ toàn bộ hệ sinh thái

### Kiến trúc tổng thể

M��t ứng dụng Angular được tổ chức theo các tầng sau:

```
Application
├── Components        — Đơn vị UI, kết hợp template + logic
├── Services          — Business logic, data fetching, shared state
├── Directives        — Mở rộng behavior của HTML element
├── Pipes             — Transform data trong template
├── Guards            — Kiểm soát điều hướng router
└── Interceptors      — Xử lý HTTP request/response toàn cục
```

Các tầng này giao tiếp với nhau thông qua **Dependency Injection** — hệ thống trung tâm của Angular sẽ được trình bày chi tiết ở Chương 2.

### Angular CLI — công cụ trung tâm

Angular CLI không chỉ là công cụ tạo project — nó là trung tâm của mọi workflow: tạo file, build, test, lint, deploy. Khác với React (CRA, Vite, Next.js đều khác nhau), Angular CLI là chuẩn duy nhất.

```bash
# Tạo project mới với các lựa chọn mặc định production-ready
ng new my-app \
  --style=scss \
  --routing=true \
  --strict=true \
  --standalone=true

# Tạo component mới
ng generate component features/user/user-profile
# Viết tắt
ng g c features/user/user-profile

# Tạo service
ng g s core/services/auth

# Build production
ng build --configuration production

# Chạy tests
ng test

# Kiểm tra bundle size
ng build --stats-json && npx webpack-bundle-analyzer dist/my-app/stats.json
```

### Cấu trúc thư mục chuẩn

Angular CLI tạo ra cấu trúc mặc định, nhưng các team chuyên nghiệp thường tổ chức theo **feature-based architecture**:

```
src/
├── app/
│   ├── core/                    # Singleton services, guards, interceptors
│   │   ├── guards/
│   │   ├── interceptors/
│   │   └── services/
│   ├── shared/                  # Components, pipes, directives dùng chung
│   │   ├── components/
│   │   ├── directives/
│   │   └── pipes/
│   ├── features/                # Mỗi feature là một thư mục độc lập
│   │   ├── auth/
│   │   │   ├── login/
│   │   │   ├── register/
│   │   │   └── auth.routes.ts
│   │   └── dashboard/
│   ├── app.component.ts
│   ├── app.config.ts            # Cấu hình ứng dụng (thay AppModule)
│   └── app.routes.ts
├── environments/
│   ├── environment.ts
│   └── environment.production.ts
└── styles/
    ├── _variables.scss
    ├── _mixins.scss
    └── styles.scss
```

---

## 1.2 Standalone Components — Mô hình mới của Angular 18

### NgModule và vấn đề của nó

Trước Angular 14, mọi component đều phải thuộc về một **NgModule** — một class tổng hợp khai báo components, imports, exports và providers. NgModule giải quyết bài toán tổ chức code, nhưng tạo ra overhead đáng kể: developer phải khai báo component ở hai nơi (file component và NgModule), và sơ đồ dependency giữa các module thường trở nên khó kiểm soát ở ứng dụng lớn.

Angular 14 giới thiệu **Standalone Components** như một tùy chọn, và Angular 18 đã biến nó thành mặc định. Standalone component tự quản lý dependency của mình — không cần NgModule trung gian.

### Khai báo Standalone Component

```typescript
// user-profile.component.ts
import { Component, inject, input, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterLink } from '@angular/router';
import { MatCardModule } from '@angular/material/card';
import { MatButtonModule } from '@angular/material/button';
import { UserService } from '@core/services/user.service';
import { User } from '@core/models/user.model';

@Component({
  selector: 'app-user-profile',
  standalone: true,                          // Khai báo standalone
  imports: [                                 // Import trực tiếp, không qua NgModule
    CommonModule,
    RouterLink,
    MatCardModule,
    MatButtonModule,
  ],
  template: `
    <mat-card>
      <mat-card-header>
        <mat-card-title>{{ user()?.displayName }}</mat-card-title>
        <mat-card-subtitle>{{ user()?.email }}</mat-card-subtitle>
      </mat-card-header>
      <mat-card-actions>
        <a mat-button [routerLink]="['/users', user()?.id, 'edit']">
          Chỉnh sửa
        </a>
      </mat-card-actions>
    </mat-card>
  `,
  styleUrl: './user-profile.component.scss',
})
export class UserProfileComponent {
  // Signal-based input (Angular 17+) — type-safe và reactive
  readonly userId = input.required<string>();

  private readonly userService = inject(UserService);
}
```

**Điểm quan trọng:** `imports` array trong standalone component đóng vai trò tương tự `imports` của NgModule — bạn khai báo những gì component cần dùng trong template. Angular sẽ tree-shake những gì không được dùng khi build production.

### Bootstrap ứng dụng standalone

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { appConfig } from './app/app.config';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, appConfig).catch(console.error);
```

```typescript
// app.config.ts — thay thế cho AppModule
import { ApplicationConfig, provideZoneChangeDetection } from '@angular/core';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { provideAnimationsAsync } from '@angular/platform-browser/animations/async';
import { routes } from './app.routes';
import { authInterceptor } from '@core/interceptors/auth.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    // Zone.js change detection — tối ưu event coalescing
    provideZoneChangeDetection({ eventCoalescing: true }),

    // Router với tính năng bind route params vào @Input
    provideRouter(routes, withComponentInputBinding()),

    // HttpClient với interceptors
    provideHttpClient(withInterceptors([authInterceptor])),

    // Angular Material animations (lazy load)
    provideAnimationsAsync(),
  ],
};
```

### Lazy Loading với Standalone Components

```typescript
// app.routes.ts
import { Routes } from '@angular/router';
import { authGuard } from '@core/guards/auth.guard';

export const routes: Routes = [
  {
    path: '',
    redirectTo: '/dashboard',
    pathMatch: 'full',
  },
  {
    path: 'auth',
    // Lazy load toàn bộ feature auth routes
    loadChildren: () =>
      import('./features/auth/auth.routes').then((m) => m.AUTH_ROUTES),
  },
  {
    path: 'dashboard',
    canActivate: [authGuard],
    // Lazy load single component
    loadComponent: () =>
      import('./features/dashboard/dashboard.component').then(
        (m) => m.DashboardComponent
      ),
  },
];
```

---

## 1.3 TypeScript trong Angular — Deeper than React

### Strict Mode là mặc định

Angular 18 bật TypeScript strict mode theo mặc định trong `tsconfig.json`. Điều này có nghĩa là:

- `strictNullChecks`: `null` và `undefined` phải được xử lý tường minh
- `noImplicitAny`: không được để TypeScript tự suy luận `any`
- `strictPropertyInitialization`: mọi property của class phải được khởi tạo

```typescript
// tsconfig.json — Angular 18 defaults
{
  "compilerOptions": {
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "paths": {
      "@core/*": ["src/app/core/*"],
      "@shared/*": ["src/app/shared/*"],
      "@features/*": ["src/app/features/*"],
      "@env/*": ["src/environments/*"]
    }
  }
}
```

**Lưu ý về `paths`:** Cấu hình path aliases giúp import rõ ràng hơn. Thay vì `import { UserService } from '../../../core/services/user.service'`, bạn viết `import { UserService } from '@core/services/user.service'`.

### Decorators — Metadata cho Angular Runtime

Decorators là cú pháp TypeScript cho phép Angular đọc metadata từ class lúc runtime. Đây là cơ chế Angular dùng để biết class nào là component, class nào là service, và cấu hình của chúng.

```typescript
// Component decorator — định nghĩa metadata của component
@Component({
  selector: 'app-user-card',     // CSS selector để dùng trong template
  standalone: true,
  imports: [MatCardModule],
  template: `<mat-card>...</mat-card>`,
  styleUrl: './user-card.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush,  // Tối ưu performance
})
export class UserCardComponent {
  // Input signal — Angular 17+ syntax
  readonly user = input.required<User>();

  // Output signal — Angular 17+ syntax
  readonly userSelected = output<User>();
}
```

```typescript
// Injectable decorator — đăng ký service với DI container
@Injectable({
  providedIn: 'root',  // Singleton, tạo một lần cho toàn app
})
export class UserService {
  private readonly http = inject(HttpClient);
  private readonly apiUrl = inject(API_URL);  // InjectionToken

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(`${this.apiUrl}/users`);
  }
}
```

### Models và Interfaces

Angular project chuyên nghiệp phân biệt rõ ba loại type:

```typescript
// models/user.model.ts

// 1. Interface — shape của data từ API
export interface User {
  readonly id: string;
  readonly email: string;
  readonly displayName: string;
  readonly role: UserRole;
  readonly createdAt: string;   // ISO 8601 string từ API
}

// 2. Enum — tập hợp giá trị có ý nghĩa
export enum UserRole {
  Admin = 'ADMIN',
  Editor = 'EDITOR',
  Viewer = 'VIEWER',
}

// 3. Type alias — union types, utility types
export type CreateUserDto = Omit<User, 'id' | 'createdAt'>;
export type UpdateUserDto = Partial<Pick<User, 'displayName' | 'role'>>;

// 4. Class — khi cần methods trên model
export class UserViewModel {
  constructor(private readonly user: User) {}

  get initials(): string {
    return this.user.displayName
      .split(' ')
      .map((word) => word[0])
      .join('')
      .toUpperCase()
      .slice(0, 2);
  }

  get isAdmin(): boolean {
    return this.user.role === UserRole.Admin;
  }
}
```

---

## 1.4 Template Syntax — Khác React JSX Hoàn Toàn

### Template là HTML — không phải JavaScript

JSX trong React là JavaScript trả về HTML-like syntax. Template của Angular là **HTML thật** với cú pháp đặc biệt được Angular xử lý lúc compile. Điều này có một hệ quả quan trọng: Angular compiler phân tích template và báo lỗi TypeScript ngay tại build time — nếu bạn dùng property không tồn tại trong template, build sẽ thất bại.

### Binding Syntax

Angular có bốn loại binding chính, mỗi loại có hướng data flow khác nhau:

```html
<!-- 1. Interpolation — hiển thị giá trị (Component → DOM) -->
<p>Xin chào, {{ user.displayName }}</p>
<p>{{ price | currency:'VND' }}</p>

<!-- 2. Property Binding — gán giá trị cho DOM property (Component → DOM) -->
<img [src]="user.avatarUrl" [alt]="user.displayName" />
<button [disabled]="isLoading()">Lưu</button>
<mat-progress-bar [value]="uploadProgress()"></mat-progress-bar>

<!-- 3. Event Binding — lắng nghe DOM event (DOM → Component) -->
<button (click)="onSave()">Lưu</button>
<input (input)="onSearchChange($event)" (keydown.enter)="onSearch()" />

<!-- 4. Two-way Binding — kết hợp property và event -->
<!-- [(ngModel)] = [ngModel] + (ngModelChange) -->
<input [(ngModel)]="searchQuery" />

<!-- Với Signals và Reactive Forms, two-way binding ít dùng hơn -->
```

**Phân biệt `[property]` và `property`:**

```html
<!-- Truyền string literal — KHÔNG dùng binding -->
<app-user-card role="admin"></app-user-card>

<!-- Truyền biến hoặc expression — PHẢI dùng binding -->
<app-user-card [role]="userRole"></app-user-card>
<app-user-card [role]="user.role === 'admin' ? 'admin' : 'viewer'"></app-user-card>

<!-- SAI — truyền string "userRole" thay vì giá trị của biến -->
<app-user-card role="userRole"></app-user-card>
```

### Control Flow mới (Angular 17+)

Angular 17 giới thiệu cú pháp control flow tích hợp trực tiếp vào template, thay thế `*ngIf`, `*ngFor`, `*ngSwitch`. Cú pháp mới rõ ràng hơn và hiệu năng tốt hơn.

```html
<!-- @if — thay *ngIf -->
@if (user(); as currentUser) {
  <app-user-card [user]="currentUser" />
} @else if (isLoading()) {
  <mat-spinner diameter="40" />
} @else {
  <p class="empty-state">Không tìm thấy người dùng</p>
}

<!-- @for — thay *ngFor -->
@for (user of users(); track user.id) {
  <app-user-card
    [user]="user"
    (userSelected)="onUserSelected($event)"
  />
} @empty {
  <p class="empty-state">Danh sách trống</p>
}

<!-- @switch — thay *ngSwitch -->
@switch (user().role) {
  @case (UserRole.Admin) {
    <app-admin-badge />
  }
  @case (UserRole.Editor) {
    <app-editor-badge />
  }
  @default {
    <app-viewer-badge />
  }
}
```

**Tại sao `track` quan trọng trong `@for`:** `track` tương tự `key` trong React — nó giúp Angular nhận diện element nào thay đổi thay vì re-render toàn bộ list. Luôn dùng `track` với một giá trị unique, thường là `id` của object.

### ng-template, ng-container, ng-content

Ba khái niệm này hay gây nhầm lẫn nhưng phục vụ mục đích khác nhau:

```html
<!-- ng-container — wrapper không tạo DOM element thật -->
<!-- Dùng khi cần áp nhiều directive mà không muốn thêm element -->
<ng-container *ngTemplateOutlet="headerTemplate; context: { $implicit: user() }">
</ng-container>

<!-- ng-template — định nghĩa template có thể tái sử dụng -->
<ng-template #loadingTemplate>
  <div class="loading-overlay">
    <mat-spinner />
  </div>
</ng-template>

<ng-template #errorTemplate let-error>
  <app-error-message [message]="error.message" />
</ng-template>

<!-- ng-content — Content Projection, tương tự children trong React -->
<!-- Trong component definition: -->
```

```typescript
// card.component.ts
@Component({
  selector: 'app-card',
  standalone: true,
  template: `
    <div class="card">
      <!-- Slot mặc định — nhận mọi content -->
      <ng-content />

      <!-- Named slot — nhận content theo selector -->
      <ng-content select="[card-header]" />
      <ng-content select="[card-footer]" />
    </div>
  `,
})
export class CardComponent {}
```

```html
<!-- Sử dụng component với content projection -->
<app-card>
  <h2 card-header>Tiêu đề</h2>
  <p>Nội dung chính của card</p>
  <button card-footer mat-button>Đóng</button>
</app-card>
```

---

## 1.5 Component Lifecycle

### Vòng đời đầy đủ

Angular component trải qua một chuỗi lifecycle hooks từ lúc khởi tạo đến khi bị hủy. Hiểu rõ thứ tự và mục đích của từng hook giúp tránh nhiều lỗi phổ biến.

```
new Component()        → constructor
                       → ngOnChanges (nếu có @Input)
                       → ngOnInit
                       → ngDoCheck
                       → ngAfterContentInit
                       → ngAfterContentChecked
                       → ngAfterViewInit
                       → ngAfterViewChecked
[Mỗi lần change]       → ngOnChanges → ngDoCheck → ngAfterContentChecked → ngAfterViewChecked
[Destroy]              → ngOnDestroy
```

### constructor vs ngOnInit

Đây là điểm khác biệt quan trọng so với React (không có constructor lifecycle):

```typescript
@Component({
  selector: 'app-user-detail',
  standalone: true,
  imports: [CommonModule, MatCardModule],
  templateUrl: './user-detail.component.html',
})
export class UserDetailComponent implements OnInit, OnDestroy {
  // inject() phải gọi trong injection context — constructor body hoặc field initializer
  private readonly userService = inject(UserService);
  private readonly route = inject(ActivatedRoute);
  private readonly destroy$ = new Subject<void>();

  user = signal<User | null>(null);
  isLoading = signal(false);
  error = signal<string | null>(null);

  constructor() {
    // constructor: CHỈ dùng để inject dependencies và khởi tạo giá trị đơn giản
    // KHÔNG gọi service, KHÔNG truy cập DOM, KHÔNG dùng @Input values
    // Lúc này @Input chưa có giá trị
  }

  ngOnInit(): void {
    // ngOnInit: @Input đã sẵn sàng, có thể gọi service
    // Đây là nơi khởi tạo data và subscriptions
    this.loadUser();
  }

  private loadUser(): void {
    const userId = this.route.snapshot.paramMap.get('id');
    if (!userId) return;

    this.isLoading.set(true);
    this.error.set(null);

    this.userService
      .getUserById(userId)
      .pipe(
        takeUntil(this.destroy$),
        finalize(() => this.isLoading.set(false))
      )
      .subscribe({
        next: (user) => this.user.set(user),
        error: (err) => this.error.set(err.message),
      });
  }

  ngOnDestroy(): void {
    // Cleanup mọi subscription để tránh memory leak
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**Pattern hiện đại hơn** với `takeUntilDestroyed`:

```typescript
@Component({
  selector: 'app-user-detail',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './user-detail.component.html',
})
export class UserDetailComponent implements OnInit {
  private readonly userService = inject(UserService);
  private readonly route = inject(ActivatedRoute);

  // takeUntilDestroyed tự động unsubscribe khi component bị destroy
  // Phải khởi tạo trong constructor context
  private readonly destroyRef = inject(DestroyRef);

  user = signal<User | null>(null);
  isLoading = signal(false);

  ngOnInit(): void {
    this.route.paramMap
      .pipe(
        map((params) => params.get('id')),
        filter((id): id is string => id !== null),
        switchMap((id) => {
          this.isLoading.set(true);
          return this.userService.getUserById(id);
        }),
        takeUntilDestroyed(this.destroyRef)  // Tự động cleanup
      )
      .subscribe({
        next: (user) => {
          this.user.set(user);
          this.isLoading.set(false);
        },
        error: () => this.isLoading.set(false),
      });
  }
}
```

### ngOnChanges — Phát hiện Input thay đổi

```typescript
@Component({
  selector: 'app-user-avatar',
  standalone: true,
  imports: [CommonModule],
  template: `
    <img
      [src]="avatarUrl()"
      [alt]="displayName()"
      class="avatar"
    />
  `,
})
export class UserAvatarComponent implements OnChanges {
  // @Input decorator — cách cổ điển, vẫn hợp lệ
  @Input({ required: true }) userId!: string;
  @Input() size: 'sm' | 'md' | 'lg' = 'md';

  private readonly avatarService = inject(AvatarService);

  avatarUrl = signal('');
  displayName = signal('');

  ngOnChanges(changes: SimpleChanges): void {
    // changes chứa giá trị cũ và mới của mỗi @Input thay đổi
    if (changes['userId'] && !changes['userId'].firstChange) {
      // userId đã thay đổi (không phải lần đầu), reload avatar
      this.loadAvatar(changes['userId'].currentValue);
    }

    if (changes['userId']?.firstChange) {
      // Lần đầu khởi tạo
      this.loadAvatar(this.userId);
    }
  }

  private loadAvatar(userId: string): void {
    this.avatarService.getAvatar(userId).subscribe((data) => {
      this.avatarUrl.set(data.url);
      this.displayName.set(data.displayName);
    });
  }
}
```

**Lưu ý:** Với Input Signals (Angular 17+), bạn không cần `ngOnChanges` để react với input changes — dùng `effect()` hoặc `computed()` thay thế. `ngOnChanges` vẫn cần thiết khi dùng `@Input` decorator truyền thống.

### ngAfterViewInit — Truy cập DOM

```typescript
@Component({
  selector: 'app-chart',
  standalone: true,
  template: `<canvas #chartCanvas></canvas>`,
})
export class ChartComponent implements AfterViewInit, OnDestroy {
  // ViewChild — lấy reference đến DOM element hoặc component con
  @ViewChild('chartCanvas') private canvasRef!: ElementRef<HTMLCanvasElement>;

  private chart: Chart | null = null;

  @Input({ required: true }) data!: ChartData;

  ngAfterViewInit(): void {
    // DOM đã render xong — an toàn để truy cập canvasRef
    this.initChart();
  }

  private initChart(): void {
    const ctx = this.canvasRef.nativeElement.getContext('2d');
    if (!ctx) return;

    this.chart = new Chart(ctx, {
      type: 'bar',
      data: this.data,
    });
  }

  ngOnDestroy(): void {
    // Cleanup chart instance để tránh memory leak
    this.chart?.destroy();
  }
}
```

---

## Tổng kết chương

Chương này đặt nền tảng cho toàn bộ hành trình học Angular. Những điểm cốt lõi cần ghi nhớ:

1. **Angular là full framework** — không cần tự lắp ghép ecosystem như React. Sự "opinionated" này là lợi thế ở scale lớn.

2. **Standalone Components** là mặc định của Angular 18 — không cần NgModule. `imports` array trong component thay thế hoàn toàn.

3. **TypeScript Strict Mode** là mặc định và bắt buộc — Angular compiler kiểm tra type ngay trong template, giúp phát hiện lỗi sớm.

4. **Template syntax là HTML** — không phải JSX. Control flow mới (`@if`, `@for`, `@switch`) rõ ràng và hiệu quả hơn structural directives cũ.

5. **Lifecycle hooks** có thứ tự và mục đích rõ ràng — `constructor` chỉ để inject, `ngOnInit` để khởi tạo data, `ngOnDestroy` để cleanup.

Chương tiếp theo sẽ đi sâu vào **Dependency Injection** — hệ thống phức tạp và mạnh mẽ nhất của Angular, là nền tảng để hiểu cách Angular quản lý toàn bộ application state và service lifecycle.
