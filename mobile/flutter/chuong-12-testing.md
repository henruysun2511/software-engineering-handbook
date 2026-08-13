# Chương 12: Testing Trong Flutter

---

## 12.1. Tại Sao Phải Test?

### 12.1.1. Thực Tế Của Dự Án Không Có Test

Không test không có nghĩa là viết code nhanh hơn — nó có nghĩa là **debug thủ công nhiều hơn** về sau. Những tình huống phổ biến:

- Sửa bug ở màn hình A → vô tình phá màn hình B, C mà không biết
- Refactor logic tính giá → tất cả cart calculation sai, chỉ phát hiện khi user complain
- Thêm field mới vào API response → crash khi parse JSON ở những màn hình không ai nhớ đến

Test tự động giải quyết những vấn đề này: **chạy 200 test trong 30 giây, biết ngay có gì vỡ không**.

### 12.1.2. Ba Tầng Test Trong Flutter

```
┌─────────────────────────────────────────────┐
│           Integration Tests                  │
│   Chạy trên thiết bị thật / emulator        │
│   Test toàn bộ flow: login → add cart → pay │
│   Chậm (phút), ít test, high confidence     │
├─────────────────────────────────────────────┤
│           Widget Tests                       │
│   Render widget, simulate tap/input         │
│   Test UI behavior không cần thiết bị       │
│   Nhanh (giây), số lượng vừa               │
├─────────────────────────────────────────────┤
│           Unit Tests                         │
│   Test function, class thuần Dart           │
│   Không cần Flutter framework               │
│   Rất nhanh (ms), nhiều nhất               │
└─────────────────────────────────────────────┘
```

**Tỷ lệ khuyến nghị:** 70% unit test, 20% widget test, 10% integration test.

---

## 12.2. Unit Test

### 12.2.1. Setup

```yaml
# pubspec.yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  mocktail: ^1.0.0        # Mock framework — đơn giản hơn mockito
  fake_async: ^1.3.0      # Control time trong test
```

```dart
// test/unit/features/products/product_repository_test.dart

// Cấu trúc file test chuẩn:
// describe → group()
// it/test → test()
// setup → setUp() / setUpAll()
// teardown → tearDown() / tearDownAll()

void main() {
  // group: nhóm các test liên quan
  group('ProductRepository', () {
    late MockDio mockDio;
    late ProductRepositoryImpl repository;

    // setUp: chạy trước MỖI test trong group
    setUp(() {
      mockDio = MockDio();
      repository = ProductRepositoryImpl(dio: mockDio);
    });

    group('fetchProducts', () {
      test('returns list of products on success', () async {
        // Arrange
        when(() => mockDio.get(
          '/products',
          queryParameters: any(named: 'queryParameters'),
        )).thenAnswer((_) async => Response(
          data: {
            'items': [
              {
                'id': '1',
                'name': 'Áo',
                'price': 200000,
                'stock': 10,
                'category_id': 'cat-1',
                'status': 'active',
                'image_url': 'https://example.com/ao.jpg',
                'created_at': '2024-01-01T00:00:00Z',
              }
            ],
            'total': 1,
            'page': 1,
            'page_size': 20,
            'total_pages': 1,
          },
          statusCode: 200,
          requestOptions: RequestOptions(),
        ));

        // Act
        final result = await repository.fetchProducts();

        // Assert
        expect(result.items, hasLength(1));
        expect(result.items.first.name, equals('Áo'));
        expect(result.items.first.price, equals(200000.0));
        expect(result.total, equals(1));
      });

      test('throws NotFoundException when server returns 404', () async {
        when(() => mockDio.get(any(), queryParameters: any(named: 'queryParameters')))
            .thenThrow(DioException(
              requestOptions: RequestOptions(),
              response: Response(
                statusCode: 404,
                requestOptions: RequestOptions(),
              ),
              type: DioExceptionType.badResponse,
              error: const NotFoundException(),
            ));

        expect(
          () => repository.fetchProducts(),
          throwsA(isA<NotFoundException>()),
        );
      });

      test('passes category filter to API', () async {
        when(() => mockDio.get(
          '/products',
          queryParameters: {'category': 'electronics', 'sort': 'newest', 'page': 1, 'page_size': 20},
        )).thenAnswer((_) async => Response(
          data: {'items': [], 'total': 0, 'page': 1, 'page_size': 20, 'total_pages': 0},
          statusCode: 200,
          requestOptions: RequestOptions(),
        ));

        await repository.fetchProducts(category: 'electronics');

        verify(() => mockDio.get(
          '/products',
          queryParameters: {'category': 'electronics', 'sort': 'newest', 'page': 1, 'page_size': 20},
        )).called(1);
      });
    });
  });
}

// Mock class — mocktail tự sinh implementation
class MockDio extends Mock implements Dio {}
```

