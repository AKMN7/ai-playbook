# Boundaries — auth, integrations, configuration, dates

This file covers everything that crosses the application boundary in either direction. Inbound identity (Part A — Clerk JWTs become normalized `User`). Outbound vendor calls (Part B — third-party SDKs through the integration layer). Sideband: configuration and time (Part C — environment reads through `ConfigService`, business dates through shared timezone helpers).

The unifying rule: domain code does not see raw external shapes. Boundary code normalizes them before they reach controllers and services.

---

## Part A — Authentication and authorization

Clerk is the default identity provider. The backend validates Clerk-issued JWTs; it does not own primary identity. Authentication normalizes identity only; business authorization data lives in internal application persistence and is loaded through authorization services, guards, or domain services.

### A.1 — Request flow

1. Client sends a Clerk-issued bearer token in the `Authorization: Bearer <token>` header.
2. `JwtAuthGuard` (registered globally via `APP_GUARD`) runs for every route unless `@Public()` is applied.
3. The NestJS JWT strategy (`JwtStrategy`) extracts the bearer token and validates it using the configured Clerk signing key via `ConfigService`.
4. The strategy **normalizes** the raw Clerk payload into one shared identity-only authenticated user shape and attaches it to `request.user`.
5. Controllers and guards consume the **normalized `User` object**, never the raw JWT payload.

### A.2 — The normalized `User` shape

