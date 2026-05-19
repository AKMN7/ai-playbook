# Code style, TypeScript, verification, and error handling

This file covers two intertwined concerns: how the code reads (Part A — style, comments, TypeScript discipline, verification, forbidden patterns) and how failures propagate (Part B — exception types, global filters, the public `ErrorResponse` shape, logging, Sentry).

The two belong together because every coding-style rule about clarity and forbidden patterns is reinforced by the error pipeline: bad style leaks into bad errors, and vice versa.

---

## Part A — Coding style and verification

### A.1 — Consistency is the highest priority

- **New code blends in.** Read nearby files in the same feature or layer before writing. Mirror their structure, naming, exports, imports, decorator usage, controller/service split, comment density, and error style.
- Do not introduce a second style next to an existing one. If the surrounding code is not ideal, **flag it separately** instead of "fixing" it inline.
- Where these rules are silent, follow **local precedent**.
- Divergence from project conventions requires direct user approval or an ADR.

### A.2 — Clarity over cleverness

- Optimize for the reader. Prefer explicit branches over compressed cleverness when business logic matters.
- Prefer boring, maintainable code over generic, clever, or "future-proof" abstractions.
- Keep functions reasonably short, but do not split trivial code just to satisfy an arbitrary size rule.
- Name things so the reader does not need to jump to read the body to understand the call site.

### A.3 — Do not overengineer

- No abstractions, base classes, factories, generic infrastructure, helper layers, or reusable utilities unless the current task materially needs them.
- No speculative "what if we need this later" extensibility.
- No half-finished implementations.
- No error handling for scenarios that cannot happen — trust internal code and framework guarantees. Validate only at real system boundaries (user input at DTOs, external APIs at integration services).
- No backwards-compat shims when you can just change the code. This is an internal codebase.
- No feature flags for hypothetical future requirements.
- Three similar lines is better than a premature abstraction.

### A.4 — Comments

#### A.4.1 Service-method step comments

- Every logical step inside a service method gets a **one-line comment above it**.
- Pattern: `// <Verb> <what>`.

    ```ts
    // Get the workshop by event ID
    const workshop = await this.workshopModel.findOne({ _id: workshopId, eventId });
    // Check if the workshop exists
    if (!workshop) throw new NotFoundException("workshop-does-not-exist");
    // Return workshop
    return workshop;
    ```

- Match the density of the **surrounding file**. Heavily commented service? Add comments. Sparsely commented service? Keep it sparse.
- Comments describe the **step/intent**, not the mechanics the code already shows.

#### A.4.2 When to add a comment outside service methods

- Non-obvious intent.
- Invariants.
- Transaction boundaries.
- Business rules that are not self-evident.
- Workarounds for a specific bug or framework quirk.

#### A.4.3 When to never add a comment

- ❌ To narrate obvious code (`// increment i`).
- ❌ To describe what a well-named identifier already says.
- ❌ To reference the current task, ticket, or caller (`// added for issue #1234`, `// used by workshop flow`).
- ❌ Multi-paragraph docstrings on internal functions. One line is almost always enough.
- ❌ Comments out code for later. Delete it. Git history is the archive.

#### A.4.4 Legacy files

- If you are editing a legacy file with comment conventions different from these rules, **preserve the local comment style** for that file. Do not reformat comments opportunistically.

### A.5 — TypeScript rules

- Target the repository's established TS config. `strictNullChecks` is expected. `ES2023` (or the project standard) is the target.
- Use `import type` for type-only imports.
- **Avoid `any`.** If a real type is available, use it. If a third-party type is missing, declare a minimal local type.
- If the repository already relaxes a TypeScript rule in a specific area, **follow the local constraint**. Do not opportunistically tighten it while doing unrelated work.
- Prefer `readonly` on class fields that never reassign (`private readonly logger = ...`, `private readonly connection: Connection`).
- Prefer `as const` for literal tuples and readonly arrays when it improves inference.
- Never use `ts-ignore` / `ts-expect-error` without a one-line comment explaining why.
- Do not cast to `any` to silence a type error. Diagnose the underlying cause.

### A.6 — Imports

- Use the repository's established path alias for cross-module imports (commonly the `src/` prefix: `import { Role } from "src/common/enums"`).
- Use **relative imports** (`./`, `../`) inside a small local feature when that is the repository standard.
- Use `import type` for type-only imports.
- Avoid barrel files unless the repository already uses them intentionally.
- Group imports: external packages first, then project imports, then relative imports. Leave one blank line between groups. Follow Prettier/ESLint ordering if the project enforces it.

