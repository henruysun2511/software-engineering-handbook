# Chương 6: Reactive Forms & Validation

## Giới thiệu chương

Reactive Forms là một trong những điểm mạnh nhất của Angular — một form system được xây dựng sẵn trong framework, mạnh hơn React Hook Form về khả năng compose và type safety, đặc biệt hiệu quả với các form phức tạp nhiều cấp. Khác với Template-driven Forms (dùng `ngModel` và xử lý validation trong template), Reactive Forms quản lý toàn bộ form state trong component class — dễ test, dễ debug, và hoàn toàn predictable.

Chương này bao gồm toàn bộ từ cơ bản đến nâng cao: FormBuilder, validators, async validation, cross-field validation, dynamic forms, và tích hợp Zod để có schema validation type-safe.

---

## 6.1 Template-driven vs Reactive Forms

Angular cung cấp hai cách tiếp cận form:

| | Template-driven | Reactive |
|---|---|---|
| Logic ở đâu | Template (HTML) | Component class |
| Two-way binding | `[(ngModel)]` | Explicit `setValue`/`patchValue` |
| Validation | Directives trong template | Functions trong class |
| Unit test | Khó (cần DOM) | Dễ (pure class) |
| Dùng khi | Form đơn giản, ít logic | Form phức tạp, cần control cao |

**Quy tắc thực tế**: Luôn dùng Reactive Forms trong Angular enterprise. Template-driven chỉ phù hợp cho form 1-2 field cực đơn giản.

---

## 6.2 Reactive Forms Cơ Bản

### FormControl — Đơn vị cơ bản

`FormControl` đại diện cho một input field đơn lẻ. Nó track giá trị, trạng thái validation, và trạng thái tương tác (touched, dirty).

```typescript
import { FormControl, Validators } from '@angular/forms';

// Tạo FormControl với giá trị ban đầu và validators
const emailControl = new FormControl('', {
  validators: [Validators.required, Validators.email],
  nonNullable: true,  // Giá trị không bao giờ là null khi reset
});

// Đọc giá trị
emailControl.value;         // '' (kiểu string | null mặc định)
emailControl.getRawValue(); // '' (luôn đúng type kể cả khi disabled)

// Cập nhật giá trị
emailControl.setValue('user@example.com');
emailControl.patchValue('user');  // Với FormControl, setValue và patchValue giống nhau

// Trạng thái
emailControl.valid;    // boolean
emailControl.invalid;  // boolean
emailControl.touched;  // boolean — đã focus rồi blur
emailControl.dirty;    // boolean — đã thay đổi giá trị
emailControl.pristine; // boolean — chưa thay đổi

// Lỗi validation
emailControl.errors; // { required: true } | { email: true } | null

// Disable/enable
emailControl.disable();
emailControl.enable();
```

### FormGroup — Nhóm các Controls

`FormGroup` quản lý một tập hợp các `FormControl` liên quan:

```typescript
import { FormBuilder, Validators } from '@angular/forms';

@Component({
  selector: 'app-login-form',
  standalone: true,
  imports: [ReactiveFormsModule, MatFormFieldModule, MatInputModule, MatButtonModule],
  templateUrl: './login-form.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class LoginFormComponent {
  private readonly fb = inject(FormBuilder);
  private readonly authService = inject(AuthService);
  private readonly router = inject(Router);

  protected readonly isLoading = signal(false);
  protected readonly serverError = signal<string | null>(null);

  // FormBuilder.group — shorthand để tạo FormGroup
  protected readonly form = this.fb.group({
    email: this.fb.control('', {
      validators: [Validators.required, Validators.email],
      nonNullable: true,
    }),
    password: this.fb.control('', {
      validators: [Validators.required, Validators.minLength(8)],
      nonNullable: true,
    }),
    rememberMe: this.fb.control(false, { nonNullable: true }),
  });

  // TypeScript biết chính xác type của form value
  // form.value: { email: string; password: string; rememberMe: boolean }

  // Shortcut để truy cập controls
  get emailControl() { return this.form.controls.email; }
  get passwordControl() { return this.form.controls.password; }

  protected onSubmit(): void {
    if (this.form.invalid) {
      this.form.markAllAsTouched(); // Hiển thị tất cả lỗi
      return;
    }

    this.isLoading.set(true);
    this.serverError.set(null);

    const { email, password } = this.form.getRawValue();

    this.authService.login({ email, password }).subscribe({
      next: () => this.router.navigate(['/dashboard']),
      error: (err) => {
        this.serverError.set(err.message);
        this.isLoading.set(false);
      },
    });
  }
}
```