### 12.2.2. Test Riverpod Provider

```dart
// test/unit/features/cart/cart_provider_test.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  group('CartProvider', () {
    late ProviderContainer container;

    setUp(() {
      // ProviderContainer thay thế ProviderScope trong test
      container = ProviderContainer();
    });

    tearDown(() {
      container.dispose(); // Luôn dispose container
    });

    test('starts with empty state', () {
      final cart = container.read(cartProvider);
      expect(cart.items, isEmpty);
      expect(cart.total, equals(0.0));
      expect(cart.isEmpty, isTrue);
    });

    test('adds item to cart', () {
      final product = _makeProduct(id: '1', price: 100000);
      container.read(cartProvider.notifier).addItem(product);

      final cart = container.read(cartProvider);
      expect(cart.items, hasLength(1));
      expect(cart.items.first.product.id, equals('1'));
      expect(cart.items.first.quantity, equals(1));
    });

    test('increments quantity when adding same product twice', () {
      final product = _makeProduct(id: '1', price: 100000);
      container.read(cartProvider.notifier).addItem(product);
      container.read(cartProvider.notifier).addItem(product);

      final cart = container.read(cartProvider);
      expect(cart.items, hasLength(1)); // Vẫn 1 item
      expect(cart.items.first.quantity, equals(2)); // Nhưng quantity = 2
    });

    test('removes item from cart', () {
      final product = _makeProduct(id: '1', price: 100000);
      container.read(cartProvider.notifier).addItem(product);
      container.read(cartProvider.notifier).removeItem('1');

      expect(container.read(cartProvider).items, isEmpty);
    });

    test('calculates total correctly with multiple items', () {
      container.read(cartProvider.notifier)
        ..addItem(_makeProduct(id: '1', price: 100000), quantity: 2)
        ..addItem(_makeProduct(id: '2', price: 50000), quantity: 3);

      // 100000 * 2 + 50000 * 3 = 350000
      expect(container.read(cartProvider).total, equals(350000.0));
    });

    test('applies coupon discount', () {
      container.read(cartProvider.notifier)
        ..addItem(_makeProduct(id: '1', price: 200000))
        ..applyCoupon('FLUTTER10');

      final cart = container.read(cartProvider);
      expect(cart.discountPercent, equals(10));
      expect(cart.discount, equals(20000.0));   // 200000 * 10%
      expect(cart.total, equals(180000.0));
    });

    test('clears entire cart', () {
      container.read(cartProvider.notifier)
        ..addItem(_makeProduct(id: '1', price: 100000))
        ..addItem(_makeProduct(id: '2', price: 200000))
        ..clear();

      expect(container.read(cartProvider).items, isEmpty);
    });
  });
}

// Helper factory
Product _makeProduct({required String id, required double price}) {
  return Product(
    id: id,
    name: 'Product $id',
    description: '',
    price: price,
    imageUrl: '',
    categoryId: 'cat-1',
    status: ProductStatus.active,
    stock: 10,
    createdAt: DateTime(2024),
  );
}
```

### 12.2.3. Test Business Logic — Formz Input

