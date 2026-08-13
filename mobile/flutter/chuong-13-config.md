# Chương 13: Flavors, ENV & Build Config

---

## 13.1. Tại Sao Cần Flavors?

### 13.1.1. Vấn Đề Thực Tế

Mọi app production đều cần ít nhất hai môi trường chạy song song:

- **Development:** API test, log verbose, mock data, không gửi analytics thật
- **Production:** API thật, không log nhạy cảm, analytics đầy đủ

Nếu không có flavor, developer phải thủ công sửa `baseUrl` mỗi khi build — rất dễ deploy bản dev lên store hoặc ngược lại dev đang gọi production API.

**Flutter Flavor** giải quyết điều này: một lệnh build duy nhất, cấu hình đúng môi trường tự động.

### 13.1.2. Ba Flavor Thông Dụng

```
dev     → API dev, mock có thể bật, log đầy đủ, icon dev
staging → API staging, giống production nhưng data test
prod    → API production, no log, analytics, icon chính thức
```

---

## 13.2. Cấu Hình Flavor

### 13.2.1. Approach Đơn Giản: --dart-define

Cách đơn giản nhất, không cần sửa native code. Phù hợp cho hầu hết dự án:

```bash
# Build dev
flutter run --dart-define=APP_ENV=dev \
            --dart-define=API_BASE_URL=https://dev-api.example.com \
            --dart-define=USE_MOCK_DATA=false

# Build staging
flutter run --dart-define=APP_ENV=staging \
            --dart-define=API_BASE_URL=https://staging-api.example.com

# Build production
flutter build apk --dart-define=APP_ENV=prod \
                  --dart-define=API_BASE_URL=https://api.example.com \
                  --dart-define=ENABLE_ANALYTICS=true
```

```dart
// lib/core/config/app_config.dart — đọc dart-define
abstract class AppConfig {
  static const env = String.fromEnvironment('APP_ENV', defaultValue: 'dev');
  static const apiBaseUrl = String.fromEnvironment(
    'API_BASE_URL',
    defaultValue: 'https://dev-api.example.com',
  );
  static const useMockData = bool.fromEnvironment('USE_MOCK_DATA');
  static const enableAnalytics = bool.fromEnvironment('ENABLE_ANALYTICS');

  static bool get isDev => env == 'dev';
  static bool get isStaging => env == 'staging';
  static bool get isProd => env == 'prod';
}
```

### 13.2.2. dart-define-from-file — Quản Lý Nhiều Biến

Khi có nhiều biến môi trường, viết dòng lệnh dài rất bất tiện. Dùng file JSON:

```json
// config/dev.json
{
  "APP_ENV": "dev",
  "APP_NAME": "MyApp Dev",
  "API_BASE_URL": "https://dev-api.example.com",
  "USE_MOCK_DATA": "false",
  "ENABLE_ANALYTICS": "false",
  "SENTRY_DSN": "",
  "FIREBASE_PROJECT_ID": "myapp-dev"
}
```

```json
// config/prod.json
{
  "APP_ENV": "prod",
  "APP_NAME": "MyApp",
  "API_BASE_URL": "https://api.example.com",
  "USE_MOCK_DATA": "false",
  "ENABLE_ANALYTICS": "true",
  "SENTRY_DSN": "https://xxx@sentry.io/xxx",
  "FIREBASE_PROJECT_ID": "myapp-prod"
}
```

```bash
# Dùng file thay vì nhiều --dart-define
flutter run --dart-define-from-file=config/dev.json
flutter build apk --dart-define-from-file=config/prod.json
flutter build ipa --dart-define-from-file=config/prod.json
```

```dart
// AppConfig mở rộng
abstract class AppConfig {
  static const env = String.fromEnvironment('APP_ENV', defaultValue: 'dev');
  static const appName = String.fromEnvironment('APP_NAME', defaultValue: 'MyApp Dev');
  static const apiBaseUrl = String.fromEnvironment('API_BASE_URL',
      defaultValue: 'https://dev-api.example.com');
  static const useMockData = bool.fromEnvironment('USE_MOCK_DATA');
  static const enableAnalytics = bool.fromEnvironment('ENABLE_ANALYTICS');
  static const sentryDsn = String.fromEnvironment('SENTRY_DSN');
  static const firebaseProjectId = String.fromEnvironment('FIREBASE_PROJECT_ID',
      defaultValue: 'myapp-dev');

  static bool get isDev => env == 'dev';
  static bool get isStaging => env == 'staging';
  static bool get isProd => env == 'prod';
}
```

> **Bảo mật:** Thêm `config/prod.json` và `config/staging.json` vào `.gitignore`. Chỉ commit `config/dev.json`. Dùng CI/CD secrets để inject prod config khi build.

---

## 13.3. Bảo Vệ API Key — envied Package

