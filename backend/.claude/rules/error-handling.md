# Error handling & validation rules

These rules define how errors, exceptions, and validation **must** be handled in this project.

## 1 — Exception types

- Use NestJS built-in HTTP exceptions exclusively: `NotFoundException`, `BadRequestException`, `ForbiddenException`, etc.
- For RPC handlers, use `RpcException` from `@nestjs/microservices`.
- **Never** create custom exception classes — the built-in set covers all cases.

## 2 — Error message format

- All exception messages are **kebab-case** strings that serve as translation keys.
- Messages should be descriptive and context-specific.

```ts
// GOOD
throw new NotFoundException("workshop-does-not-exist");
throw new BadRequestException("workshop-not-accepting-registrations");

// BAD
throw new NotFoundException("Not found");
throw new BadRequestException("Bad request");
```

## 3 — Error response shape

All global exception filters produce a consistent `ErrorResponse`:

```ts
{
    statusCode: number;
    timestamp: string;   // ISO 8601
    path?: string;       // request URL or RPC action
    message: string;     // kebab-case translation key
}
```

Do not alter this shape or add extra fields without approval.

## 4 — Global exception filters

Filters are registered in `AppModule` via `APP_FILTER`. The registration order determines priority (last registered = highest priority):

| Filter                 | Catches                                                                          | Status code               |
| ---------------------- | -------------------------------------------------------------------------------- | ------------------------- |
| `AllExceptionsFilter`  | Everything not caught by specific filters                                        | `500`                     |
| `MongoExceptionFilter` | `MongooseError.ValidationError`, `CastError`, `MongoServerError` (duplicate key) | `422` / `400`             |
| `RpcExceptionFilter`   | `RpcException`                                                                   | `503`                     |
| `HttpExceptionFilter`  | `HttpException` and subclasses                                                   | Preserves original status |

## 5 — DTO validation (class-validator)

- All DTOs use `class-validator` decorators with explicit `message` strings in **kebab-case**.
- The global `ValidationPipe` is configured with `transform: true`, `whitelist: true`, `forbidNonWhitelisted: true`.
- Validation messages should be clear and specific to the field.

```ts
// GOOD
@IsNotEmpty({ message: "please-provide-a-name" })
@Matches(/^\d{4}-\d{2}-\d{2}$/, { message: "start-date-invalid-format" })

// BAD
@IsNotEmpty()  // missing message
@IsNotEmpty({ message: "Field is required" })  // not kebab-case
```

## 6 — Mongoose schema validation

- Schema-level validation is a **secondary safety net** behind DTO validation.
- `required` fields use the tuple syntax: `required: [true, "kebab-case-message"]`.
- Custom validators use `validate: { validator: fn, message: "kebab-case-message" }`.

## 7 — Service-level error patterns

Follow this consistent pattern in every service method:

```ts
async getWorkshop(eventId: string, workshopId: string) {
    // Get the workshop by event ID
    const workshop = await this.workshopModel.findOne({ _id: workshopId, eventId });
    // Check if the workshop exists
    if (!workshop) throw new NotFoundException("workshop-does-not-exist");
    // Return workshop
    return workshop;
}
```

- Fetch the resource, then immediately check existence and throw if not found.
- Return a kebab-case success string for write operations (e.g., `"workshop-created-successfully"`).
- Return the document directly for read operations.

## 8 — Logging & monitoring

- Use NestJS `Logger` with a descriptive context string: `new Logger("service-events-<context>")`.
- Sentry is integrated via `@sentry/nestjs`. The `AllExceptionsFilter` uses `@SentryExceptionCaptured()` to auto-report unhandled errors.
- For caught errors in non-critical paths (e.g., SQS publish), log with `this.logger.error()` and call `captureExceptionSentry()` manually.
- **Never** expose internal error details to the client — log them server-side, return a generic kebab-case message.

## 9 — Fire-and-forget operations

- Use `void` keyword for async operations that should not block the response (e.g., sending emails via SQS).
- These operations should still handle their own errors internally (try/catch with logging).

```ts
// GOOD — fire-and-forget with void
void this.sqsService.publishMessage({ ... });

// BAD — awaiting a non-critical side effect
await this.sqsService.publishMessage({ ... });
```
