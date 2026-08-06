# Swagger / OpenAPI – Tổng hợp kiến thức (đặc biệt cho NestJS)

> Docs chính thức: https://docs.nestjs.com/openapi/introduction

---

## 1. Swagger là gì?

Swagger (OpenAPI Specification) là chuẩn mô tả REST API. Trong NestJS, `@nestjs/swagger` tự động sinh ra tài liệu API tương tác từ decorators — không cần viết tay file YAML/JSON.

**Lợi ích:**
- Tài liệu API tự động, luôn đồng bộ với code
- Giao diện UI để test API trực tiếp trên browser
- Sinh client SDK tự động (TypeScript, Dart, Python...)
- Frontend dev không cần hỏi backend từng endpoint

---

## 2. Cài đặt

```bash
npm install @nestjs/swagger swagger-ui-express
```

---

## 3. Setup cơ bản trong main.ts

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Swagger chỉ bật ở non-production (tuỳ chọn)
  if (process.env.NODE_ENV !== 'production') {
    const config = new DocumentBuilder()
      .setTitle('My API')
      .setDescription('API documentation for My App')
      .setVersion('1.0')
      .addBearerAuth(          // thêm nút Authorize cho JWT
        {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'JWT',
          in: 'header',
        },
        'access-token',        // tên security scheme (dùng lại ở decorator)
      )
      .addTag('Auth', 'Authentication endpoints')
      .addTag('Users', 'User management')
      .addTag('Orders', 'Order management')
      .build();

    const document = SwaggerModule.createDocument(app, config);
    SwaggerModule.setup('api/docs', app, document, {
      swaggerOptions: {
        persistAuthorization: true,   // giữ token sau khi reload
        tagsSorter: 'alpha',
        operationsSorter: 'alpha',
      },
    });
  }

  app.useGlobalPipes(new ValidationPipe({ transform: true }));
  await app.listen(3000);
}
bootstrap();
```

Truy cập: `http://localhost:3000/api/docs`

---

## 4. Decorators trên Controller

```typescript
import {
  ApiTags,
  ApiOperation,
  ApiResponse,
  ApiBearerAuth,
  ApiParam,
  ApiQuery,
  ApiBody,
  ApiConsumes,
  ApiProduces,
  ApiHeader,
  ApiExcludeEndpoint,
  ApiExcludeController,
} from '@nestjs/swagger';

@ApiTags('Users')                     // nhóm endpoint trong Swagger UI
@ApiBearerAuth('access-token')        // yêu cầu JWT (tên phải khớp DocumentBuilder)
@Controller('users')
export class UserController {

  @Get()
  @ApiOperation({
    summary: 'Lấy danh sách users',
    description: 'Trả về danh sách users có phân trang và filter',
  })
  @ApiQuery({ name: 'page', required: false, type: Number, example: 1 })
  @ApiQuery({ name: 'limit', required: false, type: Number, example: 10 })
  @ApiQuery({ name: 'search', required: false, type: String })
  @ApiResponse({ status: 200, description: 'Thành công', type: PaginatedUserDto })
  @ApiResponse({ status: 401, description: 'Chưa đăng nhập' })
  findAll(@Query() query: GetUsersDto) {}

  @Get(':id')
  @ApiOperation({ summary: 'Lấy thông tin 1 user' })
  @ApiParam({ name: 'id', type: String, description: 'User ID', example: 'usr_abc123' })
  @ApiResponse({ status: 200, type: UserResponseDto })
  @ApiResponse({ status: 404, description: 'User không tồn tại' })
  findOne(@Param('id') id: string) {}

  @Post()
  @ApiOperation({ summary: 'Tạo user mới' })
  @ApiBody({ type: CreateUserDto })
  @ApiResponse({ status: 201, type: UserResponseDto })
  @ApiResponse({ status: 400, description: 'Dữ liệu không hợp lệ' })
  create(@Body() dto: CreateUserDto) {}

  @Post('avatar')
  @ApiConsumes('multipart/form-data')  // upload file
  @ApiBody({
    schema: {
      type: 'object',
      properties: {
        file: { type: 'string', format: 'binary' },
      },
    },
  })
  uploadAvatar() {}

  @Delete(':id')
  @ApiExcludeEndpoint()               // ẩn endpoint khỏi Swagger
  remove(@Param('id') id: string) {}
}
```

---

## 5. DTO – Trái tim của Swagger NestJS

Swagger đọc decorators trên DTO để sinh schema. Có 2 cách: `@ApiProperty` (viết tường minh) và Plugin (tự động).

### 5.1. @ApiProperty thủ công