### A.7 — Exports

- **Named exports only.** No default exports.
- One primary class, type, or function per file.
- If an existing legacy area uses default exports, do not churn files only to remove them.

### A.8 — Unused parameters

- Parameters that exist only for guard activation (e.g., `@CurrentUser()` on an admin-guarded route that does not consume the user) get a `_` prefix: `@CurrentUser() _user: User`.
- TypeScript and ESLint should both accept the `_` prefix without warning.

### A.9 — When to ask vs. when to autofix

#### A.9.1 Fix silently

- Trivial lint or formatting issues inside a file you are already editing.
- Obvious typos in comments or identifiers that are not part of a public contract.
- Automatically importing a missing symbol the task clearly needs.

#### A.9.2 Ask first

- Anything affecting business logic.
- Anything affecting API contracts (routes, DTO field names, error messages, response shapes, env keys).
- Anything affecting database schemas or indexes.
- Anything affecting module wiring or global providers (filters, guards, pipes).
- Introducing a new library dependency.
- Creating a new top-level folder or module.
- Ambiguous requirements, types, or edge cases.

### A.10 — Verification (Definition of Done)

A task is **not done** when the code merely looks correct. It is done when the change is wired correctly, structurally consistent with the repository, and **verified** with the repository's actual commands.

#### A.10.1 Discover the project's scripts first

- Read `package.json` to find real script names. Typical: `npm run build`, `npm run lint`, `npm test`, `npm run start:dev`.
- Also check for `Makefile`, `justfile`, or CI config when scripts are not obvious.
- Use project scripts. Do not invent ad hoc command lines.

#### A.10.2 Narrow verification first

- For a narrow code change, run targeted verification: TypeScript build, ESLint on the changed file(s), focused tests.
- Prefer the narrowest meaningful check for the scope of the change.

#### A.10.3 Broaden verification for risky changes

When you change module wiring, DTOs, schemas, guards, filters, global providers, or shared contracts, run the higher-risk verification:

- Full build (`npm run build`).
- Full lint (`npm run lint`).
- Relevant test suites.
- Manual request if an integration-heavy endpoint is touched.

#### A.10.4 Reporting

- **Never** claim lint, tests, or builds passed unless they were actually run.
- Report what you ran and what the result was.
- If no useful automated verification exists for the change (pure config, documentation), say so explicitly.
- If automated tests are stale or commented out, do not treat them as the product specification. Ask the user whether behavior was intentional.
- For UI/frontend-adjacent backend changes, note explicitly that backend tests do not verify UI behavior.

### A.11 — Forbidden patterns (change safety)

The items below are **hard rules**. Do not do them. They represent lessons from real incidents and keep the codebase coherent.

#### A.11.1 Architecture

- ❌ No second architectural style beside the existing one.
- ❌ No silent structural drift. New top-level folders require user approval when an existing feature or shared area is the correct home.
- ❌ No new top-level framework (tRPC, raw Express, GraphQL) alongside NestJS.
- ❌ No `@nestjs/microservices` decomposition unless the user explicitly requests it.

#### A.11.2 Layering

- ❌ No business logic in controllers, guards, filters, or decorators.
- ❌ No direct vendor SDK calls from controllers or domain services. Go through `modules/integration/`.
- ❌ No `process.env` reads in domain code. Go through `ConfigService`.
- ❌ No schema hooks (`pre("save")`, etc.) carrying primary business flow unless the project already standardizes that pattern.

#### A.11.3 Contracts

- ❌ No renames of route params, public error messages, DTO field names, response fields, or env keys without auditing downstream usage first.
- ❌ No changes to the `ErrorResponse` shape without user approval.
- ❌ No breaking public contract changes in a task that was scoped as internal.

#### A.11.4 Scope

- ❌ No unrelated refactors mixed into a feature task. If you notice something worth fixing outside the task, flag it separately.
- ❌ No bulk reformatting, reordering, or reorganizing beyond the change scope.
- ❌ No converting existing default exports to named exports (or vice versa) while doing unrelated work.

#### A.11.5 Shared code

- ❌ No feature models, feature schemas, or feature authorization rules in `common/`.
- ❌ No dumping helpers in `common/` that are only used by one feature.

#### A.11.6 Testing / verification

