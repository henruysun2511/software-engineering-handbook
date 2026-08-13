# Chương 14: Tính Năng Native & Platform

---

## 14.1. Firebase Integration

### 14.1.1. Setup Firebase

```bash
# Cài FlutterFire CLI
dart pub global activate flutterfire_cli

# Login Firebase
firebase login

# Cấu hình project — tự tạo google-services.json và GoogleService-Info.plist
flutterfire configure --project=my-flutter-app-prod
```

```yaml
# pubspec.yaml
dependencies:
  firebase_core: ^3.0.0
  firebase_auth: ^5.0.0
  cloud_firestore: ^5.0.0
  firebase_storage: ^12.0.0
  firebase_crashlytics: ^4.0.0
  firebase_analytics: ^11.0.0
  firebase_remote_config: ^5.0.0
  firebase_messaging: ^15.0.0
```

### 14.1.2. Firebase Auth

```dart
// lib/features/auth/data/datasources/firebase_auth_datasource.dart

class FirebaseAuthDataSource {
  FirebaseAuthDataSource() : _auth = FirebaseAuth.instance;
  final FirebaseAuth _auth;

  // Stream: lắng nghe trạng thái auth thay đổi
  Stream<User?> get authStateChanges => _auth.authStateChanges();

  // Đăng nhập email/password
  Future<UserCredential> signInWithEmail({
    required String email,
    required String password,
  }) async {
    try {
      return await _auth.signInWithEmailAndPassword(
        email: email,
        password: password,
      );
    } on FirebaseAuthException catch (e) {
      throw _mapFirebaseAuthError(e);
    }
  }

  // Đăng ký
  Future<UserCredential> signUpWithEmail({
    required String email,
    required String password,
  }) async {
    try {
      return await _auth.createUserWithEmailAndPassword(
        email: email,
        password: password,
      );
    } on FirebaseAuthException catch (e) {
      throw _mapFirebaseAuthError(e);
    }
  }

  // Google Sign-In
  Future<UserCredential> signInWithGoogle() async {
    final googleUser = await GoogleSignIn().signIn();
    if (googleUser == null) throw const RequestCancelledException();

    final googleAuth = await googleUser.authentication;
    final credential = GoogleAuthProvider.credential(
      accessToken: googleAuth.accessToken,
      idToken: googleAuth.idToken,
    );

    return _auth.signInWithCredential(credential);
  }

  // Quên mật khẩu
  Future<void> sendPasswordResetEmail(String email) async {
    try {
      await _auth.sendPasswordResetEmail(email: email);
    } on FirebaseAuthException catch (e) {
      throw _mapFirebaseAuthError(e);
    }
  }

  // Đổi mật khẩu
  Future<void> updatePassword({
    required String currentPassword,
    required String newPassword,
  }) async {
    final user = _auth.currentUser;
    if (user == null) throw const UnauthorizedException();

    // Re-authenticate trước khi đổi mật khẩu
    final credential = EmailAuthProvider.credential(
      email: user.email!,
      password: currentPassword,
    );
    await user.reauthenticateWithCredential(credential);
    await user.updatePassword(newPassword);
  }

  Future<void> signOut() => _auth.signOut();

  User? get currentUser => _auth.currentUser;

  // Map Firebase error codes sang AppException
  AppException _mapFirebaseAuthError(FirebaseAuthException e) {
    return switch (e.code) {
      'user-not-found' || 'wrong-password' || 'invalid-credential' =>
        const UnauthorizedException('Email hoặc mật khẩu không đúng'),
      'email-already-in-use' =>
        const ConflictException('Email này đã được đăng ký'),
      'weak-password' =>
        const BadRequestException('Mật khẩu quá yếu'),
      'user-disabled' =>
        const ForbiddenException('Tài khoản đã bị vô hiệu hóa'),
      'too-many-requests' =>
        const TooManyRequestsException('Quá nhiều lần thử. Vui lòng thử lại sau'),
      'network-request-failed' =>
        const NoInternetException(),
      _ => UnknownException('Lỗi xác thực: ${e.message}'),
    };
  }
}
```

