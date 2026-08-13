# Chương 11: Kiến Trúc & Cấu Trúc Thư Mục

---

## 11.1. Tại Sao Kiến Trúc Quan Trọng?

### 11.1.1. Hậu Quả Của Không Có Kiến Trúc

Khi dự án Flutter thiếu kiến trúc rõ ràng, các triệu chứng điển hình xuất hiện sau vài tháng phát triển:

- **God widget:** Một `StatefulWidget` dài 800 dòng chứa UI, HTTP call, business logic và local storage cùng nhau.
- **Spaghetti dependency:** Widget A import trực tiếp từ widget B, C, D — không thể test hoặc thay thế riêng lẻ.
- **Duplicate code:** Logic validate email xuất hiện ở 5 màn hình khác nhau, mỗi nơi một phiên bản khác nhau.
- **Khó onboard:** Developer mới mất 2 tuần chỉ để hiểu code chạy theo thứ tự nào.

Kiến trúc tốt giải quyết những vấn đề này bằng cách **phân tách trách nhiệm** — mỗi tầng, mỗi class chỉ làm một việc, và ranh giới giữa chúng rõ ràng.

### 11.1.2. Clean Architecture Lite cho Flutter

Flutter không cần triển khai Clean Architecture nguyên bản với Use Case layer phức tạp. Phiên bản rút gọn phù hợp với team nhỏ và dự án vừa:

```
┌─────────────────────────────────────────────┐
│           Presentation Layer                 │
│   Screen / Widget / Provider (Riverpod)      │
│   - Hiển thị UI                              │
│   - Lắng nghe state                          │
│   - Gọi method trên Notifier                 │
├─────────────────────────────────────────────┤
│           Domain Layer (optional)            │
│   Use Case / Business Logic                  │
│   - Quy tắc nghiệp vụ phức tạp              │
│   - Không biết UI tồn tại                    │
│   - Không biết data source là gì             │
├─────────────────────────────────────────────┤
│           Data Layer                         │
│   Repository → Data Source (Dio / Hive)     │
│   - Fetch và lưu dữ liệu                     │
│   - Map DTO → Domain model                   │
│   - Abstract interface cho Repository        │
└─────────────────────────────────────────────┘
```

**Quy tắc phụ thuộc (Dependency Rule):** Mũi tên chỉ đi từ ngoài vào trong. Presentation biết Domain, Domain không biết Presentation. Data implement interface của Domain, Domain không import Data.

---

## 11.2. Cấu Trúc Thư Mục

### 11.2.1. Feature-First vs Layer-First

Có hai trường phái chính khi tổ chức thư mục Flutter:

**Layer-first:**
```
lib/
├── data/
│   ├── repositories/
│   ├── datasources/
│   └── models/
├── domain/
│   └── entities/
└── presentation/
    ├── screens/
    └── widgets/
```

**Feature-first:**
```
lib/
├── features/
│   ├── auth/
│   │   ├── data/
│   │   ├── providers/
│   │   └── screens/
│   └── products/
│       ├── data/
│       ├── providers/
│       └── screens/
└── core/
```

**Khuyến nghị: Feature-first** cho hầu hết dự án Flutter vì:
- Tất cả code liên quan đến một feature nằm cùng chỗ → dễ tìm, dễ xóa
- Dễ tách thành package riêng nếu cần
- Team có thể làm feature khác nhau mà ít conflict

### 11.2.2. Cấu Trúc Thư Mục Chuẩn Dự Án Thực Tế

