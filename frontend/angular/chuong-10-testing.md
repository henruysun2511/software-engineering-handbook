# Chương 10: Testing

## Giới thiệu chương

Testing trong Angular có hỗ trợ rất tốt từ framework — `TestBed` là một Angular application container thu nhỏ cho phép test component và service trong môi trường gần giống runtime thật, trong khi `HttpTestingController` cho phép kiểm soát và verify HTTP requests mà không cần server thật.

Chương này tập trung vào ba tầng testing thực tế: unit test services và stores, component testing với TestBed, và end-to-end testing với Playwright. Angular 18 dùng Jest thay Karma theo mặc định — setup nhanh hơn và DX tốt hơn.

---

## 10.1 Setup Jest trong Angular 18

### Cài đặt và Cấu hình

```bash
# Cài Jest cho Angular
ng add jest-preset-angular

# Hoặc thủ công
npm install --save-dev jest jest-preset-angular @types/jest
```

```typescript
// jest.config.ts
import type { Config } from 'jest';

const config: Config = {
  preset: 'jest-preset-angular',
  setupFilesAfterFramework: ['<rootDir>/setup-jest.ts'],
  testPathPattern: ['.*\\.spec\\.ts$'],
  transform: {
    '^.+\\.(ts|js|mjs|cjs|jsx)$': [
      'jest-preset-angular',
      {
        tsconfig: '<rootDir>/tsconfig.spec.json',
        stringifyContentPathRegex: '\\.(html|svg)$',
      },
    ],
  },
  moduleNameMapper: {
    '^@core/(.*)$': '<rootDir>/src/app/core/$1',
    '^@shared/(.*)$': '<rootDir>/src/app/shared/$1',
    '^@features/(.*)$': '<rootDir>/src/app/features/$1',
    '^@env/(.*)$': '<rootDir>/src/environments/$1',
  },
  collectCoverageFrom: [
    'src/app/**/*.ts',
    '!src/app/**/*.module.ts',
    '!src/app/**/*.routes.ts',
    '!src/app/**/index.ts',
    '!src/main.ts',
  ],
  coverageThresholds: {
    global: {
      branches: 70,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
};

export default config;
```

```typescript
// setup-jest.ts
import 'jest-preset-angular/setup-jest';
```

---

## 10.2 Testing Services

Services không có DOM — test đơn giản nhất, chỉ cần inject và gọi methods:

```typescript
// features/user/services/user.service.spec.ts
import { TestBed } from '@angular/core/testing';
import {
  HttpClientTestingModule,
  HttpTestingController,
} from '@angular/common/http/testing';
import { UserService } from './user.service';
import { API_URL } from '@core/tokens';

describe('UserService', () => {
  let service: UserService;
  let httpMock: HttpTestingController;

  const mockUsers: User[] = [
    {
      id: '1',
      displayName: 'Alice Nguyen',
      email: 'alice@example.com',
      role: 'ADMIN',
      isActive: true,
      createdAt: '2024-01-01T00:00:00Z',
      updatedAt: '2024-01-01T00:00:00Z',
    },
  ];

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [
        UserService,
        { provide: API_URL, useValue: 'http://test-api.com' },
      ],
    });

    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    // Verify không có outstanding requests
    httpMock.verify();
  });

  describe('getUsers()', () => {
    it('should return paginated users', (done) => {
      const mockResponse = {
        data: mockUsers,
        pagination: { page: 1, limit: 20, total: 1, totalPages: 1 },
      };

      service.getUsers().subscribe((response) => {
        expect(response.data).toEqual(mockUsers);
        expect(response.pagination.total).toBe(1);
        done();
      });

      const req = httpMock.expectOne('http://test-api.com/users');
      expect(req.request.method).toBe('GET');
      req.flush(mockResponse);
    });

    it('should pass query params correctly', (done) => {
      service.getUsers({ page: 2, limit: 10, search: 'alice' }).subscribe(() => done());

      const req = httpMock.expectOne(
        (r) =>
          r.url === 'http://test-api.com/users' &&
          r.params.get('page') === '2' &&
          r.params.get('limit') === '10' &&
          r.params.get('search') === 'alice'
      );
      req.flush({ data: [], pagination: { page: 2, limit: 10, total: 0, totalPages: 0 } });
    });

    it('should handle HTTP error gracefully', (done) => {
      service.getUsers().subscribe({
        error: (err) => {
          expect(err.status).toBe(500);
          done();
        },
      });

      const req = httpMock.expectOne('http://test-api.com/users');
      req.flush('Server Error', { status: 500, statusText: 'Internal Server Error' });
    });
  });

  describe('createUser()', () => {
    it('should POST and return created user', (done) => {
      const dto: CreateUserDto = {
        email: 'bob@example.com',
        displayName: 'Bob',
        role: 'EDITOR',
      };

      service.createUser(dto).subscribe((user) => {
        expect(user.email).toBe(dto.email);
        expect(user.id).toBeDefined();
        done();
      });

      const req = httpMock.expectOne('http://test-api.com/users');
      expect(req.request.method).toBe('POST');
      expect(req.request.body).toEqual(dto);
      req.flush({ ...dto, id: 'new-id', isActive: true, createdAt: '', updatedAt: '' });
    });
  });

  describe('deleteUser()', () => {
    it('should send DELETE request', (done) => {
      service.deleteUser('1').subscribe(() => done());

      const req = httpMock.expectOne('http://test-api.com/users/1');
      expect(req.request.method).toBe('DELETE');
      req.flush(null);
    });
  });
});
```

