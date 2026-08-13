# Chương 6: HTTP, Dio & Xử Lý API

---

## 6.1. Tổng Quan Kiến Trúc Data Layer

### 6.1.1. Tại Sao Không Gọi API Thẳng Trong Widget?

Trước khi đi vào Dio, cần hiểu nguyên tắc kiến trúc: **widget không bao giờ gọi API trực tiếp**. Đây là antipattern phổ biến ở người mới học.

```dart
// ❌ ANTIPATTERN — Gọi API trong widget
class ProductListScreen extends StatefulWidget {
  @override
  State<ProductListScreen> createState() => _ProductListScreenState();
}

class _ProductListScreenState extends State<ProductListScreen> {
  List<Product> products = [];

  @override
  void initState() {
    super.initState();
    // SAI: HTTP call, JSON parse, error handling đều trong widget
    http.get(Uri.parse('https://api.example.com/products')).then((response) {
      setState(() {
        products = (jsonDecode(response.body) as List)
            .map((e) => Product.fromJson(e))
            .toList();
      });
    });
  }
}
```

**Hậu quả:** Không thể test, không thể tái sử dụng logic, không có error handling nhất quán, widget bị coupled với implementation detail của API.

### 6.1.2. Data Layer Architecture

```
Widget / UI Layer
      ↓ (watch provider)
Provider / ViewModel Layer (Riverpod)
      ↓ (call method)
Repository Layer (abstract interface)
      ↓ (call method)
Data Source Layer (Dio + API / Local DB)
      ↓ (HTTP request)
Remote API
```

Mỗi tầng chỉ biết về tầng ngay dưới nó. Widget không biết Dio tồn tại. Repository không biết widget tồn tại. Điều này cho phép thay thế từng tầng mà không ảnh hưởng các tầng còn lại.

---

## 6.2. Dio — HTTP Client

### 6.2.1. Tại Sao Dio Thay Vì http Package?

Package `http` là HTTP client thuần túy, built-in của Dart. Đủ dùng cho app đơn giản nhưng thiếu nhiều tính năng quan trọng:

| Tính năng | `http` | `Dio` |
|---|---|---|
| Interceptor | ❌ | ✅ |
| Retry logic | ❌ | ✅ (với dio_smart_retry) |
| Cancel request | ❌ | ✅ (CancelToken) |
| Upload progress | Thủ công | ✅ |
| Download file | Thủ công | ✅ |
| Base URL + config | Thủ công | ✅ |
| FormData | Thủ công | ✅ |
| Timeout per request | Hạn chế | ✅ |

### 6.2.2. Cài Đặt

```yaml
# pubspec.yaml
dependencies:
  dio: ^5.4.0
  dio_smart_retry: ^6.0.0        # Auto retry khi lỗi mạng
  pretty_dio_logger: ^1.3.1      # Log đẹp trong debug
```

### 6.2.3. Cấu Hình Dio Cơ Bản

```dart
// lib/core/network/dio_client.dart

class DioClient {
  DioClient._();

  static Dio create({
    required String baseUrl,
    required TokenStorage tokenStorage,
  }) {
    final dio = Dio(
      BaseOptions(
        baseUrl: baseUrl,
        connectTimeout: const Duration(seconds: 15),
        receiveTimeout: const Duration(seconds: 30),
        sendTimeout: const Duration(seconds: 15),
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json',
          // Thêm header nhận diện platform
          'X-Platform': Platform.operatingSystem,
          'X-App-Version': PackageInfo.version,
        },
        // validateStatus: Quyết định HTTP status nào được coi là thành công
        // Mặc định: 200-299. Mở rộng nếu API trả về 201, 204...
        validateStatus: (status) => status != null && status < 500,
      ),
    );

    // Đăng ký interceptors theo thứ tự
    dio.interceptors.addAll([
      // 1. Auth interceptor — inject token vào request
      AuthInterceptor(dio: dio, tokenStorage: tokenStorage),

      // 2. Retry interceptor — tự retry khi mất mạng
      RetryInterceptor(
        dio: dio,
        retries: 3,
        retryDelays: const [
          Duration(seconds: 1),
          Duration(seconds: 2),
          Duration(seconds: 3),
        ],
        // Chỉ retry các request idempotent (GET, HEAD)
        retryEvaluator: (error, attempt) =>
            error.type != DioExceptionType.cancel &&
            error.requestOptions.method == 'GET',
      ),

      // 3. Logger — chỉ trong debug mode
      if (kDebugMode)
        PrettyDioLogger(
          requestHeader: true,
          requestBody: true,
          responseHeader: false,
          responseBody: true,
          error: true,
          compact: true,
        ),
    ]);

    return dio;
  }
}
```

