# NestJS Core – Tổng hợp kiến thức đầy đủ

> Docs chính thức: https://docs.nestjs.com

---

## 1. Kiến trúc tổng quan

```
HTTP Request
     │
     ▼
┌─────────────────────────────────────────────────┐
│              Middleware (Express)                │
└─────────────────────────────┬───────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────┐
│                    Guard                        │
│          (Authentication / Authorization)       │
└─────────────────────────────┬───────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────┐
│              Interceptor (before)               │
└─────────────────────────────┬───────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────┐
│                    Pipe                         │
│          (Validation / Transformation)          │
└─────────────────────────────┬───────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────┐
│                  Controller                     │
│              (Route Handler)                    │
└─────────────────────────────┬───────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────┐
│                   Service                       │
│              (Business Logic)                   │
└─────────────────────────────┬───────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │   Exception       │◄── Throws
                    │   Filter          │
                    └─────────┬─────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────┐
│              Interceptor (after)                │
│        (Transform response / Logging)           │
└─────────────────────────────┬───────────────────┘
                              │
                              ▼
                        HTTP Response
```

**Thứ tự thực thi:**
`Middleware → Guard → Interceptor(before) → Pipe → Controller → Service → Exception Filter → Interceptor(after)`

---

## 2. Module

Module là đơn vị tổ chức code trong NestJS. Mỗi app có ít nhất 1 module gốc (`AppModule`).

```typescript
// feature/feature.module.ts
import { Module, Global, DynamicModule } from '@nestjs/common';

@Module({
  imports: [
    TypeOrmModule.forFeature([User]),   // import module khác
    ConfigModule,
  ],
  controllers: [UserController],        // đăng ký controller
  providers: [
    UserService,                        // đăng ký service
    UserRepository,
    {
      provide: 'CUSTOM_TOKEN',          // custom provider
      useValue: { apiKey: '123' },
    },
    {
      provide: UserService,
      useClass: MockUserService,        // useClass
    },
    {
      provide: 'DB_CONFIG',
      useFactory: (config: ConfigService) => ({
        host: config.get('DB_HOST'),
        port: config.get('DB_PORT'),
      }),
      inject: [ConfigService],          // useFactory
    },
  ],
  exports: [UserService],              // expose ra cho module khác import
})
export class UserModule {}
```

```typescript
// Global Module – inject được ở mọi nơi không cần import lại
@Global()
@Module({
  providers: [PrismaService, RedisService],
  exports: [PrismaService, RedisService],
})
export class CoreModule {}
```

```typescript
// Dynamic Module – cấu hình khi import
@Module({})
export class MailModule {
  static forRoot(options: MailOptions): DynamicModule {
    return {
      module: MailModule,
      providers: [
        { provide: 'MAIL_OPTIONS', useValue: options },
        MailService,
      ],
      exports: [MailService],
    };
  }
}

// Dùng
@Module({
  imports: [
    MailModule.forRoot({ host: 'smtp.gmail.com', port: 587 }),
  ],
})
export class AppModule {}
```

---

## 3. Controller

Controller xử lý HTTP request và trả về response.

```typescript
import {
  Controller, Get, Post, Put, Patch, Delete,
  Param, Query, Body, Headers, Req, Res,
  HttpCode, HttpStatus, Redirect,
  UseGuards, UseInterceptors, UsePipes, UseFilters,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Controller('users')                        // prefix route
@ApiTags('Users')
@UseGuards(JwtAuthGuard)                    // áp dụng cho toàn controller
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get()
  @HttpCode(HttpStatus.OK)
  findAll(@Query() query: GetUsersDto) {
    return this.userService.findAll(query);
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.userService.findById(id);
  }

  @Post()
  @HttpCode(HttpStatus.CREATED)
  create(@Body() dto: CreateUserDto) {
    return this.userService.create(dto);
  }

  @Put(':id')
  update(@Param('id') id: string, @Body() dto: UpdateUserDto) {
    return this.userService.update(id, dto);
  }

  @Patch(':id/status')
  updateStatus(
    @Param('id') id: string,
    @Body('isActive') isActive: boolean,   // lấy field cụ thể trong body
  ) {
    return this.userService.updateStatus(id, isActive);
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  remove(@Param('id') id: string) {
    return this.userService.remove(id);
  }

  @Get('redirect')
  @Redirect('https://docs.nestjs.com', 302)
  redirect() {}

  // Truy cập Express request/response trực tiếp
  @Get('raw')
  raw(@Req() req: Request, @Res() res: Response) {
    res.status(200).json({ ip: req.ip });
  }

  // Lấy header
  @Post('webhook')
  webhook(@Headers('x-signature') sig: string, @Body() body: any) {
    return this.userService.handleWebhook(sig, body);
  }
}
```