### Testing Service với Dependencies

```typescript
// core/services/auth.service.spec.ts
describe('AuthService', () => {
  let service: AuthService;
  let httpMock: HttpTestingController;
  let routerSpy: jest.SpyInstance;

  beforeEach(() => {
    const routerMock = { navigate: jest.fn() };

    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [
        AuthService,
        { provide: Router, useValue: routerMock },
        { provide: API_URL, useValue: 'http://test-api.com' },
      ],
    });

    service = TestBed.inject(AuthService);
    httpMock = TestBed.inject(HttpTestingController);
    routerSpy = jest.spyOn(TestBed.inject(Router), 'navigate');
  });

  afterEach(() => httpMock.verify());

  it('should set user and token on successful login', (done) => {
    const credentials = { email: 'admin@example.com', password: 'password123' };
    const mockResponse = {
      user: { id: '1', email: 'admin@example.com', role: 'ADMIN' },
      accessToken: 'mock-jwt-token',
    };

    service.login(credentials).subscribe(() => {
      expect(service.isAuthenticated()).toBe(true);
      expect(service.currentUser()?.email).toBe('admin@example.com');
      done();
    });

    const req = httpMock.expectOne('http://test-api.com/auth/login');
    req.flush(mockResponse);
  });

  it('should clear user and navigate to login on logout', () => {
    // Arrange — set initial state
    service['currentUserSignal'].set({
      id: '1',
      email: 'user@example.com',
      role: 'VIEWER',
    } as AuthUser);

    // Act
    service.logout();

    // Assert
    expect(service.currentUser()).toBeNull();
    expect(service.isAuthenticated()).toBe(false);
    expect(routerSpy).toHaveBeenCalledWith(['/auth/login']);
  });
});
```

---

## 10.3 Testing NgRx Signals Store

Store là plain class — test không cần setup phức tạp:

```typescript
// features/user/store/user.store.spec.ts
import { TestBed } from '@angular/core/testing';
import { of, throwError } from 'rxjs';
import { UserStore } from './user.store';
import { UserService } from '../services/user.service';

describe('UserStore', () => {
  let store: InstanceType<typeof UserStore>;
  let userServiceMock: jest.Mocked<Partial<UserService>>;

  const mockUsers: User[] = [
    { id: '1', displayName: 'Alice', email: 'alice@example.com', role: 'ADMIN', isActive: true, createdAt: '', updatedAt: '' },
    { id: '2', displayName: 'Bob', email: 'bob@example.com', role: 'EDITOR', isActive: true, createdAt: '', updatedAt: '' },
    { id: '3', displayName: 'Charlie', email: 'charlie@example.com', role: 'VIEWER', isActive: false, createdAt: '', updatedAt: '' },
  ];

  beforeEach(() => {
    userServiceMock = {
      getUsers: jest.fn().mockReturnValue(
        of({ data: mockUsers, pagination: { page: 1, limit: 20, total: 3, totalPages: 1 } })
      ),
      deleteUser: jest.fn().mockReturnValue(of(undefined)),
      createUser: jest.fn(),
    };

    TestBed.configureTestingModule({
      providers: [
        UserStore,
        { provide: UserService, useValue: userServiceMock },
      ],
    });

    store = TestBed.inject(UserStore);
  });

  describe('Initial State', () => {
    it('should have empty users array', () => {
      expect(store.users()).toEqual([]);
    });

    it('should not be loading', () => {
      expect(store.isLoading()).toBe(false);
    });

    it('should have no error', () => {
      expect(store.error()).toBeNull();
    });
  });

  describe('loadUsers()', () => {
    it('should load users and update state', async () => {
      await store.loadUsers();

      expect(store.users().length).toBe(3);
      expect(store.isLoading()).toBe(false);
      expect(store.error()).toBeNull();
      expect(userServiceMock.getUsers).toHaveBeenCalledTimes(1);
    });

    it('should set error state when load fails', async () => {
      userServiceMock.getUsers!.mockReturnValue(
        throwError(() => new Error('Network error'))
      );

      await store.loadUsers();

      expect(store.users()).toEqual([]);
      expect(store.error()).not.toBeNull();
      expect(store.isLoading()).toBe(false);
    });
  });

  describe('filteredUsers()', () => {
    beforeEach(async () => {
      await store.loadUsers();
    });

    it('should return all users when no filter applied', () => {
      expect(store.filteredUsers().length).toBe(3);
    });

    it('should filter by search query', () => {
      store.setFilter({ search: 'alice' });
      expect(store.filteredUsers().length).toBe(1);
      expect(store.filteredUsers()[0].displayName).toBe('Alice');
    });

    it('should filter by role', () => {
      store.setFilter({ role: 'ADMIN' });
      expect(store.filteredUsers().length).toBe(1);
      expect(store.filteredUsers()[0].role).toBe('ADMIN');
    });

    it('should combine search and role filters', () => {
      store.setFilter({ search: 'b', role: 'EDITOR' });
      expect(store.filteredUsers().length).toBe(1);
      expect(store.filteredUsers()[0].displayName).toBe('Bob');
    });
  });

  describe('deleteUser()', () => {
    beforeEach(async () => {
      await store.loadUsers();
    });

    it('should remove user from state after deletion', async () => {
      expect(store.users().length).toBe(3);

      await store.deleteUser('1');

      expect(store.users().length).toBe(2);
      expect(store.users().find((u) => u.id === '1')).toBeUndefined();
      expect(userServiceMock.deleteUser).toHaveBeenCalledWith('1');
    });

    it('should clear selectedUserId if deleted user was selected', async () => {
      store.selectUser('1');
      expect(store.selectedUserId()).toBe('1');

      await store.deleteUser('1');

      expect(store.selectedUserId()).toBeNull();
    });
  });
});
```

---

## 10.4 Testing Components với TestBed

