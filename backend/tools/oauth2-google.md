# OAuth2 – Đăng nhập với Google (đặc biệt cho NestJS)

---

## 1. OAuth2 là gì?

OAuth2 là **giao thức ủy quyền** (authorization protocol) cho phép ứng dụng thứ ba truy cập tài nguyên của người dùng mà **không cần biết mật khẩu** của họ. Khi dùng "Đăng nhập với Google", bạn đang dùng OAuth2.

**Phân biệt OAuth2 vs OpenID Connect (OIDC):**
- **OAuth2**: Ủy quyền — "App này được phép làm gì?"
- **OIDC**: Xác thực — "Người dùng này là ai?" (xây dựng trên OAuth2, thêm `id_token`)
- Google Login dùng **OIDC** (là OAuth2 + thêm thông tin user)

---

## 2. Luồng hoạt động (Authorization Code Flow)

```
User                  Frontend              Backend              Google
 │                       │                     │                    │
 │  Click "Login Google" │                     │                    │
 │──────────────────────►│                     │                    │
 │                       │  GET /auth/google   │                    │
 │                       │────────────────────►│                    │
 │                       │   redirect to Google│                    │
 │◄──────────────────────│◄────────────────────│                    │
 │                                                                   │
 │  Đăng nhập Google + chấp thuận quyền                             │
 │──────────────────────────────────────────────────────────────────►│
 │                                                                   │
 │  Redirect về callback URL kèm ?code=AUTH_CODE                    │
 │◄──────────────────────────────────────────────────────────────────│
 │                       │                     │                    │
 │  GET /auth/google/callback?code=...         │                    │
 │──────────────────────►│────────────────────►│                    │
 │                       │                     │ Exchange code      │
 │                       │                     │ for access_token   │
 │                       │                     │───────────────────►│
 │                       │                     │◄───────────────────│
 │                       │                     │  Lấy profile user  │
 │                       │                     │───────────────────►│
 │                       │                     │◄───────────────────│
 │                       │                     │ Tạo JWT, trả về    │
 │◄──────────────────────│◄────────────────────│                    │
 │  { accessToken, refreshToken }              │                    │
```

**Tóm gọn 5 bước:**
1. User click → redirect đến Google kèm `client_id`, `scope`, `redirect_uri`
2. User đăng nhập Google + chấp thuận quyền
3. Google redirect về `callback URL` kèm `authorization code`
4. Backend dùng `code` đổi lấy `access_token` + `id_token` từ Google
5. Backend lấy thông tin user từ Google, tạo JWT của app, trả về client

---

## 3. Chuẩn bị Google Cloud Console

1. Vào https://console.cloud.google.com
2. Tạo project mới (hoặc chọn project có sẵn)
3. Vào **APIs & Services** → **Credentials**
4. Tạo **OAuth 2.0 Client ID** → Application type: **Web application**
5. Thêm **Authorized redirect URIs**:
   - Dev: `http://localhost:3000/auth/google/callback`
   - Prod: `https://yourdomain.com/auth/google/callback`
6. Copy **Client ID** và **Client Secret**
7. Vào **OAuth consent screen** → thêm scopes: `email`, `profile`, `openid`

```env
# .env
GOOGLE_CLIENT_ID=123456789-abc.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=GOCSPX-xxxxxxxxxxxxxxxx
GOOGLE_CALLBACK_URL=http://localhost:3000/auth/google/callback

JWT_ACCESS_SECRET=your_access_secret_here
JWT_REFRESH_SECRET=your_refresh_secret_here
JWT_ACCESS_EXPIRES=15m
JWT_REFRESH_EXPIRES=7d

FRONTEND_URL=http://localhost:3001
```

---

## 4. Cài đặt NestJS

```bash
npm install @nestjs/passport passport passport-google-oauth20
npm install @nestjs/jwt passport-jwt
npm install -D @types/passport-google-oauth20 @types/passport-jwt
```

---

## 5. Google Strategy

Strategy là nơi xử lý OAuth2 flow với Google.

