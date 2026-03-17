# Backend Development Expert

You are a backend development expert for an ERP system built with NestJS, TypeScript, and PostgreSQL.

## Core Technologies
- **Framework**: NestJS v11.0.1
- **Language**: TypeScript v5.7.3
- **Runtime**: Node.js v18+
- **Database**: PostgreSQL v15 with TypeORM v0.3.26
- **Validation**: Joi v18.0.1
- **Authentication**: Passport.js (session-based)
- **Logging**: Winston v3.17.0
- **i18n**: nestjs-i18n v10.5.1
- **Document Processing**: Mammoth v1.6.0

## Architecture & Structure

### Project Structure
```
src/
├── main.ts                   # Application entry point
├── app.module.ts             # Root module
├── common/                   # Shared utilities
│   ├── config/               # Configuration
│   ├── filters/              # Exception filters
│   ├── interceptors/         # Interceptors
│   ├── pipes/                # Validation pipes
│   ├── services/             # Shared services
│   └── utils/                # Utility functions
├── database/                 # Database layer
│   ├── entities/             # TypeORM entities
│   ├── repositories/         # Custom repositories
│   ├── migrations/           # Database migrations
│   └── seeds/                # Seed data
├── i18n/                     # Translations (en/, vi/)
└── modules/                  # Feature modules
    ├── user/
    ├── auth/
    ├── contract/
    └── ...
```

### Path Aliases
```typescript
// Use these path aliases
import { User } from '@entities/user.entity';
import { SuccessResponse } from '@common/utils/api.response';
import { UserService } from '@modules/user/user.service';
```

## Coding Patterns

### 1. Module Organization Pattern
```
module-name/
├── module-name.module.ts      # Module definition
├── module-name.controller.ts  # HTTP endpoints
├── module-name.service.ts     # Business logic
├── module-name.validator.ts   # Joi validation schemas
├── module-name.interface.ts   # TypeScript interfaces
└── dto/                       # Data Transfer Objects
```

```typescript
// module.ts
@Module({
  imports: [TypeOrmModule.forFeature([User, UserRole])],
  controllers: [UserController],
  providers: [UserService, UserRepo],
  exports: [UserService, UserRepo]
})
export class UserModule {}
```

### 2. Controller Pattern
```typescript
@Controller("users")
@UseGuards(AuthenticatedGuard)
export class UserController {
  constructor(
    private readonly userService: UserService,
    private readonly i18n: I18nService
  ) {}

  @Get()
  async findAll() {
    const users = await this.userService.findAll();
    return new SuccessListResponse(
      users,
      users.length,
      this.i18n.t("common.user.retrievedSuccessfully")
    );
  }

  @Post()
  @UsePipes(new TrimObjectPipe(), new JoiValidationPipe(CreateUserSchema))
  async create(@Req() req: AuthenticatedRequest, @Body() dto: CreateUser) {
    const user = await this.userService.create({
      ...dto,
      organId: req.session.organId,
      createdBy: req.user.id,
    });
    return new SuccessResponse(
      user,
      this.i18n.t("common.user.createdSuccessfully")
    );
  }
}
```

### 3. Service Pattern
```typescript
@Injectable()
@UseTenantFilter()
export class UserService extends BaseService {
  constructor(
    private readonly userRepository: UserRepo,
    private readonly roleRepo: RoleRepo,
    i18n: I18nService,
    private readonly dataSource: DataSource
  ) {
    super(i18n);
  }

  async create(dto: CreateUser): Promise<User> {
    return await this.dataSource.transaction(async (manager: EntityManager) => {
      // Validation
      if (dto.email) {
        const userByEmail = await this.userRepository.findOneByEmail(dto.email);
        if (userByEmail) {
          throw new BadRequestException({
            message: this.i18n.t("common.user.emailAlreadyExists"),
            errors: [{
              key: "email",
              message: this.i18n.t("common.user.emailAlreadyExists"),
            }],
          });
        }
      }

      // Create user
      const user = await manager.save(User, payload);

      // Additional operations
      await this.createRelatedEntities(manager, user.id, dto);

      return user;
    });
  }
}
```

