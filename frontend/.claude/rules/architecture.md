# Architecture

How the repo is structured, how the toolchain is configured, how routing and auth are wired. Read this before adding folders, routes, layouts, or anything that touches Clerk or the build.

---

## 1 — Project structure (`src/`)

How files and folders **must** be organised.

### 1.0 — Global

- **Language** — TypeScript only (`.ts` / `.tsx`).
- **Imports** — Use the absolute alias (`@/* → ./src/*`) defined in `tsconfig.app.json` through explicit `paths`, without `compilerOptions.baseUrl`. Never traverse above `src/`. Inside `src/`, prefer `@/...` over relative paths for any cross-folder import.
- **File naming**
    - Components → `PascalCase.tsx` (one component per file).
    - Hooks → `use<PascalCase>.ts` (e.g. `useCreateWidget.ts`, `useGetAllWidgets.ts`). One hook per file.
    - Contexts → `kebab-case-provider.tsx` (e.g. `theme-provider.tsx`).
    - Routes → TanStack file-system rules (kebab-case static segments, `$param.tsx`, `_layout.tsx`).
    - Services → `<Feature>.<area>.ts` with **PascalCase** prefix matching the folder feature (e.g. `Widgets.api.ts`, `Widgets.schemas.ts`, `Widgets.helpers.ts`). PascalCase is the default; lowercase prefixes only appear in early-stage / single-call files and shouldn't be a model for new work.
    - Lib helpers → `PascalCase.ts` for stand-alone helpers; add to `lib/utils.ts` for small utilities.
- **Exports**
    - Components → **default export** at the bottom of the file.
    - Hooks, services, schemas, helpers, contexts → **named exports**.
- **Styles** — Tailwind CSS classes inline only. No CSS files. Global styles live exclusively in `src/index.css`.

### 1.1 — `src/components/`

- **One folder per feature.** Feature folders mirror service/hook feature folders (1:1 with backend resources).
- UI must not import across feature folders directly. If two features need the same component, lift it into `general/`.
- **`_app/`** — global layout chrome (header, sidebar, language/theme switchers, global menus). Components here are conventionally prefixed `App` (`AppHeader`, `AppSidebar`, `AppLanguageSwitcher`, etc.).
- **`general/`** — purely generic, reusable UI that is feature-agnostic (`CopyInput`, `ProfilePic`, `SpinnerPage`, etc.).
- **`empty-states/`** — full-page or section empty/error fallbacks built on the shadcn `<Empty>` primitive (`No<Thing>Found`, `No<Thing>Selected`, `GenericError`, `NotFound`, `ErrorComponent`).
- **`skeleton/`** — page-/section-level skeletons matching real layouts (`<Feature>Skeleton`, `TableSkeleton`).
- **`ui/`** — shadcn/ui primitives. **CLI-managed; never edited manually.** ESLint ignores this directory.
- **`form-fields/`** — only created when the project needs dynamic Zod-driven inputs (forms whose fields come from runtime metadata). Skip this folder if forms are all statically defined.
- **`landing/`** — only when the project ships a marketing/landing surface used from a public layout route. Skip otherwise.

### 1.2 — `src/contexts/`

- One file per context, named `<thing>-provider.tsx` (kebab-case).
- Each file exposes:
    - `<Thing>Provider` (component, named export).
    - `use<Thing>()` hook (named export). The hook throws if the context is missing/undefined.
- Hold only **cross-feature** state (auth, theme, currently-selected scope id, etc.). Feature-local state stays in the feature.

### 1.3 — `src/hooks/`

- **A folder = a feature**, mirroring `src/services/` and `src/components/`.
- Files inside follow `use<Action><Resource>.ts`:
    - Reads → `useGetAll<Resource>`, `use<Resource>ById`, `useUser<Resource>` (when scoped to the current user/session), `useGet<Resource>Stats` (for aggregates).
    - Writes → `useCreate<Resource>`, `useUpdate<Resource>`, `useDelete<Resource>`, `useUpdate<Resource><Aspect>` for narrow patches.
- Non-feature, generic hooks live at the **root** of `src/hooks/` (e.g. `useBreadCrumbs.ts`, `useIdempotentMutation.ts`, plus shadcn-generated `use-mobile.ts`).
- Each hook wraps a single TanStack Query `useQuery` or `useMutation`. See `feature-workflow.md` for the full pattern.

### 1.4 — `src/lib/`

