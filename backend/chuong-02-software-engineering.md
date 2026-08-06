# Chương 2: Software Engineering Fundamentals

## Giới thiệu

Trước khi tìm hiểu các mẫu kiến trúc backend cụ thể ở Chương 3, cần trang bị một nền tảng tư duy chung: **lập trình hướng đối tượng (OOP)**, các **Design Pattern** phổ biến, và nguyên tắc **SOLID**. Đây không phải là kiến thức riêng của backend, mà là ngôn ngữ tư duy chung của kỹ thuật phần mềm — nhưng lại là nền móng trực tiếp cho mọi quyết định kiến trúc sẽ được trình bày ở các chương sau, đặc biệt là lý do NestJS được thiết kế theo cách nó đang có.

---

## 2.1. OOP (Lập trình hướng đối tượng)

### 2.1.1. Khái niệm

**Bản chất của OOP** không nằm ở việc "dùng class thay vì function" — đó chỉ là công cụ. Bản chất thực sự là một cách **tổ chức code theo mô hình thế giới thực**: thay vì viết code như một chuỗi lệnh xử lý dữ liệu rời rạc, OOP nhóm **dữ liệu** và **hành vi thao tác trên dữ liệu đó** lại thành một thực thể duy nhất gọi là **đối tượng (object)**.

Ví dụ: một `User` không chỉ là một tập hợp các trường dữ liệu (`name`, `email`), mà còn có thể tự mình thực hiện các hành vi liên quan (`changePassword()`, `verifyEmail()`) — dữ liệu và logic thao tác trên dữ liệu đó nằm cùng một chỗ, thay vì logic nằm rải rác ở những hàm riêng biệt thao tác lên một cấu trúc dữ liệu thuần túy.

OOP xây dựng trên bốn tính chất cốt lõi, được trình bày lần lượt dưới đây.

### 2.1.2. Encapsulation (Tính đóng gói)

**Bản chất**: Encapsulation là việc **giới hạn quyền truy cập trực tiếp vào dữ liệu nội bộ** của một đối tượng, chỉ cho phép thao tác với dữ liệu đó thông qua các phương thức được đối tượng chủ động cung cấp. Mục đích cốt lõi không phải để "giấu thông tin" một cách hình thức, mà để đối tượng có thể **tự bảo vệ tính hợp lệ của chính dữ liệu của nó** — không ai được phép đặt đối tượng vào một trạng thái sai lệch mà đối tượng không kiểm soát được.

```ts
class BankAccount {
  private balance: number; // không ai truy cập trực tiếp từ bên ngoài

  withdraw(amount: number) {
    if (amount > this.balance) {
      throw new Error('Số dư không đủ');
    }
    this.balance -= amount;
  }
}
```

Nếu `balance` là `public`, bất kỳ đoạn code nào cũng có thể gán `account.balance = -1000` một cách tùy tiện, phá vỡ tính hợp lệ của dữ liệu mà không qua bất kỳ kiểm tra nào — đây chính xác là vấn đề mà Encapsulation ngăn chặn.

### 2.1.3. Inheritance (Tính kế thừa)

**Bản chất**: Inheritance cho phép một class **tái sử dụng** thuộc tính và hành vi đã định nghĩa ở một class khác, thay vì phải viết lại từ đầu — dựa trên quan hệ "là một loại của" (is-a). Một `AdminUser` **là một loại** `User`, nên nó kế thừa mọi thứ `User` đã có, và chỉ cần bổ sung thêm phần khác biệt.

```ts
class User {
  constructor(protected name: string) {}
  getName() { return this.name; }
}

class AdminUser extends User {
  banUser(targetUser: User) { /* logic riêng của Admin */ }
}
```

**Lưu ý về bản chất**: Inheritance nên được dùng khi mối quan hệ "là một loại của" thực sự đúng về mặt logic — dùng sai (kế thừa chỉ để tái sử dụng code dù quan hệ không thực sự là is-a) sẽ tạo ra cấu trúc class cứng nhắc và khó bảo trì. Đây là lý do nguyên tắc hiện đại thường khuyến khích ưu tiên **Composition** (một đối tượng "chứa" đối tượng khác) hơn Inheritance khi có thể.

### 2.1.4. Polymorphism (Tính đa hình)