```
lib/
│
├── main.dart                    # Entry point, khởi tạo app
├── app.dart                     # MaterialApp.router, theme setup
│
├── core/                        # Code dùng chung toàn app
│   ├── config/
│   │   ├── app_config.dart      # Cấu hình theo flavor (dev/prod)
│   │   └── app_constants.dart   # Magic numbers, string constants
│   │
│   ├── errors/
│   │   ├── app_exception.dart   # Hierarchy exception
│   │   └── error_handler.dart   # Global error handler
│   │
│   ├── extensions/
│   │   ├── context_extensions.dart   # BuildContext helpers
│   │   ├── string_extensions.dart
│   │   └── datetime_extensions.dart
│   │
│   ├── network/
│   │   ├── dio_client.dart
│   │   └── interceptors/
│   │       ├── auth_interceptor.dart
│   │       └── error_interceptor.dart
│   │
│   ├── router/
│   │   ├── app_router.dart      # GoRouter config
│   │   └── app_routes.dart      # Route constants
│   │
│   ├── storage/
│   │   ├── token_storage.dart
│   │   ├── preferences_storage.dart
│   │   └── hive_initializer.dart
│   │
│   ├── theme/
│   │   ├── app_theme.dart       # ThemeData
│   │   └── app_colors.dart      # Extended color tokens
│   │
│   ├── utils/
│   │   ├── result.dart          # Result<T> type
│   │   ├── validators/
│   │   │   └── form_inputs.dart # Formz inputs
│   │   └── formatters/
│   │       └── currency_formatter.dart
│   │
│   └── widgets/                 # Shared UI components
│       ├── app_cached_image.dart
│       ├── app_text_field.dart
│       ├── skeleton_box.dart
│       ├── empty_state.dart
│       ├── error_view.dart
│       └── loading_button.dart
│
├── features/
│   │
│   ├── auth/
│   │   ├── data/
│   │   │   ├── models/
│   │   │   │   ├── auth_tokens.dart       # DTO từ API
│   │   │   │   └── auth_tokens.g.dart
│   │   │   ├── repositories/
│   │   │   │   ├── auth_repository.dart        # Abstract interface
│   │   │   │   └── auth_repository_impl.dart   # Implementation
│   │   │   └── datasources/
│   │   │       └── auth_remote_datasource.dart
│   │   ├── domain/
│   │   │   └── models/
│   │   │       └── app_user.dart          # Domain entity
│   │   ├── providers/
│   │   │   ├── auth_provider.dart
│   │   │   ├── login_form_provider.dart
│   │   │   └── register_form_provider.dart
│   │   └── screens/
│   │       ├── login_screen.dart
│   │       ├── register_screen.dart
│   │       └── forgot_password_screen.dart
│   │
│   ├── products/
│   │   ├── data/
│   │   │   ├── models/
│   │   │   │   ├── product_dto.dart
│   │   │   │   └── product_dto.g.dart
│   │   │   └── repositories/
│   │   │       ├── product_repository.dart
│   │   │       ├── product_repository_impl.dart
│   │   │       └── mock_product_repository.dart
│   │   ├── domain/
│   │   │   └── models/
│   │   │       ├── product.dart
│   │   │       └── product.freezed.dart
│   │   ├── providers/
│   │   │   ├── product_list_provider.dart
│   │   │   ├── product_detail_provider.dart
│   │   │   └── product_filter_provider.dart
│   │   ├── screens/
│   │   │   ├── product_list_screen.dart
│   │   │   └── product_detail_screen.dart
│   │   └── widgets/
│   │       ├── product_card.dart
│   │       ├── product_grid.dart
│   │       └── product_card_skeleton.dart
│   │
│   ├── cart/
│   │   ├── providers/
│   │   │   └── cart_provider.dart
│   │   ├── screens/
│   │   │   └── cart_screen.dart
│   │   └── widgets/
│   │       └── cart_item_tile.dart
│   │
│   ├── checkout/
│   │   ├── providers/
│   │   │   └── checkout_provider.dart
│   │   └── screens/
│   │       ├── checkout_screen.dart
│   │       ├── address_screen.dart
│   │       ├── payment_screen.dart
│   │       └── order_success_screen.dart
│   │
│   ├── orders/
│   │   ├── data/
│   │   │   └── repositories/
│   │   ├── providers/
│   │   └── screens/
│   │
│   ├── favorites/
│   │   ├── data/
│   │   │   └── models/
│   │   │       └── favorite_product.dart
│   │   ├── providers/
│   │   │   └── favorites_provider.dart
│   │   └── screens/
│   │       └── favorites_screen.dart
│   │
│   ├── profile/
│   │   ├── providers/
│   │   └── screens/
│   │
│   └── settings/
│       ├── providers/
│       │   └── settings_provider.dart
│       └── screens/
│           └── settings_screen.dart
│
└── l10n/                        # Localization (nếu cần)
    ├── app_en.arb
    └── app_vi.arb

test/
├── unit/
│   ├── features/
│   │   ├── auth/
│   │   └── products/
│   └── core/
├── widget/
│   └── features/
└── integration/
```

---

## 11.3. Dependency Injection với Riverpod

Riverpod đóng vai trò DI container — mọi dependency được khai báo và resolve thông qua provider. Không cần thư viện DI riêng như `get_it` (dù vẫn có thể kết hợp).