```html
<!-- login-form.component.html -->
<form [formGroup]="form" (ngSubmit)="onSubmit()">
  <mat-form-field>
    <mat-label>Email</mat-label>
    <input matInput type="email" formControlName="email" autocomplete="email" />

    @if (emailControl.hasError('required') && emailControl.touched) {
      <mat-error>Email là bắt buộc</mat-error>
    }
    @if (emailControl.hasError('email') && emailControl.touched) {
      <mat-error>Email không đúng định dạng</mat-error>
    }
  </mat-form-field>

  <mat-form-field>
    <mat-label>Mật khẩu</mat-label>
    <input matInput type="password" formControlName="password" autocomplete="current-password" />

    @if (passwordControl.hasError('required') && passwordControl.touched) {
      <mat-error>Mật khẩu là bắt buộc</mat-error>
    }
    @if (passwordControl.hasError('minlength') && passwordControl.touched) {
      <mat-error>
        Mật khẩu phải có ít nhất
        {{ passwordControl.getError('minlength')?.requiredLength }} ký tự
      </mat-error>
    }
  </mat-form-field>

  @if (serverError()) {
    <mat-error class="server-error">{{ serverError() }}</mat-error>
  }

  <button
    mat-flat-button
    color="primary"
    type="submit"
    [disabled]="isLoading()"
  >
    @if (isLoading()) {
      <mat-spinner diameter="20" />
    } @else {
      Đăng nhập
    }
  </button>
</form>
```

### FormArray — Danh sách Dynamic

`FormArray` quản lý danh sách controls có số lượng không cố định — khi cần thêm/xóa items lúc runtime:

```typescript
@Component({
  selector: 'app-skills-form',
  standalone: true,
  imports: [ReactiveFormsModule, MatFormFieldModule, MatInputModule, MatButtonModule, MatIconModule],
  templateUrl: './skills-form.component.html',
})
export class SkillsFormComponent {
  private readonly fb = inject(FormBuilder);

  protected readonly form = this.fb.group({
    name: this.fb.control('', { validators: [Validators.required], nonNullable: true }),
    skills: this.fb.array(
      [this.createSkillControl()],  // Bắt đầu với 1 skill
      { validators: [Validators.minLength(1)] }
    ),
  });

  // Typed accessor
  get skillsArray(): FormArray {
    return this.form.controls.skills;
  }

  get skillControls(): FormControl[] {
    return this.skillsArray.controls as FormControl[];
  }

  protected addSkill(): void {
    this.skillsArray.push(this.createSkillControl());
  }

  protected removeSkill(index: number): void {
    if (this.skillsArray.length > 1) {
      this.skillsArray.removeAt(index);
    }
  }

  private createSkillControl(): FormControl<string> {
    return this.fb.control('', {
      validators: [Validators.required, Validators.maxLength(50)],
      nonNullable: true,
    });
  }

  protected onSubmit(): void {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }
    console.log(this.form.getRawValue());
    // { name: string; skills: string[] }
  }
}
```