```typescript
// auth/strategies/google.strategy.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy, VerifyCallback, Profile } from 'passport-google-oauth20';
import { AuthService } from '../auth.service';

@Injectable()
export class GoogleStrategy extends PassportStrategy(Strategy, 'google') {
  constructor(
    private config: ConfigService,
    private authService: AuthService,
  ) {
    super({
      clientID: config.get('GOOGLE_CLIENT_ID'),
      clientSecret: config.get('GOOGLE_CLIENT_SECRET'),
      callbackURL: config.get('GOOGLE_CALLBACK_URL'),
      scope: ['email', 'profile', 'openid'],
    });
  }

  // Chạy sau khi Google xác thực thành công
  async validate(
    accessToken: string,
    refreshToken: string,
    profile: Profile,
    done: VerifyCallback,
  ): Promise<void> {
    const { id, emails, photos, displayName, name } = profile;

    const googleUser = {
      googleId: id,
      email: emails[0].value,
      displayName,
      firstName: name?.givenName,
      lastName: name?.familyName,
      avatar: photos?.[0]?.value,
      accessToken,
    };

    // Tìm hoặc tạo user trong DB
    const user = await this.authService.findOrCreateGoogleUser(googleUser);
    done(null, user);
  }
}
```

---

## 6. JWT Strategy

```typescript
// auth/strategies/jwt.strategy.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { UsersService } from '../../users/users.service';

export interface JwtPayload {
  sub: string;      // user ID
  email: string;
  role: string;
  iat?: number;
  exp?: number;
}

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy, 'jwt') {
  constructor(
    private config: ConfigService,
    private usersService: UsersService,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: config.get('JWT_ACCESS_SECRET'),
      ignoreExpiration: false,
    });
  }

  async validate(payload: JwtPayload) {
    const user = await this.usersService.findById(payload.sub);
    if (!user) throw new UnauthorizedException('User không tồn tại');
    return user; // gán vào request.user
  }
}
```

---

## 7. Auth Guards

```typescript
// auth/guards/google-auth.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class GoogleAuthGuard extends AuthGuard('google') {}

// auth/guards/jwt-auth.guard.ts
import { Injectable, ExecutionContext } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { Reflector } from '@nestjs/core';
import { IS_PUBLIC_KEY } from '../decorators/public.decorator';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    // Bỏ qua guard nếu có @Public()
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;
    return super.canActivate(context);
  }
}

// auth/decorators/public.decorator.ts
import { SetMetadata } from '@nestjs/common';
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);

// auth/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) =>
    ctx.switchToHttp().getRequest().user,
);
```

---

## 8. Auth Service

```typescript
// auth/auth.service.ts
import { Injectable } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from '../users/user.entity';

interface GoogleUser {
  googleId: string;
  email: string;
  displayName: string;
  firstName?: string;
  lastName?: string;
  avatar?: string;
}

@Injectable()
export class AuthService {
  constructor(
    @InjectRepository(User) private userRepo: Repository<User>,
    private jwtService: JwtService,
    private config: ConfigService,
  ) {}

  // Tìm user đã tồn tại hoặc tạo mới từ Google profile
  async findOrCreateGoogleUser(googleUser: GoogleUser): Promise<User> {
    // Tìm theo googleId trước
    let user = await this.userRepo.findOne({
      where: { googleId: googleUser.googleId },
    });

    if (!user) {
      // Tìm theo email (user đã đăng ký bằng email trước đó)
      user = await this.userRepo.findOne({
        where: { email: googleUser.email },
      });

      if (user) {
        // Liên kết tài khoản Google với tài khoản email đã có
        user.googleId = googleUser.googleId;
        user.avatar = user.avatar || googleUser.avatar;
        await this.userRepo.save(user);
      } else {
        // Tạo tài khoản mới
        user = this.userRepo.create({
          googleId: googleUser.googleId,
          email: googleUser.email,
          displayName: googleUser.displayName,
          firstName: googleUser.firstName,
          lastName: googleUser.lastName,
          avatar: googleUser.avatar,
          isEmailVerified: true,   // email đã xác thực qua Google
          authProvider: 'google',
        });
        await this.userRepo.save(user);
      }
    }

    return user;
  }

  // Tạo cặp access + refresh token
  generateTokens(user: User) {
    const payload: JwtPayload = {
      sub: user.id,
      email: user.email,
      role: user.role,
    };

    const accessToken = this.jwtService.sign(payload, {
      secret: this.config.get('JWT_ACCESS_SECRET'),
      expiresIn: this.config.get('JWT_ACCESS_EXPIRES'), // '15m'
    });

    const refreshToken = this.jwtService.sign(
      { sub: user.id },
      {
        secret: this.config.get('JWT_REFRESH_SECRET'),
        expiresIn: this.config.get('JWT_REFRESH_EXPIRES'), // '7d'
      },
    );

    return { accessToken, refreshToken };
  }

  // Refresh access token
  async refreshTokens(refreshToken: string) {
    try {
      const payload = this.jwtService.verify(refreshToken, {
        secret: this.config.get('JWT_REFRESH_SECRET'),
      });

      const user = await this.userRepo.findOne({ where: { id: payload.sub } });
      if (!user) throw new Error('User not found');

      return this.generateTokens(user);
    } catch {
      throw new UnauthorizedException('Refresh token không hợp lệ hoặc đã hết hạn');
    }
  }
}
```