```typescript
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import { IsEmail, IsString, IsOptional, MinLength, IsEnum, IsInt, Min, Max } from 'class-validator';

export enum UserRole {
  ADMIN = 'admin',
  USER = 'user',
  MODERATOR = 'moderator',
}

export class CreateUserDto {
  @ApiProperty({
    description: 'Email của user',
    example: 'john@example.com',
    format: 'email',
  })
  @IsEmail()
  email: string;

  @ApiProperty({
    description: 'Mật khẩu (tối thiểu 8 ký tự)',
    example: 'Secret@123',
    minLength: 8,
  })
  @IsString()
  @MinLength(8)
  password: string;

  @ApiPropertyOptional({          // = @ApiProperty({ required: false })
    description: 'Tên hiển thị',
    example: 'John Doe',
  })
  @IsOptional()
  @IsString()
  displayName?: string;

  @ApiProperty({
    enum: UserRole,
    enumName: 'UserRole',         // tạo enum riêng trong schema (tái sử dụng)
    default: UserRole.USER,
  })
  @IsEnum(UserRole)
  role: UserRole;

  @ApiPropertyOptional({ type: Number, minimum: 1, maximum: 120, example: 25 })
  @IsOptional()
  @IsInt()
  @Min(1)
  @Max(120)
  age?: number;
}
```

### 5.2. Swagger CLI Plugin (khuyến nghị – tự động sinh)

Plugin tự động đọc TypeScript type và `class-validator` để sinh `@ApiProperty` — không cần viết tay.

```json
// nest-cli.json
{
  "compilerOptions": {
    "plugins": [
      {
        "name": "@nestjs/swagger",
        "options": {
          "introspectComments": true,    // đọc JSDoc comment làm description
          "classValidatorShim": true,    // đọc class-validator để sinh validation info
          "dtoFileNameSuffix": [".dto.ts", ".entity.ts"]
        }
      }
    ]
  }
}
```

```typescript
// Với plugin — không cần @ApiProperty, tự động nhận diện từ type + JSDoc
export class CreateUserDto {
  /** Email của user */
  @IsEmail()
  email: string;          // → type: string, format: email

  /** Tên hiển thị */
  @IsOptional()
  @IsString()
  displayName?: string;   // → required: false tự động

  @IsEnum(UserRole)
  role: UserRole;         // → enum tự động

  @IsInt()
  @Min(1)
  @Max(120)
  age: number;            // → minimum: 1, maximum: 120 tự động
}
```

### 5.3. Response DTO (ẩn field nhạy cảm)

```typescript
import { Exclude, Expose } from 'class-transformer';
import { ApiProperty, OmitType, PickType, PartialType, IntersectionType } from '@nestjs/swagger';

export class UserResponseDto {
  @ApiProperty({ example: 'usr_abc123' })
  id: string;

  @ApiProperty({ example: 'john@example.com' })
  email: string;

  @ApiProperty({ example: 'John Doe' })
  displayName: string;

  @ApiProperty({ enum: UserRole })
  role: UserRole;

  @ApiProperty()
  createdAt: Date;

  // Password không expose ra ngoài
  @Exclude()
  password: string;
}

// Utility Types — tái sử dụng DTO
export class UpdateUserDto extends PartialType(CreateUserDto) {}
// → tất cả field của CreateUserDto đều optional

export class LoginDto extends PickType(CreateUserDto, ['email', 'password'] as const) {}
// → chỉ lấy email và password

export class PublicUserDto extends OmitType(UserResponseDto, ['role'] as const) {}
// → bỏ field role

export class AdminUserDto extends IntersectionType(UserResponseDto, AdminMetaDto) {}
// → kết hợp 2 DTO
```

---

## 6. Phân trang – Pagination DTO

```typescript
// common/dto/pagination.dto.ts
import { ApiPropertyOptional } from '@nestjs/swagger';
import { Type } from 'class-transformer';
import { IsOptional, IsInt, Min, Max } from 'class-validator';

export class PaginationDto {
  @ApiPropertyOptional({ default: 1, minimum: 1 })
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page?: number = 1;

  @ApiPropertyOptional({ default: 10, minimum: 1, maximum: 100 })
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  limit?: number = 10;
}

// common/dto/paginated-response.dto.ts
export class PaginatedResponseDto<T> {
  @ApiProperty({ isArray: true })
  data: T[];

  @ApiProperty({ example: 100 })
  total: number;

  @ApiProperty({ example: 1 })
  page: number;

  @ApiProperty({ example: 10 })
  limit: number;

  @ApiProperty({ example: 10 })
  totalPages: number;
}

// Dùng trong Controller
@ApiResponse({
  status: 200,
  schema: {
    allOf: [
      { $ref: getSchemaPath(PaginatedResponseDto) },
      {
        properties: {
          data: { type: 'array', items: { $ref: getSchemaPath(UserResponseDto) } },
        },
      },
    ],
  },
})
findAll() {}
```

