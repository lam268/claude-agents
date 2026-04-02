# Admin Coding Guidelines (React ERP Control Plane)

Coding standards for the ERP admin panel: React 18.3.1 + TypeScript 5.8.2 + Vite 6 + Tailwind CSS 4.

---

## Project Overview

`erp-admin` is a standalone admin control plane for managing the ERP system infrastructure.
It is separate from `erp-frontend` (the main ERP application for end users).

| Aspect | erp-admin | erp-frontend |
|--------|-----------|--------------|
| Purpose | System admin (tenants, health, migrations) | End-user ERP (HR, MES, PM) |
| Port | 4201 | 4200 |
| API | `/admin-api` (port 3001) | `/api` (port 3000) |
| Users | Super admins | Employees, managers |
| Complexity | Lower (focused scope) | Higher (multi-module) |

---

## Project Structure

```
erp-admin/
├── src/
│   ├── main.tsx                    # Entry point
│   ├── App.tsx                     # Router + layout
│   ├── styles.css                  # Tailwind import with tw: prefix
│   ├── pages/                      # Feature pages
│   │   ├── auth/AdminLoginPage.tsx
│   │   ├── dashboard/DashboardPage.tsx
│   │   ├── tenants/                # Tenant CRUD + setup
│   │   ├── health/                 # Health monitoring
│   │   ├── migrations/             # DB migration management
│   │   └── system/SystemPage.tsx
│   ├── common/
│   │   ├── components/             # AdminAuthContext, AdminAuthRoute, AdminLayout
│   │   └── utils/error.ts
│   ├── providers/QueryClientProvider.tsx
│   ├── services/                   # API service layer
│   │   ├── adminApiService.ts      # Base CRUD service
│   │   ├── adminAuthService.ts     # Auth endpoints
│   │   ├── adminAxiosInstance.ts   # Axios config + interceptors
│   │   ├── tenantService.ts
│   │   ├── healthService.ts
│   │   ├── migrationService.ts
│   │   └── monitoringService.ts
│   ├── hooks/useSnackbar.ts
│   └── i18n/                       # i18next (vi, en)
│       └── locales/{lang}/admin.ts
```

---

## Path Aliases

```typescript
import { SomeThing } from '@/path/to/thing';           // src/*
import { SomeThing } from '@shared/something';          // src/shared/
import { Button, Table } from 'ui-components';           // UI component library
import type { BaseMenuItem } from '@ui/components/layouts/Menu/type';
```

Available aliases:
- `@/*` -> `src/*`
- `@shared` / `@shared/*` -> `src/shared/`
- `ui-components` -> `../ui-components/src/index.ts`
- `@ui/*` -> `../ui-components/src/*`

---

## Tech Stack

- **React** 18.3.1 + **TypeScript** 5.8.2
- **Vite** 6.0.0 (build tool)
- **React Router DOM** 6.28.0 (routing, NOT TanStack Router)
- **TanStack React Query** 4.36.1 (server state)
- **React Hook Form** 7.62.0 + **Zod** 3.23.8 (forms)
- **Tailwind CSS** 4.1.12 with `tw:` prefix
- **Axios** 1.12.2 (HTTP client)
- **i18next** 25.5.2 (internationalization)
- **Sonner** 2.0.7 (toast notifications)
- **@connext-solutions/ui-components** (shared component library)
- **Recharts** 3.5.1 (dashboard metrics)

---

## Key Differences from erp-frontend

1. **Routing**: Uses React Router v6 (NOT TanStack Router)
2. **No Nx monorepo**: Standalone project, flat `src/` structure
3. **No domain libraries**: All code in `src/` directly
4. **Simpler auth**: Admin-only auth, no multi-tenant org switching
5. **Service pattern**: Extends `AdminApiService` base class (similar but separate)
6. **Fewer features**: ~10 pages vs 50+ in frontend

---

## Authentication Pattern

```typescript
// AdminAuthProvider wraps entire app
<AdminAuthProvider>
  <Routes>
    <Route path="/login" element={<AdminAuthRoute type="public"><AdminLoginPage /></AdminAuthRoute>} />
    <Route path="/" element={<ProtectedLayout><DashboardPage /></ProtectedLayout>} />
  </Routes>
</AdminAuthProvider>
```

- Session-based auth with CSRF token
- CSRF token in sessionStorage, attached to POST/PUT/PATCH/DELETE requests
- 401 response -> clear session, redirect to `/login`
- `AdminAuthRoute` wrapper handles public/protected route logic

---

## Service Layer Pattern

