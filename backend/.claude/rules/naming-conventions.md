# Naming conventions

Follow these rules to keep the codebase consistent and IDE-search-friendly.

## 1 — Feature modules

| Aspect           | Rule                                                                              |
| ---------------- | --------------------------------------------------------------------------------- |
| Folder name      | **kebab-case**, singular or plural matching the resource (`event/`, `enrollees/`) |
| Module class     | **PascalCase** + `Module` suffix (`EventModule`)                                  |
| Controller class | **PascalCase** + `Controller` suffix (`EventController`)                          |
| Service class    | **PascalCase** + `Service` suffix (`EventService`)                                |
| Controller route | Singular or plural matching the folder (`@Controller("event")`)                   |

## 2 — Files

| File type        | Naming pattern                                       | Example                          |
| ---------------- | ---------------------------------------------------- | -------------------------------- |
| Module           | `<feature>.module.ts`                                | `event.module.ts`                |
| Controller       | `<feature>.controller.ts`                            | `event.controller.ts`            |
| Service          | `<feature>.service.ts`                               | `event.service.ts`               |
| Service spec     | `<feature>.service.spec.ts`                          | `event.service.spec.ts`          |
| Create DTO       | `create-<feature>.dto.ts`                            | `create-event.dto.ts`            |
| Update DTO       | `update-<feature>.dto.ts`                            | `update-event.dto.ts`            |
| Sub-object DTO   | `<sub-object>.dto.ts`                                | `location.dto.ts`                |
| Mongoose schema  | `<entity>.schema.ts`                                 | `event.schema.ts`                |
| Common decorator | `<name>.decorator.ts`                                | `idempotency-key.decorator.ts`   |
| Common service   | `<name>.service.ts`                                  | `sqs.service.ts`                 |
| Common type      | `<name>.type.ts`                                     | `email.payload.type.ts`          |
| Exception filter | `<scope>-exception.filter.ts` or `<scope>.filter.ts` | `http-exception.filter.ts`       |
| Guard            | `<name>.guard.ts`                                    | `jwt.guard.ts`, `admin.guard.ts` |
| Strategy         | `<name>.strategy.ts`                                 | `jwt.strategy.ts`                |

All file names use **kebab-case** with dot-separated type suffixes.

## 3 — Classes and types

| Entity                   | Naming                                                           | Export style   |
| ------------------------ | ---------------------------------------------------------------- | -------------- |
| DTO classes              | **PascalCase** + descriptive suffix (`CreateEventDto`)           | **named**      |
| Mongoose schema classes  | **PascalCase** entity name (`Event`, `Workshop`)                 | **named**      |
| Document types           | **PascalCase** + `Document` (`EventDocument`)                    | **named** type |
| Schema factory constants | **PascalCase** + `Schema` (`EventSchema`)                        | **named**      |
| Guard classes            | **PascalCase** + `Guard` (`AdminGuard`, `JwtAuthGuard`)          | **named**      |
| Filter classes           | **PascalCase** + `Filter` (`HttpExceptionFilter`)                | **named**      |
| Custom decorators        | **PascalCase** function names (`Public`, `CurrentUser`, `IsRPC`) | **named**      |
| Shared types             | **PascalCase** (`ErrorResponse`, `EmailSQSPayload`, `User`)      | **named** type |
| Metadata key constants   | **SCREAMING_SNAKE_CASE** (`IS_PUBLIC_KEY`, `IS_RPC_KEY`)         | **named**      |

## 4 — Methods

| Context              | Naming                                                                              |
| -------------------- | ----------------------------------------------------------------------------------- |
| Service CRUD methods | **camelCase** verb + entity (`createEvent`, `getAllPublicEvents`, `deleteWorkshop`) |
| Controller methods   | **camelCase** matching or mirroring the service method name                         |
| Private helpers      | **camelCase** prefixed with the method's purpose (`resolvePath`)                    |

## 5 — Enums

| Aspect       | Rule                                                                                                      |
| ------------ | --------------------------------------------------------------------------------------------------------- |
| Enum name    | **PascalCase** (`EventType`, `Role`)                                                                      |
| Enum members | **SCREAMING_SNAKE_CASE** keys with **PascalCase** string values for display enums (`GENERAL = "General"`) |
|              | **lowercase** string values for role-like enums (`ADMIN = "admin"`)                                       |

## 6 — Validation & error messages

- All `class-validator` message strings and NestJS exception messages use **kebab-case** (e.g., `"please-provide-a-name"`, `"event-does-not-exist"`).
- All Mongoose schema `required` and `validate.message` strings use the same kebab-case convention.
- Success response strings follow the same pattern (e.g., `"event-created-successfully"`).