### 14.1.3. Firestore — Realtime Database

```dart
// lib/features/orders/data/datasources/order_firestore_datasource.dart

class OrderFirestoreDataSource {
  OrderFirestoreDataSource()
      : _db = FirebaseFirestore.instance,
        _auth = FirebaseAuth.instance;

  final FirebaseFirestore _db;
  final FirebaseAuth _auth;

  String get _userId => _auth.currentUser!.uid;

  // Lắng nghe đơn hàng realtime
  Stream<List<Order>> watchOrders() {
    return _db
        .collection('orders')
        .where('userId', isEqualTo: _userId)
        .orderBy('createdAt', descending: true)
        .snapshots()
        .map((snap) => snap.docs
            .map((doc) => Order.fromFirestore(doc))
            .toList());
  }

  // Tạo đơn hàng mới — dùng batch write để atomic
  Future<String> createOrder(CreateOrderRequest request) async {
    final batch = _db.batch();
    final orderId = _db.collection('orders').doc().id; // Auto-generated ID

    // Tạo order document
    final orderRef = _db.collection('orders').doc(orderId);
    batch.set(orderRef, {
      'id': orderId,
      'userId': _userId,
      'items': request.items.map((i) => i.toJson()).toList(),
      'total': request.total,
      'status': 'pending',
      'shippingAddress': request.shippingAddress.toJson(),
      'createdAt': FieldValue.serverTimestamp(), // Server timestamp — không dùng DateTime.now()
      'updatedAt': FieldValue.serverTimestamp(),
    });

    // Giảm stock từng sản phẩm — trong transaction để tránh race condition
    for (final item in request.items) {
      final productRef = _db.collection('products').doc(item.productId);
      batch.update(productRef, {
        'stock': FieldValue.increment(-item.quantity),
      });
    }

    await batch.commit();
    return orderId;
  }

  // Lấy một đơn hàng theo ID
  Future<Order?> getOrder(String orderId) async {
    final doc = await _db.collection('orders').doc(orderId).get();
    if (!doc.exists) return null;
    return Order.fromFirestore(doc);
  }

  // Cập nhật trạng thái
  Future<void> updateOrderStatus(String orderId, OrderStatus status) {
    return _db.collection('orders').doc(orderId).update({
      'status': status.value,
      'updatedAt': FieldValue.serverTimestamp(),
    });
  }
}

// Riverpod provider cho realtime stream
@riverpod
Stream<List<Order>> userOrders(UserOrdersRef ref) {
  return ref.watch(orderFirestoreDataSourceProvider).watchOrders();
}
```

---

## 14.2. Push Notification với FCM

### 14.2.1. Setup FCM

