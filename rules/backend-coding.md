# Backend Coding Guidelines (NestJS ERP)

Coding standards for the ERP backend: NestJS 11.0.1 + TypeScript 5.7.3 + PostgreSQL + TypeORM 0.3.26.

---

## Project Structure

### Monorepo Layout

```
erp-backend/
├── apps/
│   ├── api/          # Main API application (port 3000)
│   └── admin/        # Admin API application (port 3001)
├── libs/
│   ├── shared/       # Core shared infrastructure (@app/shared)
│   ├── hrm/          # Human Resource Management (@app/hrm)
│   ├── mes/          # Manufacturing Execution System (@app/mes)
│   └── pm/           # Project Management (@app/pm)
```

### Module Structure

Each feature module follows this layout:

```
modules/{feature}/
├── {feature}.module.ts           # DI container, imports, exports
├── {feature}.controller.ts       # HTTP endpoints, request validation
├── {feature}.service.ts          # Business logic, transactions
├── {feature}.validator.ts        # Joi validation schemas
├── {feature}.interface.ts        # TypeScript interfaces
├── {feature}.service.spec.ts     # Unit tests (REQUIRED)
└── dto/
    └── {feature}-response.dto.ts # Response DTOs with @Expose()
```

---

## Path Aliases (MANDATORY)

Never use relative imports. Always use path aliases:

```typescript
// Shared library
import { User } from '@entities/user.entity';
import { SuccessResponse } from '@common/utils/api.response';
import { BaseService } from '@common/services/base.service';
import { TenantContextService, UseTenantFilter } from '@common/tenant';
import { EntityMapper } from '@common/utils/entity-mapper.util';
import { JoiValidationPipe } from '@common/pipes/joi-validation.pipe';
import { TrimObjectPipe } from '@common/pipes/trim-object.pipe';
import Joi, { IdParamSchema } from '@common/plugins/joi';
import { AuthenticatedGuard } from '@modules/rbac/guards/authenticated.guard';
import { RequirePermissions } from '@modules/rbac/decorators/permissions.decorator';
import { AuditService } from '@modules/audit/audit.service';

// Domain libraries
import { UserService } from '@hrm/user/user.service';
import { ProductTeamService } from '@mes/product-teams/product-team.service';
import { PmProjectService } from '@pm/pm-project/pm-project.service';
```

Available aliases:
- `@common/*` -> `libs/shared/src/common/*`
- `@database/*` -> `libs/shared/src/database/*`
- `@entities/*` -> `libs/shared/src/database/entities/*`
- `@modules/*` -> `libs/shared/src/modules/*`
- `@hrm/*` -> `libs/hrm/src/modules/*`
- `@mes/*` -> `libs/mes/src/modules/*`
- `@pm/*` -> `libs/pm/src/modules/*`

---

## Naming Conventions

- **Classes** -> `PascalCase` (`UserService`, `CreateUserDto`, `AuthenticatedGuard`)
- **Variables & Functions** -> `camelCase` (`userName`, `getUserById`)
- **Constants** -> `UPPER_CASE` (`MAX_USERS`, `DEFAULT_PAGE_LIMIT`)
- **Enums** -> `PascalCase` with `UPPER_CASE` values
- **Files** -> `kebab-case` (`user.service.ts`, `user-response.dto.ts`)
- **API routes** -> domain prefix (`/hrm/users`, `/mes/stages`, `/pm/projects`)
- Function length < 50 lines, split into helpers if needed

---

## Service Pattern (MANDATORY)

Every service MUST:
1. Extend `BaseService` and call `super(i18n)`
2. Use `@UseTenantFilter()` decorator
3. Inject `TenantContextService`
4. Use `this.logger` (from BaseService) - NEVER `console.log`
5. Wrap methods in try-catch with logging
6. Have a corresponding `.spec.ts` test file

