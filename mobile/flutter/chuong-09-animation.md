# Chương 9: Animation Trong Flutter

---

## 9.1. Triết Lý Animation Trong Flutter

Flutter phân chia animation thành hai nhóm rõ ràng dựa trên mức độ kiểm soát:

**Implicit animations** — Flutter tự lo toàn bộ: developer chỉ thay đổi giá trị, framework tự tween từ giá trị cũ sang mới. Dễ dùng, ít code, đủ cho 80% trường hợp.

**Explicit animations** — Developer kiểm soát hoàn toàn: tạo `AnimationController`, định nghĩa `Tween`, quản lý vòng đời. Phức tạp hơn nhưng linh hoạt tuyệt đối.

**Nguyên tắc:** Bắt đầu với implicit. Chuyển sang explicit khi cần điều khiển: pause, reverse, combine nhiều animation, sync với gesture.

---

## 9.2. Implicit Animations

### 9.2.1. AnimatedContainer — Container Tự Tween

`AnimatedContainer` giống hệt `Container` nhưng tự animate khi bất kỳ property nào thay đổi.

```dart
// ✅ CHUẨN — AnimatedContainer cho trạng thái thay đổi
class AnimatedProductCard extends StatefulWidget {
  const AnimatedProductCard({super.key, required this.product});
  final Product product;

  @override
  State<AnimatedProductCard> createState() => _AnimatedProductCardState();
}

class _AnimatedProductCardState extends State<AnimatedProductCard> {
  bool _isExpanded = false;
  bool _isInCart = false;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () => setState(() => _isExpanded = !_isExpanded),
      child: AnimatedContainer(
        // Duration: thời gian animation
        duration: const Duration(milliseconds: 300),
        // Curve: kiểu easing — easeInOut là tự nhiên nhất
        curve: Curves.easeInOut,
        // Tất cả property dưới đây sẽ animate khi thay đổi
        width: double.infinity,
        height: _isExpanded ? 280 : 160,
        decoration: BoxDecoration(
          color: _isInCart
              ? context.colorScheme.primaryContainer
              : context.colorScheme.surface,
          borderRadius: BorderRadius.circular(_isExpanded ? 20 : 12),
          boxShadow: [
            BoxShadow(
              color: Colors.black.withOpacity(_isExpanded ? 0.15 : 0.05),
              blurRadius: _isExpanded ? 20 : 8,
              offset: const Offset(0, 4),
            ),
          ],
        ),
        padding: EdgeInsets.all(_isExpanded ? 20 : 12),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(widget.product.name,
                style: context.textTheme.titleMedium),
            if (_isExpanded) ...[
              const SizedBox(height: 8),
              Text(widget.product.description,
                  style: context.textTheme.bodySmall),
            ],
            const Spacer(),
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                Text(widget.product.formattedPrice,
                    style: context.textTheme.titleSmall),
                IconButton.filled(
                  onPressed: () => setState(() => _isInCart = !_isInCart),
                  icon: AnimatedSwitcher(
                    duration: const Duration(milliseconds: 200),
                    child: Icon(
                      _isInCart ? Icons.check : Icons.add,
                      key: ValueKey(_isInCart),
                    ),
                  ),
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

### 9.2.2. AnimatedOpacity và AnimatedSwitcher

```dart
// AnimatedOpacity — fade in/out
class FadingWidget extends StatefulWidget {
  const FadingWidget({super.key, required this.child});
  final Widget child;

  @override
  State<FadingWidget> createState() => _FadingWidgetState();
}

class _FadingWidgetState extends State<FadingWidget> {
  bool _visible = false;

  @override
  void initState() {
    super.initState();
    // Trigger fade in sau khi build
    WidgetsBinding.instance.addPostFrameCallback((_) {
      if (mounted) setState(() => _visible = true);
    });
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedOpacity(
      opacity: _visible ? 1.0 : 0.0,
      duration: const Duration(milliseconds: 500),
      curve: Curves.easeIn,
      child: widget.child,
    );
  }
}

// ✅ CHUẨN — AnimatedSwitcher cho chuyển đổi content
// Tự động animate khi child widget thay đổi
class LoadingButton extends StatelessWidget {
  const LoadingButton({
    super.key,
    required this.isLoading,
    required this.onPressed,
    required this.label,
  });