- ❌ No claiming tests pass without running them.
- ❌ No treating stale or abandoned tests as the product specification.
- ❌ No skipping hooks (`--no-verify`, `--no-gpg-sign`) to get a commit through. If a hook fails, fix the underlying issue.

#### A.11.7 Destructive actions

- ❌ No `git reset --hard`, `git push --force`, branch deletions, file deletions outside scope, or database-level destructive operations without explicit user confirmation.
- ❌ No amending a previously pushed commit unless the user explicitly asks.

### A.12 — Communication charter

- Keep text output short and concise. One-sentence updates at key moments, not running commentary.
- Reference files with markdown links so the user can jump: `[event.service.ts](src/modules/event/event.service.ts:42)`.
- End of turn: one or two sentences summarizing what changed and what's next. No more.
- Ask clarifying questions early. Silent assumptions cause rework.

---

## Part B — Error handling, filters, logging, and observability

This part defines how errors are thrown, caught, shaped, and logged. The public error response shape is **stable across all failures**, and internal logging is rich without leaking to clients.

### B.1 — Exception types

- Use NestJS **built-in HTTP exceptions** exclusively: `NotFoundException`, `BadRequestException`, `ForbiddenException`, `UnauthorizedException`, `ConflictException`, `UnprocessableEntityException`, `InternalServerErrorException`.
- Do **not** create custom exception classes. The built-in set covers every case in this project.
- Use `HttpException` with a specific status only when the built-in set does not match (rare).
- Use `RpcException` from `@nestjs/microservices` **only** if the project explicitly uses the microservices transport. This is not part of the default standard — see [architecture.md §B.12 — Not default unless explicitly needed](architecture.md).

### B.2 — Exception message format

- All exception messages are **kebab-case** strings that serve as translation keys.
- Messages are descriptive, context-specific, and stable. Clients may switch on them.

```ts
// GOOD
throw new NotFoundException("workshop-does-not-exist");
throw new BadRequestException("workshop-not-accepting-registrations");
throw new ForbiddenException("not-allowed-to-edit-event");
throw new ConflictException("email-already-enrolled");

// BAD
throw new NotFoundException("Not found");
throw new BadRequestException("Bad request");
throw new NotFoundException("Workshop not found in database");
```

- Do not use sentences, punctuation, uppercase, or vendor-specific phrasing.
- Never embed dynamic data in the message string (ids, emails). Use structured logging for that.

### B.3 — Public error response shape

All global exception filters produce a consistent `ErrorResponse`:

```ts
export type ErrorResponse = {
    statusCode: number;
    timestamp: string; // ISO 8601
    path?: string; // request URL
    message: string; // kebab-case translation key
};
```

