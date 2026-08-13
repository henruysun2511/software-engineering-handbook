# Chương 4: Navigation với Go Router

---

## 4.1. Sự Tiến Hóa Của Navigation Trong Flutter

### 4.1.1. Navigator 1.0 — Imperative API

Flutter ban đầu cung cấp `Navigator` với API theo phong cách lệnh (imperative): `Navigator.push()`, `Navigator.pop()`, `Navigator.pushNamed()`. Cách tiếp cận này đơn giản nhưng có nhiều hạn chế nghiêm trọng khi app phức tạp lên:

```dart
// Navigator 1.0 — Cách cũ
// Vấn đề: không hỗ trợ deep link, URL phức tạp, web navigation
Navigator.push(
  context,
  MaterialPageRoute(
    builder: (_) => ProductDetailScreen(productId: '123'),
  ),
);

// pushNamed: tệ hơn — không type-safe, không truyền object phức tạp
Navigator.pushNamed(context, '/product/123');
```

**Vấn đề của Navigator 1.0:**
- Không có URL thực sự (web app không thể bookmark, share link)
- Deep link phức tạp và dễ lỗi
- Back button của browser/OS không hoạt động đúng
- Khó test navigation logic
- Không hỗ trợ nested navigation tốt

### 4.1.2. Navigator 2.0 — Declarative API

Flutter 1.22 giới thiệu Navigator 2.0 với API khai báo (declarative) mạnh hơn nhiều — nhưng cũng phức tạp đến mức hầu hết developer không dùng trực tiếp mà sử dụng các thư viện wrapper.

### 4.1.3. Go Router — The Standard Solution

**Go Router** là thư viện navigation chính thức của Flutter team, xây dựng trên Navigator 2.0. Từ 2023, đây là lựa chọn chuẩn được khuyến nghị trong Flutter documentation.

**Ưu điểm của Go Router:**
- URL-based routing: mỗi route có URL rõ ràng
- Deep link out-of-the-box
- Type-safe route (với go_router_builder)
- Nested navigation với ShellRoute
- Redirect/guard tích hợp sẵn
- Web support tốt

---

## 4.2. Cài Đặt và Cấu Hình Cơ Bản

### 4.2.1. Dependencies

```yaml
# pubspec.yaml
dependencies:
  go_router: ^14.0.0
```

### 4.2.2. Định Nghĩa Routes

Theo quy ước dự án thực tế, toàn bộ router config được đặt trong một file riêng.

```dart
// lib/core/router/app_router.dart

import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

// Đặt tên màn hình thành constant để tránh typo
abstract class AppRoutes {
  static const home = '/';
  static const products = '/products';
  static const productDetail = '/products/:id';
  static const cart = '/cart';
  static const checkout = '/checkout';
  static const profile = '/profile';
  static const login = '/login';
  static const register = '/register';
  static const orderHistory = '/orders';
  static const orderDetail = '/orders/:orderId';
  static const settings = '/settings';
}

final appRouter = GoRouter(
  initialLocation: AppRoutes.home,
  debugLogDiagnostics: true, // Log route changes trong debug mode

  routes: [
    // Route đơn giản không có tham số
    GoRoute(
      path: AppRoutes.home,
      name: 'home',
      builder: (context, state) => const HomeScreen(),
    ),

    GoRoute(
      path: AppRoutes.products,
      name: 'products',
      builder: (context, state) {
        // Query parameters: /products?category=electronics&sort=price
        final category = state.uri.queryParameters['category'];
        final sort = state.uri.queryParameters['sort'];
        return ProductListScreen(category: category, sort: sort);
      },
    ),

    // Route với path parameter: /products/abc-123
    GoRoute(
      path: AppRoutes.productDetail,
      name: 'product-detail',
      builder: (context, state) {
        // pathParameters: lấy từ :id trong path pattern
        final productId = state.pathParameters['id']!;
        return ProductDetailScreen(productId: productId);
      },
    ),

    GoRoute(
      path: AppRoutes.cart,
      name: 'cart',
      builder: (context, state) => const CartScreen(),
    ),

    GoRoute(
      path: AppRoutes.checkout,
      name: 'checkout',
      // Truyền object phức tạp qua extra
      // Lưu ý: extra không được lưu trong URL, mất khi deep link
      builder: (context, state) {
        final cartItems = state.extra as List<CartItem>?;
        return CheckoutScreen(items: cartItems ?? []);
      },
    ),

    GoRoute(
      path: AppRoutes.login,
      name: 'login',
      builder: (context, state) => const LoginScreen(),
      routes: [
        // Sub-route: /login/register
        GoRoute(
          path: 'register', // Tương đối với parent: /login/register
          name: 'register',
          builder: (context, state) => const RegisterScreen(),
        ),
      ],
    ),

    GoRoute(
      path: AppRoutes.orderHistory,
      name: 'orders',
      builder: (context, state) => const OrderHistoryScreen(),
      routes: [
        // /orders/:orderId
        GoRoute(
          path: ':orderId',
          name: 'order-detail',
          builder: (context, state) {
            final orderId = state.pathParameters['orderId']!;
            return OrderDetailScreen(orderId: orderId);
          },
        ),
      ],
    ),
  ],

  // Xử lý route không tồn tại
  errorBuilder: (context, state) => NotFoundScreen(
    message: 'Trang "${state.uri}" không tồn tại',
  ),
);

// Đăng ký vào MaterialApp
class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      routerConfig: appRouter, // Dùng MaterialApp.router thay vì MaterialApp
      theme: AppTheme.light,
      darkTheme: AppTheme.dark,
    );
  }
}
```