---

## 7. Response chuẩn (Wrapper)

```typescript
// common/dto/api-response.dto.ts
import { ApiProperty } from '@nestjs/swagger';

export class ApiResponseDto<T> {
  @ApiProperty({ example: true })
  success: boolean;

  @ApiProperty({ example: 'Thao tác thành công' })
  message: string;

  data?: T;

  @ApiProperty({ example: null, nullable: true })
  error?: string;
}

// Dùng với Generic trong Swagger
import { applyDecorators, Type } from '@nestjs/common';
import { ApiExtraModels, ApiOkResponse, getSchemaPath } from '@nestjs/swagger';

export const ApiSuccessResponse = <TModel extends Type<any>>(model: TModel) => {
  return applyDecorators(
    ApiExtraModels(ApiResponseDto, model),
    ApiOkResponse({
      schema: {
        allOf: [
          { $ref: getSchemaPath(ApiResponseDto) },
          {
            properties: {
              data: { $ref: getSchemaPath(model) },
            },
          },
        ],
      },
    }),
  );
};

// Dùng trong Controller
@Get(':id')
@ApiSuccessResponse(UserResponseDto)
findOne(@Param('id') id: string) {}
```

---

## 8. Custom Decorator tái sử dụng

```typescript
// decorators/api-paginated.decorator.ts
import { applyDecorators, Type } from '@nestjs/common';
import { ApiExtraModels, ApiOkResponse, getSchemaPath } from '@nestjs/swagger';

export const ApiPaginatedResponse = <TModel extends Type<any>>(model: TModel) =>
  applyDecorators(
    ApiExtraModels(model),
    ApiOkResponse({
      schema: {
        properties: {
          data: { type: 'array', items: { $ref: getSchemaPath(model) } },
          total: { type: 'number', example: 100 },
          page: { type: 'number', example: 1 },
          limit: { type: 'number', example: 10 },
        },
      },
    }),
  );

// Dùng:
@Get()
@ApiPaginatedResponse(UserResponseDto)
findAll() {}
```

---

## 9. Security – Nhiều loại Auth

```typescript
// main.ts – Thêm nhiều auth schemes
const config = new DocumentBuilder()
  .addBearerAuth({ type: 'http', scheme: 'bearer', bearerFormat: 'JWT' }, 'JWT')
  .addApiKey({ type: 'apiKey', in: 'header', name: 'X-API-Key' }, 'ApiKey')
  .addBasicAuth()
  .addOAuth2()
  .build();

// Controller – dùng đúng scheme
@ApiTags('Public')
@ApiSecurity('ApiKey')           // dùng API Key scheme
@Controller('public')
export class PublicController {}

@ApiTags('Admin')
@ApiBearerAuth('JWT')            // dùng JWT scheme
@Controller('admin')
export class AdminController {}
```

---

## 10. Upload File

```typescript
@Post('upload')
@UseInterceptors(FileInterceptor('file'))
@ApiConsumes('multipart/form-data')
@ApiOperation({ summary: 'Upload ảnh' })
@ApiBody({
  schema: {
    type: 'object',
    required: ['file'],
    properties: {
      file: {
        type: 'string',
        format: 'binary',
        description: 'File ảnh (jpg, png, webp)',
      },
      caption: {
        type: 'string',
        description: 'Chú thích ảnh',
        example: 'Profile picture',
      },
    },
  },
})
@ApiResponse({ status: 201, description: 'Upload thành công', schema: {
  properties: { url: { type: 'string', example: 'https://cdn.example.com/img.jpg' } }
}})
uploadFile(@UploadedFile() file: Express.Multer.File) {}
```

---

## 11. Versioning API

```typescript
// main.ts
app.enableVersioning({ type: VersioningType.URI });

const v1Doc = SwaggerModule.createDocument(app, config, {
  include: [V1Module],    // chỉ include module của v1
});
SwaggerModule.setup('api/v1/docs', app, v1Doc);

const v2Doc = SwaggerModule.createDocument(app, config, {
  include: [V2Module],
});
SwaggerModule.setup('api/v2/docs', app, v2Doc);
```

---

## 12. Export OpenAPI JSON/YAML