```dart
// test/unit/core/validators/email_input_test.dart

void main() {
  group('EmailInput', () {
    group('pure state', () {
      test('is pure initially', () {
        const input = EmailInput.pure();
        expect(input.isPure, isTrue);
        expect(input.isValid, isFalse); // Pure không valid
        expect(input.displayError, isNull); // Pure không hiện lỗi
      });
    });

    group('validation', () {
      final validEmails = [
        'user@example.com',
        'user.name@example.co.uk',
        'user+tag@gmail.com',
        '123@numbers.org',
      ];

      final invalidEmails = [
        '',
        'not-an-email',
        '@nodomain.com',
        'noDomain@',
        'spaces in@email.com',
      ];

      for (final email in validEmails) {
        test('validates "$email" as valid', () {
          final input = EmailInput.dirty(email);
          expect(input.isValid, isTrue);
          expect(input.displayError, isNull);
        });
      }

      test('returns empty error for empty email', () {
        const input = EmailInput.dirty('');
        expect(input.displayError, equals(EmailValidationError.empty));
      });

      for (final email in invalidEmails.where((e) => e.isNotEmpty)) {
        test('validates "$email" as invalid format', () {
          final input = EmailInput.dirty(email);
          expect(input.displayError, equals(EmailValidationError.invalidFormat));
        });
      }
    });
  });

  group('PasswordInput', () {
    test('strong password passes all checks', () {
      const input = PasswordInput.dirty('Flutter@2024');
      expect(input.isValid, isTrue);
      expect(input.strength, equals(4));
    });

    test('returns tooShort for short password', () {
      const input = PasswordInput.dirty('Ab1!');
      expect(input.displayError, equals(PasswordValidationError.tooShort));
    });

    test('returns noUppercase when no uppercase', () {
      const input = PasswordInput.dirty('flutter@2024');
      expect(input.displayError, equals(PasswordValidationError.noUppercase));
    });

    test('strength increases with complexity', () {
      expect(PasswordInput.dirty('flutter1').strength, lessThan(4));
      expect(PasswordInput.dirty('Flutter@2024').strength, equals(4));
    });
  });
}
```

### 12.2.4. Test Với Mock Repository

```dart
// test/unit/features/auth/auth_provider_test.dart

void main() {
  group('AuthProvider', () {
    late MockAuthRepository mockRepo;
    late MockTokenStorage mockTokenStorage;
    late ProviderContainer container;

    setUp(() {
      mockRepo = MockAuthRepository();
      mockTokenStorage = MockTokenStorage();

      // Setup default mock behavior
      when(() => mockTokenStorage.getAccessToken())
          .thenAnswer((_) async => null);

      container = ProviderContainer(
        overrides: [
          // Override các provider bằng mock
          authRepositoryProvider.overrideWithValue(mockRepo),
          tokenStorageProvider.overrideWithValue(mockTokenStorage),
        ],
      );
    });

    tearDown(() => container.dispose());

    test('initial state is unauthenticated when no token', () async {
      final state = await container.read(authProvider.future);
      expect(state, isA<AuthUnauthenticated>());
    });

    test('login succeeds with valid credentials', () async {
      final mockUser = AppUser(id: '1', name: 'Test', email: 'test@test.com');
      final mockTokens = AuthTokens(
        accessToken: 'access_123',
        refreshToken: 'refresh_123',
        user: mockUser,
      );

      when(() => mockRepo.login(
            email: 'test@test.com',
            password: 'Password@1',
          )).thenAnswer((_) async => Result.success(mockTokens));

      when(() => mockTokenStorage.saveTokens(
            accessToken: any(named: 'accessToken'),
            refreshToken: any(named: 'refreshToken'),
          )).thenAnswer((_) async {});

      await container.read(authProvider.notifier).login(
            email: 'test@test.com',
            password: 'Password@1',
          );

      final state = container.read(authProvider).valueOrNull;
      expect(state, isA<AuthAuthenticated>());
      expect((state as AuthAuthenticated).user.id, equals('1'));
    });

    test('login fails with invalid credentials', () async {
      when(() => mockRepo.login(
            email: any(named: 'email'),
            password: any(named: 'password'),
          )).thenAnswer((_) async => Result.failure(
            const UnauthorizedException('Sai mật khẩu'),
          ));

      await container.read(authProvider.notifier).login(
            email: 'wrong@test.com',
            password: 'wrongpass',
          );

      final state = container.read(authProvider);
      expect(state.hasError, isTrue);
      expect(state.error, isA<UnauthorizedException>());
    });
  });
}

class MockAuthRepository extends Mock implements AuthRepository {}
class MockTokenStorage extends Mock implements TokenStorage {}
```

---

## 12.3. Widget Test

### 12.3.1. Cơ Bản Widget Test

