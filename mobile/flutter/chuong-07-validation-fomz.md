# Chương 7: Validation với Formz

---

## 7.1. Tại Sao Cần Validation Chuyên Biệt?

### 7.1.1. Vấn Đề Của Inline Validator

Flutter cung cấp sẵn tham số `validator` trong `TextFormField`. Cách tiếp cận này đơn giản nhưng nhanh chóng trở nên rối rắm khi form phức tạp lên.

```dart
// ❌ ANTIPATTERN — validator inline, không tái sử dụng được
TextFormField(
  validator: (value) {
    if (value == null || value.isEmpty) return 'Không được để trống';
    if (!RegExp(r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$').hasMatch(value)) {
      return 'Email không hợp lệ';
    }
    return null;
  },
)

// Vấn đề:
// 1. Logic validator bị nhúng trong UI — không test được
// 2. Không tái sử dụng — phải copy-paste sang form khác
// 3. Không có trạng thái validation — không biết field đang valid hay invalid
// 4. Khó combine nhiều điều kiện phức tạp
// 5. Validation chỉ chạy khi submit, không real-time được
```

### 7.1.2. Formz — Validation Như Một State Machine

`Formz` là thư viện nhỏ gọn của Flutter team, mô hình hóa mỗi form input như một **state machine** với ba trạng thái rõ ràng:

```
pure       → Người dùng chưa tương tác
valid      → Đã tương tác, dữ liệu hợp lệ
invalid    → Đã tương tác, dữ liệu không hợp lệ
```

Mỗi input là một class riêng biệt — có thể test độc lập, tái sử dụng toàn app, kết hợp dễ dàng với Riverpod.

---

## 7.2. Cài Đặt

```yaml
# pubspec.yaml
dependencies:
  formz: ^0.7.0
```

---

## 7.3. Tạo FormzInput

### 7.3.1. Cấu Trúc Cơ Bản

```dart
// lib/core/validators/form_inputs.dart

import 'package:formz/formz.dart';

// Bước 1: Định nghĩa enum cho các loại lỗi
enum EmailValidationError {
  empty,
  invalidFormat,
}

// Bước 2: Tạo class kế thừa FormzInput<ValueType, ErrorType>
class EmailInput extends FormzInput<String, EmailValidationError> {
  // pure: constructor khi field chưa được chạm vào
  const EmailInput.pure([super.value = '']) : super.pure();

  // dirty: constructor khi field đã được chỉnh sửa
  const EmailInput.dirty([super.value = '']) : super.dirty();

  // validator: logic kiểm tra, trả về null nếu hợp lệ
  @override
  EmailValidationError? validator(String value) {
    if (value.isEmpty) return EmailValidationError.empty;
    if (!_emailRegex.hasMatch(value)) return EmailValidationError.invalidFormat;
    return null; // Hợp lệ
  }

  static final _emailRegex = RegExp(
    r'^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$',
  );
}

// Extension tiện lợi — chuyển error enum thành message hiển thị
extension EmailValidationErrorX on EmailValidationError {
  String get message => switch (this) {
        EmailValidationError.empty => 'Email không được để trống',
        EmailValidationError.invalidFormat => 'Định dạng email không hợp lệ',
      };
}
```

### 7.3.2. Bộ FormzInput Hoàn Chỉnh Cho Dự Án

