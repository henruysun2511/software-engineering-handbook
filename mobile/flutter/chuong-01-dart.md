# Chương 1: Ngôn Ngữ Dart Cơ Bản

---

## 1.1. Giới Thiệu Dart

### 1.1.1. Dart Là Gì?

Dart là ngôn ngữ lập trình do Google phát triển, ra mắt năm 2011 và trở thành ngôn ngữ chính thức của Flutter từ năm 2018. Dart được thiết kế với mục tiêu rõ ràng: **năng suất cao cho developer, hiệu năng tốt trên mọi nền tảng**.

Điểm đặc biệt của Dart so với các ngôn ngữ cùng thời:

- **Ahead-of-Time (AOT) compilation:** Dart được biên dịch thành native machine code trước khi chạy — cho hiệu năng ngang C++ trong production.
- **Just-in-Time (JIT) compilation:** Trong development, Dart dùng JIT để hỗ trợ Hot Reload — thay đổi code và thấy kết quả trong vài trăm milliseconds mà không cần restart app.
- **Single-threaded với event loop:** Dart chạy trên một thread chính, tránh race condition. Concurrency được xử lý qua `async/await` và `Isolate`.
- **Sound null safety:** Từ Dart 2.12, null safety được tích hợp ở cấp độ type system — compiler đảm bảo biến không thể là null trừ khi được khai báo tường minh.

### 1.1.2. So Sánh Với TypeScript (Nền Tảng Quen Thuộc)

Vì bạn đã có nền React Native + TypeScript, bảng so sánh sau giúp map kiến thức cũ sang Dart nhanh hơn:

| Khái niệm | TypeScript | Dart |
|---|---|---|
| Kiểu nguyên thủy | `number`, `string`, `boolean` | `int`, `double`, `String`, `bool` |
| Kiểu dynamic | `any` | `dynamic` |
| Kiểu suy luận | `const x = 5` → `number` | `var x = 5` → `int` |
| Null safety | `string \| null` hoặc `string?` | `String?` |
| Arrow function | `const fn = (x: number) => x * 2` | `int fn(int x) => x * 2` |
| Optional chaining | `obj?.prop` | `obj?.prop` |
| Nullish coalescing | `value ?? 'default'` | `value ?? 'default'` |
| Generic | `Array<string>` | `List<String>` |
| Interface | `interface Animal {}` | `abstract interface class Animal {}` |
| Enum | `enum Color { Red, Green }` | `enum Color { red, green }` |
| Async/await | `async function fetchData()` | `Future<void> fetchData() async` |
| Destructuring | `const { name, age } = person` | `final Product(:name, :price) = product` (Dart 3) |
| Spread | `[...arr1, ...arr2]` | `[...list1, ...list2]` |
| Module import | `import { useState } from 'react'` | `import 'package:flutter/material.dart'` |

---

## 1.2. Biến và Kiểu Dữ Liệu

### 1.2.1. Khai Báo Biến

Dart cung cấp nhiều cách khai báo biến với ngữ nghĩa khác nhau:

```dart
// var — type inference, có thể gán lại
var count = 0;          // Dart tự suy luận: int
var name = 'Flutter';   // Dart tự suy luận: String
count = 5;              // ✅ OK — gán lại cùng type
// count = 'hello';     // ❌ Compile error — sai kiểu

// final — gán một lần, không thể thay đổi reference
// Giống const trong JavaScript/TypeScript
final userId = 'abc-123';
final createdAt = DateTime.now(); // Giá trị tính lúc runtime
// userId = 'xyz';      // ❌ Compile error

// const — compile-time constant, giá trị phải biết lúc compile
const pi = 3.14159;
const appName = 'My App';
const maxRetries = 3;
// const now = DateTime.now(); // ❌ Compile error — DateTime.now() là runtime

// Khai báo với type tường minh — rõ ràng hơn, dễ đọc hơn
int score = 100;
String displayName = 'Nguyễn Văn A';
bool isLoggedIn = false;
double price = 99.99;
```

**Quy tắc chọn `var`, `final`, hay `const`:**

| Từ khóa | Khi nào dùng |
|---|---|
| `const` | Giá trị biết lúc compile: magic numbers, string constants, config |
| `final` | Giá trị chỉ gán một lần nhưng tính lúc runtime: result từ function, object |
| `var` | Biến thay đổi sau khi khởi tạo: loop counter, mutable state |
| Explicit type | Khi muốn code rõ ràng hơn, hoặc khai báo không gán ngay |

```dart
// ✅ CHUẨN — Sử dụng đúng từ khóa
const apiBaseUrl = 'https://api.example.com'; // Biết lúc compile
final user = await authRepo.getCurrentUser();  // Tính lúc runtime, không đổi
var retryCount = 0;                            // Thay đổi trong vòng lặp
retryCount++;
```

### 1.2.2. Kiểu Số: int và double