Defined in `src/modules/auth/types/user.type.ts` (or the repository's equivalent):

```ts
export type User = {
    sessionId: string;
    clerkId: string;
};
```

- Include authenticated identity only. Do not pad with unused fields.
- The shape is a contract — treat it as public across features.
- Do not put business authorization state such as roles, permissions, or resource-scoped access metadata into this type by default.
- Do not add raw Clerk-specific fields (`sid`, `sub`, vendor flags) to this type. Keep vendor parsing inside the strategy.

### A.3 — Strategy layer

File: `src/modules/auth/strategies/jwt.strategy.ts`

- Uses `passport-jwt` via `@nestjs/passport`.
- Registers under a named strategy (`"jwt"`) so `JwtAuthGuard` can reference it explicitly.
- Reads required auth configuration through `ConfigService.getOrThrow(...)`.
- `validate(payload)` validates required identity claims, normalizes the payload, and returns the `User` object that becomes `request.user`.
- All Clerk-specific parsing — claim names, session extraction, and provider user id extraction — stays **inside** the strategy.

### A.4 — Guards

All guards live in `src/modules/auth/guards/` (global) or inside a feature (`src/modules/<feature>/<feature>.guard.ts`).

#### A.4.1 Global guards (registered via `APP_GUARD`)

Standard registration order in `AppModule` (first registered runs first):

1. `ThrottlerGuard` — rate limiting.
2. `JwtAuthGuard` — authentication. Bypassed for routes marked `@Public()` (reads `IS_PUBLIC_KEY` metadata via `Reflector`).

#### A.4.2 `AdminGuard` (or equivalent)

- Applied via `@UseGuards(AdminGuard)` at controller or method level.
- Checks the project's internal admin criterion, usually through an exported authorization service.
- Coarse-grained only. Does **not** enforce domain-specific permissions.

#### A.4.3 Feature-local guards

- Live at `src/modules/<feature>/<feature>.guard.ts`.
- Enforce **resource-scoped authorization** tied to that feature by consulting internal authorization data or domain policy checks.
- Applied via `@UseGuards(<Feature>Guard)` at method or controller level.
- May read route params to enforce the guard condition — this is why route param names must remain stable.

#### A.4.4 What guards are _not_ for

- ❌ Business-state workflow checks (publication state, registration window, payment status) — those go in the **service**.
- ❌ Multi-step orchestration — services.
- ❌ Vendor SDK calls — integration layer.
- ❌ Shaping the response — filters.

Rule of thumb: guards answer **"may this request enter this route?"** Services answer **"may this operation proceed against this domain state?"**

### A.5 — Decorators

All auth decorators live in `src/modules/auth/decorators/` (or the project's `common/decorators/` area, matching local precedent).

#### A.5.1 `@Public()`

- Marks a route as unauthenticated.
- Sets metadata with a known key (`IS_PUBLIC_KEY`) that `JwtAuthGuard` reads via `Reflector.getAllAndOverride(...)`.
- Use sparingly — the default is authenticated.

#### A.5.2 `@CurrentUser()`

- Parameter decorator that returns the normalized `User` from `request.user`.
- Preferred signature: `@CurrentUser() user: User`.
- When the param exists only to activate a guard (e.g., admin-guarded route that does not use the user), prefix with `_`: `@CurrentUser() _user: User`.

#### A.5.3 `@IdempotencyKey()`

- Parameter decorator that extracts the idempotency-key header.
- Validates the key format at the boundary (throws `BadRequestException("idempotency-key-invalid")` when malformed).
- Applied on create endpoints that are retry-prone.

### A.6 — Applying auth at the controller

```ts
@Controller("workshop")
export class WorkshopController {
    constructor(private readonly workshopService: WorkshopService) {}

    // Unauthenticated
    @Get(":workshopId/public")
    @Public()
    getPublicWorkshop(@Param("workshopId") workshopId: string) {
        return this.workshopService.getPublicWorkshop(workshopId);
    }

    // Admin-only, user not needed
    @Delete(":workshopId")
    @UseGuards(AdminGuard)
    deleteWorkshop(@Param("workshopId") workshopId: string, @CurrentUser() _user: User) {
        return this.workshopService.deleteWorkshop(workshopId);
    }

    // Authenticated + resource-scoped guard
    @Post(":workshopId/register")
    @UseGuards(WorkshopAccessGuard)
    registerForWorkshop(@Param("workshopId") workshopId: string, @Body() dto: CreateRegistrationDto, @CurrentUser() user: User, @IdempotencyKey() idempotencyKey: string) {
        return this.workshopService.registerForWorkshop(workshopId, dto, user, idempotencyKey);
    }
}
```

### A.7 — Authorization in services

Services handle **fine-grained** authorization that depends on data:

- Ownership checks (`if (doc.ownerId !== user.clerkId) throw new ForbiddenException("not-allowed-to-edit")`, or the repository's normalized provider id).
- Tenant isolation (`if (doc.tenantId !== user.tenantId) throw new ForbiddenException("..."))`.
- Business-state gating (`if (event.status !== "published") throw new BadRequestException("event-not-published")`).

This belongs in the service because it requires loading the document and inspecting domain state. Keep it close to the code that performs the action.

### A.8 — Route param stability

- Authorization often depends on route params (`:workshopId`, `:eventId`). Guards and services may read them by name.
- Do **not** rename route params casually. Grep for `@Param("oldName")` and `request.params["oldName"]` before any rename.
- When renaming is truly required, update guards, services, route definitions, and any downstream clients in the same change.

### A.9 — Forbidden patterns

- ❌ Raw JWT payload parsing in controllers or domain services.
- ❌ Raw Clerk claim names or vendor payload fields leaking into domain code.
- ❌ Business workflow checks inside guards.
- ❌ Domain orchestration inside decorators.
- ❌ Two parallel auth stacks (e.g., Clerk + a second identity provider) unless the user explicitly asks.
- ❌ Adding `bcrypt` unless the backend actually stores local password hashes.
- ❌ Reading Clerk signing keys or any other auth config via `process.env` directly. Always through `ConfigService`.

---

## Part B — Integrations and side effects

All third-party SDK access, vendor HTTP calls, and raw vendor payload mapping live in a dedicated integration layer. Domain services depend on integration services; they never touch vendor clients directly. This part defines the integration contract, idempotency rules, and side-effect ordering.

### B.1 — Integration layer placement

- Location: `src/modules/integration/`.
- One **service per vendor**: `sqs.service.ts`, `s3.service.ts`, `resend.service.ts`, `clerk.service.ts`, `openai.service.ts`, etc.
- One `integration.module.ts` that declares all integration services as providers and exports the ones other features consume.
- Each integration service is `@Injectable()` and is injected into domain services through normal NestJS DI.

```ts
@Module({
    providers: [SqsService, S3Service, EmailService],
    exports: [SqsService, S3Service, EmailService]
})
export class IntegrationModule {}
```

- Feature modules that need integrations import `IntegrationModule`.

### B.2 — Integration service contract

- Expose **business-meaningful methods**: `publishMessage`, `sendEmail`, `generateUploadUrl`, `downloadAsset`, `createBadge`, not vendor-shaped primitives — unless the repository explicitly standardizes the raw-call pattern.
- Hide vendor-specific field names, request shapes, and SDK quirks inside the service. Domain code must not see `Messages[0].Body.MessageAttributes` or equivalent vendor noise.
- Input types are **your domain types**, not SDK types. Output types are your domain types.
- Internally, the service reads required configuration via `ConfigService.getOrThrow(...)`. No `process.env` reads inside integration services either.
- Keep retry and idempotency logic **explicit and documented**. Do not bury retry assumptions in undocumented helper code.

#### B.2.1 Example shape (reference)

```ts
@Injectable()
export class SqsService {
    private readonly logger = new Logger("integration-sqs");
    private readonly client: SQSClient;
    private readonly queueUrl: string;

    constructor(private readonly config: ConfigService) {
        this.queueUrl = this.config.getOrThrow<string>("SQS_QUEUE_URL");
        this.client = new SQSClient({ region: this.config.getOrThrow<string>("AWS_REGION") });
    }

    async publishMessage(payload: EmailPayload): Promise<void> {
        try {
            await this.client.send(
                new SendMessageCommand({
                    QueueUrl: this.queueUrl,
                    MessageBody: JSON.stringify(payload)
                })
            );
        } catch (err) {
            this.logger.error("sqs-publish-failed", { error: err, payload });
            captureException(err);
            // Fire-and-forget by contract: do not rethrow unless the caller awaits.
        }
    }
}
```

### B.3 — How domain services consume integrations

- Domain services inject the integration service: `constructor(private readonly sqsService: SqsService) {}`.
- Domain services call the business-meaningful method: `this.sqsService.publishMessage({ ... })`.
- Domain services never import vendor SDK packages.
- If the domain needs a feature the integration service doesn't expose, **add a method to the integration service**. Do not reach around it.

### B.4 — Side-effect ordering

Writes must happen in this order inside a service method:

1. **Resolve dependencies** — load the parent aggregate, load the resource under mutation.
2. **Validate** — existence, authorization, business state.
3. **Persist core business state** — the durable write (transactional if multi-document).
4. **Trigger non-critical side effects** — emails, SQS publishes, notifications, webhooks.
5. **Return** — kebab-case success string or data.

Do **not** trigger external side effects before the primary write succeeds unless the product explicitly requires that ordering (e.g., pre-issuing a signed URL before the metadata row).

### B.5 — Fire-and-forget side effects

- Use `void` on calls whose failure must not break the primary user flow:

    ```ts
    void this.sqsService.publishMessage({ ... });
    void this.emailService.sendEmail({ ... });
    ```

- The integration service is responsible for its own error handling: `try/catch`, `this.logger.error(...)`, `captureException(err)`.
- The domain service does not await and does not catch.
- If the side effect is business-critical (payment capture, subscription state change), it is **not** fire-and-forget — await it, handle failure, surface errors.

### B.6 — Idempotency and retry safety

#### B.6.1 When to apply idempotency

- Retry-prone create flows.
- Externally triggered submissions (webhooks, public forms).
- Any write that clients may resend due to network instability.

#### B.6.2 How to apply it

- Controllers extract the key via `@IdempotencyKey() idempotencyKey: string`.
- The decorator validates key format at the boundary (throws `BadRequestException("idempotency-key-invalid")` on malformed input).
- Services accept `idempotencyKey: string` as a parameter on the affected method.
- The service pre-checks for an existing record keyed by `idempotencyKey` and returns the stable success string if the record already exists:

    ```ts
    const existing = await this.workshopModel.findOne({ idempotencyKey }).select("+idempotencyKey");
    if (existing) return "workshop-created-successfully";
    ```

- The schema **must** have a unique index on `idempotencyKey` as the race-safe backstop:

    ```ts
    @Prop({ type: String, required: true, select: false })
    idempotencyKey: string;

    WorkshopSchema.index({ idempotencyKey: 1 }, { unique: true });
    ```

- The `idempotencyKey` field uses `select: false` so it is not returned in default reads.

#### B.6.3 When _not_ to apply idempotency

- Read operations.
- Simple updates where duplicate submission is not a real risk.
- Internal admin actions where the user directly controls submission.

Do not force idempotency onto every mutation. Use it where duplicate submission is a real risk.

### B.7 — Webhooks and external triggers

- Webhook endpoints are `@Public()` (they cannot carry user JWTs).
- Webhook routes verify the vendor signature using the integration service for that vendor (`clerkService.verifyWebhook(...)`, `stripeService.verifySignature(...)`).
- Webhooks are idempotent by design — vendors retry. Apply idempotency keys (usually the vendor's event id) and a unique index.
- Webhook controllers remain thin: bind the body, call the service, return the success string.
- Webhook side effects follow the same ordering: persist → fire non-critical effects → return.

### B.8 — Configuration for integrations

- All integration configuration flows through `ConfigService`.
- Required keys use `ConfigService.getOrThrow<string>("KEY_NAME")` — fail fast at construction time.
- Env var names follow the project convention (SCREAMING_SNAKE_CASE, vendor-prefixed): `AWS_REGION`, `SQS_QUEUE_URL`, `RESEND_API_KEY`, `CLERK_ADMINS_JWT_KEY`, `S3_BUCKET_NAME`.
- See Part C below.

### B.9 — Forbidden patterns

- ❌ Vendor SDK imports (`@aws-sdk/...`, `resend`, `axios`, `openai`) inside controllers or domain services.
- ❌ `process.env` reads inside integration services (go through `ConfigService`).
- ❌ Vendor field names (`MessageAttributes`, `x-clerk-signature`, `Metadata.key`) leaking into controllers, DTOs, or domain services.
- ❌ Side effects fired before the primary persistence write succeeds.
- ❌ Swallowed integration errors without log + Sentry capture.
- ❌ Two integration services for the same vendor in parallel (one integration file per vendor).
- ❌ Hardcoded retry assumptions inside ad hoc helper functions. Retry behavior is part of the integration service's documented contract.

---

## Part C — Configuration, environment, and dates

Two orthogonal concerns that both demand **centralization**. Configuration must flow through `ConfigService`. Business dates must flow through shared timezone-aware helpers. Neither may be accessed ad hoc by feature code.

### C.1 — Configuration access

#### C.1.1 Hard rules

- All configuration and environment reads go through `@nestjs/config` `ConfigService`.
- **No direct `process.env` reads** in controllers, domain services, guards, DTOs, or schemas.
- Required values use `ConfigService.getOrThrow<T>("KEY_NAME")` — fail fast at construction or module init.
- Optional values use `ConfigService.get<T>("KEY_NAME")` with a documented default when appropriate.
- Integration and infrastructure layers (`modules/integration/`, `strategies/`, bootstrap) are the only places that read env config.
- Domain code receives config through injected infrastructure services, not through raw env reads.

#### C.1.2 Registering config at the root

In `AppModule`:

```ts
@Module({
    imports: [
        ConfigModule.forRoot({
            isGlobal: true
            // validate with the repo's standard validator (joi or class-validator-based) if the project uses one
        })
        // ...other infrastructure and feature modules
    ]
})
export class AppModule {}
```

- `isGlobal: true` so features do not re-import `ConfigModule`.
- If the project has an env schema validator, use it. Fail fast at startup if required env is missing.

#### C.1.3 Env var naming

- SCREAMING_SNAKE_CASE: `MONGO_URI`, `CLERK_ADMINS_JWT_KEY`, `RESEND_API_KEY`, `SQS_QUEUE_URL`, `AWS_REGION`.
- Vendor-prefixed: `AWS_*`, `CLERK_*`, `RESEND_*`, `SENTRY_*`.
- Stable. Do not rename an env var without auditing `ConfigService.get(...)` / `getOrThrow(...)` call sites and every deployment config (compose, k8s manifests, CI secrets).

#### C.1.4 Derived / computed config

- If a value is derived from env data and reused often (e.g., a constructed URL, a parsed list, a boolean flag), compute it **once** in the infrastructure or integration service that consumes it.
- Cache the derived value on the service instance. Do not recompute on every call.

#### C.1.5 Forbidden

- ❌ `process.env.FOO` anywhere in `src/modules/**` (feature code).
- ❌ `process.env.FOO` anywhere in `src/common/**` unless the file is a pure bootstrap utility.
- ❌ Direct env reads inside domain services, guards, or DTOs.
- ❌ Silently falling back to a default for a required value. Fail fast with `getOrThrow`.

### C.2 — Dates, times, and time zones

#### C.2.1 Hard rules

- **Date-only values are business values, not UTC instants.** Treat `YYYY-MM-DD` as a business date, not an `ISO 8601` datetime.
- **Never** parse `YYYY-MM-DD` inputs with naive `new Date("2026-03-05")` — the JavaScript engine interprets that as UTC midnight, which is almost never what the business means.
- Use `date-fns-tz` (the project default) for all business timezone conversions.
- Centralize the **canonical business timezone** in a single constant or config value. Do not sprinkle `"Asia/Dubai"` or `"America/New_York"` literals across services.

#### C.2.2 Conversion helpers

Define shared timezone helpers in `src/common/utils/` (or the project's equivalent):

```ts
// src/common/utils/timezone.util.ts
import { zonedTimeToUtc, utcToZonedTime } from "date-fns-tz";

export const BUSINESS_TIMEZONE = "Asia/Dubai"; // or from ConfigService at bootstrap

export const businessDateToUtcStart = (dateOnly: string): Date => zonedTimeToUtc(`${dateOnly}T00:00:00`, BUSINESS_TIMEZONE);

export const businessDateToUtcEnd = (dateOnly: string): Date => zonedTimeToUtc(`${dateOnly}T23:59:59.999`, BUSINESS_TIMEZONE);
```

- All services that interpret `YYYY-MM-DD` inputs use these helpers.
- Do not duplicate timezone logic inside a service. If the helper doesn't cover your case, extend the helper.

#### C.2.3 Start-of-day and end-of-day

- Be explicit. A date range `2026-03-05 → 2026-03-05` means `[2026-03-05 00:00:00 business-tz, 2026-03-05 23:59:59.999 business-tz]` — **not** a single instant.
- Always call the right helper (`businessDateToUtcStart` vs. `businessDateToUtcEnd`). Do not silently use one for both.

#### C.2.4 Persistence

- Persist instants in **UTC** (or the project's clearly defined standard — follow what already exists).
- Mongoose stores `Date` as BSON datetime, which is UTC. That is the target representation.
- Do not persist `YYYY-MM-DD` strings for timeline-sensitive fields (event start, deadline). Convert at the boundary using the helper, store as `Date`.
- Purely calendrical fields (e.g., "payment anniversary day of month") may stay as integers or date-only strings — document the decision at the schema.

#### C.2.5 DTO-level date validation

- Validate `YYYY-MM-DD` format with `@Matches(/^\d{4}-\d{2}-\d{2}$/, { message: "start-date-invalid-format" })`.
- Convert to UTC in the **service**, using the shared helper. Not in the DTO, not in the controller.

#### C.2.6 Reading dates for responses

- When returning dates to clients, use the repository's documented response format. If the project emits ISO 8601 UTC strings, do that. If it emits business-tz `YYYY-MM-DD`, use `utcToZonedTime(...)` + formatter.
- Do not mix formats within the same response.

#### C.2.7 Forbidden

- ❌ `new Date("2026-03-05")` anywhere in business code. Use the helper.
- ❌ Hardcoded timezone literals scattered across services. One canonical constant.
- ❌ Silent UTC assumptions for business date inputs.
- ❌ Mixing `moment` or `dayjs` into a `date-fns-tz`-standard project. Use the project default.

### C.3 — Where to put timezone logic

- Canonical timezone constant: `src/common/utils/timezone.util.ts` (or project equivalent).
- Date helpers: same file.
- Never a feature-local copy. If a feature needs a behavior that doesn't exist, extend the shared helper.