`--dart-define` vẫn có thể bị extract từ compiled binary bằng reverse engineering. Với API key nhạy cảm (Maps, Payment), dùng `envied` để obfuscate.

```yaml
# pubspec.yaml
dependencies:
  envied: ^0.5.0

dev_dependencies:
  envied_generator: ^0.5.0
  build_runner: ^2.4.0
```

```env
# .env — KHÔNG commit file này
GOOGLE_MAPS_KEY=AIzaSyXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
STRIPE_PUBLISHABLE_KEY=pk_live_XXXXXXXXXXXXXXXXXXXX
```

```dart
// lib/core/config/env.dart
import 'package:envied/envied.dart';
part 'env.g.dart';

@Envied(path: '.env', obfuscate: true) // obfuscate: mã hóa trong compiled code
abstract class Env {
  @EnviedField(varName: 'GOOGLE_MAPS_KEY', obfuscate: true)
  static final String googleMapsKey = _Env.googleMapsKey;

  @EnviedField(varName: 'STRIPE_PUBLISHABLE_KEY', obfuscate: true)
  static final String stripeKey = _Env.stripeKey;
}

// Generate: dart run build_runner build
```

```gitignore
# .gitignore
.env
.env.*
config/prod.json
config/staging.json
lib/core/config/env.g.dart  # Generated file chứa key đã obfuscate
```

---

## 13.4. Native Flavor Configuration

Khi cần thay đổi **App ID**, **App Name**, **App Icon** theo môi trường — phải cấu hình native.

### 13.4.1. Android Flavor

```groovy
// android/app/build.gradle

android {
    flavorDimensions += "environment"

    productFlavors {
        dev {
            dimension "environment"
            applicationIdSuffix ".dev"          // com.example.app.dev
            versionNameSuffix "-dev"
            resValue "string", "app_name", "MyApp Dev"
        }
        staging {
            dimension "environment"
            applicationIdSuffix ".staging"      // com.example.app.staging
            versionNameSuffix "-staging"
            resValue "string", "app_name", "MyApp Staging"
        }
        prod {
            dimension "environment"
            // Không suffix — App ID gốc
            resValue "string", "app_name", "MyApp"
        }
    }
}
```

```
android/app/src/
├── main/
│   └── res/
│       └── mipmap-*/          # Icon mặc định (prod)
├── dev/
│   └── res/
│       └── mipmap-*/          # Icon dev (có badge "DEV")
└── staging/
    └── res/
        └── mipmap-*/          # Icon staging
```

```bash
# Build theo flavor
flutter run --flavor dev -t lib/main_dev.dart
flutter build apk --flavor prod -t lib/main_prod.dart
flutter build appbundle --flavor prod -t lib/main_prod.dart
```

### 13.4.2. iOS Flavor (Scheme)

```bash
# Tạo scheme trong Xcode:
# Product → Scheme → New Scheme → Tên: Runner-dev, Runner-staging, Runner-prod

# Hoặc dùng flutter_flavorizr để tự động hóa
dart pub global activate flutter_flavorizr
```

```yaml
# flutter_flavorizr config trong pubspec.yaml
flavorizr:
  flavors:
    dev:
      app:
        name: "MyApp Dev"
      android:
        applicationId: "com.example.app.dev"
        icon: "assets/icons/icon_dev.png"
      ios:
        bundleId: "com.example.app.dev"
        icon: "assets/icons/icon_dev.png"

    prod:
      app:
        name: "MyApp"
      android:
        applicationId: "com.example.app"
        icon: "assets/icons/icon_prod.png"
      ios:
        bundleId: "com.example.app"
        icon: "assets/icons/icon_prod.png"
```

```bash
# Chạy flavorizr để tự động config native
dart run flutter_flavorizr
```

---

## 13.5. Multiple Entry Points

Mỗi flavor có file `main_*.dart` riêng — đây là nơi inject config vào app:

```dart
// lib/main_dev.dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Dev-only: verbose logging
  if (kDebugMode) {
    debugPrint('🚀 Running in DEV mode');
    debugPrint('   API: ${AppConfig.apiBaseUrl}');
  }

  await _initializeApp();
  runApp(const ProviderScope(child: MyApp()));
}

Future<void> _initializeApp() async {
  await Firebase.initializeApp(
    options: DevFirebaseOptions.currentPlatform, // Dev Firebase project
  );
  await HiveInitializer.init();
  ErrorHandler.setupFlutterErrorHandling();
}
```

