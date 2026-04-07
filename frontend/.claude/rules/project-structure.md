# Project structure rules

These rules codify how files and folders **must** be organised in this repository.

## 0 — Global

- **Language** — TypeScript only (`.ts` / `.tsx`).
- **Imports** — Use absolute aliases (`@/…`) defined in `tsconfig.json`; never traverse above `src/`.
- **File naming**
    - Components → `PascalCase.tsx`
    - Hooks → `useHookName.ts`
    - Contexts / routes / services → `kebab-case.ts`
- **Exports**
    - If a file exposes **exactly one** entity, prefer `export default`.
    - If multiple entities are exposed, use **named exports**.
- **Styles** — Tailwind classes inline only..

## 1 — `src/components`

- A **folder = feature**. UI must not import across feature folders directly.
- The **general** folder includes should include purely generic, reusable UI that does not belong to any feature.
- `_app` hosts global providers, top-level app base layout components.

## 2 — `src/contexts`

- One file per React context, named `<thing>-provider/`.
- Each exposes `<Thing>Provider` and `use<Thing>()`.
- Hold only _cross-feature_ state (auth, theme, etc.).

## 3 — `src/hooks`

- A **folder = feature**. Each folder resembels a feature.
- Mirror backend resources; file name pattern `use<Action><Resource>.ts`.
- CRUD hooks wrap TanStack Query mutations/queries.
- Non-feature utilities live in the root of `src/hooks`.

## 4 — `src/lib`

- Flat collection of **pure, framework-agnostic** helpers.
- Large helper → separate file (e.g. `ZodSchemaCreator.ts`).
- Small helpers → `utils.ts`.

## 5 — `src/locales`

- Exactly two flat JSON files (`en.json`, `ar.json`).
- Keys use `kebab-case`.
- Add / modify keys in **both** files within the same PR.

## 6 — `src/routes`

- File-system routing via **@tanstack/react-router**.
- Param segments use `$` prefix (`$id`).
- Optional `_layout.tsx` per folder for nested layouts.

## 7 — `src/services`

- `API.ts` centralises/reuses config.
- Folder per **feature** approach.
- `.api.ts` → Concrete network calls built on utilzing `API.ts`. _Only_ async functions and Feature type declaration here.
- `.schemas.ts` → Zod validators and `type` aliases derived with `z.infer`.
- `.helpers.ts` → Pure, feature-specific utilities. Functions are **PascalCase**.

## Do not modify

- **Never modify** `src/components/ui/` — shadcn/ui components are managed by the CLI; do not edit them manually.