```typescript
// Base class with CRUD methods
class AdminApiService {
  async getList<T>(queryString): Promise<IBodyResponse<IGetListResponse<T>>>
  async getDetail<R>(id): Promise<IBodyResponse<R>>
  async create<P, R>(params): Promise<IBodyResponse<R>>
  async update<P, R>(id, params): Promise<IBodyResponse<R>>
  async delete<R>(id): Promise<IBodyResponse<R>>
}

// Domain services extend it
class TenantService extends AdminApiService { ... }

// Export singleton
export default new TenantService('/tenants');
```

---

## Styling (Tailwind CSS with `tw:` prefix)

Same as `erp-frontend` - ALL Tailwind classes use `tw:` prefix:

```tsx
<div className="tw:flex tw:items-center tw:gap-4 tw:p-4">
<div className="tw:grid tw:grid-cols-1 tw:md:grid-cols-3">
```

---

## Form Handling (React Hook Form + Zod)

```typescript
const schema = z.object({
  organName: z.string().min(1).max(255),
  dbPort: z.coerce.number().int().min(1).max(65535).default(5432),
  enabledModules: z.array(z.enum(['hrm', 'mes', 'pm'])).default([]),
});

const form = useForm({ resolver: zodResolver(schema) });

<FormFieldInputText control={form.control} name="organName" label={t('admin.tenant.name')} />
```

---

## Internationalization

- Default: Vietnamese (vi), supports English (en)
- Detection: localStorage -> navigator -> htmlTag
- Translation files: `src/i18n/locales/{lang}/admin.ts`
- Usage: `const { t } = useTranslation(); t('admin.nav.dashboard')`

---

## Routes

| Path | Component | Auth |
|------|-----------|------|
| `/login` | AdminLoginPage | Public |
| `/` | DashboardPage | Protected |
| `/tenants` | TenantListPage | Protected |
| `/tenants/create` | TenantCreatePage | Protected |
| `/tenants/:organId` | TenantDetailPage | Protected |
| `/tenants/:organId/setup` | TenantSetupPage | Protected |
| `/health` | HealthListPage | Protected |
| `/health/:organId` | HealthDetailPage | Protected |
| `/migrations` | MigrationPage | Protected |
| `/migrations/:organId` | MigrationDetailPage | Protected |
| `/system` | SystemPage | Protected |

All pages lazy-loaded with `React.lazy()` + `Suspense`.

---

## Naming Conventions

- **Components**: PascalCase files (`AdminLoginPage.tsx`, `TenantListPage.tsx`)
- **Services/hooks/utils**: camelCase files (`adminAuthService.ts`, `useSnackbar.ts`)
- **Variables/functions**: camelCase
- **Types/Interfaces**: PascalCase
- **Constants**: UPPER_SNAKE_CASE (`ERP_MODULES`, `FILTER_STATUSES`)

---

## Code Style

- **oxfmt**: `semi: false`, `singleQuote: true` (Rust-based formatter)
- **oxlint**: TypeScript, React, and import plugins (Rust-based linter)
- No `any` type
- Use `??` instead of `||` for defaults
- Error handling: `getErrorMessage(error, fallbackMessage)` utility

---

## Testing

- **Unit**: Vitest + @testing-library/react
- **E2E**: Playwright
- Commands:
  - `npm run test` - Unit tests
  - `npm run test:e2e` - E2E tests (headless)
  - `npm run test:e2e:ui` - E2E with UI

---

## Development Commands

```bash
npm run dev              # Dev server (port 4201)
npm run build            # Production build
npm run preview          # Preview build
npm run test             # Vitest
npm run test:e2e         # Playwright E2E
npm run lint             # oxlint (Rust-based linter)
npm run fmt              # oxfmt (Rust-based formatter)
npm run fmt:check        # Check formatting without writing
make fix                 # Run both oxlint --fix and oxfmt --write
```

---

## Environment Variables

```bash
VITE_ADMIN_API_URL=http://localhost:3001/admin-api
```

---

## Code Quality Checklist

- [ ] No `any` type
- [ ] `??` used instead of `||`
- [ ] All text uses i18n keys (`t('admin.key')`)
- [ ] API calls through service layer
- [ ] Tailwind classes use `tw:` prefix
- [ ] Forms use React Hook Form + Zod
- [ ] Components from `ui-components` library
- [ ] Protected routes use `AdminAuthRoute`
- [ ] Error handling with `getErrorMessage()` utility
- [ ] Toast notifications via `useSnackbar()` or `sonner`
- [ ] `make fix` passes (oxlint + oxfmt)
