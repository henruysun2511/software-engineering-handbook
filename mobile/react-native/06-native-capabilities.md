# GIÁO TRÌNH REACT NATIVE
### Dành cho lập trình viên đã có nền tảng React / Next.js

---

# CHƯƠNG 6: NATIVE CAPABILITIES

> Đây là chương **hoàn toàn mới** với người làm Web — không có khái niệm tương đương trực tiếp, vì trình duyệt chạy trong sandbox cách ly khỏi phần cứng thiết bị, còn ứng dụng mobile được cấp quyền truy cập trực tiếp camera, vị trí, bộ nhớ, thông báo hệ thống.

## 6.1. Permissions — Hệ thống cấp quyền

### 6.1.1. Khái niệm

Mọi truy cập vào tài nguyên nhạy cảm (camera, vị trí, danh bạ, thông báo) đều phải xin quyền người dùng tường minh tại **runtime** — khác hẳn Web, nơi trình duyệt tự hỏi quyền qua popup có sẵn (vd: `navigator.geolocation`) mà lập trình viên không cần quản lý vòng đời quyền.

### 6.1.2. Bảng các loại quyền phổ biến

| Quyền | Package Expo | Dùng khi nào |
|---|---|---|
| Camera | `expo-camera` | Chụp ảnh, quét QR |
| Thư viện ảnh | `expo-image-picker` | Chọn ảnh có sẵn (đã dùng ở Chương 4) |
| Vị trí | `expo-location` | Bản đồ, giao hàng, check-in |
| Thông báo đẩy | `expo-notifications` | Push notification |
| Danh bạ | `expo-contacts` | Mời bạn bè, đồng bộ liên hệ |
| Microphone | `expo-av` | Ghi âm, gọi video |

### 6.1.3. Ví dụ production: Xin quyền Camera đúng chuẩn

```tsx
// src/hooks/useCameraPermission.ts
import { useState, useCallback } from 'react';
import { Camera } from 'expo-camera';
import { Alert, Linking } from 'react-native';

export function useCameraPermission() {
  const [status, setStatus] = useState<'undetermined' | 'granted' | 'denied'>('undetermined');

  const requestPermission = useCallback(async () => {
    const { status: currentStatus, canAskAgain } = await Camera.requestCameraPermissionsAsync();
    setStatus(currentStatus as any);

    // Trường hợp người dùng đã từ chối vĩnh viễn (canAskAgain = false)
    // — hệ thống sẽ KHÔNG hiện lại popup xin quyền nữa, phải hướng dẫn vào Settings
    if (currentStatus === 'denied' && !canAskAgain) {
      Alert.alert(
        'Cần quyền truy cập Camera',
        'Vui lòng bật quyền Camera trong Cài đặt để sử dụng tính năng này.',
        [
          { text: 'Hủy', style: 'cancel' },
          { text: 'Mở Cài đặt', onPress: () => Linking.openSettings() },
        ]
      );
    }
    return currentStatus === 'granted';
  }, []);

  return { status, requestPermission };
}
```

**Giải thích nguyên tắc thiết kế quan trọng:** `canAskAgain: false` là trạng thái đặc thù của mobile — sau khi người dùng từ chối quyền 1-2 lần (tùy hệ điều hành), hệ thống sẽ **không hiện lại popup xin quyền native nữa**, buộc ứng dụng phải tự điều hướng người dùng vào màn hình Cài đặt hệ thống (`Linking.openSettings()`). Đây là pattern UX **bắt buộc phải xử lý** trong mọi app production — thiếu bước này là lỗi UX rất phổ biến khiến người dùng "kẹt" không thể cấp lại quyền.

### 6.1.4. Cấu hình lời nhắn xin quyền (bắt buộc khi submit App Store)

```ts
// app.config.ts
export default {
  expo: {
    ios: {
      infoPlist: {
        NSCameraUsageDescription: 'Ứng dụng cần quyền Camera để bạn chụp ảnh khóa học.',
        NSPhotoLibraryUsageDescription: 'Ứng dụng cần quyền Thư viện ảnh để chọn ảnh khóa học.',
      },
    },
    android: {
      permissions: ['CAMERA', 'READ_EXTERNAL_STORAGE'],
    },
  },
};
```

