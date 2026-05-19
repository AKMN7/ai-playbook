# Feature pipeline — workflow, controllers, services, DTOs, persistence

This file is the end-to-end contract for building any feature or child resource. It bundles the canonical pipeline (**schema → DTO → service → controller → module wiring → verification**) with the layer-specific contracts each step must honor.

All business logic lives in the service. The controller is a thin transport adapter. The module is composition only. DTOs validate _shape_ at the boundary; schemas protect _data quality_; services own _workflow_.

---

## Part A — Feature workflow

Before starting, use Read/Grep/Glob to inspect **at least one** nearby existing feature and mirror its structure, decorator usage, comment density, and error style. Do not invent a parallel shape.

### A.0 — Before you write code

- Use Read/Grep/Glob to inspect the nearest existing feature in `src/modules/` and copy its shape.
- Use TodoWrite to list the steps for the feature (module, schema, DTOs, service methods, controller routes, guards, verification). Mark each task `in_progress` / `completed` as you go.
- Ask the user for any ambiguous contract, authorization boundary, or persistence decision before writing code. Clarifying questions are cheaper than rework.

### A.1 — Directory skeleton

Create the feature under `src/modules/<feature>/`:

```text
src/modules/<feature>/
  dto/
    create-<feature>.dto.ts
    update-<feature>.dto.ts
  schemas/
    <feature>.schema.ts
  <feature>.module.ts
  <feature>.controller.ts
  <feature>.service.ts
  <feature>.guard.ts            # only if the feature needs a feature-local guard
```

For a child resource of `<feature>`:

```text
src/modules/<feature>/<child>/
  dto/
  schemas/
  <child>.controller.ts
  <child>.service.ts
  <child>.guard.ts              # optional
```

### A.2 — Schema (Mongoose)

File: `src/modules/<feature>/schemas/<feature>.schema.ts`

| Step                | Rule                                                                                                                   |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Schema class        | `@Schema({ timestamps: true })` on a PascalCase class matching the entity name (`Event`, `Workshop`).                  |
| Fields              | `@Prop({ type: ..., required: [true, "kebab-case-message"], ... })` for each persisted field. Explicit `type` always.  |
| Document type       | Export `export type <Entity>Document = HydratedDocument<<Entity>>`.                                                    |
| Factory             | Export `export const <Entity>Schema = SchemaFactory.createForClass(<Entity>)`.                                         |
| Parent refs         | Use `@Prop({ type: MongooseSchema.Types.ObjectId, ref: "<ParentEntity>", required: [true, "..."] })`. Name the field `<parent>Id`; use `Types.ObjectId` for the TypeScript property type. |
| Embedded schemas    | Separate `@Schema({ _id: false })` class, referenced as `@Prop({ type: [SubSchema], default: [] })`.                   |
| Indexes             | Add `<Entity>Schema.index({ ... }, { unique: ... })` after the factory call. Compound indexes live next to the schema. |
| Validation messages | All `required` tuples and `validate.message` strings are **kebab-case**.                                               |
| Hidden fields       | Sensitive/internal fields (`idempotencyKey`, private codes) use `select: false`.                                       |

See Part D below for the full schema design contract.

### A.3 — DTOs (validation layer)

Files in `src/modules/<feature>/dto/`:

| File                      | Purpose                                                                                                             |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `create-<feature>.dto.ts` | `class-validator`-decorated class defining required input for creation. Every decorator has a kebab-case `message`. |
| `update-<feature>.dto.ts` | `export class Update<Feature>Dto extends PartialType(Create<Feature>Dto) {}`. Nothing else.                         |
| `<sub-object>.dto.ts`     | Separate DTO for embedded objects, referenced via `@ValidateNested({ each: true })` + `@Type(() => SubDto)`.        |
| `get-<purpose>.dto.ts`    | Purpose-specific query/response DTOs when they don't fit create/update.                                             |

- Shared DTOs reused across features live in `src/common/dto/`, not inside a single feature.
- See Part C below.

### A.4 — Service (business logic)

File: `src/modules/<feature>/<feature>.service.ts`

