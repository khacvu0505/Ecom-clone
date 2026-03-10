# Ecom - Project Architecture Documentation

## 1. Overview

A **NestJS 11** e-commerce backend API built with **TypeScript**, using:

- **Prisma** (PostgreSQL) - ORM & database
- **Redis** - Caching, BullMQ job queues, WebSocket adapter
- **Socket.io** - Real-time communication
- **Zod** - Request/response validation
- **JWT** - Authentication (access + refresh token pattern)
- **Swagger** - Auto-generated API docs at `/api`

---

## 2. Project Structure

```
ecom/
├── src/                         # Main source code
│   ├── main.ts                  # Bootstrap: Express, Swagger, Helmet, CORS, WebSocket
│   ├── app.module.ts            # Root module - imports all feature modules + global config
│   ├── app.controller.ts        # Root controller
│   ├── app.service.ts           # Root service
│   ├── types.ts                 # Global type definitions
│   │
│   ├── routes/                  # Feature modules (business domains)
│   ├── shared/                  # Global shared utilities (@Global module)
│   ├── websockets/              # Socket.io gateways + Redis adapter
│   ├── queues/                  # BullMQ job consumers
│   ├── cronjobs/                # Scheduled tasks (NestJS Schedule)
│   ├── i18n/                    # Translation files (en/ + vi/)
│   └── generated/               # Auto-generated i18n types
│
├── prisma/                      # Prisma schema + migrations
│   ├── schema.prisma            # Database schema definition
│   └── migrations/              # 28 database migrations
│
├── emails/                      # React email templates (@react-email)
├── initialScript/               # Database seeding scripts
├── test/                        # E2E tests
│
├── .env.example                 # Required environment variables template
├── package.json                 # Dependencies & scripts
├── tsconfig.json                # TypeScript config (strict mode, decorators, JSX)
├── eslint.config.mjs            # ESLint v9 flat config
├── .prettierrc                  # Prettier: single quotes, 120 chars, 2-space indent
└── nest-cli.json                # NestJS CLI config
```

---

## 3. Feature Modules (`src/routes/`)

### 3.1 Module Pattern

Every feature module follows a **consistent layered structure**:

```
feature/
├── feature.module.ts        # NestJS module definition
├── feature.controller.ts    # HTTP endpoints + decorators (@Auth, @ZodResponse, etc.)
├── feature.service.ts       # Business logic layer
├── feature.repo.ts          # Data access layer (Prisma queries)
├── feature.dto.ts           # DTOs = Zod schemas wrapped with createZodDto()
├── feature.model.ts         # Zod schemas + TypeScript types (inferred from Zod)
├── feature.error.ts         # Custom exceptions with i18n error keys
└── feature.producer.ts      # BullMQ job producer (only in order, payment)
```

**Responsibilities:**
- **Controller** - Only handles HTTP concerns: routing, decorators, parameter extraction
- **Service** - Contains all business logic, orchestrates between repo and other services
- **Repo** - Pure data access layer, builds Prisma queries, returns typed results
- **DTO** - Zod-based input validation classes (used by controller parameters)
- **Model** - Zod schemas that define both input validation and response serialization shapes
- **Error** - Feature-specific exception classes using i18n keys for messages

### 3.2 Module List

| Module | Route | Description |
|--------|-------|-------------|
| **auth** | `/auth` | Register, login, logout, refresh token, OTP, 2FA (TOTP), Google OAuth, forgot password |
| **user** | `/users` | User CRUD management (admin) |
| **profile** | `/profile` | Current user profile management |
| **role** | `/roles` | RBAC role management |
| **permission** | `/permissions` | Permission CRUD (path + HTTP method based) |
| **language** | `/languages` | Language/locale management |
| **product** | `/products` | Public product listing + detail. Has separate `manage-product.controller.ts` for admin CRUD |
| **product-translation** | nested under product | Multi-language product content |
| **category** | `/categories` | Product categories (supports hierarchy via `parentCategoryId`) |
| **category-translation** | nested under category | Multi-language category content |
| **brand** | `/brands` | Brand management |
| **brand-translation** | nested under brand | Multi-language brand content |
| **media** | `/media` | File upload to AWS S3 |
| **cart** | `/cart` | Shopping cart (per user + SKU unique constraint) |
| **order** | `/orders` | Order management with status flow |
| **payment** | `/payment` | Payment processing with API key authentication |
| **review** | `/reviews` | Product reviews with media attachments |

---