```dart
// int — số nguyên, không giới hạn độ lớn trên native platforms
int age = 25;
int negativeValue = -100;
int hexColor = 0xFF6750A4;    // Hex literal
int binaryFlag = 0b1010;      // Binary literal
int million = 1_000_000;      // Underscore separator (Dart 2.12+)

// double — số thực IEEE 754 64-bit
double price = 19.99;
double pi = 3.14159265358979;
double infinity = double.infinity;
double notANumber = double.nan;

// Chuyển đổi kiểu — KHÔNG tự động như JavaScript
int intValue = 42;
double doubleValue = intValue.toDouble();    // int → double
int backToInt = doubleValue.toInt();         // double → int (truncate, không round)
int rounded = doubleValue.round();           // double → int (làm tròn)

// Parse từ String
int parsed = int.parse('42');
double parsedDouble = double.parse('3.14');

// Parse an toàn — trả về null nếu không parse được
int? safeParsed = int.tryParse('abc');       // null, không throw exception
int withDefault = int.tryParse('abc') ?? 0; // 0 nếu parse thất bại

// Arithmetic — tương tự hầu hết ngôn ngữ
print(7 ~/ 2);   // 3  — integer division (khác với 7 / 2 = 3.5)
print(7 % 2);    // 1  — modulo
print(2.pow(10)); // Không tồn tại — dùng math.pow(2, 10)

import 'dart:math' as math;
print(math.pow(2, 10));    // 1024
print(math.sqrt(16));      // 4.0
print(math.max(10, 20));   // 20
```

### 1.2.3. String — Chuỗi Ký Tự

```dart
// Khai báo — dùng single quote hoặc double quote đều được
// Quy ước: dùng single quote cho code Dart
String greeting = 'Xin chào';
String path = "C:\\Users\\flutter";  // Double quote khi có single quote bên trong

// Multi-line string — dùng triple quotes
String address = '''
123 Đường ABC,
Quận 1,
TP. Hồ Chí Minh
''';

// Raw string — không xử lý escape sequence
String regex = r'\d+\.\d+'; // Regex pattern, \ không bị escape
String windowsPath = r'C:\Users\flutter\project';

// String interpolation — nhúng biến vào string
String name = 'Flutter';
int version = 3;
print('Chào mừng đến với $name $version!');   // Đơn giản
print('Tổng: ${2 + 3}');                       // Biểu thức phức tạp dùng ${}
print('${name.toUpperCase()} - v$version');    // Gọi method trong ${}

// String methods phổ biến
String text = '  Hello, Flutter!  ';
print(text.trim());                   // 'Hello, Flutter!'
print(text.toLowerCase());            // '  hello, flutter!  '
print(text.toUpperCase());            // '  HELLO, FLUTTER!  '
print(text.contains('Flutter'));      // true
print(text.startsWith('  Hello'));    // true
print(text.replaceAll('l', 'L'));     // '  HeLLo, FLutter!  '
print(text.split(', '));              // ['  Hello', 'Flutter!  ']
print(text.substring(2, 7));         // 'Hello'
print(text.trim().length);            // 16
print(text.isEmpty);                  // false
print(''.isEmpty);                    // true
print(''.isNotEmpty);                 // false
print(text.indexOf('Flutter'));       // 9 (sau trim: 7)

// String formatting
double price = 1234567.89;
// Dùng NumberFormat từ intl package trong dự án thực tế
// Đơn giản:
print(price.toStringAsFixed(2));      // '1234567.89'
print(price.toStringAsFixed(0));      // '1234568'

// String concatenation — tránh dùng + trong vòng lặp
// Dùng StringBuffer cho hiệu năng tốt hơn
final buffer = StringBuffer();
for (int i = 0; i < 5; i++) {
  buffer.write('Item $i, ');
}
final result = buffer.toString(); // 'Item 0, Item 1, Item 2, Item 3, Item 4, '
```

### 1.2.4. bool — Kiểu Logic

```dart
bool isActive = true;
bool hasPermission = false;

// Không có "truthy/falsy" như JavaScript
// Dart CHỈ chấp nhận true/false trong điều kiện — không có implicit conversion
int count = 1;
// if (count) { } // ❌ Compile error trong Dart — khác với JS
if (count > 0) { } // ✅ Phải so sánh tường minh

// Logical operators
bool andResult = true && false;   // false
bool orResult  = true || false;   // true
bool notResult = !true;           // false

// Conditional expressions
bool isAdult = age >= 18;
String label = isAdult ? 'Người lớn' : 'Trẻ em';
```

---

## 1.3. Null Safety — Tính Năng Quan Trọng Nhất

### 1.3.1. Bản Chất Của Null Safety

Null safety là hệ thống đảm bảo **null reference error không thể xảy ra trong runtime** nếu code compile thành công. Đây là tính năng phân biệt Dart hiện đại với nhiều ngôn ngữ khác.

Trước Dart 2.12, mọi biến đều có thể là null — đây là nguồn gốc của vô số lỗi runtime khó debug. Từ Dart 2.12, mặc định mọi biến đều **non-nullable**, trừ khi khai báo tường minh là nullable bằng dấu `?`.

```dart
// NON-NULLABLE — Mặc định
String name = 'Flutter';
// name = null; // ❌ Compile error ngay lập tức

// NULLABLE — Thêm dấu ? sau type
String? nullableName = null; // ✅
nullableName = 'Dart';       // ✅
nullableName = null;         // ✅

int? nullableAge;    // Mặc định null khi chưa gán
print(nullableAge);  // null

// Từ khóa null safety quan trọng
// ? — khai báo nullable type
// ?? — null-aware coalescing (giống ?? trong TS)
// ?. — null-aware access (giống ?. trong TS)
// !  — null assertion (force unwrap — dùng cẩn thận!)
// late — defer initialization
```

### 1.3.2. Toán Tử Null-Aware