```dart
// lib/main_prod.dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Production: setup crash reporting trước tất cả
  await Firebase.initializeApp(
    options: ProdFirebaseOptions.currentPlatform,
  );

  // Crash reporting phải setup trước runApp
  await FirebaseCrashlytics.instance
      .setCrashlyticsCollectionEnabled(true);

  FlutterError.onError =
      FirebaseCrashlytics.instance.recordFlutterFatalError;

  PlatformDispatcher.instance.onError = (error, stack) {
    FirebaseCrashlytics.instance.recordError(error, stack, fatal: true);
    return true;
  };

  await HiveInitializer.init();
  runApp(const ProviderScope(child: MyApp()));
}
```

---

## 13.6. VS Code Launch Configuration

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Dev",
      "request": "launch",
      "type": "dart",
      "program": "lib/main_dev.dart",
      "args": [
        "--flavor", "dev",
        "--dart-define-from-file=config/dev.json"
      ]
    },
    {
      "name": "Staging",
      "request": "launch",
      "type": "dart",
      "program": "lib/main_staging.dart",
      "args": [
        "--flavor", "staging",
        "--dart-define-from-file=config/staging.json"
      ]
    },
    {
      "name": "Prod (Release)",
      "request": "launch",
      "type": "dart",
      "program": "lib/main_prod.dart",
      "flutterMode": "release",
      "args": [
        "--flavor", "prod",
        "--dart-define-from-file=config/prod.json"
      ]
    }
  ]
}
```

---

## 13.7. Makefile — Build Commands Tập Trung

```makefile
# Makefile — shortcuts cho lệnh build dài

.PHONY: dev staging prod build-apk build-ipa clean

# Run
dev:
	flutter run --flavor dev -t lib/main_dev.dart \
		--dart-define-from-file=config/dev.json

staging:
	flutter run --flavor staging -t lib/main_staging.dart \
		--dart-define-from-file=config/staging.json

# Build
build-apk-prod:
	flutter build apk --flavor prod -t lib/main_prod.dart \
		--dart-define-from-file=config/prod.json \
		--obfuscate \
		--split-debug-info=build/debug-info

build-aab-prod:
	flutter build appbundle --flavor prod -t lib/main_prod.dart \
		--dart-define-from-file=config/prod.json \
		--obfuscate \
		--split-debug-info=build/debug-info

build-ipa-prod:
	flutter build ipa --flavor prod -t lib/main_prod.dart \
		--dart-define-from-file=config/prod.json \
		--obfuscate \
		--split-debug-info=build/debug-info

# Code generation
gen:
	dart run build_runner build --delete-conflicting-outputs

gen-watch:
	dart run build_runner watch --delete-conflicting-outputs

# Test
test:
	flutter test --coverage

test-unit:
	flutter test test/unit/

test-widget:
	flutter test test/widget/

# Cleanup
clean:
	flutter clean
	rm -rf .dart_tool/build
```

```bash
# Sử dụng
make dev           # Chạy dev
make build-aab-prod  # Build production AAB
make test          # Chạy test
```

---

## 13.8. CI/CD Với GitHub Actions

```yaml
# .github/workflows/build.yml

name: Build & Deploy

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  # ─── Test ─────────────────────────────────
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'
          channel: 'stable'
          cache: true

      - name: Get dependencies
        run: flutter pub get

      - name: Run code generation
        run: dart run build_runner build --delete-conflicting-outputs

      - name: Analyze
        run: flutter analyze

      - name: Test
        run: flutter test --coverage

  # ─── Build Android ─────────────────────────
  build-android:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: 'zulu'
          java-version: '17'

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'

      - name: Get dependencies
        run: flutter pub get

      - name: Run code generation
        run: dart run build_runner build --delete-conflicting-outputs

      # Decode keystore từ secret
      - name: Decode Keystore
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 --decode > android/app/keystore.jks

      - name: Create key.properties
        run: |
          cat > android/key.properties << EOF
          storePassword=${{ secrets.KEYSTORE_PASSWORD }}
          keyPassword=${{ secrets.KEY_PASSWORD }}
          keyAlias=${{ secrets.KEY_ALIAS }}
          storeFile=keystore.jks
          EOF

      # Inject prod config từ secret
      - name: Create prod config
        run: echo '${{ secrets.PROD_CONFIG_JSON }}' > config/prod.json

      - name: Build AAB
        run: |
          flutter build appbundle \
            --flavor prod \
            -t lib/main_prod.dart \
            --dart-define-from-file=config/prod.json \
            --obfuscate \
            --split-debug-info=build/debug-info

      - name: Upload to Play Store
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_STORE_JSON }}
          packageName: com.example.app
          releaseFiles: build/app/outputs/bundle/prodRelease/app-prod-release.aab
          track: internal  # internal → alpha → beta → production

  # ─── Build iOS ─────────────────────────────
  build-ios:
    needs: test
    runs-on: macos-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'

      - name: Install CocoaPods
        run: cd ios && pod install

      - name: Create prod config
        run: echo '${{ secrets.PROD_CONFIG_JSON }}' > config/prod.json

      - name: Build IPA
        run: |
          flutter build ipa \
            --flavor prod \
            -t lib/main_prod.dart \
            --dart-define-from-file=config/prod.json \
            --export-options-plist=ios/ExportOptions.plist

      - name: Upload to TestFlight
        uses: apple-actions/upload-testflight-build@v1
        with:
          app-path: build/ios/ipa/MyApp.ipa
          issuer-id: ${{ secrets.APPSTORE_ISSUER_ID }}
          api-key-id: ${{ secrets.APPSTORE_API_KEY_ID }}
          api-private-key: ${{ secrets.APPSTORE_API_PRIVATE_KEY }}
