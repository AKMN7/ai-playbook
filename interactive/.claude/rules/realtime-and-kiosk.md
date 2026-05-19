# Realtime and kiosk boundaries

Runtime boundaries for interactive exhibits: WebSocket state sync, realtime providers, auto-submit forms, kiosk browser lockdown, and deployment launchers. Read this before touching sockets, shared state, idle reset, or kiosk behavior.

---

## Part A - Realtime state

### A.1 - Socket singleton (`src/services/WS.ts`)

- A single `socket.io-client` instance is created and exported at module level.
- Endpoint construction lives in `WS.ts`, using `VITE_` env vars.
- Use websocket transport unless the project already requires fallback transports.
- Do not instantiate another socket in a component, hook, context, helper, or service.

Example shape:

```ts
import { io, type Socket } from "socket.io-client";

const ENDPOINT = import.meta.env.VITE_ENV === "PROD" ? import.meta.env.VITE_BE_ENDPOINT : `${import.meta.env.VITE_BE_ENDPOINT}:5000`;

export const socket: Socket = io(ENDPOINT, {
    transports: ["websocket"],
    withCredentials: true,
    autoConnect: true,
    reconnectionAttempts: Infinity
});
```

### A.2 - Event naming

- Socket events use kebab-case strings.
- Outbound client -> server events use imperative verbs: `get-state`, `update-state`, `submit-vote`.
- Inbound server -> client events use nouns or past-tense results: `state-updated`, `vote-submitted`, `exception`.
- Changing an event name is a contract change. Ask before doing it unless the task explicitly asks.

### A.3 - State schemas (`src/services/state/`)

Realtime payloads are runtime contracts and must be validated with Zod.

Every schema file exports:

| Export | Naming | Example |
| ------ | ------ | ------- |
| Zod schema | `<Name>Schema` | `ViewSchema` |
| TS type | `<Name>SchemaType` | `ViewSchemaType` |
| Defaults | `<Name>SchemaInitData` | `ViewSchemaInitData` |

Example:

```ts
import { z } from "zod";

export const ViewSchema = z.object({
    longitude: z.number().min(36).max(55.7),
    latitude: z.number().min(16.4).max(32.2),
    zoom: z.number().min(8).max(16)
});

export type ViewSchemaType = z.infer<typeof ViewSchema>;

export const ViewSchemaInitData: ViewSchemaType = {
    longitude: 46.6753,
    latitude: 24.7136,
    zoom: 8
};
```

Rules:

- Compose sub-schemas into a root `StateSchema` that represents full realtime state.
- Keep defaults next to schemas so reset behavior is explicit.
- Do not allow unvalidated `unknown` socket payloads into React state.
- Ask before changing schema shape when a backend or another display consumes it.

### A.4 - Realtime provider (`src/contexts/realtime-provider.tsx`)

Use the standard context pattern: `RealtimeProvider` plus `useRealtime()`.

Required behavior:

| Concern | Rule |
| ------- | ---- |
| Context value | Include `isConnected`, typed `state`, and `updateState` or domain-specific update methods |
| On connect | Emit `get-state` or the project-specific state bootstrap event |
| Listeners | Register `connect`, `disconnect`, `exception`, and state update events in one effect |
| Cleanup | Remove every listener on unmount |
| Errors | Route socket exceptions to `toast.error()` or the project error surface |
| Memoization | Wrap context value in `useMemo` with complete dependencies |

Rules:

- The provider owns socket listeners. Screens do not register global socket listeners directly.
- Reconnect should request fresh state from the server.
- Use functional state updates when merging partial state to avoid stale closures.
- Keep provider values stable enough to avoid unnecessary full-app re-renders.

### A.5 - State update API

- Prefer one explicit `updateState(nextState)` or narrow domain update functions exposed by the provider.
- Avoid exposing the raw socket to the rest of the app.
- If updates are partial, validate the resulting full state before storing it.
- Debounce high-frequency updates such as sliders, maps, and drag controls.

### A.6 - Auto-submit forms (`src/hooks/useAutoSubmitForm.ts`)

`useAutoSubmitForm` bridges React Hook Form and realtime updates.

Contract:

```ts
type UseAutoSubmitFormOptions<T extends FieldValues> = {
    form: UseFormReturn<T>;
    onSubmit: (data: T) => void;
    enabled: boolean;
    debounceMs?: number;
};
```

