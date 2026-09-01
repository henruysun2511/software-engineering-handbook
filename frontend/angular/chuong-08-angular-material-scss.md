# Chương 8: Angular Material & SCSS

## Giới thiệu chương

Angular Material là thư viện UI component chính thức của Angular, được duy trì bởi Google team. Nó triển khai Material Design 3 (M3) — hệ thống design system của Google — với đầy đủ accessibility, theming, animation, và CDK (Component Dev Kit) cho phép xây dựng thêm component tùy chỉnh.

Chương này tập trung vào ba khía cạnh quan trọng nhất khi làm việc với Angular Material trong production: **theming system** (cách customize giao diện đúng cách), **SCSS architecture** (tổ chức styles có thể maintain), và **component patterns** thực tế hay dùng nhất.

---

## 8.1 Cài đặt và Cấu hình

### Cài đặt Angular Material

```bash
ng add @angular/material
```

Angular CLI sẽ hỏi một số câu:
- Chọn theme: `Custom` (để tự custom sau)
- Set up typography: `Yes`
- Include animations: `Yes` (Included)

Lệnh này tự động:
- Cài package `@angular/material` và `@angular/cdk`
- Thêm `provideAnimationsAsync()` vào `app.config.ts`
- Thêm Material font và icon vào `index.html`
- Tạo custom theme trong `styles.scss`

### Cấu hình trong app.config.ts

```typescript
// app.config.ts
import { provideAnimationsAsync } from '@angular/platform-browser/animations/async';
import { MAT_FORM_FIELD_DEFAULT_OPTIONS } from '@angular/material/form-field';
import { MAT_DATE_LOCALE } from '@angular/material/core';
import { MAT_SNACK_BAR_DEFAULT_OPTIONS } from '@angular/material/snack-bar';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(),

    // Lazy load animations — tốt hơn cho bundle size
    provideAnimationsAsync(),

    // Global defaults cho Material components
    {
      provide: MAT_FORM_FIELD_DEFAULT_OPTIONS,
      useValue: {
        appearance: 'outline',     // 'outline' | 'fill'
        subscriptSizing: 'dynamic', // Không reserve space cho error message
        floatLabel: 'always',
      },
    },
    {
      provide: MAT_DATE_LOCALE,
      useValue: 'vi-VN',           // Vietnamese locale cho DatePicker
    },
    {
      provide: MAT_SNACK_BAR_DEFAULT_OPTIONS,
      useValue: {
        duration: 4000,
        horizontalPosition: 'right',
        verticalPosition: 'top',
      },
    },
  ],
};
```

---

## 8.2 Theming System — Material Design 3

### Hiểu Color System của M3

Material Design 3 dùng một hệ thống màu sắc khác M2. Thay vì primary/accent/warn, M3 có:

- **Primary**: Màu chính của brand
- **Secondary**: Màu bổ trợ
- **Tertiary**: Màu nhấn thứ ba
- **Error**: Màu lỗi
- **Surface**: Các sắc thái nền

Từ mỗi màu, M3 tự tạo ra các token như `on-primary`, `primary-container`, `on-primary-container`… cho light và dark mode.

### Tạo Custom Theme

```scss
// styles/_theme.scss

// Import Angular Material theming system
@use '@angular/material' as mat;

// Bước 1: Định nghĩa custom color palette
// Dùng Material Theme Builder: https://material-foundation.github.io/material-theme-builder/
// Hoặc tự chọn màu:

$primary-palette: (
  0: #000000,
  10: #001e2e,
  20: #003450,
  25: #003f61,
  30: #004a73,
  35: #005686,
  40: #006399,
  50: #1a7eb8,
  60: #3f99d4,
  70: #62b4f0,
  80: #93ceff,
  90: #cce5ff,
  95: #e7f2ff,
  98: #f6f9ff,
  99: #fbfcff,
  100: #ffffff,
);

// Bước 2: Tạo theme với define-theme
$light-theme: mat.define-theme((
  color: (
    theme-type: light,
    primary: mat.$azure-palette,       // Built-in palette
    tertiary: mat.$blue-palette,
    use-system-variables: true,        // M3 CSS custom properties
  ),
  typography: (
    brand-family: 'Inter',
    plain-family: 'Inter',
  ),
  density: (
    scale: 0,  // -3 (compact) đến 0 (comfortable)
  ),
));

$dark-theme: mat.define-theme((
  color: (
    theme-type: dark,
    primary: mat.$azure-palette,
    tertiary: mat.$blue-palette,
    use-system-variables: true,
  ),
  typography: (
    brand-family: 'Inter',
    plain-family: 'Inter',
  ),
));

// Expose themes cho global styles
:root {
  @include mat.theme($light-theme);
  @include mat.typography-hierarchy($light-theme);
}

// Dark mode
@media (prefers-color-scheme: dark) {
  :root {
    @include mat.theme($dark-theme);
  }
}

// Manual dark mode class
.dark-theme {
  @include mat.theme($dark-theme);
}
```