---

## 6.3. Interceptor — Tâm Điểm Của Dio

Interceptor là middleware của HTTP pipeline. Mỗi interceptor nhận request/response, có thể sửa đổi, reject, hoặc pass through.

### 6.3.1. Auth Interceptor — Token Management

Đây là interceptor quan trọng nhất trong mọi app cần authentication.

```dart
// lib/core/network/interceptors/auth_interceptor.dart

class AuthInterceptor extends Interceptor {
  AuthInterceptor({
    required this.dio,
    required this.tokenStorage,
  });

  final Dio dio;
  final TokenStorage tokenStorage;

  // Mutex ngăn nhiều request cùng lúc refresh token
  bool _isRefreshing = false;
  final _refreshCompleter = <Completer<bool>>[];

  // === onRequest: Chạy TRƯỚC khi request được gửi ===
  @override
  Future<void> onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    final token = await tokenStorage.getAccessToken();

    if (token != null) {
      // Inject Bearer token vào mọi request
      options.headers['Authorization'] = 'Bearer $token';
    }

    // Tiếp tục gửi request
    handler.next(options);
  }

  // === onResponse: Chạy SAU khi nhận response ===
  @override
  void onResponse(Response response, ResponseInterceptorHandler handler) {
    // Có thể transform response ở đây
    // Ví dụ: unwrap data từ { "success": true, "data": {...} }
    handler.next(response);
  }

  // === onError: Chạy khi có lỗi ===
  @override
  Future<void> onError(
    DioException err,
    ErrorInterceptorHandler handler,
  ) async {
    // Chỉ xử lý 401 Unauthorized — token expired
    if (err.response?.statusCode != 401) {
      handler.next(err);
      return;
    }

    // Tránh vòng lặp vô hạn: nếu request refresh token cũng bị 401 thì logout
    if (err.requestOptions.path.contains('/auth/refresh')) {
      await _handleLogout();
      handler.next(err);
      return;
    }

    // Nếu đang refresh: đợi kết quả từ request refresh đang chạy
    if (_isRefreshing) {
      final completer = Completer<bool>();
      _refreshCompleter.add(completer);
      final success = await completer.future;

      if (success) {
        // Retry request ban đầu với token mới
        handler.resolve(await _retryRequest(err.requestOptions));
      } else {
        handler.next(err);
      }
      return;
    }

    // Bắt đầu refresh token
    _isRefreshing = true;
    final refreshSuccess = await _refreshToken();
    _isRefreshing = false;

    // Notify tất cả request đang đợi
    for (final completer in _refreshCompleter) {
      completer.complete(refreshSuccess);
    }
    _refreshCompleter.clear();

    if (refreshSuccess) {
      handler.resolve(await _retryRequest(err.requestOptions));
    } else {
      await _handleLogout();
      handler.next(err);
    }
  }

  Future<bool> _refreshToken() async {
    try {
      final refreshToken = await tokenStorage.getRefreshToken();
      if (refreshToken == null) return false;

      // Gọi API refresh — dùng Dio khác không có auth interceptor
      // để tránh vòng lặp vô hạn
      final response = await Dio().post(
        '${AppConfig.apiBaseUrl}/auth/refresh',
        data: {'refresh_token': refreshToken},
      );

      final newAccessToken = response.data['access_token'] as String;
      final newRefreshToken = response.data['refresh_token'] as String?;

      await tokenStorage.saveTokens(
        accessToken: newAccessToken,
        refreshToken: newRefreshToken ?? refreshToken,
      );

      return true;
    } catch (_) {
      return false;
    }
  }

  Future<Response<dynamic>> _retryRequest(RequestOptions options) async {
    final token = await tokenStorage.getAccessToken();
    options.headers['Authorization'] = 'Bearer $token';
    return dio.fetch(options);
  }

  Future<void> _handleLogout() async {
    await tokenStorage.clearTokens();
    // Navigate về login — thông qua global event hoặc GoRouter redirect
    AppRouter.navigateToLogin();
  }
}
```