```dart
// test/widget/features/auth/login_screen_test.dart

void main() {
  group('LoginScreen', () {
    late MockAuthRepository mockRepo;

    setUp(() {
      mockRepo = MockAuthRepository();
    });

    // Helper: pumpWidget với đầy đủ setup
    Future<void> pumpLoginScreen(WidgetTester tester) async {
      await tester.pumpWidget(
        ProviderScope(
          overrides: [
            authRepositoryProvider.overrideWithValue(mockRepo),
          ],
          child: const MaterialApp(
            home: LoginScreen(),
          ),
        ),
      );
    }

    testWidgets('renders email and password fields', (tester) async {
      await pumpLoginScreen(tester);

      // Tìm widget theo type
      expect(find.byType(TextField), findsNWidgets(2));
      // Tìm theo text
      expect(find.text('Email'), findsOneWidget);
      expect(find.text('Mật khẩu'), findsOneWidget);
      expect(find.text('Đăng nhập'), findsOneWidget);
    });

    testWidgets('shows validation error for empty email on submit', (tester) async {
      await pumpLoginScreen(tester);

      // Tap submit mà không nhập gì
      await tester.tap(find.text('Đăng nhập'));
      await tester.pump(); // Rebuild sau action

      expect(find.text('Email không được để trống'), findsOneWidget);
    });

    testWidgets('shows error for invalid email format', (tester) async {
      await pumpLoginScreen(tester);

      // Nhập email sai format
      await tester.enterText(
        find.widgetWithText(TextField, '').first,
        'not-an-email',
      );
      await tester.tap(find.text('Đăng nhập'));
      await tester.pump();

      expect(find.text('Định dạng email không hợp lệ'), findsOneWidget);
    });

    testWidgets('submit button is disabled when form is invalid', (tester) async {
      await pumpLoginScreen(tester);

      // Button disabled khi form chưa valid
      final button = tester.widget<FilledButton>(find.byType(FilledButton));
      expect(button.onPressed, isNull);
    });

    testWidgets('shows loading indicator when submitting', (tester) async {
      when(() => mockRepo.login(
            email: any(named: 'email'),
            password: any(named: 'password'),
          )).thenAnswer((_) async {
        await Future.delayed(const Duration(seconds: 1));
        return Result.success(MockAuthTokens());
      });

      await pumpLoginScreen(tester);

      // Nhập thông tin hợp lệ
      await tester.enterText(find.byType(TextField).at(0), 'user@test.com');
      await tester.enterText(find.byType(TextField).at(1), 'Password@1');
      await tester.pump();

      // Tap submit
      await tester.tap(find.text('Đăng nhập'));
      await tester.pump(); // Bắt đầu submit

      // Loading indicator hiển thị
      expect(find.byType(CircularProgressIndicator), findsOneWidget);
    });

    testWidgets('shows API error message on login failure', (tester) async {
      when(() => mockRepo.login(
            email: any(named: 'email'),
            password: any(named: 'password'),
          )).thenAnswer((_) async => Result.failure(
            const UnauthorizedException('Email hoặc mật khẩu không đúng'),
          ));

      await pumpLoginScreen(tester);

      await tester.enterText(find.byType(TextField).at(0), 'user@test.com');
      await tester.enterText(find.byType(TextField).at(1), 'Password@1');
      await tester.pump();
      await tester.tap(find.text('Đăng nhập'));
      await tester.pumpAndSettle(); // Chờ hết animation và async

      expect(find.text('Email hoặc mật khẩu không đúng'), findsOneWidget);
    });
  });
}
```

### 12.3.2. Test CartItemTile

```dart
// test/widget/features/cart/cart_item_tile_test.dart

void main() {
  group('CartItemTile', () {
    late CartItem testItem;

    setUp(() {
      testItem = CartItem(
        product: Product(
          id: '1', name: 'Áo Flutter', price: 250000,
          imageUrl: 'https://example.com/ao.jpg',
          categoryId: 'cat-1', status: ProductStatus.active,
          stock: 5, createdAt: DateTime(2024),
          description: 'Áo in hình Flutter',
        ),
        quantity: 2,
      );
    });

    testWidgets('displays product name and price', (tester) async {
      await tester.pumpWidget(
        ProviderScope(
          child: MaterialApp(
            home: Scaffold(
              body: CartItemTile(item: testItem),
            ),
          ),
        ),
      );

      expect(find.text('Áo Flutter'), findsOneWidget);
      expect(find.text('250.000đ'), findsOneWidget);
    });

    testWidgets('displays correct quantity', (tester) async {
      await tester.pumpWidget(
        ProviderScope(
          child: MaterialApp(
            home: Scaffold(
              body: CartItemTile(item: testItem),
            ),
          ),
        ),
      );

      expect(find.text('2'), findsOneWidget);
    });

    testWidgets('calls removeItem when delete icon tapped', (tester) async {
      final container = ProviderContainer();

      await tester.pumpWidget(
        UncontrolledProviderScope(
          container: container,
          child: MaterialApp(
            home: Scaffold(
              body: CartItemTile(item: testItem),
            ),
          ),
        ),
      );

      // Thêm item vào cart trước
      container.read(cartProvider.notifier).addItem(testItem.product, quantity: 2);

      // Tap delete
      await tester.tap(find.byIcon(Icons.close));
      await tester.pump();

      expect(container.read(cartProvider).items, isEmpty);
    });
  });
}
```