```dart
String? middleName;

// ?? — Null coalescing: dùng giá trị bên phải nếu bên trái là null
String displayMiddle = middleName ?? '';      // ''
String label = middleName ?? 'Không có';     // 'Không có'

// ??= — Null-aware assignment: chỉ gán nếu biến đang null
middleName ??= 'Văn';   // Gán 'Văn' vì middleName đang null
middleName ??= 'Anh';   // Không gán vì middleName đã là 'Văn'

// ?. — Null-aware member access
String? email;
int? emailLength = email?.length;        // null (không throw)
String? upperEmail = email?.toUpperCase(); // null

// Chaining nhiều null-aware
String? user;
String? city;
// user?.address?.city — trả về null nếu bất kỳ bước nào là null

// ! — Null assertion operator
// Nói với compiler: "Tôi chắc chắn giá trị này không null"
// Nếu sai → runtime error NullCheckOperationException
String? possiblyNull = 'Definitely not null';
String definitelyNotNull = possiblyNull!; // ✅ OK vì thực tế không null

// ⚠️ Nguy hiểm:
String? anotherNull;
// String bad = anotherNull!; // ❌ Runtime error!

// Thực tế: dùng ! khi bạn 100% chắc chắn và có lý do rõ ràng
// Ví dụ: sau khi đã kiểm tra null
if (possiblyNull != null) {
  // Trong block này, Dart biết possiblyNull không null
  // Smart cast — không cần !
  print(possiblyNull.length); // ✅
}
```

### 1.3.3. Từ Khóa `late`

`late` cho phép khai báo biến non-nullable nhưng khởi tạo sau — hữu ích khi giá trị không thể có ngay lúc khai báo.

```dart
// late — khởi tạo sau nhưng đảm bảo trước khi dùng
class DatabaseHelper {
  // Không thể khởi tạo ngay vì cần async setup
  late Database _db;

  Future<void> initialize() async {
    _db = await openDatabase('app.db');
  }

  // Nếu dùng _db trước khi initialize() được gọi → LateInitializationError
  Future<List<Map>> query(String table) => _db.query(table);
}

// late với lazy initialization
class HeavyService {
  // Chỉ khởi tạo khi lần đầu tiên được truy cập
  late final List<String> _cache = _loadCache();

  List<String> _loadCache() {
    print('Loading cache...'); // Chỉ in một lần
    return ['item1', 'item2', 'item3'];
  }
}

// So sánh: late vs nullable
class UserProfile {
  // Dùng late khi chắc chắn sẽ được set trước khi dùng
  late String displayName;

  // Dùng nullable khi có thể thực sự không có giá trị
  String? bio; // User có thể không điền bio
}
```

---

## 1.4. Hàm (Functions)

### 1.4.1. Khai Báo Hàm

```dart
// Hàm thông thường — khai báo return type tường minh
int add(int a, int b) {
  return a + b;
}

// Arrow function — cho hàm một biểu thức
int multiply(int a, int b) => a * b;

// Hàm không trả về giá trị
void printGreeting(String name) {
  print('Xin chào, $name!');
}

// Hàm trả về null-safety
String? findUser(String id) {
  // Có thể trả về null nếu không tìm thấy
  return id == '123' ? 'Nguyễn Văn A' : null;
}

// Hàm là first-class citizen — có thể lưu vào biến
// Function type: ReturnType Function(ParamType1, ParamType2)
int Function(int, int) operation = add;
print(operation(3, 4)); // 7
operation = multiply;
print(operation(3, 4)); // 12
```

### 1.4.2. Named Parameters và Optional Parameters

Đây là tính năng Dart khác biệt nhất so với TypeScript — và là pattern dùng rất nhiều trong Flutter widget.

```dart
// Named parameters — đặt trong {}
// Caller phải viết tên param khi gọi
// Giúp code tự document và tránh nhầm thứ tự
void createUser({
  required String name,    // required: bắt buộc truyền
  required String email,
  int age = 18,            // Optional với default value
  String? bio,             // Optional nullable, default null
}) {
  print('$name ($email), tuổi $age');
}

// Gọi hàm — named parameters không cần theo thứ tự
createUser(
  name: 'Nguyễn Văn A',
  email: 'a@example.com',
);
createUser(
  email: 'b@example.com',
  name: 'Trần Thị B',
  age: 25,
  bio: 'Flutter developer',
);

// Positional optional parameters — đặt trong []
// Ít phổ biến hơn named params
String buildPath(String base, [String? segment, String? query]) {
  var path = base;
  if (segment != null) path += '/$segment';
  if (query != null) path += '?$query';
  return path;
}

print(buildPath('/api'));                    // '/api'
print(buildPath('/api', 'users'));          // '/api/users'
print(buildPath('/api', 'users', 'page=1')); // '/api/users?page=1'

// ✅ CHUẨN — Trong Flutter, named params là chuẩn mực
// Widget constructor luôn dùng named params (trừ child)
class CustomButton extends StatelessWidget {
  const CustomButton({
    super.key,               // Named, optional
    required this.label,     // Named, required
    required this.onPressed, // Named, required
    this.isLoading = false,  // Named, optional với default
    this.icon,               // Named, optional nullable
  });

  final String label;
  final VoidCallback onPressed;
  final bool isLoading;
  final IconData? icon;
  // ...
}
```

### 1.4.3. Anonymous Functions và Closures

```dart
// Anonymous function — function không có tên
// Dùng nhiều trong callback, higher-order functions
final greet = (String name) => 'Xin chào, $name!';
print(greet('Flutter')); // 'Xin chào, Flutter!'

// Closure — function nhớ biến từ scope bên ngoài
Function makeCounter() {
  int count = 0; // Biến này được closure "đóng gói"
  return () {
    count++;
    return count;
  };
}

final counter = makeCounter();
print(counter()); // 1
print(counter()); // 2
print(counter()); // 3

final counter2 = makeCounter(); // Instance mới, count riêng
print(counter2()); // 1

// Typedef — đặt tên cho function type
typedef JsonParser<T> = T Function(Map<String, dynamic> json);
typedef VoidCallback = void Function(); // Built-in trong Flutter

// Generic function
T parseJson<T>(String json, JsonParser<T> parser) {
  final map = jsonDecode(json) as Map<String, dynamic>;
  return parser(map);
}
```