  final bool isLoading;
  final VoidCallback? onPressed;
  final String label;

  @override
  Widget build(BuildContext context) {
    return FilledButton(
      onPressed: isLoading ? null : onPressed,
      child: AnimatedSwitcher(
        duration: const Duration(milliseconds: 200),
        // transitionBuilder tùy chỉnh animation
        transitionBuilder: (child, animation) => FadeTransition(
          opacity: animation,
          child: ScaleTransition(scale: animation, child: child),
        ),
        child: isLoading
            ? const SizedBox(
                key: ValueKey('loading'), // KEY quan trọng để AnimatedSwitcher nhận ra thay đổi
                width: 20,
                height: 20,
                child: CircularProgressIndicator(strokeWidth: 2),
              )
            : Text(key: const ValueKey('label'), label),
      ),
    );
  }
}
```

### 9.2.3. AnimatedList — List Với Add/Remove Animation

```dart
// ✅ CHUẨN — AnimatedList cho danh sách thay đổi động
class AnimatedCartList extends StatefulWidget {
  const AnimatedCartList({super.key});

  @override
  State<AnimatedCartList> createState() => _AnimatedCartListState();
}

class _AnimatedCartListState extends State<AnimatedCartList> {
  final _listKey = GlobalKey<AnimatedListState>();
  final _items = <CartItem>[];

  void addItem(CartItem item) {
    _items.insert(0, item);
    _listKey.currentState?.insertItem(
      0,
      duration: const Duration(milliseconds: 300),
    );
  }

  void removeItem(int index) {
    final removedItem = _items.removeAt(index);
    _listKey.currentState?.removeItem(
      index,
      (context, animation) => SizeTransition(
        sizeFactor: animation,
        child: FadeTransition(
          opacity: animation,
          child: CartItemTile(item: removedItem),
        ),
      ),
      duration: const Duration(milliseconds: 250),
    );
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedList(
      key: _listKey,
      initialItemCount: _items.length,
      itemBuilder: (context, index, animation) {
        return SizeTransition(
          sizeFactor: CurvedAnimation(
            parent: animation,
            curve: Curves.easeOut,
          ),
          child: CartItemTile(item: _items[index]),
        );
      },
    );
  }
}
```

### 9.2.4. TweenAnimationBuilder — Custom Implicit

```dart
// TweenAnimationBuilder: implicit animation cho bất kỳ giá trị nào
// Không cần AnimationController, không cần mixin

class RatingBar extends StatelessWidget {
  const RatingBar({super.key, required this.rating, required this.maxRating});
  final double rating;
  final double maxRating;

  @override
  Widget build(BuildContext context) {
    return TweenAnimationBuilder<double>(
      tween: Tween(begin: 0, end: rating / maxRating),
      duration: const Duration(milliseconds: 800),
      curve: Curves.easeOutCubic,
      builder: (context, value, child) {
        return Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              children: [
                Expanded(
                  child: LinearProgressIndicator(
                    value: value,
                    backgroundColor: context.colorScheme.surfaceContainerHighest,
                    borderRadius: BorderRadius.circular(4),
                  ),
                ),
                const SizedBox(width: 8),
                Text(
                  '${(value * maxRating).toStringAsFixed(1)}',
                  style: context.textTheme.labelMedium,
                ),
              ],
            ),
          ],
        );
      },
    );
  }
}

// TweenAnimationBuilder với Color
class ThemeTransitionWidget extends StatelessWidget {
  const ThemeTransitionWidget({
    super.key,
    required this.isDark,
    required this.child,
  });

  final bool isDark;
  final Widget child;

  @override
  Widget build(BuildContext context) {
    return TweenAnimationBuilder<Color?>(
      tween: ColorTween(
        begin: isDark ? Colors.white : Colors.black,
        end: isDark ? Colors.black : Colors.white,
      ),
      duration: const Duration(milliseconds: 400),
      builder: (context, color, _) {
        return ColoredBox(color: color ?? Colors.white, child: child);
      },
    );
  }
}
```

---

## 9.3. Explicit Animations

### 9.3.1. AnimationController và Tween

```dart
// ✅ CHUẨN — Explicit animation hoàn chỉnh
class PulsingFavoriteButton extends StatefulWidget {
  const PulsingFavoriteButton({
    super.key,
    required this.isFavorite,
    required this.onToggle,
  });