---

## 9. Auth Controller

```typescript
// auth/auth.controller.ts
import {
  Controller, Get, Post, Body, Req, Res,
  UseGuards, HttpCode, UnauthorizedException,
} from '@nestjs/common';
import { Response, Request } from 'express';
import { ApiTags, ApiOperation, ApiBearerAuth } from '@nestjs/swagger';
import { GoogleAuthGuard } from './guards/google-auth.guard';
import { JwtAuthGuard } from './guards/jwt-auth.guard';
import { Public } from './decorators/public.decorator';
import { CurrentUser } from './decorators/current-user.decorator';
import { AuthService } from './auth.service';
import { User } from '../users/user.entity';

@ApiTags('Auth')
@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  // Bước 1: Redirect đến Google
  @Get('google')
  @Public()
  @UseGuards(GoogleAuthGuard)
  @ApiOperation({ summary: 'Đăng nhập với Google' })
  googleLogin() {
    // Passport tự xử lý redirect → không cần code ở đây
  }

  // Bước 2: Google callback
  @Get('google/callback')
  @Public()
  @UseGuards(GoogleAuthGuard)
  @ApiOperation({ summary: 'Google OAuth2 callback' })
  async googleCallback(@Req() req: Request, @Res() res: Response) {
    const user = req.user as User;
    const { accessToken, refreshToken } = this.authService.generateTokens(user);

    // Lưu refresh token vào HTTP-only cookie (bảo mật hơn)
    res.cookie('refresh_token', refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      maxAge: 7 * 24 * 60 * 60 * 1000, // 7 ngày
    });

    // Redirect về frontend kèm access token
    const frontendUrl = process.env.FRONTEND_URL;
    res.redirect(`${frontendUrl}/auth/callback?token=${accessToken}`);
  }

  // Refresh token
  @Post('refresh')
  @Public()
  @HttpCode(200)
  @ApiOperation({ summary: 'Làm mới access token' })
  async refresh(@Req() req: Request) {
    const refreshToken = req.cookies?.refresh_token;
    if (!refreshToken) throw new UnauthorizedException('Không có refresh token');
    return this.authService.refreshTokens(refreshToken);
  }

  // Lấy thông tin user hiện tại
  @Get('me')
  @ApiBearerAuth('access-token')
  @ApiOperation({ summary: 'Lấy thông tin user đang đăng nhập' })
  getMe(@CurrentUser() user: User) {
    return user;
  }

  // Đăng xuất
  @Post('logout')
  @HttpCode(200)
  @ApiBearerAuth('access-token')
  @ApiOperation({ summary: 'Đăng xuất' })
  logout(@Res({ passthrough: true }) res: Response) {
    res.clearCookie('refresh_token');
    return { message: 'Đăng xuất thành công' };
  }
}
```

---

## 10. User Entity

