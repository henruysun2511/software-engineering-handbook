# Chương 3: Styling & Theming

---

## 3.1. Triết Lý Styling Trong Flutter

### 3.1.1. Flutter Không Dùng Stylesheet Toàn Cục

Một trong những điểm khác biệt lớn nhất giữa Flutter và web (CSS) hoặc React Native (StyleSheet) là Flutter **không có khái niệm stylesheet toàn cục**. Không có file CSS, không có StyleSheet.create(), không có class name. Mọi style đều được khai báo trực tiếp trên widget hoặc thông qua `ThemeData` — một đối tượng cấu hình theme duy nhất được inject vào toàn bộ widget tree thông qua `InheritedWidget`.

Đây là thiết kế cố ý với lợi ích rõ ràng: style luôn gắn liền với widget, không có vấn đề về specificity hay cascade như CSS, và compiler có thể phát hiện lỗi type ngay lúc build.

### 3.1.2. Hai Nguồn Style

Mọi style trong Flutter đến từ một trong hai nguồn:

1. **Theme (toàn cục):** `Theme.of(context).colorScheme`, `Theme.of(context).textTheme` — định nghĩa bảng màu và typography chuẩn của toàn app.
2. **Inline style (cục bộ):** `TextStyle(...)`, `BoxDecoration(...)`, `EdgeInsets(...)` — override hoặc bổ sung cho từng widget cụ thể.

**Nguyên tắc:** Ưu tiên dùng theme. Chỉ inline style khi widget cần style đặc biệt không thuộc hệ thống design.

---

## 3.2. ThemeData và Material 3

### 3.2.1. Cấu Trúc ThemeData

`ThemeData` là class cấu hình theme trung tâm. Khi Flutter 3.x ra mắt Material 3 (còn gọi là Material You), hệ thống màu và component được thiết kế lại toàn bộ xung quanh hai khái niệm cốt lõi: **ColorScheme** và **TextTheme**.

```dart
// ✅ CHUẨN — Thiết lập theme hoàn chỉnh trong MaterialApp
import 'package:flutter/material.dart';

class AppTheme {
  AppTheme._(); // Prevent instantiation

  // Seed color: Flutter tự generate toàn bộ color palette từ màu này
  // Đây là điểm mạnh của Material 3 — developer chỉ cần chọn 1 màu
  static const _seedColor = Color(0xFF6750A4); // Tím Material

  static ThemeData get light {
    final colorScheme = ColorScheme.fromSeed(
      seedColor: _seedColor,
      brightness: Brightness.light,
    );
    return _buildTheme(colorScheme);
  }

  static ThemeData get dark {
    final colorScheme = ColorScheme.fromSeed(
      seedColor: _seedColor,
      brightness: Brightness.dark,
    );
    return _buildTheme(colorScheme);
  }

  static ThemeData _buildTheme(ColorScheme colorScheme) {
    return ThemeData(
      useMaterial3: true,          // Bật Material 3
      colorScheme: colorScheme,

      // Typography — dùng font tùy chỉnh
      fontFamily: 'Inter',
      textTheme: _buildTextTheme(),

      // Component themes — override style mặc định
      appBarTheme: AppBarTheme(
        centerTitle: false,        // Android style: title trái
        elevation: 0,
        scrolledUnderElevation: 1,
        backgroundColor: colorScheme.surface,
        foregroundColor: colorScheme.onSurface,
      ),

      cardTheme: CardTheme(
        elevation: 0,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(12),
          side: BorderSide(color: colorScheme.outlineVariant),
        ),
      ),

      inputDecorationTheme: InputDecorationTheme(
        filled: true,
        fillColor: colorScheme.surfaceContainerHighest,
        contentPadding: const EdgeInsets.symmetric(
          horizontal: 16,
          vertical: 12,
        ),
        border: OutlineInputBorder(
          borderRadius: BorderRadius.circular(12),
          borderSide: BorderSide.none, // Không border khi unfocused
        ),
        focusedBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(12),
          borderSide: BorderSide(color: colorScheme.primary, width: 2),
        ),
      ),

      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          minimumSize: const Size.fromHeight(48), // Full-width height
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(12),
          ),
        ),
      ),

      filledButtonTheme: FilledButtonThemeData(
        style: FilledButton.styleFrom(
          minimumSize: const Size.fromHeight(48),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(12),
          ),
        ),
      ),
    );
  }

  static TextTheme _buildTextTheme() {
    // Không cần khai báo tất cả — chỉ override style cần thay đổi
    // Flutter sẽ tự điền các style còn lại theo Material 3 spec
    return const TextTheme(
      displayLarge: TextStyle(fontSize: 57, fontWeight: FontWeight.w400),
      displayMedium: TextStyle(fontSize: 45, fontWeight: FontWeight.w400),
      displaySmall: TextStyle(fontSize: 36, fontWeight: FontWeight.w400),
      headlineLarge: TextStyle(fontSize: 32, fontWeight: FontWeight.w600),
      headlineMedium: TextStyle(fontSize: 28, fontWeight: FontWeight.w600),
      headlineSmall: TextStyle(fontSize: 24, fontWeight: FontWeight.w600),
      titleLarge: TextStyle(fontSize: 22, fontWeight: FontWeight.w500),
      titleMedium: TextStyle(fontSize: 16, fontWeight: FontWeight.w500),
      titleSmall: TextStyle(fontSize: 14, fontWeight: FontWeight.w500),
      bodyLarge: TextStyle(fontSize: 16, fontWeight: FontWeight.w400),
      bodyMedium: TextStyle(fontSize: 14, fontWeight: FontWeight.w400),
      bodySmall: TextStyle(fontSize: 12, fontWeight: FontWeight.w400),
      labelLarge: TextStyle(fontSize: 14, fontWeight: FontWeight.w500),
      labelMedium: TextStyle(fontSize: 12, fontWeight: FontWeight.w500),
      labelSmall: TextStyle(fontSize: 11, fontWeight: FontWeight.w500),
    );
  }
}

// Đăng ký vào MaterialApp
class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter App',
      theme: AppTheme.light,
      darkTheme: AppTheme.dark,
      themeMode: ThemeMode.system, // Theo system setting
      home: const HomeScreen(),
    );
  }
}
```