```html
<!-- skills-form.component.html -->
<form [formGroup]="form" (ngSubmit)="onSubmit()">
  <mat-form-field>
    <mat-label>Họ tên</mat-label>
    <input matInput formControlName="name" />
  </mat-form-field>

  <div formArrayName="skills">
    <h3>Kỹ năng</h3>

    @for (control of skillControls; track $index) {
      <div class="skill-row">
        <mat-form-field>
          <mat-label>Kỹ năng {{ $index + 1 }}</mat-label>
          <input matInput [formControlName]="$index" />
        </mat-form-field>

        <button
          mat-icon-button
          type="button"
          color="warn"
          (click)="removeSkill($index)"
          [disabled]="skillControls.length === 1"
        >
          <mat-icon>delete</mat-icon>
        </button>
      </div>
    }

    <button mat-stroked-button type="button" (click)="addSkill()">
      <mat-icon>add</mat-icon>
      Thêm kỹ năng
    </button>
  </div>

  <button mat-flat-button color="primary" type="submit">Lưu</button>
</form>
```

---

## 6.3 Validators — Xác thực dữ liệu

### Built-in Validators

```typescript
import { Validators } from '@angular/forms';

this.fb.group({
  username: ['', [
    Validators.required,
    Validators.minLength(3),
    Validators.maxLength(20),
    Validators.pattern(/^[a-zA-Z0-9_]+$/),  // Chỉ chấp nhận alphanumeric và _
  ]],
  age: [null, [
    Validators.required,
    Validators.min(18),
    Validators.max(120),
  ]],
  website: ['', [
    Validators.pattern(/^https?:\/\/.+/),
  ]],
});
```

### Custom Validators

Custom validator là một function nhận `AbstractControl` và trả về `ValidationErrors | null`:

```typescript
// shared/validators/custom.validators.ts
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

// Validator kiểm tra mật khẩu mạnh
export function strongPasswordValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const value = control.value as string;

    if (!value) return null; // Để Validators.required xử lý

    const hasUppercase = /[A-Z]/.test(value);
    const hasLowercase = /[a-z]/.test(value);
    const hasNumber = /\d/.test(value);
    const hasSpecial = /[!@#$%^&*]/.test(value);

    const errors: ValidationErrors = {};

    if (!hasUppercase) errors['noUppercase'] = true;
    if (!hasLowercase) errors['noLowercase'] = true;
    if (!hasNumber) errors['noNumber'] = true;
    if (!hasSpecial) errors['noSpecial'] = true;

    return Object.keys(errors).length > 0 ? errors : null;
  };
}

// Validator với tham số — factory function
export function forbiddenValuesValidator(
  forbiddenValues: string[]
): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const value = (control.value as string)?.toLowerCase();
    const isForbidden = forbiddenValues
      .map((v) => v.toLowerCase())
      .includes(value);

    return isForbidden ? { forbiddenValue: { value: control.value } } : null;
  };
}

// Validator kiểm tra URL hợp lệ
export function urlValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    if (!control.value) return null;

    try {
      new URL(control.value as string);
      return null;
    } catch {
      return { invalidUrl: true };
    }
  };
}
```

### Cross-field Validation — Validator ở FormGroup level

Validator ở `FormGroup` có thể truy cập nhiều controls cùng lúc — cần thiết cho các validation như "password và confirm password phải giống nhau":