## 4. Shared Module (`src/shared/`)

The `SharedModule` is decorated with `@Global()`, making all its exports available to every module without explicit imports.

### 4.1 Services

| Service | File | Description |
|---------|------|-------------|
| `PrismaService` | `services/prisma.service.ts` | Singleton Prisma client for database access |
| `TokenService` | `services/token.service.ts` | JWT sign/verify using HS256 algorithm |
| `HashingService` | `services/hashing.service.ts` | bcrypt password hash & compare |
| `EmailService` | `services/email.service.ts` | Send emails via Resend API with React templates |
| `S3Service` | `services/s3.service.ts` | AWS S3 file upload + presigned URL generation |
| `TwoFactorService` | `services/2fa.service.ts` | TOTP generation & validation for 2FA |

### 4.2 Shared Repositories

| Repository | File | Description |
|------------|------|-------------|
| `SharedUserRepository` | `repositories/shared-user.repo.ts` | Common user queries with soft-delete filtering |
| `SharedRoleRepository` | `repositories/shared-role.repo.ts` | Role + permissions queries with Redis cache (1-hour TTL) |
| `SharedPaymentRepository` | `repositories/shared-payment.repo.ts` | Common payment queries |
| `SharedWebsocketRepository` | `repositories/shared-websocket.repo.ts` | WebSocket connection tracking |

### 4.3 Guards (Authentication & Authorization)

```
HTTP Request
    │
    ▼
AuthenticationGuard (reads @Auth() metadata from controller/method)
    ├── AuthType.None      → @IsPublic() routes → skip authentication
    ├── AuthType.Bearer    → AccessTokenGuard → JWT verify + role permission check (Redis cached)
    └── AuthType.PaymentAPIKey → PaymentAPIKeyGuard → validates API key from header
```

- **AuthenticationGuard** (`guards/authentication.guard.ts`) - Entry point guard, reads `@Auth()` decorator metadata and delegates to the appropriate guard. Supports `AND` / `OR` conditions for combining multiple auth types.
- **AccessTokenGuard** (`guards/access-token.guard.ts`) - Validates JWT access token, extracts user info, caches role permissions in Redis with key `role:{roleId}` (1-hour TTL).
- **PaymentAPIKeyGuard** (`guards/payment-api-key.guard.ts`) - Validates `authorization` header against `PAYMENT_API_KEY` env var.
- **ThrottlerBehindProxyGuard** (`guards/throttler-behind-proxy.guard.ts`) - Rate limiting that respects proxy headers.

### 4.4 Decorators

| Decorator | File | Usage |
|-----------|------|-------|
| `@Auth([AuthType.Bearer])` | `decorators/auth.decorator.ts` | Set authentication type for controller/method |
| `@IsPublic()` | `decorators/auth.decorator.ts` | Shorthand for `@Auth([AuthType.None])` - skip authentication |
| `@ActiveUser()` | `decorators/active-user.decorator.ts` | Inject current authenticated user from request. Supports property access: `@ActiveUser('userId')` |
| `@UserAgent()` | `decorators/user-agent.decorator.ts` | Extract user-agent header from request |
| `@ActiveRolePermissions()` | `decorators/active-role-permissions.decorator.ts` | Inject role permissions from request |
| `@SerializeAll()` | `constants/serialize.decorator.ts` | Mark repo class for field filtering on responses |

### 4.5 Pipes, Filters, Interceptors

| Component | File | Description |
|-----------|------|-------------|
| `CustomZodValidationPipe` | `pipes/custom-zod-validation.pipe.ts` | Global validation pipe using Zod schemas |
| `HttpExceptionFilter` | `filters/http-exception.filter.ts` | Catches `HttpException`, logs `ZodSerializationException` |
| `CatchEverythingFilter` | `filters/catch-everything.filter.ts` | Fallback filter for unhandled exceptions |
| `ZodSerializerInterceptor` | from `nestjs-zod` | Serializes responses using `@ZodResponse()` schema |
| `TransformInterceptor` | `interceptor/transform.interceptor.ts` | Response transformation |

### 4.6 Constants

| File | Contents |
|------|----------|
| `auth.constant.ts` | `AuthType` (Bearer/None/PaymentAPIKey), `ConditionGuard` (And/Or), `UserStatus`, `TypeOfVerificationCode` |
| `role.constant.ts` | Role name constants |
| `order.constant.ts` | Order status constants |
| `payment.constant.ts` | Payment status constants |
| `media.constant.ts` | Media type constants |
| `queue.constant.ts` | BullMQ queue name constants |
| `other.constant.ts` | Pagination defaults, `ALL_LANGUAGE_CODE`, sort/order options |