> **Lưu ý bắt buộc:** Apple **từ chối duyệt app** nếu thiếu `NSCameraUsageDescription`/`NSPhotoLibraryUsageDescription` — đây là dòng chữ hiển thị ngay trong popup xin quyền của iOS, giải thích cho người dùng vì sao app cần quyền này. Không có khái niệm tương đương bắt buộc bên Web.

---

## 6.2. Lưu trữ cục bộ: AsyncStorage / MMKV

### 6.2.1. So sánh với localStorage

| Tiêu chí | `localStorage` (Web) | `AsyncStorage` | `MMKV` |
|---|---|---|---|
| API | Đồng bộ | Bất đồng bộ (Promise) | Đồng bộ |
| Tốc độ | Nhanh | Chậm hơn MMKV ~10-30 lần | Rất nhanh (C++ native) |
| Dung lượng | ~5-10MB | Không giới hạn cứng, nhưng chậm với dữ liệu lớn | Tối ưu cho dữ liệu lớn |
| Dùng cho | Mọi nhu cầu lưu trữ đơn giản | Cache thông thường | Cache tần suất đọc/ghi cao (vd: theo dõi trạng thái real-time) |

**Khuyến nghị production:** Dùng `AsyncStorage` làm mặc định (API ổn định, tương thích Expo Go), chỉ chuyển sang `MMKV` khi đã đo đạc thấy nghẽn hiệu năng thực sự (MMKV yêu cầu Development Build, không chạy trên Expo Go).

### 6.2.2. Ví dụ production: Cache filter đã chọn (không dùng SecureStore vì không nhạy cảm)

```ts
// src/utils/storage.ts
import AsyncStorage from '@react-native-async-storage/async-storage';

export const storage = {
  async get<T>(key: string): Promise<T | null> {
    try {
      const raw = await AsyncStorage.getItem(key);
      return raw ? JSON.parse(raw) : null;
    } catch {
      return null;
    }
  },
  async set<T>(key: string, value: T): Promise<void> {
    await AsyncStorage.setItem(key, JSON.stringify(value));
  },
  async remove(key: string): Promise<void> {
    await AsyncStorage.removeItem(key);
  },
};
```

```ts
// Kết hợp với Zustand persist (đã giới thiệu Chương 4) cho state không nhạy cảm
import { createJSONStorage, persist } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

export const useFilterStore = create(
  persist(
    (set) => ({ lastLevel: null, setLastLevel: (level: string) => set({ lastLevel: level }) }),
    { name: 'course-filter', storage: createJSONStorage(() => AsyncStorage) }
  )
);
```

---

## 6.3. Camera, Image Picker, File System

### 6.3.1. Chụp ảnh trực tiếp — mở rộng `useImageUpload` từ Chương 4

Ở Chương 4, `pickImage` chỉ mở thư viện ảnh (`launchImageLibraryAsync`). Trong production thực tế, thường cần cho người dùng chọn giữa **Camera** và **Thư viện**:

```tsx
// src/hooks/useCourseImagePicker.ts
import * as ImagePicker from 'expo-image-picker';
import { Alert } from 'react-native';

export function useCourseImagePicker() {
  const pickFromCamera = async () => {
    const permission = await ImagePicker.requestCameraPermissionsAsync();
    if (!permission.granted) return null;

    const result = await ImagePicker.launchCameraAsync({
      quality: 0.8,
      allowsEditing: true,
      aspect: [16, 9],
    });
    return result.canceled ? null : result.assets[0];
  };

  const pickFromLibrary = async () => {
    const permission = await ImagePicker.requestMediaLibraryPermissionsAsync();
    if (!permission.granted) return null;

    const result = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ImagePicker.MediaTypeOptions.Images,
      quality: 0.8,
      allowsEditing: true,
      aspect: [16, 9],
    });
    return result.canceled ? null : result.assets[0];
  };

  const showPicker = () => {
    return new Promise<ImagePicker.ImagePickerAsset | null>((resolve) => {
      Alert.alert('Chọn ảnh khóa học', undefined, [
        { text: 'Chụp ảnh', onPress: async () => resolve(await pickFromCamera()) },
        { text: 'Chọn từ thư viện', onPress: async () => resolve(await pickFromLibrary()) },
        { text: 'Hủy', style: 'cancel', onPress: () => resolve(null) },
      ]);
    });
  };

  return { showPicker };
}
```