```typescript
// shared/validators/cross-field.validators.ts

// Kiểm tra hai field phải có cùng giá trị
export function passwordMatchValidator(
  passwordField: string,
  confirmField: string
): ValidatorFn {
  return (group: AbstractControl): ValidationErrors | null => {
    const password = group.get(passwordField)?.value as string;
    const confirm = group.get(confirmField)?.value as string;

    if (!password || !confirm) return null;

    if (password !== confirm) {
      // Gán lỗi lên confirm control để hiển thị trong template
      group.get(confirmField)?.setErrors({ passwordMismatch: true });
      return { passwordMismatch: true };
    }

    // Xóa lỗi passwordMismatch nếu đã match
    const confirmErrors = group.get(confirmField)?.errors;
    if (confirmErrors) {
      const { passwordMismatch, ...remainingErrors } = confirmErrors;
      group
        .get(confirmField)
        ?.setErrors(
          Object.keys(remainingErrors).length > 0 ? remainingErrors : null
        );
    }

    return null;
  };
}

// Kiểm tra ngày bắt đầu phải trước ngày kết thúc
export function dateRangeValidator(
  startField: string,
  endField: string
): ValidatorFn {
  return (group: AbstractControl): ValidationErrors | null => {
    const start = group.get(startField)?.value as Date | null;
    const end = group.get(endField)?.value as Date | null;

    if (!start || !end) return null;

    return start < end ? null : { invalidDateRange: { start, end } };
  };
}
```

```typescript
// Sử dụng trong form
protected readonly registerForm = this.fb.group(
  {
    email: this.fb.control('', {
      validators: [Validators.required, Validators.email],
      nonNullable: true,
    }),
    password: this.fb.control('', {
      validators: [
        Validators.required,
        Validators.minLength(8),
        strongPasswordValidator(),
      ],
      nonNullable: true,
    }),
    confirmPassword: this.fb.control('', {
      validators: [Validators.required],
      nonNullable: true,
    }),
  },
  {
    validators: [passwordMatchValidator('password', 'confirmPassword')],
  }
);
```

### Async Validators — Validate với API

Async validators thực hiện validation thông qua HTTP request — ví dụ kiểm tra username đã tồn tại chưa:

```typescript
// shared/validators/async.validators.ts
import { AsyncValidatorFn } from '@angular/forms';
import { debounceTime, map, switchMap, first, catchError } from 'rxjs/operators';

export function usernameAvailableValidator(
  userService: UserService
): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    const username = control.value as string;

    if (!username || username.length < 3) {
      return of(null);
    }

    return of(username).pipe(
      debounceTime(400),          // Đợi user ngừng gõ
      switchMap((value) =>
        userService.checkUsernameAvailable(value)
      ),
      map((isAvailable) =>
        isAvailable ? null : { usernameTaken: true }
      ),
      catchError(() => of(null)), // Nếu API lỗi, bỏ qua validation
      first()                     // Complete sau lần emit đầu tiên
    );
  };
}
```

```typescript
// Sử dụng trong form
@Component({ ... })
export class RegisterComponent {
  private readonly userService = inject(UserService);
  private readonly fb = inject(FormBuilder);

  protected readonly form = this.fb.group({
    username: this.fb.control('', {
      validators: [
        Validators.required,
        Validators.minLength(3),
        Validators.maxLength(20),
      ],
      // Async validator — chỉ chạy khi sync validators pass
      asyncValidators: [usernameAvailableValidator(this.userService)],
      // Chỉ validate khi user blur (không phải khi đang gõ)
      updateOn: 'blur',
      nonNullable: true,
    }),
  });

  get usernameControl() { return this.form.controls.username; }
}
```

```html
<mat-form-field>
  <mat-label>Tên đăng nhập</mat-label>
  <input matInput formControlName="username" />

  <!-- pending = đang chạy async validator -->
  @if (usernameControl.pending) {
    <mat-spinner matSuffix diameter="20" />
  }

  @if (usernameControl.hasError('usernameTaken') && usernameControl.touched) {
    <mat-error>Tên đăng nhập đã được sử dụng</mat-error>
  }
</mat-form-field>
```

---

## 6.4 Error Message Strategy — Hiển thị lỗi chuẩn

Thay vì lặp lại logic hiển thị lỗi ở mọi template, tạo một component/pipe tái sử dụng:

```typescript
// shared/components/form-error/form-error.component.ts
const ERROR_MESSAGES: Record<string, (error: unknown) => string> = {
  required: () => 'Trường này là bắt buộc',
  email: () => 'Email không đúng định dạng',
  minlength: (error: { requiredLength: number }) =>
    `Tối thiểu ${error.requiredLength} ký tự`,
  maxlength: (error: { requiredLength: number }) =>
    `Tối đa ${error.requiredLength} ký tự`,
  min: (error: { min: number }) => `Giá trị tối thiểu là ${error.min}`,
  max: (error: { max: number }) => `Giá trị tối đa là ${error.max}`,
  pattern: () => 'Định dạng không hợp lệ',
  passwordMismatch: () => 'Mật khẩu xác nhận không khớp',
  usernameTaken: () => 'Tên đăng nhập đã được sử dụng',
  noUppercase: () => 'Cần ít nhất một chữ hoa',
  noLowercase: () => 'Cần ít nhất một chữ thường',
  noNumber: () => 'Cần ít nhất một chữ số',
  noSpecial: () => 'Cần ít nhất một ký tự đặc biệt (!@#$%^&*)',
  invalidUrl: () => 'URL không hợp lệ',
};

@Component({
  selector: 'app-form-error',
  standalone: true,
  template: `
    @if (errorMessage()) {
      <span class="form-error">{{ errorMessage() }}</span>
    }
  `,
  styles: [`
    .form-error {
      color: var(--mat-form-field-error-text-color);
      font-size: 12px;
    }
  `],
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class FormErrorComponent {
  readonly control = input.required<AbstractControl>();

  protected readonly errorMessage = computed(() => {
    const ctrl = this.control();
    if (!ctrl.errors || !ctrl.touched) return null;

    const firstErrorKey = Object.keys(ctrl.errors)[0];
    const errorValue = ctrl.errors[firstErrorKey];
    const messageFn = ERROR_MESSAGES[firstErrorKey];

    return messageFn
      ? messageFn(errorValue)
      : `Validation error: ${firstErrorKey}`;
  });
}
```

```html
<!-- Sử dụng trong template — gọn hơn nhiều -->
<mat-form-field>
  <mat-label>Email</mat-label>
  <input matInput formControlName="email" />
  <mat-error>
    <app-form-error [control]="form.controls.email" />
  </mat-error>
</mat-form-field>
```

---

## 6.5 Kết hợp Zod với Reactive Forms

Zod có thể hoạt động song song với Reactive Forms theo hai cách: validate toàn bộ form value trước khi submit, hoặc tạo Angular validator từ Zod schema.

### Validate Form Value với Zod khi Submit

```typescript
// models/user-form.schema.ts
import { z } from 'zod';

export const CreateUserFormSchema = z.object({
  email: z.string().email('Email không hợp lệ'),
  displayName: z.string().min(2, 'Tên phải có ít nhất 2 ký tự').max(100),
  role: z.enum(['ADMIN', 'EDITOR', 'VIEWER']),
  password: z
    .string()
    .min(8, 'Mật khẩu phải có ít nhất 8 ký tự')
    .regex(/[A-Z]/, 'Cần ít nhất một chữ hoa')
    .regex(/[0-9]/, 'Cần ít nhất một chữ số'),
  confirmPassword: z.string(),
  sendWelcomeEmail: z.boolean().default(true),
}).refine(
  (data) => data.password === data.confirmPassword,
  {
    message: 'Mật khẩu xác nhận không khớp',
    path: ['confirmPassword'],  // Lỗi gán vào field cụ thể
  }
);

export type CreateUserFormData = z.infer<typeof CreateUserFormSchema>;
```

