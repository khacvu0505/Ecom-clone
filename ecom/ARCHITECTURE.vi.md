# Ecom - Tài Liệu Kiến Trúc Dự Án

## 1. Tổng Quan

Ứng dụng **backend API thương mại điện tử** xây dựng trên **NestJS 11** với **TypeScript**, sử dụng:

- **Prisma** (PostgreSQL) - ORM & cơ sở dữ liệu
- **Redis** - Caching, hàng đợi BullMQ, adapter WebSocket
- **Socket.io** - Giao tiếp thời gian thực
- **Zod** - Xác thực request/response
- **JWT** - Xác thực (cặp access token + refresh token)
- **Swagger** - Tài liệu API tự động tại `/api`

---

## 2. Cấu Trúc Dự Án

```
ecom/
├── src/                         # Mã nguồn chính
│   ├── main.ts                  # Khởi tạo app: Express, Swagger, Helmet, CORS, WebSocket
│   ├── app.module.ts            # Module gốc - import tất cả feature modules + cấu hình global
│   ├── app.controller.ts        # Controller gốc
│   ├── app.service.ts           # Service gốc
│   ├── types.ts                 # Khai báo kiểu dữ liệu global
│   │
│   ├── routes/                  # Các feature module (nghiệp vụ)
│   ├── shared/                  # Tiện ích dùng chung (@Global module)
│   ├── websockets/              # Socket.io gateways + Redis adapter
│   ├── queues/                  # BullMQ consumers (xử lý job nền)
│   ├── cronjobs/                # Tác vụ theo lịch (NestJS Schedule)
│   ├── i18n/                    # File dịch đa ngôn ngữ (en/ + vi/)
│   └── generated/               # Kiểu i18n tự động sinh
│
├── prisma/                      # Prisma schema + migrations
│   ├── schema.prisma            # Định nghĩa schema cơ sở dữ liệu
│   └── migrations/              # 28 file migration
│
├── emails/                      # Template email React (@react-email)
├── initialScript/               # Script khởi tạo dữ liệu ban đầu
├── test/                        # Test end-to-end
│
├── .env.example                 # Mẫu biến môi trường cần thiết
├── package.json                 # Dependencies & scripts
├── tsconfig.json                # Cấu hình TypeScript (strict mode, decorators, JSX)
├── eslint.config.mjs            # Cấu hình ESLint v9 flat config
├── .prettierrc                  # Prettier: single quotes, 120 ký tự, thụt 2 space
└── nest-cli.json                # Cấu hình NestJS CLI
```

---

## 3. Các Feature Module (`src/routes/`)

### 3.1 Mẫu Module Chuẩn

Mỗi feature module đều tuân theo **cấu trúc phân lớp nhất quán**:

```
feature/
├── feature.module.ts        # Định nghĩa NestJS module
├── feature.controller.ts    # Các endpoint HTTP + decorators (@Auth, @ZodResponse, ...)
├── feature.service.ts       # Lớp xử lý nghiệp vụ (business logic)
├── feature.repo.ts          # Lớp truy cập dữ liệu (Prisma queries)
├── feature.dto.ts           # DTO = Zod schemas bọc bằng createZodDto()
├── feature.model.ts         # Zod schemas + kiểu TypeScript (suy ra từ Zod)
├── feature.error.ts         # Exception riêng của feature, dùng khóa i18n
└── feature.producer.ts      # BullMQ job producer (chỉ có ở order, payment)
```

**Trách nhiệm từng lớp:**
- **Controller** - Chỉ xử lý HTTP: routing, decorators, lấy tham số từ request
- **Service** - Chứa toàn bộ business logic, điều phối giữa repo và các service khác
- **Repo** - Lớp truy cập dữ liệu thuần, xây dựng Prisma query, trả về kết quả có kiểu
- **DTO** - Class xác thực đầu vào dựa trên Zod (dùng ở tham số controller)
- **Model** - Zod schemas định nghĩa cả xác thực đầu vào và hình dạng response
- **Error** - Class exception riêng của feature, sử dụng khóa i18n cho thông báo lỗi

