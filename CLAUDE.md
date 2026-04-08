# CLAUDE.md — Matcha Project

Matcha ("Strawberry Matcha") is a full-stack dating/social-matching web application built with Next.js 15, React 19, and PostgreSQL. This document describes the codebase for AI assistants.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 15.5 (App Router) |
| UI | React 19, Tailwind CSS 4, Radix UI / shadcn/ui |
| Database | PostgreSQL (raw `pg` client, no ORM) |
| File Storage | MinIO (S3-compatible) |
| Auth | JWT (jose), bcrypt |
| Email | Nodemailer |
| Migrations | node-pg-migrate |
| Testing | Jest 30 + ts-jest |
| Language | TypeScript 5.9 (strict mode) |
| Runtime | Node.js 22 Alpine |

---

## Directory Structure

```
src/
  app/
    api/
      auth/
        register/      # POST — user signup
        signin/        # POST — login, sets session cookie
        logout/        # POST — clears session
        verify/        # POST — email token verification
        reset-password/ # POST — password reset flow
      users/profiles/[id]/  # GET/POST — profile read/update
    home/              # Protected dashboard page
    login/
    register/
    onboarding/        # Profile creation wizard
    reset-password/
    layout.tsx
    page.tsx           # Landing page
    globals.css

  server/              # All backend logic (never imported by client)
    services/
      auth.ts          # AuthService — registration, login, verification, reset
      user.ts          # UserService — profile CRUD, picture management
    repositories/
      base.ts          # Abstract base repository
      user-repository.ts  # SQL queries for users and user_profiles
    schemas/
      auth.ts          # Zod-like validation schemas for auth inputs
      user.ts          # Validation schemas for user/profile inputs
    db/
      postgres.ts      # pg connection pool (max 20 connections)
      base.ts          # IDatabase interface
      exceptions.ts    # DB-specific error classes
    storage/
      minio.ts         # MinIO implementation
      base.ts          # IStorage interface
      utils/path.ts    # Storage path helpers
    factories/
      auth-factory.ts      # createAuthService()
      user-factory.ts      # createUserService()
      storage-factory.ts   # createStorageService()
    config.ts          # Server-side env constants
    types.ts           # Shared TypeScript types

  lib/                 # Isomorphic utilities
    auth/session.ts    # JWT encrypt/decrypt (HTTP-only cookie sessions)
    exception-http-mapper/  # Decorator-based error → HTTP status mapping
    mailer/Mailer.ts   # Nodemailer wrapper
    validator/         # Custom validation engine + exceptions + file parsers
    utils.ts           # General helpers

  components/
    ui/                # shadcn/ui base components
    login-form.tsx
    register-form.tsx
    reset-password-form.tsx

  middlewares/
    withAuthorization.ts   # JWT auth guard for API routes
    core.ts                # Middleware pipeline composer
    routes-middlewares/withErrorHandler.ts

migrations/            # node-pg-migrate files (timestamp-prefixed)
tests/                 # Jest unit tests
scripts/               # DB seed scripts
public/                # Static SVG assets
```

---

## Database Schema

Two main tables (managed via migrations):

**`users`**
- `id` UUID PK
- `username` TEXT UNIQUE
- `email` TEXT UNIQUE
- `first_name`, `last_name`, `password` TEXT
- `email_verified` BOOLEAN
- `email_verification_token`, `password_reset_token` TEXT
- `created_at`, `updated_at` TIMESTAMPTZ

**`user_profiles`**
- `id` UUID PK
- `user_id` UUID → `users.id` CASCADE DELETE
- `gender` `gender_t` enum (male | female)
- `sexual_preference` `sexual_preference_t` enum (male | female | both)
- `bio` TEXT
- `interests` TEXT[]
- `pictures` TEXT[]
- `avatar_url` TEXT
- `fame_rating` INTEGER
- `created_at`, `updated_at` TIMESTAMPTZ

---

## Development Commands

```bash
npm run dev          # Start dev server with Turbopack (localhost:3000)
npm run build        # Production build
npm run start        # Start production server
npm run lint         # ESLint check
npm run lint:fix     # Auto-fix lint issues
npm run format       # Prettier format
npm test             # Run Jest tests

npm run db:create    # Create new migration file (timestamp-prefixed)
npm run db:migrate   # Run pending migrations against DATABASE_URL
```

