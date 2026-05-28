# Identity & Access Management (IAM) — Phased Development Plan

> Project: 148-identity-access-management · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (Node.js 22 LTS) | API-first IAM platform with heavy JSON processing, JWT/JWK handling, and OIDC/SAML protocol work. TypeScript provides strong typing for security-critical code, excellent JWT/crypto ecosystem (jose, @simplewebauthn), and the same language for backend + admin frontend. |
| API framework | Fastify 5 | High-performance HTTP framework with built-in JSON Schema validation, OpenAPI 3.1 auto-generation via @fastify/swagger, first-class TypeScript support, and a plugin architecture that maps cleanly to IAM domains (auth, users, oauth, scim). |
| Database | PostgreSQL 16 | Multi-tenant IAM with RBAC, governance, and audit requires relational integrity, Row-Level Security for tenant isolation, JSONB for extensible attributes, partitioning for audit tables, and ltree for organizational hierarchies. Proven at scale in Keycloak, Okta, and SailPoint backends. |
| Database toolkit | Drizzle ORM | Type-safe SQL query builder with zero runtime overhead, PostgreSQL-native features (JSONB, CIDR, array types), and first-class migration support. Avoids the abstraction leaks of heavier ORMs while maintaining type safety. |
| Migration tool | Drizzle Kit | Integrated with Drizzle ORM for declarative schema migrations with SQL output for review. |
| Cache / session store | Redis 7 (via ioredis) | Session storage, rate limiting, token revocation lists, and MFA challenge state require sub-millisecond lookups. Redis is the standard choice for IAM session management. |
| Task queue | BullMQ (Redis-backed) | Async workloads: SCIM provisioning events, directory sync, access certification email dispatch, AI role-mining jobs. BullMQ provides reliable job processing with retries, dead-letter queues, and dashboard. |
| Frontend | Next.js 15 (App Router) | Admin dashboard and self-service portal. React Server Components for dashboard pages, client components for interactive flows (login, MFA enrollment). Shadcn/ui for accessible, composable UI components. |
| Authentication library | @simplewebauthn/server + jose | @simplewebauthn implements FIDO2/WebAuthn registration and authentication ceremonies. jose handles JWT signing/verification, JWK generation/rotation, and OIDC token operations without native dependencies. |
| SAML library | samlify | TypeScript SAML 2.0 SP and IdP implementation for enterprise SSO federation. Handles metadata exchange, assertion parsing, and signature validation. |
| Password hashing | argon2 (via argon2 npm) | OWASP-recommended password hashing. Argon2id variant provides resistance to both side-channel and GPU attacks. Configurable memory/time cost per NIST SP 800-63B. |
| Testing framework | Vitest | Fast TypeScript-native test runner with built-in coverage, mocking, and snapshot testing. Compatible with Fastify test helpers. |
| E2E testing | Playwright | Browser-based E2E tests for login flows, MFA enrollment, admin dashboard, and WebAuthn ceremonies. |
| Linting / formatting | ESLint 9 + Prettier | Flat config ESLint with TypeScript-ESLint rules. Prettier for consistent formatting. |
| Type checking | TypeScript 5.6 strict mode | strictNullChecks, noUncheckedIndexedAccess, exactOptionalPropertyTypes for security-critical code. |
| Containerisation | Docker + Docker Compose | Multi-container deployment: API server, worker (BullMQ), PostgreSQL, Redis. Dockerfile with multi-stage build for production image. |
| API documentation | OpenAPI 3.1 (auto-generated) | @fastify/swagger generates OAS 3.1 from route schemas. Published at /docs with Scalar UI. |
| Logging | Pino (structured JSON) | Fastify's built-in logger. Structured JSON logs with correlation IDs for distributed tracing. |
| Package manager | pnpm 9 | Monorepo-friendly, strict dependency resolution, disk-efficient. Workspace for api, web, and shared packages. |

### Project Structure

```
identity-access-management/
├── pnpm-workspace.yaml
├── package.json
├── tsconfig.base.json
├── docker-compose.yml
├── Dockerfile.api
├── Dockerfile.web
├── .env.example
├── drizzle.config.ts
├── packages/
│   ├── shared/                          # Shared types, constants, utilities
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── types/
│   │       │   ├── user.ts              # User, Group, SCIM types
│   │       │   ├── oauth.ts             # OAuth client, token, scope types
│   │       │   ├── rbac.ts              # Role, permission types
│   │       │   ├── auth.ts              # Credential, session, policy types
│   │       │   ├── governance.ts        # Certification, SoD types
│   │       │   ├── audit.ts             # Audit event types
│   │       │   └── index.ts
│   │       ├── constants/
│   │       │   ├── aal.ts               # NIST 800-63B AAL levels
│   │       │   ├── scim.ts              # SCIM schema URIs
│   │       │   └── oauth.ts             # Grant types, response types
│   │       └── utils/
│   │           ├── crypto.ts            # Hashing, token generation
│   │           └── validation.ts        # Shared validators
│   ├── db/                              # Database schema and migrations
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── schema/
│   │       │   ├── tenants.ts
│   │       │   ├── users.ts
│   │       │   ├── groups.ts
│   │       │   ├── credentials.ts
│   │       │   ├── sessions.ts
│   │       │   ├── oauth-clients.ts
│   │       │   ├── oauth-scopes.ts
│   │       │   ├── tokens.ts
│   │       │   ├── roles.ts
│   │       │   ├── authentication-policies.ts
│   │       │   ├── directory-connections.ts
│   │       │   ├── identity-providers.ts
│   │       │   ├── certification-campaigns.ts
│   │       │   ├── audit-events.ts
│   │       │   └── index.ts
│   │       ├── migrations/
│   │       ├── seed.ts
│   │       └── client.ts                # Drizzle client factory
│   └── api/                             # Fastify API server
│       ├── package.json
│       ├── tsconfig.json
│       └── src/
│           ├── server.ts                # Fastify app setup
│           ├── config.ts                # Environment config with validation
│           ├── plugins/
│           │   ├── auth.ts              # Authentication plugin (session, JWT)
│           │   ├── tenant.ts            # Multi-tenant context plugin
│           │   ├── rate-limit.ts        # Rate limiting plugin
│           │   └── audit.ts             # Audit logging plugin
│           ├── routes/
│           │   ├── health.ts
│           │   ├── auth/                # Login, logout, MFA flows
│           │   ├── users/               # User CRUD
│           │   ├── groups/              # Group CRUD
│           │   ├── roles/               # Role and permission management
│           │   ├── oauth/               # OAuth 2.1 authorization server
│           │   ├── oidc/                # OIDC discovery, userinfo, jwks
│           │   ├── saml/                # SAML 2.0 SSO endpoints
│           │   ├── scim/                # SCIM 2.0 provisioning API
│           │   ├── admin/               # Admin management endpoints
│           │   ├── governance/          # Certification campaigns
│           │   └── agents/              # Agent identity management
│           ├── services/
│           │   ├── user.service.ts
│           │   ├── auth.service.ts
│           │   ├── session.service.ts
│           │   ├── credential.service.ts
│           │   ├── oauth.service.ts
│           │   ├── token.service.ts
│           │   ├── role.service.ts
│           │   ├── permission.service.ts
│           │   ├── group.service.ts
│           │   ├── scim.service.ts
│           │   ├── saml.service.ts
│           │   ├── federation.service.ts
│           │   ├── governance.service.ts
│           │   ├── audit.service.ts
│           │   └── directory-sync.service.ts
│           ├── middleware/
│           │   ├── authenticate.ts
│           │   ├── authorize.ts
│           │   ├── tenant-context.ts
│           │   └── request-id.ts
│           └── workers/
│               ├── provisioning.worker.ts
│               ├── directory-sync.worker.ts
│               ├── certification.worker.ts
│               └── token-cleanup.worker.ts
├── apps/
│   └── web/                             # Next.js admin dashboard + self-service
│       ├── package.json
│       ├── tsconfig.json
│       ├── next.config.ts
│       └── src/
│           ├── app/
│           │   ├── layout.tsx
│           │   ├── (auth)/              # Login, MFA, password reset pages
│           │   ├── (dashboard)/         # Admin dashboard
│           │   │   ├── users/
│           │   │   ├── groups/
│           │   │   ├── roles/
│           │   │   ├── applications/
│           │   │   ├── policies/
│           │   │   ├── governance/
│           │   │   ├── audit/
│           │   │   └── settings/
│           │   └── (self-service)/      # End-user self-service portal
│           ├── components/
│           │   ├── ui/                  # Shadcn/ui components
│           │   └── iam/                 # Domain-specific components
│           └── lib/
│               ├── api-client.ts
│               └── auth.ts
└── tests/
    ├── fixtures/
    │   ├── users.json
    │   ├── saml-metadata.xml
    │   └── webauthn-attestation.json
    ├── helpers/
    │   ├── test-server.ts               # Fastify test instance factory
    │   ├── test-db.ts                   # Test database helpers
    │   └── factories.ts                 # Test data factories
    └── e2e/
        ├── login.spec.ts
        ├── mfa-enrollment.spec.ts
        └── admin-users.spec.ts
```

---

## Phase 1: Foundation & Core Schema

### Purpose
Establish the project scaffold, database schema, configuration system, and Fastify server skeleton. After this phase, the API server starts, connects to PostgreSQL and Redis, applies migrations, and responds to health checks. All subsequent phases build on this foundation.

### Tasks

#### 1.1 — Monorepo scaffold and tooling

**What**: Initialize the pnpm workspace with shared, db, api, and web packages, plus TypeScript, ESLint, Prettier, and Vitest configuration.

**Design**:

```typescript
// pnpm-workspace.yaml
packages:
  - 'packages/*'
  - 'apps/*'
```

```typescript
// tsconfig.base.json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "strictNullChecks": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "dist",
    "rootDir": "src"
  }
}
```

Root package.json scripts:
- `dev` — run api + web concurrently
- `build` — build all packages
- `test` — run vitest across workspace
- `lint` — ESLint across workspace
- `format` — Prettier across workspace
- `db:migrate` — run Drizzle migrations
- `db:seed` — seed development data
- `docker:up` — docker-compose up

**Testing**:
- `Unit: pnpm install completes without errors`
- `Unit: pnpm build compiles all packages without type errors`
- `Unit: pnpm lint passes with zero warnings`
- `Unit: pnpm test runs and finds test files in all packages`

---

#### 1.2 — Environment configuration

**What**: Type-safe environment configuration with validation, defaults, and .env support.

**Design**:

```typescript
// packages/api/src/config.ts
import { z } from 'zod';

const configSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().default(3000),
  HOST: z.string().default('0.0.0.0'),

  // Database
  DATABASE_URL: z.string().url(),
  DATABASE_POOL_MIN: z.coerce.number().default(2),
  DATABASE_POOL_MAX: z.coerce.number().default(10),

  // Redis
  REDIS_URL: z.string().url().default('redis://localhost:6379'),

  // Security
  SESSION_SECRET: z.string().min(32),
  JWT_ISSUER: z.string().url(),
  JWT_SIGNING_KEY_ID: z.string().optional(),
  COOKIE_DOMAIN: z.string().optional(),
  CORS_ORIGINS: z.string().transform(s => s.split(',')).default('http://localhost:3001'),

  // Rate limiting
  RATE_LIMIT_MAX: z.coerce.number().default(100),
  RATE_LIMIT_WINDOW_MS: z.coerce.number().default(60_000),

  // Argon2 password hashing
  ARGON2_MEMORY_COST: z.coerce.number().default(65536),   // 64 MB
  ARGON2_TIME_COST: z.coerce.number().default(3),
  ARGON2_PARALLELISM: z.coerce.number().default(4),

  // Log level
  LOG_LEVEL: z.enum(['fatal', 'error', 'warn', 'info', 'debug', 'trace']).default('info'),
});

export type Config = z.infer<typeof configSchema>;

export function loadConfig(): Config {
  return configSchema.parse(process.env);
}
```