### Route params nâng cao

```typescript
@Get(':version/users/:id')
findVersioned(
  @Param() params: { version: string; id: string },  // lấy tất cả params
) {}

// Wildcard
@Get('ab*cd')
wildcard() {}

// Sub-domain routing
@Controller({ host: ':account.example.com' })
export class AccountController {
  @Get()
  getInfo(@HostParam('account') account: string) {}
}
```

---

## 4. Service & Dependency Injection

```typescript
// users/users.service.ts
import { Injectable, NotFoundException, Inject } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class UserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly config: ConfigService,
    @Inject('MAIL_OPTIONS') private mailOptions: MailOptions,  // inject custom token
    @Inject(forwardRef(() => OrderService)) private orderService: OrderService, // circular dep
  ) {}

  async findById(id: string) {
    const user = await this.userRepository.findOne(id);
    if (!user) throw new NotFoundException(`User #${id} không tồn tại`);
    return user;
  }
}
```

### Scope của Provider

```typescript
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.DEFAULT })   // Singleton (mặc định) – 1 instance toàn app
export class SingletonService {}

@Injectable({ scope: Scope.REQUEST })   // Mỗi request tạo 1 instance mới
export class RequestScopedService {}

@Injectable({ scope: Scope.TRANSIENT }) // Mỗi lần inject tạo 1 instance mới
export class TransientService {}
```

---

## 5. DTO & Validation

### 5.1. DTO với class-validator

```typescript
// dto/create-user.dto.ts
import {
  IsEmail, IsString, IsOptional, IsEnum, IsInt,
  MinLength, MaxLength, Min, Max, IsBoolean,
  IsUrl, IsPhoneNumber, IsDateString, IsArray,
  ValidateNested, IsNotEmpty, Matches, IsUUID,
} from 'class-validator';
import { Type } from 'class-transformer';
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

export class AddressDto {
  @ApiProperty({ example: '123 Nguyễn Huệ' })
  @IsString()
  @IsNotEmpty()
  street: string;

  @ApiProperty({ example: 'Hồ Chí Minh' })
  @IsString()
  city: string;
}

export class CreateUserDto {
  @ApiProperty({ example: 'alice@example.com' })
  @IsEmail({}, { message: 'Email không hợp lệ' })
  email: string;

  @ApiProperty({ minLength: 8 })
  @IsString()
  @MinLength(8, { message: 'Mật khẩu tối thiểu 8 ký tự' })
  @MaxLength(64)
  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/, {
    message: 'Mật khẩu phải có chữ hoa, thường và số',
  })
  password: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  @MaxLength(100)
  displayName?: string;

  @ApiProperty({ enum: UserRole, default: UserRole.USER })
  @IsEnum(UserRole)
  role: UserRole;

  @ApiPropertyOptional({ type: Number, minimum: 1, maximum: 120 })
  @IsOptional()
  @IsInt()
  @Min(1)
  @Max(120)
  age?: number;

  @ApiPropertyOptional({ type: AddressDto })
  @IsOptional()
  @ValidateNested()            // validate nested object
  @Type(() => AddressDto)      // class-transformer cần biết type
  address?: AddressDto;

  @ApiPropertyOptional({ type: [String] })
  @IsOptional()
  @IsArray()
  @IsString({ each: true })   // validate từng phần tử trong mảng
  tags?: string[];
}
```

### 5.2. Utility Types

```typescript
import { PartialType, OmitType, PickType, IntersectionType } from '@nestjs/swagger';

// Tất cả field optional
export class UpdateUserDto extends PartialType(CreateUserDto) {}