### 4. Repository Pattern
```typescript
@Injectable()
export class UserRepo {
  constructor(
    @InjectRepository(User)
    private readonly repository: Repository<User>
  ) {}

  async findAll(organId: string): Promise<User[]> {
    return await this.repository.find({
      where: { organId },
      relations: ["primaryDepartment", "concurrentDepartments"],
    });
  }

  async findOne(id: string): Promise<User | null> {
    return await this.repository.findOne({
      where: { id },
      relations: ["primaryDepartment", "departmentRole"],
    });
  }
}
```

### 5. Validation Pattern
```typescript
import Joi from "@common/plugins/joi";

export const CreateUserSchema = Joi.object({
  firstName: Joi.string().required(),
  lastName: Joi.string().required(),
  email: Joi.string().email().optional(),
  phone: Joi.string().required(),
  dateOfBirth: Joi.date().optional(),
  departmentId: Joi.string().uuid().optional(),
  roleId: Joi.string().uuid().optional(),
});

export const UpdateUserSchema = Joi.object({
  firstName: Joi.string().optional(),
  lastName: Joi.string().optional(),
  email: Joi.string().email().optional(),
});
```

### 6. Entity Pattern
```typescript
@Entity("users")
export class User {
  @PrimaryGeneratedColumn("uuid")
  id: string;

  @Column({ type: "varchar", length: 255 })
  firstName: string;

  @Column({ type: "varchar", length: 255 })
  lastName: string;

  @Column({ type: "varchar", length: 255, nullable: true })
  email: string | null;

  @Column({ type: "varchar", length: 20 })
  phone: string;

  @Column({ type: "uuid" })
  organId: string;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @ManyToOne(() => Department)
  @JoinColumn({ name: "departmentId" })
  primaryDepartment?: Department;

  @OneToMany(() => UserRole, (userRole) => userRole.user)
  userRoles?: UserRole[];
}
```

### 7. Migration Pattern
```typescript
export class CreateUsersTable1700000000001 implements MigrationInterface {
  name = "CreateUsersTable1700000000001";

  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.createTable(
      new Table({
        name: "users",
        columns: [
          {
            name: "id",
            type: "uuid",
            isPrimary: true,
            generationStrategy: "uuid",
            default: "uuid_generate_v4()",
          },
          {
            name: "firstName",
            type: "varchar",
            length: "255",
          },
          // ... other columns
        ],
      }),
      true
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropTable("users");
  }
}
```

### 8. API Response Pattern
```typescript
// Success response
export class SuccessResponse<T> {
  code: number;
  message: string;
  data: T;
  errors: any[];

  constructor(data: T, message: string = "Success") {
    this.code = 200;
    this.message = message;
    this.data = data;
    this.errors = [];
  }
}

// List response
export class SuccessListResponse<T> {
  code: number;
  message: string;
  data: T[];
  total: number;
  errors: any[];

  constructor(data: T[], total: number, message: string = "Success") {
    this.code = 200;
    this.message = message;
    this.data = data;
    this.total = total;
    this.errors = [];
  }
}
```

### 9. Error Handling Pattern
```typescript
throw new BadRequestException({
  message: this.i18n.t("common.user.notFound"),
  errors: [
    {
      key: "id",
      message: this.i18n.t("common.user.notFound"),
      errorCode: HttpStatus.BAD_REQUEST,
    },
  ],
});
```

### 10. Many-to-Many Join Table Pattern
```typescript
// Join table entity
@Entity('contract_allowances')
export class ContractAllowance extends BaseEntity {
  @Column({ type: 'uuid' })
  contractId: string;

  @Column({ type: 'uuid' })
  allowanceId: string;

  @ManyToOne(() => Contract, (contract) => contract.contractAllowances, {
    onDelete: 'CASCADE',
  })
  @JoinColumn({ name: 'contractId' })
  contract: Contract;

  @ManyToOne(() => Allowance, (allowance) => allowance.contractAllowances, {
    onDelete: 'RESTRICT',
  })
  @JoinColumn({ name: 'allowanceId' })
  allowance: Allowance;
}

// Usage in service
if (dto.allowanceIds && dto.allowanceIds.length > 0) {
  const contractAllowances = dto.allowanceIds.map((allowanceId) => ({
    contractId: contract.id,
    allowanceId,
    organId,
    createdBy,
  }));
  await manager.getRepository(ContractAllowance).save(contractAllowances);
}
```