- Flat collection of **pure, framework-agnostic** helpers.
- Large helpers → their own file, **PascalCase**, e.g. `ZodSchemaCreator.ts`. Default-export when the file exposes exactly one helper, named-export when more than one.
- Tiny utilities → add to `utils.ts` as **camelCase named exports** (`cn`, `capitalizeFirstLetter`, date utilities, etc.).
- `enums.ts` holds **all** project-wide enum-like objects (`as const` records) and their derived `type X = (typeof Xs)[keyof typeof Xs]` aliases. New enums go here, not in feature folders.

### 1.5 — `src/locales/`

- Exactly two flat JSON files: `en.json` (English, `en-US`) and `ar.json` (Arabic, `ar-SA`).
- Plus `i18n.ts` for the i18next initialization (LanguageDetector, fallback, `<html lang>` / `<html dir>` side-effects on language change).
- Keys are flat (no nesting) and `kebab-case`. See `conventions.md`.

### 1.6 — `src/routes/`

See `§3 — Routing` below for the full route file layout and conventions.

### 1.7 — `src/services/`

- `API.ts` is the **only** place axios is imported directly. It exports a default object with typed `GET`/`POST`/`PATCH`/`PUT`/`DELETE` methods plus the `setClerkGetToken` initializer.
- **Folder per feature.** Folder name is lowercase (`widgets/`, `users/`, `orders/`).
- Inside each folder, file naming is **`<Feature>.<area>.ts`** with PascalCase prefix:
    - `<Feature>.api.ts` — async network calls + the entity `type` declaration + a single top-of-file `ENDPOINT` constant. Function names are camelCase. Helper inner functions (e.g. for presigned-URL uploads) stay at the bottom of the file as non-exported helpers.
    - `<Feature>.schemas.ts` — Zod schemas (`<Feature>Schema`), inferred types (`<Feature>SchemaType`), and an init-data const (`<Feature>SchemaInitData`). All named exports.
    - `<Feature>.helpers.ts` — pure feature-specific utilities. Function names follow local precedent (PascalCase for exporters/transformers, camelCase for plain data prep).
- Larger features split schemas into multiple files (`<Feature>.main.schemas.ts`, `<Feature>.<aspect>.schemas.ts`, plus an aggregator `<Feature>.schemas.ts`). Apply this when a single schema file would exceed ~150 lines.

### 1.8 — `src/index.css` and `src/main.tsx`

- `src/index.css` is the only place global Tailwind directives, CSS variables, and base font / `@theme` config live.
- `src/main.tsx` mounts the app, creates the `QueryClient` and `Router`, wraps with `<ClerkProvider>`, and re-renders on `i18n.languageChanged` so Clerk's localization stays in sync.

### 1.9 — Do not modify

- **Never modify** `src/components/ui/` — shadcn/ui components are managed by the CLI; do not edit them manually.
- **Never modify** `src/routeTree.gen.ts` — auto-generated by the TanStack Router Vite plugin.

---

## 2 — Tooling & environment

The toolchain is fixed. New projects spun up to match this house style should reproduce these settings exactly unless there is an explicit reason to deviate.

### 2.1 — Build & dev

- **Vite 8** with `@vitejs/plugin-react`, `@tailwindcss/vite`, and `@tanstack/router-plugin/vite` (configured with `target: "react"`, `autoCodeSplitting: true`).
- `vite.config.ts` declares the alias `@ → ./src` via `path.resolve`. The same alias is mirrored in `tsconfig.app.json` `paths` without `compilerOptions.baseUrl`.
- Scripts in `package.json`:
    - `dev` → `vite`
    - `build` → `tsc -b && vite build`
    - `lint` → `eslint .`
    - `preview` → `vite preview`

### 2.2 — TypeScript (`tsconfig.app.json`)

```jsonc
{
    "compilerOptions": {
        "target": "ES2023",
        "lib": ["ES2023", "DOM", "DOM.Iterable"],
        "module": "ESNext",
        "moduleResolution": "bundler",
        "allowImportingTsExtensions": true,
        "moduleDetection": "force",
        "noEmit": true,
        "jsx": "react-jsx",
        "strict": true,
        "noUnusedLocals": true,
        "noUnusedParameters": true,
        "erasableSyntaxOnly": true,
        "noFallthroughCasesInSwitch": true,
        "noUncheckedSideEffectImports": true,
        "paths": { "@/*": ["./src/*"] }
    },
    "include": ["src"]
}
```

Notes:

- `verbatimModuleSyntax` is intentionally **disabled** because shadcn primitives currently break under it.
- Do not relax `strict` or any `noUnused*` flag.
- Do not add `compilerOptions.baseUrl`; TypeScript 6 deprecates it and TypeScript 7 removes it. Keep alias targets explicit in `paths`.

### 2.3 — ESLint (`eslint.config.js`)