**Bản chất**: Polymorphism cho phép các đối tượng thuộc **các class khác nhau** được xử lý thông qua **cùng một interface chung**, mỗi đối tượng tự quyết định cách nó phản hồi lại lời gọi đó theo cách riêng của mình. Bản chất cốt lõi là: **code gọi hàm không cần biết chính xác đối tượng đang xử lý thuộc loại cụ thể nào**, chỉ cần biết nó tuân thủ đúng interface.

```ts
interface PaymentMethod {
  pay(amount: number): void;
}

class CreditCardPayment implements PaymentMethod {
  pay(amount: number) { /* xử lý qua cổng thẻ tín dụng */ }
}

class MomoPayment implements PaymentMethod {
  pay(amount: number) { /* xử lý qua ví Momo */ }
}

function checkout(method: PaymentMethod, amount: number) {
  method.pay(amount); // không cần biết cụ thể là loại thanh toán nào
}
```

Đây chính là nền tảng để hệ thống có thể **mở rộng thêm phương thức thanh toán mới** (ví dụ thêm `ZaloPayPayment`) mà không cần sửa lại hàm `checkout` — nguyên tắc này sẽ được nhắc lại chính xác dưới tên "Open/Closed Principle" ở mục SOLID bên dưới.

### 2.1.5. Abstraction (Tính trừu tượng)

**Bản chất**: Abstraction là việc **chỉ phơi bày những gì cần thiết cho người dùng của một đối tượng**, che giấu đi chi tiết triển khai phức tạp bên trong. Khác với Encapsulation (tập trung vào việc *bảo vệ* dữ liệu), Abstraction tập trung vào việc **đơn giản hóa những gì thế giới bên ngoài cần biết** để sử dụng một đối tượng.

Ví dụ: khi gọi `userService.createUser(dto)`, người gọi không cần biết bên trong có bao nhiêu bước validate, mã hóa mật khẩu, ghi log — họ chỉ cần biết "gọi hàm này với dữ liệu này sẽ tạo ra user". Chi tiết triển khai được trừu tượng hóa đi.

### 2.1.6. Tóm tắt bốn tính chất

| Tính chất | Trả lời câu hỏi | Mục đích cốt lõi |
|---|---|---|
| Encapsulation | Ai được phép thay đổi dữ liệu này? | Bảo vệ tính hợp lệ của dữ liệu nội bộ |
| Inheritance | Đối tượng này có gì giống đối tượng khác? | Tái sử dụng thông qua quan hệ "là một loại của" |
| Polymorphism | Làm sao xử lý nhiều loại đối tượng bằng cùng một cách gọi? | Cho phép mở rộng mà không sửa code gọi |
| Abstraction | Người dùng đối tượng này cần biết những gì? | Đơn giản hóa giao diện sử dụng |

---

## 2.2. Design Pattern

### 2.2.1. Bản chất của Design Pattern

Design Pattern không phải là đoạn code có thể copy-paste, mà là **giải pháp đã được kiểm chứng cho một loại vấn đề lặp đi lặp lại trong thiết kế phần mềm**. Giá trị cốt lõi của việc học Design Pattern không nằm ở việc ghi nhớ cách triển khai, mà ở việc nhận diện được **khi nào một vấn đề đang gặp phải chính là biến thể của một vấn đề đã có lời giải chuẩn** — từ đó tránh việc tự nghĩ ra giải pháp riêng kém tối ưu hơn.

### 2.2.2. Dependency Injection (DI)

**Bản chất**: DI giải quyết vấn đề **một class tự tạo ra các đối tượng phụ thuộc của chính nó**, khiến nó bị **ràng buộc cứng (tightly coupled)** với một cách triển khai cụ thể.

```ts
// KHÔNG dùng DI — OrderService tự tạo EmailService bên trong
class OrderService {
  private emailService = new EmailService(); // ràng buộc cứng
}
```

Vấn đề: nếu muốn thay `EmailService` bằng một service giả (mock) để viết Unit Test (Chương 10), hoặc thay bằng một implementation khác, phải sửa trực tiếp bên trong `OrderService`.

```ts
// Dùng DI — dependency được "tiêm" từ bên ngoài vào
class OrderService {
  constructor(private emailService: EmailService) {} // nhận từ bên ngoài
}
```