```typescript
@Injectable()
@UseTenantFilter()
export class FeatureService extends BaseService {
  constructor(
    @InjectRepository(Feature)
    private readonly featureRepository: Repository<Feature>,
    i18n: I18nService,
    private readonly tenantContext: TenantContextService,
    private readonly dataSource: DataSource,
    private readonly auditService: AuditService,
  ) {
    super(i18n);
  }

  async findAll(page: number, limit: number, search?: string): Promise<[Feature[], number]> {
    try {
      const organId = this.tenantContext.getOrganId();
      return await this.featureRepository.findAndCount({
        where: { organId },
        skip: (page - 1) * limit,
        take: limit,
      });
    } catch (error) {
      this.logger.error(`Error finding features: ${error}`);
      if (error instanceof BadRequestException) throw error;
      throw new BadRequestException(this.i18n.t('common.error.fetchFailed'));
    }
  }
}
```

---

## Controller Pattern (MANDATORY)

Every controller MUST:
1. Use `@UseGuards(AuthenticatedGuard, PermissionsGuard)`
2. Use `TrimObjectPipe` BEFORE `JoiValidationPipe`
3. Use `IdParamSchema` for UUID path parameters
4. Return `SuccessResponse`/`SuccessListResponse`/`ErrorResponse`
5. Use `EntityMapper` for DTO transformation (NOT in service)
6. Include Swagger decorators (`@ApiTags`, `@ApiOperation`, `@ApiResponse`, `@ApiCookieAuth`)

```typescript
@Controller('hrm/features')
@UseGuards(AuthenticatedGuard, PermissionsGuard)
@ApiTags('Features')
@ApiCookieAuth()
export class FeatureController {
  constructor(
    private readonly featureService: FeatureService,
    private readonly i18n: I18nService,
  ) {}

  @Get()
  @RequirePermissions('feature.list')
  @UsePipes(new JoiValidationPipe(SimpleListQuerySchema))
  async findAll(@Query() query: BaseListQuery): Promise<SuccessListResponse> {
    const [items, total] = await this.featureService.findAll(
      query.page ?? 1,
      query.limit ?? 25,
      query.search,
    );
    return new SuccessListResponse(
      EntityMapper.toDtosWithRelations(FeatureResponseDto, items),
      total,
      this.i18n.t('common.success'),
    );
  }

  @Post()
  @RequirePermissions('feature.create')
  @UsePipes(new TrimObjectPipe(), new JoiValidationPipe(CreateFeatureSchema))
  async create(@Req() req: AuthenticatedRequest, @Body() dto: CreateFeature) {
    try {
      const result = await this.featureService.create(dto, req.user.id);
      return new SuccessResponse(
        EntityMapper.toDtoWithRelations(FeatureResponseDto, result),
        this.i18n.t('common.feature.createdSuccessfully'),
      );
    } catch (error) {
      return new ErrorResponse(
        HttpStatus.INTERNAL_SERVER_ERROR,
        this.i18n.t('common.error.internalServerError'),
        error instanceof BadRequestException ? error.getResponse()['errors'] : [],
      );
    }
  }
}
```

---

## Validation Pattern (Joi)

Define schemas in separate `*.validator.ts` files with matching TypeScript interfaces:

```typescript
import Joi from '@common/plugins/joi';
import { NAME_ALLOWED_PATTERN, USER_NAME_MAX_LENGTH } from '@common/constants';

export const CreateFeatureSchema = Joi.object({
  name: Joi.string().trim().required().pattern(NAME_ALLOWED_PATTERN).max(USER_NAME_MAX_LENGTH),
  email: Joi.string().trim().email().optional(),
  roleIds: Joi.array().items(Joi.string().uuid()).optional(),
});

export interface CreateFeature {
  name: string;
  email?: string;
  roleIds?: string[];
}
```

---

## Response DTO Pattern (class-transformer)

```typescript
import { Expose, Type } from 'class-transformer';

export class FeatureResponseDto {
  @Expose() id: string;
  @Expose() name: string;
  @Expose() @Type(() => NestedDto) nested?: NestedDto;
  @Expose() createdAt: Date;
  @Expose() updatedAt: Date;
}
```

