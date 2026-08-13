# Chương 10: Local Storage & Cache

---

## 10.1. Tổng Quan Các Lớp Storage

Flutter không có storage built-in duy nhất — mỗi nhu cầu dùng thư viện phù hợp. Hiểu đúng từng lớp giúp tránh dùng sai công cụ:

```
┌─────────────────────────────────────────────────────┐
│              Phân tầng Storage                       │
├─────────────────┬───────────────────────────────────┤
│ Loại            │ Thư viện                           │
├─────────────────┼───────────────────────────────────┤
│ Key-Value nhẹ   │ shared_preferences                 │
│ (settings, flag)│                                    │
├─────────────────┼───────────────────────────────────┤
│ Dữ liệu nhạy cảm│ flutter_secure_storage             │
│ (token, password│ (encrypted, Keychain/Keystore)     │
├─────────────────┼───────────────────────────────────┤
│ Cache ảnh       │ cached_network_image               │
│                 │                                    │
├─────────────────┼───────────────────────────────────┤
│ Database nhẹ    │ Hive / Isar                        │
│ (offline data,  │                                    │
│  complex struct)│                                    │
├─────────────────┼───────────────────────────────────┤
│ Relational DB   │ sqflite / drift                    │
│ (complex query) │                                    │
└─────────────────┴───────────────────────────────────┘
```

**Nguyên tắc chọn:** Bắt đầu với layer đơn giản nhất đáp ứng nhu cầu. Chỉ upgrade khi cần.

---

## 10.2. shared_preferences — Key-Value Storage

### 10.2.1. Khi Nào Dùng

`shared_preferences` lưu key-value pairs đơn giản, persist qua lần khởi động app. Phù hợp cho:
- User preferences: theme mode, language, notification settings
- Onboarding state: đã xem tutorial chưa
- App config: feature flags, last viewed page
- Cache nhẹ: timestamp lần cuối refresh, user display name

**Không phù hợp cho:** dữ liệu nhạy cảm (token, password), cấu trúc phức tạp, danh sách dài.

```yaml
# pubspec.yaml
dependencies:
  shared_preferences: ^2.2.0
```

### 10.2.2. Abstraction Layer — Không Dùng Trực Tiếp

Trong dự án thực tế, không bao giờ inject `SharedPreferences` trực tiếp vào business logic. Tạo abstraction để dễ test và thay thế.

```dart
// lib/core/storage/preferences_storage.dart

// Abstract interface
abstract interface class PreferencesStorage {
  Future<ThemeMode> getThemeMode();
  Future<void> saveThemeMode(ThemeMode mode);

  Future<String?> getLanguageCode();
  Future<void> saveLanguageCode(String code);

  Future<bool> hasSeenOnboarding();
  Future<void> markOnboardingSeen();

  Future<DateTime?> getLastSyncTime();
  Future<void> saveLastSyncTime(DateTime time);

  Future<void> clear();
}

// Implementation
class SharedPreferencesStorage implements PreferencesStorage {
  SharedPreferencesStorage(this._prefs);
  final SharedPreferences _prefs;

  // Keys — luôn là private constant
  static const _keyThemeMode = 'theme_mode';
  static const _keyLanguage = 'language_code';
  static const _keyOnboardingSeen = 'onboarding_seen';
  static const _keyLastSync = 'last_sync_time';

  @override
  Future<ThemeMode> getThemeMode() async {
    final value = _prefs.getString(_keyThemeMode);
    return switch (value) {
      'light' => ThemeMode.light,
      'dark' => ThemeMode.dark,
      _ => ThemeMode.system,
    };
  }

  @override
  Future<void> saveThemeMode(ThemeMode mode) async {
    final value = switch (mode) {
      ThemeMode.light => 'light',
      ThemeMode.dark => 'dark',
      ThemeMode.system => 'system',
    };
    await _prefs.setString(_keyThemeMode, value);
  }

  @override
  Future<String?> getLanguageCode() async {
    return _prefs.getString(_keyLanguage);
  }

  @override
  Future<void> saveLanguageCode(String code) async {
    await _prefs.setString(_keyLanguage, code);
  }

  @override
  Future<bool> hasSeenOnboarding() async {
    return _prefs.getBool(_keyOnboardingSeen) ?? false;
  }

  @override
  Future<void> markOnboardingSeen() async {
    await _prefs.setBool(_keyOnboardingSeen, true);
  }

  @override
  Future<DateTime?> getLastSyncTime() async {
    final ms = _prefs.getInt(_keyLastSync);
    return ms != null ? DateTime.fromMillisecondsSinceEpoch(ms) : null;
  }

  @override
  Future<void> saveLastSyncTime(DateTime time) async {
    await _prefs.setInt(_keyLastSync, time.millisecondsSinceEpoch);
  }

  @override
  Future<void> clear() async {
    await _prefs.clear();
  }
}

// Provider
@riverpod
Future<SharedPreferences> sharedPreferences(SharedPreferencesRef ref) async {
  return SharedPreferences.getInstance();
}

@riverpod
PreferencesStorage preferencesStorage(PreferencesStorageRef ref) {
  final prefs = ref.watch(sharedPreferencesProvider).requireValue;
  return SharedPreferencesStorage(prefs);
}
```