```typescript
// user-form.component.ts
@Component({ ... })
export class UserFormComponent {
  private readonly fb = inject(FormBuilder);
  private readonly userService = inject(UserService);

  protected readonly form = this.fb.group({
    email: this.fb.control('', { validators: [Validators.required], nonNullable: true }),
    displayName: this.fb.control('', { validators: [Validators.required], nonNullable: true }),
    role: this.fb.control<UserRole>('VIEWER', { nonNullable: true }),
    password: this.fb.control('', { validators: [Validators.required], nonNullable: true }),
    confirmPassword: this.fb.control('', { validators: [Validators.required], nonNullable: true }),
    sendWelcomeEmail: this.fb.control(true, { nonNullable: true }),
  });

  protected readonly formErrors = signal<Record<string, string>>({});

  protected onSubmit(): void {
    // Bước 1: Angular validation
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }

    // Bước 2: Zod validation — schema-level validation
    const result = CreateUserFormSchema.safeParse(this.form.getRawValue());

    if (!result.success) {
      // Map Zod errors vào form errors
      const errors: Record<string, string> = {};

      result.error.issues.forEach((issue) => {
        const path = issue.path.join('.');
        errors[path] = issue.message;
      });

      this.formErrors.set(errors);
      return;
    }

    // result.data đã được validate và có đúng type
    const { confirmPassword, ...createDto } = result.data;

    this.userService.createUser(createDto).subscribe({
      next: () => this.router.navigate(['/users']),
    });
  }
}
```

### Tạo Angular Validator từ Zod Schema

```typescript
// shared/validators/zod.validators.ts
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';
import { z } from 'zod';

/**
 * Tạo Angular ValidatorFn từ Zod schema.
 * Validate toàn bộ FormGroup value bằng Zod và map errors về Angular format.
 */
export function zodValidator<T extends z.ZodTypeAny>(
  schema: T
): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const result = schema.safeParse(control.value);

    if (result.success) return null;

    const errors: ValidationErrors = {};

    result.error.issues.forEach((issue) => {
      const key = issue.path.join('.') || 'zodError';
      errors[key] = issue.message;
    });

    return errors;
  };
}

/**
 * Tạo ValidatorFn cho một field đơn lẻ
 */
export function zodFieldValidator<T extends z.ZodTypeAny>(
  schema: T
): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const result = schema.safeParse(control.value);

    if (result.success) return null;

    return {
      zodError: result.error.issues[0]?.message ?? 'Giá trị không hợp lệ',
    };
  };
}
```

```typescript
// Sử dụng — validate từng field với Zod schema
const emailSchema = z.string().email('Email không đúng định dạng');
const passwordSchema = z
  .string()
  .min(8, 'Ít nhất 8 ký tự')
  .regex(/[A-Z]/, 'Cần chữ hoa');

protected readonly form = this.fb.group({
  email: this.fb.control('', {
    validators: [
      Validators.required,
      zodFieldValidator(emailSchema),
    ],
    nonNullable: true,
  }),
  password: this.fb.control('', {
    validators: [
      Validators.required,
      zodFieldValidator(passwordSchema),
    ],
    nonNullable: true,
  }),
});
```

---

## 6.6 Reactive Forms Nâng cao

### Form phức tạp nhiều cấp — Nested FormGroup