---

## 4.3. Navigation Methods — Cách Điều Hướng

```dart
// Trong bất kỳ widget nào có BuildContext:

// 1. go() — Navigate và THAY THẾ history
// Giống pushReplacement — không thể back về trang cũ
context.go(AppRoutes.home);
context.go('/products?category=electronics');

// 2. push() — Navigate và THÊM VÀO history (có thể back)
// Giống Navigator.push()
context.push('/products/abc-123');
context.push('/checkout', extra: cartItems);

// 3. pop() — Quay lại trang trước
context.pop();
context.pop(result); // Pop với return value

// 4. replace() — Thay thế trang hiện tại trong history
context.replace('/login');

// 5. pushReplacement() — Push mới và xóa trang hiện tại
context.pushReplacement('/home');

// 6. Lấy thông tin route hiện tại
final location = GoRouterState.of(context).uri.toString();

// 7. Kiểm tra có thể pop không
final canPop = context.canPop();
```

```dart
// ✅ CHUẨN — Navigation trong widget thực tế
class ProductCard extends StatelessWidget {
  const ProductCard({super.key, required this.product});
  final Product product;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () {
        // Navigate đến product detail
        // go() vì detail là destination, không cần back stack phức tạp
        context.push('/products/${product.id}');
      },
      child: Card(
        child: Column(
          children: [
            Image.network(product.imageUrl),
            Text(product.name),
            FilledButton(
              onPressed: () {
                // Push checkout với extra data
                context.push(
                  AppRoutes.checkout,
                  extra: [CartItem(product: product, quantity: 1)],
                );
              },
              child: const Text('Mua ngay'),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## 4.4. ShellRoute — Bottom Navigation Bar

`ShellRoute` là tính năng đặc biệt của Go Router cho phép giữ một shell widget (như `BottomNavigationBar`) cố định trong khi nội dung bên trong thay đổi. Đây là pattern chuẩn cho app có bottom nav.

```dart
// lib/core/router/app_router.dart — Version với ShellRoute

final appRouter = GoRouter(
  initialLocation: AppRoutes.home,
  routes: [
    // ShellRoute: Shell chứa BottomNavigationBar
    ShellRoute(
      // builder nhận navigatorKey và child (nội dung tab hiện tại)
      builder: (context, state, child) {
        return MainShell(child: child);
      },
      routes: [
        // Các tab trong Shell
        GoRoute(
          path: AppRoutes.home,
          builder: (context, state) => const HomeScreen(),
        ),
        GoRoute(
          path: AppRoutes.products,
          builder: (context, state) => const ProductListScreen(),
        ),
        GoRoute(
          path: AppRoutes.cart,
          builder: (context, state) => const CartScreen(),
        ),
        GoRoute(
          path: AppRoutes.profile,
          builder: (context, state) => const ProfileScreen(),
        ),
      ],
    ),

    // Routes ngoài Shell (không có bottom nav)
    GoRoute(
      path: AppRoutes.productDetail,
      builder: (context, state) {
        final id = state.pathParameters['id']!;
        return ProductDetailScreen(productId: id);
      },
    ),
    GoRoute(
      path: AppRoutes.login,
      builder: (context, state) => const LoginScreen(),
    ),
    GoRoute(
      path: AppRoutes.checkout,
      builder: (context, state) => const CheckoutScreen(),
    ),
  ],
);