### 10.2.3. Theme Mode Provider Với Persistence

```dart
// lib/core/providers/theme_provider.dart

@riverpod
class AppThemeMode extends _$AppThemeMode {
  @override
  Future<ThemeMode> build() async {
    // Load từ storage khi provider khởi tạo
    return ref.watch(preferencesStorageProvider).getThemeMode();
  }

  Future<void> setMode(ThemeMode mode) async {
    state = AsyncData(mode);
    await ref.read(preferencesStorageProvider).saveThemeMode(mode);
  }
}

// Settings screen sử dụng
class ThemeSettingsSection extends ConsumerWidget {
  const ThemeSettingsSection({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final themeModeAsync = ref.watch(appThemeModeProvider);

    return themeModeAsync.when(
      loading: () => const ListTile(title: Text('Giao diện')),
      error: (_, __) => const SizedBox.shrink(),
      data: (currentMode) => Column(
        children: [
          const ListTile(
            leading: Icon(Icons.palette_outlined),
            title: Text('Giao diện'),
          ),
          ...ThemeMode.values.map((mode) {
            return RadioListTile<ThemeMode>(
              title: Text(_modeLabel(mode)),
              secondary: Icon(_modeIcon(mode)),
              value: mode,
              groupValue: currentMode,
              onChanged: (selected) {
                if (selected != null) {
                  ref.read(appThemeModeProvider.notifier).setMode(selected);
                }
              },
            );
          }),
        ],
      ),
    );
  }

  String _modeLabel(ThemeMode mode) => switch (mode) {
        ThemeMode.system => 'Theo hệ thống',
        ThemeMode.light => 'Sáng',
        ThemeMode.dark => 'Tối',
      };

  IconData _modeIcon(ThemeMode mode) => switch (mode) {
        ThemeMode.system => Icons.brightness_auto,
        ThemeMode.light => Icons.light_mode,
        ThemeMode.dark => Icons.dark_mode,
      };
}
```

---

## 10.3. flutter_secure_storage — Lưu Dữ Liệu Nhạy Cảm

### 10.3.1. Tại Sao Cần Secure Storage?

`shared_preferences` lưu dữ liệu dưới dạng plain text trong file hệ thống — bất kỳ app nào có root access đều đọc được. Token, password, private key **phải** được lưu trong encrypted storage:

- **iOS:** Keychain — encrypted bởi Secure Enclave
- **Android:** EncryptedSharedPreferences + Android Keystore

```yaml
# pubspec.yaml
dependencies:
  flutter_secure_storage: ^9.0.0
```

