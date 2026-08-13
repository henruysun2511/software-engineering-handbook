# Chương 5: State Management với Riverpod

---

## 5.1. Tại Sao Cần State Management?

### 5.1.1. Vấn Đề Của setState

Khi app còn nhỏ, `setState` trong `StatefulWidget` đủ dùng. Nhưng khi app phức tạp lên, `setState` bộc lộ những hạn chế căn bản:

**Vấn đề 1: Prop drilling** — Truyền data qua nhiều tầng widget trung gian không liên quan đến data đó.

```
HomeScreen
  └── ProductListScreen
        └── ProductCard
              └── AddToCartButton  ← Cần biết cart state
```

Để `AddToCartButton` biết giỏ hàng hiện có bao nhiêu item, phải truyền `cartCount` từ `HomeScreen` xuống qua `ProductListScreen` → `ProductCard` → `AddToCartButton`. Cả `ProductListScreen` và `ProductCard` phải nhận param này dù không dùng.

**Vấn đề 2: State không đồng bộ** — Cùng một data (giỏ hàng) cần hiển thị ở nhiều nơi (badge trên icon cart, danh sách CartScreen, tổng tiền CheckoutScreen). Với `setState`, phải tự đồng bộ thủ công — rất dễ lỗi.

**Vấn đề 3: Business logic lẫn UI** — Logic tính toán, gọi API nằm trong `State` class của widget — khó test, khó tái sử dụng.

### 5.1.2. So Sánh Các Giải Pháp

| Giải pháp | Ưu điểm | Nhược điểm |
|---|---|---|
| `setState` | Đơn giản, built-in | Không scale, prop drilling |
| `Provider` | Phổ biến, đơn giản | Không type-safe tốt, khó combine |
| `Bloc/Cubit` | Rõ ràng, testable | Boilerplate nhiều, dốc học |
| **Riverpod** | Type-safe, compose tốt, testable | Cần hiểu concept provider |
| `GetX` | Minimal code | Magic quá nhiều, khó debug |

**Tại sao Riverpod là lựa chọn tốt nhất hiện tại:**
- Type-safe hoàn toàn — compiler bắt lỗi, không runtime error
- Không phụ thuộc `BuildContext` — có thể dùng provider bên ngoài widget
- Dễ combine nhiều provider
- Code generation với `riverpod_generator` giảm boilerplate
- Được Flutter team và cộng đồng lớn ủng hộ

---

## 5.2. Cài Đặt và Cấu Hình

### 5.2.1. Dependencies

```yaml
# pubspec.yaml
dependencies:
  flutter_riverpod: ^2.5.0
  riverpod_annotation: ^2.3.0

dev_dependencies:
  riverpod_generator: ^2.4.0
  build_runner: ^2.4.0
  custom_lint: ^0.6.0
  riverpod_lint: ^2.3.0
```

```bash
# Chạy code generation (chạy lại mỗi khi thêm/sửa provider)
dart run build_runner watch --delete-conflicting-outputs
```

### 5.2.2. Thiết Lập ProviderScope

```dart
// lib/main.dart
// ProviderScope PHẢI bọc toàn bộ app — đây là container chứa mọi provider

void main() {
  // Đảm bảo Flutter binding đã khởi tạo trước khi dùng async
  WidgetsFlutterBinding.ensureInitialized();

  runApp(
    const ProviderScope( // ← Bắt buộc, bọc ngoài cùng
      child: MyApp(),
    ),
  );
}

class MyApp extends ConsumerWidget { // ← Thay StatelessWidget bằng ConsumerWidget
  const MyApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) { // ← Thêm tham số WidgetRef
    return MaterialApp.router(
      routerConfig: appRouter,
      theme: AppTheme.light,
      darkTheme: AppTheme.dark,
    );
  }
}
```

---

## 5.3. Các Loại Provider Cơ Bản

Riverpod có nhiều loại provider, mỗi loại phục vụ một mục đích cụ thể. Hiểu đúng từng loại là chìa khóa để dùng Riverpod hiệu quả.

### 5.3.1. Provider — Giá Trị Bất Biến (Read-Only)

`Provider` dùng để expose một giá trị hoặc service không thay đổi. Tương đương Dependency Injection — tạo instance một lần, dùng ở nhiều nơi.