- Flat config using `@eslint/js`, `typescript-eslint`, `eslint-plugin-react-hooks`, `eslint-plugin-react-refresh`.
- Globally ignores `dist` and `src/components/ui/**`.
- Custom rules:
    - `@typescript-eslint/no-unused-vars` with `varsIgnorePattern`, `argsIgnorePattern`, `caughtErrorsIgnorePattern` all set to `^_`.
    - `react-refresh/only-export-components` is **off**.
- Don't introduce new ESLint plugins without discussion.

### 2.4 — Prettier (`.prettierrc`)

```json
{
    "printWidth": 360,
    "tabWidth": 4,
    "useTabs": false,
    "semi": true,
    "singleQuote": false,
    "trailingComma": "none",
    "bracketSpacing": true,
    "arrowParens": "always",
    "plugins": ["prettier-plugin-tailwindcss"],
    "tailwindFunctions": ["tw"]
}
```

- The very wide `printWidth: 360` is intentional. Don't artificially line-wrap to look "narrow".
- Don't hand-sort Tailwind classes; the plugin does it.

### 2.5 — Environment variables

- All env vars are **client-exposed** (Vite `VITE_*` prefix). Defined in `.env`.
- Conventional names used in this stack (rename per project as needed):
    - `VITE_ENV` — `"PROD"` toggles production endpoints in service files.
    - `VITE_BE_ENDPOINT` — base API URL (a port suffix is conventionally appended in non-prod).
    - `VITE_CLERK_PUBLISHABLE_KEY` — Clerk publishable key.
- Read via `import.meta.env.VITE_*`. Don't introduce a runtime config layer unless explicitly required.

### 2.6 — Aliases (across all configs)

| Place                      | Mapping                                                                                                      |
| -------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `tsconfig.app.json`        | `paths`: `@/* → ./src/*` with no `compilerOptions.baseUrl`                                                   |
| `vite.config.ts`           | `@` → `path.resolve(__dirname, "./src")`                                                                     |
| `components.json` (shadcn) | `components → @/components`, `ui → @/components/ui`, `utils → @/lib/utils`, `lib → @/lib`, `hooks → @/hooks` |

If you change one, change all three.

### 2.7 — Package manager

- **npm** is the package manager — always. Lockfile is `package-lock.json`.
- Use `npm install` / `npm install <pkg>` / `npm run <script>`. Never `pnpm`, `yarn`, or `bun`.
- For shadcn primitives, use `npx shadcn@latest add <name>`.

### 2.8 — Don't do

- Don't add a `postcss.config.js` — Tailwind v4 doesn't need it (it runs through `@tailwindcss/vite`).
- Don't add a `tailwind.config.js` — v4 reads everything from `@theme` directives in `src/index.css`.
- Don't introduce Prettier overrides per-file or per-folder. The repo has one config; keep it that way.
- Don't add a runtime feature-flag system, an analytics SDK, or telemetry without discussion.

---

## 3 — Routing (TanStack Router, file-based)

The app uses **`@tanstack/react-router`** with the Vite plugin in **`autoCodeSplitting: true`** mode. The route tree is generated to `src/routeTree.gen.ts` and imported by `main.tsx`.

### 3.1 — Router configuration (`src/main.tsx`)

```ts
const router = createRouter({
    routeTree,
    defaultPreload: "render",
    scrollRestoration: true,
    notFoundMode: "root",
    defaultErrorComponent: ErrorComponent,
    context: { queryClient }
});
```

Rules:

- `defaultPreload: "render"` is intentional — route components prefetch on render. Don't switch this to `"intent"` without discussion.
- `notFoundMode: "root"` routes unknown paths to the root's `notFoundComponent` (the shared `<NotFound />` empty-state).
- `defaultErrorComponent` is `ErrorComponent` from `src/components/empty-states/`. Don't add per-route error components unless the page genuinely needs special handling.
- The router context carries the shared `queryClient`, declared at the root via `createRootRouteWithContext<{ queryClient: QueryClient }>()`.

### 3.2 — File layout

The conventional skeleton (concrete page names depend on the project):

```
src/routes/
    __root.tsx                # createRootRouteWithContext — providers + Clerk getToken wiring
    _app.tsx                  # Authenticated layout (sidebar + header + outlet, gated by <Protect>)
    _app/
        <feature>.tsx         # one file per top-level authenticated page
        <feature>/
            index.tsx
            $id.tsx           # dynamic detail
    _auth.tsx                 # Unauthenticated auth shell (sign-in chrome)
    _auth/
        signin.tsx
    _base.tsx                 # Unauthenticated public/marketing shell (omit if not needed)
    _base/
        index.tsx             # Landing page
```

