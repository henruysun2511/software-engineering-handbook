# CƠ CHẾ HOẠT ĐỘNG CỦA NEST.JS — SO SÁNH VÀ ƯU ĐIỂM

## I. Đặt vấn đề

Express.js linh hoạt nhưng không áp đặt cấu trúc: lập trình viên tự do tổ chức thư mục, middleware, route theo ý mình. Khi dự án lớn dần (hàng trăm route, nhiều team cùng phát triển), sự tự do này lại trở thành nhược điểm — code dễ rối, khó tái sử dụng, khó kiểm thử.

NestJS ra đời để giải quyết vấn đề đó: đây không phải một framework thay thế Express, mà là một lớp kiến trúc (architectural framework) xây dựng trên nền Express (mặc định) hoặc Fastify, mang tư duy tổ chức module hóa của Angular vào phía backend Node.js.

---

## II. Bản chất của NestJS

### 1. NestJS là một lớp trừu tượng trên Express/Fastify
Về bản chất, NestJS không tự viết lại cơ chế xử lý HTTP. Nó dùng Express (hoặc Fastify) làm "động cơ" (HTTP adapter) bên dưới để nhận request, quản lý middleware, gửi response — đúng như cơ chế đã trình bày ở phần Express.js. Điều NestJS bổ sung là một kiến trúc bắt buộc phía trên lớp đó, được xây dựng hoàn toàn bằng TypeScript, dựa trên 3 trụ cột:
- **Decorators** (`@Controller`, `@Injectable`, `@Module`...)
- **Dependency Injection (DI)** — tiêm phụ thuộc tự động
- **Modular architecture** — chia ứng dụng thành các module độc lập, có thể tái sử dụng

### 2. Ba thành phần cốt lõi

| Thành phần | Vai trò | Ví dụ |
| :--- | :--- | :--- |
| **Module** | Đơn vị đóng gói, nhóm các controller/provider liên quan | `@Module({ controllers: [...], providers: [...] })` |
| **Controller** | Nhận request, định tuyến (tương đương route handler ở Express) | `@Controller('users')` |
| **Provider (Service)** | Chứa logic nghiệp vụ, có thể "tiêm" vào nơi khác | `@Injectable()` |

```typescript
// user.service.ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class UserService {
  findOne(id: string) { return { id, name: 'Nguyen Van A' }; }
}
```

```typescript
// user.controller.ts
import { Controller, Get, Param } from '@nestjs/common';
import { UserService } from './user.service';

@Controller('users')
export class UserController {
  constructor(private readonly userService: UserService) {} // DI tự động

  @Get(':id')
  getUser(@Param('id') id: string) {
    return this.userService.findOne(id);
  }
}
```

```typescript
// user.module.ts
import { Module } from '@nestjs/common';
import { UserController } from './user.controller';
import { UserService } from './user.service';

@Module({
  controllers: [UserController],
  providers: [UserService],
})
export class UserModule {}
```

### 3. Dependency Injection — bản chất kỹ thuật
Đây là điểm khác biệt cốt lõi nhất so với Express. NestJS duy trì một **IoC Container** (Inversion of Control — vùng chứa điều khiển ngược): thay vì `UserController` tự tạo `new UserService()`, nó chỉ khai báo phụ thuộc trong constructor; Nest tự động khởi tạo và "tiêm" instance `UserService` vào đúng lúc.

Cơ chế này hoạt động dựa trên hai công nghệ TypeScript:
- **Decorators**: Gắn metadata vào class (ví dụ đánh dấu `UserService` là có thể tiêm được bằng `@Injectable()`).
- **Reflect Metadata**: NestJS đọc metadata này lúc runtime để biết constructor của `UserController` cần loại đối tượng nào, từ đó tự động khởi tạo và truyền vào — toàn bộ diễn ra ẩn, lập trình viên không cần tự `new` bất kỳ service nào.

> **Lợi ích trực tiếp**: Dễ thay thế (swap) một implementation (ví dụ đổi service thật bằng mock khi test), dễ tái sử dụng, và giảm sự phụ thuộc chặt (tight coupling) giữa các lớp.

---

## III. Luồng xử lý một request trong NestJS

So với luồng middleware "phẳng" của Express, NestJS chia luồng xử lý thành nhiều lớp có thứ tự rõ ràng, mỗi lớp đảm nhiệm một trách nhiệm riêng:

```text
Request
   │
   ▼
┌─────────────┐
│ Middleware   │  (giống Express, chạy trước khi vào routing)
└──────┬──────┘
       ▼
┌─────────────┐
│   Guards     │  (kiểm tra quyền truy cập: AuthGuard, RolesGuard...)
└──────┬──────┘
       ▼
┌─────────────────┐
│ Interceptors     │  (trước) — log, transform trước khi vào handler
└──────┬───────────┘
       ▼
┌─────────────┐
│   Pipes      │  (validate & transform dữ liệu đầu vào: ValidationPipe...)
└──────┬──────┘
       ▼
┌─────────────────┐
│ Controller Handler│ (logic nghiệp vụ chính, gọi Service)
└──────┬───────────┘
       ▼
┌─────────────────┐
│ Interceptors     │  (sau) — biến đổi response trước khi trả về
└──────┬───────────┘
       ▼
┌─────────────────┐
│ Exception Filters │ (chỉ kích hoạt nếu có lỗi ném ra ở bất kỳ bước nào)
└──────┬───────────┘
       ▼
   Response về client
```

Mỗi lớp (Guard, Pipe, Interceptor, Filter) đều có thể áp dụng ở 3 cấp độ: **toàn cục (global)**, **theo controller**, hoặc **theo từng route** — cho phép kiểm soát rất chi tiết mà vẫn giữ code controller "sạch", chỉ chứa logic nghiệp vụ thuần túy.

---

## IV. So sánh NestJS và Express.js

| Tiêu chí | Express.js | NestJS |
| :--- | :--- | :--- |
| **Bản chất** | Thư viện định tuyến + middleware tối giản | Framework kiến trúc đầy đủ, chạy trên Express/Fastify |
| **Ngôn ngữ** | JavaScript (TypeScript tùy chọn, không bắt buộc) | TypeScript mặc định (là trung tâm thiết kế) |
| **Cấu trúc dự án** | Tự do, không quy chuẩn | Bắt buộc theo Module - Controller - Provider |
| **Quản lý phụ thuộc** | Thủ công (tự require/new) | Dependency Injection tự động (IoC Container) |
| **Xử lý luồng request** | Một chuỗi middleware tuyến tính | Nhiều lớp: Middleware $\rightarrow$ Guard $\rightarrow$ Interceptor $\rightarrow$ Pipe $\rightarrow$ Filter |
| **Validate dữ liệu** | Cần thư viện ngoài, viết thủ công | Có sẵn Pipes + tích hợp `class-validator` |
| **Kiểm thử (testing)** | Phải tự mock phụ thuộc | DI giúp mock/inject dễ dàng, có sẵn `@nestjs/testing` |
| **Độ linh hoạt** | Rất cao, không ràng buộc | Có ràng buộc quy ước, đổi lại tính nhất quán |
| **Đường cong học tập** | Thấp, dễ bắt đầu | Cao hơn do phải hiểu DI, decorator, kiến trúc module |
| **Phù hợp với** | Dự án nhỏ, API đơn giản, cần tốc độ dựng nhanh | Dự án lớn, nhiều team, cần khả năng mở rộng và bảo trì lâu dài |

---

## V. Ưu điểm nổi bật của NestJS

1. **Kiến trúc rõ ràng, nhất quán**: Mọi thành viên trong team buộc phải tổ chức code theo cùng một khuôn mẫu (Module - Controller - Service), giảm sự khác biệt phong cách giữa các lập trình viên.
2. **Dependency Injection giúp code dễ kiểm thử và mở rộng**: Có thể thay thế một Service bằng phiên bản giả (mock) mà không cần sửa Controller, rất thuận lợi cho unit test.
3. **Tận dụng tối đa TypeScript**: Kiểm tra kiểu dữ liệu tại thời điểm biên dịch, giảm lỗi runtime, hỗ trợ tự động gợi ý (autocomplete) tốt hơn nhiều so với JavaScript thuần.
4. **Hệ sinh thái tích hợp sẵn phong phú**: Hỗ trợ có sẵn cho GraphQL, WebSocket, Microservices, gRPC, TypeORM/Mongoose, Swagger — không cần tự ráp nối như khi dùng Express thuần.
5. **Có thể chuyển đổi HTTP platform**: Nhờ lớp trừu tượng adapter, có thể đổi từ Express sang Fastify (hiệu năng cao hơn) mà hầu như không phải sửa code nghiệp vụ.
6. **Dễ áp dụng các nguyên lý thiết kế phần mềm**: Cấu trúc của NestJS tự nhiên khuyến khích tuân theo SOLID, đặc biệt là nguyên lý đảo ngược phụ thuộc (Dependency Inversion) — điều mà Express không hỗ trợ sẵn.
7. **Khả năng mở rộng tốt cho hệ thống lớn**: Mô hình module hóa giúp chia nhỏ ứng dụng thành các domain độc lập, thuận lợi khi tách thành microservices sau này.