```scss
// styles.scss — entry point
@use 'styles/theme' as *;
@use '@angular/material' as mat;

// Core styles (cần thiết để Material hoạt động)
@include mat.core();

// Chỉ include component themes cần dùng — tree-shakable
// Thay vì all-component-themes (load hết)
@include mat.button-theme($light-theme);
@include mat.form-field-theme($light-theme);
@include mat.input-theme($light-theme);
@include mat.select-theme($light-theme);
@include mat.table-theme($light-theme);
@include mat.card-theme($light-theme);
@include mat.toolbar-theme($light-theme);
@include mat.dialog-theme($light-theme);
@include mat.snack-bar-theme($light-theme);
@include mat.progress-spinner-theme($light-theme);
@include mat.paginator-theme($light-theme);

// Global styles
*,
*::before,
*::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  font-family: 'Inter', sans-serif;
  background-color: var(--mat-app-background-color);
  color: var(--mat-app-text-color);
}

// CSS custom properties cho app-wide usage
:root {
  --spacing-xs: 4px;
  --spacing-sm: 8px;
  --spacing-md: 16px;
  --spacing-lg: 24px;
  --spacing-xl: 32px;
  --spacing-2xl: 48px;

  --border-radius-sm: 4px;
  --border-radius-md: 8px;
  --border-radius-lg: 16px;
  --border-radius-full: 9999px;

  --shadow-sm: 0 1px 3px rgba(0, 0, 0, 0.12);
  --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.1);
  --shadow-lg: 0 10px 15px rgba(0, 0, 0, 0.1);

  --transition-fast: 150ms ease;
  --transition-normal: 250ms ease;
  --transition-slow: 350ms ease;
}
```

---

## 8.3 SCSS Architecture trong Angular

### ViewEncapsulation — Cơ chế Scoping Styles

Angular có ba chế độ style encapsulation:

```typescript
// 1. Emulated (mặc định) — Angular thêm attribute selector vào CSS
// Ví dụ: .button[_ngcontent-xxx] — styles chỉ áp dụng cho component này
@Component({
  encapsulation: ViewEncapsulation.Emulated, // default
})

// 2. None — styles là global, có thể leak ra ngoài
@Component({
  encapsulation: ViewEncapsulation.None,
})

// 3. ShadowDom — native Shadow DOM, isolation thật sự
@Component({
  encapsulation: ViewEncapsulation.ShadowDom,
})
```

**Khi nào dùng từng loại:**
- `Emulated`: Hầu hết components — styles scoped, an toàn
- `None`: Khi cần override styles của Material components hoặc third-party
- `ShadowDom`: Hiếm dùng — chỉ khi cần isolation thật sự và biết trade-offs

### `:host` và `::ng-deep`

```scss
// user-card.component.scss

// :host — style element gốc của component (<app-user-card>)
:host {
  display: block;
  height: 100%;

  // :host() — style khi component có class/attribute cụ thể
  &.compact {
    .card-body {
      padding: var(--spacing-sm);
    }
  }

  // :host-context() — style dựa vào context bên ngoài
  :host-context(.dark-sidebar) {
    background: var(--mat-app-surface-variant);
  }
}

// ::ng-deep — pierce encapsulation để style component con hoặc Material internals
// Dùng hạn chế — chỉ khi không có cách nào khác
// Luôn kết hợp với :host để giới hạn scope
:host ::ng-deep {
  .mat-mdc-card-header {
    padding: var(--spacing-md) var(--spacing-md) 0;
  }

  // Override Material form field appearance
  .mat-mdc-form-field {
    width: 100%;
  }
}

// Component-specific styles
.user-card {
  &__header {
    display: flex;
    align-items: center;
    gap: var(--spacing-md);
  }

  &__avatar {
    width: 48px;
    height: 48px;
    border-radius: var(--border-radius-full);
    object-fit: cover;
  }

  &__info {
    flex: 1;
    min-width: 0; // Prevent overflow
  }

  &__name {
    font-weight: 600;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  &__role {
    font-size: 12px;
    color: var(--mat-app-on-surface-variant);
    text-transform: uppercase;
    letter-spacing: 0.5px;
  }
}
```