Conventions:

- Static segments are **kebab-case** (e.g. `dashboard.tsx`, `signin.tsx`).
- Dynamic segments use the `$param.tsx` form (e.g. `$widgetId.tsx`).
- Folder index files are `index.tsx`.
- Pathless layout wrappers are `_<name>.tsx` (the three established ones — `_app`, `_auth`, `_base` — cover every gating need).

### 3.3 — Route file template

```tsx
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/_app/widgets")({
    component: RouteComponent
});

function RouteComponent() {
    const { /* hooks */ } = useUserData(/* ... */);

    if (!scopeId) return <NoScopeSelected />;
    if (loading) return <WidgetsSkeleton />;
    if (isError) return <GenericError />;

    return ( /* ... */ );
}
```

- The exported variable **must** be named `Route` (TanStack Router requirement).
- The component identifier is either the page name (`Dashboard`, `SignIn`) or the generic `RouteComponent`. Pick whichever matches local precedent in the same folder.
- Read URL params with `Route.useParams()`:
    ```tsx
    const { widgetId } = Route.useParams();
    ```

### 3.4 — Linking and navigation

- Use `<Link to="/...">` from `@tanstack/react-router` for in-app navigation. Never use `<a href>` for internal links.
- Programmatic navigation uses `<Navigate to="/..." replace />` (rendered) or `useNavigate()` (event handlers). Prefer rendered `<Navigate>` for redirects in layouts.
- For "current path" reads, use `useLocation({ select: (loc) => loc.pathname })`.

### 3.5 — Breadcrumbs

- The app-wide breadcrumb hook (typically `src/hooks/useBreadCrumbs.ts`) splits the pathname into segments and feeds `t(segment)` directly. Translation keys are the segment strings themselves. See `conventions.md §4.5.1` (breadcrumb & route segment keys).
- When a page should display a different label than its segment, special-case it inside `useBreadcrumbs` rather than forking the hook.

### 3.6 — Devtools

- `@tanstack/router-devtools` and `@tanstack/react-query-devtools` are imported but commented out in `__root.tsx`. Toggle via uncommenting in development; never commit them un-commented to main.

### 3.7 — Don't do

- Don't edit `src/routeTree.gen.ts` — it's regenerated by the Vite plugin on file changes.
- Don't add a `pages/` folder or use any non-TanStack routing solution.
- Don't gate pages with manual `useEffect` redirects when one of the three pathless layouts (`_app`, `_auth`, `_base`) already does it.
- Don't pass cross-route data via `localStorage` when route search params or context would do.

---

## 4 — Authentication (Clerk)

Authentication is handled exclusively by **Clerk** (`@clerk/clerk-react`). There is no custom auth state, no JWT handling, no refresh-token logic.

### 4.1 — Provider wiring (do not touch unless deliberately changing the auth model)

`src/main.tsx` wraps the app with `<ClerkProvider>` **inside** `<QueryClientProvider>` and **outside** `<RouterProvider>`:

```tsx
<QueryClientProvider client={queryClient}>
    <ClerkProvider
        publishableKey={import.meta.env.VITE_CLERK_PUBLISHABLE_KEY}
        appearance={{ theme: shadcn }}
        localization={i18n.language === "en-US" ? enUS : arSA}
    >
        <RouterProvider router={router} />
    </ClerkProvider>
</QueryClientProvider>
```

- The Clerk theme is `shadcn` from `@clerk/themes`.
- Localization comes from `@clerk/localizations` (`enUS` / `arSA`) and is selected from the current i18n language: `localization={i18n.language === "en-US" ? enUS : arSA}`.
- The whole app re-renders on `i18n.languageChanged` so that Clerk picks up the new localization. **Don't break this `mainRender()` re-binding.**

### 4.2 — Token forwarding to Axios

`src/services/API.ts` declares a single Axios request interceptor that calls a lazily-set `clerkGetToken()`:

```ts
let clerkGetToken: GetTokenFunctionType | null = null;
export function setClerkGetToken(getToken: GetTokenFunctionType) {
    clerkGetToken = getToken;
}

axios.interceptors.request.use(async (config) => {
    if (clerkGetToken) {
        const token = await clerkGetToken();
        if (token) config.headers.Authorization = `Bearer ${token}`;
    }
    // Strip auth for known third-party endpoints (e.g. presigned S3 URLs, public lambdas):
    if (config.url?.includes("amazonaws")) config.headers.Authorization = undefined;
    if (config.url?.includes("lambda")) config.withCredentials = false;
    return config;
});
```