**Testing**:
- `Unit: valid env vars → Config object with all fields populated`
- `Unit: missing DATABASE_URL → ZodError with clear message`
- `Unit: missing SESSION_SECRET → ZodError indicating min length 32`
- `Unit: PORT as string "3000" → coerced to number 3000`
- `Unit: default values applied when optional vars omitted`
- `Unit: CORS_ORIGINS "a.com,b.com" → array ["a.com", "b.com"]`

---

#### 1.3 — Database schema (core identity tables)

**What**: Drizzle ORM schema for tenants, users, groups, group_members, credentials, sessions, and audit_events tables. Adopts the Hybrid Relational + JSONB model (Data Model Suggestion 3) for its balance of relational integrity and extensibility.

**Design**:

```typescript
// packages/db/src/schema/tenants.ts
import { pgTable, uuid, varchar, boolean, jsonb, timestamp, index } from 'drizzle-orm/pg-core';

export const tenants = pgTable('tenants', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull().unique(),
  displayName: varchar('display_name', { length: 255 }),
  isActive: boolean('is_active').notNull().default(true),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_tenants_slug').on(table.slug),
]);

// packages/db/src/schema/users.ts
import { pgTable, uuid, varchar, boolean, jsonb, timestamp, uniqueIndex, index } from 'drizzle-orm/pg-core';
import { tenants } from './tenants';

export const identityStateEnum = ['staged', 'active', 'suspended', 'deprovisioned'] as const;

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id, { onDelete: 'cascade' }),
  username: varchar('username', { length: 255 }).notNull(),
  email: varchar('email', { length: 320 }),
  emailVerified: boolean('email_verified').notNull().default(false),
  displayName: varchar('display_name', { length: 255 }),
  givenName: varchar('given_name', { length: 128 }),
  familyName: varchar('family_name', { length: 128 }),
  isActive: boolean('is_active').notNull().default(true),
  identityState: varchar('identity_state', { length: 32 }).notNull().default('active'),
  // SCIM multi-valued attributes as JSONB
  emails: jsonb('emails').notNull().default([]),
  phoneNumbers: jsonb('phone_numbers').notNull().default([]),
  addresses: jsonb('addresses').notNull().default([]),
  // SCIM Enterprise Extension
  enterpriseProfile: jsonb('enterprise_profile').notNull().default({}),
  // Tenant/jurisdiction-specific custom attributes
  customAttributes: jsonb('custom_attributes').notNull().default({}),
  // Lifecycle
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
  provisionedAt: timestamp('provisioned_at', { withTimezone: true }),
  deprovisionedAt: timestamp('deprovisioned_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_users_tenant_username').on(table.tenantId, table.username),
  index('idx_users_email').on(table.tenantId, table.email),
  index('idx_users_state').on(table.tenantId, table.identityState),
]);

// packages/db/src/schema/credentials.ts
export const credentialTypeEnum = [
  'password', 'totp', 'webauthn', 'recovery_code',
  'email_otp', 'sms_otp', 'push', 'certificate'
] as const;

export const credentials = pgTable('credentials', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  credentialType: varchar('credential_type', { length: 32 }).notNull(),
  displayName: varchar('display_name', { length: 255 }),
  aalLevel: smallint('aal_level').notNull().default(1),
  isActive: boolean('is_active').notNull().default(true),
  credentialData: jsonb('credential_data').notNull(),
  lastUsedAt: timestamp('last_used_at', { withTimezone: true }),
  expiresAt: timestamp('expires_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_credentials_user').on(table.userId, table.credentialType),
]);
```

Remaining tables (groups, group_members, sessions, audit_events) follow the same pattern as defined in Data Model Suggestion 3. The full 23-table schema is applied across phases — Phase 1 creates the core identity tables, Phase 2 adds RBAC tables, Phase 3 adds OAuth/OIDC tables, etc.

**Testing**:
- `Unit: drizzle-kit generate produces valid SQL migration`
- `Integration (real DB): migration applies cleanly to empty PostgreSQL`
- `Integration (real DB): migration is idempotent (apply twice without error)`
- `Integration (real DB): foreign key from users.tenant_id to tenants.id enforced`
- `Integration (real DB): unique constraint on (tenant_id, username) prevents duplicates`
- `Integration (real DB): cascade delete on tenant removes all users`
- `Unit: JSONB defaults produce empty arrays/objects`

---

#### 1.4 — Fastify server skeleton with health check

**What**: Fastify application factory with plugin registration, Pino structured logging, CORS, request ID, and a health endpoint.

**Design**:

```typescript
// packages/api/src/server.ts
import Fastify, { FastifyInstance } from 'fastify';
import cors from '@fastify/cors';
import { loadConfig, Config } from './config';

export async function buildApp(overrides?: Partial<Config>): Promise<FastifyInstance> {
  const config = { ...loadConfig(), ...overrides };

  const app = Fastify({
    logger: {
      level: config.LOG_LEVEL,
      transport: config.NODE_ENV === 'development'
        ? { target: 'pino-pretty' }
        : undefined,
    },
    genReqId: () => crypto.randomUUID(),
    trustProxy: true,
  });

  // Plugins
  await app.register(cors, { origin: config.CORS_ORIGINS, credentials: true });

  // Decorate with config and DB client
  app.decorate('config', config);

  // Health check
  app.get('/healthz', {
    schema: {
      response: {
        200: {
          type: 'object',
          properties: {
            status: { type: 'string', enum: ['ok'] },
            version: { type: 'string' },
            timestamp: { type: 'string', format: 'date-time' },
          },
        },
      },
    },
  }, async () => ({
    status: 'ok' as const,
    version: process.env.npm_package_version ?? '0.0.0',
    timestamp: new Date().toISOString(),
  }));

  return app;
}
```

**Testing**:
- `Unit: buildApp() creates Fastify instance without errors`
- `Integration: GET /healthz returns 200 with { status: "ok", version, timestamp }`
- `Integration: request includes x-request-id header in response`
- `Integration: CORS headers present for configured origins`
- `Unit: invalid config → app fails to start with clear error message`

---

#### 1.5 — Docker Compose development environment

**What**: Docker Compose file with PostgreSQL 16, Redis 7, API server, and web app containers. Includes volume mounts, health checks, and network configuration.

**Design**:

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: iam
      POSTGRES_USER: iam
      POSTGRES_PASSWORD: iam_dev_password
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U iam"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  api:
    build:
      context: .
      dockerfile: Dockerfile.api
      target: development
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://iam:iam_dev_password@postgres:5432/iam
      REDIS_URL: redis://redis:6379
      SESSION_SECRET: dev-session-secret-must-be-32-chars-minimum
      JWT_ISSUER: http://localhost:3000
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./packages:/app/packages
    command: pnpm --filter @iam/api dev

volumes:
  pgdata:
```

**Testing**:
- `Integration: docker compose up starts all services without errors`
- `Integration: API container connects to PostgreSQL (health check passes)`
- `Integration: API container connects to Redis (health check passes)`
- `Integration: GET http://localhost:3000/healthz returns 200`

---

## Phase 2: User Management & RBAC

### Purpose
Implement the user directory, group management, role-based access control, and the admin API for managing these entities. After this phase, administrators can create tenants, manage users (CRUD), organise users into groups, define roles with permissions, and assign roles to users and groups. This is the core identity management layer that all authentication and authorization features build upon.

### Tasks

#### 2.1 — Tenant management API

**What**: CRUD endpoints for tenant (realm) management with slug-based lookup.

**Design**:

```typescript
// packages/shared/src/types/tenant.ts
export interface Tenant {
  id: string;          // UUID
  name: string;
  slug: string;        // URL-safe, unique
  displayName: string | null;
  isActive: boolean;
  settings: TenantSettings;
  createdAt: string;   // ISO 8601
  updatedAt: string;
}

export interface TenantSettings {
  branding?: { logoUrl?: string; primaryColor?: string };
  defaultLocale?: string;
  passwordPolicy?: { minLength: number; requireMfa: boolean };
  jurisdiction?: string;
  dataResidency?: string;
  features?: Record<string, boolean>;
}

// API endpoints
// POST   /api/v1/tenants               → create tenant
// GET    /api/v1/tenants               → list tenants (paginated)
// GET    /api/v1/tenants/:slug         → get tenant by slug
// PATCH  /api/v1/tenants/:slug         → update tenant
// DELETE /api/v1/tenants/:slug         → soft-delete (set isActive=false)
```

Request/response schemas use Fastify's JSON Schema validation with `@fastify/swagger` annotations for OpenAPI generation.

**Testing**:
- `Integration: POST /api/v1/tenants with valid body → 201 with tenant object`
- `Integration: POST /api/v1/tenants with duplicate slug → 409 Conflict`
- `Integration: GET /api/v1/tenants → 200 with paginated list`
- `Integration: GET /api/v1/tenants/:slug → 200 with tenant object`
- `Integration: GET /api/v1/tenants/nonexistent → 404`
- `Integration: PATCH /api/v1/tenants/:slug → 200 with updated tenant`
- `Integration: DELETE /api/v1/tenants/:slug → 200, tenant.isActive = false`
- `Unit: slug validation rejects spaces, special characters`
- `Unit: TenantSettings JSONB validated at application layer`

---

#### 2.2 — User CRUD API (SCIM-aligned)

**What**: User management endpoints with SCIM 2.0-aligned attribute schema, multi-valued attributes (emails, phones, addresses), enterprise extension, and custom attributes.

**Design**:

```typescript
// packages/shared/src/types/user.ts
export interface User {
  id: string;
  tenantId: string;
  username: string;
  email: string | null;
  emailVerified: boolean;
  displayName: string | null;
  givenName: string | null;
  familyName: string | null;
  isActive: boolean;
  identityState: IdentityState;
  emails: ScimMultiValue[];
  phoneNumbers: ScimMultiValue[];
  addresses: ScimAddress[];
  enterpriseProfile: EnterpriseProfile;
  customAttributes: Record<string, unknown>;
  lastLoginAt: string | null;
  createdAt: string;
  updatedAt: string;
}

export type IdentityState = 'staged' | 'active' | 'suspended' | 'deprovisioned';

export interface ScimMultiValue {
  value: string;
  type?: string;     // work, home, mobile, other
  primary?: boolean;
}

export interface ScimAddress {
  streetAddress?: string;
  locality?: string;
  region?: string;
  postalCode?: string;
  country?: string;  // ISO 3166-1 alpha-2
  type?: string;
  primary?: boolean;
}

export interface EnterpriseProfile {
  employeeNumber?: string;
  costCenter?: string;
  organization?: string;
  division?: string;
  department?: string;
  managerId?: string;
}

// API endpoints
// POST   /api/v1/users                  → create user
// GET    /api/v1/users                  → list users (paginated, filterable)
// GET    /api/v1/users/:id              → get user by ID
// PATCH  /api/v1/users/:id              → update user
// DELETE /api/v1/users/:id              → deprovision user
// POST   /api/v1/users/:id/suspend      → suspend user
// POST   /api/v1/users/:id/reactivate   → reactivate user
```