// Shell widget chứa BottomNavigationBar
class MainShell extends StatelessWidget {
  const MainShell({super.key, required this.child});
  final Widget child;

  // Map từ route path sang tab index
  static const _tabs = [
    AppRoutes.home,
    AppRoutes.products,
    AppRoutes.cart,
    AppRoutes.profile,
  ];

  int _currentIndex(BuildContext context) {
    final location = GoRouterState.of(context).uri.toString();
    // Tìm tab tương ứng với location hiện tại
    final index = _tabs.indexWhere(
      (tab) => location.startsWith(tab),
    );
    return index < 0 ? 0 : index;
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: child, // Nội dung tab
      bottomNavigationBar: NavigationBar(
        selectedIndex: _currentIndex(context),
        onDestinationSelected: (index) {
          context.go(_tabs[index]);
        },
        destinations: const [
          NavigationDestination(
            icon: Icon(Icons.home_outlined),
            selectedIcon: Icon(Icons.home),
            label: 'Trang chủ',
          ),
          NavigationDestination(
            icon: Icon(Icons.grid_view_outlined),
            selectedIcon: Icon(Icons.grid_view),
            label: 'Sản phẩm',
          ),
          NavigationDestination(
            icon: Icon(Icons.shopping_cart_outlined),
            selectedIcon: Icon(Icons.shopping_cart),
            label: 'Giỏ hàng',
          ),
          NavigationDestination(
            icon: Icon(Icons.person_outlined),
            selectedIcon: Icon(Icons.person),
            label: 'Tôi',
          ),
        ],
      ),
    );
  }
}
```

---

## 4.5. Redirect và Route Guard

Route guard là kỹ thuật chặn và redirect navigation dựa trên điều kiện — phổ biến nhất là kiểm tra authentication: user chưa đăng nhập thì không cho vào màn hình cần auth.

### 4.5.1. Redirect Đơn Giản

```dart
// ✅ CHUẨN — Redirect trong GoRouter
final appRouter = GoRouter(
  initialLocation: AppRoutes.home,

  // redirect: function được gọi TRƯỚC KHI navigate đến bất kỳ route nào
  // Trả về null: cho phép navigate
  // Trả về String (path): redirect đến path đó
  redirect: (context, state) {
    // Lấy auth state từ simple singleton hoặc global state
    final isLoggedIn = AuthService.instance.isLoggedIn;
    final isGoingToAuth = state.matchedLocation == AppRoutes.login ||
        state.matchedLocation.startsWith('/login');

    // Chưa login và không đi tới auth pages → redirect về login
    if (!isLoggedIn && !isGoingToAuth) {
      // Lưu intended destination để redirect sau khi login
      final redirectPath = state.uri.toString();
      return '${AppRoutes.login}?redirect=${Uri.encodeComponent(redirectPath)}';
    }

    // Đã login mà cố vào login page → redirect về home
    if (isLoggedIn && isGoingToAuth) {
      return AppRoutes.home;
    }

    // Không redirect
    return null;
  },

  routes: [...],
);
```

### 4.5.2. Route Guard Với Riverpod (Pattern Thực Tế)

Trong dự án thực tế, auth state được quản lý bởi state management (Riverpod). Go Router cần được refresh khi auth state thay đổi.

```dart
// lib/core/router/app_router.dart
// Pattern này giải thích concept; Riverpod sẽ học ở chương 5

// Sử dụng ref để lắng nghe auth state thay đổi
GoRouter buildRouter(Ref ref) {
  // refreshListenable: GoRouter tự gọi lại redirect khi listenable thay đổi
  // Đây là cách kết nối Riverpod với GoRouter
  final authNotifier = ref.read(authStateNotifierProvider.notifier);

  return GoRouter(
    initialLocation: AppRoutes.home,
    // refreshListenable cần một Listenable, dùng ChangeNotifier wrapper
    refreshListenable: GoRouterRefreshStream(
      ref.watch(authStateChangesProvider.stream),
    ),

    redirect: (context, state) async {
      final authState = ref.read(authStateProvider);

      return authState.when(
        loading: () => null,        // Đang load auth state: không redirect
        authenticated: (_) {
          // Đã auth: nếu ở login page thì về home
          if (state.matchedLocation.startsWith('/login')) {
            return AppRoutes.home;
          }
          return null;
        },
        unauthenticated: () {
          // Chưa auth: nếu ở protected route thì về login
          final isProtected = _protectedRoutes.any(
            (route) => state.matchedLocation.startsWith(route),
          );
          if (isProtected) {
            return AppRoutes.login;
          }
          return null;
        },
      );
    },

    routes: [...],
  );
}

