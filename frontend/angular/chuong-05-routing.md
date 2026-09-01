# Chương 5: Routing

## Giới thiệu chương

Angular Router là một trong những router mạnh nhất trong các frontend framework — được tích hợp sẵn, không cần cài thêm thư viện bên ngoài. So với React Router hay Next.js App Router, Angular Router có nhiều tính năng hơn nhưng cũng đòi hỏi hiểu cấu hình rõ ràng hơn.

Chương này đi từ cấu hình cơ bản đến các kỹ thuật nâng cao được dùng trong production: lazy loading, functional guards, route resolvers, và các pattern điều hướng thực tế.

---

## 5.1 Cấu hình Router cơ bản

### Khai báo Routes

Routes trong Angular là một array của `Route` objects, mỗi object định nghĩa ánh xạ giữa URL path và component cần hiển thị.

```typescript
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  // Redirect path rỗng về /dashboard
  {
    path: '',
    redirectTo: '/dashboard',
    pathMatch: 'full',  // 'full' = chỉ match khi path CHÍNH XÁC là ''
  },

  // Route tĩnh — eager load (không lazy)
  {
    path: 'home',
    component: HomeComponent,
    title: 'Trang chủ',  // Angular 14+ — set document.title tự động
  },

  // Route với tham số
  {
    path: 'users/:id',
    component: UserDetailComponent,
    title: 'Chi tiết người dùng',
  },

  // Route lồng nhau (nested routes)
  {
    path: 'settings',
    component: SettingsLayoutComponent,
    children: [
      { path: '', redirectTo: 'profile', pathMatch: 'full' },
      { path: 'profile', component: ProfileSettingsComponent },
      { path: 'security', component: SecuritySettingsComponent },
      { path: 'notifications', component: NotificationSettingsComponent },
    ],
  },

  // Wildcard — luôn đặt cuối cùng
  {
    path: '**',
    component: NotFoundComponent,
    title: 'Trang không tìm thấy',
  },
];
```

```typescript
// app.config.ts
import { provideRouter, withComponentInputBinding, withViewTransitions } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withComponentInputBinding(),  // Bind route params vào @Input
      withViewTransitions(),         // Smooth page transitions (Angular 17+)
    ),
  ],
};
```

### RouterOutlet và Navigation

```typescript
// app.component.ts — root component chứa RouterOutlet
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet, RouterLink, RouterLinkActive],
  template: `
    <nav class="main-nav">
      <!-- routerLink — điều hướng không reload trang -->
      <a routerLink="/dashboard" routerLinkActive="active">Dashboard</a>
      <a routerLink="/users" routerLinkActive="active">Người dùng</a>
      <a routerLink="/settings" routerLinkActive="active">Cài đặt</a>

      <!-- routerLinkActiveOptions: exact match -->
      <a
        routerLink="/"
        routerLinkActive="active"
        [routerLinkActiveOptions]="{ exact: true }"
      >
        Trang chủ
      </a>
    </nav>

    <!-- Nơi Angular render component tương ứng với route hiện tại -->
    <main>
      <router-outlet />
    </main>
  `,
})
export class AppComponent {}
```

### Điều hướng bằng Code

```typescript
@Component({ ... })
export class UserFormComponent {
  private readonly router = inject(Router);
  private readonly userService = inject(UserService);

  onSubmit(dto: CreateUserDto): void {
    this.userService.createUser(dto).subscribe({
      next: (user) => {
        // Điều hướng với absolute path
        this.router.navigate(['/users', user.id]);

        // Hoặc với extras
        this.router.navigate(['/users'], {
          queryParams: { created: 'true' },
          state: { newUserId: user.id },  // State không xuất hiện trong URL
        });
      },
    });
  }

  onCancel(): void {
    // Điều hướng tương đối
    this.router.navigate(['..'], { relativeTo: this.route });
  }

  goBack(): void {
    // Dùng Location service thay vì window.history
    this.location.back();
  }
}
```

---

## 5.2 Đọc Route Parameters và Query Params

### Với `withComponentInputBinding()` — Cách hiện đại nhất

Angular 16+ cho phép bind route params trực tiếp vào `@Input` hoặc `input()` signals khi bật `withComponentInputBinding()`:

```typescript
// user-detail.component.ts
@Component({
  selector: 'app-user-detail',
  standalone: true,
  imports: [CommonModule, MatCardModule, MatButtonModule],
  templateUrl: './user-detail.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserDetailComponent implements OnInit {
  // Route param :id được bind trực tiếp vào đây
  readonly id = input.required<string>();

  // Query params cũng được bind
  readonly tab = input<string>('overview');

  private readonly userService = inject(UserService);
  private readonly destroyRef = inject(DestroyRef);

  protected readonly user = signal<UserDetail | null>(null);
  protected readonly isLoading = signal(false);
  protected readonly error = signal<string | null>(null);

  ngOnInit(): void {
    // Dùng effect để react khi id thay đổi (navigate giữa users)
    effect(
      () => {
        const userId = this.id();
        this.loadUser(userId);
      },
      { injector: inject(Injector) }
    );
  }

  private loadUser(id: string): void {
    this.isLoading.set(true);
    this.error.set(null);

    this.userService
      .getUserById(id)
      .pipe(
        takeUntilDestroyed(this.destroyRef),
        finalize(() => this.isLoading.set(false))
      )
      .subscribe({
        next: (user) => this.user.set(user),
        error: () => this.error.set('Không tìm thấy người dùng'),
      });
  }
}
```

### Với ActivatedRoute — Cách truyền thống

```typescript
@Component({ ... })
export class UserDetailComponent implements OnInit {
  private readonly route = inject(ActivatedRoute);
  private readonly userService = inject(UserService);
  private readonly destroyRef = inject(DestroyRef);

  protected readonly user = signal<UserDetail | null>(null);

  ngOnInit(): void {
    // Cách 1: snapshot — giá trị tại thời điểm route được khởi tạo
    // Dùng khi component không bị reuse (mỗi route tạo instance mới)
    const id = this.route.snapshot.paramMap.get('id');
    if (id) this.loadUser(id);

    // Cách 2: Observable — reactive, dùng khi component có thể bị reuse
    // Ví dụ: navigate từ /users/1 sang /users/2 mà không destroy component
    this.route.paramMap.pipe(
      map((params) => params.get('id')),
      filter((id): id is string => id !== null),
      distinctUntilChanged(),
      switchMap((id) => this.userService.getUserById(id)),
      takeUntilDestroyed(this.destroyRef)
    ).subscribe((user) => this.user.set(user));
  }

  ngOnInit(): void {
    // Query params
    this.route.queryParamMap.pipe(
      takeUntilDestroyed(this.destroyRef)
    ).subscribe((params) => {
      const page = Number(params.get('page')) || 1;
      const search = params.get('search') ?? '';
      this.loadUsers({ page, search });
    });
  }
}
```

### Cập nhật Query Params không reload trang

```typescript
@Component({ ... })
export class UserListComponent {
  private readonly router = inject(Router);
  private readonly route = inject(ActivatedRoute);

  protected readonly searchQuery = signal('');
  protected readonly currentPage = signal(1);

  // Sync state với URL khi state thay đổi
  constructor() {
    effect(() => {
      const query = this.searchQuery();
      const page = this.currentPage();

      this.router.navigate([], {
        relativeTo: this.route,
        queryParams: {
          search: query || null,  // null = xóa param khỏi URL
          page: page > 1 ? page : null,
        },
        queryParamsHandling: 'merge',  // Giữ các params khác
        replaceUrl: true,              // Không thêm vào browser history
      });
    });
  }
}
```

---

## 5.3 Lazy Loading

### Tại sao Lazy Loading quan trọng

Khi không dùng lazy loading, Angular bundle toàn bộ code vào một file JavaScript duy nhất. File này phải được tải và parse trước khi user thấy bất kỳ gì. Lazy loading chia nhỏ bundle thành các chunk — chỉ tải khi user thực sự navigate đến route đó.

```typescript
// app.routes.ts — lazy loading hoàn chỉnh
export const routes: Routes = [
  // Eager load — không nên dùng cho large components
  {
    path: 'home',
    component: HomeComponent,
  },

  // Lazy load single standalone component
  {
    path: 'dashboard',
    loadComponent: () =>
      import('./features/dashboard/dashboard.component').then(
        (m) => m.DashboardComponent
      ),
  },

  // Lazy load feature routes — cách khuyến khích cho features lớn
  {
    path: 'users',
    loadChildren: () =>
      import('./features/user/user.routes').then((m) => m.USER_ROUTES),
  },

  {
    path: 'products',
    loadChildren: () =>
      import('./features/product/product.routes').then(
        (m) => m.PRODUCT_ROUTES
      ),
  },

  {
    path: 'settings',
    loadChildren: () =>
      import('./features/settings/settings.routes').then(
        (m) => m.SETTINGS_ROUTES
      ),
  },
];
```