### 11.3.1. Tổ Chức Provider Files

```dart
// lib/core/providers/core_providers.dart
// Tất cả provider cốt lõi khai báo ở đây

@riverpod
Dio dio(DioRef ref) {
  return DioClient.create(
    baseUrl: AppConfig.apiBaseUrl,
    tokenStorage: ref.watch(tokenStorageProvider),
  );
}

@riverpod
FlutterSecureStorage secureStorage(SecureStorageRef ref) {
  return const FlutterSecureStorage();
}

@riverpod
TokenStorage tokenStorage(TokenStorageRef ref) {
  return SecureTokenStorage(ref.watch(secureStorageProvider));
}

@riverpod
Future<SharedPreferences> sharedPreferences(SharedPreferencesRef ref) {
  return SharedPreferences.getInstance();
}

@riverpod
PreferencesStorage preferencesStorage(PreferencesStorageRef ref) {
  return SharedPreferencesStorage(
    ref.watch(sharedPreferencesProvider).requireValue,
  );
}
```

```dart
// lib/features/products/providers/repository_providers.dart

@riverpod
ProductRepository productRepository(ProductRepositoryRef ref) {
  // Dễ dàng switch giữa mock và real
  if (AppConfig.useMockData) {
    return MockProductRepository();
  }
  return ProductRepositoryImpl(dio: ref.watch(dioProvider));
}

// lib/features/auth/providers/repository_providers.dart

@riverpod
AuthRepository authRepository(AuthRepositoryRef ref) {
  return AuthRepositoryImpl(
    dio: ref.watch(dioProvider),
    tokenStorage: ref.watch(tokenStorageProvider),
  );
}
```

### 11.3.2. Provider Override Cho Testing

```dart
// Trong test, override provider bằng mock không cần thay đổi production code
void main() {
  testWidgets('Login screen shows error on invalid credentials', (tester) async {
    // Tạo mock repository
    final mockAuthRepo = MockAuthRepository();
    when(mockAuthRepo.login(email: any, password: any))
        .thenAnswer((_) async => Result.failure(
          const UnauthorizedException('Sai email hoặc mật khẩu'),
        ));

    await tester.pumpWidget(
      ProviderScope(
        overrides: [
          // Override provider bằng mock
          authRepositoryProvider.overrideWithValue(mockAuthRepo),
        ],
        child: const MaterialApp(home: LoginScreen()),
      ),
    );

    // ... test assertions
  });
}
```

---

## 11.4. Result Pattern và Error Handling

### 11.4.1. Exception Hierarchy

```dart
// lib/core/errors/app_exception.dart
// Hierarchy rõ ràng giúp caller biết cần xử lý lỗi gì

sealed class AppException implements Exception {
  const AppException(this.message);
  final String message;

  @override
  String toString() => message;
}

// Network errors
final class NetworkTimeoutException extends AppException {
  const NetworkTimeoutException([super.message = 'Kết nối quá chậm']);
}

final class NoInternetException extends AppException {
  const NoInternetException([super.message = 'Không có kết nối mạng']);
}

final class RequestCancelledException extends AppException {
  const RequestCancelledException([super.message = 'Yêu cầu đã bị hủy']);
}

// HTTP errors
final class UnauthorizedException extends AppException {
  const UnauthorizedException([super.message = 'Phiên đăng nhập hết hạn']);
}

final class ForbiddenException extends AppException {
  const ForbiddenException([super.message = 'Không có quyền truy cập']);
}

final class NotFoundException extends AppException {
  const NotFoundException([super.message = 'Không tìm thấy dữ liệu']);
}

final class BadRequestException extends AppException {
  const BadRequestException([super.message = 'Dữ liệu không hợp lệ']);
}

final class ConflictException extends AppException {
  const ConflictException([super.message = 'Dữ liệu đã tồn tại']);
}

final class ValidationException extends AppException {
  const ValidationException(this.errors)
      : super('Dữ liệu không hợp lệ');
  final Map<String, dynamic> errors;

  // Lấy lỗi của một field cụ thể
  String? getFieldError(String field) {
    final error = errors[field];
    if (error is List) return error.first as String?;
    return error as String?;
  }
}

final class ServerException extends AppException {
  const ServerException([super.message = 'Lỗi server']);
}

final class UnknownException extends AppException {
  const UnknownException([super.message = 'Lỗi không xác định']);
}

// Business logic errors
final class UserNotFoundException extends AppException {
  const UserNotFoundException([super.message = 'Không tìm thấy người dùng']);
}

final class InsufficientStockException extends AppException {
  const InsufficientStockException(int available)
      : super('Chỉ còn $available sản phẩm trong kho');
}

final class OrderMinimumException extends AppException {
  const OrderMinimumException(double minimum)
      : super('Đơn hàng tối thiểu ${minimum.toStringAsFixed(0)}đ');
}
```