### 3.2 Danh Sách Các Module

| Module | Route | Mô tả |
|--------|-------|-------|
| **auth** | `/auth` | Đăng ký, đăng nhập, đăng xuất, refresh token, OTP, 2FA (TOTP), Google OAuth, quên mật khẩu |
| **user** | `/users` | Quản lý CRUD người dùng (admin) |
| **profile** | `/profile` | Quản lý hồ sơ người dùng hiện tại |
| **role** | `/roles` | Quản lý vai trò RBAC |
| **permission** | `/permissions` | CRUD quyền (dựa trên path + HTTP method) |
| **language** | `/languages` | Quản lý ngôn ngữ/locale |
| **product** | `/products` | Danh sách + chi tiết sản phẩm công khai. Có thêm `manage-product.controller.ts` cho admin CRUD |
| **product-translation** | lồng trong product | Nội dung sản phẩm đa ngôn ngữ |
| **category** | `/categories` | Danh mục sản phẩm (hỗ trợ phân cấp qua `parentCategoryId`) |
| **category-translation** | lồng trong category | Nội dung danh mục đa ngôn ngữ |
| **brand** | `/brands` | Quản lý thương hiệu |
| **brand-translation** | lồng trong brand | Nội dung thương hiệu đa ngôn ngữ |
| **media** | `/media` | Upload file lên AWS S3 |
| **cart** | `/cart` | Giỏ hàng (unique theo user + SKU) |
| **order** | `/orders` | Quản lý đơn hàng với luồng trạng thái |
| **payment** | `/payment` | Xử lý thanh toán với xác thực API key |
| **review** | `/reviews` | Đánh giá sản phẩm kèm media đính kèm |

---

## 4. Shared Module (`src/shared/`)

`SharedModule` được đánh dấu `@Global()`, tất cả exports sẽ tự động khả dụng ở mọi module mà không cần import tường minh.

### 4.1 Services

| Service | File | Mô tả |
|---------|------|-------|
| `PrismaService` | `services/prisma.service.ts` | Singleton Prisma client để truy cập database |
| `TokenService` | `services/token.service.ts` | Ký/xác minh JWT bằng thuật toán HS256 |
| `HashingService` | `services/hashing.service.ts` | Hash & so sánh mật khẩu bằng bcrypt |
| `EmailService` | `services/email.service.ts` | Gửi email qua Resend API với template React |
| `S3Service` | `services/s3.service.ts` | Upload file lên AWS S3 + tạo presigned URL |
| `TwoFactorService` | `services/2fa.service.ts` | Tạo & xác minh TOTP cho xác thực 2 lớp |

### 4.2 Shared Repositories

| Repository | File | Mô tả |
|------------|------|-------|
| `SharedUserRepository` | `repositories/shared-user.repo.ts` | Truy vấn user chung với lọc soft-delete |
| `SharedRoleRepository` | `repositories/shared-role.repo.ts` | Truy vấn role + permissions kèm cache Redis (TTL 1 giờ) |
| `SharedPaymentRepository` | `repositories/shared-payment.repo.ts` | Truy vấn payment chung |
| `SharedWebsocketRepository` | `repositories/shared-websocket.repo.ts` | Theo dõi kết nối WebSocket |

### 4.3 Guards (Xác Thực & Phân Quyền)

```
HTTP Request
    │
    ▼
AuthenticationGuard (đọc metadata @Auth() từ controller/method)
    ├── AuthType.None           → route @IsPublic() → bỏ qua xác thực
    ├── AuthType.Bearer         → AccessTokenGuard → xác minh JWT + kiểm tra quyền role (cache Redis)
    └── AuthType.PaymentAPIKey  → PaymentAPIKeyGuard → xác minh API key từ header
```

