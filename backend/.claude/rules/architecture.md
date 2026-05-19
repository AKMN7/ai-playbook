# Architecture, framework stack, and naming

This file is the structural source of truth for the backend. It covers three intertwined concerns: the modular-monolith architecture and directory layout (Part A), the NestJS framework and default library stack including Dockerization (Part B), and the naming conventions that bind both together (Part C).

Structural consistency is a core requirement, not a preference. New work follows this file exactly. Edits inside an existing area match the surrounding local pattern.

---

## Part A — Project structure

### A.1 — Architecture baseline

- The backend is a **modular monolith**. One repository, one runnable service, many internal feature modules.
- Do not introduce microservices, service-per-domain splits, RPC-first decomposition, or mixed architecture styles unless the user explicitly requests an architecture change.
- Each feature directory represents **one business domain or aggregate root**.
- Nested resources live inside the owning feature when they share the same parent aggregate, authorization boundary, or persistence boundary. Only promote a child to a top-level feature when the domain is meaningfully separate.
- Cross-cutting code is separated from domain code into clearly named shared areas (`common/`, `filters/`, `modules/auth/`, `modules/integration/`).
- Authentication and authorization have a dedicated area.
- Third-party integrations have a dedicated area.
- Transport-level error shaping has a dedicated area.
- Root application files focus on bootstrap and composition only, not domain logic.

### A.2 — Preferred directory layout

Use this as the default target shape for new backend projects and for all new work inside an existing repository that has not explicitly diverged:

```text
src/
  main.ts
  app.module.ts
  app.controller.ts
  app.service.ts

  common/
    constants/
    decorators/
    dto/
    types/
    utils/

  filters/

  modules/
    auth/
      decorators/
      guards/
      strategies/
      types/
      auth.module.ts

    integration/
      integration.module.ts
      <vendor>.service.ts

    <feature>/
      dto/
      schemas/
      <feature>.module.ts
      <feature>.controller.ts
      <feature>.service.ts
      <feature>.guard.ts

      <child-resource>/
        dto/
        schemas/
        <child-resource>.controller.ts
        <child-resource>.service.ts
        <child-resource>.guard.ts
```

- Treat this as the **default target shape**, not a suggestion.
- For new areas, follow this layout exactly unless the repository has a clearly established equivalent NestJS pattern that supersedes it.
- For edits inside an existing area, match the surrounding local structure exactly. Do not restructure a feature while completing an unrelated task.
- Avoid horizontal top-level folders such as `controllers/`, `services/`, or `models/`. Prefer feature colocation.

### A.3 — Global conventions

- **Language** — TypeScript only (`.ts`). No `.js` files in `src/`.
- **Exports** — Named exports only. No default exports.
- **Imports** — Use the repository's established path alias for cross-module imports (commonly `src/` prefix). Use relative paths (`./`, `../`) inside a small local feature when that is the repository standard. Use `import type` for type-only imports.
- **Barrel files** — Avoid barrel `index.ts` files unless the repository already uses them intentionally.
- **File naming** — kebab-case with dot-separated type suffixes (`create-event.dto.ts`, `event.schema.ts`, `idempotency-key.decorator.ts`). See Part C below.

### A.4 — Root application files

| File                | Purpose                                                                                                                                   |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `main.ts`           | Bootstrap: creates the Nest app, applies global pipes, middleware (Helmet, CORS, body parsers, cookie-parser when used), starts server    |
| `app.module.ts`     | Top-level composition: infrastructure imports (ConfigModule, MongooseModule root, ThrottlerModule), all feature modules, global providers |
| `app.controller.ts` | Health-check endpoint only                                                                                                                |
| `app.service.ts`    | Health-check logic only                                                                                                                   |
| `instruments.ts`    | Sentry (or equivalent monitoring) SDK initialization — imported **at the top** of `main.ts` before any framework imports                  |

- `main.ts` must not contain feature business logic.
- `app.module.ts` must not contain feature-specific business rules. It composes; it does not implement.
- `app.controller.ts` / `app.service.ts` are not a second domain layer. Do not grow them.

### A.5 — Module design rules