// Chỉ lấy 1 số field
export class LoginDto extends PickType(CreateUserDto, ['email', 'password'] as const) {}

// Bỏ field
export class RegisterDto extends OmitType(CreateUserDto, ['role'] as const) {}

// Kết hợp
export class AdminCreateUserDto extends IntersectionType(CreateUserDto, AdminMetaDto) {}
```

### 5.3. Transform với class-transformer

```typescript
import { Transform, Type, Exclude, Expose } from 'class-transformer';

export class GetUsersDto {
  @Transform(({ value }) => parseInt(value))  // query string luôn là string → parse thành number
  @IsInt()
  @Min(1)
  page: number = 1;

  @Transform(({ value }) => parseInt(value))
  @IsInt()
  @Min(1)
  @Max(100)
  limit: number = 10;

  @Transform(({ value }) => value?.trim().toLowerCase())
  @IsOptional()
  @IsString()
  search?: string;

  // Sort order
  @Transform(({ value }) => value === 'asc' ? 'asc' : 'desc')
  @IsOptional()
  sortOrder?: 'asc' | 'desc' = 'desc';
}

// Response DTO – ẩn field nhạy cảm
export class UserResponseDto {
  @Expose()
  id: string;

  @Expose()
  email: string;

  @Expose()
  displayName: string;

  @Exclude()          // không bao giờ trả về
  password: string;

  @Exclude()
  refreshToken: string;
}
```

---

## 6. Pipe

Pipe xử lý **validation** và **transformation** trước khi data vào controller.

### 6.1. Built-in Pipes

```typescript
import {
  ValidationPipe, ParseIntPipe, ParseUUIDPipe,
  ParseBoolPipe, ParseArrayPipe, ParseEnumPipe,
  DefaultValuePipe,
} from '@nestjs/common';

// Trong Controller
@Get(':id')
findOne(
  @Param('id', ParseUUIDPipe) id: string,              // validate UUID
) {}

@Get()
findAll(
  @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
  @Query('active', ParseBoolPipe) active: boolean,
) {}

@Get()
findByIds(
  @Query('ids', new ParseArrayPipe({ items: String, separator: ',' }))
  ids: string[],
) {}
```

### 6.2. ValidationPipe – Quan trọng nhất

```typescript
// main.ts – Global pipe
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,          // tự động strip field không có trong DTO
    forbidNonWhitelisted: true, // throw lỗi nếu có field lạ
    transform: true,          // tự động transform type (string → number)
    transformOptions: {
      enableImplicitConversion: true,  // tự chuyển type dựa vào TS metadata
    },
    disableErrorMessages: process.env.NODE_ENV === 'production',
    exceptionFactory: (errors) => {
      const messages = errors.map(e => ({
        field: e.property,
        errors: Object.values(e.constraints ?? {}),
      }));
      return new BadRequestException({ message: 'Validation failed', errors: messages });
    },
  }),
);
```

### 6.3. Custom Pipe

```typescript
// pipes/parse-object-id.pipe.ts
import { PipeTransform, Injectable, BadRequestException } from '@nestjs/common';
import { Types } from 'mongoose';

@Injectable()
export class ParseObjectIdPipe implements PipeTransform {
  transform(value: string) {
    if (!Types.ObjectId.isValid(value)) {
      throw new BadRequestException(`"${value}" không phải ObjectId hợp lệ`);
    }
    return value;
  }
}

// pipes/trim.pipe.ts
@Injectable()
export class TrimPipe implements PipeTransform {
  transform(value: any) {
    if (typeof value === 'string') return value.trim();
    if (typeof value === 'object' && value !== null) {
      Object.keys(value).forEach(key => {
        if (typeof value[key] === 'string') value[key] = value[key].trim();
      });
    }
    return value;
  }
}

