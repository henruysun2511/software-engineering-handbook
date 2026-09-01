# Chương 9: Architecture & Patterns Thực Tế

## Giới thiệu chương

Các chương trước đã cung cấp đầy đủ công cụ kỹ thuật của Angular. Chương này trả lời câu hỏi quan trọng hơn: **làm thế nào để tổ chức tất cả những thứ đó thành một codebase có thể maintain ở quy mô lớn?**

Kiến trúc tốt không hiện diện trong code khi mọi thứ đang nhỏ — nó chỉ thể hiện giá trị khi team tăng lên, codebase phình to, và developer mới cần đọc hiểu và mở rộng tính năng mà không phá vỡ những gì đang hoạt động.

---

## 9.1 Feature-based Architecture

### Tổ chức theo feature, không theo type

Cách tổ chức phổ biến nhưng sai của Angular developer mới:

```
// ❌ Type-based — khó scale
src/app/
├── components/
│   ├── user-list.component.ts
│   ├── user-detail.component.ts
│   ├── product-list.component.ts
│   └── dashboard.component.ts
├── services/
│   ├── user.service.ts
│   └── product.service.ts
├── models/
│   ├── user.model.ts
│   └── product.model.ts
└── pipes/
    └── currency.pipe.ts
```

Vấn đề: khi thêm tính năng user management, phải chạm vào 4 thư mục khác nhau. Khi xóa feature, phải hunt qua toàn bộ codebase.

Cách đúng:

```
// ✓ Feature-based — mỗi feature là một đơn vị độc lập
src/app/
├── core/                          # Singleton services, guards, interceptors
│   ├── guards/
│   │   ├── auth.guard.ts
│   │   └── role.guard.ts
│   ├── interceptors/
│   │   ├── auth.interceptor.ts
│   │   └── error.interceptor.ts
│   └── services/
│       ├── auth.service.ts
│       ├── api.service.ts
│       └── notification.service.ts
│
├── shared/                        # Components/pipes/directives dùng chung
│   ├── components/
│   │   ├── confirm-dialog/
│   │   ├── form-error/
│   │   └── loading-overlay/
│   ├── directives/
│   │   └── click-outside.directive.ts
│   ├── pipes/
│   │   ├── time-ago.pipe.ts
│   │   └── file-size.pipe.ts
│   └── validators/
│       ├── custom.validators.ts
│       └── async.validators.ts
│
├── features/                      # Mỗi feature là một thư mục riêng
│   ├── auth/
│   │   ├── login/
│   │   │   ├── login.component.ts
│   │   │   ├── login.component.html
│   │   │   └── login.component.scss
│   │   ├── register/
│   │   └── auth.routes.ts
│   │
│   ├── user/
│   │   ├── models/
│   │   │   ├── user.model.ts
│   │   │   └── user.schema.ts
│   │   ├── services/
│   │   │   └── user.service.ts
│   │   ├── store/
│   │   │   └── user.store.ts
│   │   ├── user-list/
│   │   ├── user-detail/
│   │   ├── user-form/
│   │   └── user.routes.ts
│   │
│   └── dashboard/
│       ├── components/
│       │   ├── stats-card/
│       │   └── recent-activity/
│       ├── dashboard.component.ts
│       └── dashboard.routes.ts
│
├── app.component.ts
├── app.config.ts
└── app.routes.ts
```

### Barrel Files — Public API của Feature

Mỗi feature expose public API qua `index.ts` — bên ngoài chỉ import từ đây, không import sâu vào bên trong:

```typescript
// features/user/index.ts — public API của user feature
export { UserListComponent } from './user-list/user-list.component';
export { UserDetailComponent } from './user-detail/user-detail.component';
export { UserFormComponent } from './user-form/user-form.component';
export { UserService } from './services/user.service';
export { UserStore } from './store/user.store';
export type { User, CreateUserDto, UpdateUserDto } from './models/user.model';
export { USER_ROUTES } from './user.routes';

// Không export: internal components, helpers, private services
```

```typescript
// Sử dụng — import từ feature barrel, không import trực tiếp
import { UserService, UserStore, type User } from '@features/user';

// ❌ Sai — import trực tiếp vào implementation detail
import { UserService } from '@features/user/services/user.service';
```