### 12.3.3. Golden Test — Snapshot UI

```dart
// test/widget/golden/product_card_golden_test.dart
// Golden test: chụp ảnh UI, so sánh với ảnh cũ
// Dùng để detect UI regression

void main() {
  group('ProductCard Golden', () {
    testWidgets('matches golden snapshot', (tester) async {
      await tester.pumpWidget(
        MaterialApp(
          theme: AppTheme.light,
          home: Scaffold(
            body: Padding(
              padding: const EdgeInsets.all(16),
              child: ProductCard(
                product: Product(
                  id: '1',
                  name: 'Áo Flutter Premium',
                  price: 350000,
                  originalPrice: 500000,
                  imageUrl: 'https://example.com/ao.jpg',
                  categoryId: 'cat-1',
                  status: ProductStatus.active,
                  stock: 5,
                  createdAt: DateTime(2024),
                  description: '',
                ),
              ),
            ),
          ),
        ),
      );

      // So sánh với file golden đã lưu
      await expectLater(
        find.byType(ProductCard),
        matchesGoldenFile('goldens/product_card.png'),
      );
    });
  });
}

// Tạo/cập nhật golden files:
// flutter test --update-goldens
```

---

## 12.4. Test Helpers và Utilities

### 12.4.1. Test Utilities Tái Sử Dụng

```dart
// test/helpers/test_helpers.dart

// Pump widget với ProviderScope đầy đủ
extension WidgetTesterX on WidgetTester {
  Future<void> pumpApp(
    Widget widget, {
    List<Override> overrides = const [],
    GoRouter? router,
  }) async {
    await pumpWidget(
      ProviderScope(
        overrides: overrides,
        child: MaterialApp(
          theme: AppTheme.light,
          darkTheme: AppTheme.dark,
          home: widget,
        ),
      ),
    );
  }

  // Nhập text vào TextField với label cụ thể
  Future<void> enterTextInField(String label, String text) async {
    final field = find.ancestor(
      of: find.text(label),
      matching: find.byType(TextField),
    );
    await enterText(field, text);
    await pump();
  }

  // Tap button theo text
  Future<void> tapButton(String label) async {
    await tap(find.widgetWithText(ElevatedButton, label)
        .evaluate()
        .isNotEmpty
        ? find.widgetWithText(ElevatedButton, label)
        : find.widgetWithText(FilledButton, label));
    await pumpAndSettle();
  }
}

// Fake repository implementations cho test
class FakeProductRepository implements ProductRepository {
  FakeProductRepository({this.products = const [], this.delay = Duration.zero});

  final List<Product> products;
  final Duration delay;

  @override
  Future<PaginatedResponse<Product>> fetchProducts({
    String? category,
    String? searchQuery,
    ProductSortOrder sort = ProductSortOrder.newest,
    int page = 1,
    int pageSize = 20,
    CancelToken? cancelToken,
  }) async {
    await Future.delayed(delay);
    return PaginatedResponse(
      items: products,
      total: products.length,
      page: page,
      pageSize: pageSize,
      totalPages: 1,
    );
  }

  @override
  Future<ProductDetail> fetchProductDetail(String id, {CancelToken? cancelToken}) async {
    await Future.delayed(delay);
    final product = products.firstWhere((p) => p.id == id);
    return ProductDetail(product: product, reviews: [], relatedIds: []);
  }

  @override
  Future<List<Product>> fetchRelatedProducts(String productId) async => [];

  @override
  Future<List<Product>> fetchFeaturedProducts() async => products.take(4).toList();
}

// Test data factories
class ProductFactory {
  static Product create({
    String? id,
    String? name,
    double? price,
    int? stock,
  }) => Product(
    id: id ?? 'product-${DateTime.now().millisecondsSinceEpoch}',
    name: name ?? 'Test Product',
    description: 'Test description',
    price: price ?? 100000,
    imageUrl: 'https://picsum.photos/200',
    categoryId: 'cat-1',
    status: ProductStatus.active,
    stock: stock ?? 10,
    createdAt: DateTime(2024),
  );

  static List<Product> createList(int count) =>
      List.generate(count, (i) => create(id: 'product-$i', name: 'Product $i'));
}
```

---

## 12.5. Integration Test