### SCSS Partials và Architecture

```
src/styles/
├── _variables.scss       # CSS custom properties và SCSS variables
├── _mixins.scss          # SCSS mixins tái sử dụng
├── _breakpoints.scss     # Responsive breakpoints
├── _typography.scss      # Text styles
├── _utilities.scss       # Utility classes
├── _animations.scss      # Shared keyframes
├── _theme.scss           # Angular Material theme
└── styles.scss           # Entry point — import tất cả
```

```scss
// styles/_mixins.scss
@use 'sass:math';

// Flexbox shortcuts
@mixin flex-center {
  display: flex;
  align-items: center;
  justify-content: center;
}

@mixin flex-between {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

// Truncate text
@mixin truncate($lines: 1) {
  @if $lines == 1 {
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  } @else {
    display: -webkit-box;
    -webkit-line-clamp: $lines;
    -webkit-box-orient: vertical;
    overflow: hidden;
  }
}

// Responsive breakpoints
$breakpoints: (
  'xs': 0,
  'sm': 600px,
  'md': 960px,
  'lg': 1280px,
  'xl': 1920px,
);

@mixin respond-to($breakpoint) {
  $value: map-get($breakpoints, $breakpoint);
  @if $value != null {
    @media (min-width: $value) {
      @content;
    }
  } @else {
    @warn "Unknown breakpoint: #{$breakpoint}";
  }
}

// Material elevation
@mixin elevation($level) {
  @if $level == 0 {
    box-shadow: none;
  } @else if $level == 1 {
    box-shadow: var(--shadow-sm);
  } @else if $level == 2 {
    box-shadow: var(--shadow-md);
  } @else {
    box-shadow: var(--shadow-lg);
  }
}

// Smooth transition
@mixin transition($properties...) {
  $result: ();
  @each $prop in $properties {
    $result: append($result, $prop var(--transition-normal), comma);
  }
  transition: $result;
}
```

```scss
// styles/_utilities.scss — utility classes dùng toàn app
// (Không dùng Tailwind nên cần tự định nghĩa)

// Spacing
@each $size, $value in (xs: 4px, sm: 8px, md: 16px, lg: 24px, xl: 32px) {
  .mt-#{$size} { margin-top: $value; }
  .mb-#{$size} { margin-bottom: $value; }
  .ml-#{$size} { margin-left: $value; }
  .mr-#{$size} { margin-right: $value; }
  .mx-#{$size} { margin-left: $value; margin-right: $value; }
  .my-#{$size} { margin-top: $value; margin-bottom: $value; }
  .m-#{$size} { margin: $value; }
  .p-#{$size} { padding: $value; }
  .px-#{$size} { padding-left: $value; padding-right: $value; }
  .py-#{$size} { padding-top: $value; padding-bottom: $value; }
}

// Display
.d-flex { display: flex; }
.d-grid { display: grid; }
.d-block { display: block; }
.d-none { display: none; }

// Flex utilities
.flex-1 { flex: 1; }
.flex-column { flex-direction: column; }
.align-center { align-items: center; }
.justify-center { justify-content: center; }
.justify-between { justify-content: space-between; }
.gap-sm { gap: 8px; }
.gap-md { gap: 16px; }
.gap-lg { gap: 24px; }

// Text
.text-center { text-align: center; }
.text-right { text-align: right; }
.font-bold { font-weight: 600; }
.text-muted { color: var(--mat-app-on-surface-variant); }

// Width/Height
.w-100 { width: 100%; }
.h-100 { height: 100%; }
.min-h-screen { min-height: 100vh; }
```

---

## 8.4 Component-level Theming

### Override Material Component Styles

Cách đúng đắn để customize Material components mà không dùng `::ng-deep` quá nhiều là dùng CSS custom properties của Material:

```scss
// user-table.component.scss
:host {
  // Override table styles thông qua CSS custom properties
  --mat-table-header-headline-color: var(--mat-app-primary);
  --mat-table-row-item-outline-color: transparent;
  --mat-table-header-container-height: 48px;
  --mat-table-footer-container-height: 48px;
  --mat-table-row-item-container-height: 56px;

  // Zebra striping
  .mat-mdc-row:nth-child(even) {
    background: var(--mat-app-surface-variant);
  }

  // Hover state
  .mat-mdc-row:hover {
    background: var(--mat-app-surface-container-highest);
    cursor: pointer;
  }

  // Selected row
  .mat-mdc-row.selected {
    background: var(--mat-app-primary-container);

    td {
      color: var(--mat-app-on-primary-container);
    }
  }
}
```

### Dark Mode Implementation

```typescript
// core/services/theme.service.ts
export type Theme = 'light' | 'dark' | 'system';

@Injectable({ providedIn: 'root' })
export class ThemeService {
  private readonly document = inject(DOCUMENT);
  private readonly storageKey = 'user-theme-preference';

  private readonly userPreference = signal<Theme>(
    (localStorage.getItem(this.storageKey) as Theme) ?? 'system'
  );

  private readonly systemPrefersDark = signal(
    window.matchMedia('(prefers-color-scheme: dark)').matches
  );

  readonly currentTheme = computed<'light' | 'dark'>(() => {
    const pref = this.userPreference();
    if (pref === 'system') {
      return this.systemPrefersDark() ? 'dark' : 'light';
    }
    return pref;
  });

  readonly isDark = computed(() => this.currentTheme() === 'dark');

  constructor() {
    // Lắng nghe system preference thay đổi
    const mediaQuery = window.matchMedia('(prefers-color-scheme: dark)');
    mediaQuery.addEventListener('change', (e) => {
      this.systemPrefersDark.set(e.matches);
    });

    // Apply theme khi thay đổi
    effect(() => {
      const isDark = this.isDark();
      this.document.documentElement.classList.toggle('dark-theme', isDark);
    });
  }

  setTheme(theme: Theme): void {
    this.userPreference.set(theme);
    localStorage.setItem(this.storageKey, theme);
  }
}
```

