# Chương 16: Deploy & CI/CD

---

## 16.1. Chuẩn Bị Trước Khi Deploy

### 16.1.1. Checklist Trước Khi Release

```
□ Version & Build Number
  □ Tăng versionName và versionCode (Android) / CFBundleShortVersionString (iOS)
  □ Đảm bảo version mới > version cũ trên store

□ App Config
  □ API URL đang trỏ đến production
  □ Analytics đã bật
  □ Debug logging đã tắt
  □ Mock data đã tắt
  □ Crashlytics collection đã bật

□ App Identity
  □ App name đúng (không có "Dev" hay "Test")
  □ App icon đúng (không có badge "DEV")
  □ Bundle ID / Package Name đúng

□ Build
  □ Build release mode (không phải debug)
  □ Code obfuscation bật
  □ Proguard rules đúng (Android)

□ Test
  □ Tất cả test pass
  □ Smoke test trên device thật
  □ Test flow chính: login → mua hàng → thanh toán
```

### 16.1.2. Version Management

```yaml
# pubspec.yaml
# version: major.minor.patch+buildNumber
# 1.2.3+45 → versionName=1.2.3, versionCode=45

version: 1.2.0+45
```

```bash
# Script tự động tăng build number
# scripts/bump_version.sh

#!/bin/bash
PUBSPEC="pubspec.yaml"
CURRENT=$(grep "^version:" $PUBSPEC | sed 's/version: //')
VERSION=$(echo $CURRENT | cut -d'+' -f1)
BUILD=$(echo $CURRENT | cut -d'+' -f2)
NEW_BUILD=$((BUILD + 1))
NEW_VERSION="${VERSION}+${NEW_BUILD}"
sed -i '' "s/^version: .*/version: $NEW_VERSION/" $PUBSPEC
echo "Version bumped: $CURRENT → $NEW_VERSION"
```

---

## 16.2. Deploy Android

### 16.2.1. Tạo Keystore

```bash
# Tạo keystore — chỉ làm MỘT LẦN, giữ file này cẩn thận!
# Mất keystore = không thể update app trên Play Store
keytool -genkey -v \
  -keystore ~/keys/my-app-release.jks \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -alias my-app-key
```

```properties
# android/key.properties — KHÔNG commit file này
storePassword=your_store_password
keyPassword=your_key_password
keyAlias=my-app-key
storeFile=/Users/username/keys/my-app-release.jks
```

```groovy
// android/app/build.gradle
def keystoreProperties = new Properties()
def keystorePropertiesFile = rootProject.file('key.properties')
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

android {
    signingConfigs {
        release {
            keyAlias keystoreProperties['keyAlias']
            keyPassword keystoreProperties['keyPassword']
            storeFile keystoreProperties['storeFile'] ? file(keystoreProperties['storeFile']) : null
            storePassword keystoreProperties['storePassword']
        }
    }

    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}
```

### 16.2.2. Build và Upload Android

```bash
# Build AAB (App Bundle) — Play Store yêu cầu AAB từ 2021
flutter build appbundle \
  --flavor prod \
  -t lib/main_prod.dart \
  --dart-define-from-file=config/prod.json \
  --obfuscate \
  --split-debug-info=build/debug-info/android

# File output: build/app/outputs/bundle/prodRelease/app-prod-release.aab

# Build APK (nếu cần distribute ngoài Play Store)
flutter build apk \
  --flavor prod \
  -t lib/main_prod.dart \
  --split-per-abi \          # Tạo APK riêng cho từng CPU architecture
  --dart-define-from-file=config/prod.json
```

### 16.2.3. Upload Lên Play Store