### 6.3.2. Error Interceptor — Chuẩn Hóa Lỗi

```dart
// lib/core/network/interceptors/error_interceptor.dart

class ErrorInterceptor extends Interceptor {
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    // Chuyển đổi DioException thành AppException có ý nghĩa hơn
    final appError = _mapDioError(err);
    // Throw AppException thay vì DioException
    handler.reject(
      DioException(
        requestOptions: err.requestOptions,
        error: appError,
        response: err.response,
        type: err.type,
      ),
    );
  }

  AppException _mapDioError(DioException err) {
    return switch (err.type) {
      DioExceptionType.connectionTimeout ||
      DioExceptionType.receiveTimeout ||
      DioExceptionType.sendTimeout =>
        const NetworkTimeoutException('Kết nối quá chậm. Vui lòng thử lại.'),

      DioExceptionType.connectionError =>
        const NoInternetException('Không có kết nối mạng.'),

      DioExceptionType.badResponse => _mapHttpError(err.response),

      DioExceptionType.cancel =>
        const RequestCancelledException('Yêu cầu đã bị hủy.'),

      _ => const UnknownException('Đã xảy ra lỗi không xác định.'),
    };
  }

  AppException _mapHttpError(Response? response) {
    if (response == null) return const UnknownException('Không có phản hồi từ server.');

    // Parse error message từ response body nếu có
    String? serverMessage;
    try {
      final data = response.data;
      if (data is Map<String, dynamic>) {
        serverMessage = data['message'] as String? ??
            data['error'] as String? ??
            data['detail'] as String?;
      }
    } catch (_) {}

    return switch (response.statusCode) {
      400 => BadRequestException(serverMessage ?? 'Dữ liệu không hợp lệ.'),
      401 => const UnauthorizedException('Phiên đăng nhập đã hết hạn.'),
      403 => const ForbiddenException('Bạn không có quyền thực hiện thao tác này.'),
      404 => NotFoundException(serverMessage ?? 'Không tìm thấy dữ liệu.'),
      409 => ConflictException(serverMessage ?? 'Dữ liệu đã tồn tại.'),
      422 => ValidationException(response.data),
      429 => const TooManyRequestsException('Quá nhiều yêu cầu. Vui lòng thử lại sau.'),
      500 || 502 || 503 =>
        const ServerException('Lỗi server. Vui lòng thử lại sau.'),
      _ => UnknownException('Lỗi HTTP ${response.statusCode}.'),
    };
  }
}
```

---

## 6.4. JSON Parsing và Data Models

### 6.4.1. Vấn Đề Với Parse Thủ Công

```dart
// ❌ ANTIPATTERN — Parse JSON thủ công, rất dễ lỗi runtime
Map<String, dynamic> json = response.data;
final product = Product(
  id: json['id'],           // Lỗi nếu API đổi field name
  name: json['name'],       // Không có type check
  price: json['price'],     // Lỗi nếu price là int nhưng expect double
);
```

### 6.4.2. freezed + json_serializable — Cách Chuẩn

`freezed` là thư viện tạo immutable value classes với `copyWith`, `==`, `hashCode`, pattern matching và JSON serialization tự động.

```yaml
# pubspec.yaml
dependencies:
  freezed_annotation: ^2.4.0
  json_annotation: ^4.8.0

dev_dependencies:
  freezed: ^2.4.0
  json_serializable: ^6.7.0
  build_runner: ^2.4.0
```

