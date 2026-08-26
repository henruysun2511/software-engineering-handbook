# GIÁO TRÌNH REACT NATIVE
### Dành cho lập trình viên đã có nền tảng React / Next.js

---

# CHƯƠNG 9: TESTING & CI/CD

## 9.1. Jest + React Native Testing Library

### 9.1.1. Khái niệm và mức độ tái sử dụng

Nếu dự án Next.js của bạn từng dùng **Jest + React Testing Library (RTL)**, tin vui là **triết lý test hoàn toàn giống nhau** — cùng nguyên tắc "test theo hành vi người dùng nhìn thấy, không test chi tiết implementation". Khác biệt chỉ nằm ở query API vì không có DOM (không có `role`, `getByText` vẫn có nhưng cách RN hiểu "text" khác HTML).

```bash
npx expo install jest-expo jest @testing-library/react-native --save-dev
```

```js
// jest.config.js
module.exports = {
  preset: 'jest-expo',
  transformIgnorePatterns: [
    'node_modules/(?!((jest-)?react-native|@react-native(-community)?)|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@unimodules/.*|unimodules|sentry-expo|native-base|react-native-svg)',
  ],
};
```

### 9.1.2. So sánh Query API: RTL (Web) vs RNTL (React Native)

| Nhu cầu | React Testing Library (Web) | React Native Testing Library |
|---|---|---|
| Tìm theo text hiển thị | `getByText('Lưu khóa học')` | `getByText('Lưu khóa học')` — giống hệt |
| Tìm theo placeholder | `getByPlaceholderText('Tên khóa học')` | `getByPlaceholderText('Tên khóa học')` — giống hệt |
| Tìm theo role | `getByRole('button')` | `getByRole('button')` — RN có ánh xạ `accessibilityRole` |
| Giả lập click | `fireEvent.click(button)` | `fireEvent.press(button)` |
| Giả lập nhập liệu | `fireEvent.change(input, {...})` | `fireEvent.changeText(input, '...')` |
| Query theo `data-testid` | `getByTestId` | `getByTestId` (dùng prop `testID` thay `data-testid`) |

### 9.1.3. Ví dụ production: Test `CourseForm` (từ Chương 4/5)

```tsx
// src/components/course/__tests__/CourseForm.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react-native';
import { CourseForm } from '../CourseForm';

describe('CourseForm', () => {
  it('hiển thị lỗi validate khi bỏ trống tên khóa học', async () => {
    const onSubmit = jest.fn();
    render(<CourseForm onSubmit={onSubmit} />);

    // Bấm nút Lưu mà không nhập gì — kích hoạt Zod validate (giống hệt logic Web)
    fireEvent.press(screen.getByText('Lưu khóa học'));

    await waitFor(() => {
      expect(screen.getByText('Tên khóa học là bắt buộc')).toBeTruthy();
    });
    expect(onSubmit).not.toHaveBeenCalled();
  });

  it('gọi onSubmit với dữ liệu hợp lệ', async () => {
    const onSubmit = jest.fn();
    render(<CourseForm onSubmit={onSubmit} />);

    fireEvent.changeText(screen.getByPlaceholderText('VD: English Foundation A1'), 'IELTS Foundation');
    fireEvent.changeText(screen.getByPlaceholderText('VD: 24'), '20');
    fireEvent.changeText(screen.getByPlaceholderText('VD: 2500000'), '3000000');
    fireEvent.press(screen.getByText('Lưu khóa học'));

    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith(
        expect.objectContaining({ courseName: 'IELTS Foundation', tuitionFee: '3000000' })
      );
    });
  });
});
```

**Giải thích:** Vì `courseInputSchema` (Zod) hoàn toàn tái sử dụng từ Web (Chương 4), **logic validate không cần test lại riêng cho RN** — chỉ cần test rằng UI hiển thị đúng thông báo lỗi và gọi đúng callback. Đây là lợi ích thực tế của việc tách schema layer độc lập UI.

### 9.1.4. Mock TanStack Query trong test

```tsx
// src/test-utils/renderWithQueryClient.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { render } from '@testing-library/react-native';

export function renderWithQueryClient(ui: React.ReactElement) {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false }, mutations: { retry: false } },
  });
  return render(<QueryClientProvider client={queryClient}>{ui}</QueryClientProvider>);
}
```