// Danh sách route cần authentication
const _protectedRoutes = [
  AppRoutes.cart,
  AppRoutes.checkout,
  AppRoutes.orderHistory,
  AppRoutes.profile,
  '/settings',
];

// Helper: Chuyển Stream thành Listenable cho GoRouter
class GoRouterRefreshStream extends ChangeNotifier {
  GoRouterRefreshStream(Stream<dynamic> stream) {
    notifyListeners();
    _subscription = stream.listen((_) => notifyListeners());
  }

  late final StreamSubscription<dynamic> _subscription;

  @override
  void dispose() {
    _subscription.cancel();
    super.dispose();
  }
}
```

### 4.5.3. Xử Lý Sau Khi Login — Redirect Back

```dart
// LoginScreen: Lấy redirect parameter từ URL và navigate sau khi login thành công
class LoginScreen extends StatelessWidget {
  const LoginScreen({super.key});

  @override
  Widget build(BuildContext context) {
    // Lấy redirect path từ query parameter
    // URL: /login?redirect=%2Fcheckout
    final redirectTo = GoRouterState.of(context)
        .uri
        .queryParameters['redirect'];

    return Scaffold(
      body: LoginForm(
        onLoginSuccess: () {
          if (redirectTo != null && redirectTo.isNotEmpty) {
            // Navigate đến intended destination sau khi login
            context.go(Uri.decodeComponent(redirectTo));
          } else {
            context.go(AppRoutes.home);
          }
        },
      ),
    );
  }
}
```

---

## 4.6. Page Transitions Tùy Chỉnh

```dart
// ✅ CHUẨN — Custom page transition với CustomTransitionPage
GoRoute(
  path: AppRoutes.productDetail,
  pageBuilder: (context, state) {
    final productId = state.pathParameters['id']!;

    return CustomTransitionPage(
      key: state.pageKey,
      child: ProductDetailScreen(productId: productId),
      // Slide up transition (như iOS bottom sheet)
      transitionsBuilder: (context, animation, secondaryAnimation, child) {
        return SlideTransition(
          position: Tween<Offset>(
            begin: const Offset(0, 1),   // Từ dưới lên
            end: Offset.zero,
          ).animate(CurvedAnimation(
            parent: animation,
            curve: Curves.easeOutCubic,
          )),
          child: child,
        );
      },
      transitionDuration: const Duration(milliseconds: 300),
    );
  },
),

// ✅ CHUẨN — Fade transition cho modals
GoRoute(
  path: '/search',
  pageBuilder: (context, state) => CustomTransitionPage(
    key: state.pageKey,
    child: const SearchScreen(),
    transitionsBuilder: (context, animation, secondaryAnimation, child) {
      return FadeTransition(opacity: animation, child: child);
    },
  ),
),
```

---

## 4.7. Deep Link Configuration

### 4.7.1. Android

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<activity>
  <!-- Thêm intent-filter cho deep link -->
  <intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <!-- Thay your.domain.com bằng domain thực -->
    <data
      android:scheme="https"
      android:host="your.domain.com" />
  </intent-filter>
</activity>
```

### 4.7.2. iOS

```xml
<!-- ios/Runner/Info.plist -->
<key>FlutterDeepLinkingEnabled</key>
<true/>
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>myapp</string>  <!-- Custom scheme: myapp://products/123 -->
    </array>
  </dict>
</array>
```

### 4.7.3. Test Deep Link

```bash
# Android: Mở deep link từ terminal
adb shell am start \
  -a android.intent.action.VIEW \
  -d "https://your.domain.com/products/abc123" \
  com.example.yourapp

# iOS Simulator:
xcrun simctl openurl booted "https://your.domain.com/products/abc123"
```

---

## 4.8. Bài Tập: Router Hoàn Chỉnh Cho E-Commerce App