```dart
// lib/features/products/models/product.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'product.freezed.dart';   // Generated bởi freezed
part 'product.g.dart';         // Generated bởi json_serializable

@freezed
class Product with _$Product {
  const factory Product({
    required String id,
    required String name,
    required String description,
    required double price,
    @Default(0) double? originalPrice,   // Optional với default value
    required String imageUrl,
    @Default([]) List<String> images,
    required String categoryId,
    required ProductStatus status,
    @Default(0) int stock,
    @JsonKey(name: 'created_at') required DateTime createdAt, // Đổi tên field
  }) = _Product;

  // factory từ JSON — json_serializable tự generate
  factory Product.fromJson(Map<String, dynamic> json) => _$ProductFromJson(json);

  // Private constructor để thêm computed properties
  const Product._();

  // Computed properties — không được serialize
  bool get isInStock => stock > 0;
  bool get isOnSale => originalPrice != null && originalPrice! > price;
  double get discountPercent =>
      isOnSale ? ((originalPrice! - price) / originalPrice! * 100) : 0;
  String get formattedPrice => '${price.toStringAsFixed(0)}đ';
}

// Enum với JSON serialization
@JsonEnum(valueField: 'value')
enum ProductStatus {
  @JsonValue('active') active('active'),
  @JsonValue('inactive') inactive('inactive'),
  @JsonValue('out_of_stock') outOfStock('out_of_stock');

  const ProductStatus(this.value);
  final String value;
}
```

```bash
# Generate code
dart run build_runner build --delete-conflicting-outputs
```

```dart
// Sau khi generate — sử dụng
final product = Product.fromJson(json);

// copyWith — immutable update, rất tiện
final updated = product.copyWith(stock: product.stock - 1);

// Pattern matching với Freezed sealed classes
sealed class ProductEvent {}
class AddToCart extends ProductEvent { final Product product; }
class RemoveFromCart extends ProductEvent { final String id; }

// Switch expression type-safe
String handleEvent(ProductEvent event) => switch (event) {
  AddToCart(:final product) => 'Thêm ${product.name}',
  RemoveFromCart(:final id) => 'Xóa sản phẩm $id',
};
```

### 6.4.3. Pagination Response Model

```dart
// lib/core/models/paginated_response.dart
@freezed
class PaginatedResponse<T> with _$PaginatedResponse<T> {
  const factory PaginatedResponse({
    required List<T> items,
    required int total,
    required int page,
    required int pageSize,
    required int totalPages,
  }) = _PaginatedResponse<T>;

  factory PaginatedResponse.fromJson(
    Map<String, dynamic> json,
    T Function(Object?) fromJsonT,
  ) => _$PaginatedResponseFromJson(json, fromJsonT);

  const PaginatedResponse._();

  bool get hasNextPage => page < totalPages;
  bool get hasPreviousPage => page > 1;
}
```

---

## 6.5. Repository Pattern

Repository là tầng abstraction giữa data source (Dio) và business logic (Riverpod provider). Widget và provider chỉ biết đến interface của repository, không biết implementation bên dưới.

```dart
// lib/features/products/repositories/product_repository.dart

// 1. Abstract interface — định nghĩa contract
abstract interface class ProductRepository {
  Future<PaginatedResponse<Product>> fetchProducts({
    String? category,
    String? searchQuery,
    ProductSortOrder sort = ProductSortOrder.newest,
    int page = 1,
    int pageSize = 20,
    CancelToken? cancelToken,
  });

  Future<ProductDetail> fetchProductDetail(
    String id, {
    CancelToken? cancelToken,
  });

  Future<List<Product>> fetchRelatedProducts(String productId);

  Future<List<Product>> fetchFeaturedProducts();
}

// 2. Implementation — dùng Dio
class ProductRepositoryImpl implements ProductRepository {
  const ProductRepositoryImpl({required this.dio});
  final Dio dio;

  @override
  Future<PaginatedResponse<Product>> fetchProducts({
    String? category,
    String? searchQuery,
    ProductSortOrder sort = ProductSortOrder.newest,
    int page = 1,
    int pageSize = 20,
    CancelToken? cancelToken,
  }) async {
    try {
      final response = await dio.get(
        '/products',
        queryParameters: {
          if (category != null) 'category': category,
          if (searchQuery != null && searchQuery.isNotEmpty) 'q': searchQuery,
          'sort': sort.value,
          'page': page,
          'page_size': pageSize,
        },
        cancelOptions: cancelToken != null
            ? CancelOptions(cancelToken: cancelToken)
            : null,
      );

      return PaginatedResponse<Product>.fromJson(
        response.data as Map<String, dynamic>,
        (json) => Product.fromJson(json as Map<String, dynamic>),
      );
    } on DioException catch (e) {
      // Re-throw AppException đã được map bởi ErrorInterceptor
      throw e.error ?? e;
    }
  }

  @override
  Future<ProductDetail> fetchProductDetail(
    String id, {
    CancelToken? cancelToken,
  }) async {
    final response = await dio.get(
      '/products/$id',
      cancelOptions: cancelToken != null
          ? CancelOptions(cancelToken: cancelToken)
          : null,
    );

    return ProductDetail.fromJson(response.data as Map<String, dynamic>);
  }

  @override
  Future<List<Product>> fetchRelatedProducts(String productId) async {
    final response = await dio.get('/products/$productId/related');
    final items = response.data['items'] as List;
    return items
        .map((e) => Product.fromJson(e as Map<String, dynamic>))
        .toList();
  }

  @override
  Future<List<Product>> fetchFeaturedProducts() async {
    final response = await dio.get('/products/featured');
    final items = response.data['items'] as List;
    return items
        .map((e) => Product.fromJson(e as Map<String, dynamic>))
        .toList();
  }
}
```