```tsx
// Test màn hình danh sách khóa học có gọi API
jest.mock('@/services/course.service');
import courseService from '@/services/course.service';

it('hiển thị danh sách khóa học sau khi tải xong', async () => {
  (courseService.getCourses as jest.Mock).mockResolvedValue({
    data: { data: [{ courseId: '1', courseName: 'React Native Cơ bản', tuitionFee: 1000000 }] },
  });

  renderWithQueryClient(<AdminCourseScreen />);

  expect(await screen.findByText('React Native Cơ bản')).toBeTruthy();
});
```

---

## 9.2. Detox — End-to-End Testing

### 9.2.1. Khái niệm và so sánh với Playwright/Cypress

Detox mô phỏng hành vi người dùng thật trên **thiết bị/giả lập thật sự** (không phải môi trường giả lập DOM như Jest) — tương đương vai trò của Playwright/Cypress bên Web, nhưng chạy trên simulator iOS/emulator Android.

| Tiêu chí | Playwright/Cypress (Web) | Detox (RN) |
|---|---|---|
| Môi trường chạy | Trình duyệt thật (headless/headed) | Simulator/Emulator/thiết bị thật |
| Tốc độ | Nhanh | Chậm hơn đáng kể (cần build app trước khi test) |
| Độ tin cậy | Cao | Cao, nhưng nhạy với animation/timing hơn |
| Đối tượng test | Toàn bộ luồng người dùng qua URL | Toàn bộ luồng người dùng qua thao tác chạm |

### 9.2.2. Ví dụ production: Test luồng thêm khóa học hoàn chỉnh

```bash
npm install detox --save-dev
```

```js
// e2e/courseFlow.test.js
describe('Luồng quản lý khóa học', () => {
  beforeAll(async () => {
    await device.launchApp();
  });

  it('thêm khóa học mới thành công', async () => {
    // Điều hướng — tương đương việc Playwright gọi page.goto()
    await element(by.id('add-course-button')).tap();

    await element(by.id('course-name-input')).typeText('IELTS Advanced');
    await element(by.id('total-sessions-input')).typeText('30');
    await element(by.id('tuition-fee-input')).typeText('5000000');

    await element(by.id('submit-course-button')).tap();

    // Kiểm tra toast thành công xuất hiện — tương tự expect(page.getByText(...)).toBeVisible()
    await expect(element(by.text('Thêm khóa học thành công'))).toBeVisible();

    // Kiểm tra item mới xuất hiện trong danh sách
    await expect(element(by.text('IELTS Advanced'))).toBeVisible();
  });

  it('xóa khóa học bằng thao tác vuốt', async () => {
    await element(by.text('IELTS Advanced')).swipe('left');
    await element(by.id('confirm-delete-button')).tap();
    await expect(element(by.text('IELTS Advanced'))).not.toBeVisible();
  });
});
```

**Lưu ý quan trọng:** Để Detox tìm được element chính xác, cần gắn `testID` tường minh vào component (tương tự `data-testid` bên Web) — nên bổ sung ngay từ khi viết component production, không đợi đến lúc viết test mới thêm vào.

```tsx
<TextInput testID="course-name-input" {...props} />
<Pressable testID="submit-course-button" onPress={handleSubmit}>...</Pressable>
```

> **Khuyến nghị thực tế:** Vì Detox tốn thời gian build và chạy chậm hơn nhiều so với Playwright, hầu hết đội ngũ chỉ viết E2E cho **luồng nghiệp vụ cốt lõi** (đăng nhập, thanh toán, luồng CRUD chính) — không cover toàn bộ ứng dụng như unit test.

---

## 9.3. EAS Build & Submit

### 9.3.1. Khái niệm

EAS (Expo Application Services) Build là dịch vụ **build trên cloud của Expo** — điểm mấu chốt: **build được cả file `.ipa` (iOS) mà không cần máy Mac**, khác hẳn React Native CLI thuần túy (bắt buộc cần Mac + Xcode để build iOS).

```bash
npm install -g eas-cli
eas login
eas build:configure
```

### 9.3.2. Cấu hình Build Profile

```json
// eas.json
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "env": { "EXPO_PUBLIC_API_URL": "https://api-dev.example.com" }
    },
    "preview": {
      "distribution": "internal",
      "env": { "EXPO_PUBLIC_API_URL": "https://api-staging.example.com" }
    },
    "production": {
      "autoIncrement": true,
      "env": { "EXPO_PUBLIC_API_URL": "https://api.example.com" }
    }
  },
  "submit": {
    "production": {}
  }
}
```