// Dùng
@Get(':id')
findOne(@Param('id', ParseObjectIdPipe) id: string) {}
```

---

## 7. Guard

Guard quyết định request có được phép vào controller không.

### 7.1. JWT Auth Guard

```typescript
// guards/jwt-auth.guard.ts
import { Injectable, ExecutionContext, UnauthorizedException } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { Reflector } from '@nestjs/core';
import { IS_PUBLIC_KEY } from '../decorators/public.decorator';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;
    return super.canActivate(context);
  }

  handleRequest(err: any, user: any) {
    if (err || !user) throw err || new UnauthorizedException('Token không hợp lệ');
    return user;
  }
}
```

### 7.2. Roles Guard

```typescript
// guards/roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.getAllAndOverride<string[]>('roles', [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!roles?.length) return true;

    const { user } = context.switchToHttp().getRequest();
    if (!roles.includes(user?.role)) {
      throw new ForbiddenException('Bạn không có quyền truy cập');
    }
    return true;
  }
}
```

### 7.3. Throttle Guard (Rate Limit)

```typescript
// guards/throttle.guard.ts
import { ThrottlerGuard, ThrottlerException } from '@nestjs/throttler';

@Injectable()
export class CustomThrottlerGuard extends ThrottlerGuard {
  protected throwThrottlingException(): void {
    throw new ThrottlerException('Bạn đang gửi quá nhiều request. Vui lòng thử lại sau.');
  }

  // Lấy key theo IP + userId thay vì chỉ IP
  protected async getTracker(req: Record<string, any>): Promise<string> {
    const userId = req.user?.id ?? 'anonymous';
    return `${req.ip}:${userId}`;
  }
}

// main.ts hoặc module
app.useGlobalGuards(new ThrottlerGuard());
```

---

## 8. Interceptor

Interceptor chạy **trước và sau** khi handler thực thi — dùng để transform response, logging, cache, timeout.

### 8.1. Response Transform Interceptor

```typescript
// interceptors/transform.interceptor.ts
import {
  Injectable, NestInterceptor, ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface ApiResponse<T> {
  success: boolean;
  message: string;
  data: T;
  timestamp: string;
}

@Injectable()
export class TransformInterceptor<T>
  implements NestInterceptor<T, ApiResponse<T>>
{
  intercept(context: ExecutionContext, next: CallHandler): Observable<ApiResponse<T>> {
    return next.handle().pipe(
      map(data => ({
        success: true,
        message: 'Thành công',
        data,
        timestamp: new Date().toISOString(),
      })),
    );
  }
}
```

### 8.2. Logging Interceptor

```typescript
// interceptors/logging.interceptor.ts
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private logger = new Logger('HTTP');

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const req = context.switchToHttp().getRequest();
    const { method, url, body, user } = req;
    const start = Date.now();

    return next.handle().pipe(
      tap({
        next: () => {
          const duration = Date.now() - start;
          this.logger.log(`${method} ${url} ${duration}ms [user: ${user?.id ?? 'anon'}]`);
        },
        error: (err) => {
          const duration = Date.now() - start;
          this.logger.error(`${method} ${url} ${duration}ms → ${err.message}`);
        },
      }),
    );
  }
}
```

### 8.3. Timeout Interceptor

```typescript
// interceptors/timeout.interceptor.ts
import { timeout, catchError } from 'rxjs/operators';
import { TimeoutError, throwError } from 'rxjs';
import { RequestTimeoutException } from '@nestjs/common';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  constructor(private readonly timeoutMs = 30_000) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(this.timeoutMs),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException('Request timeout'));
        }
        return throwError(() => err);
      }),
    );
  }
}
```

### 8.4. Cache Interceptor

```typescript
// interceptors/cache.interceptor.ts
@Injectable()
export class HttpCacheInterceptor extends CacheInterceptor {
  // Chỉ cache GET request
  protected isRequestCacheable(context: ExecutionContext): boolean {
    const req = context.switchToHttp().getRequest();
    return req.method === 'GET';
  }

  // Custom cache key
  trackBy(context: ExecutionContext): string | undefined {
    const req = context.switchToHttp().getRequest();
    const { url, user } = req;
    return user ? `${url}:${user.id}` : url;
  }
}
```

### 8.5. Serialize Interceptor

```typescript
// interceptors/serialize.interceptor.ts
import { plainToInstance } from 'class-transformer';

export function Serialize(dto: any) {
  return UseInterceptors(new SerializeInterceptor(dto));
}

@Injectable()
export class SerializeInterceptor implements NestInterceptor {
  constructor(private dto: any) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map(data =>
        plainToInstance(this.dto, data, { excludeExtraneousValues: true }),
      ),
    );
  }
}

