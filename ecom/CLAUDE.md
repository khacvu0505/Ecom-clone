# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

NestJS 11 e-commerce backend API using Prisma (PostgreSQL), Redis, BullMQ, Socket.io, and Zod validation. Supports JWT + Google OAuth authentication, RBAC, i18n (EN/VI), real-time WebSockets, background job queues, and S3 file storage.

## Common Commands

```bash
# Development
npm run start:dev          # Watch mode with hot reload
npm run start:debug        # Debug mode with watch

# Build
npm run build              # Compile TypeScript

# Testing
npm run test               # Unit tests (Jest)
npm run test:e2e           # End-to-end tests
npm run test:cov           # Coverage report
# Run single test:
npx jest --testPathPattern=<pattern>

# Linting & Formatting
npm run lint               # ESLint with auto-fix
npm run format             # Prettier

# Database
npx prisma migrate dev     # Run/create migrations
npx prisma generate        # Regenerate Prisma client
npm run init-seed-data     # Seed database
npm run create-permissions # Initialize RBAC permissions

# Email Templates (React Email)
npm run email:dev          # Local preview server
npm run email:build        # Build templates
npm run email:export       # Export templates
```

## Architecture

### Module Pattern

Every feature module in `src/routes/` follows this structure:

```
feature/
├── feature.module.ts      # NestJS module
├── feature.controller.ts  # HTTP endpoints with decorators
├── feature.service.ts     # Business logic
├── feature.repo.ts        # Prisma queries (data access layer)
├── feature.dto.ts         # Zod schemas wrapped with createZodDto()
├── feature.model.ts       # Response models & Zod schemas
├── feature.error.ts       # Feature-specific exceptions
└── feature.producer.ts    # BullMQ job producer (optional)
```

**Key:** Business logic lives in `.service.ts`, database queries in `.repo.ts`. Controllers only handle HTTP concerns (decorators, auth, routing).

### Shared Module (`src/shared/`)

`SharedModule` is `@Global()` and exports core services used across all features:

- **Services:** `PrismaService`, `TokenService`, `HashingService`, `EmailService`, `S3Service`, `TwoFactorService`
- **Repositories:** `SharedUserRepository`, `SharedRoleRepository` — common queries reused by multiple feature modules
- **Guards:** `AuthenticationGuard` (entry point) → delegates to `AccessTokenGuard` or `PaymentAPIKeyGuard`
- **Pipes:** `CustomZodValidationPipe` — global Zod validation
- **Filters:** `HttpExceptionFilter` — catches all HTTP exceptions, formats with i18n
- **Interceptors:** `ZodSerializerInterceptor` — response serialization

### Authentication & Authorization

- JWT access + refresh token pattern. Tokens signed with HS256 via `TokenService`.
- `@Auth([AuthType.Bearer])` decorator on controllers sets auth type. `@IsPublic()` skips auth.
- `@ActiveUser()` parameter decorator injects current user from request.
- `AccessTokenGuard` validates JWT and caches role permissions in Redis (1-hour TTL, key: `role:{roleId}`).
- Google OAuth flow and TOTP-based 2FA are supported.

### Validation & DTOs

All validation uses **Zod + nestjs-zod**. DTOs extend `createZodDto(schema)`. Error messages reference i18n keys (e.g., `"Error.InvalidPassword"`).

### Environment Configuration

`src/shared/config.ts` validates all env vars with Zod at startup. See `.env.example` for required variables. Key groups: database, JWT secrets, Redis, S3, Google OAuth, Resend email, admin seed credentials.

### Real-Time & Background Jobs

- **WebSockets:** Socket.io with Redis adapter (`src/websockets/`). JWT auth on connection. Gateways for payment status and chat.
- **BullMQ:** Payment queue for auto-cancelling unpaid orders (`src/queues/`).
- **Cron:** Daily expired refresh token cleanup at 1 AM (`src/cronjobs/`).

### Database

Prisma ORM with PostgreSQL. Schema at `prisma/schema.prisma`. Key entities: User, Role, Permission, Product, SKU, Order, Payment, CartItem, Review. Products support multi-language translations, JSON variants, and hierarchical categories. Soft deletes on User via `deletedAt` field.

### Internationalization

`nestjs-i18n` with English (fallback) and Vietnamese. Translation files in `src/i18n/{lang}/error.json`. Auto-generated types in `src/generated/i18n.generated.ts`. Language resolved from `?lang=` query param or `Accept-Language` header.

## Code Conventions

- **Validation:** Always use Zod schemas via `createZodDto()` — no class-validator
- **Errors:** Throw custom exceptions from `*.error.ts` files; use i18n keys for messages
- **Formatting:** Single quotes, trailing commas, 120 char line width, 2-space indent
- **TypeScript:** Strict mode enabled with decorators and metadata reflection
- **Prisma:** Run `npx prisma generate` after schema changes; migrations via `npx prisma migrate dev`