**So sánh:** Ba `profile` (`development`/`preview`/`production`) tương đương ba môi trường bạn quen thuộc từ Next.js (`.env.development`/`.env.staging`/`.env.production`), nhưng ở đây còn quyết định **cách đóng gói app** (`developmentClient` cho phép cài thêm native module để debug, `internal` phân phối nội bộ qua link không cần store).

### 9.3.3. Lệnh build production

```bash
# Build cả hai nền tảng cho production
eas build --platform all --profile production

# Build chỉ để test nội bộ (gửi link cho QA cài trực tiếp, không cần qua store)
eas build --platform android --profile preview

# Nộp bản build lên App Store / Play Store
eas submit --platform ios --latest
eas submit --platform android --latest
```

**Giải thích quy trình:** Khác với deploy Next.js (`vercel deploy` xong là live ngay), build mobile có **độ trễ nhiều lớp**: build cloud (~15-30 phút) → nộp lên store (`submit`) → **store tự review** (vài giờ đến vài ngày, đặc biệt App Store) → mới đến tay người dùng. Đây là khác biệt lớn nhất về tốc độ vận hành so với Web.

---

## 9.4. OTA Update — Cập nhật không qua Store

### 9.4.1. Khái niệm

**OTA (Over-The-Air) Update** cho phép đẩy bản cập nhật **phần JS bundle** trực tiếp đến thiết bị người dùng mà **không cần qua quy trình duyệt của App Store/Play Store** — gần giống việc bạn deploy lại một trang Next.js, người dùng refresh là thấy bản mới, nhưng có độ trễ (chờ app khởi động lại hoặc mở lại).

**Giới hạn quan trọng cần hiểu:** OTA **chỉ cập nhật được code JS**, KHÔNG cập nhật được: thay đổi native code, thêm thư viện native mới, thay đổi icon/tên app, thay đổi permission. Những thay đổi này bắt buộc phải build lại và nộp store như bình thường.

### 9.4.2. Cấu hình `expo-updates`

```bash
npx expo install expo-updates
```

```ts
// app.config.ts
export default {
  expo: {
    updates: {
      url: 'https://u.expo.dev/<project-id>',
      fallbackToCacheTimeout: 3000, // Nếu tải bản mới quá 3s, dùng bản cache cũ, không chặn người dùng
    },
    runtimeVersion: { policy: 'appVersion' }, // Đảm bảo OTA chỉ áp dụng cho version native tương thích
  },
};
```

### 9.4.3. Đẩy bản cập nhật OTA

```bash
# Đẩy cập nhật cho kênh production — tương đương "deploy" nhưng chỉ ảnh hưởng phần JS
eas update --channel production --message "Fix lỗi hiển thị học phí"
```

```tsx
// src/hooks/useOTAUpdate.ts — chủ động kiểm tra và tải cập nhật khi app mở lại
import * as Updates from 'expo-updates';
import { useEffect } from 'react';

export function useOTAUpdate() {
  useEffect(() => {
    async function checkUpdate() {
      try {
        const { isAvailable } = await Updates.checkForUpdateAsync();
        if (isAvailable) {
          await Updates.fetchUpdateAsync();
          await Updates.reloadAsync(); // Khởi động lại app với bản JS mới
        }
      } catch (e) {
        console.log('Lỗi kiểm tra cập nhật OTA', e);
      }
    }
    checkUpdate();
  }, []);
}
```

> **Lưu ý quan trọng khi dùng OTA cho production:** Tránh `reloadAsync()` đột ngột khi người dùng đang nhập liệu (vd: đang điền `CourseForm`) — nên áp dụng OTA vào lúc mở app (cold start), không áp dụng giữa phiên sử dụng, để tránh mất dữ liệu người dùng đang nhập dở.

---

## Tổng kết Chương 9

| Nhu cầu | Web | React Native |
|---|---|---|
| Unit/Component test | Jest + React Testing Library | Jest + React Native Testing Library — triết lý giống hệt |
| E2E test | Playwright/Cypress | Detox — chậm hơn, chỉ cover luồng cốt lõi |
| Deploy | `vercel deploy` (tức thì) | EAS Build (~15-30 phút) → Submit → Store review (giờ-ngày) |
| Cập nhật nhanh không qua review | Không cần khái niệm này (Web luôn "OTA") | `expo-updates` — chỉ cập nhật được phần JS |

**Chương tiếp theo (Chương 10 — chương cuối)** trình bày quy trình chuẩn bị và phát hành ứng dụng lên App Store/Google Play: icon, splash screen, versioning, và checklist trước khi submit.