- Defined in `src/common/types/error-response.type.ts` (or the project's equivalent).
- **Do not alter** this shape or add fields without user approval. Clients depend on it.
- Stack traces, raw exception objects, vendor payloads, secrets, database internals, and PII never appear in this response.

### B.4 — Global exception filters

Filters are registered via `APP_FILTER` in `AppModule`. **Registration order determines priority — the last-registered filter wins**.

| Filter                 | Catches                                                                           | Public status             |
| ---------------------- | --------------------------------------------------------------------------------- | ------------------------- |
| `AllExceptionsFilter`  | Everything not caught by specific filters (register first).                       | `500`                     |
| `MongoExceptionFilter` | `MongooseError.ValidationError`, `CastError`, `MongoServerError` (duplicate key). | `422` / `400` / `409`     |
| `HttpExceptionFilter`  | `HttpException` and subclasses (register last, highest priority).                 | Preserves original status |

Order in `AppModule` providers (top to bottom):

```ts
providers: [
    { provide: APP_FILTER, useClass: AllExceptionsFilter },
    { provide: APP_FILTER, useClass: MongoExceptionFilter },
    { provide: APP_FILTER, useClass: HttpExceptionFilter },
],
```

- `AllExceptionsFilter` is the fallback. It also integrates with Sentry (see §B.8).
- `MongoExceptionFilter` translates Mongoose/Mongo errors into the `ErrorResponse` shape with a sensible status:
    - `ValidationError` → `422`.
    - `CastError` → `400`.
    - Duplicate key (`MongoServerError` code `11000`) → `409` with a kebab-case message like `"duplicate-key"` or a schema-specific key.
- `HttpExceptionFilter` preserves the status from the thrown `HttpException` and produces the `ErrorResponse`.

Add new filters only when introducing a new exception category that the existing filters do not cover.

### B.5 — Validation errors (class-validator)

- The global `ValidationPipe` throws `BadRequestException` on validation failure. `HttpExceptionFilter` catches it.
- The default NestJS validation error shape is replaced by the project's `ErrorResponse`: the filter flattens the first validation message into `message` (kebab-case).
- Because every validator carries a kebab-case `message`, client-facing output remains consistent.

See [feature-pipeline.md Part C — DTOs and validation](feature-pipeline.md).

### B.6 — Service-level error pattern

Every service method follows this pattern:

```ts
async getWorkshop(eventId: string, workshopId: string) {
    // Get the workshop scoped by the parent event
    const workshop = await this.workshopModel.findOne({ _id: workshopId, eventId });
    // Check if the workshop exists
    if (!workshop) throw new NotFoundException("workshop-does-not-exist");
    // Return workshop
    return workshop;
}
```

- Fetch the resource. Immediately check existence. Throw `NotFoundException` if missing.
- For business-state failures, throw `BadRequestException("...-reason")`.
- For authorization failures, throw `ForbiddenException("...-not-allowed")`.
- For duplicate resource collisions (rare, usually the DB catches it), throw `ConflictException("...-already-exists")`.
- Write operations return a kebab-case success string. Read operations return the document or DTO.

See [feature-pipeline.md §B.2 — Service contract](feature-pipeline.md) for the full service contract.

### B.7 — Logging

#### B.7.1 Logger usage

- Every service has a NestJS `Logger` as a private field:

    ```ts
    private readonly logger = new Logger("service-<feature>");
    ```

- Integration services use an integration-prefixed context: `new Logger("integration-sqs")`.
- Guards, filters, and infrastructure services use descriptive contexts (`"guard-admin"`, `"filter-all-exceptions"`).

#### B.7.2 When to log

- **Operational failures at the point of failure** — integration errors, retryable failures, non-critical side-effect errors.
- **Unexpected exceptions** — captured automatically by `AllExceptionsFilter` via Sentry.
- **Fire-and-forget side effects** that fail: `this.logger.error("...", { error, payload })` + `captureException(err)`.

#### B.7.3 Never log

- ❌ Secrets, tokens, passwords.
- ❌ Full sensitive payloads (PII, card details, full auth headers).
- ❌ Raw Clerk JWTs or other bearer tokens.
- ❌ Full request bodies when they may contain sensitive fields.

#### B.7.4 Log shape

- Prefer **structured, contextual logs** over vague string logs.
- Include the minimum context needed to reproduce: resource id, parent id, operation name. Not the full document.

### B.8 — Sentry integration

- Sentry is integrated via `@sentry/nestjs`.
- Initialize Sentry in `src/instruments.ts` and import it at the **top** of `main.ts` before any framework imports:

    ```ts
    // src/main.ts
    import "./instruments";
    // ... then Nest imports
    ```

- `AllExceptionsFilter` uses `@SentryExceptionCaptured()` (or the equivalent decorator) to auto-report every unhandled exception it catches.
- For caught errors in non-critical paths (fire-and-forget SQS publishes, integration call failures), call `captureException(err)` manually inside the integration service.
- Do **not** send deliberately thrown user-facing exceptions (`NotFoundException("workshop-does-not-exist")`) to Sentry — they are expected control flow. The filter chain already distinguishes these.

### B.9 — Fire-and-forget errors

- Fire-and-forget calls (`void this.sqsService.publishMessage(...)`) handle errors **inside the integration service**:

    ```ts
    try {
        await this.client.send(command);
    } catch (err) {
        this.logger.error("sqs-publish-failed", { error: err });
        captureException(err);
        // Do not rethrow. Fire-and-forget contract.
    }
    ```

- The caller never sees the error. The ops path sees it via Sentry + structured logs.

### B.10 — Forbidden patterns

- ❌ Custom exception classes. Use NestJS built-ins.
- ❌ English-prose exception messages (`"Not found"`, `"Bad request"`).
- ❌ Exposing stack traces, vendor payloads, or database errors to clients.
- ❌ Filters that implement business logic or validation. Filters only translate responses.
- ❌ Logging secrets or sensitive payloads.
- ❌ Swallowing errors silently (`catch {}` with no log and no rethrow).
- ❌ Changing the `ErrorResponse` shape without user approval.