```dart
// ✅ CHUẨN — Provider cho singleton service
// lib/core/providers/service_providers.dart

import 'package:riverpod_annotation/riverpod_annotation.dart';
part 'service_providers.g.dart';

// @riverpod annotation tự sinh code, không cần viết Provider thủ công
@riverpod
Dio dio(DioRef ref) {
  final dio = Dio(
    BaseOptions(
      baseUrl: AppConfig.apiBaseUrl,
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 30),
      headers: {'Content-Type': 'application/json'},
    ),
  );

  // Thêm interceptors
  dio.interceptors.addAll([
    AuthInterceptor(ref),
    LogInterceptor(requestBody: true, responseBody: true),
  ]);

  return dio;
}

@riverpod
ProductRepository productRepository(ProductRepositoryRef ref) {
  // Lấy dio từ provider khác — đây là Dependency Injection
  return ProductRepository(dio: ref.watch(dioProvider));
}

@riverpod
CartRepository cartRepository(CartRepositoryRef ref) {
  return CartRepository(dio: ref.watch(dioProvider));
}
```

### 5.3.2. StateProvider — State Đơn Giản, Primitive

`StateProvider` dùng cho state đơn giản có thể thay đổi: bool, int, enum, String. Không phù hợp cho state phức tạp hoặc cần business logic.

```dart
// ✅ CHUẨN — StateProvider cho UI state đơn giản

@riverpod
class SortOrder extends _$SortOrder {
  @override
  ProductSortOrder build() => ProductSortOrder.newest; // Giá trị mặc định

  void update(ProductSortOrder order) => state = order;
}

@riverpod
class SearchQuery extends _$SearchQuery {
  @override
  String build() => '';

  void update(String query) => state = query;
  void clear() => state = '';
}

// Enum cho sort order
enum ProductSortOrder { newest, priceAsc, priceDesc, popular }

// Sử dụng trong widget:
class SortButton extends ConsumerWidget {
  const SortButton({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final sortOrder = ref.watch(sortOrderProvider);

    return PopupMenuButton<ProductSortOrder>(
      initialValue: sortOrder,
      onSelected: (order) {
        ref.read(sortOrderProvider.notifier).update(order);
      },
      itemBuilder: (_) => ProductSortOrder.values
          .map((order) => PopupMenuItem(
                value: order,
                child: Text(order.label),
              ))
          .toList(),
      child: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          Text('Sắp xếp: ${sortOrder.label}'),
          const Icon(Icons.arrow_drop_down),
        ],
      ),
    );
  }
}

extension on ProductSortOrder {
  String get label => switch (this) {
        ProductSortOrder.newest => 'Mới nhất',
        ProductSortOrder.priceAsc => 'Giá tăng dần',
        ProductSortOrder.priceDesc => 'Giá giảm dần',
        ProductSortOrder.popular => 'Phổ biến nhất',
      };
}
```

### 5.3.3. FutureProvider — Async Data Một Lần

`FutureProvider` wrap một `Future` và tự động handle các trạng thái loading/data/error. Dùng khi cần fetch data không cần refresh thủ công.

```dart
// ✅ CHUẨN — FutureProvider cho data fetch đơn giản

@riverpod
Future<List<Category>> categories(CategoriesRef ref) async {
  final repo = ref.watch(categoryRepositoryProvider);
  return repo.fetchCategories();
}

@riverpod
Future<ProductDetail> productDetail(
  ProductDetailRef ref,
  String productId, // Tham số cho family provider
) async {
  final repo = ref.watch(productRepositoryProvider);
  // ref.onDispose: cleanup khi provider bị dispose
  // Hữu ích để cancel HTTP request nếu user rời màn hình
  final cancelToken = CancelToken();
  ref.onDispose(() => cancelToken.cancel());
  return repo.fetchProductDetail(productId, cancelToken: cancelToken);
}

// Sử dụng trong widget với AsyncValue pattern:
class ProductDetailScreen extends ConsumerWidget {
  const ProductDetailScreen({super.key, required this.productId});
  final String productId;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // watch trả về AsyncValue<ProductDetail>
    final productAsync = ref.watch(productDetailProvider(productId));

    return Scaffold(
      appBar: AppBar(title: const Text('Chi tiết sản phẩm')),
      body: productAsync.when(
        // loading: Hiển thị skeleton
        loading: () => const ProductDetailSkeleton(),
        // error: Hiển thị error state với retry
        error: (error, stack) => ErrorView(
          message: error.toString(),
          onRetry: () => ref.invalidate(productDetailProvider(productId)),
        ),
        // data: Hiển thị nội dung
        data: (product) => ProductDetailContent(product: product),
      ),
    );
  }
}
```