- **AuthenticationGuard** (`guards/authentication.guard.ts`) - Guard đầu vào, đọc metadata `@Auth()` và chuyển tiếp đến guard phù hợp. Hỗ trợ điều kiện `AND` / `OR` để kết hợp nhiều loại xác thực.
- **AccessTokenGuard** (`guards/access-token.guard.ts`) - Xác minh JWT access token, trích xuất thông tin user, cache quyền role trong Redis với key `role:{roleId}` (TTL 1 giờ).
- **PaymentAPIKeyGuard** (`guards/payment-api-key.guard.ts`) - Xác minh header `authorization` với biến env `PAYMENT_API_KEY`.
- **ThrottlerBehindProxyGuard** (`guards/throttler-behind-proxy.guard.ts`) - Giới hạn tốc độ có hỗ trợ proxy header.

### 4.4 Decorators

| Decorator | File | Cách dùng |
|-----------|------|-----------|
| `@Auth([AuthType.Bearer])` | `decorators/auth.decorator.ts` | Đặt loại xác thực cho controller/method |
| `@IsPublic()` | `decorators/auth.decorator.ts` | Viết tắt của `@Auth([AuthType.None])` - bỏ qua xác thực |
| `@ActiveUser()` | `decorators/active-user.decorator.ts` | Inject user đang đăng nhập từ request. Hỗ trợ truy cập property: `@ActiveUser('userId')` |
| `@UserAgent()` | `decorators/user-agent.decorator.ts` | Trích xuất header user-agent từ request |
| `@ActiveRolePermissions()` | `decorators/active-role-permissions.decorator.ts` | Inject quyền role từ request |
| `@SerializeAll()` | `constants/serialize.decorator.ts` | Đánh dấu class repo để lọc field trả về |

### 4.5 Pipes, Filters, Interceptors

| Component | File | Mô tả |
|-----------|------|-------|
| `CustomZodValidationPipe` | `pipes/custom-zod-validation.pipe.ts` | Pipe xác thực global sử dụng Zod schemas |
| `HttpExceptionFilter` | `filters/http-exception.filter.ts` | Bắt `HttpException`, log `ZodSerializationException` |
| `CatchEverythingFilter` | `filters/catch-everything.filter.ts` | Filter dự phòng cho exception không xử lý được |
| `ZodSerializerInterceptor` | từ `nestjs-zod` | Serialize response theo schema `@ZodResponse()` |
| `TransformInterceptor` | `interceptor/transform.interceptor.ts` | Biến đổi response |

### 4.6 Constants

| File | Nội dung |
|------|----------|
| `auth.constant.ts` | `AuthType` (Bearer/None/PaymentAPIKey), `ConditionGuard` (And/Or), `UserStatus`, `TypeOfVerificationCode` |
| `role.constant.ts` | Hằng số tên vai trò |
| `order.constant.ts` | Hằng số trạng thái đơn hàng |
| `payment.constant.ts` | Hằng số trạng thái thanh toán |
| `media.constant.ts` | Hằng số loại media |
| `queue.constant.ts` | Hằng số tên hàng đợi BullMQ |
| `other.constant.ts` | Giá trị mặc định phân trang, `ALL_LANGUAGE_CODE`, tùy chọn sắp xếp |

### 4.7 Shared Models & DTOs

- **`shared/models/`** - Zod schemas và kiểu TypeScript suy ra, dùng chung giữa các module (vd: `SharedProductModel`, `SharedUserModel`, `SharedRoleModel`, ...)
- **`shared/dtos/`** - Class DTO tạo từ shared Zod schemas (`MessageResDTO`, `EmptyBodyDTO`, `PaginationDTO`)

---

## 5. Vòng Đời Request

Luồng xử lý hoàn chỉnh của một HTTP request qua ứng dụng:

```
1. HTTP Request đến
       │
2. ThrottlerBehindProxyGuard
   └── Giới hạn: 5 request/phút (short), 7 request/2 phút (long)
       │
3. AuthenticationGuard
   └── Đọc metadata @Auth() → chuyển đến guard cụ thể
   └── Mặc định: AuthType.Bearer (nếu không có decorator)
       │
4. CustomZodValidationPipe
   └── Xác thực @Body(), @Query(), @Param() bằng Zod schemas từ class DTO
       │
5. Controller method thực thi
   └── Trích xuất dữ liệu qua decorators (@ActiveUser, @UserAgent, @Ip)
   └── Gọi lớp Service
       │
6. Lớp Service
   └── Xử lý business logic, điều phối
   └── Gọi lớp Repo để truy cập dữ liệu
       │
7. Lớp Repo
   └── Truy vấn Prisma với lọc soft-delete (deletedAt: null)
   └── Trả về kết quả có kiểu dữ liệu
       │
8. ZodSerializerInterceptor
   └── Serialize response theo schema @ZodResponse()
   └── Loại bỏ các field không khai báo trong response schema
       │
9. HttpExceptionFilter (khi có lỗi)
   └── Bắt exception, định dạng response lỗi
   └── Log lỗi ZodSerializationException
       │
10. HTTP Response gửi về client
```

---

## 6. Schema Cơ Sở Dữ Liệu (Prisma)

### 6.1 Vị Trí Schema

`prisma/schema.prisma` — Cơ sở dữ liệu PostgreSQL với generator `prisma-client-js` và `prisma-json-types-generator` cho kiểu JSON.

### 6.2 Các Mẫu (Pattern) Chung Trên Tất Cả Model

| Pattern | Mô tả |
|---------|-------|
| **Xóa mềm (Soft delete)** | `deletedAt DateTime?` + `@@index([deletedAt])`. Tất cả truy vấn đều lọc `deletedAt: null` |
| **Nhật ký thao tác (Audit trail)** | `createdById`, `updatedById`, `deletedById` — theo dõi ai thực hiện mỗi thao tác |
| **Timestamp** | `createdAt DateTime @default(now())`, `updatedAt DateTime @updatedAt` |
| **Named relations** | Bắt buộc khi cùng model có nhiều quan hệ đến model khác. VD: `@relation("ProductCreatedBy")`, `@relation("ProductUpdatedBy")` |
| **Pattern đa ngôn ngữ** | Entity → EntityTranslation (1-N theo ngôn ngữ). Dùng cho Product, Category, Brand, User |
| **Nhiều-nhiều ẩn (Implicit M-N)** | Prisma tự tạo bảng trung gian cho `Product ↔ Category`, `Role ↔ Permission` |

### 6.3 Sơ Đồ Quan Hệ Entity

```
User ──1:N──> Device ──1:N──> RefreshToken
  │
  ├──N:1──> Role ──N:M──> Permission
  │
  ├──1:N──> UserTranslation ──N:1──> Language
  │
  ├──1:N──> CartItem ──N:1──> SKU
  │
  ├──1:N──> Order ──N:1──> Payment
  │   │          └──1:N──> ProductSKUSnapshot
  │   │
  │   └──1:N──> Review ──1:N──> ReviewMedia
  │
  ├──1:N──> Message (gửi/nhận)
  └──1:N──> Websocket

Product ──1:N──> SKU
   │      ──1:N──> ProductTranslation ──N:1──> Language
   │      ──N:1──> Brand ──1:N──> BrandTranslation ──N:1──> Language
   │      ──N:M──> Category ──1:N──> CategoryTranslation ──N:1──> Language
   │                  └── Tự quan hệ (parentCategoryId) cho phân cấp
   └──N:M──> Order

VerificationCode (độc lập - mã OTP email)
PaymentTransaction (độc lập - log giao dịch thanh toán)
```

### 6.4 Enums