```typescript
// features/user/user-list/user-table/user-table.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { UserTableComponent } from './user-table.component';
import { By } from '@angular/platform-browser';
import { NoopAnimationsModule } from '@angular/platform-browser/animations';

describe('UserTableComponent', () => {
  let component: UserTableComponent;
  let fixture: ComponentFixture<UserTableComponent>;

  const mockUsers: User[] = [
    { id: '1', displayName: 'Alice', email: 'alice@example.com', role: 'ADMIN', isActive: true, createdAt: '', updatedAt: '' },
    { id: '2', displayName: 'Bob', email: 'bob@example.com', role: 'EDITOR', isActive: false, createdAt: '', updatedAt: '' },
  ];

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [
        UserTableComponent,    // Standalone component — import trực tiếp
        NoopAnimationsModule,  // Tắt animations trong test
      ],
    }).compileComponents();

    fixture = TestBed.createComponent(UserTableComponent);
    component = fixture.componentInstance;
  });

  function setInputs(users = mockUsers, isLoading = false) {
    fixture.componentRef.setInput('users', users);
    fixture.componentRef.setInput('isLoading', isLoading);
    fixture.detectChanges();
  }

  describe('Rendering', () => {
    it('should render user rows', () => {
      setInputs();
      const rows = fixture.debugElement.queryAll(By.css('mat-row'));
      expect(rows.length).toBe(2);
    });

    it('should display user names', () => {
      setInputs();
      const firstRow = fixture.debugElement.query(By.css('mat-row'));
      expect(firstRow.nativeElement.textContent).toContain('Alice');
    });

    it('should show progress bar when loading', () => {
      setInputs([], true);
      const progressBar = fixture.debugElement.query(By.css('mat-progress-bar'));
      expect(progressBar).not.toBeNull();
    });

    it('should not show progress bar when not loading', () => {
      setInputs(mockUsers, false);
      const progressBar = fixture.debugElement.query(By.css('mat-progress-bar'));
      expect(progressBar).toBeNull();
    });
  });

  describe('Events', () => {
    it('should emit userSelected when row clicked', () => {
      setInputs();
      const selectedSpy = jest.fn();
      component.userSelected.subscribe(selectedSpy);

      const firstRow = fixture.debugElement.query(By.css('mat-row'));
      firstRow.nativeElement.click();

      expect(selectedSpy).toHaveBeenCalledWith('1');
    });

    it('should emit userEdit when edit button clicked', () => {
      setInputs();
      const editSpy = jest.fn();
      component.userEdit.subscribe(editSpy);

      const editButton = fixture.debugElement.query(
        By.css('[data-testid="edit-button"]')
      );
      editButton.nativeElement.click();

      expect(editSpy).toHaveBeenCalledWith(mockUsers[0]);
    });

    it('should emit userDelete when delete button clicked', () => {
      setInputs();
      const deleteSpy = jest.fn();
      component.userDelete.subscribe(deleteSpy);

      const deleteButtons = fixture.debugElement.queryAll(
        By.css('[data-testid="delete-button"]')
      );
      deleteButtons[1].nativeElement.click();

      expect(deleteSpy).toHaveBeenCalledWith(mockUsers[1]);
    });
  });

  describe('Selected State', () => {
    it('should apply selected class to selected row', () => {
      setInputs();
      fixture.componentRef.setInput('selectedUserId', '1');
      fixture.detectChanges();

      const rows = fixture.debugElement.queryAll(By.css('mat-row'));
      expect(rows[0].nativeElement.classList).toContain('selected');
      expect(rows[1].nativeElement.classList).not.toContain('selected');
    });
  });
});
```

### Testing Smart Component với Store Mock

```typescript
// features/user/user-list/user-list.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { UserListComponent } from './user-list.component';
import { UserStore } from '../store/user.store';
import { Router } from '@angular/router';
import { DialogService } from '@core/services/dialog.service';
import { of } from 'rxjs';
import { signal, computed } from '@angular/core';

describe('UserListComponent', () => {
  let component: UserListComponent;
  let fixture: ComponentFixture<UserListComponent>;
  let storeMock: Partial<InstanceType<typeof UserStore>>;
  let routerMock: jest.Mocked<Partial<Router>>;
  let dialogServiceMock: jest.Mocked<Partial<DialogService>>;

  const mockUsers: User[] = [
    { id: '1', displayName: 'Alice', email: 'alice@example.com', role: 'ADMIN', isActive: true, createdAt: '', updatedAt: '' },
  ];

  beforeEach(async () => {
    storeMock = {
      filteredUsers: signal(mockUsers),
      isLoading: signal(false),
      error: signal(null),
      isEmpty: signal(false),
      totalUsers: signal(1),
      selectedUserId: signal(null),
      filter: signal({ search: '', role: 'all' }),
      loadUsers: jest.fn().mockResolvedValue(undefined),
      deleteUser: jest.fn().mockResolvedValue(undefined),
      selectUser: jest.fn(),
      setFilter: jest.fn(),
      clearFilter: jest.fn(),
    };

    routerMock = { navigate: jest.fn() };
    dialogServiceMock = {
      confirmDelete: jest.fn().mockReturnValue(of(true)),
    };

    await TestBed.configureTestingModule({
      imports: [UserListComponent, NoopAnimationsModule],
      providers: [
        { provide: UserStore, useValue: storeMock },
        { provide: Router, useValue: routerMock },
        { provide: DialogService, useValue: dialogServiceMock },
      ],
    }).compileComponents();

    fixture = TestBed.createComponent(UserListComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should call loadUsers on init', () => {
    expect(storeMock.loadUsers).toHaveBeenCalledTimes(1);
  });

  it('should navigate to edit page when onEditUser called', () => {
    component['onEditUser'](mockUsers[0]);
    expect(routerMock.navigate).toHaveBeenCalledWith(['/users', '1', 'edit']);
  });

  it('should confirm before deleting user', async () => {
    await component['onDeleteUser'](mockUsers[0]);
    expect(dialogServiceMock.confirmDelete).toHaveBeenCalledWith('Alice');
    expect(storeMock.deleteUser).toHaveBeenCalledWith('1');
  });

  it('should not delete if user cancels confirmation', async () => {
    dialogServiceMock.confirmDelete!.mockReturnValue(of(false));
    await component['onDeleteUser'](mockUsers[0]);
    expect(storeMock.deleteUser).not.toHaveBeenCalled();
  });
});
```

