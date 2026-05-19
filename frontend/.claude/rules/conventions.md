# Conventions

How code looks: TypeScript style, naming, Tailwind/RTL styling, and i18n keys. Read this whenever you're writing or renaming anything.

---

## 1 — Coding style charter

These rules apply on every change.

### 1.1 — Consistency over cleverness

- The single highest priority is **consistency** with existing code in the repo and especially with files in the same feature folder.
- Before writing or editing code, read 2–3 nearby files in the same feature/layer and **mirror them**: structure, naming, exports, TypeScript patterns, hook/service split, Tailwind class ordering, spacing, comment density, error handling, import order, etc.
- If a written rule and local precedent disagree, prefer the written rule. If the rules are silent, follow local precedent.
- Do not introduce a parallel style. A "cleaner generic alternative" is the wrong call here; matching the surrounding files is the right call.

### 1.2 — Stack assumptions (do not deviate without approval)

- **Vite + React 19 + TypeScript (strict)**
- **TanStack Router** for routing; **TanStack Query** for server state; **TanStack Table** for data tables
- **React Hook Form + Zod** (schema-first) via `zodResolver`
- **Tailwind CSS 4** inline utility classes only (no CSS modules, no styled-components)
- **shadcn/ui** primitives in `src/components/ui/` — CLI-managed, never edited manually
- **Clerk** for authentication; **Axios** wrapped by a single `src/services/API.ts` module
- **i18next + react-i18next** for translations — English (`en-US`) + Arabic (`ar-SA`)
- **sonner** for toasts; **lucide-react** for icons; **date-fns** (+ `date-fns-tz` when timezones matter)

If the task seems to require a new library, ask first. Do not add a dependency silently.

### 1.3 — TypeScript

- TypeScript only (`.ts` / `.tsx`). No JavaScript.
- Strict mode is on. `noUnusedLocals`, `noUnusedParameters`, `noFallthroughCasesInSwitch`, `noUncheckedSideEffectImports` are all enforced.
- Prefer `type` over `interface` for object shapes. Use `interface` only when extending a third-party interface that requires it.
- Imports use the `@/` alias (`@/* → ./src/*` through explicit `paths`, without `compilerOptions.baseUrl`). Never use relative imports that traverse above `src/`. Inside `src/`, prefer `@/...` over relative `./...` for any cross-folder import.
- Prefix intentionally unused identifiers with `_` (matches the eslint config: `varsIgnorePattern`, `argsIgnorePattern`, `caughtErrorsIgnorePattern` all `^_`).

### 1.4 — React component style

- Components are **`function` declarations**, not arrow functions:
    ```tsx
    function CreateWidgetDialog({ onClose }: CreateWidgetDialogProps) { ... }
    export default CreateWidgetDialog;
    ```
- One component per file. Default-export the component at the bottom of the file (not inline `export default function ...`).
- Props are a `type`, named `<ComponentName>Props`, declared above the component:
    ```tsx
    type CreateWidgetDialogProps = { onClose: () => void };
    ```
- Co-located internal helper components (e.g. a `TranslatedHeader` used only by a column-defs file) stay in the same file as named functions, **after** the main export.
- Don't wrap with `React.memo`, `forwardRef`, or HOCs unless local precedent already does — it generally doesn't.
- Avoid `useEffect` for derived state; use `useMemo` / direct rendering. The two acceptable `useEffect` use-cases are: hydrating a form on data load (`form.reset(data, { keepDefaultValues: false })`) and global keyboard / DOM listeners.

### 1.5 — Imports & ordering

Code is consistently organized as:

1. External packages (grouped together).
2. A blank line.
3. Internal `@/` imports (grouped together).

Example:

```ts
import { format } from "date-fns";
import { useForm } from "react-hook-form";
import { useTranslation } from "react-i18next";

import { Button } from "@/components/ui/button";
import { useCreateWidget } from "@/hooks/widgets/useCreateWidget";
import { WidgetSchema } from "@/services/widgets/Widgets.schemas";
import { zodResolver } from "@hookform/resolvers/zod";
```

Don't reorder existing imports gratuitously.

### 1.6 — Comments