```typescript
// users/user.entity.ts
import { Entity, PrimaryGeneratedColumn, Column, CreateDateColumn, UpdateDateColumn } from 'typeorm';

export enum UserRole {
  USER = 'user',
  ADMIN = 'admin',
}

export enum AuthProvider {
  LOCAL = 'local',
  GOOGLE = 'google',
  FACEBOOK = 'facebook',
}

@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  email: string;

  @Column({ name: 'google_id', nullable: true, unique: true })
  googleId: string | null;

  @Column({ name: 'display_name', nullable: true })
  displayName: string | null;

  @Column({ name: 'first_name', nullable: true })
  firstName: string | null;

  @Column({ name: 'last_name', nullable: true })
  lastName: string | null;

  @Column({ nullable: true })
  avatar: string | null;

  @Column({ nullable: true, select: false })  // không SELECT password mặc định
  password: string | null;

  @Column({
    type: 'enum',
    enum: UserRole,
    default: UserRole.USER,
  })
  role: UserRole;

  @Column({
    name: 'auth_provider',
    type: 'enum',
    enum: AuthProvider,
    default: AuthProvider.LOCAL,
  })
  authProvider: AuthProvider;

  @Column({ name: 'is_email_verified', default: false })
  isEmailVerified: boolean;

  @Column({ name: 'is_active', default: true })
  isActive: boolean;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;
}
```

---

## 11. Auth Module

```typescript
// auth/auth.module.ts
import { Module } from '@nestjs/common';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { APP_GUARD } from '@nestjs/core';

import { AuthController } from './auth.controller';
import { AuthService } from './auth.service';
import { GoogleStrategy } from './strategies/google.strategy';
import { JwtStrategy } from './strategies/jwt.strategy';
import { JwtAuthGuard } from './guards/jwt-auth.guard';
import { User } from '../users/user.entity';

@Module({
  imports: [
    ConfigModule,
    PassportModule,
    TypeOrmModule.forFeature([User]),
    JwtModule.registerAsync({
      imports: [ConfigModule],
      useFactory: (config: ConfigService) => ({
        secret: config.get('JWT_ACCESS_SECRET'),
        signOptions: { expiresIn: config.get('JWT_ACCESS_EXPIRES') },
      }),
      inject: [ConfigService],
    }),
  ],
  controllers: [AuthController],
  providers: [
    AuthService,
    GoogleStrategy,
    JwtStrategy,
    // Áp dụng JwtAuthGuard toàn cục (dùng @Public() để bỏ qua)
    {
      provide: APP_GUARD,
      useClass: JwtAuthGuard,
    },
  ],
  exports: [AuthService, JwtModule],
})
export class AuthModule {}
```

---

## 12. RBAC – Phân quyền theo Role

```typescript
// auth/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';
import { UserRole } from '../../users/user.entity';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: UserRole[]) => SetMetadata(ROLES_KEY, roles);

// auth/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext, ForbiddenException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { UserRole } from '../../users/user.entity';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<UserRole[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (!requiredRoles || requiredRoles.length === 0) return true;

    const { user } = context.switchToHttp().getRequest();
    const hasRole = requiredRoles.includes(user?.role);

    if (!hasRole) throw new ForbiddenException('Bạn không có quyền thực hiện hành động này');
    return true;
  }
}

// Dùng trong Controller
@Get('admin/users')
@Roles(UserRole.ADMIN)
@UseGuards(RolesGuard)
getAllUsers() {}
```

---

## 13. Cookie vs LocalStorage – Lưu token ở đâu?

| | Cookie (httpOnly) | LocalStorage |
|---|---|---|
| XSS Attack | ✅ An toàn (JS không đọc được) | ❌ Dễ bị đánh cắp |
| CSRF Attack | ⚠️ Cần CSRF token | ✅ An toàn |
| Expire tự động | ✅ | ❌ Phải tự xóa |
| Server-side revoke | ✅ (xóa cookie) | ❌ Khó |
| **Khuyến nghị** | **Refresh token** | **Không dùng** |

**Pattern tốt nhất:**
- **Access token**: Lưu trong memory (biến JS) — ngắn hạn (15 phút)
- **Refresh token**: Lưu trong HTTP-only cookie — dài hạn (7 ngày)

---

## 14. Frontend Integration