### 4.7 Shared Models & DTOs

- **`shared/models/`** - Zod schemas and inferred TypeScript types shared across modules (e.g., `SharedProductModel`, `SharedUserModel`, `SharedRoleModel`, etc.)
- **`shared/dtos/`** - DTO classes created from shared Zod schemas (`MessageResDTO`, `EmptyBodyDTO`, `PaginationDTO`)

---

## 5. Request Lifecycle

Complete flow of an HTTP request through the application:

```
1. HTTP Request arrives
       │
2. ThrottlerBehindProxyGuard
   └── Rate limiting: 5 requests/min (short), 7 requests/2min (long)
       │
3. AuthenticationGuard
   └── Reads @Auth() metadata → delegates to specific guard
   └── Default: AuthType.Bearer (if no decorator specified)
       │
4. CustomZodValidationPipe
   └── Validates @Body(), @Query(), @Param() using Zod schemas from DTO classes
       │
5. Controller method executes
   └── Extracts data via decorators (@ActiveUser, @UserAgent, @Ip)
   └── Calls Service layer
       │
6. Service layer
   └── Business logic, orchestration
   └── Calls Repo layer for data access
       │
7. Repo layer
   └── Prisma queries with soft-delete filtering (deletedAt: null)
   └── Returns typed results
       │
8. ZodSerializerInterceptor
   └── Serializes response using @ZodResponse() schema
   └── Strips fields not defined in the response schema
       │
9. HttpExceptionFilter (on error)
   └── Catches exceptions, formats error response
   └── Logs ZodSerializationException errors
       │
10. HTTP Response sent
```

---

## 6. Database Schema (Prisma)

### 6.1 Schema Location

`prisma/schema.prisma` — PostgreSQL database with `prisma-client-js` generator and `prisma-json-types-generator` for typed JSON fields.

### 6.2 Common Patterns Across All Models

| Pattern | Description |
|---------|-------------|
| **Soft delete** | `deletedAt DateTime?` + `@@index([deletedAt])`. All queries filter `deletedAt: null` |
| **Audit trail** | `createdById`, `updatedById`, `deletedById` — tracks who performed each action |
| **Timestamps** | `createdAt DateTime @default(now())`, `updatedAt DateTime @updatedAt` |
| **Named relations** | Required when same model has multiple relations to another model. E.g., `@relation("ProductCreatedBy")`, `@relation("ProductUpdatedBy")` |
| **Translation pattern** | Entity → EntityTranslation (1-N per language). Used by Product, Category, Brand, User |
| **Implicit many-to-many** | Prisma auto-creates join tables for `Product ↔ Category`, `Role ↔ Permission` |

### 6.3 Entity Relationship Diagram

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
  ├──1:N──> Message (sent/received)
  └──1:N──> Websocket

Product ──1:N──> SKU
   │      ──1:N──> ProductTranslation ──N:1──> Language
   │      ──N:1──> Brand ──1:N──> BrandTranslation ──N:1──> Language
   │      ──N:M──> Category ──1:N──> CategoryTranslation ──N:1──> Language
   │                  └── Self-relation (parentCategoryId) for hierarchy
   └──N:M──> Order

VerificationCode (standalone - email OTP)
PaymentTransaction (standalone - payment gateway logs)
```

### 6.4 Enums

| Enum | Values |
|------|--------|
| `UserStatus` | `ACTIVE`, `INACTIVE`, `BLOCKED` |
| `OrderStatus` | `PENDING_PAYMENT`, `PENDING_PICKUP`, `PENDING_DELIVERY`, `DELIVERED`, `RETURNED`, `CANCELLED` |
| `PaymentStatus` | `PENDING`, `SUCCESS`, `FAILED` |
| `HTTPMethod` | `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `OPTIONS`, `HEAD` |
| `VerificationCodeType` | `REGISTER`, `FORGOT_PASSWORD`, `LOGIN`, `DISABLE_2FA` |
| `MediaType` | `IMAGE`, `VIDEO` |

---

## 7. Authentication & Authorization

### 7.1 Authentication Methods

| Method | Description |
|--------|-------------|
| **JWT Bearer Token** | Access token (short-lived) + Refresh token (long-lived, stored in DB) |
| **Google OAuth 2.0** | Social login via Google, redirects back with tokens |
| **Payment API Key** | Static API key for payment webhook endpoints |
| **Public (None)** | No authentication required |