---

## 6.6. Xử Lý Lỗi Đúng Cách — Result Pattern

### 6.6.1. Vấn Đề Của Exception

Exception trong Dart không được kiểm tra ở compile time. Caller không biết function có thể throw exception gì mà không đọc implementation. Điều này dẫn đến lỗi runtime không được xử lý.

```dart
// ❌ Exception-based: caller có thể quên try-catch
final products = await repo.fetchProducts(); // Có thể throw bất cứ gì!
```

### 6.6.2. Result Type Với Dart Sealed Classes

```dart
// lib/core/utils/result.dart

// Sealed class — exhaustive pattern matching ở compile time
sealed class Result<T> {
  const Result();

  // Factory constructors tiện lợi
  static Result<T> success<T>(T data) => Success(data);
  static Result<T> failure<T>(AppException error) => Failure(error);

  // Utility methods
  bool get isSuccess => this is Success<T>;
  bool get isFailure => this is Failure<T>;

  T? get valueOrNull => switch (this) {
        Success(:final data) => data,
        Failure() => null,
      };

  AppException? get errorOrNull => switch (this) {
        Success() => null,
        Failure(:final error) => error,
      };

  // Transform value nếu thành công
  Result<U> map<U>(U Function(T data) transform) => switch (this) {
        Success(:final data) => Result.success(transform(data)),
        Failure(:final error) => Result.failure(error),
      };

  // FlatMap — chain multiple Result operations
  Future<Result<U>> flatMap<U>(
    Future<Result<U>> Function(T data) transform,
  ) async =>
      switch (this) {
        Success(:final data) => transform(data),
        Failure(:final error) => Result.failure(error),
      };

  // Execute callback và bắt exception
  static Future<Result<T>> guard<T>(Future<T> Function() action) async {
    try {
      return Result.success(await action());
    } on AppException catch (e) {
      return Result.failure(e);
    } catch (e) {
      return Result.failure(UnknownException(e.toString()));
    }
  }
}

class Success<T> extends Result<T> {
  const Success(this.data);
  final T data;
}

class Failure<T> extends Result<T> {
  const Failure(this.error);
  final AppException error;
}
```

### 6.6.3. Repository Với Result Pattern

```dart
// Repository trả về Result thay vì throw exception
abstract interface class AuthRepository {
  Future<Result<AuthTokens>> login({
    required String email,
    required String password,
  });

  Future<Result<AppUser>> getCurrentUser();
  Future<Result<void>> logout();
}

class AuthRepositoryImpl implements AuthRepository {
  const AuthRepositoryImpl({required this.dio});
  final Dio dio;

  @override
  Future<Result<AuthTokens>> login({
    required String email,
    required String password,
  }) =>
      Result.guard(() async {
        final response = await dio.post('/auth/login', data: {
          'email': email,
          'password': password,
        });
        return AuthTokens.fromJson(response.data as Map<String, dynamic>);
      });

  @override
  Future<Result<AppUser>> getCurrentUser() =>
      Result.guard(() async {
        final response = await dio.get('/auth/me');
        return AppUser.fromJson(response.data as Map<String, dynamic>);
      });

  @override
  Future<Result<void>> logout() =>
      Result.guard(() => dio.post('/auth/logout'));
}

// Sử dụng trong Riverpod Notifier:
class Auth extends _$Auth {
  Future<void> login({required String email, required String password}) async {
    state = const AsyncLoading();

    final result = await ref.read(authRepositoryProvider)
        .login(email: email, password: password);

    state = switch (result) {
      Success(:final data) => AsyncData(AuthAuthenticated(data.user)),
      Failure(:final error) => AsyncError(error, StackTrace.current),
    };
  }
}
```