```dart
// lib/core/validators/form_inputs.dart

// ─── PASSWORD ─────────────────────────────────────────────
enum PasswordValidationError { empty, tooShort, noUppercase, noDigit, noSpecial }

class PasswordInput extends FormzInput<String, PasswordValidationError> {
  const PasswordInput.pure([super.value = '']) : super.pure();
  const PasswordInput.dirty([super.value = '']) : super.dirty();

  static const _minLength = 8;

  @override
  PasswordValidationError? validator(String value) {
    if (value.isEmpty) return PasswordValidationError.empty;
    if (value.length < _minLength) return PasswordValidationError.tooShort;
    if (!value.contains(RegExp(r'[A-Z]'))) return PasswordValidationError.noUppercase;
    if (!value.contains(RegExp(r'[0-9]'))) return PasswordValidationError.noDigit;
    if (!value.contains(RegExp(r'[!@#$%^&*(),.?":{}|<>]'))) {
      return PasswordValidationError.noSpecial;
    }
    return null;
  }

  // Tính độ mạnh mật khẩu (0-4)
  int get strength {
    int score = 0;
    if (value.length >= _minLength) score++;
    if (value.contains(RegExp(r'[A-Z]'))) score++;
    if (value.contains(RegExp(r'[0-9]'))) score++;
    if (value.contains(RegExp(r'[!@#$%^&*(),.?":{}|<>]'))) score++;
    return score;
  }
}

extension PasswordValidationErrorX on PasswordValidationError {
  String get message => switch (this) {
        PasswordValidationError.empty => 'Mật khẩu không được để trống',
        PasswordValidationError.tooShort => 'Mật khẩu tối thiểu 8 ký tự',
        PasswordValidationError.noUppercase => 'Phải có ít nhất 1 chữ hoa',
        PasswordValidationError.noDigit => 'Phải có ít nhất 1 chữ số',
        PasswordValidationError.noSpecial => 'Phải có ít nhất 1 ký tự đặc biệt',
      };
}

// ─── CONFIRM PASSWORD ──────────────────────────────────────
enum ConfirmPasswordError { empty, mismatch }

class ConfirmPasswordInput extends FormzInput<String, ConfirmPasswordError> {
  // Nhận password gốc để so sánh
  const ConfirmPasswordInput.pure({this.password = ''})
      : super.pure('');
  const ConfirmPasswordInput.dirty({required this.password, String value = ''})
      : super.dirty(value);

  final String password;

  @override
  ConfirmPasswordError? validator(String value) {
    if (value.isEmpty) return ConfirmPasswordError.empty;
    if (value != password) return ConfirmPasswordError.mismatch;
    return null;
  }
}

extension ConfirmPasswordErrorX on ConfirmPasswordError {
  String get message => switch (this) {
        ConfirmPasswordError.empty => 'Vui lòng xác nhận mật khẩu',
        ConfirmPasswordError.mismatch => 'Mật khẩu không khớp',
      };
}

// ─── PHONE NUMBER ──────────────────────────────────────────
enum PhoneValidationError { empty, invalidFormat }

class PhoneInput extends FormzInput<String, PhoneValidationError> {
  const PhoneInput.pure([super.value = '']) : super.pure();
  const PhoneInput.dirty([super.value = '']) : super.dirty();

  // Validate số điện thoại Việt Nam
  static final _vnPhoneRegex = RegExp(r'^(0|\+84)(3|5|7|8|9)[0-9]{8}$');

  @override
  PhoneValidationError? validator(String value) {
    final cleaned = value.replaceAll(RegExp(r'\s|-'), '');
    if (cleaned.isEmpty) return PhoneValidationError.empty;
    if (!_vnPhoneRegex.hasMatch(cleaned)) return PhoneValidationError.invalidFormat;
    return null;
  }
}

extension PhoneValidationErrorX on PhoneValidationError {
  String get message => switch (this) {
        PhoneValidationError.empty => 'Số điện thoại không được để trống',
        PhoneValidationError.invalidFormat => 'Số điện thoại không hợp lệ',
      };
}

// ─── REQUIRED TEXT ─────────────────────────────────────────
enum RequiredTextError { empty, tooShort, tooLong }

class RequiredTextInput extends FormzInput<String, RequiredTextError> {
  const RequiredTextInput.pure({
    this.minLength = 1,
    this.maxLength = 255,
  }) : super.pure('');

  const RequiredTextInput.dirty(
    String value, {
    this.minLength = 1,
    this.maxLength = 255,
  }) : super.dirty(value);

  final int minLength;
  final int maxLength;

  @override
  RequiredTextError? validator(String value) {
    if (value.trim().isEmpty) return RequiredTextError.empty;
    if (value.trim().length < minLength) return RequiredTextError.tooShort;
    if (value.length > maxLength) return RequiredTextError.tooLong;
    return null;
  }
}

extension RequiredTextErrorX on RequiredTextError {
  String message(int min, int max) => switch (this) {
        RequiredTextError.empty => 'Trường này không được để trống',
        RequiredTextError.tooShort => 'Tối thiểu $min ký tự',
        RequiredTextError.tooLong => 'Tối đa $max ký tự',
      };
}

// ─── PRICE INPUT ───────────────────────────────────────────
enum PriceValidationError { empty, invalid, tooLow, tooHigh }

class PriceInput extends FormzInput<String, PriceValidationError> {
  const PriceInput.pure() : super.pure('');
  const PriceInput.dirty([super.value = '']) : super.dirty();

  static const _minPrice = 1000.0;
  static const _maxPrice = 999999999.0;

  double? get parsedValue => double.tryParse(value.replaceAll(',', ''));

  @override
  PriceValidationError? validator(String value) {
    if (value.isEmpty) return PriceValidationError.empty;
    final price = double.tryParse(value.replaceAll(',', ''));
    if (price == null) return PriceValidationError.invalid;
    if (price < _minPrice) return PriceValidationError.tooLow;
    if (price > _maxPrice) return PriceValidationError.tooHigh;
    return null;
  }
}
```

