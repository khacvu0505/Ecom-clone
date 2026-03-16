# Learning Guide — Understand This Ecom Codebase

> **Your level:** Know Node.js + basic TypeScript, new to NestJS/Prisma
>
> **Timeline:** 3 weeks (21 days), ~2-3 hours/day
>
> **How to use:** Follow day by day. Tick `[x]` when done. Each day has **Learn → Read → Practice → Self-check**.

---

## Progress Tracker

| Week | Day | Topic | Done |
|------|-----|-------|------|
| 1 | Day 1 (Mon) | TS Decorators + NestJS First Steps | [x] |
| 1 | Day 2 (Tue) | Providers, Services, DI | [x] |
| 1 | Day 3 (Wed) | Guards, Pipes, Interceptors | [x] |
| 1 | Day 4 (Thu) | Read Project Foundation | [x] |
| 1 | Day 5 (Fri) | Prisma Fundamentals | [ ] |
| 1 | Day 6 (Sat) | Read Ecom Database Schema | [ ] |
| 1 | Day 7 (Sun) | Zod Fundamentals + Week 1 Review | [ ] |
| 2 | Day 8 (Mon) | Auth Constants + Decorators + Guards P1 | [ ] |
| 2 | Day 9 (Tue) | Auth Guards P2 + JWT + Redis Cache | [ ] |
| 2 | Day 10 (Wed) | Auth Zod Schemas + DTOs + Errors | [ ] |
| 2 | Day 11 (Thu) | Auth Repository + Helpers | [ ] |
| 2 | Day 12 (Fri) | Auth Service Deep Dive | [ ] |
| 2 | Day 13 (Sat) | Auth Controller + Google OAuth + 2FA | [ ] |
| 2 | Day 14 (Sun) | Week 2 Review + Trace Full Requests | [ ] |
| 3 | Day 15 (Mon) | Brand Module (Simple CRUD) | [ ] |
| 3 | Day 16 (Tue) | Product Module (Complex CRUD) | [ ] |
| 3 | Day 17 (Wed) | Category + Cart + Review Modules | [ ] |
| 3 | Day 18 (Thu) | Order Module (Locks + Transactions) | [ ] |
| 3 | Day 19 (Fri) | Payment + BullMQ Job Queue | [ ] |
| 3 | Day 20 (Sat) | WebSocket + Cron + i18n + Remaining | [ ] |
| 3 | Day 21 (Sun) | Final Review + Build Challenge | [ ] |

---

# WEEK 1 — NestJS Fundamentals + Project Foundation

---

## Day 1 (Mon) — TypeScript Decorators + NestJS First Steps

### Prerequisites: NestJS vs Express

If you know Express, NestJS is Express + structure + decorators:

```
Express:                           NestJS:
─────────                          ──────
app.get('/users', handler)    →    @Controller('users') + @Get()
middleware(req, res, next)    →    @Injectable() Guard / Interceptor / Pipe
require('./routes/user')      →    @Module({ imports: [UserModule] })
req.body                      →    @Body() body: CreateUserDTO
```

### Learn:
- [x] Understand TypeScript decorators (`@something`) — what they are, how they work
- [x] Read NestJS docs: **"First Steps"** — create a basic NestJS app with `nest new`
- [x] Read NestJS docs: **"Controllers"** — `@Controller`, `@Get`, `@Post`, `@Body`, `@Param`, `@Query`

### Practice:
- [x] Create a tiny NestJS project: 1 controller with GET and POST endpoint
- [x] Test with Postman or curl

### Self-check:
- [x] Can you explain what `@Controller('users')` and `@Get(':id')` generate?
- [x] Can you explain what `@Body()` does compared to Express `req.body`?

---

## Day 2 (Tue) — Providers, Services, Dependency Injection

### Learn:
- [x] Read NestJS docs: **"Providers"** — `@Injectable()`, constructor injection
- [x] Read NestJS docs: **"Modules"** — `@Module({ imports, controllers, providers, exports })`
- [x] Understand: What is Dependency Injection (DI)? Why does NestJS use it?

### Practice:
- [x] Add a Service to your Day 1 project: move logic from controller → service
- [x] Create a second module, import it, and use its exported service

### Self-check:
- [x] What does `constructor(private readonly userService: UserService)` do?
- [x] What happens if you forget to add a service to `providers` in `@Module`?
- [x] What does `exports` do in a module?

---

## Day 3 (Wed) — Guards, Pipes, Interceptors

### Learn:
- [x] Read NestJS docs: **"Guards"** — `CanActivate`, `ExecutionContext`
- [x] Read NestJS docs: **"Pipes"** — validation and transformation
- [x] Read NestJS docs: **"Interceptors"** — before/after request processing
- [x] Understand the NestJS request lifecycle order: Middleware → Guard → Interceptor (before) → Pipe → Controller → Interceptor (after) → Filter (on error)