| Enum | Giá trị |
|------|---------|
| `UserStatus` | `ACTIVE`, `INACTIVE`, `BLOCKED` |
| `OrderStatus` | `PENDING_PAYMENT`, `PENDING_PICKUP`, `PENDING_DELIVERY`, `DELIVERED`, `RETURNED`, `CANCELLED` |
| `PaymentStatus` | `PENDING`, `SUCCESS`, `FAILED` |
| `HTTPMethod` | `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `OPTIONS`, `HEAD` |
| `VerificationCodeType` | `REGISTER`, `FORGOT_PASSWORD`, `LOGIN`, `DISABLE_2FA` |
| `MediaType` | `IMAGE`, `VIDEO` |

---

## 7. Xác Thực & Phân Quyền

### 7.1 Các Phương Thức Xác Thực

| Phương thức | Mô tả |
|-------------|-------|
| **JWT Bearer Token** | Access token (ngắn hạn) + Refresh token (dài hạn, lưu trong DB) |
| **Google OAuth 2.0** | Đăng nhập mạng xã hội qua Google, redirect về kèm tokens |
| **Payment API Key** | API key tĩnh cho endpoint webhook thanh toán |
| **Public (None)** | Không yêu cầu xác thực |

### 7.2 Luồng Token

```
Đăng ký → Gửi OTP qua email → Xác minh OTP → Kích hoạt tài khoản

Đăng nhập (email + mật khẩu)
   ├── Nếu chưa bật 2FA → Trả về { accessToken, refreshToken }
   └── Nếu đã bật 2FA   → Yêu cầu mã TOTP → Trả về { accessToken, refreshToken }

Access Token hết hạn → Gọi /auth/refresh-token với refreshToken
   └── Trả về cặp { accessToken, refreshToken } mới

Đăng xuất → Vô hiệu hóa refresh token trong DB
```

### 7.3 RBAC (Phân Quyền Dựa Trên Vai Trò)

- Mỗi **User** có một **Role**
- Mỗi **Role** có nhiều **Permission**
- Mỗi **Permission** định nghĩa: `path` (pattern route API) + `method` (HTTP verb)
- `AccessTokenGuard` kiểm tra role của user có quyền truy cập endpoint được yêu cầu không
- Permissions được cache trong Redis 1 giờ theo role (`role:{roleId}`)

### 7.4 2FA (Xác Thực Hai Lớp)

- Sử dụng TOTP (Time-based One-Time Password - Mật khẩu dùng một lần dựa trên thời gian)
- Thiết lập: tạo secret + URI mã QR
- Đăng nhập: yêu cầu mã TOTP ngoài mật khẩu
- Tắt 2FA: yêu cầu xác minh OTP qua email

---

## 8. Xác Thực Dữ Liệu & Serialization (Zod)

### 8.1 Xác Thực Đầu Vào

Toàn bộ xác thực sử dụng **Zod** schemas (không dùng class-validator). Mẫu chuẩn:

```typescript
// 1. Định nghĩa Zod schema trong feature.model.ts
export const CreateProductBodySchema = z.object({
  name: z.string().min(1).max(500),
  basePrice: z.number().positive(),
  // ...
})
export type CreateProductBodyType = z.infer<typeof CreateProductBodySchema>

// 2. Tạo class DTO trong feature.dto.ts
export class CreateProductBodyDTO extends createZodDto(CreateProductBodySchema) {}