```dart
// integration_test/app_test.dart

import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('E2E: Login → Browse → Add to Cart → Checkout', () {
    testWidgets('complete purchase flow', (tester) async {
      // Launch app
      app.main();
      await tester.pumpAndSettle();

      // === LOGIN ===
      expect(find.byType(LoginScreen), findsOneWidget);

      await tester.enterText(
        find.byKey(const Key('email_field')),
        'test@example.com',
      );
      await tester.enterText(
        find.byKey(const Key('password_field')),
        'TestPassword@1',
      );
      await tester.tap(find.byKey(const Key('login_button')));
      await tester.pumpAndSettle(const Duration(seconds: 3));

      // Verify logged in
      expect(find.byType(HomeScreen), findsOneWidget);

      // === BROWSE PRODUCTS ===
      await tester.tap(find.byIcon(Icons.grid_view));
      await tester.pumpAndSettle();

      expect(find.byType(ProductListScreen), findsOneWidget);

      // === ADD TO CART ===
      await tester.tap(find.byType(ProductCard).first);
      await tester.pumpAndSettle();

      expect(find.byType(ProductDetailScreen), findsOneWidget);

      await tester.tap(find.text('Thêm vào giỏ'));
      await tester.pumpAndSettle();

      // Verify snackbar
      expect(find.textContaining('Đã thêm'), findsOneWidget);

      // === VIEW CART ===
      await tester.tap(find.byIcon(Icons.shopping_cart));
      await tester.pumpAndSettle();

      expect(find.byType(CartScreen), findsOneWidget);
      expect(find.byType(CartItemTile), findsOneWidget);

      // === CHECKOUT ===
      await tester.tap(find.textContaining('Đặt hàng'));
      await tester.pumpAndSettle();

      expect(find.byType(CheckoutScreen), findsOneWidget);
    });
  });
}
```

```bash
# Chạy integration test
flutter test integration_test/app_test.dart -d emulator-5554
```

---

## 12.6. Chạy Test và Coverage

```bash
# Chạy tất cả unit và widget test
flutter test

# Chạy test cụ thể
flutter test test/unit/features/cart/cart_provider_test.dart

# Chạy với coverage report
flutter test --coverage

# Tạo HTML report từ coverage
genhtml coverage/lcov.info -o coverage/html
open coverage/html/index.html

# Chạy test theo tag
flutter test --tags unit
flutter test --tags widget

# Watch mode — tự chạy lại khi file thay đổi
flutter test --watch
```

```yaml
# .github/workflows/test.yml — CI/CD với GitHub Actions
name: Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.x'

      - name: Install dependencies
        run: flutter pub get

      - name: Run code generation
        run: dart run build_runner build --delete-conflicting-outputs

      - name: Run tests
        run: flutter test --coverage

      - name: Check coverage threshold
        run: |
          COVERAGE=$(lcov --summary coverage/lcov.info | grep "lines" | awk '{print $2}' | tr -d '%')
          if (( $(echo "$COVERAGE < 70" | bc -l) )); then
            echo "Coverage $COVERAGE% is below 70% threshold"
            exit 1
          fi
```

---

## Tóm Tắt Chương 12

| Loại test | Tool | Mục đích | Tốc độ |
|---|---|---|---|
| Unit test | `flutter_test` + `mocktail` | Logic, provider, validator | ~ms |
| Widget test | `flutter_test` | UI behavior, interaction | ~giây |
| Golden test | `matchesGoldenFile` | UI regression detection | ~giây |
| Integration test | `integration_test` | E2E flow trên thiết bị | ~phút |

| Pattern | Điểm Cốt Lõi |
|---|---|
| ProviderContainer | Thay thế ProviderScope trong unit test |
| `overrideWithValue` | Inject mock provider không thay đổi production code |
| setUp/tearDown | Khởi tạo và dọn dẹp cho mỗi test — tránh shared state |
| Fake vs Mock | Fake: implementation thật nhưng đơn giản. Mock: verify interaction |
| Golden file | Cập nhật bằng `--update-goldens` khi UI thay đổi có chủ ý |
| Test factory | Tạo test data nhất quán — không tạo object thủ công trong từng test |

> **Thực tiễn tốt nhất:** Viết test trước khi fix bug — tạo failing test reproduce bug, rồi fix cho test pass. Đây gọi là Regression Test — đảm bảo bug không quay lại. Đây cũng là cách học test nhanh nhất: không cần test 100% coverage ngay, bắt đầu bằng test cho business logic quan trọng nhất (cart calculation, price discount, auth flow).