### Practice:
- [x] Create a simple AuthGuard that checks for a hardcoded API key in headers
- [x] Apply it to one route using `@UseGuards()`

### Self-check:
- [x] What's the difference between Guard, Pipe, and Interceptor?
- [x] What does `APP_GUARD` mean (vs `@UseGuards()` on a single route)?

### Key concept — 6 NestJS building blocks used in this project:

| # | Concept | What It Does | Express Equivalent |
|---|---------|-------------|-------------------|
| 1 | **Module** (`@Module`) | Groups related code. Every feature = 1 module | Separate router file |
| 2 | **Controller** (`@Controller`) | Handles HTTP routes. Uses decorators for GET/POST/etc | `router.get()`, `router.post()` |
| 3 | **Service** (`@Injectable`) | Business logic. Injected into controllers | Helper functions, but auto-wired |
| 4 | **Dependency Injection (DI)** | NestJS auto-creates and passes instances to constructors | `require()` but automatic |
| 5 | **Guard** (`@CanActivate`) | Runs BEFORE controller. Decides allow/deny | Auth middleware |
| 6 | **Pipe** | Transforms/validates input data BEFORE controller | `express-validator` |

---

## Day 4 (Thu) — Read the Ecom Project Foundation

**Now you start reading THIS project's actual code.**

### Read these files (in order):
- [ ] `src/main.ts` — How the app boots (Swagger, Helmet, CORS, WebSocket)
- [ ] `src/app.module.ts` — The root wiring diagram (all imports, global providers)
- [ ] `src/shared/shared.module.ts` — `@Global()` module, what services are shared
- [ ] `src/shared/config.ts` — Env validation with Zod at startup

### What's happening in `main.ts`:

```
1. NestFactory.create(AppModule)     → Creates the app from root module
2. app.useLogger(pino)               → Structured logging
3. app.enableCors()                  → Allow cross-origin requests
4. app.use(helmet())                 → Security headers
5. SwaggerModule.setup('api', ...)   → API docs available at /api
6. WebsocketAdapter                  → Socket.io with Redis
7. app.listen(3000)                  → Start server
```

### What's happening in `app.module.ts`:

```
@Module({
  imports: [
    // Infrastructure:
    LoggerModule          → logging to logs/app.log
    CacheModule           → Redis cache
    ScheduleModule        → cron jobs
    BullModule            → job queues (Redis)
    I18nModule            → multi-language (en/vi)
    ThrottlerModule       → rate limiting (5 req/min)

    // Core:
    SharedModule          → shared services (DB, auth, email, S3...)

    // Features (each = 1 business domain):
    AuthModule, UserModule, ProfileModule,
    RoleModule, PermissionModule, LanguageModule,
    BrandModule, CategoryModule, ProductModule,
    CartModule, OrderModule, PaymentModule, ReviewModule,
    + Translation modules...
  ],
  providers: [
    APP_PIPE        → CustomZodValidationPipe   (validates ALL request input)
    APP_INTERCEPTOR → ZodSerializerInterceptor  (shapes ALL responses)
    APP_FILTER      → HttpExceptionFilter       (catches ALL errors)
    APP_GUARD       → ThrottlerBehindProxyGuard (rate limits ALL requests)
    // + AuthenticationGuard registered inside SharedModule
  ]
})
```

### Key Insight — Every request goes through this pipeline:

```
Request → ThrottlerGuard → AuthGuard → ZodValidationPipe → Controller → ZodSerializerInterceptor → Response
                                                                              ↓ (on error)
                                                                    HttpExceptionFilter → Error Response
```

### What's happening in `shared.module.ts`:

```typescript
@Global()    // ← Available to ALL modules without importing
@Module({
  providers: [
    PrismaService,           // Database client
    HashingService,          // Password bcrypt
    TokenService,            // JWT sign/verify
    EmailService,            // Send emails (Resend API)
    S3Service,               // File upload (AWS S3)
    TwoFactorService,        // 2FA TOTP
    SharedUserRepository,    // Shared user queries
    SharedRoleRepository,    // Shared role queries
    AccessTokenGuard,        // JWT guard
    PaymentAPIKeyGuard,      // API key guard
    AuthenticationGuard,     // Main guard (delegates to above)
  ],
  exports: [/* all services above */]
})
```

### What's happening in `config.ts`:

```typescript
const configSchema = z.object({
  DATABASE_URL: z.string(),
  ACCESS_TOKEN_SECRET: z.string(),
  // ... 20+ required env vars
})
const configServer = configSchema.safeParse(process.env)
if (!configServer.success) {
  process.exit(1)  // ← App REFUSES to start if .env is invalid
}
```

**Pattern: Fail fast.** Don't let the app run with missing config and crash later at runtime.

### Understand:
- [ ] Draw/write the request pipeline: Request → ? → ? → ? → Response
- [ ] List the 5 global providers registered in `app.module.ts` and what each does
- [ ] Explain why `SharedModule` is `@Global()`