Bản chất của DI là đảo ngược trách nhiệm: thay vì class tự đi tìm và tạo ra thứ nó cần, một bên thứ ba (ở NestJS là **IoC Container**, đã trình bày ở Chương 5) sẽ **cung cấp sẵn** cho nó. Đây là lý do DI còn được gọi là hiện thực hóa của nguyên tắc **Inversion of Control (Đảo ngược quyền điều khiển)**.

### 2.2.3. Repository Pattern

**Bản chất**: Repository Pattern tạo ra một lớp trung gian **che giấu chi tiết truy cập dữ liệu** (câu lệnh SQL, cách gọi ORM cụ thể) khỏi tầng logic nghiệp vụ (Service). Service chỉ cần gọi các phương thức có ý nghĩa nghiệp vụ (`findActiveUsers()`) mà không cần biết đằng sau đó là Prisma, TypeORM hay raw SQL.

```ts
interface UserRepository {
  findById(id: string): Promise<User>;
  findActiveUsers(): Promise<User[]>;
}
```

**Lợi ích cốt lõi**: nếu sau này quyết định đổi từ TypeORM sang Prisma (Chương 4), chỉ cần viết lại phần triển khai của Repository — toàn bộ Service phía trên **không cần thay đổi một dòng nào**, vì chúng chỉ phụ thuộc vào interface, không phụ thuộc vào cách triển khai cụ thể.

### 2.2.4. Factory Pattern

**Bản chất**: Factory Pattern tách riêng **logic khởi tạo đối tượng phức tạp** ra khỏi nơi sử dụng đối tượng đó. Khi việc quyết định "nên khởi tạo loại đối tượng nào" phụ thuộc vào điều kiện (input, cấu hình...), việc nhét toàn bộ logic điều kiện đó vào nơi sử dụng sẽ làm code rối và khó mở rộng.

```ts
class PaymentFactory {
  static create(method: string): PaymentMethod {
    switch (method) {
      case 'credit_card': return new CreditCardPayment();
      case 'momo': return new MomoPayment();
      default: throw new Error('Phương thức không được hỗ trợ');
    }
  }
}

const payment = PaymentFactory.create(userChoice); // nơi sử dụng không cần biết chi tiết khởi tạo
```

### 2.2.5. Strategy Pattern

**Bản chất**: Strategy Pattern định nghĩa một **họ các thuật toán/hành vi có thể hoán đổi cho nhau**, đóng gói mỗi thuật toán thành một class riêng tuân theo cùng một interface, cho phép chọn thuật toán cụ thể **tại thời điểm chạy (runtime)** thay vì cố định lúc viết code.

Về bản chất, Strategy Pattern chính là ứng dụng trực tiếp của **Polymorphism** (mục 2.1.4) vào một bài toán thiết kế cụ thể — ví dụ `PaymentMethod` ở phần OOP phía trên chính là một Strategy Pattern hoàn chỉnh.

**Vì sao NestJS dùng Strategy Pattern rất nhiều?** Cơ chế xác thực của NestJS (thông qua thư viện Passport, đã thấy ở Chương 8 với `JwtStrategy`) là một ví dụ điển hình: mỗi cách xác thực (JWT, Google OAuth, Local...) là một "Strategy" tuân theo cùng giao diện chung, Guard chỉ cần gọi đến strategy được chỉ định mà không cần biết chi tiết bên trong từng loại xác thực.

---

## 2.3. SOLID Principles

### 2.3.1. Bản chất chung

SOLID là năm nguyên tắc thiết kế hướng đối tượng, không phải là quy tắc bắt buộc tuân theo cứng nhắc, mà là **kim chỉ nam giúp đánh giá một thiết kế có dễ mở rộng và bảo trì hay không**. Nhiều Design Pattern ở mục 2.2 thực chất là cách hiện thực hóa cụ thể của các nguyên tắc SOLID.

### 2.3.2. S — Single Responsibility Principle (Nguyên tắc đơn nhiệm)

**Bản chất**: một class chỉ nên có **một lý do duy nhất để thay đổi**. Nếu một class vừa xử lý logic nghiệp vụ, vừa xử lý gửi email, vừa xử lý ghi log — bất kỳ thay đổi nào ở một trong ba mối quan tâm đó đều buộc phải sửa cùng một class, làm tăng rủi ro ảnh hưởng lan sang các phần không liên quan.