// 3. Sử dụng trong controller
@Post()
create(@Body() body: CreateProductBodyDTO) { ... }
```

### 8.2 Serialization Response

Hình dạng response cũng được kiểm soát bởi Zod schemas qua `@ZodResponse()`:

```typescript
@Get()
@ZodResponse({ type: GetProductsResDTO })
list(@Query() query: GetProductsQueryDTO) {
  return this.productService.list({ query })
}
```

`ZodSerializerInterceptor` sẽ loại bỏ các field từ response mà không được định nghĩa trong response schema.

---

## 9. Đa Ngôn Ngữ (i18n)

- **Thư viện:** `nestjs-i18n` v10.5.1
- **Ngôn ngữ:** Tiếng Anh (mặc định) + Tiếng Việt
- **File dịch:** `src/i18n/en/error.json`, `src/i18n/vi/error.json`
- **Kiểu tự động sinh:** `src/generated/i18n.generated.ts`
- **Phân giải ngôn ngữ:** Query param `?lang=vi` → header `Accept-Language` → fallback về `en`
- **Sử dụng trong lỗi:** Thông báo exception dùng khóa i18n như `"Error.NotFound"`, `"Error.InvalidPassword"`

---

## 10. Tính Năng Thời Gian Thực (WebSockets)

### 10.1 Cài Đặt

- **Thư viện:** Socket.io với `@socket.io/redis-adapter` hỗ trợ multi-instance
- **Vị trí:** `src/websockets/`
- **Adapter:** `WebsocketAdapter` kế thừa NestJS `IoAdapter`, thêm middleware xác thực JWT khi kết nối
- **Luồng kết nối:** Client kết nối → JWT được xác minh → user join room `userId:{id}`

### 10.2 Gateways

| Gateway | File | Mô tả |
|---------|------|-------|
| `PaymentGateway` | `websockets/payment.gateway.ts` | Phát cập nhật trạng thái thanh toán thời gian thực |
| `ChatGateway` | `websockets/chat.gateway.ts` | Nhắn tin giữa người dùng, xác nhận đã đọc |

---

## 11. Job Nền & Tác Vụ Theo Lịch

### 11.1 Hàng Đợi BullMQ

- **Thư viện:** `@nestjs/bullmq` (dựa trên Redis)
- **Queue:** `PAYMENT_QUEUE` (định nghĩa trong `shared/constants/queue.constant.ts`)
- **Consumer:** `PaymentConsumer` (`src/queues/payment.consumer.ts`) - Xử lý job tự động hủy đơn chưa thanh toán
- **Producers:** `order.producer.ts`, `payment.producer.ts` - Đẩy job vào hàng đợi với delay

### 11.2 Cron Jobs

| Job | File | Lịch chạy | Mô tả |
|-----|------|-----------|-------|
| `RemoveRefreshTokenCronjob` | `src/cronjobs/remove-refresh-token.cronjob.ts` | Hàng ngày lúc 1:00 AM | Xóa refresh token đã hết hạn khỏi database |

---

## 12. Cấu Hình

### 12.1 Biến Môi Trường

Tất cả biến môi trường được xác thực khi khởi động bằng Zod trong `src/shared/config.ts`. Nếu bất kỳ biến bắt buộc nào thiếu hoặc không hợp lệ, ứng dụng sẽ thoát ngay lập tức.

Xem `.env.example` để biết danh sách đầy đủ. Các nhóm chính:

| Nhóm | Biến |
|------|------|
| **Database** | `DATABASE_URL` (chuỗi kết nối PostgreSQL) |
| **JWT** | `ACCESS_TOKEN_SECRET`, `ACCESS_TOKEN_EXPIRES_IN`, `REFRESH_TOKEN_SECRET`, `REFRESH_TOKEN_EXPIRES_IN` |
| **Admin khởi tạo** | `ADMIN_NAME`, `ADMIN_EMAIL`, `ADMIN_PASSWORD`, `ADMIN_PHONE_NUMBER` |
| **Email** | `RESEND_API_KEY`, `OTP_EXPIRES_IN` |
| **Google OAuth** | `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REDIRECT_URI`, `GOOGLE_CLIENT_REDIRECT_URI` |
| **AWS S3** | `S3_REGION`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_BUCKET_NAME`, `S3_ENPOINT` |
| **Redis** | `REDIS_URL` |
| **Thanh toán** | `PAYMENT_API_KEY` |
| **App** | `APP_NAME`, `PREFIX_STATIC_ENPOINT` |

### 12.2 Giới Hạn Tốc Độ (Rate Limiting)