- Each feature owns its own controller, service, DTOs, schemas, module wiring, and any feature-local guards.
- **Register Mongoose models inside the owning feature module** via `MongooseModule.forFeature([{ name: Entity.name, schema: EntitySchema }])`. Do not register feature models globally in `AppModule`.
- Export only the services other modules actually need (`exports: [FeatureService]`). When one feature needs another's behavior, depend on the **exported service**, never on internal files, schemas, or repositories.
- Avoid exporting raw schemas or models across modules unless the repository already standardizes that pattern.
- Keep child resources under the parent feature when they are not independently meaningful at the top level.
- Avoid giant feature modules that mix unrelated domains. Split by aggregate, not by convenience.
- Feature-local helpers that only make sense inside one feature stay inside that feature, not in `common/`.
- If a feature needs its own guard (resource-scoped authorization), keep the guard inside the feature, not in the global auth area.

### A.6 — Registrations in `AppModule`

- **Global guards** registered via `APP_GUARD` providers. Standard registration order (first registered runs first):
    1. `ThrottlerGuard`
    2. `JwtAuthGuard`
- **Global exception filters** registered via `APP_FILTER` providers. Registration order determines priority — **last registered wins**. See [style-and-errors.md §B.4 — Global exception filters](style-and-errors.md).
- **Global pipes** registered via `APP_PIPE` or in `main.ts` via `app.useGlobalPipes(...)`. The project standard is the NestJS `ValidationPipe` with `{ transform: true, whitelist: true, forbidNonWhitelisted: true }`.
- **Infrastructure modules** (`ConfigModule`, `MongooseModule.forRoot(...)`, `ThrottlerModule`, the auth module, the integration module) are imported at the root.
- **Feature modules** are imported at the root. Feature modules themselves register their own schemas and providers.

### A.7 — Shared code rules (`common/`)

- `common/` is for **true cross-module reuse**. It is not a dumping ground.
- Organize shared code by intent:
    - `common/constants/` — cross-cutting literal values and metadata keys.
    - `common/decorators/` — parameter decorators and metadata decorators (`@Public`, `@CurrentUser`, `@IdempotencyKey`).
    - `common/dto/` — DTOs reused by multiple features.
    - `common/types/` — shared TypeScript types, enums, payload types, response types (`error-response.type.ts`, `email.payload.type.ts`).
    - `common/utils/` — framework-agnostic helpers (date conversion, formatting, hashing adapters).
- Do not place feature-specific business logic in `common/`.
- Do not promote code to `common/` just because two files look similar once. Promote only when reuse is real, stable, and clearly beneficial.
- Keep `common/` shallow. Avoid deep sub-hierarchies.
- Shared auth concerns go in `modules/auth/`. Shared vendor concerns go in `modules/integration/`.

### A.8 — Filters area (`filters/`)

- Holds global exception filters registered via `APP_FILTER` in `AppModule`.
- One filter per concern: `http-exception.filter.ts`, `mongo-exception.filter.ts`, `all-exceptions.filter.ts`, and any transport-specific filters the project genuinely uses.
- All filters produce the consistent `ErrorResponse` shape defined in `common/types/error-response.type.ts`.

### A.9 — Auth area (`modules/auth/`)

- Holds the JWT strategy, guards, auth-specific decorators, and the normalized authenticated user type.
- `decorators/` — `public.decorator.ts`, `user.decorator.ts`, and related metadata decorators.
- `guards/` — `jwt.guard.ts`, `admin.guard.ts`, and other shared auth guards.
- `strategies/` — `jwt.strategy.ts` and any additional Passport strategies.
- `types/` — `user.type.ts` and related auth-specific types.
- See [boundaries.md Part A — Authentication and authorization](boundaries.md) for the full contract.

### A.10 — Integration area (`modules/integration/`)

- Single place for all third-party SDK access and raw HTTP integration code.
- One vendor per service file: `sqs.service.ts`, `s3.service.ts`, `email.service.ts`, `clerk.service.ts`, etc.
- Domain services depend on integration services. Controllers and domain services never talk to vendor SDKs directly.
- See [boundaries.md Part B — Integrations and side effects](boundaries.md) for the full contract.

### A.11 — Change-safety rules

- Do not add a second architectural style beside the existing one.
- Do not create new top-level folders when an existing feature or shared area is the correct home.
- Do not move feature models, schemas, or feature authorization rules into `common/`.
- Do not move business logic into controllers, guards, filters, or decorators.
- Do not rename route params, DTO field names, env keys, or public error messages without auditing downstream usage with Grep first.
- Do not mix unrelated refactors into a feature task. Flag out-of-scope cleanup separately.