- Default to **no comments**. Code is self-documenting via good names and small files.
- Add a comment only when it explains a non-obvious _why_: a hidden invariant, an external contract, a workaround. Never comment what the code already says.
- Multi-line JSDoc-style block comments are reserved for genuinely complex utilities (e.g. an idempotency-key wrapper, a shared retry policy). Don't write them for ordinary components/hooks.
- Match the comment density of the surrounding files. If the file has no comments, don't add any.

### 1.7 — Formatting (Prettier)

- 4-space indentation, semicolons, double quotes, no trailing commas, `arrowParens: "always"`, `printWidth: 360` (very wide; do not artificially line-break to look "narrow").
- `prettier-plugin-tailwindcss` runs automatically — do not hand-sort Tailwind classes; let the plugin order them. Configure `tailwindFunctions` for any tagged-template helper used as a class-name carrier.

### 1.8 — Error handling

- Network errors are normalized inside `src/services/API.ts` (returns `new Error(error?.response?.data?.message)`). Components never `try/catch` Axios calls directly; they consume hooks.
- Mutation hooks always set `onError` to `toast.error(t(error.message ?? "<fallback-i18n-key>"))`. The fallback key must exist in both `en.json` and `ar.json`.
- Throwing inside a context to enforce provider usage is the standard pattern (`if (context === undefined) throw new Error(...)`).

### 1.9 — Autofix vs. ask

- Trivial lint/format issues, missing translation keys, and obvious typos: fix silently.
- Anything affecting API contracts, schema shapes, hook return shapes, folder structure, or new dependencies: **ask before changing.**

### 1.10 — Don't do

- Don't add files to `src/components/ui/`. Use the shadcn CLI.
- Don't bypass `src/services/API.ts` by importing axios directly into components.
- Don't add backwards-compatibility shims, dead-code "TODO: remove" markers, or feature flags unless the task is explicitly about that.
- Don't internationalize `alt` text, `aria-label`s that are debug-only, or `console.*` strings.
- Don't introduce CSS files, styled-components, emotion, or any non-Tailwind styling layer.

---

## 2 — Naming conventions

Names are how the codebase stays IDE-searchable. Follow these literally.

### 2.1 — Components

| Aspect               | Rule                                                                                                                                              |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| File name            | **PascalCase.tsx** (`UserCard.tsx`, `CreateWidgetDialog.tsx`)                                                                                     |
| Component identifier | Same **PascalCase** as file (`function UserCard() { … }`)                                                                                         |
| Props shape          | Define **`type`**, never `interface` (no third-party-extension exceptions in normal application code)                                             |
| Props naming         | Must match component + `Props` suffix (`UserCardProps`)                                                                                           |
| Export               | **Default export** at the bottom of the file (`export default UserCard;`). Co-located helper components stay as named functions in the same file. |

Group prefixes used for components:

- `App*` for global chrome (`AppHeader`, `AppSidebar`, `AppLanguageSwitcher`, etc.).
- `Create<Resource>Dialog` / `Update<Resource>Dialog` / `View<Resource>Sheet` for resource modals.
- `<Feature>Table` for the table or headless collection composer; `<Feature>Table<Aspect>` for its sub-pieces (`<Feature>TableMain`, `<Feature>TableColumnDefs`, `<Feature>TableSearch`, `<Feature>TableFilter`, `<Feature>TableVisibility`, `<Feature>TableExport`, `<Feature>TablePagination`, `<Feature>TableActionCell`, `<Feature>TableDelete`, `<Feature>TableResetSelection`) and `<Feature>Card` for card-backed collection rows.
- `<Feature>Skeleton` for skeletons under `components/skeleton/`.
- `No<Thing>Found` / `No<Thing>Selected` / `GenericError` / `NotFound` for empty states under `components/empty-states/`.

### 2.2 — Hooks

- File and function start with **`use`** followed by PascalCase: `useCreateWidget.ts → export function useCreateWidget() { … }`.
- Verb-first for mutations: `useCreate<Resource>`, `useUpdate<Resource>`, `useDelete<Resource>`, `useUpdate<Resource><Aspect>` (e.g. for narrow status patches).
- Read-side patterns:
    - `useGetAll<Resource>` for unscoped lists.
    - `use<Resource>ById` for single-entity fetches.
    - `useUser<Resource>(s)` when scoped to the current user/session/parent id.
    - `useGet<Resource>Stats` for stat aggregations.
