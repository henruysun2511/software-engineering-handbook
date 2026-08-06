# Chương 5: NestJS Core

## Giới thiệu

**NestJS** là một framework backend cho Node.js, xây dựng trên nền TypeScript, lấy cảm hứng từ kiến trúc module hóa của Angular. Nếu Chương 3 trình bày các mô hình kiến trúc ở mức lý thuyết (Layered Architecture, Clean Architecture...), thì chương này cho thấy **NestJS hiện thực hóa các mô hình đó thành công cụ cụ thể** ra sao — mỗi thành phần của NestJS (Module, Guard, Pipe...) đều là câu trả lời trực tiếp cho một nguyên tắc thiết kế đã học ở Chương 2 và 3.

Chương này đi từ bức tranh tổng thể (một request đi qua những lớp nào) đến chi tiết từng thành phần. Từ chương tiếp theo trở đi, mọi ví dụ trong tài liệu đều dùng NestJS để đảm bảo tính nhất quán.

---

## 5.1. Kiến trúc tổng thể — Vòng đời một Request trong NestJS

### 5.1.1. Bản chất

Trước khi đi vào từng thành phần riêng lẻ, cần hiểu **bức tranh toàn cảnh**: một request HTTP khi đi vào một ứng dụng NestJS không "rơi thẳng" vào Controller, mà phải đi qua một chuỗi các lớp xử lý trung gian, mỗi lớp đảm nhiệm **một trách nhiệm rõ ràng, tách biệt** — đây chính là hiện thực hóa trực tiếp của nguyên tắc **Single Responsibility** (Chương 2) ở cấp độ framework: thay vì nhét mọi logic (xác thực, validate, log, xử lý lỗi...) vào một chỗ, NestJS buộc lập trình viên đặt đúng logic vào đúng lớp có trách nhiệm tương ứng.

### 5.1.2. Sơ đồ vòng đời request

```mermaid
flowchart TD
    A[Client Request] --> B[Middleware]
    B --> C[Guard]
    C -->|Từ chối| C1[401 / 403]
    C -->|Cho phép| D["Interceptor (trước)"]
    D --> E[Pipe]
    E -->|Dữ liệu không hợp lệ| E1[400 Bad Request]
    E -->|Hợp lệ| F[Controller]
    F --> G[Service]
    G --> H[Repository / Database]
    H --> G
    G --> F
    F --> I["Interceptor (sau)"]
    I --> J[Response trả về Client]

    C1 -.-> K[Exception Filter]
    E1 -.-> K
    F -.Lỗi xảy ra.-> K
    G -.Lỗi xảy ra.-> K
    K --> J
```

### 5.1.3. Vai trò tổng quan của từng lớp

Bảng dưới đây là bản đồ tổng quan — chi tiết từng thành phần sẽ được trình bày ở các mục tiếp theo trong chương:

| Thứ tự | Lớp | Câu hỏi lớp này trả lời | Ví dụ trách nhiệm |
|---|---|---|---|
| 1 | **Middleware** | Có gì cần xử lý chung trước khi biết route nào sẽ được gọi? | Ghi log request, đọc thông tin cơ bản (IP, header) |
| 2 | **Guard** | Request này có được phép đi tiếp không? | Kiểm tra token hợp lệ (Authentication), kiểm tra quyền hạn (Authorization) |
| 3 | **Interceptor (trước)** | Có cần can thiệp gì trước khi Controller xử lý? | Đo thời gian bắt đầu xử lý, cache lookup |
| 4 | **Pipe** | Dữ liệu đầu vào có đúng định dạng không, có cần biến đổi không? | Validate DTO, chuyển `string` thành `number` |
| 5 | **Controller** | Request này tương ứng với hành động nghiệp vụ nào? | Nhận dữ liệu, gọi đúng Service, không xử lý logic |
| 6 | **Service** | Logic nghiệp vụ thực sự cần làm gì? | Tính toán, điều phối gọi Repository, gọi service khác |
| 7 | **Interceptor (sau)** | Response cần được biến đổi hay ghi nhận gì trước khi trả về? | Log thời gian xử lý, định dạng lại response |
| 8 | **Exception Filter** | Nếu có lỗi ở bất kỳ bước nào phía trên, phản hồi client ra sao? | Bắt lỗi, định dạng response lỗi thống nhất |