```dart
// lib/core/storage/token_storage.dart

abstract interface class TokenStorage {
  Future<String?> getAccessToken();
  Future<String?> getRefreshToken();
  Future<void> saveTokens({
    required String accessToken,
    required String refreshToken,
  });
  Future<void> clearTokens();
}

class SecureTokenStorage implements TokenStorage {
  SecureTokenStorage(this._storage);
  final FlutterSecureStorage _storage;

  static const _accessTokenKey = 'access_token';
  static const _refreshTokenKey = 'refresh_token';

  // Android options: chọn encryption level
  static const _androidOptions = AndroidOptions(
    encryptedSharedPreferences: true, // AES256 encryption
  );

  // iOS options: accessibility scope
  static const _iosOptions = IOSOptions(
    accessibility: KeychainAccessibility.first_unlock_this_device,
    // first_unlock_this_device: accessible sau khi user mở khóa lần đầu
    // always: accessible kể cả khi thiết bị bị khóa (ít secure hơn)
  );

  @override
  Future<String?> getAccessToken() => _storage.read(
        key: _accessTokenKey,
        aOptions: _androidOptions,
        iOptions: _iosOptions,
      );

  @override
  Future<String?> getRefreshToken() => _storage.read(
        key: _refreshTokenKey,
        aOptions: _androidOptions,
        iOptions: _iosOptions,
      );

  @override
  Future<void> saveTokens({
    required String accessToken,
    required String refreshToken,
  }) async {
    await Future.wait([
      _storage.write(
        key: _accessTokenKey,
        value: accessToken,
        aOptions: _androidOptions,
        iOptions: _iosOptions,
      ),
      _storage.write(
        key: _refreshTokenKey,
        value: refreshToken,
        aOptions: _androidOptions,
        iOptions: _iosOptions,
      ),
    ]);
  }

  @override
  Future<void> clearTokens() async {
    await Future.wait([
      _storage.delete(key: _accessTokenKey, aOptions: _androidOptions),
      _storage.delete(key: _refreshTokenKey, aOptions: _androidOptions),
    ]);
  }
}

// Provider
@riverpod
FlutterSecureStorage flutterSecureStorage(FlutterSecureStorageRef ref) {
  return const FlutterSecureStorage();
}

@riverpod
TokenStorage tokenStorage(TokenStorageRef ref) {
  return SecureTokenStorage(ref.watch(flutterSecureStorageRef));
}
```

---

## 10.4. cached_network_image — Cache Ảnh

### 10.4.1. Tại Sao Không Dùng Image.network Thuần?

`Image.network` tải ảnh mỗi lần build widget — không cache, tốn bandwidth, chậm. `cached_network_image` cache ảnh vào disk sau lần tải đầu, các lần sau load từ cache — nhanh hơn nhiều và hoạt động offline.

```yaml
# pubspec.yaml
dependencies:
  cached_network_image: ^3.3.0
```

### 10.4.2. Custom CachedImage Widget

```dart
// lib/core/widgets/app_cached_image.dart

class AppCachedImage extends StatelessWidget {
  const AppCachedImage({
    super.key,
    required this.url,
    this.width,
    this.height,
    this.fit = BoxFit.cover,
    this.borderRadius,
    this.placeholder,
    this.errorWidget,
    this.memCacheWidth,    // Resize trong memory để tiết kiệm RAM
    this.memCacheHeight,
  });

  final String url;
  final double? width;
  final double? height;
  final BoxFit fit;
  final BorderRadius? borderRadius;
  final Widget? placeholder;
  final Widget? errorWidget;
  final int? memCacheWidth;
  final int? memCacheHeight;

  @override
  Widget build(BuildContext context) {
    Widget image = CachedNetworkImage(
      imageUrl: url,
      width: width,
      height: height,
      fit: fit,

      // memCacheWidth/Height: resize ảnh trong memory
      // Rất quan trọng cho GridView — giảm OOM crash
      memCacheWidth: memCacheWidth,
      memCacheHeight: memCacheHeight,

      // placeholder: hiển thị khi đang tải lần đầu
      placeholder: (context, url) =>
          placeholder ??
          Container(
            width: width,
            height: height,
            color: context.colorScheme.surfaceContainerHighest,
            child: const Center(
              child: CircularProgressIndicator(strokeWidth: 2),
            ),
          ),

      // errorWidget: hiển thị khi tải lỗi
      errorWidget: (context, url, error) =>
          errorWidget ??
          Container(
            width: width,
            height: height,
            color: context.colorScheme.surfaceContainerHighest,
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                Icon(
                  Icons.broken_image_outlined,
                  color: context.colorScheme.outline,
                  size: 32,
                ),
                const SizedBox(height: 4),
                Text(
                  'Không tải được ảnh',
                  style: context.textTheme.labelSmall?.copyWith(
                    color: context.colorScheme.outline,
                  ),
                ),
              ],
            ),
          ),

      // fadeInDuration: animation khi ảnh load xong
      fadeInDuration: const Duration(milliseconds: 300),
      fadeOutDuration: const Duration(milliseconds: 100),
    );

    if (borderRadius != null) {
      image = ClipRRect(borderRadius: borderRadius!, child: image);
    }

    return image;
  }
}

// Sử dụng với memCache để tối ưu GridView
class ProductImageThumbnail extends StatelessWidget {
  const ProductImageThumbnail({super.key, required this.url});
  final String url;

  @override
  Widget build(BuildContext context) {
    // Thumbnail trong grid: 200x200px là đủ
    // Không cần load ảnh full 1200x1200 vào RAM
    return AppCachedImage(
      url: url,
      height: 160,
      borderRadius: const BorderRadius.vertical(top: Radius.circular(12)),
      memCacheWidth: 400,   // 2x để hỗ trợ high DPI
      memCacheHeight: 400,
    );
  }
}
```