---

## 9.2 Smart vs Dumb Component Pattern

### Smart Component (Container)

Smart component biết về business logic, inject services và store, xử lý side effects:

```typescript
// features/user/user-list/user-list.component.ts — SMART
@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [UserTableComponent, UserFilterComponent, MatButtonModule],
  template: `
    <!-- Chỉ delegate xuống Dumb components -->
    <app-user-filter
      [filter]="store.filter()"
      (filterChange)="store.setFilter($event)"
    />

    <app-user-table
      [users]="store.filteredUsers()"
      [isLoading]="store.isLoading()"
      [selectedUserId]="store.selectedUserId()"
      (userSelected)="store.selectUser($event)"
      (userEdit)="onEditUser($event)"
      (userDelete)="onDeleteUser($event)"
    />
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserListComponent implements OnInit {
  // Smart — biết về store và services
  protected readonly store = inject(UserStore);
  private readonly router = inject(Router);
  private readonly dialogService = inject(DialogService);

  ngOnInit(): void {
    this.store.loadUsers();
  }

  protected onEditUser(user: User): void {
    this.router.navigate(['/users', user.id, 'edit']);
  }

  protected onDeleteUser(user: User): void {
    this.dialogService
      .confirmDelete(user.displayName)
      .pipe(filter(Boolean))
      .subscribe(() => this.store.deleteUser(user.id));
  }
}
```

### Dumb Component (Presentational)

Dumb component chỉ nhận data qua `@Input` và phát event qua `@Output` — không biết về services hay store:

```typescript
// features/user/user-list/user-table/user-table.component.ts — DUMB
@Component({
  selector: 'app-user-table',
  standalone: true,
  imports: [MatTableModule, MatButtonModule, MatIconModule, MatProgressBarModule],
  template: `
    @if (isLoading()) {
      <mat-progress-bar mode="indeterminate" />
    }

    <mat-table [dataSource]="users()">
      <ng-container matColumnDef="name">
        <mat-header-cell *matHeaderCellDef>Tên</mat-header-cell>
        <mat-cell *matCellDef="let user">{{ user.displayName }}</mat-cell>
      </ng-container>

      <ng-container matColumnDef="actions">
        <mat-header-cell *matHeaderCellDef />
        <mat-cell *matCellDef="let user">
          <button mat-icon-button (click)="userEdit.emit(user)">
            <mat-icon>edit</mat-icon>
          </button>
          <button mat-icon-button color="warn" (click)="userDelete.emit(user)">
            <mat-icon>delete</mat-icon>
          </button>
        </mat-cell>
      </ng-container>

      <mat-header-row *matHeaderRowDef="displayedColumns" />
      <mat-row
        *matRowDef="let user; columns: displayedColumns"
        [class.selected]="user.id === selectedUserId()"
        (click)="userSelected.emit(user.id)"
      />
    </mat-table>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserTableComponent {
  // Chỉ có @Input và @Output — không inject services
  readonly users = input.required<User[]>();
  readonly isLoading = input(false);
  readonly selectedUserId = input<string | null>(null);

  readonly userSelected = output<string>();
  readonly userEdit = output<User>();
  readonly userDelete = output<User>();

  protected readonly displayedColumns = ['name', 'email', 'role', 'actions'];
}
```

**Lợi ích của pattern này:**
- Dumb components dễ test — không cần mock services
- Dumb components tái sử dụng ở nhiều nơi
- Smart components rõ ràng về data flow
- Dễ debug — biết ngay lỗi ở tầng nào

---

## 9.3 Directives Thực Tế

### Attribute Directives — Thêm Behavior

```typescript
// shared/directives/auto-focus.directive.ts
@Directive({
  selector: '[appAutoFocus]',
  standalone: true,
})
export class AutoFocusDirective implements AfterViewInit {
  private readonly el = inject(ElementRef<HTMLElement>);
  readonly delay = input(0, { alias: 'appAutoFocus' });

  ngAfterViewInit(): void {
    setTimeout(() => {
      this.el.nativeElement.focus();
    }, this.delay());
  }
}
```

```typescript
// shared/directives/click-outside.directive.ts
@Directive({
  selector: '[appClickOutside]',
  standalone: true,
})
export class ClickOutsideDirective implements OnInit, OnDestroy {
  private readonly el = inject(ElementRef<HTMLElement>);
  readonly clickOutside = output<MouseEvent>();
  private readonly destroyRef = inject(DestroyRef);

  ngOnInit(): void {
    fromEvent<MouseEvent>(document, 'click')
      .pipe(
        filter((event) => !this.el.nativeElement.contains(event.target as Node)),
        takeUntilDestroyed(this.destroyRef)
      )
      .subscribe((event) => this.clickOutside.emit(event));
  }
}
```

```typescript
// shared/directives/infinite-scroll.directive.ts
@Directive({
  selector: '[appInfiniteScroll]',
  standalone: true,
})
export class InfiniteScrollDirective implements OnInit, OnDestroy {
  private readonly el = inject(ElementRef<HTMLElement>);
  readonly threshold = input(200); // px trước khi đến cuối
  readonly scrolled = output<void>();
  private readonly destroyRef = inject(DestroyRef);

  ngOnInit(): void {
    fromEvent(this.el.nativeElement, 'scroll')
      .pipe(
        throttleTime(100),
        filter(() => this.isNearBottom()),
        distinctUntilChanged(),
        takeUntilDestroyed(this.destroyRef)
      )
      .subscribe(() => this.scrolled.emit());
  }

  private isNearBottom(): boolean {
    const el = this.el.nativeElement;
    const distanceFromBottom = el.scrollHeight - el.scrollTop - el.clientHeight;
    return distanceFromBottom <= this.threshold();
  }
}
```

```typescript
// shared/directives/loading-button.directive.ts
@Directive({
  selector: 'button[appLoadingButton]',
  standalone: true,
  host: {
    '[disabled]': 'isLoading()',
    '[class.loading]': 'isLoading()',
  },
})
export class LoadingButtonDirective {
  private readonly el = inject(ElementRef<HTMLButtonElement>);
  readonly isLoading = input(false, { alias: 'appLoadingButton' });
  private originalContent = '';

  constructor() {
    effect(() => {
      const loading = this.isLoading();
      const button = this.el.nativeElement;

      if (loading) {
        this.originalContent = button.innerHTML;
        button.innerHTML = `
          <mat-spinner diameter="20" style="display:inline-block"></mat-spinner>
          Đang xử lý...
        `;
      } else {
        button.innerHTML = this.originalContent;
      }
    });
  }
}
```

### Structural Directives

```typescript
// shared/directives/has-role.directive.ts
// Tương tự *ngIf nhưng dựa trên role của user
@Directive({
  selector: '[appHasRole]',
  standalone: true,
})
export class HasRoleDirective implements OnInit {
  private readonly template = inject(TemplateRef<unknown>);
  private readonly viewContainer = inject(ViewContainerRef);
  private readonly authService = inject(AuthService);

  readonly appHasRole = input<UserRole | UserRole[]>([]);

  ngOnInit(): void {
    effect(() => {
      const requiredRoles = Array.isArray(this.appHasRole())
        ? this.appHasRole() as UserRole[]
        : [this.appHasRole() as UserRole];

      const user = this.authService.currentUser();
      const hasAccess = user && requiredRoles.includes(user.role);

      this.viewContainer.clear();
      if (hasAccess) {
        this.viewContainer.createEmbeddedView(this.template);
      }
    });
  }
}
```

```html
<!-- Sử dụng trong template -->
<button
  *appHasRole="'ADMIN'"
  mat-button
  (click)="onDelete()"
>
  Xóa
</button>

<!-- Nhiều roles -->
<div *appHasRole="['ADMIN', 'EDITOR']">
  <app-edit-toolbar />
</div>
```

---

## 9.4 Pipes Thực Tế

### Pure Pipes — Hiệu năng cao

Pure pipe chỉ chạy lại khi input thay đổi (reference change) — Angular cache kết quả:

```typescript
// shared/pipes/time-ago.pipe.ts
@Pipe({
  name: 'timeAgo',
  standalone: true,
  pure: true,  // Mặc định là true
})
export class TimeAgoPipe implements PipeTransform {
  transform(value: string | Date | null): string {
    if (!value) return '';

    const date = value instanceof Date ? value : new Date(value);
    const now = new Date();
    const diffMs = now.getTime() - date.getTime();

    const seconds = Math.floor(diffMs / 1000);
    const minutes = Math.floor(seconds / 60);
    const hours = Math.floor(minutes / 60);
    const days = Math.floor(hours / 24);
    const months = Math.floor(days / 30);
    const years = Math.floor(months / 12);

    if (seconds < 60) return 'Vừa xong';
    if (minutes < 60) return `${minutes} phút trước`;
    if (hours < 24) return `${hours} giờ trước`;
    if (days < 30) return `${days} ngày trước`;
    if (months < 12) return `${months} tháng trước`;
    return `${years} năm trước`;
  }
}
```

```typescript
// shared/pipes/file-size.pipe.ts
@Pipe({
  name: 'fileSize',
  standalone: true,
})
export class FileSizePipe implements PipeTransform {
  transform(bytes: number, precision = 2): string {
    if (bytes === 0) return '0 Bytes';

    const units = ['Bytes', 'KB', 'MB', 'GB', 'TB'];
    const i = Math.floor(Math.log(bytes) / Math.log(1024));
    const value = bytes / Math.pow(1024, i);

    return `${value.toFixed(precision)} ${units[i]}`;
  }
}
```

```typescript
// shared/pipes/highlight.pipe.ts — highlight search term
@Pipe({
  name: 'highlight',
  standalone: true,
})
export class HighlightPipe implements PipeTransform {
  transform(text: string, search: string): string {
    if (!search?.trim()) return text;

    const escaped = search.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
    const regex = new RegExp(`(${escaped})`, 'gi');

    return text.replace(
      regex,
      '<mark class="highlight">$1</mark>'
    );
  }
}
```

```typescript
// shared/pipes/vnd-currency.pipe.ts
@Pipe({
  name: 'vnd',
  standalone: true,
})
export class VndCurrencyPipe implements PipeTransform {
  private readonly formatter = new Intl.NumberFormat('vi-VN', {
    style: 'currency',
    currency: 'VND',
    maximumFractionDigits: 0,
  });

  transform(value: number | null): string {
    if (value === null || value === undefined) return '';
    return this.formatter.format(value);
  }
}
```

---

## 9.5 Performance Patterns

### OnPush + trackBy — Bắt buộc trong Production

```typescript
// Mọi component đều dùng OnPush
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <!-- trackBy bắt buộc với @for -->
    @for (user of users(); track user.id) {
      <app-user-card [user]="user" />
    }

    <!-- Với danh sách primitive -->
    @for (tag of tags(); track tag) {
      <span class="tag">{{ tag }}</span>
    }
  `,
})
export class UserListComponent {
  readonly users = input.required<User[]>();
  readonly tags = input<string[]>([]);
}
```

### Defer Block — Lazy Load Template

Angular 17 giới thiệu `@defer` — lazy load một phần template chỉ khi cần:

```html
<!-- Chỉ load HeavyChartComponent khi user scroll đến -->
@defer (on viewport) {
  <app-heavy-chart [data]="chartData()" />
} @loading (minimum 300ms) {
  <app-chart-skeleton />
} @placeholder {
  <div class="chart-placeholder">
    <mat-icon>bar_chart</mat-icon>
    <span>Biểu đồ sẽ hiển thị khi scroll đến</span>
  </div>
} @error {
  <p>Không thể tải biểu đồ</p>
}