Áp dụng tất cả kiến thức vào một router hoàn chỉnh cho app mua sắm:

```dart
// ✅ CHUẨN — Router thực tế hoàn chỉnh

final appRouter = GoRouter(
  initialLocation: '/',
  debugLogDiagnostics: kDebugMode, // Chỉ log trong debug

  redirect: (context, state) {
    // Giả sử có simple auth check
    final isAuth = AuthService.isLoggedIn;
    final loc = state.matchedLocation;

    if (!isAuth && _needsAuth(loc)) {
      return '/login?redirect=${Uri.encodeComponent(loc)}';
    }
    if (isAuth && loc.startsWith('/login')) {
      return '/';
    }
    return null;
  },

  routes: [
    // === SHELL: Màn hình có Bottom Navigation ===
    ShellRoute(
      builder: (_, __, child) => AppShell(child: child),
      routes: [
        GoRoute(path: '/', builder: (_, __) => const HomeScreen()),
        GoRoute(
          path: '/products',
          builder: (c, s) => ProductListScreen(
            category: s.uri.queryParameters['category'],
          ),
        ),
        GoRoute(path: '/cart', builder: (_, __) => const CartScreen()),
        GoRoute(path: '/profile', builder: (_, __) => const ProfileScreen()),
      ],
    ),

    // === FULL SCREEN: Không có Bottom Navigation ===
    GoRoute(
      path: '/products/:id',
      builder: (_, s) => ProductDetailScreen(
        productId: s.pathParameters['id']!,
      ),
    ),

    GoRoute(
      path: '/checkout',
      builder: (_, s) => CheckoutScreen(
        items: s.extra as List<CartItem>? ?? [],
      ),
      routes: [
        GoRoute(
          path: 'address',
          builder: (_, __) => const AddressSelectionScreen(),
        ),
        GoRoute(
          path: 'payment',
          builder: (_, __) => const PaymentScreen(),
        ),
        GoRoute(
          path: 'success',
          builder: (_, s) => OrderSuccessScreen(
            orderId: s.uri.queryParameters['orderId'] ?? '',
          ),
        ),
      ],
    ),

    GoRoute(
      path: '/orders/:id',
      builder: (_, s) => OrderDetailScreen(
        orderId: s.pathParameters['id']!,
      ),
    ),

    GoRoute(
      path: '/login',
      builder: (_, __) => const LoginScreen(),
      routes: [
        GoRoute(path: 'register', builder: (_, __) => const RegisterScreen()),
        GoRoute(path: 'forgot', builder: (_, __) => const ForgotPasswordScreen()),
      ],
    ),

    GoRoute(
      path: '/search',
      pageBuilder: (_, s) => NoTransitionPage(
        child: SearchScreen(
          initialQuery: s.uri.queryParameters['q'],
        ),
      ),
    ),
  ],

  errorBuilder: (_, s) => ErrorScreen(error: s.error),
);

bool _needsAuth(String location) {
  const protectedPaths = ['/cart', '/checkout', '/orders', '/profile'];
  return protectedPaths.any(location.startsWith);
}
```

---

## Tóm Tắt Chương 4

| Khái niệm | Điểm Cốt Lõi |
|---|---|
| Go Router vs Navigator 1.0 | URL-based, deep link, type-safe, web support tốt hơn nhiều |
| GoRoute | Đơn vị cơ bản — path, name, builder/pageBuilder |
| Path parameters | `:id` trong path → `state.pathParameters['id']` |
| Query parameters | `?key=value` → `state.uri.queryParameters['key']` |
| Extra | Truyền object phức tạp, không lưu trong URL |
| ShellRoute | Pattern chuẩn cho bottom navigation — shell cố định, content thay đổi |
| redirect | Guard logic — null cho phép, String path để redirect |
| refreshListenable | Kết nối state management với router để re-evaluate redirect |
| context.go() | Navigate không thể back — dùng cho tab switch, login success |
| context.push() | Navigate có back stack — dùng cho detail, modal |

> **Nguyên tắc thiết kế route:** Mỗi màn hình là một URL. Nếu không thể mô tả màn hình bằng URL, đó là dấu hiệu route chưa được thiết kế tốt. URL tốt: `/orders/123`, `/products?category=electronics&page=2`. URL xấu: dùng `extra` cho data mà người dùng có thể muốn share hoặc bookmark.