### 10.4.3. Cache Management

```dart
// Xóa cache khi cần (ví dụ: user đăng xuất, update profile picture)
class CacheManager {
  static final _imageCache = CachedNetworkImage.evictFromCacheIfNecessary;

  // Xóa một ảnh khỏi cache
  static Future<void> evictImage(String url) async {
    await CachedNetworkImageProvider(url).evict();
  }

  // Xóa toàn bộ image cache
  static Future<void> clearImageCache() async {
    await DefaultCacheManager().emptyCache();
  }
}

// Trong logout flow
Future<void> logout() async {
  await tokenStorage.clearTokens();
  await preferencesStorage.clear();
  await CacheManager.clearImageCache(); // Xóa ảnh user đã cache
  ref.invalidateSelf();
}
```

---

## 10.5. Hive — Local Database

### 10.5.1. Khi Nào Cần Database Thay Vì Key-Value?

Khi cần lưu trữ cấu trúc phức tạp hơn, truy vấn theo điều kiện, hoặc lưu danh sách có nhiều item — `shared_preferences` không đủ dùng. `Hive` là lựa chọn nhẹ nhàng, không cần SQL, hiệu năng tốt.

```yaml
# pubspec.yaml
dependencies:
  hive_flutter: ^1.1.0
  hive: ^2.2.0

dev_dependencies:
  hive_generator: ^2.0.0
  build_runner: ^2.4.0
```

### 10.5.2. Định Nghĩa Model Hive

```dart
// lib/features/favorites/models/favorite_product.dart

import 'package:hive/hive.dart';

part 'favorite_product.g.dart'; // Generated

@HiveType(typeId: 0) // typeId: unique ID cho mỗi HiveObject
class FavoriteProduct extends HiveObject {
  FavoriteProduct({
    required this.id,
    required this.name,
    required this.imageUrl,
    required this.price,
    required this.addedAt,
  });

  @HiveField(0) // Field index — không đổi sau khi deploy
  final String id;

  @HiveField(1)
  final String name;

  @HiveField(2)
  final String imageUrl;

  @HiveField(3)
  final double price;

  @HiveField(4)
  final DateTime addedAt;
}

// Generate: dart run build_runner build
```

### 10.5.3. Hive Repository

```dart
// lib/core/storage/hive_initializer.dart

class HiveInitializer {
  static Future<void> init() async {
    await Hive.initFlutter();

    // Đăng ký adapters
    Hive.registerAdapter(FavoriteProductAdapter());
    Hive.registerAdapter(SearchHistoryAdapter());
    Hive.registerAdapter(RecentlyViewedAdapter());

    // Mở boxes
    await Future.wait([
      Hive.openBox<FavoriteProduct>('favorites'),
      Hive.openBox<String>('search_history'),
      Hive.openBox<RecentlyViewed>('recently_viewed'),
    ]);
  }
}

// main.dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await HiveInitializer.init();
  runApp(const ProviderScope(child: MyApp()));
}

// lib/features/favorites/repositories/favorites_repository.dart
class FavoritesRepository {
  FavoritesRepository() : _box = Hive.box<FavoriteProduct>('favorites');
  final Box<FavoriteProduct> _box;

  List<FavoriteProduct> getAll() {
    return _box.values.toList()
      ..sort((a, b) => b.addedAt.compareTo(a.addedAt));
  }

  bool isFavorite(String productId) => _box.containsKey(productId);

  Future<void> add(Product product) async {
    await _box.put(
      product.id, // Dùng productId làm key — dễ lookup O(1)
      FavoriteProduct(
        id: product.id,
        name: product.name,
        imageUrl: product.imageUrl,
        price: product.price,
        addedAt: DateTime.now(),
      ),
    );
  }

  Future<void> remove(String productId) async {
    await _box.delete(productId);
  }

  Future<void> toggle(Product product) async {
    if (isFavorite(product.id)) {
      await remove(product.id);
    } else {
      await add(product);
    }
  }

  // Stream để lắng nghe thay đổi realtime
  Stream<List<FavoriteProduct>> watchAll() {
    return _box.watch().map((_) => getAll());
  }

  Future<void> clearAll() async => _box.clear();
}

// Riverpod Provider
@riverpod
FavoritesRepository favoritesRepository(FavoritesRepositoryRef ref) {
  return FavoritesRepository();
}

@riverpod
class Favorites extends _$Favorites {
  @override
  List<FavoriteProduct> build() {
    // Lắng nghe stream từ Hive
    final repo = ref.watch(favoritesRepositoryProvider);
    ref.listen(
      favoritesRepositoryProvider
          .select((_) => repo.watchAll()),
      (_, stream) {},
    );
    return repo.getAll();
  }

  Future<void> toggle(Product product) async {
    await ref.read(favoritesRepositoryProvider).toggle(product);
    state = ref.read(favoritesRepositoryProvider).getAll();
  }

  bool isFavorite(String productId) =>
      ref.read(favoritesRepositoryProvider).isFavorite(productId);
}
```