### 5.1.4. Vì sao thứ tự này lại quan trọng

Thứ tự các lớp không phải ngẫu nhiên — mỗi lớp được đặt trước lớp sau vì nó cần trả lời câu hỏi "rẻ hơn, sớm hơn" trước khi hệ thống tốn công sức cho các bước xử lý tốn kém hơn:

- **Guard** được đặt trước Pipe và Controller: không có lý do gì để validate dữ liệu hay chạy logic nghiệp vụ cho một request mà bản thân người gửi còn chưa được xác thực hoặc không có quyền — chặn sớm giúp tiết kiệm tài nguyên.
- **Pipe** được đặt trước Controller: Controller và Service chỉ nên làm việc với dữ liệu **đã chắc chắn hợp lệ**, đúng như nguyên tắc validate ở ranh giới đã trình bày tại Chương 6.
- **Exception Filter** đứng ngoài chuỗi tuyến tính, đóng vai trò "lưới an toàn" bao trùm toàn bộ vòng đời — bất kỳ lớp nào phía trên ném ra lỗi, request đều được dẫn về đây để xử lý thống nhất, thay vì mỗi lớp tự xử lý lỗi theo cách riêng.

---

## 5.2. Module

### 5.2.1. Bản chất

**Module** là đơn vị đóng gói một nhóm chức năng có liên quan chặt chẽ với nhau (ví dụ: mọi thứ liên quan đến "user" nằm trong `UserModule`). Về bản chất, Module là câu trả lời của NestJS cho vấn đề: khi ứng dụng lớn dần, làm sao để **giữ ranh giới rõ ràng giữa các domain nghiệp vụ khác nhau**, tránh tình trạng mọi thứ trộn lẫn vào một khối code duy nhất không thể tách rời.

Mỗi ứng dụng NestJS luôn có ít nhất một Module gốc (`AppModule`), và các Module con được import vào để tạo thành một cây phụ thuộc.

```ts
// user.module.ts
import { Module } from '@nestjs/common';
import { UserController } from './user.controller';
import { UserService } from './user.service';

@Module({
  imports: [],                    // các Module khác mà Module này cần dùng
  controllers: [UserController],  // Controller thuộc Module này
  providers: [UserService],       // Provider thuộc Module này
  exports: [UserService],         // cho phép Module khác sử dụng UserService
})
export class UserModule {}
```

### 5.2.2. Encapsulation ở cấp Module

**Điểm cốt lõi dễ bị bỏ qua**: mặc định, mọi Provider khai báo trong một Module chỉ **có thể nhìn thấy trong nội bộ Module đó**. Nếu `OrderModule` cần dùng `UserService`, `UserModule` phải chủ động `exports` nó ra, và `OrderModule` phải `imports` `UserModule` vào. Cơ chế này chính là **Encapsulation** (Chương 2) áp dụng ở quy mô toàn ứng dụng — mỗi Module tự quyết định phần nào của mình được phép dùng từ bên ngoài, phần nào là chi tiết nội bộ.

---

## 5.3. Controller

### 5.3.1. Bản chất

Controller là điểm tiếp nhận request từ client sau khi đã đi qua Middleware, Guard, Interceptor và Pipe. Đúng theo nguyên tắc **Thin Controller** (Chương 3), Controller chỉ nên đóng vai trò **điều phối** — nhận dữ liệu đã được đảm bảo hợp lệ, gọi đến đúng Service, trả kết quả về — tuyệt đối không chứa logic nghiệp vụ.

```ts
// user.controller.ts
@Controller('users')
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.userService.findOne(id);
  }

  @Post()
  create(@Body() dto: CreateUserDto) {
    return this.userService.create(dto);
  }
}
```

