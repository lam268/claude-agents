# Frontend Coding Guidelines (React ERP)

Coding standards for the ERP frontend: React 18.3.1 + TypeScript 5.8.2 + Vite 6 + Nx Monorepo + Tailwind CSS 4.

---

## TypeScript Conventions

### TS-1: Wrap External Libraries

- Prevents breaking changes from external library updates
- Always wrap external libraries with internal abstractions
- Exception: `ui-components` (internal component library)
- Import from `ui-components` or `@ui/*`, NOT directly from Radix/Lucide

### TS-2: Use `??` Instead of `||`

- `??` only triggers on `null`/`undefined`, `||` triggers on all falsy values
- `false ?? true` -> `false` (correct for booleans)
- `data?.items ?? []` (array default)
- `item.quantity ?? 0` (numeric default)

### TS-3: Never Use `any`

- Use `unknown` or specific types instead
- When absolutely necessary due to external library constraints, add a comment explaining why
- Create union types for dynamic values: `type FormFieldValue = string | number | boolean | null | undefined`

### TS-4: Avoid `as` Type Assertions

- If necessary, isolate in wrapper functions
- Use type guards (`typeof`, `instanceof`, `in`) instead
- Use `keyof` for type-safe property access: `item[field as keyof Entity]`

### TS-5: Array Bounds Safety

```typescript
const items = [3, 4, 5];
// Bad: items[1000] is inferred as number but is actually undefined
// Good: items[1000] ?? 0
// Good: const val = items[1000]; if (val === undefined) throw new Error();
```

### TS-6: Use `assertNever` for Exhaustive Checks

```typescript
type Status = 'active' | 'inactive';
switch (status) {
  case 'active': return handleActive();
  case 'inactive': return handleInactive();
}
assertNever(status); // Compile error if new status added
```

### TS-7: Clear oxlint/cspell Warnings

- Comments required when ignoring oxlint warnings
- Fix all cspell warnings or add to dictionary

---

## React Conventions

### R-1: Performance During Re-renders

- Use React DevTools Profiler to identify bottlenecks
- Memoize expensive computations with `useMemo`
- Use `useCallback` for handler functions passed to memoized children

### R-2: Minimize State Lifting Scope

- Avoid Redux-like patterns with useReducer + useContext
- Use children pattern to avoid prop drilling:

```tsx
// Good: Compose with children instead of drilling props
function MyComponent() {
  const [state] = useState(...)
  return (
    <Wrapper>
      <DeepChild state={state} />
    </Wrapper>
  );
}
```

### R-3: Custom Hook Callbacks Must Use `useCallback`

- Callbacks returned from custom hooks MUST be memoized with `useCallback`
- Usage pattern cannot be determined from the hook's perspective

### R-4: Avoid Inline Callbacks in JSX

```tsx
// Bad
<button onClick={() => alert(3)} />

// Good
const handleClick = () => alert(3);
<button onClick={handleClick} />
```

Exception: Inline callbacks in `.map()` loops are acceptable.

### R-5: Write Tests for Complex Logic

- Test custom hooks with React Testing Library
- Test complex utility functions
- Test form validation schemas

### R-6: Generic Components in `libs/`

- All shared/generic UI components must be in `libs/` or `ui-components`
- Always check if a component exists before creating or importing externally

```tsx
// Bad
import { Button } from '@mui/material';

// Good
import { Button } from 'ui-components';
```

### R-7: File Size Limits (200-250 Lines)

- Target: **200-250 lines** per file
- Maximum: 300 lines (exception cases only)
- If exceeding, refactor into:
  - Sub-components in `components/` directory
  - Custom hooks in `hooks/` files
  - Utility functions in `utils/` directory
- Large dialogs (>400 lines): lazy load with `React.lazy()` + `Suspense`

### R-8: Organize Hooks by Purpose

```
hooks/
├── useQuery.ts        # Data fetching with query key factory
├── useMutation.ts     # Mutations with cache invalidation + toast
├── hooks.ts           # Other custom hooks (state, callbacks)
└── index.ts           # Re-export all hooks
```

Query key factory pattern:
```typescript
export const QUERY_KEYS = {
  all: ['entities'] as const,
  lists: () => [...QUERY_KEYS.all, 'list'] as const,
  list: (filters: unknown) => [...QUERY_KEYS.lists(), filters] as const,
  details: () => [...QUERY_KEYS.all, 'detail'] as const,
  detail: (id: string) => [...QUERY_KEYS.details(), id] as const,
}
```

### R-9: Radix UI Select - No Empty Values

- NEVER use `""` or `null` as `value` prop for `InputSelectItem`
- Optional form fields: use `"__none__"` placeholder, transform to `null` in Zod schema
- Filter selects: use `"all"` value, transform to empty string in handler

---

## Page Structure Pattern

```
pages/{module}/
├── {Module}Page.tsx          # Main page (< 250 lines)
├── components/               # Page-specific sub-components
├── dialogs/                  # Modal dialogs (lazy loaded if > 400 lines)
├── schemas/                  # Zod validation schemas
├── hooks/
│   ├── useQuery.ts           # React Query data fetching
│   ├── useMutation.ts        # React Query mutations
│   └── index.ts              # Re-exports
├── types.ts                  # TypeScript interfaces
└── constants.ts              # Module-specific constants
```

---

## Path Aliases (MANDATORY)

Never use relative imports. Always use path aliases:

```typescript
// Main app
import { SomeThing } from '@/path/to/thing';

// Domain libraries
import { UserPage } from '@app/hrm';
import { SaleOrderPage } from '@app/mes';
import { ProjectPage } from '@app/pm';

// Shared library
import { usePermission, usePagination, useI18n } from '@shared/hooks';
import { Pagination, PermissionGate, LoadingState } from '@shared/components';
import { ApiService } from '@shared/services';
import { HttpStatus, ErrorCode } from '@shared/common/constants';
import { dateUtils } from '@shared/utils';

// Domain internals
import { UserService } from '@hrm/services/userService';
import { SaleOrderService } from '@mes/services/saleOrderService';

// UI components
import { Button, Table, Modal } from 'ui-components';
import type { ColumnDef } from '@ui/components/Table/type';
```

Available aliases:
- `@/*` -> `apps/main/src/*`
- `@shared` / `@shared/*` -> `libs/shared/src/`
- `@app/hrm` -> `libs/hrm/src/index.ts`
- `@app/mes` -> `libs/mes/src/index.ts`
- `@app/pm` -> `libs/pm/src/index.ts`
- `@hrm/*` -> `libs/hrm/src/*`
- `@mes/*` -> `libs/mes/src/*`
- `@pm/*` -> `libs/pm/src/*`
- `ui-components` -> `../ui-components/src/index.ts`
- `@ui/*` -> `../ui-components/src/*`

---

## State Management

| Type | Tool | Usage |
|------|------|-------|
| Server state | React Query (TanStack Query v4) | Data fetching, caching, sync |
| Auth state | React Context (`AuthContext`) | User, org, permissions |
| Component state | `useState` / `useReducer` | Local UI state |

React Query defaults:
- `refetchOnWindowFocus: false`
- `retry: 1` (queries), `0` (mutations)
- `staleTime: 30 * 1000` (30 seconds)

---

## Form Handling (React Hook Form + Zod)

```typescript
// 1. Define schema in schemas/ directory
const schema = z.object({
  name: z.string().min(1, t('validation.required')).max(255),
  email: z.string().email(t('validation.email')),
});

// 2. Use form with zodResolver
const form = useForm({
  resolver: zodResolver(schema),
  defaultValues: { name: '', email: '' },
});

// 3. Use FormField components from ui-components
<FormFieldInputText control={form.control} name="name" label={t('field.name')} />

// 4. Handle submission
const onSubmit = async (data) => {
  await mutation.mutateAsync(data);
};
```

---

## Styling (Tailwind CSS with `tw:` prefix)

All Tailwind classes MUST use `tw:` prefix:

```tsx
// Correct
<div className="tw:flex tw:items-center tw:gap-4 tw:p-4">
<div className="tw:grid tw:grid-cols-1 tw:md:grid-cols-2 tw:lg:grid-cols-3">

// Incorrect - will NOT work
<div className="flex items-center gap-4">
<div className="grid md:tw:grid-cols-2">
```

Color tokens: `primary`, `danger`, `success`, `brand`, `warning`
Semantic: `text.*`, `background.*`, `surface.*`, `border.*`, `input.*`

### Z-Index Management

```
z-[1000]  Mobile full-screen
z-[500]   Dropdown/Select
z-[200]   Modal content
z-[100]   Modal backdrop
z-50      Floating UI
z-10      Navigation/sidebar
z-[5]     Tooltips
z-1       Sticky headers
```

---

## Routing (TanStack Router)

- All routes use `lazyRouteComponent()` for code splitting
- Domain routes created via factory: `createHrmRoutes()`, `createMesRoutes()`, `createPmRoutes()`
- Default preload: `'intent'` (preload on hover)
- Protected routes require authentication via `AdminAuthRoute`
- Permission checks via `PermissionRoute` component

---

## API Integration

- All API calls through service classes extending `ApiService` (from `@shared/services`)
- NEVER call axios directly from components
- Service singleton pattern: `export default new MyService({ baseUrl: '/hrm/users' }, axiosInstance)`
- Axios interceptors handle: CSRF token, timezone headers, 401 redirects

API response shape:
```typescript
interface IBodyResponse<T> {
  success?: boolean;
  code?: HttpStatus;
  message: string;
  data: T;
  errors?: IResponseError[];
}
```

---

## Internationalization (i18n)

- Use `react-i18next` for all user-facing text
- NEVER hardcode strings
- Supported: Vietnamese (vi - default), English (en)
- Use `useTranslation()` hook or `useI18n()` wrapper from `@shared/hooks`
- Set document title: `useDocumentTitle(t('module.pageTitle'))`
- Zod i18n integration via `zod-i18n-map` for validation messages
- Date formatting: use `dayjs` (NEVER date-fns)

---

## Permissions (RBAC)

Prefix-matching permission model:
- Permission `"user"` grants: `"user"`, `"user.create"`, `"user.update"`, `"user.delete"`
- Permission `"user.create"` grants: `"user.create"` only

```tsx
// Hook usage
const { can } = usePermission();
{can('user.create') && <Button>Create</Button>}

// Component usage
<PermissionGate permission="user.create"><Button>Create</Button></PermissionGate>

// Route-level
<PermissionRoute permission="user.view" fallback={<ForbiddenPage />}>
  <UserListPage />
</PermissionRoute>
```

---

## Code Quality Checklist

- [ ] No oxlint warnings (or documented ignores)
- [ ] No `any` or `as` without comments
- [ ] `??` used instead of `||` for defaults
- [ ] All text uses i18n translation keys
- [ ] API calls through service layer (not direct axios)
- [ ] Custom hook callbacks use `useCallback`
- [ ] No inline callbacks in JSX
- [ ] Files under 250 lines (300 max)
- [ ] Forms use React Hook Form + Zod
- [ ] Tailwind classes use `tw:` prefix
- [ ] Path aliases used (no relative imports)
- [ ] Generic components from `ui-components` / `libs/`
- [ ] Large dialogs lazy-loaded
- [ ] Hooks organized: `useQuery.ts`, `useMutation.ts`, `hooks.ts`
