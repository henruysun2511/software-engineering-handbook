# TỔNG HỢP KIẾN THỨC REACT NATIVE
## Dành cho lập trình viên đã có nền tảng React

> **Chủ đề:** React Native – Kiến trúc, Component, Styling, Navigation và các khái niệm đặc thù cho Mobile  
> **Phiên bản tham chiếu:** React Native 0.78+ / Expo SDK 52+ / React Navigation v7+

---

## MỤC LỤC

1. [Giới thiệu](#1-giới-thiệu)
2. [Tổng quan – React vs React Native](#2-tổng-quan--react-vs-react-native)
3. [Kiến trúc Bridge và New Architecture](#3-kiến-trúc-bridge-và-new-architecture)
4. [Môi trường phát triển và Build](#4-môi-trường-phát-triển-và-build)
5. [Core Components – Thay thế HTML Tags](#5-core-components--thay-thế-html-tags)
6. [StyleSheet – Thay thế CSS](#6-stylesheet--thay-thế-css)
7. [Flexbox trong React Native](#7-flexbox-trong-react-native)
8. [Xử lý sự kiện và Touch](#8-xử-lý-sự-kiện-và-touch)
9. [Navigation – Điều hướng màn hình](#9-navigation--điều-hướng-màn-hình)
10. [Platform API – Truy cập tính năng thiết bị](#10-platform-api--truy-cập-tính-năng-thiết-bị)
11. [AsyncStorage và Lưu trữ cục bộ](#11-asyncstorage-và-lưu-trữ-cục-bộ)
12. [Animated API và Gestures – Hoạt ảnh](#12-animated-api-và-gestures--hoạt-ảnh)
13. [FlatList và SectionList – Danh sách hiệu năng cao](#13-flatlist-và-sectionlist--danh-sách-hiệu-năng-cao)
14. [Modal, Alert, và Overlay](#14-modal-alert-và-overlay)
15. [Permissions và Expo APIs](#15-permissions-và-expo-apis)
16. [Fonts và Assets](#16-fonts-và-assets)
17. [Hiệu năng và Tối ưu ứng dụng](#17-hiệu-năng-và-tối-ưu-ứng-dụng)
18. [Debugging và Công cụ phát triển](#18-debugging-và-công-cụ-phát-triển)
19. [Lộ trình học tập đề xuất](#19-lộ-trình-học-tập-đề-xuất)
20. [Bảng so sánh tổng hợp React vs React Native](#20-bảng-so-sánh-tổng-hợp-react-vs-react-native)
21. [Checklist chuyển từ React sang React Native](#21-checklist-chuyển-từ-react-sang-react-native)

---

## 1. Giới thiệu

React Native là framework mã nguồn mở do Meta phát triển, cho phép lập trình viên xây dựng ứng dụng di động đa nền tảng (iOS và Android) bằng JavaScript và mô hình lập trình React. Khác với các giải pháp hybrid truyền thống (Ionic, Cordova) chạy trong WebView, React Native biên dịch các thành phần giao diện sang **native widgets thực sự** của từng nền tảng, mang lại hiệu năng và trải nghiệm người dùng tiệm cận ứng dụng native thuần túy (Swift/Kotlin).

Tài liệu này hệ thống hóa các kiến thức trọng tâm mà lập trình viên đã thành thạo React web cần nắm vững khi chuyển sang React Native, kèm ví dụ mã nguồn thực tế và bảng so sánh minh họa.

---

## 2. Tổng quan – React vs React Native

### 2.1 Định nghĩa

**React** là thư viện JavaScript để xây dựng giao diện người dùng cho **web**, render ra các thẻ HTML chạy trong trình duyệt.

**React Native** là framework cho phép dùng cú pháp React để xây dựng ứng dụng **mobile native** (iOS & Android). Thay vì render HTML, React Native ánh xạ component sang UI native thực thụ của từng nền tảng.

### 2.2 Nguyên lý hoạt động

```
REACT (Web)
JSX → Virtual DOM → HTML DOM → Trình duyệt hiển thị

REACT NATIVE (Mobile)
JSX → Virtual DOM → Native Bridge / JSI → iOS UIView / Android View
```

Kết quả: ứng dụng React Native **không phải WebView**, mà là app native thật sự.

### 2.3 Những thứ giữ nguyên từ React

React Native **kế thừa toàn bộ** mô hình React:

| Giữ nguyên | Mô tả |
|-----------|-------|
| JSX | Cú pháp giống hệt |
| Component (Function/Class) | Cách viết component không đổi |
| Props & State | Hoạt động như React web |
| Tất cả Hooks | useState, useEffect, useContext, ... |
| Context API | Chia sẻ state toàn cục |
| Lifecycle | Các giai đoạn mount/update/unmount |

---

## 3. Kiến trúc Bridge và New Architecture

### 3.1 Kiến trúc cũ – JavaScript Bridge

Đây là điểm khác biệt căn bản nhất về mặt kỹ thuật giữa React web và React Native.

```
┌─────────────────────┐      JSON Messages       ┌──────────────────────┐
│   JavaScript Thread │ ◄──────────────────────► │  Native Thread       │
│  (React, Logic)     │       (async)             │  (iOS/Android UI)    │
└─────────────────────┘                           └──────────────────────┘
              ↑
         JS Bridge
    (nút thắt cổ chai)
```

**Vấn đề:** Mọi giao tiếp qua Bridge đều là bất đồng bộ và serialize/deserialize JSON → gây lag khi truyền dữ liệu lớn.

### 3.2 New Architecture (React Native 0.71+)

New Architecture thay thế Bridge bằng **JSI (JavaScript Interface)**, cho phép JavaScript gọi thẳng vào native code mà không cần serialize JSON.

```
┌─────────────────────┐   Trực tiếp (synchronous)  ┌──────────────────────┐
│   JavaScript (JSI)  │ ◄────────────────────────► │  C++ Host Objects    │
└─────────────────────┘                             └──────────────────────┘
                                                             │
                                              ┌──────────────┴──────────────┐
                                         iOS Native               Android Native
```

Hai thành phần chính:
- **Fabric:** Renderer mới, cho phép UI update đồng bộ
- **TurboModules:** Native modules nhẹ hơn, lazy-loaded

> **Lưu ý:** Từ React Native 0.74+, New Architecture được khuyến nghị bật mặc định cho project mới.

---

## 4. Môi trường phát triển và Build

Thiết lập môi trường là một trong những rào cản đầu tiên. Có hai luồng phát triển chính:

| Tiêu chí | Expo (Managed Workflow) | React Native CLI |
|---------|------------------------|------------------|
| Cài đặt ban đầu | Cực kỳ đơn giản (`npx create-expo-app`) | Yêu cầu Xcode + Android Studio |
| Native modules | Chỉ dùng modules Expo hỗ trợ | Toàn quyền cài thư viện native |
| OTA Updates | Hỗ trợ (EAS Update) | Cần tự triển khai |
| Build CI/CD | EAS Build (cloud) | Fastlane, tự quản lý |
| Phù hợp cho | Startup, MVP, học tập | Enterprise, tùy chỉnh sâu |
| Kích thước bundle | Lớn hơn (includes Expo SDK) | Nhỏ hơn, tối ưu hơn |

### 4.1 Metro Bundler

Metro là bundler mặc định của React Native, thay thế Webpack/Vite. Metro tối ưu cho mobile với Fast Refresh, module resolution ưu tiên nền tảng, và tree-shaking.

```js
// metro.config.js
const { getDefaultConfig, mergeConfig } = require('@react-native/metro-config');

const config = {
  transformer: {
    babelTransformerPath: require.resolve('react-native-svg-transformer'),
  },
  resolver: {
    assetExts: assetExts.filter((ext) => ext !== 'svg'),
    sourceExts: [...sourceExts, 'svg'],
  },
  watchFolders: [path.resolve(__dirname, '../shared')],
};

module.exports = mergeConfig(getDefaultConfig(__dirname), config);
```

---

## 5. Core Components – Thay thế HTML Tags

React Native **không dùng thẻ HTML**. Thay vào đó, có các **Core Components** tương ứng được ánh xạ sang UI native.

### 5.1 Bảng ánh xạ thành phần

| HTML (React Web) | React Native | Ghi chú |
|-----------------|--------------|---------|
| `<div>` | `<View>` | Container chính, hỗ trợ Flexbox |
| `<span>`, `<p>`, `<h1>`...`<h6>` | `<Text>` | **Mọi** text đều phải trong `<Text>` |
| `<img>` | `<Image>` | Ảnh local và remote |
| `<input>` | `<TextInput>` | Ô nhập liệu |
| `<button>` | `<TouchableOpacity>`, `<Pressable>` | Nút bấm |
| `<a>` | `<TouchableOpacity>` + Navigation | Không có link mặc định |
| `<ul>`, `<ol>` + `<li>` | `<FlatList>`, `<SectionList>` | Danh sách tối ưu |
| `<select>` | `<Picker>` (thư viện riêng) | Dropdown |
| `<textarea>` | `<TextInput multiline>` | Nhiều dòng |
| `<form>` | Không có | Dùng state thủ công |
| `<iframe>` | `<WebView>` (thư viện riêng) | Nhúng web |
| `<svg>` | `react-native-svg` (thư viện) | Cần cài thêm |
| `<ScrollView>` | `<ScrollView>` | Cuộn nội dung |
| `<SafeAreaView>` | `<SafeAreaView>` | Tránh vùng notch/camera |

### 5.2 Ví dụ: Màn hình đăng nhập

```jsx
// ❌ React Web – dùng HTML
function LoginWeb() {
  return (
    <div className="container">
      <h1>Đăng nhập</h1>
      <input type="email" placeholder="Email" />
      <input type="password" placeholder="Mật khẩu" />
      <button onClick={handleLogin}>Đăng nhập</button>
    </div>
  );
}

// ✅ React Native – dùng Core Components
import { View, Text, TextInput, TouchableOpacity, StyleSheet, KeyboardAvoidingView, Platform } from 'react-native';

function LoginScreen() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  return (
    <KeyboardAvoidingView
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
      style={styles.container}
    >
      <Text style={styles.title}>Đăng nhập</Text>

      <TextInput
        style={styles.input}
        placeholder="Email"
        value={email}
        onChangeText={setEmail}
        keyboardType="email-address"
        autoCapitalize="none"
      />

      <TextInput
        style={styles.input}
        placeholder="Mật khẩu"
        value={password}
        onChangeText={setPassword}
        secureTextEntry={true}
      />

      <TouchableOpacity style={styles.button} onPress={handleLogin}>
        <Text style={styles.buttonText}>Đăng nhập</Text>
      </TouchableOpacity>
    </KeyboardAvoidingView>
  );
}
```

### 5.3 Quy tắc quan trọng: Text phải trong `<Text>`

```jsx
// ❌ SAI – Text nằm ngoài <Text> → crash app
<View>
  Xin chào người dùng
</View>

// ✅ ĐÚNG
<View>
  <Text>Xin chào người dùng</Text>
</View>
```

### 5.4 Component `<Image>`

```jsx
import { Image } from 'react-native';

// Ảnh remote
<Image
  source={{ uri: 'https://example.com/avatar.jpg' }}
  style={{ width: 100, height: 100, borderRadius: 50 }}
/>

// Ảnh local (assets)
<Image
  source={require('./assets/logo.png')}
  style={{ width: 200, height: 80 }}
  resizeMode="contain"   // cover | contain | stretch | center
/>
```

### 5.5 Pressable – Component nút bấm hiện đại

```jsx
import { Pressable, Text } from 'react-native';

<Pressable
  onPress={() => console.log('Bấm')}
  onLongPress={() => console.log('Giữ')}
  style={({ pressed }) => [
    styles.button,
    pressed && { opacity: 0.7 }
  ]}
>
  {({ pressed }) => (
    <Text>{pressed ? 'Đang bấm...' : 'Bấm vào đây'}</Text>
  )}
</Pressable>
```

---

## 6. StyleSheet – Thay thế CSS

React Native **không dùng CSS**. Styling được định nghĩa bằng **JavaScript object** thông qua `StyleSheet.create()`.

### 6.1 Cú pháp cơ bản

```jsx
import { StyleSheet, View, Text } from 'react-native';

function Card() {
  return (
    <View style={styles.card}>
      <Text style={styles.title}>Tiêu đề</Text>
      <Text style={[styles.text, styles.highlighted]}>
        Nội dung kết hợp nhiều style (dùng mảng)
      </Text>
    </View>
  );
}

const styles = StyleSheet.create({
  card: {
    backgroundColor: '#ffffff',
    borderRadius: 12,
    padding: 16,
    marginHorizontal: 20,
    // Shadow cho iOS
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    // Shadow cho Android
    elevation: 4,
  },
  title: {
    fontSize: 18,
    fontWeight: 'bold',
    color: '#1a1a1a',
    marginBottom: 8,
  },
  text: {
    fontSize: 14,
    color: '#666',
    lineHeight: 22,
  },
  highlighted: {
    color: '#007AFF',
    fontWeight: '600',
  },
});
```

> **Lưu ý:** `StyleSheet.create()` không chỉ là convention mà còn tối ưu hiệu năng: React Native gửi style IDs thay vì toàn bộ object sang native thread, giảm tải bridge đáng kể.

### 6.2 Những khác biệt quan trọng so với CSS

| CSS (Web) | React Native | Lưu ý |
|-----------|-------------|-------|
| `background-color` | `backgroundColor` | camelCase |
| `font-size: 16px` | `fontSize: 16` | Không có đơn vị px |
| `border-radius: 8px` | `borderRadius: 8` | Không có đơn vị |
| `margin: 10px 20px` | `marginVertical: 10, marginHorizontal: 20` | Không có shorthand |
| `padding: 10px 20px` | `paddingVertical: 10, paddingHorizontal: 20` | Không có shorthand |
| `display: flex` | Mặc định (không cần khai báo) | Mọi View đều là Flexbox |
| `display: block/inline` | Không tồn tại | Chỉ có flex |
| `position: fixed` | Không tồn tại | Chỉ có `absolute` và `relative` |
| `box-shadow` | `shadowColor/Offset/Opacity/Radius` + `elevation` | iOS và Android khác nhau |
| `overflow: hidden` | `overflow: 'hidden'` | String, không phải keyword |
| `:hover`, `:focus` | Không tồn tại | Dùng sự kiện touch |
| `%` width/height | Chỉ hoạt động với width/height | Dùng Dimensions hoặc flex |
| `em`, `rem` | Không tồn tại | Chỉ có số tuyệt đối |
| Kế thừa font | KHÔNG tự động kế thừa từ cha | Ngoại trừ `<Text>` lồng nhau |

### 6.3 Lấy kích thước màn hình

```jsx
import { Dimensions, useWindowDimensions } from 'react-native';

// Cách 1: Static (không cập nhật khi xoay màn hình)
const { width, height } = Dimensions.get('window');

// Cách 2: Hook – tự cập nhật khi xoay (khuyến nghị)
function ResponsiveComponent() {
  const { width, height } = useWindowDimensions();

  return (
    <View style={{ width: width * 0.9, height: height * 0.5 }}>
      <Text>Chiều rộng: {width}px</Text>
    </View>
  );
}
```

### 6.4 Không có CSS classes hay global styles

```jsx
// ❌ Không làm được trong React Native
// .button { color: red } – CSS class toàn cục

// ✅ Phải dùng StyleSheet và truyền qua props
const styles = StyleSheet.create({
  button: { backgroundColor: 'red' }
});

// Hoặc dùng thư viện như NativeWind (Tailwind cho RN)
// <View className="bg-red-500 p-4 rounded-lg">
```

---

## 7. Flexbox trong React Native

React Native dùng **Flexbox** làm hệ thống layout duy nhất (không có Grid, Float, ...).

### 7.1 Khác biệt mặc định với CSS Flexbox

| Thuộc tính | CSS Web | React Native |
|-----------|---------|-------------|
| `flexDirection` | `row` | **`column`** ← khác! |
| `alignContent` | `stretch` | `flex-start` |
| `flexShrink` | `1` | `0` |

> **Quan trọng:** Trong React Native, các phần tử xếp **dọc theo cột** mặc định, không phải theo hàng như web.

### 7.2 Ví dụ Layout

```jsx
// Layout cột (mặc định)
<View style={{ flex: 1, flexDirection: 'column' }}>
  <View style={{ height: 60, backgroundColor: 'blue' }} />   {/* Header */}
  <View style={{ flex: 1, backgroundColor: 'white' }} />     {/* Body */}
  <View style={{ height: 50, backgroundColor: 'gray' }} />   {/* Footer */}
</View>

// Layout hàng
<View style={{ flexDirection: 'row', justifyContent: 'space-between' }}>
  <View style={{ flex: 1, backgroundColor: 'red' }} />
  <View style={{ flex: 2, backgroundColor: 'green' }} />     {/* Chiếm 2/3 */}
  <View style={{ flex: 1, backgroundColor: 'blue' }} />
</View>

// Căn giữa (phổ biến nhất)
<View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
  <Text>Nội dung ở giữa màn hình</Text>
</View>
```

### 7.3 SafeAreaView – Tránh vùng notch và home indicator

```jsx
import { SafeAreaView } from 'react-native-safe-area-context';
// hoặc
import { SafeAreaView } from 'react-native';

function App() {
  return (
    <SafeAreaView style={{ flex: 1, backgroundColor: '#fff' }}>
      <Text>Nội dung hiển thị an toàn trên mọi thiết bị</Text>
    </SafeAreaView>
  );
}
```

---

## 8. Xử lý sự kiện và Touch

React Native không có các sự kiện chuột (`onClick`, `onMouseOver`, ...) vì mobile dùng ngón tay.

### 8.1 So sánh sự kiện

| Web Event | React Native Event | Mô tả |
|-----------|-------------------|-------|
| `onClick` | `onPress` | Chạm vào |
| `onDoubleClick` | `onPress` (đếm thủ công) | Chạm đôi |
| `onMouseEnter` | Không có | Không có hover trên mobile |
| `onChange` (input) | `onChangeText` | Text thay đổi |
| `onSubmit` (form) | Không có | Tự xử lý bằng button |
| `onScroll` | `onScroll` | Tương tự |
| `onKeyDown` | `onKeyPress` | Bàn phím |
| `onFocus` | `onFocus` | Giống nhau |

### 8.2 Ví dụ xử lý Touch

```jsx
import { TouchableOpacity, Pressable } from 'react-native';

// TouchableOpacity – phổ biến nhất, giảm opacity khi bấm
<TouchableOpacity
  onPress={() => console.log('Bấm')}
  onLongPress={() => console.log('Giữ')}
  activeOpacity={0.7}
  delayLongPress={500}
>
  <Text>Bấm vào tôi</Text>
</TouchableOpacity>

// Pressable – hiện đại nhất, nhiều tùy chọn nhất
<Pressable
  onPress={handlePress}
  onPressIn={() => console.log('Ngón tay đặt xuống')}
  onPressOut={() => console.log('Ngón tay nhấc lên')}
  hitSlop={{ top: 10, bottom: 10, left: 10, right: 10 }}
>
  <Text>Nút bấm</Text>
</Pressable>
```

### 8.3 TextInput – Ô nhập liệu

```jsx
import { TextInput } from 'react-native';

<TextInput
  value={text}
  onChangeText={setText}
  placeholder="Nhập nội dung..."
  placeholderTextColor="#999"

  // Loại bàn phím
  keyboardType="numeric"              // default|numeric|email-address|phone-pad|decimal-pad|url

  // Hành vi return key
  returnKeyType="done"
  onSubmitEditing={handleSubmit}

  // Nhiều dòng
  multiline={true}
  numberOfLines={4}

  // Bảo mật
  secureTextEntry={true}

  // Auto behavior
  autoCorrect={false}
  autoCapitalize="none"

  style={styles.input}
/>
```

---

## 9. Navigation – Điều hướng màn hình

React web dùng **URL routing** (React Router, Next.js Router). React Native **không có URL hay trình duyệt**, nên phải dùng thư viện điều hướng riêng. Thư viện phổ biến nhất là **React Navigation v7**.

### 9.1 Cài đặt React Navigation

```bash
npm install @react-navigation/native
npm install @react-navigation/native-stack
npm install react-native-screens react-native-safe-area-context
```

### 9.2 Stack Navigator

```jsx
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator();

function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator initialRouteName="Home">
        <Stack.Screen name="Home" component={HomeScreen} options={{ title: 'Trang chủ' }} />
        <Stack.Screen name="Detail" component={DetailScreen} options={{ title: 'Chi tiết' }} />
        <Stack.Screen name="Profile" component={ProfileScreen} options={{ headerShown: false }} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

### 9.3 Điều hướng giữa các màn hình

```jsx
// HomeScreen.js
function HomeScreen({ navigation, route }) {
  return (
    <View>
      <Button
        title="Xem chi tiết"
        onPress={() => navigation.navigate('Detail', {
          itemId: 42,
          itemName: 'Sản phẩm A'
        })}
      />
      <Button title="Quay lại" onPress={() => navigation.goBack()} />
      <Button
        title="Về trang chủ"
        onPress={() => navigation.reset({ index: 0, routes: [{ name: 'Home' }] })}
      />
    </View>
  );
}

// DetailScreen.js
function DetailScreen({ navigation, route }) {
  const { itemId, itemName } = route.params;
  return (
    <View>
      <Text>ID: {itemId}</Text>
      <Text>Tên: {itemName}</Text>
    </View>
  );
}
```

### 9.4 Tab Navigator

```jsx
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';

const Tab = createBottomTabNavigator();

function MainTabs() {
  return (
    <Tab.Navigator
      screenOptions={({ route }) => ({
        tabBarIcon: ({ focused, color, size }) => {
          const iconName = route.name === 'Home' ? '🏠' : '👤';
          return <Text style={{ fontSize: size }}>{iconName}</Text>;
        },
        tabBarActiveTintColor: '#007AFF',
        tabBarInactiveTintColor: 'gray',
      })}
    >
      <Tab.Screen name="Home" component={HomeScreen} />
      <Tab.Screen name="Profile" component={ProfileScreen} />
    </Tab.Navigator>
  );
}
```

### 9.5 Drawer Navigator

```jsx
import { createDrawerNavigator } from '@react-navigation/drawer';

const Drawer = createDrawerNavigator();

function DrawerNav() {
  return (
    <Drawer.Navigator>
      <Drawer.Screen name="Home" component={HomeScreen} />
      <Drawer.Screen name="Settings" component={SettingsScreen} />
    </Drawer.Navigator>
  );
}

// Mở menu drawer
navigation.openDrawer();
navigation.closeDrawer();
```

### 9.6 So sánh React Router Web vs React Navigation

| | React Router (Web) | React Navigation (Mobile) |
|-|-------------------|-----------------------------|
| Điều hướng | URL (`/home`, `/detail/1`) | Tên màn hình (`'Home'`, `'Detail'`) |
| Truyền data | URL params, Query string | `route.params` |
| Quay lại | Nút Back trình duyệt | `navigation.goBack()` |
| Lịch sử | Browser history | Navigation stack |
| Link | `<Link to="/about">` | `navigation.navigate('About')` |

> **Lưu ý:** Deep Linking (mở app từ URL như `myapp://product/123`) cần cấu hình `linking` config trong `NavigationContainer`. Đây là tính năng quan trọng để tích hợp notification và marketing campaigns.

---

## 10. Platform API – Truy cập tính năng thiết bị

React Native cung cấp API để nhận biết và phân biệt giữa **iOS và Android**.

### 10.1 Platform module

```jsx
import { Platform, StyleSheet } from 'react-native';

console.log(Platform.OS);         // 'ios' | 'android' | 'web'
console.log(Platform.Version);    // iOS: '17.0' | Android: API level
console.log(Platform.isPad);      // true nếu là iPad

// Platform.select
const containerStyle = {
  paddingTop: Platform.select({
    ios: 44,
    android: 24,
    default: 0,
  }),
};

// Trong StyleSheet
const styles = StyleSheet.create({
  button: {
    ...Platform.select({
      ios: {
        shadowColor: '#000',
        shadowOffset: { width: 0, height: 2 },
        shadowOpacity: 0.3,
      },
      android: {
        elevation: 5,
      },
    }),
  },
});
```

### 10.2 Platform-specific files

Tạo file riêng cho từng nền tảng – React Native tự chọn đúng file:

```
Button.ios.js       ← Dùng cho iOS
Button.android.js   ← Dùng cho Android
Button.js           ← Fallback
```

```jsx
// DatePicker.ios.js
import DateTimePicker from '@react-native-community/datetimepicker';
export default function DatePicker({ value, onChange }) {
  return <DateTimePicker value={value} mode="date" display="spinner" onChange={onChange} />;
}

// DatePicker.android.js
import DateTimePicker from '@react-native-community/datetimepicker';
export default function DatePicker({ value, onChange }) {
  return <DateTimePicker value={value} mode="date" display="default" onChange={onChange} />;
}

// Import – Metro tự chọn đúng file
import DatePicker from './DatePicker';
```

### 10.3 StatusBar

```jsx
import { StatusBar } from 'react-native';

function App() {
  return (
    <>
      <StatusBar
        barStyle="dark-content"
        backgroundColor="#ffffff"
        translucent={true}
        hidden={false}
      />
    </>
  );
}
```

---

## 11. AsyncStorage và Lưu trữ cục bộ

React web dùng `localStorage` / `sessionStorage`. React Native **không có** các API này. Thay vào đó dùng **AsyncStorage** – key-value store bất đồng bộ, Promise-based.

### 11.1 Cài đặt và sử dụng

```bash
npm install @react-native-async-storage/async-storage
```

```jsx
import AsyncStorage from '@react-native-async-storage/async-storage';

// Lưu dữ liệu (phải stringify object)
const saveUser = async (user) => {
  try {
    await AsyncStorage.setItem('user', JSON.stringify(user));
  } catch (error) {
    console.error('Lỗi khi lưu:', error);
  }
};

// Đọc dữ liệu
const loadUser = async () => {
  try {
    const value = await AsyncStorage.getItem('user');
    return value ? JSON.parse(value) : null;
  } catch (error) {
    console.error('Lỗi khi đọc:', error);
    return null;
  }
};

// Xóa dữ liệu
const removeUser = async () => {
  await AsyncStorage.removeItem('user');
};

// Xóa nhiều key cùng lúc
const logout = async () => {
  await AsyncStorage.multiRemove(['authToken', 'userId', 'userProfile']);
};
```

### 11.2 Hook quản lý token đăng nhập

```jsx
import AsyncStorage from '@react-native-async-storage/async-storage';
import { useState, useEffect } from 'react';

function useAuthToken() {
  const [token, setToken] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    AsyncStorage.getItem('auth_token').then(value => {
      setToken(value);
      setLoading(false);
    });
  }, []);

  const saveToken = async (newToken) => {
    await AsyncStorage.setItem('auth_token', newToken);
    setToken(newToken);
  };

  const removeToken = async () => {
    await AsyncStorage.removeItem('auth_token');
    setToken(null);
  };

  return { token, loading, saveToken, removeToken };
}
```

### 11.3 So sánh lưu trữ

| | Web | React Native |
|-|-----|-------------|
| **Key-value đơn giản** | `localStorage` | `AsyncStorage` |
| **Session** | `sessionStorage` | Không có (dùng state) |
| **Database** | IndexedDB | SQLite (`expo-sqlite`) |
| **Bất đồng bộ** | ❌ Đồng bộ | ✅ Luôn async |
| **Giới hạn** | ~5–10MB | ~6MB (iOS) / không giới hạn (Android) |

### 11.4 AppState và NetInfo

React Native cung cấp API riêng để theo dõi trạng thái ứng dụng (foreground/background) và kết nối mạng.

```jsx
import { AppState } from 'react-native';
import NetInfo from '@react-native-community/netinfo';

// Theo dõi app vào nền / lên nền
useEffect(() => {
  const subscription = AppState.addEventListener('change', (nextState) => {
    if (nextState === 'active') {
      fetchLatestData();
    } else if (nextState === 'background') {
      saveCurrentState();
    }
  });
  return () => subscription.remove();
}, []);

// Theo dõi kết nối mạng
useEffect(() => {
  const unsubscribe = NetInfo.addEventListener((state) => {
    if (!state.isConnected) {
      showOfflineBanner();
    } else {
      syncPendingData();
    }
  });
  return () => unsubscribe();
}, []);
```

---

## 12. Animated API và Gestures – Hoạt ảnh

React web dùng CSS transitions/animations. React Native có **Animated API** riêng, chạy trên **Native Thread** để animation mượt mà.

### 12.1 Animated API cơ bản

```jsx
import { Animated, TouchableOpacity, Text } from 'react-native';
import { useRef } from 'react';

function FadeInView() {
  const fadeAnim = useRef(new Animated.Value(0)).current;

  const fadeIn = () => {
    Animated.timing(fadeAnim, {
      toValue: 1,
      duration: 500,
      useNativeDriver: true,  // Chạy trên native thread → mượt hơn
    }).start();
  };

  return (
    <Animated.View style={{ opacity: fadeAnim }}>
      <Text>Nội dung fade in</Text>
      <TouchableOpacity onPress={fadeIn}>
        <Text>Hiện lên</Text>
      </TouchableOpacity>
    </Animated.View>
  );
}
```

### 12.2 Các loại Animation

```jsx
// 1. timing
Animated.timing(value, {
  toValue: 1,
  duration: 300,
  easing: Easing.ease,
  useNativeDriver: true,
}).start();

// 2. spring
Animated.spring(value, {
  toValue: 1,
  tension: 40,
  friction: 7,
  useNativeDriver: true,
}).start();

// 3. sequence – chạy tuần tự
Animated.sequence([
  Animated.timing(anim1, { toValue: 1, duration: 300, useNativeDriver: true }),
  Animated.timing(anim2, { toValue: 1, duration: 300, useNativeDriver: true }),
]).start();

// 4. parallel – chạy cùng lúc
Animated.parallel([
  Animated.timing(fadeAnim, { toValue: 1, duration: 500, useNativeDriver: true }),
  Animated.timing(slideAnim, { toValue: 0, duration: 500, useNativeDriver: true }),
]).start();

// 5. loop
Animated.loop(
  Animated.timing(spinAnim, { toValue: 1, duration: 1000, useNativeDriver: true })
).start();
```

> **Lưu ý:** `useNativeDriver: true` chỉ hỗ trợ `transform` và `opacity`, không hỗ trợ layout properties như `width` hay `height`.

### 12.3 React Native Reanimated v3 (khuyến nghị cho production)

Reanimated v3 chạy animation và gesture handling hoàn toàn trên UI thread (worklet), loại bỏ độ trễ bridge.

```bash
npm install react-native-reanimated react-native-gesture-handler
```

```jsx
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
} from 'react-native-reanimated';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';

function DraggableCard() {
  const offsetX = useSharedValue(0);
  const offsetY = useSharedValue(0);

  const panGesture = Gesture.Pan()
    .onUpdate((event) => {
      offsetX.value = event.translationX;
      offsetY.value = event.translationY;
    })
    .onEnd(() => {
      offsetX.value = withSpring(0);
      offsetY.value = withSpring(0);
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: offsetX.value },
      { translateY: offsetY.value },
    ],
  }));

  return (
    <GestureDetector gesture={panGesture}>
      <Animated.View style={[styles.card, animatedStyle]}>
        <Text>Kéo tôi!</Text>
      </Animated.View>
    </GestureDetector>
  );
}
```

---

## 13. FlatList và SectionList – Danh sách hiệu năng cao

React web dùng `.map()` để render danh sách. React Native có **FlatList** và **SectionList** với cơ chế **virtualization**: chỉ render các item trong viewport.

### 13.1 FlatList

```jsx
import { FlatList, View, Text } from 'react-native';

const DATA = Array.from({ length: 1000 }, (_, i) => ({
  id: String(i),
  title: `Item số ${i + 1}`,
}));

function MyList() {
  const renderItem = useCallback(({ item }) => (
    <View style={styles.item}>
      <Text>{item.title}</Text>
    </View>
  ), []);

  const keyExtractor = useCallback((item) => item.id, []);

  return (
    <FlatList
      data={DATA}
      renderItem={renderItem}
      keyExtractor={keyExtractor}

      initialNumToRender={10}
      maxToRenderPerBatch={10}
      windowSize={5}
      removeClippedSubviews={true}

      getItemLayout={(data, index) => ({
        length: ITEM_HEIGHT,
        offset: ITEM_HEIGHT * index,
        index,
      })}

      ItemSeparatorComponent={() => <View style={{ height: 1, backgroundColor: '#eee' }} />}
      ListHeaderComponent={<Text>Danh sách</Text>}
      ListEmptyComponent={<Text>Không có dữ liệu</Text>}

      onEndReached={loadMoreData}
      onEndReachedThreshold={0.1}

      refreshing={refreshing}
      onRefresh={handleRefresh}
    />
  );
}
```

> **Quy tắc bắt buộc:** Không dùng `.map()` trong `<ScrollView>` cho danh sách lớn. Luôn dùng `FlatList`.

### 13.2 SectionList – Danh sách có nhóm

```jsx
import { SectionList, View, Text } from 'react-native';

const SECTIONS = [
  { title: 'Rau củ', data: ['Cà rốt', 'Bắp cải', 'Cải thìa'] },
  { title: 'Trái cây', data: ['Táo', 'Xoài', 'Chuối'] },
];

function GroupedList() {
  return (
    <SectionList
      sections={SECTIONS}
      keyExtractor={(item, index) => item + index}
      renderItem={({ item }) => (
        <View style={styles.item}><Text>{item}</Text></View>
      )}
      renderSectionHeader={({ section }) => (
        <View style={styles.sectionHeader}>
          <Text style={styles.sectionTitle}>{section.title}</Text>
        </View>
      )}
    />
  );
}
```

### 13.3 So sánh cách render danh sách

| | React Web | React Native |
|-|-----------|-------------|
| Cách thông thường | `array.map()` | `<FlatList>` |
| Virtualization | ❌ Render tất cả | ✅ Chỉ render items hiển thị |
| 10,000 items | Lag nặng | Mượt mà |
| Key | `key` prop | `keyExtractor` prop |
| Pull to refresh | Custom | `refreshing` + `onRefresh` |
| Infinite scroll | Custom | `onEndReached` |

---

## 14. Modal, Alert, và Overlay

### 14.1 Alert – Hộp thoại hệ thống

React Native có API `Alert` trực tiếp gọi **hộp thoại native** của iOS/Android.

```jsx
import { Alert } from 'react-native';

// Alert đơn giản
Alert.alert('Thông báo', 'Thao tác thành công!');

// Alert với nhiều nút
Alert.alert(
  'Xác nhận',
  'Bạn có chắc muốn xóa không?',
  [
    { text: 'Hủy', style: 'cancel', onPress: () => console.log('Hủy') },
    { text: 'Xóa', style: 'destructive', onPress: () => handleDelete() },
  ],
  { cancelable: true }
);

// Prompt (nhập liệu) – chỉ iOS
Alert.prompt('Nhập tên', 'Nhập tên của bạn:', (text) => console.log('Tên:', text));
```

### 14.2 Modal

```jsx
import { Modal, View, Text, TouchableOpacity } from 'react-native';
import { useState } from 'react';

function ModalExample() {
  const [visible, setVisible] = useState(false);

  return (
    <View>
      <TouchableOpacity onPress={() => setVisible(true)}>
        <Text>Mở Modal</Text>
      </TouchableOpacity>

      <Modal
        visible={visible}
        transparent={true}
        animationType="slide"
        onRequestClose={() => setVisible(false)}
        statusBarTranslucent={true}
      >
        <View style={styles.overlay}>
          <View style={styles.modalBox}>
            <Text style={styles.modalTitle}>Tiêu đề Modal</Text>
            <TouchableOpacity onPress={() => setVisible(false)} style={styles.closeButton}>
              <Text>Đóng</Text>
            </TouchableOpacity>
          </View>
        </View>
      </Modal>
    </View>
  );
}

const styles = StyleSheet.create({
  overlay: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.5)',
    justifyContent: 'center',
    alignItems: 'center',
  },
  modalBox: {
    backgroundColor: '#fff',
    borderRadius: 16,
    padding: 24,
    width: '80%',
  },
});
```

---

## 15. Permissions và Expo APIs

Truy cập phần cứng (camera, GPS, microphone, ...) trên mobile **bắt buộc phải xin quyền** từ người dùng.

### 15.1 Expo Permissions

```jsx
import * as Location from 'expo-location';
import * as ImagePicker from 'expo-image-picker';
import * as Camera from 'expo-camera';

// Vị trí GPS
async function getLocation() {
  const { status } = await Location.requestForegroundPermissionsAsync();
  if (status !== 'granted') {
    Alert.alert('Quyền bị từ chối', 'Cần quyền vị trí để dùng tính năng này');
    return;
  }
  const location = await Location.getCurrentPositionAsync({
    accuracy: Location.Accuracy.High,
  });
  console.log(location.coords.latitude, location.coords.longitude);
}

// Chọn ảnh từ thư viện
async function pickImage() {
  const { status } = await ImagePicker.requestMediaLibraryPermissionsAsync();
  if (status !== 'granted') return;

  const result = await ImagePicker.launchImageLibraryAsync({
    mediaTypes: ImagePicker.MediaTypeOptions.Images,
    allowsEditing: true,
    aspect: [4, 3],
    quality: 0.8,
  });

  if (!result.canceled) {
    console.log('Đường dẫn ảnh:', result.assets[0].uri);
  }
}

// Chụp ảnh
async function takePhoto() {
  const { status } = await Camera.requestCameraPermissionsAsync();
  if (status !== 'granted') return;

  const result = await ImagePicker.launchCameraAsync({
    allowsEditing: true,
    quality: 1,
  });

  if (!result.canceled) {
    console.log('Ảnh chụp:', result.assets[0].uri);
  }
}
```

### 15.2 Expo APIs thông dụng

| API | Package | Chức năng |
|-----|---------|-----------|
| **Location** | `expo-location` | GPS, địa chỉ |
| **Camera** | `expo-camera` | Chụp ảnh, quét QR |
| **ImagePicker** | `expo-image-picker` | Chọn ảnh/video |
| **Notifications** | `expo-notifications` | Push notification |
| **SecureStore** | `expo-secure-store` | Lưu trữ bảo mật |
| **FileSystem** | `expo-file-system` | Đọc/ghi file |
| **Haptics** | `expo-haptics` | Rung thiết bị |
| **Sensors** | `expo-sensors` | Gyroscope, accelerometer |
| **Barcode** | `expo-barcode-scanner` | Quét mã vạch |
| **Audio** | `expo-av` | Phát âm thanh, ghi âm |

---

## 16. Fonts và Assets

Quản lý fonts và assets trong React Native yêu cầu các bước cấu hình thủ công. Không có `@font-face` CSS hay link CDN.

### 16.1 Cài đặt custom fonts

```
// Bước 1: Đặt file font vào thư mục assets/fonts/
// assets/fonts/
//   Inter-Regular.ttf
//   Inter-Bold.ttf

// Bước 2: Khai báo trong react-native.config.js
module.exports = {
  assets: ['./assets/fonts/'],
};

// Bước 3: Link asset (React Native CLI)
// npx react-native-asset

// Bước 4: Dùng trong StyleSheet
const styles = StyleSheet.create({
  title: {
    fontFamily: 'Inter-Bold', // Tên file KHÔNG có extension
    fontSize: 24,
  },
  body: {
    fontFamily: 'Inter-Regular',
    fontSize: 16,
  }
});
```

### 16.2 Images và density màn hình

Thiết bị di động có nhiều mật độ điểm ảnh khác nhau. Metro hỗ trợ ảnh đa độ phân giải tự động.

```
// assets/images/
//   logo.png       (1x)
//   logo@2x.png    (2x – Retina)
//   logo@3x.png    (3x – Super Retina)
```

```jsx
import { Image, Dimensions } from 'react-native';

const { width: SCREEN_WIDTH } = Dimensions.get('window');

function Logo() {
  return (
    <Image
      source={require('../assets/images/logo.png')}
      style={{
        width: SCREEN_WIDTH * 0.4,
        height: undefined,
        aspectRatio: 1,
        resizeMode: 'contain',
      }}
    />
  );
}

// Ảnh remote – BẮT BUỘC chỉ định width và height
function Avatar({ uri }) {
  return (
    <Image
      source={{ uri }}
      style={{ width: 48, height: 48, borderRadius: 24 }}
    />
  );
}
```

---

## 17. Hiệu năng và Tối ưu ứng dụng

Kiến trúc React Native gồm hai thread song song: JavaScript thread (logic) và Native/UI thread (render). Tắc nghẽn JS thread sẽ làm giao diện bị giật.

### 17.1 FlatList thay vì map()

```jsx
// ❌ SAI: Render toàn bộ item cùng lúc
function ProductListBad({ products }) {
  return (
    <ScrollView>
      {products.map(item => (
        <ProductCard key={item.id} product={item} />
      ))}
    </ScrollView>
  );
}

// ✅ ĐÚNG: FlatList chỉ render item đang hiển thị
function ProductListGood({ products }) {
  const renderItem = useCallback(({ item }) => (
    <ProductCard product={item} />
  ), []);

  const keyExtractor = useCallback((item) => String(item.id), []);

  return (
    <FlatList
      data={products}
      renderItem={renderItem}
      keyExtractor={keyExtractor}
      initialNumToRender={10}
      maxToRenderPerBatch={10}
      windowSize={5}
      removeClippedSubviews={true}
      getItemLayout={(data, index) => ({
        length: ITEM_HEIGHT,
        offset: ITEM_HEIGHT * index,
        index,
      })}
    />
  );
}
```

### 17.2 Memo hóa và tránh re-render

```jsx
import React, { memo, useCallback, useMemo } from 'react';

// memo(): QUAN TRỌNG hơn trong RN vì re-render tốn kém hơn (bridge overhead)
const ProductCard = memo(({ product, onPress }) => {
  return (
    <TouchableOpacity onPress={() => onPress(product.id)}>
      <Text>{product.name}</Text>
    </TouchableOpacity>
  );
});

function ProductScreen() {
  const [cart, setCart] = useState([]);

  const totalPrice = useMemo(() => {
    return cart.reduce((sum, item) => sum + item.price, 0);
  }, [cart]);

  const handleAddToCart = useCallback((productId) => {
    setCart(prev => [...prev, { id: productId }]);
  }, []);

  return (...);
}
```

---

## 18. Debugging và Công cụ phát triển

### 18.1 So sánh công cụ debug

| | React Web | React Native |
|-|-----------|-------------|
| DevTools | Browser DevTools | Flipper / Expo DevTools |
| Console | Browser Console | Metro bundler console |
| Network | Network tab | Flipper Network plugin |
| Layout | Elements tab | React DevTools |
| Hot Reload | ✅ HMR | ✅ Fast Refresh |
| Remote debug | Không cần | Chrome DevTools (qua USB) |

### 18.2 Developer Menu

Trên thiết bị/emulator, lắc điện thoại hoặc nhấn `Cmd+D` (iOS) / `Cmd+M` (Android) để mở Developer Menu:

- **Reload** → Reload lại app
- **Open Debugger** → Mở Chrome DevTools
- **Show Element Inspector** → Inspect UI như browser
- **Performance Monitor** → Xem FPS, JS thread
- **Enable Fast Refresh** → Hot reload

### 18.3 Console.log trong React Native

```jsx
console.log('Debug:', someValue);
console.warn('Cảnh báo');       // Màu vàng trong Metro
console.error('Lỗi');           // Màu đỏ + Red Box trên simulator

// LogBox – kiểm soát warning/error hiển thị
import { LogBox } from 'react-native';
LogBox.ignoreLogs(['Warning: ...']);
```

---

## 19. Lộ trình học tập đề xuất

Dựa trên các nội dung đã trình bày, lộ trình học React Native cho lập trình viên đã biết React:

| Giai đoạn | Nội dung | Thời gian ước tính |
|-----------|----------|-------------------|
| **Giai đoạn 1 (Cốt lõi)** | Core Components, StyleSheet & Flexbox, React Navigation (Stack + Tabs), AsyncStorage, FlatList | 2–3 tuần |
| **Giai đoạn 2 (Nâng cao)** | Platform APIs, Camera & Permissions, Push Notifications, Animations (Animated API), Build & Deploy (Expo EAS) | 3–4 tuần |
| **Giai đoạn 3 (Chuyên sâu)** | Reanimated v3 & Gesture Handler, JSI / New Architecture, Native Modules (Kotlin/Swift), CI/CD pipeline, Performance profiling | 4–6 tuần |

Điểm mạnh của lập trình viên React chuyển sang React Native nằm ở việc toàn bộ kiến thức về hooks, state management, data fetching, và tư duy component-based đều được áp dụng trực tiếp. Phần cần đầu tư chủ yếu tập trung vào lớp rendering (Native Components thay HTML), hệ thống Navigation và môi trường build.

---

## 20. Bảng so sánh tổng hợp React vs React Native

| Khía cạnh | React (Web) | React Native (Mobile) |
|----------|------------|----------------------|
| **Render output** | HTML DOM | Native Views (UIView / android.view) |
| **UI elements** | `<div>`, `<span>`, `<input>`, ... | `<View>`, `<Text>`, `<TextInput>`, ... |
| **Styling** | CSS / SCSS / Tailwind | StyleSheet object (JavaScript) |
| **Layout** | CSS Flexbox + Grid | Flexbox (column-first) |
| **Units** | px, em, rem, % | Numbers (dp/pt tự động) |
| **Routing** | React Router / Next.js | React Navigation |
| **Storage** | localStorage, sessionStorage | AsyncStorage |
| **Sự kiện** | onClick, onChange, onMouseOver | onPress, onChangeText |
| **Ảnh** | `<img src="">` | `<Image source={{uri:''}} />` |
| **Shadow** | `box-shadow` CSS | shadowColor/Offset + elevation |
| **Animation** | CSS transitions, Framer Motion | Animated API, Reanimated |
| **Danh sách** | `array.map()` | `<FlatList>` (virtualized) |
| **API access** | Fetch/Axios + Web APIs | Fetch/Axios + Native APIs |
| **Permissions** | Browser permissions API | iOS/Android system permissions |
| **Debug tools** | Browser DevTools | Flipper, Metro, Expo DevTools |
| **Build** | Webpack/Vite → HTML/JS/CSS | Metro → APK/IPA |
| **Hot Reload** | ✅ HMR | ✅ Fast Refresh |

---

## 21. Checklist chuyển từ React sang React Native

```
COMPONENTS
☐ Thay <div> → <View>
☐ Thay <p>, <span>, <h1>... → <Text> (bắt buộc cho mọi text)
☐ Thay <img> → <Image source={{}} />
☐ Thay <input> → <TextInput onChangeText={} />
☐ Thay <button> → <TouchableOpacity> hoặc <Pressable>
☐ Thay <ul>/<li> → <FlatList> (danh sách dài)

STYLING
☐ Thay CSS file → StyleSheet.create({})
☐ Thay className → style prop
☐ Thay kebab-case → camelCase (font-size → fontSize)
☐ Bỏ đơn vị px (16px → 16)
☐ Thay shorthand margin/padding → marginVertical/Horizontal
☐ Thay box-shadow CSS → shadow* (iOS) + elevation (Android)

EVENTS
☐ Thay onClick → onPress
☐ Thay onChange (input) → onChangeText
☐ Bỏ hover/focus CSS states → dùng state JS

NAVIGATION
☐ Thay React Router → React Navigation
☐ Thay <Link to="/"> → navigation.navigate('Screen')
☐ Thay URL params → route.params

STORAGE & API
☐ Thay localStorage → AsyncStorage (async/await)
☐ Thêm xin quyền trước khi dùng camera/location/...

LAYOUT
☐ Nhớ flexDirection mặc định là 'column' (không phải 'row')
☐ Bọc toàn bộ app trong <SafeAreaView>
☐ Dùng useWindowDimensions() thay vì % kích thước
```

---

## Tài liệu tham khảo

- React Native Official Docs: https://reactnative.dev/docs/getting-started
- Expo Documentation: https://docs.expo.dev
- React Navigation: https://reactnavigation.org/docs/getting-started
- React Native New Architecture: https://reactnative.dev/docs/the-new-architecture/landing-page
- Animated API: https://reactnative.dev/docs/animated
- Reanimated: https://docs.swmansion.com/react-native-reanimated/

---

*Tài liệu được biên soạn tổng hợp từ nhiều nguồn, phục vụ mục đích học tập và báo cáo tiểu luận.*