```typescript
// shared/components/theme-toggle/theme-toggle.component.ts
@Component({
  selector: 'app-theme-toggle',
  standalone: true,
  imports: [MatButtonModule, MatIconModule, MatMenuModule],
  template: `
    <button mat-icon-button [matMenuTriggerFor]="themeMenu"
            [title]="'Giao diện: ' + currentLabel()">
      <mat-icon>{{ themeIcon() }}</mat-icon>
    </button>

    <mat-menu #themeMenu>
      <button mat-menu-item (click)="themeService.setTheme('light')">
        <mat-icon>light_mode</mat-icon>
        <span>Sáng</span>
      </button>
      <button mat-menu-item (click)="themeService.setTheme('dark')">
        <mat-icon>dark_mode</mat-icon>
        <span>Tối</span>
      </button>
      <button mat-menu-item (click)="themeService.setTheme('system')">
        <mat-icon>settings_brightness</mat-icon>
        <span>Theo hệ thống</span>
      </button>
    </mat-menu>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ThemeToggleComponent {
  protected readonly themeService = inject(ThemeService);

  protected readonly themeIcon = computed(() =>
    this.themeService.isDark() ? 'dark_mode' : 'light_mode'
  );

  protected readonly currentLabel = computed(() =>
    this.themeService.isDark() ? 'Tối' : 'Sáng'
  );
}
```

---

## 8.5 Angular Material Components Thực Tế

### Layout với MatSidenav

```typescript
// shared/layouts/main-layout/main-layout.component.ts
@Component({
  selector: 'app-main-layout',
  standalone: true,
  imports: [
    RouterOutlet,
    MatSidenavModule,
    MatToolbarModule,
    MatButtonModule,
    MatIconModule,
    MatListModule,
  ],
  template: `
    <mat-sidenav-container class="layout-container">
      <mat-sidenav
        #sidenav
        [mode]="isDesktop() ? 'side' : 'over'"
        [opened]="isDesktop() && !isSidenavCollapsed()"
        class="app-sidenav"
        fixedInViewport
        [fixedTopGap]="64"
      >
        <nav>
          <mat-nav-list>
            @for (item of navItems; track item.path) {
              <a
                mat-list-item
                [routerLink]="item.path"
                routerLinkActive="active-link"
                [routerLinkActiveOptions]="{ exact: item.exact ?? false }"
              >
                <mat-icon matListItemIcon>{{ item.icon }}</mat-icon>
                <span matListItemTitle>{{ item.label }}</span>
              </a>
            }
          </mat-nav-list>
        </nav>
      </mat-sidenav>

      <mat-sidenav-content>
        <mat-toolbar color="primary" class="app-toolbar">
          <button mat-icon-button (click)="toggleSidenav()">
            <mat-icon>menu</mat-icon>
          </button>

          <span class="app-title">{{ appTitle }}</span>
          <span class="spacer"></span>

          <app-theme-toggle />
          <app-user-menu />
        </mat-toolbar>

        <main class="content-area">
          <router-outlet />
        </main>
      </mat-sidenav-content>
    </mat-sidenav-container>
  `,
  styleUrl: './main-layout.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class MainLayoutComponent {
  protected readonly appTitle = 'My Angular App';
  protected readonly isSidenavCollapsed = signal(false);

  // Responsive breakpoint detection
  private readonly breakpointObserver = inject(BreakpointObserver);
  protected readonly isDesktop = toSignal(
    this.breakpointObserver.observe([Breakpoints.Medium, Breakpoints.Large, Breakpoints.XLarge]).pipe(
      map((result) => result.matches)
    ),
    { initialValue: true }
  );

  protected readonly navItems = [
    { path: '/dashboard', label: 'Dashboard', icon: 'dashboard', exact: true },
    { path: '/users', label: 'Người dùng', icon: 'people' },
    { path: '/products', label: 'Sản phẩm', icon: 'inventory_2' },
    { path: '/orders', label: 'Đơn hàng', icon: 'shopping_bag' },
    { path: '/reports', label: 'Báo cáo', icon: 'bar_chart' },
    { path: '/settings', label: 'Cài đặt', icon: 'settings' },
  ];

  protected toggleSidenav(): void {
    this.isSidenavCollapsed.update((v) => !v);
  }
}
```

### MatTable với DataSource pattern

```typescript
// shared/data-sources/paginated-data-source.ts
import { DataSource } from '@angular/cdk/collections';

export class PaginatedDataSource<T> implements DataSource<T> {
  private readonly data = signal<T[]>([]);

  connect(): Observable<T[]> {
    return toObservable(this.data);
  }

  disconnect(): void {}

  setData(data: T[]): void {
    this.data.set(data);
  }
}
```

```typescript
// features/user/user-list/user-list.component.ts
@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [
    MatTableModule,
    MatSortModule,
    MatPaginatorModule,
    MatProgressBarModule,
  ],
  templateUrl: './user-list.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserListComponent implements OnInit, AfterViewInit {
  private readonly userService = inject(UserService);
  private readonly destroyRef = inject(DestroyRef);

  @ViewChild(MatSort) private sort!: MatSort;
  @ViewChild(MatPaginator) private paginator!: MatPaginator;

  protected readonly dataSource = new MatTableDataSource<User>([]);
  protected readonly displayedColumns = ['avatar', 'name', 'email', 'role', 'status', 'actions'];
  protected readonly isLoading = signal(false);
  protected readonly totalItems = signal(0);

  ngOnInit(): void {
    this.loadUsers();
  }

  ngAfterViewInit(): void {
    // Kết nối sort và paginator sau khi view init
    this.dataSource.sort = this.sort;

    // Reload data khi sort hoặc page thay đổi
    merge(this.sort.sortChange, this.paginator.page)
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe(() => this.loadUsers());
  }

  protected applyFilter(event: Event): void {
    const filterValue = (event.target as HTMLInputElement).value;
    this.dataSource.filter = filterValue.trim().toLowerCase();
  }

  private loadUsers(): void {
    this.isLoading.set(true);

    const params: UserQueryParams = {
      page: this.paginator?.pageIndex + 1 || 1,
      limit: this.paginator?.pageSize || 20,
      sortField: this.sort?.active || 'displayName',
      sortDirection: this.sort?.direction || 'asc',
    };

    this.userService
      .getUsers(params)
      .pipe(
        takeUntilDestroyed(this.destroyRef),
        finalize(() => this.isLoading.set(false))
      )
      .subscribe((response) => {
        this.dataSource.data = response.data;
        this.totalItems.set(response.pagination.total);
      });
  }
}
```

### MatDialog — Dialogs và Confirmation

```typescript
// shared/components/confirm-dialog/confirm-dialog.component.ts
export interface ConfirmDialogData {
  title: string;
  message: string;
  confirmText?: string;
  cancelText?: string;
  confirmColor?: 'primary' | 'warn' | 'accent';
}