All user endpoints are tenant-scoped via the `X-Tenant-Id` header or JWT tenant claim. Pagination uses cursor-based pagination with `after` and `limit` parameters.

**Testing**:
- `Integration: POST /api/v1/users with valid body → 201 with user object`
- `Integration: POST /api/v1/users with duplicate username in same tenant → 409`
- `Integration: POST /api/v1/users with duplicate username in different tenant → 201 (tenant isolation)`
- `Integration: GET /api/v1/users → paginated list with cursor`
- `Integration: GET /api/v1/users?email=jane@example.com → filtered results`
- `Integration: PATCH /api/v1/users/:id with partial body → 200 with merged attributes`
- `Integration: DELETE /api/v1/users/:id → identityState changed to 'deprovisioned'`
- `Integration: POST /api/v1/users/:id/suspend → identityState changed to 'suspended'`
- `Unit: SCIM multi-valued email array validated (at most one primary=true)`
- `Unit: ISO 3166-1 alpha-2 country code validated in addresses`
- `Unit: enterpriseProfile JSONB validated against EnterpriseProfile schema`

---

#### 2.3 — Group management API

**What**: CRUD endpoints for groups and group membership management.

**Design**:

```typescript
// packages/shared/src/types/group.ts
export interface Group {
  id: string;
  tenantId: string;
  displayName: string;
  description: string | null;
  groupType: 'standard' | 'dynamic' | 'system';
  dynamicRules: DynamicGroupRule | null;
  metadata: Record<string, unknown>;
  memberCount: number;  // computed
  createdAt: string;
  updatedAt: string;
}

export interface DynamicGroupRule {
  match: 'all' | 'any';
  conditions: Array<{
    field: string;      // e.g. 'enterprise_profile.department'
    op: 'eq' | 'neq' | 'contains' | 'starts_with';
    value: string;
  }>;
}

// API endpoints
// POST   /api/v1/groups                          → create group
// GET    /api/v1/groups                          → list groups (paginated)
// GET    /api/v1/groups/:id                      → get group with member count
// PATCH  /api/v1/groups/:id                      → update group
// DELETE /api/v1/groups/:id                      → delete group
// GET    /api/v1/groups/:id/members              → list members (paginated)
// POST   /api/v1/groups/:id/members              → add members (batch)
// DELETE /api/v1/groups/:id/members/:userId      → remove member
```

**Testing**:
- `Integration: POST /api/v1/groups → 201 with group object`
- `Integration: POST /api/v1/groups/:id/members with [userId] → 200, member added`
- `Integration: POST /api/v1/groups/:id/members with already-member userId → 409`
- `Integration: GET /api/v1/groups/:id/members → paginated member list`
- `Integration: DELETE /api/v1/groups/:id/members/:userId → 200, member removed`
- `Integration: DELETE /api/v1/groups/:id → group deleted, memberships cascade-deleted`
- `Unit: DynamicGroupRule validation rejects unknown operators`

---

#### 2.4 — Roles and permissions (RBAC)

**What**: Role and permission management with JSONB-embedded permissions, user-role and group-role assignments, time-bounded assignments, and effective permission resolution.

**Design**:

```typescript
// packages/shared/src/types/rbac.ts
export interface Role {
  id: string;
  tenantId: string;
  name: string;
  displayName: string | null;
  description: string | null;
  roleType: 'system' | 'custom' | 'template';
  permissions: PermissionGrant[];
  isActive: boolean;
  metadata: Record<string, unknown>;
  createdAt: string;
  updatedAt: string;
}

export interface PermissionGrant {
  resource: string;       // e.g. 'user', 'application', 'role', 'group'
  actions: string[];      // e.g. ['read', 'write', 'delete', 'manage']
}

export interface RoleAssignment {
  id: string;
  userId: string;
  roleId: string;
  roleName: string;
  grantedBy: string | null;
  grantedAt: string;
  expiresAt: string | null;
  metadata: Record<string, unknown>;
}

// API endpoints
// POST   /api/v1/roles                              → create role
// GET    /api/v1/roles                              → list roles
// GET    /api/v1/roles/:id                          → get role
// PATCH  /api/v1/roles/:id                          → update role (including permissions)
// DELETE /api/v1/roles/:id                          → delete role
// POST   /api/v1/users/:id/roles                    → assign role to user
// DELETE /api/v1/users/:id/roles/:roleId            → revoke role from user
// GET    /api/v1/users/:id/roles                    → list user's direct role assignments
// GET    /api/v1/users/:id/effective-permissions     → computed: all permissions (direct + group-inherited)
// POST   /api/v1/groups/:id/roles                   → assign role to group
// DELETE /api/v1/groups/:id/roles/:roleId            → revoke role from group
```

Effective permission resolution query (from Data Model Suggestion 3 approach):

```typescript
// packages/api/src/services/permission.service.ts
export async function getEffectivePermissions(
  db: DrizzleClient,
  userId: string
): Promise<PermissionGrant[]> {
  // 1. Get direct role assignments for the user
  // 2. Get all groups the user belongs to
  // 3. Get role assignments for those groups
  // 4. Merge all roles, deduplicate permissions
  // 5. Filter out expired assignments (expires_at < now())
  // Returns flattened, deduplicated PermissionGrant[]
}
```

**Testing**:
- `Integration: POST /api/v1/roles with permissions array → 201`
- `Integration: POST /api/v1/users/:id/roles → role assigned, 200`
- `Integration: POST /api/v1/users/:id/roles with expiresAt in past → 400`
- `Integration: GET /api/v1/users/:id/effective-permissions → includes direct and group-inherited`
- `Integration: user in group with role "editor" + direct role "viewer" → effective permissions merged`
- `Integration: expired role assignment excluded from effective permissions`
- `Integration: DELETE /api/v1/roles/:id with assigned users → 409 (must unassign first) or cascade`
- `Unit: permission deduplication merges overlapping grants correctly`
- `Unit: PermissionGrant validation rejects empty actions array`

---

#### 2.5 — Audit logging service

**What**: Append-only audit event recording for all identity operations, with tenant-scoped querying.

**Design**:

```typescript
// packages/shared/src/types/audit.ts
export interface AuditEvent {
  id: string;
  tenantId: string;
  actorId: string | null;
  actorType: 'user' | 'admin' | 'system' | 'agent' | 'scim';
  eventType: string;           // e.g. 'user.created', 'role.assigned', 'session.created'
  eventCategory: AuditCategory;
  eventOutcome: 'success' | 'failure' | 'error';
  targetType: string | null;   // 'user', 'role', 'group', etc.
  targetId: string | null;
  details: Record<string, unknown>;
  occurredAt: string;
}

export type AuditCategory =
  | 'authentication' | 'authorization' | 'user_management'
  | 'group_management' | 'application_management' | 'policy_management'
  | 'provisioning' | 'governance' | 'system';

// packages/api/src/services/audit.service.ts
export class AuditService {
  async record(event: Omit<AuditEvent, 'id' | 'occurredAt'>): Promise<void>;
  async query(params: AuditQueryParams): Promise<PaginatedResult<AuditEvent>>;
}

// API endpoints
// GET /api/v1/audit-events → list audit events (paginated, filterable by type, actor, target, date range)
```

The audit service is injected as a Fastify plugin and called from all service methods. Audit events are inserted asynchronously (fire-and-forget with error logging) to avoid blocking the primary request.

**Testing**:
- `Integration: creating a user → audit event with type 'user.created' recorded`
- `Integration: assigning a role → audit event with type 'role.assigned' recorded`
- `Integration: GET /api/v1/audit-events?eventType=user.created → filtered results`
- `Integration: GET /api/v1/audit-events?actorId=:id → events by actor`
- `Integration: audit events are tenant-isolated (tenant A cannot see tenant B events)`
- `Unit: AuditService.record with missing required fields → error logged, not thrown`

---

#### 2.6 — Authorization middleware

**What**: Fastify middleware/decorator that checks effective permissions before allowing access to protected endpoints.

**Design**:

```typescript
// packages/api/src/middleware/authorize.ts
export function requirePermission(resource: string, action: string) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    const user = request.user; // set by authenticate middleware
    if (!user) {
      return reply.status(401).send({ error: 'Unauthorized' });
    }
    const permissions = await getEffectivePermissions(request.db, user.id);
    const hasPermission = permissions.some(
      p => p.resource === resource && p.actions.includes(action)
    );
    if (!hasPermission) {
      await auditService.record({
        tenantId: user.tenantId,
        actorId: user.id,
        actorType: 'user',
        eventType: 'authorization.denied',
        eventCategory: 'authorization',
        eventOutcome: 'failure',
        targetType: resource,
        targetId: null,
        details: { requiredAction: action },
      });
      return reply.status(403).send({ error: 'Forbidden' });
    }
  };
}

// Usage in routes:
app.get('/api/v1/users', {
  preHandler: [authenticate, requirePermission('user', 'read')],
}, userListHandler);
```

**Testing**:
- `Integration: request with valid permissions → handler executes`
- `Integration: request without required permission → 403 Forbidden`
- `Integration: 403 response triggers audit event with eventType 'authorization.denied'`
- `Integration: unauthenticated request → 401 Unauthorized`
- `Unit: requirePermission('user', 'write') matches role with {resource: 'user', actions: ['read','write']}`
- `Unit: requirePermission('user', 'delete') does NOT match role with {resource: 'user', actions: ['read','write']}`

---

## Phase 3: Authentication Engine

### Purpose
Implement the core authentication system: password-based login, session management, TOTP-based MFA, and credential lifecycle management. After this phase, users can register credentials, log in with password + MFA, maintain sessions, and be subject to configurable authentication policies with AAL enforcement per NIST SP 800-63B.

### Tasks

#### 3.1 — Password credential management

**What**: Password hashing (Argon2id), storage, verification, and password change flow.

**Design**:

```typescript
// packages/api/src/services/credential.service.ts
import argon2 from 'argon2';

export class CredentialService {
  async setPassword(userId: string, plaintext: string): Promise<void> {
    const hash = await argon2.hash(plaintext, {
      type: argon2.argon2id,
      memoryCost: this.config.ARGON2_MEMORY_COST,
      timeCost: this.config.ARGON2_TIME_COST,
      parallelism: this.config.ARGON2_PARALLELISM,
    });
    // Upsert credential with type 'password', credentialData: { hash, algorithm: 'argon2id' }
  }

  async verifyPassword(userId: string, plaintext: string): Promise<boolean> {
    const cred = await this.getCredential(userId, 'password');
    if (!cred || !cred.isActive) return false;
    return argon2.verify(cred.credentialData.hash, plaintext);
  }

  async changePassword(userId: string, oldPassword: string, newPassword: string): Promise<void> {
    const valid = await this.verifyPassword(userId, oldPassword);
    if (!valid) throw new UnauthorizedError('Invalid current password');
    await this.setPassword(userId, newPassword);
    // Audit: user.password_changed
  }
}
```