---

## 10.5 Testing Pipes và Directives

```typescript
// shared/pipes/time-ago.pipe.spec.ts
describe('TimeAgoPipe', () => {
  let pipe: TimeAgoPipe;

  beforeEach(() => {
    pipe = new TimeAgoPipe();
  });

  it('should return "Vừa xong" for recent dates', () => {
    const recent = new Date(Date.now() - 30 * 1000).toISOString(); // 30s ago
    expect(pipe.transform(recent)).toBe('Vừa xong');
  });

  it('should return minutes ago', () => {
    const fiveMinAgo = new Date(Date.now() - 5 * 60 * 1000).toISOString();
    expect(pipe.transform(fiveMinAgo)).toBe('5 phút trước');
  });

  it('should return empty string for null', () => {
    expect(pipe.transform(null)).toBe('');
  });

  it('should accept Date object', () => {
    const oneHourAgo = new Date(Date.now() - 60 * 60 * 1000);
    expect(pipe.transform(oneHourAgo)).toBe('1 giờ trước');
  });
});
```

```typescript
// shared/directives/click-outside.directive.spec.ts
describe('ClickOutsideDirective', () => {
  @Component({
    template: `
      <div appClickOutside (clickOutside)="onClickOutside($event)">
        <button id="inside">Trong</button>
      </div>
      <button id="outside">Ngoài</button>
    `,
  })
  class TestComponent {
    onClickOutside = jest.fn();
  }

  let fixture: ComponentFixture<TestComponent>;
  let component: TestComponent;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [TestComponent, ClickOutsideDirective],
    }).compileComponents();

    fixture = TestBed.createComponent(TestComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should emit clickOutside when clicking outside', () => {
    const outsideButton = fixture.nativeElement.querySelector('#outside');
    outsideButton.click();
    expect(component.onClickOutside).toHaveBeenCalledTimes(1);
  });

  it('should not emit clickOutside when clicking inside', () => {
    const insideButton = fixture.nativeElement.querySelector('#inside');
    insideButton.click();
    expect(component.onClickOutside).not.toHaveBeenCalled();
  });
});
```

---

## 10.6 End-to-End Testing với Playwright

### Setup Playwright

```bash
npm install --save-dev @playwright/test
npx playwright install
```

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env['CI'],
  retries: process.env['CI'] ? 2 : 0,
  workers: process.env['CI'] ? 1 : undefined,

  use: {
    baseURL: 'http://localhost:4200',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] },
    },
  ],

  webServer: {
    command: 'ng serve',
    url: 'http://localhost:4200',
    reuseExistingServer: !process.env['CI'],
    timeout: 120 * 1000,
  },
});
```

### Page Object Model

```typescript
// e2e/pages/login.page.ts
import { Page, Locator, expect } from '@playwright/test';

export class LoginPage {
  private readonly emailInput: Locator;
  private readonly passwordInput: Locator;
  private readonly submitButton: Locator;
  private readonly errorMessage: Locator;

  constructor(private readonly page: Page) {
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Mật khẩu');
    this.submitButton = page.getByRole('button', { name: 'Đăng nhập' });
    this.errorMessage = page.getByRole('alert');
  }

  async goto(): Promise<void> {
    await this.page.goto('/auth/login');
  }

  async login(email: string, password: string): Promise<void> {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async expectError(message: string): Promise<void> {
    await expect(this.errorMessage).toBeVisible();
    await expect(this.errorMessage).toContainText(message);
  }

  async expectRedirectToDashboard(): Promise<void> {
    await expect(this.page).toHaveURL('/dashboard');
  }
}
```

```typescript
// e2e/pages/user-list.page.ts
export class UserListPage {
  constructor(private readonly page: Page) {}

  async goto(): Promise<void> {
    await this.page.goto('/users');
  }