// Dùng
@Get(':id')
@Serialize(UserResponseDto)
findOne(@Param('id') id: string) {
  return this.userService.findById(id);
}
```

---

## 9. Middleware

Middleware chạy **trước tất cả** Guard, Pipe, Interceptor — tương tự Express middleware.

```typescript
// middleware/logger.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const { method, originalUrl } = req;
    const start = Date.now();

    res.on('finish', () => {
      const duration = Date.now() - start;
      console.log(`${method} ${originalUrl} ${res.statusCode} ${duration}ms`);
    });

    next();
  }
}

// middleware/ip-whitelist.middleware.ts
@Injectable()
export class IpWhitelistMiddleware implements NestMiddleware {
  private readonly whitelist = ['127.0.0.1', '::1'];

  use(req: Request, res: Response, next: NextFunction) {
    const ip = req.ip || req.connection.remoteAddress;
    if (!this.whitelist.includes(ip)) {
      res.status(403).json({ message: 'IP không được phép truy cập' });
      return;
    }
    next();
  }
}

// Đăng ký Middleware trong Module
@Module({...})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('*');                        // tất cả routes

    consumer
      .apply(IpWhitelistMiddleware)
      .forRoutes({ path: 'admin/*', method: RequestMethod.ALL });

    consumer
      .apply(AuthMiddleware)
      .exclude(
        { path: 'auth/login', method: RequestMethod.POST },
        { path: 'health', method: RequestMethod.GET },
      )
      .forRoutes(UserController);
  }
}
```

---

## 10. Exception Filters

Bắt và xử lý tất cả exception trước khi trả về response.

### 10.1. Global Exception Filter

```typescript
// filters/http-exception.filter.ts
import {
  ExceptionFilter, Catch, ArgumentsHost,
  HttpException, HttpStatus, Logger,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch()  // Bắt tất cả exception (không chỉ HttpException)
export class GlobalExceptionFilter implements ExceptionFilter {
  private logger = new Logger('ExceptionFilter');

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const req = ctx.getRequest<Request>();
    const res = ctx.getResponse<Response>();

    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let message = 'Internal server error';
    let errors: any[] = [];

    if (exception instanceof HttpException) {
      status = exception.getStatus();
      const response = exception.getResponse();

      if (typeof response === 'object' && response !== null) {
        message = (response as any).message || message;
        errors = (response as any).errors || [];
      } else {
        message = response as string;
      }
    } else if (exception instanceof Error) {
      message = exception.message;
      this.logger.error(`Unhandled: ${exception.message}`, exception.stack);
    }

    res.status(status).json({
      success: false,
      statusCode: status,
      message,
      errors: errors.length ? errors : undefined,
      path: req.url,
      timestamp: new Date().toISOString(),
    });
  }
}

// main.ts
app.useGlobalFilters(new GlobalExceptionFilter());
```

### 10.2. Built-in HTTP Exceptions

```typescript
import {
  BadRequestException,        // 400
  UnauthorizedException,      // 401
  ForbiddenException,         // 403
  NotFoundException,          // 404
  ConflictException,          // 409
  UnprocessableEntityException, // 422
  InternalServerErrorException, // 500
  ServiceUnavailableException,  // 503
  HttpException,              // base
} from '@nestjs/common';

// Ném exception trong Service
throw new NotFoundException(`User #${id} không tồn tại`);
throw new ConflictException('Email đã được sử dụng');
throw new BadRequestException({
  message: 'Validation failed',
  errors: [{ field: 'email', message: 'Email không hợp lệ' }],
});

// Custom Exception
export class BusinessException extends HttpException {
  constructor(message: string, code: string, status = HttpStatus.BAD_REQUEST) {
    super({ message, code, statusCode: status }, status);
  }
}
throw new BusinessException('Số dư không đủ', 'INSUFFICIENT_BALANCE');
```

---

## 11. Decorator

### 11.1. Custom Param Decorator

```typescript
// decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (field: string | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return field ? request.user?.[field] : request.user;
  },
);

// Dùng
@Get('me')
getMe(@CurrentUser() user: User) {}