  final bool isFavorite;
  final VoidCallback onToggle;

  @override
  State<PulsingFavoriteButton> createState() => _PulsingFavoriteButtonState();
}

class _PulsingFavoriteButtonState extends State<PulsingFavoriteButton>
    with SingleTickerProviderStateMixin {
  // SingleTickerProviderStateMixin: cần thiết cho AnimationController
  // TickerProviderStateMixin: nếu cần nhiều controller
  late final AnimationController _controller;
  late final Animation<double> _scaleAnimation;
  late final Animation<Color?> _colorAnimation;

  @override
  void initState() {
    super.initState();

    _controller = AnimationController(
      vsync: this, // vsync: dùng mixin làm Ticker provider
      duration: const Duration(milliseconds: 300),
    );

    // Tween: nội suy từ begin đến end
    _scaleAnimation = TweenSequence<double>([
      TweenSequenceItem(
        tween: Tween(begin: 1.0, end: 1.4)
            .chain(CurveTween(curve: Curves.easeOut)),
        weight: 50,
      ),
      TweenSequenceItem(
        tween: Tween(begin: 1.4, end: 1.0)
            .chain(CurveTween(curve: Curves.elasticOut)),
        weight: 50,
      ),
    ]).animate(_controller);

    _colorAnimation = ColorTween(
      begin: Colors.grey,
      end: Colors.red,
    ).animate(CurvedAnimation(
      parent: _controller,
      curve: Curves.easeInOut,
    ));

    // Đặt giá trị ban đầu dựa theo isFavorite
    if (widget.isFavorite) _controller.value = 1.0;
  }

  @override
  void didUpdateWidget(PulsingFavoriteButton oldWidget) {
    super.didUpdateWidget(oldWidget);
    // Chạy animation khi prop thay đổi
    if (widget.isFavorite != oldWidget.isFavorite) {
      if (widget.isFavorite) {
        _controller.forward(from: 0);
      } else {
        _controller.reverse();
      }
    }
  }

  @override
  void dispose() {
    _controller.dispose(); // PHẢI dispose AnimationController
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: widget.onToggle,
      child: AnimatedBuilder(
        animation: _controller,
        builder: (context, child) {
          return Transform.scale(
            scale: _scaleAnimation.value,
            child: Icon(
              widget.isFavorite ? Icons.favorite : Icons.favorite_border,
              color: _colorAnimation.value,
              size: 28,
            ),
          );
        },
      ),
    );
  }
}
```

### 9.3.2. Staggered Animations — Chuỗi Animation

```dart
// ✅ CHUẨN — Staggered animation: các element xuất hiện lần lượt
class ProductDetailAnimation extends StatefulWidget {
  const ProductDetailAnimation({super.key, required this.product});
  final Product product;

  @override
  State<ProductDetailAnimation> createState() => _ProductDetailAnimationState();
}