Rules:

- Watch all form values through `form.watch()`.
- Debounce before calling `onSubmit`; default delay is 400 ms unless local precedent differs.
- Submit only when all guards pass: `enabled`, `formState.isValid`, and `formState.isDirty`.
- The hook must not import the socket or realtime context. The caller passes `onSubmit`.
- Cleanup timers/subscriptions on unmount.

### A.7 - Realtime data flow

```text
Form/control change
  -> useAutoSubmitForm or explicit submit
  -> updateState()
  -> socket.emit("update-state", state)
  -> server validates/processes
  -> server broadcasts "state-updated"
  -> RealtimeProvider validates payload and stores state
  -> screens render from context
```

Rules:

- Server broadcast is the source of truth after a realtime mutation.
- Do not assume local optimistic state survived reconnect.
- Keep event payloads small and explicit.

---

## Part B - Kiosk and public-display runtime

### B.1 - Kiosk mode hook (`src/hooks/useKioskMode.ts`)

Use one hook, called once at the app root, to suppress browser behaviors that break kiosk use.

Must prevent:

| Behavior | Technique |
| -------- | --------- |
| Right-click menu | `contextmenu` listener with `preventDefault()` |
| Pinch-to-zoom | Block multi-touch `touchstart` with `{ passive: false }` |
| Safari gesture zoom | Block `gesturestart` and `gesturechange` |
| Ctrl-scroll zoom | Block `wheel` when `event.ctrlKey` is true |

Rules:

- Register listeners in one `useEffect`.
- Remove every listener on cleanup.
- Use `{ passive: false }` for listeners that call `preventDefault()`.
- Do not call the hook in multiple screens.

### B.2 - Global CSS lockdown

Global touch/browser constraints live in `src/index.css`.

Expected baseline:

```css
* {
    touch-action: manipulation !important;
    user-select: none !important;
}
```

Rules:

- Keep lockdown rules global only when the app truly runs as an unattended kiosk.
- If a feature needs selectable text or pinch gestures, scope the exception deliberately and document why in code only if it is non-obvious.
- Avoid component-level CSS files for kiosk behavior.

### B.3 - Session lifecycle

- Visitor sessions are ephemeral.
- Idle reset, attract mode, and completion reset must clear all visitor-entered state.
- Persistent identity, remembered visitor data, localStorage/sessionStorage, and long-lived client tokens require explicit approval.
- Flow reset must be callable from one obvious place, usually the feature flow or a session provider.

### B.4 - Browser launcher (`kiosk-launcher.txt`)

When a project includes a Windows Chrome launcher template, keep it at the project root as `kiosk-launcher.txt`.

The launcher should cover:

| Concern | Required behavior |
| ------- | ----------------- |
| Connectivity | Wait for network before launch when the app depends on the backend |
| Stale Chrome | Kill stale Chrome processes before starting a new kiosk session |
| Multi-display | Use one Chrome instance per screen with separate `--user-data-dir` values |
| Window placement | Use explicit `--window-position` for secondary screens |
| Browser chrome | Use `--kiosk` or `--start-fullscreen`, `--no-first-run`, `--disable-infobars` |
| Gesture safety | Include `--overscroll-history-navigation=0` and `--disable-pinch` |
| Popups | Disable translation UI and other browser prompts where possible |

Rules:

- URLs, monitor positions, and screen count are deployment-specific.
- Keep the template aligned with the actual deployment script when one exists.
- Ask before changing kiosk flags unless the task is explicitly about deployment.

### B.5 - Assets and performance

- Heavy images, videos, maps, or 3D assets must be lazy-loaded or preloaded deliberately.
- Avoid layout shifts after asset load.
- Animations must remain smooth on target hardware.
- Prefer CSS transforms and opacity for motion.
- Ask before adding large assets, video backgrounds, 3D libraries, or always-on polling.

### B.6 - Public-space failure modes

- Network, WebSocket, sensor, and scanner failures need a visible recovery path.
- Public copy should be short and non-technical.
- Operator-only details belong in logs, Sentry, devtools, or an operator route.
- When the backend is unavailable, avoid trapping visitors on a broken interactive step without reset/retry.
