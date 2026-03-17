# Backend ERP Architecture Rules

Architectural guidelines and constraints for the ERP backend (NestJS monorepo).

---

## Architecture Principles

### Design Constraints

- Follow **Module-Controller-Service** structure for all features
- Use **Repository pattern** for complex query logic
- Use **Joi** for request validation (NOT class-validator)
- Use **class-transformer** (`@Expose`, `@Type`) for response DTOs
- Use **internationalized responses** via nestjs-i18n for all user-facing messages
- Do NOT introduce new architectural patterns that conflict with existing ones

### Preferred Patterns

| Pattern | Implementation |
|---------|---------------|
| Authentication | Session-based (express-session + connect-pg-simple) |
| Authorization | RBAC with `AuthenticatedGuard` + `PermissionsGuard` |
| Validation | Joi schemas in `*.validator.ts` |
| Response format | `SuccessResponse` / `SuccessListResponse` / `ErrorResponse` |
| DTO mapping | `EntityMapper` with class-transformer |
| Multi-tenancy | `TenantContextService` (REQUEST-scoped) + `@UseTenantFilter()` |
| Audit logging | `AuditService` within transactions |
| Error handling | `HttpExceptionFilter` (global) + try-catch in services |
| Logging | Winston via `this.logger` from `BaseService` |
| Date handling | Day.js (NOT moment.js) |
| File storage | AWS S3 via `@aws-sdk/client-s3` |

---

## API Route Organization

Endpoints are organized by business domain:

| Domain | Prefix | Examples |
|--------|--------|----------|
| HRM (Human Resources) | `/hrm/*` | `/hrm/users`, `/hrm/departments`, `/hrm/contracts` |
| MES (Manufacturing) | `/mes/*` | `/mes/stages`, `/mes/production-processes`, `/mes/sale-orders` |
| PM (Project Management) | `/pm/*` | `/pm/projects`, `/pm/tasks`, `/pm/risks` |
| Shared / Cross-cutting | No prefix | `/auth`, `/roles`, `/permissions`, `/upload`, `/audit-logs`, `/approval` |

---

## Response Formats

### Success Response

```typescript
new SuccessResponse(data, message)
// { code: 200, message: "...", data: { ... } }
```

### List Response (Paginated)

```typescript
new SuccessListResponse(items, totalCount, message)
// { code: 200, message: "...", data: { items: [...], totalItems: 42 } }
```

### Error Response

```typescript
// Structured errors with field-level detail
{
  code: 400,
  message: "Validation failed",
  errors: [{ key: "email", errorCode: 400, message: "Email already exists" }]
}
```

---

## Security

- Session stored in PostgreSQL via `connect-pg-simple`
- Passwords hashed with bcrypt/argon2
- CSRF protection via guard
- XSS protection via isomorphic-dompurify
- Input sanitization via `TrimObjectPipe`
- File upload: whitelist MIME types, 50MB max, tenant-isolated S3 keys
- Never log sensitive data (passwords, tokens, secrets)
- Use environment variables for all secrets

---

## Database Rules

### TypeORM Conventions

- Use explicit column types
- Use UUID primary keys (`@PrimaryGeneratedColumn('uuid')`)
- Define relations with explicit cascade options
- Use explicit join table entities for many-to-many relations
- Support soft deletes with `@DeleteDateColumn()`
- Use `where: '"deletedAt" IS NULL'` in unique indexes

### Migration Rules (CRITICAL)

- ALWAYS create migrations for schema changes
- CHECK table/column existence before ALTER operations
- CHECK for NULL values before setting NOT NULL constraint
- CHECK referenced tables exist before creating foreign keys
- Implement both `up()` and `down()` methods
- NEVER modify committed migrations - always create new ones
- Migration naming: timestamp-based (`{timestamp}-{description}.ts`)

### Repository Pattern

- Return `null` for not found (don't throw)
- Load necessary relations upfront
- Encapsulate complex query logic
- Use QueryBuilder for complex filters

---

## Key Modules Reference

### Shared (`/libs/shared/src/modules/`)

auth, rbac (roles/permissions/guards), upload, audit, account, session, common

### HRM (`/libs/hrm/src/modules/`)

user, departments, attendance, contract, allowance, work-schedule, overtime, approval, hrm-reports, settings/holiday

### MES (`/libs/mes/src/modules/`)

product-teams, product-groups, stages, production-processes, production-equipment, production-sale-order, production-customer, production-execution-order, production-execution-schedule, production-execution-statistics, product-quality-*, production-plan-overall-plan, production-alternative-material

### PM (`/libs/pm/src/modules/`)

pm-project, pm-task, pm-risk, pm-team, pm-dashboard, pm-worker-request

### Warehouse/Inventory

warehouse-dictionary-unit, warehouse-dictionary-inventoryitemgroup, warehouse-dictionary-bom, warehouse-dictionary-stock, inventory-item, inventory-item-template

---

## Common Constants Reference

```typescript
// Pagination
MIN_PAGE_LIMIT = 1; MAX_PAGE_LIMIT = 100; LIMIT_PER_PAGE_DEFAULT = 10;

// Field lengths
INPUT_TEXT_MAX_LENGTH = 255;
USER_NAME_MAX_LENGTH = 128;
TEXTAREA_MAX_LENGTH = 2000;

// Validation patterns
NAME_ALLOWED_PATTERN = /^[A-Za-z\u00C0-\u1EF9\-\s']+$/u;
PHONE_NUMBER_PATTERN = /^\+?\d{8,15}$/;

// Date formats
DateFormat.YYYY_MM_DD_HYPHEN = 'YYYY-MM-DD';
DateFormat.DD_MM_YYYY_DASH = 'DD/MM/YYYY';
```

---

## Development Commands

```bash
npm run start:dev         # Dev server with hot reload
npm run build             # Production build
npm run db:setup          # Run migrations + seeds
npm run db:reset          # Revert + Migrate + Seed
npm run migration:run     # Run pending migrations
npm run migration:revert  # Revert last migration
npm run test              # Unit tests
npm run test:coverage     # Coverage report
npm run test:e2e          # E2E tests
npm run lint              # ESLint
npm run format            # Prettier
npm run i18n:check        # Check missing translation keys
```