### 11.4.2. Global Error Handler

```dart
// lib/core/errors/error_handler.dart

class ErrorHandler {
  // Chuyển AppException thành message thân thiện
  static String toUserMessage(Object error) {
    if (error is AppException) return error.message;
    if (error is FormatException) return 'Dữ liệu không đúng định dạng';
    return 'Đã xảy ra lỗi. Vui lòng thử lại.';
  }

  // Log lỗi — trong production sẽ gửi lên Crashlytics/Sentry
  static void log(Object error, StackTrace? stackTrace, {String? context}) {
    if (kDebugMode) {
      debugPrint('=== ERROR ${context != null ? '[$context]' : ''} ===');
      debugPrint(error.toString());
      if (stackTrace != null) debugPrint(stackTrace.toString());
    } else {
      // Production: gửi lên crash reporting
      FirebaseCrashlytics.instance.recordError(
        error,
        stackTrace,
        reason: context,
      );
    }
  }

  // Xử lý Flutter framework errors
  static void setupFlutterErrorHandling() {
    // Lỗi trong widget tree
    FlutterError.onError = (details) {
      log(details.exception, details.stack, context: 'FlutterError');
    };

    // Lỗi async không được bắt
    PlatformDispatcher.instance.onError = (error, stack) {
      log(error, stack, context: 'PlatformDispatcher');
      return true; // true = đã xử lý
    };
  }
}

// Trong main.dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  ErrorHandler.setupFlutterErrorHandling();

  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  await HiveInitializer.init();

  runApp(const ProviderScope(child: MyApp()));
}
```

---

## 11.5. AppConfig — Cấu Hình Theo Môi Trường

```dart
// lib/core/config/app_config.dart

// Được inject từ --dart-define khi build
// Xem Chương 13 để biết cách cấu hình flavors

abstract class AppConfig {
  // Đọc từ --dart-define
  static const apiBaseUrl = String.fromEnvironment(
    'API_BASE_URL',
    defaultValue: 'https://dev-api.example.com',
  );

  static const appEnv = String.fromEnvironment(
    'APP_ENV',
    defaultValue: 'dev',
  );

  static const useMockData = bool.fromEnvironment(
    'USE_MOCK_DATA',
    defaultValue: false,
  );

  // Computed
  static bool get isProduction => appEnv == 'prod';
  static bool get isDevelopment => appEnv == 'dev';
  static bool get isStaging => appEnv == 'staging';

  // Feature flags
  static const enableAnalytics = bool.fromEnvironment(
    'ENABLE_ANALYTICS',
    defaultValue: false,
  );
}
```

---

## 11.6. Extension Methods — Giảm Code Lặp

