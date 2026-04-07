# WebSocket integration rules

These rules define the real-time WebSocket architecture for interactive apps.

## 1 — Socket singleton (`src/services/WS.ts`)

- A **single** `socket.io-client` instance is created and exported at module level.
- Endpoint is resolved from env vars (`VITE_BE_ENDPOINT`, with port logic gated by `VITE_ENV`).
- **Never** instantiate a second socket elsewhere in the app.

```ts
import { io, Socket } from "socket.io-client";

const ENDPOINT = import.meta.env.VITE_ENV === "PROD" ? `${import.meta.env.VITE_BE_ENDPOINT}` : `${import.meta.env.VITE_BE_ENDPOINT}:5000`;

export const socket: Socket = io(ENDPOINT, {
    transports: ["websocket"],
    withCredentials: true,
    autoConnect: true,
    reconnectionAttempts: Infinity
});
```

## 2 — State schemas (`src/services/state/`)

A dedicated `state/` subfolder under `services/` holds all WebSocket state schemas.

Every schema file **must** export three things:

| Export         | Naming                 | Example              |
| -------------- | ---------------------- | -------------------- |
| Zod schema     | `<Name>Schema`         | `ViewSchema`         |
| TS type        | `<Name>SchemaType`     | `ViewSchemaType`     |
| Default values | `<Name>SchemaInitData` | `ViewSchemaInitData` |

```ts
import * as z from "zod";

export const ViewSchema = z.object({
    longitude: z.float32({ error: "longitude-is-required" }).min(36).max(55.7),
    latitude: z.float32({ error: "latitude-is-required" }).min(16.4).max(32.2),
    zoom: z.number({ error: "zoom-is-required" }).min(8).max(16)
});

export type ViewSchemaType = z.infer<typeof ViewSchema>;

export const ViewSchemaInitData: ViewSchemaType = {
    longitude: 46.6753,
    latitude: 24.7136,
    zoom: 8
};
```

- Sub-schemas are **composed** into a single root `StateSchema` that represents the full real-time state.
- File naming: `<Name>.schema.ts` for single-domain schemas, `<Name>.schemas.ts` when a file holds multiple related schemas.

## 3 — Realtime context (`src/contexts/realtime-provider.tsx`)

Follows the standard context pattern: exports `RealtimeProvider` + `useRealtime()`.

| Aspect              | Rule                                                                               |
| ------------------- | ---------------------------------------------------------------------------------- |
| Context value shape | `{ isConnected: boolean; state: StateSchemaType; updateState: (state) => void }`   |
| On `connect`        | Emit `get-state` to fetch current server state.                                    |
| Listeners           | Register `connect`, `disconnect`, `exception`, `state-updated` in one `useEffect`. |
| Cleanup             | Remove **all** listeners on unmount.                                               |
| Error handling      | `exception` event → `toast.error()` (sonner).                                      |
| Memoisation         | Context value wrapped in `useMemo` keyed on `[isConnected, state]`.                |

## 4 — Auto-submit form hook (`src/hooks/useAutoSubmitForm.ts`)

A generic hook that bridges React Hook Form with the realtime context.

| Aspect        | Rule                                                                               |
| ------------- | ---------------------------------------------------------------------------------- |
| Trigger       | Watches all form values via `form.watch()`; debounces before calling `onSubmit`.   |
| Default delay | 400 ms (configurable via `debounceMs`).                                            |
| Guards        | Skips submission unless **all three** are true: `enabled`, `isValid`, `isDirty`.   |
| Decoupling    | The hook does **not** import the socket or context — the caller passes `onSubmit`. |

```ts
type UseAutoSubmitFormOptions<T extends FieldValues> = {
    form: UseFormReturn<T>;
    onSubmit: (data: T) => void;
    enabled: boolean;
    debounceMs?: number;
};
```

## 5 — Event naming convention

- Socket events use **kebab-case** strings.
- **Outbound** (client → server): imperative verb — `get-state`, `update-state`.
- **Inbound** (server → client): past-participle or noun — `state-updated`, `exception`.

## 6 — Data flow

```
Form field change
  → useAutoSubmitForm (debounce, guard: enabled + isValid + isDirty)
    → onSubmit → updateState()
      → socket.emit("update-state", state)
        → server processes → broadcasts "state-updated"
          → socket.on("state-updated") → setState → React re-render
```