---

## 1.5. Collection: List, Map, Set

### 1.5.1. List — Danh Sách Có Thứ Tự

```dart
// Khai báo
List<String> fruits = ['Táo', 'Cam', 'Xoài'];
var numbers = [1, 2, 3, 4, 5];         // Type inferred: List<int>
final empty = <String>[];               // Empty list với type tường minh

// Thêm/xóa phần tử
fruits.add('Chuối');                    // Thêm cuối
fruits.addAll(['Ổi', 'Mận']);           // Thêm nhiều
fruits.insert(0, 'Dưa hấu');           // Thêm tại index
fruits.remove('Cam');                   // Xóa theo value (lần đầu tìm thấy)
fruits.removeAt(0);                     // Xóa theo index
fruits.removeLast();                    // Xóa phần tử cuối

// Truy cập
print(fruits[0]);            // Phần tử đầu
print(fruits.first);         // Phần tử đầu — throw nếu empty
print(fruits.last);          // Phần tử cuối — throw nếu empty
print(fruits.firstOrNull);   // null nếu empty (Dart 2.18+)
print(fruits.length);        // Số phần tử
print(fruits.isEmpty);       // true nếu rỗng
print(fruits.isNotEmpty);    // true nếu không rỗng
print(fruits.contains('Táo')); // true/false

// Immutable list — không thể thêm/xóa sau khi tạo
const fixedList = ['a', 'b', 'c'];     // const list
final readOnly = List.unmodifiable([1, 2, 3]);

// Tạo list bằng constructor
final zeros = List.filled(5, 0);                    // [0, 0, 0, 0, 0]
final generated = List.generate(5, (i) => i * 2);  // [0, 2, 4, 6, 8]

// Spread operator — gộp list
final list1 = [1, 2, 3];
final list2 = [4, 5, 6];
final combined = [...list1, ...list2];     // [1, 2, 3, 4, 5, 6]
final withNull = [...list1, ...?null];     // Spread nullable safely: [1, 2, 3]

// Collection if và collection for
final isAdmin = true;
final menuItems = [
  'Trang chủ',
  'Sản phẩm',
  if (isAdmin) 'Quản trị',    // Thêm điều kiện
  if (isAdmin) ...[            // Thêm nhiều điều kiện
    'Báo cáo',
    'Cài đặt',
  ],
];
// ['Trang chủ', 'Sản phẩm', 'Quản trị', 'Báo cáo', 'Cài đặt']

final ids = [1, 2, 3];
final labels = [
  for (final id in ids) 'Item $id',  // Collection for
];
// ['Item 1', 'Item 2', 'Item 3']
```

### 1.5.2. Higher-Order Functions — map, where, fold, reduce

Đây là các method quan trọng nhất khi làm việc với list trong Flutter:

```dart
final products = [
  Product(id: '1', name: 'Áo', price: 200000, isActive: true),
  Product(id: '2', name: 'Quần', price: 350000, isActive: false),
  Product(id: '3', name: 'Giày', price: 500000, isActive: true),
  Product(id: '4', name: 'Túi', price: 150000, isActive: true),
];

// map — transform mỗi phần tử
// Giống Array.map() trong JS
List<String> names = products
    .map((p) => p.name)
    .toList(); // ['Áo', 'Quần', 'Giày', 'Túi']

List<double> discountedPrices = products
    .map((p) => p.price * 0.9)
    .toList(); // [180000.0, 315000.0, 450000.0, 135000.0]

// where — lọc phần tử thỏa điều kiện
// Giống Array.filter() trong JS
List<Product> activeProducts = products
    .where((p) => p.isActive)
    .toList(); // [Áo, Giày, Túi]

List<Product> expensive = products
    .where((p) => p.price >= 300000)
    .toList(); // [Quần, Giày]

// fold — tích lũy, giảm về một giá trị
// Giống Array.reduce() trong JS nhưng có initial value
double totalPrice = products.fold(
  0.0,                                    // Initial value
  (sum, product) => sum + product.price,  // Accumulator function
); // 1200000.0

int activeCount = products.fold(
  0,
  (count, p) => p.isActive ? count + 1 : count,
); // 3

// reduce — như fold nhưng không có initial value (có thể throw nếu list rỗng)
double maxPrice = products
    .map((p) => p.price)
    .reduce((a, b) => a > b ? a : b); // 500000.0

// any — trả về true nếu ÍT NHẤT một phần tử thỏa điều kiện
// Giống Array.some() trong JS
bool hasExpensive = products.any((p) => p.price > 400000); // true

// every — trả về true nếu TẤT CẢ phần tử thỏa điều kiện
// Giống Array.every() trong JS
bool allActive = products.every((p) => p.isActive); // false

// firstWhere — tìm phần tử đầu tiên thỏa điều kiện
// Throw StateError nếu không tìm thấy
Product? cheapest = products
    .firstWhere((p) => p.price < 200000,
        orElse: () => products.first); // Fallback khi không tìm thấy

// Chaining methods — rất phổ biến trong Flutter
List<String> activeProductNames = products
    .where((p) => p.isActive)
    .where((p) => p.price < 300000)
    .map((p) => p.name.toUpperCase())
    .toList(); // ['ÁO', 'TÚI']

// sort — sắp xếp (mutate list gốc!)
products.sort((a, b) => a.price.compareTo(b.price)); // Tăng dần
products.sort((a, b) => b.price.compareTo(a.price)); // Giảm dần

// Sắp xếp không mutate — tạo list mới
final sorted = [...products]..sort((a, b) => a.name.compareTo(b.name));
```