### 10.5.4. Search History Với Hive

```dart
// lib/features/search/repositories/search_history_repository.dart

class SearchHistoryRepository {
  SearchHistoryRepository()
      : _box = Hive.box<String>('search_history');

  final Box<String> _box;
  static const _maxHistory = 10;

  List<String> getHistory() => _box.values.toList().reversed.toList();

  Future<void> add(String query) async {
    final trimmed = query.trim();
    if (trimmed.isEmpty) return;

    // Xóa nếu đã tồn tại (để move lên top)
    final existingKey = _box.keys.firstWhere(
      (k) => _box.get(k) == trimmed,
      orElse: () => null,
    );
    if (existingKey != null) await _box.delete(existingKey);

    // Xóa entry cũ nhất nếu đã đủ _maxHistory
    if (_box.length >= _maxHistory) {
      await _box.delete(_box.keys.first);
    }

    await _box.add(trimmed);
  }

  Future<void> remove(String query) async {
    final key = _box.keys.firstWhere(
      (k) => _box.get(k) == query,
      orElse: () => null,
    );
    if (key != null) await _box.delete(key);
  }

  Future<void> clear() => _box.clear();
}
```

---

## 10.6. Offline-First Architecture

### 10.6.1. Cache-Then-Network Pattern

```dart
// ✅ CHUẨN — Hiển thị cache ngay, refresh từ network
@riverpod
class ProductListWithCache extends _$ProductListWithCache {
  static const _cacheKey = 'product_list_cache';

  @override
  Future<List<Product>> build() async {
    // 1. Load cache ngay lập tức
    final cached = await _loadFromCache();
    if (cached.isNotEmpty) {
      // Emit cached data trước
      // Sau đó refresh ngầm từ network
      _refreshInBackground();
      return cached;
    }

    // 2. Nếu không có cache: load từ network
    return _fetchFromNetwork();
  }

  Future<List<Product>> _loadFromCache() async {
    final box = Hive.box<String>('product_cache');
    final json = box.get(_cacheKey);
    if (json == null) return [];

    try {
      final list = jsonDecode(json) as List;
      return list
          .map((e) => Product.fromJson(e as Map<String, dynamic>))
          .toList();
    } catch (_) {
      return [];
    }
  }

  Future<List<Product>> _fetchFromNetwork() async {
    final result = await ref
        .read(productRepositoryProvider)
        .fetchProducts();
    await _saveToCache(result.items);
    return result.items;
  }

  Future<void> _saveToCache(List<Product> products) async {
    final box = Hive.box<String>('product_cache');
    final json = jsonEncode(products.map((p) => p.toJson()).toList());
    await box.put(_cacheKey, json);
  }

  void _refreshInBackground() {
    // Chạy background refresh, cập nhật state khi xong
    Future.microtask(() async {
      try {
        final fresh = await _fetchFromNetwork();
        state = AsyncData(fresh);
      } catch (_) {
        // Silent fail — cache vẫn hiển thị
      }
    });
  }
}
```

### 10.6.2. Connectivity Awareness