```typescript
// React – Xử lý sau khi Google redirect về
// pages/auth/callback.tsx
import { useEffect } from 'react';
import { useRouter } from 'next/router';
import { useAuthStore } from '@/stores/auth';

export default function AuthCallback() {
  const router = useRouter();
  const setAccessToken = useAuthStore((s) => s.setAccessToken);

  useEffect(() => {
    const token = new URLSearchParams(window.location.search).get('token');
    if (token) {
      // Lưu vào memory store (Zustand/Redux) — KHÔNG lưu localStorage
      setAccessToken(token);
      router.replace('/dashboard');
    }
  }, []);

  return <div>Đang đăng nhập...</div>;
}

// axios interceptor – tự động refresh token khi 401
axiosInstance.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401 && !error.config._retry) {
      error.config._retry = true;
      // Gọi /auth/refresh — cookie refresh_token tự động gửi kèm
      const { data } = await axios.post('/auth/refresh', {}, { withCredentials: true });
      // Cập nhật access token mới vào store
      useAuthStore.getState().setAccessToken(data.accessToken);
      error.config.headers['Authorization'] = `Bearer ${data.accessToken}`;
      return axiosInstance(error.config);
    }
    return Promise.reject(error);
  }
);
```

---

## 15. Bảo mật nâng cao

### 15.1. Lưu Refresh Token vào DB (Revocation)

```typescript
// user.entity.ts – thêm field
@Column({ name: 'refresh_token_hash', nullable: true, select: false })
refreshTokenHash: string | null;

// auth.service.ts
import * as bcrypt from 'bcrypt';

async saveRefreshToken(userId: string, refreshToken: string) {
  const hash = await bcrypt.hash(refreshToken, 10);
  await this.userRepo.update(userId, { refreshTokenHash: hash });
}

async validateRefreshToken(userId: string, refreshToken: string): Promise<boolean> {
  const user = await this.userRepo.findOne({
    where: { id: userId },
    select: ['refreshTokenHash'],
  });
  if (!user?.refreshTokenHash) return false;
  return bcrypt.compare(refreshToken, user.refreshTokenHash);
}

async revokeRefreshToken(userId: string) {
  await this.userRepo.update(userId, { refreshTokenHash: null });
}
```

### 15.2. State parameter – Chống CSRF trong OAuth

```typescript
// Passport tự xử lý state parameter — bật lên:
super({
  clientID: config.get('GOOGLE_CLIENT_ID'),
  clientSecret: config.get('GOOGLE_CLIENT_SECRET'),
  callbackURL: config.get('GOOGLE_CALLBACK_URL'),
  scope: ['email', 'profile'],
  state: true,       // tự động sinh và verify state
});
```

### 15.3. Validate redirect_uri

```typescript
// Chỉ cho phép redirect về các domain tin cậy
googleCallback(@Req() req, @Res() res: Response, @Query('state') state: string) {
  const allowedOrigins = [process.env.FRONTEND_URL];
  const redirectUrl = new URL(`${process.env.FRONTEND_URL}/auth/callback`);
  // Luôn redirect về URL cố định, KHÔNG dùng URL từ query string
  res.redirect(redirectUrl.toString() + `?token=${accessToken}`);
}
```

---

## 16. Checklist OAuth2 Google NestJS

- [ ] Tạo Google OAuth credentials trên Google Cloud Console
- [ ] Thêm đúng Authorized redirect URIs (dev + prod)
- [ ] Không commit `.env` lên Git
- [ ] `GoogleStrategy` có `validate()` xử lý `findOrCreate` user
- [ ] `JwtStrategy` dùng làm Global Guard với `@Public()` để opt-out
- [ ] Access token ngắn hạn (15 phút), Refresh token dài hạn (7 ngày)
- [ ] Refresh token lưu trong HTTP-only cookie
- [ ] Access token lưu trong memory, KHÔNG lưu localStorage
- [ ] Lưu hash của refresh token vào DB để revoke được
- [ ] Bật `state: true` trong GoogleStrategy để chống CSRF
- [ ] Redirect URL sau callback là URL cố định, không lấy từ user input
- [ ] HTTPS bắt buộc ở production (cookie `secure: true`)
- [ ] `sameSite: 'lax'` hoặc `'strict'` cho cookie
- [ ] Xử lý trường hợp user dùng cả email/password lẫn Google (account linking)
