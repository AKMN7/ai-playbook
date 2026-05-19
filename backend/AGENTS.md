# Backend Project Standard For Codex

NestJS modular-monolith backend. Clerk-issued JWTs validated server-side. MongoDB via Mongoose. AWS S3/SQS, Resend, and Sentry through a dedicated integration layer.

## Codex operating notes

- **Search**: use `rg` for content search and `rg --files` for file listing. Avoid `grep -R`, `find`, and `ls -R`.
- **Edits**: use `apply_patch` for all file modifications. Do not stream full file rewrites when a targeted patch will do.
- **Preserve user changes and dirty worktree state.** Do not discard uncommitted edits, reset branches, or force-push without explicit user confirmation.
- **Never edit generated or CLI-managed files** (`dist/`, `tsconfig.build.tsbuildinfo`, `package-lock.json` unless adding/removing deps, lockfiles in general, Nest-CLI-generated artifacts).
- **Run the repository's verification commands after meaningful changes** — discover them from `package.json` (typical: `npm run build`, `npm run lint`, `npm test`).
- **AGENTS.md scope-nesting**: this repo intentionally uses one repo-root `AGENTS.md`. Do not create nested `AGENTS.md` or per-feature override files unless the user explicitly asks.
- Treat `.cursor/`, `.claude/CLAUDE.md`, `TEAM_GUIDE.md`, and old project docs as historical reference only.

## Charter

**Structural consistency is the highest priority — not a preference.** Read the nearest existing feature before writing, mirror its layout, naming, decorator usage, and comment density.

When the codebase and these rules disagree:

- **Narrow edits in an existing area** → preserve current behavior and match the local pattern, even when the local pattern deviates from these rules. Consistency with the surrounding file wins.
- **New modules, new files, or deliberate refactors** → follow this ruleset exactly.
- **Clearly broken local patterns** → do not silently "fix" them inside an unrelated task. Flag the inconsistency and ask before changing.

Treat Codex `.rules` files as command-approval and sandbox policy, not as a place for architecture or coding conventions.

## Priority order when rules compete

1. Preserve correctness and the user's explicit requirements for this turn.
2. Preserve existing runtime behavior unless change is explicitly requested.
3. Match the established pattern of the area being edited.
4. For new work, follow the target architecture defined in `.claude/rules/*.md`.
5. Prefer simpler and more maintainable code over clever or generic code.

## Tech stack

- **Framework**: NestJS (`@nestjs/common`, `@nestjs/core`, `@nestjs/platform-express`), TypeScript with `strictNullChecks`
- **Persistence**: MongoDB + Mongoose via `@nestjs/mongoose`
- **Validation**: `class-validator` + `class-transformer` + `@nestjs/mapped-types`; Zod only for dynamic runtime payloads
- **Auth**: Clerk (identity) + `passport-jwt` / `@nestjs/passport` / `@nestjs/jwt` (server-side validation)
- **Config**: `@nestjs/config` — all env reads via `ConfigService.getOrThrow(...)`
- **Security/transport**: `helmet`, `@nestjs/throttler`, `cookie-parser` (only when cookies are read)
- **Dates**: `date-fns-tz` with a single canonical business-timezone constant
- **Integrations**: `@aws-sdk/client-s3`, `@aws-sdk/client-sqs`, `@aws-sdk/s3-request-presigner`, `resend`, `axios` — all behind `modules/integration/`
- **Observability**: `@sentry/nestjs`, initialized in `src/instruments.ts`
- **Test/lint**: `jest`, `@nestjs/testing`, `ts-jest`, `eslint`, `typescript-eslint`, `prettier`

## Imports

- Path alias: `src/` prefix for cross-module imports (`import { ErrorResponse } from "src/common/types/error-response.type"`)
- Relative imports inside a small local feature only
- `import type` for type-only imports
- Named exports only — no default exports
- Avoid barrel `index.ts` files

## Directory map

| Path                       | Purpose                                                                      |
| -------------------------- | ---------------------------------------------------------------------------- |
| `src/main.ts`              | Bootstrap: app creation, global pipes, Helmet, CORS, server start            |
| `src/app.module.ts`        | Root composition: infra modules, feature modules, global guards/filters/pipe |
| `src/instruments.ts`       | Sentry init — imported at the **top** of `main.ts` before any Nest imports   |
| `src/common/`              | Shared `constants/`, `decorators/`, `dto/`, `types/`, `utils/`               |
| `src/filters/`             | Global exception filters (`APP_FILTER`)                                      |
| `src/modules/auth/`        | JWT strategy, guards, decorators, normalized `User` type                     |
| `src/modules/integration/` | One service per vendor (`sqs.service.ts`, `s3.service.ts`, etc.)             |
| `src/modules/<feature>/`   | One feature per business domain: module, controller, service, DTOs, schemas  |

## Rule files — shared source of truth

The rule files in `.claude/rules/` are the **shared source of truth** for both Codex (this file) and Claude (`.claude/CLAUDE.md`). **Do not fork rule content into this AGENTS.md** — that causes drift between the two assistants. If a rule needs updating, edit the file in `.claude/rules/` directly.

| File                                  | Covers                                                                                                                  |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| [.claude/rules/architecture.md](.claude/rules/architecture.md)         | Modular-monolith layout, root files, module wiring, library stack, Dockerization, file/folder/class/method/enum naming  |
| [.claude/rules/feature-pipeline.md](.claude/rules/feature-pipeline.md) | End-to-end feature workflow (schema → DTO → service → controller → module), controller/service contracts, Mongoose schemas/indexes, DTO validation, idempotency |
| [.claude/rules/boundaries.md](.claude/rules/boundaries.md)             | Clerk JWT auth + normalized `User`, integration layer (vendor SDKs, fire-and-forget side effects, webhooks), `ConfigService` access, timezone-aware date handling |
| [.claude/rules/style-and-errors.md](.claude/rules/style-and-errors.md) | Comment style, TypeScript discipline, verification (Definition of Done), forbidden patterns, NestJS exception types, global filters, `ErrorResponse` shape, logging, Sentry |

## Common commands

| Command              | Purpose                                                 |
| -------------------- | ------------------------------------------------------- |
| `npm run dev`        | Dev server (`nest start --watch`)                       |
| `npm run build`      | Compile (`nest build`)                                  |
| `npm run lint`       | ESLint + auto-fix                                       |
| `npm test`           | Jest unit tests (in-band: `jest -i`)                    |
| `npm run test:e2e`   | E2E suite                                               |
| `npm run start:prod` | Production start (`node dist/main`) — used in Docker    |

## Explicit vs. autonomous

- **Autofix silently**: trivial lint, formatting, or typos inside files you are already editing.
- **Ask before**: business logic, API contracts, DB schemas, folder structure, module wiring, guards, public error messages, new top-level modules, new shared utilities, new library dependencies, new architectural patterns.