```dart
// lib/core/providers/connectivity_provider.dart
// Detect khi nào có/mất mạng

@riverpod
Stream<ConnectivityResult> connectivity(ConnectivityRef ref) {
  return Connectivity().onConnectivityChanged;
}

@riverpod
bool hasInternet(HasInternetRef ref) {
  final result = ref.watch(connectivityProvider).valueOrNull;
  return result != null && result != ConnectivityResult.none;
}

// Offline banner — hiển thị khi mất mạng
class OfflineBanner extends ConsumerWidget {
  const OfflineBanner({super.key, required this.child});
  final Widget child;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final isOnline = ref.watch(hasInternetProvider);

    return Column(
      children: [
        AnimatedContainer(
          duration: const Duration(milliseconds: 300),
          height: isOnline ? 0 : 36,
          color: context.colorScheme.error,
          child: isOnline
              ? null
              : Center(
                  child: Row(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      const Icon(Icons.wifi_off, color: Colors.white, size: 16),
                      const SizedBox(width: 8),
                      Text(
                        'Không có kết nối mạng',
                        style: context.textTheme.labelMedium?.copyWith(
                          color: Colors.white,
                        ),
                      ),
                    ],
                  ),
                ),
        ),
        Expanded(child: child),
      ],
    );
  }
}
```

---

## 10.7. Bài Tập: Offline Favorites

Xây dựng tính năng yêu thích hoàn chỉnh với Hive:

```dart
// FavoriteButton widget — tích hợp animation + Hive storage
class FavoriteButton extends ConsumerWidget {
  const FavoriteButton({super.key, required this.product});
  final Product product;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // Chỉ watch isFavorite, không rebuild cho các thay đổi khác
    final isFavorite = ref.watch(
      favoritesProvider.select((list) =>
          list.any((f) => f.id == product.id)),
    );

    return PulsingFavoriteButton( // Widget animation từ chương 9
      isFavorite: isFavorite,
      onToggle: () async {
        await ref.read(favoritesProvider.notifier).toggle(product);
        if (context.mounted) {
          AppSnackBar.showSuccess(
            context,
            isFavorite
                ? 'Đã xóa khỏi yêu thích'
                : 'Đã thêm vào yêu thích',
          );
        }
      },
    );
  }
}

// FavoritesScreen
class FavoritesScreen extends ConsumerWidget {
  const FavoritesScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final favorites = ref.watch(favoritesProvider);

    if (favorites.isEmpty) {
      return const Scaffold(
        body: EmptyState.favorites(),
      );
    }

    return Scaffold(
      appBar: AppBar(
        title: Text('Yêu thích (${favorites.length})'),
      ),
      body: ListView.separated(
        padding: const EdgeInsets.all(16),
        itemCount: favorites.length,
        separatorBuilder: (_, __) => const SizedBox(height: 12),
        itemBuilder: (context, index) {
          final item = favorites[index];
          return Dismissible(
            key: ValueKey(item.id),
            direction: DismissDirection.endToStart,
            background: Container(
              alignment: Alignment.centerRight,
              padding: const EdgeInsets.only(right: 16),
              decoration: BoxDecoration(
                color: context.colorScheme.error,
                borderRadius: BorderRadius.circular(12),
              ),
              child: const Icon(Icons.delete_outline, color: Colors.white),
            ),
            onDismissed: (_) {
              ref.read(favoritesProvider.notifier).toggle(
                Product(id: item.id, name: item.name, price: item.price,
                    imageUrl: item.imageUrl, categoryId: '', status: ProductStatus.active,
                    createdAt: item.addedAt),
              );
            },
            child: FavoriteProductTile(item: item),
          );
        },
      ),
    );
  }
}
```

---

## Tóm Tắt Chương 10

| Công cụ | Dùng cho | Đặc điểm |
|---|---|---|
| `shared_preferences` | Settings, flags, simple config | Plain text, sync-ish API |
| `flutter_secure_storage` | Token, password, private key | Encrypted, Keychain/Keystore |
| `cached_network_image` | Cache ảnh từ URL | Disk cache, placeholder, error |
| `Hive` | Structured local data, danh sách | NoSQL, nhanh, code gen |
| `sqflite/drift` | Relational data, complex query | SQL, powerful nhưng verbose |

| Pattern | Điểm Cốt Lõi |
|---|---|
| Abstraction layer | Không inject SharedPreferences/Hive trực tiếp — dễ test |
| Cache key là const | Tránh typo — dùng `static const _key = 'key'` |
| memCacheWidth/Height | Resize ảnh trong RAM — tránh OOM trong GridView |
| Cache-then-network | Hiển thị cache ngay, refresh ngầm — UX tốt hơn |
| Offline banner | Thông báo rõ ràng khi mất mạng |

> **Nguyên tắc quan trọng nhất:** Không bao giờ lưu access token trong `shared_preferences`. Đây là lỗi bảo mật nghiêm trọng và phổ biến ở junior developer. Token **phải** được lưu trong `flutter_secure_storage` — đây là yêu cầu tối thiểu để app pass security review của App Store và Google Play.