```typescript
// features/user/user.routes.ts
export const USER_ROUTES: Routes = [
  {
    path: '',
    // Layout component cho feature — chứa sidebar, breadcrumb...
    loadComponent: () =>
      import('./user-layout.component').then((m) => m.UserLayoutComponent),
    children: [
      {
        path: '',
        loadComponent: () =>
          import('./user-list/user-list.component').then(
            (m) => m.UserListComponent
          ),
        title: 'Danh sách người dùng',
      },
      {
        path: 'create',
        loadComponent: () =>
          import('./user-form/user-form.component').then(
            (m) => m.UserFormComponent
          ),
        title: 'Tạo người dùng mới',
      },
      {
        path: ':id',
        loadComponent: () =>
          import('./user-detail/user-detail.component').then(
            (m) => m.UserDetailComponent
          ),
        title: 'Chi tiết người dùng',
      },
      {
        path: ':id/edit',
        loadComponent: () =>
          import('./user-form/user-form.component').then(
            (m) => m.UserFormComponent
          ),
        title: 'Chỉnh sửa người dùng',
      },
    ],
  },
];
```

### Preloading Strategy

```typescript
// app.config.ts — preload tất cả lazy modules sau khi app load xong
import { PreloadAllModules, withPreloading } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withPreloading(PreloadAllModules)  // Preload tất cả
    ),
  ],
};
```

```typescript
// Custom preloading strategy — chỉ preload routes được đánh dấu
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class SelectivePreloadingStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<unknown>): Observable<unknown> {
    // Chỉ preload routes có data.preload = true
    if (route.data?.['preload'] === true) {
      return load();
    }
    return of(null);
  }
}

// Sử dụng trong routes
export const routes: Routes = [
  {
    path: 'dashboard',
    data: { preload: true },  // Sẽ được preload
    loadComponent: () => import('./dashboard.component').then(m => m.DashboardComponent),
  },
  {
    path: 'reports',
    // Không có data.preload → chỉ load khi user navigate đến
    loadComponent: () => import('./reports.component').then(m => m.ReportsComponent),
  },
];
```

---

## 5.4 Route Guards — Bảo vệ Routes

Guards kiểm soát khả năng truy cập vào một route. Angular 15+ dùng **functional guards** — đơn giản hơn nhiều so với class-based guards cũ.

### `canActivate` — Kiểm tra quyền truy cập

```typescript
// core/guards/auth.guard.ts
import { CanActivateFn, Router } from '@angular/router';

export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }

  // Lưu URL để redirect sau khi login
  return router.createUrlTree(['/auth/login'], {
    queryParams: { returnUrl: state.url },
  });
};
```

```typescript
// core/guards/role.guard.ts
export const roleGuard = (requiredRoles: UserRole[]): CanActivateFn => {
  return () => {
    const authService = inject(AuthService);
    const router = inject(Router);

    const user = authService.currentUser();

    if (!user) {
      return router.createUrlTree(['/auth/login']);
    }

    if (requiredRoles.includes(user.role)) {
      return true;
    }

    // Đủ quyền nhưng không đủ role → forbidden
    return router.createUrlTree(['/forbidden']);
  };
};

// Sử dụng trong routes
{
  path: 'admin',
  canActivate: [authGuard, roleGuard([UserRole.Admin])],
  loadChildren: () => import('./admin/admin.routes').then(m => m.ADMIN_ROUTES),
}
```

### `canDeactivate` — Cảnh báo khi rời trang

```typescript
// core/guards/unsaved-changes.guard.ts

// Interface cho components có unsaved changes
export interface HasUnsavedChanges {
  hasUnsavedChanges(): boolean;
}

export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = (
  component
) => {
  if (!component.hasUnsavedChanges()) {
    return true;
  }

  // Trong production nên dùng MatDialog thay vì confirm
  return confirm(
    'Bạn có thay đổi chưa được lưu. Bạn có chắc muốn rời trang này không?'
  );
};

// Dùng với MatDialog — UX tốt hơn
export const unsavedChangesDialogGuard: CanDeactivateFn<HasUnsavedChanges> = (
  component
) => {
  if (!component.hasUnsavedChanges()) {
    return true;
  }

  const dialog = inject(MatDialog);

  return dialog
    .open(ConfirmDialogComponent, {
      data: {
        title: 'Rời trang?',
        message: 'Bạn có thay đổi chưa được lưu. Nếu rời trang, thay đổi sẽ bị mất.',
        confirmText: 'Rời trang',
        cancelText: 'Ở lại',
      },
    })
    .afterClosed()
    .pipe(map((confirmed) => confirmed === true));
};
```