| Step                         | Rule                                                                                                                                                                    |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Decorator                    | `@Injectable()`.                                                                                                                                                        |
| Logger                       | `private readonly logger = new Logger("service-<feature>");` as the first class field.                                                                                  |
| Constructor DI               | Inject Mongoose models via `@InjectModel(Entity.name)`, plus `ConfigService`, integration services, or other feature services as needed.                                |
| One method per use case      | `async createFeature`, `async getFeature`, `async updateFeature`, `async deleteFeature`, plus query methods as needed.                                                  |
| Comment every logical block  | Above each step, `// <Verb> <what>`. Match the density of the surrounding service files.                                                                                |
| Ordering inside a method     | 1) resolve parent / load resource → 2) existence + authorization + business-state checks → 3) durable writes → 4) non-critical side effects → 5) return result.         |
| Exceptions                   | Throw NestJS built-ins (`NotFoundException`, `BadRequestException`, `ForbiddenException`, `ConflictException`) with **kebab-case** messages.                            |
| Success returns              | Write operations return a kebab-case success string (`"event-created-successfully"`). Read operations return the document or DTO.                                       |
| Idempotency                  | Create flows accept an `idempotencyKey`, pre-check for an existing record, rely on a unique Mongo index as the final safety net.                                        |
| Transactions                 | Use Mongoose sessions for multi-document writes (cascading deletes, batched approvals). Wrap in `try/catch/finally`, abort on error, and always `session.endSession()`. |
| Fire-and-forget side effects | `void this.sqsService.publishMessage(...)` for non-critical work. Errors handled inside the integration service.                                                        |

See Part B below for the full service contract.

### A.5 — Controller (transport layer)

File: `src/modules/<feature>/<feature>.controller.ts`

| Step                     | Rule                                                                                                                                                       |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Route prefix             | `@Controller("<feature>")` using the kebab-case resource name.                                                                                             |
| Constructor DI           | Inject **only** the feature's own service.                                                                                                                 |
| Auth decorators          | `@Public()` for unauthenticated endpoints. `@UseGuards(AdminGuard)` for admin-only. Feature-local guards for resource-scoped authorization.                |
| User decorator           | `@CurrentUser() user: User` when the method needs the authenticated user. Prefix with `_` (e.g., `_user`) when the param exists only for guard activation. |
| Idempotency decorator    | `@IdempotencyKey() idempotencyKey: string` on create endpoints that require retry safety.                                                                  |
| DTOs                     | `@Body()`, `@Query()`, `@Param()` typed to feature DTOs. Validation happens automatically via the global `ValidationPipe`.                                 |
| Zero business logic      | Every method is a one-liner (or near) that delegates to the service: `return this.featureService.createFeature(dto, user, idempotencyKey);`.               |
| Manual response handling | Allowed only when transport requires it (streaming, file download, SSE, framework-specific response piping). Otherwise return the service result.          |

See Part B below and [boundaries.md Part A — Authentication and authorization](boundaries.md) for the full controller contract.

### A.6 — Module wiring

File: `src/modules/<feature>/<feature>.module.ts`

```ts
@Module({
    imports: [
        MongooseModule.forFeature([{ name: <Entity>.name, schema: <Entity>Schema }]),
        // Only features whose exported services this feature consumes:
        // e.g., IntegrationModule, AuthModule if its public exports are needed.
    ],
    controllers: [<Feature>Controller],
    providers: [<Feature>Service],
    exports: [<Feature>Service], // only if other features genuinely need it
})
export class <Feature>Module {}
```

Then import `<Feature>Module` into `src/app.module.ts`.

- **Do not** register the feature's Mongoose model globally in `AppModule`. Register it in the feature module.
- **Do not** export internal files, schemas, or repositories across modules. Export the service only when another feature needs its behavior.

### A.7 — Guards (feature-local)

File: `src/modules/<feature>/<feature>.guard.ts`

- Use for resource-scoped authorization tied to this feature (ownership checks, parent-child validity checks that must gate entry).
- Keep business-state checks (publication state, registration windows) in the **service**, not the guard.
- Apply via `@UseGuards(<Feature>Guard)` at method or controller level.
- See [boundaries.md Part A — Authentication and authorization](boundaries.md).

### A.8 — Child resources

- Place child resources under the parent feature (`src/modules/<feature>/<child>/`) when they share the parent's authorization or persistence boundary.
- The child controller routes nest under the parent (`@Controller("<feature>/:<parent>Id/<child>")` or equivalent).
- The child service loads the parent aggregate first, then operates on the child collection scoped by `<parent>Id`.
- Do not promote a child to a top-level module unless it genuinely becomes an independent domain.

### A.9 — Verification (Definition of Done)