Use `EntityMapper` in controllers:
- `EntityMapper.toDto(DtoClass, entity)` - Single entity
- `EntityMapper.toDtos(DtoClass, entities)` - Array
- `EntityMapper.toDtoWithRelations(DtoClass, entity)` - With nested relations
- `EntityMapper.toDtosWithRelations(DtoClass, entities)` - Array with relations

---

## Database Patterns

### Entity Design

All entities extend `BaseEntity` with: `id` (UUID), `organId`, `createdAt`, `updatedAt`, `createdBy`, `updatedBy`.

Support soft deletes with `@DeleteDateColumn()`:
```typescript
@DeleteDateColumn({ nullable: true })
deletedAt?: Date;
```

### Many-to-Many Relations

Use explicit join table entities (NOT implicit `@JoinTable()`):
```typescript
@Entity('contract_allowances')
export class ContractAllowance extends BaseEntity {
  @Column({ type: 'uuid' }) contractId: string;
  @Column({ type: 'uuid' }) allowanceId: string;
  @ManyToOne(() => Contract) @JoinColumn({ name: 'contractId' }) contract: Contract;
  @ManyToOne(() => Allowance) @JoinColumn({ name: 'allowanceId' }) allowance: Allowance;
}
```

### Migration Safety (CRITICAL)

- CHECK table/column existence before ALTER operations
- CHECK for NULL values before setting NOT NULL constraint
- CHECK referenced tables exist before creating foreign keys
- Use `where: '"deletedAt" IS NULL'` in unique indexes
- Implement both `up()` and `down()` methods
- NEVER modify committed migrations - create new ones

---

## Audit Logging

Wrap CUD operations in transactions with audit:

```typescript
async create(dto: CreateDto, createdBy: string): Promise<Entity> {
  const organId = this.tenantContext.getOrganId();
  return await this.dataSource.transaction(async (manager: EntityManager) => {
    const entity = manager.create(Entity, { ...dto, organId });
    const saved = await manager.save(Entity, entity);
    await this.auditService.logCreateWithManager(manager, 'Entity', saved.id, { ...saved }, createdBy, organId);
    return saved;
  });
}
```

Use `@SkipAudit()` for health/ping endpoints.

---

## Multi-Tenancy

- `TenantContextService` is REQUEST-scoped, provides `getOrganId()`
- Services get organId via `this.tenantContext.getOrganId()` - NEVER pass as parameter
- All queries MUST filter by `organId`
- Tenant middleware extracts organId from session

---

## Internationalization (i18n)

- Use `this.i18n.t('key')` for all user-facing text - NEVER hardcode strings
- Supported languages: English (en), Vietnamese (vi) - fallback: vi
- Translation files: `/libs/shared/src/i18n/{lang}/common.json`

---

## Pagination (Standard)

```typescript
interface BaseListQuery {
  page?: number;     // Default: 1, Min: 1, Max: 1000
  limit?: number;    // Default: 10, Min: 1, Max: 100
  search?: string;   // Max: 255 chars
  sortBy?: string;
  sortOrder?: 'ASC' | 'DESC';
  fromDate?: Date;
  toDate?: Date;
}
```

---

## Testing

- Framework: Jest v30.0.0
- Every service MUST have `.spec.ts` co-located with source
- Mock: `Repository`, `I18nService`, `DataSource`, `TenantContextService`, `AuditService`
- Test both success and error paths
- Coverage target: > 70%

---

## PR Checklist

- [ ] No `console.log` - use `this.logger` from BaseService
- [ ] Services extend `BaseService` with `@UseTenantFilter()`
- [ ] Controllers use `TrimObjectPipe` before `JoiValidationPipe`
- [ ] Response DTOs have `@Expose()` decorators
- [ ] `EntityMapper` used for transformation in controller (not service)
- [ ] `IdParamSchema` used for UUID params
- [ ] i18n keys for all user-facing text
- [ ] Swagger docs updated for API changes
- [ ] Audit logging for CUD operations
- [ ] Unit tests with `.spec.ts` file
- [ ] No TypeScript `any` - use `unknown` or specific types
- [ ] No relative imports - use path aliases
- [ ] Migrations have safety checks and both up/down
