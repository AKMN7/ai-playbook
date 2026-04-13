# Feature workflow guide

This document outlines the **canonical flow** for implementing CRUD operations and new feature modules.
Stick to it, and always stay consistent.

Every feature follows the **module → schema → DTO → service → controller** pipeline. All business logic lives in the service; the controller is a thin dispatcher.

## 1 — Module setup

| Step                  | Responsibility                                                                                                 | File                                |
| --------------------- | -------------------------------------------------------------------------------------------------------------- | ----------------------------------- |
| **1. Module**         | Create a NestJS module that declares the controller and service as providers.                                  | `src/<feature>/<feature>.module.ts` |
| **2. Register**       | Import the new module in `AppModule`.                                                                          | `src/app.module.ts`                 |
| **3. Schema binding** | If the feature introduces a new Mongoose model, register it in `MongooseModule.forFeature` inside `AppModule`. | `src/app.module.ts`                 |

## 2 — Schema (Mongoose model)

| Step                       | Responsibility                                                                              | File                                        |
| -------------------------- | ------------------------------------------------------------------------------------------- | ------------------------------------------- |
| **1. Define class**        | Create a class decorated with `@Schema({ timestamps: true })` and `@Prop()` for each field. | `src/<feature>/schemas/<feature>.schema.ts` |
| **2. Document type**       | Export `type <Feature>Document = HydratedDocument<<Feature>>`.                              | same file                                   |
| **3. SchemaFactory**       | Export `const <Feature>Schema = SchemaFactory.createForClass(<Feature>)`.                   | same file                                   |
| **4. Indexes**             | Add compound or unique indexes after the schema factory call if needed.                     | same file                                   |
| **5. Validation messages** | All Mongoose `required` and `validate.message` strings use **kebab-case** translation keys. | same file                                   |

## 3 — DTOs (validation layer)

| Step               | Responsibility                                                                                            | File                                        |
| ------------------ | --------------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| **1. Create DTO**  | Define a class with `class-validator` decorators. Each validator's `message` is a kebab-case string.      | `src/<feature>/dto/create-<feature>.dto.ts` |
| **2. Update DTO**  | Extend `PartialType(Create<Feature>Dto)` from `@nestjs/mapped-types`. Nothing else needed.                | `src/<feature>/dto/update-<feature>.dto.ts` |
| **3. Nested DTOs** | If a field is an embedded object, create a separate DTO and use `@ValidateNested()` + `@Type(() => ...)`. | `src/<feature>/dto/<sub-object>.dto.ts`     |
| **4. Shared DTOs** | Reusable DTOs that span multiple features live in `src/common/`.                                          | `src/common/<name>.dto.ts`                  |

## 4 — Service (business logic)

| Step                      | Responsibility                                                                                                                                              | File                                 |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ |
| **1. Injectable**         | Decorate the class with `@Injectable()`.                                                                                                                    | `src/<feature>/<feature>.service.ts` |
| **2. Constructor DI**     | Inject Mongoose models via `@InjectModel(<Entity>.name)`, plus any needed services (`ConfigService`, `SqsService`, etc.).                                   | same file                            |
| **3. Method per action**  | One `async` method per CRUD action (`create<Feature>`, `get<Feature>`, `update<Feature>`, `delete<Feature>`).                                               | same file                            |
| **4. Comment every step** | Add a `// <Verb> <what>` comment before each logical block inside a method.                                                                                 | same file                            |
| **5. Exceptions**         | Throw NestJS built-in exceptions (`NotFoundException`, `BadRequestException`, etc.) with kebab-case message strings.                                        | same file                            |
| **6. Success returns**    | Return a plain kebab-case string on success (e.g., `"event-created-successfully"`).                                                                         | same file                            |
| **7. Idempotency**        | For create operations, accept an `idempotencyKey` parameter, check for existing records, and rely on a unique MongoDB index as a race-condition safety net. | same file                            |
| **8. Transactions**       | Use Mongoose sessions for multi-document writes (e.g., cascading deletes). Always wrap in try/catch/finally to abort on error and end the session.          | same file                            |

## 5 — Controller (route handlers)

| Step                       | Responsibility                                                                                                                             | File                                    |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------- |
| **1. Route prefix**        | Set the resource name in `@Controller("<feature>")`.                                                                                       | `src/<feature>/<feature>.controller.ts` |
| **2. Constructor DI**      | Inject only the feature's own service.                                                                                                     | same file                               |
| **3. Decorators**          | Use `@Public()` for unauthenticated endpoints, `@UseGuards(AdminGuard)` for admin-only, `@IsRPC()` + `@MessagePattern()` for TCP handlers. | same file                               |
| **4. User extraction**     | Use `@CurrentUser() user: User` (or `_user` if unused) to get the authenticated user.                                                      | same file                               |
| **5. Idempotency**         | Use `@IdempotencyKey()` decorator on create endpoints to extract and validate the header.                                                  | same file                               |
| **6. Delegate to service** | Controller methods should contain zero business logic—only call `this.<feature>Service.<method>(...)`.                                     | same file                               |

## 6 — Testing

| Step                     | Responsibility                                                      | File                                      |
| ------------------------ | ------------------------------------------------------------------- | ----------------------------------------- | ---------- | -------------------- | --------- |
| **1. Spec file**         | Create a colocated `<feature>.service.spec.ts` next to the service. | `src/<feature>/<feature>.service.spec.ts` |
| **2. Test module**       | Use `Test.createTestingModule` to set up providers.                 | same file                                 |
| **3. Naming convention** | Test descriptions follow: `"<##>                                    | {<method>}                                | <scenario> | <pass/fail emoji>"`. | same file |