```dart
// lib/core/extensions/context_extensions.dart

extension BuildContextX on BuildContext {
  ThemeData get theme => Theme.of(this);
  ColorScheme get colorScheme => Theme.of(this).colorScheme;
  TextTheme get textTheme => Theme.of(this).textTheme;
  bool get isDarkMode => Theme.of(this).brightness == Brightness.dark;
  MediaQueryData get mediaQuery => MediaQuery.of(this);
  Size get screenSize => MediaQuery.sizeOf(this);
  double get screenWidth => MediaQuery.sizeOf(this).width;
  double get screenHeight => MediaQuery.sizeOf(this).height;
  EdgeInsets get padding => MediaQuery.paddingOf(this);

  bool get isTablet => screenWidth >= 600;
  bool get isDesktop => screenWidth >= 1024;

  void showSnackBar(String message) {
    ScaffoldMessenger.of(this).showSnackBar(
      SnackBar(content: Text(message)),
    );
  }

  void pop<T>([T? result]) => Navigator.of(this).pop(result);
}

// lib/core/extensions/string_extensions.dart
extension StringX on String {
  String get capitalize =>
      isEmpty ? this : '${this[0].toUpperCase()}${substring(1)}';

  String get titleCase => split(' ').map((w) => w.capitalize).join(' ');

  bool get isValidEmail =>
      RegExp(r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$').hasMatch(this);

  bool get isValidPhone =>
      RegExp(r'^(0|\+84)(3|5|7|8|9)[0-9]{8}$').hasMatch(this);

  String truncate(int maxLength, {String ellipsis = '...'}) {
    if (length <= maxLength) return this;
    return '${substring(0, maxLength - ellipsis.length)}$ellipsis';
  }

  // Định dạng số điện thoại: 0901234567 → 090 123 4567
  String get formattedPhone {
    if (length != 10) return this;
    return '${substring(0, 3)} ${substring(3, 6)} ${substring(6)}';
  }
}

// lib/core/extensions/datetime_extensions.dart
extension DateTimeX on DateTime {
  String get timeAgo {
    final diff = DateTime.now().difference(this);
    if (diff.inSeconds < 60) return 'Vừa xong';
    if (diff.inMinutes < 60) return '${diff.inMinutes} phút trước';
    if (diff.inHours < 24) return '${diff.inHours} giờ trước';
    if (diff.inDays < 7) return '${diff.inDays} ngày trước';
    return '${day.toString().padLeft(2, '0')}/${month.toString().padLeft(2, '0')}/$year';
  }

  String get formattedDate =>
      '${day.toString().padLeft(2, '0')}/${month.toString().padLeft(2, '0')}/$year';

  String get formattedDateTime =>
      '$formattedDate ${hour.toString().padLeft(2, '0')}:${minute.toString().padLeft(2, '0')}';

  bool get isToday {
    final now = DateTime.now();
    return year == now.year && month == now.month && day == now.day;
  }
}

// lib/core/extensions/num_extensions.dart
extension NumX on num {
  String get formattedCurrency {
    if (this >= 1000000) {
      return '${(this / 1000000).toStringAsFixed(1)}M đ';
    }
    if (this >= 1000) {
      final formatted = toString().replaceAllMapped(
        RegExp(r'(\d)(?=(\d{3})+(?!\d))'),
        (match) => '${match[1]}.',
      );
      return '$formatted đ';
    }
    return '$this đ';
  }

  Duration get milliseconds => Duration(milliseconds: toInt());
  Duration get seconds => Duration(seconds: toInt());
  Duration get minutes => Duration(minutes: toInt());
}
```

---

## 11.7. Coding Conventions

### 11.7.1. Naming Conventions

```dart
// Files: snake_case
// product_list_screen.dart ✅
// ProductListScreen.dart   ❌

// Classes: PascalCase
class ProductRepository {}
class AuthNotifier {}
enum OrderStatus {}

// Variables, functions: camelCase
final userProfile = UserProfile();
void fetchProducts() {}
bool isLoggedIn = false;

// Constants: camelCase hoặc SCREAMING_SNAKE_CASE
const apiBaseUrl = 'https://api.example.com';
const int MAX_RETRY_COUNT = 3;

// Private members: _prefixed
class _ProductCardState {}
final _controller = TextEditingController();
void _handleSubmit() {}

// Providers: camelCase + Provider suffix (tự động với riverpod_generator)
// productListProvider, authProvider, cartProvider

// Notifiers: PascalCase, không có suffix
// class ProductList extends _$ProductList  (sinh ra productListProvider)

// Screens: PascalCase + Screen suffix
class LoginScreen {}
class ProductDetailScreen {}

// Widgets: PascalCase, không suffix
class ProductCard {}
class UserAvatar {}
class SkeletonBox {}
```

### 11.7.2. Import Ordering

```dart
// Thứ tự import chuẩn (dart format tự sắp xếp)

// 1. Dart SDK
import 'dart:async';
import 'dart:convert';

// 2. Flutter SDK
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

// 3. Third-party packages
import 'package:dio/dio.dart';
import 'package:go_router/go_router.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

// 4. Local imports — relative
import '../../core/network/dio_client.dart';
import '../../core/utils/result.dart';
import '../models/product.dart';

// Part files luôn ở cuối
part 'product_list_provider.g.dart';
```

### 11.7.3. Widget Best Practices