```typescript
@Component({ ... })
export class CheckoutFormComponent {
  private readonly fb = inject(FormBuilder);

  protected readonly form = this.fb.group({
    // Personal info
    personal: this.fb.group({
      firstName: this.fb.control('', { validators: [Validators.required], nonNullable: true }),
      lastName: this.fb.control('', { validators: [Validators.required], nonNullable: true }),
      email: this.fb.control('', {
        validators: [Validators.required, Validators.email],
        nonNullable: true,
      }),
      phone: this.fb.control('', { nonNullable: true }),
    }),

    // Shipping address
    shippingAddress: this.fb.group({
      street: this.fb.control('', { validators: [Validators.required], nonNullable: true }),
      city: this.fb.control('', { validators: [Validators.required], nonNullable: true }),
      province: this.fb.control('', { validators: [Validators.required], nonNullable: true }),
      postalCode: this.fb.control('', { nonNullable: true }),
    }),

    // Same as shipping
    useSameAddress: this.fb.control(true, { nonNullable: true }),

    // Billing address — conditional
    billingAddress: this.fb.group({
      street: this.fb.control('', { nonNullable: true }),
      city: this.fb.control('', { nonNullable: true }),
      province: this.fb.control('', { nonNullable: true }),
      postalCode: this.fb.control('', { nonNullable: true }),
    }),

    // Payment
    paymentMethod: this.fb.control<'credit_card' | 'bank_transfer'>('credit_card', {
      validators: [Validators.required],
      nonNullable: true,
    }),
  });

  constructor() {
    // Dynamic validation — billing address bắt buộc khi không dùng same address
    this.form.controls.useSameAddress.valueChanges.subscribe((useSame) => {
      const billingGroup = this.form.controls.billingAddress;

      if (useSame) {
        billingGroup.disable();
        billingGroup.clearValidators();
      } else {
        billingGroup.enable();
        Object.values(billingGroup.controls).forEach((ctrl) => {
          ctrl.setValidators([Validators.required]);
          ctrl.updateValueAndValidity();
        });
      }

      billingGroup.updateValueAndValidity();
    });
  }

  protected onSubmit(): void {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }

    // getRawValue() lấy giá trị kể cả disabled controls
    const formValue = this.form.getRawValue();
    console.log(formValue);
  }
}
```

### ControlValueAccessor — Custom Form Control

`ControlValueAccessor` là interface cho phép tạo custom component hoạt động như một native form control — có thể dùng với `formControlName` và `ngModel`:

```typescript
// shared/components/phone-input/phone-input.component.ts
@Component({
  selector: 'app-phone-input',
  standalone: true,
  imports: [ReactiveFormsModule, MatFormFieldModule, MatInputModule, MatSelectModule],
  template: `
    <mat-form-field>
      <mat-label>Số điện thoại</mat-label>
      <div class="phone-input">
        <mat-select [formControl]="countryCodeControl" class="country-code">
          @for (country of countries; track country.code) {
            <mat-option [value]="country.dialCode">
              {{ country.flag }} +{{ country.dialCode }}
            </mat-option>
          }
        </mat-select>
        <input
          matInput
          [formControl]="numberControl"
          type="tel"
          placeholder="912 345 678"
        />
      </div>
    </mat-form-field>
  `,
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => PhoneInputComponent),
      multi: true,
    },
    {
      provide: NG_VALIDATORS,
      useExisting: forwardRef(() => PhoneInputComponent),
      multi: true,
    },
  ],
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class PhoneInputComponent implements ControlValueAccessor, Validator {
  private readonly fb = inject(FormBuilder);

  readonly countries = COUNTRIES;

  readonly countryCodeControl = this.fb.control('84', { nonNullable: true });
  readonly numberControl = this.fb.control('', { nonNullable: true });

  private onChange: (value: string) => void = () => {};
  private onTouched: () => void = () => {};

  constructor() {
    // Combine hai controls thành một giá trị duy nhất
    merge(
      this.countryCodeControl.valueChanges,
      this.numberControl.valueChanges
    ).subscribe(() => {
      const fullNumber = `+${this.countryCodeControl.value}${this.numberControl.value}`;
      this.onChange(fullNumber);
    });
  }

  // ControlValueAccessor interface
  writeValue(value: string | null): void {
    if (!value) {
      this.numberControl.reset('');
      return;
    }

    // Parse "+84912345678" thành country code và number
    const match = value.match(/^\+(\d{1,3})(\d+)$/);
    if (match) {
      this.countryCodeControl.setValue(match[1], { emitEvent: false });
      this.numberControl.setValue(match[2], { emitEvent: false });
    }
  }

  registerOnChange(fn: (value: string) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    if (isDisabled) {
      this.countryCodeControl.disable();
      this.numberControl.disable();
    } else {
      this.countryCodeControl.enable();
      this.numberControl.enable();
    }
  }

  // Validator interface
  validate(control: AbstractControl): ValidationErrors | null {
    const value = control.value as string;
    if (!value) return null;

    const phoneRegex = /^\+\d{1,3}\d{6,14}$/;
    return phoneRegex.test(value) ? null : { invalidPhone: true };
  }
}
```

