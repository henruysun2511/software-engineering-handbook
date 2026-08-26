# GIÁO TRÌNH REACT NATIVE
### Dành cho lập trình viên đã có nền tảng React / Next.js

---

# CHƯƠNG 10: PUBLISH LÊN APP STORE / GOOGLE PLAY

> Đây là bước **không có khái niệm tương đương trên Web** — deploy một trang Next.js là "public ngay lập tức", còn phát hành app mobile đi qua một quy trình chuẩn bị tài sản (assets), đăng ký tài khoản nhà phát triển, và chờ store duyệt.

## 10.1. Chuẩn bị tài khoản nhà phát triển

### 10.1.1. Yêu cầu của hai nền tảng

| Tiêu chí | Apple App Store | Google Play |
|---|---|---|
| Chi phí tài khoản | 99 USD/năm | 25 USD (đóng 1 lần duy nhất) |
| Thời gian duyệt lần đầu | Thường 1-3 ngày, có thể lâu hơn | Thường vài giờ đến 1-2 ngày |
| Yêu cầu pháp lý | Cần D-U-N-S Number nếu đăng ký dạng Tổ chức (Organization) | Đơn giản hơn, chủ yếu cần giấy tờ cá nhân/doanh nghiệp |
| Độ khắt khe duyệt | Khắt khe hơn đáng kể (UI/UX, quyền riêng tư, nội dung) | Ít khắt khe hơn nhưng vẫn có bộ quy tắc riêng |

> **Khuyến nghị thực tế:** Đăng ký tài khoản Apple Developer **sớm nhất có thể** trong vòng đời dự án — vì quy trình xác minh (đặc biệt tài khoản Tổ chức) có thể mất vài ngày đến vài tuần, thường là điểm nghẽn tiến độ bị đánh giá thấp nhất khi lập kế hoạch release.

---

## 10.2. App Icon, Splash Screen

### 10.2.1. Khái niệm

Không giống favicon Web (1 file nhỏ, ít yêu cầu), icon app mobile cần **nhiều kích thước khác nhau** cho từng ngữ cảnh hiển thị (màn hình chính, Settings, thông báo, App Store listing). Với Expo, quy trình này được tự động hoá — bạn chỉ cần cung cấp 1 file gốc độ phân giải cao.

### 10.2.2. Cấu hình chuẩn production

```ts
// app.config.ts
export default {
  expo: {
    icon: './assets/icon.png', // Khuyến nghị: 1024x1024px, không có góc bo tròn (hệ điều hành tự bo)
    splash: {
      image: './assets/splash.png',
      resizeMode: 'contain',
      backgroundColor: '#ffffff',
    },
    android: {
      adaptiveIcon: {
        // Android yêu cầu Adaptive Icon riêng — icon có thể bị cắt theo nhiều hình dạng (tròn, vuông bo, giọt nước)
        // tùy launcher của từng hãng máy, nên foregroundImage cần chừa margin an toàn ở giữa
        foregroundImage: './assets/adaptive-icon.png',
        backgroundColor: '#4F46E5',
      },
    },
    ios: {
      icon: './assets/icon.png',
    },
  },
};
```

**Giải thích khái niệm mới `Adaptive Icon` (chỉ có trên Android):** Khác với iOS (icon luôn hình vuông bo góc cố định), Android cho phép nhà sản xuất thiết bị (Samsung, Xiaomi...) tự quyết định hình dạng khung icon hiển thị trên màn hình chính. Vì vậy Google yêu cầu tách icon thành 2 lớp: `foregroundImage` (logo chính, cần chừa khoảng đệm ~66% vùng an toàn ở giữa) và `backgroundColor`/`backgroundImage` (nền), để hệ thống tự crop theo đúng hình dạng khung của từng máy mà không cắt mất logo.

### 10.2.3. Splash Screen nâng cao (SDK 51+)

```bash
npx expo install expo-splash-screen
```

```tsx
// app/_layout.tsx
import * as SplashScreen from 'expo-splash-screen';
import { useEffect } from 'react';
import { useFonts } from 'expo-font';

SplashScreen.preventAutoHideAsync(); // Giữ splash hiển thị cho đến khi ta chủ động ẩn

export default function RootLayout() {
  const [fontsLoaded] = useFonts({
    'Inter-Bold': require('../assets/fonts/Inter-Bold.ttf'),
  });

  useEffect(() => {
    if (fontsLoaded) {
      SplashScreen.hideAsync(); // Chỉ ẩn splash sau khi font đã sẵn sàng — tránh hiện tượng "nháy font"
    }
  }, [fontsLoaded]);

  if (!fontsLoaded) return null;

  return <Stack />;
}
```