`@Controller('users')` khai báo tiền tố route chung (`/users`), các decorator `@Get`, `@Post`... ứng với từng HTTP Method đã trình bày ở Chương 1 và 3.

---

## 5.4. Provider

### 5.4.1. Bản chất

**Provider** là khái niệm tổng quát nhất trong NestJS cho bất kỳ class nào có thể được **quản lý vòng đời và "tiêm" vào nơi khác** thông qua Dependency Injection (mục 5.6). Service là loại Provider phổ biến nhất, nhưng Provider còn bao gồm cả Repository, Factory, hay bất kỳ class tiện ích nào được đánh dấu `@Injectable()`.

**Bản chất then chốt**: "Provider" là tên gọi theo *vai trò trong hệ thống DI*, còn "Service" là tên gọi theo *ý nghĩa nghiệp vụ*. Mọi Service đều là Provider, nhưng không phải mọi Provider đều nên được gọi là Service (ví dụ một Repository cũng là Provider nhưng không mang ý nghĩa "xử lý nghiệp vụ").

```ts
@Injectable()
export class UserRepository {
  // đây cũng là một Provider, dù không gọi là "Service"
}
```

---

## 5.5. Service

### 5.5.1. Bản chất

Service là nơi **logic nghiệp vụ thực sự** được đặt — đúng phần "Fat Service" trong nguyên tắc Thin Controller - Fat Service đã trình bày ở Chương 3. Vì Service hoàn toàn không phụ thuộc vào khái niệm HTTP (không biết gì về `Request`, `Response`), nó có thể được gọi từ bất kỳ đâu: từ Controller, từ một Cron Job (Chương 7), từ một Queue Worker (Chương 7) — và quan trọng không kém, nó dễ dàng được kiểm thử độc lập (Unit Test, Chương 10) mà không cần giả lập một request HTTP giả.

```ts
@Injectable()
export class UserService {
  constructor(private userRepository: UserRepository) {}

  async findOne(id: string) {
    const user = await this.userRepository.findById(id);
    if (!user) throw new NotFoundException('Không tìm thấy người dùng');
    return user;
  }
}
```

---

## 5.6. Dependency Injection

### 5.6.1. Bản chất

Dependency Injection (DI) đã được giới thiệu ở Chương 2 như một Design Pattern hiện thực hóa nguyên tắc Dependency Inversion. NestJS đưa DI trở thành **cơ chế lõi của toàn bộ framework** thông qua **IoC Container** — một bộ quản lý tự động chịu trách nhiệm khởi tạo, lưu trữ, và cung cấp đúng instance của mỗi Provider cho nơi cần dùng đến nó.

```mermaid
flowchart LR
    A["@Injectable()<br/>UserService"] -->|Đăng ký vào| B[IoC Container]
    B -->|Tự động cung cấp khi cần| C["UserController<br/>constructor(userService)"]
```

```ts
@Injectable()
export class UserController {
  // NestJS tự động tra cứu trong IoC Container và truyền UserService vào đây
  constructor(private readonly userService: UserService) {}
}
```

### 5.6.2. Phạm vi (Scope) của Provider

Mặc định, mỗi Provider trong NestJS có phạm vi **Singleton** — chỉ được khởi tạo **một lần duy nhất** cho toàn bộ vòng đời ứng dụng, và mọi nơi cần đến nó đều dùng chung một instance. Đây là lựa chọn hợp lý cho phần lớn trường hợp (tiết kiệm tài nguyên, tránh khởi tạo lặp lại không cần thiết), nhưng NestJS cũng cho phép cấu hình phạm vi `Request` (một instance mới cho mỗi request) hoặc `Transient` (một instance mới mỗi lần được inject) khi nghiệp vụ thực sự cần trạng thái riêng biệt.

### 5.6.3. Lợi ích cốt lõi