- Generic root-level hooks may use `kebab-case` only when they come from shadcn (`use-mobile.ts`); for everything else use `useCamelCase.ts`.

### 2.3 — Helpers (`src/lib/`)

| Category                            | File / function                                                                                            | Export                                            |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------- |
| Stand-alone heavy helper            | **PascalCase.ts** + **PascalCase function** (`ZodSchemaCreator.ts → ZodSchemaCreator`)                     | `default` if only one helper, `named` if multiple |
| Several related small helpers       | **PascalCase.ts** + **PascalCase functions** (`ZodSchemaDefaultValues.ts → ZodSchemaDefaultValues`)        | named                                             |
| Tiny utilities                      | **`utils.ts`** + **camelCase functions** (`cn`, `capitalizeFirstLetter`, `toYMD`, date helpers, etc.)      | named                                             |
| Project-wide enums / discriminators | **`enums.ts`** — `<Plural>Types` (`as const` record) + `<Singular>Type` alias (`WidgetTypes` + `WidgetType`) | named                                             |

### 2.4 — Routes (`src/routes/`)

- TanStack file-system rules:
    - Static segments → **kebab-case** (e.g. `dashboard.tsx`, `signin.tsx`).
    - Dynamic segments → `$param.tsx` (e.g. `$widgetId.tsx`).
    - Folder index → `index.tsx`.
    - Layout / pathless wrapper → `_<name>.tsx` (`_app.tsx`, `_auth.tsx`, `_base.tsx`).
- Route component identifiers are **PascalCase**. When the page name is generic, `RouteComponent` is the accepted default identifier.

### 2.5 — Services (`src/services/<feature>/`)

| File                   | Entity                          | Naming                                                                                                 | Export |
| ---------------------- | ------------------------------- | ------------------------------------------------------------------------------------------------------ | ------ |
| `<Feature>.api.ts`     | Network call functions          | **camelCase** (`getAllWidgets`, `createWidget`, `updateWidget`, `deleteWidget`, `broadcastWidget`)     | named  |
| `<Feature>.api.ts`     | Entity TS type                  | **PascalCase**, singular (`Widget`, `User`, `Order`)                                                   | named  |
| `<Feature>.api.ts`     | Endpoint constant               | `ENDPOINT` (uppercase, file-local; never exported)                                                     | local  |
| `<Feature>.schemas.ts` | Zod schema                      | **PascalCase** + `Schema` (`WidgetSchema`, `WidgetMainSchema`)                                         | named  |
| `<Feature>.schemas.ts` | Inferred TS type                | Schema name + `Type` (`WidgetSchemaType`)                                                              | named  |
| `<Feature>.schemas.ts` | Default form values             | Schema name + `InitData` (`WidgetSchemaInitData`)                                                      | named  |
| `<Feature>.helpers.ts` | Exporters / transformers        | **PascalCase** (`Structure<Feature>ToExport`, `<Feature>CSVExporter`, `<Feature>JSONExporter`)         | named  |
| `<Feature>.helpers.ts` | Plain data prep / status checks | **camelCase** (`prepare<Feature>Data`, `init<Feature>Data`, `get<Feature>Status`)                      | named  |
| `<Feature>` filename   | Service file prefix             | **PascalCase** matching folder feature (`Widgets.api.ts`, `Widgets.schemas.ts`, `Widgets.helpers.ts`)  | —      |

### 2.6 — Translation keys (`src/locales/*.json`)

- All keys are **flat** and **kebab-case**: `sign-in`, `email-is-required`, `placeholder-widget-name`, `something-went-wrong`.
- Suffixes:
    - `-q` → question (`forgot-password-q`).
    - `-dots` → placeholder ending in "…" (`search-widget-dots`).
    - `-description` → field/help description (`widget-name-description`).
    - `-suffix` → trailing inline italicized label (`widgets-suffix`).
- Singular vs plural by context. Reuse keys when the value is identical across contexts. See `§4 — Translation` below for full rules.

### 2.7 — Constants