Cấu hình trong `app.module.ts` qua `@nestjs/throttler`:

| Throttle | TTL | Giới hạn |
|----------|-----|----------|
| `short` | 60 giây | 5 request |
| `long` | 120 giây | 7 request |

Controller có thể bỏ qua throttle bằng `@SkipThrottle()`.

### 12.3 Logging

- **Thư viện:** `nestjs-pino`
- **Đầu ra:** `logs/app.log` (ghi file bất đồng bộ)
- **Serialization:** Log method, URL, query, params của request + status code của response

---

## 13. Global Providers (app.module.ts)

Các provider được đăng ký ở cấp ứng dụng:

| Provider | Loại | Mô tả |
|----------|------|-------|
| `CustomZodValidationPipe` | `APP_PIPE` | Xác thực request toàn cục |
| `ZodSerializerInterceptor` | `APP_INTERCEPTOR` | Serialization response toàn cục |
| `HttpExceptionFilter` | `APP_FILTER` | Xử lý exception toàn cục |
| `ThrottlerBehindProxyGuard` | `APP_GUARD` | Giới hạn tốc độ toàn cục |
| `AuthenticationGuard` | `APP_GUARD` (qua SharedModule) | Xác thực toàn cục |
| `PaymentConsumer` | Provider | BullMQ worker cho job thanh toán |
| `RemoveRefreshTokenCronjob` | Provider | Dọn dẹp token theo lịch |

---

## 14. Tham Khảo Thư Viện Chính

| Thư viện | Phiên bản | Mục đích |
|----------|-----------|----------|
| `@nestjs/core` | 11.1.6 | Framework NestJS |
| `@prisma/client` | 6.14.0 | ORM cho PostgreSQL |
| `zod` | 4.1.0 | Xác thực schema |
| `nestjs-zod` | 5.0.0 | Tích hợp NestJS + Zod (DTOs, serialization, Swagger) |
| `@nestjs/jwt` | 11.0.0 | Xử lý JWT token |
| `@nestjs/bullmq` | 5.58.0 | Hàng đợi job nền |
| `socket.io` | 4.8.1 | WebSocket server |
| `@socket.io/redis-adapter` | - | Hỗ trợ WebSocket multi-instance |
| `nestjs-i18n` | 10.5.1 | Đa ngôn ngữ |
| `@nestjs/throttler` | 6.4.0 | Giới hạn tốc độ |
| `nestjs-pino` | 4.4.0 | Structured logging |
| `@nestjs/cache-manager` | - | Cache Redis |
| `@aws-sdk/client-s3` | - | Lưu trữ file S3 |
| `resend` | 6.0.1 | Email giao dịch |
| `@react-email/components` | - | Template email |
| `helmet` | 8.1.0 | Header bảo mật HTTP |
| `@nestjs/swagger` | - | Tài liệu API tại `/api` |

---

## 15. Các Lệnh Phát Triển Thường Dùng

```bash
# Phát triển
npm run start:dev              # Chế độ watch với hot reload
npm run start:debug            # Chế độ debug với watch

# Build
npm run build                  # Biên dịch TypeScript

# Testing
npm run test                   # Unit tests (Jest)
npm run test:e2e               # Test end-to-end
npm run test:cov               # Báo cáo coverage
npx jest --testPathPattern=<pattern>  # Chạy một test cụ thể

# Lint & Format
npm run lint                   # ESLint với auto-fix
npm run format                 # Prettier

# Cơ sở dữ liệu
npx prisma migrate dev         # Tạo/chạy migrations
npx prisma generate            # Tạo lại Prisma client sau khi thay đổi schema
npm run init-seed-data         # Khởi tạo dữ liệu mẫu
npm run create-permissions     # Khởi tạo quyền RBAC

# Template Email
npm run email:dev              # Server xem trước React Email
npm run email:build            # Build template email
npm run email:export           # Export template email
```