---

## 7.4. Form State Với Riverpod

Kết hợp Formz với Riverpod để quản lý toàn bộ trạng thái form, bao gồm validation, loading state và submit.

### 7.4.1. Login Form

```dart
// lib/features/auth/providers/login_form_provider.dart

import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'package:formz/formz.dart';

part 'login_form_provider.freezed.dart';
part 'login_form_provider.g.dart';

@freezed
class LoginFormState with _$LoginFormState {
  const factory LoginFormState({
    @Default(EmailInput.pure()) EmailInput email,
    @Default(PasswordInput.pure()) PasswordInput password,
    @Default(FormzSubmissionStatus.initial) FormzSubmissionStatus status,
    String? errorMessage,
  }) = _LoginFormState;

  const LoginFormState._();

  // Form hợp lệ khi tất cả inputs đều valid
  bool get isValid => Formz.validate([email, password]);

  // Có thể submit khi form valid và không đang loading
  bool get canSubmit => isValid && !status.isInProgress;
}

@riverpod
class LoginForm extends _$LoginForm {
  @override
  LoginFormState build() => const LoginFormState();

  // Cập nhật email — chuyển sang dirty khi user nhập
  void emailChanged(String value) {
    state = state.copyWith(
      email: EmailInput.dirty(value),
      status: FormzSubmissionStatus.initial,
      errorMessage: null,
    );
  }

  // Cập nhật password
  void passwordChanged(String value) {
    state = state.copyWith(
      password: PasswordInput.dirty(value),
      status: FormzSubmissionStatus.initial,
      errorMessage: null,
    );
  }

  // Submit form
  Future<void> submit() async {
    // Validate toàn bộ form (chuyển tất cả sang dirty)
    final email = EmailInput.dirty(state.email.value);
    final password = PasswordInput.dirty(state.password.value);

    state = state.copyWith(
      email: email,
      password: password,
      status: FormzSubmissionStatus.inProgress,
      errorMessage: null,
    );

    // Không submit nếu invalid
    if (!state.isValid) {
      state = state.copyWith(status: FormzSubmissionStatus.initial);
      return;
    }

    final result = await ref.read(authRepositoryProvider).login(
      email: state.email.value,
      password: state.password.value,
    );

    state = switch (result) {
      Success(:final data) => state.copyWith(
          status: FormzSubmissionStatus.success,
        ),
      Failure(:final error) => state.copyWith(
          status: FormzSubmissionStatus.failure,
          errorMessage: error.message,
        ),
    };

    if (state.status.isSuccess) {
      ref.read(authProvider.notifier).onLoginSuccess(result.valueOrNull!);
    }
  }

  void reset() => state = const LoginFormState();
}
```

### 7.4.2. Register Form — Form Phức Tạp Hơn