### Self-check:
- [ ] What happens if `.env` file is missing?
- [ ] Which guard runs on EVERY request? (ThrottlerBehindProxyGuard + AuthenticationGuard)
- [ ] What is `APP_PIPE` vs `APP_GUARD` vs `APP_FILTER` vs `APP_INTERCEPTOR`?

---

## Day 5 (Fri) — Prisma Fundamentals

### Prerequisites: Prisma vs Raw SQL

```sql
-- SQL:
SELECT * FROM users WHERE email = 'a@b.com' AND deleted_at IS NULL;

-- Prisma:
prisma.user.findFirst({
  where: { email: 'a@b.com', deletedAt: null }
})
```

```sql
-- SQL:
SELECT u.*, r.* FROM users u JOIN roles r ON u.role_id = r.id;

-- Prisma:
prisma.user.findFirst({
  where: { id: 1 },
  include: { role: true }   // ← auto-joins
})
```

### Learn:
- [ ] Read Prisma docs: **"Quickstart"** — setup Prisma with PostgreSQL
- [ ] Read Prisma docs: **"CRUD"** — `findMany`, `findUnique`, `create`, `update`, `delete`
- [ ] Read Prisma docs: **"Relations"** — `include`, 1-1, 1-N, N-M relations
- [ ] Read Prisma docs: **"Transactions"** — `$transaction`

### Practice:
- [ ] Setup Prisma in your test project with a simple `User` model
- [ ] Write: `create`, `findMany`, `findUnique`, `update`, `delete` queries

### Self-check:
- [ ] What does `include: { role: true }` do?
- [ ] What is the difference between `findUnique` and `findFirst`?
- [ ] What does `npx prisma migrate dev` do?

---

## Day 6 (Sat) — Read the Ecom Database Schema

### Read this project file carefully:
- [ ] `prisma/schema.prisma` — All 18 models (576 lines)

### Focus on understanding:
- [ ] **Soft delete pattern**: `deletedAt DateTime?` + `@@index([deletedAt])` on every model
- [ ] **Audit trail**: `createdById`, `updatedById`, `deletedById` fields
- [ ] **Translation pattern**: `Product` → `ProductTranslation` (1-N per language)
- [ ] **Self-relation**: `Category.parentCategoryId` → `Category` (hierarchy)
- [ ] **Named relations**: Why `@relation("ProductCreatedBy")` is needed
- [ ] **Implicit M-N**: `Product ↔ Category`, `Role ↔ Permission` (no join table model)
- [ ] **Enums**: `OrderStatus` (6 values), `PaymentStatus` (3), `UserStatus` (3)
- [ ] **JSON fields**: `Product.variants`, `Order.receiver` (with type comments `/// [Variants]`)

### Entity Relationship Overview:

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
  │   └──1:N──> Review ──1:N──> ReviewMedia
  │
  ├──1:N──> Message (sent/received)
  └──1:N──> Websocket

Product ──1:N──> SKU
   │      ──1:N──> ProductTranslation ──N:1──> Language
   │      ──N:1──> Brand ──1:N──> BrandTranslation
   │      ──N:M──> Category ──1:N──> CategoryTranslation
   │                  └── Self-relation (parentCategoryId)
   └──N:M──> Order
```

### Also read:
- [ ] `luu-y-ve-prisma.md` (in project root parent) — Author's Prisma notes (Vietnamese)

### Self-check:
- [ ] Draw the relationship: User → Role → Permission
- [ ] Draw the relationship: Product → SKU → CartItem → Order
- [ ] What is the translation pattern and why is it used?
- [ ] Why does `User` have so many `@relation("...")` fields?

---

## Day 7 (Sun) — Zod Fundamentals + Review Week 1

### Prerequisites: Zod vs Manual Validation

```typescript
// Manual (Express style):
if (!req.body.email || !req.body.email.includes('@')) {
  return res.status(400).json({ error: 'Invalid email' })
}

// Zod:
const schema = z.object({
  email: z.email(),
  password: z.string().min(6).max(100),
})
schema.parse(req.body) // throws if invalid, with detailed errors
```

### Key Zod methods used in this project:

```typescript
// Define schema
const UserSchema = z.object({ id: z.number(), email: z.email(), name: z.string() })

// Derive TypeScript type (single source of truth!)
type UserType = z.infer<typeof UserSchema>

// Compose schemas
UserSchema.pick({ email: true, name: true })       // only email + name
UserSchema.omit({ password: true })                 // everything except password
UserSchema.extend({ confirmPassword: z.string() })  // add new field
schema.strict()                                      // reject unknown fields