**Giải thích:** Đây là pattern giải quyết vấn đề **FOUT (Flash of Unstyled Text)** — tương tự vấn đề font-loading trên Web mà Next.js xử lý bằng `next/font`, nhưng trên RN phải tự kiểm soát thời điểm ẩn splash screen thủ công cho đến khi mọi tài nguyên cần thiết (font, config ban đầu) đã sẵn sàng.

---

## 10.3. Versioning

### 10.3.1. Hai khái niệm version cần phân biệt

| Khái niệm | iOS | Android | Ý nghĩa |
|---|---|---|---|
| Version hiển thị người dùng | `CFBundleShortVersionString` | `versionName` | Số bạn thấy trên store, vd: `1.2.0` — tăng theo Semantic Versioning |
| Build number nội bộ | `CFBundleVersion` | `versionCode` | Số nguyên tăng dần **mỗi lần build**, không hiển thị người dùng, dùng để store phân biệt các bản build trùng version hiển thị |

```ts
// app.config.ts
export default {
  expo: {
    version: '1.2.0', // Version hiển thị — đổi khi release tính năng/sửa lỗi cho người dùng thấy
    ios: { buildNumber: '14' },
    android: { versionCode: 14 },
  },
};
```

> **Lỗi rất hay gặp:** Quên tăng `buildNumber`/`versionCode` khi build lại để nộp store sau khi bị từ chối (reject) — cả hai store đều **từ chối nhận bản build có số build trùng với bản đã nộp trước đó**, kể cả khi `version` hiển thị không đổi. Với `eas.json` đã cấu hình `"autoIncrement": true` (xem Chương 9), EAS sẽ tự tăng số này — nên bật cấu hình này ngay từ đầu để tránh lỗi thủ công.

---

## 10.4. Checklist trước khi Submit

### 10.4.1. Checklist kỹ thuật

| Hạng mục | Ghi chú |
|---|---|
| Đã test trên thiết bị thật (không chỉ simulator/emulator) | Một số bug (camera, push, GPS) chỉ xuất hiện trên máy thật |
| Đã cấu hình đầy đủ `NSCameraUsageDescription` và các lời nhắn quyền khác | Xem lại Chương 6, mục 6.1.4 — thiếu là bị Apple từ chối ngay |
| Đã ẩn/xóa toàn bộ `console.log`, mã debug | |
| Đã kiểm tra flow đăng nhập/đăng ký hoàn chỉnh không lỗi | Store reviewer luôn test luồng này đầu tiên |
| Đã cấu hình đúng `EXPO_PUBLIC_API_URL` trỏ về server production | Lỗi phổ biến: quên đổi từ URL staging |
| App hoạt động đúng khi mất mạng (không crash trắng màn hình) | Xem lại xử lý `onlineManager` ở Chương 4 |

### 10.4.2. Checklist nội dung & pháp lý (đặc biệt quan trọng với Apple)

| Hạng mục | Ghi chú |
|---|---|
| Có trang Chính sách quyền riêng tư (Privacy Policy URL) | Bắt buộc với cả hai store nếu app thu thập dữ liệu người dùng |
| Screenshot store đúng kích thước yêu cầu từng loại thiết bị | Apple yêu cầu nhiều kích thước (iPhone, iPad) nếu hỗ trợ |
| Mô tả app không chứa từ khóa gây hiểu nhầm/spam | Google/Apple đều quét tự động, dễ bị từ chối |
| Nếu có tính năng đăng nhập, cần cung cấp tài khoản demo cho reviewer | Bắt buộc với Apple nếu app yêu cầu đăng nhập mới xem được nội dung |
| App không được yêu cầu quyền không sử dụng đến (over-permission) | Cả hai store đều kiểm tra permission khai báo có thực sự dùng trong app không |

### 10.4.3. Quy trình submit thực tế

```bash
# 1. Build bản production cuối cùng
eas build --platform all --profile production

# 2. Nộp lên store (yêu cầu đã cấu hình App Store Connect API Key / Google Service Account trước)
eas submit --platform ios --latest
eas submit --platform android --latest

# 3. Theo dõi trạng thái duyệt qua App Store Connect / Google Play Console
# 4. Nếu bị từ chối (rejected): đọc kỹ lý do, sửa, tăng buildNumber/versionCode, build & submit lại
```