```dart
// lib/features/auth/providers/register_form_provider.dart

@freezed
class RegisterFormState with _$RegisterFormState {
  const factory RegisterFormState({
    @Default(RequiredTextInput.pure()) RequiredTextInput fullName,
    @Default(EmailInput.pure()) EmailInput email,
    @Default(PhoneInput.pure()) PhoneInput phone,
    @Default(PasswordInput.pure()) PasswordInput password,
    @Default(ConfirmPasswordInput.pure()) ConfirmPasswordInput confirmPassword,
    @Default(false) bool acceptedTerms,
    @Default(FormzSubmissionStatus.initial) FormzSubmissionStatus status,
    String? errorMessage,
  }) = _RegisterFormState;

  const RegisterFormState._();

  bool get isValid =>
      Formz.validate([fullName, email, phone, password, confirmPassword]) &&
      acceptedTerms;

  bool get canSubmit => isValid && !status.isInProgress;
}

@riverpod
class RegisterForm extends _$RegisterForm {
  @override
  RegisterFormState build() => const RegisterFormState();

  void fullNameChanged(String value) => state = state.copyWith(
        fullName: RequiredTextInput.dirty(value, minLength: 2),
        errorMessage: null,
      );

  void emailChanged(String value) => state = state.copyWith(
        email: EmailInput.dirty(value),
        errorMessage: null,
      );

  void phoneChanged(String value) => state = state.copyWith(
        phone: PhoneInput.dirty(value),
        errorMessage: null,
      );

  void passwordChanged(String value) {
    state = state.copyWith(
      password: PasswordInput.dirty(value),
      // Cập nhật lại confirmPassword với password mới để validate lại
      confirmPassword: state.confirmPassword.isPure
          ? state.confirmPassword
          : ConfirmPasswordInput.dirty(
              password: value,
              value: state.confirmPassword.value,
            ),
      errorMessage: null,
    );
  }

  void confirmPasswordChanged(String value) => state = state.copyWith(
        confirmPassword: ConfirmPasswordInput.dirty(
          password: state.password.value,
          value: value,
        ),
        errorMessage: null,
      );

  void toggleTerms(bool? value) => state = state.copyWith(
        acceptedTerms: value ?? false,
      );

  Future<void> submit() async {
    // Validate tất cả fields
    state = state.copyWith(
      fullName: RequiredTextInput.dirty(state.fullName.value, minLength: 2),
      email: EmailInput.dirty(state.email.value),
      phone: PhoneInput.dirty(state.phone.value),
      password: PasswordInput.dirty(state.password.value),
      confirmPassword: ConfirmPasswordInput.dirty(
        password: state.password.value,
        value: state.confirmPassword.value,
      ),
      status: FormzSubmissionStatus.inProgress,
    );

    if (!state.isValid) {
      state = state.copyWith(status: FormzSubmissionStatus.initial);
      return;
    }

    final result = await ref.read(authRepositoryProvider).register(
      fullName: state.fullName.value,
      email: state.email.value,
      phone: state.phone.value,
      password: state.password.value,
    );

    state = switch (result) {
      Success() => state.copyWith(status: FormzSubmissionStatus.success),
      Failure(:final error) => state.copyWith(
          status: FormzSubmissionStatus.failure,
          errorMessage: error.message,
        ),
    };
  }
}
```

---

## 7.5. Xây Dựng UI Form

### 7.5.1. Custom TextField Widget

```dart
// lib/core/widgets/app_text_field.dart
// Widget tái sử dụng, nhận error message từ bên ngoài

class AppTextField extends StatefulWidget {
  const AppTextField({
    super.key,
    required this.label,
    this.hint,
    this.errorText,        // Nhận error message từ Formz state
    this.onChanged,
    this.onSubmitted,
    this.keyboardType,
    this.textInputAction,
    this.isPassword = false,
    this.prefixIcon,
    this.suffixIcon,
    this.enabled = true,
    this.autofocus = false,
    this.maxLines = 1,
    this.maxLength,
    this.initialValue,
    this.focusNode,
  });

  final String label;
  final String? hint;
  final String? errorText;
  final ValueChanged<String>? onChanged;
  final ValueChanged<String>? onSubmitted;
  final TextInputType? keyboardType;
  final TextInputAction? textInputAction;
  final bool isPassword;
  final Widget? prefixIcon;
  final Widget? suffixIcon;
  final bool enabled;
  final bool autofocus;
  final int? maxLines;
  final int? maxLength;
  final String? initialValue;
  final FocusNode? focusNode;

  @override
  State<AppTextField> createState() => _AppTextFieldState();
}

class _AppTextFieldState extends State<AppTextField> {
  late final TextEditingController _controller;
  bool _obscureText = true;

  @override
  void initState() {
    super.initState();
    _controller = TextEditingController(text: widget.initialValue);
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);

    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      mainAxisSize: MainAxisSize.min,
      children: [
        // Label
        Text(
          widget.label,
          style: theme.textTheme.labelLarge?.copyWith(
            color: theme.colorScheme.onSurface,
          ),
        ),
        const SizedBox(height: 6),

        // TextField
        TextField(
          controller: _controller,
          focusNode: widget.focusNode,
          onChanged: widget.onChanged,
          onSubmitted: widget.onSubmitted,
          keyboardType: widget.keyboardType,
          textInputAction: widget.textInputAction,
          obscureText: widget.isPassword && _obscureText,
          enabled: widget.enabled,
          autofocus: widget.autofocus,
          maxLines: widget.isPassword ? 1 : widget.maxLines,
          maxLength: widget.maxLength,
          decoration: InputDecoration(
            hintText: widget.hint,
            errorText: widget.errorText,
            prefixIcon: widget.prefixIcon,
            // Toggle show/hide password
            suffixIcon: widget.isPassword
                ? IconButton(
                    icon: Icon(
                      _obscureText
                          ? Icons.visibility_outlined
                          : Icons.visibility_off_outlined,
                    ),
                    onPressed: () =>
                        setState(() => _obscureText = !_obscureText),
                  )
                : widget.suffixIcon,
          ),
        ),
      ],
    );
  }
}
```