### 7.2 Token Flow

```
Register → Send OTP email → Verify OTP → Account activated

Login (email + password)
   ├── If 2FA disabled → Return { accessToken, refreshToken }
   └── If 2FA enabled  → Require TOTP code → Return { accessToken, refreshToken }

Access Token expired → Call /auth/refresh-token with refreshToken
   └── Returns new { accessToken, refreshToken }

Logout → Invalidates refresh token in DB
```

### 7.3 RBAC (Role-Based Access Control)

- Each **User** has one **Role**
- Each **Role** has many **Permissions**
- Each **Permission** defines: `path` (API route pattern) + `method` (HTTP verb)
- `AccessTokenGuard` checks if the user's role has permission to access the requested endpoint
- Permissions are cached in Redis for 1 hour per role (`role:{roleId}`)

### 7.4 2FA (Two-Factor Authentication)

- Uses TOTP (Time-based One-Time Password)
- Setup: generates secret + QR code URI
- Login: requires TOTP code in addition to password
- Disable: requires OTP email verification

---

## 8. Validation & Serialization (Zod)

### 8.1 Input Validation

All validation uses **Zod** schemas (not class-validator). The pattern:

```typescript
// 1. Define Zod schema in feature.model.ts
export const CreateProductBodySchema = z.object({
  name: z.string().min(1).max(500),
  basePrice: z.number().positive(),
  // ...
})
export type CreateProductBodyType = z.infer<typeof CreateProductBodySchema>

// 2. Create DTO class in feature.dto.ts
export class CreateProductBodyDTO extends createZodDto(CreateProductBodySchema) {}

// 3. Use in controller
@Post()
create(@Body() body: CreateProductBodyDTO) { ... }
```

### 8.2 Response Serialization

Response shapes are also controlled by Zod schemas via `@ZodResponse()`:

```typescript
@Get()
@ZodResponse({ type: GetProductsResDTO })
list(@Query() query: GetProductsQueryDTO) {
  return this.productService.list({ query })
}
```

The `ZodSerializerInterceptor` strips any fields from the response that are not defined in the response schema.

---

## 9. Internationalization (i18n)

- **Library:** `nestjs-i18n` v10.5.1
- **Languages:** English (fallback) + Vietnamese
- **Translation files:** `src/i18n/en/error.json`, `src/i18n/vi/error.json`
- **Auto-generated types:** `src/generated/i18n.generated.ts`
- **Language resolution:** `?lang=vi` query parameter → `Accept-Language` header → fallback to `en`
- **Usage in errors:** Exception messages use i18n keys like `"Error.NotFound"`, `"Error.InvalidPassword"`

---

## 10. Real-Time Features (WebSockets)

### 10.1 Setup

- **Library:** Socket.io with `@socket.io/redis-adapter` for multi-instance support
- **Location:** `src/websockets/`
- **Adapter:** `WebsocketAdapter` extends NestJS `IoAdapter`, adds JWT authentication middleware on connection
- **Connection flow:** Client connects → JWT validated → user joins room `userId:{id}`

### 10.2 Gateways

| Gateway | File | Description |
|---------|------|-------------|
| `PaymentGateway` | `websockets/payment.gateway.ts` | Emits real-time payment status updates |
| `ChatGateway` | `websockets/chat.gateway.ts` | User-to-user messaging, message read receipts |

---

## 11. Background Jobs & Scheduled Tasks

### 11.1 BullMQ Job Queue

- **Library:** `@nestjs/bullmq` (Redis-backed)
- **Queue:** `PAYMENT_QUEUE` (defined in `shared/constants/queue.constant.ts`)
- **Consumer:** `PaymentConsumer` (`src/queues/payment.consumer.ts`) - Processes auto-cancel jobs for unpaid orders
- **Producers:** `order.producer.ts`, `payment.producer.ts` - Enqueue jobs with delay

### 11.2 Cron Jobs

| Job | File | Schedule | Description |
|-----|------|----------|-------------|
| `RemoveRefreshTokenCronjob` | `src/cronjobs/remove-refresh-token.cronjob.ts` | Daily at 1:00 AM | Deletes expired refresh tokens from database |

---

## 12. Configuration

### 12.1 Environment Variables

All environment variables are validated at startup using Zod in `src/shared/config.ts`. If any required variable is missing or invalid, the application exits immediately.

