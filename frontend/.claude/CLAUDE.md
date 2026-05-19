# sideproject-fe

Front-end for the admin-side events management app.

## Top priority: consistency

**Consistency is the highest priority** for any change in this repo. When implementing a task, do not invent a new style or pattern—mirror what already exists.

Before writing or editing code, read nearby files in the same feature or layer and **match them**—**literally everything**: structure and placement of files and folders, naming, exports, TypeScript and React patterns, logic and data flow (how hooks, services, and components are split), Tailwind usage and class ordering where a local pattern exists, spacing and formatting, comment style and how often comments appear (including matching a file that stays uncommented), error handling, and any other habit the surrounding code establishes.

When asked to do something, **default to the current structure** of the repo and that feature; do not introduce a parallel style. If something is ambiguous, prefer **consistency with the surrounding codebase** over a "cleaner" or generic alternative. The written rules in `.claude/rules/*.md` apply; where they are silent, **follow local precedent** in the files you touch.

**In short:** match the house style of the code and files next to your change—spacing, style, comments, logic, structure, and placement included.

## Tech stack

- **Vite** + **React 19** + **TypeScript** (strict)
- **TanStack Router** — file-system routing under `src/routes/`
- **TanStack Query** — all API reads/writes go through `useQuery` / `useMutation` (see `.claude/rules/feature-workflow.md`)
- **React Hook Form** + **Zod** — schema-first forms (`zodResolver`)
- **Tailwind CSS 4** — inline utility classes only (`@tailwindcss/vite`)
- **i18next** + **react-i18next** — English + Arabic
- **Clerk** — authentication
- **Axios** — HTTP; config centralised in `src/services/API.ts`
- **shadcn/ui** — primitives in `src/components/ui/` (CLI-managed; do not edit manually)
- **sonner** — toasts

## Import alias

- `@/` maps to `./src/*` through explicit `paths` entries (no `compilerOptions.baseUrl`).
- Use absolute imports only; do not traverse above `src/`.

## Directory map (`src/`)

| Path                  | Purpose                                                                                       |
| --------------------- | --------------------------------------------------------------------------------------------- |
| `components/`         | One folder per feature; `general/` for shared UI; `_app/` for global layout / providers       |
| `contexts/`           | Cross-feature React contexts (`<Thing>Provider` + `useThing`)                                 |
| `hooks/`              | One folder per feature; TanStack Query hooks; `use<Action><Resource>.ts`                      |
| `lib/`                | Pure, framework-agnostic helpers                                                              |
| `locales/`            | `en.json` + `ar.json` (flat keys, kebab-case)                                                 |
| `routes/`             | TanStack file-system routes (`kebab-case.tsx`, `$param.tsx`, `_layout.tsx`)                   |
| `services/`           | Per-feature API + schemas; `API.ts` is the Axios entry                                        |
| `services/<feature>/` | `<Feature>.api.ts` (calls), `<Feature>.schemas.ts` (Zod), `<Feature>.helpers.ts` (pure utils) — PascalCase prefix |

## Rule files

Conventions and workflows live in `.claude/rules/*.md`. Read the relevant one before writing code:

| File                  | Covers                                                                                                                            |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `architecture.md`     | Folder layout, file naming per area, routing, Clerk auth wiring, tooling (Vite / TS / ESLint / Prettier / env vars / aliases)     |
| `feature-workflow.md` | End-to-end CRUD pattern: service → hook → form → table → loading/error/empty states → where state lives                           |
| `conventions.md`      | TypeScript / React style, naming (component / hook / helper / route / service / schema / query keys), Tailwind + RTL, i18n keys   |