A task is not done when the code merely looks correct. Before reporting completion:

1. **Discover scripts** — Read `package.json` once for the real verification commands.
2. **Narrow verification first** — run targeted TypeScript check, lint, or focused tests for the file/feature you changed. Use the project's scripts (e.g., `npm run build`, `npm run lint`, `npm test -- <path>`).
3. **Broaden for risky changes** — if you touched module wiring, global providers, DTOs, schemas, guards, or shared contracts, run a broader build/lint/typecheck.
4. **Never claim verification passed unless it was actually run.** Report what you ran and what passed.
5. If no useful automated verification exists for the change (e.g., a pure config tweak), say so explicitly.
6. For UI-integrated changes, explicitly note that backend tests do not verify UI behavior.

See [style-and-errors.md §A.10 — Verification (Definition of Done)](style-and-errors.md) for the full contract.

### A.10 — Quick checklist before you call the task done

- [ ] Files placed under `src/modules/<feature>/` following the directory skeleton.
- [ ] Naming conforms to [architecture.md Part C — Naming conventions](architecture.md) (kebab-case files, PascalCase classes, camelCase methods).
- [ ] Mongoose model registered in the **feature** module, not the root module.
- [ ] Feature module imported in `AppModule`.
- [ ] All validation / error / success strings are kebab-case.
- [ ] Controller is a thin dispatcher with zero business logic.
- [ ] Service methods have step-by-step `// <Verb> <what>` comments matching the surrounding code's density.
- [ ] No `process.env` reads inside domain code; config flows through `ConfigService`.
- [ ] No third-party SDK calls from controllers or domain services; all vendor calls go through `modules/integration/`.
- [ ] Existing public contracts (routes, DTO fields, error messages, env keys) unchanged unless the task explicitly required it.
- [ ] Verification actually executed. Results reported.

---

## Part B — Controllers and services

Controllers are transport. Services are business logic. The split is strict. This part defines both contracts, plus the orchestration and transaction rules that services must follow.

### B.1 — Controller contract

A controller method does exactly five things:

1. **Define the route** — method verb decorator (`@Get`, `@Post`, `@Patch`, `@Put`, `@Delete`) and path.
2. **Bind the request** — `@Body()`, `@Query()`, `@Param()`, `@Headers()`, `@Req()`, `@UploadedFile()` as needed, typed to DTOs.
3. **Apply guards and decorators** — `@UseGuards(...)`, `@Public()`, `@CurrentUser()`, `@IdempotencyKey()`, rate limiting overrides.
4. **Call the service** — one call to `this.<feature>Service.<method>(...)`.
5. **Return the result** — the service's return value, or a transport-specific response when required.

#### B.1.1 Hard rules

- **Zero business logic in controllers.** No conditionals on domain state, no multi-step orchestration, no vendor SDK calls, no database queries, no cross-feature composition.
- **No direct vendor SDK access.** Vendor calls go through `modules/integration/` services, and those services are consumed by **domain services**, not controllers.
- **No direct `process.env` reads.** Config flows through `ConfigService`, and only infrastructure layers read it.
- **No repeated inline preconditions.** If the same check appears in multiple routes, promote it to a guard or a shared service call.
- **Inject only the feature's own service** in the controller constructor. If you find yourself injecting a second service, rethink the design — services can call services, controllers should not compose them.
- **Never leak raw exceptions, stack traces, vendor payloads, or database internals** to the response. Let global exception filters shape the response. See [style-and-errors.md Part B — Error handling, filters, logging, and observability](style-and-errors.md).

#### B.1.2 Route shape

- Prefer route shapes that reflect resource structure: `/event`, `/event/:eventId`, `/event/:eventId/workshop`, `/event/:eventId/workshop/:workshopId`.
- Path params use camelCase when multi-word (`:workshopId`, not `:workshop_id` or `:workshopid`).
- Keep param names stable. Guards and services may read `request.params[...]` by name — renaming is a cross-cutting change.

#### B.1.3 Auth decorators

- Public routes must be **explicitly** marked with the repository's public marker (`@Public()`). The default is authenticated.
- Admin-only endpoints apply the global admin guard (`@UseGuards(AdminGuard)`).
- Resource-scoped authorization uses feature-local guards applied at method level.
- Unused auth params exist only to activate a guard — prefix them with `_` (e.g., `_user: User`).

#### B.1.4 Manual response handling