class _ProductDetailAnimationState extends State<ProductDetailAnimation>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;

  // Mỗi element có animation riêng, offset theo thời gian
  late final Animation<Offset> _imageSlide;
  late final Animation<double> _titleFade;
  late final Animation<double> _priceFade;
  late final Animation<double> _buttonFade;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 900),
    );

    // Image: từ trên xuống, 0-40% thời gian
    _imageSlide = Tween<Offset>(
      begin: const Offset(0, -0.1),
      end: Offset.zero,
    ).animate(CurvedAnimation(
      parent: _controller,
      curve: const Interval(0.0, 0.4, curve: Curves.easeOut),
    ));

    // Title: fade in, 20-60% thời gian
    _titleFade = Tween<double>(begin: 0, end: 1).animate(
      CurvedAnimation(
        parent: _controller,
        curve: const Interval(0.2, 0.6, curve: Curves.easeIn),
      ),
    );

    // Price: fade in, 40-80%
    _priceFade = Tween<double>(begin: 0, end: 1).animate(
      CurvedAnimation(
        parent: _controller,
        curve: const Interval(0.4, 0.8, curve: Curves.easeIn),
      ),
    );

    // Button: fade in, 60-100%
    _buttonFade = Tween<double>(begin: 0, end: 1).animate(
      CurvedAnimation(
        parent: _controller,
        curve: const Interval(0.6, 1.0, curve: Curves.easeIn),
      ),
    );

    // Bắt đầu animation ngay
    _controller.forward();
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _controller,
      builder: (context, child) {
        return Column(
          children: [
            // Image slide in
            SlideTransition(
              position: _imageSlide,
              child: AppNetworkImage(
                url: widget.product.imageUrl,
                height: 300,
              ),
            ),

            // Title fade in
            FadeTransition(
              opacity: _titleFade,
              child: Text(
                widget.product.name,
                style: context.textTheme.headlineSmall,
              ),
            ),

            // Price fade in
            FadeTransition(
              opacity: _priceFade,
              child: Text(
                widget.product.formattedPrice,
                style: context.textTheme.titleLarge?.copyWith(
                  color: context.colorScheme.primary,
                ),
              ),
            ),

            // Button fade in
            FadeTransition(
              opacity: _buttonFade,
              child: FilledButton(
                onPressed: () {},
                child: const Text('Thêm vào giỏ'),
              ),
            ),
          ],
        );
      },
    );
  }
}
```

---

## 9.4. Hero Animation — Shared Element Transition

Hero animation tạo hiệu ứng một element "bay" từ màn hình này sang màn hình khác. Rất phổ biến trong app thương mại điện tử khi tap vào ảnh sản phẩm.

```dart
// ✅ CHUẨN — Hero animation đúng cách

// Màn hình danh sách
class ProductListItem extends StatelessWidget {
  const ProductListItem({super.key, required this.product});
  final Product product;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () => context.push('/products/${product.id}'),
      child: Row(
        children: [
          // Hero tag PHẢI unique — dùng product.id
          Hero(
            tag: 'product-image-${product.id}',
            child: ClipRRect(
              borderRadius: BorderRadius.circular(8),
              child: Image.network(
                product.imageUrl,
                width: 80,
                height: 80,
                fit: BoxFit.cover,
              ),
            ),
          ),
          const SizedBox(width: 12),
          Text(product.name),
        ],
      ),
    );
  }
}

// Màn hình detail — Hero tag PHẢI khớp với tag ở màn hình list
class ProductDetailScreen extends StatelessWidget {
  const ProductDetailScreen({super.key, required this.productId});
  final String productId;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: CustomScrollView(
        slivers: [
          SliverAppBar(
            expandedHeight: 350,
            flexibleSpace: FlexibleSpaceBar(
              background: Hero(
                tag: 'product-image-$productId', // Khớp với tag ở list
                // flightShuttleBuilder: tùy chỉnh widget trong lúc bay
                flightShuttleBuilder: (flightContext, animation,
                    flightDirection, fromContext, toContext) {
                  return AnimatedBuilder(
                    animation: animation,
                    builder: (_, child) => ClipRRect(
                      // Bo tròn nhỏ dần khi mở rộng
                      borderRadius: BorderRadius.circular(
                        Tween<double>(begin: 8, end: 0)
                            .evaluate(animation),
                      ),
                      child: child,
                    ),
                    child: Image.network(
                      'product-image-url',
                      fit: BoxFit.cover,
                    ),
                  );
                },
                child: Image.network(
                  'product-image-url',
                  fit: BoxFit.cover,
                ),
              ),
            ),
          ),
          // ... content
        ],
      ),
    );
  }
}
```

---

## 9.5. Page Transition Animations

```dart
// ✅ CHUẨN — Custom page transition trong GoRouter

// Slide từ phải sang (mặc định iOS style)
class SlideRightRoute extends CustomTransitionPage {
  SlideRightRoute({required super.child, super.key})
      : super(
          transitionDuration: const Duration(milliseconds: 300),
          reverseTransitionDuration: const Duration(milliseconds: 250),
          transitionsBuilder: (context, animation, secondaryAnimation, child) {
            // Màn hình mới: slide từ phải vào
            final slideIn = Tween<Offset>(
              begin: const Offset(1, 0),
              end: Offset.zero,
            ).animate(CurvedAnimation(
              parent: animation,
              curve: Curves.easeOutCubic,
            ));

            // Màn hình cũ: slide sang trái một chút (parallax)
            final slideOut = Tween<Offset>(
              begin: Offset.zero,
              end: const Offset(-0.3, 0),
            ).animate(CurvedAnimation(
              parent: secondaryAnimation,
              curve: Curves.easeOutCubic,
            ));

            return SlideTransition(
              position: slideIn,
              child: SlideTransition(
                position: slideOut,
                child: child,
              ),
            );
          },
        );
}