### 5.3.4. StreamProvider — Realtime Data

`StreamProvider` dùng cho data realtime như Firestore listeners, WebSocket, hoặc bất kỳ Stream nào.

```dart
// ✅ CHUẨN — StreamProvider cho Firestore realtime

@riverpod
Stream<List<Notification>> userNotifications(UserNotificationsRef ref) {
  final userId = ref.watch(currentUserProvider)?.id;
  if (userId == null) return const Stream.empty();

  return FirebaseFirestore.instance
      .collection('notifications')
      .where('userId', isEqualTo: userId)
      .where('isRead', isEqualTo: false)
      .orderBy('createdAt', descending: true)
      .snapshots()
      .map((snap) => snap.docs
          .map((doc) => Notification.fromFirestore(doc))
          .toList());
}

// Sử dụng — y hệt FutureProvider, dùng .when()
class NotificationBadge extends ConsumerWidget {
  const NotificationBadge({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final notificationsAsync = ref.watch(userNotificationsProvider);

    return notificationsAsync.when(
      loading: () => const Icon(Icons.notifications_outlined),
      error: (_, __) => const Icon(Icons.notifications_outlined),
      data: (notifications) => Badge.count(
        count: notifications.length,
        isLabelVisible: notifications.isNotEmpty,
        child: const Icon(Icons.notifications_outlined),
      ),
    );
  }
}
```

---

## 5.4. NotifierProvider — State Phức Tạp Với Logic

`NotifierProvider` (và `AsyncNotifierProvider`) là loại provider quan trọng nhất cho business logic thực sự. Tương đương Zustand's store — một class quản lý state và expose các method để thay đổi state.

### 5.4.1. NotifierProvider — State Đồng Bộ

```dart
// ✅ CHUẨN — Cart với đầy đủ business logic
// lib/features/cart/providers/cart_provider.dart

import 'package:riverpod_annotation/riverpod_annotation.dart';
part 'cart_provider.g.dart';

// Model
class CartItem {
  const CartItem({
    required this.product,
    required this.quantity,
    this.selectedVariant,
  });

  final Product product;
  final int quantity;
  final ProductVariant? selectedVariant;

  double get subtotal => product.price * quantity;

  CartItem copyWith({int? quantity, ProductVariant? selectedVariant}) {
    return CartItem(
      product: product,
      quantity: quantity ?? this.quantity,
      selectedVariant: selectedVariant ?? this.selectedVariant,
    );
  }

  @override
  bool operator ==(Object other) =>
      other is CartItem && other.product.id == product.id;

  @override
  int get hashCode => product.id.hashCode;
}

// State
class CartState {
  const CartState({
    this.items = const [],
    this.couponCode,
    this.discountPercent = 0,
  });

  final List<CartItem> items;
  final String? couponCode;
  final int discountPercent;

  // Computed properties — tính toán từ state, không lưu trữ riêng
  int get totalQuantity => items.fold(0, (sum, item) => sum + item.quantity);
  double get subtotal => items.fold(0.0, (sum, item) => sum + item.subtotal);
  double get discount => subtotal * discountPercent / 100;
  double get total => subtotal - discount;
  bool get isEmpty => items.isEmpty;

  CartState copyWith({
    List<CartItem>? items,
    String? couponCode,
    int? discountPercent,
  }) {
    return CartState(
      items: items ?? this.items,
      couponCode: couponCode ?? this.couponCode,
      discountPercent: discountPercent ?? this.discountPercent,
    );
  }
}

// Notifier
@riverpod
class Cart extends _$Cart {
  @override
  CartState build() => const CartState(); // State khởi tạo

  // === Thêm sản phẩm ===
  void addItem(Product product, {int quantity = 1, ProductVariant? variant}) {
    final existingIndex = state.items.indexWhere(
      (item) => item.product.id == product.id,
    );

    if (existingIndex >= 0) {
      // Đã có trong giỏ: tăng số lượng
      final updatedItems = [...state.items];
      final existing = updatedItems[existingIndex];
      updatedItems[existingIndex] = existing.copyWith(
        quantity: existing.quantity + quantity,
      );
      state = state.copyWith(items: updatedItems);
    } else {
      // Chưa có: thêm mới
      state = state.copyWith(
        items: [...state.items, CartItem(product: product, quantity: quantity)],
      );
    }
  }

  // === Xóa sản phẩm ===
  void removeItem(String productId) {
    state = state.copyWith(
      items: state.items.where((i) => i.product.id != productId).toList(),
    );
  }

  // === Cập nhật số lượng ===
  void updateQuantity(String productId, int quantity) {
    if (quantity <= 0) {
      removeItem(productId);
      return;
    }

    state = state.copyWith(
      items: state.items.map((item) {
        if (item.product.id == productId) {
          return item.copyWith(quantity: quantity);
        }
        return item;
      }).toList(),
    );
  }

  // === Áp mã giảm giá ===
  void applyCoupon(String code) {
    // Logic validate coupon — thực tế sẽ gọi API
    final discount = _validateCoupon(code);
    state = state.copyWith(
      couponCode: code,
      discountPercent: discount,
    );
  }

  void removeCoupon() {
    state = state.copyWith(couponCode: null, discountPercent: 0);
  }

  // === Xóa toàn bộ giỏ ===
  void clear() => state = const CartState();

  int _validateCoupon(String code) {
    return switch (code.toUpperCase()) {
      'FLUTTER10' => 10,
      'FLUTTER20' => 20,
      _ => 0,
    };
  }
}

// === Derived providers — tính toán từ CartProvider ===
// Riverpod khuyến khích tách computed state thành provider riêng
// thay vì getter trong NotifierProvider

@riverpod
int cartItemCount(CartItemCountRef ref) {
  return ref.watch(cartProvider.select((state) => state.totalQuantity));
  // .select(): Chỉ rebuild khi totalQuantity thay đổi, không rebuild cho mọi thay đổi của cart
}

@riverpod
bool isInCart(IsInCartRef ref, String productId) {
  return ref.watch(
    cartProvider.select(
      (state) => state.items.any((item) => item.product.id == productId),
    ),
  );
}
```