- Allowed only when transport requires it: streaming, file downloads, SSE, or framework-specific response piping.
- In those cases, document why briefly (one-line comment) and still keep all decisions-making inside the service.

#### B.1.5 Controller example shape (reference, not a template to copy blindly)

```ts
@Controller("event")
export class EventController {
    constructor(private readonly eventService: EventService) {}

    @Post()
    @UseGuards(AdminGuard)
    createEvent(@Body() dto: CreateEventDto, @CurrentUser() user: User, @IdempotencyKey() idempotencyKey: string) {
        return this.eventService.createEvent(dto, user, idempotencyKey);
    }

    @Get(":eventId")
    @Public()
    getEvent(@Param("eventId") eventId: string) {
        return this.eventService.getEvent(eventId);
    }
}
```

### B.2 — Service contract

Services own business logic, orchestration, persistence flow, and coordination with integration services.

#### B.2.1 Hard rules

- Services are the **default home for state transitions**.
- Services are the only place where write orchestration, business-state validation, multi-document transactions, and domain workflows live.
- Services depend on integration services (`SqsService`, `EmailService`, `S3Service`, `ClerkService`) — never on raw vendor SDKs.
- Services throw **framework-native NestJS exceptions** with kebab-case messages. See [style-and-errors.md Part B — Error handling, filters, logging, and observability](style-and-errors.md).
- Services use the repository's `Logger` with a context string: `new Logger("service-<feature>")`.

#### B.2.2 Standard method ordering

Every service method that mutates state should follow this order:

1. **Resolve dependencies** — load the parent aggregate or required context first.
2. **Existence check** — throw `NotFoundException("...-does-not-exist")` if the resource is missing.
3. **Authorization / ownership check** — throw `ForbiddenException(...)` when the user cannot act on the resource. (Coarse auth is already handled by guards; this step enforces fine-grained domain rules.)
4. **Business-state check** — registration windows, publication state, idempotency pre-check, parent-child validity.
5. **Durable write** — the primary database mutation, inside a transaction if multi-document.
6. **Non-critical side effects** — fire-and-forget SQS, email, webhook dispatch. Wrapped in `void ...`.
7. **Return** — kebab-case success string for writes, document or DTO for reads.

#### B.2.3 Comment every logical block

- Every logical step in a service method gets a comment on the line **above** it.
- Pattern: `// <Verb> <what>` — e.g., `// Get the workshop by event ID`, `// Check if the workshop exists`, `// Return success message`.
- Match the density of the surrounding code. If the feature is heavily commented, follow that. If it is not, keep the same level.
- Do not explain _what_ trivially obvious code does; comment the _intent_ and the _step_.

#### B.2.4 Service method example shape (reference)

```ts
async getWorkshop(eventId: string, workshopId: string) {
    // Get the workshop by event ID and workshop ID
    const workshop = await this.workshopModel.findOne({ _id: workshopId, eventId });
    // Check if the workshop exists
    if (!workshop) throw new NotFoundException("workshop-does-not-exist");
    // Return workshop
    return workshop;
}
```

```ts
async createWorkshop(eventId: string, dto: CreateWorkshopDto, idempotencyKey: string) {
    // Get the parent event
    const event = await this.eventModel.findById(eventId);
    // Check if the event exists
    if (!event) throw new NotFoundException("event-does-not-exist");
    // Check idempotency — return existing record if retry
    const existing = await this.workshopModel.findOne({ idempotencyKey }).select("+idempotencyKey");
    if (existing) return "workshop-created-successfully";
    // Create the workshop scoped to the event
    await this.workshopModel.create({ ...dto, eventId, idempotencyKey });
    // Publish a non-critical workshop-created event
    void this.sqsService.publishMessage({ type: "workshop.created", eventId, workshopId: dto.slug });
    // Return success message
    return "workshop-created-successfully";
}
```

#### B.2.5 Document mutation vs. direct update

- Use **direct updates** (`updateOne`, `findByIdAndUpdate`) for simple field updates after validation, where the logic is flat.
- Use **load-mutate-save** (`findById` → mutate document → `save()`) when document-level logic, conditional field handling, or subdocument operations make it clearer.
- Prefer whichever reads as one clear use case. Do not mix both styles in one method.

#### B.2.6 Transactions

- Use Mongoose sessions for any operation that must mutate multiple documents as a single unit (cascading deletes, bulk approvals, cross-collection state changes).
- Always open, commit/abort, and end the session in `try/catch/finally`:

```ts
const session = await this.connection.startSession();
session.startTransaction();
try {
    // Perform multi-document writes with { session }
    await session.commitTransaction();
    return "...-successfully";
} catch (err) {
    await session.abortTransaction();
    throw err;
} finally {
    await session.endSession();
}
```

- Inject the Mongoose connection via `@InjectConnection()`.
- Never trigger external side effects from inside a transaction. Fire them **after** the transaction commits.

#### B.2.7 Fire-and-forget side effects

- Use `void` for async operations that should not block the response (SQS publishes, email sends, non-critical webhooks):

```ts
void this.sqsService.publishMessage({ ... });
```

- The integration service catches, logs, and reports these errors to Sentry internally. The domain service does not await.
- If a side effect is business-critical (payment, subscription state change), it is **not** fire-and-forget — await it and handle failure as part of the primary flow.
- Do not trigger external side effects before the primary write succeeds, unless the product explicitly requires that order.

#### B.2.8 Idempotency

- Create flows that are retry-prone, webhook-like, or public-form-driven must accept an `idempotencyKey` parameter.
- Pre-check for an existing record keyed by `idempotencyKey` for a clean fast-path return.
- Back idempotency with a **unique Mongo index** on `idempotencyKey` as the final race-condition safety net. Pre-checks are not sufficient.
- Do not force idempotency onto every mutation. Use it where duplicate submission is a real risk.

#### B.2.9 Method size and splitting

- Keep service methods focused on **one use case**. Split when a method stops reading as one use case.
- Do not split trivial code just to satisfy an arbitrary size rule.
- Extract private helpers (`private resolveParent(...)`, `private assertOwnership(...)`) when the same sub-step is used by multiple methods within the service.
- Do not move primary business flow into schema hooks, ORM callbacks, middleware, or decorators unless the repository already uses that pattern intentionally.

#### B.2.10 Forbidden in services

- No direct vendor SDK imports. Go through `modules/integration/`.
- No `process.env` reads. Go through `ConfigService`.
- No HTTP response manipulation. That belongs to the controller's transport layer.
- No swallowing errors silently. Either handle and recover, or rethrow. Logged-and-dropped errors must use `this.logger.error(...)` + Sentry capture.
- No cross-feature persistence writes. If feature A needs to write to feature B's collection, it calls B's exported service method.

---

## Part C — DTOs and validation

DTOs are the application's transport-level contract. They define what the outside world may send, what a response looks like, and what validation the application trusts at the boundary. This part covers DTO design, `class-validator` usage, and API contract stability.

### C.1 — DTO placement

- **Feature-local DTOs** live in `src/modules/<feature>/dto/`.
- **Shared DTOs** reused across features live in `src/common/dto/`.
- Do not place feature-specific DTOs in `common/` just because another feature _might_ use them one day. Promote to shared only when reuse is real.
- One DTO per file. File name follows the kebab-case convention (`create-<feature>.dto.ts`, `update-<feature>.dto.ts`, `<sub-object>.dto.ts`, `get-<purpose>.dto.ts`).

### C.2 — Create / Update DTO pattern

- **Create DTO** defines required input for a creation flow. Use `class-validator` decorators on every field with an explicit **kebab-case** `message`.
- **Update DTO** is derived via `PartialType(Create<Feature>Dto)` from `@nestjs/mapped-types`. The file should contain nothing else:

    ```ts
    import { PartialType } from "@nestjs/mapped-types";
    import { CreateEventDto } from "./create-event.dto";

    export class UpdateEventDto extends PartialType(CreateEventDto) {}
    ```

- If you need a partial that omits some fields, use `PartialType(OmitType(CreateEventDto, ["...",  "..."] as const))`.

### C.3 — `class-validator` rules

- Every field on every DTO must have at least one validator. No undeclared fields.
- Every validator must have an explicit **kebab-case `message`**:

    ```ts
    @IsNotEmpty({ message: "please-provide-a-name" })
    @MaxLength(120, { message: "name-too-long" })
    name: string;

    @Matches(/^\d{4}-\d{2}-\d{2}$/, { message: "start-date-invalid-format" })
    startDate: string;

    @IsEnum(EventType, { message: "event-type-invalid" })
    type: EventType;

    @IsOptional()
    @IsMongoId({ message: "workshop-id-invalid" })
    workshopId?: string;
    ```

