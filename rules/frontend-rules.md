# Frontend ERP Architecture Rules

Architectural guidelines for the ERP frontend: React 18.3.1 + Nx Monorepo + TanStack Router + React Query.

---

## Monorepo Architecture (Nx)

### Project Structure

```
erp-frontend/
├── apps/
│   └── main/               # Main ERP application (port 4200)
│       └── src/
│           ├── app/         # App entry
│           ├── contexts/    # AuthContext
│           ├── i18n/        # Translations (vi, en)
│           ├── pages/       # Auth, approval, home pages
│           ├── providers/   # QueryClient, Auth providers
│           ├── router/      # TanStack Router setup
│           └── services/    # Axios instance, base service
├── libs/
│   ├── shared/              # Cross-cutting concerns (@shared)
│   │   └── src/
│   │       ├── common/      # Constants (HttpStatus, ErrorCode, DateFormat)
│   │       ├── components/  # Pagination, PermissionGate, LoadingState
│   │       ├── contexts/    # AuthContext type defs + hook
│   │       ├── hooks/       # useQuery, useMutation, usePagination, usePermission, useI18n
│   │       ├── plugins/     # i18n, zod-i18n setup
│   │       ├── services/    # ApiService base, uploadService, approvalService
│   │       └── utils/       # dateUtils, encryption, helpers
│   ├── hrm/                 # HRM domain (@app/hrm)
│   ├── mes/                 # MES domain (@app/mes)
│   └── pm/                  # PM domain (@app/pm)
```

### Dependency Constraints

- **Main app** -> can depend on everything
- **HRM, MES, PM** -> can ONLY depend on `shared`
- **Shared** -> can ONLY depend on itself
- Domain libraries MUST NOT import from each other

---

## Routing (TanStack Router)

Route tree assembled from domain route factories:

```typescript
const routeTree = rootRoute.addChildren([
  loginRoute,
  authenticatedRoute.addChildren([
    homeRoute,
    createHrmRoutes(authenticatedRoute),
    createMesRoutes(authenticatedRoute),
    createPmRoutes(authenticatedRoute),
  ]),
  forbiddenRoute,
  notFoundRoute,
])
```

Key routes:
- `/` - Home (protected)
- `/login` - Auth (public)
- `/select-organ` - Org selection (multi-tenant)
- `/hrm/*` - HRM module pages
- `/mes/*` - MES module pages
- `/pm/*` - PM module pages

All pages lazy-loaded with `lazyRouteComponent()`.

---

## Authentication Flow

1. Login -> POST `/auth/login` with credentials
2. Server returns session cookie + CSRF token
3. CSRF token stored in sessionStorage, attached to all state-changing requests
4. User data cached in localStorage for optimistic UI
5. On mount: validate session with `/auth/me`
6. 401 response -> clear auth, redirect to `/login`

Multi-tenant: Users can have multiple `organIds`, switch via `changeOrgan()`.

---

## Service Layer Pattern

```typescript
// Base class provides CRUD methods
class MyService extends ApiService {
  constructor() {
    super({ baseUrl: '/hrm/users' }, axiosInstance)
  }

  // Inherited: _getList, _getDetail, _create, _update, _delete
  // Add domain-specific methods as needed
  async getOverview(id: string) { ... }
}

export default new MyService();
```

Axios interceptors handle:
- Request: CSRF token (`x-csrf-token`), timezone headers
- Response: 401 redirect, error unwrapping

---

## Key UI Patterns

### Data Tables

```tsx
const columns: ColumnDef<Entity>[] = [
  { header: t('field.name'), accessorKey: 'name' },
  { header: t('field.status'), cell: ({ row }) => <Badge>{row.status}</Badge> },
];

<Table data={data?.items ?? []} columns={columns} />
<Pagination total={data?.totalItems ?? 0} {...pagination} />
```

### Form Dialogs

```tsx
const CreateDialog = ({ open, onClose }) => {
  const form = useForm({ resolver: zodResolver(schema) });
  const mutation = useCreateEntity();

  const onSubmit = async (data) => {
    await mutation.mutateAsync(data);
    onClose();
  };

  return (
    <Modal open={open} onClose={onClose}>
      <form onSubmit={form.handleSubmit(onSubmit)}>
        <FormFieldInputText control={form.control} name="name" label={t('name')} />
        <Button type="submit">{t('common.save')}</Button>
      </form>
    </Modal>
  );
};
```

### Dependent Dropdowns

```tsx
const handleDeptChange = useCallback(async (deptId: string) => {
  if (!deptId) { setRoles([]); form.setValue('roleId', ''); return; }
  const dept = await deptService.getDetail(deptId);
  setRoles(dept.roles);
}, [form]);
```

---

## Code Splitting & Performance

- Route-level: `lazyRouteComponent()` for all pages
- Component-level: `React.lazy()` + `Suspense` for dialogs > 400 lines
- Vite chunks configured for optimal splitting:
  - `vendor-query`, `vendor-ui`, `vendor-icons`, `vendor-editor`
  - `vendor-date`, `vendor-i18n`, `vendor-forms`
  - `shared`, `hrm`, `mes-core`, `pm`

---

## Date Handling

- Use `dayjs` exclusively (NEVER `date-fns` or `moment`)
- Format: `dayjs(date).format('DD/MM/YYYY')`
- Comparison: `dayjs(d1).isSameOrAfter(d2)`
- Manipulation: `dayjs().add(1, 'day').subtract(2, 'hours')`

---

## Mobile Support

- Responsive layouts with `tw:md:`, `tw:lg:` breakpoints
- `MobileListLayout` for mobile-optimized list views
- `MobileBottomNav` for mobile navigation

---

## Testing

### Unit Tests (Vitest)
- Environment: jsdom
- Libraries: @testing-library/react, @testing-library/jest-dom
- Commands: `npm run test`, `npm run test:coverage`

### E2E Tests (Playwright)
- Page Object Model with `BasePage` abstract class
- Accessible selectors: `getByRole` > `getByLabel` > `getByPlaceholder`
- Commands: `npm run test:e2e`, `npm run seed:e2e`

### Storybook
- Port: 4400
- Commands: `npm run storybook`, `npm run build-storybook`

---

## Development Commands

```bash
npm run dev              # Dev server (port 4200)
npm run build            # Production build
npm run test             # Vitest
npm run test:coverage    # Coverage report
npm run test:e2e         # Playwright E2E
npm run storybook        # Storybook (port 4400)
npm run lint             # oxlint (Rust-based linter)
```

---

## Environment Variables

```bash
VITE_NODE_ENV="development"
VITE_API_URL=localhost:3000/api
```