### 5.4.2. AsyncNotifierProvider — State Bất Đồng Bộ

`AsyncNotifierProvider` dùng khi state cần load từ API lần đầu và sau đó có thể được cập nhật bởi các action.

```dart
// ✅ CHUẨN — AsyncNotifier cho list với pagination và actions
// lib/features/products/providers/product_list_provider.dart

@immutable
class ProductListState {
  const ProductListState({
    this.products = const [],
    this.currentPage = 1,
    this.hasMore = true,
    this.isLoadingMore = false,
  });

  final List<Product> products;
  final int currentPage;
  final bool hasMore;
  final bool isLoadingMore;

  ProductListState copyWith({
    List<Product>? products,
    int? currentPage,
    bool? hasMore,
    bool? isLoadingMore,
  }) =>
      ProductListState(
        products: products ?? this.products,
        currentPage: currentPage ?? this.currentPage,
        hasMore: hasMore ?? this.hasMore,
        isLoadingMore: isLoadingMore ?? this.isLoadingMore,
      );
}

@riverpod
class ProductList extends _$ProductList {
  static const _pageSize = 20;

  @override
  Future<ProductListState> build({String? category}) async {
    // Build: Load trang đầu tiên
    final repo = ref.watch(productRepositoryProvider);
    final result = await repo.fetchProducts(
      category: category,
      page: 1,
      pageSize: _pageSize,
    );

    return ProductListState(
      products: result.items,
      hasMore: result.items.length >= _pageSize,
    );
  }

  // Load thêm (pagination)
  Future<void> loadMore() async {
    final currentState = state.valueOrNull;
    if (currentState == null || !currentState.hasMore || currentState.isLoadingMore) {
      return;
    }

    // Cập nhật isLoadingMore mà không mất data hiện tại
    state = AsyncData(currentState.copyWith(isLoadingMore: true));

    try {
      final repo = ref.read(productRepositoryProvider);
      final nextPage = currentState.currentPage + 1;
      final result = await repo.fetchProducts(
        category: (state as AsyncData).value.products.first.category,
        page: nextPage,
        pageSize: _pageSize,
      );

      state = AsyncData(currentState.copyWith(
        products: [...currentState.products, ...result.items],
        currentPage: nextPage,
        hasMore: result.items.length >= _pageSize,
        isLoadingMore: false,
      ));
    } catch (e, st) {
      // Lỗi load more: giữ data cũ, tắt loading flag
      state = AsyncData(currentState.copyWith(isLoadingMore: false));
    }
  }

  // Refresh từ đầu
  Future<void> refresh() => ref.refresh(productListProvider(category: null).future);

  // Xóa product khỏi list sau khi delete
  void removeProduct(String productId) {
    final currentData = state.valueOrNull;
    if (currentData == null) return;

    state = AsyncData(currentData.copyWith(
      products: currentData.products
          .where((p) => p.id != productId)
          .toList(),
    ));
  }
}
```