```dart
// lib/core/notifications/notification_service.dart

class NotificationService {
  NotificationService._();
  static final instance = NotificationService._();

  final _messaging = FirebaseMessaging.instance;

  Future<void> initialize() async {
    // 1. Xin quyền notification (iOS bắt buộc, Android 13+ bắt buộc)
    final settings = await _messaging.requestPermission(
      alert: true,
      badge: true,
      sound: true,
      provisional: false,
    );

    if (settings.authorizationStatus == AuthorizationStatus.denied) {
      debugPrint('Notification permission denied');
      return;
    }

    // 2. Lấy FCM token — gửi lên server để gửi notification
    final token = await _messaging.getToken();
    if (token != null) {
      await _uploadTokenToServer(token);
    }

    // 3. Lắng nghe token refresh
    _messaging.onTokenRefresh.listen(_uploadTokenToServer);

    // 4. Setup local notification (hiển thị khi app foreground)
    await _setupLocalNotifications();

    // 5. Lắng nghe message khi app đang foreground
    FirebaseMessaging.onMessage.listen(_handleForegroundMessage);

    // 6. Lắng nghe khi user tap notification và app đang background
    FirebaseMessaging.onMessageOpenedApp.listen(_handleNotificationTap);

    // 7. Kiểm tra notification mở app từ terminated state
    final initialMessage = await _messaging.getInitialMessage();
    if (initialMessage != null) {
      _handleNotificationTap(initialMessage);
    }
  }

  Future<void> _uploadTokenToServer(String token) async {
    // Gửi token lên backend để server biết device này cần nhận notification
    await GetIt.I<UserRepository>().updateFcmToken(token);
  }

  Future<void> _setupLocalNotifications() async {
    final localNotifications = FlutterLocalNotificationsPlugin();

    const androidSettings = AndroidInitializationSettings('@mipmap/ic_launcher');
    const iosSettings = DarwinInitializationSettings(
      requestAlertPermission: false, // Đã xin permission ở trên
      requestBadgePermission: false,
      requestSoundPermission: false,
    );

    await localNotifications.initialize(
      const InitializationSettings(
        android: androidSettings,
        iOS: iosSettings,
      ),
      onDidReceiveNotificationResponse: (details) {
        // User tap local notification
        _handleLocalNotificationTap(details.payload);
      },
    );
  }

  Future<void> _handleForegroundMessage(RemoteMessage message) async {
    debugPrint('Foreground message: ${message.notification?.title}');

    // Hiển thị local notification vì FCM không tự hiển thị khi foreground
    final localNotifications = FlutterLocalNotificationsPlugin();
    final notification = message.notification;
    if (notification == null) return;

    await localNotifications.show(
      notification.hashCode,
      notification.title,
      notification.body,
      NotificationDetails(
        android: AndroidNotificationDetails(
          'default_channel',
          'Thông báo',
          importance: Importance.high,
          priority: Priority.high,
        ),
        iOS: const DarwinNotificationDetails(),
      ),
      payload: jsonEncode(message.data),
    );
  }

  void _handleNotificationTap(RemoteMessage message) {
    // Navigate dựa trên data payload
    final data = message.data;
    final type = data['type'] as String?;
    final id = data['id'] as String?;

    switch (type) {
      case 'order':
        AppRouter.instance.push('/orders/$id');
      case 'promotion':
        AppRouter.instance.push('/promotions/$id');
      case 'chat':
        AppRouter.instance.push('/chat/$id');
    }
  }

  void _handleLocalNotificationTap(String? payload) {
    if (payload == null) return;
    final data = jsonDecode(payload) as Map<String, dynamic>;
    _handleNotificationTap(RemoteMessage(data: data));
  }
}
```

---

## 14.3. Camera và Image Picker

```dart
// lib/core/services/media_picker_service.dart

class MediaPickerService {
  MediaPickerService._();
  static final instance = MediaPickerService._();

  final _picker = ImagePicker();

  // Chọn ảnh từ gallery
  Future<File?> pickImage({
    int? maxWidth = 1200,
    int? maxHeight = 1200,
    int? imageQuality = 85,
  }) async {
    try {
      final xFile = await _picker.pickImage(
        source: ImageSource.gallery,
        maxWidth: maxWidth?.toDouble(),
        maxHeight: maxHeight?.toDouble(),
        imageQuality: imageQuality,
      );
      return xFile != null ? File(xFile.path) : null;
    } on PlatformException catch (e) {
      if (e.code == 'photo_access_denied') {
        throw const ForbiddenException('Cần quyền truy cập thư viện ảnh');
      }
      rethrow;
    }
  }

  // Chụp ảnh từ camera
  Future<File?> takePhoto({
    int? maxWidth = 1200,
    int? imageQuality = 85,
  }) async {
    try {
      final xFile = await _picker.pickImage(
        source: ImageSource.camera,
        maxWidth: maxWidth?.toDouble(),
        imageQuality: imageQuality,
      );
      return xFile != null ? File(xFile.path) : null;
    } on PlatformException catch (e) {
      if (e.code == 'camera_access_denied') {
        throw const ForbiddenException('Cần quyền truy cập camera');
      }
      rethrow;
    }
  }

  // Chọn nhiều ảnh
  Future<List<File>> pickMultipleImages({
    int limit = 9,
    int? imageQuality = 85,
  }) async {
    final xFiles = await _picker.pickMultiImage(
      imageQuality: imageQuality,
      limit: limit,
    );
    return xFiles.map((f) => File(f.path)).toList();
  }

  // Chọn video
  Future<File?> pickVideo({Duration? maxDuration}) async {
    final xFile = await _picker.pickVideo(
      source: ImageSource.gallery,
      maxDuration: maxDuration,
    );
    return xFile != null ? File(xFile.path) : null;
  }

  // Hiển thị bottom sheet chọn nguồn
  Future<File?> showPickerSheet(BuildContext context) async {
    File? result;

    await showModalBottomSheet(
      context: context,
      builder: (_) => SafeArea(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            ListTile(
              leading: const Icon(Icons.camera_alt_outlined),
              title: const Text('Chụp ảnh'),
              onTap: () async {
                Navigator.pop(context);
                result = await takePhoto();
              },
            ),
            ListTile(
              leading: const Icon(Icons.photo_library_outlined),
              title: const Text('Chọn từ thư viện'),
              onTap: () async {
                Navigator.pop(context);
                result = await pickImage();
              },
            ),
          ],
        ),
      ),
    );

    return result;
  }
}
```