@Get('profile')
getProfile(@CurrentUser('id') userId: string) {}
```

```typescript
// decorators/ip.decorator.ts
export const ClientIp = createParamDecorator(
  (_, ctx: ExecutionContext): string => {
    const req = ctx.switchToHttp().getRequest();
    return req.headers['x-forwarded-for']?.split(',')[0] ?? req.ip;
  },
);

// decorators/user-agent.decorator.ts
export const UserAgent = createParamDecorator(
  (_, ctx: ExecutionContext): string =>
    ctx.switchToHttp().getRequest().headers['user-agent'],
);
```

### 11.2. Custom Method Decorator

```typescript
// decorators/public.decorator.ts
import { SetMetadata } from '@nestjs/common';
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);

// decorators/roles.decorator.ts
export const Roles = (...roles: UserRole[]) => SetMetadata('roles', roles);

// decorators/skip-transform.decorator.ts
export const SkipTransform = () => SetMetadata('skipTransform', true);

// Kết hợp nhiều decorator
import { applyDecorators } from '@nestjs/common';

export function Auth(...roles: UserRole[]) {
  return applyDecorators(
    Roles(...roles),
    UseGuards(JwtAuthGuard, RolesGuard),
    ApiBearerAuth('access-token'),
    ApiUnauthorizedResponse({ description: 'Chưa đăng nhập' }),
    ApiForbiddenResponse({ description: 'Không có quyền' }),
  );
}

// Dùng
@Get('admin')
@Auth(UserRole.ADMIN)
adminOnly() {}
```

### 11.3. Custom Class Decorator

```typescript
// Kết hợp nhiều decorator trên Controller
export function ApiController(prefix: string, tag: string) {
  return applyDecorators(
    Controller(prefix),
    ApiTags(tag),
    ApiBearerAuth('access-token'),
    UseGuards(JwtAuthGuard),
    UseInterceptors(TransformInterceptor),
    UseFilters(GlobalExceptionFilter),
  );
}

// Dùng
@ApiController('users', 'Users')
export class UserController {}
```

---

## 12. Lifecycle Hooks

```typescript
import {
  OnModuleInit, OnModuleDestroy,
  OnApplicationBootstrap, OnApplicationShutdown,
  BeforeApplicationShutdown,
} from '@nestjs/common';

@Injectable()
export class DatabaseService
  implements OnModuleInit, OnModuleDestroy, OnApplicationShutdown
{
  // Chạy khi module được init
  async onModuleInit() {
    await this.connect();
    console.log('Database connected');
  }

  // Chạy khi module bị destroy
  async onModuleDestroy() {
    await this.disconnect();
  }

  // Chạy khi nhận tín hiệu shutdown (SIGTERM)
  async onApplicationShutdown(signal?: string) {
    console.log(`Shutting down on signal: ${signal}`);
    await this.disconnect();
  }
}

// main.ts – bắt shutdown signal
app.enableShutdownHooks();
```

---

## 13. Configuration

```typescript
// config/configuration.ts
export default () => ({
  port: parseInt(process.env.PORT, 10) || 3000,
  database: {
    host: process.env.DB_HOST,
    port: parseInt(process.env.DB_PORT, 10) || 5432,
    name: process.env.DB_NAME,
  },
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRES_IN || '15m',
  },
});

// config/validation.schema.ts
import * as Joi from 'joi';
export const validationSchema = Joi.object({
  PORT: Joi.number().default(3000),
  DB_HOST: Joi.string().required(),
  DB_PORT: Joi.number().default(5432),
  JWT_SECRET: Joi.string().min(32).required(),
  NODE_ENV: Joi.string().valid('development', 'production', 'test').default('development'),
});

// app.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      load: [configuration],
      validationSchema,         // validate .env khi startup
      cache: true,
    }),
  ],
})
export class AppModule {}

// Dùng trong Service
@Injectable()
export class AppService {
  constructor(private config: ConfigService) {}

  getPort() {
    return this.config.get<number>('port');
  }

  getDbConfig() {
    return this.config.get('database');
  }
}
```

---

## 14. Health Check

```typescript
// npm install @nestjs/terminus
// health/health.module.ts
import { TerminusModule } from '@nestjs/terminus';
import { PrismaHealthIndicator } from './prisma.health';