```

---

## 13.9. App Icon Theo Flavor

```yaml
# pubspec.yaml — flutter_launcher_icons
dev_dependencies:
  flutter_launcher_icons: ^0.13.0

flutter_launcher_icons:
  # Sẽ override bằng flavor-specific config
  image_path: "assets/icons/icon_dev.png"
  android: true
  ios: true
```

```yaml
# flutter_launcher_icons-dev.yaml
flutter_launcher_icons:
  image_path: "assets/icons/icon_dev.png"
  android: "launcher_icon"
  ios: true
  # Thêm badge "DEV" trên icon
  background_color: "#FF5722"
  adaptive_icon_foreground: "assets/icons/icon_dev_foreground.png"
  adaptive_icon_background: "#FF5722"
```

```yaml
# flutter_launcher_icons-prod.yaml
flutter_launcher_icons:
  image_path: "assets/icons/icon_prod.png"
  android: "launcher_icon"
  ios: true
  adaptive_icon_foreground: "assets/icons/icon_prod_foreground.png"
  adaptive_icon_background: "#6750A4"
```

```bash
# Generate icon theo flavor
dart run flutter_launcher_icons -f flutter_launcher_icons-dev.yaml
dart run flutter_launcher_icons -f flutter_launcher_icons-prod.yaml
```

---

## 13.10. Debug Banner và Dev Tools

```dart
// lib/app.dart
class MyApp extends ConsumerWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final themeMode = ref.watch(appThemeModeProvider).valueOrNull
        ?? ThemeMode.system;

    return MaterialApp.router(
      title: AppConfig.appName,
      theme: AppTheme.light,
      darkTheme: AppTheme.dark,
      themeMode: themeMode,
      routerConfig: appRouter,

      // Tắt debug banner trong production
      debugShowCheckedModeBanner: AppConfig.isDev,

      // Performance overlay chỉ trong dev
      showPerformanceOverlay: false,

      builder: (context, child) {
        return Stack(
          children: [
            child!,
            // Env badge chỉ hiển thị trong dev/staging
            if (!AppConfig.isProd)
              const _EnvBadge(),
          ],
        );
      },
    );
  }
}

// Badge nhỏ góc màn hình cho biết đang chạy env nào
class _EnvBadge extends StatelessWidget {
  const _EnvBadge();

  @override
  Widget build(BuildContext context) {
    return Positioned(
      top: MediaQuery.paddingOf(context).top + 4,
      right: 8,
      child: IgnorePointer(
        child: Container(
          padding: const EdgeInsets.symmetric(horizontal: 6, vertical: 2),
          decoration: BoxDecoration(
            color: AppConfig.isDev ? Colors.orange : Colors.blue,
            borderRadius: BorderRadius.circular(4),
          ),
          child: Text(
            AppConfig.env.toUpperCase(),
            style: const TextStyle(
              color: Colors.white,
              fontSize: 10,
              fontWeight: FontWeight.bold,
            ),
          ),
        ),
      ),
    );
  }
}
```

---

## Tóm Tắt Chương 13

| Khái niệm | Điểm Cốt Lõi |
|---|---|
| `--dart-define` | Inject config lúc build — không hardcode trong code |
| `--dart-define-from-file` | Quản lý nhiều biến bằng file JSON — gọn hơn |
| `envied` | Obfuscate API key trong compiled binary |
| `.gitignore` | Không bao giờ commit prod config và `.env` |
| Native flavor | Thay App ID, App Name, App Icon — cần sửa Android/iOS native |
| Multiple main | `main_dev.dart`, `main_prod.dart` — inject config khác nhau |
| `Makefile` | Tập trung lệnh build dài — dễ dùng, tránh lỗi typo |
| GitHub Actions | Test → Build → Deploy tự động khi push/merge |
| Env badge | Badge nhỏ góc màn hình — biết ngay đang chạy env nào |

> **Nguyên tắc bảo mật quan trọng:** API key, secret, connection string **không bao giờ** được commit lên Git — kể cả trong private repo. Một lần lộ là tồn tại mãi trong Git history. Dùng CI/CD secrets và file config trong `.gitignore`. Rotate key ngay nếu phát hiện lộ.