See `.env.example` for the full list. Key groups:

| Group | Variables |
|-------|-----------|
| **Database** | `DATABASE_URL` (PostgreSQL connection string) |
| **JWT** | `ACCESS_TOKEN_SECRET`, `ACCESS_TOKEN_EXPIRES_IN`, `REFRESH_TOKEN_SECRET`, `REFRESH_TOKEN_EXPIRES_IN` |
| **Admin seed** | `ADMIN_NAME`, `ADMIN_EMAIL`, `ADMIN_PASSWORD`, `ADMIN_PHONE_NUMBER` |
| **Email** | `RESEND_API_KEY`, `OTP_EXPIRES_IN` |
| **Google OAuth** | `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REDIRECT_URI`, `GOOGLE_CLIENT_REDIRECT_URI` |
| **AWS S3** | `S3_REGION`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_BUCKET_NAME`, `S3_ENPOINT` |
| **Redis** | `REDIS_URL` |
| **Payment** | `PAYMENT_API_KEY` |
| **App** | `APP_NAME`, `PREFIX_STATIC_ENPOINT` |

### 12.2 Rate Limiting

Configured in `app.module.ts` via `@nestjs/throttler`:

| Throttle | TTL | Limit |
|----------|-----|-------|
| `short` | 60 seconds | 5 requests |
| `long` | 120 seconds | 7 requests |

Controllers can skip throttling with `@SkipThrottle()`.

### 12.3 Logging

- **Library:** `nestjs-pino`
- **Output:** `logs/app.log` (async file logging)
- **Serialization:** Logs request method, URL, query, params + response status code

---

## 13. Global Providers (app.module.ts)

These are registered at the application level:

| Provider | Type | Description |
|----------|------|-------------|
| `CustomZodValidationPipe` | `APP_PIPE` | Global request validation |
| `ZodSerializerInterceptor` | `APP_INTERCEPTOR` | Global response serialization |
| `HttpExceptionFilter` | `APP_FILTER` | Global exception handling |
| `ThrottlerBehindProxyGuard` | `APP_GUARD` | Global rate limiting |
| `AuthenticationGuard` | `APP_GUARD` (via SharedModule) | Global authentication |
| `PaymentConsumer` | Provider | BullMQ worker for payment jobs |
| `RemoveRefreshTokenCronjob` | Provider | Scheduled token cleanup |

---

## 14. Key Libraries Reference

| Library | Version | Purpose |
|---------|---------|---------|
| `@nestjs/core` | 11.1.6 | NestJS framework |
| `@prisma/client` | 6.14.0 | PostgreSQL ORM |
| `zod` | 4.1.0 | Schema validation |
| `nestjs-zod` | 5.0.0 | NestJS + Zod integration (DTOs, serialization, Swagger) |
| `@nestjs/jwt` | 11.0.0 | JWT token handling |
| `@nestjs/bullmq` | 5.58.0 | Background job queues |
| `socket.io` | 4.8.1 | WebSocket server |
| `@socket.io/redis-adapter` | - | Multi-instance WebSocket support |
| `nestjs-i18n` | 10.5.1 | Internationalization |
| `@nestjs/throttler` | 6.4.0 | Rate limiting |
| `nestjs-pino` | 4.4.0 | Structured logging |
| `@nestjs/cache-manager` | - | Redis caching |
| `@aws-sdk/client-s3` | - | S3 file storage |
| `resend` | 6.0.1 | Transactional emails |
| `@react-email/components` | - | Email templates |
| `helmet` | 8.1.0 | HTTP security headers |
| `@nestjs/swagger` | - | API documentation at `/api` |

---

## 15. Common Development Commands

```bash
# Development
npm run start:dev              # Watch mode with hot reload
npm run start:debug            # Debug mode with watch

# Build
npm run build                  # Compile TypeScript

# Testing
npm run test                   # Unit tests (Jest)
npm run test:e2e               # End-to-end tests
npm run test:cov               # Coverage report
npx jest --testPathPattern=<pattern>  # Run single test

# Linting & Formatting
npm run lint                   # ESLint with auto-fix
npm run format                 # Prettier

# Database
npx prisma migrate dev         # Create/run migrations
npx prisma generate            # Regenerate Prisma client after schema changes
npm run init-seed-data         # Seed initial data
npm run create-permissions     # Initialize RBAC permissions

# Email Templates
npm run email:dev              # React Email local preview server
npm run email:build            # Build email templates
npm run email:export           # Export email templates
```