<!-- Load khi user click vào tab -->
@defer (on interaction(statsTab)) {
  <app-stats-dashboard />
} @placeholder {
  <div #statsTab class="tab-trigger">Thống kê chi tiết</div>
}

<!-- Load sau khi idle — không block UI chính -->
@defer (on idle) {
  <app-recommendations-panel />
}

<!-- Load khi điều kiện thỏa mãn -->
@defer (when isUserLoggedIn()) {
  <app-personalized-content />
}
```

### Memoization trong Component

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <!-- computed() tự động memoize -->
    <p>Tổng: {{ total() | vnd }}</p>
    <p>Thuế: {{ tax() | vnd }}</p>
    <p>Thanh toán: {{ finalAmount() | vnd }}</p>
  `,
})
export class OrderSummaryComponent {
  readonly items = input.required<OrderItem[]>();
  readonly taxRate = input(0.1);

  // computed chỉ recalculate khi dependency thay đổi
  protected readonly subtotal = computed(() =>
    this.items().reduce((sum, item) => sum + item.price * item.quantity, 0)
  );

  protected readonly tax = computed(
    () => this.subtotal() * this.taxRate()
  );

  protected readonly total = computed(
    () => this.subtotal() + this.tax()
  );

  // Nếu cần memoize một hàm thuần
  protected readonly processedItems = computed(() =>
    this.items()
      .filter((item) => item.quantity > 0)
      .sort((a, b) => b.price * b.quantity - a.price * a.quantity)
  );
}
```

### Global Error Handler

```typescript
// core/handlers/global-error.handler.ts
@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  private readonly notificationService = inject(NotificationService);
  private readonly loggerService = inject(LoggerService);

  handleError(error: unknown): void {
    // Không log duplicate errors
    const message = this.extractMessage(error);

    // Log cho monitoring (Sentry, DataDog...)
    this.loggerService.error(message, { error });

    // Hiển thị user-friendly message
    if (!this.isKnownError(error)) {
      this.notificationService.error('Đã xảy ra lỗi không mong muốn');
    }

    // Trong development: re-throw để thấy stack trace
    if (!environment.production) {
      console.error('[GlobalErrorHandler]', error);
    }
  }

  private extractMessage(error: unknown): string {
    if (error instanceof Error) return error.message;
    if (typeof error === 'string') return error;
    return 'Unknown error';
  }

  private isKnownError(error: unknown): boolean {
    // HttpErrorResponse được xử lý bởi error interceptor
    return error instanceof HttpErrorResponse;
  }
}