// Cross-field validation
schema.superRefine(({ password, confirmPassword }, ctx) => {
  if (password !== confirmPassword) {
    ctx.addIssue({ code: 'custom', message: '...', path: ['confirmPassword'] })
  }
})
```

### Learn Zod:
- [ ] Read Zod docs: **"Basic usage"** — `z.string()`, `z.number()`, `z.object()`
- [ ] Read Zod docs: **"Objects"** — `.pick()`, `.omit()`, `.extend()`, `.strict()`
- [ ] Read Zod docs: **"Type inference"** — `z.infer<typeof schema>`
- [ ] Read about `.superRefine()` — cross-field validation

### Read project files:
- [ ] `src/shared/models/shared-user.model.ts` — How UserSchema is defined and composed
- [ ] `src/shared/dtos/response.dto.ts` — `createZodDto()` pattern

### Week 1 Review — can you answer ALL of these?
- [ ] What is `@Module`, `@Controller`, `@Injectable`, `@Global`?
- [ ] What is the NestJS request lifecycle order?
- [ ] What are the 5 global providers in this project?
- [ ] What is Prisma `include`? What is soft delete?
- [ ] What does `z.infer<typeof Schema>` do?

---

# WEEK 2 — Auth Module Deep Dive

**Why Auth first?** It touches ALL the patterns: guards, decorators, Zod DTOs, services, repos, error handling, JWT, Redis caching. Understand Auth = understand 80% of this codebase.

---

## Day 8 (Mon) — Auth Constants, Decorators, Guards (Part 1)

### Read these files (in order):
- [ ] `src/shared/constants/auth.constant.ts` — `AuthType`, `ConditionGuard`, `as const` pattern
- [ ] `src/shared/decorators/auth.decorator.ts` — `@Auth()`, `@IsPublic()`, `SetMetadata`
- [ ] `src/shared/guards/authentication.guard.ts` — Strategy pattern: routes to correct guard

### What you learn in each file:

**auth.constant.ts** — `as const` pattern:
```typescript
export const AuthType = {
  Bearer: 'Bearer',
  None: 'None',
  PaymentAPIKey: 'PaymentAPIKey',
} as const
// Creates readonly object with literal types. Used instead of TypeScript enum.
```

**auth.decorator.ts** — Custom decorators:
```typescript
export const Auth = (authTypes: AuthTypeType[]) => {
  return SetMetadata(AUTH_TYPE_KEY, { authTypes, ... })
}
export const IsPublic = () => Auth([AuthType.None])
// "Tags" a controller/method with metadata that guards read later.
```

**authentication.guard.ts** — Strategy Pattern:
```
This guard does NOT authenticate itself. It reads the @Auth() metadata
and DELEGATES to the correct guard:

@Auth([AuthType.Bearer])        → AccessTokenGuard (JWT)
@Auth([AuthType.PaymentAPIKey]) → PaymentAPIKeyGuard
@Auth([AuthType.None])          → always pass (public route)

If no @Auth() decorator → defaults to Bearer (all routes require JWT by default!)
```

### Understand:
- [ ] What is `as const` and why use it instead of `enum`?
- [ ] What does `SetMetadata(AUTH_TYPE_KEY, payload)` do?
- [ ] How does `AuthenticationGuard` decide which guard to use?
- [ ] What is the default auth type when no `@Auth()` decorator exists? (Bearer)
- [ ] What is AND vs OR condition in guard logic?

---

## Day 9 (Tue) — Auth Guards (Part 2) + JWT

### Read these files:
- [ ] `src/shared/guards/access-token.guard.ts` — JWT verify + RBAC + Redis cache
- [ ] `src/shared/guards/payment-api-key.guard.ts` — Simple API key check
- [ ] `src/shared/services/token.service.ts` — JWT sign/verify with HS256
- [ ] `src/shared/types/jwt.type.ts` — Token payload types
- [ ] `src/shared/decorators/active-user.decorator.ts` — `@ActiveUser()` param decorator

### What you learn:

**access-token.guard.ts** — JWT + RBAC + Redis caching:
```
1. Extract "Bearer xxx" from Authorization header
2. Verify JWT with tokenService.verifyAccessToken()
3. Attach decoded user to request[REQUEST_USER_KEY]
4. Check permissions:
   a. Redis cache hit (key: "role:{roleId}")? Use cached permissions
   b. Cache miss? Query DB: role → permissions, cache for 1 hour
   c. Check: does role have permission for "path:method"?
   d. No → 403 Forbidden