**Giải thích quyết định thiết kế:** Dùng `Alert.alert` làm action sheet lựa chọn nguồn ảnh — pattern chuẩn, quen thuộc trên cả iOS/Android (tương tự việc app Facebook/Zalo hỏi "Chụp ảnh hay chọn từ thư viện"). Với thiết kế branding riêng, có thể thay bằng `<Modal>` tùy biến giống `<Select>` đã dựng ở Chương 5.

### 6.3.2. Nén và resize ảnh trước khi upload

**Vấn đề thực tế:** Ảnh chụp trực tiếp từ camera điện thoại hiện đại thường 3-5MB, gây chậm khi upload trên mạng di động. Cần nén trước khi gửi lên server:

```bash
npx expo install expo-image-manipulator
```

```ts
// src/utils/imageCompress.ts
import * as ImageManipulator from 'expo-image-manipulator';

export async function compressImage(uri: string) {
  const result = await ImageManipulator.manipulateAsync(
    uri,
    [{ resize: { width: 1200 } }], // Resize chiều rộng tối đa 1200px, chiều cao tự tính theo tỉ lệ
    { compress: 0.7, format: ImageManipulator.SaveFormat.JPEG }
  );
  return result.uri;
}
```

```ts
// Tích hợp vào useImageUpload (Chương 4)
const uploaded = await handleUpload(await compressImage(asset.uri), 'courses');
```

---

## 6.4. Push Notification

### 6.4.1. Khái niệm

Push Notification cho phép backend gửi thông báo đến thiết bị ngay cả khi app đang đóng — không có khái niệm tương đương hoàn chỉnh trên Web (Web Push tồn tại nhưng hạn chế hơn nhiều, đặc biệt trên iOS Safari).

### 6.4.2. Luồng hoạt động

```
App khởi động → Đăng ký nhận push → Lấy Expo Push Token
     → Gửi token lên Backend lưu theo user
     → Backend gọi Expo Push API kèm token khi cần gửi thông báo
     → Thiết bị nhận thông báo (kể cả khi app đóng)
```

### 6.4.3. Ví dụ production: Đăng ký nhận Push Notification

```ts
// src/hooks/usePushNotifications.ts
import { useEffect, useRef, useState } from 'react';
import * as Notifications from 'expo-notifications';
import * as Device from 'expo-device';
import { Platform } from 'react-native';
import http from '@/utils/http';

// Cấu hình cách notification hiển thị khi app đang mở (foreground)
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,
    shouldPlaySound: true,
    shouldSetBadge: false,
  }),
});

export function usePushNotifications() {
  const [expoPushToken, setExpoPushToken] = useState<string>();

  useEffect(() => {
    registerForPushNotifications().then((token) => {
      if (token) {
        setExpoPushToken(token);
        // Gửi lên backend để lưu token theo user — tương tự việc lưu FCM token
        http.post('/notifications/register-token', { token, platform: Platform.OS });
      }
    });

    // Lắng nghe khi người dùng bấm vào thông báo — điều hướng đến màn hình liên quan
    const subscription = Notifications.addNotificationResponseReceivedListener((response) => {
      const courseId = response.notification.request.content.data?.courseId;
      if (courseId) {
        // router.push(`/course/${courseId}`) — điều hướng deep-link nội bộ
      }
    });

    return () => subscription.remove();
  }, []);

  return { expoPushToken };
}

async function registerForPushNotifications() {
  if (!Device.isDevice) return null; // Push notification không hoạt động trên giả lập

  const { status: existingStatus } = await Notifications.getPermissionsAsync();
  let finalStatus = existingStatus;

  if (existingStatus !== 'granted') {
    const { status } = await Notifications.requestPermissionsAsync();
    finalStatus = status;
  }
  if (finalStatus !== 'granted') return null;

  const token = (await Notifications.getExpoPushTokenAsync()).data;

  if (Platform.OS === 'android') {
    await Notifications.setNotificationChannelAsync('default', {
      name: 'default',
      importance: Notifications.AndroidImportance.MAX,
    });
  }

  return token;
}
```