// Đăng ký trong app.config.ts
{
  provide: ErrorHandler,
  useClass: GlobalErrorHandler,
}
```

---

## 9.6 Logger Service

```typescript
// core/services/logger.service.ts
export type LogLevel = 'debug' | 'info' | 'warn' | 'error';

export interface LogEntry {
  level: LogLevel;
  message: string;
  timestamp: string;
  context?: Record<string, unknown>;
}

@Injectable({ providedIn: 'root' })
export class LoggerService {
  private readonly isProd = inject(ENVIRONMENT).production;

  debug(message: string, context?: Record<string, unknown>): void {
    if (!this.isProd) {
      this.log('debug', message, context);
    }
  }

  info(message: string, context?: Record<string, unknown>): void {
    this.log('info', message, context);
  }

  warn(message: string, context?: Record<string, unknown>): void {
    this.log('warn', message, context);
  }

  error(message: string, context?: Record<string, unknown>): void {
    this.log('error', message, context);
    // Gửi lên error monitoring service
    this.reportToMonitoring({ level: 'error', message, context });
  }

  private log(
    level: LogLevel,
    message: string,
    context?: Record<string, unknown>
  ): void {
    const entry: LogEntry = {
      level,
      message,
      timestamp: new Date().toISOString(),
      context,
    };

    const logFn = {
      debug: console.debug,
      info: console.info,
      warn: console.warn,
      error: console.error,
    }[level];

    if (context) {
      logFn(`[${entry.timestamp}] [${level.toUpperCase()}] ${message}`, context);
    } else {
      logFn(`[${entry.timestamp}] [${level.toUpperCase()}] ${message}`);
    }
  }