---

## 5.5. ref.watch, ref.read, ref.listen

Đây là ba phương thức cốt lõi để tương tác với provider. Dùng sai sẽ gây bug hoặc hiệu năng kém.

```dart
// Trong ConsumerWidget — có sẵn WidgetRef từ tham số build()
class MyWidget extends ConsumerWidget {
  const MyWidget({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {

    // ✅ ref.watch — Lắng nghe và rebuild khi provider thay đổi
    // CHỈ dùng trong build() — không dùng trong callback/event handler
    final cart = ref.watch(cartProvider);
    final itemCount = ref.watch(cartItemCountProvider);

    // ✅ ref.read — Đọc giá trị MỘT LẦN, không subscribe
    // Dùng trong callback, event handler, initState
    // KHÔNG dùng trong build() — sẽ không rebuild khi thay đổi
    return ElevatedButton(
      onPressed: () {
        // Đúng: read trong callback
        ref.read(cartProvider.notifier).addItem(product);
      },
      child: Text('Thêm vào giỏ ($itemCount)'),
    );
  }
}

// ref.listen — Thực hiện side effect khi provider thay đổi
// Dùng để show snackbar, navigate, log — KHÔNG để rebuild UI
class CheckoutScreen extends ConsumerStatefulWidget {
  const CheckoutScreen({super.key});

  @override
  ConsumerState<CheckoutScreen> createState() => _CheckoutScreenState();
}

class _CheckoutScreenState extends ConsumerState<CheckoutScreen> {
  @override
  void initState() {
    super.initState();

    // listen trong initState (hoặc trong build với WidgetRef)
    // Callback nhận previous và next value
    ref.listenManual(
      orderSubmitProvider,
      (previous, next) {
        next.whenOrNull(
          data: (order) {
            // Navigate khi đặt hàng thành công
            context.go('/orders/${order.id}/success');
          },
          error: (error, _) {
            // Show lỗi
            ScaffoldMessenger.of(context).showSnackBar(
              SnackBar(content: Text(error.toString())),
            );
          },
        );
      },
    );
  }

  @override
  Widget build(BuildContext context) {
    // Trong build cũng có thể dùng ref.listen
    ref.listen(
      cartProvider.select((state) => state.isEmpty),
      (_, isEmpty) {
        if (isEmpty) context.go('/cart');
      },
    );

    return const CheckoutContent();
  }
}
```

**Tóm tắt quy tắc:**

| Method | Dùng ở đâu | Tác dụng |
|---|---|---|
| `ref.watch()` | Trong `build()` | Subscribe, rebuild khi thay đổi |
| `ref.read()` | Trong callback, event handler | Đọc một lần, không subscribe |
| `ref.listen()` | Trong `build()` hoặc `initState` | Side effect khi thay đổi |
| `ref.invalidate()` | Trong callback | Buộc provider rebuild lại từ đầu |
| `ref.refresh()` | Trong callback | Invalidate + đọc ngay giá trị mới |

---

## 5.6. select() — Tối Ưu Rebuild

`.select()` là kỹ thuật tối ưu quan trọng: chỉ rebuild widget khi một phần cụ thể của provider thay đổi, không rebuild cho mọi thay đổi.

```dart
// ❌ KHÔNG TỐI ƯU — Rebuild mỗi khi bất kỳ thứ gì trong cart thay đổi
// Kể cả khi chỉ đổi quantity của 1 item
final cart = ref.watch(cartProvider);
Text('${cart.totalQuantity} sản phẩm')

// ✅ TỐI ƯU — Chỉ rebuild khi totalQuantity thực sự đổi
final count = ref.watch(
  cartProvider.select((state) => state.totalQuantity),
);
Text('$count sản phẩm')

// ✅ TỐI ƯU — Badge chỉ rebuild khi count thay đổi
class CartBadge extends ConsumerWidget {
  const CartBadge({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // select trả về int — chỉ rebuild khi int này đổi
    final count = ref.watch(
      cartProvider.select((s) => s.totalQuantity),
    );
    return Badge.count(
      count: count,
      isLabelVisible: count > 0,
      child: const Icon(Icons.shopping_cart_outlined),
    );
  }
}
```