### 1.5.3. Map — Dictionary / Key-Value

```dart
// Khai báo
Map<String, int> scores = {
  'Alice': 95,
  'Bob': 87,
  'Charlie': 92,
};

var config = <String, dynamic>{  // dynamic value cho mixed types
  'debug': true,
  'maxRetries': 3,
  'baseUrl': 'https://api.example.com',
};

// Thêm và truy cập
scores['Dave'] = 78;           // Thêm hoặc cập nhật
int? aliceScore = scores['Alice'];    // 95 (trả về nullable)
int bobScore = scores['Bob']!;        // 87 (assert not null)
int eveScore = scores['Eve'] ?? 0;    // 0 (default nếu key không tồn tại)

// Kiểm tra
bool hasAlice = scores.containsKey('Alice');   // true
bool hasScore100 = scores.containsValue(100);  // false
print(scores.length);   // 4

// Xóa
scores.remove('Dave');
scores.removeWhere((key, value) => value < 90); // Xóa theo điều kiện

// Lặp qua Map
scores.forEach((name, score) {
  print('$name: $score điểm');
});

for (final entry in scores.entries) {
  print('${entry.key}: ${entry.value}');
}

for (final key in scores.keys) { }
for (final value in scores.values) { }

// Map methods
Map<String, int> doubled = scores.map(
  (key, value) => MapEntry(key, value * 2),
);

// Chuyển List thành Map
final List<Product> productList = [...products];
Map<String, Product> productMap = {
  for (final p in productList) p.id: p,
};
// Truy cập O(1) theo ID thay vì O(n) với list.firstWhere

// fromIterable — cách khác tạo Map từ List
final idToName = Map.fromIterable(
  productList,
  key: (p) => (p as Product).id,
  value: (p) => (p as Product).name,
);
```

### 1.5.4. Set — Tập Hợp Không Trùng Lặp

```dart
// Set — không có phần tử trùng, không có thứ tự
Set<String> tags = {'flutter', 'dart', 'mobile'};
tags.add('flutter');    // Không thêm vì đã tồn tại
print(tags.length);     // 3

// Set operations — rất hữu ích
Set<String> setA = {'a', 'b', 'c', 'd'};
Set<String> setB = {'c', 'd', 'e', 'f'};

Set<String> union = setA.union(setB);         // {'a','b','c','d','e','f'}
Set<String> intersection = setA.intersection(setB); // {'c','d'}
Set<String> difference = setA.difference(setB);     // {'a','b'}

// Ứng dụng phổ biến: lưu trạng thái "đã chọn"
class FilterState {
  Set<String> selectedCategories = {};

  void toggleCategory(String category) {
    if (selectedCategories.contains(category)) {
      selectedCategories.remove(category);
    } else {
      selectedCategories.add(category);
    }
  }

  bool isSelected(String category) => selectedCategories.contains(category);
}
```

---

## 1.6. OOP Trong Dart

### 1.6.1. Class và Constructor

```dart
// ✅ CHUẨN — Class với các loại constructor
class Product {
  // Fields
  final String id;
  final String name;
  final double price;
  int stock;
  String? description;

  // Default constructor
  Product({
    required this.id,
    required this.name,
    required this.price,
    this.stock = 0,
    this.description,
  });

  // Named constructor — tạo object từ JSON
  factory Product.fromJson(Map<String, dynamic> json) {
    return Product(
      id: json['id'] as String,
      name: json['name'] as String,
      price: (json['price'] as num).toDouble(),
      stock: json['stock'] as int? ?? 0,
      description: json['description'] as String?,
    );
  }

  // Named constructor — tạo instance rỗng / placeholder
  factory Product.empty() {
    return Product(
      id: '',
      name: 'Chưa có tên',
      price: 0,
    );
  }

  // Constant constructor — tất cả field phải là final
  const Product.placeholder()
    : id = 'placeholder',
      name = 'Placeholder',
      price = 0,
      stock = 0,
      description = null;

  // Computed property (getter)
  bool get isInStock => stock > 0;
  bool get isFree => price == 0;
  String get formattedPrice {
    if (price >= 1000000) {
      return '${(price / 1000000).toStringAsFixed(1)}M đ';
    }
    return '${price.toStringAsFixed(0)}đ';
  }

  // Setter — validate khi gán
  set stockLevel(int value) {
    if (value < 0) throw ArgumentError('Stock không được âm');
    stock = value;
  }

  // Instance method
  Product copyWith({
    String? name,
    double? price,
    int? stock,
    String? description,
  }) {
    return Product(
      id: id,
      name: name ?? this.name,
      price: price ?? this.price,
      stock: stock ?? this.stock,
      description: description ?? this.description,
    );
  }

  // toJson — để serialize
  Map<String, dynamic> toJson() => {
    'id': id,
    'name': name,
    'price': price,
    'stock': stock,
    if (description != null) 'description': description,
  };

  // toString — debug friendly
  @override
  String toString() => 'Product(id: $id, name: $name, price: $price)';

  // Equality — so sánh theo value, không theo reference
  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      other is Product && other.id == id;

  @override
  int get hashCode => id.hashCode;
}
```

