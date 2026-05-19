# Interactive Project Standard For Codex

Touch-first exhibit and kiosk front-end for public displays. Built as a Vite/React application with realtime state sync, short visitor sessions, and kiosk deployment constraints.

## Codex operating notes

- **Search**: use `rg` for content search and `rg --files` for file listing. Avoid `grep -R`, `find`, and `ls -R`.
- **Edits**: use `apply_patch` for all manual file modifications.
- **Preserve user changes and dirty worktree state.** Do not discard uncommitted edits, reset branches, or force-push without explicit user confirmation.
- **Never edit generated or CLI-managed files** (`src/components/ui/`, `src/routeTree.gen.ts`, build output, lockfiles unless dependencies change).
- **Run the repository's verification commands after meaningful changes** - discover them from `package.json` (typical: `npm run lint`, `npm run build`).
- **Browser-check UI work** on the target viewport when possible. Kiosk and touch changes need visual verification, not just type checks.
- **AGENTS.md scope-nesting**: this repo intentionally uses one repo-root `AGENTS.md`. Do not create nested `AGENTS.md` or per-feature overrides unless the user explicitly asks.

## Charter

**Structural consistency is the highest priority.** Before writing code, read nearby files in the same feature or layer, then mirror their layout, naming, exports, imports, hook/service split, Tailwind usage, comment density, and error handling.

Interactive work has extra constraints:

- Visitor sessions are short and resettable.
- Screens must work with fingers, public-space lighting, and non-technical users.
- Realtime state and kiosk lockdown are runtime boundaries, not incidental utilities.
- Browser-only behavior must be checked in the browser whenever the change affects interaction, layout, routing, WebSocket state, or kiosk mode.

When the codebase and these rules disagree:

- **Narrow edits in an existing area** -> preserve current behavior and match the local pattern, even when it differs from this ruleset.
- **New screens, features, runtime providers, or deliberate refactors** -> follow `.claude/rules/*.md`.
- **Clearly broken local patterns** -> flag the inconsistency and ask before changing it as part of an unrelated task.

Treat Codex `.rules` files as command-approval and sandbox policy, not as a place for architecture or coding conventions.

## Priority order when rules compete

1. Preserve correctness and the user's explicit requirements for this turn.
2. Preserve the exhibit runtime behavior unless change is explicitly requested.
3. Match the established pattern of the area being edited.
4. For new work, follow the target architecture defined in `.claude/rules/*.md`.
5. Prefer simple, observable, resettable flows over clever abstractions.

## Tech stack

- **Vite** + **React 19** + **TypeScript** (strict)
- **TanStack Router** - file-system routing under `src/routes/`
- **TanStack Query** - all HTTP reads/writes go through `useQuery` / `useMutation`
- **React Hook Form** + **Zod** - schema-first forms (`zodResolver`)
- **Tailwind CSS 4** - inline utility classes only (`@tailwindcss/vite`)
- **shadcn/ui** - primitives in `src/components/ui/` (CLI-managed; never edited manually)
- **i18next** + **react-i18next** - English and Arabic minimum; event-specific locales when required
- **Socket.IO** (`socket.io-client`) - realtime exhibit state sync through a single socket singleton
- **Axios** - HTTP; config centralized in `src/services/API.ts`
- **sonner** - toasts
- **lucide-react** - icons

Do not add Redux, Zustand, CSS modules, styled-components, another router, another HTTP client, another icon set, or another UI kit without explicit approval.

## Imports

- `@/` maps to `./src/*`; mirror it in Vite and TypeScript config.
- Use `@/...` imports for cross-folder source imports.
- Use relative imports only inside a small local folder when that is the local precedent.
- Use `import type` for type-only imports.

## Directory map

| Path                       | Purpose                                                                                         |
| -------------------------- | ----------------------------------------------------------------------------------------------- |
| `src/main.tsx`             | App mount, router/query providers, i18n wiring, global providers                                |
| `src/index.css`            | Tailwind directives, theme variables, global kiosk/touch CSS lockdown                           |
| `src/components/_app/`     | App shell, root wrappers, provider-adjacent UI                                                  |
| `src/components/general/`  | Feature-agnostic reusable UI                                                                    |
| `src/components/<feature>/` | One feature or visitor flow; screens use `*Screen.tsx`                                          |
| `src/contexts/`            | Cross-feature providers (`<Thing>Provider` + `useThing`)                                        |
| `src/hooks/`               | Feature query/mutation hooks plus root-level runtime hooks (`useKioskMode`, `useAutoSubmitForm`) |
| `src/lib/`                 | Pure helpers and constants                                                                      |
| `src/locales/`             | Flat locale JSON files (`en.json`, `ar.json`, optional event locales)                           |
| `src/routes/`              | TanStack file-system routes; shallow routes that host screen flows                              |
| `src/services/API.ts`      | Axios entrypoint                                                                                |
| `src/services/WS.ts`       | Socket.IO singleton                                                                             |
| `src/services/state/`      | Realtime state Zod schemas and defaults                                                         |
| `src/services/<feature>/`  | `<Feature>.api.ts`, `<Feature>.schemas.ts`, `<Feature>.helpers.ts`                              |

## Rule files - shared source of truth

The rule files in `.claude/rules/` are the **shared source of truth** for both Codex and Claude. Do not fork rule content into this AGENTS.md. If a rule needs updating, edit the file in `.claude/rules/` directly.

| File                                                            | Covers                                                                                                  |
| --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| [.claude/rules/architecture.md](.claude/rules/architecture.md)  | Folder layout, toolchain, routing, imports, generated files, project structure                          |
| [.claude/rules/feature-workflow.md](.claude/rules/feature-workflow.md) | Screen-flow features, query/mutation flows, forms, loading/error states, session state ownership |
| [.claude/rules/realtime-and-kiosk.md](.claude/rules/realtime-and-kiosk.md) | Socket singleton, realtime schemas/provider, auto-submit, kiosk lockdown, deployment launcher   |
| [.claude/rules/conventions.md](.claude/rules/conventions.md)    | TypeScript/React style, naming, touch-first UI, Tailwind/RTL, translation, verification                 |

## Common commands

Discover exact commands from `package.json`. Typical Vite commands:

| Command         | Purpose                      |
| --------------- | ---------------------------- |
| `npm run dev`   | Local dev server             |
| `npm run build` | Type-check and production build |
| `npm run lint`  | ESLint                       |
| `npm run preview` | Preview built app          |

## Explicit vs. autonomous

- **Autofix silently**: trivial lint, formatting, typos, missing translation counterparts in files already being edited.
- **Ask before**: changing screen flow, realtime state schema, socket event names, kiosk launcher behavior, public copy strategy, folder structure, API contracts, new dependencies, or persistent storage.
