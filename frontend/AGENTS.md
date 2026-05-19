# Codex Project Rules

Scope: this file applies to the entire repository. If a nested `AGENTS.md` is added later, its instructions apply to files under that nested directory and override conflicting root instructions.

## Operating rules

Consistency is the highest priority. Before editing code, inspect nearby files in the same feature and layer, then match their structure, naming, exports, imports, formatting, data flow, Tailwind usage, error handling, and comment density.

Keep changes tightly scoped to the user request. Do not refactor unrelated code, rename files, change APIs, add dependencies, or introduce new architectural patterns unless explicitly asked.

Use Codex like a maintainer:

- Search with `rg` / `rg --files` first.
- Read relevant existing files before editing.
- Use `apply_patch` for manual edits.
- Preserve user changes and dirty worktree state.
- Never edit generated or CLI-managed files (`src/components/ui/`, `src/routeTree.gen.ts`).
- Run project verification after meaningful code changes: `npm run lint` and `npm run build`. If verification cannot run, state that in the final response.

Default to local precedent when rules are silent. If written rules and local precedent conflict, follow the written rules.

## Tech stack

- **Vite** + **React 19** + **TypeScript** (strict)
- **TanStack Router** — file-system routing under `src/routes/`
- **TanStack Query** — all API reads/writes go through `useQuery` / `useMutation`
- **TanStack Table** — data tables
- **React Hook Form** + **Zod** — schema-first forms (`zodResolver`)
- **Tailwind CSS 4** — inline utility classes only (`@tailwindcss/vite`)
- **shadcn/ui** — primitives in `src/components/ui/` (CLI-managed; never edited manually)
- **Clerk** — authentication
- **Axios** — HTTP; config centralised in `src/services/API.ts`
- **i18next** + **react-i18next** — English (`en-US`) + Arabic (`ar-SA`)
- **sonner** — toasts
- **lucide-react** — icons
- **date-fns** / **date-fns-tz** — date work
- **PapaParse** — CSV export
- **npm** with `package-lock.json` — package manager

Do not add Redux, Zustand, CSS modules, styled-components, another router, another HTTP client, another icon set, or another UI kit.

## Import alias

- `@/` maps to `./src/*` through explicit `paths` entries (no `compilerOptions.baseUrl`).
- Use absolute imports only; do not traverse above `src/`.

## Directory map (`src/`)

| Path                  | Purpose                                                                                                          |
| --------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `components/`         | One folder per feature; `general/` for shared UI; `_app/` for global layout / providers                          |
| `contexts/`           | Cross-feature React contexts (`<Thing>Provider` + `useThing`)                                                    |
| `hooks/`              | One folder per feature; TanStack Query hooks; `use<Action><Resource>.ts`                                         |
| `lib/`                | Pure, framework-agnostic helpers                                                                                 |
| `locales/`            | `en.json` + `ar.json` (flat keys, kebab-case)                                                                    |
| `routes/`             | TanStack file-system routes (`kebab-case.tsx`, `$param.tsx`, `_layout.tsx`)                                      |
| `services/`           | Per-feature API + schemas; `API.ts` is the Axios entry                                                           |
| `services/<feature>/` | `<Feature>.api.ts` (calls), `<Feature>.schemas.ts` (Zod), `<Feature>.helpers.ts` (pure utils) — PascalCase prefix |

## Rule files

The detailed rules live in `.claude/rules/` and are the **shared source of truth** for both Codex and Claude — keep them in sync rather than duplicating content here. Read the relevant file before writing code in that area:

| File                                | Covers                                                                                                                          |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `.claude/rules/architecture.md`     | Folder layout, file naming per area, routing, Clerk auth wiring, tooling (Vite / TS / ESLint / Prettier / env vars / aliases)   |
| `.claude/rules/feature-workflow.md` | End-to-end CRUD pattern: service → hook → form → table → loading/error/empty states → where state lives                         |
| `.claude/rules/conventions.md`      | TypeScript / React style, naming (component / hook / helper / route / service / schema / query keys), Tailwind + RTL, i18n keys |

When editing or adding rules, update the file under `.claude/rules/`. Do not fork rule content into this AGENTS.md.