  private reportToMonitoring(entry: Partial<LogEntry>): void {
    // Tích hợp Sentry, DataDog, hoặc custom monitoring
    // Sentry.captureMessage(entry.message, { level: 'error', extra: entry.context });
  }
}
```

---

## 9.7 Tổ chức Shared Module Đúng Cách

```typescript
// shared/components/loading-overlay/loading-overlay.component.ts
@Component({
  selector: 'app-loading-overlay',
  standalone: true,
  imports: [MatProgressSpinnerModule, CommonModule],
  template: `
    @if (isLoading()) {
      <div class="overlay" [class.transparent]="transparent()">
        <mat-spinner [diameter]="diameter()" />
        @if (message()) {
          <p class="message">{{ message() }}</p>
        }
      </div>
    }
  `,
  styles: [`
    .overlay {
      position: absolute;
      inset: 0;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      background: rgba(255, 255, 255, 0.8);
      z-index: 100;

      &.transparent {
        background: transparent;
      }
    }

    .message {
      margin-top: 16px;
      color: var(--mat-app-on-surface-variant);
      font-size: 14px;
    }
  `],
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class LoadingOverlayComponent {
  readonly isLoading = input(false);
  readonly message = input<string | null>(null);
  readonly transparent = input(false);
  readonly diameter = input(48);
}
```

```typescript
// shared/components/empty-state/empty-state.component.ts
@Component({
  selector: 'app-empty-state',
  standalone: true,
  imports: [MatIconModule, MatButtonModule],
  template: `
    <div class="empty-state">
      <mat-icon class="icon">{{ icon() }}</mat-icon>
      <h3 class="title">{{ title() }}</h3>
      @if (description()) {
        <p class="description">{{ description() }}</p>
      }
      @if (actionLabel() && actionClick.observed) {
        <button mat-flat-button color="primary" (click)="actionClick.emit()">
          {{ actionLabel() }}
        </button>
      }
    </div>
  `,
  styles: [`
    .empty-state {
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      padding: 48px 24px;
      text-align: center;
      gap: 16px;
    }

    .icon {
      font-size: 64px;
      width: 64px;
      height: 64px;
      color: var(--mat-app-on-surface-variant);
      opacity: 0.5;
    }

    .title {
      font-size: 18px;
      font-weight: 500;
      color: var(--mat-app-on-surface);
    }

    .description {
      font-size: 14px;
      color: var(--mat-app-on-surface-variant);
      max-width: 400px;
    }
  `],
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class EmptyStateComponent {
  readonly icon = input('inbox');
  readonly title = input('Không có dữ liệu');
  readonly description = input<string | null>(null);
  readonly actionLabel = input<string | null>(null);
  readonly actionClick = output<void>();
}
```

---

## 9.8 Environment Configuration

```typescript
// environments/environment.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000/api',
  wsUrl: 'ws://localhost:3000',
  timeout: 30000,
  features: {
    newDashboard: true,
    advancedReports: false,
  },
} as const;

// environments/environment.production.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.example.com/api',
  wsUrl: 'wss://api.example.com',
  timeout: 30000,
  features: {
    newDashboard: true,
    advancedReports: true,
  },
} as const;

export type Environment = typeof environment;
```

```typescript
// core/tokens/environment.token.ts
import { InjectionToken } from '@angular/core';
import { Environment } from '@env/environment';

export const ENVIRONMENT = new InjectionToken<Environment>('ENVIRONMENT');

// app.config.ts
import { environment } from '@env/environment';

export const appConfig: ApplicationConfig = {
  providers: [
    { provide: ENVIRONMENT, useValue: environment },
    {
      provide: API_URL,
      useFactory: () => inject(ENVIRONMENT).apiUrl,
    },
  ],
};
```

---

## Tổng kết chương

Architecture tốt là nền tảng để ứng dụng có thể phát triển theo thời gian. Những điểm cốt lõi:

1. **Feature-based architecture** — mỗi feature là một thư mục độc lập với models, services, store, và components riêng. Barrel files (`index.ts`) định nghĩa public API của feature.

2. **Smart/Dumb component pattern** — Smart component xử lý business logic và data fetching; Dumb component chỉ nhận Input và phát Output. Dumb components dễ test và tái sử dụng hơn.

3. **Directives** cho behavior tái sử dụng (click-outside, auto-focus, infinite-scroll, role-based rendering). Structural directive cho conditional rendering phức tạp hơn `@if`.

4. **Pure pipes** được memoize tự động — ưu tiên pipe hơn method trong template để tránh re-execute không cần thiết.

5. **`@defer`** là công cụ performance mạnh nhất của Angular 17 — lazy load template theo viewport, interaction, idle, hoặc điều kiện tùy chỉnh.

6. **Global Error Handler và Logger Service** là hai thành phần bắt buộc trong production — đảm bảo lỗi được bắt, log, và report một cách nhất quán.

Chương tiếp theo sẽ đi vào **Testing** — cách viết unit test và e2e test cho Angular application, bao gồm Jest, TestBed, và Playwright.