```dart
// ✅ CHUẨN — const constructor khi có thể
class ProductBadge extends StatelessWidget {
  const ProductBadge({super.key, required this.label}); // const
  final String label;

  @override
  Widget build(BuildContext context) {
    return Container(
      // const cho widget không phụ thuộc vào biến runtime
      child: const Text('New'), // ✅ const
    );
  }
}

// ✅ Tách widget nhỏ thay vì build method dài
class ProductDetailScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          const _ProductImage(),    // ✅ Widget riêng, const
          const _ProductInfo(),
          const _ProductActions(),
        ],
      ),
    );
  }
}

// Widget private trong cùng file nếu chỉ dùng ở đây
class _ProductImage extends StatelessWidget {
  const _ProductImage();
  // ...
}

// ✅ Dùng Consumer chỉ quanh widget cần rebuild, không quanh cả screen
class ProductDetailScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Chi tiết')),
      body: Column(
        children: [
          // Chỉ phần này rebuild khi cart thay đổi
          Consumer(
            builder: (context, ref, _) {
              final isInCart = ref.watch(isInCartProvider(productId));
              return AddToCartButton(isInCart: isInCart);
            },
          ),
          // Phần này không rebuild
          const _ProductDescription(),
        ],
      ),
    );
  }
}
```

---

## 11.8. Bài Tập: Refactor God Widget

Cho đoạn code antipattern sau, refactor theo đúng kiến trúc:

```dart
// ❌ TRƯỚC KHI REFACTOR — God widget
class OrderScreen extends StatefulWidget { /* ... */ }

class _OrderScreenState extends State<OrderScreen> {
  List<Order> orders = [];
  bool isLoading = false;
  String? error;

  @override
  void initState() {
    super.initState();
    // HTTP call trong widget
    Dio().get('https://api.example.com/orders').then((response) {
      setState(() {
        orders = (response.data as List)
            .map((e) => Order.fromJson(e))
            .toList();
        isLoading = false;
      });
    }).catchError((e) {
      setState(() {
        error = e.toString();
        isLoading = false;
      });
    });
  }

  @override
  Widget build(BuildContext context) {
    // 200 dòng UI trộn lẫn với logic...
  }
}

// ✅ SAU KHI REFACTOR — Tách đúng tầng

// 1. Repository (data layer)
abstract interface class OrderRepository {
  Future<Result<List<Order>>> fetchOrders();
}

// 2. Provider (presentation layer - state)
@riverpod
class OrderList extends _$OrderList {
  @override
  Future<List<Order>> build() async {
    final result = await ref.watch(orderRepositoryProvider).fetchOrders();
    return switch (result) {
      Success(:final data) => data,
      Failure(:final error) => throw error,
    };
  }
}

// 3. Screen (presentation layer - UI)
class OrderScreen extends ConsumerWidget {
  const OrderScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final ordersAsync = ref.watch(orderListProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Đơn hàng')),
      body: ordersAsync.when(
        loading: () => const OrderListSkeleton(),
        error: (e, _) => ErrorView(
          message: ErrorHandler.toUserMessage(e),
          onRetry: () => ref.invalidate(orderListProvider),
        ),
        data: (orders) => orders.isEmpty
            ? const EmptyState(
                icon: Icons.receipt_long_outlined,
                title: 'Chưa có đơn hàng',
              )
            : _OrderListView(orders: orders),
      ),
    );
  }
}
```

---

## Tóm Tắt Chương 11

| Khái niệm | Điểm Cốt Lõi |
|---|---|
| Feature-first | Code theo feature, không theo layer — dễ tìm, dễ scale |
| Dependency Rule | Presentation → Domain → Data. Không bao giờ ngược lại |
| Riverpod = DI | Provider là DI container — không cần get_it riêng |
| Exception Hierarchy | Sealed class — compiler bắt buộc xử lý mọi case |
| Result<T> | Explicit error, không throw exception bất ngờ |
| AppConfig | Cấu hình từ --dart-define, không hardcode trong code |
| Extension methods | Giảm boilerplate — context.colorScheme thay vì Theme.of(context).colorScheme |
| const constructor | Giảm rebuild — luôn thêm const khi có thể |
| Widget nhỏ | Tách build method dài thành widget riêng — testable, readable |

> **Nguyên tắc vàng:** Nếu không thể viết unit test cho một class mà không khởi động cả app Flutter — class đó đang có quá nhiều trách nhiệm hoặc dependency không được inject đúng cách. Kiến trúc tốt = testability tốt. Hai thứ này không tách rời nhau.