// Fade + Scale (giống dialog mở)
class FadeScaleRoute extends CustomTransitionPage {
  FadeScaleRoute({required super.child, super.key})
      : super(
          transitionDuration: const Duration(milliseconds: 250),
          transitionsBuilder: (context, animation, _, child) {
            return FadeTransition(
              opacity: CurvedAnimation(
                parent: animation,
                curve: Curves.easeOut,
              ),
              child: ScaleTransition(
                scale: Tween<double>(begin: 0.92, end: 1.0).animate(
                  CurvedAnimation(
                    parent: animation,
                    curve: Curves.easeOutCubic,
                  ),
                ),
                child: child,
              ),
            );
          },
        );
}

// Đăng ký trong GoRouter
GoRoute(
  path: '/products/:id',
  pageBuilder: (context, state) => SlideRightRoute(
    key: state.pageKey,
    child: ProductDetailScreen(productId: state.pathParameters['id']!),
  ),
),

GoRoute(
  path: '/search',
  pageBuilder: (context, state) => FadeScaleRoute(
    key: state.pageKey,
    child: const SearchScreen(),
  ),
),
```

---

## 9.6. Lottie Animation

Lottie cho phép chạy các animation phức tạp được tạo bởi designer trong After Effects, xuất ra file JSON nhẹ.

```yaml
# pubspec.yaml
dependencies:
  lottie: ^3.1.0
```

```dart
// ✅ CHUẨN — Lottie với controller để kiểm soát
class OrderSuccessAnimation extends StatefulWidget {
  const OrderSuccessAnimation({super.key, required this.onComplete});
  final VoidCallback onComplete;

  @override
  State<OrderSuccessAnimation> createState() => _OrderSuccessAnimationState();
}

class _OrderSuccessAnimationState extends State<OrderSuccessAnimation>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(vsync: this);
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            // Lottie từ assets
            Lottie.asset(
              'assets/animations/order_success.json',
              controller: _controller,
              width: 200,
              height: 200,
              // onLoaded: callback khi file đã load
              onLoaded: (composition) {
                _controller
                  ..duration = composition.duration
                  ..forward().whenComplete(widget.onComplete);
              },
              // fit: cách scale animation trong box
              fit: BoxFit.contain,
            ),
            const SizedBox(height: 24),
            Text(
              'Đặt hàng thành công!',
              style: context.textTheme.headlineSmall,
            ),
          ],
        ),
      ),
    );
  }
}

// Lottie từ network (CDN)
Lottie.network(
  'https://assets.lottiefiles.com/packages/lf20_success.json',
  repeat: false,
)

// Lottie loading skeleton — thay cho CircularProgressIndicator
class LottieLoading extends StatelessWidget {
  const LottieLoading({super.key});

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Lottie.asset(
        'assets/animations/loading.json',
        width: 120,
        height: 120,
        repeat: true, // Loop khi loading
      ),
    );
  }
}
```

**Nguồn tìm file Lottie miễn phí:**
- [lottiefiles.com](https://lottiefiles.com) — Thư viện lớn nhất
- [lordicon.com](https://lordicon.com) — Icon animation đẹp

---

## 9.7. Bài Tập: Add-to-Cart Animation

Hiệu ứng khi tap "Thêm vào giỏ": sản phẩm bay đến icon giỏ hàng trên navigation bar.

```dart
// Kỹ thuật: GlobalKey + Overlay + AnimationController
class AddToCartEffect {
  // Chạy animation từ widget nguồn đến target
  static Future<void> run({
    required BuildContext context,
    required GlobalKey sourceKey,   // Key của nút "Thêm"
    required GlobalKey targetKey,   // Key của icon giỏ hàng
    required Widget thumbnail,      // Widget hiển thị khi bay
  }) async {
    // Lấy vị trí của source widget
    final sourceBox = sourceKey.currentContext?.findRenderObject() as RenderBox?;
    final targetBox = targetKey.currentContext?.findRenderObject() as RenderBox?;
    if (sourceBox == null || targetBox == null) return;

    final sourcePosition = sourceBox.localToGlobal(Offset.zero);
    final targetPosition = targetBox.localToGlobal(Offset.zero);
    final sourceSize = sourceBox.size;
    final targetSize = targetBox.size;

    // Thêm overlay entry
    final overlay = Overlay.of(context);
    late final OverlayEntry entry;

    entry = OverlayEntry(
      builder: (context) => _CartFlyAnimation(
        startPosition: sourcePosition + Offset(sourceSize.width / 2, sourceSize.height / 2),
        endPosition: targetPosition + Offset(targetSize.width / 2, targetSize.height / 2),
        thumbnail: thumbnail,
        onComplete: () => entry.remove(),
      ),
    );

    overlay.insert(entry);
  }
}