  async searchUsers(query: string): Promise<void> {
    await this.page.getByPlaceholder('Tìm kiếm...').fill(query);
    await this.page.waitForResponse('**/api/users**');
  }

  async getUserCount(): Promise<number> {
    const rows = await this.page.locator('mat-row').count();
    return rows;
  }

  async deleteUser(userName: string): Promise<void> {
    const row = this.page.locator('mat-row', { hasText: userName });
    await row.getByTitle('Xóa').click();

    // Confirm dialog
    await this.page.getByRole('button', { name: 'Xóa' }).click();
    await this.page.waitForResponse('**/api/users/**');
  }

  async expectUserVisible(name: string): Promise<void> {
    await expect(this.page.locator('mat-row', { hasText: name })).toBeVisible();
  }

  async expectUserNotVisible(name: string): Promise<void> {
    await expect(this.page.locator('mat-row', { hasText: name })).not.toBeVisible();
  }
}
```

### E2E Tests

```typescript
// e2e/auth/login.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/login.page';

test.describe('Login Flow', () => {
  let loginPage: LoginPage;

  test.beforeEach(async ({ page }) => {
    loginPage = new LoginPage(page);
    await loginPage.goto();
  });

  test('should login successfully with valid credentials', async ({ page }) => {
    await loginPage.login('admin@example.com', 'password123');
    await loginPage.expectRedirectToDashboard();
  });

  test('should show error with invalid password', async () => {
    await loginPage.login('admin@example.com', 'wrongpassword');
    await loginPage.expectError('Sai email hoặc mật khẩu');
  });

  test('should validate required fields', async ({ page }) => {
    await page.getByRole('button', { name: 'Đăng nhập' }).click();

    await expect(page.getByText('Email là bắt buộc')).toBeVisible();
    await expect(page.getByText('Mật khẩu là bắt buộc')).toBeVisible();
  });

  test('should validate email format', async ({ page }) => {
    await page.getByLabel('Email').fill('not-an-email');
    await page.getByLabel('Email').blur();

    await expect(page.getByText('Email không đúng định dạng')).toBeVisible();
  });
});
```

```typescript
// e2e/users/user-management.spec.ts
import { test } from '@playwright/test';
import { LoginPage } from '../pages/login.page';
import { UserListPage } from '../pages/user-list.page';

test.describe('User Management', () => {
  test.beforeEach(async ({ page }) => {
    // Login trước mỗi test
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('admin@example.com', 'password123');
  });

  test('should display user list', async ({ page }) => {
    const userListPage = new UserListPage(page);
    await userListPage.goto();

    const count = await userListPage.getUserCount();
    expect(count).toBeGreaterThan(0);
  });

  test('should filter users by search', async ({ page }) => {
    const userListPage = new UserListPage(page);
    await userListPage.goto();

    const initialCount = await userListPage.getUserCount();
    await userListPage.searchUsers('alice');
    const filteredCount = await userListPage.getUserCount();

    expect(filteredCount).toBeLessThanOrEqual(initialCount);
    await userListPage.expectUserVisible('Alice');
  });

  test('should delete user and remove from list', async ({ page }) => {
    const userListPage = new UserListPage(page);
    await userListPage.goto();

    await userListPage.deleteUser('Test User');
    await userListPage.expectUserNotVisible('Test User');
  });
});
```

---

## Tổng kết chương

Testing Angular application theo ba tầng rõ ràng:

1. **Unit test Services**: Dùng `HttpClientTestingModule` và `HttpTestingController` — test mọi HTTP scenario bao gồm success, error, và request params.

2. **Unit test Components**: `TestBed.createComponent()` cho Dumb components (test rendering và events), mock Store/Service cho Smart components.

3. **Unit test Store**: Store là plain class — inject và gọi methods như một object thông thường. Test state transitions, computed values, và error handling.

4. **E2E với Playwright**: Page Object Model tách logic tương tác ra khỏi test spec — dễ maintain khi UI thay đổi. Test happy path, error cases, và validation flows.

5. **Mocking strategy**: Service tests dùng `HttpTestingController`; Component tests dùng `jest.fn()` cho service methods; E2E dùng real backend hoặc MSW (Mock Service Worker).

Chương tiếp theo sẽ đi vào **Authentication** — implement JWT auth đầy đủ với refresh token, guards, và interceptors trong production Angular app.