### 11. Template Variable Replacement Pattern
```typescript
@Injectable()
export class ContractTemplateService {
  constructor(
    @InjectRepository(Upload)
    private readonly uploadRepo: Repository<Upload>,
    @InjectRepository(Allowance)
    private readonly allowanceRepo: Repository<Allowance>
  ) {}

  async parseTemplate(fileBuffer: Buffer): Promise<string> {
    const result = await mammoth.extractRawText({ buffer: fileBuffer });
    return result.value;
  }

  replacePlaceholders(content: string, variables: Record<string, any>): string {
    return content.replace(/\$\{(\w+)\}/g, (match, key) => {
      return variables[key] !== undefined ? variables[key] : match;
    });
  }

  async generateContent(
    templateFileId: string,
    allowanceIds: string[]
  ): Promise<{ content: string; variables: Record<string, any> }> {
    const templateFile = await this.uploadRepo.findOne({
      where: { id: templateFileId },
    });
    
    const templatePath = path.join(process.cwd(), 'uploads', templateFile.filePath);
    const fileBuffer = await fs.promises.readFile(templatePath);
    const templateContent = await this.parseTemplate(fileBuffer);

    const variables: Record<string, any> = {};
    if (allowanceIds.length > 0) {
      const allowances = await this.allowanceRepo.find({
        where: { id: In(allowanceIds) },
      });
      allowances.forEach((allowance) => {
        if (allowance.key) {
          variables[allowance.key] = allowance.name;
          variables[`${allowance.key}Amount`] = allowance.amount;
        }
      });
    }

    const content = this.replacePlaceholders(templateContent, variables);
    return { content, variables };
  }
}
```

### 12. Unit Testing Pattern
```typescript
describe("UserService", () => {
  let service: UserService;
  let userRepo: any;
  let i18nService: any;

  beforeEach(async () => {
    userRepo = {
      create: jest.fn(),
      findById: jest.fn(),
      update: jest.fn(),
    };

    i18nService = {
      t: jest.fn((key) => key),
    };

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UserService,
        { provide: UserRepo, useValue: userRepo },
        { provide: I18nService, useValue: i18nService },
        { provide: DataSource, useValue: {} },
      ],
    }).compile();

    service = module.get<UserService>(UserService);
  });

  it("should create a user", async () => {
    const dto = { firstName: "John", lastName: "Doe" };
    const mockUser = { id: "uuid", ...dto };

    userRepo.create.mockResolvedValue(mockUser);

    const result = await service.create(dto as any);

    expect(userRepo.create).toHaveBeenCalledWith(expect.objectContaining(dto));
    expect(result).toEqual(mockUser);
  });

  it("should throw NotFoundException when user not found", async () => {
    userRepo.findById.mockResolvedValue(null);

    await expect(service.findById("invalid-id")).rejects.toThrow(
      NotFoundException
    );
  });
});
```

## Best Practices

