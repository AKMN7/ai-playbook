# Feature workflow

How to build interactive features: screen flows, data hooks, forms, realtime updates, feedback states, and session ownership. Read this before adding or changing a visitor flow.

---

## 1 - Before you write code

- Read 2-3 nearby files in the same feature or layer.
- Identify whether the feature is HTTP-driven, realtime-driven, local-only, or a mix.
- Preserve the current visitor journey unless the user explicitly asks to change it.
- Keep screens small and flow state resettable.
- Decide where state belongs before adding new state.

State ownership order:

1. **Server state** -> TanStack Query hooks.
2. **Realtime exhibit state** -> `RealtimeProvider` and `services/state/` schemas.
3. **Current visitor flow state** -> feature flow component or local reducer.
4. **Component-only UI state** -> `useState`.
5. **Persistent browser storage** -> only with explicit approval.

---

## 2 - Layout of an interactive feature

New features should follow this shape unless local precedent says otherwise:

```text
src/components/<feature>/
    <Feature>Flow.tsx
    WelcomeScreen.tsx
    <StepName>Screen.tsx
    CompletionScreen.tsx

src/hooks/<feature>/
    useGet<Feature>.ts
    useSubmit<Feature>.ts

src/services/<feature>/
    <Feature>.api.ts
    <Feature>.schemas.ts
    <Feature>.helpers.ts
```

Optional realtime state lives under `src/services/state/` and is wired through `RealtimeProvider`, not feature-local sockets.

---

## 3 - Screen-flow pattern

Interactive apps are screen-flow driven. A route renders a flow container; the flow container chooses the active screen; each screen renders one visitor step.

| Step | Responsibility | Placement |
| ---- | -------------- | --------- |
| Route | Host route params, wrappers, and the flow container | `src/routes/` |
| Flow | Own active step, reset behavior, session data, cross-screen callbacks | `src/components/<feature>/<Feature>Flow.tsx` |
| Screen | Render one touch-first step and call callbacks | `src/components/<feature>/*Screen.tsx` |
| Schema | Validate input and define defaults | `src/services/<feature>/<Feature>.schemas.ts` |
| Hook | Wrap HTTP or mutation behavior | `src/hooks/<feature>/` |
| API | Make HTTP calls through `API.ts` | `src/services/<feature>/<Feature>.api.ts` |

Rules:

- Screens receive `onNext`, `onBack`, `onReset`, `onComplete`, or explicit domain callbacks through props.
- Screens do not directly mutate route state unless the screen itself is route-specific.
- Resetting the flow should clear visitor-entered data, form dirty state, pending local selections, and transient errors.
- Completion screens should have a deterministic path back to attract/start mode.

---

## 4 - Read flow

All HTTP reads use TanStack Query.

| Step | Responsibility |
| ---- | -------------- |
| API call | Add `getFeature`, `listFeatures`, or a specific fetcher in `<Feature>.api.ts` |
| Query hook | Wrap the fetcher in `useQuery` under `src/hooks/<feature>/` |
| Component | Consume `data`, `isLoading`, `error`, and refetch affordances from the hook |

Rules:

- Query keys must be stable and match mutation invalidation keys.
- Keep query key definitions close to the hook unless the project already centralizes them.
- Components do not call `API.ts` directly.
- Use enabled queries for flow steps that depend on prior visitor input.

---

## 5 - Create and submit flow

Interactive submits usually represent a vote, scan, registration, check-in, or completion event. Use a mutation hook even when the component looks simple.

| Step | Responsibility |
| ---- | -------------- |
| Schema | Define the submitted payload with Zod and export its inferred type |
| API call | Add `submitFeature`, `createFeature`, or the local verb in `<Feature>.api.ts` |
| Mutation hook | Implement `useSubmit<Feature>()` or `useCreate<Feature>()` with `useMutation` |
| Screen/flow | Call the mutation from `handleSubmit` or an explicit touch action |
| Success | Invalidate relevant queries, show feedback, advance the flow, or reset to attract mode |

Rules:

- Keep idempotency in mind for public touchscreens. If duplicate taps are possible, disable the control while pending and make the mutation safe when the API supports it.
- Do not fire a mutation from render.
- Prefer explicit pending states on the initiating control over global spinners.

---

## 6 - Update and delete flows

Updates and deletes follow the same API -> hook -> UI path.

- Update hooks are named for the narrow action when that is clearer (`useUpdateMapView`, `useSetActiveQuestion`).
- Delete hooks use `useDelete<Resource>()`.
- On success, invalidate list/detail query keys or update realtime state intentionally.
- Ask before changing destructive behavior, public confirmation copy, or API contracts.