---

## 5.7. Provider Families — Provider Có Tham Số

Khi cần nhiều instance của cùng một provider với tham số khác nhau — ví dụ detail của từng product — dùng family pattern.

```dart
// Với riverpod_generator: thêm tham số vào hàm build() là đủ
@riverpod
Future<ProductDetail> productDetail(
  ProductDetailRef ref,
  String productId, // Đây là family parameter
) async {
  return ref.watch(productRepositoryProvider)
      .fetchProductDetail(productId);
}

// Sử dụng: truyền tham số khi watch
final detail = ref.watch(productDetailProvider('abc-123'));
final detail2 = ref.watch(productDetailProvider('xyz-456'));
// Hai provider này HOÀN TOÀN ĐỘC LẬP — khác cache, khác state

// Family với nhiều tham số — dùng record (Dart 3)
@riverpod
Future<List<Product>> filteredProducts(
  FilteredProductsRef ref, {
  required String category,
  required ProductSortOrder sort,
  int page = 1,
}) async {
  return ref.watch(productRepositoryProvider)
      .fetchProducts(category: category, sort: sort, page: page);
}

// Sử dụng:
final products = ref.watch(filteredProductsProvider(
  category: 'electronics',
  sort: ProductSortOrder.priceAsc,
  page: 2,
));
```

---

## 5.8. Pattern Thực Tế: Auth State

Authentication là use case phức tạp nhất, cần kết hợp nhiều loại provider.

```dart
// lib/features/auth/providers/auth_provider.dart

// Model
sealed class AuthState {
  const AuthState();
}
class AuthLoading extends AuthState { const AuthLoading(); }
class AuthAuthenticated extends AuthState {
  const AuthAuthenticated(this.user);
  final AppUser user;
}
class AuthUnauthenticated extends AuthState { const AuthUnauthenticated(); }

// Provider
@riverpod
class Auth extends _$Auth {
  @override
  Future<AuthState> build() async {
    // Kiểm tra session lưu ở local khi app khởi động
    final token = await ref.watch(secureStorageProvider)
        .read(key: 'access_token');

    if (token == null) return const AuthUnauthenticated();

    try {
      final user = await ref.watch(authRepositoryProvider)
          .getCurrentUser(token: token);
      return AuthAuthenticated(user);
    } catch (_) {
      // Token expired hoặc invalid
      await ref.read(secureStorageProvider).delete(key: 'access_token');
      return const AuthUnauthenticated();
    }
  }

  Future<void> login({
    required String email,
    required String password,
  }) async {
    state = const AsyncLoading();

    state = await AsyncValue.guard(() async {
      final result = await ref.read(authRepositoryProvider).login(
        email: email,
        password: password,
      );
      // Lưu token vào secure storage
      await ref.read(secureStorageProvider).write(
        key: 'access_token',
        value: result.accessToken,
      );
      return AuthAuthenticated(result.user);
    });
  }

  Future<void> logout() async {
    await ref.read(secureStorageProvider).delete(key: 'access_token');
    // Invalidate tất cả provider phụ thuộc vào user
    ref.invalidateSelf();
    // Xóa giỏ hàng khi logout
    ref.invalidate(cartProvider);
  }
}

// Convenience providers
@riverpod
AppUser? currentUser(CurrentUserRef ref) {
  final authState = ref.watch(authProvider).valueOrNull;
  return authState is AuthAuthenticated ? authState.user : null;
}

@riverpod
bool isAuthenticated(IsAuthenticatedRef ref) {
  return ref.watch(currentUserProvider) != null;
}
```

---

## 5.9. Bài Tập: Shopping Cart Hoàn Chỉnh

Xây dựng màn hình Cart hoàn chỉnh với:

```dart
// CartScreen sử dụng CartNotifier đã định nghĩa ở 5.4.1

class CartScreen extends ConsumerWidget {
  const CartScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final cart = ref.watch(cartProvider);

    if (cart.isEmpty) {
      return Scaffold(
        appBar: AppBar(title: const Text('Giỏ hàng')),
        body: Center(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              const Icon(Icons.shopping_cart_outlined, size: 64),
              const SizedBox(height: 16),
              Text('Giỏ hàng trống', style: context.textTheme.titleMedium),
              const SizedBox(height: 24),
              FilledButton(
                onPressed: () => context.go('/products'),
                child: const Text('Tiếp tục mua sắm'),
              ),
            ],
          ),
        ),
      );
    }

    return Scaffold(
      appBar: AppBar(
        title: Text('Giỏ hàng (${cart.totalQuantity})'),
        actions: [
          TextButton(
            onPressed: () => ref.read(cartProvider.notifier).clear(),
            child: const Text('Xóa tất cả'),
          ),
        ],
      ),
      body: Column(
        children: [
          Expanded(
            child: ListView.separated(
              padding: const EdgeInsets.all(16),
              itemCount: cart.items.length,
              separatorBuilder: (_, __) => const SizedBox(height: 12),
              itemBuilder: (context, index) {
                final item = cart.items[index];
                return CartItemTile(item: item);
              },
            ),
          ),
          _CartSummary(cart: cart),
        ],
      ),
    );
  }
}

class CartItemTile extends ConsumerWidget {
  const CartItemTile({super.key, required this.item});
  final CartItem item;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(12),
        child: Row(
          children: [
            AppNetworkImage(
              url: item.product.imageUrl,
              width: 80,
              height: 80,
              borderRadius: BorderRadius.circular(8),
            ),
            const SizedBox(width: 12),
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(item.product.name, style: context.textTheme.titleSmall),
                  const SizedBox(height: 4),
                  Text(item.product.formattedPrice,
                      style: context.textTheme.bodyMedium?.copyWith(
                        color: context.colorScheme.primary,
                      )),
                ],
              ),
            ),
            const SizedBox(width: 8),
            // Reuse QuantitySelector từ chương 2
            QuantitySelector(
              initialValue: item.quantity,
              onChanged: (qty) {
                ref.read(cartProvider.notifier).updateQuantity(
                  item.product.id,
                  qty,
                );
              },
            ),
            IconButton(
              icon: const Icon(Icons.close),
              onPressed: () =>
                  ref.read(cartProvider.notifier).removeItem(item.product.id),
            ),
          ],
        ),
      ),
    );
  }
}

class _CartSummary extends StatelessWidget {
  const _CartSummary({required this.cart});
  final CartState cart;

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(16),
      decoration: BoxDecoration(
        color: context.colorScheme.surface,
        border: Border(top: BorderSide(color: context.colorScheme.outlineVariant)),
      ),
      child: SafeArea(
        top: false,
        child: Column(
          children: [
            _SummaryRow('Tạm tính', cart.subtotal),
            if (cart.discountPercent > 0)
              _SummaryRow('Giảm giá (${cart.discountPercent}%)', -cart.discount,
                  color: context.colorScheme.error),
            const Divider(height: 24),
            _SummaryRow('Tổng cộng', cart.total, isTotal: true),
            const SizedBox(height: 16),
            FilledButton(
              onPressed: () => context.push('/checkout', extra: cart.items),
              style: FilledButton.styleFrom(
                minimumSize: const Size.fromHeight(52),
              ),
              child: Text('Đặt hàng • ${_formatPrice(cart.total)}'),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## Tóm Tắt Chương 5

| Khái niệm | Điểm Cốt Lõi |
|---|---|
| ProviderScope | Bọc ngoài cùng — container chứa tất cả provider |
| Provider | DI — expose singleton, không thay đổi |
| StateProvider | UI state đơn giản: bool, int, enum |
| FutureProvider | Async data một lần từ API |
| StreamProvider | Realtime data từ Firestore, WebSocket |
| NotifierProvider | Business logic + state phức tạp |
| AsyncNotifierProvider | Business logic + async state |
| ref.watch | Subscribe trong build() — tự rebuild |
| ref.read | Đọc một lần trong callback — không rebuild |
| ref.listen | Side effect: navigate, snackbar, log |
| .select() | Tối ưu rebuild — chỉ listen phần cần |
| Family | Provider có tham số — mỗi tham số là instance độc lập |

> **Nguyên tắc kiến trúc:** Provider không phải chỉ để quản lý state UI — chúng là lớp Dependency Injection của toàn app. Repository, Service, Config đều nên được expose qua Provider. Khi cần test, chỉ cần override provider bằng mock — không cần thay đổi widget code.