### 1.6.2. Inheritance — Kế Thừa

```dart
// Abstract class — không thể instantiate trực tiếp
abstract class Shape {
  final String color;

  const Shape({required this.color});

  // Abstract method — subclass PHẢI implement
  double get area;
  double get perimeter;

  // Concrete method — subclass có thể dùng hoặc override
  String describe() => '$runtimeType màu $color, diện tích ${area.toStringAsFixed(2)}';
}

class Circle extends Shape {
  const Circle({required super.color, required this.radius});
  final double radius;

  @override
  double get area => 3.14159 * radius * radius;

  @override
  double get perimeter => 2 * 3.14159 * radius;
}

class Rectangle extends Shape {
  const Rectangle({
    required super.color,
    required this.width,
    required this.height,
  });

  final double width;
  final double height;

  @override
  double get area => width * height;

  @override
  double get perimeter => 2 * (width + height);

  @override
  String describe() => '${super.describe()}, ${width}x${height}cm';
}

// Đa hình — xử lý nhiều loại qua interface chung
double totalArea(List<Shape> shapes) {
  return shapes.fold(0, (sum, shape) => sum + shape.area);
}

void main() {
  final shapes = [
    Circle(color: 'đỏ', radius: 5),
    Rectangle(color: 'xanh', width: 4, height: 6),
  ];

  for (final shape in shapes) {
    print(shape.describe());
  }
  print('Tổng diện tích: ${totalArea(shapes)}');
}
```

### 1.6.3. Mixin — Tái Sử Dụng Code Linh Hoạt

Mixin là cơ chế tái sử dụng code mạnh hơn đa kế thừa. Một class có thể `with` nhiều mixin cùng lúc.

```dart
// Mixin — tái sử dụng behavior
mixin Timestamped {
  late final DateTime createdAt = DateTime.now();
  DateTime? updatedAt;

  void markUpdated() => updatedAt = DateTime.now();
}

mixin Serializable {
  Map<String, dynamic> toJson();

  String toJsonString() => jsonEncode(toJson());
}

mixin Validatable {
  List<String> validate();

  bool get isValid => validate().isEmpty;

  void throwIfInvalid() {
    final errors = validate();
    if (errors.isNotEmpty) {
      throw ValidationException(errors);
    }
  }
}

// Kết hợp mixin
class Order with Timestamped, Serializable, Validatable {
  Order({
    required this.id,
    required this.items,
    required this.customerId,
  });

  final String id;
  final List<OrderItem> items;
  final String customerId;

  double get total => items.fold(0, (sum, item) => sum + item.subtotal);

  @override
  Map<String, dynamic> toJson() => {
    'id': id,
    'customerId': customerId,
    'total': total,
    'createdAt': createdAt.toIso8601String(),
  };

  @override
  List<String> validate() {
    final errors = <String>[];
    if (items.isEmpty) errors.add('Đơn hàng phải có ít nhất 1 sản phẩm');
    if (customerId.isEmpty) errors.add('Thiếu thông tin khách hàng');
    if (total <= 0) errors.add('Tổng tiền phải lớn hơn 0');
    return errors;
  }
}

// abstract interface class — chỉ định nghĩa contract, không có implementation
abstract interface class Repository<T, ID> {
  Future<T?> findById(ID id);
  Future<List<T>> findAll();
  Future<T> save(T entity);
  Future<void> delete(ID id);
}
```

### 1.6.4. Sealed Classes — Pattern Matching An Toàn

Dart 3 giới thiệu `sealed` class — một tập hợp các subtype đóng kín. Compiler bắt buộc phải xử lý tất cả các case khi switch.

```dart
// Sealed class — đặc biệt hữu ích cho state management
sealed class ApiResponse<T> {
  const ApiResponse();
}

class ApiSuccess<T> extends ApiResponse<T> {
  const ApiSuccess(this.data);
  final T data;
}

class ApiLoading<T> extends ApiResponse<T> {
  const ApiLoading();
}

class ApiError<T> extends ApiResponse<T> {
  const ApiError(this.message, {this.code});
  final String message;
  final int? code;
}

// Switch expression — exhaustive, compile error nếu thiếu case
Widget buildFromResponse<T>(ApiResponse<T> response, Widget Function(T) builder) {
  return switch (response) {
    ApiLoading() => const CircularProgressIndicator(),
    ApiSuccess(:final data) => builder(data),         // Destructuring
    ApiError(:final message, :final code) => Column(
      children: [
        Icon(Icons.error, color: Colors.red),
        Text('$message (code: $code)'),
      ],
    ),
    // Không cần default — compiler đã verify exhaustive
  };
}

// Ứng dụng thực tế với auth state
sealed class AuthState {}
class AuthInitial extends AuthState {}
class AuthLoading extends AuthState {}
class AuthAuthenticated extends AuthState {
  AuthAuthenticated(this.user);
  final AppUser user;
}
class AuthUnauthenticated extends AuthState {}
class AuthError extends AuthState {
  AuthError(this.message);
  final String message;
}

// Pattern matching với switch
String getAuthMessage(AuthState state) => switch (state) {
  AuthInitial() => 'Đang khởi động...',
  AuthLoading() => 'Đang đăng nhập...',
  AuthAuthenticated(:final user) => 'Xin chào, ${user.name}!',
  AuthUnauthenticated() => 'Vui lòng đăng nhập',
  AuthError(:final message) => 'Lỗi: $message',
};
```

---