- Enum-style records live in `src/lib/enums.ts` as **`as const`** records using **SCREAMING_SNAKE_CASE keys** and human-readable string values:
    ```ts
    export const WidgetTypes = { STANDARD: "Standard", PREMIUM: "Premium" } as const;
    export type WidgetType = (typeof WidgetTypes)[keyof typeof WidgetTypes];
    ```
- Sets used for runtime checks live in the same file with **PascalCase** names (e.g. `MustHaveOptions`, `SearchableTypes`, `FilterableTypes`).
- One-off module constants stay at the top of the file using camelCase or UPPER_CASE depending on local precedent. Don't introduce a `src/constants/` folder unless the project already has one.

### 2.8 — Query keys

- Always an **array** starting with a stable, kebab-case feature/resource name. Append entity ids as additional elements.
- Example shapes: `["widgets"]`, `["widget"]`, `["widget-stats"]`, `["widgets", parentId]`, `["widget-children", parentId, childId]`.
- Mutations invalidate **every related** key — list-level, detail-level, and any aggregated stats key.

---

## 3 — Styling (Tailwind v4 + shadcn/ui + RTL)

### 3.1 — Tailwind setup

- **Tailwind CSS 4** via `@tailwindcss/vite`. The single global stylesheet is `src/index.css` and it owns all `@theme`, CSS-variable, and base-layer config. Do not create additional CSS files.
- All styles are **inline utility classes** on JSX. No `className={styles.foo}`, no `styled-components`, no emotion, no CSS-in-JS.
- `prettier-plugin-tailwindcss` orders classes automatically. Don't hand-sort. Configure `tailwindFunctions` (e.g. `["tw"]`) for any tagged-template helpers used as class-name carriers.
- `tw-animate-css` is available for short, named keyframe animations.

### 3.2 — `cn()` helper

Always use `cn()` from `@/lib/utils` for any class composition that involves conditionals or merging:

```tsx
<Button className={cn("text-white", isActive ? "bg-green-700" : "bg-red-700")}>...</Button>
```

`cn` wraps `clsx` + `tailwind-merge`, so it deduplicates conflicting Tailwind utilities. Don't compose with template strings or array `.join(" ")`.

### 3.3 — shadcn/ui (`src/components/ui/`)

- Configured via `components.json`. Conventional values used in this stack:
    - `style: "radix-vega"` (or whichever style the project picks once), `baseColor: "neutral"`, `cssVariables: true`.
    - `iconLibrary: "lucide"`, `rtl: true` when supporting RTL locales.
    - Aliases: `components` → `@/components`, `ui` → `@/components/ui`, `utils` → `@/lib/utils`, `lib` → `@/lib`, `hooks` → `@/hooks`.
- **Never edit files under `src/components/ui/` manually.** They are CLI-managed. Add primitives via `npx shadcn@latest add <name>`.
- ESLint ignores `src/components/ui/**`.
- When customizing a primitive's appearance, do it on the consumer with utility classes — never by editing the primitive.

The most-used primitives in this stack: `Button`, `ButtonGroup`, `Dialog`, `Sheet`, `DropdownMenu`, `Popover`, `Calendar`, `Input`, `InputGroup`, `Textarea`, `Select`, `Switch`, `Checkbox`, `Field` family, `Empty` family, `Sidebar` family, `Tabs`, `Tooltip`, `Avatar`, `Badge`, `Card`, `Command`, `Separator`, `Table`, `Skeleton`, `Sonner`. Prefer composing these over rolling new primitives.

### 3.4 — Icons

- **Lucide React** (`lucide-react`) is the only icon set. Import individual icons by name: `import { Mail, Calendar } from "lucide-react"`.
- Icon size is set with Tailwind: typically `size-4`, `size-5`, `size-8`, `size-10` rather than `width={...} height={...}`.
- Place icons inside `<InputGroupAddon>`, `<SidebarMenuButton>` children, table cells, etc., per local pattern.

### 3.5 — Theming and dark mode

- Theme state lives in `@/contexts/theme-provider`. The provider stores the user's choice in `localStorage` (key conventionally `vite-ui-theme`) and toggles `light`/`dark` classes on `<html>`.
- `defaultTheme="light"` is configured at `__root.tsx`. Don't change without discussion.
- `next-themes` is installed and used by some shadcn primitives; the app-level provider remains `@/contexts/theme-provider`.

