# Architecture

How the interactive app is structured, how the toolchain is configured, and how routing hosts touch-first screen flows. Read this before adding folders, routes, providers, or generated artifacts.

---

## 1 - Project structure (`src/`)

How files and folders must be organized.

### 1.0 - Global

- **Language** - TypeScript only (`.ts` / `.tsx`). No JavaScript source files.
- **Imports** - Use the absolute alias (`@/* -> ./src/*`) for cross-folder source imports. Never traverse above `src/`.
- **File naming**
    - Components -> `PascalCase.tsx` (one component per file).
    - Visitor screens -> `*Screen.tsx` (`WelcomeScreen.tsx`, `ScanBadgeScreen.tsx`, `AttractScreen.tsx`).
    - Hooks -> `use<PascalCase>.ts` (one hook per file).
    - Contexts -> `kebab-case-provider.tsx` (`realtime-provider.tsx`).
    - Routes -> TanStack file-system rules (`kebab-case.tsx`, `$param.tsx`, `_layout.tsx`).
    - Services -> `<Feature>.<area>.ts` with a PascalCase prefix matching the feature folder (`Voting.api.ts`, `Voting.schemas.ts`, `Voting.helpers.ts`). Existing lowercase prefixes may remain, but do not model new work on them.
    - Realtime state schemas -> `<Name>.schema.ts` for one schema, `<Name>.schemas.ts` for related schemas.
- **Exports**
    - Components -> default export at the bottom of the file.
    - Hooks, services, schemas, helpers, contexts -> named exports.
- **Styles** - Tailwind CSS utility classes inline. Global CSS belongs only in `src/index.css`.

### 1.1 - `src/components/`

- **One folder per feature or visitor flow.** Feature folders mirror `src/hooks/` and `src/services/` where possible.
- Screens are the user-visible steps in an exhibit flow and use the `*Screen.tsx` suffix.
- Screens receive navigation callbacks (`onNext`, `onBack`, `onReset`) and data callbacks through props. Do not couple reusable screens directly to the router.
- UI must not import across feature folders directly. If two features need the same component, lift it into `general/`.
- **`_app/`** - app shell, global layout wrappers, root provider wrappers, and app-level runtime UI.
- **`general/`** - feature-agnostic reusable UI.
- **`ui/`** - shadcn/ui primitives. CLI-managed; never edited manually.
- Create specialized folders (`overlays/`, `screensaver/`, `calibration/`) only when the project already has that concept or the task explicitly introduces it.

### 1.2 - `src/contexts/`

- One file per cross-feature provider, named `<thing>-provider.tsx`.
- Each file exposes:
    - `<Thing>Provider` as a named export.
    - `use<Thing>()` as a named export. The hook throws if used outside the provider.
- Keep contexts for cross-feature state only: realtime state, language, app shell state, kiosk/session runtime state.
- Feature-local screen state stays in the feature flow component or a local reducer.

### 1.3 - `src/hooks/`

- **A folder = a feature**, mirroring `src/services/` and `src/components/`.
- Query/mutation hook files follow `use<Action><Resource>.ts`:
    - Reads -> `useGet<Resource>`, `useGetAll<Resource>`, `use<Resource>ById`, `use<Resource>Stats`.
    - Writes -> `useCreate<Resource>`, `useUpdate<Resource>`, `useDelete<Resource>`, `useSubmit<Resource>`.
- Runtime hooks that are intentionally app-wide live at the root:
    - `useKioskMode.ts`
    - `useAutoSubmitForm.ts`
    - `useIdleReset.ts` or equivalent only when the app implements idle/attract mode.
- Each server-state hook wraps a single TanStack Query `useQuery` or `useMutation`.

### 1.4 - `src/lib/`

- Flat collection of pure helpers and constants.
- Large helpers -> their own `PascalCase.ts` file.
- Tiny utilities -> `utils.ts` as camelCase named exports.
- Do not import React, browser globals, sockets, or Axios into `src/lib/` unless the helper is explicitly browser-bound and local precedent already allows it.

### 1.5 - `src/locales/`

- Flat JSON files per supported locale.
- English and Arabic are the baseline (`en.json`, `ar.json`).
- Additional locale files are allowed for international events, but they must follow the same key rules.
- Locale setup (`i18n.ts`) owns language initialization and `<html lang>` / `<html dir>` side effects.

### 1.6 - `src/routes/`