### 14.3.1. Permission Handling

```dart
// lib/core/services/permission_service.dart

class PermissionService {
  // Kiểm tra và xin quyền
  static Future<bool> requestCameraPermission(BuildContext context) async {
    var status = await Permission.camera.status;

    if (status.isGranted) return true;

    if (status.isPermanentlyDenied) {
      // Hướng user vào Settings
      await _showPermissionDeniedDialog(
        context,
        title: 'Quyền Camera',
        message: 'Vui lòng cấp quyền camera trong Cài đặt để tiếp tục.',
      );
      return false;
    }

    status = await Permission.camera.request();
    return status.isGranted;
  }

  static Future<bool> requestPhotoPermission(BuildContext context) async {
    // iOS 14+ dùng limitedPhoto, Android 13+ dùng READ_MEDIA_IMAGES
    var status = await Permission.photos.status;
    if (status.isGranted || status.isLimited) return true;

    if (status.isPermanentlyDenied) {
      await _showPermissionDeniedDialog(
        context,
        title: 'Quyền Thư Viện Ảnh',
        message: 'Vui lòng cấp quyền thư viện ảnh trong Cài đặt.',
      );
      return false;
    }

    status = await Permission.photos.request();
    return status.isGranted || status.isLimited;
  }

  static Future<void> _showPermissionDeniedDialog(
    BuildContext context, {
    required String title,
    required String message,
  }) async {
    final openSettings = await showDialog<bool>(
      context: context,
      builder: (_) => AlertDialog(
        title: Text(title),
        content: Text(message),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: const Text('Hủy'),
          ),
          FilledButton(
            onPressed: () => Navigator.pop(context, true),
            child: const Text('Mở Cài đặt'),
          ),
        ],
      ),
    );

    if (openSettings == true) await openAppSettings();
  }
}
```

---

## 14.4. Maps và Location

```yaml
# pubspec.yaml
dependencies:
  google_maps_flutter: ^2.6.0
  geolocator: ^11.0.0
  geocoding: ^3.0.0
  flutter_polyline_points: ^2.0.0
```