### 3.6 — RTL & internationalization-aware styling

- The two locales are English (`en-US`, LTR) and Arabic (`ar-SA`, RTL). `dir` is toggled on `<html>` by `i18n.ts` and propagated through `<DirectionProvider dir={...}>` from shadcn. Do not set `dir` manually inside components except for special cases like read-only LTR strings inside an Arabic page (e.g. phone numbers, URLs — use `dir="ltr"` on the wrapper).
- Use **logical properties** in Tailwind:
    - Margins/padding: `ms-*`, `me-*`, `ps-*`, `pe-*` (start/end), not `ml-*`, `mr-*`, `pl-*`, `pr-*`.
    - Borders: prefer `border-s`, `border-e` when the side matters semantically.
    - Use `rtl:` and `ltr:` variants when a layout needs to flip differently than the default mirroring (e.g. `rtl:translate-x-0` to neutralize a translation).
- Sidebar `side` flips based on language: `side={i18n.language === "en-US" ? "left" : "right"}`. Mirror this when adding new docked panels.

### 3.7 — Layout idioms (recurring patterns)

Common page-level patterns — copy them rather than reinventing:

- Page wrapper: `<div className="flex flex-1 flex-col gap-4 p-4">`.
- Page header row: `<div className="flex flex-wrap items-center justify-between gap-4">` with an `<h1 className="text-2xl font-semibold tracking-tight md:text-3xl lg:text-4xl">`.
- Inline italicized suffix on a heading: `<span className="text-xl font-light italic">{t("<thing>-suffix")}</span>`.
- Stat grid: `<div className="grid grid-cols-1 divide-y rounded-md border md:grid-cols-N md:divide-x md:divide-y-0">` with `<div className="hover:bg-muted space-y-4 px-6 py-4 transition-colors md:border-t lg:border-t-0">`.
- Table wrapper: `<div className="overflow-hidden rounded-md border">`.

### 3.8 — Styling — don't do

- Don't add CSS files, SCSS, or PostCSS plugins.
- Don't write inline `style={{ }}` objects unless interpolating runtime values that Tailwind cannot express.
- Don't introduce a second icon set.
- Don't hard-code `ltr`/`rtl`-specific margins; use logical properties.
- Don't import shadcn primitives from anywhere other than `@/components/ui/...`.

---

## 4 — Translation

These rules define how translations **must** be managed.

### 4.1 — Supported languages

- **English (`en-US`)** and **Arabic (`ar-SA`)** only. No other locales.
- The detection order is `localStorage` only (configured in `src/locales/i18n.ts`). Document `lang` and `dir` attributes are toggled side-effect-style on language change — that infrastructure is in place; never reimplement it.
- Arabic forces `dir="rtl"`; English forces `dir="ltr"`. RTL handling cascades through the shadcn `<DirectionProvider>` mounted at `__root.tsx` — see `§3.6` above.

### 4.2 — File locations

| Language  | File                  |
| --------- | --------------------- |
| English   | `src/locales/en.json` |
| Arabic    | `src/locales/ar.json` |
| i18n init | `src/locales/i18n.ts` |

### 4.3 — Structure: flat object only

- All translation keys live in a **single flat object**.
- **No nesting** — no `{ "auth": { "signIn": "..." } }`.
- Use a single level of key-value pairs.

```json
// ✅ GOOD
{
  "sign-in": "Sign In",
  "forgot-password": "Forgot Password",
  "email-is-required": "Email is required."
}

// ❌ BAD
{
  "auth": {
    "signIn": "Sign In",
    "forgotPassword": "Forgot Password"
  }
}
```

### 4.4 — Key naming: kebab-case

- All keys **must** use `kebab-case`.
- Words are lowercase and separated by hyphens.

```json
// ✅ GOOD
"confirm-password"
"email-is-required"
"select-date-range"

// ❌ BAD
"confirmPassword"
"email_is_required"
"SelectDateRange"
```

### 4.5 — Naming: clear and context-relevant

- Keys should be **descriptive** and reflect where they are used.
- Prefer specificity over brevity when it clarifies context.
- Use suffixes for variants when needed (e.g. `-q` for question, `-dots` for placeholder).