---

## 6.7. Upload File và FormData

```dart
// ✅ CHUẨN — Upload ảnh sản phẩm với progress tracking
class ProductImageUploader {
  const ProductImageUploader({required this.dio});
  final Dio dio;

  Future<Result<String>> uploadImage(
    File imageFile, {
    void Function(int sent, int total)? onProgress,
  }) =>
      Result.guard(() async {
        // FormData cho multipart upload
        final formData = FormData.fromMap({
          'file': await MultipartFile.fromFile(
            imageFile.path,
            filename: path.basename(imageFile.path),
            contentType: MediaType('image', _getExtension(imageFile.path)),
          ),
          'folder': 'products',
        });

        final response = await dio.post(
          '/upload',
          data: formData,
          onSendProgress: onProgress, // Callback progress
          options: Options(
            // Tăng timeout cho upload
            sendTimeout: const Duration(minutes: 2),
          ),
        );

        return response.data['url'] as String;
      });

  String _getExtension(String filePath) {
    final ext = path.extension(filePath).toLowerCase().replaceAll('.', '');
    return switch (ext) {
      'jpg' || 'jpeg' => 'jpeg',
      'png' => 'png',
      'webp' => 'webp',
      _ => 'jpeg',
    };
  }
}

// Sử dụng trong widget:
class ImageUploadButton extends ConsumerStatefulWidget {
  const ImageUploadButton({super.key, required this.onUploaded});
  final ValueChanged<String> onUploaded;

  @override
  ConsumerState<ImageUploadButton> createState() => _ImageUploadButtonState();
}

class _ImageUploadButtonState extends ConsumerState<ImageUploadButton> {
  double _progress = 0;
  bool _isUploading = false;

  Future<void> _pickAndUpload() async {
    final picker = ImagePicker();
    final picked = await picker.pickImage(
      source: ImageSource.gallery,
      maxWidth: 1200,
      imageQuality: 85,
    );
    if (picked == null) return;

    setState(() {
      _isUploading = true;
      _progress = 0;
    });

    final uploader = ref.read(productImageUploaderProvider);
    final result = await uploader.uploadImage(
      File(picked.path),
      onProgress: (sent, total) {
        setState(() => _progress = sent / total);
      },
    );

    setState(() => _isUploading = false);

    switch (result) {
      case Success(:final data):
        widget.onUploaded(data);
      case Failure(:final error):
        if (mounted) {
          ScaffoldMessenger.of(context).showSnackBar(
            SnackBar(content: Text(error.message)),
          );
        }
    }
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: _isUploading ? null : _pickAndUpload,
      child: Container(
        width: 120,
        height: 120,
        decoration: BoxDecoration(
          color: context.colorScheme.surfaceContainerHighest,
          borderRadius: BorderRadius.circular(12),
          border: Border.all(color: context.colorScheme.outlineVariant),
        ),
        child: _isUploading
            ? Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  CircularProgressIndicator(value: _progress),
                  const SizedBox(height: 8),
                  Text('${(_progress * 100).toInt()}%'),
                ],
              )
            : const Icon(Icons.add_photo_alternate_outlined, size: 32),
      ),
    );
  }
}
```

---

## 6.8. Caching Với Riverpod

Riverpod có cơ chế caching tích hợp sẵn: provider giữ giá trị trong memory cho đến khi bị invalidate hoặc không còn ai watch nó.