## 1.7. Lập Trình Bất Đồng Bộ: Future và Stream

### 1.7.1. Future — Kết Quả Trong Tương Lai

`Future<T>` đại diện cho một giá trị sẽ có trong tương lai — giống `Promise<T>` trong JavaScript.

```dart
// Future cơ bản
Future<String> fetchUsername(String userId) async {
  // await: đợi Future hoàn thành, tạm dừng function này
  // nhưng KHÔNG block thread — event loop vẫn xử lý việc khác
  final response = await dio.get('/users/$userId');
  return response.data['username'] as String;
}

// Xử lý lỗi với try-catch — giống async/await trong TS
Future<User> getUser(String id) async {
  try {
    final response = await dio.get('/users/$id');
    return User.fromJson(response.data);
  } on DioException catch (e) {
    // Bắt lỗi cụ thể
    if (e.response?.statusCode == 404) {
      throw UserNotFoundException(id);
    }
    rethrow; // Re-throw các lỗi khác
  } catch (e, stackTrace) {
    // Bắt tất cả lỗi
    log('Unexpected error', error: e, stackTrace: stackTrace);
    throw UnknownException(e.toString());
  }
}

// Future combinators — chạy song song
Future<void> loadDashboard() async {
  // ❌ Chậm — chạy tuần tự, 3 giây tổng cộng
  final user = await fetchUser();       // 1 giây
  final products = await fetchProducts(); // 1 giây
  final orders = await fetchOrders();    // 1 giây

  // ✅ Nhanh — chạy song song, ~1 giây tổng cộng
  final results = await Future.wait([
    fetchUser(),
    fetchProducts(),
    fetchOrders(),
  ]);
  final user2 = results[0] as User;
  final products2 = results[1] as List<Product>;
  final orders2 = results[2] as List<Order>;

  // Future.wait với type-safe (Dart 3 Records)
  final (user3, products3, orders3) = await (
    fetchUser(),
    fetchProducts(),
    fetchOrders(),
  ).wait;
}

// Future.delayed — delay thực thi
Future<void> showSplashScreen() async {
  await Future.delayed(const Duration(seconds: 2));
  navigateToHome();
}

// Future.value — wrap giá trị sync thành Future
Future<int> alwaysReturns42() => Future.value(42);

// FutureOr<T> — có thể là T hoặc Future<T>
// Hữu ích khi function có thể trả về sync hoặc async
FutureOr<String> maybeAsync(bool useCache) {
  if (useCache) return 'cached value'; // Sync
  return fetchFromApi();               // Async
}
```

### 1.7.2. Stream — Dữ Liệu Liên Tục Theo Thời Gian

`Stream<T>` đại diện cho một chuỗi dữ liệu phát sinh theo thời gian — giống `Observable` trong RxJS.

```dart
// Stream từ Firestore
Stream<List<Message>> getChatMessages(String chatId) {
  return FirebaseFirestore.instance
      .collection('chats')
      .doc(chatId)
      .collection('messages')
      .orderBy('timestamp', descending: true)
      .snapshots()
      .map((snapshot) => snapshot.docs
          .map((doc) => Message.fromFirestore(doc))
          .toList());
}

// StreamController — tạo stream thủ công
class LocationService {
  final _controller = StreamController<LatLng>.broadcast();

  // Expose stream, ẩn controller
  Stream<LatLng> get locationUpdates => _controller.stream;

  void startTracking() {
    // Giả lập vị trí thay đổi
    Timer.periodic(const Duration(seconds: 5), (_) {
      final newLocation = getCurrentLocation();
      _controller.add(newLocation); // Emit giá trị mới
    });
  }

  void dispose() {
    _controller.close(); // QUAN TRỌNG: phải close để giải phóng resource
  }
}

// Lắng nghe Stream trong Dart
final subscription = locationService.locationUpdates.listen(
  (location) => print('Vị trí mới: ${location.lat}, ${location.lng}'),
  onError: (error) => print('Lỗi: $error'),
  onDone: () => print('Stream đã đóng'),
);

// Hủy subscription khi không cần — tránh memory leak
subscription.cancel();

// Stream transformation
Stream<String> locationDescriptions = locationService.locationUpdates
    .where((loc) => loc.lat != 0 && loc.lng != 0)  // Lọc
    .map((loc) => '${loc.lat.toStringAsFixed(4)}, ${loc.lng.toStringAsFixed(4)}') // Transform
    .distinct(); // Bỏ qua giá trị trùng liền nhau

// await for — lặp qua stream trong async function
Future<void> processStream() async {
  await for (final location in locationService.locationUpdates) {
    print('Processing: $location');
    if (shouldStop) break; // Có thể break ra khỏi vòng lặp
  }
}
```

---

## 1.8. Bài Tập Tổng Hợp: Mini Quiz Logic

Bài tập kết hợp tất cả kiến thức chương 1 — viết thuần Dart, không Flutter:

```dart
// lib/quiz_engine.dart
// Một engine quiz đơn giản: câu hỏi, đáp án, tính điểm

// Models
class Question {
  const Question({
    required this.id,
    required this.text,
    required this.options,
    required this.correctIndex,
    this.explanation,
    this.points = 10,
  });

  final String id;
  final String text;
  final List<String> options;
  final int correctIndex;
  final String? explanation;
  final int points;

  String get correctAnswer => options[correctIndex];

  bool isCorrect(int selectedIndex) => selectedIndex == correctIndex;
}

sealed class QuizEvent {}
class AnswerQuestion extends QuizEvent {
  const AnswerQuestion(this.questionId, this.selectedIndex);
  final String questionId;
  final int selectedIndex;
}
class ResetQuiz extends QuizEvent {}
class SkipQuestion extends QuizEvent {}

class QuizResult {
  const QuizResult({
    required this.totalQuestions,
    required this.correctAnswers,
    required this.totalPoints,
    required this.earnedPoints,
    required this.duration,
  });

  final int totalQuestions;
  final int correctAnswers;
  final int totalPoints;
  final int earnedPoints;
  final Duration duration;

  double get accuracy => totalQuestions > 0
      ? correctAnswers / totalQuestions * 100
      : 0;

  String get grade => switch (accuracy) {
        >= 90 => 'A',
        >= 80 => 'B',
        >= 70 => 'C',
        >= 60 => 'D',
        _ => 'F',
      };

  @override
  String toString() =>
      'Kết quả: $correctAnswers/$totalQuestions đúng '
      '($earnedPoints/$totalPoints điểm) - Loại $grade';
}

// Quiz Engine
class QuizEngine {
  QuizEngine({required this.questions}) : _startTime = DateTime.now();

  final List<Question> questions;
  final DateTime _startTime;
  final Map<String, int> _answers = {};
  int _currentIndex = 0;
  int _skippedCount = 0;

  Question? get currentQuestion =>
      _currentIndex < questions.length ? questions[_currentIndex] : null;

  int get currentIndex => _currentIndex;
  bool get isCompleted => _currentIndex >= questions.length;
  double get progress => questions.isEmpty
      ? 0
      : _currentIndex / questions.length;

  void processEvent(QuizEvent event) {
    switch (event) {
      case AnswerQuestion(:final questionId, :final selectedIndex):
        _answers[questionId] = selectedIndex;
        _currentIndex++;

      case ResetQuiz():
        _answers.clear();
        _currentIndex = 0;
        _skippedCount = 0;

      case SkipQuestion():
        _skippedCount++;
        _currentIndex++;
    }
  }

  QuizResult getResult() {
    final duration = DateTime.now().difference(_startTime);

    final correctAnswers = questions
        .where((q) {
          final answer = _answers[q.id];
          return answer != null && q.isCorrect(answer);
        })
        .length;

    final earnedPoints = questions.fold<int>(
      0,
      (sum, q) {
        final answer = _answers[q.id];
        return sum + (answer != null && q.isCorrect(answer) ? q.points : 0);
      },
    );

    final totalPoints = questions.fold<int>(
      0,
      (sum, q) => sum + q.points,
    );

    return QuizResult(
      totalQuestions: questions.length,
      correctAnswers: correctAnswers,
      totalPoints: totalPoints,
      earnedPoints: earnedPoints,
      duration: duration,
    );
  }
}

// Test engine
void main() {
  final questions = [
    const Question(
      id: 'q1',
      text: 'Flutter dùng ngôn ngữ lập trình nào?',
      options: ['Kotlin', 'Swift', 'Dart', 'JavaScript'],
      correctIndex: 2,
      explanation: 'Flutter được viết bằng Dart.',
    ),
    const Question(
      id: 'q2',
      text: 'Widget nào dùng cho danh sách dài?',
      options: ['Column', 'ListView.builder', 'GridView', 'Stack'],
      correctIndex: 1,
      explanation: 'ListView.builder lazy-load widget, tiết kiệm bộ nhớ.',
    ),
    const Question(
      id: 'q3',
      text: 'Riverpod dùng để làm gì?',
      options: ['HTTP request', 'State management', 'Navigation', 'Animation'],
      correctIndex: 1,
    ),
  ];

  final engine = QuizEngine(questions: questions);

  // Simulate answering
  engine.processEvent(const AnswerQuestion('q1', 2)); // Đúng
  engine.processEvent(const AnswerQuestion('q2', 0)); // Sai
  engine.processEvent(const AnswerQuestion('q3', 1)); // Đúng

  final result = engine.getResult();
  print(result); // Kết quả: 2/3 đúng (20/30 điểm) - Loại C
  print('Thời gian: ${result.duration.inSeconds}s');
  print('Độ chính xác: ${result.accuracy.toStringAsFixed(1)}%');
}
```

---

## Tóm Tắt Chương 1

| Chủ đề | Điểm Cốt Lõi |
|---|---|
| Biến | `const` compile-time, `final` runtime one-time, `var` mutable |
| Null safety | Non-nullable mặc định; `?` nullable; `??`, `?.`, `!`, `late` |
| String | Interpolation `${}`, immutable, nhiều method built-in |
| List | `where` filter, `map` transform, `fold` accumulate — không dùng vòng lặp thủ công |
| Map | Key-value, nullable access, fromIterable để convert List |
| Named params | `{required this.x}` — pattern chuẩn trong Flutter widget |
| Class | factory constructor, getter, copyWith, toJson/fromJson |
| Mixin | Tái sử dụng behavior linh hoạt, không cần đa kế thừa |
| Sealed class | Exhaustive pattern matching, không cần default case |
| Future | `async/await`, `Future.wait` song song, try-catch xử lý lỗi |
| Stream | Dữ liệu realtime, phải cancel subscription để tránh memory leak |

> **Lời khuyên học tập:** Dart cú pháp gần với Java và TypeScript — nếu đã biết TypeScript, phần lớn Dart sẽ quen thuộc. Tập trung vào **null safety** và **named parameters** vì đây là hai điểm khác biệt lớn nhất mà Flutter khai thác triệt để. Mọi Flutter widget đều dùng named parameters và null safety nghiêm ngặt.