```typescript
// Implement interface trong component
@Component({ ... })
export class UserFormComponent implements HasUnsavedChanges {
  protected readonly form = this.fb.group({ ... });
  private readonly initialValue = signal<unknown>(null);

  ngOnInit(): void {
    this.initialValue.set(this.form.value);
  }

  hasUnsavedChanges(): boolean {
    return (
      this.form.dirty &&
      JSON.stringify(this.form.value) !==
        JSON.stringify(this.initialValue())
    );
  }
}

// Trong routes
{
  path: ':id/edit',
  canDeactivate: [unsavedChangesDialogGuard],
  loadComponent: () => import('./user-form.component').then(m => m.UserFormComponent),
}
```

### `resolve` — Load data trước khi render

Resolver giải quyết bài toán "hiển thị skeleton hay load data xong mới render". Với resolver, data đã sẵn sàng trước khi component được tạo.

```typescript
// features/user/resolvers/user-detail.resolver.ts
import { ResolveFn } from '@angular/router';

export const userDetailResolver: ResolveFn<UserDetail> = (route) => {
  const userService = inject(UserService);
  const router = inject(Router);
  const id = route.paramMap.get('id');

  if (!id) {
    router.navigate(['/users']);
    return EMPTY; // Hủy navigation
  }

  return userService.getUserById(id).pipe(
    catchError(() => {
      router.navigate(['/users'], {
        queryParams: { error: 'user-not-found' },
      });
      return EMPTY;
    })
  );
};

// Trong routes
{
  path: ':id',
  resolve: { user: userDetailResolver },
  loadComponent: () =>
    import('./user-detail.component').then(m => m.UserDetailComponent),
}
```

```typescript
// Component đọc resolved data — không cần gọi service nữa
@Component({ ... })
export class UserDetailComponent {
  private readonly route = inject(ActivatedRoute);

  // Với withComponentInputBinding() — resolve data được bind vào @Input
  readonly user = input.required<UserDetail>();

  // Hoặc đọc từ route.snapshot.data (cách truyền thống)
  protected readonly user = signal<UserDetail>(
    this.route.snapshot.data['user'] as UserDetail
  );
}
```

### `canMatch` — Điều kiện load route

Khác với `canActivate` (route đã match, chỉ chặn access), `canMatch` quyết định liệu route có được xem xét để match hay không — hữu ích cho A/B testing hoặc feature flags:

```typescript
// core/guards/feature-flag.guard.ts
export const featureFlagGuard = (flagName: string): CanMatchFn => {
  return () => {
    const featureFlagService = inject(FeatureFlagService);
    return featureFlagService.isEnabled(flagName);
  };
};

// Trong routes — hai routes cho cùng path
export const routes: Routes = [
  {
    path: 'dashboard',
    canMatch: [featureFlagGuard('new-dashboard')],
    loadComponent: () =>
      import('./new-dashboard.component').then(m => m.NewDashboardComponent),
  },
  {
    path: 'dashboard',
    // Không có canMatch — fallback khi flag tắt
    loadComponent: () =>
      import('./old-dashboard.component').then(m => m.OldDashboardComponent),
  },
];
```

---

## 5.5 Dynamic Title và Breadcrumbs

### Dynamic Route Title

```typescript
// Angular 14+ — tự động set document.title

// Cách 1: Title tĩnh trong route config
{ path: 'users', title: 'Danh sách người dùng', ... }

// Cách 2: TitleStrategy custom — title động dựa trên route data
@Injectable({ providedIn: 'root' })
export class AppTitleStrategy extends TitleStrategy {
  private readonly title = inject(Title);

  override updateTitle(snapshot: RouterStateSnapshot): void {
    const title = this.buildTitle(snapshot);
    const appName = 'My Angular App';

    this.title.setTitle(title ? `${title} | ${appName}` : appName);
  }
}

// Đăng ký trong app.config.ts
{
  provide: TitleStrategy,
  useClass: AppTitleStrategy,
}
```

### Breadcrumb Service