```dart
// ✅ CHUẨN — Cache control với keepAlive và invalidation

// Provider tự động dispose khi không có widget nào watch
// Nếu muốn giữ cache lâu hơn, dùng keepAlive
@riverpod
Future<List<Category>> categories(CategoriesRef ref) async {
  // keepAlive: giữ provider trong memory kể cả khi không có widget nào watch
  // Phù hợp cho data ít thay đổi như danh mục, cấu hình
  ref.keepAlive();

  return ref.watch(categoryRepositoryProvider).fetchCategories();
}

// Cache với time-based expiry
@riverpod
Future<List<Product>> featuredProducts(FeaturedProductsRef ref) async {
  // Tự invalidate sau 5 phút
  final timer = Timer(const Duration(minutes: 5), () {
    ref.invalidateSelf();
  });
  ref.onDispose(() => timer.cancel());
  ref.keepAlive();

  return ref.watch(productRepositoryProvider).fetchFeaturedProducts();
}

// Manual cache invalidation — khi user thực hiện action làm data thay đổi
class ProductActions {
  static void onProductUpdated(WidgetRef ref, String productId) {
    // Invalidate detail của product đó
    ref.invalidate(productDetailProvider(productId));
    // Invalidate list (vì có thể item trong list đã thay đổi)
    ref.invalidate(productListProvider);
    // Invalidate featured nếu product đó có thể trong featured
    ref.invalidate(featuredProductsProvider);
  }
}
```

---

## 6.9. Mock Repository Cho Development

Trong giai đoạn đầu phát triển, API backend thường chưa sẵn sàng. Mock repository cho phép frontend team làm việc độc lập.

```dart
// lib/features/products/repositories/mock_product_repository.dart

class MockProductRepository implements ProductRepository {
  MockProductRepository({Duration delay = const Duration(milliseconds: 800)})
      : _delay = delay;

  final Duration _delay;

  // Mock data
  static final _mockProducts = List.generate(
    50,
    (i) => Product(
      id: 'product-$i',
      name: 'Sản phẩm ${i + 1}',
      description: 'Mô tả sản phẩm ${i + 1}',
      price: (100000 + i * 50000).toDouble(),
      imageUrl: 'https://picsum.photos/seed/$i/400/400',
      categoryId: 'cat-${i % 5}',
      status: ProductStatus.active,
      stock: i % 3 == 0 ? 0 : 10,
      createdAt: DateTime.now().subtract(Duration(days: i)),
    ),
  );

  @override
  Future<PaginatedResponse<Product>> fetchProducts({
    String? category,
    String? searchQuery,
    ProductSortOrder sort = ProductSortOrder.newest,
    int page = 1,
    int pageSize = 20,
    CancelToken? cancelToken,
  }) async {
    await Future.delayed(_delay); // Simulate network latency

    var filtered = _mockProducts.where((p) {
      if (category != null && p.categoryId != category) return false;
      if (searchQuery != null && searchQuery.isNotEmpty) {
        return p.name.toLowerCase().contains(searchQuery.toLowerCase());
      }
      return true;
    }).toList();

    // Simulate sorting
    filtered.sort((a, b) => switch (sort) {
          ProductSortOrder.priceAsc => a.price.compareTo(b.price),
          ProductSortOrder.priceDesc => b.price.compareTo(a.price),
          _ => b.createdAt.compareTo(a.createdAt),
        });

    // Simulate pagination
    final start = (page - 1) * pageSize;
    final end = (start + pageSize).clamp(0, filtered.length);
    final items = start < filtered.length ? filtered.sublist(start, end) : <Product>[];

    return PaginatedResponse(
      items: items,
      total: filtered.length,
      page: page,
      pageSize: pageSize,
      totalPages: (filtered.length / pageSize).ceil(),
    );
  }

  @override
  Future<ProductDetail> fetchProductDetail(String id, {CancelToken? cancelToken}) async {
    await Future.delayed(_delay);
    final product = _mockProducts.firstWhere(
      (p) => p.id == id,
      orElse: () => throw const NotFoundException('Không tìm thấy sản phẩm'),
    );
    return ProductDetail(product: product, reviews: [], relatedIds: []);
  }

  @override
  Future<List<Product>> fetchRelatedProducts(String productId) async {
    await Future.delayed(_delay);
    return _mockProducts.take(6).toList();
  }

  @override
  Future<List<Product>> fetchFeaturedProducts() async {
    await Future.delayed(_delay);
    return _mockProducts.take(8).toList();
  }
}

// Chuyển đổi giữa mock và real implementation qua Riverpod override
// lib/core/providers/repository_providers.dart
@riverpod
ProductRepository productRepository(ProductRepositoryRef ref) {
  // Đổi giữa mock và real dựa trên build flavor
  if (AppConfig.useMockData) {
    return MockProductRepository();
  }
  return ProductRepositoryImpl(dio: ref.watch(dioProvider));
}
```