---

### 3.2.2. ColorScheme — Hệ Thống Màu Material 3

Material 3 định nghĩa một hệ thống màu với **29 màu có tên** (roles) thay vì tập màu tuỳ tiện. Mỗi màu có một vai trò cụ thể, và chúng được sinh tự động từ seed color. Hiểu đúng ColorScheme giúp design nhất quán và đẹp tự động.

```
ColorScheme — Cấu trúc chính:

primary          → Màu chủ đạo: button, icon chọn, indicator
onPrimary        → Text/icon trên primary background
primaryContainer → Background nhẹ hơn: chip selected, card accent
onPrimaryContainer → Text/icon trên primaryContainer

secondary        → Màu thứ cấp: accent nhẹ hơn
tertiary         → Màu điểm nhấn thứ ba

surface          → Background của card, sheet, dialog
onSurface        → Text/icon mặc định
surfaceContainerHighest → Input fill, chip fill
surfaceContainerHigh    → Overlay nhẹ

outline          → Border mặc định
outlineVariant   → Border nhẹ hơn: card border

error            → Trạng thái lỗi
onError          → Text trên error
```

```dart
// ✅ CHUẨN — Sử dụng ColorScheme đúng cách
class StatusBadge extends StatelessWidget {
  const StatusBadge({super.key, required this.status});
  final OrderStatus status;

  @override
  Widget build(BuildContext context) {
    final colorScheme = Theme.of(context).colorScheme;

    final (label, bgColor, textColor) = switch (status) {
      OrderStatus.pending => (
          'Chờ xử lý',
          colorScheme.tertiaryContainer,
          colorScheme.onTertiaryContainer,
        ),
      OrderStatus.processing => (
          'Đang xử lý',
          colorScheme.primaryContainer,
          colorScheme.onPrimaryContainer,
        ),
      OrderStatus.shipped => (
          'Đang giao',
          colorScheme.secondaryContainer,
          colorScheme.onSecondaryContainer,
        ),
      OrderStatus.delivered => (
          'Đã giao',
          // Không có successContainer trong M3 — tự định nghĩa hoặc dùng tertiary
          const Color(0xFFD4EDD7),
          const Color(0xFF1B5E20),
        ),
      OrderStatus.cancelled => (
          'Đã huỷ',
          colorScheme.errorContainer,
          colorScheme.onErrorContainer,
        ),
    };

    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 10, vertical: 4),
      decoration: BoxDecoration(
        color: bgColor,
        borderRadius: BorderRadius.circular(20),
      ),
      child: Text(
        label,
        style: Theme.of(context).textTheme.labelSmall?.copyWith(
          color: textColor,
          fontWeight: FontWeight.w600,
        ),
      ),
    );
  }
}

enum OrderStatus { pending, processing, shipped, delivered, cancelled }
```