```typescript
// core/services/breadcrumb.service.ts
export interface Breadcrumb {
  label: string;
  url: string;
}

@Injectable({ providedIn: 'root' })
export class BreadcrumbService {
  private readonly router = inject(Router);
  private readonly activatedRoute = inject(ActivatedRoute);

  readonly breadcrumbs$: Observable<Breadcrumb[]> = this.router.events.pipe(
    filter((event) => event instanceof NavigationEnd),
    startWith(null),
    map(() => this.buildBreadcrumbs(this.activatedRoute.root)),
    shareReplay(1)
  );

  private buildBreadcrumbs(
    route: ActivatedRoute,
    url = '',
    breadcrumbs: Breadcrumb[] = []
  ): Breadcrumb[] {
    const children = route.children;

    for (const child of children) {
      const routeURL = child.snapshot.url
        .map((segment) => segment.path)
        .join('/');

      const fullUrl = routeURL ? `${url}/${routeURL}` : url;
      const label = child.snapshot.data['breadcrumb'] as string | undefined;

      if (label) {
        breadcrumbs.push({ label, url: fullUrl });
      }

      return this.buildBreadcrumbs(child, fullUrl, breadcrumbs);
    }

    return breadcrumbs;
  }
}

// Thêm breadcrumb data vào routes
{
  path: 'users',
  data: { breadcrumb: 'Người dùng' },
  children: [
    {
      path: ':id',
      data: { breadcrumb: 'Chi tiết' },
      ...
    }
  ]
}
```

---

## 5.6 Router Events và Điều hướng nâng cao

### Lắng nghe Router Events

```typescript
@Injectable({ providedIn: 'root' })
export class RouterEventsService {
  private readonly router = inject(Router);

  // Tracking navigation để hiển thị loading bar
  readonly isNavigating$ = this.router.events.pipe(
    filter(
      (event) =>
        event instanceof NavigationStart ||
        event instanceof NavigationEnd ||
        event instanceof NavigationCancel ||
        event instanceof NavigationError
    ),
    map((event) => event instanceof NavigationStart),
    distinctUntilChanged(),
    shareReplay(1)
  );

  // Scroll to top khi navigate
  constructor() {
    this.router.events
      .pipe(filter((event) => event instanceof NavigationEnd))
      .subscribe(() => {
        window.scrollTo({ top: 0, behavior: 'smooth' });
      });
  }
}
```

### returnUrl Pattern — Redirect sau login

```typescript
// auth/login/login.component.ts
@Component({ ... })
export class LoginComponent {
  private readonly authService = inject(AuthService);
  private readonly router = inject(Router);
  private readonly route = inject(ActivatedRoute);

  protected readonly form = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', Validators.required],
  });

  protected readonly isLoading = signal(false);

  protected onSubmit(): void {
    if (this.form.invalid) return;

    this.isLoading.set(true);
    const { email, password } = this.form.getRawValue();

    this.authService.login({ email, password }).subscribe({
      next: () => {
        // Redirect về trang user muốn truy cập, hoặc dashboard
        const returnUrl =
          this.route.snapshot.queryParams['returnUrl'] ?? '/dashboard';

        // Tránh open redirect vulnerability — chỉ chấp nhận relative URLs
        const safeReturnUrl = returnUrl.startsWith('/')
          ? returnUrl
          : '/dashboard';

        this.router.navigateByUrl(safeReturnUrl);
      },
      error: () => {
        this.isLoading.set(false);
      },
    });
  }
}
```

---

## Tổng kết chương

Angular Router là một hệ thống đầy đủ tính năng, được thiết kế cho ứng dụng lớn. Những điểm cốt lõi:

1. **Lazy loading là bắt buộc** cho ứng dụng production — dùng `loadComponent` cho single component và `loadChildren` cho feature routes có nhiều sub-routes.

2. **Functional guards** (Angular 15+) đơn giản và composable hơn class-based guards. Dùng factory function (`roleGuard([UserRole.Admin])`) để tạo guards có tham số.

3. **`withComponentInputBinding()`** giúp nhận route params, query params, và resolved data trực tiếp qua `@Input`/`input()` — code sạch hơn nhiều so với inject `ActivatedRoute`.

4. **Resolvers** load data trước khi render component — giải quyết bài toán "skeleton vs data-ready" một cách rõ ràng.

5. **Query params** là cách đúng đắn để lưu filter/sort/pagination state vào URL — cho phép share link và browser back hoạt động đúng.

Chương tiếp theo sẽ đi vào **Reactive Forms và Validation** — form system mạnh nhất trong các frontend framework, và cách kết hợp với Zod để có validation type-safe.