```

**token.service.ts** — Simple JWT wrapper. **Separate secrets** for access vs refresh tokens, UUID in payload for uniqueness.

**active-user.decorator.ts** — Custom param decorator:
```typescript
export const ActiveUser = createParamDecorator(
  (field: keyof AccessTokenPayload | undefined, context: ExecutionContext) => {
    const request = context.switchToHttp().getRequest()
    const user = request[REQUEST_USER_KEY]  // set by AccessTokenGuard
    return field ? user?.[field] : user
  },
)
// Usage: @ActiveUser('userId') userId: number
```

### Understand:
- [ ] Trace the full flow: HTTP header → extract token → verify → attach to request → check permission
- [ ] How are permissions cached in Redis? What is the key format? (`role:{roleId}`)
- [ ] What does `keyBy(permissions, "path:method")` do? Why?
- [ ] How does `@ActiveUser('userId')` get the userId from the request?

---

## Day 10 (Wed) — Auth Zod Schemas + DTOs + Errors

### Read these files:
- [ ] `src/routes/auth/auth.model.ts` — All Zod schemas for auth (RegisterBody, LoginBody, etc.)
- [ ] `src/routes/auth/auth.dto.ts` — `createZodDto()` wrappers
- [ ] `src/routes/auth/auth.error.ts` — Pre-instantiated exceptions with i18n keys

### What you learn:

**auth.model.ts** — How Zod schemas build DTOs:
```typescript
// Build RegisterBody from UserSchema by picking fields + adding new ones
const RegisterBodySchema = UserSchema.pick({ email: true, password: true, name: true, phoneNumber: true })
  .extend({ confirmPassword: z.string(), code: z.string().length(6) })
  .strict()
  .superRefine(/* password match check */)

// Build response schema by removing sensitive fields
const RegisterResSchema = UserSchema.omit({ password: true, totpSecret: true })

// Types are DERIVED from schemas — never manually written
type RegisterBodyType = z.infer<typeof RegisterBodySchema>
```

**auth.dto.ts** — Bridges Zod → NestJS:
```typescript
export class RegisterBodyDTO extends createZodDto(RegisterBodySchema) {}
// Now NestJS pipes can validate request body against this schema
```

**auth.error.ts** — Pre-instantiated exceptions:
```typescript
export const EmailAlreadyExistsException = new UnprocessableEntityException([
  { message: 'Error.EmailAlreadyExists', path: 'email' },
])
// Usage: throw EmailAlreadyExistsException
// message uses i18n key, not hardcoded text
```

### Understand:
- [ ] How `RegisterBodySchema` is built from `UserSchema.pick().extend().strict().superRefine()`
- [ ] Why `LoginBodySchema` has `.superRefine()` that prevents sending both `totpCode` AND `code`
- [ ] Pattern: Schema (model.ts) → DTO (dto.ts) → Controller uses DTO
- [ ] Pattern: Pre-instantiated errors with `{ message: 'Error.Key', path: 'field' }`
- [ ] Why types are derived with `z.infer<>` and never written manually

### Self-check:
- [ ] Given a new feature, can you write a Zod schema → DTO → error file?

---

## Day 11 (Thu) — Auth Repository + Helpers

### Read these files:
- [ ] `src/routes/auth/auth.repo.ts` — All Prisma queries for auth
- [ ] `src/shared/repositories/shared-user.repo.ts` — Shared user queries
- [ ] `src/shared/constants/serialize.decorator.ts` — How `@SerializeAll()` works
- [ ] `src/shared/helpers.ts` — Utility functions (Prisma error checks, OTP generation)

### What you learn:

**auth.repo.ts** — Repository = pure data access, no business logic:
```typescript
@Injectable()
@SerializeAll()  // ← auto JSON.parse(JSON.stringify()) all returns (strips Prisma internals)
export class AuthRepository {
  constructor(private readonly prismaService: PrismaService) {}