---

### 3.2.3. TextTheme — Hệ Thống Typography

Material 3 định nghĩa **15 text style** với tên rõ ràng thay vì kích thước tùy tiện. Sử dụng đúng text style giúp hierarchy nội dung rõ ràng và responsive.

```dart
// ✅ CHUẨN — Sử dụng TextTheme đúng semantic
Widget _buildArticleCard(BuildContext context, Article article) {
  final textTheme = Theme.of(context).textTheme;

  return Column(
    crossAxisAlignment: CrossAxisAlignment.start,
    children: [
      // headlineSmall: Tiêu đề bài viết — nổi bật nhất
      Text(
        article.title,
        style: textTheme.headlineSmall,
        maxLines: 2,
        overflow: TextOverflow.ellipsis,
      ),
      const SizedBox(height: 8),

      // bodyMedium: Nội dung chính — đọc dễ
      Text(
        article.summary,
        style: textTheme.bodyMedium?.copyWith(
          color: Theme.of(context).colorScheme.onSurfaceVariant,
        ),
        maxLines: 3,
        overflow: TextOverflow.ellipsis,
      ),
      const SizedBox(height: 12),

      // labelSmall: Metadata — ít quan trọng nhất
      Text(
        '${article.author} · ${article.readTime} phút đọc',
        style: textTheme.labelSmall?.copyWith(
          color: Theme.of(context).colorScheme.outline,
        ),
      ),
    ],
  );
}
```

**Bảng mapping TextTheme sang use case:**

| Text Style | Font Size | Dùng cho |
|---|---|---|
| `displayLarge/Medium/Small` | 57/45/36px | Hero text, landing page |
| `headlineLarge/Medium/Small` | 32/28/24px | Tiêu đề trang, section header |
| `titleLarge/Medium/Small` | 22/16/14px | Card title, dialog title, app bar |
| `bodyLarge/Medium/Small` | 16/14/12px | Nội dung chính |
| `labelLarge/Medium/Small` | 14/12/11px | Button, chip, metadata, caption |

---

## 3.3. Dark Mode và Dynamic Theme Switching

### 3.3.1. Cấu Hình Dark Mode

Flutter hỗ trợ dark mode ở cấp framework. Khi `ThemeMode.system` được đặt, Flutter tự động chọn light hoặc dark theme theo setting của OS. `ThemeMode.dark` và `ThemeMode.light` để override.

```dart
// ✅ CHUẨN — ThemeMode được quản lý bởi state/provider
// Ví dụ này dùng Riverpod (sẽ học chi tiết ở chương 5)
// Hiểu ý tưởng: ThemeMode là state, widget lắng nghe và rebuild khi đổi

class MyApp extends ConsumerWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // Lấy ThemeMode từ state management
    final themeMode = ref.watch(themeModeProvider);

    return MaterialApp(
      theme: AppTheme.light,
      darkTheme: AppTheme.dark,
      themeMode: themeMode, // Đây là nơi duy nhất cần đổi
      home: const HomeScreen(),
    );
  }
}
```

### 3.3.2. Theme-Aware Widget

Khi viết custom widget, **luôn lấy màu từ Theme**, không bao giờ hardcode màu hex trực tiếp. Đây là nguyên tắc để dark mode hoạt động tự động.