- Validation messages are translation keys. They must be stable, specific to the failing field, and lowercase with hyphens.
- Do not use generic messages like `"Bad request"`, `"Invalid"`, `"Field is required"`.
- Never change a published validation message casually. Clients may switch on it.

### C.4 — Nested DTOs

For nested objects or arrays of objects, define a separate DTO and validate the nested structure:

```ts
export class TargetDto {
    @IsMongoId({ message: "target-id-invalid" })
    id: string;

    @IsEnum(TargetType, { message: "target-type-invalid" })
    type: TargetType;
}

export class CreateNotificationDto {
    @IsString({ message: "title-must-be-string" })
    @IsNotEmpty({ message: "please-provide-a-title" })
    title: string;

    @IsArray({ message: "targets-must-be-array" })
    @ArrayMinSize(1, { message: "please-provide-at-least-one-target" })
    @ValidateNested({ each: true })
    @Type(() => TargetDto)
    targets: TargetDto[];
}
```

- Always pair `@ValidateNested({ each: true })` with `@Type(() => Dto)` from `class-transformer`.
- Nested DTOs live in the same `dto/` folder when feature-local, or in `common/dto/` when shared.

### C.5 — Global ValidationPipe

- The global `ValidationPipe` is configured in `main.ts` with:

    ```ts
    app.useGlobalPipes(
        new ValidationPipe({
            transform: true,
            whitelist: true,
            forbidNonWhitelisted: true
        })
    );
    ```

- **`transform: true`** — incoming plain objects become DTO class instances. Type coercion happens here.
- **`whitelist: true`** — properties not decorated on the DTO are stripped.
- **`forbidNonWhitelisted: true`** — unknown properties cause a `BadRequestException`. This rejects extra fields at the boundary.
- Do not override these globally without a strong reason.

### C.6 — Runtime (dynamic) validation