Password policy enforcement is driven by the tenant's `settings.passwordPolicy`:
- Minimum length (default 12, per NIST SP 800-63B)
- Check against breached password lists (HaveIBeenPwned API, optional)
- No composition rules (NIST SP 800-63B explicitly discourages them)

**Testing**:
- `Unit: setPassword stores argon2id hash in credential_data`
- `Unit: verifyPassword with correct password → true`
- `Unit: verifyPassword with wrong password → false`
- `Unit: verifyPassword with inactive credential → false`
- `Integration: changePassword with correct old password → new hash stored, audit event`
- `Integration: changePassword with wrong old password → 401`
- `Unit: password shorter than tenant minLength → validation error`
- `Unit: argon2id hash format verified (starts with $argon2id$)`

---

#### 3.2 — Session management

**What**: Secure session creation, validation, refresh, and revocation using Redis-backed session tokens.

**Design**:

```typescript
// packages/api/src/services/session.service.ts
export class SessionService {
  async createSession(params: {
    tenantId: string;
    userId: string;
    aalAchieved: 1 | 2 | 3;
    ip: string;
    userAgent: string;
  }): Promise<{ sessionToken: string; expiresAt: Date }> {
    const token = crypto.randomBytes(32).toString('base64url');
    const tokenHash = createHash('sha256').update(token).digest('hex');
    const expiresAt = new Date(Date.now() + this.config.SESSION_TTL_MS);
    // Insert into sessions table
    // Store session in Redis for fast lookup: `session:${tokenHash}` → JSON
    return { sessionToken: token, expiresAt };
  }

  async validateSession(token: string): Promise<Session | null> {
    const tokenHash = createHash('sha256').update(token).digest('hex');
    // Check Redis first, fall back to DB
    // Return null if expired, revoked, or not found
  }

  async revokeSession(sessionId: string): Promise<void> {
    // Set is_active=false in DB, delete from Redis
    // Audit: session.terminated
  }

  async revokeAllUserSessions(userId: string): Promise<number> {
    // Revoke all active sessions for a user (e.g., on password change)
  }
}
```

Session token is sent as an HttpOnly, Secure, SameSite=Lax cookie named `__iam_session`.

**Testing**:
- `Integration: createSession → session stored in DB and Redis`
- `Integration: validateSession with valid token → Session object`
- `Integration: validateSession with expired session → null`
- `Integration: validateSession with revoked session → null`
- `Integration: revokeSession → Redis key deleted, DB row updated`
- `Integration: revokeAllUserSessions → all user sessions invalidated`
- `Unit: session token is 32 bytes of crypto-random, base64url-encoded`
- `Unit: token hash is SHA-256 (raw token never stored)`

---

#### 3.3 — Login flow (password + optional MFA)

**What**: Authentication endpoint that processes password verification, checks authentication policy for MFA requirement, and issues a session.

**Design**:

```typescript
// API endpoints
// POST /api/v1/auth/login
//   Request: { username: string, password: string, tenantSlug: string }
//   Response (no MFA): { session: { token, expiresAt }, user: { id, username, ... } }
//   Response (MFA required): { mfaChallengeId: string, availableMethods: ['totp', 'webauthn'] }
//
// POST /api/v1/auth/mfa/verify
//   Request: { mfaChallengeId: string, method: 'totp', code: string }
//   Response: { session: { token, expiresAt }, user: { id, username, ... } }

// Login flow:
// 1. Look up user by (tenantSlug, username)
// 2. Verify password credential
// 3. Evaluate authentication_policies for the tenant (ordered by priority)
//    - Check if MFA is required based on conditions (IP, risk level, etc.)
// 4. If MFA not required → AAL1 session, return session token
// 5. If MFA required → store MFA challenge in Redis (TTL 5min), return challenge ID
// 6. User submits MFA code → verify → AAL2 session, return session token
```

Rate limiting: Max 5 failed login attempts per username per 15-minute window (Redis counter).

**Testing**:
- `Integration: valid username/password, no MFA required → 200 with session token`
- `Integration: valid username/password, MFA required → 200 with mfaChallengeId`
- `Integration: invalid password → 401, audit event 'user.login_failed'`
- `Integration: nonexistent username → 401 (same response time as invalid password to prevent enumeration)`
- `Integration: deprovisioned user → 401`
- `Integration: suspended user → 403 with 'account_suspended' error code`
- `Integration: 6th failed attempt in 15min → 429 Too Many Requests`
- `Integration: successful login → audit event 'user.login_succeeded'`
- `Integration: successful login updates user.lastLoginAt`
- `Integration: MFA challenge expires after 5 minutes → 410 Gone`

---

#### 3.4 — TOTP MFA enrollment and verification

**What**: TOTP (RFC 6238) second-factor enrollment, QR code generation, verification, and recovery codes.

**Design**:

```typescript
// API endpoints
// POST /api/v1/users/:id/mfa/totp/enroll
//   Response: { secret: string, qrCodeDataUrl: string, recoveryCodes: string[] }
//
// POST /api/v1/users/:id/mfa/totp/confirm
//   Request: { code: string }  (verifies enrollment with first valid code)
//   Response: 200
//
// POST /api/v1/auth/mfa/verify
//   Request: { mfaChallengeId: string, method: 'totp', code: string }

// TOTP credential_data schema:
// {
//   "secret_encrypted": "base64...",
//   "algorithm": "SHA1",         // SHA1 for maximum authenticator app compatibility
//   "digits": 6,
//   "period": 30
// }

// Recovery codes: 10 one-time codes generated at enrollment.
// Stored as credential type 'recovery_code' with:
// {
//   "codes": [
//     { "hash": "sha256...", "used": false },
//     ...
//   ]
// }
```

TOTP secret is encrypted at rest using AES-256-GCM with a key derived from the server's `SESSION_SECRET`.

**Testing**:
- `Integration: POST /api/v1/users/:id/mfa/totp/enroll → 200 with secret and QR code`
- `Integration: POST /api/v1/users/:id/mfa/totp/confirm with valid code → 200, credential activated`
- `Integration: POST /api/v1/users/:id/mfa/totp/confirm with invalid code → 400`
- `Integration: TOTP verification accepts code for current and previous time step (clock drift tolerance)`
- `Integration: recovery code can be used once, then marked as used`
- `Integration: all recovery codes used → user must re-enroll MFA`
- `Unit: TOTP generation produces valid 6-digit code for known secret and time`
- `Unit: TOTP secret encrypted at rest (not stored in plaintext)`

---

#### 3.5 — Authentication policies

**What**: Configurable per-tenant authentication policies with conditional access rules that determine MFA requirements, AAL levels, and session parameters.

**Design**:

```typescript
// packages/db/src/schema/authentication-policies.ts
export const authenticationPolicies = pgTable('authentication_policies', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  priority: integer('priority').notNull().default(0),
  isActive: boolean('is_active').notNull().default(true),
  conditions: jsonb('conditions').notNull().default({}),
  actions: jsonb('actions').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// Policy evaluation logic:
export async function evaluateAuthPolicies(
  tenantId: string,
  context: AuthContext
): Promise<PolicyDecision> {
  // 1. Load active policies for tenant, sorted by priority DESC
  // 2. For each policy, evaluate conditions against context
  //    - IP in allowed ranges?
  //    - Country in allowed list?
  //    - Device type matches?
  //    - Risk level acceptable?
  // 3. First matching policy determines the decision
  // 4. If no policy matches, use tenant default (AAL1, no MFA)
  // Returns: { requiredAal, requireMfa, mfaTypes, sessionMaxAge }
}

interface AuthContext {
  ip: string;
  userAgent: string;
  country: string | null;
  deviceType: string | null;
  riskLevel: 'low' | 'medium' | 'high' | 'critical';
}

interface PolicyDecision {
  requiredAal: 1 | 2 | 3;
  requireMfa: boolean;
  mfaTypes: string[];
  sessionMaxAgeSeconds: number;
}
```

**Testing**:
- `Unit: policy with condition {countries: ["US"]} matches context {country: "US"}`
- `Unit: policy with condition {countries: ["US"]} does NOT match context {country: "DE"}`
- `Unit: policy with condition {ip_ranges: ["10.0.0.0/8"]} matches IP 10.0.1.5`
- `Unit: multiple policies → highest priority match wins`
- `Unit: no matching policy → default decision (AAL1, no MFA)`
- `Integration: POST /api/v1/policies → 201 with policy object`
- `Integration: login from non-US IP with US-only policy → MFA required`

---

## Phase 4: OAuth 2.1 Authorization Server

### Purpose
Implement a standards-compliant OAuth 2.1 authorization server with PKCE enforcement, OpenID Connect Core, token management, and JWK rotation. After this phase, the platform can serve as an identity provider for external applications using industry-standard OAuth 2.1/OIDC flows.

### Tasks

#### 4.1 — OAuth client (application) registration

**What**: CRUD for OAuth client registration with support for confidential, public, and machine client types.

**Design**:

```typescript
// API endpoints
// POST   /api/v1/applications                → register OAuth client
// GET    /api/v1/applications                → list applications
// GET    /api/v1/applications/:id            → get application
// PATCH  /api/v1/applications/:id            → update application
// DELETE /api/v1/applications/:id            → deactivate application
// POST   /api/v1/applications/:id/rotate-secret → rotate client secret

// OAuth client registration fields (from Data Model Suggestion 3):
// - client_id: auto-generated UUID-based string
// - client_secret_hash: argon2id hash (null for public clients)
// - client_type: 'confidential' | 'public' | 'machine'
// - oauth_config: JSONB with redirect_uris, grant_types, response_types, token TTLs
// - saml_config: JSONB for SAML SP configuration (optional)
// - agent_config: JSONB for AI agent identity (optional)
```

**Testing**:
- `Integration: register confidential client → client_id and client_secret returned`
- `Integration: register public client → client_id returned, no secret`
- `Integration: rotate-secret → new secret returned, old secret invalidated`
- `Integration: register client with invalid redirect_uri (http in production) → 400`
- `Unit: client_id format is URL-safe, unique`
- `Unit: client_secret is 32 bytes of crypto-random, base64url-encoded`

---

#### 4.2 — Authorization code flow with PKCE

**What**: OAuth 2.1 authorization endpoint supporting the authorization code grant with mandatory PKCE (RFC 7636).

**Design**:

```typescript
// GET /api/v1/oauth/authorize
//   Query: response_type=code, client_id, redirect_uri, scope, state,
//          code_challenge, code_challenge_method=S256
//   Flow:
//     1. Validate client_id and redirect_uri
//     2. Validate code_challenge is present (PKCE mandatory per OAuth 2.1)
//     3. If user not authenticated → redirect to login
//     4. If user authenticated → show consent screen (if required)
//     5. Generate authorization code, store with code_challenge
//     6. Redirect to redirect_uri with code and state

// POST /api/v1/oauth/token
//   Body: grant_type=authorization_code, code, redirect_uri, client_id,
//         code_verifier (PKCE)
//   Flow:
//     1. Validate authorization code (not expired, not used)
//     2. Verify code_verifier against stored code_challenge (S256)
//     3. Authenticate client (client_secret for confidential clients)
//     4. Issue access_token (JWT), refresh_token, and id_token (OIDC)
//     5. Mark authorization code as used

// Authorization code storage (from schema):
// authorization_codes table with code_hash, code_params JSONB containing
// code_challenge, code_challenge_method, redirect_uri, nonce, state
```