```dart
// ❌ SAI — Hardcode màu, vỡ dark mode
Container(
  color: const Color(0xFFFFFFFF),   // Trắng cứng — vô hình trong dark mode
  child: Text(
    'Hello',
    style: TextStyle(color: Color(0xFF000000)), // Đen cứng — vô hình trong dark
  ),
)

// ✅ ĐÚNG — Lấy từ theme, dark mode tự hoạt động
Container(
  color: Theme.of(context).colorScheme.surface,
  child: Text(
    'Hello',
    style: TextStyle(
      color: Theme.of(context).colorScheme.onSurface,
    ),
  ),
)

// ✅ CÒN NGẮN HƠN — Dùng extension (sẽ định nghĩa dưới)
Container(
  color: context.colorScheme.surface,
  child: Text(
    'Hello',
    style: TextStyle(color: context.colorScheme.onSurface),
  ),
)
```

### 3.3.3. Extension Tiện Lợi cho Theme

```dart
// ✅ CHUẨN — Extension để truy cập theme ngắn gọn hơn
// Đặt trong lib/core/extensions/context_extensions.dart

extension BuildContextExtensions on BuildContext {
  ThemeData get theme => Theme.of(this);
  ColorScheme get colorScheme => Theme.of(this).colorScheme;
  TextTheme get textTheme => Theme.of(this).textTheme;
  MediaQueryData get mediaQuery => MediaQuery.of(this);
  Size get screenSize => MediaQuery.sizeOf(this);
  double get screenWidth => MediaQuery.sizeOf(this).width;
  double get screenHeight => MediaQuery.sizeOf(this).height;
  bool get isDarkMode => Theme.of(this).brightness == Brightness.dark;
}

// Sử dụng:
Text(
  'Hello',
  style: context.textTheme.bodyLarge?.copyWith(
    color: context.colorScheme.primary,
  ),
)
```

---

## 3.4. BoxDecoration — Styling Container

`BoxDecoration` là công cụ styling chính cho `Container` và `DecoratedBox`. Nó bao gồm background, border, border radius, shadow và gradient.

```dart
// ✅ CHUẨN — BoxDecoration toàn diện
class GlassCard extends StatelessWidget {
  const GlassCard({
    super.key,
    required this.child,
    this.padding = const EdgeInsets.all(16),
  });

  final Widget child;
  final EdgeInsets padding;

  @override
  Widget build(BuildContext context) {
    final colorScheme = context.colorScheme;
    final isDark = context.isDarkMode;

    return Container(
      padding: padding,
      decoration: BoxDecoration(
        // Background solid từ theme
        color: isDark
            ? colorScheme.surfaceContainerHigh
            : colorScheme.surfaceContainerLowest,

        // Border radius đồng nhất
        borderRadius: BorderRadius.circular(16),

        // Border nhẹ
        border: Border.all(
          color: colorScheme.outlineVariant,
          width: 1,
        ),

        // Box shadow — chỉ dùng ở light mode
        boxShadow: isDark
            ? null
            : [
                BoxShadow(
                  color: colorScheme.shadow.withOpacity(0.06),
                  blurRadius: 12,
                  offset: const Offset(0, 4),
                ),
              ],
      ),
      child: child,
    );
  }
}

// ✅ CHUẨN — Gradient background cho hero section
class HeroSection extends StatelessWidget {
  const HeroSection({super.key, required this.child});
  final Widget child;

  @override
  Widget build(BuildContext context) {
    final colorScheme = context.colorScheme;

    return DecoratedBox(
      decoration: BoxDecoration(
        gradient: LinearGradient(
          begin: Alignment.topLeft,
          end: Alignment.bottomRight,
          colors: [
            colorScheme.primaryContainer,
            colorScheme.secondaryContainer,
          ],
          stops: const [0.0, 1.0],
        ),
      ),
      child: child,
    );
  }
}
```

### 3.4.1. So Sánh Các Widget Styling

| Widget | Khi nào dùng |
|---|---|
| `Container` | Cần decoration + padding + size — widget đa năng nhất |
| `DecoratedBox` | Chỉ cần decoration, không cần sizing — nhẹ hơn Container |
| `ColoredBox` | Chỉ cần background color — nhẹ nhất, tối ưu nhất |
| `Padding` | Chỉ cần padding — không tạo thêm RenderObject |
| `SizedBox` | Chỉ cần kích thước cố định hoặc tạo khoảng trống |