@Component({
  selector: 'app-confirm-dialog',
  standalone: true,
  imports: [MatDialogModule, MatButtonModule],
  template: `
    <h2 mat-dialog-title>{{ data.title }}</h2>

    <mat-dialog-content>
      <p>{{ data.message }}</p>
    </mat-dialog-content>

    <mat-dialog-actions align="end">
      <button mat-button [mat-dialog-close]="false">
        {{ data.cancelText ?? 'Hủy' }}
      </button>
      <button
        mat-flat-button
        [color]="data.confirmColor ?? 'primary'"
        [mat-dialog-close]="true"
        cdkFocusInitial
      >
        {{ data.confirmText ?? 'Xác nhận' }}
      </button>
    </mat-dialog-actions>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ConfirmDialogComponent {
  protected readonly data = inject<ConfirmDialogData>(MAT_DIALOG_DATA);
}
```

```typescript
// core/services/dialog.service.ts — Wrapper cho MatDialog
@Injectable({ providedIn: 'root' })
export class DialogService {
  private readonly dialog = inject(MatDialog);

  confirm(data: ConfirmDialogData): Observable<boolean> {
    return this.dialog
      .open<ConfirmDialogComponent, ConfirmDialogData, boolean>(
        ConfirmDialogComponent,
        {
          data,
          width: '400px',
          disableClose: true,
        }
      )
      .afterClosed()
      .pipe(map((result) => result === true));
  }

  confirmDelete(entityName: string): Observable<boolean> {
    return this.confirm({
      title: 'Xác nhận xóa',
      message: `Bạn có chắc muốn xóa "${entityName}"? Hành động này không thể hoàn tác.`,
      confirmText: 'Xóa',
      cancelText: 'Hủy',
      confirmColor: 'warn',
    });
  }

  open<T, D = unknown, R = unknown>(
    component: ComponentType<T>,
    config?: MatDialogConfig<D>
  ): MatDialogRef<T, R> {
    return this.dialog.open(component, {
      width: '600px',
      maxWidth: '95vw',
      ...config,
    });
  }
}
```

```typescript
// Sử dụng trong component
@Component({ ... })
export class UserListComponent {
  private readonly dialogService = inject(DialogService);
  private readonly store = inject(UserStore);

  protected deleteUser(user: User): void {
    this.dialogService
      .confirmDelete(user.displayName)
      .pipe(filter(Boolean))
      .subscribe(() => this.store.deleteUser(user.id));
  }

  protected openEditDialog(user: User): void {
    const dialogRef = this.dialogService.open<
      UserFormDialogComponent,
      User,
      User | undefined
    >(UserFormDialogComponent, { data: user });

    dialogRef.afterClosed().pipe(
      filter((result): result is User => result !== undefined)
    ).subscribe((updatedUser) => {
      this.store.updateUser(updatedUser.id, updatedUser);
    });
  }
}
```

### MatSnackBar — Notifications

```typescript
// core/services/notification.service.ts
@Injectable({ providedIn: 'root' })
export class NotificationService {
  private readonly snackBar = inject(MatSnackBar);

  success(message: string, duration = 4000): void {
    this.show(message, 'success-snackbar', duration);
  }

  error(message: string, duration = 6000): void {
    this.show(message, 'error-snackbar', duration);
  }

  warning(message: string, duration = 5000): void {
    this.show(message, 'warning-snackbar', duration);
  }

  info(message: string, duration = 4000): void {
    this.show(message, 'info-snackbar', duration);
  }

  private show(message: string, panelClass: string, duration: number): void {
    this.snackBar.open(message, 'Đóng', {
      duration,
      panelClass: [panelClass],
      horizontalPosition: 'right',
      verticalPosition: 'top',
    });
  }
}
```

```scss
// styles/_snackbar.scss — Global snackbar styles
.success-snackbar {
  --mdc-snackbar-container-color: #2e7d32;
  --mat-snack-bar-button-color: #a5d6a7;
}

.error-snackbar {
  --mdc-snackbar-container-color: #c62828;
  --mat-snack-bar-button-color: #ef9a9a;
}

.warning-snackbar {
  --mdc-snackbar-container-color: #e65100;
  --mat-snack-bar-button-color: #ffcc80;
}

.info-snackbar {
  --mdc-snackbar-container-color: #01579b;
  --mat-snack-bar-button-color: #81d4fa;
}
```

---

## 8.6 Angular CDK — Khi Material Không Đủ

CDK (Component Dev Kit) cung cấp các primitive để xây dựng custom components:

### Virtual Scrolling — Danh sách dài

```typescript
// Chỉ render items trong viewport — tối ưu cho danh sách hàng nghìn items
@Component({
  selector: 'app-large-list',
  standalone: true,
  imports: [ScrollingModule],
  template: `
    <cdk-virtual-scroll-viewport itemSize="72" class="viewport">
      <div *cdkVirtualFor="let item of items; trackBy: trackById" class="list-item">
        <app-list-item [data]="item" />
      </div>
    </cdk-virtual-scroll-viewport>
  `,
  styles: [`
    .viewport {
      height: 600px;
    }
    .list-item {
      height: 72px;
    }
  `],
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class LargeListComponent {
  readonly items = input.required<Item[]>();

  protected trackById = (_: number, item: Item) => item.id;
}
```

### Drag and Drop

```typescript
@Component({
  selector: 'app-kanban-board',
  standalone: true,
  imports: [DragDropModule, MatCardModule],
  template: `
    <div class="board">
      @for (column of columns(); track column.id) {
        <div class="column">
          <h3>{{ column.title }}</h3>
          <div
            cdkDropList
            [cdkDropListData]="column.tasks"
            [cdkDropListConnectedTo]="dropListIds()"
            [id]="column.id"
            (cdkDropListDropped)="onDrop($event)"
            class="task-list"
          >
            @for (task of column.tasks; track task.id) {
              <mat-card cdkDrag class="task-card">
                <mat-card-content>{{ task.title }}</mat-card-content>
              </mat-card>
            }
          </div>
        </div>
      }
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class KanbanBoardComponent {
  protected readonly columns = signal<KanbanColumn[]>([
    { id: 'todo', title: 'Cần làm', tasks: [] },
    { id: 'doing', title: 'Đang làm', tasks: [] },
    { id: 'done', title: 'Hoàn thành', tasks: [] },
  ]);

  protected readonly dropListIds = computed(() =>
    this.columns().map((col) => col.id)
  );

  protected onDrop(event: CdkDragDrop<Task[]>): void {
    if (event.previousContainer === event.container) {
      // Di chuyển trong cùng column
      this.columns.update((columns) =>
        columns.map((col) => {
          if (col.id !== event.container.id) return col;
          const tasks = [...col.tasks];
          moveItemInArray(tasks, event.previousIndex, event.currentIndex);
          return { ...col, tasks };
        })
      );
    } else {
      // Di chuyển sang column khác
      this.columns.update((columns) => {
        const prevCol = columns.find(
          (c) => c.id === event.previousContainer.id
        )!;
        const currCol = columns.find((c) => c.id === event.container.id)!;
        const prevTasks = [...prevCol.tasks];
        const currTasks = [...currCol.tasks];

        transferArrayItem(prevTasks, currTasks, event.previousIndex, event.currentIndex);

        return columns.map((col) => {
          if (col.id === prevCol.id) return { ...col, tasks: prevTasks };
          if (col.id === currCol.id) return { ...col, tasks: currTasks };
          return col;
        });
      });
    }
  }
}
```

---

## Tổng kết chương

Angular Material và SCSS là nền tảng UI của mọi Angular enterprise app. Những điểm cốt lõi:

1. **Material Design 3 theming** dùng `mat.define-theme()` và CSS custom properties — linh hoạt, hỗ trợ dark mode, và tree-shakable.

2. **ViewEncapsulation.Emulated** là mặc định và đủ cho hầu hết trường hợp. `::ng-deep` chỉ dùng khi thực sự cần thiết, luôn kết hợp với `:host` để giới hạn scope.

3. **SCSS architecture** theo pattern partials: variables, mixins, breakpoints, utilities riêng biệt — import theo thứ tự đúng trong `styles.scss`.

4. **DialogService và NotificationService** là wrappers cần thiết cho `MatDialog` và `MatSnackBar` — đảm bảo API nhất quán và type-safe khắp ứng dụng.

5. **Angular CDK** cung cấp virtual scrolling, drag-drop, overlay, và accessibility utilities — là nền tảng xây dựng custom components khi Material không đủ.

Chương tiếp theo sẽ đi vào **Architecture & Patterns thực tế** — cách tổ chức Angular project ở quy mô lớn, bao gồm feature-based architecture, directives, pipes, và performance optimization.