```typescript
// Token response format:
interface TokenResponse {
  access_token: string;       // JWT signed with RS256
  token_type: 'Bearer';
  expires_in: number;         // seconds
  refresh_token: string;      // opaque token
  id_token?: string;          // JWT with OIDC claims (if openid scope requested)
  scope: string;
}
```

**Testing**:
- `Integration: full authorization code flow with PKCE → access_token + refresh_token`
- `Integration: authorization request without code_challenge → 400 (PKCE mandatory)`
- `Integration: token request with wrong code_verifier → 400`
- `Integration: token request with already-used code → 400`
- `Integration: token request with expired code → 400`
- `Integration: token request with mismatched redirect_uri → 400`
- `Integration: confidential client without client_secret → 401`
- `Integration: public client with code_verifier → token issued`
- `Unit: S256 code_challenge verification correct for known verifier`
- `Unit: authorization code is single-use (second attempt fails)`

---

#### 4.3 — Client credentials grant (machine-to-machine)

**What**: OAuth 2.1 client credentials grant for server-to-server and AI agent authentication.

**Design**:

```typescript
// POST /api/v1/oauth/token
//   Body: grant_type=client_credentials, client_id, client_secret, scope
//   Flow:
//     1. Authenticate client (client_id + client_secret)
//     2. Validate requested scopes against client's allowed scopes
//     3. Issue access_token (JWT) — no refresh_token for client_credentials
//     4. access_token.sub = client_id (no user context)

// For agent clients (is_agent=true in agent_config):
// - access_token includes agent_owner claim linking to human principal
// - scope is restricted to agent's max_scopes
// - shorter TTL (default 15 minutes vs. 1 hour)
```

**Testing**:
- `Integration: client_credentials with valid credentials → access_token`
- `Integration: client_credentials with invalid secret → 401`
- `Integration: client_credentials requesting scope beyond client allowance → 400`
- `Integration: client_credentials for public client → 400 (requires secret)`
- `Integration: agent client → access_token includes agent_owner claim`
- `Integration: agent client → TTL shorter than non-agent default`

---

#### 4.4 — Token refresh and rotation

**What**: Refresh token grant with token rotation per OAuth 2.1 best practices.

**Design**:

```typescript
// POST /api/v1/oauth/token
//   Body: grant_type=refresh_token, refresh_token, client_id, [client_secret]
//   Flow:
//     1. Validate refresh token (not expired, not revoked)
//     2. Authenticate client
//     3. Issue new access_token and NEW refresh_token (rotation)
//     4. Revoke old refresh_token
//     5. If a revoked refresh_token is presented → revoke ALL tokens in family
//        (refresh token reuse detection per OAuth 2.1)

// POST /api/v1/oauth/revoke
//   Body: token, token_type_hint (access_token | refresh_token)
//   Flow: Revoke the specified token
```

**Testing**:
- `Integration: refresh_token grant → new access_token + new refresh_token`
- `Integration: old refresh_token cannot be reused after rotation`
- `Integration: reuse of old refresh_token → entire token family revoked (security)`
- `Integration: refresh_token with expired token → 400`
- `Integration: POST /api/v1/oauth/revoke with valid token → 200, token invalidated`
- `Unit: refresh_token rotation increments rotation_count`

---

#### 4.5 — JWK management and OIDC discovery

**What**: JSON Web Key generation, rotation, JWKS endpoint, and OpenID Connect Discovery metadata.

**Design**:

```typescript
// GET /.well-known/openid-configuration
// Returns OIDC Discovery document:
{
  "issuer": "https://iam.example.com",
  "authorization_endpoint": "https://iam.example.com/api/v1/oauth/authorize",
  "token_endpoint": "https://iam.example.com/api/v1/oauth/token",
  "userinfo_endpoint": "https://iam.example.com/api/v1/oidc/userinfo",
  "jwks_uri": "https://iam.example.com/.well-known/jwks.json",
  "revocation_endpoint": "https://iam.example.com/api/v1/oauth/revoke",
  "scopes_supported": ["openid", "profile", "email", "groups"],
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "client_credentials", "refresh_token"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "token_endpoint_auth_methods_supported": ["client_secret_post", "client_secret_basic"],
  "code_challenge_methods_supported": ["S256"]
}

// GET /.well-known/jwks.json
// Returns current + previous signing keys for seamless rotation:
{
  "keys": [
    { "kty": "RSA", "kid": "current-key-id", "use": "sig", "alg": "RS256", "n": "...", "e": "AQAB" },
    { "kty": "RSA", "kid": "previous-key-id", "use": "sig", "alg": "RS256", "n": "...", "e": "AQAB" }
  ]
}

// GET /api/v1/oidc/userinfo
// Returns OIDC UserInfo claims for the authenticated user (Bearer token):
{
  "sub": "user-uuid",
  "name": "Jane Smith",
  "given_name": "Jane",
  "family_name": "Smith",
  "email": "jane@example.com",
  "email_verified": true
}
```

JWK rotation: Keys are generated as RS256 2048-bit RSA key pairs. Rotation creates a new key and keeps the previous key active for token verification (grace period). Keys stored in PostgreSQL with `active_from` and `retired_at` timestamps.

**Testing**:
- `Integration: GET /.well-known/openid-configuration → valid OIDC Discovery document`
- `Integration: GET /.well-known/jwks.json → at least one RSA key with RS256`
- `Integration: issued JWT verifiable against JWKS endpoint`
- `Integration: GET /api/v1/oidc/userinfo with valid Bearer token → user claims`
- `Integration: GET /api/v1/oidc/userinfo without token → 401`
- `Unit: JWK rotation creates new key, previous key remains in JWKS`
- `Unit: JWT signed with previous key still validates (grace period)`
- `Unit: JWT signed with retired key → verification fails`

---

## Phase 5: FIDO2/WebAuthn Passwordless Authentication

### Purpose
Implement WebAuthn (W3C Level 2) for phishing-resistant passwordless authentication, meeting NIST SP 800-63B AAL2 requirements. After this phase, users can register security keys and passkeys, authenticate without passwords, and achieve AAL2 with a single phishing-resistant authenticator.

### Tasks

#### 5.1 — WebAuthn registration ceremony

**What**: WebAuthn credential registration flow using @simplewebauthn/server, supporting both platform authenticators (passkeys) and roaming authenticators (security keys).

**Design**:

```typescript
// API endpoints
// POST /api/v1/users/:id/mfa/webauthn/register/options
//   Response: PublicKeyCredentialCreationOptions (challenge, rp, user, pubKeyCredParams, etc.)
//
// POST /api/v1/users/:id/mfa/webauthn/register/verify
//   Request: AuthenticatorAttestationResponse from browser
//   Response: { credentialId, credentialType: 'webauthn', displayName }

// Credential storage in credentials.credential_data JSONB:
// {
//   "credential_id": "base64url...",
//   "public_key": "base64...",
//   "sign_count": 0,
//   "aaguid": "00000000-0000-0000-0000-000000000000",
//   "transports": ["internal", "usb"],
//   "attestation_type": "none",
//   "user_verified": true,
//   "backup_eligible": true,    // passkey/syncable
//   "backup_state": true        // currently synced
// }
```

WebAuthn Relying Party (RP) configuration:
- `rpName`: from tenant displayName
- `rpID`: from `COOKIE_DOMAIN` config
- `attestation`: "none" (direct attestation not required for most use cases)
- `authenticatorSelection.residentKey`: "preferred" (support passkeys)
- `authenticatorSelection.userVerification`: "preferred"

**Testing**:
- `Integration: register/options → valid PublicKeyCredentialCreationOptions with challenge`
- `Integration: register/verify with valid attestation → credential stored, 200`
- `Integration: register/verify with invalid attestation → 400`
- `Integration: registered credential has AAL level 2`
- `Unit: challenge is unique per registration attempt`
- `Unit: challenge expires after 5 minutes`
- `Fixture-based: verify against known WebAuthn attestation fixtures`

---

#### 5.2 — WebAuthn authentication ceremony

**What**: WebAuthn authentication flow for login and step-up authentication.

**Design**:

```typescript
// API endpoints
// POST /api/v1/auth/webauthn/authenticate/options
//   Request: { username?: string }
//   Response: PublicKeyCredentialRequestOptions (challenge, allowCredentials, etc.)
//
// POST /api/v1/auth/webauthn/authenticate/verify
//   Request: AuthenticatorAssertionResponse from browser
//   Response: { session: { token, expiresAt }, user: { ... } }

// Authentication flow:
// 1. Generate challenge, store in Redis with 5-minute TTL
// 2. If username provided → include allowCredentials with user's registered credential IDs
// 3. If no username (passkey/discoverable credential) → empty allowCredentials
// 4. Browser performs WebAuthn ceremony
// 5. Server verifies assertion: signature, challenge, origin, sign_count
// 6. Update sign_count in stored credential (replay detection)
// 7. Issue AAL2 session
```

**Testing**:
- `Integration: full WebAuthn authentication flow with platform authenticator mock → AAL2 session`
- `Integration: sign_count mismatch (replay attack) → authentication rejected`
- `Integration: challenge reuse → rejected`
- `Integration: WebAuthn as MFA step-up → session AAL upgraded from 1 to 2`
- `Unit: sign_count incremented after successful authentication`
- `Fixture-based: verify assertion against known test vectors`

---

## Phase 6: SAML 2.0 & Federation

### Purpose
Implement SAML 2.0 Identity Provider functionality and upstream Identity Provider federation (OIDC and SAML). After this phase, the platform can federate with enterprise identity providers (Okta, Azure AD, Google Workspace) and act as a SAML IdP for legacy enterprise applications.

### Tasks

#### 6.1 — SAML 2.0 Identity Provider

**What**: SAML 2.0 IdP endpoints: metadata, SSO (SP-initiated and IdP-initiated), Single Logout (SLO), and assertion signing.

**Design**:

```typescript
// API endpoints
// GET  /api/v1/saml/metadata                    → SAML IdP metadata XML
// GET  /api/v1/saml/sso                         → SP-initiated SSO (redirect binding)
// POST /api/v1/saml/sso                         → SP-initiated SSO (POST binding)
// POST /api/v1/saml/slo                         → Single Logout
// GET  /api/v1/saml/slo                         → Single Logout (redirect binding)

// SP-initiated SSO flow:
// 1. SP sends AuthnRequest (via redirect or POST)
// 2. Parse and validate AuthnRequest (issuer, signature, destination)
// 3. Look up oauth_client by saml_config.entity_id
// 4. If user not authenticated → redirect to login
// 5. Build SAML Response with signed Assertion containing:
//    - NameID (from saml_config.name_id_format: email, persistent, transient)
//    - Attributes mapped from user profile (configurable per SP)
//    - Conditions (audience, notBefore, notOnOrAfter)
// 6. POST Response to SP's ACS URL
```

