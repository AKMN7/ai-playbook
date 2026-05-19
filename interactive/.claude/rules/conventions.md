# Conventions

How interactive code looks: TypeScript style, React components, naming, touch-first UI, Tailwind/RTL styling, translation, and verification. Read this whenever writing or renaming anything.

---

## 1 - Coding style charter

### 1.1 - Consistency over cleverness

- The single highest priority is consistency with existing code, especially files in the same feature folder.
- Before editing, read nearby files and mirror structure, naming, exports, imports, hook/service split, Tailwind class style, spacing, comment density, and error handling.
- If written rules and local precedent disagree, prefer the written rule for new work and preserve local behavior for narrow edits.
- Do not introduce a parallel style next to an existing one.

### 1.2 - Stack assumptions

- Vite + React 19 + TypeScript strict.
- TanStack Router for routes.
- TanStack Query for HTTP server state.
- React Hook Form + Zod for schema-first forms.
- Tailwind CSS 4 inline utilities only.
- shadcn/ui primitives in `src/components/ui/` - CLI-managed, never edited manually.
- i18next + react-i18next for public copy.
- Socket.IO only through `src/services/WS.ts`.
- Axios only through `src/services/API.ts`.
- sonner for toasts.
- lucide-react for icons.

Ask before adding a dependency or replacing any of these choices.

### 1.3 - TypeScript

- TypeScript only. No `.js` or `.jsx` source files.
- Prefer `type` over `interface` for app-owned object shapes.
- Use `import type` for type-only imports.
- Avoid `any`; validate unknown runtime payloads with Zod.
- Prefix intentionally unused values with `_` when lint config allows it.
- Do not use `ts-ignore` / `ts-expect-error` without a one-line reason.
- Do not cast to `any` to avoid fixing a real type issue.

### 1.4 - React component style

- Components are function declarations.
- One component per file when exported.
- Component files default-export the component at the bottom.
- Props use `type <ComponentName>Props`.
- Screens use explicit callback props for navigation or submission.
- Avoid `useEffect` for derived state. Use it for real side effects: subscriptions, timers, browser listeners, form reset on loaded data, and external APIs.
- Cleanup timers, listeners, socket subscriptions, and animation loops.

Example:

```tsx
type WelcomeScreenProps = {
    onNext: () => void;
};

function WelcomeScreen({ onNext }: WelcomeScreenProps) {
    return <button onClick={onNext}>Start</button>;
}

export default WelcomeScreen;
```

### 1.5 - Imports and ordering

Group imports as:

1. External packages.
2. Blank line.
3. Internal `@/` imports.
4. Blank line.
5. Same-folder relative imports when needed.

Do not reorder existing imports gratuitously.

### 1.6 - Comments

- Default to no comments.
- Add comments only for non-obvious why: hardware constraints, browser quirks, event contracts, deployment assumptions, or hidden invariants.
- Do not narrate obvious code.
- Do not leave commented-out code.
- Match the surrounding file's comment density.

### 1.7 - Formatting

- Follow the project's Prettier/ESLint config.
- If the project follows the house style, expect 4-space indentation, semicolons, double quotes, and wide line wrapping.
- Do not hand-sort Tailwind classes when a Tailwind Prettier plugin handles it.

### 1.8 - Error handling

- HTTP errors are normalized in `src/services/API.ts`.
- Socket exceptions are handled in `RealtimeProvider` or a dedicated runtime boundary.
- Components consume hooks/providers and render visitor-appropriate feedback.
- Visitor-facing errors are short and actionable.
- Technical details do not belong in public kiosk copy.

### 1.9 - Autofix vs. ask

- Autofix silently: local lint/format issues, obvious typos, missing locale counterparts in files already being edited, small import cleanup.
- Ask before: screen-flow changes, realtime schema shape, event names, kiosk flags, persistent storage, new dependencies, API contracts, public copy strategy, folder structure.

### 1.10 - Do not do

- Do not edit `src/components/ui/` manually.
- Do not edit `src/routeTree.gen.ts`.
- Do not import Axios outside `src/services/API.ts`.
- Do not instantiate Socket.IO outside `src/services/WS.ts`.
- Do not add hover-only interactions.
- Do not store visitor identity or session data persistently without approval.
- Do not add CSS modules, styled-components, or component CSS files.
- Do not add speculative abstractions for future exhibits.

---

## 2 - Naming conventions

### 2.1 - Components and screens

| Aspect | Rule |
| ------ | ---- |
| Component file | `PascalCase.tsx` |
| Component identifier | Same PascalCase name as the file |
| Props type | `<ComponentName>Props` |
| Export | Default export at bottom when file exposes one component |
| Visitor screen | `*Screen.tsx` suffix |
| Flow container | `<Feature>Flow.tsx` |

Examples:

```text
WelcomeScreen.tsx
ScanBadgeScreen.tsx
AttractScreen.tsx
IdleOverlay.tsx
VotingFlow.tsx
```

### 2.2 - Hooks

- File and function both start with `use`.
- Server-state hooks include the action and resource: `useGetQuestions`, `useSubmitVote`.
- Runtime hooks describe the browser/runtime concern: `useKioskMode`, `useIdleReset`, `useAutoSubmitForm`.