**Giải thích các điểm cần lưu ý production:**
- `Device.isDevice`: kiểm tra thiết bị thật vì **push notification không hoạt động trên simulator/emulator** — bug rất hay gặp khi mới học (tưởng code sai nhưng thực ra do test trên giả lập).
- `setNotificationChannelAsync`: Android yêu cầu khai báo "Notification Channel" (nhóm loại thông báo, người dùng có thể tắt riêng từng nhóm trong Cài đặt) — khái niệm không tồn tại trên iOS/Web.
- `addNotificationResponseReceivedListener`: xử lý khi người dùng **bấm vào** thông báo (khác với khi thông báo chỉ hiện ra) — thường dùng để điều hướng sâu vào màn hình liên quan, đúng tinh thần Deep Linking đã học ở Chương 3.

---

## 6.5. Native Modules — Khi cần viết code Native thật sự

### 6.5.1. Khi nào cần

Tuyệt đại đa số nhu cầu (camera, vị trí, storage, notification...) đã có sẵn qua Expo SDK hoặc thư viện cộng đồng. Chỉ cần viết Native Module khi:
- Tích hợp SDK native độc quyền chưa có JS wrapper (vd: SDK thanh toán ngân hàng nội địa)
- Cần hiệu năng cực hạn cho tác vụ nặng (xử lý ảnh/video phức tạp)
- Tái sử dụng code Swift/Kotlin có sẵn từ dự án native cũ

### 6.5.2. Ví dụ minh họa cấu trúc (dùng Expo Modules API — cách hiện đại nhất)

```swift
// modules/device-info/ios/DeviceInfoModule.swift
import ExpoModulesCore

public class DeviceInfoModule: Module {
  public func definition() -> ModuleDefinition {
    Name("DeviceInfo")

    Function("getBatteryLevel") { () -> Float in
      UIDevice.current.isBatteryMonitoringEnabled = true
      return UIDevice.current.batteryLevel
    }
  }
}
```

```ts
// modules/device-info/index.ts — phía JS gọi vào
import { requireNativeModule } from 'expo-modules-core';

const DeviceInfoModule = requireNativeModule('DeviceInfo');

export function getBatteryLevel(): number {
  return DeviceInfoModule.getBatteryLevel();
}
```

> **Ghi chú phạm vi giáo trình:** Native Module là chủ đề chuyên sâu, đòi hỏi kiến thức Swift/Kotlin riêng — nằm ngoài phạm vi tài liệu này. Với 95% nhu cầu production, bạn sẽ không cần viết Native Module — nên xem đây là kiến thức "biết để không sợ", không phải kỹ năng bắt buộc học ngay.

---

## Tổng kết Chương 6

| Nhu cầu | Không tồn tại trên Web | Giải pháp RN |
|---|---|---|
| Truy cập phần cứng | — | Xin quyền runtime (`Permissions`) + xử lý `canAskAgain` |
| Lưu trữ cục bộ hiệu năng cao | `localStorage` đơn giản, giới hạn 5-10MB | `AsyncStorage` (mặc định) / `MMKV` (hiệu năng cao) |
| Nhận thông báo khi app đóng | Web Push hạn chế | `expo-notifications` + Expo Push Token |
| Tích hợp SDK native độc quyền | — | Expo Modules API (Swift/Kotlin) |

**Chương tiếp theo (Chương 7)** đi vào Animation & Gesture — nơi Reanimated cho phép tạo animation chạy mượt 60fps trên native thread, khác hẳn animation CSS/Framer Motion chạy trên main thread của trình duyệt.