---

## 10.5. Sau khi Publish: Giám sát Production

### 10.5.1. Vì sao cần công cụ giám sát riêng cho Mobile

Không giống Web (bạn có thể xem log server trực tiếp), lỗi xảy ra trên **thiết bị của người dùng** — bạn không có quyền truy cập trực tiếp để debug. Cần tích hợp công cụ crash reporting/error tracking **trước khi** phát hành phiên bản đầu tiên.

```bash
npx expo install @sentry/react-native
```

```tsx
// app/_layout.tsx
import * as Sentry from '@sentry/react-native';

Sentry.init({
  dsn: 'https://xxxx@xxxx.ingest.sentry.io/xxxx',
  tracesSampleRate: 0.2, // Chỉ theo dõi chi tiết 20% session để tiết kiệm chi phí, vẫn đủ dữ liệu thống kê
  enabled: !__DEV__, // Không gửi lỗi khi đang phát triển local
});
```

**So sánh:** Nếu dự án Next.js của bạn đã dùng Sentry cho error tracking, đây là **cùng một nền tảng, cùng dashboard theo dõi** — chỉ khác SDK khởi tạo, giúp đội ngũ có 1 nơi duy nhất giám sát lỗi cho cả Web lẫn Mobile.

### 10.5.2. Theo dõi chỉ số nghiệp vụ

Ngoài crash, nên theo dõi thêm: tỷ lệ giữ chân người dùng (retention), thời gian tải màn hình chính, tỷ lệ hoàn thành luồng CRUD (vd: % người vào form thêm khóa học nhưng bỏ dở) — dùng công cụ như Firebase Analytics hoặc Amplitude, tích hợp tương tự cách bạn dùng Google Analytics/Mixpanel bên Web.

---

## Tổng kết Chương 10

| Giai đoạn | Mức độ tương đồng với Web |
|---|---|
| Tài khoản nhà phát triển | Khái niệm mới hoàn toàn, cần chuẩn bị sớm (đặc biệt Apple) |
| Icon/Splash | Yêu cầu nhiều kích thước/biến thể hơn favicon Web |
| Versioning | Có thêm khái niệm build number không hiển thị người dùng |
| Quy trình duyệt | Độ trễ lớn (giờ-ngày), khác hẳn deploy tức thì của Web |
| Giám sát production | Tái sử dụng được công cụ quen thuộc (Sentry, Analytics) với SDK riêng cho RN |

---

# TỔNG KẾT TOÀN GIÁO TRÌNH

Qua 10 chương, bạn đã đi từ nền tảng kiến trúc React Native, qua styling/component, navigation, state management (tái sử dụng gần như toàn bộ hệ sinh thái Zustand/TanStack Query/Zod/Axios đã có), forms & design system thay thế shadcn/ui, các năng lực đặc thù thiết bị, animation/gesture, tối ưu hiệu năng, đến quy trình testing và phát hành sản phẩm hoàn chỉnh.

**Bản đồ tổng thể mức độ tái sử dụng kiến thức Web → React Native:**

| Lớp kiến trúc | Mức tái sử dụng |
|---|---|
| Business logic (Zod, Axios service, TanStack Query hooks, Zustand) | 90–100% |
| Tư duy component hóa, hooks, unidirectional data flow | 100% (bản chất React không đổi) |
| Routing/Navigation (nếu dùng Expo Router) | ~70–80% tư duy file-based |
| UI Component (View/Text thay div/span, StyleSheet/NativeWind) | ~20–30%, cần viết lại phần lớn |
| Native-specific (Permission, Camera, Push, Gesture) | 0%, kiến thức hoàn toàn mới |
| Testing | ~80% triết lý, khác API bề mặt |
| Deploy/Release | Khác biệt quy trình hoàn toàn |

Với nền tảng bạn đã có sẵn (React, Next.js, TanStack Query, Zustand, RTK, Tailwind, shadcn, Axios, Zod), phần khó nhất khi chuyển sang React Native **không phải là học lại tư duy lập trình**, mà là làm quen với **ràng buộc và pattern UX đặc thù của thiết bị di động** — đúng như tinh thần xuyên suốt giáo trình này.