```html
<!-- Sử dụng như native form control -->
<form [formGroup]="form">
  <app-phone-input formControlName="phone" />
</form>
```

---

## 6.7 Form State Persistence — Lưu Draft

Pattern lưu form state vào localStorage để không mất dữ liệu khi user refresh:

```typescript
// shared/services/form-persistence.service.ts
@Injectable({ providedIn: 'root' })
export class FormPersistenceService {
  saveFormState(key: string, value: unknown): void {
    try {
      localStorage.setItem(key, JSON.stringify(value));
    } catch {
      // Bỏ qua nếu localStorage đầy
    }
  }

  loadFormState<T>(key: string): T | null {
    try {
      const stored = localStorage.getItem(key);
      return stored ? (JSON.parse(stored) as T) : null;
    } catch {
      return null;
    }
  }

  clearFormState(key: string): void {
    localStorage.removeItem(key);
  }
}

// Sử dụng trong form component
@Component({ ... })
export class LongFormComponent implements OnInit, OnDestroy {
  private readonly fb = inject(FormBuilder);
  private readonly formPersistence = inject(FormPersistenceService);
  private readonly destroyRef = inject(DestroyRef);

  private readonly STORAGE_KEY = 'long-form-draft';

  protected readonly form = this.fb.group({
    title: this.fb.control('', { nonNullable: true }),
    content: this.fb.control('', { nonNullable: true }),
    tags: this.fb.control<string[]>([], { nonNullable: true }),
  });

  ngOnInit(): void {
    // Restore từ localStorage
    const saved = this.formPersistence.loadFormState<typeof this.form.value>(
      this.STORAGE_KEY
    );
    if (saved) {
      this.form.patchValue(saved);
    }

    // Auto-save mỗi 30 giây và khi form thay đổi
    this.form.valueChanges.pipe(
      debounceTime(2000),  // Chờ 2 giây sau lần thay đổi cuối
      takeUntilDestroyed(this.destroyRef)
    ).subscribe((value) => {
      this.formPersistence.saveFormState(this.STORAGE_KEY, value);
    });
  }

  protected onSubmit(): void {
    if (this.form.invalid) return;

    // Xóa draft sau khi submit thành công
    this.submitService.submit(this.form.getRawValue()).subscribe({
      next: () => {
        this.formPersistence.clearFormState(this.STORAGE_KEY);
        this.router.navigate(['/success']);
      },
    });
  }
}
```

---

## Tổng kết chương

Reactive Forms là form system đầy đủ nhất trong các frontend framework. Những điểm cốt lõi:

1. **FormGroup + FormControl + FormArray** là ba building block cơ bản. Chọn đúng loại cho từng trường hợp — FormArray cho danh sách dynamic, FormGroup cho nhóm fields liên quan.

2. **Custom validators** là pure functions — dễ test và tái sử dụng. Group-level validators cho phép cross-field validation (password match, date range).

3. **Async validators** với `debounceTime` và `first()` để tránh API call quá nhiều. Dùng `updateOn: 'blur'` để chỉ validate khi user blur.

4. **Zod kết hợp Reactive Forms** theo hai cách: validate form value trước submit, hoặc tạo Angular validator từ Zod schema. Zod là single source of truth cho shape và business rules.

5. **ControlValueAccessor** để tạo custom form control tái sử dụng — thành phần mạnh nhất của Reactive Forms ecosystem.

Chương tiếp theo sẽ đi vào **NgRx Signals Store** — hệ thống state management chính thức cho Angular, nơi Signals được tích hợp sâu vào kiến trúc state toàn ứng dụng.