`__root.tsx` runs this once:

```tsx
const { getToken } = useAuth();
useEffect(() => {
    setClerkGetToken(getToken);
}, [getToken]);
```

Rules:

- Never import `axios` directly into a component or hook. Always go through `@/services/API`.
- Don't add new auth interceptors. If a route bypasses bearer auth, special-case it in the same interceptor file by inspecting `config.url`.

### 4.3 — Route-level gating

Route gating is done via three pathless layout routes — pick the right one for the page you're adding:

| Layout      | Purpose                         | Behavior                                                                                                                                    |
| ----------- | ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `_app.tsx`  | Authenticated app shell         | If `!isLoaded` → `<SpinnerPage />`. Else `<Protect fallback={<Navigate to="/signin" replace />}>`.                                          |
| `_auth.tsx` | Unauthenticated auth screens    | If `!isLoaded` → `<SpinnerPage />`. If signed in → `<Navigate to="/<post-login-route>" replace />`. Else render `<SignedOut>{children}</SignedOut>`. |
| `_base.tsx` | Unauthenticated public pages    | Same gating as `_auth.tsx` but renders the public/marketing layout. Skip if the project has no public surface.                              |

Rules:

- Use `useAuth().isLoaded` and the `<Protect>` / `<SignedOut>` components from `@clerk/clerk-react`. Don't roll your own auth-state booleans.
- The Clerk-managed sign-in screen is mounted at `/signin` via `<SignIn />` from `@clerk/clerk-react`. Don't build a custom sign-in form.
- The header avatar / sign-out menu is `<UserButton />` from `@clerk/clerk-react`, placed in `AppHeader.tsx`.

### 4.4 — Reading role and permissions (per-scope access endpoint)

User role and per-scope permissions are fetched from the backend via `GET /access/:scopeId` — never from Clerk `publicMetadata`. The response is a discriminated union; the exact shape lives in `src/services/access/Access.api.ts`:

```ts
type Permission = { /* per-feature booleans */ };
type AdminAccess = { eventId: string; role: typeof RoleTypes.ADMIN; hasAccess: true };
type StaffAccess = { eventId: string; role: typeof RoleTypes.STAFF; hasAccess: true; permissions: Permission };
type Access = AdminAccess | StaffAccess;
type AccessAccount = { clerkId: string; email: string | null; name: string | null; imageUrl: string | null; permissions: Permission };
```

Read it via the `useGetAccess` hook:

```tsx
const { selectedEvent } = useSelectedEvent();
const { access } = useGetAccess(selectedEvent);

const isAdmin = access?.role === RoleTypes.ADMIN;
const permissions = access?.role === RoleTypes.STAFF ? access.permissions : null;
const showFeatureNav = isAdmin || permissions?.<feature>;
```

- `RoleTypes` lives in `src/lib/enums.ts`. The per-scope `Permission` and `Access` types live in `src/services/access/Access.api.ts`.
- Access grant management uses `GET /access/grants/:eventId`, `POST /access/grants/:eventId`, `PATCH /access/grants/:eventId/:clerkId`, and `DELETE /access/grants/:eventId/:clerkId`. Grant rows are `AccessAccount` records keyed by `clerkId`, not `_id`.
- The hook is keyed `["access", scopeId]` with `enabled: !!scopeId` and `retry: false`. A 401/403 leaves `access` undefined, which fails closed (no nav, no admin controls).
- Always derive UI gating from `role === RoleTypes.ADMIN` **OR** the per-scope permission boolean.
- Admin-only surfaces such as the access nav item render only when `access?.role === RoleTypes.ADMIN`. Direct route access still relies on backend authorization and renders the normal route error state if the access query fails.
- Capabilities not scoped to a single event (e.g. "create event") cannot be gated by this endpoint — render them for any signed-in user and let the backend enforce.
- Don't trust client-side gates for data security — they're UX hints only. The backend enforces real authorization.

### 4.5 — Sign-out and post-auth redirect

- Sign-out happens via `<UserButton />` (Clerk-managed).
- Post-sign-in redirect is handled by `_auth.tsx` checking `isSignedIn` and `<Navigate to="/<post-login-route>" replace />`.
- Don't add manual `signOut()` calls or `router.navigate` after auth events.

### 4.6 — Don't do

- Don't store auth tokens in `localStorage`, `sessionStorage`, or React state.
- Don't add a custom `axios.create()` instance with its own interceptor.
- Don't read `getToken()` from inside individual API call functions; the root-level interceptor already handles it.
- Don't create new pathless layouts (`_*.tsx`) for auth state. The three established ones cover every case.