```dart
// lib/core/services/location_service.dart

class LocationService {
  // Lấy vị trí hiện tại
  static Future<Position?> getCurrentPosition(BuildContext context) async {
    // Kiểm tra location service có bật không
    final serviceEnabled = await Geolocator.isLocationServiceEnabled();
    if (!serviceEnabled) {
      if (context.mounted) {
        await _showLocationServiceDialog(context);
      }
      return null;
    }

    // Kiểm tra permission
    var permission = await Geolocator.checkPermission();
    if (permission == LocationPermission.denied) {
      permission = await Geolocator.requestPermission();
      if (permission == LocationPermission.denied) return null;
    }

    if (permission == LocationPermission.deniedForever) {
      if (context.mounted) await _showPermissionDeniedDialog(context);
      return null;
    }

    return Geolocator.getCurrentPosition(
      locationSettings: const LocationSettings(
        accuracy: LocationAccuracy.high,
        distanceFilter: 10, // Cập nhật khi di chuyển ≥10m
      ),
    );
  }

  // Stream vị trí liên tục
  static Stream<Position> getPositionStream() {
    return Geolocator.getPositionStream(
      locationSettings: const LocationSettings(
        accuracy: LocationAccuracy.high,
        distanceFilter: 10,
      ),
    );
  }

  // Tính khoảng cách giữa hai điểm (km)
  static double distanceBetween(
    double lat1, double lng1,
    double lat2, double lng2,
  ) {
    return Geolocator.distanceBetween(lat1, lng1, lat2, lng2) / 1000;
  }

  // Geocoding: tọa độ → địa chỉ
  static Future<String?> coordinatesToAddress(double lat, double lng) async {
    try {
      final placemarks = await placemarkFromCoordinates(lat, lng);
      if (placemarks.isEmpty) return null;

      final place = placemarks.first;
      return [
        place.street,
        place.subAdministrativeArea,
        place.administrativeArea,
      ].where((s) => s != null && s.isNotEmpty).join(', ');
    } catch (_) {
      return null;
    }
  }

  // Geocoding: địa chỉ → tọa độ
  static Future<LatLng?> addressToCoordinates(String address) async {
    try {
      final locations = await locationFromAddress(address);
      if (locations.isEmpty) return null;
      return LatLng(locations.first.latitude, locations.first.longitude);
    } catch (_) {
      return null;
    }
  }

  static Future<void> _showLocationServiceDialog(BuildContext context) async {
    await showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: const Text('Dịch vụ vị trí'),
        content: const Text('Vui lòng bật dịch vụ vị trí để tiếp tục.'),
        actions: [
          FilledButton(
            onPressed: () async {
              Navigator.pop(context);
              await Geolocator.openLocationSettings();
            },
            child: const Text('Mở Cài đặt'),
          ),
        ],
      ),
    );
  }

  static Future<void> _showPermissionDeniedDialog(BuildContext context) async {
    await showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: const Text('Quyền vị trí'),
        content: const Text('Vui lòng cấp quyền vị trí trong Cài đặt.'),
        actions: [
          FilledButton(
            onPressed: () async {
              Navigator.pop(context);
              await Geolocator.openAppSettings();
            },
            child: const Text('Mở Cài đặt'),
          ),
        ],
      ),
    );
  }
}

// Google Maps Widget
class StoreMapView extends ConsumerStatefulWidget {
  const StoreMapView({super.key, required this.storeLocation});
  final LatLng storeLocation;

  @override
  ConsumerState<StoreMapView> createState() => _StoreMapViewState();
}

class _StoreMapViewState extends ConsumerState<StoreMapView> {
  GoogleMapController? _mapController;
  Position? _userPosition;

  @override
  void initState() {
    super.initState();
    _loadUserLocation();
  }

  Future<void> _loadUserLocation() async {
    final pos = await LocationService.getCurrentPosition(context);
    if (mounted && pos != null) {
      setState(() => _userPosition = pos);
    }
  }

  @override
  Widget build(BuildContext context) {
    final initialTarget = _userPosition != null
        ? LatLng(_userPosition!.latitude, _userPosition!.longitude)
        : widget.storeLocation;

    return GoogleMap(
      initialCameraPosition: CameraPosition(
        target: initialTarget,
        zoom: 14,
      ),
      onMapCreated: (controller) => _mapController = controller,
      myLocationEnabled: true,
      myLocationButtonEnabled: true,
      markers: {
        Marker(
          markerId: const MarkerId('store'),
          position: widget.storeLocation,
          infoWindow: const InfoWindow(title: 'Cửa hàng'),
          icon: BitmapDescriptor.defaultMarkerWithHue(
            BitmapDescriptor.hueViolet,
          ),
        ),
      },
    );
  }

  @override
  void dispose() {
    _mapController?.dispose();
    super.dispose();
  }
}
```

---

## 14.5. Performance & Crashlytics

### 14.5.1. Firebase Crashlytics