**Manual:** Vào [play.google.com/console](https://play.google.com/console) → Release → Production → Create new release → Upload AAB.

**Automatic với Fastlane:**

```ruby
# fastlane/Fastfile

platform :android do
  desc "Deploy to Play Store Internal Testing"
  lane :internal do
    gradle(
      task: "bundle",
      build_type: "Release",
      flavor: "prod",
      properties: {
        "android.injected.signing.store.file" => ENV["KEYSTORE_PATH"],
        "android.injected.signing.store.password" => ENV["KEYSTORE_PASSWORD"],
        "android.injected.signing.key.alias" => ENV["KEY_ALIAS"],
        "android.injected.signing.key.password" => ENV["KEY_PASSWORD"],
      }
    )

    upload_to_play_store(
      track: "internal",
      aab: "build/app/outputs/bundle/prodRelease/app-prod-release.aab",
      json_key: ENV["PLAY_STORE_JSON_KEY"],
      skip_upload_apk: true,
      skip_upload_metadata: true,
      skip_upload_screenshots: true,
    )
  end
end
```

---

## 16.3. Deploy iOS

### 16.3.1. Chuẩn Bị Certificate và Provisioning Profile

```
Yêu cầu:
├── Apple Developer Account ($99/năm)
├── App ID trong developer.apple.com
├── Distribution Certificate (p12 file)
└── Provisioning Profile (mobileprovision file)

Cách tạo:
1. Xcode → Preferences → Accounts → Manage Certificates
2. Tạo "Apple Distribution" certificate
3. Vào developer.apple.com → Profiles → New Profile
4. Chọn "App Store" → Chọn App ID → Chọn Certificate
5. Download .mobileprovision
```

### 16.3.2. Build iOS

```bash
# Cài CocoaPods dependencies trước
cd ios && pod install && cd ..

# Build IPA
flutter build ipa \
  --flavor prod \
  -t lib/main_prod.dart \
  --dart-define-from-file=config/prod.json \
  --obfuscate \
  --split-debug-info=build/debug-info/ios

# File output: build/ios/ipa/MyApp.ipa
```

```xml
<!-- ios/ExportOptions.plist — cấu hình export -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>app-store</string>
    <key>teamID</key>
    <string>YOUR_TEAM_ID</string>
    <key>uploadBitcode</key>
    <false/>
    <key>signingStyle</key>
    <string>manual</string>
    <key>provisioningProfiles</key>
    <dict>
        <key>com.example.myapp</key>
        <string>MyApp Production Distribution</string>
    </dict>
</dict>
</plist>
```

### 16.3.3. Upload Lên TestFlight

```bash
# Upload bằng Transporter (macOS app từ Mac App Store)
# Hoặc dùng xcrun altool
xcrun altool --upload-app \
  --type ios \
  --file build/ios/ipa/MyApp.ipa \
  --username "your@apple.id" \
  --password "@keychain:app-specific-password"

# Hoặc dùng Fastlane
```

```ruby
# fastlane/Fastfile

platform :ios do
  desc "Deploy to TestFlight"
  lane :beta do
    # Build
    build_app(
      scheme: "Runner",
      configuration: "Release-prod",
      export_options: {
        method: "app-store",
        provisioningProfiles: {
          "com.example.myapp" => "MyApp Production Distribution"
        }
      }
    )

    # Upload TestFlight
    upload_to_testflight(
      api_key: app_store_connect_api_key(
        key_id: ENV["ASC_KEY_ID"],
        issuer_id: ENV["ASC_ISSUER_ID"],
        key_content: ENV["ASC_PRIVATE_KEY"],
      ),
      skip_waiting_for_build_processing: true,
    )
  end
end
```

---

## 16.4. GitHub Actions CI/CD Hoàn Chỉnh

```yaml
# .github/workflows/release.yml

name: Release

on:
  push:
    tags:
      - 'v*'  # Trigger khi push tag v1.0.0, v1.2.3, ...

jobs:
  # ─── Test ─────────────────────────────────────────────
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'
          channel: 'stable'
          cache: true

      - run: flutter pub get
      - run: dart run build_runner build --delete-conflicting-outputs
      - run: flutter analyze --no-fatal-infos
      - run: flutter test --coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  # ─── Android Release ──────────────────────────────────
  android:
    name: Build & Deploy Android
    needs: test
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: 'zulu'
          java-version: '17'

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'
          cache: true

      - run: flutter pub get
      - run: dart run build_runner build --delete-conflicting-outputs

      # Decode keystore từ GitHub Secret (base64 encoded)
      - name: Setup signing
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 --decode > android/app/keystore.jks
          cat > android/key.properties << EOF
          storePassword=${{ secrets.KEYSTORE_PASSWORD }}
          keyPassword=${{ secrets.KEY_PASSWORD }}
          keyAlias=${{ secrets.KEY_ALIAS }}
          storeFile=keystore.jks
          EOF

      - name: Create prod config
        run: echo '${{ secrets.PROD_CONFIG_JSON }}' > config/prod.json

      - name: Bump build number
        run: |
          BUILD_NUMBER=$(date +%Y%m%d%H%M)
          sed -i "s/^version: .*/version: $(grep '^version:' pubspec.yaml | sed 's/version: //' | cut -d'+' -f1)+$BUILD_NUMBER/" pubspec.yaml

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
          packageName: com.example.myapp
          releaseFiles: build/app/outputs/bundle/prodRelease/app-prod-release.aab
          track: internal
          status: completed

      # Lưu debug info để symbolicate crash
      - name: Archive debug info
        uses: actions/upload-artifact@v4
        with:
          name: android-debug-info
          path: build/debug-info

  # ─── iOS Release ──────────────────────────────────────
  ios:
    name: Build & Deploy iOS
    needs: test
    runs-on: macos-latest

    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'
          cache: true

      - run: flutter pub get
      - run: dart run build_runner build --delete-conflicting-outputs

      # Setup certificate và provisioning profile
      - name: Setup signing
        uses: apple-actions/import-codesign-certs@v3
        with:
          p12-file-base64: ${{ secrets.IOS_DISTRIBUTION_CERT_BASE64 }}
          p12-password: ${{ secrets.IOS_DISTRIBUTION_CERT_PASSWORD }}

      - name: Install provisioning profile
        run: |
          mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
          echo "${{ secrets.IOS_PROVISIONING_PROFILE_BASE64 }}" | base64 --decode > \
            ~/Library/MobileDevice/Provisioning\ Profiles/distribution.mobileprovision

      - name: Create prod config
        run: echo '${{ secrets.PROD_CONFIG_JSON }}' > config/prod.json

      - name: Install pods
        run: cd ios && pod install

      - name: Build IPA
        run: |
          flutter build ipa \
            --flavor prod \
            -t lib/main_prod.dart \
            --dart-define-from-file=config/prod.json \
            --obfuscate \
            --split-debug-info=build/debug-info \
            --export-options-plist=ios/ExportOptions.plist

      - name: Upload to TestFlight
        uses: apple-actions/upload-testflight-build@v1
        with:
          app-path: build/ios/ipa/MyApp.ipa
          issuer-id: ${{ secrets.ASC_ISSUER_ID }}
          api-key-id: ${{ secrets.ASC_KEY_ID }}
          api-private-key: ${{ secrets.ASC_PRIVATE_KEY }}

  # ─── Notification ─────────────────────────────────────
  notify:
    name: Notify Team
    needs: [android, ios]
    runs-on: ubuntu-latest
    if: always()

    steps:
      - name: Send Slack notification
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "${{ needs.android.result == 'success' && needs.ios.result == 'success' && '✅' || '❌' }} Release ${{ github.ref_name }} - Android: ${{ needs.android.result }}, iOS: ${{ needs.ios.result }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 16.5. Gitflow và Release Strategy

```
Branch Strategy:
├── main          → Production code. Tag với version khi release.
├── develop       → Integration branch. CI test tự động.
├── feature/*     → Tính năng mới. PR vào develop.
├── hotfix/*      → Bug fix khẩn cấp. PR vào main VÀ develop.
└── release/*     → Chuẩn bị release. QA testing ở đây.

Release Flow:
1. Tạo release branch từ develop: git checkout -b release/1.2.0
2. Bump version, test, fix bugs nhỏ trên release branch
3. Merge vào main: PR với approval
4. Tag release: git tag v1.2.0
5. Push tag: git push origin v1.2.0
6. GitHub Actions tự động build và deploy
7. Merge release branch ngược lại develop
```

---

## Tóm Tắt Chương 16

| Bước | Android | iOS |
|---|---|---|
| Signing | Keystore (.jks) | Certificate (.p12) + Profile (.mobileprovision) |
| Build | `flutter build appbundle` | `flutter build ipa` |
| Upload | Play Console hoặc Fastlane | Transporter hoặc `altool` hoặc Fastlane |
| Testing channel | Internal → Alpha → Beta → Production | TestFlight → App Store |

| CI/CD Secret | Mục đích |
|---|---|
| `KEYSTORE_BASE64` | Android signing keystore (base64) |
| `KEYSTORE_PASSWORD` | Keystore password |
| `PLAY_STORE_JSON` | Google Play service account JSON |
| `IOS_DISTRIBUTION_CERT_BASE64` | iOS distribution certificate |
| `IOS_PROVISIONING_PROFILE_BASE64` | iOS provisioning profile |
| `ASC_KEY_ID` / `ASC_ISSUER_ID` | App Store Connect API keys |
| `PROD_CONFIG_JSON` | Production config (API URL, keys) |

> **Quan trọng tuyệt đối:** Keystore Android và Certificate iOS là tài sản quý giá nhất của app. Backup chúng ở nhiều nơi an toàn (password manager, encrypted cloud storage). Mất keystore = không thể cập nhật app trên Play Store mãi mãi, buộc phải publish app mới với Package Name mới, mất hết review và download.