```dart
// ❌ KHÔNG TỐI ƯU — Dùng Container chỉ để tạo màu nền
Container(
  color: Colors.blue,
  child: child,
)

// ✅ TỐI ƯU HƠN — ColoredBox cho màu đơn thuần
ColoredBox(
  color: Colors.blue,
  child: child,
)
```

---

## 3.5. Responsive Layout

### 3.5.1. MediaQuery — Thông Tin Màn Hình

`MediaQuery` cung cấp thông tin về kích thước màn hình, orientation, text scale factor và nhiều thuộc tính khác của device.

```dart
// ✅ CHUẨN — Responsive layout với MediaQuery
class AdaptiveHomePage extends StatelessWidget {
  const AdaptiveHomePage({super.key});

  @override
  Widget build(BuildContext context) {
    // Dùng MediaQuery.sizeOf thay vì MediaQuery.of(context).size
    // để chỉ rebuild khi SIZE thay đổi, không phải mọi property của MediaQuery
    final screenWidth = MediaQuery.sizeOf(context).width;

    // Điểm breakpoint quy ước
    // <600px  → Mobile
    // 600-840 → Tablet compact
    // >840px  → Tablet / Desktop
    final isTablet = screenWidth >= 600;
    final isDesktop = screenWidth >= 840;

    if (isDesktop) {
      return const _DesktopLayout();
    } else if (isTablet) {
      return const _TabletLayout();
    } else {
      return const _MobileLayout();
    }
  }
}
```

### 3.5.2. LayoutBuilder — Responsive Theo Parent

`LayoutBuilder` khác `MediaQuery` ở chỗ nó phản hồi với **kích thước parent widget**, không phải toàn màn hình. Phù hợp khi widget được dùng ở nhiều context khác nhau.

```dart
// ✅ CHUẨN — LayoutBuilder cho component tự responsive
class AdaptiveProductCard extends StatelessWidget {
  const AdaptiveProductCard({super.key, required this.product});
  final Product product;

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        // Nếu container rộng hơn 400px, dùng layout ngang
        if (constraints.maxWidth > 400) {
          return _HorizontalCard(product: product);
        }
        // Ngược lại, layout dọc mặc định
        return _VerticalCard(product: product);
      },
    );
  }
}

class _HorizontalCard extends StatelessWidget {
  const _HorizontalCard({required this.product});
  final Product product;

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Row(
        children: [
          // Ảnh cố định chiều rộng
          SizedBox(
            width: 120,
            height: 120,
            child: ClipRRect(
              borderRadius: const BorderRadius.only(
                topLeft: Radius.circular(12),
                bottomLeft: Radius.circular(12),
              ),
              child: Image.network(product.imageUrl, fit: BoxFit.cover),
            ),
          ),
          // Nội dung chiếm phần còn lại
          Expanded(
            child: Padding(
              padding: const EdgeInsets.all(12),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(product.name, style: context.textTheme.titleSmall),
                  const SizedBox(height: 4),
                  Text(product.formattedPrice, style: context.textTheme.bodyMedium),
                ],
              ),
            ),
          ),
        ],
      ),
    );
  }
}
```

### 3.5.3. SafeArea — Tránh Notch và System Bars

`SafeArea` tự động thêm padding để tránh các phần của màn hình bị che khuất bởi notch, status bar, home indicator (iOS), và navigation bar (Android gesture navigation).

```dart
// ✅ CHUẨN — SafeArea cho nội dung toàn màn hình
Scaffold(
  // Scaffold tự xử lý AppBar và BottomNavigationBar
  // Chỉ cần SafeArea khi dùng Stack toàn màn hình hoặc không dùng Scaffold
  body: SafeArea(
    // bottom: false khi Scaffold đã có BottomNavigationBar
    // (BottomNavigationBar tự handle safe area)
    bottom: false,
    child: YourContent(),
  ),
)

// ✅ CHUẨN — Lấy padding của SafeArea để tính toán
Widget build(BuildContext context) {
  final topPadding = MediaQuery.paddingOf(context).top;
  // topPadding = chiều cao status bar + notch
  return SizedBox(height: topPadding + 56); // AppBar height = 56
}
```