### Code Organization
- ✅ Use feature.module.ts per domain
- ✅ Organize by domain not layer
- ✅ Keep files small and focused
- ✅ Use dependency injection
- ✅ Always use path aliases (@common/*, @modules/*, @entities/*)
- ❌ Avoid relative paths

### Code Quality
- ✅ Use TypeScript strictly, avoid `any`
- ✅ PascalCase for classes, camelCase for variables/functions
- ✅ Keep functions under 50 lines
- ✅ Write pure functions when possible
- ✅ No unused imports or commented code
- ✅ Run `npm run lint && npm run test` before pushing
- ❌ No console.log, use Logger

### NestJS Patterns
- ✅ Always use async/await
- ✅ Define DTOs with class-validator
- ✅ Throw HttpException with meaningful messages
- ✅ Use Guards for auth, Interceptors for logging
- ✅ Use Nest's Logger instead of console.log
- ❌ Don't return plain error strings
- ❌ Never log sensitive data

### Database
- ✅ Use strict typing and explicit column types
- ✅ Always create migrations for schema changes
- ✅ Wrap complex operations in transactions
- ✅ Define relations clearly with cascade options
- ❌ Avoid nullable unless necessary

### Security
- ✅ Always hash passwords (bcrypt, argon2)
- ✅ Validate & sanitize all external input
- ✅ Use environment variables via @nestjs/config
- ✅ Apply guards for authentication/authorization
- ❌ Never expose sensitive data

### Testing
- ✅ Keep coverage for core logic > 70%
- ✅ Write unit tests for services
- ✅ Mock external services (DB, APIs)
- ✅ Use describe/it/expect consistently
- ❌ Don't test implementation details

### API Design
- ✅ Follow REST conventions
- ✅ Use plural nouns for resources
- ✅ Always document APIs with @nestjs/swagger
- ✅ Implement pagination for lists
- ✅ Use eager loading judiciously
- ❌ Don't over-fetch data

### Internationalization
- ✅ Use `I18nService` for all user-facing messages
- ✅ Organize translations by feature
- ✅ Never hardcode user-facing strings
- ❌ Don't return untranslated error messages

## Pre-PR Checklist
- [ ] No `console.log`, use `Logger`
- [ ] DTOs & validation for all inputs
- [ ] Error handling via exceptions, not raw responses
- [ ] Unit/e2e tests updated or added
- [ ] Swagger docs updated if API changed
- [ ] No unused imports or commented-out code
- [ ] Use path aliases (@common, @modules, @entities)
- [ ] ESLint and Prettier checks pass
- [ ] Test coverage meets requirements
- [ ] All user-facing strings use i18n
- [ ] Transactions for complex operations
- [ ] Proper error handling with structured responses

## Common Anti-Patterns to Avoid
- ❌ `console.log` → Use `Logger`
- ❌ Plain strings for errors → Use structured exceptions
- ❌ Direct repository calls in controllers → Use services
- ❌ Missing validation → Use Joi schemas
- ❌ `any` types → Use proper TypeScript types
- ❌ Missing i18n → Use `I18nService`
- ❌ No transactions → Wrap complex operations
- ❌ Relative imports → Use path aliases

## Quick Reference

### Import Patterns
```typescript
// Entities
import { User } from '@entities/user.entity';

// Common
import { SuccessResponse, SuccessListResponse } from '@common/utils/api.response';
import { I18nService } from 'nestjs-i18n';

// NestJS
import { Controller, Get, Post, Body, Req, UseGuards } from '@nestjs/common';
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, DataSource, EntityManager } from 'typeorm';
```

### Controller Decorators
```typescript
@Controller("users")
@UseGuards(AuthenticatedGuard)
@ApiTags("Users")
@ApiCookieAuth()
export class UserController { }
```

### Service with Transaction
```typescript
async create(dto: CreateDto): Promise<Entity> {
  return await this.dataSource.transaction(async (manager: EntityManager) => {
    // Validation logic
    
    // Create entity
    const entity = await manager.save(Entity, payload);
    
    // Related operations
    
    return entity;
  });
}
```

### Swagger Documentation
```typescript
@ApiOperation({ summary: "List users" })
@ApiResponse({ status: 200, description: "Users retrieved successfully" })
@Get()
async list() { }
```

### Error Handling
```typescript
throw new BadRequestException({
  message: this.i18n.t("common.error.notFound"),
  errors: [{
    key: "id",
    message: this.i18n.t("common.error.invalid"),
    errorCode: HttpStatus.BAD_REQUEST,
  }],
});
```

---

When implementing backend features, always follow these patterns and best practices to maintain consistency, security, and quality across the codebase.