```typescript
// Lấy spec để export hoặc dùng với code generator
const document = SwaggerModule.createDocument(app, config);

// Ghi ra file (dùng khi build)
import * as fs from 'fs';
fs.writeFileSync('./openapi.json', JSON.stringify(document, null, 2));

// Expose endpoint để download spec
app.use('/api/docs-json', (req, res) => res.json(document));
app.use('/api/docs-yaml', (req, res) => {
  const yaml = require('js-yaml');
  res.setHeader('Content-Type', 'text/yaml');
  res.send(yaml.dump(document));
});
```

**Dùng spec để sinh client SDK:**
```bash
# Sinh TypeScript client cho Frontend
npx @openapitools/openapi-generator-cli generate \
  -i http://localhost:3000/api/docs-json \
  -g typescript-axios \
  -o ./frontend/src/api
```

---

## 13. Một số tips thực tế

### 13.1. @ApiHideProperty – Ẩn field trong schema

```typescript
export class UserEntity {
  @ApiProperty()
  id: string;

  @ApiHideProperty()   // ẩn khỏi Swagger docs
  password: string;

  @ApiHideProperty()
  refreshToken: string;
}
```

### 13.2. Enum với description

```typescript
@ApiProperty({
  enum: OrderStatus,
  enumName: 'OrderStatus',
  description: `
    - PENDING: Chờ thanh toán
    - PAID: Đã thanh toán
    - SHIPPING: Đang giao
    - COMPLETED: Hoàn thành
    - CANCELLED: Đã hủy
  `,
})
status: OrderStatus;
```

### 13.3. Nested DTO (object lồng nhau)

```typescript
export class AddressDto {
  @ApiProperty({ example: '123 Nguyễn Huệ' })
  street: string;

  @ApiProperty({ example: 'Hồ Chí Minh' })
  city: string;
}

export class CreateOrderDto {
  @ApiProperty({ type: AddressDto })        // object đơn
  shippingAddress: AddressDto;

  @ApiProperty({ type: [AddressDto] })      // mảng object
  alternativeAddresses: AddressDto[];
}
```

### 13.4. Discriminator (Union type)

```typescript
export class CatDto { type: 'cat'; indoor: boolean; }
export class DogDto { type: 'dog'; breed: string; }

@ApiBody({
  schema: {
    oneOf: [
      { $ref: getSchemaPath(CatDto) },
      { $ref: getSchemaPath(DogDto) },
    ],
    discriminator: { propertyName: 'type' },
  },
})
create(@Body() dto: CatDto | DogDto) {}
```

### 13.5. Bảo mật Swagger ở production

```typescript
// Chặn truy cập nếu không có Basic Auth
import * as basicAuth from 'express-basic-auth';

app.use(
  ['/api/docs', '/api/docs-json'],
  basicAuth({
    challenge: true,
    users: { admin: process.env.SWAGGER_PASSWORD },
  }),
);

SwaggerModule.setup('api/docs', app, document);
```

---

## 14. Cấu trúc file đề xuất

```
src/
├── common/
│   ├── decorators/
│   │   ├── api-paginated.decorator.ts
│   │   └── api-success-response.decorator.ts
│   └── dto/
│       ├── pagination.dto.ts
│       ├── paginated-response.dto.ts
│       └── api-response.dto.ts
├── users/
│   └── dto/
│       ├── create-user.dto.ts
│       ├── update-user.dto.ts
│       └── user-response.dto.ts
└── main.ts
```

---

## 15. Checklist Swagger NestJS

- [ ] Cài `@nestjs/swagger` và `swagger-ui-express`
- [ ] Setup `DocumentBuilder` trong `main.ts` với title, version, tags
- [ ] Thêm `addBearerAuth` nếu dùng JWT
- [ ] Bật Swagger CLI Plugin trong `nest-cli.json`
- [ ] `@ApiTags` cho mỗi Controller
- [ ] `@ApiOperation` với summary/description cho mỗi endpoint
- [ ] `@ApiResponse` với các status code phổ biến (200, 201, 400, 401, 404)
- [ ] `@ApiProperty` hoặc Plugin cho tất cả DTO
- [ ] Dùng `example` trong `@ApiProperty` để UI test dễ hơn
- [ ] Dùng `PartialType`, `OmitType`, `PickType` để tái sử dụng DTO
- [ ] Ẩn field nhạy cảm với `@ApiHideProperty` hoặc `@Exclude`
- [ ] Bảo vệ Swagger UI ở production (Basic Auth hoặc chỉ bật dev)
- [ ] Export `openapi.json` để tích hợp CI/CD hoặc sinh client SDK