- **Giảm ràng buộc cứng**: Controller không tự tạo `new UserService()`, nên có thể dễ dàng thay thế bằng một implementation khác (ví dụ mock Service khi viết Unit Test ở Chương 10).
- **Tái sử dụng instance**: nhờ phạm vi Singleton mặc định, tránh lãng phí tài nguyên khởi tạo lặp lại.
- **Tuân thủ Dependency Inversion**: các class cấp cao (Controller, Service) chỉ phụ thuộc vào interface/abstraction, không phụ thuộc vào cách một Provider cụ thể được khởi tạo.

---

## 5.7. Middleware

### 5.7.1. Bản chất

Middleware là lớp đầu tiên mà request đi qua, kế thừa trực tiếp khái niệm middleware của Express. Đặc điểm bản chất của Middleware là nó hoạt động **trước khi NestJS xác định route nào sẽ được gọi** — vì vậy Middleware phù hợp cho các tác vụ **hoàn toàn chung**, không cần biết ngữ cảnh cụ thể của route đích (ví dụ: ghi log mọi request, đọc thông tin cơ bản từ header).

```ts
@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log(`${req.method} ${req.originalUrl}`);
    next(); // bắt buộc gọi để chuyển tiếp sang lớp kế tiếp
  }
}
```

```ts
// app.module.ts
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes('*');
  }
}
```

---

## 5.8. Pipe

### 5.8.1. Bản chất

Pipe đứng ngay trước Controller, với trách nhiệm duy nhất: đảm bảo dữ liệu đi vào Controller **đã đúng định dạng và đã được biến đổi về đúng kiểu mong muốn** — đây chính là điểm triển khai cụ thể của DTO Validation đã trình bày sâu ở Chương 6.

```ts
// create-user.dto.ts
export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(6)
  password: string;
}
```

```ts
// main.ts — áp dụng ValidationPipe cho toàn ứng dụng
app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
```

**Bản chất của tên gọi "Pipe"**: dữ liệu đi qua như qua một đường ống — Pipe có thể vừa kiểm tra tính hợp lệ, vừa **biến đổi** dữ liệu (ví dụ `ParseIntPipe` tự động chuyển tham số dạng chuỗi `"5"` trên URL thành kiểu `number`) trước khi nó đến tay Controller.

---

## 5.9. Guard

### 5.9.1. Bản chất

Guard trả lời chính xác một câu hỏi: **"request này có được phép đi tiếp hay không?"** — hiện thực hóa cả Authentication lẫn Authorization đã trình bày ở Chương 8. Guard được đặt trước Interceptor và Pipe vì lý do đã giải thích ở mục 5.1: không nên tốn công xử lý dữ liệu cho một request chưa có quyền truy cập.

```ts
@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    return !!request.headers['authorization'];
  }
}
```

```ts
@UseGuards(AuthGuard)
@Get('profile')
getProfile() {
  return 'protected data';
}
```

---

## 5.10. Interceptor

### 5.10.1. Bản chất

Interceptor là thành phần linh hoạt nhất trong chuỗi xử lý, vì nó **bao quanh** toàn bộ quá trình Controller xử lý — có thể can thiệp cả **trước khi** Controller chạy lẫn **sau khi** Controller đã trả về kết quả, nhờ dựa trên mô hình RxJS Observable.

```mermaid
flowchart LR
    A[Request vào] --> B["Interceptor:<br/>logic TRƯỚC"]
    B --> C[Controller xử lý]
    C --> D["Interceptor:<br/>logic SAU (trên response)"]
    D --> E[Response ra]
```

```ts
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler) {
    const now = Date.now(); // logic TRƯỚC
    return next.handle().pipe(
      tap(() => console.log(`Xử lý mất ${Date.now() - now}ms`)), // logic SAU
    );
  }
}
```

**Ứng dụng phổ biến**: đo thời gian xử lý, định dạng lại response theo cấu trúc chung (ví dụ tự động bọc mọi response vào `{ data, timestamp }`), triển khai cache ở tầng ứng dụng.

---

## 5.11. Exception Filter

### 5.11.1. Bản chất