### 7.5.2. Login Screen Hoàn Chỉnh

```dart
// lib/features/auth/screens/login_screen.dart

class LoginScreen extends ConsumerWidget {
  const LoginScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // Lắng nghe submit status để navigate hoặc show error
    ref.listen(
      loginFormProvider.select((s) => s.status),
      (_, status) {
        if (status.isSuccess) {
          context.go('/');
        }
      },
    );

    final formState = ref.watch(loginFormProvider);
    final notifier = ref.read(loginFormProvider.notifier);

    return Scaffold(
      body: SafeArea(
        child: SingleChildScrollView(
          padding: const EdgeInsets.all(24),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.stretch,
            children: [
              const SizedBox(height: 48),

              // Logo / Header
              Text(
                'Chào mừng trở lại',
                style: context.textTheme.headlineMedium,
                textAlign: TextAlign.center,
              ),
              const SizedBox(height: 8),
              Text(
                'Đăng nhập để tiếp tục',
                style: context.textTheme.bodyMedium?.copyWith(
                  color: context.colorScheme.onSurfaceVariant,
                ),
                textAlign: TextAlign.center,
              ),
              const SizedBox(height: 40),

              // Email field
              AppTextField(
                label: 'Email',
                hint: 'example@email.com',
                keyboardType: TextInputType.emailAddress,
                textInputAction: TextInputAction.next,
                prefixIcon: const Icon(Icons.email_outlined),
                // errorText: chỉ hiển thị khi field đã dirty (đã chạm vào)
                errorText: formState.email.displayError?.message,
                onChanged: notifier.emailChanged,
              ),
              const SizedBox(height: 16),

              // Password field
              AppTextField(
                label: 'Mật khẩu',
                hint: '••••••••',
                isPassword: true,
                textInputAction: TextInputAction.done,
                prefixIcon: const Icon(Icons.lock_outlined),
                errorText: formState.password.displayError?.message,
                onChanged: notifier.passwordChanged,
                onSubmitted: (_) => notifier.submit(),
              ),
              const SizedBox(height: 8),

              // Forgot password
              Align(
                alignment: Alignment.centerRight,
                child: TextButton(
                  onPressed: () => context.push('/login/forgot'),
                  child: const Text('Quên mật khẩu?'),
                ),
              ),
              const SizedBox(height: 16),

              // Error message từ API
              if (formState.errorMessage != null) ...[
                _ErrorBanner(message: formState.errorMessage!),
                const SizedBox(height: 16),
              ],

              // Submit button
              FilledButton(
                onPressed: formState.canSubmit ? notifier.submit : null,
                child: formState.status.isInProgress
                    ? const SizedBox(
                        height: 20,
                        width: 20,
                        child: CircularProgressIndicator(
                          strokeWidth: 2,
                          color: Colors.white,
                        ),
                      )
                    : const Text('Đăng nhập'),
              ),
              const SizedBox(height: 24),

              // Register link
              Row(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Text(
                    'Chưa có tài khoản? ',
                    style: context.textTheme.bodyMedium,
                  ),
                  TextButton(
                    onPressed: () => context.push('/login/register'),
                    child: const Text('Đăng ký ngay'),
                  ),
                ],
              ),
            ],
          ),
        ),
      ),
    );
  }
}

class _ErrorBanner extends StatelessWidget {
  const _ErrorBanner({required this.message});
  final String message;

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(12),
      decoration: BoxDecoration(
        color: context.colorScheme.errorContainer,
        borderRadius: BorderRadius.circular(8),
      ),
      child: Row(
        children: [
          Icon(Icons.error_outline,
              color: context.colorScheme.onErrorContainer, size: 20),
          const SizedBox(width: 8),
          Expanded(
            child: Text(
              message,
              style: context.textTheme.bodySmall?.copyWith(
                color: context.colorScheme.onErrorContainer,
              ),
            ),
          ),
        ],
      ),
    );
  }
}
```

