# TỔNG HỢP KIẾN THỨC REACT NATIVE
## (Những điểm khác biệt so với React Web)

> **Chủ đề:** React Native – Kiến trúc, Component, Styling, Navigation và các khái niệm đặc thù cho Mobile  
> **Phiên bản tham chiếu:** React Native 0.73+ / Expo SDK 50+

---

## MỤC LỤC

1. [Tổng quan – React vs React Native](#1-tổng-quan--react-vs-react-native)
2. [Kiến trúc Bridge và New Architecture](#2-kiến-trúc-bridge-và-new-architecture)
3. [Core Components – Thay thế HTML Tags](#3-core-components--thay-thế-html-tags)
4. [StyleSheet – Thay thế CSS](#4-stylesheet--thay-thế-css)
5. [Flexbox trong React Native](#5-flexbox-trong-react-native)
6. [Xử lý sự kiện và Touch](#6-xử-lý-sự-kiện-và-touch)
7. [Navigation – Điều hướng màn hình](#7-navigation--điều-hướng-màn-hình)
8. [Platform API – Truy cập tính năng thiết bị](#8-platform-api--truy-cập-tính-năng-thiết-bị)
9. [AsyncStorage và Lưu trữ cục bộ](#9-asyncstorage-và-lưu-trữ-cục-bộ)
10. [Animated API – Hoạt ảnh](#10-animated-api--hoạt-ảnh)
11. [FlatList và SectionList – Danh sách hiệu năng cao](#11-flatlist-và-sectionlist--danh-sách-hiệu-năng-cao)
12. [Modal, Alert, và Overlay](#12-modal-alert-và-overlay)
13. [Permissions và Native Modules](#13-permissions-và-native-modules)
14. [Debugging và Công cụ phát triển](#14-debugging-và-công-cụ-phát-triển)
15. [Bảng so sánh tổng hợp React vs React Native](#15-bảng-so-sánh-tổng-hợp-react-vs-react-native)

---

## 1. Tổng quan – React vs React Native

### 1.1 Định nghĩa

**React** là thư viện JavaScript để xây dựng giao diện người dùng cho **web**, render ra các thẻ HTML chạy trong trình duyệt.

**React Native** là framework cho phép dùng cú pháp React để xây dựng ứng dụng **mobile native** (iOS & Android). Thay vì render HTML, React Native ánh xạ component sang UI native thực thụ của từng nền tảng.

### 1.2 Nguyên lý hoạt động

```
REACT (Web)
JSX → Virtual DOM → HTML DOM → Trình duyệt hiển thị

REACT NATIVE (Mobile)
JSX → Virtual DOM → Native Bridge → iOS UIView / Android View
```

Kết quả: ứng dụng React Native **không phải WebView**, mà là app native thật sự, có hiệu năng và trải nghiệm như app viết bằng Swift/Kotlin.

### 1.3 Những thứ giữ nguyên từ React

Điều quan trọng là React Native **kế thừa toàn bộ** mô hình React:

| Giữ nguyên | Mô tả |
|-----------|-------|
| JSX | Cú pháp giống hệt |
| Component (Function/Class) | Cách viết component không đổi |
| Props & State | Hoạt động như React web |
| Tất cả Hooks | useState, useEffect, useContext, ... |
| Context API | Chia sẻ state toàn cục |
| Lifecycle | Các giai đoạn mount/update/unmount |

---

## 2. Kiến trúc Bridge và New Architecture

### 2.1 Kiến trúc cũ – JavaScript Bridge

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

### 2.2 New Architecture (React Native 0.71+)

New Architecture thay thế Bridge bằng **JSI (JavaScript Interface)**, cho phép JavaScript gọi thẳng vào native code mà không cần serialize JSON.

```
┌─────────────────────┐   Trực tiếp (synchronous)  ┌──────────────────────┐
│   JavaScript (JSI)  │ ◄────────────────────────► │  C++ Host Objects    │
└─────────────────────┘                             └──────────────────────┘
                                                             │
                                              ┌──────────────┴──────────────┐
                                         iOS Native               Android Native
```

Hai thành phần chính của New Architecture:

- **Fabric:** Renderer mới, cho phép UI update đồng bộ
- **TurboModules:** Native modules nhẹ hơn, lazy-loaded

---

## 3. Core Components – Thay thế HTML Tags

### 3.1 Định nghĩa

React Native **không dùng thẻ HTML**. Thay vào đó, có các **Core Components** tương ứng được ánh xạ sang UI native của iOS và Android.

### 3.2 Bảng so sánh HTML vs React Native Components

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

### 3.3 Ví dụ: Màn hình đăng nhập

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
import { View, Text, TextInput, TouchableOpacity, StyleSheet } from 'react-native';

function LoginScreen() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Đăng nhập</Text>

      <TextInput
        style={styles.input}
        placeholder="Email"
        value={email}
        onChangeText={setEmail}       // Không phải onChange
        keyboardType="email-address"  // Bàn phím email trên mobile
        autoCapitalize="none"         // Không tự viết hoa
      />

      <TextInput
        style={styles.input}
        placeholder="Mật khẩu"
        value={password}
        onChangeText={setPassword}
        secureTextEntry={true}        // Che mật khẩu
      />

      <TouchableOpacity style={styles.button} onPress={handleLogin}>
        <Text style={styles.buttonText}>Đăng nhập</Text>
      </TouchableOpacity>
    </View>
  );
}
```

### 3.4 Quy tắc quan trọng: Text phải trong `<Text>`

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

### 3.5 Component `<Image>`

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

### 3.6 Pressable – Component nút bấm hiện đại

```jsx
import { Pressable, Text } from 'react-native';

// Pressable thay thế TouchableOpacity, linh hoạt hơn
<Pressable
  onPress={() => console.log('Bấm')}
  onLongPress={() => console.log('Giữ')}
  style={({ pressed }) => [
    styles.button,
    pressed && { opacity: 0.7 }   // Style thay đổi khi đang bấm
  ]}
>
  {({ pressed }) => (
    <Text>{pressed ? 'Đang bấm...' : 'Bấm vào đây'}</Text>
  )}
</Pressable>
```

---

## 4. StyleSheet – Thay thế CSS

### 4.1 Định nghĩa

React Native **không dùng CSS**. Thay vào đó, styling được định nghĩa bằng **JavaScript object** thông qua `StyleSheet.create()`. Cú pháp gần giống CSS nhưng có nhiều khác biệt quan trọng.

### 4.2 Cú pháp cơ bản

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

### 4.3 Những khác biệt quan trọng so với CSS

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
| `%` width/height | Không khuyến nghị | Dùng Dimensions hoặc flex |
| `em`, `rem` | Không tồn tại | Chỉ có số tuyệt đối |

### 4.4 Lấy kích thước màn hình

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

### 4.5 Không có CSS classes hay global styles

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

## 5. Flexbox trong React Native

### 5.1 Định nghĩa

React Native dùng **Flexbox** làm hệ thống layout duy nhất (không có Grid, Float, ...). Cú pháp gần giống web nhưng có **hai khác biệt mặc định quan trọng**.

### 5.2 Khác biệt mặc định với CSS Flexbox

| Thuộc tính | CSS Web | React Native |
|-----------|---------|-------------|
| `flexDirection` | `row` | **`column`** ← khác! |
| `alignContent` | `stretch` | `flex-start` |
| `flexShrink` | `1` | `0` |

> **Quan trọng:** Trong React Native, các phần tử xếp **dọc theo cột** mặc định, không phải theo hàng như web.

### 5.3 Ví dụ Layout

```jsx
// Layout cột (mặc định)
<View style={{ flex: 1, flexDirection: 'column' }}>
  <View style={{ height: 60, backgroundColor: 'blue' }} />   {/* Header */}
  <View style={{ flex: 1, backgroundColor: 'white' }} />     {/* Body - chiếm hết không gian còn lại */}
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

### 5.4 SafeAreaView – Tránh vùng notch và home indicator

```jsx
import { SafeAreaView } from 'react-native-safe-area-context';
// hoặc
import { SafeAreaView } from 'react-native';

// Đảm bảo nội dung không bị che bởi notch, status bar, home indicator
function App() {
  return (
    <SafeAreaView style={{ flex: 1, backgroundColor: '#fff' }}>
      <Text>Nội dung hiển thị an toàn trên mọi thiết bị</Text>
    </SafeAreaView>
  );
}
```

---

## 6. Xử lý sự kiện và Touch

### 6.1 Định nghĩa

React Native không có các sự kiện chuột (`onClick`, `onMouseOver`, ...) vì mobile dùng ngón tay. Thay vào đó là hệ thống sự kiện **touch** và **gesture**.

### 6.2 So sánh sự kiện

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

### 6.3 Ví dụ xử lý Touch

```jsx
import {
  TouchableOpacity,
  TouchableHighlight,
  TouchableNativeFeedback,
  Pressable,
  Platform
} from 'react-native';

// TouchableOpacity – phổ biến nhất, giảm opacity khi bấm
<TouchableOpacity
  onPress={() => console.log('Bấm')}
  onLongPress={() => console.log('Giữ')}
  activeOpacity={0.7}      // Opacity khi bấm (mặc định 0.2)
  delayLongPress={500}     // Thời gian giữ để trigger onLongPress (ms)
>
  <Text>Bấm vào tôi</Text>
</TouchableOpacity>

// Pressable – hiện đại nhất, nhiều tùy chọn nhất
<Pressable
  onPress={handlePress}
  onPressIn={() => console.log('Ngón tay đặt xuống')}
  onPressOut={() => console.log('Ngón tay nhấc lên')}
  hitSlop={{ top: 10, bottom: 10, left: 10, right: 10 }} // Vùng bấm mở rộng
>
  <Text>Nút bấm</Text>
</Pressable>
```

### 6.4 TextInput – Ô nhập liệu

```jsx
import { TextInput, Platform } from 'react-native';

<TextInput
  value={text}
  onChangeText={setText}              // Nhận string thẳng, không phải event
  placeholder="Nhập nội dung..."
  placeholderTextColor="#999"

  // Loại bàn phím
  keyboardType="numeric"              // default|numeric|email-address|phone-pad|decimal-pad|url

  // Hành vi return key
  returnKeyType="done"               // done|next|go|search|send
  onSubmitEditing={handleSubmit}     // Khi bấm return key

  // Nhiều dòng
  multiline={true}
  numberOfLines={4}

  // Bảo mật
  secureTextEntry={true}

  // Auto behavior
  autoCorrect={false}
  autoCapitalize="none"              // none|sentences|words|characters

  // Style
  style={styles.input}
/>
```

---

## 7. Navigation – Điều hướng màn hình

### 7.1 Định nghĩa

React web dùng **URL routing** (React Router, Next.js Router). React Native **không có URL hay trình duyệt**, nên phải dùng thư viện điều hướng riêng. Thư viện phổ biến nhất là **React Navigation**.

### 7.2 Cài đặt React Navigation

```bash
npm install @react-navigation/native
npm install @react-navigation/native-stack
npm install react-native-screens react-native-safe-area-context
```

### 7.3 Stack Navigator – Màn hình chồng lên nhau

```jsx
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator();

// App.js – Định nghĩa các màn hình
function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator initialRouteName="Home">
        <Stack.Screen
          name="Home"
          component={HomeScreen}
          options={{ title: 'Trang chủ' }}
        />
        <Stack.Screen
          name="Detail"
          component={DetailScreen}
          options={{ title: 'Chi tiết' }}
        />
        <Stack.Screen
          name="Profile"
          component={ProfileScreen}
          options={{ headerShown: false }}  // Ẩn header
        />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

### 7.4 Điều hướng giữa các màn hình

```jsx
// HomeScreen.js
function HomeScreen({ navigation, route }) {
  return (
    <View>
      {/* Chuyển màn hình và truyền params */}
      <Button
        title="Xem chi tiết"
        onPress={() => navigation.navigate('Detail', {
          itemId: 42,
          itemName: 'Sản phẩm A'
        })}
      />

      {/* Quay lại màn hình trước */}
      <Button title="Quay lại" onPress={() => navigation.goBack()} />

      {/* Reset stack – không thể quay lại */}
      <Button
        title="Về trang chủ"
        onPress={() => navigation.reset({ index: 0, routes: [{ name: 'Home' }] })}
      />
    </View>
  );
}

// DetailScreen.js – Nhận params
function DetailScreen({ navigation, route }) {
  const { itemId, itemName } = route.params;  // Lấy params từ màn hình trước

  return (
    <View>
      <Text>ID: {itemId}</Text>
      <Text>Tên: {itemName}</Text>
    </View>
  );
}
```

### 7.5 Tab Navigator – Điều hướng dạng tab

```jsx
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';

const Tab = createBottomTabNavigator();

function MainTabs() {
  return (
    <Tab.Navigator
      screenOptions={({ route }) => ({
        tabBarIcon: ({ focused, color, size }) => {
          // Hiển thị icon khác nhau khi focused
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

### 7.6 Drawer Navigator – Menu kéo từ cạnh

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

// Mở menu drawer từ bất kỳ màn hình nào
navigation.openDrawer();
navigation.closeDrawer();
```

### 7.7 So sánh React Router Web vs React Navigation

| | React Router (Web) | React Navigation (Mobile) |
|-|-------------------|-----------------------------|
| Điều hướng | URL (`/home`, `/detail/1`) | Tên màn hình (`'Home'`, `'Detail'`) |
| Truyền data | URL params, Query string | `route.params` |
| Quay lại | Nút Back trình duyệt | `navigation.goBack()` |
| Lịch sử | Browser history | Navigation stack |
| Link | `<Link to="/about">` | `navigation.navigate('About')` |

---

## 8. Platform API – Truy cập tính năng thiết bị

### 8.1 Định nghĩa

React Native cung cấp API để nhận biết và phân biệt giữa **iOS và Android**, vì hai nền tảng có thể cần xử lý khác nhau về UI lẫn behavior.

### 8.2 Platform module

```jsx
import { Platform, StyleSheet } from 'react-native';

// Kiểm tra nền tảng
console.log(Platform.OS);         // 'ios' | 'android' | 'web'
console.log(Platform.Version);    // iOS: '17.0' | Android: 33 (API level)
console.log(Platform.isPad);      // true nếu là iPad
console.log(Platform.isTV);       // true nếu là TV

// Điều kiện đơn giản
const statusBarHeight = Platform.OS === 'ios' ? 44 : 24;

// Platform.select – trả về giá trị theo nền tảng
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

### 8.3 Platform-specific files

Tạo file riêng cho từng nền tảng – React Native tự chọn đúng file:

```
Button.ios.js       ← Dùng cho iOS
Button.android.js   ← Dùng cho Android
Button.js           ← Fallback (web hoặc khi không có file riêng)
```

```jsx
// Button.ios.js
export default function Button({ title, onPress }) {
  return (
    <TouchableOpacity style={styles.iosButton} onPress={onPress}>
      <Text>{title}</Text>
    </TouchableOpacity>
  );
}

// Button.android.js
export default function Button({ title, onPress }) {
  return (
    <TouchableNativeFeedback onPress={onPress}>
      <View style={styles.androidButton}>
        <Text>{title}</Text>
      </View>
    </TouchableNativeFeedback>
  );
}

// Import ở file khác – RN tự chọn đúng file
import Button from './Button';  // Tự dùng .ios.js hoặc .android.js
```

### 8.4 StatusBar

```jsx
import { StatusBar } from 'react-native';

function App() {
  return (
    <>
      <StatusBar
        barStyle="dark-content"     // dark-content | light-content | default
        backgroundColor="#ffffff"  // Android only
        translucent={true}         // Android only – nội dung hiển thị dưới status bar
        hidden={false}
      />
      {/* Nội dung app */}
    </>
  );
}
```

---

## 9. AsyncStorage và Lưu trữ cục bộ

### 9.1 Định nghĩa

React web dùng `localStorage` / `sessionStorage`. React Native **không có** các API này. Thay vào đó dùng **AsyncStorage** – một key-value store bất đồng bộ, tương tự localStorage nhưng hoạt động không đồng bộ (Promise-based).

### 9.2 Cài đặt và sử dụng

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

// Xóa toàn bộ
const clearAll = async () => {
  await AsyncStorage.clear();
};
```

### 9.3 Ví dụ: Hook quản lý token đăng nhập

```jsx
import AsyncStorage from '@react-native-async-storage/async-storage';
import { useState, useEffect } from 'react';

function useAuthToken() {
  const [token, setToken] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Đọc token khi app khởi động
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

### 9.4 So sánh lưu trữ

| | Web | React Native |
|-|-----|-------------|
| **Key-value đơn giản** | `localStorage` | `AsyncStorage` |
| **Session** | `sessionStorage` | Không có (dùng state) |
| **Database** | IndexedDB | SQLite (`expo-sqlite`) |
| **Bất đồng bộ** | ❌ Đồng bộ | ✅ Luôn async |
| **Giới hạn** | ~5–10MB | ~6MB (iOS) / không giới hạn (Android) |

---

## 10. Animated API – Hoạt ảnh

### 10.1 Định nghĩa

React web dùng CSS transitions/animations. React Native có **Animated API** riêng, chạy trên **Native Thread** để animation mượt mà, không bị chặn bởi JS thread.

### 10.2 Animation cơ bản

```jsx
import { Animated, TouchableOpacity, Text } from 'react-native';
import { useRef } from 'react';

function FadeInView() {
  // useRef giữ giá trị Animated.Value giữa các render
  const fadeAnim = useRef(new Animated.Value(0)).current;   // Bắt đầu từ 0 (ẩn)

  const fadeIn = () => {
    Animated.timing(fadeAnim, {
      toValue: 1,             // Kết thúc tại 1 (hiện)
      duration: 500,          // Thời gian (ms)
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

### 10.3 Các loại Animation

```jsx
// 1. timing – theo thời gian (phổ biến nhất)
Animated.timing(value, {
  toValue: 1,
  duration: 300,
  easing: Easing.ease,       // Easing function
  useNativeDriver: true,
}).start(({ finished }) => {
  // Callback khi animation kết thúc
  console.log('Xong:', finished);
});

// 2. spring – lò xo, tự nhiên
Animated.spring(value, {
  toValue: 1,
  tension: 40,
  friction: 7,
  useNativeDriver: true,
}).start();

// 3. decay – giảm dần theo quán tính
Animated.decay(value, {
  velocity: 0.5,
  deceleration: 0.997,
  useNativeDriver: true,
}).start();

// 4. sequence – chạy tuần tự
Animated.sequence([
  Animated.timing(anim1, { toValue: 1, duration: 300, useNativeDriver: true }),
  Animated.timing(anim2, { toValue: 1, duration: 300, useNativeDriver: true }),
]).start();

// 5. parallel – chạy cùng lúc
Animated.parallel([
  Animated.timing(fadeAnim, { toValue: 1, duration: 500, useNativeDriver: true }),
  Animated.timing(slideAnim, { toValue: 0, duration: 500, useNativeDriver: true }),
]).start();

// 6. loop – lặp lại
Animated.loop(
  Animated.timing(spinAnim, { toValue: 1, duration: 1000, useNativeDriver: true })
).start();
```

### 10.4 Ví dụ: Slide in từ phải

```jsx
function SlideInCard() {
  const slideAnim = useRef(new Animated.Value(300)).current; // Bắt đầu ngoài màn hình

  useEffect(() => {
    Animated.spring(slideAnim, {
      toValue: 0,            // Về vị trí ban đầu
      useNativeDriver: true,
      tension: 50,
      friction: 8,
    }).start();
  }, []);

  return (
    <Animated.View style={{
      transform: [{ translateX: slideAnim }]  // Di chuyển theo trục X
    }}>
      <Text>Card slide vào từ phải</Text>
    </Animated.View>
  );
}
```

---

## 11. FlatList và SectionList – Danh sách hiệu năng cao

### 11.1 Định nghĩa

React web dùng `.map()` để render danh sách – đơn giản nhưng render toàn bộ DOM ngay cả khi không nhìn thấy. React Native có **FlatList** và **SectionList** với cơ chế **virtualization**: chỉ render các item đang hiển thị trong viewport, giúp xử lý danh sách hàng nghìn item mượt mà.

### 11.2 FlatList – Danh sách phẳng

```jsx
import { FlatList, View, Text, StyleSheet } from 'react-native';

const DATA = Array.from({ length: 1000 }, (_, i) => ({
  id: String(i),
  title: `Item số ${i + 1}`,
}));

function MyList() {
  // renderItem: hàm render từng item
  const renderItem = ({ item, index }) => (
    <View style={styles.item}>
      <Text style={styles.title}>{item.title}</Text>
    </View>
  );

  return (
    <FlatList
      data={DATA}
      renderItem={renderItem}
      keyExtractor={item => item.id}      // Key duy nhất cho mỗi item

      // Hiệu năng
      initialNumToRender={10}             // Số item render lần đầu
      maxToRenderPerBatch={5}             // Render thêm mỗi batch
      windowSize={5}                      // Kích thước cửa sổ render

      // UI
      ItemSeparatorComponent={() => (     // Đường kẻ giữa các item
        <View style={{ height: 1, backgroundColor: '#eee' }} />
      )}
      ListHeaderComponent={<Text style={styles.header}>Danh sách</Text>}
      ListFooterComponent={<Text style={styles.footer}>Đã tải hết</Text>}
      ListEmptyComponent={<Text>Không có dữ liệu</Text>}

      // Events
      onEndReached={loadMoreData}         // Khi scroll gần cuối → load thêm
      onEndReachedThreshold={0.1}         // Trigger khi còn 10% cuối

      // Horizontal list
      horizontal={false}

      // Pull to refresh
      refreshing={refreshing}
      onRefresh={handleRefresh}
    />
  );
}
```

### 11.3 SectionList – Danh sách có nhóm

```jsx
import { SectionList, View, Text } from 'react-native';

const SECTIONS = [
  {
    title: 'Rau củ',
    data: ['Cà rốt', 'Bắp cải', 'Cải thìa'],
  },
  {
    title: 'Trái cây',
    data: ['Táo', 'Xoài', 'Chuối'],
  },
];

function GroupedList() {
  return (
    <SectionList
      sections={SECTIONS}
      keyExtractor={(item, index) => item + index}
      renderItem={({ item }) => (
        <View style={styles.item}>
          <Text>{item}</Text>
        </View>
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

### 11.4 So sánh cách render danh sách

| | React Web | React Native |
|-|-----------|-------------|
| Cách thông thường | `array.map()` | `<FlatList>` |
| Virtualization | ❌ Render tất cả | ✅ Chỉ render items hiển thị |
| 10,000 items | Lag nặng | Mượt mà |
| Key | `key` prop | `keyExtractor` prop |
| Pull to refresh | Custom | `refreshing` + `onRefresh` |
| Infinite scroll | Custom | `onEndReached` |

---

## 12. Modal, Alert, và Overlay

### 12.1 Alert – Hộp thoại hệ thống

React Native có API `Alert` trực tiếp gọi **hộp thoại native** của iOS/Android.

```jsx
import { Alert } from 'react-native';

// Alert đơn giản
Alert.alert('Thông báo', 'Thao tác thành công!');

// Alert với nhiều nút
Alert.alert(
  'Xác nhận',                          // Tiêu đề
  'Bạn có chắc muốn xóa không?',       // Nội dung
  [
    {
      text: 'Hủy',
      style: 'cancel',                  // style: 'default' | 'cancel' | 'destructive'
      onPress: () => console.log('Hủy'),
    },
    {
      text: 'Xóa',
      style: 'destructive',             // Màu đỏ trên iOS
      onPress: () => handleDelete(),
    },
  ],
  { cancelable: true }                  // Android: bấm ngoài để đóng
);

// Prompt (nhập liệu) – chỉ iOS
Alert.prompt(
  'Nhập tên',
  'Nhập tên của bạn:',
  (text) => console.log('Tên:', text)
);
```

### 12.2 Modal

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
        transparent={true}              // Nền trong suốt → thấy màn hình phía sau
        animationType="slide"           // none | slide | fade
        onRequestClose={() => setVisible(false)}  // Android: nút Back
        statusBarTranslucent={true}
      >
        {/* Overlay */}
        <View style={styles.overlay}>
          {/* Nội dung modal */}
          <View style={styles.modalBox}>
            <Text style={styles.modalTitle}>Tiêu đề Modal</Text>
            <Text>Nội dung bên trong modal</Text>
            <TouchableOpacity
              onPress={() => setVisible(false)}
              style={styles.closeButton}
            >
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

## 13. Permissions và Native Modules

### 13.1 Định nghĩa

Truy cập các tính năng phần cứng (camera, GPS, microphone, thư viện ảnh, ...) trên mobile **bắt buộc phải xin quyền** từ người dùng. Đây là điểm hoàn toàn khác biệt so với React web.

### 13.2 Expo Permissions (Expo)

```jsx
import * as Location from 'expo-location';
import * as ImagePicker from 'expo-image-picker';
import * as Camera from 'expo-camera';

// Vị trí GPS
async function getLocation() {
  // Xin quyền trước khi dùng
  const { status } = await Location.requestForegroundPermissionsAsync();

  if (status !== 'granted') {
    Alert.alert('Quyền bị từ chối', 'Cần quyền vị trí để dùng tính năng này');
    return;
  }

  // Sau khi có quyền mới lấy vị trí
  const location = await Location.getCurrentPositionAsync({
    accuracy: Location.Accuracy.High,
  });

  console.log('Vĩ độ:', location.coords.latitude);
  console.log('Kinh độ:', location.coords.longitude);
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

// Chụp ảnh bằng camera
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

### 13.3 Expo APIs thông dụng

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

## 14. Debugging và Công cụ phát triển

### 14.1 So sánh công cụ debug

| | React Web | React Native |
|-|-----------|-------------|
| DevTools | Browser DevTools | Flipper / Expo DevTools |
| Console | Browser Console | Metro bundler console |
| Network | Network tab | Flipper Network plugin |
| Layout | Elements tab | React DevTools |
| Hot Reload | ✅ | ✅ Fast Refresh |
| Remote debug | Không cần | Chrome DevTools (qua USB) |

### 14.2 Developer Menu

Trên thiết bị/emulator, lắc điện thoại hoặc nhấn `Cmd+D` (iOS) / `Cmd+M` (Android) để mở Developer Menu:

```
Developer Menu
├── Reload                  → Reload lại app
├── Open Debugger           → Mở Chrome DevTools
├── Show Element Inspector  → Inspect UI như browser
├── Performance Monitor     → Xem FPS, JS thread
├── Show Perf Monitor       → Hiệu năng realtime
└── Enable Fast Refresh     → Hot reload
```

### 14.3 Console.log trong React Native

```jsx
// Hoạt động bình thường, hiển thị trong terminal
console.log('Debug:', someValue);
console.warn('Cảnh báo');       // Màu vàng trong Metro
console.error('Lỗi');           // Màu đỏ + Red Box trên simulator

// LogBox – kiểm soát warning/error hiển thị
import { LogBox } from 'react-native';

// Ẩn warning cụ thể (không khuyến nghị dùng thường xuyên)
LogBox.ignoreLogs(['Warning: ...']);
LogBox.ignoreAllLogs();  // Ẩn tất cả (chỉ dùng khi debug production)
```

---

## 15. Bảng so sánh tổng hợp React vs React Native

### 15.1 Bảng khác biệt chính

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

### 15.2 Checklist chuyển từ React sang React Native

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

---

*Tài liệu được biên soạn phục vụ mục đích học tập và báo cáo tiểu luận.*