---

## 3.6. Custom Font

### 3.6.1. Đăng Ký Font Trong pubspec.yaml

```yaml
# pubspec.yaml
flutter:
  fonts:
    - family: Inter
      fonts:
        - asset: assets/fonts/Inter-Regular.ttf
          weight: 400
        - asset: assets/fonts/Inter-Medium.ttf
          weight: 500
        - asset: assets/fonts/Inter-SemiBold.ttf
          weight: 600
        - asset: assets/fonts/Inter-Bold.ttf
          weight: 700
```

```dart
// Sử dụng font trong ThemeData
ThemeData(
  fontFamily: 'Inter', // Áp dụng cho toàn app
  textTheme: GoogleFonts.interTextTheme(), // Hoặc dùng google_fonts package
)

// Override font cho widget cụ thể
Text(
  'Heading',
  style: TextStyle(
    fontFamily: 'Playfair Display', // Font khác cho display text
    fontWeight: FontWeight.w700,
    fontSize: 32,
  ),
)
```

---

## 3.7. Bài Tập: Theme Switcher Hoàn Chỉnh

Bài tập này yêu cầu xây dựng một màn hình Settings với khả năng:
1. Switch giữa light/dark/system theme
2. Chọn seed color từ danh sách preset
3. Preview theme thay đổi ngay lập tức

```dart
// Gợi ý: Quản lý theme state với ValueNotifier đơn giản
// (Sẽ refactor sang Riverpod ở chương 5)

class ThemeController extends ChangeNotifier {
  ThemeMode _mode = ThemeMode.system;
  Color _seedColor = const Color(0xFF6750A4);

  ThemeMode get mode => _mode;
  Color get seedColor => _seedColor;

  ThemeData get lightTheme => ThemeData(
    useMaterial3: true,
    colorScheme: ColorScheme.fromSeed(
      seedColor: _seedColor,
      brightness: Brightness.light,
    ),
  );

  ThemeData get darkTheme => ThemeData(
    useMaterial3: true,
    colorScheme: ColorScheme.fromSeed(
      seedColor: _seedColor,
      brightness: Brightness.dark,
    ),
  );

  void setMode(ThemeMode mode) {
    _mode = mode;
    notifyListeners();
  }

  void setSeedColor(Color color) {
    _seedColor = color;
    notifyListeners();
  }
}

// Danh sách preset color để người dùng chọn
const List<({String name, Color color})> kPresetColors = [
  (name: 'Tím', color: Color(0xFF6750A4)),
  (name: 'Xanh dương', color: Color(0xFF1565C0)),
  (name: 'Xanh lá', color: Color(0xFF2E7D32)),
  (name: 'Cam', color: Color(0xFFE65100)),
  (name: 'Hồng', color: Color(0xFFC2185B)),
  (name: 'Xanh ngọc', color: Color(0xFF00695C)),
];
```

---

## Tóm Tắt Chương 3

| Khái niệm | Điểm Cốt Lõi |
|---|---|
| Không có global stylesheet | Style gắn widget hoặc ThemeData — không có CSS cascade |
| ThemeData | Cấu hình theme trung tâm; dùng `useMaterial3: true` |
| ColorScheme.fromSeed | 1 seed color → Flutter tự sinh 29 màu chuẩn M3 |
| TextTheme | 15 text style có tên semantics; không dùng fontSize tuỳ tiện |
| Dark mode | Đặt `darkTheme` + `ThemeMode.system`; widget lấy màu từ colorScheme |
| BoxDecoration | Container styling: color, border, radius, shadow, gradient |
| MediaQuery.sizeOf | Responsive theo màn hình; LayoutBuilder responsive theo parent |
| SafeArea | Tránh notch, status bar, home indicator |

> **Lời khuyên thực tiễn:** Đầu tư thời gian định nghĩa `ThemeData` hoàn chỉnh ngay từ đầu dự án. Thay đổi theme sau khi đã có nhiều màn hình rất tốn công. Một theme tốt giảm 30–40% lượng code styling trùng lặp trên toàn app.