  createUser(user: Pick<UserType, 'email' | 'name' | ...>) {
    return this.prismaService.user.create({ data: user, omit: { password: true } })
  }
}
```

**helpers.ts** — Type Predicate pattern:
```typescript
function isUniqueConstraintPrismaError(error: any): error is PrismaClientKnownRequestError {
  return error instanceof PrismaClientKnownRequestError && error.code === 'P2002'
}
// P2002 = unique constraint violation, P2025 = not found
```

### Understand:
- [ ] Repository = ONLY Prisma queries, no business logic
- [ ] `@SerializeAll()` wraps every method with `JSON.parse(JSON.stringify(result))` — why?
- [ ] `Pick<UserType, 'email' | 'name'>` — restricts function input to specific fields
- [ ] `isUniqueConstraintPrismaError` — Type Predicate pattern (`error is PrismaError`)
- [ ] `upsert` in `createVerificationCode` — create if not exists, update if exists

---

## Day 12 (Fri) — Auth Service Deep Dive

### Read carefully:
- [ ] `src/routes/auth/auth.service.ts` — ALL business logic (375 lines)

### Trace each flow:
- [ ] `register()` — OTP validation → hash password → create user → handle duplicate email
- [ ] `sendOTP()` — Check user exists/not → generate OTP → save to DB → send email
- [ ] `login()` — Find user → check password → check 2FA → create device → generate tokens
- [ ] `generateTokens()` — Sign access + refresh JWT → save refresh to DB
- [ ] `refreshToken()` — Verify → check DB → delete old → create new (token rotation)
- [ ] `logout()` — Verify → delete refresh token → deactivate device
- [ ] `forgotPassword()` — Find user → validate OTP → hash new password → update
- [ ] `setupTwoFactorAuth()` — Check not already enabled → generate TOTP secret → save
- [ ] `disableTwoFactorAuth()` — Verify TOTP or OTP → set secret to null

### Key concept — Refresh Token Rotation:
```
Client sends refreshToken → verify JWT → check exists in DB → delete old → create new pair
If token not in DB → it was already used → possible theft → throw error
```
This is a security best practice: each refresh token is single-use.

### Self-check:
- [ ] Why does `refreshToken()` throw `RefreshTokenAlreadyUsedException` when token not in DB?
- [ ] Why does `register()` use `Promise.all` for createUser + deleteOTP?
- [ ] Why does `login()` create a Device record?

---

## Day 13 (Sat) — Auth Controller + Google OAuth + 2FA

### Read these files:
- [ ] `src/routes/auth/auth.controller.ts` — HTTP endpoints
- [ ] `src/routes/auth/google.service.ts` — Google OAuth flow
- [ ] `src/shared/services/2fa.service.ts` — TOTP generation/verification
- [ ] `src/shared/services/hashing.service.ts` — bcrypt hash/compare
- [ ] `src/shared/services/email.service.ts` — Send emails via Resend

### What you learn:

**auth.controller.ts** — Controller = decorators only, ZERO logic:
```typescript
@Controller('auth')
export class AuthController {
  @Post('register')
  @IsPublic()                              // no auth needed
  @ZodResponse({ type: RegisterResDTO })   // shape response
  register(@Body() body: RegisterBodyDTO) { // validate input
    return this.authService.register(body)  // delegate to service
  }
}
```
Every controller in this project follows this pattern.

### Understand:
- [ ] Controller has ZERO logic — only decorators + delegate to service
- [ ] `@IsPublic()` on register, login, sendOTP (no auth needed)
- [ ] `@ZodResponse({ type: DTO })` controls response shape
- [ ] Google OAuth flow: get URL → redirect → callback → create/find user → return tokens

---

## Day 14 (Sun) — Review Week 2 + Trace Full Requests

### Exercise 1: Trace a complete POST `/auth/login` request:
- [ ] Step 1: ThrottlerBehindProxyGuard checks rate limit
- [ ] Step 2: AuthenticationGuard reads `@IsPublic()` → AuthType.None → skip auth
- [ ] Step 3: CustomZodValidationPipe validates body with LoginBodyDTO (Zod schema)
- [ ] Step 4: Controller calls `authService.login(body)`
- [ ] Step 5: Service finds user, checks password, checks 2FA, creates device, generates tokens
- [ ] Step 6: ZodSerializerInterceptor shapes response with LoginResDTO
- [ ] Step 7: Response sent: `{ accessToken, refreshToken }`

### Exercise 2: Trace a GET `/profile` request (requires auth):
- [ ] Step 1: ThrottlerGuard
- [ ] Step 2: AuthenticationGuard → default Bearer → AccessTokenGuard
- [ ] Step 3: AccessTokenGuard extracts JWT → verifies → checks permissions (Redis cache) → attaches user
- [ ] Step 4: Controller uses `@ActiveUser('userId')` to get userId
- [ ] Step 5: Service calls repo → Prisma query with `deletedAt: null`
- [ ] Step 6: Response serialized

### Week 2 Review — can you answer ALL of these?
- [ ] How does `@Auth([AuthType.Bearer])` work end-to-end?
- [ ] What is refresh token rotation? Why is it secure?
- [ ] How are role permissions cached and checked?
- [ ] What is the pattern: `.model.ts` → `.dto.ts` → `.error.ts` → `.repo.ts` → `.service.ts` → `.controller.ts`?

---

# WEEK 3 — CRUD Patterns + Advanced Topics

---

## Day 15 (Mon) — Brand Module (Simplest CRUD)

### Read ALL files in `src/routes/brand/`:
- [ ] `brand.module.ts` — Module definition
- [ ] `brand.model.ts` — Zod schemas
- [ ] `brand.dto.ts` — DTOs
- [ ] `brand.controller.ts` — CRUD endpoints
- [ ] `brand.service.ts` — Business logic (thin)
- [ ] `brand.repo.ts` — Prisma queries

### Compare with Auth:
- [ ] Same patterns: `@Auth`, `@ZodResponse`, `@ActiveUser`, `@SerializeAll`
- [ ] Simpler: no OTP, no JWT, no 2FA — just CRUD
- [ ] Notice: soft delete in repo (`deletedAt: null` everywhere)

### Also read Brand Translation sub-module:
- [ ] `src/routes/brand/brand-translation/` — All files

### Self-check:
- [ ] Can you describe the complete brand CRUD flow (create, read, update, delete)?

---

## Day 16 (Tue) — Product Module (Complex CRUD)

### Read these files:
- [ ] `src/routes/product/product.module.ts` — Two controllers in one module
- [ ] `src/routes/product/product.controller.ts` — Public: `@IsPublic()` + `@SkipThrottle()`
- [ ] `src/routes/product/manage-product.controller.ts` — Admin: requires Bearer auth
- [ ] `src/routes/product/product.service.ts` + `manage-product.service.ts`
- [ ] `src/routes/product/product.repo.ts` — Complex Prisma queries
- [ ] `src/routes/product/product.model.ts` + `product.dto.ts`
- [ ] `src/routes/product/sku.model.ts` — SKU Zod schema

### Focus on understanding:
- [ ] Why 2 controllers? (Public product listing vs Admin management)
- [ ] Complex filtering in `product.repo.ts`: name search, brand filter, category filter, price range
- [ ] `Promise.all([count, findMany])` for pagination
- [ ] SKU update logic: compare existing vs payload → delete/update/create
- [ ] Soft delete cascade: deleting product also soft-deletes translations + SKUs

---

## Day 17 (Wed) — Category, Cart, Review Modules

### Skim these modules (same patterns, quick read):
- [ ] `src/routes/category/` — CRUD + hierarchy (parentCategoryId)
- [ ] `src/routes/category/category-translation/` — Translation sub-module
- [ ] `src/routes/cart/` — Add/remove/update cart items (userId + skuId unique)
- [ ] `src/routes/review/` — Product reviews with media attachments

### Focus on what's NEW (not repeated):
- [ ] Category: self-relation for parent/child hierarchy
- [ ] Cart: `@@unique([userId, skuId])` constraint in Prisma
- [ ] Review: `@@unique([orderId, productId])` — one review per product per order

---

## Day 18 (Thu) — Order Module (Most Complex)

### Read carefully:
- [ ] `src/routes/order/order.repo.ts` — The entire file, especially `create()` (lines 69-260)
- [ ] `src/routes/order/order.producer.ts` — BullMQ job scheduling
- [ ] `src/routes/order/order.error.ts` — Order-specific errors
- [ ] `src/shared/redis.ts` — Redlock setup
- [ ] `src/shared/error.ts` — `VersionConflictException`

### Understand the order creation flow:
- [ ] Step 1: Get all cart item SKU IDs
- [ ] Step 2: Acquire Redis locks for all SKUs (`lock:sku:{id}`, 3 seconds)
- [ ] Step 3: Inside `$transaction`:
  - [ ] Validate cart items exist
  - [ ] Check stock >= quantity
  - [ ] Check products not deleted/hidden
  - [ ] Validate SKUs belong to correct shop
  - [ ] Create Payment (PENDING)
  - [ ] Create Orders with ProductSKUSnapshots
  - [ ] Delete cart items
  - [ ] Decrement SKU stock (with `updatedAt` optimistic check)
  - [ ] Schedule auto-cancel job via BullMQ
- [ ] Step 4: `finally` block releases all Redis locks

### Key concept — Why 3 layers of safety?
```
Redis Lock      → Prevents race condition across multiple server instances
$transaction    → Ensures ALL database changes succeed or ALL rollback
updatedAt check → Detects if another process modified the SKU concurrently
```

### Self-check:
- [ ] Why Redis lock AND Prisma transaction? (Lock = cross-instance, Transaction = DB atomicity)
- [ ] What happens if 2 users buy the last item at the same time?
- [ ] What does `updatedAt` check prevent? (Concurrent modification)
- [ ] Why use `ProductSKUSnapshot` instead of just referencing the product? (Price/name may change later)

---

## Day 19 (Fri) — Payment + BullMQ Job Queue

### Read these files:
- [ ] `src/routes/payment/payment.controller.ts` — Payment endpoints
- [ ] `src/routes/payment/payment.service.ts` — Payment logic
- [ ] `src/routes/payment/payment.repo.ts` — Payment queries
- [ ] `src/routes/payment/payment.producer.ts` — Schedule cancel job
- [ ] `src/queues/payment.consumer.ts` — Process cancel job
- [ ] `src/shared/constants/queue.constant.ts` — Queue/job name constants
- [ ] `src/shared/repositories/shared-payment.repo.ts` — `cancelPaymentAndOrder()`

### Understand BullMQ pattern:

```typescript
// Producer (payment.producer.ts):
queue.add(jobName, { paymentId }, { delay: 30min, jobId: `paymentId-${id}` })