@Module({
  imports: [TerminusModule],
  providers: [PrismaHealthIndicator],
  controllers: [HealthController],
})
export class HealthModule {}

// health/health.controller.ts
import { HealthCheck, HealthCheckService, PrismaHealthIndicator } from '@nestjs/terminus';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: PrismaHealthIndicator,
    private redis: RedisHealthIndicator,
  ) {}

  @Get()
  @Public()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.isHealthy('database'),
      () => this.redis.isHealthy('redis'),
    ]);
  }
}
```

---

## 15. main.ts chuẩn

```typescript
// main.ts
import { NestFactory, Reflector } from '@nestjs/core';
import { ValidationPipe, ClassSerializerInterceptor, VersioningType } from '@nestjs/common';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import * as helmet from 'helmet';
import * as compression from 'compression';
import * as cookieParser from 'cookie-parser';
import { AppModule } from './app.module';
import { GlobalExceptionFilter } from './filters/http-exception.filter';
import { TransformInterceptor } from './interceptors/transform.interceptor';
import { LoggingInterceptor } from './interceptors/logging.interceptor';
import { TimeoutInterceptor } from './interceptors/timeout.interceptor';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    logger: ['error', 'warn', 'log', 'debug'],
  });

  // Security
  app.use(helmet());
  app.enableCors({
    origin: process.env.FRONTEND_URL?.split(',') ?? '*',
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
  });

  // Middlewares
  app.use(compression());
  app.use(cookieParser());

  // Global prefix
  app.setGlobalPrefix('api/v1');

  // Versioning
  app.enableVersioning({ type: VersioningType.URI });

  // Global Pipes
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
      transformOptions: { enableImplicitConversion: true },
    }),
  );

  // Global Interceptors
  const reflector = app.get(Reflector);
  app.useGlobalInterceptors(
    new LoggingInterceptor(),
    new TransformInterceptor(),
    new TimeoutInterceptor(30_000),
    new ClassSerializerInterceptor(reflector),  // tự động apply @Exclude/@Expose
  );

  // Global Filters
  app.useGlobalFilters(new GlobalExceptionFilter());

  // Swagger
  if (process.env.NODE_ENV !== 'production') {
    const config = new DocumentBuilder()
      .setTitle('My API')
      .setVersion('1.0')
      .addBearerAuth({ type: 'http', scheme: 'bearer', bearerFormat: 'JWT' }, 'access-token')
      .build();
    const document = SwaggerModule.createDocument(app, config);
    SwaggerModule.setup('api/docs', app, document);
  }

  // Graceful Shutdown
  app.enableShutdownHooks();

  const port = process.env.PORT || 3000;
  await app.listen(port);
  console.log(`Application running on: http://localhost:${port}/api/v1`);
}
bootstrap();
```

---

## 16. Checklist NestJS Core

### Bắt buộc
- [ ] `ValidationPipe` global với `whitelist: true`, `transform: true`
- [ ] `GlobalExceptionFilter` để format lỗi chuẩn
- [ ] `TransformInterceptor` để wrap response chuẩn `{ success, data, message }`
- [ ] `LoggingInterceptor` để log request/response
- [ ] `ClassSerializerInterceptor` để tự động `@Exclude` field nhạy cảm
- [ ] `JwtAuthGuard` global với `@Public()` để opt-out
- [ ] `ConfigModule.forRoot({ isGlobal: true, validationSchema })` validate .env khi startup
- [ ] `app.enableShutdownHooks()` graceful shutdown
- [ ] `helmet()` bảo mật HTTP headers
- [ ] `cors` đúng origin, không dùng `*` ở production

### Nên có
- [ ] `TimeoutInterceptor` tránh request treo vô hạn
- [ ] `ThrottlerModule` rate limiting
- [ ] `HealthController` endpoint `/health` check DB, Redis
- [ ] `compression()` gzip response
- [ ] DTO với `@ApiProperty` + Swagger CLI Plugin
- [ ] Custom Exception class theo nghiệp vụ
- [ ] `@Auth()` decorator kết hợp Guard + Swagger + Roles
- [ ] `ParseObjectIdPipe` hoặc `ParseUUIDPipe` validate params