```dart
// lib/core/monitoring/crashlytics_service.dart

class CrashlyticsService {
  static final _crashlytics = FirebaseCrashlytics.instance;

  static Future<void> setup() async {
    // Bật collection trong production
    await _crashlytics.setCrashlyticsCollectionEnabled(AppConfig.isProd);

    // Bắt Flutter framework errors
    FlutterError.onError = _crashlytics.recordFlutterFatalError;

    // Bắt Dart errors không được xử lý
    PlatformDispatcher.instance.onError = (error, stack) {
      _crashlytics.recordError(error, stack, fatal: true);
      return true;
    };
  }

  // Gắn thông tin user vào crash report
  static Future<void> setUser(AppUser user) async {
    await _crashlytics.setUserIdentifier(user.id);
    await _crashlytics.setCustomKey('user_email', user.email);
    await _crashlytics.setCustomKey('user_role', user.role);
  }

  // Log lỗi không fatal (đã được xử lý)
  static Future<void> logError(
    Object error,
    StackTrace? stackTrace, {
    String? reason,
    bool fatal = false,
  }) async {
    await _crashlytics.recordError(
      error,
      stackTrace,
      reason: reason,
      fatal: fatal,
    );
  }

  // Log event để debug
  static Future<void> log(String message) async {
    await _crashlytics.log(message);
  }
}
```

### 14.5.2. Firebase Remote Config

```dart
// lib/core/config/remote_config_service.dart
// Remote Config: thay đổi cấu hình app mà không cần update store

class RemoteConfigService {
  RemoteConfigService._();
  static final instance = RemoteConfigService._();

  final _config = FirebaseRemoteConfig.instance;

  // Default values — dùng khi chưa fetch được từ server
  static const _defaults = {
    'min_app_version': '1.0.0',
    'show_maintenance_banner': false,
    'maintenance_message': '',
    'max_cart_items': 20,
    'enable_new_checkout': false,
    'free_shipping_threshold': 300000.0,
  };

  Future<void> initialize() async {
    await _config.setConfigSettings(RemoteConfigSettings(
      fetchTimeout: const Duration(seconds: 10),
      // Cache duration — bao lâu thì fetch lại
      minimumFetchInterval: AppConfig.isProd
          ? const Duration(hours: 1)
          : Duration.zero, // Dev: fetch ngay lập tức
    ));

    await _config.setDefaults(_defaults);

    // Fetch và activate
    await _config.fetchAndActivate();
  }

  String get minAppVersion => _config.getString('min_app_version');
  bool get showMaintenanceBanner => _config.getBool('show_maintenance_banner');
  String get maintenanceMessage => _config.getString('maintenance_message');
  int get maxCartItems => _config.getInt('max_cart_items');
  bool get enableNewCheckout => _config.getBool('enable_new_checkout');
  double get freeShippingThreshold => _config.getDouble('free_shipping_threshold');
}

// Provider
@riverpod
RemoteConfigService remoteConfig(RemoteConfigRef ref) {
  return RemoteConfigService.instance;
}

// Sử dụng trong widget
class ShippingThresholdBanner extends ConsumerWidget {
  const ShippingThresholdBanner({super.key, required this.cartTotal});
  final double cartTotal;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final threshold = ref.watch(
      remoteConfigProvider.select((c) => c.freeShippingThreshold),
    );

    if (cartTotal >= threshold) {
      return const _FreeShippingBadge();
    }

    final remaining = threshold - cartTotal;
    return _ShippingProgressBanner(remaining: remaining, threshold: threshold);
  }
}
```

---

## 14.6. Deep Link & Dynamic Link

```dart
// lib/core/router/deep_link_handler.dart

class DeepLinkHandler {
  DeepLinkHandler._();
  static final instance = DeepLinkHandler._();

  Future<void> initialize() async {
    // Xử lý link mở app từ terminated state
    final initialLink = await getInitialLink();
    if (initialLink != null) {
      _handleLink(initialLink);
    }

    // Lắng nghe link khi app đang chạy
    linkStream.listen((link) {
      if (link != null) _handleLink(link);
    });
  }

  void _handleLink(String link) {
    final uri = Uri.parse(link);

    // Scheme: myapp://products/abc123
    // HTTPS: https://myapp.com/products/abc123
    if (uri.host == 'products' || uri.pathSegments.contains('products')) {
      final productId = uri.pathSegments.last;
      AppRouter.instance.push('/products/$productId');
    } else if (uri.host == 'orders' || uri.pathSegments.contains('orders')) {
      final orderId = uri.pathSegments.last;
      AppRouter.instance.push('/orders/$orderId');
    } else if (uri.host == 'promotions') {
      final code = uri.queryParameters['code'];
      if (code != null) {
        AppRouter.instance.push('/checkout?promo=$code');
      }
    }
  }
}
```