---

## Docker / Local Setup

```bash
docker compose up           # Start app + PostgreSQL + MinIO + Neo4j
npm run db:migrate          # Apply migrations
# Optional: load seed data
bash scripts/populate_users.sh
```

The `compose.yaml` wires up:
- `app` — Next.js on port 3000
- `postgresDb` — PostgreSQL on port 5432
- `minio` — MinIO on ports 9000 (API) / 9001 (console)
- `neo4jDb` — Neo4j on ports 7474/7687 (configured but currently unused)

---

## Environment Variables

Copy `.env.example` → `.env`. Key variables:

```
# Database
DATABASE_URL=postgresql://matcha:matcha@localhost:5432/matcha_db?sslmode=disable

# Session
SESSION_SECRET=<base64 key>

# Auth toggle (set false to skip auth in dev)
ENABLE_AUTH=true

# Email (Nodemailer)
APP_SERVICE=gmail
APP_EMAIL=you@gmail.com
APP_PASSWORD=<app-password>

# Storage
STORAGE_TYPE=minio
STORAGE_BUCKET_NAME=matcha
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin
MINIO_ENDPOINT=minio
MINIO_PORT=9000

# Public URLs
NEXT_PUBLIC_API_URL=http://localhost:3000/api
NEXT_PUBLIC_CLIENT_URL=http://localhost:3000
```

---

## Architecture Patterns

### Repository → Service → API Route
Data access lives exclusively in `src/server/repositories/`. Business logic lives in `src/server/services/`. API route handlers in `src/app/api/` call services via factories and never touch repositories directly.

### Factories
`src/server/factories/` instantiate services with their dependencies. Always use factories when calling services from API routes — do not instantiate services inline.

### Error Handling
Custom exceptions are defined per layer (`src/server/db/exceptions.ts`, `src/lib/validator/exceptions.ts`). The `httpExceptionMapper` in `src/lib/exception-http-mapper/` maps domain exceptions to HTTP status codes using decorators. API routes should use `withErrorHandler` middleware to catch and format errors.

### Validation
Use the custom validator in `src/lib/validator/` for input validation. Schemas live in `src/server/schemas/`. File uploads are validated with `parseFiles()` from `src/lib/validator/parsers.ts` (5 MB limit per file).

### Auth Middleware
`withAuthorization` reads the `session` cookie, decrypts the JWT, and attaches the user payload to the request context. Controlled by the `ENABLE_AUTH` env flag.

---

## Code Style

- **TypeScript strict mode** — no implicit `any`, all types explicit
- **Prettier:** single quotes, semicolons, trailing commas, 80-char width, 2-space indent
- **ESLint:** `next/core-web-vitals` + `next/typescript` (explicit `any` rule disabled)
- **Path alias:** `@/*` → `src/*`
- **Naming:** camelCase variables/functions, PascalCase classes/components, snake_case DB columns
- **Decorators:** enabled in tsconfig for HTTP mapper

---

## Testing

Tests live in `tests/`. Currently covers:
- `exception-http-mapper.test.ts` — error-to-HTTP mapping
- `validator.test.ts` — input validation logic

Run with `npm test`. Tests use `ts-jest` and target the Node environment.

When adding features, add corresponding tests for new validators, services, or error cases.

---

## Known Issues / TODOs

- `UserRepository.findAll()` is unimplemented (throws immediately) — needs SQL implementation
- Error handling on delete operations in `user-repository.ts` is incomplete (see `// ! handle error` comments)
- Neo4j is wired in `compose.yaml` and `src/server/db/` but not used anywhere in the application logic
- `graphDb` field in `UserRepository` is commented out

---

## Quick Reference

| Task | Location |
|---|---|
| Add API route | `src/app/api/<path>/route.ts` |
| Add DB migration | `npm run db:create <name>`, edit file in `migrations/` |
| Add UI component | `src/components/ui/` (shadcn/ui pattern) |
| Add service | `src/server/services/`, wire in `src/server/factories/` |
| Add validation schema | `src/server/schemas/` |
| Session access | `src/lib/auth/session.ts` — `encrypt()` / `decrypt()` |
| Send email | `src/lib/mailer/Mailer.ts` |
