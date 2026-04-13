# {REPO_NAME}

Back-end {SERVICE_NAME} for the {PROJECT_NAME} {BREIF_DESC}.

## Top priority: consistency

**Consistency is the highest priority** for any change in this repo. When implementing a task, do not invent a new style or pattern—mirror what already exists.

Before writing or editing code, read nearby files in the same feature or layer and **match them**—**literally everything**: module structure and file placement, naming, exports, TypeScript and NestJS patterns, service and controller logic flow (how guards, decorators, DTOs, services, and schemas are split), commenting style and frequency, error handling and exception usage, and any other habit the surrounding code establishes.

When asked to do something, **default to the current structure** of the repo and that feature; do not introduce a parallel style. If something is ambiguous, prefer **consistency with the surrounding codebase** over a "cleaner" or generic alternative. The written rules in `.claude/rules/*.md` apply; where they are silent, **follow local precedent** in the files you touch.

**In short:** match the house style of the code and files next to your change—spacing, style, comments, logic, structure, and placement included.

## Tech stack

- **NestJS** + **TypeScript** (ES2023 target, `strictNullChecks`)
- **Mongoose** + **@nestjs/mongoose** — MongoDB ODM with decorator-based schemas
- **Passport** + **@nestjs/jwt** — JWT authentication (Clerk-issued tokens)
- **class-validator** + **class-transformer** — DTO validation via decorators
- **@nestjs/throttler** — rate limiting (global guard)
- **@nestjs/microservices** — TCP transport for inter-service communication
- **AWS SQS** (`@aws-sdk/client-sqs`) — async queues
- **Sentry** (`@sentry/nestjs`) — error monitoring and tracing
- **Helmet** — HTTP security headers
- **bcrypt** — hashing
- **date-fns-tz** — timezone-aware date conversion

## Directory map (`src/`)

| Path                                  | Purpose                                                                                                                                   |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `common/`                             | Shared enums, types, decorators, and services used across feature modules                                                                 |
| `filters/`                            | Global exception filters (HTTP, Mongo, RPC, catch-all)                                                                                    |
| `strategies/`                         | JWT strategy, auth guards (`JwtAuthGuard`, `AdminGuard`, `PermissionsGuard`), and custom decorators (`@Public`, `@CurrentUser`, `@IsRPC`) |
| `<feature>/`                          | One folder per feature module (e.g., `event/`, `workshop/`, `enrollees/`)                                                                 |
| `<feature>/dto/`                      | `create-<feature>.dto.ts` and `update-<feature>.dto.ts`                                                                                   |
| `<feature>/schemas/`                  | Mongoose schema classes and their `SchemaFactory` exports                                                                                 |
| `<feature>/<feature>.controller.ts`   | HTTP + RPC route handlers                                                                                                                 |
| `<feature>/<feature>.service.ts`      | Business logic                                                                                                                            |
| `<feature>/<feature>.module.ts`       | NestJS module wiring                                                                                                                      |
| `<feature>/<feature>.service.spec.ts` | Unit tests (abandoned)                                                                                                                    |

Conventions and workflows live in `.claude/rules/*.md` (loaded automatically with this file).