- TanStack Router file-system routing.
- Routes should be shallow. Exhibit flows usually need one route per flow or mode, not deeply nested navigation.
- Route files assemble providers, layouts, and flow containers. They should not hold the full implementation of every screen.
- Route params use `$param.tsx`; layouts use `_layout.tsx`.
- Use TanStack navigation APIs for links and redirects. Do not use `window.location` for internal app navigation.

### 1.7 - `src/services/`

- `API.ts` is the only Axios entrypoint. Components and hooks do not import Axios directly.
- `WS.ts` is the only Socket.IO client singleton. No second socket instance elsewhere.
- **Folder per feature.** Folder names are lowercase (`voting/`, `registration/`, `map/`).
- Inside each feature folder:
    - `<Feature>.api.ts` - async HTTP calls, endpoint constants, API response types.
    - `<Feature>.schemas.ts` - Zod schemas, inferred types, init/default data.
    - `<Feature>.helpers.ts` - pure feature-specific utilities.
- `services/state/` holds realtime state schemas and default values. These are runtime contracts and should change deliberately.

### 1.8 - `src/index.css` and `src/main.tsx`

- `src/index.css` is the only place for Tailwind directives, theme tokens, global font setup, and kiosk/touch CSS lockdown.
- `src/main.tsx` mounts the app, creates query/router providers, and wires top-level providers.
- App-wide browser listeners belong in root providers or dedicated hooks called once at the app root, not scattered across screens.

### 1.9 - Do not modify

- **Never modify** `src/components/ui/` manually. Use the shadcn CLI.
- **Never modify** `src/routeTree.gen.ts`. It is generated by TanStack Router tooling.
- Do not edit build output (`dist/`, `build/`) or generated cache files.

---

## 2 - Tooling and environment

The toolchain is fixed unless the user explicitly asks to change it.

### 2.1 - Build and dev

- Vite with React, Tailwind CSS 4, and TanStack Router tooling.
- Package manager is npm when `package-lock.json` is present.
- Discover scripts from `package.json`. Expected scripts:
    - `dev` -> Vite dev server
    - `build` -> TypeScript and production build
    - `lint` -> ESLint
    - `preview` -> Vite preview

### 2.2 - TypeScript

- Strict mode is expected.
- Prefer `type` over `interface` for app-owned object shapes.
- Use `import type` for type-only imports.
- Avoid `any`. If a payload is unknown at runtime, validate it with Zod at the boundary.
- Do not silence TypeScript errors with casts unless the cast documents a real boundary.

### 2.3 - Environment variables

- Client-exposed variables use the `VITE_` prefix.
- Centralize URL construction in `API.ts` and `WS.ts`.
- Do not read env vars directly from components.
- Keep deployment-specific values out of source files unless the project intentionally ships a kiosk launcher template.

### 2.4 - Aliases

- `@` maps to `./src`.
- Mirror the alias in Vite and TypeScript config.
- Do not rely on `compilerOptions.baseUrl` as a hidden import root unless the existing app already does.

### 2.5 - Do not add by default

- Redux, Zustand, MobX, or another state library.
- A second router.
- A second HTTP client.
- A second socket library.
- CSS modules, styled-components, emotion, or custom CSS files for component styling.
- A new UI kit or icon set.

---

## 3 - Routing and screen flow architecture

### 3.1 - Route responsibility

- A route owns URL-level concerns: loading route params, selecting the flow, wrapping providers, and rendering the flow container.
- A route should not become a large screen implementation file.
- For visitor journeys, create a feature flow component under `src/components/<feature>/` and let the route render that flow.

### 3.2 - Screen responsibility

- A screen represents one clear visitor step.
- Screen props define explicit inputs and callbacks.
- Screens should be easy to reset by their parent flow.
- Avoid long-lived side effects inside screens. Put app-wide listeners in providers/hooks and put feature side effects in hooks.

### 3.3 - Navigation between screens

- Use local flow state for wizard-style exhibit steps.
- Use router navigation only when the URL meaningfully changes.
- Prefer `onNext`, `onBack`, `onReset`, `onComplete` callbacks over importing router APIs into every screen.

### 3.4 - Multi-display and operator modes

- If a project has operator, attract, or audience display routes, keep them explicit in route names and feature folders.
- Do not overload one route with hidden mode switches unless realtime state or deployment constraints require it.
- Document assumptions in the feature flow or launcher template, not in scattered comments.