class _CartFlyAnimation extends StatefulWidget {
  const _CartFlyAnimation({
    required this.startPosition,
    required this.endPosition,
    required this.thumbnail,
    required this.onComplete,
  });

  final Offset startPosition;
  final Offset endPosition;
  final Widget thumbnail;
  final VoidCallback onComplete;

  @override
  State<_CartFlyAnimation> createState() => __CartFlyAnimationState();
}

class __CartFlyAnimationState extends State<_CartFlyAnimation>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;
  late final Animation<Offset> _positionAnimation;
  late final Animation<double> _scaleAnimation;
  late final Animation<double> _opacityAnimation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 600),
    );

    _positionAnimation = Tween<Offset>(
      begin: widget.startPosition,
      end: widget.endPosition,
    ).animate(CurvedAnimation(
      parent: _controller,
      curve: Curves.easeInCubic,
    ));

    _scaleAnimation = Tween<double>(begin: 1.0, end: 0.2).animate(
      CurvedAnimation(parent: _controller, curve: Curves.easeIn),
    );

    _opacityAnimation = Tween<double>(begin: 1.0, end: 0.0).animate(
      CurvedAnimation(
        parent: _controller,
        curve: const Interval(0.7, 1.0),
      ),
    );

    _controller.forward().whenComplete(widget.onComplete);
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _controller,
      builder: (context, child) {
        return Positioned(
          left: _positionAnimation.value.dx - 25,
          top: _positionAnimation.value.dy - 25,
          child: Opacity(
            opacity: _opacityAnimation.value,
            child: Transform.scale(
              scale: _scaleAnimation.value,
              child: SizedBox(
                width: 50,
                height: 50,
                child: ClipRRect(
                  borderRadius: BorderRadius.circular(8),
                  child: widget.thumbnail,
                ),
              ),
            ),
          ),
        );
      },
    );
  }
}
```

---

## Tóm Tắt Chương 9

| Khái niệm | Điểm Cốt Lõi |
|---|---|
| Implicit animation | Chỉ đổi giá trị, Flutter tự animate. 80% use case. |
| AnimatedContainer | Animate mọi property của Container tự động |
| AnimatedSwitcher | Animate khi child widget thay đổi — cần `key` |
| TweenAnimationBuilder | Implicit cho giá trị bất kỳ — không cần mixin |
| AnimationController | Trung tâm explicit animation; `vsync` để tối ưu |
| Tween + Interval | Staggered animation — offset thời gian từng element |
| Hero | Shared element transition — tag phải unique và khớp |
| Lottie | Import animation phức tạp từ designer, file JSON nhẹ |
| dispose() | LUÔN dispose AnimationController — không thì memory leak |
| Curves | `easeOutCubic` cho UI tự nhiên; `elasticOut` cho vui vẻ |

> **Nguyên tắc vàng:** Animation nên phục vụ UX, không phải để showcase kỹ thuật. Animation tốt: người dùng không để ý nhưng cảm thấy app mượt. Animation xấu: người dùng chú ý vào animation thay vì content. Guideline: duration 200-350ms cho micro-interaction, 400-600ms cho screen transition, không quá 1 giây cho bất kỳ animation nào.