---

## 14.7. Biometric Authentication

```dart
// lib/features/auth/services/biometric_service.dart

class BiometricService {
  BiometricService._();
  static final instance = BiometricService._();

  final _localAuth = LocalAuthentication();

  // Kiểm tra thiết bị có hỗ trợ biometric không
  Future<bool> get isAvailable async {
    return _localAuth.canCheckBiometrics;
  }

  // Lấy danh sách biometric được hỗ trợ
  Future<List<BiometricType>> get availableBiometrics async {
    return _localAuth.getAvailableBiometrics();
  }

  // Xác thực
  Future<bool> authenticate({
    String reason = 'Xác nhận danh tính của bạn',
  }) async {
    try {
      return await _localAuth.authenticate(
        localizedReason: reason,
        options: const AuthenticationOptions(
          stickyAuth: true,         // Giữ auth dialog khi rời app
          biometricOnly: false,     // Cho phép fallback PIN/pattern
          sensitiveTransaction: false,
        ),
      );
    } on PlatformException catch (e) {
      debugPrint('Biometric error: ${e.code} - ${e.message}');
      return false;
    }
  }
}

// Sử dụng: Quick login với Face ID / Fingerprint
class BiometricLoginButton extends ConsumerWidget {
  const BiometricLoginButton({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return FutureBuilder<bool>(
      future: BiometricService.instance.isAvailable,
      builder: (context, snapshot) {
        if (snapshot.data != true) return const SizedBox.shrink();

        return OutlinedButton.icon(
          onPressed: () async {
            final authenticated = await BiometricService.instance.authenticate(
              reason: 'Đăng nhập bằng Face ID / Vân tay',
            );

            if (authenticated && context.mounted) {
              // Lấy credentials đã lưu và đăng nhập
              await ref.read(authProvider.notifier).loginWithBiometric();
            }
          },
          icon: const Icon(Icons.fingerprint),
          label: const Text('Đăng nhập bằng sinh trắc học'),
        );
      },
    );
  }
}
```

---

## 14.8. Bài Tập: Profile Screen Hoàn Chỉnh

Tích hợp tất cả tính năng native vào ProfileScreen:

```dart
class ProfileScreen extends ConsumerWidget {
  const ProfileScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final user = ref.watch(currentUserProvider);
    if (user == null) return const LoginPrompt();

    return Scaffold(
      appBar: AppBar(
        title: const Text('Tài khoản'),
        actions: [
          IconButton(
            icon: const Icon(Icons.settings_outlined),
            onPressed: () => context.push('/settings'),
          ),
        ],
      ),
      body: ListView(
        children: [
          // Avatar với camera picker
          _AvatarSection(user: user),
          const Divider(),

          // Thông tin tài khoản
          ListTile(
            leading: const Icon(Icons.person_outlined),
            title: const Text('Họ và tên'),
            subtitle: Text(user.name),
            trailing: const Icon(Icons.chevron_right),
            onTap: () => context.push('/profile/edit-name'),
          ),
          ListTile(
            leading: const Icon(Icons.email_outlined),
            title: const Text('Email'),
            subtitle: Text(user.email),
          ),
          ListTile(
            leading: const Icon(Icons.phone_outlined),
            title: const Text('Số điện thoại'),
            subtitle: Text(user.phone ?? 'Chưa cập nhật'),
            trailing: const Icon(Icons.chevron_right),
            onTap: () => context.push('/profile/edit-phone'),
          ),
          const Divider(),

          // Đơn hàng
          ListTile(
            leading: const Icon(Icons.receipt_long_outlined),
            title: const Text('Đơn hàng của tôi'),
            trailing: const Icon(Icons.chevron_right),
            onTap: () => context.push('/orders'),
          ),

          // Địa chỉ
          ListTile(
            leading: const Icon(Icons.location_on_outlined),
            title: const Text('Địa chỉ giao hàng'),
            trailing: const Icon(Icons.chevron_right),
            onTap: () => context.push('/profile/addresses'),
          ),
          const Divider(),

          // Bảo mật
          ListTile(
            leading: const Icon(Icons.lock_outlined),
            title: const Text('Đổi mật khẩu'),
            trailing: const Icon(Icons.chevron_right),
            onTap: () => context.push('/profile/change-password'),
          ),
          const BiometricToggle(),
          const Divider(),

          // Đăng xuất
          ListTile(
            leading: Icon(Icons.logout, color: context.colorScheme.error),
            title: Text('Đăng xuất',
                style: TextStyle(color: context.colorScheme.error)),
            onTap: () async {
              final confirmed = await showDeleteConfirmDialog(
                context,
                itemName: 'phiên đăng nhập',
              );
              if (confirmed == true && context.mounted) {
                await ref.read(authProvider.notifier).logout();
              }
            },
          ),
        ],
      ),
    );
  }
}

class _AvatarSection extends ConsumerWidget {
  const _AvatarSection({required this.user});
  final AppUser user;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Padding(
      padding: const EdgeInsets.all(24),
      child: Column(
        children: [
          GestureDetector(
            onTap: () async {
              final file = await MediaPickerService.instance
                  .showPickerSheet(context);
              if (file != null && context.mounted) {
                await ref.read(profileProvider.notifier)
                    .updateAvatar(file);
              }
            },
            child: Stack(
              children: [
                CircleAvatar(
                  radius: 48,
                  backgroundImage: user.avatarUrl != null
                      ? CachedNetworkImageProvider(user.avatarUrl!)
                      : null,
                  child: user.avatarUrl == null
                      ? Text(user.name[0].toUpperCase(),
                          style: context.textTheme.headlineMedium)
                      : null,
                ),
                Positioned(
                  bottom: 0,
                  right: 0,
                  child: CircleAvatar(
                    radius: 16,
                    backgroundColor: context.colorScheme.primary,
                    child: Icon(Icons.camera_alt,
                        size: 16,
                        color: context.colorScheme.onPrimary),
                  ),
                ),
              ],
            ),
          ),
          const SizedBox(height: 12),
          Text(user.name, style: context.textTheme.titleLarge),
          Text(user.email,
              style: context.textTheme.bodyMedium?.copyWith(
                color: context.colorScheme.onSurfaceVariant,
              )),
        ],
      ),
    );
  }
}
```

---

## Tóm Tắt Chương 14

| Tính năng | Package | Điểm chú ý |
|---|---|---|
| Firebase Auth | `firebase_auth` | Map FirebaseAuthException → AppException |
| Firestore | `cloud_firestore` | Dùng `FieldValue.serverTimestamp()`, không `DateTime.now()` |
| FCM | `firebase_messaging` | Xử lý 3 trạng thái: foreground, background, terminated |
| Image Picker | `image_picker` | Kiểm tra permission trước, compress ảnh trước upload |
| Location | `geolocator` | Kiểm tra service enabled + permission, handle denied forever |
| Google Maps | `google_maps_flutter` | Dispose controller, memCacheWidth cho markers |
| Crashlytics | `firebase_crashlytics` | Setup trước `runApp`, setUser sau login |
| Remote Config | `firebase_remote_config` | Đặt defaults, fetch interval dài hơn trong production |
| Biometric | `local_auth` | Fallback PIN, `stickyAuth: true` |
| Deep Link | `app_links` | Handle cả initial link và stream link |

> **Lưu ý quan trọng:** Mọi tính năng truy cập hardware (camera, location, biometric) đều cần khai báo permission trong `AndroidManifest.xml` (Android) và `Info.plist` (iOS) với description rõ ràng tại sao app cần quyền đó. App Store và Google Play đều review kỹ mô tả permission — thiếu hoặc mô tả không rõ sẽ bị reject.