SAML assertions signed with the IdP's X.509 certificate (same key used for JWK, or dedicated SAML signing key).

**Testing**:
- `Integration: GET /api/v1/saml/metadata → valid SAML IdP metadata XML`
- `Integration: SP-initiated SSO with valid AuthnRequest → SAML Response with assertion`
- `Integration: SAML assertion contains correct NameID and mapped attributes`
- `Integration: SAML assertion signature validates with IdP certificate`
- `Integration: AuthnRequest with unknown issuer → 400`
- `Fixture-based: parse known AuthnRequest XML fixtures`
- `Unit: NameID format mapping (email → user.email, persistent → user.id)`

---

#### 6.2 — Upstream Identity Provider federation (OIDC)

**What**: Federated login via upstream OIDC identity providers (Google, GitHub, Azure AD, generic OIDC).

**Design**:

```typescript
// Database: identity_providers table + federated_identities table
// (from Data Model Suggestion 3)

// API endpoints
// POST   /api/v1/identity-providers               → configure upstream IdP
// GET    /api/v1/identity-providers               → list configured IdPs
// GET    /api/v1/auth/federated/:idpId/authorize   → redirect to upstream IdP
// GET    /api/v1/auth/federated/:idpId/callback    → handle callback from upstream IdP

// OIDC Federation flow:
// 1. Admin configures upstream IdP (issuer_url, client_id, client_secret)
// 2. User clicks "Sign in with Google" → redirect to IdP's authorization endpoint
// 3. IdP redirects back with authorization code
// 4. Exchange code for tokens at IdP's token endpoint
// 5. Fetch userinfo or parse id_token for user claims
// 6. Look up federated_identities for (idp_id, idp_user_id)
//    - If found → log in the linked local user
//    - If not found → JIT provision a new local user and create the link
// 7. Issue local session
```

**Testing**:
- `Integration (mocked IdP): OIDC federation flow → local session created`
- `Integration (mocked IdP): JIT provisioning creates user from IdP claims`
- `Integration (mocked IdP): returning user → existing account linked, session created`
- `Integration (mocked IdP): IdP returns error → 401 with error details`
- `Integration: POST /api/v1/identity-providers with OIDC config → IdP created`
- `Unit: attribute mapping transforms IdP claims to local user fields`

---

#### 6.3 — Upstream SAML federation

**What**: Federated login via upstream SAML 2.0 identity providers.

**Design**:

```typescript
// API endpoints
// GET  /api/v1/auth/federated/:idpId/saml/metadata   → SP metadata for this IAM platform
// POST /api/v1/auth/federated/:idpId/saml/acs         → Assertion Consumer Service (receives SAML Response)

// SAML federation flow:
// 1. Admin configures upstream SAML IdP (entity_id, sso_url, certificate)
// 2. User clicks "Sign in with Corporate SSO"
// 3. Generate AuthnRequest, redirect to IdP's SSO URL
// 4. IdP authenticates user, posts SAML Response to ACS
// 5. Validate Response signature against IdP's certificate
// 6. Extract NameID and attributes from assertion
// 7. JIT provision or link existing user (same as OIDC federation)
// 8. Issue local session
```

**Testing**:
- `Integration (mocked IdP): SAML federation flow → local session created`
- `Integration: SAML Response with invalid signature → rejected`
- `Integration: SAML Response with expired assertion → rejected`
- `Fixture-based: parse known SAML Response XML fixtures`
- `Unit: X.509 certificate validation for assertion signature`

---

## Phase 7: SCIM 2.0 Provisioning API

### Purpose
Implement the SCIM 2.0 (RFC 7643/7644) provisioning API so external directories (Okta, Azure AD, Google Workspace) can automatically provision and deprovision users and groups. After this phase, enterprises can connect their existing identity provider and have user lifecycle events automatically synchronize.

### Tasks

#### 7.1 — SCIM 2.0 User endpoints

**What**: SCIM 2.0 User resource endpoints per RFC 7644.

**Design**:

```typescript
// SCIM 2.0 API endpoints (per RFC 7644):
// POST   /scim/v2/Users                 → create user
// GET    /scim/v2/Users                 → list users (with SCIM filter, pagination)
// GET    /scim/v2/Users/:id             → get user
// PUT    /scim/v2/Users/:id             → replace user
// PATCH  /scim/v2/Users/:id             → modify user (SCIM PATCH operations)
// DELETE /scim/v2/Users/:id             → delete/deprovision user

// SCIM User resource maps to internal User model:
// - schemas: ["urn:ietf:params:scim:schemas:core:2.0:User",
//             "urn:ietf:params:scim:schemas:extension:enterprise:2.0:User"]
// - userName → users.username
// - name.givenName → users.givenName
// - name.familyName → users.familyName
// - emails → users.emails JSONB
// - phoneNumbers → users.phoneNumbers JSONB
// - active → users.isActive
// - Enterprise extension → users.enterpriseProfile JSONB

// SCIM filter support (minimal viable):
// - eq, ne, co, sw, gt, ge, lt, le operators
// - userName, email, displayName, active attributes
// - Example: filter=userName eq "jane"
```

Authentication for SCIM endpoints: Bearer token (long-lived API token generated per directory connection).

**Testing**:
- `Integration: POST /scim/v2/Users with SCIM User JSON → 201 with SCIM User response`
- `Integration: GET /scim/v2/Users?filter=userName eq "jane" → filtered results`
- `Integration: PATCH /scim/v2/Users/:id with SCIM PATCH → attributes updated`
- `Integration: DELETE /scim/v2/Users/:id → user deprovisioned`
- `Integration: SCIM response includes correct schemas array`
- `Integration: SCIM response includes meta.resourceType, meta.created, meta.lastModified`
- `Unit: SCIM filter parser handles eq, co, sw operators`
- `Unit: SCIM PATCH operation 'replace' on multi-valued attribute works`
- `Unit: SCIM PATCH operation 'add' on emails array appends correctly`

---

#### 7.2 — SCIM 2.0 Group endpoints

**What**: SCIM 2.0 Group resource endpoints per RFC 7644.

**Design**:

```typescript
// SCIM 2.0 Group endpoints:
// POST   /scim/v2/Groups                → create group
// GET    /scim/v2/Groups                → list groups
// GET    /scim/v2/Groups/:id            → get group (with members)
// PUT    /scim/v2/Groups/:id            → replace group
// PATCH  /scim/v2/Groups/:id            → modify group (add/remove members)
// DELETE /scim/v2/Groups/:id            → delete group

// SCIM Group members are serialized as:
// { "value": "user-uuid", "display": "Jane Smith", "$ref": "/scim/v2/Users/user-uuid" }
```

**Testing**:
- `Integration: POST /scim/v2/Groups with members → group created with members linked`
- `Integration: PATCH /scim/v2/Groups/:id to add member → member added`
- `Integration: PATCH /scim/v2/Groups/:id to remove member → member removed`
- `Integration: GET /scim/v2/Groups/:id → includes members array`
- `Unit: SCIM Group PATCH 'add' to members array`
- `Unit: SCIM Group PATCH 'remove' from members array by value filter`

---

#### 7.3 — SCIM service provider configuration

**What**: SCIM 2.0 ServiceProviderConfig, Schemas, and ResourceTypes discovery endpoints.

**Design**:

```typescript
// GET /scim/v2/ServiceProviderConfig → supported features
// GET /scim/v2/Schemas              → User and Group schemas
// GET /scim/v2/ResourceTypes        → User and Group resource type definitions

// ServiceProviderConfig response:
{
  "schemas": ["urn:ietf:params:scim:schemas:core:2.0:ServiceProviderConfig"],
  "patch": { "supported": true },
  "bulk": { "supported": false },
  "filter": { "supported": true, "maxResults": 200 },
  "changePassword": { "supported": true },
  "sort": { "supported": false },
  "etag": { "supported": false },
  "authenticationSchemes": [
    { "type": "oauthbearertoken", "name": "OAuth Bearer Token" }
  ]
}
```

**Testing**:
- `Integration: GET /scim/v2/ServiceProviderConfig → valid SCIM config`
- `Integration: GET /scim/v2/Schemas → User and Group schemas`
- `Integration: GET /scim/v2/ResourceTypes → User and Group resource types`

---

## Phase 8: Admin Dashboard & Self-Service Portal

### Purpose
Build the web-based admin dashboard for managing users, groups, roles, applications, policies, and audit logs, plus the end-user self-service portal for profile management, MFA enrollment, and session management. After this phase, the platform has a complete web interface for both administrators and end users.

### Tasks

#### 8.1 — Next.js application scaffold

**What**: Initialize the Next.js 15 App Router application with authentication, layout, and API client.

**Design**:

```typescript
// apps/web/src/lib/api-client.ts
export class IAMApiClient {
  constructor(private baseUrl: string, private sessionToken?: string) {}

  async getUsers(params?: ListParams): Promise<PaginatedResult<User>>;
  async getUser(id: string): Promise<User>;
  async createUser(data: CreateUserRequest): Promise<User>;
  async updateUser(id: string, data: UpdateUserRequest): Promise<User>;
  // ... other resource methods
}

// apps/web/src/app/layout.tsx
// Root layout with:
// - Session validation middleware (redirect to login if unauthenticated)
// - Tenant context from session
// - Navigation sidebar (admin) or top bar (self-service)
```

Shadcn/ui components: Button, Input, Select, Dialog, Table, Card, Badge, DropdownMenu, Command, Sheet.

**Testing**:
- `E2E: unauthenticated visit to /dashboard → redirects to /login`
- `E2E: authenticated visit to /dashboard → renders dashboard layout`
- `Unit: IAMApiClient.getUsers sends correct headers and query params`

---

#### 8.2 — User management pages

**What**: Admin pages for listing, viewing, creating, editing, and deprovisioning users.

**Design**:

Pages:
- `/dashboard/users` — paginated user list with search, filter by state, bulk actions
- `/dashboard/users/[id]` — user detail view with tabs (Profile, Roles, Groups, Sessions, Audit)
- `/dashboard/users/new` — create user form
- `/dashboard/users/[id]/edit` — edit user form

Components:
- `UserTable` — sortable, filterable data table with column visibility
- `UserForm` — form for user creation/editing with SCIM field validation
- `UserRolesTab` — manage role assignments with add/remove
- `UserGroupsTab` — show group memberships
- `UserSessionsTab` — list active sessions with revoke action
- `UserAuditTab` — recent audit events for the user

**Testing**:
- `E2E: navigate to /dashboard/users → user list renders with data`
- `E2E: search for user by email → filtered results`
- `E2E: click user row → navigates to user detail`
- `E2E: create new user → user appears in list`
- `E2E: suspend user → identity state changes to 'suspended'`
- `E2E: assign role to user → role appears in user's role tab`

---

#### 8.3 — Application and policy management pages

**What**: Admin pages for managing OAuth clients (applications) and authentication policies.

**Design**:

Pages:
- `/dashboard/applications` — list registered applications
- `/dashboard/applications/[id]` — application detail (OAuth config, SAML config, agent config)
- `/dashboard/applications/new` — register new application wizard
- `/dashboard/policies` — list authentication policies with priority ordering
- `/dashboard/policies/[id]` — policy detail with condition builder