### 2.3.3. O — Open/Closed Principle (Nguyên tắc đóng/mở)

**Bản chất**: một class nên **mở để mở rộng, nhưng đóng để sửa đổi** — khi cần thêm hành vi mới, nên thêm code mới (ví dụ một class mới) thay vì sửa lại code đã có và đang hoạt động ổn định. Ví dụ `PaymentMethod` ở trên: thêm phương thức thanh toán mới chỉ cần thêm một class mới, không cần sửa hàm `checkout` đã có.

### 2.3.4. L — Liskov Substitution Principle (Nguyên tắc thay thế Liskov)

**Bản chất**: nếu class `B` kế thừa từ class `A`, thì ở bất kỳ đâu code đang dùng `A`, phải có thể thay bằng `B` **mà không làm sai lệch hành vi mong đợi**. Đây là tiêu chuẩn để đánh giá một quan hệ Inheritance (mục 2.1.3) có thực sự hợp lý hay không — nếu class con thay đổi hành vi theo cách phá vỡ kỳ vọng của class cha, đó là dấu hiệu Inheritance đã bị dùng sai.

### 2.3.5. I — Interface Segregation Principle (Nguyên tắc phân tách interface)

**Bản chất**: không nên ép một class phải triển khai những phương thức **nó không dùng đến**, chỉ vì chúng nằm chung trong một interface lớn. Nên chia thành nhiều interface nhỏ, chuyên biệt hơn, để mỗi class chỉ cần triển khai đúng những gì thực sự liên quan đến nó.

### 2.3.6. D — Dependency Inversion Principle (Nguyên tắc đảo ngược phụ thuộc)

**Bản chất**: các module cấp cao (chứa logic nghiệp vụ quan trọng) **không nên phụ thuộc trực tiếp** vào các module cấp thấp (chi tiết triển khai kỹ thuật cụ thể) — cả hai nên cùng phụ thuộc vào một **abstraction (interface) chung**. Đây chính là nguyên lý nền tảng đứng sau cả **Dependency Injection** lẫn **Repository Pattern** đã trình bày ở mục 2.2: Service (cấp cao) phụ thuộc vào interface `UserRepository` (abstraction), chứ không phụ thuộc trực tiếp vào chi tiết Prisma hay TypeORM (cấp thấp).

### 2.3.7. Tóm tắt

| Nguyên tắc | Câu hỏi cốt lõi | Pattern liên quan trực tiếp |
|---|---|---|
| Single Responsibility | Class này có đang làm nhiều hơn một việc không? | — |
| Open/Closed | Thêm tính năng mới có cần sửa code cũ không? | Strategy Pattern |
| Liskov Substitution | Class con có thay thế được class cha một cách an toàn không? | Inheritance |
| Interface Segregation | Class có bị ép triển khai thứ nó không cần không? | Repository Pattern (interface nhỏ, chuyên biệt) |
| Dependency Inversion | Module cấp cao có phụ thuộc trực tiếp vào chi tiết cấp thấp không? | Dependency Injection, Repository Pattern |

---

## Tổng kết chương

Chương này xây dựng nền tảng tư duy thiết kế phần mềm: OOP cung cấp bộ công cụ cơ bản để tổ chức dữ liệu và hành vi (Encapsulation, Inheritance, Polymorphism, Abstraction); Design Pattern là những giải pháp chuẩn cho các vấn đề thiết kế lặp lại, xây dựng trực tiếp trên nền OOP; còn SOLID là bộ tiêu chuẩn để đánh giá và định hướng các quyết định thiết kế đó. Điểm mấu chốt cần ghi nhớ là các khái niệm này **không tách rời nhau**: Dependency Injection là cách hiện thực hóa Dependency Inversion Principle, Strategy Pattern là một dạng cụ thể của Polymorphism, và Repository Pattern đồng thời phục vụ cả Dependency Inversion lẫn Interface Segregation. Đây chính là bộ nền tảng mà Chương 3 (Backend Architecture) và Chương 5 (NestJS Core) sẽ liên tục quay lại tham chiếu.