- Use `class-validator` decorators for normal, statically-shaped DTOs.
- Use **Zod** (or the repository's established runtime-validation library) only when the payload shape is **dynamic**, runtime-defined, or schema-composed (e.g., form-builder submissions, config-driven payloads).
- Do not force all validation into decorators when the real problem requires runtime schema composition.
- Do not replace `class-validator` with Zod across the codebase. Zod is a secondary tool for specific cases.

### C.7 — Schema-level validation is a secondary safety net

- DTO validation is the **primary** boundary.
- Mongoose schema validators (`required`, `enum`, `match`, `validate`) are a **secondary** safety net behind DTOs.
- When both exist, keep messages consistent (kebab-case translation keys). See Part D below.

### C.8 — Response shape stability

- **Write operations** typically return a stable kebab-case success string (`"event-created-successfully"`).
- **Read operations** return structured domain data — either the Mongoose document or an explicitly shaped object.
- **Special cases** (signed URLs, stats aggregations, streams, token exchanges, file uploads) may return a purpose-specific response object. Define a clear response type and keep it stable.
- Do not casually change response shapes. Clients are coupled to them.

### C.9 — API contract stability

- DTO field names, validation messages, route paths, path param names, and response field names are **public contracts**.
- Before renaming any of them: Grep the codebase, check client repos if referenced, and ask the user. Document the migration plan.
- When you must rename, prefer additive changes (add the new field, deprecate the old, remove after clients migrate) over in-place rewrites.

### C.10 — Forbidden patterns

- ❌ No undocumented fields on DTOs (every field validated).
- ❌ No missing `message` on validators.
- ❌ No English-prose validation messages. Always kebab-case.
- ❌ No business logic inside DTOs or class-validator custom validators. DTOs validate _shape_, not _workflow_.
- ❌ No mixing update DTOs that re-declare fields. Always derive from create via `PartialType`.
- ❌ No circular DTO imports. If two DTOs need the same nested shape, extract it into `common/dto/` or a sibling DTO file.

---

## Part D — Persistence (MongoDB + Mongoose)

Default database: **MongoDB**. Default access layer: **Mongoose** via `@nestjs/mongoose`. This part defines the schema design contract, indexing strategy, and query/mutation style.

Do not introduce Prisma, TypeORM, Sequelize, Drizzle, raw SQL, or a second ORM stack unless the user explicitly approves the change.

### D.1 — Placement

- Schemas live inside the **owning feature** at `src/modules/<feature>/schemas/<entity>.schema.ts`.
- Register models in the **feature module** via `MongooseModule.forFeature([{ name: Entity.name, schema: EntitySchema }])`. Never register feature-owned schemas in `AppModule`.
- Indexes live next to the schema they belong to (same file, after `SchemaFactory.createForClass`).
- Embedded subdocument schemas live in the same `schemas/` folder as the parent (or colocated in the parent file when tiny and tightly coupled).
- Do not put domain models or schemas in `common/` unless they are genuinely shared across multiple features.

### D.2 — Schema definition style

- Top-level schemas are **NestJS Mongoose classes** decorated with `@Schema()` and `@Prop()`.
- One main schema class per file. Tiny embedded helper schemas may share the file when they are tightly coupled to the parent.
- Class names are **singular PascalCase** for aggregate roots (`Event`, `Workshop`, `Participant`, `Feedback`).
- Top-level schemas use `@Schema({ timestamps: true })` — `createdAt` and `updatedAt` are part of the project standard.
- Embedded subdocument schemas use `@Schema({ _id: false })` when the subdoc should not carry its own identity.
- Always use **explicit `type`** in `@Prop(...)`. Do not rely on implicit inference when field shape matters.
- For Mongoose runtime schema type declarations, import `Schema as MongooseSchema` and `Types` from `mongoose`; use `MongooseSchema.Types.*` inside `@Prop({ type: ... })`, and use `Types.*` for TypeScript property types.

#### D.2.1 Required export shape

```ts
@Schema({ timestamps: true })
export class Event {
    @Prop({ type: String, required: [true, "please-provide-a-name"], trim: true })
    name: string;

    @Prop({ type: String, enum: EventType, required: [true, "please-provide-event-type"] })
    type: EventType;

    @Prop({ type: MongooseSchema.Types.ObjectId, ref: "Organizer", required: [true, "organizer-id-required"] })
    organizerId: Types.ObjectId;

    @Prop({ type: String, required: true, select: false })
    idempotencyKey: string;
}

export type EventDocument = HydratedDocument<Event>;

export const EventSchema = SchemaFactory.createForClass(Event);

EventSchema.index({ organizerId: 1, createdAt: -1 });
EventSchema.index({ idempotencyKey: 1 }, { unique: true });
```

- Export the class (named).
- Export `type <Entity>Document = HydratedDocument<<Entity>>`.
- Export `const <Entity>Schema = SchemaFactory.createForClass(<Entity>)`.
- Add indexes **after** the factory call.

### D.3 — Field design

Every persisted field has an intentional shape, validation, and default.

- **Required fields** use the tuple form: `required: [true, "kebab-case-message"]`. Conditional requireds use a function: `required: [function () { return this.type === "scheduled"; }, "start-date-required-for-scheduled"]`.
- **String normalization**: apply `trim: true` and `lowercase: true` when the invariant belongs to the stored data (emails, slugs, codes).
- **Enums**: `enum: MyEnum` on enum-typed fields.
- **Pattern constraints**: `match: /regex/` for format invariants at the data layer.
- **Custom validators**: `validate: { validator: fn, message: "kebab-case-message" }`.
- **Optional fields**: prefer `default: undefined` when that produces cleaner documents and better conditional validation.
- **Boolean flags**: always explicit, `default: false` or `default: true`. Do not rely on absent fields to mean "false".
- **Computed defaults**: only when they are deterministic and clearly part of the data model.
- **Hidden fields**: use `select: false` for internal-only values (`idempotencyKey`, private codes, sensitive tokens) so they are not returned by default queries.
- Keep validation messages kebab-case and stable. See [architecture.md Part C — Naming conventions](architecture.md).

### D.4 — IDs, references, and relationships

- **Primary IDs** are Mongo `ObjectId`s unless there is a clear, justified reason to use a different type.
- **Parent-reference fields** are named `<parent>Id` exactly: `eventId`, `workshopId`, `participantId`, `provisionId`, `attendanceId`. No variations (`event_id`, `eventID`, `parent`, `parentEvent`).
- Parent refs declare `ref` explicitly: `@Prop({ type: MongooseSchema.Types.ObjectId, ref: "Event", required: [true, "..."] })`.
- **UUIDs** are acceptable for idempotency keys and special internal identifiers. Use `MongooseSchema.Types.UUID` as the stored schema type and expose app-facing fields as `string` or `string | undefined`.
- **`select: false`** on hidden identifiers (idempotency keys, private codes) so they do not appear in default reads.
- **Prefer parent-scoped queries** (`{ _id: childId, parentId }`) over unconstrained global lookups.
- **Prefer explicit loads in services** over `populate`-heavy designs.
- **Do not** make `populate`, virtual populate, or autopopulate the default relationship strategy. When you need related data, load it explicitly in the service with a second query — it is clearer and easier to reason about.

### D.5 — Embedded documents and dynamic shapes

- **Embedded subdocument schemas** for tightly owned nested structures with no independent lifecycle: entries, categories, targets, date ranges, form items, settings snapshots.
- **`type: [SubSchema]`** for ordered embedded collections with a known item shape.
- **`type: Map`** only when the shape is truly dynamic and keyed by runtime-defined field names. Use `Map` with an explicit `of` schema where possible.
- **`MongooseSchema.Types.Mixed`** only for truly variable leaf values that cannot be modeled more precisely.
- Do not default to `Mixed` or loose `Map` when a concrete schema is practical.

### D.6 — Validation layers

- **Schema validators** protect **data quality**. They catch invariants of the persisted data itself: required fields, enum membership, format constraints, parent-scoped uniqueness via indexes, embedded-array validity, conditional required fields.
- **DTO validators** validate **request shape** at the application boundary. See Part C above.
- **Service code** validates **workflow and process rules**: registration windows, publication state, parent-child validity, ownership checks.
- Schema validators are a safety net **behind** DTO validation. They do not replace service orchestration.

### D.7 — Indexing and uniqueness

- **Single-field uniqueness**: `@Prop({ ..., unique: true })`. This declares a unique index on the field.
- **Compound indexes and compound unique constraints**: after `SchemaFactory.createForClass`, call `<Entity>Schema.index({ field1: 1, field2: 1 }, { unique: true })`.
- **Parent-scoped uniqueness** is the common case for child resources. Examples: `{ email: 1, workshopId: 1 }` unique; `{ identification: 1, participantId: 1 }` unique.
- **Plain indexes** for frequently queried foreign keys and high-value lookup fields (`organizerId`, `eventId`, `createdAt` sort).
- **Keep indexes beside the schema definition** so they stay visible during changes.
- The **database** is the final uniqueness authority. Service pre-checks may short-circuit for cleaner error messages, but the unique index is the race-safe backstop.

### D.8 — Query and mutation style

- **Query children through their parent boundary** whenever practical. For a parent-scoped route, ensure the parent exists before reading or mutating the child collection.
- **Avoid document graphs** that require broad eager loading to answer normal requests.
- **Keep reads explicit**. Prefer named projections, parent-scoped filters, and obvious query shapes.
- **Direct updates** (`updateOne`, `findOneAndUpdate`) for simple post-validation field updates.
- **Load-mutate-save** (`findById` → mutate → `save()`) when document-level logic, conditional subdocument handling, or middleware-style behavior is clearer that way.
- **Transactions** for multi-document atomic writes (cascading deletes, batched approvals, cross-collection state changes). See Part B §B.2.6 above.

### D.9 — Schema safety (anti-patterns)

- ❌ Do not hide important persistence behavior in pre-save, post-save, or `pre("findOneAndUpdate")` hooks unless the repository explicitly standardizes that pattern. Writes should be traceable from the service.
- ❌ No autopopulate by default.
- ❌ No plugin magic that rewrites queries, injects fields, or mutates state invisibly.
- ❌ No over-normalization into many tiny collections without a clear query or lifecycle reason.
- ❌ No blind denormalization. Embed when ownership is tight and lifecycle is shared. Reference when lifecycle, cardinality, or query patterns make a ref cleaner.
- ❌ No feature models in `common/` unless they are genuinely shared across modules.

### D.10 — Migrations and schema changes

- When adding a required field to an existing schema, coordinate with data: either provide a default, make it conditionally required, or plan a backfill.
- Renaming a persisted field is a contract change. Grep for the field name, check any downstream consumers, and plan a migration before editing the schema.
- Adding a unique index to an existing collection will fail if duplicates exist. Verify the data shape before merging.
- Never edit a schema file without also updating the indexes next to it if field semantics changed.

### D.11 — Before editing any schema

- Use Read/Grep to find every consumer of the schema fields you are about to change.
- Use Grep to check for references by the Mongoose model name (`@InjectModel(Entity.name)`).
- Use Grep to check for raw queries that may hardcode field names.
- Ask the user before any rename, required-field addition, or index change on a production model.