| Context         | Example Key          | Example Value         |
| --------------- | -------------------- | --------------------- |
| Button label    | `sign-in`            | "Sign In"             |
| Form validation | `email-is-required`  | "Email is required."  |
| Placeholder     | `search-widget-dots` | "Search widget..."    |
| Question        | `forgot-password-q`  | "Forgot Password?"    |

#### 4.5.1 — Breadcrumb & route segment keys

- **Use segment names directly as keys** — No `breadcrumb-` or other prefix.
- The breadcrumb hook returns segment names. These names **are** the translation keys.
- Use `t(el.name)` and `t(active.name)` directly in the component.

```tsx
// ✅ GOOD
<Link to={el.path}>{t(el.name)}</Link>
<BreadcrumbPage>{t(active.name)}</BreadcrumbPage>

// ❌ BAD
<Link to={el.path}>{t(`breadcrumb-${el.name}`)}</Link>
```

#### 4.5.2 — Singular vs plural

- Use **singular** when the context is about one item (selecting, searching, displaying one).
- Use **plural** when the context is about many items (list heading, "no items found" for an empty list).

| Context                          | Key                  | Example Value      |
| -------------------------------- | -------------------- | ------------------ |
| Placeholder: select one widget   | `select-widget-dots` | "Select widget..." |
| Placeholder: search one widget   | `search-widget-dots` | "Search widget..." |
| Empty state: no widget selected  | `no-widget-found`    | "No widget found"  |
| Empty state: no widgets in list  | `no-widgets-found`   | "No widgets found" |
| List/group heading               | `widgets`            | "Widgets"          |

#### 4.5.3 — Reuse keys across contexts

- When the same word appears in multiple places (nav label, breadcrumb, tooltip) with the **same value**, use a **single key**.
- Do not create context-specific keys (e.g. `breadcrumb-dashboard` vs `dashboard`) when the translation value is identical.

```json
// ✅ GOOD — one key for "Dashboard" everywhere
"dashboard": "Dashboard"

// ❌ BAD — redundant keys for same value
"dashboard": "Dashboard",
"breadcrumb-dashboard": "Dashboard"
```

#### 4.5.4 — Do not internationalise `alt` text

- **Do not** use `t()` or locale keys for HTML **`alt`** attributes on images (or other elements that use `alt`).
- Keep `alt` as a **static string** in the component (concise English is fine), or use **`alt=""`** when the image is purely decorative and accessibility guidance allows it.
- **Do not** add translation keys whose sole purpose is `alt` copy.

```tsx
// ✅ GOOD
<img src={url} alt="Avatar" />

// ❌ BAD
<img src={url} alt={t("avatar-alt")} />
```

#### 4.5.5 — Zod errors are translation keys

- Every Zod `error` value is a **kebab-case translation key**, never literal English:
    ```ts
    z.string({ error: "name-is-required" }).trim().min(1, { error: "name-can-not-be-empty" });
    z.email({ error: "email-is-required" });
    ```
- The matching key must exist in **both** `en.json` and `ar.json`. The mutation `onError` already wraps the message in `t(...)`, so the i18n key surfaces correctly in toasts.
- Reuse generic keys across schemas (`name-is-required`, `name-can-not-be-empty`, `email-is-required`, `phone-is-required`, `this-field-is-required`, `this-field-can-not-be-empty`, `invalid-field-entry`, `something-went-wrong-please-try-again-later`) instead of inventing per-field copies.

#### 4.5.6 — Server response strings are also translation keys

- The backend conventionally returns plain-text codes that are themselves kebab-case i18n keys. The mutation hook calls `t(response || "<fallback-key>")`, so any new server-side string must also be added to **both** `en.json` and `ar.json`.

### 4.6 — Adding or modifying keys

- Add or modify keys in **both** `en.json` and `ar.json` in the same change.
- Keep keys **alphabetically sorted** for easier maintenance.
- Never leave a key in one file without its counterpart in the other.
- No duplicate keys allowed. Always make sure of that.

### 4.7 — Do not modify

- **Never modify** `src/components/ui/` — shadcn/ui components are managed by the CLI; do not edit them manually.