---

## 6.10. Bài Tập: API Integration Hoàn Chỉnh

Bài tập xây dựng luồng tìm kiếm sản phẩm end-to-end:

```dart
// 1. Debounced search provider — tránh gọi API liên tục khi user gõ
@riverpod
class SearchController extends _$SearchController {
  Timer? _debounceTimer;

  @override
  String build() => '';

  void updateQuery(String query) {
    // Hủy timer cũ
    _debounceTimer?.cancel();

    // Đặt timer mới — chỉ update state sau 500ms không gõ
    _debounceTimer = Timer(const Duration(milliseconds: 500), () {
      state = query;
    });
  }

  @override
  void dispose() {
    _debounceTimer?.cancel();
    super.dispose();
  }
}

// 2. Search results tự động cập nhật khi query thay đổi
@riverpod
Future<PaginatedResponse<Product>> searchResults(SearchResultsRef ref) async {
  final query = ref.watch(searchControllerProvider);

  // Nếu query trống: không tìm kiếm
  if (query.isEmpty) return const PaginatedResponse(
    items: [], total: 0, page: 1, pageSize: 20, totalPages: 0,
  );

  // CancelToken: hủy request cũ khi query mới đến
  final cancelToken = CancelToken();
  ref.onDispose(() => cancelToken.cancel());

  return ref.watch(productRepositoryProvider).fetchProducts(
    searchQuery: query,
    cancelToken: cancelToken,
  );
}

// 3. Search Screen
class SearchScreen extends ConsumerWidget {
  const SearchScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Scaffold(
      appBar: AppBar(
        title: TextField(
          autofocus: true,
          decoration: const InputDecoration(
            hintText: 'Tìm kiếm sản phẩm...',
            border: InputBorder.none,
          ),
          onChanged: (query) {
            ref.read(searchControllerProvider.notifier).updateQuery(query);
          },
        ),
      ),
      body: Consumer(
        builder: (context, ref, _) {
          final query = ref.watch(searchControllerProvider);
          if (query.isEmpty) return const SearchSuggestions();

          final resultsAsync = ref.watch(searchResultsProvider);
          return resultsAsync.when(
            loading: () => const SearchSkeleton(),
            error: (e, _) => ErrorView(message: e.toString()),
            data: (results) => results.items.isEmpty
                ? EmptyState(query: query)
                : ProductGrid(products: results.items),
          );
        },
      ),
    );
  }
}
```

---

## Tóm Tắt Chương 6

| Khái niệm | Điểm Cốt Lõi |
|---|---|
| Widget không gọi API | Mọi HTTP call đi qua Repository → Provider → Widget |
| Dio vs http | Dio: interceptor, retry, cancel, upload progress |
| AuthInterceptor | Inject token, auto refresh khi 401, handle logout |
| ErrorInterceptor | Map DioException → AppException có ý nghĩa |
| freezed | Immutable model, copyWith, JSON, pattern matching |
| Repository pattern | Abstract interface → dễ mock, dễ test, decoupled |
| Result type | Explicit error handling, không throw exception ngẫu nhiên |
| CancelToken | Hủy request khi widget dispose — tránh memory leak |
| keepAlive | Cache provider trong memory khi không có widget nào watch |
| Mock repository | Phát triển frontend độc lập với backend |

> **Bài học quan trọng nhất:** Đầu tư vào error handling ngay từ đầu. App thực tế luôn gặp lỗi mạng, timeout, server error, token expired. Code không xử lý được những trường hợp này sẽ tạo ra trải nghiệm tệ cho người dùng và bug khó debug trong production. Một `ErrorInterceptor` tốt và `Result` type nhất quán sẽ tiết kiệm hàng chục giờ debug sau này.