### 2.3 - Helpers

| Category | Naming | Export style |
| -------- | ------ | ------------ |
| Standalone helper file | `PascalCase.ts` | Default export only when there is exactly one helper |
| `src/lib/utils.ts` helpers | `camelCase` | Named exports |
| Feature helpers | Match local precedent, prefer named exports for multiple helpers |

### 2.4 - Routes

- Static route segments use kebab-case.
- Dynamic route files use `$param.tsx`.
- Layout files use `_layout.tsx`.
- Route component identifiers are PascalCase.

### 2.5 - Services

| File | Entity | Naming | Export |
| ---- | ------ | ------ | ------ |
| `<Feature>.api.ts` | API functions | camelCase verbs (`getQuestion`, `submitVote`) | named |
| `<Feature>.schemas.ts` | Zod schemas | PascalCase (`VoteSchema`) | named |
| `<Feature>.schemas.ts` | Inferred types | `<SchemaName>Type` (`VoteSchemaType`) | named |
| `<Feature>.schemas.ts` | Defaults | `<SchemaName>InitData` | named |
| `<Feature>.helpers.ts` | Pure utilities | clear verb/noun names | named unless one-helper local pattern |

### 2.6 - Realtime names

- Socket event names are kebab-case string literals.
- Outbound events are imperative (`get-state`, `update-state`).
- Inbound events are nouns or results (`state-updated`, `exception`).
- Realtime schema files use `<Name>.schema.ts`.

### 2.7 - Constants and enums

- Constants use `SCREAMING_SNAKE_CASE` only for true constants.
- Prefer `as const` objects plus derived union types over TypeScript enums unless local precedent uses enums.
- Keep app-wide constants in `src/lib/` or a local `constants` file only when that pattern exists.

---

## 3 - Touch-first UI and styling

### 3.1 - Touch-first rules

- Every primary action must be usable by touch.
- No hover-only controls or hidden hover-only affordances.
- Tap targets must be comfortably sized for fingers.
- Prefer large buttons, segmented controls, sliders, toggles, scan actions, and cards over free-text input.
- Disable or debounce controls that can be double-tapped.
- Keep flows short and obvious for public visitors.

### 3.2 - Public-space accessibility

- Use high contrast and readable type sizes.
- Do not rely on color alone to communicate state.
- Avoid dense paragraphs on visitor-facing screens.
- Copy should be concise; visitors scan.
- Critical controls must remain visible under kiosk viewport constraints.

### 3.3 - Tailwind and shadcn/ui

- Use inline Tailwind utility classes.
- Global styles live in `src/index.css`.
- Use `cn()` helper when composing conditional classes if the project has it.
- Use shadcn/ui primitives from `src/components/ui/`; do not modify them directly.
- Use `lucide-react` icons when an icon is needed.

### 3.4 - Responsive and fixed-installation layout

- Design for the target exhibit viewport first.
- Use stable dimensions for controls, screen regions, media, maps, and cards to avoid layout shifts.
- Check widescreen, portrait, and multi-display assumptions when the deployment uses them.
- Avoid tiny fixed text or controls that become unusable on touch hardware.

### 3.5 - RTL and language-aware layout

- Arabic support means layout must handle RTL.
- Prefer logical spacing utilities and direction-aware placement where available.
- Do not hardcode left/right assumptions into flow logic.
- Keep icon direction meaningful; arrows may need RTL-aware handling.

---

## 4 - Translation

### 4.1 - Supported languages

- English and Arabic are the minimum.
- Additional event locales may be added when required.
- Every locale file follows the same structure and key rules.

### 4.2 - File locations

- Locale files live in `src/locales/`.
- One flat JSON file per language.

### 4.3 - Structure

- Translation files are flat objects.
- No nested objects.
- Keys use kebab-case.

Good:

```json
{
    "start": "Start",
    "scan-badge": "Scan badge",
    "submission-failed": "Something went wrong."
}
```

Bad:

```json
{
    "flow": {
        "start": "Start"
    }
}
```

### 4.4 - Key naming

- Keys are lowercase kebab-case.
- Prefer clear, context-relevant names.
- Reuse one key when the same visible value is used in the same meaning.
- Use suffixes for variants when helpful (`-q`, `-dots`, `-error`).

### 4.5 - Adding or modifying keys

- Add or update keys in every locale file in the same change.
- Keep keys sorted when the existing files are sorted.
- Do not leave duplicate keys.
- Keep visitor copy short.

### 4.6 - Alt text

- Do not internationalize HTML `alt` text by default.
- Use static alt text for meaningful images.
- Use `alt=""` for decorative images.

---

## 5 - Verification

### 5.1 - Definition of done

- Run `npm run lint` and `npm run build` after meaningful code changes when scripts exist.
- For UI changes, open the app and verify the relevant route.
- For touch/kiosk changes, verify the target viewport and interaction path.
- For realtime changes, verify connect, update, broadcast handling, disconnect, and cleanup when possible.
- State any verification you could not run in the final response.

### 5.2 - Browser checks

- Check that text does not overlap or overflow controls.
- Check that controls remain touch-sized.
- Check that loading and error states do not trap the visitor.
- Check that no screen depends on hover to proceed.