### 7.5.3. Password Strength Indicator

```dart
// Thêm visual feedback về độ mạnh mật khẩu
class PasswordStrengthIndicator extends ConsumerWidget {
  const PasswordStrengthIndicator({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final password = ref.watch(
      registerFormProvider.select((s) => s.password),
    );

    if (password.isPure || password.value.isEmpty) return const SizedBox.shrink();

    final strength = password.strength; // 0-4
    final (label, color) = switch (strength) {
      0 || 1 => ('Rất yếu', context.colorScheme.error),
      2 => ('Yếu', Colors.orange),
      3 => ('Trung bình', Colors.yellow.shade700),
      _ => ('Mạnh', Colors.green),
    };

    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        const SizedBox(height: 8),
        Row(
          children: List.generate(4, (index) {
            return Expanded(
              child: Container(
                height: 4,
                margin: EdgeInsets.only(right: index < 3 ? 4 : 0),
                decoration: BoxDecoration(
                  color: index < strength ? color : context.colorScheme.outlineVariant,
                  borderRadius: BorderRadius.circular(2),
                ),
              ),
            );
          }),
        ),
        const SizedBox(height: 4),
        Text(
          'Độ mạnh: $label',
          style: context.textTheme.labelSmall?.copyWith(color: color),
        ),
      ],
    );
  }
}
```

---

## 7.6. Bài Tập: Form Thêm Sản Phẩm

Xây dựng form thêm sản phẩm với các trường: tên, mô tả, giá, số lượng, danh mục — có validation đầy đủ.

```dart
// Gợi ý state
@freezed
class AddProductFormState with _$AddProductFormState {
  const factory AddProductFormState({
    @Default(RequiredTextInput.pure()) RequiredTextInput name,
    @Default(RequiredTextInput.pure()) RequiredTextInput description,
    @Default(PriceInput.pure()) PriceInput price,
    @Default(null) String? selectedCategoryId,
    @Default(FormzSubmissionStatus.initial) FormzSubmissionStatus status,
    String? errorMessage,
  }) = _AddProductFormState;

  const AddProductFormState._();

  bool get isValid =>
      Formz.validate([name, description, price]) &&
      selectedCategoryId != null;
}
```

---

## Tóm Tắt Chương 7

| Khái niệm | Điểm Cốt Lõi |
|---|---|
| Formz | Mô hình hóa input như state machine: pure / valid / invalid |
| FormzInput | Class riêng cho mỗi loại input — tái sử dụng toàn app |
| `pure` vs `dirty` | pure: chưa chạm; dirty: đã tương tác — chỉ hiện lỗi khi dirty |
| `displayError` | Tự động trả về null nếu field còn pure — không spam lỗi |
| Form state | Riverpod NotifierProvider chứa toàn bộ trạng thái form |
| `isValid` | `Formz.validate([...inputs])` — kiểm tra tất cả cùng lúc |
| `canSubmit` | isValid && !isInProgress — guard double-submit |
| AppTextField | Custom widget nhận `errorText` từ bên ngoài — tách UI khỏi logic |

> **Thực tiễn quan trọng:** Không bao giờ validate bằng cách show lỗi ngay khi user chưa chạm vào field — đây là UX tệ. Formz giải quyết đúng vấn đề này: `displayError` chỉ không null khi field đã `dirty`. Tốt hơn nữa là validate `onBlur` (khi focus rời khỏi field) thay vì `onChange` liên tục.