Exception Filter là "lưới an toàn" cuối cùng — như đã thấy trong sơ đồ ở mục 5.1, bất kể lỗi phát sinh từ lớp nào (Guard từ chối, Pipe validate thất bại, Service ném lỗi nghiệp vụ, hay lỗi hệ thống không lường trước), request đều được dẫn về đây để **định dạng lại thành một response lỗi thống nhất**, thay vì mỗi nơi tự trả lỗi theo cấu trúc riêng.

```ts
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const response = host.switchToHttp().getResponse();
    const status =
      exception instanceof HttpException ? exception.getStatus() : 500;
    const message =
      exception instanceof HttpException ? exception.message : 'Lỗi hệ thống';

    response.status(status).json({
      statusCode: status,
      message,
      timestamp: new Date().toISOString(),
    });
  }
}
```

Đây là ứng dụng trực tiếp trong NestJS của nguyên tắc Error Handling đã trình bày ở Chương 7.

---

## 5.12. Custom Decorator

### 5.12.1. Bản chất

**Decorator** là tính năng của TypeScript cho phép gắn thêm metadata hoặc hành vi vào class, method, property mà không thay đổi logic gốc — bản thân `@Controller`, `@Injectable`, `@Get` đều là decorator do NestJS cung cấp sẵn. **Custom Decorator** cho phép lập trình viên tự định nghĩa decorator riêng nhằm loại bỏ code lặp lại rải rác khắp nơi.

```ts
// current-user.decorator.ts
export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user; // đã được Guard gắn vào trước đó
  },
);
```

```ts
@Get('me')
getMe(@CurrentUser() user: User) {
  return user; // thay vì phải viết request.user lặp lại ở mọi Controller
}
```

**Bản chất giá trị mang lại**: giảm code lặp, tăng khả năng đọc hiểu ý định (`@CurrentUser()` tự giải thích rõ hơn nhiều so với `request.user`), và tách biệt rõ ràng cách lấy dữ liệu khỏi logic nghiệp vụ trong Controller.

---

## 5.13. Configuration Module

### 5.13.1. Bản chất

Ứng dụng backend cần đọc các giá trị cấu hình thay đổi theo môi trường (địa chỉ database, secret key, cổng chạy server...) mà **không được phép hard-code trực tiếp trong source code** — vừa để bảo mật (không lộ secret trong mã nguồn), vừa để cùng một codebase có thể chạy đúng ở nhiều môi trường khác nhau (dev, staging, production) chỉ bằng cách thay đổi cấu hình. `@nestjs/config` là module chính thức của NestJS giải quyết bài toán này, đọc giá trị từ file `.env`.

```
# .env
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
JWT_SECRET=supersecret
PORT=3000
```

```ts
// app.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true, // dùng được ở mọi Module khác mà không cần import lại
    }),
  ],
})
export class AppModule {}
```

```ts
@Injectable()
export class DatabaseService {
  constructor(private configService: ConfigService) {
    const url = this.configService.get<string>('DATABASE_URL');
  }
}
```

Việc quản lý cấu hình một cách bài bản theo từng môi trường triển khai (dev/staging/production) và quản lý secrets an toàn sẽ được trình bày sâu hơn ở Chương 9, trong phần Configuration & Environment.

---

## Tổng kết chương

Chương này trình bày NestJS không phải như một danh sách API rời rạc, mà như một **hệ thống các lớp trách nhiệm được tổ chức có chủ đích**, phản chiếu trực tiếp các nguyên tắc thiết kế đã học ở Chương 2 và 3: Module hiện thực hóa việc phân chia ranh giới domain và Encapsulation; chuỗi Middleware → Guard → Interceptor → Pipe → Controller → Service tuân thủ Single Responsibility, mỗi lớp chỉ trả lời đúng một câu hỏi; Dependency Injection là hiện thực hóa trực tiếp của Dependency Inversion Principle; và Exception Filter đóng vai trò lưới an toàn thống nhất cho toàn hệ thống. Nắm được bản đồ tổng thể này quan trọng hơn nhiều so với việc ghi nhớ cú pháp từng decorator — vì nó giúp trả lời đúng câu hỏi "logic này nên đặt ở lớp nào" trong mọi tình huống thực tế sau này.