// Consumer (payment.consumer.ts):
@Processor(PAYMENT_QUEUE_NAME)
export class PaymentConsumer extends WorkerHost {
  async process(job: Job) {
    switch (job.name) {
      case CANCEL_PAYMENT_JOB_NAME:
        await this.sharedPaymentRepo.cancelPaymentAndOrder(job.data.paymentId)
    }
  }
}

// Flow: Order created → Producer adds "cancel" job with delay → If not paid, Consumer cancels
```

### Also understand `@Auth` with API key:
- [ ] Payment webhook uses `@Auth([AuthType.PaymentAPIKey])` — not JWT but API key
- [ ] `PaymentAPIKeyGuard` checks `authorization` header vs env `PAYMENT_API_KEY`

---

## Day 20 (Sat) — WebSocket + Cron + i18n + Remaining

### WebSocket (real-time):
- [ ] `src/websockets/websocket.adapter.ts` — Custom IoAdapter + Redis + JWT auth
- [ ] `src/websockets/websocket.module.ts` — Module setup
- [ ] `src/websockets/payment.gateway.ts` — Payment status events
- [ ] `src/websockets/chat.gateway.ts` — Chat messaging

### Key concept — WebSocket auth:
```
1. Client connects with Authorization header containing JWT
2. Adapter middleware verifies token with tokenService
3. User joins room "userId-{id}"
4. Gateways emit events to specific user rooms
5. Redis adapter enables pub/sub across multiple server instances
```

### Cron Jobs:
- [ ] `src/cronjobs/remove-refresh-token.cronjob.ts` — Daily cleanup at 1 AM

### i18n (internationalization):
- [ ] `src/i18n/en/error.json` — English error messages
- [ ] `src/i18n/vi/error.json` — Vietnamese error messages
- [ ] `src/generated/i18n.generated.ts` — Auto-generated types

### Remaining modules (skim — same patterns):
- [ ] `src/routes/user/` — User CRUD (admin)
- [ ] `src/routes/profile/` — Current user profile
- [ ] `src/routes/role/` — Role management
- [ ] `src/routes/permission/` — Permission management
- [ ] `src/routes/language/` — Language management
- [ ] `src/routes/media/` — S3 file upload

---

## Day 21 (Sun) — Final Review + Build Something

### Exercise: Trace these requests end-to-end:
- [ ] `POST /auth/register` (public, with OTP)
- [ ] `POST /auth/login` (public, with optional 2FA)
- [ ] `GET /products` (public, no auth, skip throttle)
- [ ] `POST /manage/products` (admin, Bearer auth, permission check)
- [ ] `POST /orders` (Bearer auth, locks + transaction)
- [ ] `POST /payment/webhook` (API key auth)

### Final self-check — answer ALL of these:
- [ ] Describe the module pattern: `.module` → `.controller` → `.service` → `.repo` → `.model` → `.dto` → `.error`
- [ ] What are the 3 auth types and how does `AuthenticationGuard` route them?
- [ ] What is `@SerializeAll()` and why do repos use it?
- [ ] What is refresh token rotation?
- [ ] How does RBAC permission checking + Redis caching work?
- [ ] Why does order creation use Redis lock + Prisma transaction + optimistic concurrency?
- [ ] How does the WebSocket adapter authenticate users?
- [ ] How does BullMQ producer → consumer pattern work?

### Challenge (optional):
- [ ] Try creating a new module (e.g., "Coupon") following the project pattern:
  - `coupon.module.ts`, `coupon.controller.ts`, `coupon.service.ts`, `coupon.repo.ts`
  - `coupon.model.ts` (Zod), `coupon.dto.ts`, `coupon.error.ts`
  - Add Prisma model in `schema.prisma`

---

## Summary: What This Project Teaches You

| Skill | Where You Learn It |
|-------|-------------------|
| NestJS module architecture | `app.module.ts` + any feature module |
| Dependency Injection | Every `constructor(private readonly ...)` |
| Guard-based authentication | `shared/guards/` |
| Custom decorators | `shared/decorators/` |
| JWT access + refresh token | `auth.service.ts` + `token.service.ts` |
| RBAC (role-based access control) | `access-token.guard.ts` |
| Zod validation (not class-validator) | Every `*.model.ts` + `*.dto.ts` |
| Prisma ORM (relations, transactions) | Every `*.repo.ts` |
| Soft delete pattern | Every `*.repo.ts` (`deletedAt: null`) |
| Redis caching | `access-token.guard.ts` (permission cache) |
| BullMQ job queues | `queues/` + `*.producer.ts` |
| Socket.io + Redis adapter | `websockets/` |
| Cron jobs | `cronjobs/` |
| i18n (multi-language errors) | `i18n/` + `*.error.ts` |
| React email templates | `emails/` + `shared/email-templates/` |
| S3 file upload | `shared/services/s3.service.ts` |
| Google OAuth | `routes/auth/google.service.ts` |
| 2FA TOTP | `shared/services/2fa.service.ts` |
| Distributed locking (Redlock) | `order.repo.ts` |
| Optimistic concurrency | `order.repo.ts` (updatedAt check) |
| Rate limiting | `app.module.ts` (ThrottlerModule) |
| Swagger auto-docs | `main.ts` |