---

## Part B — Framework and library stack

Backend projects governed by this standard run on NestJS. This part defines the default framework, the default library stack for each concern, and the Dockerization standard. Do not swap defaults casually. When a default exists for a concern, use it first unless the user explicitly requires a different choice or the repository already has a deliberate established alternative.

### B.1 — Framework

- **NestJS** is the backend framework. Use NestJS official patterns for modules, controllers, providers, guards, pipes, decorators, interceptors, and exception filters.
- Do not build new backend features in raw Express, Fastify (without NestJS), Hono, Adonis, Next.js API routes, tRPC servers, or custom ad hoc server structures.
- Do not introduce a second backend framework or a parallel framework stack alongside NestJS.
- When framework behavior is in question, prefer NestJS official documentation over memory or blogs. WebFetch to docs rather than guessing lifecycle, DI, or decorator behavior.

### B.2 — Core application stack

- **`typescript`** — the backend is written entirely in TypeScript. No `.js` in `src/`.
- **`@nestjs/common`, `@nestjs/core`, `@nestjs/platform-express`** — default NestJS application + HTTP platform stack.
- **`reflect-metadata`, `rxjs`** — framework-level runtime dependencies. Treat them as infrastructure, not optional libraries.
- Target modern `ES2023` (or the repository's established target) with `strictNullChecks` enabled.

### B.3 — Configuration

- **`@nestjs/config`** — default configuration and environment-access layer.
- Centralize config loading. Use `ConfigService.getOrThrow(...)` (or the equivalent) for required values — fail fast.
- Do not read `process.env` directly from controllers, services, or domain code. Integration and infrastructure layers may read through `ConfigService`; domain code must not.
- See [boundaries.md Part C — Configuration, environment, and dates](boundaries.md).

### B.4 — Persistence

- **`mongoose` + `@nestjs/mongoose`** — default persistence stack.
- Register models inside the **owning feature module** via `MongooseModule.forFeature([...])`, never globally from `AppModule`.
- **`mongodb`** — include only when raw driver types or low-level behavior are actually needed. Do not default to raw driver usage in domain code.
- Do not introduce Prisma, TypeORM, Sequelize, Drizzle, or a second persistence stack unless the user explicitly requests a database/ORM change.
- See [feature-pipeline.md Part D — Persistence (MongoDB + Mongoose)](feature-pipeline.md).

### B.5 — Validation and transformation

- **`class-validator` + `class-transformer`** — default request validation and DTO transformation stack.
- **`@nestjs/mapped-types`** — default helper for update DTOs and derived DTOs (`PartialType`, `OmitType`, `PickType`).
- **`zod`** — allowed only as a _secondary_ validation library for dynamic, runtime-defined, or schema-composed payloads. Do not use Zod as a replacement for normal DTO validation.
- Global `ValidationPipe` is configured once with `{ transform: true, whitelist: true, forbidNonWhitelisted: true }`.
- See [feature-pipeline.md Part C — DTOs and validation](feature-pipeline.md).

### B.6 — Authentication and authorization

- **`passport`, `@nestjs/passport`, `passport-jwt`, `@nestjs/jwt`** — default JWT auth stack.
- **Clerk** — default external identity provider. Clerk issues tokens; the NestJS JWT strategy validates them using the configured Clerk signing key.
- Do not introduce a second identity provider, a parallel login system, or a second auth stack unless the user explicitly requests one.
- **`bcrypt`** — not part of the default baseline. Include it only when the backend manages local credentials that require password hashing. Do not add it proactively on Clerk-only projects.
- See [boundaries.md Part A — Authentication and authorization](boundaries.md).

### B.7 — Security, transport, and request controls

- **`helmet`** — default HTTP security header middleware. Apply in `main.ts` during bootstrap.
- **`cookie-parser`** — only when the backend actually reads cookies. Do not add it by default to header-only auth projects.
- **`@nestjs/throttler`** — default rate-limiting library. Register `ThrottlerGuard` globally via `APP_GUARD`. Prefer it over ad hoc custom throttling logic.

### B.8 — Dates, times, and time zones

- **`date-fns-tz`** — default timezone-aware date conversion library.
- Use it for business timezone conversion, date-only interpretation, and explicit timezone transforms.
- Do not rely on naive `new Date("YYYY-MM-DD")` parsing for business dates when timezone behavior matters.
- See [boundaries.md Part C — Configuration, environment, and dates](boundaries.md).

### B.9 — Integrations and external communication

- **`axios`** — default HTTP client when no better official SDK exists. Keep it inside the integration layer; do not use it from controllers or domain services.
- **`@aws-sdk/client-s3`, `@aws-sdk/client-sqs`, `@aws-sdk/s3-request-presigner`** — default AWS SDK family. Use AWS SDK v3 packages, not older monolithic SDK patterns.
- **`resend`** — default Resend client when Resend is the chosen email provider. Keep provider-specific code inside integration services.
- Add new vendor SDKs only inside `modules/integration/`. See [boundaries.md Part B — Integrations and side effects](boundaries.md).

### B.10 — Observability

- **`@sentry/nestjs`** — default error monitoring and tracing integration.
- Initialize Sentry in `instruments.ts`, imported at the **top** of `main.ts` before any framework imports.
- `AllExceptionsFilter` should use `@SentryExceptionCaptured()` (or equivalent) to auto-report unhandled errors.
- For caught errors in non-critical paths, log with NestJS `Logger` and call `captureException` manually.

### B.11 — Testing, linting, formatting

- **`jest`, `@nestjs/testing`, `ts-jest`** — default testing stack. Colocate unit tests as `<feature>.service.spec.ts`.
- **`eslint`, `typescript-eslint`, `prettier`** — default lint/format stack. Always run the repository's `npm run lint` and `npm run format` scripts rather than reinventing commands.

### B.12 — Not default unless explicitly needed

- **`@nestjs/microservices`** — not part of the default architecture. Do not introduce RPC handlers, transport wiring, or internal microservice decomposition unless the user explicitly requests an architecture change.
- **`ai`, `@ai-sdk/openai`, `openai`** — feature-specific. Add only when the project genuinely includes AI functionality, and keep it inside the integration layer.
- **GraphQL, gRPC, Kafka, Redis, message brokers** — not defaults. Introduce only when the project requires them.

### B.13 — Discovering the project's scripts

- Before running verification, discover the project's real scripts and tooling.
- Check `package.json` `scripts` (Read). Typical names:
    - `npm run start:dev` — dev server
    - `npm run start:prod` — production start
    - `npm run build` — compile
    - `npm run lint` / `npm run format`
    - `npm test` / `npm run test:watch` / `npm run test:e2e`
- Also check for `Makefile`, `justfile`, or CI config when scripts are not obvious.
- Use the project scripts. Do not invent ad hoc command lines when a script already exists.

### B.14 — Dockerization standard

Backend projects using this standard must be Dockerized.

#### B.14.1 Structure

- Keep Dockerization project-local with a repo-root `Dockerfile` and `.dockerignore`.
- One backend service produces **one main application image**. Do not pack multiple unrelated backend processes into one container.

#### B.14.2 Dockerfile style

- Default to a **simple, readable** Dockerfile.
- Prefer a single-stage build unless the project has a clear reason for multi-stage (smaller production image, separate build-only tooling, etc.).
- Use an **official Node.js LTS image** matching the project's runtime expectations.
- Default to `node:<lts>-slim` for compatibility and debuggability. Use Alpine only when image size materially matters and native dependencies are known to work there.
- **Copy dependency manifests first** (`package.json`, lockfile) to maximize layer caching. Then `npm ci`. Then copy the rest of the source.
- Use `npm ci` when a lockfile exists, not `npm install`.
- Copy only the files needed for build and runtime. Do not `COPY . .` when a narrower copy is clearly sufficient.
- Start the app with the repository's production start command (e.g., `CMD ["npm", "run", "start:prod"]`).

#### B.14.3 Secrets and environment

- Do not hardcode application environment variables in the Dockerfile unless the user explicitly asks for that.
- Keep secrets out of the image. Inject them at runtime through environment configuration or deployment tooling.

#### B.14.4 `.dockerignore`

Maintain a `.dockerignore` that excludes at least:

- `.git`
- `node_modules`
- local env files (`.env`, `.env.*`)
- coverage output (`coverage/`)
- local caches (`.cache/`, `.turbo/`, etc.)
- editor metadata (`.vscode/`, `.idea/`, `.DS_Store`)
- Claude artifacts (`.claude/`, `CLAUDE.md`) if they should not ship in the image

#### B.14.5 Compose and orchestration

- Add `docker-compose.yml` only when the backend genuinely needs local multi-service orchestration (database, queue, cache, emulator).
- Do not add Compose by default for a single backend service that runs fine with a plain `docker build` + `docker run`.

---

## Part C — Naming conventions

Naming is a core part of structural consistency. Follow these rules exactly for new work, and match local precedent in legacy areas. All naming must be IDE-search-friendly and unambiguous.

### C.1 — Files

- **All source files use kebab-case** with dot-separated type suffixes. The suffix communicates the file role.
- One class or primary export per file.
- No default exports. Named exports only.

| File type            | Pattern                                              | Example                              |
| -------------------- | ---------------------------------------------------- | ------------------------------------ |
| Module               | `<feature>.module.ts`                                | `event.module.ts`                    |
| Controller           | `<feature>.controller.ts`                            | `event.controller.ts`                |
| Service              | `<feature>.service.ts`                               | `event.service.ts`                   |
| Service spec         | `<feature>.service.spec.ts`                          | `event.service.spec.ts`              |
| Feature-local guard  | `<feature>.guard.ts`                                 | `workshop.guard.ts`                  |
| Create DTO           | `create-<feature>.dto.ts`                            | `create-event.dto.ts`                |
| Update DTO           | `update-<feature>.dto.ts`                            | `update-event.dto.ts`                |
| Purpose-specific DTO | `get-<purpose>.dto.ts`                               | `get-event-stats.dto.ts`             |
| Sub-object DTO       | `<sub-object>.dto.ts`                                | `location.dto.ts`                    |
| Mongoose schema      | `<entity>.schema.ts`                                 | `event.schema.ts`                    |
| Embedded sub-schema  | `<sub-entity>.schema.ts` (colocated in `schemas/`)   | `form-item.schema.ts`                |
| Decorator            | `<name>.decorator.ts`                                | `idempotency-key.decorator.ts`       |
| Integration service  | `<vendor>.service.ts`                                | `sqs.service.ts`, `email.service.ts` |
| Shared type          | `<name>.type.ts`                                     | `email-payload.type.ts`              |
| Exception filter     | `<scope>-exception.filter.ts` or `<scope>.filter.ts` | `http-exception.filter.ts`           |
| Guard (global)       | `<name>.guard.ts`                                    | `jwt.guard.ts`, `admin.guard.ts`     |
| Strategy             | `<name>.strategy.ts`                                 | `jwt.strategy.ts`                    |
| Constants            | `<name>.constants.ts` or `<name>.ts`                 | `metadata-keys.ts`                   |
| Utility              | `<name>.util.ts` or `<name>.ts`                      | `timezone.util.ts`                   |

- Match the established naming of the local area rather than imposing a second convention.

### C.2 — Folders

| Aspect                   | Rule                                                                                                        |
| ------------------------ | ----------------------------------------------------------------------------------------------------------- |
| Feature folder           | **kebab-case**, singular by default (`event/`, `workshop/`, `participant/`)                                 |
| Collection-style feature | Plural only when the repository already uses plural naming there (`enrollees/`)                             |
| Child resource folder    | Same convention as a top-level feature, nested under the parent (`event/workshop/`, `workshop/attendance/`) |
| Shared subfolders        | Fixed names: `constants/`, `decorators/`, `dto/`, `types/`, `utils/`, `schemas/`, `guards/`, `strategies/`  |

### C.3 — Classes and types

| Entity                  | Naming                                                               | Export     |
| ----------------------- | -------------------------------------------------------------------- | ---------- |
| Module class            | PascalCase + `Module` (`EventModule`)                                | named      |
| Controller class        | PascalCase + `Controller` (`EventController`)                        | named      |
| Service class           | PascalCase + `Service` (`EventService`)                              | named      |
| DTO class               | PascalCase + descriptive suffix (`CreateEventDto`, `UpdateEventDto`) | named      |
| Mongoose schema class   | PascalCase entity name (`Event`, `Workshop`, `Participant`)          | named      |
| Document type           | PascalCase + `Document` (`EventDocument`)                            | named type |
| Schema factory constant | PascalCase entity + `Schema` (`EventSchema`)                         | named      |
| Embedded schema class   | PascalCase (`FormItem`, `LocationEmbed`)                             | named      |
| Guard class             | PascalCase + `Guard` (`AdminGuard`, `JwtAuthGuard`)                  | named      |
| Filter class            | PascalCase + `Filter` (`HttpExceptionFilter`)                        | named      |
| Strategy class          | PascalCase + `Strategy` (`JwtStrategy`)                              | named      |
| Custom decorator        | PascalCase function (`Public`, `CurrentUser`, `IdempotencyKey`)      | named      |
| Shared type             | PascalCase (`ErrorResponse`, `EmailPayload`, `User`)                 | named type |
| Metadata key constant   | SCREAMING_SNAKE_CASE (`IS_PUBLIC_KEY`, `IDEMPOTENCY_KEY_HEADER`)     | named      |

- Use `export type` (or `import type`) for anything that is purely a type.
- Do not use default exports anywhere.

### C.4 — Methods

| Context                | Naming                                                                                |
| ---------------------- | ------------------------------------------------------------------------------------- |
| Service CRUD methods   | camelCase verb + entity (`createEvent`, `getEvent`, `updateEvent`, `deleteEvent`)     |
| Service query methods  | camelCase verb describing the query (`getAllPublicEvents`, `listEnrolleesByWorkshop`) |
| Controller methods     | camelCase mirroring the service method name                                           |
| Private helper methods | camelCase prefixed by the action (`resolvePath`, `buildLookup`, `assertOwnership`)    |
| Event handlers         | camelCase + handler verb (`handleSubscription`, `onPaymentSucceeded`)                 |

- Controller method names should align 1:1 with the service method they call.
- Avoid generic names like `handle`, `process`, or `execute` without a domain qualifier.

### C.5 — Enums

- Enum names: PascalCase (`EventType`, `Role`, `WorkshopStatus`).
- Enum member keys: SCREAMING_SNAKE_CASE.
- Enum member string values:
    - **PascalCase** for user-facing display enums (`GENERAL = "General"`, `WORKSHOP = "Workshop"`).
    - **lowercase** for role-like / identifier enums (`ADMIN = "admin"`, `USER = "user"`).
- Do not mix both casings within the same enum.
- One enum per concern per file. Group tightly related enums only when they are consumed together.

### C.6 — Validation, error, and success messages

- All `class-validator` message strings use **kebab-case** (`"please-provide-a-name"`, `"start-date-invalid-format"`).
- All Mongoose schema `required` tuples and `validate.message` strings use **kebab-case**.
- All NestJS exception messages (`NotFoundException`, `BadRequestException`, etc.) use **kebab-case** and behave as translation keys (`"event-does-not-exist"`, `"workshop-not-accepting-registrations"`).
- Success response strings from service write operations follow the same pattern (`"event-created-successfully"`, `"workshop-deleted-successfully"`).
- Do not casually rename any public-facing message (validation, error, success). Clients may depend on it — Grep for usages first.

### C.7 — Route paths and params

- Controller route prefix uses kebab-case and usually matches the folder name (`@Controller("event")`, `@Controller("workshop")`).
- Path params are camelCase when multi-word (`:workshopId`, `:eventId`).
- Keep param names stable. Guards and services may read them by name.
- Prefer nested route shapes that reflect resource structure (`/event/:eventId/workshop/:workshopId`).

### C.8 — Database naming

- Schema class names mirror domain naming (singular, PascalCase).
- Foreign-key fields follow `<parent>Id` exactly (`eventId`, `workshopId`, `participantId`). No variations like `event_id`, `eventID`, or `parentEvent`.
- Index names are auto-generated by Mongoose. If custom naming is required, use kebab-case.

### C.9 — Environment variables

- Environment variable names use **SCREAMING_SNAKE_CASE** (`MONGO_URI`, `CLERK_ADMINS_JWT_KEY`, `RESEND_API_KEY`).
- Group by vendor or concern prefix (`AWS_`, `CLERK_`, `RESEND_`, `SENTRY_`).
- Do not rename env vars casually. Audit `ConfigService.get(...)` usage and deployment configs first.