Components:
- `PolicyConditionBuilder` — visual editor for policy conditions (IP ranges, countries, device types)
- `ApplicationWizard` — step-by-step registration (client type → OAuth config → scopes)

**Testing**:
- `E2E: register new OAuth application → client_id and secret displayed`
- `E2E: create authentication policy with MFA required for non-US IPs`
- `E2E: reorder policy priorities via drag-and-drop`

---

#### 8.4 — Self-service portal

**What**: End-user portal for profile management, MFA enrollment, active session management, and password changes.

**Design**:

Pages:
- `/account/profile` — view and edit own profile
- `/account/security` — change password, manage MFA credentials
- `/account/security/totp/enroll` — TOTP enrollment with QR code
- `/account/security/webauthn/register` — WebAuthn credential registration
- `/account/sessions` — view and revoke own active sessions

**Testing**:
- `E2E: user edits own display name → updated in profile`
- `E2E: user enrolls TOTP → QR code displayed, recovery codes shown`
- `E2E: user registers WebAuthn credential → credential appears in security page`
- `E2E: user revokes a session → session removed from list`
- `E2E: user changes password → old sessions revoked, new session created`

---

#### 8.5 — Audit log viewer

**What**: Admin page for searching and filtering audit events with real-time streaming.

**Design**:

Pages:
- `/dashboard/audit` — searchable audit log with filters for event type, actor, target, date range
- Export: CSV download of filtered audit events

Components:
- `AuditEventTable` — time-ordered event list with expandable details
- `AuditFilters` — filter bar for event type, category, actor, outcome, date range
- `AuditEventDetail` — expanded view of event details JSONB

**Testing**:
- `E2E: navigate to /dashboard/audit → recent events displayed`
- `E2E: filter by event type 'user.login_failed' → only login failures shown`
- `E2E: filter by date range → events within range shown`
- `E2E: expand event row → details JSON displayed`

---

## Phase 9: Identity Governance & Access Reviews

### Purpose
Implement access certification campaigns, separation-of-duties policy enforcement, and access review workflows. After this phase, compliance teams can run periodic access reviews, enforce SoD policies, and produce audit evidence for SOX, HIPAA, and ISO 27001 compliance.

### Tasks

#### 9.1 — Certification campaign management

**What**: API and UI for creating, launching, and managing access certification campaigns.

**Design**:

```typescript
// packages/shared/src/types/governance.ts
export interface CertificationCampaign {
  id: string;
  tenantId: string;
  name: string;
  campaignType: 'user_access' | 'role_membership' | 'application_access'
    | 'privileged_access' | 'separation_of_duties';
  status: 'draft' | 'active' | 'in_review' | 'completed' | 'cancelled';
  ownerId: string;
  config: CampaignConfig;
  completedAt: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface CampaignConfig {
  scope: {
    applications?: string[];
    roles?: string[];
    groups?: string[];
  };
  schedule: {
    startsAt: string;
    dueAt: string;
  };
  reminders: {
    frequency: string;    // e.g., '3d'
    escalateAfter: string; // e.g., '7d'
  };
  autoRevokeOnExpiry: boolean;
}

export interface CertificationItem {
  id: string;
  campaignId: string;
  userId: string;
  reviewerId: string | null;
  accessDetails: {
    type: 'role' | 'group' | 'application';
    id: string;
    name: string;
    grantedAt: string;
    lastUsed: string | null;
    usageCount30d: number;
  };
  decision: 'approve' | 'revoke' | 'delegate' | 'pending' | null;
  decisionReason: string | null;
  riskScore: number | null;
  aiRecommendation: AIRecommendation | null;
  decidedAt: string | null;
}

export interface AIRecommendation {
  action: 'approve' | 'revoke';
  confidence: number;       // 0.0 - 1.0
  reason: string;
  similarUsersRevokedPct: number;
}

// API endpoints
// POST   /api/v1/governance/campaigns                      → create campaign
// GET    /api/v1/governance/campaigns                      → list campaigns
// GET    /api/v1/governance/campaigns/:id                  → get campaign with stats
// POST   /api/v1/governance/campaigns/:id/launch           → launch (draft → active)
// POST   /api/v1/governance/campaigns/:id/complete         → complete campaign
// GET    /api/v1/governance/campaigns/:id/items            → list review items
// PATCH  /api/v1/governance/campaigns/:id/items/:itemId    → submit decision
```

When a campaign is launched, the system generates certification items by scanning the scope (e.g., all role assignments for users in the specified groups) and assigns reviewers (manager or campaign owner).

**Testing**:
- `Integration: POST create campaign → 201 with draft status`
- `Integration: POST launch campaign → status changes to 'active', items generated`
- `Integration: items generated match scope (users with roles in specified groups)`
- `Integration: PATCH item with decision 'revoke' → role revoked from user`
- `Integration: PATCH item with decision 'approve' → no change to access`
- `Integration: complete campaign → status 'completed', audit events recorded`
- `Unit: item generation for role_membership campaign type scans correct data`

---

#### 9.2 — Separation-of-duties policies

**What**: API for defining conflicting role pairs and detecting violations.

**Design**:

```typescript
// API endpoints
// POST   /api/v1/governance/sod-policies              → create SoD policy
// GET    /api/v1/governance/sod-policies              → list policies
// GET    /api/v1/governance/sod-violations             → detect current violations
// POST   /api/v1/governance/sod-policies/:id/check     → check specific user against policy

// Violation detection query:
// For each active SoD policy (role_a, role_b):
//   Find all users who have BOTH role_a AND role_b
//   (considering both direct and group-inherited assignments)
```

SoD enforcement modes:
- `info`: Log violation, allow assignment
- `warning`: Log violation, allow assignment with justification
- `block`: Prevent role assignment that would create violation

**Testing**:
- `Integration: assign role_b when user already has role_a with 'block' SoD → 409`
- `Integration: assign role_b when user already has role_a with 'warning' SoD → 200 with warning`
- `Integration: GET /api/v1/governance/sod-violations → lists all current violations`
- `Unit: violation detection finds users with both conflicting roles via group inheritance`

---

#### 9.3 — Governance dashboard pages

**What**: Admin UI for managing certification campaigns, reviewing access items, and viewing SoD violations.

**Design**:

Pages:
- `/dashboard/governance` — overview with active campaigns, pending reviews, SoD violations
- `/dashboard/governance/campaigns` — campaign list with status filters
- `/dashboard/governance/campaigns/[id]` — campaign detail with item list and bulk actions
- `/dashboard/governance/campaigns/[id]/review` — reviewer interface for deciding on items
- `/dashboard/governance/sod` — SoD policy list and violation dashboard

Components:
- `CertificationItemReview` — card showing user, access details, AI recommendation, approve/revoke buttons
- `SoDViolationTable` — violations with user, conflicting roles, severity

**Testing**:
- `E2E: create certification campaign → visible in campaign list`
- `E2E: launch campaign → items appear in review interface`
- `E2E: approve item → item status updates to 'approved'`
- `E2E: revoke item → user's role removed, status updates to 'revoked'`
- `E2E: SoD violation dashboard shows detected violations`

---

## Phase 10: Directory Sync & Provisioning Workers

### Purpose
Implement outbound provisioning workers and directory synchronisation for upstream identity sources (LDAP, Azure AD, Google Workspace). After this phase, the platform can sync user and group data from external directories and push provisioning events to downstream applications via SCIM.

### Tasks

#### 10.1 — Directory connection management

**What**: API for configuring directory connections and monitoring sync status.

**Design**:

```typescript
// API endpoints
// POST   /api/v1/directories                  → create directory connection
// GET    /api/v1/directories                  → list connections with sync status
// PATCH  /api/v1/directories/:id              → update connection config
// POST   /api/v1/directories/:id/sync         → trigger manual sync
// GET    /api/v1/directories/:id/events        → list provisioning events

// Directory connection types:
// - scim: outbound SCIM provisioning to downstream apps
// - ldap: inbound sync from LDAP/Active Directory
// - azure_ad: inbound sync from Azure AD via Microsoft Graph
// - google_workspace: inbound sync from Google Workspace
```

**Testing**:
- `Integration: POST create SCIM directory connection → 201`
- `Integration: POST trigger sync → sync job enqueued`
- `Integration: GET events → provisioning event history`
- `Unit: connection config validated per provider type`

---

#### 10.2 — Provisioning worker (BullMQ)

**What**: Background worker that processes provisioning events (user created, updated, deprovisioned) and pushes changes to connected downstream applications via SCIM.

**Design**:

```typescript
// packages/api/src/workers/provisioning.worker.ts
import { Worker, Job } from 'bullmq';

// Job types:
// - scim.user.create   → POST /scim/v2/Users to downstream
// - scim.user.update   → PATCH /scim/v2/Users/:id to downstream
// - scim.user.delete   → DELETE /scim/v2/Users/:id to downstream
// - scim.group.update  → PATCH /scim/v2/Groups/:id to downstream

// Worker processes jobs with:
// - 3 retries with exponential backoff
// - Dead-letter queue for persistent failures
// - Provisioning event logged with status (success/failure) and response
```

**Testing**:
- `Integration (mocked downstream): user created → SCIM POST sent to downstream`
- `Integration (mocked downstream): downstream returns 201 → event status 'success'`
- `Integration (mocked downstream): downstream returns 500 → retry with backoff`
- `Integration (mocked downstream): 3 retries exhausted → event status 'failure', dead-lettered`
- `Unit: SCIM user create payload matches RFC 7643 User schema`

---

#### 10.3 — Token cleanup worker

**What**: Scheduled worker that purges expired tokens, sessions, and authorization codes.

**Design**:

```typescript
// packages/api/src/workers/token-cleanup.worker.ts
// Runs every 15 minutes via BullMQ repeatable job:
// 1. DELETE FROM authorization_codes WHERE expires_at < now() - INTERVAL '1 hour'
// 2. DELETE FROM access_tokens WHERE expires_at < now() - INTERVAL '1 day'
// 3. DELETE FROM refresh_tokens WHERE expires_at < now() - INTERVAL '7 days'
// 4. UPDATE sessions SET is_active = false WHERE expires_at < now() AND is_active = true
// 5. Log cleanup counts
```

**Testing**:
- `Integration: expired authorization codes deleted after cleanup`
- `Integration: expired access tokens deleted after cleanup`
- `Integration: active sessions with past expiry marked inactive`
- `Unit: cleanup queries use correct retention intervals`

---

## Phase 11: AI-Augmented Features

### Purpose
Implement AI-driven role mining, access review recommendations, and anomaly detection. These are the key differentiators identified in the research as underserved by current IAM platforms. After this phase, the platform offers AI-powered governance automation and continuous security monitoring.

### Tasks

#### 11.1 — Risk scoring engine

**What**: Calculate risk scores for users and access assignments based on behavioural signals, access patterns, and contextual factors.

**Design**:

```typescript
// packages/api/src/services/risk-scoring.service.ts
export interface RiskFactors {
  unusedAccessDays: number;        // days since role/permission was last exercised
  accessBreadth: number;           // number of distinct permissions held
  privilegedRoleCount: number;     // count of admin/system roles
  loginAnomalyScore: number;       // deviation from normal login patterns
  failedLoginCount30d: number;
  locationDeviation: boolean;      // login from unusual location
  deviceDeviation: boolean;        // login from unknown device
}

export function calculateRiskScore(factors: RiskFactors): number {
  // Weighted sum producing 0-100 score
  // Weights tuned based on IAM security research
  let score = 0;
  score += Math.min(factors.unusedAccessDays / 90, 1) * 25;   // stale access
  score += Math.min(factors.accessBreadth / 50, 1) * 15;      // over-provisioned
  score += Math.min(factors.privilegedRoleCount / 5, 1) * 20; // privilege concentration
  score += factors.loginAnomalyScore * 20;                      // behavioural anomaly
  score += Math.min(factors.failedLoginCount30d / 10, 1) * 10; // brute force signal
  score += (factors.locationDeviation ? 5 : 0);
  score += (factors.deviceDeviation ? 5 : 0);
  return Math.round(Math.min(score, 100));
}
```

Risk scores are computed nightly via a BullMQ scheduled job and stored on certification items and user profiles.

**Testing**:
- `Unit: user with 90+ days unused access → risk score >= 25`
- `Unit: user with many privileged roles → risk score elevated`
- `Unit: user with normal patterns → low risk score`
- `Unit: risk score capped at 100`
- `Integration: risk scores populated on certification items when campaign launched`

---

#### 11.2 — Access review AI recommendations

**What**: Generate AI-powered approve/revoke recommendations for certification campaign items.

**Design**:

```typescript
// packages/api/src/services/ai-recommendation.service.ts
export class AIRecommendationService {
  async generateRecommendations(
    campaignId: string
  ): Promise<void> {
    const items = await getCertificationItems(campaignId);
    for (const item of items) {
      const riskScore = await this.riskScoringService.calculateForItem(item);
      const recommendation = this.generateRecommendation(item, riskScore);
      await updateCertificationItem(item.id, {
        riskScore,
        aiRecommendation: recommendation,
      });
    }
  }

  private generateRecommendation(
    item: CertificationItem,
    riskScore: number
  ): AIRecommendation {
    // Rule-based recommendations (v1, no LLM dependency):
    // - Access unused for 90+ days → recommend revoke (confidence: 0.85)
    // - User deprovisioned but access remains → recommend revoke (confidence: 0.95)
    // - Access used within 30 days and low risk → recommend approve (confidence: 0.80)
    // - Medium risk → recommend approve with low confidence (0.50) for human review
  }
}
```

V1 uses rule-based recommendations. LLM-based natural language summaries and reasoning are a v2 enhancement.

**Testing**:
- `Unit: unused access 90+ days → recommendation 'revoke' with confidence >= 0.80`
- `Unit: recently used access → recommendation 'approve'`
- `Unit: deprovisioned user with remaining access → recommendation 'revoke' high confidence`
- `Integration: launch campaign → items have AI recommendations populated`
- `Unit: medium risk items → lower confidence recommendations (human review needed)`

---

#### 11.3 — Authentication anomaly detection

**What**: Detect anomalous authentication events by comparing login patterns against user baselines.

**Design**:

```typescript
// packages/api/src/services/anomaly-detection.service.ts
export class AnomalyDetectionService {
  async evaluateLogin(event: LoginEvent): Promise<AnomalyResult> {
    const baseline = await this.getUserBaseline(event.userId);
    const anomalies: AnomalySignal[] = [];

    // Geographic anomaly: login from country never seen before
    if (!baseline.knownCountries.includes(event.country)) {
      anomalies.push({ type: 'new_country', severity: 'high', detail: event.country });
    }

    // Impossible travel: login from location X, then location Y faster than travel time
    if (baseline.lastLoginAt && baseline.lastLoginCountry) {
      const timeDiffHours = (event.timestamp - baseline.lastLoginAt) / 3600000;
      if (timeDiffHours < 2 && event.country !== baseline.lastLoginCountry) {
        anomalies.push({ type: 'impossible_travel', severity: 'critical' });
      }
    }

    // Time anomaly: login outside normal hours
    const hour = new Date(event.timestamp).getUTCHours();
    if (!baseline.normalHours.includes(hour)) {
      anomalies.push({ type: 'unusual_time', severity: 'medium' });
    }

    // New device
    if (!baseline.knownDevices.includes(event.deviceFingerprint)) {
      anomalies.push({ type: 'new_device', severity: 'medium' });
    }

    return { anomalies, riskLevel: this.aggregateRisk(anomalies) };
  }
}
```

Anomaly results are stored in audit event details and can trigger step-up authentication (require MFA for anomalous logins).

**Testing**:
- `Unit: login from new country → 'new_country' anomaly detected`
- `Unit: impossible travel (2 countries, 30 minutes) → 'impossible_travel' anomaly`
- `Unit: login at normal time → no time anomaly`
- `Unit: login at 3am when baseline is 9-5 → 'unusual_time' anomaly`
- `Integration: anomalous login → audit event includes anomaly details`
- `Integration: high-risk anomaly → step-up MFA required`

---

## Phase 12: Agent Identity & Advanced Features

### Purpose
Implement non-human identity management for AI agents and MCP tool-calling pipelines, plus advanced features including self-service password reset, application integrations, and API documentation. This phase addresses the emerging agentic identity gap identified in the research as a key differentiator.

### Tasks

#### 12.1 — Agent identity management

**What**: API for registering, managing, and auditing AI agent identities that act on behalf of human principals.

**Design**:

```typescript
// API endpoints
// POST   /api/v1/agents                         → register agent
// GET    /api/v1/agents                         → list agents
// GET    /api/v1/agents/:id                     → get agent with activity log
// PATCH  /api/v1/agents/:id                     → update agent config
// POST   /api/v1/agents/:id/revoke              → revoke agent credentials
// GET    /api/v1/agents/:id/actions              → list actions taken by agent

// Agent registration creates an oauth_client with agent_config:
// {
//   "is_agent": true,
//   "owner_id": "uuid-of-human-principal",
//   "agent_type": "mcp_tool" | "rpa_bot" | "ai_assistant",
//   "max_scopes": ["read:users", "write:documents"],
//   "auto_revoke_on_anomaly": true,
//   "ttl_seconds": 900  // 15 minute default for agent tokens
// }

// Agent action auditing:
// Every API call made with an agent's access token records:
// - agent_id, action, target resource, on_behalf_of (human principal)
// - Full audit trail enabling "what did this agent do?"
```

Agent tokens use the OAuth 2.1 client_credentials grant with restricted scopes. Token TTL is shorter than human tokens (default 15 minutes vs. 1 hour).

**Testing**:
- `Integration: POST /api/v1/agents → agent registered as oauth_client with is_agent=true`
- `Integration: agent token request → access_token with agent_owner claim`
- `Integration: agent API call → audit event with actorType='agent' and on_behalf_of`
- `Integration: agent scope exceeds max_scopes → 403`
- `Integration: POST /api/v1/agents/:id/revoke → agent tokens invalidated`
- `Unit: agent token TTL defaults to 900 seconds`

---

#### 12.2 — Self-service password reset

**What**: Email-based password reset flow with rate limiting and audit logging.

**Design**:

```typescript
// API endpoints
// POST /api/v1/auth/forgot-password
//   Request: { email: string, tenantSlug: string }
//   Response: 200 (always, to prevent email enumeration)
//
// POST /api/v1/auth/reset-password
//   Request: { token: string, newPassword: string }
//   Response: 200

// Flow:
// 1. User submits email
// 2. If email exists → generate reset token (32 bytes, stored as SHA-256 hash)
// 3. Send email with reset link (token TTL: 1 hour)
// 4. User clicks link, submits new password
// 5. Verify token, update password, revoke all existing sessions
// 6. Audit: user.password_reset

// Rate limit: 3 reset requests per email per hour
```

**Testing**:
- `Integration: forgot-password with valid email → 200, reset token stored`
- `Integration: forgot-password with invalid email → 200 (no enumeration)`
- `Integration: reset-password with valid token → password changed, sessions revoked`
- `Integration: reset-password with expired token → 400`
- `Integration: reset-password with already-used token → 400`
- `Integration: 4th reset request in 1 hour → 429`

---

#### 12.3 — OpenAPI documentation and SDK generation

**What**: Auto-generated OpenAPI 3.1 specification published at /docs with Scalar UI, plus SDK generation guidance.

**Design**:

```typescript
// @fastify/swagger configuration:
{
  openapi: {
    openapi: '3.1.0',
    info: {
      title: 'IAM Platform API',
      version: '1.0.0',
      description: 'Identity & Access Management API',
    },
    servers: [{ url: 'https://iam.example.com' }],
    components: {
      securitySchemes: {
        bearerAuth: { type: 'http', scheme: 'bearer', bearerFormat: 'JWT' },
        cookieAuth: { type: 'apiKey', in: 'cookie', name: '__iam_session' },
      },
    },
  },
}

// @scalar/fastify-api-reference for interactive API documentation at /docs
```

**Testing**:
- `Integration: GET /docs → Scalar API reference page renders`
- `Integration: GET /openapi.json → valid OpenAPI 3.1 specification`
- `Unit: all routes have JSON Schema request/response definitions`
- `Unit: OpenAPI spec includes all security schemes`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Core Schema            ─── required by everything
    │
Phase 2: User Management & RBAC             ─── requires Phase 1
    │
Phase 3: Authentication Engine              ─── requires Phase 2
    │
    ├── Phase 4: OAuth 2.1 Authorization Server  ─── requires Phase 3
    │       │
    │       ├── Phase 5: FIDO2/WebAuthn          ─── requires Phase 3, can parallel Phase 4
    │       │
    │       └── Phase 7: SCIM 2.0 Provisioning   ─── requires Phase 2, can parallel Phases 4-6
    │
    ├── Phase 6: SAML 2.0 & Federation          ─── requires Phase 3
    │
    └── Phase 8: Admin Dashboard & Self-Service  ─── requires Phases 2-4
         │
         ├── Phase 9: Identity Governance        ─── requires Phases 2, 8
         │
         ├── Phase 10: Directory Sync & Workers  ─── requires Phases 2, 7
         │
         ├── Phase 11: AI-Augmented Features     ─── requires Phases 3, 9
         │
         └── Phase 12: Agent Identity & Advanced ─── requires Phases 4, 11
```

**Parallelism opportunities:**
- Phases 4, 5, and 6 can be developed concurrently after Phase 3 (they all build on the authentication engine but are independent of each other)
- Phase 7 (SCIM) can be developed in parallel with Phases 4-6 (it depends only on Phase 2)
- Phases 9, 10, 11 can be partially parallelised after Phase 8

---

## Definition of Done (per phase)

1. All tasks in the phase are implemented.
2. All unit tests pass (`pnpm test`).
3. All integration tests pass (with test database and Redis).
4. ESLint passes with zero errors (`pnpm lint`).
5. TypeScript strict mode compiles without errors (`pnpm build`).
6. Prettier formatting applied (`pnpm format --check`).
7. Docker build succeeds (`docker compose build`).
8. Feature works end-to-end via manual verification or E2E test.
9. New database migrations created and tested (apply + rollback).
10. New API endpoints appear in auto-generated OpenAPI spec.
11. New config options documented in `.env.example`.
12. Audit events recorded for all state-changing operations.
13. No secrets or credentials committed to version control.
14. Rate limiting applied to authentication and token endpoints.