---

## 7 - Forms

Use React Hook Form plus Zod for any visitor input more complex than one direct button tap.

### 7.1 - Schema-first contract

- Schema lives in `<Feature>.schemas.ts`.
- The form type is inferred with `z.infer<typeof Schema>`.
- Default values are exported next to the schema as `<Feature>SchemaInitData` or a local precedent equivalent.
- Form validation messages use locale keys when shown to visitors.

### 7.2 - Form setup

```tsx
const form = useForm<FormSchemaType>({
    resolver: zodResolver(FormSchema),
    defaultValues: FormSchemaInitData,
    mode: "onChange"
});
```

Rules:

- Use `mode: "onChange"` when auto-submit, live preview, or touch controls depend on current validity.
- Reset the form when the flow resets or the active visitor session ends.
- Do not keep a second unsynchronized copy of form values in component state.

### 7.3 - Touch-first input choices

- Prefer buttons, segmented controls, sliders, toggles, large cards, QR scans, RFID/NFC scans, or camera/scanner events over free text.
- Text input must assume an on-screen keyboard and public-space interruptions.
- Every action must be possible by touch. Hover-only behavior is not acceptable.
- Disable or debounce controls that can be double-tapped into duplicate submissions.

---

## 8 - Realtime feature flow

Use realtime state only when multiple displays, an operator console, sensors, or shared exhibit state need synchronization.

Flow:

```text
Screen/control change
  -> React Hook Form or local flow state
  -> useAutoSubmitForm or explicit submit
  -> updateState()
  -> socket emits update event
  -> server broadcasts updated state
  -> RealtimeProvider updates context
  -> screens re-render from context
```

Rules:

- Do not instantiate sockets inside screens or feature hooks.
- Realtime payloads must be validated by Zod schemas in `services/state/`.
- Keep socket event names stable. Changing them is an API contract change.
- Reconnect should resync state from the server rather than trusting stale local state.

---

## 9 - Loading, error, empty, idle, and attract states

Interactive apps need states visitors can understand quickly.

### 9.1 - Render order

At route or flow level:

1. Critical runtime unavailable state (for example, no realtime connection when the flow requires it).
2. Loading or bootstrapping state.
3. Recoverable error state.
4. Empty/not-ready state.
5. Active screen.

### 9.2 - Loading

- Prefer skeletons or inline pending states that preserve layout.
- Avoid layout shifts after assets load.
- Use large, clear copy for unavoidable waits.

### 9.3 - Error

- Visitor-facing errors should be short and actionable.
- Technical details belong in logs, Sentry, or developer consoles, not public copy.
- Toasts are acceptable for transient non-blocking errors; blocking errors need an on-screen fallback.

### 9.4 - Empty/not-ready

- Empty states should explain what the visitor or operator can do next.
- Avoid dead ends. Provide reset, retry, or return-to-start actions where appropriate.

### 9.5 - Idle and attract mode

- Visitor sessions are ephemeral.
- If idle reset exists, it must reset the full flow from a single point.
- Attract mode should not depend on stale form state from the previous visitor.
- Ask before adding persistent "remember me" or long-lived visitor identity behavior.

---

## 10 - Mutation lifecycle and feedback

- Use `sonner` for non-blocking toasts when local precedent uses toasts.
- Every mutation hook should define success and error behavior.
- Prefer passing callbacks into hooks for flow advancement instead of hardcoding navigation inside generic hooks.

Example pattern:

```ts
export function useSubmitVote(options?: { onSuccess?: () => void }) {
    const queryClient = useQueryClient();

    return useMutation({
        mutationFn: submitVote,
        onSuccess: () => {
            queryClient.invalidateQueries({ queryKey: ["votes"] });
            options?.onSuccess?.();
        },
        onError: (error) => {
            toast.error(error.message || "submission-failed");
        }
    });
}
```

Rules:

- Disable submit controls while pending.
- Avoid duplicate toasts for the same action.
- Invalidate only the query keys affected by the mutation.

---

## 11 - Verification checklist

Before calling interactive work done:

- Run lint/build commands discovered from `package.json` when available.
- Open the app in the browser for layout or interaction changes.
- Check the target viewport(s), including kiosk aspect ratios when known.
- Verify touch-sized controls, no hover-only path, and no obvious overlap.
- For realtime changes, verify connect, disconnect, resync, and cleanup paths when possible.
- For kiosk changes, verify that listeners are registered once and cleaned up.
